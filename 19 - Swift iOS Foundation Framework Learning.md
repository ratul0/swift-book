# iOS Foundation Framework: Essential Building Blocks for Your Health App

The Foundation framework is iOS's core toolkit for handling fundamental programming tasks that aren't directly related to the user interface. Think of it as iOS's equivalent to Java's standard library combined with Spring's core utilities, or TypeScript's built-in objects plus Node.js core modules. These are the tools you'll use constantly for data management, networking, persistence, and system interaction in your health tracking app.

## Context & Background

The Foundation framework provides the essential infrastructure for iOS apps, handling everything from data serialization to file management. If you're coming from web development, imagine combining the functionality of JavaScript's built-in objects, the Fetch API, localStorage, and Node.js's fs module - that's essentially what Foundation offers for iOS.

In your Angular experience, you've likely used HttpClient for networking, RxJS for reactive programming, and various browser APIs for storage. Foundation provides iOS equivalents for all of these, but with some Apple-specific patterns that reflect iOS's focus on performance, security, and user privacy. The key difference is that Foundation is deeply integrated with iOS's sandboxed environment and permission system, something you don't encounter as strictly in web development.

For your health tracking app, Foundation will be the backbone for syncing data with servers, storing user preferences, managing cached health data, and coordinating between different parts of your app. It's the layer that sits between your SwiftUI interface and the iOS system itself.

## Core Understanding

Let me break down each Foundation component and help you build the right mental model for each:

### Codable Protocol (JSON Handling)

Codable is Swift's type-safe approach to serialization, similar to Java's Serializable interface or TypeScript interfaces with JSON.parse/stringify, but far more powerful. The mental model here is that Swift wants to guarantee at compile-time that your data transformations will work, eliminating the runtime errors you might encounter with JSON.parse() in TypeScript.

```swift
// Basic Codable structure - think of this as TypeScript interface + automatic serialization
struct HealthMetric: Codable {
    let timestamp: Date
    let heartRate: Int
    let steps: Int
    
    // Swift automatically generates encoding/decoding logic
    // No need for manual parsing like you might do in TypeScript
}
```

### URLSession (Networking)

URLSession is iOS's built-in networking API, similar to Angular's HttpClient or Axios in the JavaScript world, but with more control over caching, background transfers, and authentication. The key mental shift is that URLSession is designed for mobile constraints - it handles network interruptions, background app states, and battery optimization automatically.

```swift
// URLSession follows a task-based model, not promise-based like fetch()
let task = URLSession.shared.dataTask(with: url) { data, response, error in
    // Handle response
}
task.resume() // Must explicitly start the task
```

### NotificationCenter and Combine

NotificationCenter is like a system-wide event bus (similar to Angular's EventEmitter or RxJS Subject), while Combine is Apple's reactive programming framework (directly comparable to RxJS). NotificationCenter handles app-wide events like "app went to background," while Combine handles data streams and reactive transformations.

### UserDefaults and Property Lists

UserDefaults is iOS's simple key-value storage, similar to localStorage in web browsers but type-safe and integrated with iOS's preference system. Property Lists (plists) are XML-based configuration files, similar to JSON config files in Node.js projects but with strict type requirements.

### FileManager and Bundle

FileManager is your interface to the iOS file system (like Node.js's fs module), while Bundle represents your app's resources (similar to webpack's bundled assets). The crucial difference is iOS's sandboxed file system - each app has its own isolated storage areas with specific purposes.

## Practical Code Examples

### 1. Basic Example: Simple JSON Encoding/Decoding with Codable

Let me show you how Codable transforms the way you handle JSON compared to TypeScript:

```swift
import Foundation

// Define your data model - similar to a TypeScript interface but with automatic serialization
struct MealEntry: Codable {
    let id: UUID
    let mealType: String
    let calories: Int
    let timestamp: Date
    
    // Custom property names (like @JsonProperty in Java)
    enum CodingKeys: String, CodingKey {
        case id = "meal_id"  // Maps to meal_id in JSON
        case mealType = "meal_type"
        case calories
        case timestamp
    }
}

// Basic encoding and decoding
func demonstrateCodable() {
    let meal = MealEntry(
        id: UUID(),
        mealType: "breakfast",
        calories: 450,
        timestamp: Date()
    )
    
    // Encoding to JSON
    let encoder = JSONEncoder()
    encoder.dateEncodingStrategy = .iso8601  // Handle dates properly
    
    do {
        let jsonData = try encoder.encode(meal)
        let jsonString = String(data: jsonData, encoding: .utf8)!
        print("JSON: \(jsonString)")
        
        // Decoding from JSON
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        let decodedMeal = try decoder.decode(MealEntry.self, from: jsonData)
        print("Decoded: \(decodedMeal)")
        
    } catch {
        // In Swift, we always handle errors explicitly
        print("Encoding/Decoding failed: \(error)")
    }
}

// Common mistake from TypeScript/JavaScript background:
// DON'T try to parse JSON manually like this:
func wrongWayFromJavaScript(jsonString: String) {
    // ❌ Don't do manual parsing like in JavaScript
    // let data = JSON.parse(jsonString)  // This doesn't exist in Swift
    // let meal = data as MealEntry  // This won't work
}

// ✅ The Swift way: Always use Codable for type safety
func rightWayInSwift(jsonData: Data) throws -> MealEntry {
    let decoder = JSONDecoder()
    return try decoder.decode(MealEntry.self, from: jsonData)
}
```

### 2. Real-World Example: Health App Data Synchronization

Now let's build something practical for your health tracking app that combines multiple Foundation features:

```swift
import Foundation
import Combine

// Health data model with nested structures
struct HealthDataSync: Codable {
    let userId: String
    let syncDate: Date
    let metrics: [DailyHealthMetric]
    
    struct DailyHealthMetric: Codable {
        let date: Date
        let steps: Int
        let activeCalories: Double
        let restingHeartRate: Int?  // Optional, like TypeScript's number | undefined
        let meals: [MealEntry]
        
        struct MealEntry: Codable {
            let time: Date
            let calories: Int
            let foodItems: [String]
            let fastingPeriodBefore: TimeInterval?  // Seconds since last meal
        }
    }
}

// Service class combining URLSession, Codable, and UserDefaults
class HealthDataSyncService {
    // Using Combine for reactive updates (like RxJS in Angular)
    @Published private(set) var isSyncing = false
    @Published private(set) var lastSyncDate: Date?
    
    private let baseURL = "https://api.yourhealthapp.com"
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        // Load last sync date from UserDefaults (like localStorage)
        lastSyncDate = UserDefaults.standard.object(forKey: "lastSyncDate") as? Date
    }
    
    // Async function - similar to async/await in TypeScript
    func syncHealthData() async throws -> HealthDataSync {
        // Update sync state reactively
        await MainActor.run {
            self.isSyncing = true
        }
        
        defer {
            Task { @MainActor in
                self.isSyncing = false
            }
        }
        
        // Construct URL with query parameters
        var components = URLComponents(string: "\(baseURL)/sync")!
        if let lastSync = lastSyncDate {
            components.queryItems = [
                URLQueryItem(name: "since", value: ISO8601DateFormatter().string(from: lastSync))
            ]
        }
        
        guard let url = components.url else {
            throw URLError(.badURL)
        }
        
        // Create request with headers (like Axios config)
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        
        // Get auth token from Keychain (more secure than UserDefaults)
        if let token = loadAuthToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        // Perform network request with modern async/await
        let (data, response) = try await URLSession.shared.data(for: request)
        
        // Check response status (like checking response.ok in fetch)
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw URLError(.badServerResponse)
        }
        
        // Decode response
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        let healthData = try decoder.decode(HealthDataSync.self, from: data)
        
        // Save sync date to UserDefaults
        UserDefaults.standard.set(healthData.syncDate, forKey: "lastSyncDate")
        lastSyncDate = healthData.syncDate
        
        // Post notification for other parts of app (like EventEmitter)
        NotificationCenter.default.post(
            name: .healthDataSynced,
            object: nil,
            userInfo: ["data": healthData]
        )
        
        return healthData
    }
    
    // Upload local data with retry logic
    func uploadHealthMetrics(_ metrics: [HealthDataSync.DailyHealthMetric]) async throws {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        let jsonData = try encoder.encode(metrics)
        
        var request = URLRequest(url: URL(string: "\(baseURL)/upload")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = jsonData
        
        // Retry logic for network failures
        var retries = 3
        var lastError: Error?
        
        while retries > 0 {
            do {
                let (_, response) = try await URLSession.shared.data(for: request)
                
                guard let httpResponse = response as? HTTPURLResponse,
                      (200...299).contains(httpResponse.statusCode) else {
                    throw URLError(.badServerResponse)
                }
                
                // Success - save upload timestamp
                UserDefaults.standard.set(Date(), forKey: "lastUploadDate")
                return
                
            } catch {
                lastError = error
                retries -= 1
                
                if retries > 0 {
                    // Exponential backoff
                    try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(3 - retries)) * 1_000_000_000))
                }
            }
        }
        
        throw lastError ?? URLError(.unknown)
    }
    
    private func loadAuthToken() -> String? {
        // In production, use Keychain instead of UserDefaults for sensitive data
        // This is simplified for demonstration
        return UserDefaults.standard.string(forKey: "authToken")
    }
}

// Custom notification name extension (like defining event types)
extension Notification.Name {
    static let healthDataSynced = Notification.Name("healthDataSynced")
}
```

### 3. Production Example: Complete Data Management System

Here's a production-ready example that combines all Foundation concepts for your health app:

```swift
import Foundation
import Combine
import SwiftUI
import SwiftData

// MARK: - Data Models with Advanced Codable Features

struct HealthAPIResponse: Codable {
    let status: ResponseStatus
    let data: HealthPayload?
    let error: APIError?
    
    enum ResponseStatus: String, Codable {
        case success, error, partial
    }
    
    struct HealthPayload: Codable {
        let user: UserProfile
        let recentMetrics: [HealthMetric]
        let insights: [HealthInsight]
        
        // Nested types with custom decoding
        struct HealthMetric: Codable {
            let id: UUID
            let type: MetricType
            let value: Double
            let unit: String
            let recordedAt: Date
            let source: DataSource
            
            enum MetricType: String, Codable, CaseIterable {
                case steps, heartRate, calories, sleep, water
                
                var displayName: String {
                    switch self {
                    case .steps: return "Steps"
                    case .heartRate: return "Heart Rate"
                    case .calories: return "Calories"
                    case .sleep: return "Sleep"
                    case .water: return "Water Intake"
                    }
                }
            }
            
            struct DataSource: Codable {
                let device: String
                let app: String?
                let isManual: Bool
            }
        }
    }
    
    struct APIError: Codable, LocalizedError {
        let code: String
        let message: String
        let details: [String: AnyCodable]?  // Generic JSON handling
        
        var errorDescription: String? { message }
    }
}

// Generic Codable wrapper for unknown JSON types (like any in TypeScript)
struct AnyCodable: Codable {
    let value: Any
    
    init(_ value: Any) {
        self.value = value
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let intVal = try? container.decode(Int.self) {
            value = intVal
        } else if let doubleVal = try? container.decode(Double.self) {
            value = doubleVal
        } else if let boolVal = try? container.decode(Bool.self) {
            value = boolVal
        } else if let stringVal = try? container.decode(String.self) {
            value = stringVal
        } else if let arrayVal = try? container.decode([AnyCodable].self) {
            value = arrayVal.map { $0.value }
        } else if let dictVal = try? container.decode([String: AnyCodable].self) {
            value = dictVal.mapValues { $0.value }
        } else {
            throw DecodingError.dataCorruptedError(in: container, debugDescription: "Cannot decode value")
        }
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        switch value {
        case let val as Int: try container.encode(val)
        case let val as Double: try container.encode(val)
        case let val as Bool: try container.encode(val)
        case let val as String: try container.encode(val)
        default: throw EncodingError.invalidValue(value, EncodingError.Context(codingPath: [], debugDescription: "Cannot encode value"))
        }
    }
}

// MARK: - Production Network Layer with Combine

protocol NetworkServiceProtocol {
    func request<T: Decodable>(_ endpoint: Endpoint) -> AnyPublisher<T, Error>
    func upload<T: Encodable>(_ data: T, to endpoint: Endpoint) -> AnyPublisher<Void, Error>
}

struct Endpoint {
    let path: String
    let method: HTTPMethod
    let headers: [String: String]
    let queryItems: [URLQueryItem]
    let timeoutInterval: TimeInterval
    
    enum HTTPMethod: String {
        case get = "GET", post = "POST", put = "PUT", delete = "DELETE"
    }
    
    init(path: String,
         method: HTTPMethod = .get,
         headers: [String: String] = [:],
         queryItems: [URLQueryItem] = [],
         timeoutInterval: TimeInterval = 30) {
        self.path = path
        self.method = method
        self.headers = headers
        self.queryItems = queryItems
        self.timeoutInterval = timeoutInterval
    }
}

class NetworkService: NetworkServiceProtocol {
    private let baseURL: String
    private let session: URLSession
    private let decoder: JSONDecoder
    private let encoder: JSONEncoder
    
    // Dependency injection for testability
    init(baseURL: String,
         session: URLSession = .shared,
         decoder: JSONDecoder? = nil,
         encoder: JSONEncoder? = nil) {
        self.baseURL = baseURL
        self.session = session
        
        // Configure JSON handling
        self.decoder = decoder ?? {
            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .custom { decoder in
                let container = try decoder.singleValueContainer()
                let dateString = try container.decode(String.self)
                
                // Try multiple date formats
                let formatters = [
                    ISO8601DateFormatter(),
                    DateFormatter.apiDateFormatter
                ]
                
                for formatter in formatters {
                    if let date = formatter.date(from: dateString) {
                        return date
                    }
                }
                
                throw DecodingError.dataCorruptedError(in: container, debugDescription: "Cannot decode date")
            }
            return decoder
        }()
        
        self.encoder = encoder ?? {
            let encoder = JSONEncoder()
            encoder.dateEncodingStrategy = .iso8601
            encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
            return encoder
        }()
    }
    
    func request<T: Decodable>(_ endpoint: Endpoint) -> AnyPublisher<T, Error> {
        guard let request = makeRequest(for: endpoint) else {
            return Fail(error: URLError(.badURL))
                .eraseToAnyPublisher()
        }
        
        return session.dataTaskPublisher(for: request)
            .tryMap { [weak self] data, response in
                try self?.validateResponse(response, data: data)
                return data
            }
            .decode(type: T.self, decoder: decoder)
            .receive(on: DispatchQueue.main)  // UI updates on main thread
            .retry(2)  // Automatic retry on failure
            .handleEvents(
                receiveCompletion: { completion in
                    if case .failure(let error) = completion {
                        self.logError(error, endpoint: endpoint)
                    }
                }
            )
            .eraseToAnyPublisher()
    }
    
    func upload<T: Encodable>(_ data: T, to endpoint: Endpoint) -> AnyPublisher<Void, Error> {
        guard var request = makeRequest(for: endpoint) else {
            return Fail(error: URLError(.badURL))
                .eraseToAnyPublisher()
        }
        
        do {
            request.httpBody = try encoder.encode(data)
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        } catch {
            return Fail(error: error)
                .eraseToAnyPublisher()
        }
        
        return session.dataTaskPublisher(for: request)
            .tryMap { [weak self] data, response in
                try self?.validateResponse(response, data: data)
                return ()
            }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }
    
    private func makeRequest(for endpoint: Endpoint) -> URLRequest? {
        var components = URLComponents(string: baseURL + endpoint.path)
        components?.queryItems = endpoint.queryItems.isEmpty ? nil : endpoint.queryItems
        
        guard let url = components?.url else { return nil }
        
        var request = URLRequest(url: url, timeoutInterval: endpoint.timeoutInterval)
        request.httpMethod = endpoint.method.rawValue
        
        // Add default headers
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.setValue(UIDevice.current.identifierForVendor?.uuidString ?? "unknown", 
                        forHTTPHeaderField: "X-Device-ID")
        
        // Add custom headers
        endpoint.headers.forEach { key, value in
            request.setValue(value, forHTTPHeaderField: key)
        }
        
        // Add auth token if available
        if let token = KeychainManager.shared.getAuthToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        return request
    }
    
    private func validateResponse(_ response: URLResponse, data: Data) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw URLError(.badServerResponse)
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            return  // Success
        case 401:
            // Handle authentication failure
            NotificationCenter.default.post(name: .authenticationRequired, object: nil)
            throw NetworkError.unauthorized
        case 400...499:
            // Try to decode error message
            if let apiError = try? decoder.decode(HealthAPIResponse.self, from: data).error {
                throw apiError
            }
            throw NetworkError.clientError(statusCode: httpResponse.statusCode)
        case 500...599:
            throw NetworkError.serverError(statusCode: httpResponse.statusCode)
        default:
            throw NetworkError.unknown(statusCode: httpResponse.statusCode)
        }
    }
    
    private func logError(_ error: Error, endpoint: Endpoint) {
        #if DEBUG
        print("❌ Network Error for \(endpoint.path): \(error.localizedDescription)")
        #endif
        
        // In production, send to analytics service
        // Analytics.log(event: "network_error", parameters: ["endpoint": endpoint.path, "error": error.localizedDescription])
    }
}

enum NetworkError: LocalizedError {
    case unauthorized
    case clientError(statusCode: Int)
    case serverError(statusCode: Int)
    case unknown(statusCode: Int)
    
    var errorDescription: String? {
        switch self {
        case .unauthorized:
            return "Authentication required. Please log in again."
        case .clientError(let code):
            return "Request failed with error code \(code)"
        case .serverError(let code):
            return "Server error (\(code)). Please try again later."
        case .unknown(let code):
            return "Unexpected error occurred (code: \(code))"
        }
    }
}

// MARK: - Local Storage Management

class LocalStorageManager {
    static let shared = LocalStorageManager()
    
    private let documentsDirectory: URL
    private let cachesDirectory: URL
    private let userDefaults: UserDefaults
    private let fileManager: FileManager
    
    private init() {
        fileManager = FileManager.default
        userDefaults = UserDefaults.standard
        
        // Get app directories
        documentsDirectory = fileManager.urls(for: .documentDirectory, 
                                             in: .userDomainMask).first!
        cachesDirectory = fileManager.urls(for: .cachesDirectory, 
                                          in: .userDomainMask).first!
    }
    
    // MARK: - UserDefaults Management
    
    @propertyWrapper
    struct UserDefault<T> {
        let key: String
        let defaultValue: T
        
        var wrappedValue: T {
            get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
            set { UserDefaults.standard.set(newValue, forKey: key) }
        }
    }
    
    // Type-safe UserDefaults properties
    @UserDefault(key: "lastSyncDate", defaultValue: nil)
    static var lastSyncDate: Date?
    
    @UserDefault(key: "enableNotifications", defaultValue: true)
    static var enableNotifications: Bool
    
    @UserDefault(key: "dailyStepGoal", defaultValue: 10000)
    static var dailyStepGoal: Int
    
    // MARK: - File Management
    
    func saveHealthData<T: Codable>(_ data: T, filename: String) throws {
        let url = documentsDirectory.appendingPathComponent("\(filename).json")
        let encoder = JSONEncoder()
        encoder.outputFormatting = .prettyPrinted
        let jsonData = try encoder.encode(data)
        
        // Write with data protection
        try jsonData.write(to: url, options: [.atomic, .completeFileProtection])
        
        // Exclude from iCloud backup if it's cache data
        var resourceValues = URLResourceValues()
        resourceValues.isExcludedFromBackup = filename.contains("cache")
        try url.setResourceValues(resourceValues)
    }
    
    func loadHealthData<T: Codable>(_ type: T.Type, filename: String) throws -> T {
        let url = documentsDirectory.appendingPathComponent("\(filename).json")
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode(type, from: data)
    }
    
    func deleteHealthData(filename: String) throws {
        let url = documentsDirectory.appendingPathComponent("\(filename).json")
        if fileManager.fileExists(atPath: url.path) {
            try fileManager.removeItem(at: url)
        }
    }
    
    // Cache management with expiration
    func cacheData<T: Codable>(_ data: T, key: String, expirationHours: Int = 24) throws {
        let cacheEntry = CacheEntry(data: data, expirationDate: Date().addingTimeInterval(TimeInterval(expirationHours * 3600)))
        let url = cachesDirectory.appendingPathComponent("\(key).cache")
        let jsonData = try JSONEncoder().encode(cacheEntry)
        try jsonData.write(to: url, options: .atomic)
    }
    
    func getCachedData<T: Codable>(_ type: T.Type, key: String) -> T? {
        let url = cachesDirectory.appendingPathComponent("\(key).cache")
        
        guard fileManager.fileExists(atPath: url.path),
              let data = try? Data(contentsOf: url),
              let cacheEntry = try? JSONDecoder().decode(CacheEntry<T>.self, from: data),
              cacheEntry.expirationDate > Date() else {
            return nil
        }
        
        return cacheEntry.data
    }
    
    func clearCache() throws {
        let cacheFiles = try fileManager.contentsOfDirectory(at: cachesDirectory,
                                                            includingPropertiesForKeys: nil)
        for file in cacheFiles where file.pathExtension == "cache" {
            try fileManager.removeItem(at: file)
        }
    }
    
    // Get storage size information
    func getStorageInfo() -> (documents: Int64, cache: Int64) {
        let documentsSize = directorySize(at: documentsDirectory)
        let cacheSize = directorySize(at: cachesDirectory)
        return (documentsSize, cacheSize)
    }
    
    private func directorySize(at url: URL) -> Int64 {
        guard let enumerator = fileManager.enumerator(at: url,
                                                     includingPropertiesForKeys: [.fileSizeKey]) else {
            return 0
        }
        
        var totalSize: Int64 = 0
        for case let fileURL as URL in enumerator {
            if let fileSize = try? fileURL.resourceValues(forKeys: [.fileSizeKey]).fileSize {
                totalSize += Int64(fileSize)
            }
        }
        return totalSize
    }
    
    private struct CacheEntry<T: Codable>: Codable {
        let data: T
        let expirationDate: Date
    }
}

// MARK: - Notification Center Integration

class NotificationManager {
    static let shared = NotificationManager()
    private var cancellables = Set<AnyCancellable>()
    
    // Define notification events as static properties for type safety
    struct Events {
        static let healthDataUpdated = Notification.Name("healthDataUpdated")
        static let syncCompleted = Notification.Name("syncCompleted")
        static let syncFailed = Notification.Name("syncFailed")
        static let authenticationRequired = Notification.Name("authenticationRequired")
    }
    
    // Combine publishers for reactive updates
    let healthDataPublisher = NotificationCenter.default
        .publisher(for: Events.healthDataUpdated)
        .compactMap { $0.userInfo?["data"] as? HealthAPIResponse.HealthPayload }
        .eraseToAnyPublisher()
    
    func postHealthDataUpdate(_ data: HealthAPIResponse.HealthPayload) {
        NotificationCenter.default.post(
            name: Events.healthDataUpdated,
            object: nil,
            userInfo: ["data": data, "timestamp": Date()]
        )
    }
    
    // Observe notifications with Combine
    func observeAuthenticationRequired(action: @escaping () -> Void) -> AnyCancellable {
        NotificationCenter.default
            .publisher(for: Events.authenticationRequired)
            .sink { _ in action() }
    }
}

// MARK: - SwiftUI Integration Example

struct HealthDashboardView: View {
    @StateObject private var viewModel = HealthDashboardViewModel()
    @AppStorage("dailyStepGoal") private var stepGoal = 10000
    
    var body: some View {
        NavigationView {
            List {
                if viewModel.isLoading {
                    ProgressView("Syncing health data...")
                        .frame(maxWidth: .infinity)
                        .padding()
                } else if let error = viewModel.error {
                    ErrorView(error: error) {
                        Task {
                            await viewModel.refresh()
                        }
                    }
                } else {
                    ForEach(viewModel.metrics) { metric in
                        HealthMetricRow(metric: metric)
                    }
                }
            }
            .navigationTitle("Health Dashboard")
            .refreshable {
                await viewModel.refresh()
            }
            .onAppear {
                viewModel.startObserving()
            }
        }
    }
}

@MainActor
class HealthDashboardViewModel: ObservableObject {
    @Published var metrics: [HealthAPIResponse.HealthPayload.HealthMetric] = []
    @Published var isLoading = false
    @Published var error: Error?
    
    private let networkService = NetworkService(baseURL: "https://api.health.com")
    private var cancellables = Set<AnyCancellable>()
    
    func startObserving() {
        // Observe health data updates using Combine
        NotificationManager.shared.healthDataPublisher
            .receive(on: DispatchQueue.main)
            .sink { [weak self] payload in
                self?.metrics = payload.recentMetrics
            }
            .store(in: &cancellables)
        
        // Load cached data first
        if let cached = LocalStorageManager.shared.getCachedData(
            [HealthAPIResponse.HealthPayload.HealthMetric].self,
            key: "recent_metrics"
        ) {
            metrics = cached
        }
        
        // Then fetch fresh data
        Task {
            await refresh()
        }
    }
    
    func refresh() async {
        isLoading = true
        error = nil
        
        do {
            // Fetch from network
            let endpoint = Endpoint(path: "/metrics/recent")
            let response: HealthAPIResponse = try await networkService
                .request(endpoint)
                .async()  // Convert Combine to async/await
            
            if let data = response.data {
                metrics = data.recentMetrics
                
                // Cache the data
                try LocalStorageManager.shared.cacheData(
                    data.recentMetrics,
                    key: "recent_metrics"
                )
                
                // Update last sync date
                LocalStorageManager.lastSyncDate = Date()
                
                // Post notification
                NotificationManager.shared.postHealthDataUpdate(data)
            }
            
        } catch {
            self.error = error
            // If network fails, keep showing cached data
        }
        
        isLoading = false
    }
}

// Helper extensions
extension DateFormatter {
    static let apiDateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
        formatter.locale = Locale(identifier: "en_US_POSIX")
        formatter.timeZone = TimeZone(secondsFromGMT: 0)
        return formatter
    }()
}

extension Publisher {
    func async() async throws -> Output {
        try await withCheckedThrowingContinuation { continuation in
            var cancellable: AnyCancellable?
            cancellable = first()
                .sink(
                    receiveCompletion: { completion in
                        if case .failure(let error) = completion {
                            continuation.resume(throwing: error)
                        }
                        cancellable?.cancel()
                    },
                    receiveValue: { value in
                        continuation.resume(returning: value)
                    }
                )
        }
    }
}

extension Notification.Name {
    static let authenticationRequired = Notification.Name("authenticationRequired")
}

// Simplified Keychain wrapper for the example
struct KeychainManager {
    static let shared = KeychainManager()
    
    func getAuthToken() -> String? {
        // In production, use proper Keychain API
        return UserDefaults.standard.string(forKey: "auth_token_demo")
    }
}

// Supporting Views
struct HealthMetricRow: View {
    let metric: HealthAPIResponse.HealthPayload.HealthMetric
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(metric.type.displayName)
                    .font(.headline)
                Text("\(metric.value, specifier: "%.0f") \(metric.unit)")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }
            Spacer()
            Text(metric.recordedAt, style: .time)
                .font(.caption)
        }
        .padding(.vertical, 4)
    }
}

struct ErrorView: View {
    let error: Error
    let retry: () -> Void
    
    var body: some View {
        VStack(spacing: 16) {
            Image(systemName: "exclamationmark.triangle")
                .font(.largeTitle)
                .foregroundColor(.orange)
            
            Text(error.localizedDescription)
                .multilineTextAlignment(.center)
            
            Button("Retry", action: retry)
                .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

## Common Pitfalls & Best Practices

Coming from TypeScript and Java, there are several Foundation-specific patterns that might trip you up initially:

**JSON Handling Mistakes:**
The most common error is trying to handle JSON like you would in JavaScript. In TypeScript, you might use `JSON.parse()` and cast the result, but Swift requires explicit type definitions through Codable. Always define your data structures completely - Swift won't let you access undefined properties at runtime like JavaScript would.

**Networking Anti-Patterns:**
Don't create new URLSession instances for each request like you might instantiate Axios. URLSession is designed to be reused and manages connection pooling internally. Also, remember that URLSession tasks don't start automatically - you must call `resume()` or use the async/await versions.

**Storage Misconceptions:**
UserDefaults isn't meant for large data storage like localStorage might be used in web apps. It has a size limit and synchronizes to iCloud. For anything over a few KB, use file storage with FileManager. Also, never store sensitive data like passwords in UserDefaults - use the Keychain instead.

**Memory Management with Closures:**
When using completion handlers with URLSession or NotificationCenter observers, always use `[weak self]` in closures to avoid retain cycles. This is similar to being careful with subscriptions in RxJS, but Swift makes it explicit.

**File System Differences:**
iOS apps are sandboxed, meaning you can only write to specific directories. Always use FileManager's URL methods to get the correct paths - don't hardcode paths like you might in Node.js. Also, remember that the Documents directory is backed up to iCloud by default, while Caches isn't.

## Integration with SwiftUI & iOS Development

Foundation components integrate seamlessly with SwiftUI through property wrappers and Combine publishers. Here's how to connect Foundation to your UI layer effectively:

```swift
import SwiftUI
import Combine

// SwiftUI View with Foundation integration
struct HealthSyncView: View {
    @StateObject private var syncManager = SyncManager()
    @AppStorage("autoSync") private var autoSyncEnabled = true  // UserDefaults wrapper
    @State private var showingError = false
    
    var body: some View {
        VStack(spacing: 20) {
            // Display last sync from UserDefaults
            if let lastSync = syncManager.lastSyncDate {
                Label("Last synced: \(lastSync, formatter: RelativeDateTimeFormatter())", 
                      systemImage: "arrow.triangle.2.circlepath")
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
            
            // Sync button with loading state
            Button(action: { Task { await syncManager.performSync() } }) {
                if syncManager.isSyncing {
                    ProgressView()
                        .progressViewStyle(CircularProgressViewStyle())
                } else {
                    Label("Sync Now", systemImage: "arrow.clockwise")
                }
            }
            .disabled(syncManager.isSyncing)
            .buttonStyle(.borderedProminent)
            
            // Auto-sync toggle
            Toggle("Auto-sync", isOn: $autoSyncEnabled)
                .onChange(of: autoSyncEnabled) { newValue in
                    syncManager.configureAutoSync(enabled: newValue)
                }
        }
        .padding()
        .alert("Sync Failed", isPresented: $showingError) {
            Button("OK") { }
            Button("Retry") {
                Task { await syncManager.performSync() }
            }
        } message: {
            Text(syncManager.lastError?.localizedDescription ?? "Unknown error")
        }
        .onReceive(syncManager.$lastError) { error in
            showingError = error != nil
        }
    }
}

// Manager class bridging Foundation and SwiftUI
@MainActor
class SyncManager: ObservableObject {
    @Published var isSyncing = false
    @Published var lastError: Error?
    @Published var lastSyncDate: Date? {
        didSet {
            // Persist to UserDefaults when changed
            UserDefaults.standard.set(lastSyncDate, forKey: "lastSyncDate")
        }
    }
    
    private var cancellables = Set<AnyCancellable>()
    private var syncTimer: Timer?
    
    init() {
        // Load from UserDefaults on init
        lastSyncDate = UserDefaults.standard.object(forKey: "lastSyncDate") as? Date
        
        // Listen for app lifecycle events
        NotificationCenter.default.publisher(for: UIApplication.willEnterForegroundNotification)
            .sink { [weak self] _ in
                Task { await self?.performSync() }
            }
            .store(in: &cancellables)
    }
    
    func performSync() async {
        isSyncing = true
        lastError = nil
        
        do {
            // Perform network request
            let url = URL(string: "https://api.health.com/sync")!
            let (data, _) = try await URLSession.shared.data(from: url)
            
            // Decode and save
            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .iso8601
            let response = try decoder.decode(SyncResponse.self, from: data)
            
            // Update SwiftData models
            await updateHealthData(from: response)
            
            lastSyncDate = Date()
            
        } catch {
            lastError = error
        }
        
        isSyncing = false
    }
    
    func configureAutoSync(enabled: Bool) {
        syncTimer?.invalidate()
        
        if enabled {
            // Set up periodic sync every hour
            syncTimer = Timer.scheduledTimer(withTimeInterval: 3600, repeats: true) { _ in
                Task { await self.performSync() }
            }
        }
    }
    
    private func updateHealthData(from response: SyncResponse) async {
        // Update your SwiftData models here
    }
}

struct SyncResponse: Codable {
    let data: [HealthMetric]
    let lastUpdated: Date
}

struct HealthMetric: Codable {
    let id: UUID
    let value: Double
    let type: String
    let date: Date
}
```

## Production Considerations

When deploying Foundation code to production, several factors become critical for app reliability and performance:

**Testing Strategies:**
Test your Codable implementations with malformed JSON to ensure graceful failure. Use URLProtocol to mock network requests in unit tests, avoiding actual network calls. For UserDefaults testing, create a test-specific suite to avoid polluting the main app's settings. Always test file operations with both full and empty disk scenarios.

**Debugging Techniques:**
Use Charles Proxy or Proxyman to inspect network traffic during development. For Codable debugging, implement custom `init(from decoder:)` methods temporarily to set breakpoints and inspect the decoding process. Enable `NSURLSession` logging with environment variables to see detailed network activity.

**iOS Version Considerations:**
While targeting iOS 17+, be aware that async/await networking is fully mature, but some older third-party libraries might still use completion handlers. Use the new SwiftData instead of Core Data, and prefer Observation framework over Combine for simpler state management when possible.

**Performance Optimizations:**
Cache decoded JSON objects to avoid repeated parsing. Use background queues for file operations and JSON encoding/decoding of large datasets. Implement progressive loading for large data sets rather than loading everything at once. Consider using `JSONDecoder`'s `userInfo` dictionary to pass context and avoid creating multiple decoder instances.

**Security Best Practices:**
Always validate and sanitize data from network responses before saving locally. Use HTTPS exclusively and implement certificate pinning for sensitive health data. Store authentication tokens in the Keychain, never in UserDefaults. Implement proper data protection classes for files containing health information.

## Exercises for Mastery

Let me give you three exercises that progressively build your Foundation skills while working toward your health app:

### Exercise 1: Basic Health Metric Tracker
Create a simple health metric logger that saves daily measurements to a JSON file. Start with a structure similar to what you'd build in TypeScript:
- Define a `HealthMeasurement` struct with date, weight, and steps properties
- Implement save and load functions using FileManager and Codable
- Add a method to export the last 7 days of data as JSON
- Challenge: Implement automatic cleanup of data older than 90 days

### Exercise 2: Fasting Window Calculator with Network Sync
Build a fasting tracker that syncs with a mock API:
- Create models for fasting windows (start time, end time, type of fast)
- Implement a network service that uploads fasting data using URLSession
- Store the user's fasting schedule in UserDefaults
- Use NotificationCenter to broadcast when a fasting window starts/ends
- Challenge: Add retry logic with exponential backoff for failed uploads

### Exercise 3: Complete Health Data Pipeline
Create a production-ready data synchronization system:
- Design a comprehensive health data model with meals, exercises, and biometrics
- Implement bidirectional sync with conflict resolution (latest-write-wins)
- Create a caching layer that stores recent data for offline access
- Add background sync using iOS background tasks
- Use Combine to create a reactive data pipeline that updates the UI automatically
- Challenge: Implement data compression for network requests and storage optimization

Each exercise builds on the previous one, taking you from TypeScript-familiar patterns to iOS-specific implementations, ultimately creating components you can use directly in your health tracking app. Remember to test each component thoroughly and consider edge cases like network failures, invalid data, and storage limitations.

These Foundation components form the backbone of any iOS app, handling everything that happens behind your beautiful SwiftUI interface. Understanding them deeply will make the difference between an app that works and an app that works reliably in production, handles errors gracefully, and provides a smooth user experience even under challenging conditions.

