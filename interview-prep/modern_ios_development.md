---
layout: page
title: "Modern iOS Development Guide"
subtitle: "SwiftUI, Combine, async/await & Latest Best Practices"
---

# Modern iOS Development Guide
## SwiftUI, Combine, async/await & Latest Best Practices

---

## Table of Contents
1. [SwiftUI Fundamentals](#swiftui-fundamentals)
2. [Combine Framework](#combine-framework)
3. [async/await & Structured Concurrency](#asyncawait--structured-concurrency)
4. [Modern App Architecture](#modern-app-architecture)
5. [New iOS APIs & Features](#new-ios-apis--features)
6. [SwiftUI vs UIKit - When to Use What](#swiftui-vs-uikit---when-to-use-what)
7. [Modern Development Tools](#modern-development-tools)
8. [Best Practices 2025](#best-practices-2025)
9. [Migration Strategies](#migration-strategies)
10. [Interview Questions](#interview-questions)

---

## SwiftUI Fundamentals

### Basic View Structure

```swift
import SwiftUI

// Simple view
struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
            .font(.title)
            .foregroundColor(.blue)
    }
}

// View with state
struct CounterView: View {
    @State private var count = 0
    
    var body: some View {
        VStack {
            Text("Count: \(count)")
                .font(.largeTitle)
            
            HStack {
                Button("-") {
                    count -= 1
                }
                
                Button("+") {
                    count += 1
                }
            }
        }
        .padding()
    }
}
```

### SwiftUI Layout

```swift
// VStack - Vertical stack
VStack(alignment: .leading, spacing: 10) {
    Text("Title")
    Text("Subtitle")
}

// HStack - Horizontal stack
HStack {
    Image(systemName: "star.fill")
    Text("Rating")
}

// ZStack - Layered stack
ZStack {
    Color.blue
    Text("Overlay")
        .foregroundColor(.white)
}

// LazyVStack - Lazy loading (better performance)
ScrollView {
    LazyVStack {
        ForEach(0..<1000) { i in
            Text("Row \(i)")
        }
    }
}
```

### Lists & Navigation

```swift
struct UserListView: View {
    let users = [
        User(name: "John", email: "john@example.com"),
        User(name: "Jane", email: "jane@example.com")
    ]
    
    var body: some View {
        NavigationView {
            List(users) { user in
                NavigationLink(destination: UserDetailView(user: user)) {
                    HStack {
                        Image(systemName: "person.circle.fill")
                        VStack(alignment: .leading) {
                            Text(user.name)
                                .font(.headline)
                            Text(user.email)
                                .font(.subheadline)
                                .foregroundColor(.gray)
                        }
                    }
                }
            }
            .navigationTitle("Users")
            .navigationBarTitleDisplayMode(.large)
        }
    }
}
```

### State Management in SwiftUI

```swift
// @State - Simple value types, view-local
struct ExampleView: View {
    @State private var username = ""
    
    var body: some View {
        TextField("Username", text: $username)
    }
}

// @Binding - Two-way connection to parent
struct ChildView: View {
    @Binding var text: String
    
    var body: some View {
        TextField("Enter text", text: $text)
    }
}

struct ParentView: View {
    @State private var text = ""
    
    var body: some View {
        ChildView(text: $text)
    }
}

// @StateObject - Create and own ObservableObject
struct ContentView: View {
    @StateObject private var viewModel = ContentViewModel()
    
    var body: some View {
        Text(viewModel.data)
            .onAppear {
                viewModel.loadData()
            }
    }
}

// @ObservedObject - Observe passed-in object
struct DetailView: View {
    @ObservedObject var viewModel: DetailViewModel
    
    var body: some View {
        Text(viewModel.title)
    }
}

// @EnvironmentObject - Shared across view hierarchy
@main
struct MyApp: App {
    @StateObject private var settings = AppSettings()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(settings)
        }
    }
}

struct SomeDeepView: View {
    @EnvironmentObject var settings: AppSettings
    
    var body: some View {
        Toggle("Dark Mode", isOn: $settings.isDarkMode)
    }
}
```

### Modern SwiftUI Features

```swift
// iOS 17+ - @Observable (replaces ObservableObject)
@Observable
class UserViewModel {
    var users: [User] = []
    var isLoading = false
    
    func loadUsers() async {
        isLoading = true
        users = try? await fetchUsers()
        isLoading = false
    }
}

// Usage - no property wrappers needed!
struct ContentView: View {
    var viewModel = UserViewModel()
    
    var body: some View {
        List(viewModel.users) { user in
            Text(user.name)
        }
        .task {
            await viewModel.loadUsers()
        }
    }
}

// SwiftUI Charts (iOS 16+)
import Charts

struct SalesChart: View {
    let data = [
        (month: "Jan", sales: 100),
        (month: "Feb", sales: 150),
        (month: "Mar", sales: 120)
    ]
    
    var body: some View {
        Chart(data, id: \.month) { item in
            BarMark(
                x: .value("Month", item.month),
                y: .value("Sales", item.sales)
            )
        }
    }
}

// NavigationStack (iOS 16+) - Modern navigation
struct ModernNavigation: View {
    @State private var path = NavigationPath()
    
    var body: some View {
        NavigationStack(path: $path) {
            List(items) { item in
                NavigationLink(value: item) {
                    Text(item.name)
                }
            }
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
        }
    }
}

// Scrolling improvements (iOS 17+)
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item)
                .scrollTransition { content, phase in
                    content
                        .opacity(phase.isIdentity ? 1 : 0.5)
                        .scaleEffect(phase.isIdentity ? 1 : 0.95)
                }
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.viewAligned)
```

### Custom Views & Modifiers

```swift
// Custom View
struct CardView<Content: View>: View {
    let content: Content
    
    init(@ViewBuilder content: () -> Content) {
        self.content = content()
    }
    
    var body: some View {
        content
            .padding()
            .background(Color.white)
            .cornerRadius(10)
            .shadow(radius: 5)
    }
}

// Usage
CardView {
    Text("Hello")
}

// Custom ViewModifier
struct RoundedButton: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(Color.blue)
            .foregroundColor(.white)
            .cornerRadius(10)
    }
}

extension View {
    func roundedButton() -> some View {
        modifier(RoundedButton())
    }
}

// Usage
Button("Tap Me") { }
    .roundedButton()
```

---

## Combine Framework

### Basic Publishers

```swift
import Combine

// Simple publisher
let numbers = [1, 2, 3, 4, 5].publisher

numbers.sink { value in
    print(value)
}

// Just - Publishes single value
Just(42)
    .sink { print($0) }

// Future - Async single value
Future<String, Error> { promise in
    DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
        promise(.success("Done"))
    }
}
.sink(
    receiveCompletion: { _ in },
    receiveValue: { print($0) }
)
```

### Operators

```swift
var cancellables = Set<AnyCancellable>()

// Map - Transform values
[1, 2, 3].publisher
    .map { $0 * 2 }
    .sink { print($0) }  // 2, 4, 6
    .store(in: &cancellables)

// Filter
[1, 2, 3, 4, 5].publisher
    .filter { $0 % 2 == 0 }
    .sink { print($0) }  // 2, 4
    .store(in: &cancellables)

// Debounce - Wait for pause
searchTextField.textPublisher
    .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
    .removeDuplicates()
    .sink { searchText in
        performSearch(searchText)
    }
    .store(in: &cancellables)

// CombineLatest - Combine multiple publishers
Publishers.CombineLatest(
    usernamePublisher,
    passwordPublisher
)
.map { username, password in
    username.count > 3 && password.count > 6
}
.assign(to: &$isFormValid)

// FlatMap - Transform and flatten
userIDPublisher
    .flatMap { userID in
        fetchUserDetails(id: userID)
    }
    .sink { user in
        print(user.name)
    }
    .store(in: &cancellables)

// Retry
networkPublisher
    .retry(3)
    .catch { error -> Just<Data> in
        return Just(Data())
    }
    .sink { data in
        process(data)
    }
    .store(in: &cancellables)
```

### Real-World Example

```swift
class SearchViewModel: ObservableObject {
    @Published var searchText = ""
    @Published var results: [SearchResult] = []
    @Published var isLoading = false
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .map { [weak self] query -> AnyPublisher<[SearchResult], Never> in
                self?.isLoading = true
                
                return self?.searchService.search(query: query)
                    .catch { _ in Just([]) }
                    .eraseToAnyPublisher() ?? Just([]).eraseToAnyPublisher()
            }
            .switchToLatest()  // Cancel previous search
            .receive(on: DispatchQueue.main)
            .sink { [weak self] results in
                self?.results = results
                self?.isLoading = false
            }
            .store(in: &cancellables)
    }
}
```

---

## async/await & Structured Concurrency

### Basic async/await

```swift
// Define async function
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Call async function
Task {
    do {
        let user = try await fetchUser(id: "123")
        print(user.name)
    } catch {
        print("Error: \(error)")
    }
}

// In SwiftUI
struct ContentView: View {
    @State private var user: User?
    
    var body: some View {
        Text(user?.name ?? "Loading...")
            .task {
                user = try? await fetchUser(id: "123")
            }
    }
}
```

### Structured Concurrency

```swift
// Sequential (slow)
func loadDataSequential() async throws {
    let users = try await fetchUsers()
    let posts = try await fetchPosts()
    let comments = try await fetchComments()
    // Total: 3 seconds if each takes 1 second
}

// Parallel (fast)
func loadDataParallel() async throws {
    async let users = fetchUsers()
    async let posts = fetchPosts()
    async let comments = fetchComments()
    
    let (u, p, c) = try await (users, posts, comments)
    // Total: 1 second (all execute simultaneously)
}

// Task Groups - Dynamic parallelism
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

### Task Management

```swift
// Creating tasks
let task = Task {
    let data = try await fetchData()
    return data
}

// Get result
let data = try await task.value

// Cancel task
task.cancel()

// Check cancellation
Task {
    for i in 0..<100 {
        try Task.checkCancellation()  // Throws if cancelled
        // or
        guard !Task.isCancelled else { return }
        
        await processItem(i)
    }
}

// Detached task (doesn't inherit context)
Task.detached(priority: .background) {
    // Independent execution
}

// Task priority
Task(priority: .high) {
    // High priority work
}
```

### Actors - Thread-Safe State

```swift
// Actor automatically serializes access
actor BankAccount {
    private var balance: Double = 0
    
    func deposit(amount: Double) {
        balance += amount
    }
    
    func withdraw(amount: Double) -> Bool {
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }
    
    func getBalance() -> Double {
        return balance
    }
}

// Usage (automatically async)
let account = BankAccount()

Task {
    await account.deposit(amount: 100)
    let balance = await account.getBalance()
    print("Balance: \(balance)")
}

// @MainActor - Ensure main thread execution
@MainActor
class ViewModel: ObservableObject {
    @Published var users: [User] = []
    
    func loadUsers() async {
        // Automatically runs on main thread
        users = try? await fetchUsers()
    }
}
```

### async/await in View Models

```swift
@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let repository: UserRepository
    
    init(repository: UserRepository = UserRepository()) {
        self.repository = repository
    }
    
    func loadUsers() async {
        isLoading = true
        errorMessage = nil
        
        do {
            users = try await repository.fetchUsers()
        } catch {
            errorMessage = error.localizedDescription
        }
        
        isLoading = false
    }
    
    func refreshUsers() async {
        await loadUsers()
    }
}

// Usage in SwiftUI
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()
    
    var body: some View {
        List(viewModel.users) { user in
            Text(user.name)
        }
        .task {
            await viewModel.loadUsers()
        }
        .refreshable {
            await viewModel.refreshUsers()
        }
    }
}
```

---

## Modern App Architecture

### MVVM with SwiftUI & async/await

```swift
// Model
struct Movie: Identifiable, Codable {
    let id: String
    let title: String
    let posterURL: String
}

// Repository (Data Layer)
protocol MovieRepository {
    func fetchMovies() async throws -> [Movie]
    func searchMovies(query: String) async throws -> [Movie]
}

class DefaultMovieRepository: MovieRepository {
    private let apiService: APIService
    private let cache: CacheService
    
    init(apiService: APIService, cache: CacheService) {
        self.apiService = apiService
        self.cache = cache
    }
    
    func fetchMovies() async throws -> [Movie] {
        if let cached = cache.get(key: "movies") as? [Movie] {
            return cached
        }
        
        let movies = try await apiService.fetchMovies()
        cache.set(movies, forKey: "movies")
        return movies
    }
    
    func searchMovies(query: String) async throws -> [Movie] {
        return try await apiService.searchMovies(query: query)
    }
}

// ViewModel
@MainActor
class MovieListViewModel: ObservableObject {
    @Published var movies: [Movie] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var searchQuery = ""
    
    private let repository: MovieRepository
    private var searchTask: Task<Void, Never>?
    
    init(repository: MovieRepository) {
        self.repository = repository
        observeSearchQuery()
    }
    
    func loadMovies() async {
        isLoading = true
        errorMessage = nil
        
        do {
            movies = try await repository.fetchMovies()
        } catch {
            errorMessage = error.localizedDescription
        }
        
        isLoading = false
    }
    
    private func observeSearchQuery() {
        // Debounce search
        Task {
            for await query in $searchQuery.values {
                await performSearch(query: query)
            }
        }
    }
    
    private func performSearch(query: String) async {
        searchTask?.cancel()
        
        guard !query.isEmpty else {
            await loadMovies()
            return
        }
        
        searchTask = Task {
            try? await Task.sleep(nanoseconds: 300_000_000)  // 300ms debounce
            
            guard !Task.isCancelled else { return }
            
            isLoading = true
            do {
                movies = try await repository.searchMovies(query: query)
            } catch {
                errorMessage = error.localizedDescription
            }
            isLoading = false
        }
    }
}

// View
struct MovieListView: View {
    @StateObject private var viewModel: MovieListViewModel
    
    init(repository: MovieRepository = DefaultMovieRepository()) {
        _viewModel = StateObject(wrappedValue: MovieListViewModel(repository: repository))
    }
    
    var body: some View {
        NavigationStack {
            Group {
                if viewModel.isLoading {
                    ProgressView()
                } else if let error = viewModel.errorMessage {
                    ErrorView(message: error) {
                        Task { await viewModel.loadMovies() }
                    }
                } else {
                    moviesList
                }
            }
            .navigationTitle("Movies")
            .searchable(text: $viewModel.searchQuery)
            .task {
                await viewModel.loadMovies()
            }
        }
    }
    
    private var moviesList: some View {
        List(viewModel.movies) { movie in
            NavigationLink(value: movie) {
                MovieRow(movie: movie)
            }
        }
        .navigationDestination(for: Movie.self) { movie in
            MovieDetailView(movie: movie)
        }
    }
}
```

---

## New iOS APIs & Features

### iOS 16+ Features

```swift
// 1. NavigationStack
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            Text(item.name)
        }
    }
    .navigationDestination(for: Item.self) { item in
        DetailView(item: item)
    }
}

// 2. ShareLink
ShareLink(item: url) {
    Label("Share", systemImage: "square.and.arrow.up")
}

// 3. Charts
import Charts

Chart(data) { item in
    LineMark(
        x: .value("Date", item.date),
        y: .value("Value", item.value)
    )
}

// 4. Layout Protocol (Custom layouts)
struct WaterfallLayout: Layout {
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        // Calculate size
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        // Position subviews
    }
}
```

### iOS 17+ Features

```swift
// 1. @Observable (replaces ObservableObject)
@Observable
class ViewModel {
    var count = 0
    
    func increment() {
        count += 1
    }
}

// Usage - no @StateObject/@ObservedObject needed
struct ContentView: View {
    let viewModel = ViewModel()
    
    var body: some View {
        Button("Count: \(viewModel.count)") {
            viewModel.increment()
        }
    }
}

// 2. Swift Data (replaces Core Data)
import SwiftData

@Model
class Movie {
    var title: String
    var releaseDate: Date
    
    init(title: String, releaseDate: Date) {
        self.title = title
        self.releaseDate = releaseDate
    }
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: Movie.self)
    }
}

// 3. #Preview macro (replaces PreviewProvider)
#Preview {
    ContentView()
}

#Preview("Dark Mode") {
    ContentView()
        .preferredColorScheme(.dark)
}

// 4. Observation in view models
@Observable
class UserViewModel {
    var users: [User] = []
    var isLoading = false
    
    func loadUsers() async {
        isLoading = true
        users = try? await fetchUsers()
        isLoading = false
    }
}
```

### Widgets (iOS 14+)

```swift
import WidgetKit
import SwiftUI

struct SimpleWidget: Widget {
    let kind = "SimpleWidget"
    
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            SimpleWidgetView(entry: entry)
        }
        .configurationDisplayName("My Widget")
        .description("This is my widget")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}

struct SimpleWidgetView: View {
    var entry: Provider.Entry
    
    var body: some View {
        VStack {
            Text(entry.date, style: .time)
            Text("Widget Content")
        }
    }
}
```

### iOS 18 & Swift 6.0 Features (2024-2025)

```swift
// iOS 18 Features

// 1. Control Center Widget API
import WidgetKit
import AppIntents

struct MusicControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        AppIntentControlConfiguration(
            kind: "com.example.music-control",
            provider: MusicControlProvider()
        ) { config in
            ControlWidgetButton(action: PlayPauseIntent()) {
                Label("Play/Pause", systemImage: "playpause.fill")
            }
        }
    }
}

// 2. Enhanced App Intents
struct AddTaskIntent: AppIntent {
    static let title: LocalizedStringResource = "Add Task"

    @Parameter(title: "Task Title")
    var taskTitle: String

    @Parameter(title: "Due Date")
    var dueDate: Date?

    func perform() async throws -> some IntentResult {
        let task = await TaskManager.shared.addTask(
            title: taskTitle,
            dueDate: dueDate
        )
        return .result(value: task)
    }
}

// 3. SwiftUI Animation Enhancements
struct AnimationView: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            // New keyframe animator
            Text("Swift 6.0")
                .keyframeAnimator(
                    initialValue: AnimationValues(),
                    repeating: true
                ) { content, value in
                    content
                        .scaleEffect(value.scale)
                        .rotationEffect(value.rotation)
                } keyframes: { _ in
                    KeyframeTrack(\.scale) {
                        LinearKeyframe(1.2, duration: 0.5)
                        SpringKeyframe(1.0, spring: .bouncy)
                    }
                }

            // SF Symbols 6 animations
            Image(systemName: "heart.fill")
                .symbolEffect(.bounce, value: isExpanded)
                .symbolEffect(.variableColor.iterative)
        }
    }
}

// Swift 6.0 Language Features

// 1. Complete Concurrency Checking
// Enable: SWIFT_STRICT_CONCURRENCY = complete
actor DataManager {
    private var cache: [String: Data] = [:]

    func store(_ data: Data, key: String) {
        cache[key] = data
    }

    func process(_ items: [any Sendable]) async {
        // Enforces Sendable checking
    }
}

// 2. Typed Throws
enum NetworkError: Error {
    case timeout
    case invalidResponse
    case serverError(Int)
}

func fetchData() throws(NetworkError) -> Data {
    throw NetworkError.timeout
}

// Usage
do {
    let data = try fetchData()
} catch let error: NetworkError {
    switch error {
    case .timeout:
        print("Timeout")
    case .invalidResponse:
        print("Invalid")
    case .serverError(let code):
        print("Error: \(code)")
    }
}

// 3. Noncopyable Types
@noncopyable
struct FileHandle {
    private let descriptor: Int32

    init(path: String) {
        descriptor = open(path, O_RDONLY)
    }

    deinit {
        close(descriptor)
    }

    consuming func transfer() -> FileHandle {
        return self
    }
}

// 4. Pack Iteration
func processMultiple<each T>(_ items: repeat each T) {
    repeat print(each items)
}

// 5. Enhanced @MainActor
@MainActor
class ViewModel {
    var text = ""

    func updateUI() {
        text = "Updated"
    }

    nonisolated func backgroundWork() async {
        let result = await compute()
        await MainActor.run {
            text = result
        }
    }
}

// 6. Live Activities Push-to-Start
import ActivityKit

func startLiveActivityViaPush() async {
    let attributes = DeliveryActivity(orderNumber: "12345")

    let activity = try? Activity.request(
        attributes: attributes,
        content: .init(state: .init(
            status: "Preparing",
            time: Date()
        ), staleDate: nil),
        pushType: .token  // iOS 18
    )

    if let token = activity?.pushToken {
        await registerToken(token)
    }
}
```

---

## SwiftUI vs UIKit - When to Use What

### Decision Matrix

| Feature | SwiftUI | UIKit |
|---------|---------|-------|
| New apps | ✅ Preferred | ⚠️ If needed |
| Rapid prototyping | ✅ Fast | ❌ Slower |
| Complex animations | ⚠️ Limited | ✅ Full control |
| Custom drawing | ⚠️ Basic | ✅ Advanced |
| iOS version support | iOS 13+ | All versions |
| Learning curve | Medium | High |
| Code volume | Less | More |
| Performance | Good | Excellent |
| Third-party libs | Growing | Mature |

### When to Use SwiftUI

✅ **Use SwiftUI when:**
- Starting new project (iOS 15+)
- Building forms and settings screens
- Creating simple to medium complexity UIs
- Rapid prototyping
- Cross-platform (iOS/macOS/watchOS)
- Team comfortable with declarative UI

### When to Use UIKit

✅ **Use UIKit when:**
- Need iOS 12 or earlier support
- Complex custom animations
- Advanced CALayer manipulation
- Pixel-perfect control needed
- Existing large UIKit codebase
- Third-party lib requires it
- Performance-critical rendering

### Hybrid Approach (Best for Most Apps)

```swift
// Use SwiftUI for new screens
struct NewFeatureView: View {
    var body: some View {
        // SwiftUI code
    }
}

// Wrap UIKit when needed
struct CustomUIKitView: UIViewRepresentable {
    func makeUIView(context: Context) -> UITableView {
        let tableView = UITableView()
        // Configure complex UIKit view
        return tableView
    }
    
    func updateUIView(_ uiView: UITableView, context: Context) {
        // Update when SwiftUI state changes
    }
}

// Use in SwiftUI
struct ContentView: View {
    var body: some View {
        VStack {
            Text("SwiftUI Header")
            CustomUIKitView()  // Embed UIKit
            Text("SwiftUI Footer")
        }
    }
}
```

---

## Modern Development Tools

### Xcode Features (Xcode 15+)

```swift
// 1. String Catalogs (Localization)
// Automatic extraction of localizable strings
Text("Hello, World!")  // Automatically added to catalog

// 2. Swift Macros
@attached(member, names: named(init))
macro Codable() = #externalMacro(...)

@Codable
struct User {
    let name: String
    let email: String
    // init automatically generated
}

// 3. Bookmarks
// Add bookmarks in code:
// TODO: Implement feature
// FIXME: Bug here
// MARK: - Section

// 4. Swift Testing (Xcode 16+)
import Testing

@Test func addition() {
    #expect(2 + 2 == 4)
}

@Test func asyncOperation() async throws {
    let result = try await fetchData()
    #expect(result.count > 0)
}
```

### SwiftUI Preview Variants

```swift
#Preview {
    ContentView()
}

#Preview("Dark Mode") {
    ContentView()
        .preferredColorScheme(.dark)
}

#Preview("Large Text") {
    ContentView()
        .environment(\.sizeCategory, .accessibilityExtraExtraLarge)
}

#Preview("Landscape", traits: .landscapeLeft) {
    ContentView()
}

#Preview("iPad") {
    ContentView()
        .previewDevice("iPad Pro (12.9-inch)")
}
```

### Modern Debugging

```swift
// 1. os_log (Structured logging)
import os.log

let logger = Logger(subsystem: "com.myapp", category: "network")

logger.info("Fetching users")
logger.error("Failed to fetch: \(error)")
logger.debug("Response: \(data)")

// 2. Signpost (Performance tracking)
import os.signpost

let log = OSLog(subsystem: "com.myapp", category: .pointsOfInterest)
let signpostID = OSSignpostID(log: log)

os_signpost(.begin, log: log, name: "Fetch Users", signpostID: signpostID)
// ... work ...
os_signpost(.end, log: log, name: "Fetch Users", signpostID: signpostID)
```

---

## Best Practices 2025

### Modern Architecture Pattern

```swift
// Clean Architecture with modern Swift

// Domain Layer
protocol UseCase {
    associatedtype Input
    associatedtype Output
    
    func execute(_ input: Input) async throws -> Output
}

struct FetchMoviesUseCase: UseCase {
    let repository: MovieRepository
    
    func execute(_ input: Void) async throws -> [Movie] {
        return try await repository.fetchMovies()
    }
}

// Presentation Layer
@Observable
class MovieListViewModel {
    var movies: [Movie] = []
    var isLoading = false
    
    private let fetchMoviesUseCase: FetchMoviesUseCase
    
    init(fetchMoviesUseCase: FetchMoviesUseCase) {
        self.fetchMoviesUseCase = fetchMoviesUseCase
    }
    
    @MainActor
    func loadMovies() async {
        isLoading = true
        movies = try? await fetchMoviesUseCase.execute(())
        isLoading = false
    }
}
```

### Error Handling Best Practices

```swift
// Modern error handling

enum AppError: LocalizedError {
    case networkError(NetworkError)
    case databaseError(DatabaseError)
    case validationError(String)
    
    var errorDescription: String? {
        switch self {
        case .networkError(let error):
            return "Network error: \(error.localizedDescription)"
        case .databaseError(let error):
            return "Database error: \(error.localizedDescription)"
        case .validationError(let message):
            return message
        }
    }
    
    var recoverySuggestion: String? {
        switch self {
        case .networkError:
            return "Check your internet connection and try again."
        case .databaseError:
            return "Try restarting the app."
        case .validationError:
            return "Please check your input and try again."
        }
    }
}

// Error view
struct ErrorView: View {
    let error: Error
    let retry: () -> Void
    
    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "exclamationmark.triangle")
                .font(.largeTitle)
                .foregroundColor(.red)
            
            Text(error.localizedDescription)
                .multilineTextAlignment(.center)
            
            if let suggestion = (error as? LocalizedError)?.recoverySuggestion {
                Text(suggestion)
                    .font(.caption)
                    .foregroundColor(.gray)
            }
            
            Button("Try Again", action: retry)
                .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

---

## Practical AsyncStream Patterns

### Location Updates with AsyncStream

```swift
import CoreLocation
import Foundation

// Location manager with AsyncStream
@MainActor
class LocationManager: NSObject, ObservableObject {
    @Published var location: CLLocation?
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined

    private let manager = CLLocationManager()
    private var continuation: AsyncStream<CLLocation>.Continuation?

    override init() {
        super.init()
        manager.delegate = self
    }

    func locationUpdates() -> AsyncStream<CLLocation> {
        AsyncStream { continuation in
            self.continuation = continuation

            manager.requestWhenInUseAuthorization()
            manager.startUpdatingLocation()

            continuation.onTermination = { [weak self] _ in
                self?.manager.stopUpdatingLocation()
            }
        }
    }

    func startMonitoring() {
        Task {
            for await location in locationUpdates() {
                self.location = location
                print("📍 Location: \(location.coordinate)")
            }
        }
    }
}

extension LocationManager: CLLocationManagerDelegate {
    func locationManager(_ manager: CLLocationManager,
                        didUpdateLocations locations: [CLLocation]) {
        for location in locations {
            continuation?.yield(location)
        }
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus
    }
}

// Usage in SwiftUI
struct LocationView: View {
    @StateObject private var locationManager = LocationManager()

    var body: some View {
        VStack {
            if let location = locationManager.location {
                Text("Lat: \(location.coordinate.latitude)")
                Text("Lon: \(location.coordinate.longitude)")
            } else {
                Text("Waiting for location...")
            }
        }
        .task {
            locationManager.startMonitoring()
        }
    }
}
```

### Core Bluetooth Streaming

```swift
import CoreBluetooth

// Bluetooth device scanner with AsyncStream
class BluetoothScanner: NSObject {
    private var centralManager: CBCentralManager!
    private var peripheralContinuation: AsyncStream<CBPeripheral>.Continuation?

    func scanForPeripherals() -> AsyncStream<CBPeripheral> {
        AsyncStream { continuation in
            self.peripheralContinuation = continuation

            centralManager = CBCentralManager(delegate: self, queue: nil)

            continuation.onTermination = { [weak self] _ in
                self?.centralManager.stopScan()
            }
        }
    }

    // Stream data from characteristic
    func characteristicUpdates(
        for characteristic: CBCharacteristic
    ) -> AsyncStream<Data> {
        AsyncStream { continuation in
            // Set up notification for characteristic
            // Yield data when received

            continuation.onTermination = { _ in
                // Clean up notifications
            }
        }
    }
}

extension BluetoothScanner: CBCentralManagerDelegate {
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            central.scanForPeripherals(withServices: nil, options: nil)
        }
    }

    func centralManager(_ central: CBCentralManager,
                       didDiscover peripheral: CBPeripheral,
                       advertisementData: [String : Any],
                       rssi RSSI: NSNumber) {
        peripheralContinuation?.yield(peripheral)
    }
}

// Usage
let scanner = BluetoothScanner()
Task {
    for await peripheral in scanner.scanForPeripherals() {
        print("Found device: \(peripheral.name ?? "Unknown")")
    }
}
```

### NotificationCenter as AsyncStream

```swift
import UIKit

extension NotificationCenter {
    // Convert notifications to AsyncStream
    func notifications(
        named name: Notification.Name,
        object: Any? = nil
    ) -> AsyncStream<Notification> {
        AsyncStream { continuation in
            let observer = self.addObserver(
                forName: name,
                object: object,
                queue: .main
            ) { notification in
                continuation.yield(notification)
            }

            continuation.onTermination = { _ in
                self.removeObserver(observer)
            }
        }
    }
}

// Monitor app lifecycle
class AppLifecycleMonitor {
    func startMonitoring() {
        Task {
            await withTaskGroup(of: Void.self) { group in
                // Monitor foreground events
                group.addTask {
                    for await _ in NotificationCenter.default.notifications(
                        named: UIApplication.willEnterForegroundNotification
                    ) {
                        print("App will enter foreground")
                        await self.handleForeground()
                    }
                }

                // Monitor background events
                group.addTask {
                    for await _ in NotificationCenter.default.notifications(
                        named: UIApplication.didEnterBackgroundNotification
                    ) {
                        print("App did enter background")
                        await self.handleBackground()
                    }
                }

                // Monitor memory warnings
                group.addTask {
                    for await _ in NotificationCenter.default.notifications(
                        named: UIApplication.didReceiveMemoryWarningNotification
                    ) {
                        print("Memory warning!")
                        await self.handleMemoryWarning()
                    }
                }
            }
        }
    }

    private func handleForeground() async {
        // Refresh data, resume operations
    }

    private func handleBackground() async {
        // Save state, pause operations
    }

    private func handleMemoryWarning() async {
        // Clear caches, reduce memory usage
    }
}
```

### Timer/Ticker Implementation

```swift
import Foundation

// Ticker with AsyncStream
struct Ticker {
    let interval: TimeInterval

    func tick() -> AsyncStream<Date> {
        AsyncStream { continuation in
            let timer = Timer.scheduledTimer(
                withTimeInterval: interval,
                repeats: true
            ) { _ in
                continuation.yield(Date())
            }

            RunLoop.current.add(timer, forMode: .common)

            continuation.onTermination = { _ in
                timer.invalidate()
            }
        }
    }
}

// Countdown timer
struct CountdownTimer {
    let duration: TimeInterval
    let interval: TimeInterval

    func countdown() -> AsyncStream<TimeInterval> {
        AsyncStream { continuation in
            var remaining = duration

            let timer = Timer.scheduledTimer(
                withTimeInterval: interval,
                repeats: true
            ) { timer in
                remaining -= interval

                if remaining <= 0 {
                    continuation.yield(0)
                    continuation.finish()
                    timer.invalidate()
                } else {
                    continuation.yield(remaining)
                }
            }

            RunLoop.current.add(timer, forMode: .common)

            continuation.onTermination = { _ in
                timer.invalidate()
            }
        }
    }
}

// Usage in SwiftUI
struct TimerView: View {
    @State private var currentTime = Date()
    @State private var countdown: TimeInterval = 60

    var body: some View {
        VStack {
            Text("Current: \(currentTime.formatted())")
            Text("Countdown: \(Int(countdown))s")
        }
        .task {
            // Update clock every second
            for await time in Ticker(interval: 1.0).tick() {
                currentTime = time
            }
        }
        .task {
            // Countdown from 60 seconds
            for await remaining in CountdownTimer(duration: 60, interval: 1).countdown() {
                countdown = remaining
            }
            print("Countdown finished!")
        }
    }
}
```

### Combine Publisher to AsyncStream Bridge

```swift
import Combine
import SwiftUI

// Bridge Combine to async/await
extension Publisher {
    func asAsyncStream() -> AsyncStream<Output> {
        AsyncStream { continuation in
            let cancellable = self.sink(
                receiveCompletion: { _ in
                    continuation.finish()
                },
                receiveValue: { value in
                    continuation.yield(value)
                }
            )

            continuation.onTermination = { _ in
                cancellable.cancel()
            }
        }
    }
}

// Use existing Combine publishers with async/await
class DataService: ObservableObject {
    @Published var searchText = ""
    @Published var results: [SearchResult] = []

    private var cancellables = Set<AnyCancellable>()

    func setupSearch() {
        Task {
            // Convert Combine publisher to AsyncStream
            for await query in $searchText
                .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
                .removeDuplicates()
                .asAsyncStream() {

                await performSearch(query)
            }
        }
    }

    private func performSearch(_ query: String) async {
        guard !query.isEmpty else {
            results = []
            return
        }

        do {
            // Perform async search
            results = try await searchAPI(query: query)
        } catch {
            print("Search error: \(error)")
        }
    }
}
```

### Sensor Data Streaming

```swift
import CoreMotion

// Motion sensor data as AsyncStream
class MotionManager {
    private let motionManager = CMMotionManager()

    func accelerometerUpdates(
        interval: TimeInterval = 0.1
    ) -> AsyncThrowingStream<CMAccelerometerData, Error> {
        AsyncThrowingStream { continuation in
            guard motionManager.isAccelerometerAvailable else {
                continuation.finish(throwing: MotionError.notAvailable)
                return
            }

            motionManager.accelerometerUpdateInterval = interval
            motionManager.startAccelerometerUpdates(
                to: .main
            ) { data, error in
                if let error = error {
                    continuation.finish(throwing: error)
                } else if let data = data {
                    continuation.yield(data)
                }
            }

            continuation.onTermination = { [weak self] _ in
                self?.motionManager.stopAccelerometerUpdates()
            }
        }
    }

    func gyroUpdates(
        interval: TimeInterval = 0.1
    ) -> AsyncThrowingStream<CMGyroData, Error> {
        AsyncThrowingStream { continuation in
            guard motionManager.isGyroAvailable else {
                continuation.finish(throwing: MotionError.notAvailable)
                return
            }

            motionManager.gyroUpdateInterval = interval
            motionManager.startGyroUpdates(
                to: .main
            ) { data, error in
                if let error = error {
                    continuation.finish(throwing: error)
                } else if let data = data {
                    continuation.yield(data)
                }
            }

            continuation.onTermination = { [weak self] _ in
                self?.motionManager.stopGyroUpdates()
            }
        }
    }
}

enum MotionError: Error {
    case notAvailable
}

// Usage
let motion = MotionManager()
Task {
    do {
        for try await data in motion.accelerometerUpdates() {
            print("X: \(data.acceleration.x)")
            print("Y: \(data.acceleration.y)")
            print("Z: \(data.acceleration.z)")
        }
    } catch {
        print("Motion error: \(error)")
    }
}
```

### File System Monitoring

```swift
import Foundation

// Monitor file changes with AsyncStream
class FileMonitor {
    private var source: DispatchSourceFileSystemObject?

    func monitorFile(at url: URL) -> AsyncStream<FileEvent> {
        AsyncStream { continuation in
            let fileDescriptor = open(url.path, O_EVTONLY)
            guard fileDescriptor >= 0 else {
                continuation.finish()
                return
            }

            let source = DispatchSource.makeFileSystemObjectSource(
                fileDescriptor: fileDescriptor,
                eventMask: [.write, .delete, .rename],
                queue: .main
            )

            source.setEventHandler { [weak self] in
                let flags = source.data

                if flags.contains(.write) {
                    continuation.yield(.modified)
                }
                if flags.contains(.delete) {
                    continuation.yield(.deleted)
                    continuation.finish()
                }
                if flags.contains(.rename) {
                    continuation.yield(.renamed)
                }
            }

            source.setCancelHandler {
                close(fileDescriptor)
            }

            source.resume()
            self.source = source

            continuation.onTermination = { [weak self] _ in
                self?.source?.cancel()
                self?.source = nil
            }
        }
    }
}

enum FileEvent {
    case modified
    case deleted
    case renamed
}

// Usage
let monitor = FileMonitor()
Task {
    for await event in monitor.monitorFile(at: configFileURL) {
        switch event {
        case .modified:
            print("Config file changed, reloading...")
            reloadConfiguration()
        case .deleted:
            print("Config file deleted!")
        case .renamed:
            print("Config file renamed")
        }
    }
}
```

---

## Migration Strategies

### Gradual UIKit → SwiftUI Migration

```swift
// Step 1: Start with small screens
// Convert settings, about, simple forms first

class SettingsViewController: UIHostingController<SettingsView> {
    init() {
        super.init(rootView: SettingsView())
    }
}

// Step 2: Embed SwiftUI in UIKit
class MainViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Embed SwiftUI view
        let swiftUIView = NewFeatureView()
        let hostingController = UIHostingController(rootView: swiftUIView)
        
        addChild(hostingController)
        view.addSubview(hostingController.view)
        hostingController.didMove(toParent: self)
        
        // Layout
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
}

// Step 3: Share ViewModels
// Use same ViewModels for both UIKit and SwiftUI
class SharedViewModel: ObservableObject {
    @Published var data: [Item] = []
    
    func loadData() async {
        // Works with both UIKit and SwiftUI
    }
}
```

---

## Interview Questions

### Q: "What are the main differences between SwiftUI and UIKit?"

**A:** 
- **Paradigm**: SwiftUI is declarative, UIKit is imperative
- **State**: SwiftUI auto-updates on state changes, UIKit requires manual updates
- **Code volume**: SwiftUI requires less boilerplate
- **Performance**: UIKit has more control, SwiftUI is optimized
- **Flexibility**: UIKit for complex custom views, SwiftUI for rapid development

### Q: "Explain @State, @StateObject, @ObservedObject, @EnvironmentObject"

**A:**
- **@State**: Simple value types, view-owned, private
- **@StateObject**: Reference types, view creates and owns
- **@ObservedObject**: Reference types, passed from parent
- **@EnvironmentObject**: Shared across hierarchy, injected from ancestor

### Q: "When would you use SwiftUI vs UIKit?"

**A:**
- **SwiftUI**: New projects (iOS 15+), rapid prototyping, simple-medium UIs
- **UIKit**: Legacy support (< iOS 13), complex animations, pixel-perfect control
- **Hybrid**: Most real apps - SwiftUI for new features, UIKit where needed

### Q: "What's the difference between async/await and Combine?"

**A:**
- **async/await**: Sequential async operations, easier to read, modern approach
- **Combine**: Reactive streams, event-driven, complex transformations
- **Use async/await** for: API calls, sequential operations
- **Use Combine** for: Reactive UI, complex event streams, multiple publishers

### Q: "Explain Actors and why they're useful"

**A:** Actors provide thread-safe encapsulation:
- Automatically serialize access to mutable state
- Prevent data races at compile time
- Methods are async (cross-actor calls)
- @MainActor for UI updates

Example: BankAccount actor ensures deposits/withdrawals are thread-safe.

---

Your ObjC experience is valuable - you understand the "why" behind iOS patterns. Modern Swift is just cleaner syntax for concepts you already know! Focus on SwiftUI and async/await - these are the biggest changes from the ObjC era.
