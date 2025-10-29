---
layout: page
title: "iOS Testing Guide"
subtitle: "Complete Reference for Senior iOS Engineers"
---

# iOS Testing Guide
## Complete Reference for Senior iOS Engineers

---

## Table of Contents
1. [Unit Testing with XCTest](#unit-testing-with-xctest)
2. [Testing Error Handling](#testing-error-handling)
3. [Mocking & Dependency Injection](#mocking--dependency-injection)
4. [Testing Async Code](#testing-async-code)
5. [UI Testing](#ui-testing)
6. [Testing ViewModels](#testing-viewmodels)
7. [Test-Driven Development (TDD)](#test-driven-development)
8. [Test Best Practices](#test-best-practices)
9. [Code Coverage](#code-coverage)
10. [Interview Questions](#interview-questions)

---

## Unit Testing with XCTest

### Basic Test Structure

```swift
import XCTest
@testable import MyApp  // Access internal types

class UserViewModelTests: XCTestCase {
    
    // MARK: - Properties
    var sut: UserViewModel!  // SUT = System Under Test
    var mockService: MockUserService!
    
    // MARK: - Setup & Teardown
    
    override func setUp() {
        super.setUp()
        
        // Arrange: Create test objects
        mockService = MockUserService()
        sut = UserViewModel(userService: mockService)
    }
    
    override func tearDown() {
        // Clean up
        sut = nil
        mockService = nil
        
        super.tearDown()
    }
    
    // MARK: - Tests
    
    func testFetchUsers_Success_UpdatesUsers() {
        // Arrange
        let expectedUsers = [
            User(id: "1", name: "John", email: "john@example.com")
        ]
        mockService.mockUsers = expectedUsers
        
        // Act
        sut.fetchUsers()
        
        // Assert
        XCTAssertEqual(sut.users.count, 1)
        XCTAssertEqual(sut.users.first?.name, "John")
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.errorMessage)
    }
}
```

### XCTest Assertions

```swift
// Equality
XCTAssertEqual(2 + 2, 4)
XCTAssertNotEqual(2 + 2, 5)

// Boolean
XCTAssertTrue(1 < 2)
XCTAssertFalse(1 > 2)

// Nil
XCTAssertNil(optionalValue)
XCTAssertNotNil("Hello")

// Throws
XCTAssertThrowsError(try someFunctionThatThrows())
XCTAssertNoThrow(try someSafeFunction())

// Custom message
XCTAssertEqual(actual, expected, "Values should match")
```

---

## Testing Error Handling

### Testing That a Function Throws Any Error

```swift
enum ValidationError: Error {
    case invalidEmail
    case passwordTooShort
    case usernameTaken
}

class Validator {
    func validateEmail(_ email: String) throws {
        guard email.contains("@") else {
            throw ValidationError.invalidEmail
        }
    }
}

class ValidatorTests: XCTestCase {
    var sut: Validator!

    override func setUp() {
        super.setUp()
        sut = Validator()
    }

    // Test that function DOES throw an error
    func testValidateEmail_InvalidFormat_ThrowsError() {
        // Act & Assert
        XCTAssertThrowsError(try sut.validateEmail("notanemail")) {
            error in
            // This closure is optional - runs if error is thrown
            print("Error thrown as expected: \(error)")
        }
    }

    // Test that function DOES NOT throw
    func testValidateEmail_ValidFormat_DoesNotThrow() {
        // Act & Assert
        XCTAssertNoThrow(try sut.validateEmail("test@example.com"))
    }
}
```

### Testing That a Specific Error is Thrown

```swift
class ValidatorTests: XCTestCase {

    // Method 1: Using XCTAssertThrowsError with closure
    func testValidateEmail_InvalidFormat_ThrowsInvalidEmailError() {
        // Act & Assert
        XCTAssertThrowsError(try sut.validateEmail("notanemail")) { error in
            // Check the specific error type
            XCTAssertEqual(error as? ValidationError, .invalidEmail)
        }
    }

    // Method 2: Manual do-catch (more control)
    func testValidatePassword_TooShort_ThrowsPasswordTooShortError() {
        do {
            // Act
            try sut.validatePassword("123")

            // Assert - should not reach here
            XCTFail("Expected error to be thrown")
        } catch ValidationError.passwordTooShort {
            // Assert - expected error caught
            XCTAssert(true, "Correct error thrown")
        } catch {
            // Assert - wrong error caught
            XCTFail("Wrong error type: \(error)")
        }
    }

    // Method 3: Using guard with XCTUnwrap
    func testValidateUsername_Taken_ThrowsUsernameTakenError() throws {
        // Arrange
        var caughtError: Error?

        // Act
        do {
            try sut.validateUsername("admin")
        } catch {
            caughtError = error
        }

        // Assert
        let validationError = try XCTUnwrap(caughtError as? ValidationError)
        XCTAssertEqual(validationError, .usernameTaken)
    }
}
```

### Testing Throwing Functions with Return Values

```swift
class DataParser {
    enum ParseError: Error {
        case invalidFormat
        case missingData
    }

    func parseUser(from data: Data) throws -> User {
        guard !data.isEmpty else {
            throw ParseError.missingData
        }

        guard let user = try? JSONDecoder().decode(User.self, from: data) else {
            throw ParseError.invalidFormat
        }

        return user
    }
}

class DataParserTests: XCTestCase {
    var sut: DataParser!

    override func setUp() {
        super.setUp()
        sut = DataParser()
    }

    // Test successful parse (no throw)
    func testParseUser_ValidData_ReturnsUser() throws {
        // Arrange
        let json = """
        {
            "id": "1",
            "name": "John",
            "email": "john@example.com"
        }
        """.data(using: .utf8)!

        // Act
        let user = try sut.parseUser(from: json)

        // Assert
        XCTAssertEqual(user.id, "1")
        XCTAssertEqual(user.name, "John")
        XCTAssertEqual(user.email, "john@example.com")
    }

    // Test throws on empty data
    func testParseUser_EmptyData_ThrowsMissingDataError() {
        // Arrange
        let emptyData = Data()

        // Act & Assert
        XCTAssertThrowsError(try sut.parseUser(from: emptyData)) { error in
            XCTAssertEqual(error as? DataParser.ParseError, .missingData)
        }
    }

    // Test throws on invalid format
    func testParseUser_InvalidJSON_ThrowsInvalidFormatError() {
        // Arrange
        let invalidJSON = "not json".data(using: .utf8)!

        // Act & Assert
        XCTAssertThrowsError(try sut.parseUser(from: invalidJSON)) { error in
            XCTAssertEqual(error as? DataParser.ParseError, .invalidFormat)
        }
    }
}
```

### Testing Async Throwing Functions

```swift
class NetworkService {
    enum NetworkError: Error {
        case invalidURL
        case noData
        case serverError(statusCode: Int)
    }

    func fetchUser(id: String) async throws -> User {
        guard let url = URL(string: "https://api.example.com/users/\(id)") else {
            throw NetworkError.invalidURL
        }

        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.noData
        }

        guard httpResponse.statusCode == 200 else {
            throw NetworkError.serverError(statusCode: httpResponse.statusCode)
        }

        return try JSONDecoder().decode(User.self, from: data)
    }
}

class NetworkServiceTests: XCTestCase {
    var sut: NetworkService!

    override func setUp() {
        super.setUp()
        sut = NetworkService()
    }

    // Test async throwing function
    func testFetchUser_Success_ReturnsUser() async throws {
        // Note: This requires mocking URLSession (see Mocking section)
        // For now, showing the test structure

        // Act
        let user = try await sut.fetchUser(id: "123")

        // Assert
        XCTAssertNotNil(user)
    }

    // Test that async function throws
    func testFetchUser_InvalidID_ThrowsError() async {
        // Act & Assert
        do {
            _ = try await sut.fetchUser(id: "invalid")
            XCTFail("Should have thrown error")
        } catch {
            // Expected to throw
            XCTAssert(true)
        }
    }

    // Test specific async error
    func testFetchUser_ServerError_ThrowsServerError() async {
        do {
            _ = try await sut.fetchUser(id: "error")
            XCTFail("Should have thrown error")
        } catch NetworkService.NetworkError.serverError(let statusCode) {
            XCTAssertEqual(statusCode, 500)
        } catch {
            XCTFail("Wrong error type: \(error)")
        }
    }
}
```

### Testing Error Messages and LocalizedError

```swift
enum UserError: LocalizedError {
    case notFound
    case unauthorized
    case serverError(message: String)

    var errorDescription: String? {
        switch self {
        case .notFound:
            return "User not found"
        case .unauthorized:
            return "Not authorized to access this resource"
        case .serverError(let message):
            return "Server error: \(message)"
        }
    }
}

class UserErrorTests: XCTestCase {

    func testUserError_NotFound_HasCorrectMessage() {
        // Arrange
        let error = UserError.notFound

        // Assert
        XCTAssertEqual(error.errorDescription, "User not found")
    }

    func testUserError_ServerError_HasCorrectMessage() {
        // Arrange
        let error = UserError.serverError(message: "Database unavailable")

        // Assert
        XCTAssertEqual(error.errorDescription, "Server error: Database unavailable")
    }

    // Testing that ViewModel handles error properly
    func testViewModel_FetchFails_SetsErrorMessage() async {
        // Arrange
        let mockService = MockUserService()
        mockService.shouldFail = true
        mockService.errorToThrow = UserError.notFound
        let viewModel = UserViewModel(service: mockService)

        // Act
        await viewModel.fetchUser(id: "123")

        // Assert
        XCTAssertEqual(viewModel.errorMessage, "User not found")
    }
}
```

---

## Mocking & Dependency Injection

### Protocol-Based Mocking

```swift
// 1. Define protocol
protocol UserServiceProtocol {
    func fetchUsers() async throws -> [User]
    func createUser(_ user: User) async throws -> User
    func deleteUser(id: String) async throws
}

// 2. Real implementation
class UserService: UserServiceProtocol {
    func fetchUsers() async throws -> [User] {
        // Real network call
        let (data, _) = try await URLSession.shared.data(from: apiURL)
        return try JSONDecoder().decode([User].self, from: data)
    }

    func createUser(_ user: User) async throws -> User {
        // Real network call
        // ...
    }

    func deleteUser(id: String) async throws {
        // Real network call
        // ...
    }
}

// 3. Mock implementation
class MockUserService: UserServiceProtocol {
    // Mock data to return
    var mockUsers: [User] = []
    var mockCreatedUser: User?

    // Control behavior
    var shouldFail = false
    var errorToThrow: Error = NetworkError.noData

    // Track calls (spy pattern)
    var fetchUsersCalled = false
    var createUserCalledWith: User?
    var deleteUserCalledWith: String?
    var fetchUsersCallCount = 0

    func fetchUsers() async throws -> [User] {
        fetchUsersCalled = true
        fetchUsersCallCount += 1

        if shouldFail {
            throw errorToThrow
        }
        return mockUsers
    }

    func createUser(_ user: User) async throws -> User {
        createUserCalledWith = user

        if shouldFail {
            throw errorToThrow
        }

        return mockCreatedUser ?? user
    }

    func deleteUser(id: String) async throws {
        deleteUserCalledWith = id

        if shouldFail {
            throw errorToThrow
        }
    }
}

// 4. ViewModel with DI
class UserViewModel {
    private let userService: UserServiceProtocol
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    init(userService: UserServiceProtocol = UserService()) {
        self.userService = userService
    }

    func fetchUsers() async {
        isLoading = true
        errorMessage = nil

        do {
            users = try await userService.fetchUsers()
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }
}

// 5. Test with mock
class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!
    var mockService: MockUserService!

    override func setUp() {
        super.setUp()
        mockService = MockUserService()
        sut = UserViewModel(userService: mockService)
    }

    func testFetchUsers_Success_UpdatesUsers() async {
        // Arrange
        let expectedUsers = [
            User(id: "1", name: "John", email: "john@example.com"),
            User(id: "2", name: "Jane", email: "jane@example.com")
        ]
        mockService.mockUsers = expectedUsers

        // Act
        await sut.fetchUsers()

        // Assert
        XCTAssertEqual(sut.users.count, 2)
        XCTAssertEqual(sut.users.first?.name, "John")
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.errorMessage)
        XCTAssertTrue(mockService.fetchUsersCalled)
        XCTAssertEqual(mockService.fetchUsersCallCount, 1)
    }

    func testFetchUsers_Failure_SetsErrorMessage() async {
        // Arrange
        mockService.shouldFail = true
        mockService.errorToThrow = UserError.notFound

        // Act
        await sut.fetchUsers()

        // Assert
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertNotNil(sut.errorMessage)
        XCTAssertFalse(sut.isLoading)
    }
}
```

### Advanced Mocking: Stub vs Spy vs Mock

```swift
// Stub: Returns predetermined values
class StubUserService: UserServiceProtocol {
    private let users: [User]

    init(users: [User]) {
        self.users = users
    }

    func fetchUsers() async throws -> [User] {
        return users
    }
}

// Spy: Records interactions for verification
class SpyUserService: UserServiceProtocol {
    var fetchUsersCallCount = 0
    var createUserCalls: [User] = []
    var deleteUserCalls: [String] = []

    func fetchUsers() async throws -> [User] {
        fetchUsersCallCount += 1
        return []
    }

    func createUser(_ user: User) async throws -> User {
        createUserCalls.append(user)
        return user
    }

    func deleteUser(id: String) async throws {
        deleteUserCalls.append(id)
    }
}

// Mock: Stub + Spy + behavior verification
class MockUserService: UserServiceProtocol {
    var expectedFetchCallCount = 0
    var actualFetchCallCount = 0

    func fetchUsers() async throws -> [User] {
        actualFetchCallCount += 1
        return []
    }

    func verify() {
        assert(actualFetchCallCount == expectedFetchCallCount,
               "Expected \(expectedFetchCallCount) calls but got \(actualFetchCallCount)")
    }
}

// Usage in tests
func testViewModel_LoadsUsersOnInit() async {
    // Arrange
    let spy = SpyUserService()
    let viewModel = UserViewModel(userService: spy)

    // Act
    await viewModel.fetchUsers()

    // Assert - verify interactions
    XCTAssertEqual(spy.fetchUsersCallCount, 1)
}
```

### Mocking URLSession

```swift
// 1. Protocol wrapper for URLSession
protocol URLSessionProtocol {
    func data(from url: URL) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol {}

// 2. Network service using protocol
class NetworkService {
    private let session: URLSessionProtocol

    init(session: URLSessionProtocol = URLSession.shared) {
        self.session = session
    }

    func fetchData(from url: URL) async throws -> Data {
        let (data, _) = try await session.data(from: url)
        return data
    }
}

// 3. Mock URLSession
class MockURLSession: URLSessionProtocol {
    var mockData: Data?
    var mockResponse: URLResponse?
    var mockError: Error?

    func data(from url: URL) async throws -> (Data, URLResponse) {
        if let error = mockError {
            throw error
        }

        guard let data = mockData,
              let response = mockResponse else {
            throw URLError(.badServerResponse)
        }

        return (data, response)
    }
}

// 4. Test with mocked URLSession
class NetworkServiceTests: XCTestCase {
    var sut: NetworkService!
    var mockSession: MockURLSession!

    override func setUp() {
        super.setUp()
        mockSession = MockURLSession()
        sut = NetworkService(session: mockSession)
    }

    func testFetchData_Success_ReturnsData() async throws {
        // Arrange
        let expectedData = "test".data(using: .utf8)!
        let url = URL(string: "https://example.com")!
        let response = HTTPURLResponse(
            url: url,
            statusCode: 200,
            httpVersion: nil,
            headerFields: nil
        )!

        mockSession.mockData = expectedData
        mockSession.mockResponse = response

        // Act
        let data = try await sut.fetchData(from: url)

        // Assert
        XCTAssertEqual(data, expectedData)
    }

    func testFetchData_NetworkError_ThrowsError() async {
        // Arrange
        let url = URL(string: "https://example.com")!
        mockSession.mockError = URLError(.notConnectedToInternet)

        // Act & Assert
        do {
            _ = try await sut.fetchData(from: url)
            XCTFail("Should have thrown error")
        } catch {
            XCTAssert(error is URLError)
        }
    }
}
```

---

## Testing Async Code

### async/await Tests

```swift
func testAsyncFunction() async throws {
    let result = try await fetchData()
    XCTAssertEqual(result.count, 10)
}

// With expectations
func testAsyncWithExpectation() {
    let expectation = expectation(description: "Completes")
    
    Task {
        let result = try await fetchData()
        XCTAssertEqual(result.count, 10)
        expectation.fulfill()
    }
    
    wait(for: [expectation], timeout: 5)
}
```

---

## Testing Error Scenarios

### Comprehensive Error Testing

```swift
import XCTest
@testable import MyApp

class ErrorHandlingTests: XCTestCase {

    // Test specific error types
    func testNetworkErrorMapping() {
        // Test each error case
        let testCases: [(URLError.Code, NetworkError)] = [
            (.notConnectedToInternet, .noConnection),
            (.timedOut, .timeout),
            (.cannotFindHost, .invalidURL),
            (.badServerResponse, .invalidResponse)
        ]

        for (urlErrorCode, expectedNetworkError) in testCases {
            let urlError = URLError(urlErrorCode)
            let mappedError = NetworkError.from(urlError)

            XCTAssertEqual(
                mappedError,
                expectedNetworkError,
                "URLError code \(urlErrorCode) should map to \(expectedNetworkError)"
            )
        }
    }

    // Test error recovery
    func testRetryMechanism() async throws {
        // Arrange
        let service = MockService()
        service.failureCount = 2  // Fail first 2 attempts
        let retryHandler = RetryHandler(maxAttempts: 3)

        // Act
        let result = try await retryHandler.execute {
            try await service.fetchData()
        }

        // Assert
        XCTAssertNotNil(result)
        XCTAssertEqual(service.callCount, 3)  // Called 3 times
    }

    // Test error propagation
    func testErrorPropagation() async {
        let viewModel = DataViewModel()
        let expectedError = NetworkError.serverError(statusCode: 500)

        viewModel.service = MockService(error: expectedError)

        await viewModel.loadData()

        XCTAssertNil(viewModel.data)
        XCTAssertNotNil(viewModel.error)
        XCTAssertEqual(viewModel.error as? NetworkError, expectedError)
    }
}
```

### Testing Throwing Functions

```swift
class ThrowingFunctionTests: XCTestCase {

    func testFunctionThrowsCorrectError() {
        // Using XCTAssertThrowsError
        XCTAssertThrowsError(
            try validateEmail("invalid"),
            "Should throw validation error"
        ) { error in
            // Verify specific error type
            XCTAssertEqual(error as? ValidationError, ValidationError.invalidFormat)
        }
    }

    func testFunctionDoesNotThrow() {
        XCTAssertNoThrow(
            try validateEmail("user@example.com"),
            "Valid email should not throw"
        )
    }

    func testMultipleErrorConditions() {
        let testCases: [(input: String, expectedError: ValidationError?)] = [
            ("", .empty),
            ("a", .tooShort),
            ("user@", .invalidFormat),
            ("user@example.com", nil)  // Valid case
        ]

        for testCase in testCases {
            if let expectedError = testCase.expectedError {
                XCTAssertThrowsError(try validate(testCase.input)) { error in
                    XCTAssertEqual(error as? ValidationError, expectedError)
                }
            } else {
                XCTAssertNoThrow(try validate(testCase.input))
            }
        }
    }
}
```

### Testing Error UI States

```swift
class ErrorUITests: XCTestCase {
    var sut: ErrorViewController!

    override func setUp() {
        super.setUp()
        let storyboard = UIStoryboard(name: "Main", bundle: nil)
        sut = storyboard.instantiateViewController(withIdentifier: "ErrorViewController") as? ErrorViewController
        sut.loadViewIfNeeded()
    }

    func testErrorStateDisplay() {
        // Given
        let error = NetworkError.noConnection

        // When
        sut.showError(error)

        // Then
        XCTAssertFalse(sut.contentView.isHidden)
        XCTAssertTrue(sut.errorView.isHidden == false)
        XCTAssertEqual(sut.errorLabel.text, error.localizedDescription)
        XCTAssertTrue(sut.retryButton.isEnabled)
    }

    func testErrorRecovery() {
        // Given
        let expectation = expectation(description: "Retry called")
        sut.onRetry = {
            expectation.fulfill()
        }

        // When
        sut.showError(NetworkError.timeout)
        sut.retryButton.sendActions(for: .touchUpInside)

        // Then
        wait(for: [expectation], timeout: 1)
    }
}
```

### Testing Error Analytics

```swift
class ErrorAnalyticsTests: XCTestCase {
    var mockAnalytics: MockAnalytics!
    var errorReporter: ErrorReporter!

    override func setUp() {
        super.setUp()
        mockAnalytics = MockAnalytics()
        errorReporter = ErrorReporter(analytics: mockAnalytics)
    }

    func testErrorLogging() {
        // Given
        let error = NetworkError.serverError(statusCode: 503)
        let context = ["user_id": "12345", "action": "fetch_profile"]

        // When
        errorReporter.report(error, context: context)

        // Then
        XCTAssertEqual(mockAnalytics.loggedEvents.count, 1)

        let event = mockAnalytics.loggedEvents.first
        XCTAssertEqual(event?.name, "error_occurred")
        XCTAssertEqual(event?.parameters["error_type"] as? String, "NetworkError")
        XCTAssertEqual(event?.parameters["error_code"] as? Int, 503)
        XCTAssertEqual(event?.parameters["user_id"] as? String, "12345")
    }
}
```

---

## UI Testing

### UI Testing Basics

```swift
import XCUITest

class MyAppUITests: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        super.setUp()

        continueAfterFailure = false
        app = XCUIApplication()
        app.launch()
    }

    func testLoginFlow() {
        // Find UI elements
        let usernameField = app.textFields["UsernameField"]
        let passwordField = app.secureTextFields["PasswordField"]
        let loginButton = app.buttons["LoginButton"]

        // Interact with elements
        XCTAssertTrue(usernameField.exists)
        usernameField.tap()
        usernameField.typeText("testuser")

        passwordField.tap()
        passwordField.typeText("password123")

        loginButton.tap()

        // Verify navigation
        let welcomeLabel = app.staticTexts["WelcomeLabel"]
        XCTAssertTrue(welcomeLabel.waitForExistence(timeout: 5))
        XCTAssertEqual(welcomeLabel.label, "Welcome, testuser!")
    }

    func testErrorHandling() {
        let loginButton = app.buttons["LoginButton"]

        // Tap without entering credentials
        loginButton.tap()

        // Verify error message
        let errorAlert = app.alerts["Error"]
        XCTAssertTrue(errorAlert.waitForExistence(timeout: 2))

        let errorMessage = errorAlert.staticTexts["Please enter username and password"]
        XCTAssertTrue(errorMessage.exists)

        // Dismiss alert
        errorAlert.buttons["OK"].tap()
    }
}
```

### Accessibility Identifiers for Testing

```swift
// In your view code (UIKit)
class LoginViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        // Set accessibility identifiers for UI testing
        usernameTextField.accessibilityIdentifier = "UsernameField"
        passwordTextField.accessibilityIdentifier = "PasswordField"
        loginButton.accessibilityIdentifier = "LoginButton"
        welcomeLabel.accessibilityIdentifier = "WelcomeLabel"
    }
}

// SwiftUI
struct LoginView: View {
    var body: some View {
        VStack {
            TextField("Username", text: $username)
                .accessibilityIdentifier("UsernameField")

            SecureField("Password", text: $password)
                .accessibilityIdentifier("PasswordField")

            Button("Login") {
                login()
            }
            .accessibilityIdentifier("LoginButton")
        }
    }
}
```

### UI Test Queries and Interactions

```swift
// Buttons
app.buttons["LoginButton"]
app.buttons.containing(.staticText, identifier: "Login").firstMatch

// Text fields
app.textFields["UsernameField"]
app.secureTextFields["PasswordField"]

// Labels
app.staticTexts["WelcomeLabel"]

// Tables
app.tables.cells.element(boundBy: 0)
app.tables.cells["UserCell-1"]

// Navigation
app.navigationBars["Settings"]
app.navigationBars.buttons["Back"]

// Tabs
app.tabBars.buttons["Profile"]

// Scrolling
let table = app.tables.firstMatch
table.swipeUp()

let collectionView = app.collectionViews.firstMatch
collectionView.cells.element(boundBy: 5).swipeLeft()

// Waiting for elements
let element = app.buttons["SubmitButton"]
XCTAssertTrue(element.waitForExistence(timeout: 5))

// Predicates
let predicate = NSPredicate(format: "label CONTAINS[c] 'Welcome'")
let welcome = app.staticTexts.containing(predicate).firstMatch
XCTAssertTrue(welcome.exists)
```

### Launch Arguments and Environment

```swift
class MyAppUITests: XCTestCase {

    func testWithMockData() {
        // Pass launch arguments
        app.launchArguments = ["-UITesting", "-MockData"]

        // Pass environment variables
        app.launchEnvironment = [
            "MOCK_API": "true",
            "API_BASE_URL": "https://mock.example.com"
        ]

        app.launch()

        // Now app can check for these in code:
        // if ProcessInfo.processInfo.arguments.contains("-UITesting") {
        //     // Use mock data
        // }
    }

    func testWithTestUser() {
        app.launchEnvironment = ["TEST_USER_ID": "123"]
        app.launch()

        // App can read: ProcessInfo.processInfo.environment["TEST_USER_ID"]
    }
}
```

---

## Testing ViewModels

### Testing @Published Properties with Combine

```swift
import Combine

class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let service: UserServiceProtocol
    private var cancellables = Set<AnyCancellable>()

    init(service: UserServiceProtocol) {
        self.service = service
    }

    func fetchUsers() async {
        isLoading = true
        errorMessage = nil

        do {
            users = try await service.fetchUsers()
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }
}

class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!
    var mockService: MockUserService!
    var cancellables: Set<AnyCancellable>!

    override func setUp() {
        super.setUp()
        mockService = MockUserService()
        sut = UserViewModel(service: mockService)
        cancellables = []
    }

    // Test using expectations
    func testFetchUsers_Success_PublishesUsers() {
        // Arrange
        let expectedUsers = [User(id: "1", name: "John", email: "john@example.com")]
        mockService.mockUsers = expectedUsers

        let expectation = expectation(description: "Users published")
        var receivedUsers: [User] = []

        // Subscribe to @Published property
        sut.$users
            .dropFirst() // Skip initial empty value
            .sink { users in
                receivedUsers = users
                expectation.fulfill()
            }
            .store(in: &cancellables)

        // Act
        Task {
            await sut.fetchUsers()
        }

        // Assert
        wait(for: [expectation], timeout: 2)
        XCTAssertEqual(receivedUsers.count, 1)
        XCTAssertEqual(receivedUsers.first?.name, "John")
    }

    // Test loading state transitions
    func testFetchUsers_LoadingStates() {
        // Arrange
        let expectation = expectation(description: "Loading states")
        expectation.expectedFulfillmentCount = 2
        var loadingStates: [Bool] = []

        sut.$isLoading
            .dropFirst() // Skip initial value
            .sink { isLoading in
                loadingStates.append(isLoading)
                expectation.fulfill()
            }
            .store(in: &cancellables)

        // Act
        Task {
            await sut.fetchUsers()
        }

        // Assert
        wait(for: [expectation], timeout: 2)
        XCTAssertEqual(loadingStates, [true, false])
    }

    // Test error handling
    func testFetchUsers_Failure_PublishesError() async {
        // Arrange
        mockService.shouldFail = true
        mockService.errorToThrow = UserError.notFound

        // Act
        await sut.fetchUsers()

        // Assert
        XCTAssertNotNil(sut.errorMessage)
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertFalse(sut.isLoading)
    }
}
```

### Testing State Machines

```swift
enum LoadingState<T> {
    case idle
    case loading
    case success(T)
    case failure(Error)
}

class DataViewModel: ObservableObject {
    @Published var state: LoadingState<[User]> = .idle

    func loadData() async {
        state = .loading

        do {
            let users = try await fetchUsers()
            state = .success(users)
        } catch {
            state = .failure(error)
        }
    }
}

class DataViewModelTests: XCTestCase {

    func testLoadData_StateTransitions() async {
        // Arrange
        var states: [String] = []
        let sut = DataViewModel()

        let expectation = expectation(description: "State changes")
        expectation.expectedFulfillmentCount = 3

        sut.$state
            .sink { state in
                switch state {
                case .idle:
                    states.append("idle")
                case .loading:
                    states.append("loading")
                case .success:
                    states.append("success")
                case .failure:
                    states.append("failure")
                }
                expectation.fulfill()
            }
            .store(in: &cancellables)

        // Act
        await sut.loadData()

        // Assert
        await fulfillment(of: [expectation], timeout: 2)
        XCTAssertEqual(states, ["idle", "loading", "success"])
    }
}
```

---

## Test-Driven Development

### Red → Green → Refactor

```swift
// 1. RED - Write failing test
func testAdd_TwoNumbers_ReturnsSum() {
    let calculator = Calculator()
    let result = calculator.add(2, 3)
    XCTAssertEqual(result, 5)
}

// 2. GREEN - Write minimum code
class Calculator {
    func add(_ a: Int, _ b: Int) -> Int {
        return a + b
    }
}

// 3. REFACTOR - Improve code (tests still pass)
```

---

## Test Best Practices

### The FIRST Principles

Tests should be **FIRST**:
- **F**ast - Run quickly (milliseconds)
- **I**ndependent - No dependencies between tests
- **R**epeatable - Same result every time
- **S**elf-validating - Pass or fail, no manual verification
- **T**imely - Written alongside (or before) production code

### Test Naming Convention

```swift
// Pattern: test_UnitOfWork_StateUnderTest_ExpectedBehavior

func testFetchUsers_WhenNetworkFails_SetsErrorMessage() { }

func testValidateEmail_WithInvalidFormat_ThrowsError() { }

func testLoginButton_WhenCredentialsEmpty_IsDisabled() { }

// Or: Given_When_Then pattern
func testGivenEmptyCart_WhenAddingItem_ThenCartCountIsOne() { }
```

### AAA Pattern (Arrange-Act-Assert)

```swift
func testAddItem_UpdatesCart() {
    // Arrange - Set up test data and conditions
    let cart = ShoppingCart()
    let item = Item(id: "1", name: "Widget", price: 9.99)

    // Act - Execute the code being tested
    cart.add(item)

    // Assert - Verify the expected outcome
    XCTAssertEqual(cart.items.count, 1)
    XCTAssertEqual(cart.total, 9.99)
}
```

### Test Organization

```swift
class UserViewModelTests: XCTestCase {

    // MARK: - Properties
    var sut: UserViewModel!  // System Under Test
    var mockService: MockUserService!

    // MARK: - Setup & Teardown
    override func setUp() {
        super.setUp()
        mockService = MockUserService()
        sut = UserViewModel(service: mockService)
    }

    override func tearDown() {
        sut = nil
        mockService = nil
        super.tearDown()
    }

    // MARK: - Init Tests
    func testInit_SetsDefaultValues() {
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.errorMessage)
    }

    // MARK: - Fetch Users Tests
    func testFetchUsers_Success_UpdatesUsers() async { }
    func testFetchUsers_Failure_SetsError() async { }
    func testFetchUsers_Loading_SetsLoadingState() async { }

    // MARK: - Delete User Tests
    func testDeleteUser_Success_RemovesUser() async { }
    func testDeleteUser_Failure_KeepsUser() async { }
}
```

### What to Test (and What Not to Test)

```swift
// ✅ DO TEST:
// - Business logic
// - Edge cases and error conditions
// - State transitions
// - Data transformations
// - User interactions (UI tests)

// ❌ DON'T TEST:
// - Framework code (UIKit, SwiftUI, Foundation)
// - Third-party libraries
// - Private implementation details
// - Getters/setters with no logic
// - Generated code (like Codable synthesis)

// Example: Test business logic, not implementation
// ❌ Bad - testing implementation
func testFetchUsers_CallsServiceFetchMethod() {
    sut.fetchUsers()
    XCTAssertTrue(mockService.fetchUsersCalled)  // Testing implementation
}

// ✅ Good - testing behavior
func testFetchUsers_UpdatesUsersList() async {
    mockService.mockUsers = [testUser]
    await sut.fetchUsers()
    XCTAssertEqual(sut.users.count, 1)  // Testing outcome
}
```

### Test Isolation

```swift
// ❌ Bad - tests depend on each other
class BadTests: XCTestCase {
    var sharedData: [User] = []

    func test1_AddUser() {
        sharedData.append(User(id: "1", name: "John"))
        XCTAssertEqual(sharedData.count, 1)
    }

    func test2_RemoveUser() {
        // Depends on test1 running first!
        sharedData.removeAll()
        XCTAssertTrue(sharedData.isEmpty)
    }
}

// ✅ Good - each test is independent
class GoodTests: XCTestCase {
    func testAddUser_IncreasesCount() {
        let users: [User] = []
        let updated = users + [User(id: "1", name: "John")]
        XCTAssertEqual(updated.count, 1)
    }

    func testRemoveUser_DecreasesCount() {
        var users = [User(id: "1", name: "John")]
        users.removeAll()
        XCTAssertTrue(users.isEmpty)
    }
}
```

### Testing Edge Cases

```swift
func testDivide() {
    // Test normal case
    XCTAssertEqual(calculator.divide(10, 2), 5)

    // Test edge cases
    XCTAssertEqual(calculator.divide(0, 5), 0)  // Zero numerator
    XCTAssertThrowsError(try calculator.divide(5, 0))  // Division by zero
    XCTAssertEqual(calculator.divide(-10, 2), -5)  // Negative numbers
    XCTAssertEqual(calculator.divide(1, 3), 0.333, accuracy: 0.001)  // Floating point
}

// Boundary testing
func testArrayAccess() {
    let array = [1, 2, 3]

    XCTAssertEqual(array[0], 1)  // First element
    XCTAssertEqual(array[2], 3)  // Last element
    // Test out of bounds is handled
}

// Null/nil testing
func testProcessUser() {
    XCTAssertNoThrow(processor.process(validUser))
    XCTAssertThrowsError(try processor.process(nil))
}
```

### Avoiding Flaky Tests

```swift
// ❌ Flaky - depends on timing
func testAsyncOperation() {
    startAsyncTask()
    sleep(1)  // Bad! Might not be enough time
    XCTAssertTrue(taskCompleted)
}

// ✅ Reliable - uses expectations
func testAsyncOperation() {
    let expectation = expectation(description: "Task completes")

    startAsyncTask {
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 5)
}

// ❌ Flaky - depends on network
func testFetchUser() async throws {
    let user = try await service.fetchUser(id: "123")  // Real network call
    XCTAssertEqual(user.name, "John")
}

// ✅ Reliable - uses mock
func testFetchUser() async throws {
    mockService.mockUser = User(id: "123", name: "John")
    let user = try await mockService.fetchUser(id: "123")
    XCTAssertEqual(user.name, "John")
}
```

---

## Code Coverage

### Measuring Code Coverage

```swift
// In Xcode:
// 1. Edit Scheme → Test → Options → Code Coverage
// 2. Check "Gather coverage for some targets"
// 3. Run tests
// 4. View Report Navigator → Coverage tab

// From command line:
// xcodebuild test -scheme MyApp -enableCodeCoverage YES
```

### Understanding Coverage Metrics

```
Line Coverage: Percentage of lines executed
Branch Coverage: Percentage of decision branches taken
Function Coverage: Percentage of functions called
```

### Coverage Guidelines

```swift
// Target coverage by layer:
// - Business Logic: 80-90%
// - ViewModels: 70-85%
// - Services: 70-80%
// - UI Code: 30-50% (mostly UI tests)
// - Models/DTOs: Low priority (simple structs)

// Focus on:
// ✅ Critical business logic
// ✅ Error handling paths
// ✅ Complex algorithms
// ✅ Edge cases

// Don't obsess over:
// ❌ Simple getters/setters
// ❌ Generated code
// ❌ Framework code
// ❌ Simple view controllers with no logic
```

### Example: Testing All Branches

```swift
func calculateDiscount(for price: Double, quantity: Int) -> Double {
    guard price > 0, quantity > 0 else {
        return 0
    }

    if quantity >= 10 {
        return price * 0.2  // 20% discount
    } else if quantity >= 5 {
        return price * 0.1  // 10% discount
    } else {
        return 0  // No discount
    }
}

class DiscountTests: XCTestCase {
    // Test all branches for 100% coverage

    func testCalculateDiscount_InvalidPrice_ReturnsZero() {
        XCTAssertEqual(calculateDiscount(for: -10, quantity: 5), 0)
        XCTAssertEqual(calculateDiscount(for: 0, quantity: 5), 0)
    }

    func testCalculateDiscount_InvalidQuantity_ReturnsZero() {
        XCTAssertEqual(calculateDiscount(for: 100, quantity: -1), 0)
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 0), 0)
    }

    func testCalculateDiscount_QuantityTenOrMore_ReturnsTwentyPercent() {
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 10), 20)
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 15), 20)
    }

    func testCalculateDiscount_QuantityFiveToNine_ReturnsTenPercent() {
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 5), 10)
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 9), 10)
    }

    func testCalculateDiscount_QuantityLessThanFive_ReturnsZero() {
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 1), 0)
        XCTAssertEqual(calculateDiscount(for: 100, quantity: 4), 0)
    }
}
```

---

## Interview Questions

### Q: "What's the difference between unit, integration, and UI tests?"

**A:**
- **Unit Tests**: Test individual components in isolation (functions, classes, methods). Fast (milliseconds), run frequently, most numerous. Mock dependencies.
- **Integration Tests**: Test how multiple components work together (ViewModel + Service, Service + Database). Medium speed, fewer than unit tests. May use real dependencies.
- **UI Tests**: Test complete user flows end-to-end through the UI. Slow (seconds), run less frequently, fewest in number. Test from user's perspective.

**Test Pyramid**: Many unit tests (base), fewer integration tests (middle), fewest UI tests (top).

### Q: "How do you make code testable?"

**A:**
1. **Dependency Injection**: Pass dependencies instead of creating them internally
2. **Protocol-Based Design**: Use protocols for dependencies so you can mock them
3. **Single Responsibility**: Each class/function does one thing
4. **Avoid Singletons**: Hard to mock and test in isolation
5. **Pure Functions**: Same input always produces same output
6. **Separate Concerns**: Business logic separate from UI

Example:
```swift
// ❌ Not testable - creates URLSession internally
class NetworkService {
    func fetchData() {
        URLSession.shared.dataTask(...)  // Can't mock
    }
}

// ✅ Testable - dependency injection
class NetworkService {
    private let session: URLSessionProtocol

    init(session: URLSessionProtocol = URLSession.shared) {
        self.session = session
    }

    func fetchData() async throws {
        try await session.data(...)  // Can inject mock
    }
}
```

### Q: "What's your ideal code coverage?"**

**A:** 80-90% for unit tests, but **quality > quantity**. Focus on:
- Critical business logic
- Error handling paths
- Complex algorithms
- Edge cases

Don't obsess over 100% coverage. Some code (simple getters, framework code, generated code) doesn't need tests. A well-tested codebase with 80% coverage is better than poorly-tested code with 100% coverage.

### Q: "What's the difference between a mock, stub, and spy?"

**A:**
- **Stub**: Returns predetermined values. Used to control indirect inputs.
  ```swift
  class StubService: ServiceProtocol {
      func fetchData() -> Data { return fixedData }
  }
  ```

- **Mock**: Stub + verification. Asserts methods were called correctly.
  ```swift
  class MockService: ServiceProtocol {
      var fetchDataCalled = false
      func fetchData() -> Data {
          fetchDataCalled = true
          return fixedData
      }
  }
  // Later: XCTAssertTrue(mock.fetchDataCalled)
  ```

- **Spy**: Records interactions for later verification.
  ```swift
  class SpyService: ServiceProtocol {
      var fetchDataCallCount = 0
      var fetchDataParameters: [String] = []
      func fetchData(id: String) -> Data {
          fetchDataCallCount += 1
          fetchDataParameters.append(id)
          return fixedData
      }
  }
  ```

### Q: "How do you test async code in Swift?"

**A:** Three main approaches:

1. **async/await** (Modern, preferred):
```swift
func testFetchData() async throws {
    let data = try await service.fetchData()
    XCTAssertNotNil(data)
}
```

2. **XCTestExpectation**:
```swift
func testFetchData() {
    let expectation = expectation(description: "Fetches data")
    service.fetchData { result in
        XCTAssertNotNil(result)
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 5)
}
```

3. **Combine**:
```swift
func testFetchData() {
    let expectation = expectation(description: "Publishes data")
    service.dataPublisher
        .sink { data in
            XCTAssertNotNil(data)
            expectation.fulfill()
        }
        .store(in: &cancellables)
    wait(for: [expectation], timeout: 5)
}
```

### Q: "What is TDD and what are its benefits?"

**A:** **Test-Driven Development**: Write tests before writing production code.

**Process**:
1. **Red**: Write failing test
2. **Green**: Write minimum code to pass
3. **Refactor**: Improve code while keeping tests green

**Benefits**:
- Forces you to think about requirements first
- Creates testable design (you can't write a test if code isn't testable)
- Built-in regression tests
- Confidence to refactor
- Living documentation (tests show how code should work)

**When to use TDD**:
- Complex business logic
- APIs with clear contracts
- Bug fixes (write test first, then fix)

**When not to use TDD**:
- Prototyping/spike code
- UI layout (better done visually)
- Unclear requirements

### Q: "How do you test a ViewModel that uses @Published properties?"

**A:** Use Combine to observe changes:

```swift
func testViewModel_FetchUpdatesUsers() {
    let expectation = expectation(description: "Users updated")
    var receivedUsers: [User] = []

    viewModel.$users
        .dropFirst() // Skip initial value
        .sink { users in
            receivedUsers = users
            expectation.fulfill()
        }
        .store(in: &cancellables)

    Task {
        await viewModel.fetchUsers()
    }

    wait(for: [expectation], timeout: 2)
    XCTAssertEqual(receivedUsers.count, expectedCount)
}
```

### Q: "What are flaky tests and how do you avoid them?"

**A:** **Flaky tests** pass sometimes and fail other times without code changes.

**Common causes**:
1. **Time-dependent** - Using sleep() or fixed delays
2. **Network-dependent** - Real network calls
3. **Shared state** - Tests affecting each other
4. **Race conditions** - Async timing issues
5. **External dependencies** - Databases, files, APIs

**Solutions**:
- Use mocks instead of real dependencies
- Use expectations instead of sleep()
- Ensure test isolation (setUp/tearDown)
- Use dependency injection
- Avoid global state

### Q: "How do you test error handling?"

**A:** Three main patterns:

**1. XCTAssertThrowsError** (any error):
```swift
XCTAssertThrowsError(try validator.validate(email))
```

**2. XCTAssertThrowsError with closure** (specific error):
```swift
XCTAssertThrowsError(try validator.validate(email)) { error in
    XCTAssertEqual(error as? ValidationError, .invalidFormat)
}
```

**3. do-catch** (full control):
```swift
do {
    try validator.validate(email)
    XCTFail("Should have thrown")
} catch ValidationError.invalidFormat {
    // Expected error
} catch {
    XCTFail("Wrong error: \(error)")
}
```

**Also test**:
- Error messages are correct
- ViewModel handles errors properly (sets errorMessage, etc.)
- UI shows errors to users

### Q: "What's the difference between @testable import and regular import?"

**A:**
- **Regular import**: Only access public API
  ```swift
  import MyApp
  ```

- **@testable import**: Access internal types and methods too
  ```swift
  @testable import MyApp
  ```

**Use @testable when**:
- Testing internal implementation
- Most unit tests

**Use regular import when**:
- Testing public API only
- Testing frameworks/SDKs you distribute
- Ensures you're not exposing internals

**Note**: Private members are still inaccessible even with @testable.

### Q: "How do you test view controllers/SwiftUI views?"

**A:**

**View Controllers** (UIKit):
- Test business logic in ViewModel (unit tests)
- Test view updates by checking outlets
- Test user interactions by calling IBActions
- Use UI tests for complex flows

```swift
func testLoginButton_WhenTapped_CallsViewModel() {
    let vc = LoginViewController()
    vc.loginButton.sendActions(for: .touchUpInside)
    XCTAssertTrue(mockViewModel.loginCalled)
}
```

**SwiftUI Views**:
- Test ViewModels (most logic should be there)
- Use UI tests for user flows
- Test computed properties if complex
- Use PreviewProvider for visual testing

```swift
func testLoginView_ValidCredentials_EnablesButton() {
    let viewModel = LoginViewModel()
    viewModel.username = "test"
    viewModel.password = "password"
    XCTAssertTrue(viewModel.isLoginEnabled)
}
```

### Q: "What testing frameworks/tools have you used besides XCTest?"

**A:** Common iOS testing tools:
- **Quick/Nimble**: BDD-style testing ("describe", "it", "expect")
- **OCMock/OCMockito**: Objective-C mocking frameworks
- **XCTest**: Built-in Apple framework
- **SnapshotTesting**: Visual regression testing
- **KIF**: Integration/UI testing framework
- **Appium**: Cross-platform UI testing
- **Fastlane**: Test automation and CI/CD

### Q: "How do you handle testing URLSession and network calls?"

**A:**

**Option 1: Protocol wrapper** (preferred):
```swift
protocol URLSessionProtocol {
    func data(from url: URL) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol {}

class MockURLSession: URLSessionProtocol {
    var mockData: Data?
    func data(from url: URL) async throws -> (Data, URLResponse) {
        return (mockData!, mockResponse)
    }
}
```

**Option 2: URLProtocol subclass** (for more complex scenarios):
```swift
class MockURLProtocol: URLProtocol {
    static var mockResponses: [URL: (Data?, URLResponse?, Error?)] = [:]
    // Implement protocol methods...
}
```

**Option 3: Dependency injection** (inject entire network layer):
```swift
protocol NetworkServiceProtocol {
    func fetchUser(id: String) async throws -> User
}
```

---

## Quick Reference

### Test Patterns Cheat Sheet

```swift
// Basic test structure
func testFeature_Condition_ExpectedResult() {
    // Arrange
    let sut = SystemUnderTest()

    // Act
    let result = sut.performAction()

    // Assert
    XCTAssertEqual(result, expected)
}

// Async test
func testAsync() async throws {
    let result = try await asyncFunction()
    XCTAssertNotNil(result)
}

// Test throwing
XCTAssertThrowsError(try throwingFunction())
XCTAssertNoThrow(try safeFunction())

// Test with expectation
let expectation = expectation(description: "Completes")
asyncTask {
    expectation.fulfill()
}
wait(for: [expectation], timeout: 5)

// Test Combine publisher
viewModel.$property
    .dropFirst()
    .sink { value in
        XCTAssertEqual(value, expected)
        expectation.fulfill()
    }
    .store(in: &cancellables)
```

---

**Key Takeaway:** Testing is about confidence. Good tests let you refactor fearlessly and catch bugs early. Focus on testing behavior, not implementation details. Write tests that describe what your code does, not how it does it.
