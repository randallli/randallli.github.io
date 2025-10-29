---
layout: page
title: "iOS Architecture & Design Patterns"
subtitle: "Complete Guide for Senior iOS Engineers"
---

# iOS Architecture & Design Patterns
## Complete Guide for Senior iOS Engineers

---

## Table of Contents
1. [MVC (Model-View-Controller)](#mvc-model-view-controller)
2. [MVVM (Model-View-ViewModel)](#mvvm-model-view-viewmodel)
3. [MVVM + Coordinator](#mvvm--coordinator)
4. [VIPER](#viper)
5. [Clean Architecture](#clean-architecture)
6. [Design Patterns](#design-patterns)
7. [Dependency Injection](#dependency-injection)
8. [Repository Pattern](#repository-pattern)
9. [Architectural Decision Making](#architectural-decision-making)
10. [Interview Questions](#interview-questions)

---

## MVC (Model-View-Controller)

### Apple's MVC

```
┌─────────────┐
│    View     │ ←─────┐
└─────────────┘       │
       ↕               │ Observes
┌─────────────┐       │
│ Controller  │ ──────┘
└─────────────┘
       ↕
┌─────────────┐
│    Model    │
└─────────────┘
```

### The Reality: "Massive View Controller"

```swift
// ❌ Typical bloated view controller
class UserViewController: UIViewController {
    // View components
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var searchBar: UISearchBar!
    
    // Model
    var users: [User] = []
    var filteredUsers: [User] = []
    
    // Business logic
    func fetchUsers() {
        // Networking code here
        let url = URL(string: "https://api.example.com/users")!
        URLSession.shared.dataTask(with: url) { data, response, error in
            // Parsing logic
            // Error handling
            // State management
            DispatchQueue.main.async {
                self.users = parsedUsers
                self.tableView.reloadData()
            }
        }.resume()
    }
    
    // View logic
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        setupTableView()
        fetchUsers()
    }
    
    // Delegate methods
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { }
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell { }
    
    // Search logic
    func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
        // Filtering logic
    }
    
    // Navigation
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let detailVC = UserDetailViewController()
        detailVC.user = users[indexPath.row]
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

### Problems with Massive View Controllers

1. **God Object**: Does too many things
2. **Hard to Test**: Tightly coupled to UIKit
3. **No Separation**: Business logic mixed with view logic
4. **Difficult Reuse**: Logic tied to specific view controller
5. **Poor Maintainability**: 1000+ line files

### Better MVC (with thin controllers)

```swift
// Model
struct User: Codable {
    let id: String
    let name: String
    let email: String
}

// Separate networking service
class UserService {
    func fetchUsers() async throws -> [User] {
        let url = URL(string: "https://api.example.com/users")!
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode([User].self, from: data)
    }
}

// Separate data source
class UserDataSource: NSObject, UITableViewDataSource {
    var users: [User] = []
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return users.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "UserCell", for: indexPath) as! UserCell
        cell.configure(with: users[indexPath.row])
        return cell
    }
}

// Thin controller
class UserViewController: UIViewController {
    private let tableView = UITableView()
    private let service = UserService()
    private let dataSource = UserDataSource()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupTableView()
        loadUsers()
    }
    
    private func setupTableView() {
        tableView.dataSource = dataSource
        tableView.delegate = self
        // Layout setup
    }
    
    private func loadUsers() {
        Task {
            do {
                dataSource.users = try await service.fetchUsers()
                tableView.reloadData()
            } catch {
                showError(error)
            }
        }
    }
}

extension UserViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let user = dataSource.users[indexPath.row]
        let detailVC = UserDetailViewController(user: user)
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

**When to Use MVC:**
- Small apps or prototypes
- Simple screens with minimal logic
- When team is most familiar with it
- Apple's preferred pattern for sample code

---

## MVVM (Model-View-ViewModel)

### Architecture Diagram

```
┌──────────┐
│   View   │ ←──── Binding/Observation ────┐
└──────────┘                                │
                                            │
┌──────────────────────────────────────────┴┐
│           ViewModel                        │
│  • Presentation Logic                     │
│  • State Management                       │
│  • Business Logic                         │
└──────────────────────────────────────────┬┘
                                            │
┌──────────┐                                │
│  Model   │ ←──────────────────────────────┘
└──────────┘
```

### Key Principles

1. **View** doesn't know about Model (only ViewModel)
2. **ViewModel** doesn't know about View (uses protocols/bindings)
3. **Model** is pure data
4. **Testable**: ViewModel has no UIKit dependencies

### MVVM Implementation (UIKit)

```swift
// Model
struct User: Codable {
    let id: String
    let name: String
    let email: String
}

// ViewModel
class UserListViewModel {
    // Published properties (observable)
    @Published private(set) var users: [User] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    
    private let userService: UserServiceProtocol
    
    init(userService: UserServiceProtocol = UserService()) {
        self.userService = userService
    }
    
    // Input
    func loadUsers() {
        isLoading = true
        errorMessage = nil
        
        Task {
            do {
                users = try await userService.fetchUsers()
                isLoading = false
            } catch {
                errorMessage = error.localizedDescription
                isLoading = false
            }
        }
    }
    
    func searchUsers(with query: String) {
        // Search logic
    }
    
    // Presentation logic
    func numberOfUsers() -> Int {
        return users.count
    }
    
    func user(at index: Int) -> User {
        return users[index]
    }
    
    func userDisplayName(at index: Int) -> String {
        let user = users[index]
        return "\(user.name) (\(user.email))"
    }
}

// View
class UserListViewController: UIViewController {
    private let tableView = UITableView()
    private let viewModel: UserListViewModel
    private var cancellables = Set<AnyCancellable>()
    
    init(viewModel: UserListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
        viewModel.loadUsers()
    }
    
    private func setupUI() {
        view.addSubview(tableView)
        tableView.dataSource = self
        tableView.delegate = self
        // Layout
    }
    
    private func bindViewModel() {
        // Observe users changes
        viewModel.$users
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
        
        // Observe loading state
        viewModel.$isLoading
            .receive(on: DispatchQueue.main)
            .sink { [weak self] isLoading in
                if isLoading {
                    self?.showLoadingIndicator()
                } else {
                    self?.hideLoadingIndicator()
                }
            }
            .store(in: &cancellables)
        
        // Observe errors
        viewModel.$errorMessage
            .receive(on: DispatchQueue.main)
            .compactMap { $0 }
            .sink { [weak self] errorMessage in
                self?.showError(errorMessage)
            }
            .store(in: &cancellables)
    }
}

extension UserListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.numberOfUsers()
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        cell.textLabel?.text = viewModel.userDisplayName(at: indexPath.row)
        return cell
    }
}
```

### MVVM with SwiftUI (Natural Fit)

```swift
// ViewModel (same as above, but ObservableObject)
class UserListViewModel: ObservableObject {
    @Published private(set) var users: [User] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    
    private let userService: UserServiceProtocol
    
    init(userService: UserServiceProtocol = UserService()) {
        self.userService = userService
    }
    
    func loadUsers() {
        isLoading = true
        errorMessage = nil
        
        Task { @MainActor in
            do {
                users = try await userService.fetchUsers()
                isLoading = false
            } catch {
                errorMessage = error.localizedDescription
                isLoading = false
            }
        }
    }
}

// View (SwiftUI)
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()
    
    var body: some View {
        NavigationView {
            Group {
                if viewModel.isLoading {
                    ProgressView()
                } else if let errorMessage = viewModel.errorMessage {
                    ErrorView(message: errorMessage)
                } else {
                    List(viewModel.users, id: \.id) { user in
                        NavigationLink(destination: UserDetailView(user: user)) {
                            UserRow(user: user)
                        }
                    }
                }
            }
            .navigationTitle("Users")
            .onAppear {
                viewModel.loadUsers()
            }
        }
    }
}
```

### Testing ViewModel

```swift
class UserListViewModelTests: XCTestCase {
    var sut: UserListViewModel!
    var mockService: MockUserService!
    
    override func setUp() {
        super.setUp()
        mockService = MockUserService()
        sut = UserListViewModel(userService: mockService)
    }
    
    func testLoadUsers_Success() async {
        // Arrange
        let expectedUsers = [
            User(id: "1", name: "John", email: "john@example.com"),
            User(id: "2", name: "Jane", email: "jane@example.com")
        ]
        mockService.mockUsers = expectedUsers
        
        // Act
        sut.loadUsers()
        try? await Task.sleep(nanoseconds: 100_000_000) // Wait for async
        
        // Assert
        XCTAssertEqual(sut.users.count, 2)
        XCTAssertEqual(sut.users.first?.name, "John")
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.errorMessage)
    }
    
    func testLoadUsers_Failure() async {
        // Arrange
        mockService.shouldFail = true
        
        // Act
        sut.loadUsers()
        try? await Task.sleep(nanoseconds: 100_000_000)
        
        // Assert
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertFalse(sut.isLoading)
        XCTAssertNotNil(sut.errorMessage)
    }
}
```

**When to Use MVVM:**
- Medium to large apps
- When testability is important
- SwiftUI apps (natural fit)
- When you need to share view logic across platforms
- Team comfortable with reactive programming

---

## MVVM + Coordinator

### The Problem MVVM Doesn't Solve: Navigation

```swift
// ❌ ViewModel shouldn't know about navigation
class UserListViewModel {
    func didSelectUser(_ user: User) {
        // How to navigate? ViewModel shouldn't create/push view controllers
    }
}

// ❌ View shouldn't handle complex navigation logic
class UserListViewController {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let user = viewModel.user(at: indexPath.row)
        let detailVC = UserDetailViewController(user: user)
        let detailViewModel = UserDetailViewModel(user: user)
        detailVC.viewModel = detailViewModel
        navigationController?.pushViewController(detailVC, animated: true)
        // View knows too much about navigation setup
    }
}
```

### Coordinator Pattern

```
┌────────────────┐
│  Coordinator   │ ──→ Creates Views & ViewModels
│  • Navigation  │ ──→ Manages Flow
│  • Flow Logic  │ ──→ Child Coordinators
└────────────────┘
        │
        ├──→ ViewController 1 + ViewModel 1
        ├──→ ViewController 2 + ViewModel 2
        └──→ ViewController 3 + ViewModel 3
```

### Coordinator Protocol

```swift
// Base coordinator protocol
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    var navigationController: UINavigationController { get set }
    
    func start()
}

extension Coordinator {
    func addChildCoordinator(_ coordinator: Coordinator) {
        childCoordinators.append(coordinator)
    }
    
    func removeChildCoordinator(_ coordinator: Coordinator) {
        childCoordinators = childCoordinators.filter { $0 !== coordinator }
    }
}
```

### App Coordinator (Root)

```swift
class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var navigationController: UINavigationController
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }
    
    func start() {
        showLogin()
    }
    
    func showLogin() {
        let loginCoordinator = LoginCoordinator(navigationController: navigationController)
        loginCoordinator.delegate = self
        addChildCoordinator(loginCoordinator)
        loginCoordinator.start()
    }
    
    func showMainApp() {
        let mainCoordinator = MainCoordinator(navigationController: navigationController)
        addChildCoordinator(mainCoordinator)
        mainCoordinator.start()
    }
}

extension AppCoordinator: LoginCoordinatorDelegate {
    func didFinishLogin(_ coordinator: LoginCoordinator) {
        removeChildCoordinator(coordinator)
        showMainApp()
    }
}
```

### Feature Coordinator

```swift
protocol UserListCoordinatorDelegate: AnyObject {
    func didSelectUser(_ user: User, from coordinator: UserListCoordinator)
}

class UserListCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var navigationController: UINavigationController
    weak var delegate: UserListCoordinatorDelegate?
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }
    
    func start() {
        let viewModel = UserListViewModel()
        viewModel.coordinator = self
        
        let viewController = UserListViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func showUserDetail(_ user: User) {
        let detailCoordinator = UserDetailCoordinator(
            navigationController: navigationController,
            user: user
        )
        addChildCoordinator(detailCoordinator)
        detailCoordinator.delegate = self
        detailCoordinator.start()
    }
}

extension UserListCoordinator: UserDetailCoordinatorDelegate {
    func didFinishViewingUser(_ coordinator: UserDetailCoordinator) {
        removeChildCoordinator(coordinator)
    }
}
```

### ViewModel with Coordinator

```swift
class UserListViewModel {
    weak var coordinator: UserListCoordinator?
    
    @Published private(set) var users: [User] = []
    
    func didSelectUser(_ user: User) {
        coordinator?.showUserDetail(user)
    }
}
```

### Complete Flow Example

```swift
// AppDelegate
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    window = UIWindow(frame: UIScreen.main.bounds)
    
    let navigationController = UINavigationController()
    let appCoordinator = AppCoordinator(navigationController: navigationController)
    appCoordinator.start()
    
    window?.rootViewController = navigationController
    window?.makeKeyAndVisible()
    
    return true
}
```

**Benefits of Coordinator:**
- **Separation of Concerns**: Navigation logic separate from view logic
- **Reusability**: ViewModels/Views can be reused with different flows
- **Testability**: Can test navigation flows independently
- **Deep Linking**: Easier to handle deep links and URL routing
- **A/B Testing**: Easy to swap different flows

**Drawbacks:**
- More boilerplate code
- Steeper learning curve
- Overkill for simple apps

---

## VIPER

### Architecture Diagram

```
┌─────────┐
│  View   │ ←──────→ Presenter ←──────→ Interactor
└─────────┘              ↕                   ↕
                         │              ┌─────────┐
                         │              │ Entity  │
                         │              └─────────┘
                         ↓
                    ┌─────────┐
                    │ Router  │
                    └─────────┘
```

**Components:**
- **View**: Displays UI, sends user events to Presenter
- **Interactor**: Business logic, independent of UI
- **Presenter**: Presentation logic, prepares data for View
- **Entity**: Plain data models
- **Router**: Navigation logic

### VIPER Implementation

```swift
// MARK: - Entity
struct User {
    let id: String
    let name: String
    let email: String
}

// MARK: - View Protocol
protocol UserListViewProtocol: AnyObject {
    func showUsers(_ users: [UserListViewModel])
    func showLoading()
    func hideLoading()
    func showError(_ message: String)
}

// MARK: - Presenter Protocol
protocol UserListPresenterProtocol: AnyObject {
    func viewDidLoad()
    func didSelectUser(at index: Int)
    func didPullToRefresh()
}

// MARK: - Interactor Protocol
protocol UserListInteractorProtocol: AnyObject {
    func fetchUsers()
}

// MARK: - Router Protocol
protocol UserListRouterProtocol: AnyObject {
    func navigateToUserDetail(with user: User)
}

// MARK: - View
class UserListViewController: UIViewController {
    var presenter: UserListPresenterProtocol!
    
    private let tableView = UITableView()
    private var viewModels: [UserListViewModel] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        presenter.viewDidLoad()
    }
    
    private func setupUI() {
        view.addSubview(tableView)
        tableView.dataSource = self
        tableView.delegate = self
    }
}

extension UserListViewController: UserListViewProtocol {
    func showUsers(_ users: [UserListViewModel]) {
        self.viewModels = users
        tableView.reloadData()
    }
    
    func showLoading() {
        // Show loading indicator
    }
    
    func hideLoading() {
        // Hide loading indicator
    }
    
    func showError(_ message: String) {
        // Show error alert
    }
}

extension UserListViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        presenter.didSelectUser(at: indexPath.row)
    }
}

// MARK: - Presenter
class UserListPresenter {
    weak var view: UserListViewProtocol?
    var interactor: UserListInteractorProtocol!
    var router: UserListRouterProtocol!
    
    private var users: [User] = []
    
    init(view: UserListViewProtocol,
         interactor: UserListInteractorProtocol,
         router: UserListRouterProtocol) {
        self.view = view
        self.interactor = interactor
        self.router = router
    }
}

extension UserListPresenter: UserListPresenterProtocol {
    func viewDidLoad() {
        view?.showLoading()
        interactor.fetchUsers()
    }
    
    func didSelectUser(at index: Int) {
        let user = users[index]
        router.navigateToUserDetail(with: user)
    }
    
    func didPullToRefresh() {
        interactor.fetchUsers()
    }
}

extension UserListPresenter: UserListInteractorOutputProtocol {
    func didFetchUsers(_ users: [User]) {
        self.users = users
        let viewModels = users.map { UserListViewModel(name: $0.name, email: $0.email) }
        view?.hideLoading()
        view?.showUsers(viewModels)
    }
    
    func didFailToFetchUsers(with error: Error) {
        view?.hideLoading()
        view?.showError(error.localizedDescription)
    }
}

// MARK: - Interactor
protocol UserListInteractorOutputProtocol: AnyObject {
    func didFetchUsers(_ users: [User])
    func didFailToFetchUsers(with error: Error)
}

class UserListInteractor {
    weak var output: UserListInteractorOutputProtocol?
    private let userService: UserServiceProtocol
    
    init(userService: UserServiceProtocol) {
        self.userService = userService
    }
}

extension UserListInteractor: UserListInteractorProtocol {
    func fetchUsers() {
        Task {
            do {
                let users = try await userService.fetchUsers()
                await MainActor.run {
                    output?.didFetchUsers(users)
                }
            } catch {
                await MainActor.run {
                    output?.didFailToFetchUsers(with: error)
                }
            }
        }
    }
}

// MARK: - Router
class UserListRouter {
    weak var viewController: UIViewController?
}

extension UserListRouter: UserListRouterProtocol {
    func navigateToUserDetail(with user: User) {
        let detailModule = UserDetailModuleBuilder.build(with: user)
        viewController?.navigationController?.pushViewController(detailModule, animated: true)
    }
}

// MARK: - Module Builder
class UserListModuleBuilder {
    static func build() -> UIViewController {
        let view = UserListViewController()
        let interactor = UserListInteractor(userService: UserService())
        let router = UserListRouter()
        let presenter = UserListPresenter(view: view, interactor: interactor, router: router)
        
        view.presenter = presenter
        interactor.output = presenter
        router.viewController = view
        
        return view
    }
}
```

**When to Use VIPER:**
- Large, complex apps
- Multiple developers/teams
- When extreme testability is required
- Enterprise applications
- When clear separation is more important than simplicity

**Drawbacks:**
- Lots of boilerplate
- Overkill for small features
- Steep learning curve
- Can be over-engineered

---

## Clean Architecture

### Layers

```
┌─────────────────────────────────────────┐
│         Presentation Layer              │
│  (UI, ViewModels, Views)                │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         Domain Layer                    │
│  (Use Cases, Entities, Protocols)       │
│  • Business Logic                       │
│  • No Framework Dependencies            │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         Data Layer                      │
│  (Repositories, API, Database, Cache)   │
└─────────────────────────────────────────┘
```

### Implementation

```swift
// MARK: - Domain Layer

// Entity (pure Swift, no dependencies)
struct User {
    let id: String
    let name: String
    let email: String
}

// Use Case Protocol
protocol FetchUsersUseCase {
    func execute() async throws -> [User]
}

// Repository Protocol (in Domain, implemented in Data)
protocol UserRepositoryProtocol {
    func fetchUsers() async throws -> [User]
}

// MARK: - Data Layer

// API Response DTO (Data Transfer Object)
struct UserDTO: Codable {
    let id: String
    let name: String
    let email: String
    
    func toDomain() -> User {
        return User(id: id, name: name, email: email)
    }
}

// Repository Implementation
class UserRepository: UserRepositoryProtocol {
    private let apiService: APIServiceProtocol
    private let cache: CacheProtocol
    
    init(apiService: APIServiceProtocol, cache: CacheProtocol) {
        self.apiService = apiService
        self.cache = cache
    }
    
    func fetchUsers() async throws -> [User] {
        // Try cache first
        if let cachedUsers: [User] = cache.get(key: "users") {
            return cachedUsers
        }
        
        // Fetch from API
        let dtos: [UserDTO] = try await apiService.request(endpoint: .users)
        let users = dtos.map { $0.toDomain() }
        
        // Cache result
        cache.set(users, forKey: "users")
        
        return users
    }
}

// MARK: - Domain Layer (Use Case Implementation)

class FetchUsersUseCaseImpl: FetchUsersUseCase {
    private let repository: UserRepositoryProtocol
    
    init(repository: UserRepositoryProtocol) {
        self.repository = repository
    }
    
    func execute() async throws -> [User] {
        return try await repository.fetchUsers()
    }
}

// MARK: - Presentation Layer

class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let fetchUsersUseCase: FetchUsersUseCase
    
    init(fetchUsersUseCase: FetchUsersUseCase) {
        self.fetchUsersUseCase = fetchUsersUseCase
    }
    
    func loadUsers() {
        isLoading = true
        
        Task { @MainActor in
            do {
                users = try await fetchUsersUseCase.execute()
                isLoading = false
            } catch {
                errorMessage = error.localizedDescription
                isLoading = false
            }
        }
    }
}
```

### Dependency Injection Container

```swift
class AppDependencyContainer {
    // Data Layer
    private lazy var apiService: APIServiceProtocol = {
        return APIService(baseURL: "https://api.example.com")
    }()
    
    private lazy var cache: CacheProtocol = {
        return MemoryCache()
    }()
    
    private lazy var userRepository: UserRepositoryProtocol = {
        return UserRepository(apiService: apiService, cache: cache)
    }()
    
    // Domain Layer
    func makeFetchUsersUseCase() -> FetchUsersUseCase {
        return FetchUsersUseCaseImpl(repository: userRepository)
    }
    
    // Presentation Layer
    func makeUserListViewModel() -> UserListViewModel {
        return UserListViewModel(fetchUsersUseCase: makeFetchUsersUseCase())
    }
}
```

**Benefits:**
- **Independence**: Business logic independent of frameworks
- **Testability**: Easy to test each layer independently
- **Flexibility**: Easy to swap implementations
- **Maintainability**: Clear boundaries and responsibilities

---

## Design Patterns

### 1. Delegate Pattern

```swift
// Protocol
protocol UserViewDelegate: AnyObject {
    func didSelectUser(_ user: User)
    func didDeleteUser(_ user: User)
}

// Delegator
class UserView: UIView {
    weak var delegate: UserViewDelegate?  // Always weak!
    
    @objc private func handleTap() {
        delegate?.didSelectUser(currentUser)
    }
}

// Delegate
class UserViewController: UIViewController {
    let userView = UserView()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        userView.delegate = self
    }
}

extension UserViewController: UserViewDelegate {
    func didSelectUser(_ user: User) {
        // Handle selection
    }
    
    func didDeleteUser(_ user: User) {
        // Handle deletion
    }
}
```

### 2. Observer Pattern (NotificationCenter)

```swift
// Post notification
NotificationCenter.default.post(
    name: .userDidLogin,
    object: nil,
    userInfo: ["userId": userId]
)

// Observe notification
class MyViewController: UIViewController {
    var observer: NSObjectProtocol?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        observer = NotificationCenter.default.addObserver(
            forName: .userDidLogin,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            if let userId = notification.userInfo?["userId"] as? String {
                self?.handleUserLogin(userId: userId)
            }
        }
    }
    
    deinit {
        if let observer = observer {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

// Define notification names
extension Notification.Name {
    static let userDidLogin = Notification.Name("userDidLogin")
    static let userDidLogout = Notification.Name("userDidLogout")
}
```

### 3. Singleton Pattern

```swift
// ⚠️ Use sparingly!
class NetworkManager {
    static let shared = NetworkManager()
    
    private init() {
        // Private init prevents creating multiple instances
    }
    
    func request(url: URL) async throws -> Data {
        // Implementation
    }
}

// Usage
let data = try await NetworkManager.shared.request(url: url)

// ✅ Better: Use dependency injection instead
class MyViewModel {
    private let networkManager: NetworkManagerProtocol
    
    init(networkManager: NetworkManagerProtocol = NetworkManager.shared) {
        self.networkManager = networkManager
    }
}
```

### 4. Factory Pattern

```swift
// Factory protocol
protocol ViewControllerFactory {
    func makeUserListViewController() -> UIViewController
    func makeUserDetailViewController(user: User) -> UIViewController
}

// Concrete factory
class DefaultViewControllerFactory: ViewControllerFactory {
    private let dependencyContainer: AppDependencyContainer
    
    init(dependencyContainer: AppDependencyContainer) {
        self.dependencyContainer = dependencyContainer
    }
    
    func makeUserListViewController() -> UIViewController {
        let viewModel = dependencyContainer.makeUserListViewModel()
        return UserListViewController(viewModel: viewModel)
    }
    
    func makeUserDetailViewController(user: User) -> UIViewController {
        let viewModel = dependencyContainer.makeUserDetailViewModel(user: user)
        return UserDetailViewController(viewModel: viewModel)
    }
}
```

### 5. Strategy Pattern

```swift
// Strategy protocol
protocol SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T]
}

// Concrete strategies
class AscendingSortStrategy: SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T] {
        return items.sorted(by: <)
    }
}

class DescendingSortStrategy: SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T] {
        return items.sorted(by: >)
    }
}

// Context
class DataManager {
    private var sortStrategy: SortStrategy
    
    init(sortStrategy: SortStrategy) {
        self.sortStrategy = sortStrategy
    }
    
    func setSortStrategy(_ strategy: SortStrategy) {
        self.sortStrategy = strategy
    }
    
    func sortedData<T: Comparable>(_ data: [T]) -> [T] {
        return sortStrategy.sort(data)
    }
}

// Usage
let manager = DataManager(sortStrategy: AscendingSortStrategy())
let sorted = manager.sortedData([3, 1, 4, 1, 5])

manager.setSortStrategy(DescendingSortStrategy())
let reverseSorted = manager.sortedData([3, 1, 4, 1, 5])
```

### 6. Builder Pattern

```swift
// Complex object
struct User {
    let id: String
    let name: String
    let email: String
    let age: Int?
    let address: String?
    let phoneNumber: String?
}

// Builder
class UserBuilder {
    private var id: String = ""
    private var name: String = ""
    private var email: String = ""
    private var age: Int?
    private var address: String?
    private var phoneNumber: String?
    
    func setId(_ id: String) -> UserBuilder {
        self.id = id
        return self
    }
    
    func setName(_ name: String) -> UserBuilder {
        self.name = name
        return self
    }
    
    func setEmail(_ email: String) -> UserBuilder {
        self.email = email
        return self
    }
    
    func setAge(_ age: Int) -> UserBuilder {
        self.age = age
        return self
    }
    
    func setAddress(_ address: String) -> UserBuilder {
        self.address = address
        return self
    }
    
    func setPhoneNumber(_ phoneNumber: String) -> UserBuilder {
        self.phoneNumber = phoneNumber
        return self
    }
    
    func build() -> User {
        return User(
            id: id,
            name: name,
            email: email,
            age: age,
            address: address,
            phoneNumber: phoneNumber
        )
    }
}

// Usage
let user = UserBuilder()
    .setId("123")
    .setName("John")
    .setEmail("john@example.com")
    .setAge(30)
    .build()
```

---

## Dependency Injection

### 1. Constructor Injection (Preferred)

```swift
// Protocol
protocol UserServiceProtocol {
    func fetchUsers() async throws -> [User]
}

// Implementation
class UserService: UserServiceProtocol {
    func fetchUsers() async throws -> [User] {
        // Implementation
    }
}

// ViewModel with constructor injection
class UserListViewModel {
    private let userService: UserServiceProtocol
    
    init(userService: UserServiceProtocol) {
        self.userService = userService
    }
    
    func loadUsers() async throws {
        let users = try await userService.fetchUsers()
    }
}

// Usage
let service = UserService()
let viewModel = UserListViewModel(userService: service)

// Testing with mock
class MockUserService: UserServiceProtocol {
    var mockUsers: [User] = []
    
    func fetchUsers() async throws -> [User] {
        return mockUsers
    }
}

let mockService = MockUserService()
mockService.mockUsers = [/* test data */]
let testViewModel = UserListViewModel(userService: mockService)
```

### 2. Property Injection

```swift
class UserListViewController: UIViewController {
    var viewModel: UserListViewModel!  // Injected after init
    
    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.loadUsers()
    }
}

// Usage
let viewController = UserListViewController()
viewController.viewModel = UserListViewModel(userService: userService)
```

### 3. Method Injection

```swift
class DataProcessor {
    func process(data: Data, using parser: DataParser) {
        // Use injected parser
        let result = parser.parse(data)
    }
}
```

### 4. Service Locator (Anti-pattern, but common)

```swift
// ⚠️ Less preferred - makes dependencies implicit
class ServiceLocator {
    static let shared = ServiceLocator()
    
    private var services: [String: Any] = [:]
    
    func register<T>(_ service: T, for type: T.Type) {
        let key = String(describing: type)
        services[key] = service
    }
    
    func resolve<T>(_ type: T.Type) -> T? {
        let key = String(describing: type)
        return services[key] as? T
    }
}

// Registration
ServiceLocator.shared.register(UserService(), for: UserServiceProtocol.self)

// Usage
class UserListViewModel {
    private let userService: UserServiceProtocol
    
    init() {
        self.userService = ServiceLocator.shared.resolve(UserServiceProtocol.self)!
    }
}
```

---

## Repository Pattern

### Purpose
Abstract data sources (API, database, cache) behind a common interface.

```swift
// MARK: - Domain Layer (Protocol)
protocol UserRepositoryProtocol {
    func fetchUsers() async throws -> [User]
    func fetchUser(id: String) async throws -> User
    func saveUser(_ user: User) async throws
    func deleteUser(id: String) async throws
}

// MARK: - Data Layer (Implementation)
class UserRepository: UserRepositoryProtocol {
    private let remoteDataSource: RemoteUserDataSource
    private let localDataSource: LocalUserDataSource
    private let cacheDataSource: CacheUserDataSource
    
    init(
        remoteDataSource: RemoteUserDataSource,
        localDataSource: LocalUserDataSource,
        cacheDataSource: CacheUserDataSource
    ) {
        self.remoteDataSource = remoteDataSource
        self.localDataSource = localDataSource
        self.cacheDataSource = cacheDataSource
    }
    
    func fetchUsers() async throws -> [User] {
        // 1. Try cache first
        if let cachedUsers = cacheDataSource.getUsers() {
            return cachedUsers
        }
        
        // 2. Try local database
        let localUsers = try await localDataSource.fetchUsers()
        if !localUsers.isEmpty {
            cacheDataSource.saveUsers(localUsers)
            return localUsers
        }
        
        // 3. Fetch from remote API
        let remoteUsers = try await remoteDataSource.fetchUsers()
        
        // 4. Save to local database
        try await localDataSource.saveUsers(remoteUsers)
        
        // 5. Update cache
        cacheDataSource.saveUsers(remoteUsers)
        
        return remoteUsers
    }
    
    func fetchUser(id: String) async throws -> User {
        // Similar multi-layer logic
    }
    
    func saveUser(_ user: User) async throws {
        // Save to all layers
        try await remoteDataSource.saveUser(user)
        try await localDataSource.saveUser(user)
        cacheDataSource.saveUser(user)
    }
    
    func deleteUser(id: String) async throws {
        // Delete from all layers
        try await remoteDataSource.deleteUser(id: id)
        try await localDataSource.deleteUser(id: id)
        cacheDataSource.deleteUser(id: id)
    }
}

// MARK: - Data Sources
class RemoteUserDataSource {
    func fetchUsers() async throws -> [User] {
        // API call
    }
}

class LocalUserDataSource {
    func fetchUsers() async throws -> [User] {
        // Core Data or SQLite
    }
}

class CacheUserDataSource {
    private var cache: [String: User] = [:]
    
    func getUsers() -> [User]? {
        // In-memory cache
    }
}
```

---

## Architectural Decision Making

### Decision Matrix

| Factor | MVC | MVVM | MVVM+C | VIPER | Clean |
|--------|-----|------|--------|-------|-------|
| Simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| Testability | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Scalability | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Learning Curve | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| Boilerplate | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐ |

### When to Use What

**MVC:**
- ✅ Prototypes, MVPs
- ✅ Very simple apps
- ✅ Learning iOS
- ❌ Large teams
- ❌ Complex business logic

**MVVM:**
- ✅ SwiftUI apps
- ✅ Medium to large apps
- ✅ When testability matters
- ✅ Reactive programming
- ❌ Very simple screens

**MVVM + Coordinator:**
- ✅ Large apps with complex navigation
- ✅ Deep linking support
- ✅ A/B testing different flows
- ✅ Multiple entry points
- ❌ Simple linear navigation

**VIPER:**
- ✅ Large enterprise apps
- ✅ Multiple developers/teams
- ✅ Extreme testability requirements
- ❌ Small apps
- ❌ Rapid prototyping

**Clean Architecture:**
- ✅ Multi-platform apps (iOS + Android)
- ✅ Long-term maintenance
- ✅ Framework independence
- ✅ Business logic reuse
- ❌ Small apps
- ❌ Tight deadlines

---

## Interview Questions

### Q: "Why is MVVM better than MVC?"

**A:** MVVM separates presentation logic from view logic:
- **Testability**: ViewModel has no UIKit dependencies, easy to unit test
- **Reusability**: Same ViewModel can work with different Views
- **Separation**: View only displays, ViewModel handles logic
- **SwiftUI**: Natural fit with SwiftUI's declarative approach

However, MVC isn't "bad" - for simple screens, it's perfectly fine. Choose architecture based on complexity.

### Q: "What's the role of a Coordinator?"

**A:** Coordinator handles navigation flow:
- Removes navigation logic from ViewControllers
- Makes ViewControllers reusable (don't know about next screen)
- Easier to handle deep linking
- Can swap flows for A/B testing
- Parent coordinator manages child coordinators

### Q: "How do you decide between MVVM and VIPER?"

**A:** 
- **MVVM**: Good for most apps. Simpler, less boilerplate.
- **VIPER**: Better for large teams/apps where you need extreme separation.

VIPER has more layers (5 vs 3), more protocols, more files. Use it when the benefits outweigh the complexity cost.

### Q: "What's the Repository pattern and why use it?"

**A:** Repository abstracts data sources:
- Single interface for multiple data sources (API, DB, cache)
- Business logic doesn't care where data comes from
- Easy to swap implementations (mock for testing)
- Handles caching strategy internally
- Follows Dependency Inversion Principle

### Q: "How do you structure a large iOS app?"

**A:** 
1. **Modular architecture**: Divide into feature modules
2. **Clear layers**: Presentation → Domain → Data
3. **Dependency injection**: Explicit dependencies
4. **Navigation**: Coordinator pattern for complex flows
5. **Shared code**: Core module for common utilities
6. **Testing**: Unit tests for ViewModels/UseCases, UI tests for flows

### Q: "What are the downsides of Singleton?"

**A:**
- Hard to test (global state)
- Hidden dependencies
- Thread-safety issues
- Can't swap implementations
- Tight coupling

Use dependency injection instead when possible.

### Q: "Explain Dependency Injection and its benefits"

**A:** DI passes dependencies to objects instead of objects creating them:
- **Testability**: Easy to inject mocks
- **Flexibility**: Swap implementations
- **Explicit dependencies**: Clear what object needs
- **Loose coupling**: Object doesn't know about concrete types

Constructor injection is preferred - dependencies are clear and required.

---

This guide covers the architecture patterns you'll need to know. Focus on MVVM and Coordinator - these are most commonly used in modern iOS apps.
