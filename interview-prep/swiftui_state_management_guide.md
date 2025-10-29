---
layout: page
title: "SwiftUI State Management: Complete Guide"
subtitle: "Understanding @State, @Binding, @StateObject, @ObservedObject, @EnvironmentObject, and @Observable"
---

# SwiftUI State Management: Complete Guide
## Understanding @State, @Binding, @StateObject, @ObservedObject, @EnvironmentObject, and @Observable

---

## Overview Table

### iOS 13-16 (ObservableObject)

| Property Wrapper | Ownership | Lifecycle | Use Case | Value/Reference |
|-----------------|-----------|-----------|----------|-----------------|
| @State | View owns data | Tied to view | Simple value types in single view | Value type |
| @Binding | View borrows data | Parent controls | Two-way connection to parent's state | Reference to value |
| @StateObject | View owns object | View creates & owns | Reference type owned by view (ObservableObject) | Reference type |
| @ObservedObject | View observes object | External ownership | Reference type owned elsewhere (ObservableObject) | Reference type |
| @EnvironmentObject | View observes object | Injected from ancestor | Shared data across view hierarchy (ObservableObject) | Reference type |

### iOS 17+ (@Observable - Modern Way)

| Property Wrapper | Ownership | Lifecycle | Use Case | Value/Reference |
|-----------------|-----------|-----------|----------|-----------------|
| @State | View owns data | Tied to view | Simple value types in single view | Value type |
| @Binding | View borrows data | Parent controls | Two-way connection to parent's state | Reference to value |
| `var` (no wrapper!) | View owns/observes object | Automatic | Reference type with @Observable macro | Reference type |
| @Bindable | Provides $ syntax | Same as parent | Two-way binding to @Observable properties | Reference type |
| @Environment | Observes from environment | Injected from ancestor | Shared @Observable across view hierarchy | Reference type |

---

## Table of Contents
1. [@State](#state)
2. [@Binding](#binding)
3. [@StateObject vs @ObservedObject](#stateobject-vs-observedobject)
4. [@EnvironmentObject](#environmentobject)
5. [ObservableObject Protocol](#observableobject-protocol)
6. [@Observable - The Modern Way (iOS 17+)](#observable---the-modern-way-ios-17)
7. [Decision Flow Chart](#decision-flow-chart)
8. [Common Patterns & Best Practices](#common-patterns--best-practices)
9. [Common Mistakes & How to Fix](#common-mistakes--how-to-fix)
10. [Interview Questions](#interview-questions-you-might-get)

---

## @State

### What It Is
- For **simple value types** (Int, String, Bool, structs)
- The view **owns** this data
- **Private** to the view
- SwiftUI manages the storage and lifecycle

### When to Use
- Simple, view-local data that doesn't need to be shared
- Temporary UI state (toggles, text field values, selection states)
- When the data is a **value type** (struct, enum, primitives)

### Example
```swift
struct CounterView: View {
    @State private var count = 0  // View owns this
    
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1  // Modifying causes view to redraw
            }
        }
    }
}
```

### Key Points
- Always mark as `private` (it's view-local)
- SwiftUI recreates the view struct on state changes, but preserves @State values
- Stored **outside** the view struct in SwiftUI's internal storage
- Perfect for simple UI state that stays within one view

### What Happens Under the Hood
```swift
// Conceptually, SwiftUI does something like:
// 1. View struct is created
// 2. @State property is stored in separate SwiftUI-managed storage
// 3. View gets a reference to that storage
// 4. When state changes, view is marked for redraw
```

---

## @Binding

### What It Is
- A **two-way connection** to someone else's @State or other property
- The view **does not own** the data
- Creates a reference to the parent's source of truth

### When to Use
- Child view needs to read **and write** parent's data
- Creating reusable components that modify external state
- When you need two-way data flow

### Example
```swift
// Parent owns the data
struct ParentView: View {
    @State private var isOn = false
    
    var body: some View {
        ToggleChildView(isOn: $isOn)  // Pass binding with $
    }
}

// Child receives and can modify the data
struct ToggleChildView: View {
    @Binding var isOn: Bool  // No $ here in declaration
    
    var body: some View {
        Toggle("Power", isOn: $isOn)  // Use $ when passing to another view
    }
}
```

### Key Points
- Use `$` prefix to pass a binding: `$propertyName`
- Child can read and write, but parent maintains ownership
- Changes propagate back to the source of truth
- Enables component reusability

### Common Pattern: Custom Components
```swift
struct CustomTextField: View {
    @Binding var text: String
    let placeholder: String
    
    var body: some View {
        TextField(placeholder, text: $text)
            .textFieldStyle(RoundedBorderTextFieldStyle())
            .padding()
    }
}

// Usage
struct FormView: View {
    @State private var username = ""
    
    var body: some View {
        CustomTextField(text: $username, placeholder: "Username")
    }
}
```

---

## @StateObject vs @ObservedObject

These two are the most confusing. The key difference is **ownership**.

### @StateObject

#### What It Is
- For **reference types** (classes conforming to `ObservableObject`)
- The view **owns and creates** the object
- Object persists for the view's entire lifetime
- SwiftUI ensures it's only initialized **once**

#### When to Use
- When THIS view should create and own the object
- The view is the "source of truth" for this object
- You want the object to survive view updates

#### Example
```swift
class ViewModel: ObservableObject {
    @Published var items: [String] = []
    
    func loadItems() {
        // Load data...
        items = ["Item 1", "Item 2", "Item 3"]
    }
}

struct ContentView: View {
    @StateObject private var viewModel = ViewModel()  // Created once
    
    var body: some View {
        List(viewModel.items, id: \.self) { item in
            Text(item)
        }
        .onAppear {
            viewModel.loadItems()
        }
    }
}
```

#### Critical Behavior
```swift
struct ParentView: View {
    @State private var counter = 0
    
    var body: some View {
        VStack {
            ChildView(count: counter)
            Button("Rerender Child") { counter += 1 }
        }
    }
}

struct ChildView: View {
    let count: Int
    @StateObject private var viewModel = ViewModel()  // Only created ONCE
    
    var body: some View {
        Text("Count: \(count), VM items: \(viewModel.items.count)")
    }
}
// Even though ChildView re-renders when count changes,
// viewModel is NOT recreated. It persists.
```

### @ObservedObject

#### What It Is
- For **reference types** (classes conforming to `ObservableObject`)
- The view **does not own** the object
- Object is created and owned **elsewhere**
- Passed in from a parent

#### When to Use
- When the object is created by a parent or dependency injection
- Multiple views need to observe the same object
- You're passing an existing object down the view hierarchy

#### Example
```swift
class ViewModel: ObservableObject {
    @Published var items: [String] = []
}

struct ParentView: View {
    @StateObject private var viewModel = ViewModel()  // Parent creates
    
    var body: some View {
        ChildView(viewModel: viewModel)  // Pass to child
    }
}

struct ChildView: View {
    @ObservedObject var viewModel: ViewModel  // Child observes
    
    var body: some View {
        List(viewModel.items, id: \.self) { item in
            Text(item)
        }
    }
}
```

#### Critical Behavior - The Gotcha!
```swift
struct ParentView: View {
    @State private var counter = 0
    
    var body: some View {
        VStack {
            ChildView(count: counter)
            Button("Rerender Child") { counter += 1 }
        }
    }
}

struct ChildView: View {
    let count: Int
    @ObservedObject var viewModel = ViewModel()  // ❌ WRONG! Created on EVERY render
    
    var body: some View {
        Text("Count: \(count), VM items: \(viewModel.items.count)")
    }
}
// Every time count changes, ChildView re-renders and creates a NEW ViewModel!
// You'll lose all state. Use @StateObject instead.
```

### The Golden Rule

```
If the view creates the object → @StateObject
If the object is passed in → @ObservedObject
```

---

## @EnvironmentObject

### What It Is
- For **reference types** shared across many views
- Injected into the environment by an ancestor view
- Available to all descendant views without explicit passing
- Like dependency injection at the view hierarchy level

### When to Use
- Data needs to be accessible by many views at different levels
- Avoiding "prop drilling" (passing data through many layers)
- App-wide state (user session, settings, theme)

### Example
```swift
class UserSettings: ObservableObject {
    @Published var username = "Guest"
    @Published var isDarkMode = false
}

@main
struct MyApp: App {
    @StateObject private var settings = UserSettings()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(settings)  // Inject into environment
        }
    }
}

// Deep in the hierarchy - no need to pass through intermediate views
struct DeepNestedView: View {
    @EnvironmentObject var settings: UserSettings  // Access directly
    
    var body: some View {
        Text("Hello, \(settings.username)")
    }
}

// Another view can also access it
struct ProfileView: View {
    @EnvironmentObject var settings: UserSettings
    
    var body: some View {
        Toggle("Dark Mode", isOn: $settings.isDarkMode)
    }
}
```

### Key Points
- Must be injected with `.environmentObject()` by an ancestor
- App will crash if you use @EnvironmentObject but nobody provided it
- Great for avoiding prop drilling
- Use for truly shared, app-wide state

### Preview Support
```swift
struct ProfileView_Previews: PreviewProvider {
    static var previews: some View {
        ProfileView()
            .environmentObject(UserSettings())  // Provide for preview
    }
}
```

---

## ObservableObject Protocol

All reference types used with @StateObject, @ObservedObject, and @EnvironmentObject must conform to `ObservableObject`.

### Basic Implementation
```swift
class MyViewModel: ObservableObject {
    @Published var name = ""      // Auto-notifies on change
    @Published var count = 0
    
    var notObserved = ""           // Changes don't trigger updates
    
    func doSomething() {
        name = "Updated"            // View automatically updates
    }
}
```

### @Published
- Marks properties that should trigger view updates
- Automatically calls `objectWillChange.send()` before the value changes
- Only works with class properties (not struct)

### Manual Change Notification
```swift
class MyViewModel: ObservableObject {
    var items: [String] = [] {
        didSet {
            objectWillChange.send()  // Manual notification
        }
    }
}
```

---

## @Observable - The Modern Way (iOS 17+)

### What Changed in iOS 17

Apple introduced the **Observation framework** in iOS 17, which fundamentally changes how state management works in SwiftUI. The new `@Observable` macro **replaces** `ObservableObject` and dramatically simplifies code.

### Old Way vs New Way

```swift
// ❌ OLD WAY (iOS 13-16) - ObservableObject
class UserViewModel: ObservableObject {
    @Published var name = ""
    @Published var email = ""
    @Published var isLoading = false
    @Published var users: [User] = []
}

struct ContentView: View {
    @StateObject private var viewModel = UserViewModel()  // Need @StateObject

    var body: some View {
        Text(viewModel.name)
    }
}

// ✅ NEW WAY (iOS 17+) - @Observable
@Observable
class UserViewModel {
    var name = ""            // No @Published needed!
    var email = ""
    var isLoading = false
    var users: [User] = []
}

struct ContentView: View {
    var viewModel = UserViewModel()  // No property wrapper needed!

    var body: some View {
        Text(viewModel.name)  // Still reactive!
    }
}
```

### Key Differences

| Feature | ObservableObject (Old) | @Observable (New) |
|---------|----------------------|-------------------|
| **Property Wrapper for Properties** | @Published required | None needed |
| **View Property Wrapper** | @StateObject/@ObservedObject | None needed (just `var`) |
| **Protocol Conformance** | Must conform to ObservableObject | Just add @Observable macro |
| **Observation Granularity** | Whole object | Per-property (more efficient!) |
| **Boilerplate** | High | Minimal |
| **iOS Version** | iOS 13+ | iOS 17+ |

### How @Observable Works

```swift
@Observable
class ShoppingCart {
    var items: [Item] = []
    var total: Double = 0.0
    var discountCode: String = ""

    func addItem(_ item: Item) {
        items.append(item)
        calculateTotal()
    }

    private func calculateTotal() {
        total = items.reduce(0) { $0 + $1.price }
    }
}

struct CartView: View {
    var cart = ShoppingCart()  // No @StateObject!

    var body: some View {
        VStack {
            // View automatically updates when cart.items or cart.total changes
            Text("Items: \(cart.items.count)")
            Text("Total: $\(cart.total, specifier: "%.2f")")

            Button("Add Item") {
                cart.addItem(Item(name: "Widget", price: 9.99))
            }
        }
    }
}
```

### @Observable with Different Patterns

#### Pattern 1: Simple View Model
```swift
@Observable
class CounterViewModel {
    var count = 0

    func increment() {
        count += 1
    }
}

struct CounterView: View {
    var viewModel = CounterViewModel()

    var body: some View {
        VStack {
            Text("Count: \(viewModel.count)")
            Button("Increment") {
                viewModel.increment()
            }
        }
    }
}
```

#### Pattern 2: Passing to Child Views
```swift
@Observable
class AppState {
    var user: User?
    var isAuthenticated = false
}

struct ParentView: View {
    var appState = AppState()

    var body: some View {
        ChildView(appState: appState)  // Just pass it!
    }
}

struct ChildView: View {
    var appState: AppState  // No property wrapper!

    var body: some View {
        if appState.isAuthenticated {
            Text("Welcome, \(appState.user?.name ?? "User")")
        }
    }
}
```

#### Pattern 3: Environment (replaces @EnvironmentObject)
```swift
@Observable
class AppSettings {
    var isDarkMode = false
    var fontSize: Double = 16
}

@main
struct MyApp: App {
    var settings = AppSettings()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(settings)  // Inject into environment
        }
    }
}

struct SettingsView: View {
    @Environment(AppSettings.self) var settings  // Access from environment

    var body: some View {
        Toggle("Dark Mode", isOn: $settings.isDarkMode)
    }
}
```

### Bindable for Two-Way Binding

When you need `$` syntax for binding with @Observable, use `@Bindable`:

```swift
@Observable
class FormViewModel {
    var username = ""
    var email = ""
}

struct FormView: View {
    var viewModel = FormViewModel()

    var body: some View {
        Form {
            // ❌ This won't work: TextField("Username", text: $viewModel.username)

            // ✅ Use @Bindable to get $ syntax
            BindableFormFields(viewModel: viewModel)
        }
    }
}

struct BindableFormFields: View {
    @Bindable var viewModel: FormViewModel

    var body: some View {
        TextField("Username", text: $viewModel.username)
        TextField("Email", text: $viewModel.email)
    }
}
```

### Computed Properties Still Work

```swift
@Observable
class UserProfileViewModel {
    var firstName = ""
    var lastName = ""

    // Computed properties are automatically observed!
    var fullName: String {
        "\(firstName) \(lastName)"
    }

    var initials: String {
        let first = firstName.prefix(1)
        let last = lastName.prefix(1)
        return "\(first)\(last)"
    }
}

struct ProfileView: View {
    var viewModel = UserProfileViewModel()

    var body: some View {
        VStack {
            Text("Full Name: \(viewModel.fullName)")  // Updates when firstName/lastName change
            Text("Initials: \(viewModel.initials)")
        }
    }
}
```

### Ignoring Properties with @ObservationIgnored

```swift
@Observable
class DataViewModel {
    var publicData = ""              // Observed

    @ObservationIgnored
    var cachedData: [String] = []    // Not observed (won't trigger view updates)

    @ObservationIgnored
    private var internalState = 0    // Not observed
}
```

### Migration Guide: ObservableObject → @Observable

```swift
// BEFORE (iOS 13-16)
class OldViewModel: ObservableObject {
    @Published var name = ""
    @Published var items: [Item] = []

    var notPublished = ""  // Won't trigger updates
}

struct OldView: View {
    @StateObject private var viewModel = OldViewModel()

    var body: some View {
        Text(viewModel.name)
    }
}

// AFTER (iOS 17+)
@Observable
class NewViewModel {
    var name = ""       // Remove @Published
    var items: [Item] = []

    @ObservationIgnored
    var notPublished = ""  // Explicitly mark as not observed
}

struct NewView: View {
    var viewModel = NewViewModel()  // Remove @StateObject

    var body: some View {
        Text(viewModel.name)
    }
}
```

### Performance Benefits

The new Observation framework is **more efficient** than ObservableObject:

```swift
@Observable
class OptimizedViewModel {
    var name = ""      // Only observes THIS property
    var email = ""     // Only observes THIS property
    var age = 0        // Only observes THIS property
}

// With ObservableObject:
// - Any @Published property change triggers ALL observers
// - Views re-render even if they don't use the changed property

// With @Observable:
// - Only views that read the changed property re-render
// - Fine-grained observation at the property level
```

### When to Use Which

| Use Case | Use This |
|----------|----------|
| **New iOS 17+ project** | @Observable |
| **Supporting iOS 13-16** | ObservableObject |
| **Need backwards compatibility** | ObservableObject |
| **SwiftUI-only modern app** | @Observable |
| **Integrating with UIKit** | Either works |

### Mixing Old and New (Transition Period)

You can mix both in the same app:

```swift
// Old code (still works)
class LegacyViewModel: ObservableObject {
    @Published var data = ""
}

struct LegacyView: View {
    @StateObject var viewModel = LegacyViewModel()
    // ...
}

// New code
@Observable
class ModernViewModel {
    var data = ""
}

struct ModernView: View {
    var viewModel = ModernViewModel()
    // ...
}
```

### Common Gotchas

#### 1. @Bindable is needed for bindings
```swift
@Observable class VM { var text = "" }

struct MyView: View {
    var vm = VM()

    var body: some View {
        // ❌ Won't work
        // TextField("Text", text: $vm.text)

        // ✅ Correct - wrap in child view with @Bindable
        InputField(vm: vm)
    }
}

struct InputField: View {
    @Bindable var vm: VM

    var body: some View {
        TextField("Text", text: $vm.text)  // ✅ Now works!
    }
}
```

#### 2. Environment syntax is different
```swift
// Old way
struct View1: View {
    @EnvironmentObject var settings: AppSettings  // ObservableObject
}

// New way
struct View2: View {
    @Environment(AppSettings.self) var settings  // @Observable
}
```

#### 3. @Observable only works with classes
```swift
// ❌ Error: Can't use @Observable with structs
@Observable
struct MyStruct {  // Won't compile!
    var value = 0
}

// ✅ Use class
@Observable
class MyClass {
    var value = 0
}
```

### Advanced @Observable Patterns (iOS 17+)

#### Nested @Observable Objects
```swift
@Observable
class Address {
    var street = ""
    var city = ""
    var zipCode = ""
}

@Observable
class UserProfile {
    var name = ""
    var address = Address()  // Nested @Observable
    var contacts: [Contact] = []

    // Changes to nested objects are automatically observed!
}

struct ProfileEditView: View {
    var profile = UserProfile()

    var body: some View {
        Form {
            TextField("Name", text: Bindable(profile).name)

            // Nested object properties are reactive too
            TextField("Street", text: Bindable(profile.address).street)
            TextField("City", text: Bindable(profile.address).city)

            Text("Full Address: \(profile.address.street), \(profile.address.city)")
                // Updates when ANY address property changes!
        }
    }
}
```

#### Async Operations with @Observable
```swift
@Observable
class WeatherViewModel {
    var temperature: Double?
    var isLoading = false
    var errorMessage: String?

    // Use nonisolated for async operations
    nonisolated init() {}

    func fetchWeather() async {
        // @MainActor ensures UI updates on main thread
        await MainActor.run {
            isLoading = true
            errorMessage = nil
        }

        do {
            // Simulate API call
            try await Task.sleep(nanoseconds: 1_000_000_000)
            let temp = Double.random(in: -20...40)

            await MainActor.run {
                self.temperature = temp
                self.isLoading = false
            }
        } catch {
            await MainActor.run {
                self.errorMessage = error.localizedDescription
                self.isLoading = false
            }
        }
    }
}

struct WeatherView: View {
    var viewModel = WeatherViewModel()

    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else if let temp = viewModel.temperature {
                Text("Temperature: \(temp, specifier: "%.1f")°C")
            } else if let error = viewModel.errorMessage {
                Text("Error: \(error)")
                    .foregroundColor(.red)
            }

            Button("Refresh") {
                Task {
                    await viewModel.fetchWeather()
                }
            }
        }
        .task {
            await viewModel.fetchWeather()
        }
    }
}
```

#### @Observable with Combine Publishers
```swift
import Combine

@Observable
class SearchViewModel {
    var searchText = ""
    var searchResults: [String] = []
    private var cancellables = Set<AnyCancellable>()

    init() {
        // @Observable properties can still work with Combine
        $searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .sink { [weak self] text in
                self?.performSearch(text)
            }
            .store(in: &cancellables)
    }

    private func performSearch(_ query: String) {
        guard !query.isEmpty else {
            searchResults = []
            return
        }

        // Simulate search
        searchResults = ["Result for: \(query)"]
    }
}
```

#### Testing @Observable Classes
```swift
import XCTest
@testable import MyApp

// Testing @Observable is simpler than ObservableObject
final class ViewModelTests: XCTestCase {
    func testObservableViewModel() {
        // Given
        let viewModel = UserViewModel()
        var observedChanges = 0

        // Use withObservationTracking for testing
        withObservationTracking {
            _ = viewModel.name
        } onChange: {
            observedChanges += 1
        }

        // When
        viewModel.name = "John"

        // Then
        XCTAssertEqual(observedChanges, 1)
        XCTAssertEqual(viewModel.name, "John")
    }

    func testNestedObservation() {
        let profile = UserProfile()
        var addressChanges = 0

        withObservationTracking {
            _ = profile.address.city
        } onChange: {
            addressChanges += 1
        }

        profile.address.city = "San Francisco"

        XCTAssertEqual(addressChanges, 1)
    }
}
```

#### @Observable with SwiftData (iOS 17+)
```swift
import SwiftData

// SwiftData models are automatically observable in iOS 17+
@Model
class TodoItem {
    var title: String
    var isCompleted: Bool
    var createdDate: Date

    init(title: String) {
        self.title = title
        self.isCompleted = false
        self.createdDate = Date()
    }
}

// ViewModel using SwiftData models
@Observable
class TodoListViewModel {
    var todos: [TodoItem] = []
    var showCompleted = true

    var visibleTodos: [TodoItem] {
        showCompleted ? todos : todos.filter { !$0.isCompleted }
    }

    func toggleTodo(_ todo: TodoItem) {
        todo.isCompleted.toggle()
        // SwiftData model changes are automatically observed!
    }
}
```

#### Optimizing Performance with @Observable
```swift
@Observable
class OptimizedViewModel {
    var frequentlyChangingValue = 0

    @ObservationIgnored
    var cachedComputations: [String: Any] = [:]

    @ObservationIgnored
    private var updateTimer: Timer?

    // Use computed properties wisely
    var expensiveComputation: String {
        // This runs every time it's accessed if dependencies change
        // Consider caching if expensive
        if let cached = cachedComputations["expensive"] as? String {
            return cached
        }

        let result = performExpensiveWork()
        cachedComputations["expensive"] = result
        return result
    }

    private func performExpensiveWork() -> String {
        // Expensive operation
        return "Result"
    }
}
```

---

## Decision Flow Chart

```
Need to store data in a view?
│
├─ Is it a simple value type (Int, String, Bool, struct)?
│  └─ Use @State
│
├─ Is it a reference type (class)?
│  │
│  ├─ Does THIS view create and own it?
│  │  └─ Use @StateObject
│  │
│  └─ Is it passed from a parent?
│     └─ Use @ObservedObject
│
└─ Need to modify parent's data?
   └─ Use @Binding

Need data across many unrelated views?
└─ Use @EnvironmentObject
```

---

## Common Patterns & Best Practices

### Pattern 1: View Model with Child Views
```swift
struct ParentView: View {
    @StateObject private var viewModel = ParentViewModel()
    
    var body: some View {
        VStack {
            HeaderView(title: $viewModel.title)
            ContentView(viewModel: viewModel)
        }
    }
}

struct HeaderView: View {
    @Binding var title: String  // Two-way binding for editing
    
    var body: some View {
        TextField("Title", text: $title)
    }
}

struct ContentView: View {
    @ObservedObject var viewModel: ParentViewModel  // Observe parent's VM
    
    var body: some View {
        Text(viewModel.content)
    }
}
```

### Pattern 2: App-Wide State
```swift
class AppState: ObservableObject {
    @Published var user: User?
    @Published var isAuthenticated = false
}

@main
struct MyApp: App {
    @StateObject private var appState = AppState()
    
    var body: some Scene {
        WindowGroup {
            if appState.isAuthenticated {
                HomeView()
            } else {
                LoginView()
            }
        }
        .environmentObject(appState)
    }
}
```

### Pattern 3: Reusable Components
```swift
struct RatingView: View {
    @Binding var rating: Int
    let maxRating: Int
    
    var body: some View {
        HStack {
            ForEach(1...maxRating, id: \.self) { star in
                Image(systemName: star <= rating ? "star.fill" : "star")
                    .onTapGesture { rating = star }
            }
        }
    }
}

// Usage
struct ProductView: View {
    @State private var rating = 3
    
    var body: some View {
        RatingView(rating: $rating, maxRating: 5)
    }
}
```

---

## Common Mistakes & How to Fix

### Mistake 1: Using @ObservedObject When You Should Use @StateObject
```swift
// ❌ WRONG
struct MyView: View {
    @ObservedObject var viewModel = ViewModel()  // Recreated on every render!
    
    var body: some View { ... }
}

// ✅ CORRECT
struct MyView: View {
    @StateObject private var viewModel = ViewModel()  // Created once
    
    var body: some View { ... }
}
```

### Mistake 2: Forgetting $ When Passing Bindings
```swift
// ❌ WRONG
ChildView(isOn: isOn)  // Passes the value, not a binding

// ✅ CORRECT
ChildView(isOn: $isOn)  // Passes a binding
```

### Mistake 3: Making @State Public
```swift
// ❌ WRONG
struct MyView: View {
    @State var count = 0  // Should be private
}

// ✅ CORRECT
struct MyView: View {
    @State private var count = 0
}
```

### Mistake 4: Using @State for Reference Types
```swift
// ❌ WRONG
struct MyView: View {
    @State var viewModel = ViewModel()  // Reference type with @State
}

// ✅ CORRECT
struct MyView: View {
    @StateObject private var viewModel = ViewModel()
}
```

---

## Interview Questions You Might Get

### Q: "What's the difference between @State and @StateObject?"
**A:** @State is for value types (structs, enums, primitives) that are owned by the view, while @StateObject is for reference types (classes conforming to ObservableObject) that are owned and created by the view. @StateObject ensures the object is only created once during the view's lifetime, even if the view re-renders.

### Q: "When would you use @ObservedObject vs @StateObject?"
**A:** Use @StateObject when the view creates and owns the object - it ensures the object is initialized once and persists for the view's lifetime. Use @ObservedObject when the object is created elsewhere (like a parent view) and passed in - the view just observes it but doesn't own it.

### Q: "How do @Binding and @State work together?"
**A:** @State creates a source of truth in a parent view, and @Binding creates a two-way connection to that state from a child view. The parent passes a binding using the $ prefix ($propertyName), allowing the child to both read and write the parent's state.

### Q: "Why would you use @EnvironmentObject instead of passing @ObservedObject through the view hierarchy?"
**A:** @EnvironmentObject avoids "prop drilling" - passing data through multiple intermediate views that don't need it. It's ideal for app-wide state like user authentication or theme settings that many disconnected views need to access.

### Q: "What happens if you use @ObservedObject where you should use @StateObject?"
**A:** The object will be recreated every time the view re-renders, losing all its state. This is a common bug that can cause unexpected behavior and performance issues.

### Q: "What's the difference between ObservableObject and @Observable?"
**A:** ObservableObject is the old iOS 13-16 approach requiring `@Published` on properties and `@StateObject/@ObservedObject` in views. @Observable is the modern iOS 17+ approach using a macro - no property wrappers needed on properties or in views. @Observable is more efficient with per-property observation instead of whole-object observation, and has less boilerplate.

### Q: "When would you use @Observable vs ObservableObject?"
**A:** Use @Observable for new iOS 17+ projects or when you can drop support for older iOS versions. It's simpler, more efficient, and has less boilerplate. Use ObservableObject when you need to support iOS 13-16 or are maintaining existing code. You can mix both in the same app during migration.

### Q: "How do you use bindings with @Observable?"
**A:** Use `@Bindable` wrapper in the view that needs binding access. Unlike @StateObject which automatically provides bindings, @Observable requires explicit @Bindable annotation to get the `$` syntax for two-way binding.

Example:
```swift
@Observable class VM { var text = "" }

struct FormView: View {
    @Bindable var viewModel: VM  // @Bindable provides $ syntax

    var body: some View {
        TextField("Text", text: $viewModel.text)
    }
}
```

### Q: "What's @ObservationIgnored?"
**A:** It's used with @Observable to mark properties that shouldn't trigger view updates. By default, all properties in an @Observable class are observed. Use @ObservationIgnored for cached data, internal state, or properties that don't affect the UI.

Example:
```swift
@Observable
class ViewModel {
    var displayName = ""  // Observed

    @ObservationIgnored
    var cachedData: [Item] = []  // Not observed
}
```

---

## Memory Aid

**Ownership:**
- @State: I own it (value type)
- @StateObject: I own it (reference type)
- @Binding: Someone else owns it, I can modify it
- @ObservedObject: Someone else owns it, they passed it to me
- @EnvironmentObject: Someone up the tree owns it, everyone can access it

**Type:**
- Value types → @State or @Binding
- Reference types → @StateObject, @ObservedObject, or @EnvironmentObject

**Lifecycle:**
- @State: Tied to view, recreated with view
- @StateObject: Created once, survives view updates
- @ObservedObject: Managed externally
- @EnvironmentObject: Managed by ancestor

---

## Quick Reference Code

### Old Way (iOS 13-16)
```swift
// LOCAL VALUE TYPE
@State private var count = 0

// LOCAL REFERENCE TYPE (ObservableObject)
@StateObject private var viewModel = ViewModel()

// RECEIVED BINDING
@Binding var isOn: Bool

// PASSED REFERENCE TYPE
@ObservedObject var sharedVM: SharedViewModel

// ENVIRONMENT SHARED TYPE
@EnvironmentObject var settings: AppSettings

// PASSING A BINDING
ChildView(text: $myText)

// INJECTING ENVIRONMENT
ContentView()
    .environmentObject(mySettings)
```

### New Way (iOS 17+)
```swift
// LOCAL VALUE TYPE (unchanged)
@State private var count = 0

// LOCAL REFERENCE TYPE (@Observable)
var viewModel = ViewModel()  // No property wrapper!

// RECEIVED BINDING (unchanged)
@Binding var isOn: Bool

// PASSED REFERENCE TYPE (@Observable)
var sharedVM: SharedViewModel  // No @ObservedObject!

// BINDABLE FOR $ SYNTAX
@Bindable var viewModel: ViewModel  // For TextField, Toggle, etc.

// ENVIRONMENT (@Observable)
@Environment(AppSettings.self) var settings

// PASSING OBSERVABLE
ChildView(viewModel: viewModel)

// INJECTING ENVIRONMENT
ContentView()
    .environment(mySettings)
```

Good luck with your interview! These concepts are fundamental to SwiftUI and will definitely come up.
