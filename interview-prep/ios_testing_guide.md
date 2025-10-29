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
2. [UI Testing](#ui-testing)
3. [Test-Driven Development (TDD)](#test-driven-development-tdd)
4. [Mocking & Dependency Injection](#mocking--dependency-injection)
5. [Testing Async Code](#testing-async-code)
6. [Testing ViewModels](#testing-viewmodels)
7. [Test Best Practices](#test-best-practices)
8. [Code Coverage](#code-coverage)
9. [Interview Questions](#interview-questions)

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

## Mocking & Dependency Injection

### Protocol-Based Mocking

```swift
// 1. Define protocol
protocol UserServiceProtocol {
    func fetchUsers() async throws -> [User]
}

// 2. Real implementation
class UserService: UserServiceProtocol {
    func fetchUsers() async throws -> [User] {
        // Real network call
    }
}

// 3. Mock implementation
class MockUserService: UserServiceProtocol {
    var mockUsers: [User] = []
    var shouldFail = false
    
    func fetchUsers() async throws -> [User] {
        if shouldFail {
            throw NetworkError.noData
        }
        return mockUsers
    }
}

// 4. ViewModel with DI
class UserViewModel {
    private let userService: UserServiceProtocol
    
    init(userService: UserServiceProtocol = UserService()) {
        self.userService = userService
    }
}

// 5. Test with mock
let mockService = MockUserService()
let viewModel = UserViewModel(userService: mockService)
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

## Interview Questions

**Q: "What's the difference between unit, integration, and UI tests?"**

**A:** 
- Unit: Test individual components in isolation (fast, many)
- Integration: Test components working together (medium)
- UI: Test user flows end-to-end (slow, few)

**Q: "How do you make code testable?"**

**A:**
1. Dependency injection
2. Protocol-based design
3. Single responsibility
4. Avoid singletons
5. Pure functions

**Q: "What's your ideal code coverage?"**

**A:** 80-90% for unit tests, but quality > quantity. Focus on business logic, edge cases, and critical paths.

---

**Key Takeaway:** Testing is about confidence. Good tests let you refactor fearlessly and catch bugs early. Focus on testing behavior, not implementation details.
