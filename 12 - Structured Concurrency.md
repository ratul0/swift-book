# Swift Structured Concurrency: Advanced Patterns for Production Apps

Given your strong background in TypeScript and Java, you're already familiar with promises, async/await, and thread management. Swift's structured concurrency takes these concepts further by providing compile-time guarantees about task lifecycles and data races. Let me break down these five essential concepts that will help you build robust, concurrent iOS apps.

## Context & Background

Swift's structured concurrency, introduced in Swift 5.5, solves the same problems you've dealt with in TypeScript and Java but with stricter guarantees. Think of it as combining TypeScript's async/await with Java's ExecutorService, but with compile-time safety that prevents common concurrency bugs.

In your Angular apps, you've used RxJS observables for streaming data and managing async operations. Swift's AsyncSequence is the native equivalent, while AsyncStream bridges the gap between callback-based APIs and modern async/await. Task cancellation works like AbortController in JavaScript but is more deeply integrated. AsyncLet enables parallel execution similar to Promise.all(), and Swift's actor model provides thread safety without manual locking.

In your health tracking app, you'll use these patterns for streaming HealthKit updates, processing large datasets in parallel, syncing with cloud services, and ensuring UI updates happen safely on the main thread.

## Core Understanding

### AsyncSequence and AsyncStream

AsyncSequence is Swift's protocol for asynchronous iteration, similar to AsyncIterable in TypeScript or Java's Stream API with CompletableFuture. The mental model is simple: instead of getting all values at once, you receive them over time, one by one, using async/await syntax.

AsyncStream is a concrete implementation that lets you create AsyncSequences from callback-based APIs or custom logic. Think of it as creating an Observable in RxJS, but with native language support.

### Continuations for Legacy Code

Continuations bridge callback-based APIs to async/await, similar to how you'd wrap a callback in a Promise in TypeScript. Swift provides CheckedContinuation (safe but slower) and UnsafeContinuation (faster but requires careful handling) to convert any asynchronous callback pattern into async/await.

### Task Cancellation and Priority

Tasks in Swift are like futures in Java or promises in TypeScript, but with built-in cancellation support and priority levels. Every task can check if it's been cancelled and respond appropriately, similar to checking an AbortSignal in JavaScript but more integrated into the language.

### AsyncLet and Parallel Execution

AsyncLet creates child tasks that run in parallel, similar to Promise.all() in TypeScript or parallel streams in Java. The key difference is that Swift enforces structured concurrency rules: child tasks must complete before their parent, preventing orphaned operations.

### Thread Safety and Synchronization

Swift's approach to thread safety uses actors (isolated state containers) and Sendable protocols (compile-time thread safety checks) rather than manual locks. This is like combining Java's synchronized blocks with compile-time verification that you're not accidentally sharing mutable state between threads.

## Practical Code Examples

### 1. Basic Example: Understanding the Fundamentals

```swift
import Foundation

// AsyncSequence: Stream health measurements over time
struct HealthMeasurementStream: AsyncSequence {
    typealias Element = Double
    
    // This is like creating an Observable in RxJS
    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator()
    }
    
    struct AsyncIterator: AsyncIteratorProtocol {
        var count = 0
        
        mutating func next() async -> Double? {
            // Simulate fetching data every second
            guard count < 5 else { return nil }
            count += 1
            
            // In TypeScript: await new Promise(resolve => setTimeout(resolve, 1000))
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            
            return Double.random(in: 60...100) // Heart rate
        }
    }
}

// AsyncStream: Bridge callback APIs to async/await
func createHeartRateStream() -> AsyncStream<Double> {
    // Similar to creating a Subject in RxJS
    AsyncStream { continuation in
        // This is where you'd set up your callback-based API
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { timer in
            let heartRate = Double.random(in: 60...100)
            continuation.yield(heartRate)
            
            // After 5 readings, stop
            if timer.fireDate.timeIntervalSinceNow > 5 {
                continuation.finish()
                timer.invalidate()
            }
        }
    }
}

// Continuation: Converting callbacks to async/await
func fetchLegacyHealthData() async throws -> String {
    // Like wrapping a callback in Promise in TypeScript
    return try await withCheckedThrowingContinuation { continuation in
        // Simulating a legacy callback API
        DispatchQueue.global().async {
            Thread.sleep(forTimeInterval: 1)
            // In Java, this would be like CompletableFuture.complete()
            continuation.resume(returning: "Health data loaded")
            // Or for errors: continuation.resume(throwing: error)
        }
    }
}

// Task cancellation and priority
func processHealthData() async throws {
    let task = Task(priority: .high) {
        for i in 0..<1000 {
            // Check cancellation like checking AbortSignal in JS
            try Task.checkCancellation()
            
            // Or manually check without throwing
            if Task.isCancelled {
                print("Task cancelled at iteration \(i)")
                return
            }
            
            // Simulate work
            print("Processing batch \(i)")
            try await Task.sleep(nanoseconds: 100_000_000)
        }
    }
    
    // Cancel after 2 seconds
    Task {
        try await Task.sleep(nanoseconds: 2_000_000_000)
        task.cancel()
    }
    
    try await task.value
}

// AsyncLet for parallel execution
func fetchMultipleDataSources() async -> (steps: Int, calories: Int, distance: Double) {
    // Like Promise.all() in TypeScript
    async let steps = fetchSteps()
    async let calories = fetchCalories()
    async let distance = fetchDistance()
    
    // All three run in parallel, await collects results
    return await (steps, calories, distance)
}

// Helper functions for parallel execution example
func fetchSteps() async -> Int {
    try? await Task.sleep(nanoseconds: 1_000_000_000)
    return 10000
}

func fetchCalories() async -> Int {
    try? await Task.sleep(nanoseconds: 1_500_000_000)
    return 500
}

func fetchDistance() async -> Double {
    try? await Task.sleep(nanoseconds: 800_000_000)
    return 5.2
}

// Thread safety with actors
actor HealthDataCache {
    // This is like a synchronized class in Java
    private var cache: [String: Any] = [:]
    
    func getValue(for key: String) -> Any? {
        cache[key]
    }
    
    func setValue(_ value: Any, for key: String) {
        cache[key] = value
    }
    
    // Actors ensure only one task accesses this at a time
    func clear() {
        cache.removeAll()
    }
}
```

### 2. Real-World Example: Health Tracking App Integration

```swift
import SwiftUI
import HealthKit

// Real-world AsyncStream for continuous health monitoring
class HealthKitManager {
    private let healthStore = HKHealthStore()
    
    // Stream heart rate updates like you'd use an Observable in Angular
    func heartRateStream() -> AsyncStream<Double> {
        AsyncStream { continuation in
            let heartRateType = HKObjectType.quantityType(
                forIdentifier: .heartRate
            )!
            
            // This is the "wrong way" from a Java/TypeScript perspective:
            // DON'T try to use callbacks and manual threading
            // let queue = DispatchQueue(label: "health")
            // queue.async { ... }
            
            // The "Swift way" - use structured concurrency
            let query = HKObserverQuery(
                sampleType: heartRateType,
                predicate: nil
            ) { query, completionHandler, error in
                // Bridge the callback to async stream
                Task {
                    if let samples = await self.fetchLatestHeartRate() {
                        for sample in samples {
                            continuation.yield(sample)
                        }
                    }
                    completionHandler()
                }
            }
            
            // Handle cleanup when stream is terminated
            continuation.onTermination = { @Sendable termination in
                self.healthStore.stop(query)
            }
            
            healthStore.execute(query)
        }
    }
    
    // Converting HealthKit's callback API to async/await
    func requestAuthorization() async throws -> Bool {
        let types = Set([
            HKObjectType.quantityType(forIdentifier: .heartRate)!,
            HKObjectType.quantityType(forIdentifier: .stepCount)!
        ])
        
        // Wrong way (Java/TypeScript thinking):
        // return new Promise((resolve) => { ... })
        
        // Swift way using continuation
        return try await withCheckedThrowingContinuation { continuation in
            healthStore.requestAuthorization(
                toShare: nil,
                read: types
            ) { success, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume(returning: success)
                }
            }
        }
    }
    
    private func fetchLatestHeartRate() async -> [Double]? {
        // Implementation details...
        return [75.0, 76.0, 74.0]
    }
}

// SwiftUI View with proper task management
struct HealthDashboardView: View {
    @StateObject private var viewModel = HealthDashboardViewModel()
    @State private var dataTask: Task<Void, Never>?
    
    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView("Loading health data...")
            } else {
                VStack(spacing: 20) {
                    HealthMetricView(
                        title: "Steps",
                        value: "\(viewModel.steps)"
                    )
                    HealthMetricView(
                        title: "Calories",
                        value: "\(viewModel.calories)"
                    )
                    HealthMetricView(
                        title: "Heart Rate",
                        value: "\(viewModel.heartRate) bpm"
                    )
                }
            }
        }
        .task {
            // This automatically cancels when view disappears
            // Like useEffect cleanup in React
            await viewModel.startMonitoring()
        }
        .onDisappear {
            // Explicit cancellation if needed
            dataTask?.cancel()
        }
    }
}

// ViewModel with parallel data fetching
@MainActor
class HealthDashboardViewModel: ObservableObject {
    @Published var steps = 0
    @Published var calories = 0
    @Published var heartRate = 0
    @Published var isLoading = true
    
    private let healthManager = HealthKitManager()
    private var monitoringTask: Task<Void, Never>?
    
    func startMonitoring() async {
        // Parallel fetching like Promise.all()
        async let todaySteps = fetchTodaySteps()
        async let todayCalories = fetchTodayCalories()
        async let currentHeartRate = fetchCurrentHeartRate()
        
        // Wait for all three (structured concurrency ensures they complete)
        let (steps, calories, heartRate) = await (
            todaySteps,
            todayCalories,
            currentHeartRate
        )
        
        self.steps = steps
        self.calories = calories
        self.heartRate = heartRate
        self.isLoading = false
        
        // Start continuous monitoring
        monitoringTask = Task {
            for await heartRate in healthManager.heartRateStream() {
                // Check if task is cancelled
                guard !Task.isCancelled else { break }
                
                // Update on main thread (like zone.run() in Angular)
                self.heartRate = Int(heartRate)
            }
        }
    }
    
    func stopMonitoring() {
        monitoringTask?.cancel()
    }
    
    private func fetchTodaySteps() async -> Int {
        // Simulate API call
        try? await Task.sleep(nanoseconds: 1_000_000_000)
        return 8543
    }
    
    private func fetchTodayCalories() async -> Int {
        try? await Task.sleep(nanoseconds: 1_200_000_000)
        return 1850
    }
    
    private func fetchCurrentHeartRate() async -> Int {
        try? await Task.sleep(nanoseconds: 800_000_000)
        return 72
    }
}

struct HealthMetricView: View {
    let title: String
    let value: String
    
    var body: some View {
        VStack {
            Text(title)
                .font(.caption)
            Text(value)
                .font(.title)
                .bold()
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(Color.gray.opacity(0.1))
        .cornerRadius(10)
    }
}

// Actor for thread-safe data caching
actor HealthDataStore {
    private var cache: [Date: HealthSnapshot] = [:]
    private var observers: [UUID: (HealthSnapshot) -> Void] = [:]
    
    struct HealthSnapshot {
        let steps: Int
        let calories: Int
        let heartRate: Int
        let timestamp: Date
    }
    
    func store(_ snapshot: HealthSnapshot) {
        cache[snapshot.timestamp] = snapshot
        
        // Notify observers (thread-safe)
        for observer in observers.values {
            observer(snapshot)
        }
    }
    
    func getSnapshots(from startDate: Date, to endDate: Date) -> [HealthSnapshot] {
        cache.compactMap { date, snapshot in
            date >= startDate && date <= endDate ? snapshot : nil
        }
    }
    
    func subscribe(_ observer: @escaping (HealthSnapshot) -> Void) -> UUID {
        let id = UUID()
        observers[id] = observer
        return id
    }
    
    func unsubscribe(_ id: UUID) {
        observers[id] = nil
    }
}
```

### 3. Production Example: Complete Health Sync System

```swift
import SwiftUI
import HealthKit
import SwiftData

// Production-grade health sync system with all concepts
final class HealthSyncService {
    enum SyncError: LocalizedError {
        case authorizationDenied
        case noDataAvailable
        case syncCancelled
        case networkError(underlying: Error)
        
        var errorDescription: String? {
            switch self {
            case .authorizationDenied:
                return "Health data access denied"
            case .noDataAvailable:
                return "No health data available"
            case .syncCancelled:
                return "Sync was cancelled"
            case .networkError(let error):
                return "Network error: \(error.localizedDescription)"
            }
        }
    }
    
    private let healthStore = HKHealthStore()
    private let syncQueue = TaskQueue()
    private let dataStore: HealthDataStore
    
    // Track active syncs for cancellation
    private var activeSyncTasks: [UUID: Task<Void, Error>] = [:]
    
    init(dataStore: HealthDataStore) {
        self.dataStore = dataStore
    }
    
    // Production AsyncStream with proper error handling and backpressure
    func continuousHealthStream(
        types: Set<HKQuantityType>,
        updateInterval: TimeInterval = 60
    ) -> AsyncThrowingStream<HealthUpdate, Error> {
        AsyncThrowingStream { continuation in
            let syncId = UUID()
            
            // Set up buffering strategy for backpressure
            continuation.onTermination = { @Sendable [weak self] _ in
                Task { @MainActor in
                    self?.activeSyncTasks[syncId]?.cancel()
                    self?.activeSyncTasks[syncId] = nil
                }
            }
            
            // Create monitoring task with proper priority
            let task = Task(priority: .userInitiated) {
                do {
                    // Request authorization first
                    try await self.requestHealthKitAuthorization(types: types)
                    
                    // Set up continuous monitoring
                    await withThrowingTaskGroup(of: Void.self) { group in
                        for type in types {
                            group.addTask {
                                try await self.monitorHealthType(
                                    type,
                                    continuation: continuation,
                                    updateInterval: updateInterval
                                )
                            }
                        }
                        
                        // Handle cancellation properly
                        try await group.waitForAll()
                    }
                } catch {
                    continuation.finish(throwing: error)
                }
            }
            
            activeSyncTasks[syncId] = task
        }
    }
    
    // Production-grade continuation usage with timeout and retry
    private func requestHealthKitAuthorization(
        types: Set<HKQuantityType>
    ) async throws {
        // Add timeout to prevent hanging
        let authTask = Task {
            try await withCheckedThrowingContinuation { continuation in
                healthStore.requestAuthorization(
                    toShare: nil,
                    read: types
                ) { success, error in
                    if let error = error {
                        continuation.resume(throwing: SyncError.networkError(underlying: error))
                    } else if !success {
                        continuation.resume(throwing: SyncError.authorizationDenied)
                    } else {
                        continuation.resume()
                    }
                }
            }
        }
        
        // Race between auth and timeout
        let timeoutTask = Task {
            try await Task.sleep(nanoseconds: 10_000_000_000) // 10 seconds
            authTask.cancel()
            throw SyncError.networkError(
                underlying: URLError(.timedOut)
            )
        }
        
        // First one to complete wins
        _ = try await withThrowingTaskGroup(of: Void.self) { group in
            group.addTask { try await authTask.value }
            group.addTask { try await timeoutTask.value }
            
            // Get first result and cancel the other
            let result = try await group.next()
            group.cancelAll()
            return result
        }
    }
    
    // Advanced parallel execution with error recovery
    func performComprehensiveSync() async throws -> SyncResult {
        // Use TaskGroup for dynamic parallelism
        try await withThrowingTaskGroup(of: PartialSyncResult.self) { group in
            // Add tasks dynamically based on available data types
            let dataTypes = try await getAvailableDataTypes()
            
            for dataType in dataTypes {
                group.addTask(priority: self.priorityForDataType(dataType)) {
                    // Each task has its own error handling
                    do {
                        return try await self.syncDataType(dataType)
                    } catch {
                        // Log error but don't fail entire sync
                        print("Failed to sync \(dataType): \(error)")
                        return PartialSyncResult(
                            type: dataType,
                            success: false,
                            error: error
                        )
                    }
                }
            }
            
            // Collect results with cancellation check
            var results: [PartialSyncResult] = []
            for try await result in group {
                try Task.checkCancellation()
                results.append(result)
            }
            
            return SyncResult(partialResults: results)
        }
    }
    
    // Thread-safe task queue using Actor
    private func monitorHealthType(
        _ type: HKQuantityType,
        continuation: AsyncThrowingStream<HealthUpdate, Error>.Continuation,
        updateInterval: TimeInterval
    ) async throws {
        // Create a timer-based stream
        let timer = AsyncStream<Date> { timerContinuation in
            let timer = Timer.scheduledTimer(
                withTimeInterval: updateInterval,
                repeats: true
            ) { _ in
                timerContinuation.yield(Date())
            }
            
            timerContinuation.onTermination = { @Sendable _ in
                timer.invalidate()
            }
        }
        
        // Process updates with cancellation support
        for await _ in timer {
            guard !Task.isCancelled else {
                throw SyncError.syncCancelled
            }
            
            let samples = try await fetchSamples(for: type)
            let update = HealthUpdate(
                type: type.identifier,
                samples: samples,
                timestamp: Date()
            )
            
            continuation.yield(update)
        }
    }
    
    private func fetchSamples(
        for type: HKQuantityType
    ) async throws -> [HealthSample] {
        // Implementation with proper error handling
        []
    }
    
    private func syncDataType(
        _ type: String
    ) async throws -> PartialSyncResult {
        // Simulate sync with random delay
        let delay = UInt64.random(in: 500_000_000...2_000_000_000)
        try await Task.sleep(nanoseconds: delay)
        
        return PartialSyncResult(
            type: type,
            success: true,
            error: nil
        )
    }
    
    private func getAvailableDataTypes() async throws -> [String] {
        ["steps", "heartRate", "calories", "distance"]
    }
    
    private func priorityForDataType(_ type: String) -> TaskPriority {
        switch type {
        case "heartRate": return .high
        case "steps", "calories": return .medium
        default: return .low
        }
    }
}

// Thread-safe task queue using Actor
actor TaskQueue {
    private var tasks: [UUID: Task<Void, Error>] = [:]
    private var concurrentLimit = 3
    private var runningCount = 0
    
    func enqueue(
        priority: TaskPriority = .medium,
        operation: @escaping () async throws -> Void
    ) async -> UUID {
        let id = UUID()
        
        // Wait if we're at capacity
        while runningCount >= concurrentLimit {
            try? await Task.sleep(nanoseconds: 100_000_000)
        }
        
        runningCount += 1
        
        let task = Task(priority: priority) {
            defer {
                Task { await self.taskCompleted() }
            }
            try await operation()
        }
        
        tasks[id] = task
        return id
    }
    
    func cancel(_ id: UUID) {
        tasks[id]?.cancel()
        tasks[id] = nil
    }
    
    func cancelAll() {
        tasks.values.forEach { $0.cancel() }
        tasks.removeAll()
        runningCount = 0
    }
    
    private func taskCompleted() {
        runningCount -= 1
    }
}

// Supporting types
struct HealthUpdate {
    let type: String
    let samples: [HealthSample]
    let timestamp: Date
}

struct HealthSample {
    let value: Double
    let timestamp: Date
}

struct PartialSyncResult {
    let type: String
    let success: Bool
    let error: Error?
}

struct SyncResult {
    let partialResults: [PartialSyncResult]
    
    var successRate: Double {
        let successful = partialResults.filter { $0.success }.count
        return Double(successful) / Double(partialResults.count)
    }
}

// SwiftUI integration with proper lifecycle management
struct HealthSyncView: View {
    @StateObject private var syncManager = SyncManager()
    @State private var syncTask: Task<Void, Never>?
    
    var body: some View {
        VStack(spacing: 20) {
            SyncStatusView(status: syncManager.syncStatus)
            
            ForEach(syncManager.dataStreams, id: \.id) { stream in
                DataStreamView(stream: stream)
            }
            
            HStack {
                Button("Start Sync") {
                    syncTask = Task {
                        await syncManager.startContinuousSync()
                    }
                }
                .disabled(syncManager.isSyncing)
                
                Button("Cancel") {
                    syncTask?.cancel()
                    syncManager.cancelSync()
                }
                .disabled(!syncManager.isSyncing)
            }
        }
        .task {
            // Auto-start sync when view appears
            await syncManager.initialize()
        }
        .onDisappear {
            // Clean up when view disappears
            syncTask?.cancel()
        }
    }
}

@MainActor
class SyncManager: ObservableObject {
    @Published var syncStatus: SyncStatus = .idle
    @Published var dataStreams: [DataStream] = []
    @Published var isSyncing = false
    
    private let syncService = HealthSyncService(
        dataStore: HealthDataStore()
    )
    private var streamTask: Task<Void, Error>?
    
    enum SyncStatus {
        case idle
        case syncing(progress: Double)
        case completed(successRate: Double)
        case failed(error: Error)
    }
    
    struct DataStream: Identifiable {
        let id = UUID()
        let type: String
        var latestValue: Double?
        var lastUpdated: Date?
    }
    
    func initialize() async {
        // Set up initial state
    }
    
    func startContinuousSync() async {
        isSyncing = true
        syncStatus = .syncing(progress: 0)
        
        streamTask = Task {
            do {
                let types = Set([
                    HKObjectType.quantityType(forIdentifier: .heartRate)!,
                    HKObjectType.quantityType(forIdentifier: .stepCount)!
                ])
                
                for try await update in syncService.continuousHealthStream(types: types) {
                    processUpdate(update)
                }
            } catch {
                syncStatus = .failed(error: error)
            }
            isSyncing = false
        }
    }
    
    func cancelSync() {
        streamTask?.cancel()
        isSyncing = false
        syncStatus = .idle
    }
    
    private func processUpdate(_ update: HealthUpdate) {
        // Update UI with new data
        if let index = dataStreams.firstIndex(where: { $0.type == update.type }) {
            dataStreams[index].latestValue = update.samples.last?.value
            dataStreams[index].lastUpdated = Date()
        } else {
            var newStream = DataStream(type: update.type)
            newStream.latestValue = update.samples.last?.value
            newStream.lastUpdated = Date()
            dataStreams.append(newStream)
        }
    }
}

struct SyncStatusView: View {
    let status: SyncManager.SyncStatus
    
    var body: some View {
        switch status {
        case .idle:
            Text("Ready to sync")
        case .syncing(let progress):
            ProgressView(value: progress)
                .progressViewStyle(.linear)
        case .completed(let rate):
            Text("Sync completed: \(rate * 100, specifier: "%.0f")% success")
        case .failed(let error):
            Text("Sync failed: \(error.localizedDescription)")
                .foregroundColor(.red)
        }
    }
}

struct DataStreamView: View {
    let stream: SyncManager.DataStream
    
    var body: some View {
        HStack {
            Text(stream.type)
            Spacer()
            if let value = stream.latestValue {
                Text("\(value, specifier: "%.1f")")
            }
            if let date = stream.lastUpdated {
                Text(date, style: .time)
                    .font(.caption)
            }
        }
    }
}
```

## Common Pitfalls & Best Practices

Coming from TypeScript and Java, you'll likely make these mistakes initially:

**AsyncSequence Pitfalls:**
The biggest mistake is treating AsyncSequence like a regular array or trying to use forEach. Remember that each iteration is async, so you must use `for await`. Also, unlike RxJS observables, AsyncSequences are pull-based, not push-based, meaning the consumer controls the pace of iteration.

**Continuation Pitfalls:**
The most dangerous mistake is calling resume() multiple times on a continuation, which will crash your app. Each continuation must be resumed exactly once. Also, forgetting to resume a continuation creates a permanent hang. Always use withCheckedContinuation during development to catch these errors.

**Task Cancellation Pitfalls:**
Unlike AbortController in JavaScript, Swift's task cancellation is cooperative. Simply calling cancel() doesn't stop the task; the task must check for cancellation and respond appropriately. Also, cancellation propagates to child tasks automatically, which can be surprising if you're used to manual propagation.

**Parallel Execution Pitfalls:**
The async let syntax creates child tasks that must complete before the parent. If you forget to await all async let bindings, you'll get a compile error. This is stricter than Promise.all() where you can ignore the result.

**Thread Safety Pitfalls:**
Coming from Java's synchronized blocks, you might try to use locks or semaphores. Don't. Use actors instead. Also, remember that @MainActor is like Android's runOnUiThread() but enforced at compile time.

**Memory and Performance Implications:**
Task creation has overhead, so don't create thousands of tiny tasks. Use TaskGroup for dynamic task creation. AsyncStreams buffer values by default, which can cause memory issues with large datasets. Consider using AsyncStream.makeStream with custom buffering strategies for production apps.

## Integration with SwiftUI & iOS Development

SwiftUI's task modifier automatically manages task lifecycle, cancelling tasks when views disappear. This is similar to useEffect cleanup in React but more automatic. Always use .task for async work in views rather than creating tasks in onAppear.

For HealthKit integration, always convert their callback-based APIs to async/await using continuations. This makes your code cleaner and easier to test. SwiftData queries can be made async using actors to ensure thread safety when accessing the model context.

Here's a complete example integrating all concepts with SwiftUI:

```swift
struct HealthMonitorView: View {
    @StateObject private var monitor = HealthMonitor()
    
    var body: some View {
        List {
            ForEach(monitor.readings, id: \.timestamp) { reading in
                HStack {
                    Text(reading.type)
                    Spacer()
                    Text("\(reading.value, specifier: "%.1f")")
                }
            }
        }
        .task {
            // This task is automatically cancelled when view disappears
            await monitor.startMonitoring()
        }
        .refreshable {
            // Pull-to-refresh with async support
            await monitor.refresh()
        }
    }
}

@MainActor
class HealthMonitor: ObservableObject {
    @Published var readings: [HealthReading] = []
    private var monitoringTask: Task<Void, Never>?
    
    struct HealthReading {
        let type: String
        let value: Double
        let timestamp: Date
    }
    
    func startMonitoring() async {
        // Create a long-running task for monitoring
        monitoringTask = Task {
            await withTaskGroup(of: Void.self) { group in
                group.addTask { await self.monitorHeartRate() }
                group.addTask { await self.monitorSteps() }
                group.addTask { await self.monitorCalories() }
            }
        }
    }
    
    func refresh() async {
        // Cancel existing monitoring
        monitoringTask?.cancel()
        
        // Fetch latest data in parallel
        async let heartRate = fetchLatestHeartRate()
        async let steps = fetchLatestSteps()
        async let calories = fetchLatestCalories()
        
        // Update UI with all results
        let results = await [heartRate, steps, calories]
        readings = results
        
        // Restart monitoring
        await startMonitoring()
    }
    
    private func monitorHeartRate() async {
        // Implementation using AsyncStream
    }
    
    private func monitorSteps() async {
        // Implementation
    }
    
    private func monitorCalories() async {
        // Implementation
    }
    
    private func fetchLatestHeartRate() async -> HealthReading {
        HealthReading(type: "Heart Rate", value: 72, timestamp: Date())
    }
    
    private func fetchLatestSteps() async -> HealthReading {
        HealthReading(type: "Steps", value: 8500, timestamp: Date())
    }
    
    private func fetchLatestCalories() async -> HealthReading {
        HealthReading(type: "Calories", value: 1850, timestamp: Date())
    }
}
```

## Production Considerations

**Testing Strategies:**
Test async code using XCTest's async support. Create mock AsyncSequences for testing streams, use continuation-based mocks for callback APIs, and test cancellation by explicitly cancelling tasks and verifying cleanup. Always test with both CheckedContinuation (debug) and UnsafeContinuation (release) to ensure proper behavior.

**Debugging Techniques:**
Use Instruments' Concurrency template to visualize task relationships and identify bottlenecks. Set breakpoints in async functions just like synchronous code. The Xcode debugger shows the task hierarchy and helps identify retain cycles in captured closures. Use os_signpost for performance profiling of async operations.

**iOS Version Considerations:**
All these features require iOS 15+ for basic support, but iOS 17+ provides improved performance and additional async algorithms. The Swift 6 concurrency model (iOS 18+) adds complete data race safety. For your iOS 17+ target, you have access to all modern concurrency features including async algorithms package.

**Performance Optimization:**
Batch operations when possible rather than creating individual tasks. Use TaskLocal for passing context through async call chains without parameters. Consider using AsyncStream.makeStream for custom buffering strategies. Profile with Instruments to identify unnecessary task creation or excessive context switching.

## Exercises for Mastery

### Exercise 1: Observable to AsyncSequence Migration
Create an AsyncSequence that mimics an RxJS Observable you're familiar with. Implement a heart rate monitor that emits values every second, supports filtering (only emit if change > 5 bpm), and handles backpressure by dropping old values if the consumer is slow.

Start with this TypeScript/RxJS pattern and convert it:
```typescript
interval(1000).pipe(
  map(() => getHeartRate()),
  filter(rate => Math.abs(rate - lastRate) > 5),
  bufferTime(5000)
)
```

### Exercise 2: Parallel Data Aggregation
Build a daily health summary that fetches data from multiple sources in parallel. Implement proper cancellation, timeout handling (10 seconds max), and partial failure recovery. The summary should include steps, calories, distance, active minutes, and heart rate zones, each fetched from different "APIs" with varying delays.

### Exercise 3: Production Sync System
Create a complete sync system for your health app that continuously syncs local changes to a "cloud" service (mock this), handles offline scenarios with retry logic, implements exponential backoff for failures, and provides real-time progress updates to the UI. Use actors for thread-safe state management and AsyncStream for progress reporting.

Requirements:
- Sync queue that processes items one at a time
- Automatic retry with exponential backoff
- Progress reporting via AsyncStream
- Cancellation support at any point
- Proper error recovery and logging

These exercises will help you transition from thinking in promises and observables to truly understanding Swift's structured concurrency model, preparing you to build robust, production-ready iOS apps that handle complex asynchronous operations safely and efficiently.

