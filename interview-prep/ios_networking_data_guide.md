---
layout: page
title: "iOS Networking & Data Guide"
subtitle: "Complete Reference for Senior iOS Engineers"
---

# iOS Networking & Data Guide
## Complete Reference for Senior iOS Engineers

---

## Table of Contents
1. [URLSession Fundamentals](#urlsession-fundamentals)
2. [Modern async/await Networking](#modern-asyncawait-networking)
3. [Combine Framework](#combine-framework)
4. [JSON Parsing with Codable](#json-parsing-with-codable)
5. [Error Handling](#error-handling)
6. [Networking Layer Architecture](#networking-layer-architecture)
7. [Caching Strategies](#caching-strategies)
8. [Authentication & Security](#authentication--security)
9. [Data Persistence](#data-persistence)
10. [Image Loading & Caching](#image-loading--caching)
11. [Offline Support](#offline-support)
12. [Interview Questions](#interview-questions)

---

## URLSession Fundamentals

### Basic Request

```swift
// Simple data task
let url = URL(string: "https://api.example.com/users")!

let task = URLSession.shared.dataTask(with: url) { data, response, error in
    // Handle response
    if let error = error {
        print("Error: \(error)")
        return
    }
    
    guard let httpResponse = response as? HTTPURLResponse else {
        print("Invalid response")
        return
    }
    
    guard (200...299).contains(httpResponse.statusCode) else {
        print("Status code: \(httpResponse.statusCode)")
        return
    }
    
    guard let data = data else {
        print("No data")
        return
    }
    
    // Process data
    print("Received \(data.count) bytes")
}

task.resume()  // Don't forget!
```

### URLRequest Configuration

```swift
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
request.timeoutInterval = 30
request.cachePolicy = .reloadIgnoringLocalCacheData

// Body
let body = ["name": "John", "email": "john@example.com"]
request.httpBody = try? JSONEncoder().encode(body)

let task = URLSession.shared.dataTask(with: request) { data, response, error in
    // Handle response
}
task.resume()
```

### URLSession Configuration

```swift
// Default session
let session = URLSession.shared

// Custom configuration
let configuration = URLSessionConfiguration.default
configuration.timeoutIntervalForRequest = 30
configuration.timeoutIntervalForResource = 300
configuration.waitsForConnectivity = true
configuration.allowsCellularAccess = true
configuration.httpMaximumConnectionsPerHost = 5

// HTTP headers for all requests
configuration.httpAdditionalHeaders = [
    "User-Agent": "MyApp/1.0",
    "Accept": "application/json"
]

// Cache policy
configuration.requestCachePolicy = .returnCacheDataElseLoad
configuration.urlCache = URLCache(
    memoryCapacity: 10 * 1024 * 1024,  // 10 MB
    diskCapacity: 50 * 1024 * 1024      // 50 MB
)

let customSession = URLSession(configuration: configuration)

// Background session (for downloads/uploads)
let backgroundConfig = URLSessionConfiguration.background(withIdentifier: "com.example.app.background")
let backgroundSession = URLSession(configuration: backgroundConfig, delegate: self, delegateQueue: nil)
```

### Task Types

```swift
// 1. Data Task - In-memory data
let dataTask = URLSession.shared.dataTask(with: url) { data, response, error in
    // Handle in-memory data
}

// 2. Download Task - Save to disk
let downloadTask = URLSession.shared.downloadTask(with: url) { localURL, response, error in
    guard let localURL = localURL else { return }
    
    // Move to permanent location
    let destinationURL = /* ... */
    try? FileManager.default.moveItem(at: localURL, to: destinationURL)
}

// 3. Upload Task - Upload data
var request = URLRequest(url: url)
request.httpMethod = "POST"

let uploadTask = URLSession.shared.uploadTask(with: request, from: data) { data, response, error in
    // Handle response
}

// 4. Stream Task - Bidirectional streaming
let streamTask = URLSession.shared.streamTask(withHostName: "example.com", port: 80)
```

### URLSessionDelegate

```swift
class NetworkManager: NSObject, URLSessionDelegate, URLSessionTaskDelegate, URLSessionDataDelegate {
    lazy var session: URLSession = {
        let config = URLSessionConfiguration.default
        return URLSession(configuration: config, delegate: self, delegateQueue: nil)
    }()
    
    // MARK: - URLSessionDelegate
    
    func urlSession(_ session: URLSession, didBecomeInvalidWithError error: Error?) {
        print("Session invalidated: \(error?.localizedDescription ?? "Unknown")")
    }
    
    // MARK: - URLSessionTaskDelegate
    
    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        if let error = error {
            print("Task failed: \(error)")
        }
    }
    
    func urlSession(_ session: URLSession, task: URLSessionTask, didSendBodyData bytesSent: Int64, totalBytesSent: Int64, totalBytesExpectedToSend: Int64) {
        let progress = Double(totalBytesSent) / Double(totalBytesExpectedToSend)
        print("Upload progress: \(progress * 100)%")
    }
    
    // MARK: - URLSessionDataDelegate
    
    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        print("Received \(data.count) bytes")
    }
    
    // MARK: - Authentication
    
    func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        
        if challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust {
            // SSL pinning
            if let serverTrust = challenge.protectionSpace.serverTrust {
                let credential = URLCredential(trust: serverTrust)
                completionHandler(.useCredential, credential)
                return
            }
        }
        
        completionHandler(.performDefaultHandling, nil)
    }
}
```

---

## Modern async/await Networking

### Basic async Request

```swift
// URLSession has built-in async/await support (iOS 15+)
func fetchUsers() async throws -> [User] {
    let url = URL(string: "https://api.example.com/users")!
    
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }
    
    let users = try JSONDecoder().decode([User].self, from: data)
    return users
}

// Usage
Task {
    do {
        let users = try await fetchUsers()
        print("Fetched \(users.count) users")
    } catch {
        print("Error: \(error)")
    }
}
```

### async/await with URLRequest

```swift
func createUser(name: String, email: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users")!
    
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    
    let body = ["name": name, "email": email]
    request.httpBody = try JSONEncoder().encode(body)
    
    let (data, response) = try await URLSession.shared.data(for: request)
    
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }
    
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}
```

### Cancellation Support

```swift
class UserViewModel {
    private var loadTask: Task<Void, Never>?
    
    func loadUsers() {
        // Cancel previous task
        loadTask?.cancel()
        
        loadTask = Task {
            do {
                let users = try await fetchUsers()
                
                // Check if cancelled
                guard !Task.isCancelled else {
                    print("Task was cancelled")
                    return
                }
                
                await MainActor.run {
                    self.users = users
                }
            } catch {
                print("Error: \(error)")
            }
        }
    }
    
    func cancelLoading() {
        loadTask?.cancel()
    }
}
```

### Parallel Requests

```swift
// Sequential (slow)
func fetchDataSequential() async throws -> (users: [User], posts: [Post]) {
    let users = try await fetchUsers()
    let posts = try await fetchPosts()
    return (users, posts)
}

// Parallel (fast)
func fetchDataParallel() async throws -> (users: [User], posts: [Post]) {
    async let users = fetchUsers()
    async let posts = fetchPosts()
    
    return try await (users, posts)
}

// Task group for dynamic number of requests
func fetchMultipleUsers(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask {
                try await fetchUser(id: id)
            }
        }
        
        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

---

## Combine Framework

### Basic Publisher

```swift
import Combine

// Create a data task publisher
func fetchUsers() -> AnyPublisher<[User], Error> {
    let url = URL(string: "https://api.example.com/users")!
    
    return URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)  // Extract data
        .decode(type: [User].self, decoder: JSONDecoder())  // Decode JSON
        .receive(on: DispatchQueue.main)  // Switch to main thread
        .eraseToAnyPublisher()
}

// Usage
class UserViewModel {
    private var cancellables = Set<AnyCancellable>()
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: Error?
    
    func loadUsers() {
        isLoading = true
        
        fetchUsers()
            .sink(
                receiveCompletion: { [weak self] completion in
                    self?.isLoading = false
                    
                    if case .failure(let error) = completion {
                        self?.error = error
                    }
                },
                receiveValue: { [weak self] users in
                    self?.users = users
                }
            )
            .store(in: &cancellables)
    }
}
```

### Combine Operators

```swift
// Map - Transform values
publisher
    .map { $0.uppercased() }

// Filter - Keep only matching values
publisher
    .filter { $0.count > 3 }

// FlatMap - Transform and flatten
publisher
    .flatMap { id in
        fetchUserDetails(id: id)
    }

// Debounce - Wait for pause in events
searchTextField.textPublisher
    .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
    .sink { searchText in
        // Search after user stops typing
    }

// RemoveDuplicates - Ignore duplicate values
publisher
    .removeDuplicates()

// Retry - Retry on failure
publisher
    .retry(3)

// Catch - Handle errors
publisher
    .catch { error -> AnyPublisher<[User], Never> in
        print("Error: \(error)")
        return Just([]).eraseToAnyPublisher()
    }

// CombineLatest - Combine multiple publishers
Publishers.CombineLatest(publisher1, publisher2)
    .sink { value1, value2 in
        // Both have emitted
    }

// Zip - Pair values from multiple publishers
Publishers.Zip(publisher1, publisher2)
    .sink { value1, value2 in
        // Both have emitted, paired together
    }
```

### Custom Publisher

```swift
class UserService {
    private var cancellables = Set<AnyCancellable>()
    
    func fetchUsers() -> AnyPublisher<[User], Error> {
        let url = URL(string: "https://api.example.com/users")!
        
        var request = URLRequest(url: url)
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        
        return URLSession.shared.dataTaskPublisher(for: request)
            .tryMap { output in
                // Validate response
                guard let response = output.response as? HTTPURLResponse,
                      (200...299).contains(response.statusCode) else {
                    throw NetworkError.invalidResponse
                }
                return output.data
            }
            .decode(type: [User].self, decoder: JSONDecoder())
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }
}
```

### Subject (Manual Publisher)

```swift
class UserViewModel {
    // PassthroughSubject - doesn't store value
    let userSelected = PassthroughSubject<User, Never>()
    
    // CurrentValueSubject - stores current value
    let currentUser = CurrentValueSubject<User?, Never>(nil)
    
    func selectUser(_ user: User) {
        userSelected.send(user)
        currentUser.send(user)
    }
}

// Usage
viewModel.userSelected
    .sink { user in
        print("Selected: \(user.name)")
    }
    .store(in: &cancellables)

// Get current value
if let user = viewModel.currentUser.value {
    print("Current user: \(user.name)")
}
```

---

## JSON Parsing with Codable

### Basic Codable

```swift
struct User: Codable {
    let id: String
    let name: String
    let email: String
}

// Encoding
let user = User(id: "123", name: "John", email: "john@example.com")
let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted
let jsonData = try encoder.encode(user)

// Decoding
let decoder = JSONDecoder()
let decodedUser = try decoder.decode(User.self, from: jsonData)
```

### Custom Keys (CodingKeys)

```swift
// API returns: { "user_id": "123", "full_name": "John Doe" }

struct User: Codable {
    let id: String
    let name: String
    
    enum CodingKeys: String, CodingKey {
        case id = "user_id"
        case name = "full_name"
    }
}
```

### Nested JSON

```swift
// JSON:
// {
//   "id": "123",
//   "profile": {
//     "name": "John",
//     "email": "john@example.com"
//   }
// }

struct User: Codable {
    let id: String
    let profile: Profile
    
    struct Profile: Codable {
        let name: String
        let email: String
    }
}
```

### Flattening Nested JSON

```swift
// Want to flatten nested structure

struct User: Codable {
    let id: String
    let name: String
    let email: String
    
    enum CodingKeys: String, CodingKey {
        case id
        case profile
    }
    
    enum ProfileKeys: String, CodingKey {
        case name
        case email
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(String.self, forKey: .id)
        
        let profileContainer = try container.nestedContainer(keyedBy: ProfileKeys.self, forKey: .profile)
        name = try profileContainer.decode(String.self, forKey: .name)
        email = try profileContainer.decode(String.self, forKey: .email)
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        
        var profileContainer = container.nestedContainer(keyedBy: ProfileKeys.self, forKey: .profile)
        try profileContainer.encode(name, forKey: .name)
        try profileContainer.encode(email, forKey: .email)
    }
}
```

### Optional and Default Values

```swift
struct User: Codable {
    let id: String
    let name: String
    let email: String?  // Optional - allowed to be null
    let age: Int        // Required
    let bio: String     // Has default if missing
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        
        id = try container.decode(String.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)
        email = try container.decodeIfPresent(String.self, forKey: .email)
        age = try container.decode(Int.self, forKey: .age)
        
        // Provide default if missing
        bio = try container.decodeIfPresent(String.self, forKey: .bio) ?? "No bio"
    }
}
```

### Date Handling

```swift
let decoder = JSONDecoder()

// ISO8601 format: "2023-10-24T12:00:00Z"
decoder.dateDecodingStrategy = .iso8601

// Unix timestamp (seconds since 1970)
decoder.dateDecodingStrategy = .secondsSince1970

// Milliseconds timestamp
decoder.dateDecodingStrategy = .millisecondsSince1970

// Custom format
let formatter = DateFormatter()
formatter.dateFormat = "yyyy-MM-dd"
decoder.dateDecodingStrategy = .formatted(formatter)

// Custom decoder
decoder.dateDecodingStrategy = .custom { decoder in
    let container = try decoder.singleValueContainer()
    let dateString = try container.decode(String.self)
    
    // Try multiple formats
    let formatters = [
        DateFormatter.iso8601,
        DateFormatter.custom("yyyy-MM-dd")
    ]
    
    for formatter in formatters {
        if let date = formatter.date(from: dateString) {
            return date
        }
    }
    
    throw DecodingError.dataCorruptedError(in: container, debugDescription: "Invalid date format")
}
```

### Snake Case Conversion

```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase

// API: { "user_id": "123", "full_name": "John" }
struct User: Codable {
    let userId: String      // Automatically maps from "user_id"
    let fullName: String    // Automatically maps from "full_name"
}
```

### Polymorphic JSON

```swift
// Different types based on "type" field

enum Media: Codable {
    case image(Image)
    case video(Video)
    
    enum CodingKeys: String, CodingKey {
        case type
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let type = try container.decode(String.self, forKey: .type)
        
        switch type {
        case "image":
            let image = try Image(from: decoder)
            self = .image(image)
        case "video":
            let video = try Video(from: decoder)
            self = .video(video)
        default:
            throw DecodingError.dataCorrupted(
                DecodingError.Context(
                    codingPath: decoder.codingPath,
                    debugDescription: "Unknown media type: \(type)"
                )
            )
        }
    }
}

struct Image: Codable {
    let url: String
    let width: Int
    let height: Int
}

struct Video: Codable {
    let url: String
    let duration: Int
    let thumbnail: String
}
```

---

## Error Handling

### Custom Error Types

```swift
enum NetworkError: Error {
    case invalidURL
    case noData
    case decodingFailed
    case invalidResponse
    case serverError(statusCode: Int)
    case unauthorized
    case timeout
    case noConnection
}

extension NetworkError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "The URL is invalid"
        case .noData:
            return "No data received from server"
        case .decodingFailed:
            return "Failed to parse response"
        case .invalidResponse:
            return "Invalid response from server"
        case .serverError(let code):
            return "Server error: \(code)"
        case .unauthorized:
            return "You are not authorized"
        case .timeout:
            return "Request timed out"
        case .noConnection:
            return "No internet connection"
        }
    }
}
```

### Error Handling in async/await

```swift
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    
    do {
        let (data, response) = try await URLSession.shared.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            break
        case 401:
            throw NetworkError.unauthorized
        case 500...599:
            throw NetworkError.serverError(statusCode: httpResponse.statusCode)
        default:
            throw NetworkError.invalidResponse
        }
        
        guard !data.isEmpty else {
            throw NetworkError.noData
        }
        
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            return user
        } catch {
            throw NetworkError.decodingFailed
        }
        
    } catch let error as URLError {
        // Map URLError to our error
        switch error.code {
        case .notConnectedToInternet, .networkConnectionLost:
            throw NetworkError.noConnection
        case .timedOut:
            throw NetworkError.timeout
        default:
            throw error
        }
    }
}
```

### Advanced Error Recovery

```swift
// Retry logic with exponential backoff
class NetworkRetryManager {
    static func retry<T>(
        maxAttempts: Int = 3,
        initialDelay: TimeInterval = 1.0,
        operation: @escaping () async throws -> T
    ) async throws -> T {
        var lastError: Error?
        var delay = initialDelay

        for attempt in 1...maxAttempts {
            do {
                return try await operation()
            } catch NetworkError.timeout, NetworkError.noConnection {
                lastError = error

                if attempt < maxAttempts {
                    print("Attempt \(attempt) failed, retrying in \(delay)s...")
                    try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                    delay *= 2  // Exponential backoff
                }
            } catch {
                // Don't retry for other errors
                throw error
            }
        }

        throw lastError ?? NetworkError.timeout
    }
}

// Usage
func fetchDataWithRetry() async throws -> Data {
    try await NetworkRetryManager.retry {
        try await fetchData()
    }
}
```

### Error Aggregation

```swift
// Handling multiple errors in parallel operations
struct MultipleErrors: Error {
    let errors: [Error]

    var localizedDescription: String {
        errors.map { $0.localizedDescription }.joined(separator: ", ")
    }
}

func fetchMultipleResources() async throws -> (users: [User], posts: [Post]) {
    async let usersResult = Task { try await fetchUsers() }
    async let postsResult = Task { try await fetchPosts() }

    var errors: [Error] = []
    var users: [User] = []
    var posts: [Post] = []

    do {
        users = try await usersResult.value
    } catch {
        errors.append(error)
    }

    do {
        posts = try await postsResult.value
    } catch {
        errors.append(error)
    }

    if !errors.isEmpty {
        throw MultipleErrors(errors: errors)
    }

    return (users, posts)
}
```

### Error Context Enhancement

```swift
// Adding context to errors
struct ErrorContext {
    let originalError: Error
    let context: String
    let file: String
    let line: Int
    let function: String

    init(
        _ error: Error,
        context: String,
        file: String = #file,
        line: Int = #line,
        function: String = #function
    ) {
        self.originalError = error
        self.context = context
        self.file = file
        self.line = line
        self.function = function
    }
}

extension ErrorContext: LocalizedError {
    var errorDescription: String? {
        """
        Error: \(originalError.localizedDescription)
        Context: \(context)
        Location: \(file):\(line) in \(function)
        """
    }
}

// Usage
func processData() throws {
    do {
        let data = try loadData()
        try validateData(data)
    } catch {
        throw ErrorContext(
            error,
            context: "Failed to process user data for sync"
        )
    }
}
```

### Result Type

```swift
func fetchUser(id: String, completion: @escaping (Result<User, NetworkError>) -> Void) {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(.failure(.noConnection))
            return
        }
        
        guard let data = data else {
            completion(.failure(.noData))
            return
        }
        
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(.decodingFailed))
        }
    }.resume()
}

// Usage
fetchUser(id: "123") { result in
    switch result {
    case .success(let user):
        print("Got user: \(user.name)")
    case .failure(let error):
        print("Error: \(error.localizedDescription)")
    }
}
```

---

## Networking Layer Architecture

### Protocol-Based Service

```swift
// MARK: - Protocols

protocol APIEndpoint {
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var parameters: [String: Any]? { get }
}

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

protocol APIServiceProtocol {
    func request<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T
}

// MARK: - Implementation

class APIService: APIServiceProtocol {
    private let baseURL: String
    private let session: URLSession
    
    init(baseURL: String, session: URLSession = .shared) {
        self.baseURL = baseURL
        self.session = session
    }
    
    func request<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T {
        let request = try buildRequest(from: endpoint)
        
        let (data, response) = try await session.data(for: request)
        
        try validate(response: response, data: data)
        
        return try decode(data: data)
    }
    
    private func buildRequest(from endpoint: APIEndpoint) throws -> URLRequest {
        guard let url = URL(string: baseURL + endpoint.path) else {
            throw NetworkError.invalidURL
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = endpoint.method.rawValue
        
        // Headers
        endpoint.headers?.forEach { key, value in
            request.setValue(value, forHTTPHeaderField: key)
        }
        
        // Parameters (for POST/PUT)
        if let parameters = endpoint.parameters,
           endpoint.method == .post || endpoint.method == .put {
            request.httpBody = try JSONSerialization.data(withJSONObject: parameters)
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        }
        
        return request
    }
    
    private func validate(response: URLResponse, data: Data) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            break
        case 401:
            throw NetworkError.unauthorized
        default:
            throw NetworkError.serverError(statusCode: httpResponse.statusCode)
        }
    }
    
    private func decode<T: Decodable>(data: Data) throws -> T {
        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        
        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            throw NetworkError.decodingFailed
        }
    }
}

// MARK: - Endpoints

enum UserEndpoint: APIEndpoint {
    case list
    case detail(id: String)
    case create(name: String, email: String)
    case update(id: String, name: String)
    case delete(id: String)
    
    var path: String {
        switch self {
        case .list:
            return "/users"
        case .detail(let id), .update(let id, _), .delete(let id):
            return "/users/\(id)"
        case .create:
            return "/users"
        }
    }
    
    var method: HTTPMethod {
        switch self {
        case .list, .detail:
            return .get
        case .create:
            return .post
        case .update:
            return .put
        case .delete:
            return .delete
        }
    }
    
    var headers: [String: String]? {
        return [
            "Authorization": "Bearer \(AuthManager.shared.token ?? "")",
            "Accept": "application/json"
        ]
    }
    
    var parameters: [String: Any]? {
        switch self {
        case .create(let name, let email):
            return ["name": name, "email": email]
        case .update(_, let name):
            return ["name": name]
        default:
            return nil
        }
    }
}

// MARK: - Usage

class UserService {
    private let apiService: APIServiceProtocol
    
    init(apiService: APIServiceProtocol = APIService(baseURL: "https://api.example.com")) {
        self.apiService = apiService
    }
    
    func fetchUsers() async throws -> [User] {
        return try await apiService.request(UserEndpoint.list)
    }
    
    func fetchUser(id: String) async throws -> User {
        return try await apiService.request(UserEndpoint.detail(id: id))
    }
    
    func createUser(name: String, email: String) async throws -> User {
        return try await apiService.request(UserEndpoint.create(name: name, email: email))
    }
}
```

### Generic Response Wrapper

```swift
// API returns: { "data": [...], "meta": {...} }

struct APIResponse<T: Decodable>: Decodable {
    let data: T
    let meta: Meta?
    
    struct Meta: Decodable {
        let page: Int?
        let totalPages: Int?
        let totalCount: Int?
    }
}

// Usage
func fetchUsers() async throws -> [User] {
    let response: APIResponse<[User]> = try await apiService.request(UserEndpoint.list)
    return response.data
}
```

---

## Caching Strategies

### URLCache (HTTP Caching)

```swift
// Configure cache
let cache = URLCache(
    memoryCapacity: 10 * 1024 * 1024,   // 10 MB memory
    diskCapacity: 50 * 1024 * 1024,      // 50 MB disk
    diskPath: "network_cache"
)
URLCache.shared = cache

// Cache policies
var request = URLRequest(url: url)

// Always fetch from network
request.cachePolicy = .reloadIgnoringLocalCacheData

// Use cache if available, else network
request.cachePolicy = .returnCacheDataElseLoad

// Use cache only (fail if not cached)
request.cachePolicy = .returnCacheDataDontLoad

// Check cache manually
if let cachedResponse = URLCache.shared.cachedResponse(for: request) {
    let data = cachedResponse.data
    // Use cached data
} else {
    // Fetch from network
}

// Store in cache manually
let response = URLResponse(/* ... */)
let cachedResponse = CachedURLResponse(response: response, data: data)
URLCache.shared.storeCachedResponse(cachedResponse, for: request)

// Clear cache
URLCache.shared.removeAllCachedResponses()
URLCache.shared.removeCachedResponse(for: request)
```

### In-Memory Cache

```swift
class MemoryCache<Key: Hashable, Value> {
    private var cache = NSCache<NSNumber, CacheEntry>()
    private let lock = NSLock()
    
    private class CacheEntry {
        let value: Value
        let expirationDate: Date
        
        init(value: Value, expirationDate: Date) {
            self.value = value
            self.expirationDate = expirationDate
        }
        
        var isExpired: Bool {
            return Date() > expirationDate
        }
    }
    
    func set(_ value: Value, forKey key: Key, ttl: TimeInterval = 300) {
        lock.lock()
        defer { lock.unlock() }
        
        let expirationDate = Date().addingTimeInterval(ttl)
        let entry = CacheEntry(value: value, expirationDate: expirationDate)
        let nsKey = NSNumber(value: key.hashValue)
        cache.setObject(entry, forKey: nsKey)
    }
    
    func get(forKey key: Key) -> Value? {
        lock.lock()
        defer { lock.unlock() }
        
        let nsKey = NSNumber(value: key.hashValue)
        guard let entry = cache.object(forKey: nsKey) else {
            return nil
        }
        
        if entry.isExpired {
            cache.removeObject(forKey: nsKey)
            return nil
        }
        
        return entry.value
    }
    
    func remove(forKey key: Key) {
        lock.lock()
        defer { lock.unlock() }
        
        let nsKey = NSNumber(value: key.hashValue)
        cache.removeObject(forKey: nsKey)
    }
    
    func removeAll() {
        lock.lock()
        defer { lock.unlock() }
        
        cache.removeAllObjects()
    }
}

// Usage
let userCache = MemoryCache<String, User>()

// Store
userCache.set(user, forKey: user.id, ttl: 300)  // 5 minutes

// Retrieve
if let cachedUser = userCache.get(forKey: userId) {
    return cachedUser
}
```

### Multi-Layer Cache

```swift
class CachedAPIService {
    private let apiService: APIServiceProtocol
    private let memoryCache = MemoryCache<String, Data>()
    private let diskCache: DiskCache
    
    init(apiService: APIServiceProtocol, diskCache: DiskCache) {
        self.apiService = apiService
        self.diskCache = diskCache
    }
    
    func fetch<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T {
        let cacheKey = endpoint.path
        
        // 1. Try memory cache
        if let cachedData = memoryCache.get(forKey: cacheKey) {
            return try decode(data: cachedData)
        }
        
        // 2. Try disk cache
        if let cachedData = try? diskCache.data(forKey: cacheKey) {
            memoryCache.set(cachedData, forKey: cacheKey)
            return try decode(data: cachedData)
        }
        
        // 3. Fetch from network
        let data: Data = try await apiService.request(endpoint)
        
        // 4. Store in caches
        memoryCache.set(data, forKey: cacheKey)
        try? diskCache.setData(data, forKey: cacheKey)
        
        return try decode(data: data)
    }
    
    private func decode<T: Decodable>(data: Data) throws -> T {
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

---

## Authentication & Security

### Token-Based Auth

```swift
class AuthManager {
    static let shared = AuthManager()
    
    private let tokenKey = "auth_token"
    private let refreshTokenKey = "refresh_token"
    
    var token: String? {
        get {
            return KeychainManager.shared.get(key: tokenKey)
        }
        set {
            if let token = newValue {
                KeychainManager.shared.set(token, forKey: tokenKey)
            } else {
                KeychainManager.shared.delete(key: tokenKey)
            }
        }
    }
    
    var refreshToken: String? {
        get {
            return KeychainManager.shared.get(key: refreshTokenKey)
        }
        set {
            if let token = newValue {
                KeychainManager.shared.set(token, forKey: refreshTokenKey)
            } else {
                KeychainManager.shared.delete(key: refreshTokenKey)
            }
        }
    }
    
    func login(email: String, password: String) async throws -> AuthResponse {
        let endpoint = AuthEndpoint.login(email: email, password: password)
        let response: AuthResponse = try await apiService.request(endpoint)
        
        token = response.token
        refreshToken = response.refreshToken
        
        return response
    }
    
    func refreshAccessToken() async throws {
        guard let refreshToken = refreshToken else {
            throw AuthError.noRefreshToken
        }
        
        let endpoint = AuthEndpoint.refresh(token: refreshToken)
        let response: AuthResponse = try await apiService.request(endpoint)
        
        token = response.token
        self.refreshToken = response.refreshToken
    }
    
    func logout() {
        token = nil
        refreshToken = nil
    }
}
```

### Keychain Storage

```swift
import Security

class KeychainManager {
    static let shared = KeychainManager()
    
    func set(_ value: String, forKey key: String) {
        guard let data = value.data(using: .utf8) else { return }
        
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]
        
        // Delete existing
        SecItemDelete(query as CFDictionary)
        
        // Add new
        SecItemAdd(query as CFDictionary, nil)
    }
    
    func get(key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        guard status == errSecSuccess,
              let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            return nil
        }
        
        return value
    }
    
    func delete(key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        
        SecItemDelete(query as CFDictionary)
    }
}
```

### Request Interceptor (Token Refresh)

```swift
class AuthenticatedAPIService: APIServiceProtocol {
    private let baseService: APIServiceProtocol
    private let authManager: AuthManager
    
    init(baseService: APIServiceProtocol, authManager: AuthManager) {
        self.baseService = baseService
        self.authManager = authManager
    }
    
    func request<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T {
        do {
            return try await baseService.request(endpoint)
        } catch NetworkError.unauthorized {
            // Token expired, refresh and retry
            try await authManager.refreshAccessToken()
            return try await baseService.request(endpoint)
        }
    }
}
```

### SSL Pinning

```swift
class SSLPinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedCertificates: [Data]
    
    init(pinnedCertificates: [Data]) {
        self.pinnedCertificates = pinnedCertificates
    }
    
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Get server certificate
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data
        
        // Check if matches pinned certificates
        if pinnedCertificates.contains(serverCertificateData) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}

// Usage
let pinnedCert = /* Load from bundle */
let delegate = SSLPinningDelegate(pinnedCertificates: [pinnedCert])
let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)
```

### Privacy Manifest (iOS 17+)

Starting with iOS 17, Apple requires apps to declare their use of certain APIs through a privacy manifest file (`PrivacyInfo.xcprivacy`). This is crucial for networking code that collects data.

#### Required Reasons APIs for Networking

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>

    <key>NSPrivacyTrackingDomains</key>
    <array>
        <!-- List domains used for tracking if applicable -->
    </array>

    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeUserID</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>

    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string> <!-- Access user defaults for app functionality -->
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>35F9.1</string> <!-- Used for measuring network request timing -->
            </array>
        </dict>
    </array>
</dict>
</plist>
```

#### Common Networking-Related Privacy Requirements

```swift
// When using device identifiers in network requests
struct NetworkTracker {
    // ⚠️ Requires privacy manifest declaration
    func trackEvent(_ event: String) async {
        let deviceID = UIDevice.current.identifierForVendor?.uuidString

        // Must declare use of device identifier in privacy manifest
        let body = [
            "event": event,
            "device_id": deviceID,
            "timestamp": Date().timeIntervalSince1970
        ]

        // Send to analytics endpoint
        try? await apiService.post("/analytics", body: body)
    }
}

// When collecting user data
struct UserDataCollector {
    // ⚠️ Must declare in NSPrivacyCollectedDataTypes
    func collectUserData() -> UserData {
        return UserData(
            userId: getUserId(),          // NSPrivacyCollectedDataTypeUserID
            email: getEmail(),            // NSPrivacyCollectedDataTypeEmailAddress
            location: getLocation(),      // NSPrivacyCollectedDataTypeCoarseLocation
            deviceInfo: getDeviceInfo()   // NSPrivacyCollectedDataTypeDeviceID
        )
    }
}
```

#### Third-Party SDK Compliance

```swift
// Check third-party SDKs for privacy manifest requirements
class NetworkingSDKManager {
    func validateSDKPrivacyCompliance() {
        // Common networking SDKs that require privacy manifests:
        // - Analytics SDKs (Firebase, Amplitude, Mixpanel)
        // - Crash reporting (Crashlytics, Sentry)
        // - Ad networks (AdMob, Facebook Ads)
        // - Social SDKs (Facebook, Twitter)

        // Example: Ensure Firebase includes its privacy manifest
        #if canImport(FirebaseAnalytics)
        // Firebase SDK must include its own PrivacyInfo.xcprivacy
        // Your app inherits these requirements
        #endif
    }
}
```

#### Best Practices for Privacy Compliance

```swift
// 1. Minimize data collection
struct MinimalDataNetworkClient {
    func makeRequest() async throws {
        // ✅ Good: Only collect necessary data
        let request = APIRequest(
            endpoint: "/api/data",
            headers: ["Content-Type": "application/json"]
        )

        // ❌ Avoid: Unnecessary data collection
        // let request = APIRequest(
        //     endpoint: "/api/data",
        //     headers: [
        //         "X-Device-ID": UIDevice.current.identifierForVendor,
        //         "X-User-Location": getCurrentLocation(),
        //         "X-Contacts-Count": getContactsCount()
        //     ]
        // )
    }
}

// 2. Declare all API usage accurately
extension NetworkManager {
    // If using UserDefaults for caching
    func cacheResponse(_ data: Data, for key: String) {
        // ⚠️ Requires NSPrivacyAccessedAPICategoryUserDefaults
        UserDefaults.standard.set(data, forKey: key)
    }

    // If measuring timing
    func measureRequestDuration() -> TimeInterval {
        // ⚠️ Requires NSPrivacyAccessedAPICategorySystemBootTime
        let bootTime = ProcessInfo.processInfo.systemUptime
        return bootTime
    }
}

// 3. Document privacy practices
/// NetworkClient handles all API communication
/// Privacy: This class collects user ID for authentication only.
/// No tracking or third-party sharing occurs.
class NetworkClient {
    // Implementation
}
```

---

## Data Persistence

### UserDefaults

```swift
// Simple values
UserDefaults.standard.set("John", forKey: "username")
let username = UserDefaults.standard.string(forKey: "username")

// Custom objects (Codable)
struct Settings: Codable {
    let darkMode: Bool
    let fontSize: Int
}

let settings = Settings(darkMode: true, fontSize: 14)

// Save
let encoder = JSONEncoder()
if let encoded = try? encoder.encode(settings) {
    UserDefaults.standard.set(encoded, forKey: "settings")
}

// Load
if let data = UserDefaults.standard.data(forKey: "settings") {
    let decoder = JSONDecoder()
    if let decoded = try? decoder.decode(Settings.self, from: data) {
        // Use decoded settings
    }
}

// Property wrapper for UserDefaults
@propertyWrapper
struct UserDefault<T: Codable> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get {
            guard let data = UserDefaults.standard.data(forKey: key) else {
                return defaultValue
            }
            return (try? JSONDecoder().decode(T.self, from: data)) ?? defaultValue
        }
        set {
            let data = try? JSONEncoder().encode(newValue)
            UserDefaults.standard.set(data, forKey: key)
        }
    }
}

// Usage
struct AppSettings {
    @UserDefault(key: "username", defaultValue: "Guest")
    var username: String
    
    @UserDefault(key: "isDarkMode", defaultValue: false)
    var isDarkMode: Bool
}
```

### Core Data Basics

```swift
// ⚠️ Core Data is complex - this is simplified

import CoreData

class CoreDataManager {
    static let shared = CoreDataManager()
    
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "MyApp")
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Core Data failed to load: \(error)")
            }
        }
        return container
    }()
    
    var context: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
    
    func saveContext() {
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                print("Failed to save: \(error)")
            }
        }
    }
}

// Create
func createUser(name: String, email: String) {
    let context = CoreDataManager.shared.context
    let user = UserEntity(context: context)
    user.id = UUID()
    user.name = name
    user.email = email
    
    CoreDataManager.shared.saveContext()
}

// Read
func fetchUsers() -> [UserEntity] {
    let context = CoreDataManager.shared.context
    let fetchRequest: NSFetchRequest<UserEntity> = UserEntity.fetchRequest()
    
    do {
        return try context.fetch(fetchRequest)
    } catch {
        print("Failed to fetch: \(error)")
        return []
    }
}

// Update
func updateUser(_ user: UserEntity, name: String) {
    user.name = name
    CoreDataManager.shared.saveContext()
}

// Delete
func deleteUser(_ user: UserEntity) {
    let context = CoreDataManager.shared.context
    context.delete(user)
    CoreDataManager.shared.saveContext()
}
```

### File System Storage

```swift
class FileStorageManager {
    static let shared = FileStorageManager()
    
    private let fileManager = FileManager.default
    
    private var documentsDirectory: URL {
        return fileManager.urls(for: .documentDirectory, in: .userDomainMask)[0]
    }
    
    func save<T: Encodable>(_ object: T, to filename: String) throws {
        let url = documentsDirectory.appendingPathComponent(filename)
        let encoder = JSONEncoder()
        let data = try encoder.encode(object)
        try data.write(to: url)
    }
    
    func load<T: Decodable>(from filename: String) throws -> T {
        let url = documentsDirectory.appendingPathComponent(filename)
        let data = try Data(contentsOf: url)
        let decoder = JSONDecoder()
        return try decoder.decode(T.self, from: data)
    }
    
    func delete(filename: String) throws {
        let url = documentsDirectory.appendingPathComponent(filename)
        try fileManager.removeItem(at: url)
    }
    
    func fileExists(filename: String) -> Bool {
        let url = documentsDirectory.appendingPathComponent(filename)
        return fileManager.fileExists(atPath: url.path)
    }
}

// Usage
let users = [User(id: "1", name: "John", email: "john@example.com")]
try? FileStorageManager.shared.save(users, to: "users.json")

let loadedUsers: [User]? = try? FileStorageManager.shared.load(from: "users.json")
```

---

## Image Loading & Caching

### Basic Image Loading

```swift
func loadImage(from url: URL) async throws -> UIImage {
    let (data, _) = try await URLSession.shared.data(from: url)
    
    guard let image = UIImage(data: data) else {
        throw ImageError.invalidData
    }
    
    return image
}

// Usage in UIImageView
Task {
    if let image = try? await loadImage(from: url) {
        imageView.image = image
    }
}
```

### Image Cache

```swift
class ImageCache {
    static let shared = ImageCache()
    
    private let cache = NSCache<NSURL, UIImage>()
    private let fileManager = FileManager.default
    
    private var cacheDirectory: URL {
        let urls = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)
        return urls[0].appendingPathComponent("ImageCache")
    }
    
    init() {
        // Create cache directory
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        
        // Set cache limits
        cache.countLimit = 100  // Max 100 images
        cache.totalCostLimit = 50 * 1024 * 1024  // 50 MB
    }
    
    func image(for url: URL) -> UIImage? {
        // Check memory cache
        if let cached = cache.object(forKey: url as NSURL) {
            return cached
        }
        
        // Check disk cache
        let filename = url.lastPathComponent
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        if let data = try? Data(contentsOf: fileURL),
           let image = UIImage(data: data) {
            // Store in memory cache
            cache.setObject(image, forKey: url as NSURL)
            return image
        }
        
        return nil
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        // Store in memory cache
        cache.setObject(image, forKey: url as NSURL)
        
        // Store in disk cache
        let filename = url.lastPathComponent
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        if let data = image.jpegData(compressionQuality: 0.8) {
            try? data.write(to: fileURL)
        }
    }
    
    func clearCache() {
        cache.removeAllObjects()
        try? fileManager.removeItem(at: cacheDirectory)
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
    }
}
```

### Image Loader with Cache

```swift
class ImageLoader {
    static let shared = ImageLoader()
    
    private var tasks: [URL: Task<UIImage, Error>] = [:]
    
    func loadImage(from url: URL) async throws -> UIImage {
        // Check cache first
        if let cachedImage = ImageCache.shared.image(for: url) {
            return cachedImage
        }
        
        // Check if already loading
        if let existingTask = tasks[url] {
            return try await existingTask.value
        }
        
        // Create new task
        let task = Task<UIImage, Error> {
            let (data, _) = try await URLSession.shared.data(from: url)
            
            guard let image = UIImage(data: data) else {
                throw ImageError.invalidData
            }
            
            // Cache the image
            ImageCache.shared.setImage(image, for: url)
            
            return image
        }
        
        tasks[url] = task
        
        defer {
            tasks[url] = nil
        }
        
        return try await task.value
    }
}

// Usage in cell
class ImageCell: UITableViewCell {
    @IBOutlet weak var cellImageView: UIImageView!
    
    private var loadTask: Task<Void, Never>?
    
    func configure(with imageURL: URL) {
        cellImageView.image = nil
        
        // Cancel previous load
        loadTask?.cancel()
        
        loadTask = Task {
            if let image = try? await ImageLoader.shared.loadImage(from: imageURL) {
                // Check if cell wasn't reused
                guard !Task.isCancelled else { return }
                
                await MainActor.run {
                    self.cellImageView.image = image
                }
            }
        }
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        loadTask?.cancel()
        cellImageView.image = nil
    }
}
```

---

## Offline Support

### Network Reachability

```swift
import Network

class NetworkMonitor {
    static let shared = NetworkMonitor()
    
    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "NetworkMonitor")
    
    @Published private(set) var isConnected = true
    @Published private(set) var connectionType: ConnectionType = .unknown
    
    enum ConnectionType {
        case wifi
        case cellular
        case ethernet
        case unknown
    }
    
    private init() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async {
                self?.isConnected = path.status == .satisfied
                
                if path.usesInterfaceType(.wifi) {
                    self?.connectionType = .wifi
                } else if path.usesInterfaceType(.cellular) {
                    self?.connectionType = .cellular
                } else if path.usesInterfaceType(.wiredEthernet) {
                    self?.connectionType = .ethernet
                } else {
                    self?.connectionType = .unknown
                }
            }
        }
        
        monitor.start(queue: queue)
    }
}

// Usage
if NetworkMonitor.shared.isConnected {
    // Fetch from network
} else {
    // Use cached data
}
```

### Offline-First Architecture

```swift
protocol UserRepositoryProtocol {
    func fetchUsers() async throws -> [User]
}

class UserRepository: UserRepositoryProtocol {
    private let apiService: APIServiceProtocol
    private let localDatabase: LocalDatabase
    private let networkMonitor: NetworkMonitor
    
    func fetchUsers() async throws -> [User] {
        if networkMonitor.isConnected {
            // Fetch from API
            do {
                let users = try await apiService.fetchUsers()
                
                // Save to local database
                try await localDatabase.save(users)
                
                return users
            } catch {
                // Network failed, fallback to local
                return try await localDatabase.fetchUsers()
            }
        } else {
            // Offline, use local database
            return try await localDatabase.fetchUsers()
        }
    }
}
```

### Sync Queue for Offline Operations

```swift
struct PendingOperation: Codable {
    let id: UUID
    let type: OperationType
    let data: Data
    let timestamp: Date
    
    enum OperationType: String, Codable {
        case create
        case update
        case delete
    }
}

class SyncManager {
    static let shared = SyncManager()
    
    private var pendingOperations: [PendingOperation] = []
    private let queueKey = "pending_operations"
    
    init() {
        loadQueue()
        observeNetworkChanges()
    }
    
    func addOperation(_ operation: PendingOperation) {
        pendingOperations.append(operation)
        saveQueue()
        
        if NetworkMonitor.shared.isConnected {
            Task {
                await syncPendingOperations()
            }
        }
    }
    
    private func syncPendingOperations() async {
        guard NetworkMonitor.shared.isConnected else { return }
        
        for operation in pendingOperations {
            do {
                try await execute(operation)
                
                // Remove from queue on success
                pendingOperations.removeAll { $0.id == operation.id }
                saveQueue()
            } catch {
                print("Failed to sync operation: \(error)")
                // Keep in queue for retry
            }
        }
    }
    
    private func execute(_ operation: PendingOperation) async throws {
        // Execute the operation via API
    }
    
    private func loadQueue() {
        if let data = UserDefaults.standard.data(forKey: queueKey) {
            pendingOperations = (try? JSONDecoder().decode([PendingOperation].self, from: data)) ?? []
        }
    }
    
    private func saveQueue() {
        if let data = try? JSONEncoder().encode(pendingOperations) {
            UserDefaults.standard.set(data, forKey: queueKey)
        }
    }
    
    private func observeNetworkChanges() {
        // When network becomes available, sync
        NotificationCenter.default.addObserver(
            forName: .networkBecameAvailable,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            Task {
                await self?.syncPendingOperations()
            }
        }
    }
}
```

---

## Interview Questions

### Q: "Explain the difference between URLSession data task, download task, and upload task"

**A:**
- **Data task**: Retrieves data into memory. Good for API requests, small files. Data is lost if app terminates.
- **Download task**: Saves directly to disk. Good for large files, videos. Can resume after app termination with background session.
- **Upload task**: Sends data to server. Supports background uploads. Can resume after app termination.

Background sessions persist across app launches, regular sessions don't.

### Q: "How would you implement a networking layer for a large app?"

**A:** Key components:
1. **Protocol-based**: Define APIEndpoint protocol for endpoints
2. **Generic service**: APIService that handles any endpoint
3. **Error handling**: Custom error types with proper categorization
4. **Authentication**: Interceptor pattern for token refresh
5. **Caching**: Multi-layer (memory → disk → network)
6. **Dependency injection**: Testable with mock services
7. **Type safety**: Codable for parsing, strong types everywhere

### Q: "How do you handle offline support?"

**A:**
1. **Network monitoring**: NWPathMonitor to detect connectivity
2. **Local database**: Core Data or SQLite for offline storage
3. **Repository pattern**: Abstract data source (API vs local)
4. **Sync queue**: Queue operations when offline, sync when online
5. **Cache strategy**: Aggressive caching with TTL
6. **UI feedback**: Show offline banner, disable features gracefully

### Q: "What's your approach to image caching?"

**A:**
1. **Two-tier cache**: Memory (NSCache) + Disk (FileManager)
2. **Memory limits**: Set countLimit and totalCostLimit on NSCache
3. **Request deduplication**: Don't load same image multiple times
4. **Cell reuse**: Cancel tasks in prepareForReuse()
5. **Placeholder**: Show placeholder while loading
6. **Error handling**: Show error state if load fails

Consider using SDWebImage or Kingfisher in production.

### Q: "How do you secure API calls?"

**A:**
1. **HTTPS only**: Never use HTTP
2. **Token auth**: Bearer tokens, not credentials in each request
3. **Keychain storage**: Store tokens in Keychain, not UserDefaults
4. **SSL pinning**: Pin certificates for critical APIs
5. **Token refresh**: Automatic refresh when 401 received
6. **Request signing**: HMAC signatures for sensitive operations
7. **Rate limiting**: Client-side throttling to prevent abuse

### Q: "Explain Codable and how you'd handle complex JSON"

**A:** Codable = Encodable + Decodable. Swift automatically synthesizes encoding/decoding.

For complex JSON:
- **CodingKeys**: Map different property names
- **Custom init(from:)**: Handle nested structures, default values
- **Associated types**: Polymorphic decoding based on type field
- **KeyDecodingStrategy**: convertFromSnakeCase for API consistency
- **DateDecodingStrategy**: Handle different date formats

### Q: "What's the difference between synchronous and asynchronous networking?"

**A:**
- **Synchronous**: Blocks thread until complete. Never use on main thread - freezes UI.
- **Asynchronous**: Returns immediately, calls completion handler when done. Always use for networking.

Modern approach is async/await which looks synchronous but is asynchronous under the hood.

---

This covers everything you need to know about networking and data for iOS interviews. Focus on async/await, caching strategies, and offline support - these are critical topics.
