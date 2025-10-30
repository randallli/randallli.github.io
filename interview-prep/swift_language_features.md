---
layout: page
title: "Swift Language Features Guide"
subtitle: "From Objective-C to Modern Swift"
---

# Swift Language Features Guide
## From Objective-C to Modern Swift

---

## Table of Contents
1. [Optionals](#optionals)
2. [Value Types vs Reference Types](#value-types-vs-reference-types)
3. [Protocols & Protocol-Oriented Programming](#protocols--protocol-oriented-programming)
4. [Generics](#generics)
5. [Closures](#closures)
6. [Error Handling](#error-handling)
7. [Memory Management in Swift](#memory-management-in-swift)
8. [Property Observers & Wrappers](#property-observers--wrappers)
9. [Extensions](#extensions)
10. [Access Control](#access-control)
11. [Enums & Pattern Matching](#enums--pattern-matching)
12. [Modern Concurrency (async/await)](#modern-concurrency-asyncawait)
13. [Collection Operators](#collection-operators)
14. [Swift vs Objective-C Key Differences](#swift-vs-objective-c-key-differences)

---

## Optionals

### The Basics

Coming from Objective-C, optionals are Swift's way of handling `nil` safely at compile-time.

```swift
// Objective-C: nil could be anywhere
NSString *name = nil;
NSInteger length = [name length];  // Crashes at runtime

// Swift: nil is explicit
var name: String? = nil  // Optional String
let length = name?.count // Optional chaining, returns nil safely
```

### Optional Types

```swift
// Optional declaration
var optionalString: String?           // nil by default
var optionalInt: Int? = 42
var optionalArray: [String]? = nil

// Implicitly Unwrapped Optional (use sparingly)
var definitelyHasValue: String!       // Will crash if nil when accessed
// Similar to: __nonnull in Objective-C
```

### Unwrapping Strategies

```swift
var name: String? = "John"

// 1. Optional Binding (if let) - MOST COMMON
if let unwrappedName = name {
    print("Hello, \(unwrappedName)")
} else {
    print("No name provided")
}

// 2. Guard statement - PREFERRED for early exit
func greet(name: String?) {
    guard let unwrappedName = name else {
        print("No name")
        return  // Must exit scope
    }
    
    // unwrappedName available for rest of function
    print("Hello, \(unwrappedName)")
}

// 3. Multiple optional bindings
if let firstName = user.firstName,
   let lastName = user.lastName,
   let age = user.age,
   age >= 18 {
    print("\(firstName) \(lastName) is an adult")
}

// 4. Force unwrap (!) - AVOID unless certain
let definiteValue = name!  // Crashes if nil
// Only use when you're 100% sure it's not nil

// 5. Nil coalescing operator (??)
let displayName = name ?? "Guest"  // Provides default

// 6. Optional chaining
let count = user.profile?.bio?.count  // Returns nil if any link is nil

// 7. Optional pattern matching
if case let name? = optionalName {
    print("Has value: \(name)")
}
```

### Optional Chaining vs Force Unwrapping

```swift
class Person {
    var address: Address?
}

class Address {
    var street: String?
}

let person: Person? = Person()

// ❌ Force unwrapping - crashes if any is nil
let street = person!.address!.street!

// ✅ Optional chaining - returns nil safely
let street = person?.address?.street  // Type: String?

// ✅ With default value
let street = person?.address?.street ?? "Unknown"
```

### Common Patterns

```swift
// Pattern 1: Unwrap and use
func process(data: Data?) {
    guard let data = data else { return }
    // Use data safely
}

// Pattern 2: Map optional value
let uppercased = name?.uppercased()  // String?

// Pattern 3: Unwrap with condition
if let name = name, !name.isEmpty {
    print(name)
}

// Pattern 4: Try optional operations
let url = URL(string: urlString)  // Returns nil if invalid
```

### Interview Question: Optional vs IUO

**Q: "When should you use Implicitly Unwrapped Optionals?"**

**A:** Very rarely. Main use cases:
1. IBOutlets that will be set by Interface Builder
2. Two-phase initialization where you can't initialize in init but it will definitely be set before use
3. Breaking initialization dependency cycles

```swift
// IBOutlet - set by Interface Builder
@IBOutlet weak var tableView: UITableView!

// Two-phase init
class MyViewController: UIViewController {
    var viewModel: ViewModel!  // Set in configure() before use
    
    func configure(viewModel: ViewModel) {
        self.viewModel = viewModel
    }
}
```

---

## Value Types vs Reference Types

### The Fundamental Difference

```swift
// Value Type (struct, enum) - COPIED
struct Point {
    var x: Int
    var y: Int
}

var point1 = Point(x: 0, y: 0)
var point2 = point1      // COPY made
point2.x = 10

print(point1.x)  // 0 - unchanged
print(point2.x)  // 10

// Reference Type (class) - SHARED
class Person {
    var name: String
    init(name: String) { self.name = name }
}

var person1 = Person(name: "John")
var person2 = person1    // SAME instance
person2.name = "Jane"

print(person1.name)  // "Jane" - same object
print(person2.name)  // "Jane"
```

### Struct vs Class Decision Tree

```
Use STRUCT when:
├─ Represents simple data (Point, Size, Range)
├─ Should be compared by value equality
├─ Needs to be copied when passed around
├─ Properties are also value types
└─ No inheritance needed

Use CLASS when:
├─ Needs inheritance
├─ Need to control identity (same vs different instance)
├─ Need deinitializer
├─ Reference semantics required (shared state)
└─ Objective-C interoperability needed
```

### Copy-on-Write (COW)

Swift's collections use copy-on-write optimization:

```swift
// Arrays are value types but use COW
var array1 = [1, 2, 3]
var array2 = array1        // No copy yet (shares storage)

array2.append(4)           // NOW copy is made

// array1 = [1, 2, 3]
// array2 = [1, 2, 3, 4]

// Built-in types with COW: Array, Dictionary, Set, String
```

### Implementing COW in Your Types

```swift
// Advanced pattern for custom COW
final class Storage {
    var data: [Int]
    
    init(data: [Int]) {
        self.data = data
    }
}

struct MyArray {
    private var storage: Storage
    
    init(data: [Int]) {
        self.storage = Storage(data: data)
    }
    
    var data: [Int] {
        get {
            return storage.data
        }
        set {
            // Copy-on-write: only copy if not uniquely owned
            if !isKnownUniquelyReferenced(&storage) {
                storage = Storage(data: storage.data)
            }
            storage.data = newValue
        }
    }
}
```

### Mutability

```swift
// Struct: let = immutable struct AND properties
struct Point {
    var x: Int
    var y: Int
}

let point = Point(x: 0, y: 0)
point.x = 10  // ❌ Error: point is immutable

var mutablePoint = Point(x: 0, y: 0)
mutablePoint.x = 10  // ✅ OK

// Class: let = immutable reference, but properties can change
class Person {
    var name: String
    init(name: String) { self.name = name }
}

let person = Person(name: "John")
person.name = "Jane"  // ✅ OK - reference is constant, not object
person = Person(name: "Bob")  // ❌ Error: can't reassign let reference
```

### Mutating Methods

```swift
struct Point {
    var x: Int
    var y: Int
    
    // Must mark as mutating if it modifies self
    mutating func moveBy(x: Int, y: Int) {
        self.x += x
        self.y += y
    }
}

var point = Point(x: 0, y: 0)
point.moveBy(x: 10, y: 20)  // ✅ OK

let immutablePoint = Point(x: 0, y: 0)
immutablePoint.moveBy(x: 10, y: 20)  // ❌ Error: immutable
```

---

## Protocols & Protocol-Oriented Programming

### Basic Protocol

```swift
// Define behavior contract
protocol Drawable {
    func draw()
}

// Conform to protocol
struct Circle: Drawable {
    func draw() {
        print("Drawing circle")
    }
}

class Square: Drawable {
    func draw() {
        print("Drawing square")
    }
}
```

### Protocol Requirements

```swift
protocol Vehicle {
    // Property requirements
    var numberOfWheels: Int { get }          // Read-only
    var color: String { get set }            // Read-write
    
    // Method requirements
    func startEngine()
    mutating func refuel()  // For value types that need to modify
    
    // Initializer requirements
    init(color: String)
}

struct Car: Vehicle {
    var numberOfWheels: Int { return 4 }
    var color: String
    
    func startEngine() {
        print("Vroom")
    }
    
    mutating func refuel() {
        print("Refueling")
    }
    
    init(color: String) {
        self.color = color
    }
}
```

### Protocol Extensions (The Power!)

```swift
protocol Greetable {
    var name: String { get }
    func greet()
}

// Provide default implementation
extension Greetable {
    func greet() {
        print("Hello, \(name)")
    }
    
    // Add new methods to all conforming types
    func greetLoudly() {
        print("HELLO, \(name.uppercased())!")
    }
}

struct Person: Greetable {
    var name: String
    // Gets greet() and greetLoudly() for free!
}

let person = Person(name: "John")
person.greet()         // "Hello, John"
person.greetLoudly()   // "HELLO, JOHN!"
```

### Protocol Composition

```swift
protocol Named {
    var name: String { get }
}

protocol Aged {
    var age: Int { get }
}

// Compose multiple protocols
func celebrate(person: Named & Aged) {
    print("\(person.name) is \(person.age) years old")
}

// Or create type alias
typealias Person = Named & Aged

func celebrate(person: Person) {
    print("\(person.name) is \(person.age) years old")
}
```

### Associated Types (Like Generics for Protocols)

```swift
protocol Container {
    associatedtype Item
    
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntStack: Container {
    // Type inference determines Item = Int
    var items: [Int] = []
    
    var count: Int {
        return items.count
    }
    
    mutating func append(_ item: Int) {
        items.append(item)
    }
    
    subscript(i: Int) -> Int {
        return items[i]
    }
}

// Generic implementation
struct Stack<Element>: Container {
    var items: [Element] = []
    
    var count: Int {
        return items.count
    }
    
    mutating func append(_ item: Element) {
        items.append(item)
    }
    
    subscript(i: Int) -> Element {
        return items[i]
    }
}
```

### Protocol-Oriented Programming Pattern

```swift
// Define protocol
protocol RequestProtocol {
    var urlRequest: URLRequest { get }
}

// Provide default implementation
extension RequestProtocol {
    func execute() async throws -> Data {
        let (data, _) = try await URLSession.shared.data(for: urlRequest)
        return data
    }
}

// Specific requests just define their URL
struct UserRequest: RequestProtocol {
    let userId: String
    
    var urlRequest: URLRequest {
        let url = URL(string: "https://api.example.com/users/\(userId)")!
        return URLRequest(url: url)
    }
}

// Use it
let request = UserRequest(userId: "123")
let data = try await request.execute()  // Gets execute() from protocol extension
```

### Class-Only Protocols

```swift
// Restrict to reference types only
protocol SomeClassOnlyProtocol: AnyObject {
    func doSomething()
}

// Now only classes can conform
class MyClass: SomeClassOnlyProtocol {
    func doSomething() {}
}

// ❌ Error: structs can't conform
struct MyStruct: SomeClassOnlyProtocol {  // Error!
    func doSomething() {}
}

// Use case: Delegates (need weak reference)
protocol MyDelegate: AnyObject {
    func didComplete()
}

class MyClass {
    weak var delegate: MyDelegate?  // ✅ Can be weak
}
```

---

## Generics

### Basic Generic Functions

```swift
// Non-generic (repetitive)
func swapInts(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

// Generic (works with any type)
func swap<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var x = 5, y = 10
swap(&x, &y)  // x = 10, y = 5

var str1 = "Hello", str2 = "World"
swap(&str1, &str2)  // str1 = "World", str2 = "Hello"
```

### Generic Types

```swift
// Generic stack
struct Stack<Element> {
    private var items: [Element] = []
    
    var isEmpty: Bool {
        return items.isEmpty
    }
    
    mutating func push(_ item: Element) {
        items.append(item)
    }
    
    mutating func pop() -> Element? {
        return items.popLast()
    }
    
    func peek() -> Element? {
        return items.last
    }
}

// Usage
var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
print(intStack.pop())  // Optional(2)

var stringStack = Stack<String>()
stringStack.push("Hello")
```

### Generic Constraints

```swift
// Constrain to types that conform to Equatable
func findIndex<T: Equatable>(of valueToFind: T, in array: [T]) -> Int? {
    for (index, value) in array.enumerated() {
        if value == valueToFind {  // Requires Equatable
            return index
        }
    }
    return nil
}

let index = findIndex(of: "John", in: ["Jane", "John", "Bob"])  // 1

// Multiple constraints
func combine<T: Numeric & Comparable>(a: T, b: T) -> T {
    return a > b ? a : b
}

// Where clause for complex constraints
func allItemsMatch<C1: Container, C2: Container>(
    _ container1: C1,
    _ container2: C2
) -> Bool where C1.Item == C2.Item, C1.Item: Equatable {
    
    guard container1.count == container2.count else { return false }
    
    for i in 0..<container1.count {
        if container1[i] != container2[i] {
            return false
        }
    }
    
    return true
}
```

### Generic Extensions

```swift
// Extend generic types
extension Stack where Element: Equatable {
    func contains(_ item: Element) -> Bool {
        return items.contains(item)
    }
}

// Only Stack<T> where T is Equatable gets this method
let intStack = Stack<Int>()
print(intStack.contains(5))  // ✅ Int is Equatable

// Extend with specific type
extension Stack where Element == String {
    func joinedString() -> String {
        return items.joined(separator: ", ")
    }
}

// Only Stack<String> gets this method
let stringStack = Stack<String>()
print(stringStack.joinedString())
```

### Opaque Types (some keyword)

```swift
// Returns "some" protocol type - hides concrete type
protocol Shape {
    func draw() -> String
}

struct Circle: Shape {
    func draw() -> String { return "○" }
}

struct Square: Shape {
    func draw() -> String { return "□" }
}

// Return specific type but hide which one
func makeShape() -> some Shape {
    return Circle()  // Concrete type hidden
}

let shape = makeShape()
// Compiler knows it's a specific type, but caller doesn't know which
// Allows for optimizations

// Common in SwiftUI
var body: some View {
    Text("Hello")  // Actual type is complex, abstracted as "some View"
}
```

---

## Closures

### Closure Syntax

```swift
// Full syntax
let closure: (Int, Int) -> Int = { (a: Int, b: Int) -> Int in
    return a + b
}

// Inferred types
let closure = { (a, b) in
    return a + b
}

// Implicit return (single expression)
let closure = { (a, b) in a + b }

// Shorthand argument names
let closure = { $0 + $1 }

// Trailing closure (if last parameter)
numbers.map { $0 * 2 }
```

### Escaping vs Non-Escaping

```swift
// Non-escaping (default) - closure executes before function returns
func processImmediately(completion: () -> Void) {
    completion()  // Executes now
}

// Escaping - closure may execute after function returns
func processLater(completion: @escaping () -> Void) {
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        completion()  // Executes later
    }
}

// Common use case: storing closure
class NetworkManager {
    var completionHandlers: [() -> Void] = []
    
    func fetchData(completion: @escaping () -> Void) {
        completionHandlers.append(completion)  // Must be @escaping
    }
}
```

### Capture Lists

```swift
class MyViewController: UIViewController {
    var name = "John"
    
    // ❌ Retain cycle - closure captures self strongly
    func setupBadClosure() {
        someAsyncOperation { result in
            self.name = result  // Strong reference to self
        }
    }
    
    // ✅ Weak self - breaks retain cycle
    func setupWeakClosure() {
        someAsyncOperation { [weak self] result in
            guard let self = self else { return }
            self.name = result
        }
    }
    
    // Unowned self - use when self definitely won't be nil
    func setupUnownedClosure() {
        someAsyncOperation { [unowned self] result in
            self.name = result  // Crashes if self is deallocated
        }
    }
    
    // Capture specific properties
    func setupPropertyCapture() {
        let name = self.name
        someAsyncOperation { [name] result in
            print(name)  // Captured copy of name
        }
    }
    
    // Multiple captures
    func setupMultipleCaptures() {
        someAsyncOperation { [weak self, weak delegate = self.delegate, id = self.id] in
            guard let self = self else { return }
            // Use captured values
        }
    }
}
```

### Autoclosure

```swift
// Delays execution of expression
func logIfTrue(_ condition: @autoclosure () -> Bool, message: String) {
    if condition() {
        print(message)
    }
}

// Called like this (no closure syntax needed)
logIfTrue(2 > 1, message: "It's true!")  // Expression automatically becomes closure

// Common use: assert
func assert(_ condition: @autoclosure () -> Bool) {
    #if DEBUG
    if !condition() {
        fatalError("Assertion failed")
    }
    #endif
}

assert(x > 0)  // Only evaluated if DEBUG
```

### Common Closure Patterns

```swift
// Pattern 1: Completion handler
func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(.failure(error))
        } else if let data = data {
            completion(.success(data))
        }
    }.resume()
}

// Pattern 2: Configuration closure
class Button {
    var tapHandler: (() -> Void)?
    
    func onTap(_ handler: @escaping () -> Void) -> Self {
        self.tapHandler = handler
        return self
    }
}

let button = Button()
    .onTap { print("Tapped!") }

// Pattern 3: Lazy initialization
class ViewController: UIViewController {
    lazy var tableView: UITableView = {
        let table = UITableView()
        table.delegate = self
        table.dataSource = self
        return table
    }()
}
```

---

## Error Handling

### Throwing Functions

```swift
// Define error types
enum NetworkError: Error {
    case invalidURL
    case noData
    case decodingFailed
}

// Throwing function
func fetchUser(id: String) throws -> User {
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }
    
    let (data, _) = try await URLSession.shared.data(from: url)
    
    guard !data.isEmpty else {
        throw NetworkError.noData
    }
    
    do {
        let user = try JSONDecoder().decode(User.self, from: data)
        return user
    } catch {
        throw NetworkError.decodingFailed
    }
}
```

### Calling Throwing Functions

```swift
// do-catch
do {
    let user = try fetchUser(id: "123")
    print("Fetched: \(user.name)")
} catch NetworkError.invalidURL {
    print("Invalid URL")
} catch NetworkError.noData {
    print("No data received")
} catch {
    print("Unknown error: \(error)")
}

// try? - converts to optional (nil on error)
if let user = try? fetchUser(id: "123") {
    print("Fetched: \(user.name)")
}

// try! - force try (crashes on error)
let user = try! fetchUser(id: "123")  // Use only when 100% certain it won't throw
```

### Result Type

```swift
// Modern approach: Result<Success, Failure>
func fetchUser(id: String, completion: @escaping (Result<User, NetworkError>) -> Void) {
    // ... async work
    
    if let user = user {
        completion(.success(user))
    } else {
        completion(.failure(.noData))
    }
}

// Using Result
fetchUser(id: "123") { result in
    switch result {
    case .success(let user):
        print("Got user: \(user.name)")
    case .failure(let error):
        print("Error: \(error)")
    }
}

// Converting between Result and throwing
func fetchUser(id: String) async throws -> User {
    return try await withCheckedThrowingContinuation { continuation in
        fetchUser(id: id) { result in
            continuation.resume(with: result)
        }
    }
}
```

### Error Protocol

```swift
// Custom error with additional info
struct ValidationError: Error {
    let field: String
    let message: String
}

// Localized error
extension NetworkError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "The URL is invalid"
        case .noData:
            return "No data was received"
        case .decodingFailed:
            return "Failed to decode the response"
        }
    }
}
```

---

## Memory Management in Swift

### ARC Basics

```swift
// Automatic Reference Counting (like Objective-C ARC)
class Person {
    let name: String
    init(name: String) {
        self.name = name
        print("\(name) initialized")
    }
    
    deinit {
        print("\(name) deallocated")
    }
}

var person: Person? = Person(name: "John")  // Reference count = 1
person = nil  // Reference count = 0, deinit called
```

### Strong Reference Cycles

```swift
// ❌ Retain cycle
class Person {
    var apartment: Apartment?
}

class Apartment {
    var tenant: Person?  // Strong reference
}

var person: Person? = Person()
var apartment: Apartment? = Apartment()

person!.apartment = apartment  // person → apartment (strong)
apartment!.tenant = person     // apartment → person (strong)

// Cycle! Neither will deallocate
person = nil
apartment = nil
// Both still in memory!
```

### Weak References

```swift
class Person {
    var apartment: Apartment?
}

class Apartment {
    weak var tenant: Person?  // ✅ Weak reference
}

var person: Person? = Person()
var apartment: Apartment? = Apartment()

person!.apartment = apartment
apartment!.tenant = person

person = nil  // Person deallocated
print(apartment!.tenant)  // nil (weak reference automatically set to nil)
```

### Unowned References

```swift
// Use when reference will NEVER be nil during its lifetime
class Customer {
    let name: String
    var card: CreditCard?
    
    init(name: String) {
        self.name = name
    }
}

class CreditCard {
    let number: String
    unowned let customer: Customer  // Customer will always exist
    
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
}

var customer: Customer? = Customer(name: "John")
customer!.card = CreditCard(number: "1234", customer: customer!)

customer = nil  // Both deallocated correctly
```

### Weak vs Unowned Decision

```
Use WEAK when:
├─ Reference might become nil
├─ Reference is optional
└─ Example: delegate, parent reference

Use UNOWNED when:
├─ Reference will never be nil during lifetime
├─ Reference is non-optional
├─ Guaranteed lifetime relationship
└─ Example: child has unowned reference to parent
```

### Closure Capture List Patterns

```swift
// Pattern 1: Weak self with guard
someAsync { [weak self] in
    guard let self = self else { return }
    self.updateUI()
}

// Pattern 2: Weak self without guard (ok if operation is safe)
someAsync { [weak self] in
    self?.updateUI()  // Does nothing if self is nil
}

// Pattern 3: Capture specific strong references
class ViewController {
    let viewModel: ViewModel
    
    func setup() {
        someAsync { [viewModel] in
            // Captures viewModel strongly, not self
            viewModel.process()
        }
    }
}

// Pattern 4: Weak self, then strong for duration
someAsync { [weak self] in
    guard let self = self else { return }
    
    // self is strong for duration of closure
    self.performLongOperation()
    self.updateUI()
    self.cleanup()
}
```

---

## Property Observers & Wrappers

### Property Observers

```swift
class StepCounter {
    var totalSteps: Int = 0 {
        willSet {
            print("About to set totalSteps to \(newValue)")
        }
        didSet {
            print("Added \(totalSteps - oldValue) steps")
            
            if totalSteps > oldValue {
                print("Made progress!")
            }
        }
    }
}

let counter = StepCounter()
counter.totalSteps = 100
// "About to set totalSteps to 100"
// "Added 100 steps"
// "Made progress!"
```

### Computed Properties

```swift
struct Rectangle {
    var width: Double
    var height: Double
    
    // Computed property (no storage)
    var area: Double {
        get {
            return width * height
        }
        set {
            // newValue is implicit parameter
            width = sqrt(newValue)
            height = sqrt(newValue)
        }
    }
    
    // Read-only computed property
    var perimeter: Double {
        return 2 * (width + height)
    }
}
```

### Lazy Properties

```swift
class ImageLoader {
    // Only created when first accessed
    lazy var cache: NSCache<NSURL, UIImage> = {
        let cache = NSCache<NSURL, UIImage>()
        cache.countLimit = 100
        return cache
    }()
    
    // Lazy with value type
    lazy var expensiveArray: [Int] = {
        return (0..<10000).map { $0 * $0 }
    }()
}

// Not created yet
let loader = ImageLoader()

// NOW cache is created
loader.cache.setObject(image, forKey: url)
```

### Property Wrappers

```swift
// Define a property wrapper
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    private let range: ClosedRange<Value>
    
    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
    
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
}

// Use it
struct Audio {
    @Clamped(0...100) var volume: Int = 50
}

var audio = Audio()
audio.volume = 150  // Clamped to 100
audio.volume = -10  // Clamped to 0

// Built-in property wrappers in SwiftUI
@State private var count = 0
@Binding var isOn: Bool
@StateObject private var viewModel = ViewModel()
@Published var items: [String]
```

### Common Property Wrapper Patterns

```swift
// UserDefaults wrapper
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}

// Usage
struct Settings {
    @UserDefault(key: "username", defaultValue: "Guest")
    var username: String
    
    @UserDefault(key: "isDarkMode", defaultValue: false)
    var isDarkMode: Bool
}

var settings = Settings()
settings.username = "John"  // Automatically saves to UserDefaults
```

---

## Extensions

### Basic Extensions

```swift
// Add computed property
extension Int {
    var squared: Int {
        return self * self
    }
}

print(5.squared)  // 25

// Add methods
extension String {
    func trimmed() -> String {
        return self.trimmingCharacters(in: .whitespacesAndNewlines)
    }
    
    func isValidEmail() -> Bool {
        let emailRegex = "[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}"
        return NSPredicate(format: "SELF MATCHES %@", emailRegex).evaluate(with: self)
    }
}

let email = "  test@example.com  ".trimmed()
print(email.isValidEmail())  // true
```

### Protocol Conformance

```swift
// Add protocol conformance retroactively
extension Int: CustomStringConvertible {
    public var description: String {
        return "The number \(self)"
    }
}

print(42)  // "The number 42"

// Conditional conformance
extension Array: Equatable where Element: Equatable {
    // Array is Equatable only when its elements are
}
```

### Constrained Extensions

```swift
// Extend only specific types
extension Collection where Element == Int {
    func sum() -> Int {
        return reduce(0, +)
    }
}

[1, 2, 3].sum()  // 6
// ["a", "b"].sum()  // ❌ Error: String is not Int

// Extend protocol with constraints
extension Collection where Element: Comparable {
    func sorted() -> [Element] {
        return sorted(by: <)
    }
}
```

### Extending Your Own Types

```swift
struct User {
    let firstName: String
    let lastName: String
}

// Group related functionality
extension User {
    var fullName: String {
        return "\(firstName) \(lastName)"
    }
    
    var initials: String {
        let first = firstName.prefix(1)
        let last = lastName.prefix(1)
        return "\(first)\(last)"
    }
}

// Conformance in extension (organizational)
extension User: Codable {}
extension User: Equatable {}
extension User: CustomStringConvertible {
    var description: String {
        return fullName
    }
}
```

---

## Access Control

### Access Levels

```swift
// open - Most permissive (can subclass/override from other modules)
open class OpenClass {
    open func openMethod() {}
}

// public - Accessible from other modules (can't subclass from other modules)
public class PublicClass {
    public func publicMethod() {}
}

// internal - Default, accessible within same module
internal class InternalClass {  // 'internal' keyword optional
    func internalMethod() {}
}

// fileprivate - Accessible within same file
fileprivate class FilePrivateClass {
    fileprivate func filePrivateMethod() {}
}

// private - Most restrictive, accessible within same declaration
private class PrivateClass {
    private func privateMethod() {}
}
```

### Access Control Hierarchy

```
open > public > internal > fileprivate > private
```

### Practical Usage

```swift
// Public API, internal implementation
public struct User {
    public let id: String
    public let name: String
    private var internalData: [String: Any] = [:]
    
    public init(id: String, name: String) {
        self.id = id
        self.name = name
    }
    
    // Public method
    public func displayName() -> String {
        return processName()  // Can call private method
    }
    
    // Private implementation detail
    private func processName() -> String {
        return name.uppercased()
    }
}

// Subclassing control
open class BaseViewController {
    open func setupUI() {}  // Can override from anywhere
}

public class SpecialViewController {
    public func setupUI() {}  // Can't override from other modules
}

// File-level sharing
class MyViewController {
    fileprivate var sharedData: String = ""
}

extension MyViewController {
    fileprivate func processData() {
        // Can access sharedData (same file)
    }
}
```

---

## Enums & Pattern Matching

### Basic Enums

```swift
enum Direction {
    case north
    case south
    case east
    case west
}

// Shorthand when type is known
var direction: Direction = .north
direction = .east
```

### Associated Values

```swift
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}

// Different data for each case
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

let productBarcode = Barcode.upc(8, 85909, 51226, 3)
let qrBarcode = Barcode.qrCode("ABCDEFG")

// Extract values
switch productBarcode {
case .upc(let numberSystem, let manufacturer, let product, let check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check)")
case .qrCode(let code):
    print("QR code: \(code)")
}

// Shorthand
switch productBarcode {
case let .upc(numberSystem, manufacturer, product, check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check)")
case let .qrCode(code):
    print("QR code: \(code)")
}
```

### Raw Values

```swift
// String raw values
enum Planet: String {
    case mercury = "Mercury"
    case venus = "Venus"
    case earth = "Earth"
}

print(Planet.earth.rawValue)  // "Earth"

// Int raw values (auto-increment)
enum Priority: Int {
    case low = 1
    case medium  // 2
    case high    // 3
}

// Initialize from raw value (returns optional)
if let priority = Priority(rawValue: 2) {
    print(priority)  // medium
}
```

### Recursive Enums

```swift
indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}

// (5 + 4) * 2
let five = ArithmeticExpression.number(5)
let four = ArithmeticExpression.number(4)
let sum = ArithmeticExpression.addition(five, four)
let product = ArithmeticExpression.multiplication(sum, ArithmeticExpression.number(2))

func evaluate(_ expression: ArithmeticExpression) -> Int {
    switch expression {
    case let .number(value):
        return value
    case let .addition(left, right):
        return evaluate(left) + evaluate(right)
    case let .multiplication(left, right):
        return evaluate(left) * evaluate(right)
    }
}

print(evaluate(product))  // 18
```

### Pattern Matching

```swift
let point = (x: 1, y: 2)

// Switch with tuples
switch point {
case (0, 0):
    print("Origin")
case (_, 0):
    print("On x-axis")
case (0, _):
    print("On y-axis")
case (-2...2, -2...2):
    print("Inside the box")
default:
    print("Outside the box")
}

// Value binding
switch point {
case (let x, 0):
    print("On x-axis at \(x)")
case (0, let y):
    print("On y-axis at \(y)")
case let (x, y):
    print("At (\(x), \(y))")
}

// Where clause
switch point {
case let (x, y) where x == y:
    print("On the diagonal")
case let (x, y) where x == -y:
    print("On the negative diagonal")
case let (x, y):
    print("At (\(x), \(y))")
}
```

### If case / guard case

```swift
enum Result {
    case success(String)
    case failure(Error)
}

let result = Result.success("Data loaded")

// Extract associated value with if case
if case let .success(message) = result {
    print(message)  // "Data loaded"
}

// Guard with pattern matching
func handle(result: Result) {
    guard case let .success(message) = result else {
        print("Failed")
        return
    }
    print("Success: \(message)")
}
```

---

## Modern Concurrency (async/await)

### Basic async/await

```swift
// Async function
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}

// Calling async function
Task {
    do {
        let user = try await fetchUser(id: "123")
        print("Fetched: \(user.name)")
    } catch {
        print("Error: \(error)")
    }
}
```

### Tasks

```swift
// Create a task
let task = Task {
    let user = try await fetchUser(id: "123")
    return user
}

// Get result
let user = try await task.value

// Cancel task
task.cancel()

// Check if cancelled
Task {
    guard !Task.isCancelled else { return }
    let data = try await fetchData()
    processData(data)
}

// Detached task (doesn't inherit context)
Task.detached {
    // Runs independently
}
```

### Task Groups

```swift
// Parallel execution
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

// Usage
let users = try await fetchMultipleUsers(ids: ["1", "2", "3"])
```

### Actors

```swift
// Thread-safe class
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

// Usage (all methods are async)
let account = BankAccount()

Task {
    await account.deposit(amount: 100)
    let balance = await account.getBalance()
    print(balance)  // 100
}
```

### @MainActor

```swift
// Ensure code runs on main thread
@MainActor
class ViewModel: ObservableObject {
    @Published var users: [User] = []
    
    func loadUsers() async {
        let users = try? await fetchUsers()
        self.users = users ?? []  // Safely updates on main thread
    }
}

// Mark specific functions
class MyClass {
    @MainActor
    func updateUI() {
        // Always runs on main thread
    }
}

// Use in closures
Task { @MainActor in
    // Runs on main thread
    label.text = "Updated"
}
```

### AsyncSequence

```swift
// Async iteration
for try await line in URL(string: "https://example.com/file.txt")!.lines {
    print(line)
}

// Custom AsyncSequence
struct Counter: AsyncSequence {
    typealias Element = Int
    let limit: Int
    
    struct AsyncIterator: AsyncIteratorProtocol {
        let limit: Int
        var current = 0
        
        mutating func next() async -> Int? {
            guard current < limit else { return nil }
            current += 1
            try? await Task.sleep(nanoseconds: 1_000_000_000)  // 1 second
            return current
        }
    }
    
    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(limit: limit)
    }
}

// Usage
for await number in Counter(limit: 5) {
    print(number)  // 1, 2, 3, 4, 5 (one per second)
}
```

### AsyncStream & AsyncSequence

AsyncStream provides a way to create asynchronous sequences of values, perfect for converting callback-based APIs to async/await.

#### Basic AsyncStream Creation

```swift
// Simple AsyncStream
func countdown(from: Int) -> AsyncStream<Int> {
    AsyncStream { continuation in
        Task {
            for i in (0...from).reversed() {
                continuation.yield(i)
                try? await Task.sleep(nanoseconds: 1_000_000_000)
            }
            continuation.finish()
        }
    }
}

// Usage
for await number in countdown(from: 5) {
    print("Countdown: \(number)")
}

// Using makeStream for more control
func generateNumbers() -> AsyncStream<Int> {
    let (stream, continuation) = AsyncStream<Int>.makeStream()

    Task {
        for i in 1...10 {
            continuation.yield(i)
            try? await Task.sleep(nanoseconds: 500_000_000)
        }
        continuation.finish()
    }

    return stream
}
```

#### Converting Callbacks to AsyncStream

```swift
import CoreLocation

// Convert delegate-based API to AsyncStream
class LocationStreamer: NSObject {
    func locationUpdates() -> AsyncStream<CLLocation> {
        AsyncStream { continuation in
            let manager = CLLocationManager()
            let delegate = StreamDelegate(continuation: continuation)

            manager.delegate = delegate
            manager.requestWhenInUseAuthorization()
            manager.startUpdatingLocation()

            continuation.onTermination = { _ in
                manager.stopUpdatingLocation()
            }
        }
    }
}

private class StreamDelegate: NSObject, CLLocationManagerDelegate {
    let continuation: AsyncStream<CLLocation>.Continuation

    init(continuation: AsyncStream<CLLocation>.Continuation) {
        self.continuation = continuation
    }

    func locationManager(_ manager: CLLocationManager,
                        didUpdateLocations locations: [CLLocation]) {
        for location in locations {
            continuation.yield(location)
        }
    }
}

// Usage
let streamer = LocationStreamer()
for await location in streamer.locationUpdates() {
    print("New location: \(location.coordinate)")
}
```

#### AsyncThrowingStream for Error Handling

```swift
// Stream that can throw errors
func fetchDataStream(urls: [URL]) -> AsyncThrowingStream<Data, Error> {
    AsyncThrowingStream { continuation in
        Task {
            for url in urls {
                do {
                    let (data, _) = try await URLSession.shared.data(from: url)
                    continuation.yield(data)
                } catch {
                    continuation.finish(throwing: error)
                    return
                }
            }
            continuation.finish()
        }
    }
}

// Usage with error handling
do {
    for try await data in fetchDataStream(urls: myURLs) {
        print("Received \(data.count) bytes")
    }
} catch {
    print("Stream error: \(error)")
}
```

#### Buffering and Backpressure

```swift
// AsyncStream with buffering policy
func bufferedStream() -> AsyncStream<Int> {
    AsyncStream(bufferingPolicy: .bufferingNewest(5)) { continuation in
        // If consumer is slow, keeps only the 5 newest values
        Task {
            for i in 1...100 {
                continuation.yield(i)
                // Simulating fast producer
                try? await Task.sleep(nanoseconds: 100_000_000)
            }
            continuation.finish()
        }
    }
}

// Different buffering strategies
enum BufferingExample {
    static func unbuffered() -> AsyncStream<Int> {
        AsyncStream(bufferingPolicy: .unbounded) { continuation in
            // Keeps all values (may use lots of memory)
        }
    }

    static func dropOldest() -> AsyncStream<Int> {
        AsyncStream(bufferingPolicy: .bufferingOldest(10)) { continuation in
            // Keeps oldest 10 values, drops new ones if buffer full
        }
    }
}
```

#### Custom AsyncSequence Implementation

```swift
// Custom async sequence for paginated API
struct PaginatedDataSequence: AsyncSequence {
    typealias Element = [Item]

    let baseURL: URL
    let pageSize: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        let baseURL: URL
        let pageSize: Int
        var currentPage = 0
        var hasMore = true

        mutating func next() async throws -> [Item]? {
            guard hasMore else { return nil }

            var components = URLComponents(url: baseURL, resolvingAgainstBaseURL: false)!
            components.queryItems = [
                URLQueryItem(name: "page", value: "\(currentPage)"),
                URLQueryItem(name: "size", value: "\(pageSize)")
            ]

            let (data, _) = try await URLSession.shared.data(from: components.url!)
            let response = try JSONDecoder().decode(PageResponse.self, from: data)

            hasMore = response.hasNext
            currentPage += 1

            return response.items
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(baseURL: baseURL, pageSize: pageSize)
    }
}

// Usage
let paginated = PaginatedDataSequence(baseURL: apiURL, pageSize: 20)
for try await items in paginated {
    print("Processing \(items.count) items")
    // Automatically fetches next page when needed
}
```

#### Combining Multiple AsyncStreams

```swift
// Merge multiple streams
func merge<T>(_ streams: AsyncStream<T>...) -> AsyncStream<T> {
    AsyncStream { continuation in
        Task {
            await withTaskGroup(of: Void.self) { group in
                for stream in streams {
                    group.addTask {
                        for await value in stream {
                            continuation.yield(value)
                        }
                    }
                }
            }
            continuation.finish()
        }
    }
}

// Zip streams together
func zip<T, U>(_ first: AsyncStream<T>, _ second: AsyncStream<U>) -> AsyncStream<(T, U)> {
    AsyncStream { continuation in
        Task {
            var firstIterator = first.makeAsyncIterator()
            var secondIterator = second.makeAsyncIterator()

            while let value1 = await firstIterator.next(),
                  let value2 = await secondIterator.next() {
                continuation.yield((value1, value2))
            }
            continuation.finish()
        }
    }
}
```

#### Real-Time Updates with AsyncStream

```swift
// NotificationCenter as AsyncStream
extension NotificationCenter {
    func notifications(named name: Notification.Name,
                       object: Any? = nil) -> AsyncStream<Notification> {
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

// Usage
for await notification in NotificationCenter.default.notifications(
    named: UIApplication.didBecomeActiveNotification
) {
    print("App became active")
}

// Timer as AsyncStream
func timer(interval: TimeInterval) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            continuation.yield(Date())
        }

        continuation.onTermination = { _ in
            timer.invalidate()
        }
    }
}

// Usage
for await timestamp in timer(interval: 1.0) {
    print("Tick at \(timestamp)")
}
```

#### Testing AsyncStreams

```swift
import XCTest

class AsyncStreamTests: XCTestCase {
    func testAsyncStreamValues() async {
        // Create test stream
        let stream = AsyncStream<Int> { continuation in
            continuation.yield(1)
            continuation.yield(2)
            continuation.yield(3)
            continuation.finish()
        }

        // Collect values
        var collected: [Int] = []
        for await value in stream {
            collected.append(value)
        }

        // Assert
        XCTAssertEqual(collected, [1, 2, 3])
    }

    func testAsyncStreamWithError() async throws {
        // Create throwing stream
        let stream = AsyncThrowingStream<Int, Error> { continuation in
            continuation.yield(1)
            continuation.finish(throwing: TestError.failed)
        }

        // Test error handling
        var values: [Int] = []
        do {
            for try await value in stream {
                values.append(value)
            }
            XCTFail("Should have thrown")
        } catch {
            XCTAssertEqual(values, [1])
            XCTAssertTrue(error is TestError)
        }
    }

    func testAsyncStreamTimeout() async {
        let expectation = expectation(description: "Stream completes")

        let stream = AsyncStream<Int> { continuation in
            Task {
                try? await Task.sleep(nanoseconds: 100_000_000)
                continuation.yield(42)
                continuation.finish()
                expectation.fulfill()
            }
        }

        var result: Int?
        for await value in stream {
            result = value
        }

        await fulfillment(of: [expectation], timeout: 1)
        XCTAssertEqual(result, 42)
    }
}
```

### Sendable Protocol & Data Race Safety

The `Sendable` protocol (Swift 5.5+) marks types that can be safely shared across concurrent contexts without causing data races. **Swift 6.0 enforces strict concurrency checking by default.**

#### Understanding Sendable

```swift
// Sendable types are safe to share across threads
protocol Sendable {}

// Most value types are implicitly Sendable
struct User: Sendable {  // Automatic conformance
    let name: String
    let age: Int
}

// Reference types need explicit conformance
final class Counter: @unchecked Sendable {
    // @unchecked means you're responsible for thread safety
    private let lock = NSLock()
    private var value = 0

    func increment() {
        lock.lock()
        defer { lock.unlock() }
        value += 1
    }
}
```

#### When Types are Sendable

```swift
// ✅ Automatically Sendable:
// - Basic value types (Int, String, Bool, etc.)
// - Structs with all Sendable properties
// - Enums with Sendable associated values
// - Immutable classes with Sendable properties
// - Actors (always Sendable)

struct Location: Sendable {  // ✅ Auto-conforms
    let latitude: Double
    let longitude: Double
}

enum Status: Sendable {  // ✅ Auto-conforms
    case active
    case inactive(reason: String)
}

final class ImmutableUser: Sendable {  // ✅ Can conform
    let name: String
    let id: UUID

    init(name: String, id: UUID) {
        self.name = name
        self.id = id
    }
}

// ❌ Not automatically Sendable:
// - Classes with mutable state
// - Structs/enums containing non-Sendable types
// - Closures capturing non-Sendable values

class MutableUser {  // ❌ Cannot be Sendable
    var name: String = ""
    var loginCount: Int = 0
}
```

#### Sendable Closures

```swift
// @Sendable marks closures that can be shared safely
func performAsync(work: @Sendable @escaping () -> Void) {
    Task.detached {
        work()  // Safe to call from any thread
    }
}

// Using @Sendable closures
func example() {
    let immutableValue = 42  // Captured values must be Sendable

    performAsync { @Sendable in
        print(immutableValue)  // ✅ Int is Sendable
    }

    var mutableValue = 0
    performAsync { @Sendable in
        // ❌ Error: Capture of 'mutableValue' with non-sendable type
        // print(mutableValue)
    }
}
```

#### Sendable with async/await

```swift
// Functions taking Sendable parameters
func process(user: sending User) async {
    // 'sending' keyword ensures the value is sent, not shared
    await saveToDatabase(user)
}

// Actor isolation and Sendable
actor DataStore {
    private var cache: [String: any Sendable] = [:]

    func store<T: Sendable>(_ value: T, key: String) {
        cache[key] = value
    }

    func retrieve<T: Sendable>(_ type: T.Type, key: String) -> T? {
        cache[key] as? T
    }
}
```

#### Conditional Sendable Conformance

```swift
// Generic types can conditionally conform
struct Box<T> {
    let value: T
}

// Box is Sendable only if T is Sendable
extension Box: Sendable where T: Sendable {}

// Usage
let intBox = Box(value: 42)  // ✅ Sendable
let userBox = Box(value: MutableUser())  // ❌ Not Sendable
```

#### @unchecked Sendable

```swift
// Use when you guarantee thread safety manually
final class ThreadSafeCache: @unchecked Sendable {
    private var storage: [String: Any] = [:]
    private let queue = DispatchQueue(label: "cache.queue",
                                     attributes: .concurrent)

    func set(_ value: Any, for key: String) {
        queue.async(flags: .barrier) {
            self.storage[key] = value
        }
    }

    func get(_ key: String) -> Any? {
        queue.sync {
            storage[key]
        }
    }
}
```

#### Global Variables and Sendable

```swift
// Global variables must be Sendable in strict concurrency
let globalConfig = Configuration()  // ✅ If Configuration is Sendable

// Use @unchecked for legacy code
@unchecked Sendable
var legacyGlobalState: LegacyClass?  // Be careful!
```

#### Best Practices

```swift
// 1. Prefer value types for concurrent code
struct Message: Sendable {
    let id: UUID
    let text: String
    let timestamp: Date
}

// 2. Use actors for mutable shared state
actor MessageQueue {
    private var messages: [Message] = []

    func enqueue(_ message: Message) {
        messages.append(message)
    }

    func dequeue() -> Message? {
        messages.isEmpty ? nil : messages.removeFirst()
    }
}

// 3. Mark completion handlers as @Sendable
func fetchData(completion: @Sendable @escaping (Result<Data, Error>) -> Void) {
    Task.detached {
        do {
            let data = try await loadData()
            completion(.success(data))
        } catch {
            completion(.failure(error))
        }
    }
}

// 4. Use sending parameters for ownership transfer
func transfer(data: sending Data) async {
    // Takes ownership of data, original can't be used
    await process(data)
}
```

#### Common Errors and Solutions

```swift
// Error: Type does not conform to Sendable
struct BadExample {
    let closure: () -> Void  // ❌ Closures aren't automatically Sendable
}

// Solution: Mark closure as @Sendable
struct GoodExample: Sendable {
    let closure: @Sendable () -> Void  // ✅
}

// Error: Capture of non-Sendable type
class ViewModel {
    var count = 0

    func startTimer() {
        Task.detached {
            // ❌ Error: Capture of 'self' with non-sendable type
            self.count += 1
        }
    }
}

// Solution: Use actor or ensure Sendable
actor ViewModelActor {
    var count = 0

    func startTimer() {
        Task.detached {
            await self.incrementCount()  // ✅ Actor is Sendable
        }
    }

    func incrementCount() {
        count += 1
    }
}
```

### Swift 6.0 New Features (2024-2025)

#### Typed Throws
```swift
// Define specific error types
enum ValidationError: Error {
    case tooShort
    case tooLong
    case invalidFormat
}

// Function can only throw ValidationError
func validate(_ input: String) throws(ValidationError) -> Bool {
    guard input.count >= 3 else {
        throw ValidationError.tooShort
    }
    guard input.count <= 20 else {
        throw ValidationError.tooLong
    }
    return true
}

// Compiler knows the error type
do {
    try validate("ab")
} catch {
    // error is ValidationError, not Error
    switch error {
    case .tooShort:
        print("Input too short")
    case .tooLong:
        print("Input too long")
    case .invalidFormat:
        print("Invalid format")
    }
}
```

#### Complete Concurrency Checking
```swift
// Swift 6.0 enables strict concurrency by default
// Compile with: -strict-concurrency=complete

class LegacyClass {  // Warning: not Sendable
    var counter = 0
}

// Swift 6.0 will warn about data races
func problematicCode() {
    let legacy = LegacyClass()

    Task {
        legacy.counter += 1  // ⚠️ Data race warning
    }

    Task {
        print(legacy.counter)  // ⚠️ Data race warning
    }
}

// Fix with actor or @MainActor
@MainActor
class SafeClass {
    var counter = 0
}
```

#### Noncopyable Types
```swift
// Resource that shouldn't be copied
@noncopyable
struct UniqueResource {
    private let handle: Int

    init() {
        handle = createResource()
    }

    deinit {
        destroyResource(handle)
    }

    // Use 'consuming' to transfer ownership
    consuming func take() -> UniqueResource {
        return self
    }

    // Use 'borrowing' for read-only access
    borrowing func read() -> Int {
        return handle
    }
}

// Usage
func useResource() {
    let resource = UniqueResource()
    let value = resource.read()  // Borrowing
    let transferred = resource.take()  // Consuming - resource no longer usable
    // print(resource.handle)  // ❌ Error: resource was consumed
}
```

#### Parameter Packs (Variadic Generics)
```swift
// Work with multiple types at once
func zip<each T, each U>(_ first: repeat each T, with second: repeat each U) -> (repeat (each T, each U)) {
    return (repeat (each first, each second))
}

// Usage
let result = zip(1, "hello", true, with: 2.0, "world", false)
// result is ((Int, Double), (String, String), (Bool, Bool))
```

#### Improved Existential Types
```swift
// Swift 6.0 allows 'any' keyword for clarity
protocol Drawable {
    func draw()
}

// Old way (still works)
let shapes: [Drawable] = []

// New way (clearer)
let shapes: [any Drawable] = []

// Use 'some' for opaque types
func makeShape() -> some Drawable {
    return Circle()
}
```

---

## Collection Operators

### Map, Filter, Reduce

```swift
let numbers = [1, 2, 3, 4, 5]

// map - transform each element
let doubled = numbers.map { $0 * 2 }  // [2, 4, 6, 8, 10]

// filter - keep elements that match condition
let evens = numbers.filter { $0 % 2 == 0 }  // [2, 4]

// reduce - combine elements into single value
let sum = numbers.reduce(0, +)  // 15
let product = numbers.reduce(1, *)  // 120

// reduce with closure
let concatenated = numbers.reduce("") { result, number in
    return result + String(number)
}  // "12345"
```

### CompactMap

```swift
// Remove nils and unwrap
let strings = ["1", "2", "three", "4"]
let numbers = strings.compactMap { Int($0) }  // [1, 2, 4]

// vs map (keeps nils)
let maybeNumbers = strings.map { Int($0) }  // [Optional(1), Optional(2), nil, Optional(4)]
```

### FlatMap

```swift
// Flatten nested arrays
let nested = [[1, 2], [3, 4], [5, 6]]
let flattened = nested.flatMap { $0 }  // [1, 2, 3, 4, 5, 6]

// Combine map + flatten
let words = ["Hello", "World"]
let letters = words.flatMap { $0 }  // ["H", "e", "l", "l", "o", "W", "o", "r", "l", "d"]
```

### Chaining

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

let result = numbers
    .filter { $0 % 2 == 0 }     // [2, 4, 6, 8, 10]
    .map { $0 * $0 }             // [4, 16, 36, 64, 100]
    .reduce(0, +)                // 220
```

### Other Useful Methods

```swift
let numbers = [1, 2, 3, 4, 5]

// contains
numbers.contains(3)  // true
numbers.contains { $0 > 3 }  // true

// first(where:)
numbers.first { $0 > 3 }  // Optional(4)

// allSatisfy
numbers.allSatisfy { $0 > 0 }  // true

// sorted
numbers.sorted()  // [1, 2, 3, 4, 5]
numbers.sorted(by: >)  // [5, 4, 3, 2, 1]

// prefix/suffix
Array(numbers.prefix(3))  // [1, 2, 3]
Array(numbers.suffix(2))  // [4, 5]

// dropFirst/dropLast
Array(numbers.dropFirst(2))  // [3, 4, 5]
Array(numbers.dropLast(2))  // [1, 2, 3]
```

---

## Swift vs Objective-C Key Differences

### Nullability

```objc
// Objective-C
NSString *name;  // Could be nil
NSString *_Nullable optionalName;  // Explicitly optional
NSString *_Nonnull requiredName;    // Can't be nil

// Swift
var name: String  // Never nil
var optionalName: String?  // Can be nil
```

### Method Calls

```objc
// Objective-C
[object methodWithParameter:value];
[object methodWithFirst:value1 second:value2];

// Swift
object.method(parameter: value)
object.method(first: value1, second: value2)
```

### Properties

```objc
// Objective-C
@property (nonatomic, strong) NSString *name;
@property (nonatomic, weak) id<MyDelegate> delegate;
@property (nonatomic, copy) NSString *title;

// Swift
var name: String  // Strong by default
weak var delegate: MyDelegate?
var title: String  // Value type, no need for copy
```

### Blocks vs Closures

```objc
// Objective-C
typedef void (^CompletionHandler)(NSString *result, NSError *error);

- (void)fetchDataWithCompletion:(CompletionHandler)completion {
    completion(@"Data", nil);
}

// Swift
typealias CompletionHandler = (String?, Error?) -> Void

func fetchData(completion: @escaping CompletionHandler) {
    completion("Data", nil)
}
```

### Collections

```objc
// Objective-C (mutable vs immutable)
NSArray *immutable = @[@"a", @"b"];
NSMutableArray *mutable = [NSMutableArray arrayWithArray:immutable];

// Swift (let vs var)
let immutable = ["a", "b"]  // Immutable array
var mutable = ["a", "b"]    // Mutable array
```

### Protocols

```objc
// Objective-C
@protocol MyProtocol <NSObject>
@required
- (void)requiredMethod;
@optional
- (void)optionalMethod;
@end

// Swift
protocol MyProtocol {
    func requiredMethod()
}

// Optional methods via extension
extension MyProtocol {
    func optionalMethod() {
        // Default implementation
    }
}
```

---

## Interview Questions

### Q: "What's the difference between class and struct?"
- **Struct**: Value type, copied on assignment, no inheritance, no deinit
- **Class**: Reference type, shared on assignment, inheritance, has deinit

### Q: "When would you use map vs flatMap vs compactMap?"
- **map**: Transform each element (keeps structure)
- **flatMap**: Transform and flatten nested collections
- **compactMap**: Transform and remove nils

### Q: "Explain weak vs unowned"
- **weak**: Optional, automatically becomes nil when deallocated
- **unowned**: Non-optional, crashes if accessed after deallocation
- Use weak when reference might become nil, unowned when it never will

### Q: "What's the difference between @escaping and non-escaping closures?"
- **Non-escaping** (default): Executes before function returns
- **@escaping**: May execute after function returns (needs annotation)

### Q: "How do you prevent retain cycles?"
- Use `[weak self]` in closures
- Use `weak` for delegates
- Use `unowned` for guaranteed lifetime relationships
- Avoid strong reference cycles in closures

---

## Quick Reference Card

```swift
// Optionals
var optional: String?
if let unwrapped = optional { }
guard let unwrapped = optional else { return }
let value = optional ?? "default"

// Value vs Reference
struct ValueType { }  // Copied
class ReferenceType { }  // Shared

// Closures
{ (params) -> ReturnType in
    // body
}
{ $0 + $1 }  // Shorthand

// Memory
weak var delegate: Delegate?
unowned let parent: Parent
[weak self] in { }

// Collections
array.map { }
array.filter { }
array.reduce(0, +)
array.compactMap { }

// Async
async throws -> Value
try await function()
Task { }
@MainActor

// Error Handling
do { try } catch { }
try? optional()
try! force()
```

This covers the essential Swift features you'll need. Focus on optionals, value/reference types, and closures - these are where most Objective-C developers trip up initially.
