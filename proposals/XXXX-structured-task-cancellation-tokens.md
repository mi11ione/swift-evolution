# Structured Task Cancellation

* Proposal: [SE-XXXX](XXXX-structured-task-cancellation.md)
* Author: [Roman Zhuzhgov](https://github.com/mi11ione)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: TBD
* Review: TBD

## Introduction

This proposal introduces structured cancellation scopes to Swift's concurrency system, enabling context-aware cancellation through composable, domain-specific cancellation primitives. This design provides fine-grained cancellation control while maintaining Swift's structured concurrency principles

## Motivation

Swift's current task cancellation mechanism provides only binary state information through `Task.isCancelled` and `Task.checkCancellation()`. While simple and effective for basic use cases, this design presents critical limitations in production concurrent systems:

### 1. Lack of Cancellation Context

Tasks cannot determine why they were cancelled, making it impossible to implement different cleanup strategies based on cancellation reason:

```swift
// Current approach, no context about cancellation
func processVideo(url: URL) async throws {
    let videoData = try await download(url)
    try Task.checkCancellation() // Was this user cancellation or timeout?
    
    let decoded = try await decode(videoData)
    try Task.checkCancellation() // Should we save partial progress?
    
    try await upload(decoded)
}
```

### 2. Manual Cancellation Propagation

Complex systems require explicit cancellation token management:

```swift
// Current workaround, manual token threading
class VideoProcessor {
    private var downloadTask: Task<Data, Error>?
    private var processTask: Task<Video, Error>?
    
    func process(url: URL) async throws -> Video {
        // Manually coordinate cancellation across tasks
        downloadTask = Task { try await download(url) }
        let data = try await downloadTask!.value
        
        processTask = Task { try await decode(data) }
        return try await processTask!.value
    }
    
    func cancel() {
        downloadTask?.cancel()
        processTask?.cancel()
    }
}
```

### 3. No Composable Cancellation Patterns

Different cancellation conditions cannot be easily combined:

```swift
// Difficult to express: "Cancel if timeout OR memory pressure"
func syncData() async throws {
    // No built-in way to compose multiple cancellation conditions
    // Must manually implement timeout + resource monitoring
}
```

## Proposed solution

I propose introducing structured cancellation scopes that make cancellation context part of the ambient task environment:

```swift
// Timeout cancellation
await withTimeout(.seconds(30)) { timeout in
    let data = try await download(url)
    
    if timeout.isCancelled {
        // Handle timeout specifically
        try await savePartialProgress(data)
        throw TimeoutError()
    }
    
    try await process(data)
}

// Composable cancellation scopes
await withTimeout(.seconds(60)) { timeout in
    await withResourceLimit(.memory(bytes: 500_000_000)) { resource in
        // Both timeout and resource contexts available
        try await processLargeDataset()
    }
}
```

### Core Protocol

```swift
/// Base protocol for scoped contexts
public protocol ScopedContext {
    associatedtype Configuration
    init(config: Configuration)
}

/// Protocol for cancellable scoped contexts
public protocol CancellableContext: ScopedContext {
    associatedtype Reason
    
    /// Whether this context has been cancelled
    var isCancelled: Bool { get }
    
    /// The cancellation reason, if cancelled
    var reason: Reason? { get }
    
    /// Cancel this context with a reason
    func cancel(reason: Reason)
}
```

### Built-in Cancellation Scopes

```swift
/// Timeout cancellation scope
public struct TimeoutContext: CancellableContext {
    public struct Configuration {
        let duration: Duration
    }
    
    public enum Reason {
        case deadline
    }
    
    public var isCancelled: Bool { get }
    public var reason: Reason? { get }
    public var remaining: Duration { get }
    
    public func cancel(reason: Reason) { ... }
}

/// Resource limit cancellation scope  
public struct ResourceLimitContext: CancellableContext {
    public struct Configuration {
        let limit: ResourceLimit
    }
    
    public enum Reason {
        case limitExceeded
        case systemPressure
    }
    
    public enum ResourceLimit {
        case memory(bytes: Int)
        case diskSpace(bytes: Int)
        case cpuUsage(percentage: Double)
    }
    
    public var isCancelled: Bool { get }
    public var reason: Reason? { get }
    public var currentUsage: Double { get }
    
    public func cancel(reason: Reason) { ... }
}

/// Manual cancellation scope
public struct CancellationContext: CancellableContext {
    public struct Configuration {}
    
    public enum Reason {
        case userInitiated
        case custom(String)
    }
    
    public var isCancelled: Bool { get }
    public var reason: Reason? { get }
    
    public func cancel(reason: Reason) { ... }
}
```

### Scope Functions

```swift
/// Execute an operation with a timeout
public func withTimeout<T>(
    _ timeout: Duration,
    operation: (TimeoutContext) async throws -> T
) async rethrows -> T

/// Execute an operation with resource limits
public func withResourceLimit<T>(
    _ limit: ResourceLimitContext.ResourceLimit,
    operation: (ResourceLimitContext) async throws -> T  
) async rethrows -> T

/// Execute an operation with manual cancellation control
public func withCancellation<T>(
    operation: (CancellationContext) async throws -> T
) async rethrows -> T
```

## Detailed design

### Implementation Strategy

Each cancellation scope:
1. Creates a context object with cancellation state
2. Monitors its specific cancellation condition
3. Propagates cancellation to child tasks when triggered
4. Provides context-specific information to the operation
5. Maintains its own cancellation state independent of external task cancellation

```swift
public func withTimeout<T>(
    _ timeout: Duration,
    operation: (TimeoutContext) async throws -> T
) async rethrows -> T {
    let context = TimeoutContext(config: .init(duration: timeout))
    
    return try await withThrowingTaskGroup(of: T.self) { group in
        // Add timeout task
        group.addTask {
            try await Task.sleep(for: timeout)
            context.cancel(reason: .deadline)
            throw CancellationError()
        }
        
        // Add operation task
        group.addTask {
            // Make context available to child tasks via task-local
            try await TimeoutContext.$current.withValue(context) {
                try await operation(context)
            }
        }
        
        // Return first to complete (operation or timeout)
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}
```

### Task-Local Context Propagation

Contexts are propagated to child tasks through task-local values:

```swift
extension TimeoutContext {
    @TaskLocal
    static var current: TimeoutContext?
}

extension ResourceLimitContext {
    @TaskLocal  
    static var current: ResourceLimitContext?
}

extension CancellationContext {
    @TaskLocal
    static var current: CancellationContext?
}
```

This enables child tasks to access parent contexts:

```swift
await withTimeout(.seconds(30)) { timeout in
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            // Child task can access parent timeout context
            if let timeout = TimeoutContext.current {
                print("Remaining time: \(timeout.remaining)")
            }
        }
        
        group.addTask {
            // Deep nesting also works
            await someFunction()
        }
    }
}

func someFunction() async {
    // Even deeply nested functions can check timeout
    if let timeout = TimeoutContext.current, timeout.remaining < .seconds(5) {
        // Skip non-critical work when running out of time
        return
    }
    // ... do work ...
}
```

### Task Cancellation

Scoped contexts maintain their own cancellation state independently from task cancellation. When a task is cancelled externally (via `task.cancel()`), only `Task.isCancelled` becomes true, while context-specific cancellation remains separate:

```swift
extension Task {
    /// Check if cancelled by a specific context type
    static func isCancelled<C: CancellableContext>(
        by contextType: C.Type
    ) -> Bool {
        // Implementation retrieves context from task-local storage
    }
    
    /// Get cancellation reason from a specific context
    static func cancellationReason<C: CancellableContext>(
        from contextType: C.Type
    ) -> C.Reason? {
        // Implementation retrieves reason from task-local storage
    }
}
```

This ensures proper separation of concerns:

```swift
let task = Task {
    await withTimeout(.seconds(30)) { timeout in
        // Cancellation states are independent:
        // - If timeout expires: timeout.isCancelled = true, reason = .deadline
        // - If task.cancel() called: Task.isCancelled = true, but timeout.isCancelled = false, reason = nil
        // - Each cancellation mechanism tracks its own state
        
        if Task.isCancelled {
            // External cancellation
            throw CancellationError()
        }
        
        if timeout.isCancelled {
            // Timeout-specific cancellation
            try await savePartialProgress()
            throw TimeoutError()
        }
    }
}

// External cancellation doesn't affect timeout context state
task.cancel()
```

### Usage Examples

#### Video Processing with Timeout and Cleanup

```swift
func processVideo(url: URL) async throws -> ProcessedVideo {
    try await withTimeout(.seconds(300)) { timeout in
        let rawData = try await download(url)
        
        // Check for timeout before expensive operation
        if timeout.remaining < .seconds(60) {
            throw InsufficientTimeError()
        }
        
        let decoded = try await decode(rawData)
        
        // Different cleanup based on cancellation
        if timeout.isCancelled {
            try await saveIntermediateState(decoded)
            throw TimeoutError()
        }
        
        return try await enhance(decoded)
    }
}
```

#### Composing Multiple Cancellation Conditions

```swift
func syncUserData() async throws {
    try await withTimeout(.seconds(120)) { timeout in
        try await withResourceLimit(.memory(bytes: 500_000_000)) { resource in
            try await withThrowingTaskGroup(of: Void.self) { group in
                group.addTask {
                    // High priority, continue even under pressure
                    try await syncCriticalData()
                }
                
                group.addTask {
                    // Lower priority, skip if resources constrained
                    if resource.currentUsage > 0.8 {
                        print("Skipping photo sync due to memory pressure")
                        return
                    }
                    try await syncPhotos()
                }
                
                group.addTask {
                    // Lowest priority, only if plenty of time
                    if timeout.remaining < .seconds(30) {
                        print("Skipping video sync due to time constraints")  
                        return
                    }
                    try await syncVideos()
                }
                
                try await group.waitForAll()
            }
        }
    }
}
```

#### Manual Cancellation Control

```swift
func downloadWithCancel(url: URL) async throws -> Data {
    try await withCancellation { cancellation in
        let downloadTask = Task {
            try await URLSession.shared.data(from: url).0
        }
        
        await MainActor.run {
            cancelButton.handler = {
                cancellation.cancel(reason: .userInitiated)
                downloadTask.cancel()
            }
        }
        
        do {
            return try await downloadTask.value
        } catch {
            if cancellation.reason == .userInitiated {
                throw UserCancelledError()
            }
            throw error
        }
    }
}
```

#### Custom Cancellation Scope

```swift
// Framework-specific cancellation scope
struct NetworkConditionContext: CancellableContext {
    struct Configuration {
        let requiredCondition: NetworkCondition
    }
    
    enum Reason {
        case offline
        case insufficientBandwidth
        case meteredConnection
    }
    
    var isCancelled: Bool { get }
    var reason: Reason? { get }
    var currentCondition: NetworkCondition { get }
    
    func cancel(reason: Reason) { ... }
}

func withNetworkCondition<T>(
    _ condition: NetworkCondition,
    operation: (NetworkConditionContext) async throws -> T
) async rethrows -> T { ... }

// Usage
try await withNetworkCondition(.wifi) { network in
    if network.currentCondition == .cellular {
        // Reduce quality for cellular
        try await uploadCompressed(data)
    } else {
        try await uploadFullQuality(data)
    }
}
```

## Source compatibility

This proposal is purely additive. All existing code continues to work unchanged

## Effect on ABI stability

The design uses protocol-based extension points and struct-based contexts, allowing future enhancement without ABI breaks

## Effect on API resilience

Protocol-based design allows frameworks to add custom scopes, each scope defines its own reason types, avoiding central coordination and context structs can add new properties without breaking existing code

## Alternatives considered

### Alternative 1: Explicit Token Passing (Original Design)

The initial version of this proposal used explicit cancellation tokens that required manual propagation:

```swift
let token = CancellationToken()
Task(cancellationToken: token) {
    try await process()
}
token.cancel(reason: .timeout)
```

After community feedback highlighting Trio's successful cancellation scope pattern, we pivoted to the ambient scope design which provides better ergonomics, natural composition, and aligns with Swift's structured concurrency principles

### Alternative 2: Extend existing Task cancellation

Adding cancellation reasons directly to Task:

```swift
extension Task {
    func cancel(reason: CancellationReason)
    var cancellationReason: CancellationReason? { get }
}
```

Rejected because it breaks existing Task API contracts and would require significant runtime changes

### Alternative 3: Protocol-Based Cancellation Tokens

Making cancellation tokens protocol-based for extensibility:

```swift
protocol CancellationToken {
    associatedtype Reason
    var reason: Reason? { get }
}
```

Rejected because coordinating cancellation across frameworks becomes complex with existential types

### Alternative 4: Error-Based Cancellation Reasons

Following Go's pattern of using errors for cancellation:

```swift
func cancel(error: Error)
var cancellationError: Error? { get }
```

Rejected because it conflates cancellation (coordination) with errors (failures) and loses type safety

### Alternative 5: Centralized Reason Enum

A single enum for all cancellation reasons:

```swift
enum CancellationReason {
    case timeout
    case userInitiated
    case resourceExhaustion
    case custom(String)
}
```

Rejected because it requires central coordination and frameworks can't add domain-specific reasons cleanly

## Acknowledgments

Thanks to the Swift community for valuable feedback, particularly on the benefits of scoped cancellation over explicit tokens. Special thanks to Joseph Groff for pointing toward Trio's cancellation scope design, which inspired the scoped approach in this proposal