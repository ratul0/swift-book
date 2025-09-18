---
title: "Chapter 20"
weight: 200
---

# iOS Data Persistence with SwiftData & Security

## Context & Background

### What is SwiftData?
SwiftData is Apple's modern persistence framework introduced in iOS 17, designed as the spiritual successor to Core Data. Think of it as **Hibernate/JPA for iOS** or **TypeORM for Swift** - it's an ORM that handles object persistence with minimal boilerplate.

### Comparison to Your Existing Knowledge:

| Concept | Your Background | iOS Equivalent |
|---------|----------------|----------------|
| **ORM** | Hibernate (Java), Eloquent (Laravel), TypeORM | SwiftData |
| **Entity Classes** | `@Entity` (JPA), Eloquent Models | `@Model` macro |
| **Query Builder** | JPQL, Eloquent queries, LINQ | `@Query` property wrapper |
| **Migrations** | Flyway, Laravel migrations | SwiftData Schema Migrations |
| **Cloud Sync** | Custom REST APIs + sync logic | CloudKit (automatic sync) |
| **Secure Storage** | Environment variables, encrypted DB | Keychain Services |

### When to Use Each:
- **SwiftData**: Primary app data (user content, settings, cached data)
- **CloudKit**: When you need cross-device sync without building a backend
- **Keychain**: Sensitive data (API keys, tokens, passwords)

## Core Understanding

### SwiftData Architecture

```
┌─────────────────────────────────────┐
│         SwiftUI Views               │
│         (@Query binds here)         │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│      @Model Classes                 │
│   (Your data model definitions)     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│      ModelContainer                 │
│   (Database connection/context)     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│    SQLite Database / CloudKit       │
└─────────────────────────────────────┘
```

### Mental Model
Think of SwiftData as **Angular Services + RxJS Observables + SQLite**:
- `@Model` = Your TypeScript interface + ORM decorators
- `@Query` = Observable that auto-updates UI (like Angular's async pipe)
- `ModelContainer` = Database connection pool (like Spring's DataSource)
- `ModelContext` = Unit of Work pattern (like JPA's EntityManager)

## Practical Code Examples

### 1. Basic Example: Simple Model and Query

```swift
import SwiftData
import SwiftUI

// WRONG WAY (Java/TypeScript thinking)
// class Meal {  // ❌ Missing @Model
//     var id: UUID = UUID()  // ❌ SwiftData handles ID automatically
//     var name: String = ""  // ❌ Don't initialize with defaults
// }

// SWIFT WAY ✅
@Model  // Like @Entity in JPA
final class Meal {  // 'final' for performance (like 'sealed' in Kotlin)
    // SwiftData automatically creates a unique identifier
    var name: String
    var calories: Int
    var consumedAt: Date
    
    // Unlike JPA, initializer is required
    init(name: String, calories: Int, consumedAt: Date = Date()) {
        self.name = name
        self.calories = calories
        self.consumedAt = consumedAt
    }
}

// In your SwiftUI View
struct MealListView: View {
    // Like @Autowired + Observable in Spring/Angular
    @Query(sort: \Meal.consumedAt, order: .reverse) 
    private var meals: [Meal]
    
    // Like Angular's DI - gets the "EntityManager"
    @Environment(\.modelContext) private var modelContext
    
    var body: some View {
        List(meals) { meal in
            Text("\(meal.name): \(meal.calories) cal")
        }
    }
    
    func addMeal() {
        let meal = Meal(name: "Lunch", calories: 500)
        modelContext.insert(meal)  // Like entityManager.persist()
        // No need to call save() - auto-saves!
    }
}
```

### 2. Real-World Example: Health Tracking App Models

```swift
import SwiftData
import Foundation

// Main model for eating patterns
@Model
final class EatingPattern {
    var patternName: String
    var isActive: Bool = true
    var createdAt: Date = Date()
    
    // Relationships (like JPA @OneToMany)
    @Relationship(deleteRule: .cascade)  // Like CASCADE DELETE
    var meals: [TrackedMeal] = []
    
    @Relationship(deleteRule: .nullify)
    var fastingWindows: [FastingWindow] = []
    
    // Computed property (like TypeScript getter)
    var dailyCalories: Int {
        meals.filter { Calendar.current.isDateInToday($0.consumedAt) }
            .reduce(0) { $0 + $1.calories }
    }
    
    init(patternName: String) {
        self.patternName = patternName
    }
}

@Model
final class TrackedMeal {
    var name: String
    var calories: Int
    var protein: Double  // grams
    var carbs: Double
    var fat: Double
    var consumedAt: Date
    
    // Inverse relationship (like JPA @ManyToOne)
    @Relationship(inverse: \EatingPattern.meals)
    var pattern: EatingPattern?
    
    // CloudKit sync metadata
    @Attribute(.externalStorage)  // Store large data separately
    var mealPhoto: Data?
    
    @Attribute(.unique)  // Like UNIQUE constraint
    var healthKitIdentifier: String?
    
    init(name: String, calories: Int, protein: Double = 0, 
         carbs: Double = 0, fat: Double = 0) {
        self.name = name
        self.calories = calories
        self.protein = protein
        self.carbs = carbs
        self.fat = fat
        self.consumedAt = Date()
    }
}

@Model
final class FastingWindow {
    var startTime: Date
    var plannedDuration: TimeInterval  // seconds
    var actualEndTime: Date?
    
    var isActive: Bool {
        actualEndTime == nil && Date() < startTime.addingTimeInterval(plannedDuration)
    }
    
    init(startTime: Date, plannedDuration: TimeInterval) {
        self.startTime = startTime
        self.plannedDuration = plannedDuration
    }
}

// SwiftUI View with complex queries
struct PatternAnalysisView: View {
    // Complex predicate (like JPQL WHERE clause)
    @Query(
        filter: #Predicate<TrackedMeal> { meal in
            meal.calories > 300 && 
            meal.consumedAt > Date.now.addingTimeInterval(-604800)  // Last week
        },
        sort: [SortDescriptor(\.consumedAt, order: .reverse)],
        animation: .smooth  // Automatic animations on data changes!
    ) private var significantMeals: [TrackedMeal]
    
    @Environment(\.modelContext) private var modelContext
    
    var body: some View {
        // Your UI here
        EmptyView()
    }
}
```

### 3. Production Example: Full Integration with CloudKit & Keychain

```swift
import SwiftData
import CloudKit
import SwiftUI

// MARK: - Configuration for Production App
struct DataPersistenceConfiguration {
    
    // Setup ModelContainer with CloudKit
    static func createContainer() -> ModelContainer {
        let schema = Schema([
            EatingPattern.self,
            TrackedMeal.self,
            FastingWindow.self,
            UserPreferences.self
        ])
        
        let modelConfiguration = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false,
            cloudKitDatabase: .automatic  // Enables CloudKit sync
        )
        
        do {
            let container = try ModelContainer(
                for: schema,
                configurations: [modelConfiguration]
            )
            
            // Setup background sync
            setupBackgroundSync(for: container)
            
            return container
        } catch {
            fatalError("Could not create ModelContainer: \(error)")
        }
    }
    
    private static func setupBackgroundSync(for container: ModelContainer) {
        // Configure CloudKit subscription for real-time updates
        Task {
            do {
                try await container.mainContext.setQueryGenerationFrom(.current)
            } catch {
                print("Failed to setup query generation: \(error)")
            }
        }
    }
}

// MARK: - Secure API Key Storage with Keychain
enum KeychainService {
    static let serviceName = "com.yourapp.healthtracker"
    
    enum KeychainError: Error {
        case duplicateItem
        case itemNotFound
        case unexpectedData
        case unhandledError(status: OSStatus)
    }
    
    // Save sensitive data (like API keys, tokens)
    static func save(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: serviceName,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]
        
        let status = SecItemAdd(query as CFDictionary, nil)
        
        if status == errSecDuplicateItem {
            // Update existing item
            let updateQuery: [String: Any] = [
                kSecClass as String: kSecClassGenericPassword,
                kSecAttrService as String: serviceName,
                kSecAttrAccount as String: key
            ]
            
            let attributes: [String: Any] = [
                kSecValueData as String: data
            ]
            
            let updateStatus = SecItemUpdate(
                updateQuery as CFDictionary,
                attributes as CFDictionary
            )
            
            guard updateStatus == errSecSuccess else {
                throw KeychainError.unhandledError(status: updateStatus)
            }
        } else if status != errSecSuccess {
            throw KeychainError.unhandledError(status: status)
        }
    }
    
    // Retrieve sensitive data
    static func load(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: serviceName,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        
        var item: CFTypeRef?
        let status = SecItemCopyMatching(query as CFDictionary, &item)
        
        guard status != errSecItemNotFound else {
            throw KeychainError.itemNotFound
        }
        
        guard status == errSecSuccess else {
            throw KeychainError.unhandledError(status: status)
        }
        
        guard let data = item as? Data else {
            throw KeychainError.unexpectedData
        }
        
        return data
    }
}

// MARK: - Migration Support
extension ModelContainer {
    
    // Custom migration for schema changes
    func performMigrations() async throws {
        let context = self.mainContext
        
        // Example: Migrate from old version
        if needsMigration() {
            try await context.transaction {
                // Fetch all old data
                let descriptor = FetchDescriptor<TrackedMeal>()
                let meals = try context.fetch(descriptor)
                
                // Perform migration logic
                for meal in meals {
                    if meal.healthKitIdentifier == nil {
                        meal.healthKitIdentifier = UUID().uuidString
                    }
                }
            }
        }
    }
    
    private func needsMigration() -> Bool {
        // Check UserDefaults for migration flag
        !UserDefaults.standard.bool(forKey: "DidMigrateToV2")
    }
}

// MARK: - Production-Ready View Model
@MainActor
class HealthDataViewModel: ObservableObject {
    private let modelContext: ModelContext
    private let keychainService = KeychainService.self
    
    @Published var syncStatus: SyncStatus = .idle
    @Published var error: Error?
    
    enum SyncStatus {
        case idle, syncing, success, failure(Error)
    }
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
        setupCloudKitNotifications()
    }
    
    private func setupCloudKitNotifications() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleCloudKitSync),
            name: .CKAccountChanged,
            object: nil
        )
    }
    
    @objc private func handleCloudKitSync() {
        Task {
            await checkCloudKitStatus()
        }
    }
    
    func checkCloudKitStatus() async {
        do {
            let container = CKContainer.default()
            let status = try await container.accountStatus()
            
            switch status {
            case .available:
                syncStatus = .success
            case .noAccount:
                throw CloudKitError.noAccount
            case .restricted, .couldNotDetermine:
                throw CloudKitError.unavailable
            @unknown default:
                throw CloudKitError.unknown
            }
        } catch {
            syncStatus = .failure(error)
            self.error = error
        }
    }
    
    func saveAPIKey(_ key: String) async throws {
        guard let data = key.data(using: .utf8) else {
            throw KeychainError.unexpectedData
        }
        try KeychainService.save(key: "healthkit_api_key", data: data)
    }
    
    func loadAPIKey() async throws -> String {
        let data = try KeychainService.load(key: "healthkit_api_key")
        guard let key = String(data: data, encoding: .utf8) else {
            throw KeychainError.unexpectedData
        }
        return key
    }
}

enum CloudKitError: LocalizedError {
    case noAccount, unavailable, unknown
    
    var errorDescription: String? {
        switch self {
        case .noAccount:
            return "iCloud account required for sync"
        case .unavailable:
            return "iCloud sync unavailable"
        case .unknown:
            return "Unknown iCloud error"
        }
    }
}
```

## Common Pitfalls & Best Practices

### Mistakes from Java/TypeScript Background:

1. **Over-engineering Models**
   ```swift
   // ❌ Java thinking - too much boilerplate
   @Model class Meal {
       private var _id: UUID = UUID()  // SwiftData handles this
       var getId() -> UUID { return _id }  // Not needed
   }
   
   // ✅ Swift way - lean and clean
   @Model class Meal {
       var name: String
       init(name: String) { self.name = name }
   }
   ```

2. **Manual Transaction Management**
   ```swift
   // ❌ Spring/JPA thinking
   modelContext.beginTransaction()  // Doesn't exist
   modelContext.insert(meal)
   modelContext.commit()  // Not needed
   
   // ✅ SwiftData auto-manages transactions
   modelContext.insert(meal)  // Auto-saves
   ```

3. **Forgetting SwiftUI Integration**
   ```swift
   // ❌ Manually refreshing UI
   class ViewModel {
       var meals: [Meal] = []
       func refresh() { meals = fetchMeals() }
   }
   
   // ✅ Use @Query for automatic updates
   @Query private var meals: [Meal]  // Auto-refreshes!
   ```

### Memory/Performance Implications:
- SwiftData lazy-loads relationships (like JPA's FetchType.LAZY)
- Use `@Attribute(.externalStorage)` for large binary data
- Batch operations with `context.transaction { }` for better performance
- CloudKit has quota limits (40 requests/second)

## Integration with SwiftUI & iOS Development

### Complete SwiftUI Integration Example:

```swift
import SwiftUI
import SwiftData

@main
struct HealthTrackerApp: App {
    let container = DataPersistenceConfiguration.createContainer()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)  // Inject like Angular providers
    }
}

struct MealTrackerView: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var todaysMeals: [TrackedMeal]
    
    @State private var showingAddMeal = false
    @State private var mealName = ""
    @State private var calories = ""
    
    init() {
        // Configure query with predicate
        let calendar = Calendar.current
        let today = calendar.startOfDay(for: Date())
        let tomorrow = calendar.date(byAdding: .day, value: 1, to: today)!
        
        _todaysMeals = Query(
            filter: #Predicate<TrackedMeal> { meal in
                meal.consumedAt >= today && meal.consumedAt < tomorrow
            },
            sort: \.consumedAt
        )
    }
    
    var body: some View {
        NavigationStack {
            List {
                ForEach(todaysMeals) { meal in
                    MealRow(meal: meal)
                }
                .onDelete(perform: deleteMeals)
            }
            .navigationTitle("Today's Meals")
            .toolbar {
                Button("Add Meal") { showingAddMeal = true }
            }
            .sheet(isPresented: $showingAddMeal) {
                AddMealSheet()
            }
        }
    }
    
    func deleteMeals(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(todaysMeals[index])
        }
    }
}
```

## Production Considerations

### Testing SwiftData:
```swift
import XCTest
@testable import HealthTracker

class MealTests: XCTestCase {
    var container: ModelContainer!
    var context: ModelContext!
    
    override func setUp() {
        super.setUp()
        
        // In-memory container for testing
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try! ModelContainer(
            for: TrackedMeal.self,
            configurations: [config]
        )
        context = container.mainContext
    }
    
    func testMealCreation() throws {
        let meal = TrackedMeal(name: "Test", calories: 100)
        context.insert(meal)
        
        let descriptor = FetchDescriptor<TrackedMeal>()
        let meals = try context.fetch(descriptor)
        
        XCTAssertEqual(meals.count, 1)
        XCTAssertEqual(meals.first?.name, "Test")
    }
}
```

### Debugging Techniques:
- Enable Core Data debugging: `-com.apple.CoreData.SQLDebug 1`
- Use Instruments' Core Data template
- CloudKit Dashboard for sync issues
- `print(modelContext.autosaveEnabled)` to check auto-save

### iOS 17+ Considerations:
- SwiftData requires iOS 17.0+
- CloudKit sync requires iCloud account
- Use `@available(iOS 18.0, *)` for newer features

### Performance Optimization:
- Batch inserts for large datasets
- Use predicates to limit fetched data
- Index frequently queried properties with `@Attribute(.index)`
- Profile with Instruments

## Exercises for Mastery

### Exercise 1: Basic CRUD (Similar to Spring Data Repository)
Create a generic repository pattern for SwiftData:
```swift
// Your task: Implement a generic DataRepository
protocol DataRepository {
    associatedtype Model: PersistentModel
    func save(_ model: Model)
    func delete(_ model: Model)
    func findAll() -> [Model]
    func findById(_ id: PersistentIdentifier) -> Model?
}

// Implement for your TrackedMeal model
// Bonus: Add pagination support
```

### Exercise 2: Migration Challenge
Your app is live, and you need to add a `userId` field to all existing meals:
```swift
// Current model (v1)
@Model class TrackedMeal {
    var name: String
    var calories: Int
}

// New model (v2) 
@Model class TrackedMeal {
    var name: String
    var calories: Int
    var userId: String  // New field!
}

// Task: Write a migration that:
// 1. Adds userId to existing meals
// 2. Sets default value from Keychain
// 3. Handles errors gracefully
```

### Exercise 3: Health App Mini-Challenge
Build a complete feature that:
1. Tracks water intake with SwiftData
2. Syncs across devices with CloudKit
3. Stores daily goal in Keychain (secure)
4. Shows a weekly chart with Swift Charts
5. Sends local notifications when behind goal

Requirements:
- Handle offline mode gracefully
- Implement data validation
- Add unit tests for business logic
- Profile and optimize for 1000+ entries

---

**Next Steps**: Practice building a simple meal tracker with just SwiftData first. Once comfortable, add CloudKit sync. Save Keychain for truly sensitive data only. Remember: SwiftData handles 90% of your persistence needs elegantly!

