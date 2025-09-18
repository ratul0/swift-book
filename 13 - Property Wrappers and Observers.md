I'll teach you about Property Wrappers and Observers in Swift - these are powerful features that will feel familiar yet different from what you know in TypeScript and Java. Let me break this down comprehensively for your journey to becoming an iOS indie developer.

## Context & Background

Property wrappers and observers in Swift are mechanisms for adding behavior to properties without cluttering your code with boilerplate. Think of them as sophisticated ways to intercept and augment property access and modification.

Coming from your Angular background, property observers are similar to Angular's ngOnChanges lifecycle hook or RxJS's distinctUntilChanged operator - they let you react to value changes. Property wrappers are like Angular's decorators (@Input, @Output) or Java's annotations, but they're far more powerful because they can actually wrap and transform the property's storage and behavior.

In TypeScript, you might use getters and setters to add logic around property access. Swift takes this concept and elevates it with cleaner syntax and more sophisticated patterns. While Java has annotations that mark metadata, Swift's property wrappers actually contain executable code that modifies behavior.

You'll use these constantly in production iOS apps. Property observers help you update UI when data changes, validate input, or trigger side effects. Property wrappers are the foundation of SwiftUI's reactive system (@State, @Binding) and handle everything from thread safety to persistent storage. Understanding these concepts is crucial for building professional iOS apps.

## Core Understanding

Let me break down each concept to build your mental model:

**Stored Properties** are the simplest - they directly store values in memory, just like regular class fields in Java or properties in TypeScript. They allocate memory to hold their value.

**Computed Properties** don't store values at all. They calculate their value on-demand using getter and setter blocks, similar to TypeScript getters/setters or Kotlin's custom accessors. They're pure computation with no storage overhead.

**Property Observers** (willSet and didSet) are hooks that fire before and after a stored property changes. Think of them as built-in change detection that's more efficient than Angular's change detection cycle because they're compile-time features, not runtime watchers.

**Lazy Stored Properties** delay initialization until first access, similar to Kotlin's lazy delegates or creating a singleton with double-checked locking in Java. They're perfect for expensive operations you might not need.

**Property Wrappers** encapsulate property behavior in a reusable way. They're like aspect-oriented programming for properties - you define the wrapper once and apply it anywhere with an @ symbol.

Here's the basic syntax to get you oriented:

```swift
class HealthDataManager {
    // Stored property - holds actual value in memory
    var userName: String = "Guest"
    
    // Property observers - react to changes
    var dailySteps: Int = 0 {
        willSet {
            print("About to change from \(dailySteps) to \(newValue)")
        }
        didSet {
            print("Changed from \(oldValue) to \(dailySteps)")
            // Trigger UI update, save to database, etc.
        }
    }
    
    // Computed property - calculates on demand
    var stepsGoalProgress: Double {
        get {
            return Double(dailySteps) / 10000.0
        }
        set {
            dailySteps = Int(newValue * 10000)
        }
    }
    
    // Lazy property - initialized on first access
    lazy var healthStore = HKHealthStore()
    
    // Property wrapper usage
    @Clamped(min: 0, max: 200) var heartRate: Int = 75
}
```

## Practical Code Examples

### 1. Basic Example - Understanding the Fundamentals

```swift
// Basic example showing all property types
class NutritionTracker {
    // STORED PROPERTY - Simple storage
    var calories: Int = 0
    
    // PROPERTY OBSERVERS - React to changes
    var proteinGrams: Double = 0.0 {
        willSet {
            // 'newValue' is implicitly available in willSet
            print("Protein will change to \(newValue)g")
        }
        didSet {
            // 'oldValue' is implicitly available in didSet
            if proteinGrams > 200 {
                print("Warning: High protein intake!")
            }
            // Common pattern: Update dependent values
            updateMacroPercentages()
        }
    }
    
    // COMPUTED PROPERTY - No storage, just calculation
    var proteinCalories: Double {
        // In TypeScript/Java, you'd write: getProteinCalories() { return this.proteinGrams * 4 }
        get {
            return proteinGrams * 4
        }
        // Optional setter
        set {
            proteinGrams = newValue / 4
        }
    }
    
    // Read-only computed property (no setter)
    var totalMacroCalories: Double {
        // Like a TypeScript getter without setter
        proteinCalories + (carbGrams * 4) + (fatGrams * 9)
    }
    
    var carbGrams: Double = 0.0
    var fatGrams: Double = 0.0
    
    // LAZY PROPERTY - Delayed initialization
    lazy var formatter: NumberFormatter = {
        // This closure runs ONCE on first access
        // Like Kotlin's: val formatter by lazy { ... }
        let fmt = NumberFormatter()
        fmt.numberStyle = .decimal
        fmt.maximumFractionDigits = 1
        return fmt
    }()
    
    private func updateMacroPercentages() {
        // Update UI or trigger calculations
    }
}

// WRONG WAY (Java/TypeScript thinking):
class WrongNutritionTracker {
    private var _proteinGrams: Double = 0.0
    
    // Don't manually create getters/setters like Java!
    func getProteinGrams() -> Double {
        return _proteinGrams
    }
    
    func setProteinGrams(_ value: Double) {
        _proteinGrams = value
        // Manual change notification
    }
}

// SWIFT WAY: Use computed properties or property observers instead
```

### 2. Real-World Example - Health Tracking App Context

```swift
import SwiftUI
import HealthKit

// Model for your eating pattern tracker
class EatingPatternModel: ObservableObject {
    // PROPERTY OBSERVERS for business logic
    @Published var lastMealTime: Date? {
        didSet {
            // Automatically calculate fasting duration when meal time changes
            updateFastingStatus()
            
            // Save to persistent storage
            if let time = lastMealTime {
                UserDefaults.standard.set(time, forKey: "lastMealTime")
            }
        }
    }
    
    // COMPUTED PROPERTIES for derived state
    var currentFastingHours: Int {
        guard let lastMeal = lastMealTime else { return 0 }
        let hours = Date().timeIntervalSince(lastMeal) / 3600
        return Int(hours)
    }
    
    var isInFastingWindow: Bool {
        // Business logic based on user's fasting schedule
        currentFastingHours >= minimumFastingHours
    }
    
    var fastingProgress: Double {
        // For progress bars in UI
        guard targetFastingHours > 0 else { return 0 }
        return min(Double(currentFastingHours) / Double(targetFastingHours), 1.0)
    }
    
    // STORED PROPERTIES with observers for validation
    var targetFastingHours: Int = 16 {
        willSet {
            // Validation before setting
            if newValue < 12 || newValue > 24 {
                print("Invalid fasting target, keeping current value")
                // In production, you'd throw or handle this better
            }
        }
        didSet {
            // Only save if value actually changed
            if oldValue != targetFastingHours {
                UserDefaults.standard.set(targetFastingHours, forKey: "targetHours")
                objectWillChange.send() // Manually trigger SwiftUI update
            }
        }
    }
    
    var minimumFastingHours: Int = 12
    
    // LAZY PROPERTIES for expensive operations
    lazy var healthStore: HKHealthStore = {
        // Only create when actually needed
        // This is expensive, so we delay it
        let store = HKHealthStore()
        requestHealthKitPermissions(store: store)
        return store
    }()
    
    lazy var dateFormatter: DateFormatter = {
        // Formatters are expensive to create
        let formatter = DateFormatter()
        formatter.dateStyle = .medium
        formatter.timeStyle = .short
        return formatter
    }()
    
    // Private helper methods
    private func updateFastingStatus() {
        // Complex logic to update fasting state
        if isInFastingWindow {
            scheduleEndOfFastNotification()
        }
    }
    
    private func requestHealthKitPermissions(store: HKHealthStore) {
        // Setup HealthKit permissions
    }
    
    private func scheduleEndOfFastNotification() {
        // Schedule local notification
    }
}

// BASIC PROPERTY WRAPPER
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    private let range: ClosedRange<Value>
    
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        // Ensure initial value is within range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    
    var wrappedValue: Value {
        get { value }
        set { 
            // Automatically clamp to range
            value = min(max(newValue, range.lowerBound), range.upperBound)
        }
    }
}

// Using the property wrapper
class VitalSigns {
    @Clamped(30...250) var heartRate: Int = 75  // Always stays in valid range
    @Clamped(0.0...100.0) var bloodOxygenPercentage: Double = 98.5
}
```

### 3. Production Example - Advanced Patterns with Error Handling

```swift
import SwiftUI
import Combine

// Advanced property wrapper with projection
@propertyWrapper
struct Validated<Value> {
    private var value: Value
    private var validator: (Value) -> Bool
    private(set) var isValid: Bool
    
    init(wrappedValue: Value, validator: @escaping (Value) -> Bool) {
        self.value = wrappedValue
        self.validator = validator
        self.isValid = validator(wrappedValue)
    }
    
    var wrappedValue: Value {
        get { value }
        set {
            value = newValue
            isValid = validator(newValue)
        }
    }
    
    // Projected value (accessed with $)
    var projectedValue: Binding<Value> {
        Binding(
            get: { self.wrappedValue },
            set: { self.wrappedValue = $0 }
        )
    }
}

// Thread-safe property wrapper
@propertyWrapper
struct Atomic<Value> {
    private var value: Value
    private let lock = NSLock()
    
    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
    
    var wrappedValue: Value {
        get {
            lock.lock()
            defer { lock.unlock() }
            return value
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            value = newValue
        }
    }
}

// Production-ready health data manager
class HealthDataService: ObservableObject {
    // Thread-safe counter for concurrent updates
    @Atomic private(set) var syncOperationCount: Int = 0
    
    // Validated properties with business rules
    @Validated(validator: { $0 >= 0 && $0 <= 50000 })
    var dailySteps: Int = 0
    
    @Validated(validator: { !$0.isEmpty && $0.count <= 100 })
    var userName: String = "User"
    
    // Complex computed property with error handling
    var healthScore: Result<Double, HealthError> {
        do {
            guard dailySteps > 0 else {
                throw HealthError.insufficientData("No step data available")
            }
            
            guard let lastSync = lastSyncDate else {
                throw HealthError.syncRequired
            }
            
            // Check if data is stale
            if Date().timeIntervalSince(lastSync) > 86400 {
                throw HealthError.staleData
            }
            
            // Calculate score based on multiple factors
            let stepScore = min(Double(dailySteps) / 10000.0, 1.0) * 40
            let activityScore = calculateActivityScore() * 30
            let nutritionScore = calculateNutritionScore() * 30
            
            return .success(stepScore + activityScore + nutritionScore)
            
        } catch {
            return .failure(error as? HealthError ?? .unknown)
        }
    }
    
    // Property with complex observation and side effects
    @Published var mealEntries: [MealEntry] = [] {
        willSet {
            // Validate before accepting new entries
            let invalidEntries = newValue.filter { !$0.isValid }
            if !invalidEntries.isEmpty {
                print("Warning: \(invalidEntries.count) invalid entries")
            }
        }
        didSet {
            // Expensive operations only when needed
            if oldValue != mealEntries {
                Task {
                    await recalculateNutritionMetrics()
                    await syncWithHealthKit()
                }
                
                // Update cache
                mealCache.setValue(mealEntries, forKey: cacheKey)
                
                // Analytics
                logMealUpdate(oldCount: oldValue.count, newCount: mealEntries.count)
            }
        }
    }
    
    // Lazy initialization with error handling
    lazy var healthKitManager: Result<HKHealthStore, HealthError> = {
        guard HKHealthStore.isHealthDataAvailable() else {
            return .failure(.healthKitUnavailable)
        }
        
        let store = HKHealthStore()
        // Additional setup...
        return .success(store)
    }()
    
    // Expensive computed property with caching
    private var _cachedNutritionSummary: NutritionSummary?
    private var _lastNutritionCalculation: Date?
    
    var nutritionSummary: NutritionSummary {
        // Cache for 5 minutes to avoid recalculation
        if let cached = _cachedNutritionSummary,
           let lastCalc = _lastNutritionCalculation,
           Date().timeIntervalSince(lastCalc) < 300 {
            return cached
        }
        
        // Expensive calculation
        let summary = calculateNutritionSummary()
        _cachedNutritionSummary = summary
        _lastNutritionCalculation = Date()
        return summary
    }
    
    // Private properties and methods
    private var lastSyncDate: Date?
    private let mealCache = NSCache<NSString, NSArray>()
    private let cacheKey = NSString("meals")
    
    private func calculateActivityScore() -> Double { 0.8 }
    private func calculateNutritionScore() -> Double { 0.75 }
    private func calculateNutritionSummary() -> NutritionSummary {
        NutritionSummary(calories: 2000, protein: 80, carbs: 250, fat: 70)
    }
    
    private func recalculateNutritionMetrics() async {
        // Async calculation
    }
    
    private func syncWithHealthKit() async {
        syncOperationCount += 1
        defer { syncOperationCount -= 1 }
        // Sync logic
    }
    
    private func logMealUpdate(oldCount: Int, newCount: Int) {
        // Analytics logging
    }
}

// Supporting types
enum HealthError: LocalizedError {
    case insufficientData(String)
    case syncRequired
    case staleData
    case healthKitUnavailable
    case unknown
    
    var errorDescription: String? {
        switch self {
        case .insufficientData(let detail):
            return "Insufficient data: \(detail)"
        case .syncRequired:
            return "Sync with HealthKit required"
        case .staleData:
            return "Health data is outdated"
        case .healthKitUnavailable:
            return "HealthKit is not available on this device"
        case .unknown:
            return "An unknown error occurred"
        }
    }
}

struct MealEntry: Equatable {
    let id: UUID
    let name: String
    let calories: Int
    let timestamp: Date
    
    var isValid: Bool {
        !name.isEmpty && calories > 0 && calories < 10000
    }
}

struct NutritionSummary {
    let calories: Int
    let protein: Int
    let carbs: Int
    let fat: Int
}
```

## Common Pitfalls & Best Practices

Coming from Java/TypeScript/Kotlin, here are the critical mistakes to avoid:

**Overusing Computed Properties**: In TypeScript, you might use getters liberally. In Swift, computed properties should only be used when the calculation is lightweight. Heavy computations should be methods or cached. A computed property that queries a database or makes network calls is a code smell.

**Ignoring Property Observer Infinite Loops**: Setting a property within its own didSet can cause infinite recursion. This is different from Java where setters explicitly control flow. Always check if the value actually changed before triggering side effects.

**Misunderstanding Lazy Properties**: Unlike Kotlin's lazy delegates which are thread-safe by default, Swift's lazy properties are NOT thread-safe. In concurrent contexts, you need additional synchronization. Also, lazy properties can't have property observers.

**Property Wrapper Overengineering**: Just because property wrappers exist doesn't mean you should create them for everything. They add complexity. Use built-in ones first, create custom ones only for truly reusable patterns.

**Memory Implications**: Stored properties increase instance size, computed properties don't but cost CPU on access. Property observers add minimal overhead. Lazy properties keep their closure until initialized, potentially creating retain cycles if the closure captures self.

**iOS Conventions**: Always use @Published for ObservableObject properties that affect UI. Use UserDefaults for simple persistence, not complex objects. Follow Apple's naming: use "is" prefix for boolean properties (isLoading, not loading).

## Integration with SwiftUI & iOS Development

Property wrappers are the backbone of SwiftUI's reactivity. Here's how they work together:

```swift
import SwiftUI
import SwiftData

// SwiftUI View using property wrappers
struct NutritionTrackerView: View {
    // SwiftUI property wrappers
    @State private var currentCalories: Int = 0
    @Binding var dailyGoal: Int  // Passed from parent
    @AppStorage("username") private var userName: String = "User"
    @Environment(\.dismiss) private var dismiss
    
    // SwiftData integration
    @Query(sort: \MealRecord.timestamp, order: .reverse)
    private var recentMeals: [MealRecord]
    
    // Computed property for UI logic
    private var progressColor: Color {
        let progress = Double(currentCalories) / Double(dailyGoal)
        if progress < 0.5 { return .blue }
        else if progress < 0.9 { return .green }
        else if progress <= 1.0 { return .yellow }
        else { return .red }
    }
    
    var body: some View {
        VStack {
            // Computed property updates automatically
            ProgressView(value: Double(currentCalories), total: Double(dailyGoal))
                .tint(progressColor)
            
            // Property observers in action
            Stepper("Calories: \(currentCalories)", value: $currentCalories, step: 50)
                .onChange(of: currentCalories) { oldValue, newValue in
                    // React to changes - SwiftUI's property observer
                    if newValue > dailyGoal {
                        triggerGoalExceededAlert()
                    }
                    saveToHealthKit(calories: newValue)
                }
            
            // Using lazy loading for expensive views
            LazyVStack {
                ForEach(recentMeals) { meal in
                    MealRowView(meal: meal)
                }
            }
        }
    }
    
    private func triggerGoalExceededAlert() {
        // Handle goal exceeded
    }
    
    private func saveToHealthKit(calories: Int) {
        // Save to HealthKit
    }
}

// SwiftData model with property observers
@Model
final class MealRecord {
    var name: String {
        didSet {
            // Automatically update search index
            updateSearchIndex()
        }
    }
    
    var calories: Int {
        didSet {
            // Validate and update related data
            if calories < 0 { calories = 0 }
            lastModified = Date()
        }
    }
    
    var timestamp: Date
    var lastModified: Date
    
    // Computed property for SwiftData queries
    var isHighCalorie: Bool {
        calories > 500
    }
    
    init(name: String, calories: Int) {
        self.name = name
        self.calories = calories
        self.timestamp = Date()
        self.lastModified = Date()
    }
    
    private func updateSearchIndex() {
        // Update search index logic
    }
}

// Custom ObservableObject with property patterns
class NutritionViewModel: ObservableObject {
    // Published properties trigger UI updates
    @Published var meals: [MealRecord] = [] {
        didSet {
            updateDailySummary()
        }
    }
    
    // Computed property for UI binding
    var totalCalories: Int {
        meals.reduce(0) { $0 + $1.calories }
    }
    
    var averageCaloriesPerMeal: Double {
        guard !meals.isEmpty else { return 0 }
        return Double(totalCalories) / Double(meals.count)
    }
    
    // Lazy loading for expensive resources
    lazy var nutritionCalculator: NutritionCalculator = {
        NutritionCalculator(profile: userProfile)
    }()
    
    private let userProfile: UserProfile
    
    init(userProfile: UserProfile) {
        self.userProfile = userProfile
    }
    
    private func updateDailySummary() {
        // Update summary calculations
    }
}

struct UserProfile {
    let age: Int
    let weight: Double
}

struct NutritionCalculator {
    let profile: UserProfile
}
```

## Production Considerations

**Testing Property-Based Code**: Property observers and computed properties need special testing attention. You must test not just the final value but also the side effects:

```swift
import XCTest

class NutritionTrackerTests: XCTestCase {
    func testPropertyObservers() {
        let tracker = NutritionTracker()
        
        var willSetCalled = false
        var didSetCalled = false
        
        // Mock or use test doubles to verify observer calls
        let expectation = expectation(description: "Property observer called")
        
        tracker.proteinGrams = 50.0
        
        // Verify side effects occurred
        XCTAssertTrue(didSetCalled)
        
        wait(for: [expectation], timeout: 1.0)
    }
    
    func testComputedPropertyConsistency() {
        let model = EatingPatternModel()
        model.targetFastingHours = 16
        
        // Test computed property updates correctly
        XCTAssertEqual(model.fastingProgress, 0.0)
        
        model.lastMealTime = Date().addingTimeInterval(-8 * 3600)
        XCTAssertEqual(model.currentFastingHours, 8)
        XCTAssertEqual(model.fastingProgress, 0.5, accuracy: 0.01)
    }
    
    func testLazyPropertyInitialization() {
        let service = HealthDataService()
        
        // Verify lazy property not initialized
        // Use Mirror for reflection (be careful in production)
        let mirror = Mirror(reflecting: service)
        
        // Access lazy property
        _ = service.healthKitManager
        
        // Verify it's now initialized
    }
}
```

**Debugging Techniques**: Use breakpoints in property observers to track state changes. In Xcode, you can set symbolic breakpoints on property setters. Use the Memory Graph Debugger to identify retain cycles with lazy properties. Print statements in willSet/didSet are invaluable during development.

**iOS 17+ Considerations**: The new @Observable macro (iOS 17+) changes how you write observable objects, replacing @Published with simpler syntax. SwiftData's @Model macro automatically makes properties observable. The Observation framework provides more efficient updates than Combine.

**Performance Optimizations**: Computed properties are recalculated on every access - cache expensive calculations. Property observers fire even if the value doesn't change (setting 5 to 5), so check oldValue != newValue for expensive operations. Lazy properties are perfect for heavyweight objects that might not be used. Avoid complex logic in willSet - it blocks the setter. Do heavy work in didSet or async tasks.

## Exercises for Mastery

**Exercise 1: TypeScript to Swift Migration**
Convert this TypeScript class to Swift using appropriate property patterns:

```typescript
class FitnessTracker {
    private _steps: number = 0;
    private _lastSync?: Date;
    
    get steps(): number { return this._steps; }
    set steps(value: number) {
        if (value < 0) throw new Error("Invalid");
        this._steps = value;
        this.syncToCloud();
    }
    
    get needsSync(): boolean {
        if (!this._lastSync) return true;
        return Date.now() - this._lastSync.getTime() > 3600000;
    }
    
    private syncToCloud() { /* ... */ }
}
```

**Exercise 2: Build a Water Intake Tracker**
Create a SwiftUI view with a model that tracks daily water intake using:
- Property observers to validate intake amounts (0-5000ml)
- Computed property for percentage of daily goal
- Lazy property for notification manager
- Custom property wrapper @Persisted that automatically saves to UserDefaults

**Exercise 3: Health App Mini-Challenge**
Implement a fasting timer for your health app that:
- Uses property observers to trigger notifications when fasting windows start/end
- Has computed properties for "time until next meal" and "fasting streak days"
- Implements a @Clamped property wrapper for safe fasting hour limits (12-24 hours)
- Includes a lazy-loaded chart view that only initializes when the user opens statistics
- Integrates with SwiftData to persist fasting sessions

Your solution should handle edge cases like time zone changes, daylight saving time, and invalid date inputs.

Remember, these property features are fundamental to iOS development. They're not just syntax sugar - they're the foundation of reactive programming in SwiftUI and essential for building professional apps. Focus on understanding when to use each type rather than memorizing syntax. The patterns you learn here will apply to every iOS app you build.

