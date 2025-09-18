---
title: "Chapter 15"
weight: 150
---

# Swift Collections and Data Structures: A Comprehensive Guide

## Context & Background

Swift's collection types form the backbone of data manipulation in iOS apps, much like how you use arrays, maps, and sets in TypeScript or ArrayList, HashMap, and HashSet in Java. However, Swift takes a unique approach that blends safety, performance, and expressiveness.

In your TypeScript/Angular world, you're familiar with arrays (`number[]`), objects as dictionaries (`{[key: string]: value}`), and ES6 Sets. In Java, you work with `List<T>`, `Map<K,V>`, and `Set<T>`. Swift's collections share these concepts but with crucial differences: they're value types (structs) rather than reference types, which means they have copy-on-write semantics - a concept that might feel foreign coming from Java's reference-based collections.

In production iOS apps, you'll use these collections constantly. Arrays will store your health data readings from HealthKit, dictionaries will map dates to eating patterns, sets will help you track unique meal types, and you'll implement collection protocols when creating custom data structures for specialized health metrics. The collection protocols are particularly powerful - they're like Java's `Iterable` or TypeScript's iterator protocol, but with a richer hierarchy that enables sophisticated generic programming.

## Core Understanding

Let me break down Swift's collection architecture into its fundamental layers. At the base, we have protocols that define collection behavior. Think of these like interfaces in TypeScript or Java, but more powerful because they can provide default implementations. The `Sequence` protocol is the foundation - anything you can iterate over once. Above that, `Collection` adds the ability to traverse multiple times and access elements by index. `BidirectionalCollection` adds backward traversal, similar to Java's `ListIterator`.

The mental model you should adopt is this: Swift collections are value types that efficiently share storage until mutation occurs. When you assign an array to another variable, no copying happens immediately. Only when you modify one of them does Swift create a separate copy. This is radically different from Java where `ArrayList a = b` creates two references to the same object, or JavaScript where arrays are always references.

Here's the basic syntax and usage patterns:

```swift
// Arrays - ordered, allows duplicates
var meals: [String] = ["breakfast", "lunch", "dinner"]
let firstMeal = meals[0]  // subscript access
meals.append("snack")      // mutation

// Dictionaries - unordered key-value pairs
var caloriesByMeal: [String: Int] = ["breakfast": 400, "lunch": 600]
let breakfastCalories = caloriesByMeal["breakfast"] // returns Optional<Int>
caloriesByMeal["dinner"] = 800  // add or update

// Sets - unordered, unique values only
var uniqueFoods: Set<String> = ["apple", "banana", "apple"] // duplicate removed
uniqueFoods.insert("orange")  // returns (inserted: Bool, memberAfterInsert: String)

// Collection protocols in action
let numbers = [1, 2, 3, 4, 5]
for num in numbers {  // Sequence protocol enables for-in
    print(num)
}
let reversed = numbers.reversed()  // BidirectionalCollection feature
```

## Practical Code Examples

### 1. Basic Example: Simple Collection Operations

```swift
import Foundation

// Basic array operations - similar to TypeScript arrays
var healthMetrics: [Double] = [120.5, 118.3, 122.1]  // blood pressure readings

// Swift way: using append (like push in JS)
healthMetrics.append(119.7)

// Wrong way (TypeScript habit): trying to use push
// healthMetrics.push(119.7)  // ❌ No push method in Swift

// Array slices - powerful but different from JS slice()
let recentReadings = healthMetrics[1...3]  // Creates an ArraySlice, not Array
// Note: ArraySlice shares memory with original array - efficient but be careful!

// Convert slice back to Array if needed (important for SwiftUI)
let recentArray = Array(recentReadings)

// Dictionary with default values - no undefined/null checks needed!
var mealCalories: [String: Int] = [:]

// Wrong way (Java/TS habit): manual null checking
if let calories = mealCalories["breakfast"] {
    print("Breakfast: \(calories)")
} else {
    print("Breakfast: 0")  // Default handling
}

// Swift way: using default values
let breakfastCals = mealCalories["breakfast", default: 0]  // Clean!
print("Breakfast: \(breakfastCals)")

// Or use defaulting subscript for mutations
mealCalories["lunch", default: 0] += 500  // Adds to existing or starts at 0

// Sets for unique collections
var vitamins: Set<String> = ["A", "B12", "C", "D"]
let supplements: Set<String> = ["D", "E", "K", "B12"]

// Set operations - much cleaner than Java's retainAll, removeAll, etc.
let commonVitamins = vitamins.intersection(supplements)  // {B12, D}
let allVitamins = vitamins.union(supplements)  // {A, B12, C, D, E, K}
let uniqueToVitamins = vitamins.subtracting(supplements)  // {A, C}
```

### 2. Real-World Example: Health Tracking App Context

```swift
import Foundation
import SwiftData

// Model for your health app
@Model
class EatingSession {
    var timestamp: Date
    var mealType: String
    var calories: Int
    var foods: [String]
    
    init(timestamp: Date, mealType: String, calories: Int, foods: [String]) {
        self.timestamp = timestamp
        self.foods = foods
        self.mealType = mealType
        self.calories = calories
    }
}

class HealthDataProcessor {
    // Using dictionary to group meals by day
    func groupMealsByDay(sessions: [EatingSession]) -> [Date: [EatingSession]] {
        // Wrong way (Java thinking): manual grouping with null checks
        var wrongWay: [Date: [EatingSession]] = [:]
        for session in sessions {
            let dayStart = Calendar.current.startOfDay(for: session.timestamp)
            if var existing = wrongWay[dayStart] {
                existing.append(session)
                wrongWay[dayStart] = existing  // Need to reassign!
            } else {
                wrongWay[dayStart] = [session]
            }
        }
        
        // Swift way: using Dictionary's grouping initializer
        let grouped = Dictionary(grouping: sessions) { session in
            Calendar.current.startOfDay(for: session.timestamp)
        }
        
        return grouped
    }
    
    // Using Set for finding unique foods across time periods
    func analyzeUniqueFoods(sessions: [EatingSession]) -> FoodAnalysis {
        // Flatten all foods into a single set
        let allFoods = sessions.flatMap { $0.foods }
        let uniqueFoods = Set(allFoods)
        
        // Find most common foods using dictionary
        var foodFrequency: [String: Int] = [:]
        for food in allFoods {
            foodFrequency[food, default: 0] += 1
        }
        
        // Sort by frequency (demonstrating collection protocol usage)
        let topFoods = foodFrequency
            .sorted { $0.value > $1.value }  // Sort by count
            .prefix(10)  // Take top 10 (Collection protocol method)
            .map { $0.key }  // Extract just the food names
        
        return FoodAnalysis(
            uniqueCount: uniqueFoods.count,
            uniqueFoods: Array(uniqueFoods),
            topFoods: Array(topFoods),
            frequency: foodFrequency
        )
    }
    
    // Array slices for windowed analysis
    func calculateMovingAverage(values: [Double], windowSize: Int) -> [Double] {
        guard values.count >= windowSize else { return [] }
        
        var averages: [Double] = []
        
        // Using array slices for efficiency
        for i in 0...(values.count - windowSize) {
            let window = values[i..<i+windowSize]  // This is an ArraySlice
            let sum = window.reduce(0, +)
            averages.append(sum / Double(windowSize))
        }
        
        return averages
    }
}

struct FoodAnalysis {
    let uniqueCount: Int
    let uniqueFoods: [String]
    let topFoods: [String]
    let frequency: [String: Int]
}
```

### 3. Production Example: Custom Collection with Full Error Handling

```swift
import Foundation

// Custom collection for circular buffer of health readings
// Useful for maintaining last N readings efficiently
struct CircularHealthBuffer<T>: Collection, BidirectionalCollection {
    private var storage: [T?]
    private var head: Int = 0
    private var count: Int = 0
    let capacity: Int
    
    init(capacity: Int) {
        precondition(capacity > 0, "Capacity must be positive")
        self.capacity = capacity
        self.storage = Array(repeating: nil, count: capacity)
    }
    
    // MARK: - Collection Protocol Requirements
    
    var startIndex: Int { 0 }
    var endIndex: Int { count }
    
    func index(after i: Int) -> Int {
        precondition(i >= 0 && i < endIndex, "Index out of bounds")
        return i + 1
    }
    
    func index(before i: Int) -> Int {
        precondition(i > startIndex && i <= endIndex, "Index out of bounds")
        return i - 1
    }
    
    subscript(position: Int) -> T {
        precondition(position >= 0 && position < count, "Index out of bounds")
        let actualIndex = (head + position) % capacity
        return storage[actualIndex]!  // Safe because we maintain invariants
    }
    
    // MARK: - Custom Methods
    
    mutating func append(_ element: T) {
        let index = (head + count) % capacity
        storage[index] = element
        
        if count < capacity {
            count += 1
        } else {
            // Buffer is full, move head forward
            head = (head + 1) % capacity
        }
    }
    
    var isFull: Bool { count == capacity }
    
    // Convert to regular array (useful for SwiftUI)
    var array: [T] {
        Array(self)
    }
}

// Production-ready health data manager with comprehensive error handling
class HealthDataManager {
    enum DataError: LocalizedError {
        case invalidDateRange
        case insufficientData
        case calculationError(String)
        
        var errorDescription: String? {
            switch self {
            case .invalidDateRange:
                return "Start date must be before end date"
            case .insufficientData:
                return "Not enough data points for calculation"
            case .calculationError(let message):
                return "Calculation failed: \(message)"
            }
        }
    }
    
    private var recentHeartRates = CircularHealthBuffer<Double>(capacity: 100)
    private var dailyCalories: [Date: Int] = [:]
    private var foodDatabase: Set<FoodItem> = []
    
    // Thread-safe access using actor (modern Swift concurrency)
    private let dataLock = NSLock()
    
    func addHeartRateReading(_ rate: Double) throws {
        guard rate > 30 && rate < 250 else {
            throw DataError.calculationError("Heart rate \(rate) is outside valid range")
        }
        
        dataLock.lock()
        defer { dataLock.unlock() }
        
        recentHeartRates.append(rate)
    }
    
    func getStatistics(from startDate: Date, to endDate: Date) throws -> HealthStatistics {
        guard startDate < endDate else {
            throw DataError.invalidDateRange
        }
        
        dataLock.lock()
        defer { dataLock.unlock() }
        
        // Filter calories by date range using collection operations
        let relevantCalories = dailyCalories
            .filter { $0.key >= startDate && $0.key <= endDate }
            .map { $0.value }
        
        guard !relevantCalories.isEmpty else {
            throw DataError.insufficientData
        }
        
        // Calculate statistics using collection protocol methods
        let totalCalories = relevantCalories.reduce(0, +)
        let averageCalories = Double(totalCalories) / Double(relevantCalories.count)
        let maxCalories = relevantCalories.max() ?? 0
        let minCalories = relevantCalories.min() ?? 0
        
        // Recent heart rate analysis
        let recentHRArray = recentHeartRates.array
        let averageHR = recentHRArray.isEmpty ? 0 : 
            recentHRArray.reduce(0, +) / Double(recentHRArray.count)
        
        return HealthStatistics(
            averageCalories: averageCalories,
            maxCalories: maxCalories,
            minCalories: minCalories,
            averageHeartRate: averageHR,
            dataPoints: relevantCalories.count
        )
    }
}

struct HealthStatistics {
    let averageCalories: Double
    let maxCalories: Int
    let minCalories: Int
    let averageHeartRate: Double
    let dataPoints: Int
}

struct FoodItem: Hashable {
    let id: UUID
    let name: String
    let calories: Int
    
    // Hashable conformance for Set storage
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
    
    static func == (lhs: FoodItem, rhs: FoodItem) -> Bool {
        lhs.id == rhs.id
    }
}
```

## Common Pitfalls & Best Practices

Coming from Java/TypeScript/Kotlin, you'll likely encounter several conceptual hurdles with Swift collections. First, the value semantics will trip you up. In Java, when you pass an ArrayList to a method, that method can modify the original list. In Swift, arrays are copied (conceptually) when passed to functions, though the actual copying is deferred until mutation through copy-on-write optimization. This means you need to explicitly use `inout` parameters or return modified collections.

Memory implications are quite different too. Swift's copy-on-write means collections share storage until mutation, making passing them around cheap. However, be careful with array slices - they maintain a reference to the entire original array, potentially keeping large amounts of memory alive. Always convert slices to arrays if you're storing them long-term.

Performance-wise, Swift collections are generally very efficient, but there are gotchas. Inserting at the beginning of an array is O(n), just like JavaScript. Use `deque` (in Swift Collections package) if you need efficient insertion at both ends. Dictionary lookups are O(1) average case, but remember that Swift's dictionaries maintain insertion order as of recent versions, unlike Java's HashMap.

Apple's conventions favor immutability where possible. Use `let` for collections you won't modify, and prefer functional methods like `map`, `filter`, and `reduce` over manual loops. This aligns well with SwiftUI's declarative nature and makes your code more predictable.

## Integration with SwiftUI & iOS Development

SwiftUI is built around the idea of deriving views from data, and collections are central to this. The `ForEach` view requires collections that conform to `RandomAccessCollection` with `Identifiable` elements or stable IDs. This is why understanding collection protocols matters - you can't just throw any sequence at SwiftUI.

When working with HealthKit, you'll receive arrays of `HKSample` objects that you'll need to transform and group. SwiftData queries return arrays that SwiftUI can observe for changes. Here's how these pieces fit together:

```swift
import SwiftUI
import SwiftData
import HealthKit

struct HealthDashboard: View {
    @Query(sort: \EatingSession.timestamp, order: .reverse) 
    private var recentSessions: [EatingSession]
    
    @State private var groupedByMeal: [String: [EatingSession]] = [:]
    @State private var healthKitData: [HKQuantitySample] = []
    
    var body: some View {
        List {
            // SwiftUI automatically handles collection changes
            Section("Recent Meals") {
                ForEach(recentSessions) { session in
                    MealRow(session: session)
                }
            }
            
            // Using computed property for derived collections
            Section("By Meal Type") {
                ForEach(mealTypeSummaries, id: \.mealType) { summary in
                    HStack {
                        Text(summary.mealType)
                        Spacer()
                        Text("\(summary.totalCalories) cal")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            
            // Working with HealthKit arrays
            Section("Heart Rate Trends") {
                if !healthKitData.isEmpty {
                    HeartRateChart(samples: healthKitData)
                }
            }
        }
        .task {
            // Load and transform HealthKit data
            await loadHealthKitData()
            
            // Group sessions using Dictionary(grouping:)
            groupedByMeal = Dictionary(grouping: recentSessions) { $0.mealType }
        }
    }
    
    private var mealTypeSummaries: [MealSummary] {
        groupedByMeal.map { mealType, sessions in
            MealSummary(
                mealType: mealType,
                totalCalories: sessions.reduce(0) { $0 + $1.calories }
            )
        }
        .sorted { $0.mealType < $1.mealType }  // Collections are sortable
    }
    
    private func loadHealthKitData() async {
        // HealthKit returns arrays of samples
        // Transform them for your UI needs
        guard HKHealthStore.isHealthDataAvailable() else { return }
        
        let store = HKHealthStore()
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        
        // This would return an array of HKQuantitySample
        // Process using collection methods for display
    }
}

struct MealSummary {
    let mealType: String
    let totalCalories: Int
}

struct MealRow: View {
    let session: EatingSession
    
    var body: some View {
        VStack(alignment: .leading) {
            Text(session.mealType)
                .font(.headline)
            
            // Display foods as comma-separated list
            Text(session.foods.joined(separator: ", "))
                .font(.caption)
                .foregroundStyle(.secondary)
        }
    }
}

struct HeartRateChart: View {
    let samples: [HKQuantitySample]
    
    var body: some View {
        // Chart implementation using the samples array
        Text("Chart with \(samples.count) data points")
    }
}
```

## Production Considerations

Testing collections requires attention to edge cases. Empty collections, single-element collections, and collections at capacity limits all need coverage. Use XCTest assertions specifically designed for collections:

```swift
import XCTest

class CollectionTests: XCTestCase {
    func testCircularBuffer() {
        var buffer = CircularHealthBuffer<Int>(capacity: 3)
        
        // Test empty state
        XCTAssertTrue(buffer.isEmpty)
        XCTAssertEqual(buffer.count, 0)
        
        // Test filling
        buffer.append(1)
        buffer.append(2)
        buffer.append(3)
        XCTAssertEqual(Array(buffer), [1, 2, 3])
        
        // Test overflow behavior
        buffer.append(4)
        XCTAssertEqual(Array(buffer), [2, 3, 4])  // First element dropped
        
        // Test collection protocol conformance
        let doubled = buffer.map { $0 * 2 }
        XCTAssertEqual(doubled, [4, 6, 8])
    }
    
    func testDictionaryDefaults() {
        var dict: [String: Int] = [:]
        
        // Test default subscript
        dict["missing", default: 0] += 1
        XCTAssertEqual(dict["missing"], 1)
        
        // Test grouping
        let numbers = [1, 2, 3, 4, 5]
        let grouped = Dictionary(grouping: numbers) { $0 % 2 == 0 ? "even" : "odd" }
        XCTAssertEqual(grouped["odd"], [1, 3, 5])
        XCTAssertEqual(grouped["even"], [2, 4])
    }
}
```

For debugging, Xcode's debugger has excellent collection visualization. Use `po` command to print collections, and the Variables View shows collection contents inline. For performance profiling, Instruments' Time Profiler can identify collection operations that are bottlenecks.

Since you're targeting iOS 17+, you have access to all modern collection features including the new Swift Algorithms package methods. Performance is generally excellent, but watch for unnecessary copying - use `inout` parameters when modifying large collections in functions, and be mindful of array slice lifetimes.

## Exercises for Mastery

### Exercise 1: Migration from TypeScript Patterns
Create a food diary manager that tracks meals over the past 7 days. Start by implementing it how you would in TypeScript (using classes and reference-based updates), then refactor to Swift's value-type approach:

```swift
// Your task: Implement both versions and compare the approaches
protocol FoodDiaryProtocol {
    func addMeal(_ meal: Meal, for date: Date)
    func getMeals(for date: Date) -> [Meal]
    func getCaloriesByDay() -> [Date: Int]
    func findDaysExceedingCalories(_ limit: Int) -> Set<Date>
}

struct Meal {
    let id: UUID
    let name: String
    let calories: Int
    let timestamp: Date
}

// Version 1: How you'd approach it from TS/Java background
class FoodDiaryReference: FoodDiaryProtocol {
    // Implement using reference semantics (classes)
}

// Version 2: The Swift way
struct FoodDiaryValue: FoodDiaryProtocol {
    // Implement using value semantics and proper collection usage
}
```

### Exercise 2: Collection Protocol Mastery
Implement a custom `SlidingWindow` collection that provides a sliding window view over any collection, perfect for analyzing trends in health data:

```swift
// Your task: Make this work with any Collection, not just Array
struct SlidingWindow<Base: Collection>: Collection {
    let base: Base
    let windowSize: Int
    
    // Implement Collection protocol requirements
    // Each element should be a SubSequence of the base collection
    // representing a window of the specified size
}

// Should work like this:
let heartRates = [60, 62, 65, 70, 68, 66, 64]
let windows = SlidingWindow(base: heartRates, windowSize: 3)
// windows should yield: [60,62,65], [62,65,70], [65,70,68], [70,68,66], [68,66,64]
```

### Exercise 3: Health App Mini-Challenge
Build a meal pattern analyzer that identifies eating patterns using all three collection types:

```swift
struct MealPatternAnalyzer {
    private var meals: [EatingSession] = []
    
    // Find patterns like "always eats dessert after dinner"
    // or "skips breakfast on weekdays"
    func analyzePatterns(over days: Int) -> PatternReport {
        // Use:
        // - Arrays for ordered meal sequences
        // - Dictionaries for frequency analysis
        // - Sets for unique pattern identification
        
        // Your implementation here
    }
    
    struct PatternReport {
        let mealSequences: [[String]]  // Common meal orders
        let mealFrequency: [String: Int]  // How often each meal type occurs
        let uniquePatterns: Set<String>  // Unique behavioral patterns found
        let recommendations: [String]  // Based on the analysis
    }
}
```

These exercises progressively build your understanding from familiar patterns toward Swift-specific idioms, directly applicable to your health tracking app. Focus on understanding when each collection type is most appropriate and how the collection protocols enable generic, reusable code that works with any collection type.

