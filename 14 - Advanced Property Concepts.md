# Advanced Property Concepts in Swift

Let me guide you through these powerful Swift property features that will elevate your iOS development skills. Coming from your TypeScript/Angular background, you'll find some familiar patterns here, but with Swift's unique performance optimizations that make them particularly powerful for iOS apps.

## Context & Background

These advanced property concepts represent Swift's sophisticated approach to managing state, access patterns, and memory efficiency. Think of them as Swift's answer to several patterns you know from other languages, but unified under a coherent property system.

In TypeScript and Angular, you've worked with decorators, getters/setters, proxy objects, and dependency injection. Swift's property wrappers are like TypeScript decorators on steroids - they can transform how properties behave, store additional state, and even project multiple values. The dollar sign projection is similar to Angular's template reference variables, but more powerful. Static properties work like Java's static fields but with better namespace management. Subscripts are like TypeScript's index signatures but can include complex logic. And Copy-on-Write is Swift's secret weapon for making value types performant - imagine if every Angular component could be immutable without performance penalties.

In production iOS apps, you'll use these features constantly. Property wrapper composition helps you build reusable state management patterns (crucial for SwiftUI). Projected values give you direct access to binding mechanisms in forms. Static properties manage shared configuration across your app. Subscripts make your custom types feel native. And understanding Copy-on-Write ensures your health data processing remains efficient even with large datasets.

## Core Understanding

Let's break down each concept to build your mental model:

**@propertyWrapper Composition** allows you to stack multiple property wrappers on a single property, like middleware in Express or interceptors in Angular. Each wrapper transforms the value in sequence, creating powerful combinations of behaviors. The key mental model is a pipeline where each wrapper adds its own behavior layer.

**Projected Values with $** provide a secondary interface to a property. When you see `$someProperty`, you're accessing the property wrapper's projected value, not the wrapped value itself. Think of it as having both `user` and `userForm` in Angular - one for the data, one for the reactive form controls.

**Static Properties and Type Properties** belong to the type itself, not instances. Unlike Java's static fields which are just class-level variables, Swift's static properties can have computed logic and observers. They're perfect for shared configuration, singleton access points, and factory methods.

**Subscripts** let you access elements using square bracket notation, but with custom logic. If TypeScript's index signatures and Java's array access had a baby with full method capabilities, you'd get Swift subscripts. They can have multiple parameters, different types, and even generic constraints.

**Copy-on-Write (CoW) Optimization** is how Swift makes value types efficient. When you copy a struct or array, Swift doesn't actually copy the data until you modify it. It's like having immutable data structures with mutable performance characteristics - the best of both worlds.

## Practical Code Examples

### 1. Basic Example: Understanding Each Concept

```swift
import SwiftUI

// MARK: - Property Wrapper Composition
// Think of these as TypeScript decorators that can stack
@propertyWrapper
struct Clamped<Value: Comparable> {
    var value: Value
    let range: ClosedRange<Value>
    
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(range.lowerBound, wrappedValue), range.upperBound)
    }
    
    var wrappedValue: Value {
        get { value }
        set { value = min(max(range.lowerBound, newValue), range.upperBound) }
    }
    
    // Projected value exposes the full wrapper
    var projectedValue: Clamped<Value> { self }
}

@propertyWrapper
struct Logged<Value> {
    private var value: Value
    private let key: String
    
    init(wrappedValue: Value, key: String) {
        self.value = wrappedValue
        self.key = key
        print("[\(key)] Initialized with: \(wrappedValue)")
    }
    
    var wrappedValue: Value {
        get { 
            print("[\(key)] Read: \(value)")
            return value 
        }
        set { 
            print("[\(key)] Changed from \(value) to \(newValue)")
            value = newValue 
        }
    }
}

// Using composition - both wrappers apply!
struct HealthMetrics {
    // Wrong way (TypeScript/Java thinking): Try to handle all logic in one place
    // var heartRate: Int {
    //     didSet {
    //         if heartRate < 40 { heartRate = 40 }
    //         if heartRate > 220 { heartRate = 220 }
    //         print("Heart rate changed to \(heartRate)")
    //     }
    // }
    
    // Swift way: Compose reusable behaviors
    @Logged(key: "HeartRate")
    @Clamped(40...220)
    var heartRate: Int = 60
    
    // Static property for shared configuration
    // Like Java's static final, but can be computed
    static let targetHeartRateZones = [
        "rest": 40...60,
        "light": 60...100,
        "moderate": 100...140,
        "hard": 140...180,
        "maximum": 180...220
    ]
    
    // Subscript for zone access
    // Like TypeScript's index signatures but with logic
    subscript(zone: String) -> ClosedRange<Int>? {
        Self.targetHeartRateZones[zone]
    }
}

// Using the basic example
var metrics = HealthMetrics()
metrics.heartRate = 250  // Will be clamped to 220 and logged
print(metrics.$heartRate.range)  // Access the wrapper itself via $
if let moderateZone = metrics["moderate"] {
    print("Moderate zone: \(moderateZone)")
}
```

### 2. Real-World Example: Health Tracking App Context

```swift
import SwiftUI
import Combine

// MARK: - Real-world property wrappers for your health app
@propertyWrapper
struct Persisted<Value: Codable> {
    private let key: String
    private let defaultValue: Value
    
    init(key: String, defaultValue: Value) {
        self.key = key
        self.defaultValue = defaultValue
    }
    
    var wrappedValue: Value {
        get {
            // Like localStorage in web, but type-safe
            guard let data = UserDefaults.standard.data(forKey: key),
                  let value = try? JSONDecoder().decode(Value.self, from: data) else {
                return defaultValue
            }
            return value
        }
        set {
            let data = try? JSONEncoder().encode(newValue)
            UserDefaults.standard.set(data, forKey: key)
        }
    }
    
    // Projected value provides a publisher for SwiftUI binding
    var projectedValue: AnyPublisher<Value, Never> {
        UserDefaults.standard.publisher(for: \.self)
            .map { _ in self.wrappedValue }
            .eraseToAnyPublisher()
    }
}

@propertyWrapper
struct ValidatedInput<Value> {
    private var value: Value
    private let validation: (Value) -> Bool
    private(set) var isValid: Bool
    
    init(wrappedValue: Value, validation: @escaping (Value) -> Bool) {
        self.value = wrappedValue
        self.validation = validation
        self.isValid = validation(wrappedValue)
    }
    
    var wrappedValue: Value {
        get { value }
        set {
            value = newValue
            isValid = validation(newValue)
        }
    }
    
    // Projected value exposes validation state
    var projectedValue: ValidatedInput<Value> { self }
}

// Your health tracking model with advanced properties
class HealthDataStore: ObservableObject {
    // Composed property wrappers for persistent, validated data
    @Persisted(key: "targetCalories", defaultValue: 2000)
    @ValidatedInput(validation: { $0 > 500 && $0 < 5000 })
    @Published var targetCalories: Int = 2000
    
    // Static configuration - shared across all instances
    static let mealTypes = ["breakfast", "lunch", "dinner", "snack"]
    static let defaultMacroRatios = MacroRatios(carbs: 0.5, protein: 0.3, fat: 0.2)
    
    // Private static for singleton pattern (like Angular services)
    private static var _shared: HealthDataStore?
    static var shared: HealthDataStore {
        if _shared == nil {
            _shared = HealthDataStore()
        }
        return _shared!
    }
    
    // Subscript for accessing daily data by date
    private var dailyData: [Date: DailyHealthData] = [:]
    
    subscript(date: Date) -> DailyHealthData {
        get {
            // Normalize to start of day
            let normalizedDate = Calendar.current.startOfDay(for: date)
            return dailyData[normalizedDate] ?? DailyHealthData(date: normalizedDate)
        }
        set {
            let normalizedDate = Calendar.current.startOfDay(for: date)
            dailyData[normalizedDate] = newValue
        }
    }
    
    // Multi-parameter subscript for specific meal data
    subscript(date: Date, meal: String) -> MealData? {
        get { self[date].meals[meal] }
        set { self[date].meals[meal] = newValue }
    }
}

struct DailyHealthData {
    let date: Date
    var meals: [String: MealData] = [:]
    var steps: Int = 0
    var activeCalories: Int = 0
}

struct MealData {
    var calories: Int
    var timestamp: Date
}

struct MacroRatios {
    let carbs: Double
    let protein: Double
    let fat: Double
}

// SwiftUI View using these concepts
struct CalorieTargetView: View {
    @StateObject private var store = HealthDataStore.shared
    
    var body: some View {
        VStack {
            // Access the wrapped value normally
            Text("Target: \(store.targetCalories) calories")
            
            // Use $ to access validation state
            if !store.$targetCalories.isValid {
                Text("Please enter a value between 500 and 5000")
                    .foregroundColor(.red)
            }
            
            // Access today's data via subscript
            Text("Today's calories: \(store[Date()].meals.values.reduce(0) { $0 + $1.calories })")
        }
    }
}
```

### 3. Production Example: Advanced Usage with Edge Cases

```swift
import SwiftUI
import Combine
import HealthKit

// MARK: - Production-grade implementation with CoW optimization
struct OptimizedHealthDataBuffer {
    // Internal storage with CoW optimization
    private var _storage: Storage
    
    private class Storage {
        var data: [HealthSample]
        var isKnownUniquelyReferenced = true
        
        init(data: [HealthSample] = []) {
            self.data = data
        }
    }
    
    init() {
        _storage = Storage()
    }
    
    // Copy-on-Write implementation
    private mutating func makeUnique() {
        // Wrong way (Java thinking): Always copy
        // _storage = Storage(data: _storage.data)
        
        // Swift way: Only copy when shared
        if !isKnownUniquelyReferenced(&_storage) {
            _storage = Storage(data: _storage.data)
        }
    }
    
    // Subscript with bounds checking and CoW
    subscript(index: Int) -> HealthSample? {
        get {
            guard index >= 0 && index < _storage.data.count else { return nil }
            return _storage.data[index]
        }
        set {
            guard let newValue = newValue,
                  index >= 0 && index < _storage.data.count else { return }
            makeUnique()  // Trigger CoW before mutation
            _storage.data[index] = newValue
        }
    }
    
    // Advanced subscript with range and type filtering
    subscript<T: HealthSample>(range: Range<Int>, type: T.Type) -> [T] {
        let validRange = range.clamped(to: 0..<_storage.data.count)
        return _storage.data[validRange].compactMap { $0 as? T }
    }
    
    mutating func append(_ sample: HealthSample) {
        makeUnique()
        _storage.data.append(sample)
    }
}

// Production property wrapper with thread safety
@propertyWrapper
final class ThreadSafe<Value> {
    private var value: Value
    private let queue = DispatchQueue(label: "ThreadSafe", attributes: .concurrent)
    
    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
    
    var wrappedValue: Value {
        get { queue.sync { value } }
        set { queue.async(flags: .barrier) { self.value = newValue } }
    }
    
    // Projected value provides atomic update capability
    var projectedValue: ThreadSafe<Value> { self }
    
    func update(_ transform: @escaping (inout Value) -> Void) {
        queue.async(flags: .barrier) {
            transform(&self.value)
        }
    }
}

// Advanced property wrapper composition with error handling
@propertyWrapper
struct CachedHealthKit<Value: Codable> {
    private let healthStore = HKHealthStore()
    private let cacheKey: String
    private let fetchBlock: (HKHealthStore) async throws -> Value
    private var cachedValue: Value?
    private var lastFetch: Date?
    
    init(
        key: String,
        fetch: @escaping (HKHealthStore) async throws -> Value
    ) {
        self.cacheKey = key
        self.fetchBlock = fetch
        
        // Load from cache if available
        if let data = UserDefaults.standard.data(forKey: key),
           let cached = try? JSONDecoder().decode(Value.self, from: data) {
            self.cachedValue = cached
        }
    }
    
    var wrappedValue: Value {
        get {
            // This is a simplified sync version - in production, use async
            if let cached = cachedValue,
               let lastFetch = lastFetch,
               Date().timeIntervalSince(lastFetch) < 300 { // 5 min cache
                return cached
            }
            
            // In production, this would be async
            // For demo, return cached or throw
            return cachedValue ?? (Value.self == Int.self ? 0 as! Value : [] as! Value)
        }
        set {
            cachedValue = newValue
            lastFetch = Date()
            
            // Persist to cache
            if let data = try? JSONEncoder().encode(newValue) {
                UserDefaults.standard.set(data, forKey: cacheKey)
            }
        }
    }
    
    // Projected value provides async refresh capability
    var projectedValue: AsyncRefreshable<Value> {
        AsyncRefreshable { [self] in
            let value = try await fetchBlock(healthStore)
            self.cachedValue = value
            self.lastFetch = Date()
            return value
        }
    }
}

struct AsyncRefreshable<Value> {
    let refresh: () async throws -> Value
}

// Production view model with all concepts combined
@MainActor
final class ProductionHealthViewModel: ObservableObject {
    // Static configuration with computed properties
    static let healthKitTypes: Set<HKSampleType> = {
        var types = Set<HKSampleType>()
        types.insert(HKQuantityType(.stepCount))
        types.insert(HKQuantityType(.activeEnergyBurned))
        types.insert(HKQuantityType(.heartRate))
        return types
    }()
    
    // Thread-safe property wrapper composition
    @ThreadSafe
    @Published
    private(set) var processingQueue: [ProcessingTask] = []
    
    // Cached HealthKit data with projection
    @CachedHealthKit(key: "dailySteps", fetch: { store in
        // Actual HealthKit query would go here
        return 10000
    })
    var dailySteps: Int
    
    // Advanced subscript for date-ranged data with error handling
    subscript(from startDate: Date, to endDate: Date) -> Result<[HealthSample], HealthDataError> {
        get {
            do {
                let samples = try fetchSamples(from: startDate, to: endDate)
                return .success(samples)
            } catch {
                return .failure(HealthDataError.fetchFailed(error))
            }
        }
    }
    
    private func fetchSamples(from: Date, to: Date) throws -> [HealthSample] {
        // Implementation would query HealthKit
        return []
    }
    
    // Method using projected value for thread-safe updates
    func addProcessingTask(_ task: ProcessingTask) {
        $processingQueue.update { queue in
            queue.append(task)
            // Sort by priority
            queue.sort { $0.priority > $1.priority }
        }
    }
}

// Supporting types
protocol HealthSample: Codable {
    var timestamp: Date { get }
    var value: Double { get }
}

struct ProcessingTask {
    let id = UUID()
    let priority: Int
    let action: () async -> Void
}

enum HealthDataError: Error {
    case fetchFailed(Error)
    case invalidDateRange
    case insufficientData
}

// Extension for Range clamping (utility)
extension Range where Bound: Comparable {
    func clamped(to limits: Range) -> Range {
        let lower = Swift.max(lowerBound, limits.lowerBound)
        let upper = Swift.min(upperBound, limits.upperBound)
        return lower..<upper
    }
}
```

## Common Pitfalls & Best Practices

Coming from TypeScript/Java/Kotlin, watch out for these common mistakes:

**Property Wrapper Composition Order Matters**: Unlike TypeScript decorators which often work independently, Swift property wrapper order is significant. The outermost wrapper is applied first when setting, but accessed last when getting. Always put validation wrappers outside of transformation wrappers.

**Projected Values Aren't Magic**: The dollar sign doesn't automatically give you a binding like in SwiftUI. Each property wrapper defines what its projected value is. Some return self, others return publishers, bindings, or completely different types. Always check the documentation.

**Static Properties and Memory**: Unlike Java where static fields are initialized once at class load, Swift's static properties are lazily initialized on first access. This means they can capture self in closures accidentally. Use `[weak self]` or `[unowned self]` in static closures.

**Subscripts Can't Throw by Default**: Coming from Java where array access can throw, remember Swift subscripts return optionals for safety by default. If you need throwing subscripts, you must explicitly mark them as `throws`.

**Copy-on-Write Breaks with Classes**: CoW only works automatically with value types (structs, enums). If your struct contains class references, you must implement CoW manually for those references, or you'll get unexpected sharing.

Memory and performance implications to remember: Property wrappers add a small overhead per property (usually 16-24 bytes). Projected values don't create additional storage unless the wrapper explicitly stores them. Static properties are shared across all instances, so be careful with large static data structures. Subscripts are just syntactic sugar for methods, so complex subscripts have method call overhead. Copy-on-Write makes value types efficient, but the first mutation after a copy has overhead for the actual copy.

## Integration with SwiftUI & iOS Development

SwiftUI is built on property wrappers, and understanding these concepts is crucial for effective SwiftUI development. Here's how they integrate:

```swift
import SwiftUI
import HealthKit
import SwiftData

// SwiftUI integration example
struct HealthDashboardView: View {
    // SwiftUI's property wrappers compose with custom ones
    @StateObject private var viewModel = HealthViewModel()
    @Environment(\.modelContext) private var modelContext
    
    // Custom wrapper for SwiftData queries
    @Query(sort: \HealthRecord.date, order: .reverse)
    private var healthRecords: [HealthRecord]
    
    var body: some View {
        List {
            Section("Today's Metrics") {
                // Using subscripts in SwiftUI
                if let todayData = viewModel[Date()] {
                    MetricRow(title: "Steps", value: todayData.steps)
                    MetricRow(title: "Calories", value: todayData.calories)
                }
            }
            
            Section("Weekly Trends") {
                // Using static properties for configuration
                ForEach(HealthViewModel.metricTypes, id: \.self) { metric in
                    TrendView(metric: metric, data: healthRecords)
                }
            }
        }
        .refreshable {
            // Using projected value for async refresh
            try? await viewModel.$healthKitData.refresh()
        }
    }
}

// HealthKit integration with property wrappers
@propertyWrapper
struct HealthKitQuery<Value> {
    private let query: HKQuery
    private var result: Value?
    
    var wrappedValue: Value? { result }
    
    // Projected value provides the query for advanced usage
    var projectedValue: HKQuery { query }
}

// SwiftData model with computed properties and subscripts
@Model
final class HealthRecord {
    var date: Date
    var steps: Int
    var calories: Int
    var heartRates: [HeartRateReading]
    
    // Computed property for averages
    var averageHeartRate: Double {
        guard !heartRates.isEmpty else { return 0 }
        return heartRates.reduce(0.0) { $0 + $1.value } / Double(heartRates.count)
    }
    
    // Subscript for accessing readings by time
    subscript(hour: Int) -> [HeartRateReading] {
        heartRates.filter { reading in
            Calendar.current.component(.hour, from: reading.timestamp) == hour
        }
    }
    
    init(date: Date) {
        self.date = date
        self.steps = 0
        self.calories = 0
        self.heartRates = []
    }
}

@Model
final class HeartRateReading {
    var timestamp: Date
    var value: Double
    
    init(timestamp: Date, value: Double) {
        self.timestamp = timestamp
        self.value = value
    }
}
```

## Production Considerations

For testing these concepts, create focused unit tests for each property wrapper. Test both the wrapped value and projected value behaviors. For subscripts, test boundary conditions and invalid inputs. For CoW optimization, use `isKnownUniquelyReferenced` to verify your implementation.

```swift
import XCTest

class PropertyWrapperTests: XCTestCase {
    func testClampedWrapper() {
        @Clamped(0...100) var percentage = 50
        
        percentage = 150
        XCTAssertEqual(percentage, 100)
        
        percentage = -10
        XCTAssertEqual(percentage, 0)
        
        // Test projected value
        XCTAssertEqual($percentage.range, 0...100)
    }
    
    func testCoWOptimization() {
        var buffer1 = OptimizedHealthDataBuffer()
        var buffer2 = buffer1  // Should share storage
        
        // Verify sharing (would need internal access in real tests)
        // Reading shouldn't trigger copy
        _ = buffer2[0]
        
        // Writing should trigger copy
        buffer2.append(MockHealthSample())
        
        // Now they should be independent
        XCTAssertNotEqual(buffer1[0]?.timestamp, buffer2[0]?.timestamp)
    }
}
```

For debugging, use breakpoints in property wrapper getters/setters to trace data flow. The Xcode memory graph debugger helps identify retain cycles in static properties. Use Instruments to profile CoW behavior and ensure copies happen only when needed.

iOS 17+ considerations include using the new Observable macro which changes how property wrappers work with SwiftUI. The @Model macro in SwiftData has specific requirements for property wrappers. Prefer the new @Bindable over @ObservedObject for iOS 17+.

Performance optimization opportunities include implementing custom CoW for complex types to avoid unnecessary copies, using lazy initialization for expensive static properties, caching computed subscript results when appropriate, and combining multiple property wrappers into one when composition overhead becomes significant.

## Exercises for Mastery

### Exercise 1: Migration Pattern (Familiar Ground)
Create a property wrapper that mimics Angular's FormControl behavior. It should track value, validation state, and dirty/pristine status. Include a projected value that exposes validation errors.

```swift
// Your task: Implement @FormControl
@FormControl(validators: [.required, .email])
var emailAddress: String = ""

// Should support:
// emailAddress (get/set the value)
// $emailAddress.isValid
// $emailAddress.errors
// $emailAddress.isDirty
```

### Exercise 2: Swift-Specific Pattern (Health App Context)
Build a sophisticated health data cache that combines multiple property wrappers: @Persisted for disk storage, @Expiring for time-based invalidation, and @ThreadSafe for concurrent access. Create a subscript that allows filtering by date range and health metric type.

```swift
// Your implementation should support:
class HealthCache {
    @ThreadSafe
    @Expiring(duration: 300)
    @Persisted(key: "health_cache")
    var data: [HealthSample] = []
    
    // Implement this subscript
    subscript(from: Date, to: Date, type: HealthMetricType) -> [HealthSample] {
        // Filter and return appropriate samples
    }
}
```

### Exercise 3: Production Challenge
Implement a Copy-on-Write buffer for streaming HealthKit data that efficiently handles large datasets. Include subscripts for time-based access, statistical queries (mean, median, percentiles), and bulk updates. Add property wrappers for automatic data validation and unit conversion.

Requirements: The buffer should handle 10,000+ samples efficiently, support concurrent reads with single writer, automatically convert between metric and imperial units, validate heart rate data is within human ranges (30-250 bpm), and provide O(1) access for recent data and O(log n) for historical queries.

These exercises progressively build your understanding from familiar patterns to Swift-specific optimizations critical for production iOS apps. Focus on understanding when each pattern provides value rather than using them everywhere - the best Swift code uses advanced features judiciously to solve real problems.

