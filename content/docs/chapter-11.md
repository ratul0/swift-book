---
title: "Chapter 11"
weight: 110
---

# Swift Modern Concurrency: Complete Guide for Production iOS Development

## Context & Background

Swift's modern concurrency system, introduced in Swift 5.5, provides structured, safe, and efficient asynchronous programming. Think of it as Swift's answer to JavaScript's async/await combined with Kotlin coroutines, but with stronger compile-time safety guarantees and built-in data race prevention.

Coming from your background, here's how Swift concurrency maps to what you know:

**From TypeScript/JavaScript:** You're familiar with Promises and async/await. Swift's async/await works similarly for handling asynchronous operations, but instead of Promises, Swift uses Tasks (more like Kotlin's coroutines). The key difference is that Swift's concurrency is deeply integrated with the type system to prevent data races at compile time.

**From Java:** If you've used CompletableFuture or virtual threads (Project Loom), Swift's Task system is conceptually similar but with automatic cancellation propagation and structured concurrency by default. Unlike Java's thread-based model, Swift uses cooperative multitasking with continuations.

**From Kotlin:** This will feel most familiar! Swift's async/await is very similar to Kotlin coroutines with suspend functions. Tasks are like coroutines, Task Groups are like coroutineScope, and Actors are similar to Kotlin's experimental actors but more mature and integrated into the language.

**In production iOS apps, you'll use this for:**
- Network requests to health APIs or backend services
- HealthKit data fetching (which can be slow with large datasets)
- SwiftData queries and migrations
- Image processing or heavy computations
- Coordinating multiple async operations (like syncing health data while updating UI)

## Core Understanding

Let's break down Swift's concurrency into its fundamental building blocks:

### The Mental Model

Think of Swift concurrency as a cooperative system where:
1. **Async functions** are like toll roads - you must "pay" (await) to use them
2. **Tasks** are vehicles carrying your code execution
3. **Actors** are thread-safe parking garages where only one car can enter at a time
4. **MainActor** is the special VIP lane reserved for UI updates
5. **Sendable** is like a safety inspection ensuring data can cross thread boundaries

### Basic Syntax and Concepts

```swift
// Async function declaration - like TypeScript's async functions
func fetchHealthData() async throws -> [HealthRecord] {
    // Implementation
}

// Calling async functions - must use await
let records = try await fetchHealthData()

// Task creation - like starting a new Promise or coroutine
Task {
    await doSomethingAsync()
}

// Actor declaration - thread-safe by design
actor DataManager {
    private var cache: [String: Data] = [:]
    
    func store(data: Data, key: String) {
        cache[key] = data
    }
}
```

## Practical Code Examples

### 1. Basic Example: Simple Async Operations

```swift
import Foundation

// Basic async function - similar to TypeScript's async function
func fetchUserProfile(id: String) async throws -> UserProfile {
    // This is like fetch() in TypeScript but with native Swift types
    let url = URL(string: "https://api.example.com/users/\(id)")!
    
    // URLSession's data method is async - no callbacks needed!
    let (data, response) = try await URLSession.shared.data(from: url)
    
    // Check response status
    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw NetworkError.invalidResponse
    }
    
    // Decode JSON - similar to what you'd do in TypeScript
    let decoder = JSONDecoder()
    return try decoder.decode(UserProfile.self, from: data)
}

// WRONG WAY (TypeScript/JavaScript thinking):
// Trying to use .then() or callbacks
func fetchUserProfileWrong(id: String, completion: @escaping (UserProfile?) -> Void) {
    // This is the old way - avoid this pattern!
    URLSession.shared.dataTask(with: URL(string: "...")!) { data, _, _ in
        // Callback hell starts here...
        completion(nil)
    }.resume()
}

// SWIFT WAY - Using async/await with proper error handling:
func loadUserData() async {
    do {
        // Clean, linear flow - no callback pyramids
        let profile = try await fetchUserProfile(id: "123")
        print("Loaded profile for \(profile.name)")
    } catch {
        // Centralized error handling
        print("Failed to load profile: \(error)")
    }
}

// Creating and managing Tasks
class ProfileViewModel {
    private var loadTask: Task<Void, Never>?
    
    func startLoading() {
        // Cancel previous task if exists (like RxJS subscription management)
        loadTask?.cancel()
        
        // Create new task - similar to launching a coroutine in Kotlin
        loadTask = Task {
            await loadUserData()
        }
    }
    
    deinit {
        // Clean up - important for preventing memory leaks
        loadTask?.cancel()
    }
}
```

### 2. Real-World Example: Health Data Synchronization

```swift
import HealthKit
import SwiftData

// Actor for thread-safe health data management
actor HealthDataSynchronizer {
    private let healthStore = HKHealthStore()
    private let modelContext: ModelContext
    private var lastSyncDate: Date = .distantPast
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    // Main synchronization function using async/await
    func synchronizeHealthData() async throws -> SyncResult {
        // Check authorization first
        guard HKHealthStore.isHealthDataAvailable() else {
            throw HealthError.notAvailable
        }
        
        // Fetch multiple data types concurrently using TaskGroup
        let syncResult = try await withThrowingTaskGroup(of: PartialSyncResult.self) { group in
            // Add tasks for different health metrics
            group.addTask { [weak self] in
                guard let self = self else { throw HealthError.deallocated }
                return try await self.syncHeartRateData()
            }
            
            group.addTask { [weak self] in
                guard let self = self else { throw HealthError.deallocated }
                return try await self.syncStepData()
            }
            
            group.addTask { [weak self] in
                guard let self = self else { throw HealthError.deallocated }
                return try await self.syncSleepData()
            }
            
            // Collect results as they complete
            var results = SyncResult()
            for try await partialResult in group {
                // This processes results as soon as each task completes
                // Similar to Promise.allSettled() but with streaming results
                results.merge(partialResult)
            }
            
            return results
        }
        
        // Update last sync date
        lastSyncDate = Date()
        
        // Save to SwiftData on the main thread
        await MainActor.run {
            try? modelContext.save()
        }
        
        return syncResult
    }
    
    // Individual sync functions
    private func syncHeartRateData() async throws -> PartialSyncResult {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        
        // Create query predicate
        let predicate = HKQuery.predicateForSamples(
            withStart: lastSyncDate,
            end: Date(),
            options: .strictStartDate
        )
        
        // Use async wrapper for HealthKit query
        let samples = try await queryHealthKit(type: heartRateType, predicate: predicate)
        
        // Process samples
        let records = samples.map { sample in
            HeartRateRecord(
                value: sample.quantity.doubleValue(for: .beatsPerMinute()),
                date: sample.startDate
            )
        }
        
        return PartialSyncResult(heartRateRecords: records)
    }
    
    // Async wrapper for HealthKit queries using continuations
    private func queryHealthKit(
        type: HKSampleType,
        predicate: NSPredicate
    ) async throws -> [HKQuantitySample] {
        // Bridge callback-based API to async/await
        return try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: type,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: nil
            ) { _, samples, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    let quantitySamples = samples as? [HKQuantitySample] ?? []
                    continuation.resume(returning: quantitySamples)
                }
            }
            
            healthStore.execute(query)
        }
    }
    
    private func syncStepData() async throws -> PartialSyncResult {
        // Similar implementation for steps
        fatalError("Implement step syncing")
    }
    
    private func syncSleepData() async throws -> PartialSyncResult {
        // Similar implementation for sleep
        fatalError("Implement sleep syncing")
    }
}

// SwiftUI View using the synchronizer
struct HealthSyncView: View {
    @State private var isSyncing = false
    @State private var syncError: Error?
    @State private var lastSyncResult: SyncResult?
    
    // Create synchronizer as a state object
    private let synchronizer: HealthDataSynchronizer
    
    init(modelContext: ModelContext) {
        self.synchronizer = HealthDataSynchronizer(modelContext: modelContext)
    }
    
    var body: some View {
        VStack {
            if isSyncing {
                ProgressView("Syncing health data...")
                    .progressViewStyle(CircularProgressViewStyle())
            } else {
                Button("Sync Health Data") {
                    syncHealthData()
                }
                .buttonStyle(.borderedProminent)
            }
            
            if let result = lastSyncResult {
                Text("Synced \(result.totalRecords) records")
                    .foregroundColor(.green)
            }
            
            if let error = syncError {
                Text("Error: \(error.localizedDescription)")
                    .foregroundColor(.red)
            }
        }
        .padding()
    }
    
    private func syncHealthData() {
        // Launch async work from synchronous context
        Task {
            // Automatically runs on MainActor because View is @MainActor
            isSyncing = true
            syncError = nil
            
            do {
                // Call actor method - automatically switches context
                let result = try await synchronizer.synchronizeHealthData()
                
                // Back on MainActor automatically for UI updates
                lastSyncResult = result
            } catch {
                syncError = error
            }
            
            isSyncing = false
        }
    }
}
```

### 3. Production Example: Complete Health App Data Layer

```swift
import SwiftUI
import SwiftData
import HealthKit
import Combine

// MARK: - Sendable Data Types
// Ensure thread safety for data crossing actor boundaries

struct HealthMetrics: Sendable {
    let heartRate: Double
    let steps: Int
    let sleepHours: Double
    let timestamp: Date
}

enum DataError: Error, Sendable {
    case networkFailure(String)
    case parseError
    case unauthorized
    case rateLimited(retryAfter: TimeInterval)
}

// MARK: - Global Actor for Analytics
@globalActor
actor AnalyticsActor {
    static let shared = AnalyticsActor()
    
    private var events: [AnalyticsEvent] = []
    private var sessionId = UUID()
    
    func track(event: AnalyticsEvent) {
        events.append(event)
        
        // Batch send events every 10 events
        if events.count >= 10 {
            Task {
                await sendBatch()
            }
        }
    }
    
    private func sendBatch() async {
        let batch = events
        events.removeAll()
        
        // Send to analytics service
        do {
            try await NetworkManager.shared.sendAnalytics(batch)
        } catch {
            // Re-queue failed events
            events.insert(contentsOf: batch, at: 0)
        }
    }
}

// MARK: - Main Data Coordinator with Advanced Patterns
@MainActor
class HealthDataCoordinator: ObservableObject {
    @Published private(set) var isLoading = false
    @Published private(set) var healthMetrics: HealthMetrics?
    @Published private(set) var syncProgress: Double = 0.0
    @Published private(set) var errors: [DataError] = []
    
    private let dataActor = DataProcessingActor()
    private var activeTasks: Set<Task<Void, Never>> = []
    private let taskSemaphore = AsyncSemaphore(value: 3) // Limit concurrent operations
    
    // Advanced task management with cancellation
    func startDataSync() {
        // Cancel any existing sync
        cancelAllTasks()
        
        let syncTask = Task { [weak self] in
            guard let self = self else { return }
            
            await self.setSyncState(loading: true, progress: 0)
            
            do {
                // Use task group for structured concurrency
                try await withThrowingTaskGroup(of: Void.self) { group in
                    // Add priority-based tasks
                    group.addTask(priority: .high) {
                        try await self.syncCriticalHealth()
                    }
                    
                    group.addTask(priority: .medium) {
                        try await self.syncHistoricalData()
                    }
                    
                    group.addTask(priority: .low) {
                        try await self.syncSupplementaryData()
                    }
                    
                    // Monitor progress
                    var completed = 0
                    for try await _ in group {
                        completed += 1
                        await self.updateProgress(Double(completed) / 3.0)
                    }
                }
                
                // Track successful sync
                await AnalyticsActor.shared.track(
                    event: AnalyticsEvent(name: "sync_completed", properties: [:])
                )
                
            } catch {
                await self.handleError(error)
            }
            
            await self.setSyncState(loading: false, progress: 1.0)
        }
        
        activeTasks.insert(syncTask)
    }
    
    // Sophisticated retry mechanism with exponential backoff
    private func syncCriticalHealth() async throws {
        var retryCount = 0
        let maxRetries = 3
        
        while retryCount < maxRetries {
            do {
                // Acquire semaphore to limit concurrent operations
                await taskSemaphore.wait()
                defer { 
                    Task {
                        await taskSemaphore.signal()
                    }
                }
                
                // Perform the actual sync
                let metrics = try await dataActor.fetchLatestMetrics()
                
                // Update UI on MainActor
                await MainActor.run {
                    self.healthMetrics = metrics
                }
                
                return // Success, exit retry loop
                
            } catch DataError.rateLimited(let retryAfter) {
                // Handle rate limiting with smart retry
                try await Task.sleep(nanoseconds: UInt64(retryAfter * 1_000_000_000))
                retryCount += 1
                
            } catch {
                // Exponential backoff for other errors
                let delay = pow(2.0, Double(retryCount)) * 1.0
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                retryCount += 1
                
                if retryCount >= maxRetries {
                    throw error
                }
            }
        }
    }
    
    private func syncHistoricalData() async throws {
        // Implementation with proper cancellation checking
        for date in last30Days() {
            // Check for cancellation
            try Task.checkCancellation()
            
            let data = try await dataActor.fetchHistoricalData(for: date)
            
            // Process in batches to avoid overwhelming the system
            if Task.isCancelled { break }
            
            await processHistoricalBatch(data)
        }
    }
    
    private func syncSupplementaryData() async throws {
        // Low-priority background sync
        try await Task.sleep(nanoseconds: 2_000_000_000) // 2 second delay
        
        try Task.checkCancellation()
        
        // Sync non-critical data
        async let nutritionData = dataActor.fetchNutritionData()
        async let exerciseData = dataActor.fetchExerciseData()
        
        // Await both concurrently
        let (nutrition, exercise) = try await (nutritionData, exerciseData)
        
        // Process results
        await processSupplementaryData(nutrition: nutrition, exercise: exercise)
    }
    
    // Helper methods
    @MainActor
    private func setSyncState(loading: Bool, progress: Double) {
        self.isLoading = loading
        self.syncProgress = progress
    }
    
    @MainActor
    private func updateProgress(_ progress: Double) {
        self.syncProgress = progress
    }
    
    @MainActor
    private func handleError(_ error: Error) {
        if let dataError = error as? DataError {
            self.errors.append(dataError)
        } else {
            self.errors.append(.networkFailure(error.localizedDescription))
        }
    }
    
    private func cancelAllTasks() {
        activeTasks.forEach { $0.cancel() }
        activeTasks.removeAll()
    }
    
    private func last30Days() -> [Date] {
        // Generate array of last 30 days
        (0..<30).compactMap { days in
            Calendar.current.date(byAdding: .day, value: -days, to: Date())
        }
    }
    
    private func processHistoricalBatch(_ data: [HealthRecord]) async {
        // Process historical data
    }
    
    private func processSupplementaryData(nutrition: NutritionData, exercise: ExerciseData) async {
        // Process supplementary data
    }
    
    deinit {
        cancelAllTasks()
    }
}

// MARK: - Actor for Data Processing
actor DataProcessingActor {
    private var cache: [String: CachedData] = [:]
    private let networkClient = NetworkClient()
    
    func fetchLatestMetrics() async throws -> HealthMetrics {
        // Check cache first
        if let cached = cache["latest_metrics"],
           cached.isValid {
            return cached.metrics
        }
        
        // Fetch from network
        let metrics = try await networkClient.getLatestHealthMetrics()
        
        // Update cache
        cache["latest_metrics"] = CachedData(
            metrics: metrics,
            timestamp: Date()
        )
        
        return metrics
    }
    
    func fetchHistoricalData(for date: Date) async throws -> [HealthRecord] {
        let key = "historical_\(date.formatted())"
        
        if let cached = cache[key], cached.isValid {
            return cached.records ?? []
        }
        
        let records = try await networkClient.getHealthRecords(for: date)
        
        cache[key] = CachedData(
            records: records,
            timestamp: Date()
        )
        
        return records
    }
    
    func fetchNutritionData() async throws -> NutritionData {
        // Implementation
        fatalError("Implement nutrition fetching")
    }
    
    func fetchExerciseData() async throws -> ExerciseData {
        // Implementation
        fatalError("Implement exercise fetching")
    }
    
    // Cache invalidation
    func invalidateCache() {
        cache.removeAll()
    }
    
    private struct CachedData {
        let metrics: HealthMetrics?
        let records: [HealthRecord]?
        let timestamp: Date
        
        var isValid: Bool {
            // Cache valid for 5 minutes
            Date().timeIntervalSince(timestamp) < 300
        }
        
        init(metrics: HealthMetrics? = nil, records: [HealthRecord]? = nil, timestamp: Date) {
            self.metrics = metrics
            self.records = records
            self.timestamp = timestamp
        }
    }
}

// MARK: - Async Semaphore for Resource Management
actor AsyncSemaphore {
    private var value: Int
    private var waiters: [CheckedContinuation<Void, Never>] = []
    
    init(value: Int) {
        self.value = value
    }
    
    func wait() async {
        if value > 0 {
            value -= 1
            return
        }
        
        await withCheckedContinuation { continuation in
            waiters.append(continuation)
        }
    }
    
    func signal() {
        if let waiter = waiters.first {
            waiters.removeFirst()
            waiter.resume()
        } else {
            value += 1
        }
    }
}

// MARK: - SwiftUI Integration
struct HealthDashboardView: View {
    @StateObject private var coordinator = HealthDataCoordinator()
    @Environment(\.scenePhase) private var scenePhase
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // Sync status card
                    SyncStatusCard(
                        isLoading: coordinator.isLoading,
                        progress: coordinator.syncProgress
                    )
                    
                    // Health metrics display
                    if let metrics = coordinator.healthMetrics {
                        HealthMetricsCard(metrics: metrics)
                    }
                    
                    // Error handling
                    ForEach(coordinator.errors, id: \.self) { error in
                        ErrorCard(error: error)
                    }
                }
                .padding()
            }
            .navigationTitle("Health Dashboard")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button("Sync") {
                        coordinator.startDataSync()
                    }
                    .disabled(coordinator.isLoading)
                }
            }
        }
        .task {
            // Initial load when view appears
            coordinator.startDataSync()
        }
        .onChange(of: scenePhase) { _, newPhase in
            // Sync when app becomes active
            if newPhase == .active {
                coordinator.startDataSync()
            }
        }
    }
}
```

## Common Pitfalls & Best Practices

### Mistakes from Java/TypeScript/Kotlin Backgrounds

1. **Forgetting Task Cancellation** - Unlike Promises in JavaScript which can't be cancelled, Swift Tasks support cooperative cancellation. Always check for cancellation in long-running operations and clean up tasks when views disappear.

2. **Blocking the Main Thread** - Coming from Java, you might try to use synchronous operations on the main thread. In iOS, this freezes the UI. Always use async operations for any potentially slow work.

3. **Not Understanding Actor Reentrancy** - Unlike Java's synchronized blocks, Swift actors can be re-entered while awaiting. This means state can change between await points within the same actor method.

4. **Overusing Actors** - Not everything needs to be an actor. Use them for shared mutable state, but regular classes with proper async methods are fine for stateless operations or immutable data.

5. **Memory Leaks with Task Retention** - Tasks hold strong references by default. Always use `[weak self]` in Task closures within classes to avoid retain cycles.

### Memory and Performance Implications

The concurrency system in Swift is designed for efficiency, using cooperative scheduling rather than OS threads. Each Task uses minimal memory (around 2KB) compared to threads (around 1MB). However, be mindful of creating too many concurrent tasks as they still consume resources. Use TaskGroups to manage large numbers of related tasks, and consider using AsyncSequence for streaming data rather than loading everything into memory at once.

### iOS Ecosystem Conventions

Always perform UI updates on the MainActor. This is enforced by SwiftUI but needs explicit handling in other contexts. Use the `@MainActor` attribute liberally for UI-related types and methods. For background work that shouldn't block the UI, create dedicated actors or use Task with appropriate priority levels.

## Integration with SwiftUI & iOS Development

SwiftUI views are automatically isolated to the MainActor, making them thread-safe by default. When you create a Task within a SwiftUI view, it inherits the MainActor context, ensuring UI updates happen on the main thread.

Here's how concurrency integrates with your health app's key frameworks:

```swift
import SwiftUI
import HealthKit
import SwiftData

// SwiftUI View with integrated async operations
struct HealthDataView: View {
    @Query private var healthRecords: [HealthRecord]
    @State private var isLoadingHealth = false
    @State private var latestHeartRate: Double?
    
    var body: some View {
        List {
            Section("Real-time Data") {
                if isLoadingHealth {
                    ProgressView()
                } else if let heartRate = latestHeartRate {
                    Label("\(Int(heartRate)) BPM", systemImage: "heart.fill")
                        .foregroundColor(.red)
                }
            }
            
            Section("Historical Data") {
                ForEach(healthRecords) { record in
                    HealthRecordRow(record: record)
                }
            }
        }
        .task {
            // This automatically runs on MainActor
            await loadHealthData()
        }
        .refreshable {
            // Pull-to-refresh with async support
            await refreshAllData()
        }
    }
    
    private func loadHealthData() async {
        isLoadingHealth = true
        
        do {
            // Fetch from HealthKit
            let heartRate = try await HealthKitManager.shared.getCurrentHeartRate()
            
            // Update UI state - already on MainActor
            latestHeartRate = heartRate
            
            // Update SwiftData in background
            await updateDatabase(heartRate: heartRate)
            
        } catch {
            print("Failed to load health data: \(error)")
        }
        
        isLoadingHealth = false
    }
    
    @ModelActor
    private func updateDatabase(heartRate: Double) async {
        // This runs on ModelActor, not MainActor
        let record = HealthRecord(
            type: .heartRate,
            value: heartRate,
            timestamp: Date()
        )
        
        modelContext.insert(record)
        try? modelContext.save()
    }
    
    private func refreshAllData() async {
        // Parallel refresh of multiple data sources
        async let healthData = HealthKitManager.shared.syncAllData()
        async let weatherData = WeatherService.shared.fetchCurrent()
        
        // Wait for both to complete
        let (health, weather) = await (healthData, weatherData)
        
        // Process results
        print("Refreshed with \(health) health records and weather: \(weather)")
    }
}

// HealthKit Manager using modern concurrency
@MainActor
class HealthKitManager: ObservableObject {
    static let shared = HealthKitManager()
    
    private let store = HKHealthStore()
    @Published var isAuthorized = false
    
    func getCurrentHeartRate() async throws -> Double {
        // Move authorization check off MainActor
        let authorized = await checkAuthorization()
        guard authorized else {
            throw HealthError.unauthorized
        }
        
        // Fetch data using continuation
        return try await withCheckedThrowingContinuation { continuation in
            let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
            let sortDescriptor = NSSortDescriptor(key: HKSampleSortIdentifierStartDate, ascending: false)
            
            let query = HKSampleQuery(
                sampleType: heartRateType,
                predicate: nil,
                limit: 1,
                sortDescriptors: [sortDescriptor]
            ) { _, samples, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else if let sample = samples?.first as? HKQuantitySample {
                    let heartRate = sample.quantity.doubleValue(for: .beatsPerMinute())
                    continuation.resume(returning: heartRate)
                } else {
                    continuation.resume(throwing: HealthError.noData)
                }
            }
            
            store.execute(query)
        }
    }
    
    nonisolated func checkAuthorization() async -> Bool {
        // Run authorization check off MainActor
        let types = Set([HKQuantityType.quantityType(forIdentifier: .heartRate)!])
        
        do {
            try await store.requestAuthorization(toShare: nil, read: types)
            return true
        } catch {
            return false
        }
    }
    
    func syncAllData() async -> Int {
        // Implementation for syncing all data
        try? await Task.sleep(nanoseconds: 2_000_000_000)
        return 42
    }
}
```

## Production Considerations

### Testing Async Code

Testing async code requires special considerations. XCTest provides full support for async tests:

```swift
import XCTest
@testable import YourHealthApp

class HealthDataTests: XCTestCase {
    
    func testAsyncDataFetch() async throws {
        // Async test - note the async keyword
        let manager = HealthDataManager()
        
        let data = try await manager.fetchHealthData()
        
        XCTAssertFalse(data.isEmpty)
        XCTAssertEqual(data.first?.type, .heartRate)
    }
    
    func testTaskCancellation() async throws {
        let manager = DataSyncManager()
        
        let task = Task {
            try await manager.performLongSync()
        }
        
        // Cancel after short delay
        try await Task.sleep(nanoseconds: 100_000_000)
        task.cancel()
        
        // Verify task was cancelled
        let result = await task.result
        
        if case .failure(let error) = result {
            XCTAssertTrue(error is CancellationError)
        } else {
            XCTFail("Task should have been cancelled")
        }
    }
    
    func testActorIsolation() async {
        let cache = DataCache()
        
        // Test concurrent access
        await withTaskGroup(of: Void.self) { group in
            for i in 0..<100 {
                group.addTask {
                    await cache.store(value: i, key: "item_\(i)")
                }
            }
        }
        
        let count = await cache.itemCount
        XCTAssertEqual(count, 100)
    }
}
```

### Debugging Techniques

Use these debugging strategies for async code:

1. **Thread Sanitizer** - Enable in Xcode's scheme editor to detect data races
2. **Async Breakpoints** - Set breakpoints in async functions; Xcode handles context switching
3. **Task Local Values** - Use for debugging context across async boundaries
4. **Instruments** - Use the Concurrency instrument to visualize task execution

### iOS Version Considerations

For iOS 17+, you have access to all modern concurrency features including:
- Async sequences and streams
- Task groups with automatic cancellation
- Actor reentrancy improvements
- Better SwiftData integration with actors
- Observation framework that works seamlessly with async code

### Performance Optimization

Key optimization strategies include:
- Use TaskGroup for parallel operations instead of multiple individual Tasks
- Implement proper caching in actors to avoid redundant network calls
- Use AsyncSequence for streaming large datasets instead of loading all at once
- Set appropriate task priorities (.userInitiated for UI-blocking work, .background for non-critical)
- Leverage cooperative cancellation to stop unnecessary work early

## Exercises for Mastery

### Exercise 1: Async Network Manager (Familiar Pattern)

Create a network manager similar to what you'd build in TypeScript with Axios, but using Swift's async/await:

```swift
// Your task: Implement this network manager
protocol NetworkManaging {
    func get<T: Decodable>(_ endpoint: String, parameters: [String: Any]?) async throws -> T
    func post<T: Decodable, U: Encodable>(_ endpoint: String, body: U) async throws -> T
}

class NetworkManager: NetworkManaging {
    // Implement with proper error handling, retry logic, and cancellation support
    // Include rate limiting using an actor
    // Add request caching
}

// Test your implementation:
struct User: Codable {
    let id: Int
    let name: String
}

let manager = NetworkManager()
let users = try await manager.get("/users", parameters: nil) as [User]
```

### Exercise 2: Health Data Pipeline (Swift-Specific)

Build a complete data pipeline for your health app that processes data from multiple sources concurrently:

```swift
// Create a system that:
// 1. Fetches data from HealthKit, your API, and SwiftData simultaneously
// 2. Processes and merges the data
// 3. Updates the UI progressively as data arrives
// 4. Handles errors gracefully with retry logic
// 5. Supports cancellation

actor HealthDataPipeline {
    func processHealthData(for dateRange: DateRange) async throws -> ProcessedHealthData {
        // Your implementation here
        // Must use TaskGroup, proper error handling, and progress reporting
    }
}
```

### Exercise 3: Mini-Challenge - Intermittent Fasting Tracker

Create a complete intermittent fasting tracker component that:
- Monitors eating windows in real-time
- Sends notifications at the right times (using async scheduling)
- Syncs data with HealthKit in the background
- Updates SwiftUI views reactively
- Handles all operations concurrently without blocking the UI

This should integrate everything you've learned about async/await, actors, MainActor, and Sendable types in a production-ready component for your app.

---

Remember, Swift's concurrency model is about safety and structure. Unlike the callback-based or Promise-based approaches you're used to, Swift enforces correct concurrent programming at compile time. This might feel restrictive at first, but it prevents entire classes of bugs that are common in JavaScript and Java applications. The key is to embrace the structured concurrency model and let the compiler guide you toward safe, efficient concurrent code.

