# Error Handling and Safety in Swift

## Context & Background

Error handling in Swift represents Apple's approach to building safe, predictable applications that gracefully handle failure conditions. Unlike many languages that use exceptions for error handling, Swift uses a more explicit, type-safe approach that forces developers to acknowledge and handle potential failures at compile time.

Coming from your TypeScript, Java, and Kotlin background, you'll find Swift's error handling feels like a hybrid between Java's checked exceptions and Kotlin's Result type. Where Java uses `throws Exception` and `try-catch` blocks, and TypeScript often relies on rejected Promises or thrown Errors, Swift implements a similar but more refined pattern. The key difference is that Swift's error handling is designed to be lightweight—throwing an error in Swift doesn't unwind the call stack like Java exceptions do, making it much more performant.

In your production iOS apps, especially your health tracking app, you'll use error handling constantly. Every HealthKit query might fail due to permissions, every network request could timeout, and every SwiftData operation might encounter database issues. Swift's error handling ensures you're thinking about these failure cases during development rather than discovering them in production. The compiler becomes your safety net, refusing to compile code that doesn't properly handle potential errors.

## Core Understanding

Swift's error handling revolves around four fundamental concepts that work together to create a robust safety system.

First, the **Error protocol** serves as the foundation. Any type conforming to Error can be thrown as an error—this is similar to how any class extending Exception can be thrown in Java, but Swift typically uses enums rather than classes. This makes errors value types that can carry associated data about what went wrong.

Second, **throwing functions** explicitly declare they might fail using the `throws` keyword. This is like Java's checked exceptions but more elegant—a function either throws or it doesn't, and there's no need to specify which errors it throws. When you see `func fetchHealthData() throws -> [HealthRecord]`, you immediately know this operation might fail and must be handled.

Third, the **do-try-catch pattern** provides structured error handling. The `try` keyword marks potentially failing operations, making them visible in your code. This explicitness helps you reason about control flow—you can't accidentally ignore an error like you might with an unchecked exception in Java.

Finally, Swift offers **multiple try variants** for different scenarios. Beyond standard `try`, you have `try?` which converts errors to nil (like Kotlin's safe calls), and `try!` which force-unwraps and crashes on error (use sparingly, like force-unwrapping optionals).

The mental model you should adopt is this: errors in Swift are expected, explicit, and efficient. They're part of your function's contract, not exceptional circumstances. Think of them as another return type rather than disruptions to normal flow.

## Practical Code Examples

### 1. Basic Example: Simple Error Handling

```swift
// Define custom errors using an enum (Swift's preferred approach)
enum ValidationError: Error {
    case tooShort
    case tooLong
    case invalidCharacters
}

// A throwing function - note the 'throws' keyword before the return type
func validateUsername(_ username: String) throws -> String {
    // Guard statements are idiomatic Swift for early returns
    guard username.count >= 3 else {
        throw ValidationError.tooShort
    }
    
    guard username.count <= 20 else {
        throw ValidationError.tooLong
    }
    
    // Check for valid characters using regular expression
    let validPattern = "^[a-zA-Z0-9_]+$"
    guard username.range(of: validPattern, options: .regularExpression) != nil else {
        throw ValidationError.invalidCharacters
    }
    
    return username.lowercased()
}

// Using the throwing function
func demonstrateBasicErrorHandling() {
    // The "Java/TypeScript way" you might expect (but wrong in Swift):
    // let result = validateUsername("ab")  // ❌ Won't compile without 'try'
    
    // The Swift way - explicit error handling
    do {
        // 'try' makes it clear this might fail
        let validUsername = try validateUsername("ab")
        print("Valid username: \(validUsername)")
    } catch ValidationError.tooShort {
        // Specific error handling
        print("Username must be at least 3 characters")
    } catch ValidationError.tooLong {
        print("Username must be 20 characters or less")
    } catch ValidationError.invalidCharacters {
        print("Username can only contain letters, numbers, and underscores")
    } catch {
        // Swift requires exhaustive catching - 'error' is implicitly available
        print("Unexpected error: \(error)")
    }
    
    // Alternative: Using try? for optional result (like Kotlin's safe calls)
    let maybeUsername = try? validateUsername("test_user")
    // maybeUsername is String? - nil if validation failed
    
    // Alternative: Using try! when you're certain it won't fail (dangerous!)
    let definitelyValid = try! validateUsername("john_doe")
    // Crashes if validation fails - use only when failure is programmer error
}
```

### 2. Real-World Example: Health Data Fetching

```swift
import HealthKit
import SwiftData

// More sophisticated error handling for your health app
enum HealthDataError: LocalizedError {
    case healthKitUnavailable
    case authorizationDenied
    case noDataAvailable(dateRange: DateInterval)
    case dataProcessingFailed(reason: String)
    
    // LocalizedError protocol provides user-friendly messages
    var errorDescription: String? {
        switch self {
        case .healthKitUnavailable:
            return "Health data is not available on this device"
        case .authorizationDenied:
            return "Please grant permission to access health data in Settings"
        case .noDataAvailable(let dateRange):
            let formatter = DateFormatter()
            formatter.dateStyle = .medium
            return "No data found between \(formatter.string(from: dateRange.start)) and \(formatter.string(from: dateRange.end))"
        case .dataProcessingFailed(let reason):
            return "Failed to process health data: \(reason)"
        }
    }
    
    // Additional context for debugging
    var failureReason: String? {
        switch self {
        case .healthKitUnavailable:
            return "HealthKit is not supported on this device or simulator"
        case .authorizationDenied:
            return "User has not granted necessary HealthKit permissions"
        case .noDataAvailable:
            return "Query returned empty dataset"
        case .dataProcessingFailed:
            return "Data transformation or calculation failed"
        }
    }
}

// Service class for health data operations
class HealthDataService {
    private let healthStore = HKHealthStore()
    
    // Async throwing function - combines modern concurrency with error handling
    func fetchDailySteps(for date: Date) async throws -> Int {
        // Check HealthKit availability
        guard HKHealthStore.isHealthDataAvailable() else {
            throw HealthDataError.healthKitUnavailable
        }
        
        // Request authorization (this is also async and can throw)
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let authStatus = healthStore.authorizationStatus(for: stepType)
        
        guard authStatus != .notDetermined else {
            // Request permission - this would be more complex in production
            throw HealthDataError.authorizationDenied
        }
        
        // Create date range for query
        let calendar = Calendar.current
        let startOfDay = calendar.startOfDay(for: date)
        let endOfDay = calendar.date(byAdding: .day, value: 1, to: startOfDay)!
        
        // Fetch data using HealthKit query
        let predicate = HKQuery.predicateForSamples(
            withStart: startOfDay,
            end: endOfDay,
            options: .strictStartDate
        )
        
        // In real app, this would use HKStatisticsQuery
        // Simulating potential failure points
        let randomSuccess = Bool.random()
        if !randomSuccess {
            throw HealthDataError.noDataAvailable(
                dateRange: DateInterval(start: startOfDay, end: endOfDay)
            )
        }
        
        return Int.random(in: 5000...15000)
    }
    
    // Demonstrating Result type as alternative to throwing
    func processStepsData(_ steps: Int) -> Result<String, HealthDataError> {
        guard steps > 0 else {
            return .failure(.dataProcessingFailed(reason: "Invalid step count"))
        }
        
        let analysis = switch steps {
        case 0..<5000: "Below recommended daily activity"
        case 5000..<10000: "Good activity level"
        case 10000...: "Excellent! Goal achieved!"
        default: "Unable to analyze"
        }
        
        return .success(analysis)
    }
}

// SwiftUI View demonstrating error handling in UI
struct DailyStepsView: View {
    @State private var steps: Int?
    @State private var errorMessage: String?
    @State private var isLoading = false
    
    let healthService = HealthDataService()
    
    var body: some View {
        VStack(spacing: 20) {
            if isLoading {
                ProgressView("Fetching health data...")
            } else if let steps = steps {
                Text("\(steps) steps")
                    .font(.largeTitle)
                
                // Using Result type for additional processing
                if case .success(let analysis) = healthService.processStepsData(steps) {
                    Text(analysis)
                        .foregroundColor(.secondary)
                }
            } else if let error = errorMessage {
                Label(error, systemImage: "exclamationmark.triangle")
                    .foregroundColor(.red)
            }
            
            Button("Fetch Today's Steps") {
                Task {
                    await fetchSteps()
                }
            }
        }
        .padding()
    }
    
    @MainActor
    private func fetchSteps() async {
        isLoading = true
        errorMessage = nil
        
        do {
            // Clean async/await with error handling
            let fetchedSteps = try await healthService.fetchDailySteps(for: Date())
            self.steps = fetchedSteps
        } catch let error as HealthDataError {
            // Handle our custom errors with localized messages
            self.errorMessage = error.localizedDescription
        } catch {
            // Handle unexpected errors
            self.errorMessage = "An unexpected error occurred: \(error.localizedDescription)"
        }
        
        isLoading = false
    }
}
```

### 3. Production Example: Complete Error Handling System

```swift
import SwiftData
import OSLog

// Production-grade error system with logging and recovery strategies
enum AppError: LocalizedError, CustomDebugStringConvertible {
    case network(NetworkError)
    case database(DatabaseError)
    case validation(ValidationError)
    case business(BusinessError)
    
    enum NetworkError {
        case noConnection
        case timeout(TimeInterval)
        case httpError(statusCode: Int, response: String?)
        case decodingFailed(Error)
    }
    
    enum DatabaseError {
        case migrationFailed
        case queryFailed(String)
        case constraintViolation(String)
        case diskFull
    }
    
    enum ValidationError {
        case required(field: String)
        case invalid(field: String, reason: String)
        case businessRule(String)
    }
    
    enum BusinessError {
        case insufficientData
        case conflictingState(String)
        case limitExceeded(limit: Int, current: Int)
    }
    
    // User-facing error messages
    var errorDescription: String? {
        switch self {
        case .network(let netError):
            return netError.userMessage
        case .database:
            return "Unable to save your data. Please try again."
        case .validation(let valError):
            return valError.userMessage
        case .business(let bizError):
            return bizError.userMessage
        }
    }
    
    // Developer-facing debug information
    var debugDescription: String {
        switch self {
        case .network(let error):
            return "Network: \(error)"
        case .database(let error):
            return "Database: \(error)"
        case .validation(let error):
            return "Validation: \(error)"
        case .business(let error):
            return "Business: \(error)"
        }
    }
    
    // Recovery suggestions for the user
    var recoverySuggestion: String? {
        switch self {
        case .network(.noConnection):
            return "Check your internet connection and try again"
        case .network(.timeout):
            return "The request took too long. Please try again"
        case .database(.diskFull):
            return "Free up some space on your device"
        default:
            return nil
        }
    }
}

// Extension for nested error types
extension AppError.NetworkError {
    var userMessage: String {
        switch self {
        case .noConnection:
            return "No internet connection"
        case .timeout(let duration):
            return "Request timed out after \(Int(duration)) seconds"
        case .httpError(let code, _):
            return "Server error (\(code))"
        case .decodingFailed:
            return "Unable to process server response"
        }
    }
}

extension AppError.ValidationError {
    var userMessage: String {
        switch self {
        case .required(let field):
            return "\(field) is required"
        case .invalid(let field, let reason):
            return "\(field) is invalid: \(reason)"
        case .businessRule(let message):
            return message
        }
    }
}

extension AppError.BusinessError {
    var userMessage: String {
        switch self {
        case .insufficientData:
            return "Not enough data to perform this analysis"
        case .conflictingState(let description):
            return "Cannot perform this action: \(description)"
        case .limitExceeded(let limit, let current):
            return "Limit exceeded: \(current)/\(limit)"
        }
    }
}

// Protocol for recoverable errors
protocol RecoverableError: Error {
    var recoveryOptions: [RecoveryOption] { get }
}

struct RecoveryOption {
    let title: String
    let action: () async throws -> Void
}

// Advanced error handling coordinator
@MainActor
class ErrorHandlingCoordinator: ObservableObject {
    private let logger = Logger(subsystem: "com.yourapp.health", category: "ErrorHandling")
    
    @Published var currentError: AppError?
    @Published var isShowingError = false
    
    // Centralized error handling with logging
    func handle(_ error: Error, context: String? = nil) {
        let appError = self.normalize(error)
        
        // Log error with appropriate level
        switch appError {
        case .network(.noConnection):
            logger.info("Network unavailable\(context.map { " in \($0)" } ?? "")")
        case .database:
            logger.error("Database error: \(appError.debugDescription)")
        case .validation:
            logger.warning("Validation failed: \(appError.debugDescription)")
        case .business:
            logger.notice("Business rule violation: \(appError.debugDescription)")
        default:
            logger.error("Unhandled error: \(error)")
        }
        
        // Update UI
        self.currentError = appError
        self.isShowingError = true
        
        // Send to analytics (in production)
        // Analytics.track(.error(appError, context: context))
    }
    
    // Convert any error to AppError for consistent handling
    private func normalize(_ error: Error) -> AppError {
        if let appError = error as? AppError {
            return appError
        }
        
        // Handle system errors
        if (error as NSError).domain == NSURLErrorDomain {
            let nsError = error as NSError
            switch nsError.code {
            case NSURLErrorNotConnectedToInternet:
                return .network(.noConnection)
            case NSURLErrorTimedOut:
                return .network(.timeout(60))
            default:
                return .network(.httpError(statusCode: nsError.code, response: nil))
            }
        }
        
        // Default fallback
        return .network(.decodingFailed(error))
    }
}

// Repository pattern with comprehensive error handling
class MealRepository {
    private let modelContext: ModelContext
    private let logger = Logger(subsystem: "com.yourapp.health", category: "MealRepository")
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    // Demonstrating throwing with detailed error context
    func saveMeal(_ meal: Meal) throws {
        // Validation
        guard !meal.name.isEmpty else {
            throw AppError.validation(.required(field: "Meal name"))
        }
        
        guard meal.calories >= 0 else {
            throw AppError.validation(.invalid(
                field: "Calories",
                reason: "Must be non-negative"
            ))
        }
        
        // Business rules
        let todaysMeals = try fetchTodaysMeals()
        let totalCalories = todaysMeals.reduce(0) { $0 + $1.calories } + meal.calories
        
        guard totalCalories <= 10000 else {
            throw AppError.business(.limitExceeded(
                limit: 10000,
                current: totalCalories
            ))
        }
        
        // Database operation with error transformation
        do {
            modelContext.insert(meal)
            try modelContext.save()
        } catch {
            // Transform SwiftData errors to our domain errors
            logger.error("Failed to save meal: \(error)")
            
            if (error as NSError).code == 133021 {  // Constraint violation
                throw AppError.database(.constraintViolation("Duplicate meal entry"))
            } else {
                throw AppError.database(.queryFailed(error.localizedDescription))
            }
        }
    }
    
    // Using Result type for queries
    func searchMeals(query: String) -> Result<[Meal], AppError> {
        do {
            let predicate = #Predicate<Meal> { meal in
                meal.name.localizedStandardContains(query)
            }
            
            let descriptor = FetchDescriptor<Meal>(
                predicate: predicate,
                sortBy: [SortDescriptor(\.timestamp, order: .reverse)]
            )
            
            let meals = try modelContext.fetch(descriptor)
            return .success(meals)
            
        } catch {
            logger.error("Search failed: \(error)")
            return .failure(.database(.queryFailed("Search operation failed")))
        }
    }
    
    private func fetchTodaysMeals() throws -> [Meal] {
        // Implementation would fetch today's meals
        return []
    }
}

// SwiftUI integration with production error handling
struct MealEntryView: View {
    @Environment(\.modelContext) private var modelContext
    @StateObject private var errorCoordinator = ErrorHandlingCoordinator()
    
    @State private var mealName = ""
    @State private var calories = ""
    @State private var isSaving = false
    
    private var repository: MealRepository {
        MealRepository(modelContext: modelContext)
    }
    
    var body: some View {
        Form {
            TextField("Meal Name", text: $mealName)
            TextField("Calories", text: $calories)
                .keyboardType(.numberPad)
            
            Button("Save Meal") {
                Task {
                    await saveMeal()
                }
            }
            .disabled(isSaving || mealName.isEmpty)
        }
        .alert("Error", isPresented: $errorCoordinator.isShowingError) {
            Button("OK") {
                errorCoordinator.isShowingError = false
            }
            
            if let recovery = errorCoordinator.currentError?.recoverySuggestion {
                Button("Try Again") {
                    Task { await saveMeal() }
                }
            }
        } message: {
            if let error = errorCoordinator.currentError {
                Text(error.localizedDescription)
                if let suggestion = error.recoverySuggestion {
                    Text(suggestion)
                }
            }
        }
    }
    
    @MainActor
    private func saveMeal() async {
        isSaving = true
        defer { isSaving = false }
        
        let meal = Meal(
            name: mealName,
            calories: Int(calories) ?? 0,
            timestamp: Date()
        )
        
        do {
            try repository.saveMeal(meal)
            // Clear form on success
            mealName = ""
            calories = ""
        } catch {
            errorCoordinator.handle(error, context: "Saving meal")
        }
    }
}
```

## Common Pitfalls & Best Practices

Coming from Java, TypeScript, and Kotlin, you'll encounter several Swift-specific patterns that might trip you up initially.

The most common mistake is treating Swift errors like Java exceptions. In Java, you might catch a broad `Exception` type and handle everything generically. Swift encourages specific error handling—define clear error cases and handle each appropriately. Don't create a single "GenericError" type; instead, use enums with associated values to provide context.

Another pitfall is overusing `try!` and `try?`. Java developers often want to suppress errors during prototyping, similar to adding `throws Exception` to everything. Resist this urge. Use `try?` only when nil is a valid result (like optional chaining in TypeScript), and `try!` only for programmer errors that should never occur in production (like force unwrapping a resource you know exists).

Memory and performance implications differ significantly from Java exceptions. Swift's error handling has near-zero cost when no error occurs—it's essentially a special return value, not stack unwinding. This means you can use errors liberally without performance concerns, unlike Java where exception creation involves significant overhead.

Apple's ecosystem conventions favor explicit, recoverable errors over crashes. Your app should never crash due to network issues, missing data, or user input. Always provide fallback behavior and clear user communication. This differs from server-side development where failing fast might be preferred.

## Integration with SwiftUI & iOS Development

SwiftUI handles errors through state management and view modifiers, creating a reactive error-handling system that updates your UI automatically.

The key pattern is using `@State` or `@StateObject` properties to track errors, then displaying them with `.alert()` or custom error views. SwiftUI's declarative nature means you describe what should appear when an error exists, rather than imperatively showing error dialogs.

When working with HealthKit, you'll encounter both synchronous throws (for permission checks) and asynchronous errors (for data queries). SwiftData operations throw synchronously but should be wrapped in Tasks when called from views. The integration looks like this:

```swift
struct HealthDashboardView: View {
    @State private var healthKitError: Error?
    @State private var isLoadingHealth = false
    
    var body: some View {
        List {
            Section("Today's Activity") {
                if isLoadingHealth {
                    ProgressView()
                } else {
                    HealthMetricsView()
                }
            }
        }
        .task {
            do {
                isLoadingHealth = true
                try await loadHealthData()
            } catch {
                healthKitError = error
            }
            isLoadingHealth = false
        }
        .alert("Health Data Error", 
               isPresented: .constant(healthKitError != nil),
               presenting: healthKitError) { _ in
            Button("Retry") {
                Task { try? await loadHealthData() }
            }
            Button("Dismiss", role: .cancel) {
                healthKitError = nil
            }
        } message: { error in
            Text(error.localizedDescription)
        }
    }
    
    func loadHealthData() async throws {
        // Your HealthKit operations here
    }
}
```

## Production Considerations

Testing error handling requires deliberate strategies. Create test doubles that can simulate various error conditions. Unlike Java where you might use Mockito, Swift testing often uses protocols and test implementations:

```swift
protocol DataService {
    func fetchData() async throws -> [DataPoint]
}

struct TestDataService: DataService {
    var shouldFail = false
    var errorToThrow: Error?
    
    func fetchData() async throws -> [DataPoint] {
        if shouldFail {
            throw errorToThrow ?? TestError.deliberateFailure
        }
        return [DataPoint.sample]
    }
}
```

For debugging, Xcode's error breakpoints are invaluable. Set a Swift Error breakpoint to pause execution whenever any error is thrown. This helps identify unexpected error paths during development.

iOS 17+ introduces improved async error handling with typed throws (SE-0413), though this is still evolving. Structured concurrency makes error propagation more predictable—child tasks automatically propagate errors to parent tasks.

Performance-wise, prefer throwing functions over Result types for new code. The compiler can optimize throw/catch better than Result's enum cases. However, Result remains useful for storing errors or passing them between non-throwing contexts.

## Exercises for Mastery

### Exercise 1: TypeScript Promise to Swift Async/Await Migration
Convert this TypeScript error handling pattern to Swift:

```typescript
// TypeScript version
async function fetchUserProfile(id: string): Promise<UserProfile> {
    try {
        const response = await fetch(`/api/users/${id}`);
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }
        return await response.json();
    } catch (error) {
        console.error('Failed to fetch profile:', error);
        return DEFAULT_PROFILE;
    }
}
```

Your Swift version should:
- Define appropriate error types
- Use proper async/throws syntax  
- Handle both network and decoding errors
- Provide a similar fallback mechanism

### Exercise 2: Build a Validation Chain
Create a meal entry validator that:
- Validates multiple fields with specific rules
- Accumulates all validation errors (not just the first)
- Returns either a validated meal or all errors found
- Uses Swift's Result type for the return value

Requirements:
- Name: Required, 3-50 characters
- Calories: Required, 0-2000 range
- Protein: Optional, but if provided must be 0-200g
- Timestamp: Must not be in the future

### Exercise 3: Health App Integration Challenge
Build a `FastingWindowManager` that:
- Fetches meal times from SwiftData
- Calculates fasting windows between meals
- Throws appropriate errors for:
  - No meals recorded
  - Insufficient data (less than 2 meals)
  - Data inconsistencies (meals with identical timestamps)
- Provides recovery suggestions for each error type
- Integrates with SwiftUI using @MainActor
- Includes proper logging with OSLog

This manager should demonstrate production-ready error handling for your health app, including user-friendly messages and developer debugging information.

Remember, Swift's error handling is about making failure cases explicit and manageable. Unlike exceptions that interrupt flow, Swift errors are part of your program's normal operation. Embrace them as a tool for building robust, user-friendly iOS applications that gracefully handle the unexpected.

