# Swift Metaprogramming: Complete Guide for Production iOS Development

## Context & Background

Metaprogramming in Swift encompasses several powerful features that allow you to write code that inspects, modifies, or generates other code at compile-time or runtime. Think of it as Swift's answer to Java's reflection API, TypeScript's decorators, and Angular's metadata annotations combined, but with Apple's focus on performance and safety.

Coming from your background, here's how Swift's metaprogramming maps to what you know:

- **Mirror API** is like Java's Reflection API but safer and more limited (by design)
- **Swift Macros** are similar to TypeScript decorators or Java annotations with code generation capabilities, but they run at compile-time
- **String Interpolation Extensions** are like template literals in TypeScript but far more powerful
- **Dynamic Callable/Member** provides Python-like dynamic behavior (think of TypeScript's index signatures on steroids)
- **Compiler Directives** are like C preprocessor directives or Angular's environment configurations

In your health tracking app, you'll use these features for:
- Creating flexible data models that adapt to different HealthKit data types
- Building debug tools that inspect your app's state
- Generating boilerplate code for SwiftData models
- Creating type-safe configuration systems
- Building flexible API clients

## Core Understanding

### Mental Model

Think of Swift's metaprogramming as a toolkit with different levels of power and safety:

1. **Compile-time tools** (Macros, Compiler Directives): Generate or modify code before it runs, ensuring maximum performance
2. **Runtime introspection** (Mirror API): Inspect objects at runtime with limited modification capabilities
3. **Dynamic features** (Dynamic Callable/Member): Add flexibility for interop with dynamic systems
4. **Custom DSLs** (String Interpolation): Build domain-specific languages within Swift

The key difference from Java/TypeScript is that Swift prioritizes compile-time safety and performance over runtime flexibility. Where Java gives you full reflection powers, Swift gives you just enough to be useful while maintaining its performance guarantees.

## Part 1: Mirror API and Reflection

### Basic Syntax and Usage

```swift
// Mirror API provides runtime introspection of types and values
let mirror = Mirror(reflecting: someObject)

// Access properties
for child in mirror.children {
    print("\(child.label ?? "unknown"): \(child.value)")
}

// Check type information
print("Type: \(mirror.subjectType)")
print("Display style: \(mirror.displayStyle)")
```

### Practical Code Examples

#### 1. Basic Example: Simple Property Inspection

```swift
import Foundation

// Simple struct to inspect
struct UserProfile {
    let id: UUID
    let name: String
    let age: Int
    let isPremium: Bool
}

func inspectObject<T>(_ object: T) {
    let mirror = Mirror(reflecting: object)
    
    print("Inspecting \(mirror.subjectType):")
    print("Properties found: \(mirror.children.count)")
    
    // Iterate through all properties
    for (index, child) in mirror.children.enumerated() {
        // Note: label is optional because not all types have named properties
        let propertyName = child.label ?? "index_\(index)"
        let propertyValue = child.value
        let propertyType = type(of: propertyValue)
        
        print("  - \(propertyName): \(propertyValue) (Type: \(propertyType))")
    }
}

// Usage
let profile = UserProfile(
    id: UUID(),
    name: "John Doe",
    age: 30,
    isPremium: true
)

inspectObject(profile)

// WRONG WAY (Java/TypeScript thinking):
// Trying to modify properties through reflection
// This DOESN'T work in Swift - Mirror is read-only!
/*
for child in mirror.children {
    if child.label == "age" {
        child.value = 31  // ❌ This won't compile - Mirror is immutable
    }
}
*/

// SWIFT WAY: Use Mirror only for inspection, not modification
// For modification, use proper Swift patterns like protocols or builders
```

#### 2. Real-World Example: Health Data Inspector for Debugging

```swift
import SwiftUI
import HealthKit

// Health data model that matches your app's needs
struct HealthMeasurement: Codable {
    let id: UUID
    let type: String
    let value: Double
    let unit: String
    let timestamp: Date
    let metadata: [String: String]?
}

// Advanced reflection utility for your health app
class HealthDataInspector {
    
    /// Converts any health measurement to a debug-friendly dictionary
    /// Useful for logging, debugging, and crash reporting
    static func toDictionary<T>(_ object: T) -> [String: Any] {
        var dictionary: [String: Any] = [:]
        let mirror = Mirror(reflecting: object)
        
        // Handle different display styles appropriately
        switch mirror.displayStyle {
        case .struct, .class:
            for child in mirror.children {
                if let label = child.label {
                    // Recursively handle nested structures
                    dictionary[label] = unwrapValue(child.value)
                }
            }
        case .optional:
            // Handle optional unwrapping
            if let firstChild = mirror.children.first {
                return toDictionary(firstChild.value)
            }
            return [:]
        case .collection:
            // Convert collections to arrays
            var array: [Any] = []
            for child in mirror.children {
                array.append(unwrapValue(child.value))
            }
            return ["items": array]
        case .dictionary:
            // Handle dictionary types
            var dict: [String: Any] = [:]
            for child in mirror.children {
                if let (key, value) = child.value as? (String, Any) {
                    dict[key] = unwrapValue(value)
                }
            }
            return dict
        default:
            return ["value": object as Any]
        }
        
        return dictionary
    }
    
    private static func unwrapValue(_ value: Any) -> Any {
        // Handle common types that need special treatment
        if let date = value as? Date {
            return ISO8601DateFormatter().string(from: date)
        }
        if let uuid = value as? UUID {
            return uuid.uuidString
        }
        
        // Check if it's a primitive type
        let mirror = Mirror(reflecting: value)
        if mirror.children.count == 0 {
            return value
        }
        
        // Recursively process complex types
        return toDictionary(value)
    }
    
    /// Generates a debug report for health measurements
    static func generateDebugReport(for measurements: [HealthMeasurement]) -> String {
        var report = "Health Data Debug Report\n"
        report += "========================\n\n"
        
        for (index, measurement) in measurements.enumerated() {
            report += "Measurement #\(index + 1):\n"
            let dict = toDictionary(measurement)
            
            for (key, value) in dict {
                report += "  \(key): \(value)\n"
            }
            report += "\n"
        }
        
        return report
    }
}

// Usage in SwiftUI View
struct DebugView: View {
    @State private var measurements: [HealthMeasurement] = []
    @State private var debugOutput = ""
    
    var body: some View {
        VStack {
            Text("Debug Inspector")
                .font(.title)
            
            ScrollView {
                Text(debugOutput)
                    .font(.system(.caption, design: .monospaced))
                    .padding()
            }
            
            Button("Inspect Health Data") {
                // Create sample data
                let sample = HealthMeasurement(
                    id: UUID(),
                    type: "heartRate",
                    value: 72.0,
                    unit: "BPM",
                    timestamp: Date(),
                    metadata: ["source": "Apple Watch", "accuracy": "high"]
                )
                measurements = [sample]
                
                // Generate debug report using reflection
                debugOutput = HealthDataInspector.generateDebugReport(for: measurements)
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

#### 3. Production Example: Generic Data Validator with Reflection

```swift
import Foundation

// Protocol for validatable models
protocol Validatable {
    func validate() throws
}

// Custom validation errors
enum ValidationError: LocalizedError {
    case missingRequiredField(String)
    case invalidValue(field: String, reason: String)
    case validationFailed(reasons: [String])
    
    var errorDescription: String? {
        switch self {
        case .missingRequiredField(let field):
            return "Required field '\(field)' is missing"
        case .invalidValue(let field, let reason):
            return "Invalid value for '\(field)': \(reason)"
        case .validationFailed(let reasons):
            return "Validation failed: \(reasons.joined(separator: ", "))"
        }
    }
}

// Property wrapper for validation rules
@propertyWrapper
struct Validated<Value> {
    private var value: Value
    private let validator: (Value) -> Bool
    private let errorMessage: String
    
    var wrappedValue: Value {
        get { value }
        set {
            if !validator(newValue) {
                print("Warning: Invalid value for property - \(errorMessage)")
            }
            value = newValue
        }
    }
    
    init(wrappedValue: Value, 
         validator: @escaping (Value) -> Bool,
         errorMessage: String) {
        self.value = wrappedValue
        self.validator = validator
        self.errorMessage = errorMessage
    }
}

// Production-ready health profile with validation
struct ValidatedHealthProfile: Validatable {
    let id: UUID
    
    @Validated(
        validator: { !$0.isEmpty && $0.count <= 100 },
        errorMessage: "Name must be 1-100 characters"
    )
    var name: String
    
    @Validated(
        validator: { $0 >= 0 && $0 <= 150 },
        errorMessage: "Age must be between 0 and 150"
    )
    var age: Int
    
    @Validated(
        validator: { $0 >= 0 && $0 <= 500 },
        errorMessage: "Weight must be between 0 and 500 kg"
    )
    var weight: Double
    
    var healthConditions: [String]?
    
    init(id: UUID = UUID(), 
         name: String, 
         age: Int, 
         weight: Double,
         healthConditions: [String]? = nil) {
        self.id = id
        self.name = name
        self.age = age
        self.weight = weight
        self.healthConditions = healthConditions
    }
    
    func validate() throws {
        var errors: [String] = []
        
        // Use Mirror to inspect all properties
        let mirror = Mirror(reflecting: self)
        
        for child in mirror.children {
            guard let propertyName = child.label else { continue }
            
            // Check for nil values in required fields
            if propertyName != "healthConditions" {
                let valueMirror = Mirror(reflecting: child.value)
                if valueMirror.displayStyle == .optional {
                    if valueMirror.children.count == 0 {
                        errors.append("Required field '\(propertyName)' is nil")
                    }
                }
            }
            
            // Type-specific validation using reflection
            switch child.value {
            case let string as String where string.isEmpty && propertyName != "healthConditions":
                errors.append("\(propertyName) cannot be empty")
            case let number as Int where number < 0:
                errors.append("\(propertyName) cannot be negative")
            case let number as Double where number < 0:
                errors.append("\(propertyName) cannot be negative")
            default:
                break
            }
        }
        
        if !errors.isEmpty {
            throw ValidationError.validationFailed(reasons: errors)
        }
    }
}

// Generic validator using Mirror API
class GenericValidator {
    static func validateNonNilProperties<T>(_ object: T) -> [String] {
        var missingFields: [String] = []
        let mirror = Mirror(reflecting: object)
        
        for child in mirror.children {
            guard let propertyName = child.label else { continue }
            
            // Check if property is optional and nil
            let valueMirror = Mirror(reflecting: child.value)
            if valueMirror.displayStyle == .optional && valueMirror.children.count == 0 {
                missingFields.append(propertyName)
            }
        }
        
        return missingFields
    }
    
    /// Production-ready validation with detailed reporting
    static func performValidation<T: Validatable>(_ object: T) -> Result<T, ValidationError> {
        do {
            try object.validate()
            
            // Additional generic validation
            let missingFields = validateNonNilProperties(object)
            if !missingFields.isEmpty {
                print("Warning: Optional fields not set: \(missingFields)")
            }
            
            return .success(object)
        } catch let error as ValidationError {
            return .failure(error)
        } catch {
            return .failure(.validationFailed(reasons: [error.localizedDescription]))
        }
    }
}
```

## Part 2: Swift Macros (Swift 5.9+)

### Basic Syntax and Usage

```swift
// Macros are compile-time code generators
// They're defined in separate modules and used with @ or # syntax

// Attached macros (like decorators)
@Observable  // Makes a class observable
class MyModel { }

// Freestanding macros
#warning("This needs refactoring")  // Compile-time warning
```

### Practical Code Examples

#### 1. Basic Example: Using Built-in Macros

```swift
import SwiftUI
import Observation  // Required for @Observable macro

// Using the @Observable macro (Swift 5.9+)
// This is like Angular's change detection but at compile-time
@Observable
class UserSettings {
    var theme: String = "light"
    var fontSize: Double = 14.0
    var notificationsEnabled: Bool = true
    
    // The macro automatically generates property observation code
    // Similar to Angular's OnPush change detection strategy
}

// WRONG WAY (Java/TypeScript thinking):
// Trying to manually implement property observation
/*
class UserSettingsManual {
    private var _theme: String = "light" {
        didSet { notifyObservers() }  // ❌ Verbose and error-prone
    }
    var theme: String {
        get { _theme }
        set { _theme = newValue }
    }
    // ... repeat for every property
}
*/

// SWIFT WAY: Let the macro handle the boilerplate
// The @Observable macro generates all the observation code at compile-time

// Usage in SwiftUI
struct SettingsView: View {
    @State private var settings = UserSettings()
    
    var body: some View {
        Form {
            Picker("Theme", selection: $settings.theme) {
                Text("Light").tag("light")
                Text("Dark").tag("dark")
            }
            // View automatically updates when settings change
            // No manual subscription management needed!
        }
    }
}
```

#### 2. Real-World Example: Custom Macros for Health Data Models

```swift
import SwiftData
import Foundation

// Custom macro usage for your health app
// Note: Custom macro implementation requires a separate package

// Using SwiftData's @Model macro for persistence
@Model  // This macro generates all Core Data-like functionality
final class HealthRecord {
    // The @Model macro handles:
    // - Persistence logic
    // - Change tracking
    // - Relationship management
    // - Query optimization
    
    var id: UUID
    var recordType: String
    var value: Double
    var unit: String
    var timestamp: Date
    
    // Relationships are automatically managed
    @Relationship(deleteRule: .cascade)
    var measurements: [Measurement]?
    
    init(recordType: String, value: Double, unit: String) {
        self.id = UUID()
        self.recordType = recordType
        self.value = value
        self.unit = unit
        self.timestamp = Date()
    }
}

// Using expression macros for type-safe keys
extension HealthRecord {
    // Hypothetical custom macro for generating typed keypaths
    // #keyPath macro would generate compile-time checked paths
    static let allKeypaths = [
        #keyPath(recordType),
        #keyPath(value),
        #keyPath(timestamp)
    ]
    
    // URL macro for compile-time URL validation
    static let apiEndpoint = #URL("https://api.health.example.com/v1/records")
    
    // Compile-time assertions
    static let maxRecords = #assert(1000 > 0, "Max records must be positive")
}

// Production-ready macro usage patterns
@MainActor  // Ensures UI updates happen on main thread
@Observable
final class HealthDataViewModel {
    private(set) var records: [HealthRecord] = []
    private(set) var isLoading = false
    private(set) var error: Error?
    
    // Compile-time environment check
    #if DEBUG
    private let apiURL = "https://staging.api.health.example.com"
    #else
    private let apiURL = "https://api.health.example.com"
    #endif
    
    @MainActor
    func loadRecords() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            // The async/await pattern works seamlessly with @Observable
            records = try await fetchHealthRecords()
        } catch {
            self.error = error
            
            // Compile-time debug helpers
            #if DEBUG
            print("Error loading records: \(error)")
            #endif
        }
    }
    
    private func fetchHealthRecords() async throws -> [HealthRecord] {
        // Implementation here
        return []
    }
}
```

#### 3. Production Example: Advanced Macro Patterns

```swift
import SwiftUI
import SwiftData

// Complex production example combining multiple macros
@Model
final class PatientHealthProfile {
    // Unique identifier
    @Attribute(.unique) 
    var patientId: String
    
    // Indexed for fast queries
    @Attribute(.index)
    var lastUpdated: Date
    
    // Encrypted storage (hypothetical custom macro)
    // @Encrypted 
    var sensitiveHealthData: Data?
    
    // Relationships with cascade rules
    @Relationship(deleteRule: .cascade, inverse: \HealthMetric.profile)
    var metrics: [HealthMetric]
    
    // Transient property (not persisted)
    @Transient
    var calculatedRiskScore: Double = 0.0
    
    init(patientId: String) {
        self.patientId = patientId
        self.lastUpdated = Date()
        self.metrics = []
    }
}

@Model
final class HealthMetric {
    var id: UUID
    var type: MetricType
    var value: Double
    var timestamp: Date
    
    // Back reference to profile
    var profile: PatientHealthProfile?
    
    enum MetricType: String, Codable, CaseIterable {
        case heartRate = "heart_rate"
        case bloodPressure = "blood_pressure"
        case glucose = "glucose"
        case weight = "weight"
        
        // Compile-time generation of display names
        var displayName: String {
            #switch(self) {  // Hypothetical macro for exhaustive switch
                case .heartRate: "Heart Rate"
                case .bloodPressure: "Blood Pressure"
                case .glucose: "Glucose Level"
                case .weight: "Body Weight"
            }
        }
    }
    
    init(type: MetricType, value: Double) {
        self.id = UUID()
        self.type = type
        self.value = value
        self.timestamp = Date()
    }
}

// ViewModifier using macros for reusable UI components
@ViewModifier  // Hypothetical macro for generating ViewModifier boilerplate
struct HealthCardStyle: ViewModifier {
    @Environment(\.colorScheme) var colorScheme
    
    func body(content: Content) -> some View {
        content
            .padding()
            .background(
                RoundedRectangle(cornerRadius: 12)
                    .fill(Color(uiColor: .systemBackground))
                    .shadow(
                        color: colorScheme == .dark ? .clear : .black.opacity(0.1),
                        radius: 8,
                        x: 0,
                        y: 2
                    )
            )
            .overlay(
                RoundedRectangle(cornerRadius: 12)
                    .stroke(Color(uiColor: .separator), lineWidth: 0.5)
            )
    }
}

// Production view using multiple macro-enhanced models
struct HealthDashboardView: View {
    @Query(sort: \PatientHealthProfile.lastUpdated, order: .reverse)
    private var profiles: [PatientHealthProfile]
    
    @Environment(\.modelContext) private var modelContext
    @State private var selectedProfile: PatientHealthProfile?
    
    var body: some View {
        NavigationStack {
            List(profiles) { profile in
                HealthProfileCard(profile: profile)
                    .modifier(HealthCardStyle())
                    .onTapGesture {
                        selectedProfile = profile
                    }
            }
            .navigationTitle("Health Profiles")
            .toolbar {
                #if DEBUG
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Add Test Data") {
                        addTestData()
                    }
                }
                #endif
            }
        }
    }
    
    #if DEBUG
    private func addTestData() {
        let profile = PatientHealthProfile(patientId: "TEST_\(UUID().uuidString)")
        
        // Add sample metrics
        let heartRate = HealthMetric(type: .heartRate, value: 72)
        let weight = HealthMetric(type: .weight, value: 75.5)
        
        profile.metrics.append(heartRate)
        profile.metrics.append(weight)
        
        modelContext.insert(profile)
    }
    #endif
}

struct HealthProfileCard: View {
    let profile: PatientHealthProfile
    
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("Patient: \(profile.patientId)")
                .font(.headline)
            
            Text("Metrics: \(profile.metrics.count)")
                .font(.subheadline)
                .foregroundColor(.secondary)
            
            Text("Updated: \(profile.lastUpdated.formatted())")
                .font(.caption)
                .foregroundColor(.secondary)
        }
        .padding(.vertical, 4)
    }
}
```

## Part 3: String Interpolation Extensions

### Basic Syntax and Usage

```swift
// Custom string interpolation allows building DSLs
// You extend String.StringInterpolation to add custom behavior

extension String.StringInterpolation {
    mutating func appendInterpolation(_ value: SomeType, format: SomeFormat) {
        // Custom formatting logic
    }
}
```

### Practical Code Examples

#### 1. Basic Example: Custom Formatting

```swift
import Foundation

// Custom string interpolation for health measurements
extension String.StringInterpolation {
    // Format weight with units
    mutating func appendInterpolation(weight value: Double, unit: String = "kg") {
        let formatted = String(format: "%.1f", value)
        appendLiteral("\(formatted) \(unit)")
    }
    
    // Format heart rate
    mutating func appendInterpolation(heartRate value: Int) {
        appendLiteral("\(value) BPM")
    }
    
    // Format blood pressure
    mutating func appendInterpolation(bloodPressure systolic: Int, _ diastolic: Int) {
        appendLiteral("\(systolic)/\(diastolic) mmHg")
    }
}

// Usage
let weight = 75.5
let heartRate = 72
let systolic = 120
let diastolic = 80

// Clean, readable health data formatting
let report = """
    Health Report:
    Weight: \(weight: weight)
    Heart Rate: \(heartRate: heartRate)
    Blood Pressure: \(bloodPressure: systolic, diastolic)
    """

print(report)

// WRONG WAY (Java/TypeScript thinking):
// Using string concatenation or manual formatting
/*
let badReport = "Weight: " + String(format: "%.1f", weight) + " kg\n" +  // ❌ Verbose
                "Heart Rate: " + String(heartRate) + " BPM"
*/

// SWIFT WAY: Use string interpolation extensions for domain-specific formatting
```

#### 2. Real-World Example: Health Report Builder

```swift
import Foundation

// Comprehensive health data formatting system
struct HealthValue {
    let value: Double
    let unit: String
    let timestamp: Date
    let quality: DataQuality
    
    enum DataQuality {
        case high, medium, low, estimated
    }
}

// Advanced string interpolation for health reports
extension String.StringInterpolation {
    // Format health values with quality indicators
    mutating func appendInterpolation(health value: HealthValue) {
        let formatter = NumberFormatter()
        formatter.maximumFractionDigits = 2
        
        let formattedValue = formatter.string(from: NSNumber(value: value.value)) ?? "N/A"
        let qualityIndicator = value.quality == .estimated ? "~" : ""
        
        appendLiteral("\(qualityIndicator)\(formattedValue) \(value.unit)")
    }
    
    // Format date ranges for health data
    mutating func appendInterpolation(dateRange start: Date, to end: Date) {
        let formatter = DateFormatter()
        formatter.dateStyle = .medium
        formatter.timeStyle = .none
        
        let startStr = formatter.string(from: start)
        let endStr = formatter.string(from: end)
        
        appendLiteral("\(startStr) - \(endStr)")
    }
    
    // Format percentile rankings
    mutating func appendInterpolation(percentile value: Double, category: String) {
        let percentileInt = Int(value * 100)
        let descriptor: String
        
        switch percentileInt {
        case 90...100:
            descriptor = "Excellent"
        case 70..<90:
            descriptor = "Good"
        case 50..<70:
            descriptor = "Average"
        case 30..<50:
            descriptor = "Below Average"
        default:
            descriptor = "Needs Improvement"
        }
        
        appendLiteral("\(percentileInt)th percentile (\(descriptor)) for \(category)")
    }
    
    // Format health trends
    mutating func appendInterpolation(trend values: [Double], over period: String) {
        guard values.count >= 2 else {
            appendLiteral("Insufficient data")
            return
        }
        
        let first = values.first!
        let last = values.last!
        let change = ((last - first) / first) * 100
        
        let arrow = change > 0 ? "↑" : change < 0 ? "↓" : "→"
        let changeStr = String(format: "%.1f%%", abs(change))
        
        appendLiteral("\(arrow) \(changeStr) over \(period)")
    }
}

// Production-ready health report generator
class HealthReportGenerator {
    static func generateReport(for data: [HealthValue], period: (Date, Date)) -> String {
        let averageValue = data.map { $0.value }.reduce(0, +) / Double(data.count)
        let trend = data.map { $0.value }
        
        let report = """
            === Health Report ===
            Period: \(dateRange: period.0, to: period.1)
            
            Current Status:
            Latest Reading: \(health: data.last!)
            Average: \(health: HealthValue(value: averageValue, unit: data.first!.unit, timestamp: Date(), quality: .high))
            
            Performance:
            You're at the \(percentile: 0.75, category: "heart health")
            
            Trend Analysis:
            Your metrics show: \(trend: trend, over: "last week")
            
            ---
            Generated: \(Date().formatted())
            """
        
        return report
    }
}

// Usage example
let healthData = [
    HealthValue(value: 72, unit: "BPM", timestamp: Date().addingTimeInterval(-86400 * 7), quality: .high),
    HealthValue(value: 74, unit: "BPM", timestamp: Date().addingTimeInterval(-86400 * 3), quality: .high),
    HealthValue(value: 70, unit: "BPM", timestamp: Date(), quality: .high)
]

let report = HealthReportGenerator.generateReport(
    for: healthData,
    period: (Date().addingTimeInterval(-86400 * 7), Date())
)
print(report)
```

#### 3. Production Example: Type-Safe SQL Query Builder

```swift
import Foundation

// Type-safe SQL query builder using string interpolation
// This is similar to Laravel's Query Builder but compile-time safe

struct SQLColumn {
    let table: String?
    let name: String
    
    var qualified: String {
        if let table = table {
            return "\(table).\(name)"
        }
        return name
    }
}

struct SQLValue {
    let value: Any
    let needsQuotes: Bool
    
    static func string(_ value: String) -> SQLValue {
        SQLValue(value: value, needsQuotes: true)
    }
    
    static func number(_ value: Double) -> SQLValue {
        SQLValue(value: value, needsQuotes: false)
    }
    
    static func date(_ value: Date) -> SQLValue {
        let formatter = ISO8601DateFormatter()
        return SQLValue(value: formatter.string(from: value), needsQuotes: true)
    }
}

extension String.StringInterpolation {
    // SELECT clause
    mutating func appendInterpolation(SELECT columns: [SQLColumn]) {
        let columnList = columns.map { $0.qualified }.joined(separator: ", ")
        appendLiteral("SELECT \(columnList)")
    }
    
    // FROM clause
    mutating func appendInterpolation(FROM table: String) {
        appendLiteral("FROM \(table)")
    }
    
    // WHERE clause with type safety
    mutating func appendInterpolation(WHERE column: SQLColumn, _ op: String, _ value: SQLValue) {
        let valueStr = value.needsQuotes ? "'\(value.value)'" : "\(value.value)"
        appendLiteral("WHERE \(column.qualified) \(op) \(valueStr)")
    }
    
    // JOIN clause
    mutating func appendInterpolation(JOIN table: String, ON leftColumn: SQLColumn, equals rightColumn: SQLColumn) {
        appendLiteral("JOIN \(table) ON \(leftColumn.qualified) = \(rightColumn.qualified)")
    }
    
    // Safe parameterized values
    mutating func appendInterpolation(param value: Any) {
        // In production, this would create parameterized queries
        // For demonstration, we're escaping values
        if let string = value as? String {
            appendLiteral("'\(string.replacingOccurrences(of: "'", with: "''"))'")
        } else {
            appendLiteral("\(value)")
        }
    }
}

// Production query builder for health data
class HealthDataQueryBuilder {
    static func buildHealthMetricsQuery(
        patientId: String,
        metricTypes: [String],
        startDate: Date,
        endDate: Date
    ) -> String {
        
        let columns = [
            SQLColumn(table: "metrics", name: "id"),
            SQLColumn(table: "metrics", name: "type"),
            SQLColumn(table: "metrics", name: "value"),
            SQLColumn(table: "metrics", name: "timestamp"),
            SQLColumn(table: "patients", name: "name")
        ]
        
        // Type-safe query building
        let query = """
            \(SELECT: columns)
            \(FROM: "health_metrics AS metrics")
            \(JOIN: "patients", ON: SQLColumn(table: "metrics", name: "patient_id"), equals: SQLColumn(table: "patients", name: "id"))
            \(WHERE: SQLColumn(table: "metrics", name: "patient_id"), "=", SQLValue.string(patientId))
            AND metrics.type IN (\(metricTypes.map { "'\($0)'" }.joined(separator: ", ")))
            AND metrics.timestamp BETWEEN \(param: startDate) AND \(param: endDate)
            ORDER BY metrics.timestamp DESC
            LIMIT 100
            """
        
        return query
    }
}

// Usage
let query = HealthDataQueryBuilder.buildHealthMetricsQuery(
    patientId: "user123",
    metricTypes: ["heart_rate", "blood_pressure"],
    startDate: Date().addingTimeInterval(-86400 * 30),
    endDate: Date()
)
print(query)
```

## Part 4: Dynamic Callable and Dynamic Member

### Basic Syntax and Usage

```swift
// Dynamic Callable - makes types callable like functions
@dynamicCallable
struct DynamicType {
    func dynamicallyCall(withArguments args: [Int]) -> Int {
        return args.reduce(0, +)
    }
}

// Dynamic Member - access properties dynamically
@dynamicMemberLookup
struct DynamicObject {
    subscript(dynamicMember member: String) -> String {
        return "Value for \(member)"
    }
}
```

### Practical Code Examples

#### 1. Basic Example: Dynamic API Client

```swift
import Foundation

// Dynamic API client similar to JavaScript/TypeScript dynamic objects
@dynamicMemberLookup
struct DynamicHealthAPI {
    private let baseURL: String
    private var path: [String] = []
    
    init(baseURL: String = "https://api.health.example.com") {
        self.baseURL = baseURL
    }
    
    // Dynamic member lookup for building API paths
    subscript(dynamicMember member: String) -> DynamicHealthAPI {
        var newAPI = self
        newAPI.path.append(member)
        return newAPI
    }
    
    // Build the final URL
    var url: String {
        return baseURL + "/" + path.joined(separator: "/")
    }
}

// Usage - feels like JavaScript/TypeScript
let api = DynamicHealthAPI()
let patientsEndpoint = api.v1.patients.records.url
// Results in: "https://api.health.example.com/v1/patients/records"

// WRONG WAY (Static typing approach):
/*
struct StaticAPI {
    let v1 = V1API()  // ❌ Need to define every possible path
    struct V1API {
        let patients = PatientsAPI()
        struct PatientsAPI {
            let records = RecordsAPI()
            // ... endless nesting
        }
    }
}
*/

// SWIFT WAY: Use dynamic member lookup for flexible API paths
```

#### 2. Real-World Example: Flexible Health Data Container

```swift
import Foundation

// Dynamic health data container for varying data schemas
@dynamicMemberLookup
@dynamicCallable
class HealthDataContainer {
    private var storage: [String: Any] = [:]
    private var computedProperties: [String: () -> Any] = [:]
    
    // Dynamic member for getting/setting values
    subscript(dynamicMember member: String) -> Any? {
        get {
            // Check computed properties first
            if let computed = computedProperties[member] {
                return computed()
            }
            return storage[member]
        }
        set {
            storage[member] = newValue
        }
    }
    
    // Type-safe subscript variants
    subscript<T>(dynamicMember member: String) -> T? {
        return storage[member] as? T
    }
    
    // Dynamic callable for method-like behavior
    func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Any>) -> HealthDataContainer {
        for (key, value) in args {
            storage[key] = value
        }
        return self
    }
    
    // Register computed properties
    func addComputedProperty(_ name: String, calculation: @escaping () -> Any) {
        computedProperties[name] = calculation
    }
    
    // Convert to dictionary for serialization
    func toDictionary() -> [String: Any] {
        return storage
    }
}

// Extension for health-specific functionality
extension HealthDataContainer {
    // BMI calculation as computed property
    func setupBMICalculation() {
        addComputedProperty("bmi") { [weak self] in
            guard let weight: Double = self?["weight"] as? Double,
                  let height: Double = self?["height"] as? Double else {
                return 0.0
            }
            return weight / (height * height)
        }
    }
    
    // Risk score calculation
    func setupRiskScore() {
        addComputedProperty("riskScore") { [weak self] in
            var score = 0.0
            
            if let age: Int = self?["age"] as? Int {
                score += Double(age) * 0.1
            }
            
            if let bmi = self?["bmi"] as? Double {
                if bmi > 30 { score += 20 }
                else if bmi > 25 { score += 10 }
            }
            
            if let smoker: Bool = self?["smoker"] as? Bool, smoker {
                score += 30
            }
            
            return min(score, 100)
        }
    }
}

// Usage in production
let patient = HealthDataContainer()

// Dynamic property setting
patient.name = "John Doe"
patient.age = 45
patient.weight = 85.0  // kg
patient.height = 1.75  // meters
patient.smoker = false

// Method-like calling
patient(bloodPressure: "120/80", heartRate: 72, lastCheckup: Date())

// Setup computed properties
patient.setupBMICalculation()
patient.setupRiskScore()

// Access computed properties dynamically
if let bmi = patient.bmi as? Double {
    print("BMI: \(bmi)")
}

if let risk = patient.riskScore as? Double {
    print("Risk Score: \(risk)")
}

// Serialize for API
let jsonData = try? JSONSerialization.data(withJSONObject: patient.toDictionary())
```

#### 3. Production Example: Dynamic Configuration System

```swift
import Foundation
import SwiftUI

// Production-ready dynamic configuration system
@dynamicMemberLookup
final class DynamicConfiguration: ObservableObject {
    @Published private var config: [String: Any] = [:]
    private let defaults: [String: Any]
    private let validators: [String: (Any) -> Bool] = [:]
    
    init(defaults: [String: Any] = [:]) {
        self.defaults = defaults
        loadConfiguration()
    }
    
    // Dynamic member with type inference
    subscript<T>(dynamicMember member: String) -> T? {
        get {
            return (config[member] ?? defaults[member]) as? T
        }
        set {
            // Validate before setting
            if let validator = validators[member], let value = newValue {
                guard validator(value) else {
                    print("Validation failed for \(member)")
                    return
                }
            }
            
            config[member] = newValue
            saveConfiguration()
            objectWillChange.send()
        }
    }
    
    // Register validators
    func addValidator(for key: String, validator: @escaping (Any) -> Bool) {
        validators[key] = validator
    }
    
    // Persistence
    private func loadConfiguration() {
        if let data = UserDefaults.standard.data(forKey: "app_config"),
           let decoded = try? JSONSerialization.jsonObject(with: data) as? [String: Any] {
            config = decoded
        }
    }
    
    private func saveConfiguration() {
        if let data = try? JSONSerialization.data(withJSONObject: config) {
            UserDefaults.standard.set(data, forKey: "app_config")
        }
    }
}

// Health app specific configuration
extension DynamicConfiguration {
    static func healthAppConfig() -> DynamicConfiguration {
        let config = DynamicConfiguration(defaults: [
            "theme": "system",
            "dataRetentionDays": 365,
            "syncInterval": 3600,
            "enableNotifications": true,
            "targetSteps": 10000,
            "waterGoal": 2.0,
            "apiEndpoint": "https://api.health.example.com"
        ])
        
        // Add validators
        config.addValidator(for: "dataRetentionDays") { value in
            guard let days = value as? Int else { return false }
            return days >= 30 && days <= 730
        }
        
        config.addValidator(for: "syncInterval") { value in
            guard let interval = value as? Int else { return false }
            return interval >= 300 && interval <= 86400
        }
        
        config.addValidator(for: "targetSteps") { value in
            guard let steps = value as? Int else { return false }
            return steps >= 1000 && steps <= 50000
        }
        
        return config
    }
}

// Dynamic callable for batch updates
@dynamicCallable
extension DynamicConfiguration {
    func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Any>) {
        for (key, value) in args {
            config[key] = value
        }
        saveConfiguration()
        objectWillChange.send()
    }
}

// SwiftUI integration
struct DynamicConfigView: View {
    @StateObject private var config = DynamicConfiguration.healthAppConfig()
    
    var body: some View {
        Form {
            Section("Display") {
                Picker("Theme", selection: Binding(
                    get: { config.theme ?? "system" },
                    set: { config.theme = $0 }
                )) {
                    Text("System").tag("system")
                    Text("Light").tag("light")
                    Text("Dark").tag("dark")
                }
            }
            
            Section("Goals") {
                Stepper(
                    "Daily Steps: \(config.targetSteps ?? 10000)",
                    value: Binding(
                        get: { config.targetSteps ?? 10000 },
                        set: { config.targetSteps = $0 }
                    ),
                    in: 1000...50000,
                    step: 1000
                )
                
                HStack {
                    Text("Water Goal")
                    Spacer()
                    Text("\(config.waterGoal ?? 2.0, specifier: "%.1f") L")
                }
            }
            
            Section("Sync") {
                Toggle("Enable Notifications", isOn: Binding(
                    get: { config.enableNotifications ?? true },
                    set: { config.enableNotifications = $0 }
                ))
                
                if let interval: Int = config.syncInterval {
                    Text("Sync every \(interval / 60) minutes")
                }
            }
            
            Button("Reset to Defaults") {
                // Batch update using dynamic callable
                config(
                    theme: "system",
                    targetSteps: 10000,
                    waterGoal: 2.0,
                    enableNotifications: true
                )
            }
        }
        .navigationTitle("Settings")
    }
}
```

## Part 5: Compiler Directives

### Basic Syntax and Usage

```swift
// Conditional compilation
#if DEBUG
    print("Debug mode")
#endif

// OS-specific code
#if os(iOS)
    import UIKit
#elseif os(macOS)
    import AppKit
#endif

// API availability
if #available(iOS 17.0, *) {
    // Use iOS 17+ features
}

// Compile-time warnings/errors
#warning("TODO: Implement this feature")
#error("This code requires Swift 5.9+")
```

### Practical Code Examples

#### 1. Basic Example: Environment-Based Configuration

```swift
import Foundation

// Configuration based on build environment
struct AppConfiguration {
    // API endpoints change based on build configuration
    static let apiBaseURL: String = {
        #if DEBUG
        return "https://staging-api.health.example.com"
        #elseif BETA
        return "https://beta-api.health.example.com"
        #else
        return "https://api.health.example.com"
        #endif
    }()
    
    // Feature flags
    static let features: Set<Feature> = {
        var features: Set<Feature> = [.basicHealth, .charts]
        
        #if DEBUG
        features.insert(.debugMenu)
        features.insert(.mockData)
        #endif
        
        #if PREMIUM
        features.insert(.advancedAnalytics)
        features.insert(.exportData)
        #endif
        
        return features
    }()
    
    enum Feature {
        case basicHealth
        case charts
        case debugMenu
        case mockData
        case advancedAnalytics
        case exportData
    }
    
    // Logging configuration
    static let loggingLevel: LogLevel = {
        #if DEBUG
        return .verbose
        #elseif BETA
        return .info
        #else
        return .error
        #endif
    }()
    
    enum LogLevel: Int {
        case verbose = 0
        case debug = 1
        case info = 2
        case warning = 3
        case error = 4
    }
}

// WRONG WAY (Runtime configuration):
/*
struct BadConfiguration {
    static let apiURL = ProcessInfo.processInfo.environment["API_URL"] ?? ""  // ❌ Runtime overhead
    static let isDebug = ProcessInfo.processInfo.environment["DEBUG"] == "true"  // ❌ Can be modified
}
*/

// SWIFT WAY: Use compiler directives for compile-time configuration
```

#### 2. Real-World Example: Cross-Platform Health Data Manager

```swift
import SwiftUI
#if os(iOS)
import HealthKit
import UIKit
#elseif os(watchOS)
import HealthKit
import WatchKit
#endif

// Cross-platform health data manager
class HealthDataManager: ObservableObject {
    #if os(iOS) || os(watchOS)
    private let healthStore = HKHealthStore()
    #endif
    
    @Published var latestHeartRate: Double?
    @Published var stepCount: Int = 0
    
    init() {
        #if DEBUG
        print("HealthDataManager initialized")
        setupDebugData()
        #endif
        
        requestAuthorization()
    }
    
    func requestAuthorization() {
        #if os(iOS) || os(watchOS)
        guard HKHealthStore.isHealthDataAvailable() else {
            #if DEBUG
            print("HealthKit not available on this device")
            #endif
            return
        }
        
        let typesToRead: Set<HKObjectType> = [
            HKObjectType.quantityType(forIdentifier: .heartRate)!,
            HKObjectType.quantityType(forIdentifier: .stepCount)!
        ]
        
        healthStore.requestAuthorization(toShare: nil, read: typesToRead) { success, error in
            #if DEBUG
            if let error = error {
                print("Authorization error: \(error)")
            } else {
                print("Authorization success: \(success)")
            }
            #endif
            
            if success {
                self.startMonitoring()
            }
        }
        #else
        // macOS or other platforms
        #warning("HealthKit not available on this platform")
        setupMockData()
        #endif
    }
    
    private func startMonitoring() {
        #if os(iOS) || os(watchOS)
        // Real HealthKit monitoring
        observeHeartRate()
        observeSteps()
        #endif
    }
    
    #if os(iOS) || os(watchOS)
    private func observeHeartRate() {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        
        let query = HKObserverQuery(sampleType: heartRateType, predicate: nil) { [weak self] _, _, error in
            if error == nil {
                self?.fetchLatestHeartRate()
            }
        }
        
        healthStore.execute(query)
    }
    
    private func fetchLatestHeartRate() {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        let sortDescriptor = NSSortDescriptor(key: HKSampleSortIdentifierStartDate, ascending: false)
        
        let query = HKSampleQuery(
            sampleType: heartRateType,
            predicate: nil,
            limit: 1,
            sortDescriptors: [sortDescriptor]
        ) { [weak self] _, samples, error in
            guard let sample = samples?.first as? HKQuantitySample else { return }
            
            let heartRate = sample.quantity.doubleValue(for: HKUnit.count().unitDivided(by: .minute()))
            
            DispatchQueue.main.async {
                self?.latestHeartRate = heartRate
                
                #if DEBUG
                print("Latest heart rate: \(heartRate) BPM")
                #endif
            }
        }
        
        healthStore.execute(query)
    }
    
    private func observeSteps() {
        // Similar implementation for steps
    }
    #endif
    
    #if DEBUG
    private func setupDebugData() {
        // Generate mock data for debugging
        Timer.scheduledTimer(withTimeInterval: 5.0, repeats: true) { _ in
            self.latestHeartRate = Double.random(in: 60...100)
            self.stepCount = Int.random(in: 5000...15000)
        }
    }
    #endif
    
    private func setupMockData() {
        // Fallback for platforms without HealthKit
        latestHeartRate = 72
        stepCount = 10000
    }
}

// Platform-specific UI
struct HealthDataView: View {
    @StateObject private var healthManager = HealthDataManager()
    
    var body: some View {
        VStack(spacing: 20) {
            #if os(iOS)
            // iOS-specific UI
            healthDataCard
                .navigationBarTitle("Health Data", displayMode: .large)
            #elseif os(watchOS)
            // watchOS-specific UI
            healthDataCard
                .navigationTitle("Health")
            #else
            // macOS or other platforms
            healthDataCard
                .frame(minWidth: 300, minHeight: 200)
            #endif
            
            #if DEBUG
            debugSection
            #endif
        }
        .padding()
    }
    
    private var healthDataCard: some View {
        VStack(alignment: .leading, spacing: 12) {
            if let heartRate = healthManager.latestHeartRate {
                Label("\(Int(heartRate)) BPM", systemImage: "heart.fill")
                    .font(.title2)
                    .foregroundColor(.red)
            }
            
            Label("\(healthManager.stepCount) steps", systemImage: "figure.walk")
                .font(.title2)
                .foregroundColor(.blue)
        }
        .padding()
        .background(Color(uiColor: .secondarySystemBackground))
        .cornerRadius(12)
    }
    
    #if DEBUG
    private var debugSection: some View {
        VStack(alignment: .leading) {
            Text("Debug Info")
                .font(.caption)
                .foregroundColor(.secondary)
            
            Text("Platform: \(currentPlatform)")
                .font(.caption2)
            
            Text("API: \(AppConfiguration.apiBaseURL)")
                .font(.caption2)
            
            Button("Generate Random Data") {
                healthManager.latestHeartRate = Double.random(in: 60...100)
                healthManager.stepCount = Int.random(in: 5000...15000)
            }
            .buttonStyle(.bordered)
        }
        .padding()
        .background(Color.yellow.opacity(0.1))
        .cornerRadius(8)
    }
    
    private var currentPlatform: String {
        #if os(iOS)
        return "iOS"
        #elseif os(watchOS)
        return "watchOS"
        #elseif os(macOS)
        return "macOS"
        #else
        return "Unknown"
        #endif
    }
    #endif
}
```

#### 3. Production Example: Advanced Build Configuration

```swift
import SwiftUI
import os.log

// Production-ready logging system with compiler directives
struct HealthLogger {
    private static let subsystem = "com.example.healthapp"
    
    enum Category: String {
        case healthKit = "HealthKit"
        case networking = "Networking"
        case database = "Database"
        case ui = "UI"
        case sync = "Sync"
    }
    
    private static func logger(for category: Category) -> Logger {
        Logger(subsystem: subsystem, category: category.rawValue)
    }
    
    static func log(_ message: String, category: Category = .ui, level: OSLogType = .info) {
        #if DEBUG
        // Verbose logging in debug
        let logger = logger(for: category)
        logger.log(level: level, "\(message)")
        
        // Also print to console for easier debugging
        let timestamp = Date().formatted(date: .omitted, time: .standard)
        print("[\(timestamp)] [\(category.rawValue)] \(message)")
        
        #elseif BETA
        // Moderate logging in beta
        if level == .error || level == .fault {
            let logger = logger(for: category)
            logger.log(level: level, "\(message)")
        }
        
        #else
        // Minimal logging in production
        if level == .fault {
            let logger = logger(for: category)
            logger.log(level: level, "\(message)")
        }
        #endif
    }
    
    static func measure<T>(_ label: String, operation: () throws -> T) rethrows -> T {
        #if DEBUG
        let startTime = CFAbsoluteTimeGetCurrent()
        defer {
            let timeElapsed = CFAbsoluteTimeGetCurrent() - startTime
            log("⏱ \(label) took \(String(format: "%.3f", timeElapsed))s", category: .ui, level: .debug)
        }
        #endif
        
        return try operation()
    }
}

// Advanced feature flags with compiler directives
struct FeatureFlags {
    // Compile-time feature detection
    static var isHealthKitAvailable: Bool {
        #if os(iOS) || os(watchOS)
        return true
        #else
        return false
        #endif
    }
    
    static var isIPadOptimized: Bool {
        #if os(iOS)
        return UIDevice.current.userInterfaceIdiom == .pad
        #else
        return false
        #endif
    }
    
    static var supportsWidgets: Bool {
        #if os(iOS)
        if #available(iOS 14.0, *) {
            return true
        }
        #endif
        return false
    }
    
    static var supportsLiveActivities: Bool {
        #if os(iOS)
        if #available(iOS 16.1, *) {
            return true
        }
        #endif
        return false
    }
}

// Production app with comprehensive compiler directive usage
@main
struct HealthApp: App {
    #if os(iOS)
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    #endif
    
    init() {
        setupApp()
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                #if os(macOS)
                .frame(minWidth: 800, minHeight: 600)
                #endif
        }
        #if os(macOS)
        .windowStyle(.titleBar)
        .windowToolbarStyle(.unified)
        #endif
        
        #if os(iOS) || os(macOS)
        .commands {
            #if DEBUG
            CommandMenu("Debug") {
                Button("Clear Cache") {
                    clearCache()
                }
                .keyboardShortcut("K", modifiers: [.command, .shift])
                
                Button("Reset Database") {
                    resetDatabase()
                }
                
                Divider()
                
                Button("Show Logs") {
                    showLogs()
                }
            }
            #endif
        }
        #endif
    }
    
    private func setupApp() {
        HealthLogger.log("App launched", category: .ui)
        
        #if DEBUG
        // Debug-only setup
        setupDebugMenu()
        enableVerboseLogging()
        
        #warning("Remove test data before release")
        loadTestData()
        #endif
        
        #if BETA
        // Beta testing setup
        setupCrashReporting()
        setupAnalytics(verbose: true)
        #endif
        
        #if !DEBUG && !BETA
        // Production setup
        setupCrashReporting()
        setupAnalytics(verbose: false)
        optimizePerformance()
        #endif
        
        // Platform-specific setup
        #if os(iOS)
        setupNotifications()
        setupBackgroundTasks()
        #elseif os(watchOS)
        setupComplications()
        #elseif os(macOS)
        setupMenuBar()
        #endif
        
        // Version-specific features
        if #available(iOS 17.0, macOS 14.0, *) {
            setupLatestFeatures()
        } else {
            #warning("Consider dropping support for older OS versions")
            setupLegacyFeatures()
        }
    }
    
    #if DEBUG
    private func setupDebugMenu() {
        HealthLogger.log("Setting up debug menu", category: .ui)
    }
    
    private func enableVerboseLogging() {
        UserDefaults.standard.set(true, forKey: "VerboseLogging")
    }
    
    private func loadTestData() {
        HealthLogger.log("Loading test data", category: .database)
    }
    #endif
    
    private func clearCache() {
        HealthLogger.log("Clearing cache", category: .database)
    }
    
    private func resetDatabase() {
        HealthLogger.log("Resetting database", category: .database)
    }
    
    private func showLogs() {
        #if os(macOS)
        NSWorkspace.shared.open(URL(fileURLWithPath: "/Applications/Console.app"))
        #endif
    }
    
    private func setupCrashReporting() {
        // Crash reporting setup
    }
    
    private func setupAnalytics(verbose: Bool) {
        // Analytics setup
    }
    
    private func optimizePerformance() {
        // Production optimizations
    }
    
    #if os(iOS)
    private func setupNotifications() {
        HealthLogger.log("Setting up notifications", category: .ui)
    }
    
    private func setupBackgroundTasks() {
        HealthLogger.log("Setting up background tasks", category: .sync)
    }
    #endif
    
    #if os(watchOS)
    private func setupComplications() {
        HealthLogger.log("Setting up complications", category: .ui)
    }
    #endif
    
    #if os(macOS)
    private func setupMenuBar() {
        HealthLogger.log("Setting up menu bar", category: .ui)
    }
    #endif
    
    @available(iOS 17.0, macOS 14.0, *)
    private func setupLatestFeatures() {
        HealthLogger.log("Setting up iOS 17+ features", category: .ui)
    }
    
    private func setupLegacyFeatures() {
        HealthLogger.log("Setting up legacy features", category: .ui)
    }
}

#if os(iOS)
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil
    ) -> Bool {
        HealthLogger.log("App delegate setup complete", category: .ui)
        return true
    }
}
#endif
```

## Common Pitfalls & Best Practices

### Mirror API Pitfalls
1. **Performance Impact**: Mirror reflection is slow - never use in hot paths
2. **Read-Only Nature**: Unlike Java, you can't modify properties through Mirror
3. **Limited Type Information**: Swift's Mirror provides less info than Java reflection
4. **No Method Access**: Can't invoke methods through Mirror (unlike Java)

### Macro Best Practices
1. **Compile-Time Over Runtime**: Prefer macros for code generation vs runtime reflection
2. **Version Requirements**: Check Swift version compatibility (5.9+ for custom macros)
3. **Separate Packages**: Custom macros require separate Swift packages
4. **Testing**: Test macro expansions thoroughly

### String Interpolation Best Practices
1. **Domain-Specific**: Create interpolations for your domain (health metrics, dates)
2. **Type Safety**: Maintain type safety even with dynamic strings
3. **Performance**: String interpolation is resolved at compile-time - very efficient

### Dynamic Features Best Practices
1. **Limited Use**: Use sparingly - Swift prefers static typing
2. **API Boundaries**: Good for dynamic APIs or JavaScript interop
3. **Type Erasure**: Be careful with type information loss

### Compiler Directives Best Practices
1. **Build Configurations**: Define custom flags in build settings
2. **Platform Checks**: Always check platform availability
3. **Debug Code**: Use #if DEBUG to strip debug code from release builds
4. **API Availability**: Use #available for runtime checks, @available for declarations

## Integration with SwiftUI & iOS Development

SwiftUI and iOS frameworks work seamlessly with metaprogramming features. The @Observable macro is essential for SwiftUI's reactive updates, while compiler directives enable cross-platform code. String interpolation creates readable formatters for health data display, and the Mirror API helps with debugging complex view hierarchies.

For HealthKit integration, use compiler directives to handle platform differences between iOS and watchOS. SwiftData's @Model macro eliminates Core Data boilerplate. Dynamic features can create flexible configuration systems that adapt to different health data schemas without recompilation.

## Production Considerations

### Testing Strategies
- **Mirror API**: Test serialization/deserialization thoroughly
- **Macros**: Unit test macro expansions, verify generated code
- **String Interpolation**: Test edge cases and localization
- **Dynamic Features**: Extensive testing needed due to lack of compile-time checks
- **Compiler Directives**: Test all build configurations and platforms

### Debugging Techniques
- Use `#if DEBUG` blocks for additional logging
- Mirror API for runtime inspection of complex objects
- Macro expansions visible in Xcode's generated interface
- String interpolation for readable debug output

### iOS Version Considerations (iOS 17+)
- @Observable macro requires iOS 17+
- SwiftData @Model requires iOS 17+
- Older iOS versions need fallback implementations
- Use `#available` and `@available` appropriately

### Performance Implications
- Mirror reflection has significant overhead - cache results
- Macros have zero runtime cost - preferred for performance
- String interpolation is compile-time - no runtime penalty
- Dynamic features have lookup overhead - use judiciously
- Compiler directives enable dead code elimination

## Exercises for Mastery

### Exercise 1: Build a Health Data Debugger
Create a generic debugger that uses Mirror API to inspect any health-related model and generate a human-readable report. Include type information, nested object support, and collection handling. This mirrors Java's toString() generation but with Swift's safety.

### Exercise 2: Create a Macro-Enhanced Model
Design a health tracking model using SwiftData's @Model macro combined with custom validation. Add computed properties for BMI, risk scores, and trend analysis. Compare this to your experience with JPA entities or Angular models.

### Exercise 3: Dynamic Configuration System
Build a production-ready configuration system using dynamic member lookup that supports validation, persistence, and SwiftUI integration. Make it feel as flexible as TypeScript's dynamic objects but with Swift's type safety. Include health-specific settings like sync intervals, data retention, and feature flags.

## Summary

Swift's metaprogramming features provide powerful tools for building flexible, maintainable iOS apps while preserving performance and type safety. Unlike Java's extensive runtime reflection or TypeScript's dynamic nature, Swift takes a balanced approach emphasizing compile-time code generation and limited runtime introspection. These features are essential for production iOS development, especially when building adaptive health tracking apps that need to handle varied data schemas, support multiple platforms, and maintain high performance standards.

Remember that Swift's philosophy differs from your Java/TypeScript background - it prefers compile-time solutions over runtime flexibility. Embrace macros for code generation, use Mirror API sparingly for debugging, leverage string interpolation for domain-specific formatting, apply dynamic features only at API boundaries, and use compiler directives extensively for platform-specific code. This approach will help you build robust, performant iOS apps that leverage Swift's unique strengths.