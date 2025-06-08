# Structured Task Cancellation Tokens

* Proposal: [SE-XXXX](XXXX-structured-task-cancellation-tokens.md)
* Author: [Roman Zhuzhgov](https://github.com/mi11ione)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: TBD
* Review: TBD

## Introduction

This proposal introduces structured cancellation tokens to Swift's concurrency system, enabling fine-grained cancellation control with explicit cancellation reasons and partial task cancellation support. This enhancement addresses limitations in the current binary cancellation model while maintaining backward compatibility and Swift's core design principles.

## Motivation

Swift's current task cancellation mechanism provides only binary state information through `Task.isCancelled` and `Task.checkCancellation()`. This design, while simple and effective for basic use cases, presents several critical limitations in production concurrent systems:

### 1. Lack of Cancellation Context

Tasks cannot determine why they were cancelled, making it impossible to implement different cleanup strategies based on cancellation reason. Consider a video processing pipeline:

```swift
// Current approach - no context about cancellation
func processVideo(url: URL) async throws {
    let videoData = try await download(url)
    try Task.checkCancellation() // Was this user cancellation or timeout?
    
    let decoded = try await decode(videoData)
    try Task.checkCancellation() // Should we save partial progress?
    
    let enhanced = try await enhance(decoded)
    try Task.checkCancellation() // Do we need emergency cleanup?
    
    try await upload(enhanced)
}

// Real-world consequences:
// - Timeout cancellation might want to save partial progress
// - User cancellation should clean up immediately
// - Resource exhaustion needs different recovery strategy
```

### 2. All-or-Nothing Cancellation

Complex tasks with multiple components cannot selectively cancel parts of their operation:

```swift
// Current limitation - cancelling everything
func syncUserData() async throws {
    await withThrowingTaskGroup(of: Void.self) { group in
        group.addTask { try await syncPhotos() }      // Want to keep running
        group.addTask { try await syncVideos() }      // Want to cancel (too large)
        group.addTask { try await syncDocuments() }   // Want to keep running
        
        // Currently: cancel() affects all tasks equally
    }
}
```

### 3. Debugging and Observability Challenges

Production systems lack visibility into cancellation flows:

```swift
// Current debugging nightmare
class DownloadManager {
    func debugCancellation() {
        // Who cancelled?
        // When did they cancel?
        // Why did they cancel?
        // What was the call stack?
        // Which subsystems were affected?
        
        // Current answer: ¯\_(ツ)_/¯
        print("Task was cancelled: \(Task.isCancelled)")
    }
}
```

## Proposed solution

I propose introducing a streamlined `CancellationToken` type that provides structured cancellation information:

```swift
public struct CancellationToken: Sendable {
    /// Reasons for task cancellation
    public enum Reason: Sendable, Hashable {
        case userInitiated
        case timeout(TimeInterval)
        case resourceExhaustion(String)
        case parentCancelled
        case custom(String)
    }
    
    /// Create a new cancellation token
    public init()
    
    /// Current cancellation state
    public var isCancelled: Bool { get }
    
    /// Cancellation reason if cancelled
    public var reason: Reason? { get }
    
    /// Cancel with specific reason
    public func cancel(reason: Reason)
}
```

### Task Integration

```swift
public enum CancellationContext {
    /// Current task-local cancellation token
    @TaskLocal
    public static var current: CancellationToken?
}

extension Task where Failure == Error {
    /// Create task with cancellation token
    public init(
        priority: TaskPriority? = nil,
        cancellationToken token: CancellationToken,
        operation: @Sendable @escaping () async throws -> Success
    ) {
        self.init(priority: priority) {
            try await CancellationContext.$current.withValue(token) {
                try await operation()
            }
        }
    }
}

extension Task where Failure == Never {
    /// Create non-throwing task with cancellation token
    public init(
        priority: TaskPriority? = nil,
        cancellationToken token: CancellationToken,
        operation: @Sendable @escaping () async -> Success
    )
}
```

### Context Access

```swift
/// Access current cancellation token, if any
public var currentCancellationToken: CancellationToken? {
    CancellationContext.current
}


/// Execute code with access to cancellation token
public func withCancellationToken<T>(
    _ body: (CancellationToken?) async throws -> T
) async rethrows -> T {
    try await body(CancellationContext.current)
}
```

## Detailed design

### Internal Architecture

```swift
internal final class CancellationTokenStorage: @unchecked Sendable {
    private struct State {
        var isCancelled: Bool = false
        var reason: CancellationToken.Reason?
    }
    
    private var lock = os_unfair_lock()
    private var state = State()
    
    init() {}
    
    var isCancelled: Bool {
        os_unfair_lock_lock(&lock)
        defer { os_unfair_lock_unlock(&lock) }
        return state.isCancelled
    }
    
    var reason: CancellationToken.Reason? {
        os_unfair_lock_lock(&lock)
        defer { os_unfair_lock_unlock(&lock) }
        return state.reason
    }
    
    func cancel(reason: CancellationToken.Reason) {
        os_unfair_lock_lock(&lock)
        defer { os_unfair_lock_unlock(&lock) }
        
        guard !state.isCancelled else { return }
        
        state.isCancelled = true
        state.reason = reason
    }
}
```

### Public API Implementation

```swift
public struct CancellationToken: Sendable {
    internal let storage: CancellationTokenStorage
    
    public init() {
        self.storage = CancellationTokenStorage()
    }
    
    public var isCancelled: Bool {
        storage.isCancelled
    }
    
    public var reason: Reason? {
        storage.reason
    }
    
    public func cancel(reason: Reason) {
        storage.cancel(reason: reason)
    }
}
```

### Task Cancellation Checking

The design extends existing cancellation checking with reason awareness:

```swift
public extension Task {
    /// Check if current task is cancelled with optional reason checking
    static func checkCancellation(
        for expectedReason: CancellationToken.Reason? = nil
    ) throws {
        // Always check task cancellation first
        if Task.isCancelled {
            throw CancellationError()
        }
        
        // Then check token if available
        guard let token = currentCancellationToken else { return }
        
        if let expectedReason = expectedReason {
            // Check for specific reason
            if token.reason == expectedReason {
                throw CancellationError()
            }
        } else if token.isCancelled {
            // Check for any cancellation
            throw CancellationError()
        }
    }
    
    /// Get current cancellation reason if available
    static var cancellationReason: CancellationToken.Reason? {
        currentCancellationToken?.reason
    }
}
```

### Integration with Structured Concurrency

Simple integration with existing concurrency primitives:

```swift
extension TaskGroup {
    /// Add task with cancellation token
    public mutating func addTask(
        priority: TaskPriority? = nil,
        cancellationToken token: CancellationToken? = nil,
        operation: @Sendable @escaping () async throws -> ChildTaskResult
    ) {
        let effectiveToken = token ?? currentCancellationToken
        
        addTask(priority: priority) {
            if let effectiveToken = effectiveToken {
                try await CancellationContext.$current.withValue(effectiveToken) {
                    try await operation()
                }
            } else {
                try await operation()
            }
        }
    }
}
```

### Real-World Usage Examples

With the simplified API, common patterns become straightforward:

```swift
// Example 1: Different cleanup strategies based on cancellation reason
func processVideo(url: URL) async throws {
    let token = CancellationToken()
    
    let task = Task(cancellationToken: token) {
        let videoData = try await download(url)
        
        // Check cancellation with context
        if let reason = Task.cancellationReason {
            switch reason {
            case .timeout:
                // Save partial progress for resume
                try await saveProgress(videoData)
            case .userInitiated:
                // Clean up immediately
                break
            case .resourceExhaustion:
                // Emergency save before cleanup
                try await emergencySave(videoData)
            default:
                break
            }
            throw CancellationError()
        }
        
        let decoded = try await decode(videoData)
        let enhanced = try await enhance(decoded)
        try await upload(enhanced)
    }
    
    // Cancel with reason
    token.cancel(reason: .timeout(30))
}

// Example 2: Task group with cancellation tokens
func syncUserData() async throws {
    let syncToken = CancellationToken()
    
    try await withThrowingTaskGroup(of: Void.self) { group in
        // High priority task
        group.addTask(cancellationToken: syncToken) {
            try await syncPhotos()
        }
        
        // Lower priority task - check for resource exhaustion
        group.addTask(cancellationToken: syncToken) {
            try await withCancellationToken { token in
                if token?.reason == .resourceExhaustion("memory") {
                    print("Skipping video sync due to memory pressure")
                    return
                }
                try await syncVideos()
            }
        }
        
        // Monitor memory pressure
        Task {
            if await isMemoryPressureHigh() {
                syncToken.cancel(reason: .resourceExhaustion("memory"))
            }
        }
        
        try await group.waitForAll()
    }
}

// Example 3: Debugging with cancellation context
class DownloadManager {
    func download(url: URL, token: CancellationToken) async throws -> Data {
        let task = Task(cancellationToken: token) {
            try await URLSession.shared.data(from: url).0
        }
        
        do {
            return try await task.value
        } catch {
            // Rich debugging information
            if let reason = Task.cancellationReason {
                print("Download cancelled: \(reason)")
                // Log additional context based on reason
                switch reason {
                case .timeout(let duration):
                    Logger.shared.log("Download timed out after \(duration)s")
                case .userInitiated:
                    Logger.shared.log("User cancelled download")
                default:
                    Logger.shared.log("Download cancelled: \(reason)")
                }
            }
            throw error
        }
    }
}
```

## Source compatibility

This proposal maintains full source compatibility. All changes are additive and existing code continues to work unchanged.

## Effect on ABI stability
- New types use class-based storage for flexibility
- Public structs have private stored properties allowing future changes

## Effect on API resilience
- `Reason` enum includes `custom` case for extensibility
- Storage is completely private allowing implementation changes

## Alternatives considered

### Alternative 1: Extend existing Task cancellation

I considered adding cancellation reasons directly to Task:

```swift
extension Task {
    func cancel(reason: CancellationReason)
    var cancellationReason: CancellationReason? { get }
}
```

This was rejected, breaks existing Task API contracts, would require significant runtime changes

### Alternative 2: Component-based cancellation

An earlier iteration included complex component-based partial cancellation:

```swift
token.cancel(reason: .timeout, components: ["network", "storage"])
```

This was deferred, API complexity

### Alternative 3: Async observation

I considered async sequences for cancellation observation:

```swift
for await reason in token.cancellations {
    // React to cancellation
}
```

This was rejected because its overly complex for the common case and can easily be built on top of basic API

---
Better to define this in the language itself than leave each team to reinvent cancellation