# Advanced Functional Concepts in Swift

Given that you're covering five substantial advanced functional concepts, I'll provide a comprehensive guide that connects each to your existing knowledge while building toward production iOS development skills. These concepts will significantly enhance your ability to write clean, expressive Swift code for your health tracking app.

## Context & Background

Swift's advanced functional features represent a sophisticated blend of functional programming paradigms with practical iOS development needs. These five concepts form the backbone of modern Swift's expressiveness and power.

**Result Type and Error Handling** provides a functional alternative to traditional try-catch mechanisms, similar to Java's Optional or Kotlin's Result type, but with deeper integration into Swift's type system. Unlike TypeScript's Promise rejection or Java's checked exceptions, Swift's Result type makes error handling explicit in the type signature while remaining composable.

**Lazy Properties and Sequences** optimize performance by deferring computation, much like Java's Stream API or Kotlin's sequences, but with compile-time guarantees about initialization. This differs from Angular's lazy loading modules in that it operates at the property level rather than the module level.

**Custom Operators and Operator Overloading** allows you to create domain-specific languages within Swift, similar to Kotlin's operator overloading but more flexible. Unlike Java which prohibits operator overloading, or TypeScript which has limited support, Swift embraces this for creating expressive APIs.

**Function Builders (now Result Builders)** enable DSL creation for declarative syntax, powering SwiftUI's entire view system. Think of them as sophisticated template engines that work at compile-time, similar to Angular's template syntax but type-safe and composable. They're like Kotlin's type-safe builders but more powerful.

**KeyPaths and Dynamic Member Lookup** provide type-safe reflection and property access, combining the safety you expect from statically-typed languages with the flexibility of dynamic languages. They're similar to TypeScript's keyof operator and mapped types, but with runtime capabilities.

In production iOS apps, you'll use these concepts constantly. Result types handle network responses and data validation in your health app. Lazy properties optimize memory when loading health data. Result builders create your entire UI in SwiftUI. KeyPaths enable reactive bindings and data observation patterns throughout your app.

## Core Understanding

Let's break down each concept into its fundamental components:

### Result Type and Error Handling

The Result type encapsulates either a success value or a failure error, making error handling explicit and composable. Think of it as a type-safe union that forces you to handle both success and failure cases. The mental model is a box that contains either your data or an error, never both, never neither.

```swift
// Basic syntax
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}
```

### Lazy Properties and Sequences

Lazy properties delay initialization until first access, while lazy sequences defer computation of elements. The mental model is a promise to compute something later, not a computed value. This differs from computed properties which recalculate on every access.

```swift
// Lazy stored property - computed once on first access
lazy var expensiveData = computeExpensiveData()

// Lazy sequence - transforms elements only when needed
let lazyMapped = array.lazy.map { transform($0) }
```

### Custom Operators

Custom operators let you define new symbols or overload existing ones for your types. The mental model is creating a mini-language for your domain. Swift categorizes operators by position (prefix, infix, postfix) and precedence.

```swift
// Define a custom operator
infix operator ≈: ComparisonPrecedence

// Implement it for your type
static func ≈ (lhs: Double, rhs: Double) -> Bool {
    abs(lhs - rhs) < 0.001
}
```

### Result Builders

Result builders transform a series of statements into a single value, enabling DSL creation. The mental model is a compiler plugin that rewrites your code. SwiftUI's entire declarative syntax is built on this.

```swift
@resultBuilder
struct ArrayBuilder {
    static func buildBlock(_ components: Int...) -> [Int] {
        components
    }
}
```

### KeyPaths

KeyPaths represent references to properties that can be stored and applied later. Think of them as type-safe, composable property accessors. The mental model is a strongly-typed path through your object graph.

```swift
// Creating and using a keypath
let keyPath = \Person.name
let name = person[keyPath: keyPath]
```

## Practical Code Examples

### 1. Basic Example: Understanding the Fundamentals

```swift
import Foundation

// MARK: - Result Type Basic Usage
enum HealthDataError: Error {
    case noData
    case invalidFormat
    case insufficientSamples
}

// Simple function returning Result
func calculateBMI(weight: Double?, height: Double?) -> Result<Double, HealthDataError> {
    // Guard against nil values - common pattern
    guard let weight = weight, let height = height else {
        return .failure(.noData)
    }
    
    guard height > 0 else {
        return .failure(.invalidFormat)
    }
    
    let bmi = weight / (height * height)
    return .success(bmi)
}

// Using the Result
let bmiResult = calculateBMI(weight: 70, height: 1.75)

// Pattern matching approach (Swift way)
switch bmiResult {
case .success(let bmi):
    print("Your BMI is: \(bmi)")
case .failure(let error):
    print("Error calculating BMI: \(error)")
}

// Map and flatMap work on Result (functional approach)
let bmiCategory = bmiResult.map { bmi in
    switch bmi {
    case ..<18.5: return "Underweight"
    case 18.5..<25: return "Normal"
    case 25..<30: return "Overweight"
    default: return "Obese"
    }
}

// MARK: - Lazy Properties Basic
class HealthDataProcessor {
    // This expensive computation only happens once, on first access
    // Similar to Kotlin's lazy delegate
    lazy var historicalAnalysis: [String: Double] = {
        print("Computing historical analysis...") // This only prints once
        // Simulate expensive computation
        return [
            "averageSteps": 8543.2,
            "averageCalories": 2234.5,
            "averageSleep": 7.3
        ]
    }()
    
    // Common mistake from Java/TypeScript: trying to use 'let' with lazy
    // lazy let analysis = ... // ❌ Won't compile! lazy only works with var
}

// MARK: - Lazy Sequences Basic
let steps = [1000, 2000, 3000, 4000, 5000]

// Eager evaluation (TypeScript/Java style) - processes all immediately
let doubledEager = steps.map { step in
    print("Eager: Doubling \(step)")
    return step * 2
}

// Lazy evaluation (Swift way) - processes only when needed
let doubledLazy = steps.lazy.map { step in
    print("Lazy: Doubling \(step)")
    return step * 2
}

// Nothing printed yet for lazy!
print("First lazy value: \(doubledLazy.first!)") // Only processes first element

// MARK: - Custom Operators Basic
// Define an operator for "approximately equals" useful for floating point comparison
infix operator ≈: ComparisonPrecedence

extension Double {
    static func ≈ (lhs: Double, rhs: Double) -> Bool {
        abs(lhs - rhs) < 0.001
    }
}

// Usage in health calculations
let calculatedCalories = 2234.4999
let expectedCalories = 2234.5
if calculatedCalories ≈ expectedCalories {
    print("Calorie calculations match!")
}

// MARK: - Result Builder Basic
@resultBuilder
struct HealthMetricsBuilder {
    // Build a single metric
    static func buildBlock(_ metrics: HealthMetric...) -> [HealthMetric] {
        metrics
    }
    
    // Handle optional metrics
    static func buildOptional(_ metrics: [HealthMetric]?) -> [HealthMetric] {
        metrics ?? []
    }
}

struct HealthMetric {
    let name: String
    let value: Double
}

// Using the result builder
func dailyMetrics(@HealthMetricsBuilder _ content: () -> [HealthMetric]) -> [HealthMetric] {
    content()
}

let metrics = dailyMetrics {
    HealthMetric(name: "Steps", value: 8500)
    HealthMetric(name: "Calories", value: 2200)
    HealthMetric(name: "Sleep", value: 7.5)
}

// MARK: - KeyPaths Basic
struct UserHealth {
    var weight: Double
    var height: Double
    var age: Int
}

// Store keypaths for later use
let weightPath = \UserHealth.weight
let heightPath = \UserHealth.height

var user = UserHealth(weight: 70, height: 1.75, age: 30)

// Read using keypath
let currentWeight = user[keyPath: weightPath]

// Write using keypath
user[keyPath: weightPath] = 72

// Compose keypaths
struct HealthProfile {
    var userData: UserHealth
}

let composedPath = \HealthProfile.userData.weight
```

### 2. Real-World Example: Health Tracking App Context

```swift
import SwiftUI
import HealthKit

// MARK: - Result Type in HealthKit Integration
class HealthKitManager {
    // Custom error types for your domain
    enum HealthKitError: LocalizedError {
        case authorizationDenied
        case dataNotAvailable
        case queryFailed(underlying: Error)
        
        var errorDescription: String? {
            switch self {
            case .authorizationDenied:
                return "Please enable Health access in Settings"
            case .dataNotAvailable:
                return "No health data available for this period"
            case .queryFailed(let error):
                return "Failed to fetch data: \(error.localizedDescription)"
            }
        }
    }
    
    // Fetch steps with Result type - cleaner than completion handlers
    func fetchSteps(for date: Date) async -> Result<Double, HealthKitError> {
        // Check authorization first
        guard await checkAuthorization() else {
            return .failure(.authorizationDenied)
        }
        
        do {
            let steps = try await queryStepsFromHealthKit(date: date)
            return .success(steps)
        } catch {
            return .failure(.queryFailed(underlying: error))
        }
    }
    
    // Chain multiple operations with Result
    func fetchCompleteHealthData(for date: Date) async -> Result<DailyHealthData, HealthKitError> {
        // Fetch multiple metrics and combine
        let stepsResult = await fetchSteps(for: date)
        let caloriesResult = await fetchCalories(for: date)
        let sleepResult = await fetchSleep(for: date)
        
        // Combine Results functionally
        return stepsResult.flatMap { steps in
            caloriesResult.flatMap { calories in
                sleepResult.map { sleep in
                    DailyHealthData(
                        steps: steps,
                        calories: calories,
                        sleepHours: sleep,
                        date: date
                    )
                }
            }
        }
    }
    
    // Helper functions (simplified)
    private func checkAuthorization() async -> Bool { true }
    private func queryStepsFromHealthKit(date: Date) async throws -> Double { 8500 }
    private func fetchCalories(for date: Date) async -> Result<Double, HealthKitError> { .success(2200) }
    private func fetchSleep(for date: Date) async -> Result<Double, HealthKitError> { .success(7.5) }
}

// MARK: - Lazy Loading in SwiftData Models
import SwiftData

@Model
class HealthRecord {
    var date: Date
    var steps: Int
    var calories: Double
    
    // Lazy load expensive computed analysis
    // Only computed when accessed, cached afterward
    @Transient
    lazy var weeklyTrend: TrendAnalysis = {
        // This would normally query related records
        print("Computing weekly trend for \(date)")
        return TrendAnalysis(
            average: Double(steps),
            trend: .increasing,
            percentageChange: 5.2
        )
    }()
    
    // Lazy sequence for processing large datasets
    @Transient
    lazy var processedDataPoints: LazyMapSequence<[DataPoint], ProcessedPoint> = {
        // Process data points only as needed
        return fetchDataPoints().lazy.map { point in
            ProcessedPoint(
                timestamp: point.timestamp,
                normalizedValue: point.value / 1000.0,
                category: categorize(point.value)
            )
        }
    }()
    
    init(date: Date, steps: Int, calories: Double) {
        self.date = date
        self.steps = steps
        self.calories = calories
    }
    
    private func fetchDataPoints() -> [DataPoint] {
        // Simulate fetching from database
        return (0..<1000).map { DataPoint(timestamp: Date(), value: Double($0 * 100)) }
    }
    
    private func categorize(_ value: Double) -> String {
        switch value {
        case ..<5000: return "Low"
        case 5000..<10000: return "Medium"
        default: return "High"
        }
    }
}

// MARK: - Custom Operators for Health Calculations
// Operator for combining health metrics
infix operator <+>: AdditionPrecedence

struct HealthScore {
    let value: Double
    let weight: Double
    
    // Weighted addition of health scores
    static func <+> (lhs: HealthScore, rhs: HealthScore) -> HealthScore {
        let totalWeight = lhs.weight + rhs.weight
        let weightedValue = (lhs.value * lhs.weight + rhs.value * rhs.weight) / totalWeight
        return HealthScore(value: weightedValue, weight: totalWeight)
    }
}

// Usage in calculating overall health score
let stepsScore = HealthScore(value: 0.8, weight: 1.0)  // 80% of goal
let caloriesScore = HealthScore(value: 0.9, weight: 0.8)  // 90% of goal
let sleepScore = HealthScore(value: 0.7, weight: 1.2)  // 70% of goal

let overallScore = stepsScore <+> caloriesScore <+> sleepScore

// MARK: - Result Builders for SwiftUI Views
@resultBuilder
struct ConditionalViewBuilder {
    static func buildBlock<Content: View>(_ components: Content...) -> some View {
        VStack(spacing: 12) {
            ForEach(0..<components.count, id: \.self) { index in
                components[index]
            }
        }
    }
    
    static func buildOptional<Content: View>(_ component: Content?) -> some View {
        component
    }
    
    static func buildEither<TrueContent: View, FalseContent: View>(
        first component: TrueContent
    ) -> ConditionalView<TrueContent, FalseContent> {
        ConditionalView(trueContent: component, falseContent: nil)
    }
    
    static func buildEither<TrueContent: View, FalseContent: View>(
        second component: FalseContent
    ) -> ConditionalView<TrueContent, FalseContent> {
        ConditionalView(trueContent: nil, falseContent: component)
    }
}

struct ConditionalView<TrueContent: View, FalseContent: View>: View {
    let trueContent: TrueContent?
    let falseContent: FalseContent?
    
    var body: some View {
        if let trueContent = trueContent {
            trueContent
        } else if let falseContent = falseContent {
            falseContent
        }
    }
}

// Using the custom result builder
struct HealthDashboard: View {
    @State private var healthData: DailyHealthData?
    @State private var isLoading = false
    
    var body: some View {
        ScrollView {
            healthContent {
                if isLoading {
                    ProgressView("Loading health data...")
                } else if let data = healthData {
                    HealthMetricCard(title: "Steps", value: "\(Int(data.steps))")
                    HealthMetricCard(title: "Calories", value: "\(Int(data.calories))")
                    HealthMetricCard(title: "Sleep", value: "\(data.sleepHours)h")
                } else {
                    Text("No data available")
                }
            }
        }
    }
    
    @ViewBuilder
    func healthContent<Content: View>(@ConditionalViewBuilder content: () -> Content) -> some View {
        content()
    }
}

// MARK: - KeyPaths for Reactive Bindings
class HealthViewModel: ObservableObject {
    @Published var currentUser = UserHealth(weight: 70, height: 1.75, age: 30)
    @Published var goals = HealthGoals()
    
    // Store keypaths for dynamic property access
    let editableProperties: [(name: String, keyPath: WritableKeyPath<UserHealth, Double>)] = [
        ("Weight", \.weight),
        ("Height", \.height)
    ]
    
    // Generic function using keypaths
    func updateProperty<T>(_ keyPath: WritableKeyPath<UserHealth, T>, value: T) {
        currentUser[keyPath: keyPath] = value
    }
    
    // Observe changes using keypaths
    func observeChanges() {
        // Combine keypaths with KVO for powerful observation
        let observation = currentUser.observe(\.weight, options: [.new, .old]) { user, change in
            print("Weight changed from \(change.oldValue ?? 0) to \(change.newValue ?? 0)")
        }
    }
}

struct HealthGoals {
    var dailySteps: Int = 10000
    var dailyCalories: Double = 2000
}

// Supporting types
struct DailyHealthData {
    let steps: Double
    let calories: Double
    let sleepHours: Double
    let date: Date
}

struct TrendAnalysis {
    enum Trend { case increasing, decreasing, stable }
    let average: Double
    let trend: Trend
    let percentageChange: Double
}

struct DataPoint {
    let timestamp: Date
    let value: Double
}

struct ProcessedPoint {
    let timestamp: Date
    let normalizedValue: Double
    let category: String
}

struct HealthMetricCard: View {
    let title: String
    let value: String
    
    var body: some View {
        VStack {
            Text(title).font(.headline)
            Text(value).font(.title)
        }
        .padding()
        .background(Color.blue.opacity(0.1))
        .cornerRadius(10)
    }
}
```

### 3. Production Example: Advanced Usage with Error Handling

```swift
import SwiftUI
import SwiftData
import Combine

// MARK: - Production-Ready Result Type with Async/Await
protocol HealthDataService {
    func fetchHealthData(for dateRange: DateRange) async -> Result<HealthDataCollection, HealthServiceError>
}

enum HealthServiceError: LocalizedError, Equatable {
    case networkError(code: Int, message: String)
    case parsingError(details: String)
    case authenticationRequired
    case rateLimitExceeded(retryAfter: TimeInterval)
    case validationFailed(fields: [String])
    
    var errorDescription: String? {
        switch self {
        case .networkError(_, let message):
            return message
        case .parsingError(let details):
            return "Data format error: \(details)"
        case .authenticationRequired:
            return "Please sign in to continue"
        case .rateLimitExceeded(let retryAfter):
            return "Too many requests. Try again in \(Int(retryAfter)) seconds"
        case .validationFailed(let fields):
            return "Invalid data in fields: \(fields.joined(separator: ", "))"
        }
    }
    
    var isRetryable: Bool {
        switch self {
        case .networkError(let code, _):
            return code >= 500 || code == 408
        case .rateLimitExceeded:
            return true
        default:
            return false
        }
    }
}

class ProductionHealthDataService: HealthDataService {
    private let session: URLSession
    private let cache: HealthDataCache
    private let retryPolicy: RetryPolicy
    
    init(session: URLSession = .shared,
         cache: HealthDataCache = .shared,
         retryPolicy: RetryPolicy = .default) {
        self.session = session
        self.cache = cache
        self.retryPolicy = retryPolicy
    }
    
    func fetchHealthData(for dateRange: DateRange) async -> Result<HealthDataCollection, HealthServiceError> {
        // Try cache first
        if let cached = cache.get(for: dateRange) {
            return .success(cached)
        }
        
        // Fetch with retry logic
        return await withRetry(maxAttempts: retryPolicy.maxAttempts) { attempt in
            await performFetch(dateRange: dateRange, attempt: attempt)
        }
    }
    
    private func performFetch(dateRange: DateRange, attempt: Int) async -> Result<HealthDataCollection, HealthServiceError> {
        // Network call with comprehensive error handling
        do {
            let url = buildURL(for: dateRange)
            let (data, response) = try await session.data(from: url)
            
            guard let httpResponse = response as? HTTPURLResponse else {
                return .failure(.networkError(code: 0, message: "Invalid response"))
            }
            
            switch httpResponse.statusCode {
            case 200..<300:
                return parseResponse(data)
            case 401:
                return .failure(.authenticationRequired)
            case 429:
                let retryAfter = httpResponse.value(forHTTPHeaderField: "Retry-After")
                    .flatMap(Double.init) ?? 60
                return .failure(.rateLimitExceeded(retryAfter: retryAfter))
            default:
                return .failure(.networkError(
                    code: httpResponse.statusCode,
                    message: HTTPURLResponse.localizedString(forStatusCode: httpResponse.statusCode)
                ))
            }
        } catch {
            return .failure(.networkError(code: -1, message: error.localizedDescription))
        }
    }
    
    private func parseResponse(_ data: Data) -> Result<HealthDataCollection, HealthServiceError> {
        do {
            let decoded = try JSONDecoder().decode(HealthDataCollection.self, from: data)
            
            // Validate the decoded data
            let validationResult = validate(decoded)
            switch validationResult {
            case .success:
                cache.store(decoded, for: decoded.dateRange)
                return .success(decoded)
            case .failure(let error):
                return .failure(error)
            }
        } catch {
            return .failure(.parsingError(details: error.localizedDescription))
        }
    }
    
    private func validate(_ data: HealthDataCollection) -> Result<Void, HealthServiceError> {
        var invalidFields = [String]()
        
        if data.entries.isEmpty {
            invalidFields.append("entries")
        }
        
        for entry in data.entries {
            if entry.steps < 0 {
                invalidFields.append("steps")
            }
            if entry.calories < 0 {
                invalidFields.append("calories")
            }
        }
        
        return invalidFields.isEmpty ? .success(()) : .failure(.validationFailed(fields: invalidFields))
    }
    
    private func buildURL(for dateRange: DateRange) -> URL {
        // URL building logic
        URL(string: "https://api.health.app/data")!
    }
    
    private func withRetry<T>(
        maxAttempts: Int,
        operation: @escaping (Int) async -> Result<T, HealthServiceError>
    ) async -> Result<T, HealthServiceError> {
        for attempt in 1...maxAttempts {
            let result = await operation(attempt)
            
            switch result {
            case .success:
                return result
            case .failure(let error) where error.isRetryable && attempt < maxAttempts:
                let delay = retryPolicy.delay(for: attempt)
                try? await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                continue
            case .failure:
                return result
            }
        }
        
        return await operation(maxAttempts)
    }
}

// MARK: - Production Lazy Loading with Memory Management
@MainActor
class OptimizedHealthDataViewModel: ObservableObject {
    @Published private(set) var visibleData: [HealthDataPoint] = []
    @Published private(set) var isLoading = false
    @Published private(set) var error: HealthServiceError?
    
    // Lazy sequence for infinite scrolling
    private lazy var dataLoader: AsyncLazySequence<HealthDataPoint> = {
        AsyncLazySequence { [weak self] continuation in
            guard let self = self else { return }
            
            Task {
                var offset = 0
                let batchSize = 50
                
                while !Task.isCancelled {
                    let batch = await self.loadBatch(offset: offset, size: batchSize)
                    
                    if batch.isEmpty {
                        continuation.finish()
                        break
                    }
                    
                    for point in batch {
                        continuation.yield(point)
                    }
                    
                    offset += batchSize
                }
            }
        }
    }()
    
    // Lazy cache with automatic memory management
    private lazy var imageCache: NSCache<NSString, UIImage> = {
        let cache = NSCache<NSString, UIImage>()
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024 // 50MB
        
        // Listen for memory warnings
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(clearCache),
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
        
        return cache
    }()
    
    // Lazy computed aggregations with caching
    private lazy var statisticsCalculator = StatisticsCalculator()
    
    private var _weeklyStatistics: WeeklyStatistics?
    var weeklyStatistics: WeeklyStatistics {
        if let cached = _weeklyStatistics,
           cached.isValid(for: Date()) {
            return cached
        }
        
        let stats = statisticsCalculator.calculate(from: visibleData)
        _weeklyStatistics = stats
        return stats
    }
    
    func loadInitialData() async {
        isLoading = true
        error = nil
        
        do {
            // Use lazy sequence to load only what's needed
            visibleData = await dataLoader.prefix(20).collect()
            isLoading = false
        } catch {
            self.error = .networkError(code: -1, message: error.localizedDescription)
            isLoading = false
        }
    }
    
    func loadMore() async {
        guard !isLoading else { return }
        
        // Load next batch lazily
        let nextBatch = await dataLoader
            .dropFirst(visibleData.count)
            .prefix(20)
            .collect()
        
        await MainActor.run {
            visibleData.append(contentsOf: nextBatch)
        }
    }
    
    private func loadBatch(offset: Int, size: Int) async -> [HealthDataPoint] {
        // Simulate async loading
        try? await Task.sleep(nanoseconds: 100_000_000)
        
        guard offset < 500 else { return [] }
        
        return (offset..<min(offset + size, 500)).map { index in
            HealthDataPoint(
                id: UUID(),
                timestamp: Date().addingTimeInterval(TimeInterval(index * 3600)),
                value: Double.random(in: 5000...15000)
            )
        }
    }
    
    @objc private func clearCache() {
        imageCache.removeAllObjects()
        _weeklyStatistics = nil
    }
}

// MARK: - Advanced Operator Overloading with Type Safety
// Complex number type for advanced health calculations (e.g., Fourier analysis of sleep patterns)
struct ComplexNumber: CustomStringConvertible {
    let real: Double
    let imaginary: Double
    
    var description: String {
        if imaginary >= 0 {
            return "\(real) + \(imaginary)i"
        } else {
            return "\(real) - \(abs(imaginary))i"
        }
    }
    
    var magnitude: Double {
        sqrt(real * real + imaginary * imaginary)
    }
    
    var phase: Double {
        atan2(imaginary, real)
    }
}

// Define multiple operators with proper precedence
precedencegroup ComplexPrecedence {
    higherThan: AdditionPrecedence
    lowerThan: MultiplicationPrecedence
    associativity: left
}

infix operator +: AdditionPrecedence
infix operator *: MultiplicationPrecedence
infix operator **: ComplexPrecedence  // Power operator

extension ComplexNumber {
    static func + (lhs: ComplexNumber, rhs: ComplexNumber) -> ComplexNumber {
        ComplexNumber(
            real: lhs.real + rhs.real,
            imaginary: lhs.imaginary + rhs.imaginary
        )
    }
    
    static func * (lhs: ComplexNumber, rhs: ComplexNumber) -> ComplexNumber {
        ComplexNumber(
            real: lhs.real * rhs.real - lhs.imaginary * rhs.imaginary,
            imaginary: lhs.real * rhs.imaginary + lhs.imaginary * rhs.real
        )
    }
    
    static func ** (base: ComplexNumber, exponent: Int) -> ComplexNumber {
        guard exponent > 0 else {
            return ComplexNumber(real: 1, imaginary: 0)
        }
        
        var result = base
        for _ in 1..<exponent {
            result = result * base
        }
        return result
    }
}

// Use in sleep pattern analysis
class SleepPatternAnalyzer {
    func analyzeSleepCycles(_ samples: [Double]) -> [ComplexNumber] {
        // Simplified FFT using complex numbers
        return samples.enumerated().map { index, sample in
            let angle = 2.0 * .pi * Double(index) / Double(samples.count)
            return ComplexNumber(
                real: sample * cos(angle),
                imaginary: sample * sin(angle)
            )
        }
    }
}

// MARK: - Production Result Builders with Error Handling
@resultBuilder
struct SafeViewBuilder {
    static func buildBlock<each Content: View>(_ content: repeat each Content) -> TupleView<(repeat each Content)> {
        TupleView((repeat each content))
    }
    
    static func buildOptional<Content: View>(_ component: Content?) -> AnyView {
        if let component = component {
            return AnyView(component)
        } else {
            return AnyView(EmptyView())
        }
    }
    
    static func buildEither<TrueContent: View, FalseContent: View>(
        first component: TrueContent
    ) -> AnyView {
        AnyView(component)
    }
    
    static func buildEither<TrueContent: View, FalseContent: View>(
        second component: FalseContent
    ) -> AnyView {
        AnyView(component)
    }
    
    static func buildArray<Content: View>(_ components: [Content]) -> AnyView {
        AnyView(ForEach(0..<components.count, id: \.self) { index in
            components[index]
        })
    }
    
    static func buildLimitedAvailability<Content: View>(_ component: Content) -> AnyView {
        AnyView(component)
    }
}

// Production view using custom result builder
struct HealthDataView: View {
    @StateObject private var viewModel = OptimizedHealthDataViewModel()
    @State private var selectedMetric: HealthMetric?
    
    var body: some View {
        ScrollView {
            content {
                HeaderSection(statistics: viewModel.weeklyStatistics)
                
                if viewModel.isLoading {
                    LoadingView()
                } else if let error = viewModel.error {
                    ErrorView(error: error) {
                        Task {
                            await viewModel.loadInitialData()
                        }
                    }
                } else {
                    DataSection(data: viewModel.visibleData)
                    
                    if #available(iOS 18.0, *) {
                        AdvancedVisualization(data: viewModel.visibleData)
                    }
                }
                
                LoadMoreButton {
                    Task {
                        await viewModel.loadMore()
                    }
                }
            }
        }
        .task {
            await viewModel.loadInitialData()
        }
    }
    
    @ViewBuilder
    private func content<Content: View>(@SafeViewBuilder builder: () -> Content) -> some View {
        VStack(alignment: .leading, spacing: 16) {
            builder()
        }
        .padding()
    }
}

// MARK: - Advanced KeyPath Usage with Dynamic Member Lookup
@dynamicMemberLookup
struct DynamicHealthData {
    private var storage: [String: Any] = [:]
    
    subscript(dynamicMember member: String) -> Any? {
        get { storage[member] }
        set { storage[member] = newValue }
    }
    
    // Type-safe subscript using keypaths
    subscript<T>(keyPath keyPath: KeyPath<HealthDataSchema, T>) -> T? {
        let key = String(describing: keyPath)
        return storage[key] as? T
    }
}

struct HealthDataSchema {
    let steps: Int
    let calories: Double
    let heartRate: Int
    let sleepHours: Double
}

// KeyPath-based sorting and filtering
extension Array where Element == HealthDataPoint {
    func sorted<T: Comparable>(by keyPath: KeyPath<Element, T>) -> [Element] {
        self.sorted { $0[keyPath: keyPath] < $1[keyPath: keyPath] }
    }
    
    func filter<T: Equatable>(where keyPath: KeyPath<Element, T>, equals value: T) -> [Element] {
        self.filter { $0[keyPath: keyPath] == value }
    }
    
    func sum<T: AdditiveArithmetic>(of keyPath: KeyPath<Element, T>) -> T {
        self.reduce(T.zero) { $0 + $1[keyPath: keyPath] }
    }
}

// Production-ready keypath-based observation
class KeyPathObserver<Root: NSObject, Value> {
    private var observations: [NSKeyValueObservation] = []
    
    func observe(
        _ object: Root,
        keyPaths: [KeyPath<Root, Value>],
        changeHandler: @escaping (Root, Value) -> Void
    ) {
        observations = keyPaths.map { keyPath in
            object.observe(keyPath, options: [.new]) { object, change in
                if let newValue = change.newValue {
                    changeHandler(object, newValue)
                }
            }
        }
    }
    
    func invalidate() {
        observations.forEach { $0.invalidate() }
        observations.removeAll()
    }
}

// Supporting types for production example
struct DateRange: Codable {
    let start: Date
    let end: Date
}

struct HealthDataCollection: Codable {
    let dateRange: DateRange
    let entries: [HealthDataEntry]
}

struct HealthDataEntry: Codable {
    let date: Date
    let steps: Int
    let calories: Double
}

struct HealthDataPoint: Identifiable {
    let id: UUID
    let timestamp: Date
    let value: Double
}

class HealthDataCache {
    static let shared = HealthDataCache()
    private var cache: [String: (data: HealthDataCollection, timestamp: Date)] = [:]
    
    func get(for range: DateRange) -> HealthDataCollection? {
        let key = "\(range.start)-\(range.end)"
        guard let cached = cache[key],
              Date().timeIntervalSince(cached.timestamp) < 300 else {
            return nil
        }
        return cached.data
    }
    
    func store(_ data: HealthDataCollection, for range: DateRange) {
        let key = "\(range.start)-\(range.end)"
        cache[key] = (data, Date())
    }
}

struct RetryPolicy {
    let maxAttempts: Int
    let baseDelay: TimeInterval
    let maxDelay: TimeInterval
    
    static let `default` = RetryPolicy(
        maxAttempts: 3,
        baseDelay: 1.0,
        maxDelay: 10.0
    )
    
    func delay(for attempt: Int) -> TimeInterval {
        let exponentialDelay = baseDelay * pow(2.0, Double(attempt - 1))
        return min(exponentialDelay, maxDelay)
    }
}

// Async sequence implementation
struct AsyncLazySequence<Element> {
    let producer: (Continuation) -> Void
    
    struct Continuation {
        let yield: (Element) -> Void
        let finish: () -> Void
    }
    
    func prefix(_ maxLength: Int) -> AsyncPrefixSequence<Element> {
        AsyncPrefixSequence(base: self, maxLength: maxLength)
    }
    
    func dropFirst(_ count: Int) -> AsyncDropFirstSequence<Element> {
        AsyncDropFirstSequence(base: self, count: count)
    }
}

struct AsyncPrefixSequence<Element> {
    let base: AsyncLazySequence<Element>
    let maxLength: Int
    
    func collect() async -> [Element] {
        // Simplified implementation
        []
    }
}

struct AsyncDropFirstSequence<Element> {
    let base: AsyncLazySequence<Element>
    let count: Int
    
    func prefix(_ maxLength: Int) -> AsyncPrefixSequence<Element> {
        AsyncPrefixSequence(base: base, maxLength: maxLength)
    }
}

// View components
struct HeaderSection: View {
    let statistics: WeeklyStatistics
    var body: some View {
        Text("Weekly Stats").font(.headline)
    }
}

struct LoadingView: View {
    var body: some View {
        ProgressView()
    }
}

struct ErrorView: View {
    let error: HealthServiceError
    let retry: () -> Void
    
    var body: some View {
        VStack {
            Text(error.localizedDescription)
            Button("Retry", action: retry)
        }
    }
}

struct DataSection: View {
    let data: [HealthDataPoint]
    var body: some View {
        ForEach(data) { point in
            Text("\(point.value)")
        }
    }
}

struct AdvancedVisualization: View {
    let data: [HealthDataPoint]
    var body: some View {
        Text("Advanced Visualization")
    }
}

struct LoadMoreButton: View {
    let action: () -> Void
    var body: some View {
        Button("Load More", action: action)
    }
}

struct WeeklyStatistics {
    let average: Double
    let total: Double
    let calculatedAt: Date
    
    func isValid(for date: Date) -> Bool {
        date.timeIntervalSince(calculatedAt) < 3600
    }
}

class StatisticsCalculator {
    func calculate(from data: [HealthDataPoint]) -> WeeklyStatistics {
        WeeklyStatistics(
            average: data.reduce(0) { $0 + $1.value } / Double(data.count),
            total: data.reduce(0) { $0 + $1.value },
            calculatedAt: Date()
        )
    }
}
```

## Common Pitfalls & Best Practices

Coming from TypeScript, Java, and Kotlin, you'll encounter several Swift-specific patterns that require adjustment in your mental model.

**Result Type Pitfalls**: The most common mistake is treating Result like a Java Optional or TypeScript union type. Unlike Java's Optional which primarily handles null values, Swift's Result explicitly models success and failure paths. Avoid the temptation to use try-catch everywhere like in Java - Result types are often cleaner for expected failures. Never force unwrap Results with `.get()` without checking - this crashes on failure. Instead, use pattern matching or the functional methods like `map` and `flatMap`.

**Lazy Property Mistakes**: Java developers often confuse Swift's lazy properties with computed properties. Lazy properties compute once and cache, while computed properties recalculate every access. The biggest mistake is trying to use `let` with lazy - it must be `var` because it mutates on first access. Be careful with lazy properties in structs - they can cause unexpected copying behavior since accessing them mutates the struct. Always use `[weak self]` in lazy closure properties to avoid retain cycles.

**Operator Overloading Issues**: Coming from Java which prohibits operator overloading, developers often go overboard creating too many custom operators. Stick to domain-appropriate operators that enhance readability. Never override operators in ways that violate mathematical expectations - if you override `+`, it should be commutative if the standard `+` is. Be extremely careful with precedence groups - getting these wrong leads to subtle bugs.

**Result Builder Confusion**: TypeScript developers might expect result builders to work like template literals or JSX, but they're compile-time transformations. The builder methods must be static and pure - no side effects. Each builder method has a specific purpose in the DSL transformation pipeline. Don't try to use instance state in result builders.

**KeyPath Memory Traps**: KeyPaths can create retain cycles if stored in closures without weak references. Unlike Java reflection which is runtime-only, Swift KeyPaths are type-safe but can still hold strong references. When using KeyPaths for observation, always clean up observations to prevent memory leaks.

**Performance Implications**: Lazy sequences are not always faster - for small collections, eager evaluation is often more efficient due to reduced overhead. Result types add minimal overhead compared to throws, but excessive chaining can impact readability. Custom operators are resolved at compile-time, so there's no runtime penalty, but complex precedence can slow compilation. Result builders can generate substantial code - check the expanded code size for complex builders. KeyPaths are generally efficient but avoid using them in tight loops where direct property access would be faster.

**iOS Ecosystem Conventions**: Always prefer Result types for asynchronous operations that can fail predictably. Use lazy properties for expensive computed values that don't change. Reserve custom operators for domain-specific languages where they add clear value. Result builders should follow SwiftUI's patterns - support optional content and conditional rendering. KeyPaths are idiomatic for SwiftUI bindings and Combine publishers.

## Integration with SwiftUI & iOS Development

These functional concepts are deeply integrated into SwiftUI and iOS frameworks, forming the foundation of modern iOS development patterns.

**Result in SwiftUI State Management**: SwiftUI views often need to display loading, success, and error states. Result types model this perfectly. Create a generic `LoadableResult` enum that wraps Result with a loading case. This integrates seamlessly with SwiftUI's declarative syntax, allowing you to switch over the state in your view body.

**Lazy Loading in SwiftUI Lists**: When displaying large datasets from HealthKit or SwiftData, lazy sequences prevent memory issues. SwiftUI's List view works naturally with lazy sequences, only rendering visible items. Combine this with `@StateObject` for ViewModels that lazily load data as users scroll.

**Result Builders Power SwiftUI**: Every SwiftUI view uses the `@ViewBuilder` result builder. Understanding result builders helps you create custom DSLs for complex UI patterns. For your health app, create builders for conditional health metric displays or chart configurations.

**KeyPaths in SwiftUI Bindings**: SwiftUI's `$` syntax for bindings is built on KeyPaths. Use KeyPaths to create dynamic forms where fields are configured from data. This is perfect for settings screens in your health app where different users might see different options.

```swift
// SwiftUI Integration Example
struct HealthDataListView: View {
    @StateObject private var dataManager = HealthDataManager()
    @State private var sortKeyPath: KeyPath<HealthRecord, Date> = \.date
    
    var body: some View {
        List {
            // Result type for state management
            switch dataManager.dataState {
            case .loading:
                ProgressView("Loading health data...")
                
            case .success(let records):
                // Lazy sequence for efficient rendering
                ForEach(records.lazy.sorted(by: sortKeyPath)) { record in
                    HealthRecordRow(record: record)
                }
                
            case .failure(let error):
                ErrorView(error: error) {
                    Task {
                        await dataManager.reload()
                    }
                }
            }
        }
        .refreshable {
            await dataManager.reload()
        }
    }
}

// Custom Result Builder for Health Metrics
@resultBuilder
struct HealthMetricsBuilder {
    static func buildBlock(_ metrics: HealthMetricView...) -> some View {
        VStack(spacing: 12) {
            ForEach(0..<metrics.count, id: \.self) { index in
                metrics[index]
                    .transition(.slide)
            }
        }
    }
}

struct HealthDashboardView: View {
    @EnvironmentObject var healthStore: HealthStore
    
    var body: some View {
        ScrollView {
            metricsSection {
                if healthStore.hasStepsData {
                    HealthMetricView(
                        title: "Steps",
                        value: healthStore[keyPath: \.stepsCount],
                        unit: "steps"
                    )
                }
                
                if healthStore.hasHeartRateData {
                    HealthMetricView(
                        title: "Heart Rate",
                        value: healthStore[keyPath: \.heartRate],
                        unit: "bpm"
                    )
                }
            }
        }
    }
    
    @ViewBuilder
    func metricsSection<Content: View>(
        @HealthMetricsBuilder content: () -> Content
    ) -> some View {
        content()
            .padding()
    }
}
```

**HealthKit Integration**: Result types are perfect for HealthKit's authorization flows. Create a `HealthKitResult<T>` that includes specific health-related errors. Use lazy sequences when processing large health datasets to avoid memory spikes. KeyPaths can dynamically access different health metrics based on user preferences.

**SwiftData Integration**: SwiftData queries return Results naturally. Use lazy properties in your `@Model` classes for expensive computed properties. Create custom operators for complex predicate building. Result builders can construct complex fetch requests declaratively.

## Production Considerations

**Testing Strategies**: Test Result types by explicitly testing both success and failure paths. Create test helpers that construct specific Result states. For lazy properties, test that computation only happens once by adding counters in test. Mock expensive operations in lazy property tests. Test custom operators with edge cases including precedence. Verify result builders generate expected view hierarchies using ViewInspector. Test KeyPath-based code by verifying the correct properties are accessed.

**Debugging Techniques**: Use `po` in the debugger to inspect Result values. Set breakpoints in lazy property initializers to verify single execution. Custom operators can be stepped through like normal functions in the debugger. Use `print(type(of:))` to debug result builder output types. KeyPaths can be printed to see their string representation for debugging.

**iOS 17+ Considerations**: Observation framework in iOS 17+ uses KeyPaths extensively. New SwiftData macro system relies on KeyPaths for queries. Result builders have improved performance in Swift 5.9+. Lazy properties work seamlessly with the new Observable macro. Custom operators can use new Unicode characters in iOS 17+.

**Performance Optimization**: Profile lazy sequence operations with Instruments. Result type chains should be limited to 3-4 levels for readability. Cache expensive lazy properties at appropriate scope levels. Avoid custom operators in performance-critical code paths. Precompile complex result builders when possible. Use KeyPath subscripts instead of repeated KeyPath applications.

**Memory Management**: Always use weak references in lazy closures that capture self. Clear lazy property caches when receiving memory warnings. Result types with large associated values should be processed quickly. Custom operators should not retain objects unexpectedly. Result builders should generate memory-efficient code. KeyPath observations must be invalidated to prevent leaks.

## Exercises for Mastery

### Exercise 1: Result Type Migration
Start with a familiar TypeScript-style Promise-based function and convert it to Swift's Result type.

```swift
// TypeScript style you might write:
/*
async function fetchUserHealth(): Promise<HealthData> {
    try {
        const response = await fetch('/api/health');
        return await response.json();
    } catch (error) {
        throw new Error('Failed to fetch');
    }
}
*/

// Your task: Implement this in Swift using Result
// 1. Create a proper error enum
// 2. Return Result<HealthData, YourError>
// 3. Add retry logic using Result's map/flatMap
// 4. Create a SwiftUI view that displays all states

// Bonus: Create a generic NetworkResult type that can be reused
```

### Exercise 2: Lazy Loading System
Build a lazy-loading image cache system for health achievement badges.

```swift
// Requirements:
// 1. Create a lazy property that initializes an image cache
// 2. Implement lazy sequence for loading images from URLs
// 3. Add memory pressure handling
// 4. Integrate with SwiftUI using @StateObject
// 5. Compare performance with eager loading

// Start with this structure:
class AchievementBadgeCache {
    // Add lazy cache property here
    // Implement lazy loading sequence
    // Handle memory warnings
}

// Advanced: Make it generic to cache any Identifiable type
```

### Exercise 3: Health App DSL
Create a domain-specific language for defining health goals using result builders and custom operators.

```swift
// Goal: Create syntax like this:
/*
let weeklyGoals = HealthGoals {
    Steps >= 10000
    Calories <= 2200
    Sleep ~= 8.0  // approximately 8 hours
    Water >> 2.0  // much greater than 2 liters
    
    if isPremiumUser {
        HeartRateVariability > 50
    }
}
*/

// Your tasks:
// 1. Define the custom operators with appropriate precedence
// 2. Create the result builder
// 3. Make it work with SwiftUI views
// 4. Add validation that runs at compile time
// 5. Create KeyPath-based accessors for the goals

// This combines all five concepts in a practical health app feature
```

These exercises progressively build from familiar patterns toward Swift-specific implementations, culminating in a sophisticated feature that combines all five functional concepts in a production-ready health app context. Each exercise directly applies to your app development goals while building mastery of Swift's functional programming capabilities.

