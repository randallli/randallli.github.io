---
layout: page
title: "UIKit Mastery Guide"
subtitle: "Deep Technical Knowledge for Senior iOS Engineers"
---

# UIKit Mastery Guide
## Deep Technical Knowledge for Senior iOS Engineers

---

## Table of Contents
1. [View Controller Lifecycle](#view-controller-lifecycle)
2. [Auto Layout Deep Dive](#auto-layout-deep-dive)
3. [View Rendering Pipeline](#view-rendering-pipeline)
4. [UICollectionView Advanced](#uicollectionview-advanced)
5. [UITableView Optimization](#uitableview-optimization)
6. [Memory Management](#memory-management)
7. [Responder Chain & Touch Handling](#responder-chain--touch-handling)
8. [Animations & CALayer](#animations--calayer)
9. [UIKit → SwiftUI Bridge](#uikit--swiftui-bridge)
10. [Common Gotchas & Interview Questions](#common-gotchas--interview-questions)

---

## View Controller Lifecycle

### The Complete Order

```swift
// Initial Load
init(coder:) or init(nibName:bundle:)
    ↓
loadView()              // Creates the view hierarchy
    ↓
viewDidLoad()           // View hierarchy is loaded into memory
    ↓
viewWillAppear(_:)      // About to be added to view hierarchy
    ↓
viewWillLayoutSubviews() // About to layout subviews
    ↓
viewDidLayoutSubviews()  // Subviews have been laid out
    ↓
viewDidAppear(_:)       // Now visible on screen

// Going Away
viewWillDisappear(_:)   // About to be removed
    ↓
viewDidDisappear(_:)    // No longer visible
    ↓
deinit                  // Being deallocated (if no retain cycles)
```

### Critical Details

#### loadView()
```swift
// When to override:
override func loadView() {
    // ❌ DON'T call super.loadView()
    // ❌ DON'T access self.view
    
    // Create your custom root view
    let customView = CustomView()
    self.view = customView  // Set the root view
}

// When NOT to override:
// - If you're using Storyboards/XIBs
// - If you just want to add subviews (use viewDidLoad instead)
```

#### viewDidLoad()
```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    // ✅ Good uses:
    // - One-time setup
    // - Add subviews
    // - Register cells
    // - Set up observers
    // - Initialize data models
    
    // ❌ Don't:
    // - Access view.bounds/frame (not finalized yet)
    // - Start animations
    // - Fetch user-visible data (use viewWillAppear)
}
```

#### viewWillAppear vs viewDidAppear
```swift
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    
    // ✅ Good uses:
    // - Refresh data that might have changed
    // - Start observing notifications
    // - Update UI based on current state
    // - Analytics (screen view tracking)
    
    // Called every time view appears (not just once)
    // View bounds are known here
}

override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    
    // ✅ Good uses:
    // - Start animations
    // - Begin playing video
    // - Request user permissions
    // - Show popups/alerts
    // - Start timers
    
    // View is fully visible and interactive
}
```

#### viewWillLayoutSubviews / viewDidLayoutSubviews
```swift
override func viewWillLayoutSubviews() {
    super.viewWillLayoutSubviews()
    
    // Called multiple times:
    // - When view first appears
    // - When device rotates
    // - When keyboard appears/disappears
    // - When view.setNeedsLayout() is called
    
    // ✅ Good for: Reading current bounds
}

override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    
    // ✅ Good uses:
    // - Update frames based on final layout
    // - Adjust CALayer properties
    // - Update gradient layers
    // - Adjust scroll view content size
    
    // ⚠️ Be careful: Can be called many times
    // Don't do expensive operations here
}
```

### Common Interview Questions

**Q: "When is viewDidLoad called vs viewWillAppear?"**
- `viewDidLoad()`: Once, when view is first loaded into memory
- `viewWillAppear()`: Every time the view is about to appear (after navigation, modal dismiss, etc.)

**Q: "Where should you start a network request?"**
- Depends on use case:
  - **One-time data**: `viewDidLoad()` if it won't change
  - **Refreshable data**: `viewWillAppear()` to get latest
  - **User-initiated**: In response to user action

**Q: "Why might viewDidDisappear be called but not deinit?"**
- View controller is retained somewhere (navigation stack, retain cycle, etc.)
- Strong reference from closure, delegate, notification observer

---

## Auto Layout Deep Dive

### Constraint Priorities

```swift
// Priority scale: 1 to 1000
// UILayoutPriority values:
.required           // 1000 - Must be satisfied
.defaultHigh        // 750
.defaultLow         // 250
.fittingSizeLevel   // 50
```

### Content Hugging vs Compression Resistance

```swift
// Content Hugging: Resistance to growing
// "I don't want to be bigger than my intrinsic content size"
// Default: 250

label.setContentHuggingPriority(.defaultHigh, for: .horizontal)
// Higher priority = more resistant to growing

// Compression Resistance: Resistance to shrinking
// "I don't want to be smaller than my intrinsic content size"
// Default: 750

label.setContentCompressionResistancePriority(.required, for: .horizontal)
// Higher priority = more resistant to shrinking
```

### Common Layout Scenarios

#### Scenario 1: Two Labels, One Should Expand
```swift
// [Label1] [     Label2     ]
//  ^^^^^    ^^^^^^^^^^^^^^^^
//  hugs     can grow

label1.setContentHuggingPriority(.defaultHigh, for: .horizontal)
label2.setContentHuggingPriority(.defaultLow, for: .horizontal)
```

#### Scenario 2: Two Labels, One Should Truncate
```swift
// [Label1...] [Label2]
//  ^^^^^^^^^^  ^^^^^^^
//  truncates   protected

label1.setContentCompressionResistancePriority(.defaultLow, for: .horizontal)
label2.setContentCompressionResistancePriority(.required, for: .horizontal)
```

#### Scenario 3: Dynamic Height with Multi-line Label
```swift
let label = UILabel()
label.numberOfLines = 0  // Allow multiple lines
label.translatesAutoresizingMaskIntoConstraints = false

// Constraints
NSLayoutConstraint.activate([
    label.topAnchor.constraint(equalTo: view.topAnchor, constant: 20),
    label.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    label.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16)
    // No height constraint - it calculates from content
])

// UILabel provides intrinsic content size automatically
```

### Constraint Conflicts & Debugging

```swift
// Enable constraint debugging
// Add to scheme: -UIConstraintBasedLayoutLogUnsatisfiable YES

// Identify constraints with identifiers
let constraint = label.widthAnchor.constraint(equalToConstant: 100)
constraint.identifier = "LabelWidth"
constraint.isActive = true

// Break at symbolic breakpoint: UIViewAlertForUnsatisfiableConstraints
```

### Layout Anchors vs NSLayoutConstraint

```swift
// Modern way (Layout Anchors) - PREFER THIS
label.centerXAnchor.constraint(equalTo: view.centerXAnchor).isActive = true

// Old way (NSLayoutConstraint)
NSLayoutConstraint(
    item: label,
    attribute: .centerX,
    relatedBy: .equal,
    toItem: view,
    attribute: .centerX,
    multiplier: 1.0,
    constant: 0
).isActive = true
```

### Safe Area Considerations

```swift
// iOS 11+: Safe Area Layout Guide
view.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor)

// Pre-iOS 11 (don't use these anymore):
// topLayoutGuide, bottomLayoutGuide (deprecated)
```

### Intrinsic Content Size

```swift
// Custom views can provide their own intrinsic content size
class CustomView: UIView {
    override var intrinsicContentSize: CGSize {
        return CGSize(width: 100, height: 50)
    }
    
    func contentChanged() {
        // Tell Auto Layout to recalculate
        invalidateIntrinsicContentSize()
    }
}
```

### Common Auto Layout Mistakes

```swift
// ❌ MISTAKE 1: Forgetting to set translatesAutoresizingMaskIntoConstraints
let view = UIView()
view.translatesAutoresizingMaskIntoConstraints = true  // Default!
// This creates conflicting constraints with auto-resizing mask

// ✅ CORRECT
view.translatesAutoresizingMaskIntoConstraints = false

// ❌ MISTAKE 2: Not activating constraints
let constraint = view.widthAnchor.constraint(equalToConstant: 100)
// Forgot: constraint.isActive = true

// ✅ CORRECT
constraint.isActive = true
// Or use NSLayoutConstraint.activate([constraint])

// ❌ MISTAKE 3: Over-constraining
view.widthAnchor.constraint(equalToConstant: 100).isActive = true
view.widthAnchor.constraint(equalToConstant: 200).isActive = true  // Conflict!

// ✅ CORRECT: Update existing constraint or deactivate old one
```

---

## View Rendering Pipeline

### The Three Steps

```
1. Update Constraints (Bottom-Up)
   updateConstraints() / setNeedsUpdateConstraints()
        ↓
2. Layout Subviews (Top-Down)  
   layoutSubviews() / setNeedsLayout() / layoutIfNeeded()
        ↓
3. Display/Draw (Top-Down)
   draw(_:) / setNeedsDisplay()
```

### Update Constraints Phase

```swift
override func updateConstraints() {
    // Called when constraints need updating
    // Bottom-up: children before parents
    
    // Update your constraints here
    myConstraint.constant = newValue
    
    // MUST call super at the END
    super.updateConstraints()
}

// Trigger constraint update (async)
view.setNeedsUpdateConstraints()
```

### Layout Phase

```swift
override func layoutSubviews() {
    // Called when view's bounds change or constraints are resolved
    // Top-down: parents before children
    
    // MUST call super FIRST
    super.layoutSubviews()
    
    // Update frames, CALayers, etc.
    gradientLayer.frame = bounds
    
    // Don't call setNeedsLayout() here (infinite loop!)
}

// Trigger layout (async)
view.setNeedsLayout()

// Force immediate layout
view.layoutIfNeeded()
```

### Display/Draw Phase

```swift
override func draw(_ rect: CGRect) {
    // Called when view needs to render its content
    // Use Core Graphics to draw
    
    guard let context = UIGraphicsGetCurrentContext() else { return }
    
    context.setFillColor(UIColor.red.cgColor)
    context.fill(rect)
    
    // ⚠️ EXPENSIVE - avoid if possible
    // Use UIImageView, CALayer, or other optimized approaches
}

// Trigger redraw (async)
view.setNeedsDisplay()

// Trigger partial redraw
view.setNeedsDisplay(specificRect)
```

### When Each Method is Called

| Method | When Called | Frequency |
|--------|-------------|-----------|
| updateConstraints() | Constraints change | As needed |
| layoutSubviews() | Bounds change, constraints resolve | Often |
| draw(_:) | View needs rendering | Rarely (cached) |

### setNeedsLayout vs layoutIfNeeded

```swift
// setNeedsLayout() - Async
UIView.animate(withDuration: 0.3) {
    self.view.setNeedsLayout()  // ❌ Won't animate
}

// layoutIfNeeded() - Immediate
UIView.animate(withDuration: 0.3) {
    self.constraint.constant = 100
    self.view.layoutIfNeeded()  // ✅ Animates the layout change
}
```

### Common Pattern: Animating Constraint Changes

```swift
// 1. Update constraint
heightConstraint.constant = 200

// 2. Mark for layout
view.setNeedsLayout()

// 3. Animate the layout
UIView.animate(withDuration: 0.3) {
    self.view.layoutIfNeeded()
}
```

---

## UICollectionView Advanced

### Custom Flow Layout

```swift
class CustomFlowLayout: UICollectionViewFlowLayout {
    override func prepare() {
        super.prepare()
        
        guard let collectionView = collectionView else { return }
        
        // Calculate item size based on collection view size
        let availableWidth = collectionView.bounds.width - sectionInset.left - sectionInset.right
        let itemWidth = (availableWidth / 2) - minimumInteritemSpacing
        
        itemSize = CGSize(width: itemWidth, height: itemWidth)
    }
    
    override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
        // Return true if layout should recalculate on bounds change (e.g., rotation)
        guard let oldBounds = collectionView?.bounds else { return false }
        return oldBounds.width != newBounds.width
    }
}
```

### Completely Custom Layout

```swift
class WaterfallLayout: UICollectionViewLayout {
    private var cache: [UICollectionViewLayoutAttributes] = []
    private var contentHeight: CGFloat = 0
    private var contentWidth: CGFloat {
        guard let collectionView = collectionView else { return 0 }
        let insets = collectionView.contentInset
        return collectionView.bounds.width - (insets.left + insets.right)
    }
    
    override var collectionViewContentSize: CGSize {
        return CGSize(width: contentWidth, height: contentHeight)
    }
    
    override func prepare() {
        guard cache.isEmpty,
              let collectionView = collectionView else { return }
        
        // Calculate layout attributes for each item
        let columnWidth = contentWidth / 2
        var xOffsets: [CGFloat] = []
        var yOffsets: [CGFloat] = Array(repeating: 0, count: 2)
        
        for column in 0..<2 {
            xOffsets.append(CGFloat(column) * columnWidth)
        }
        
        for item in 0..<collectionView.numberOfItems(inSection: 0) {
            let indexPath = IndexPath(item: item, section: 0)
            
            // Determine which column to use (shortest)
            let column = yOffsets[0] < yOffsets[1] ? 0 : 1
            
            let x = xOffsets[column]
            let y = yOffsets[column]
            
            // Calculate height (from delegate or fixed)
            let height: CGFloat = 200  // Or get from delegate
            
            let frame = CGRect(x: x, y: y, width: columnWidth, height: height)
            
            let attributes = UICollectionViewLayoutAttributes(forCellWith: indexPath)
            attributes.frame = frame
            cache.append(attributes)
            
            contentHeight = max(contentHeight, frame.maxY)
            yOffsets[column] += height
        }
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        return cache.filter { $0.frame.intersects(rect) }
    }
    
    override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        return cache[indexPath.item]
    }
    
    override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
        return true
    }
    
    override func invalidateLayout() {
        super.invalidateLayout()
        cache.removeAll()
        contentHeight = 0
    }
}
```

### Prefetching

```swift
class MyViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        collectionView.prefetchDataSource = self
    }
}

extension MyViewController: UICollectionViewDataSourcePrefetching {
    func collectionView(_ collectionView: UICollectionView, 
                       prefetchItemsAt indexPaths: [IndexPath]) {
        // Start loading data/images for these cells
        // They're about to scroll into view
        
        for indexPath in indexPaths {
            imageLoader.loadImage(for: items[indexPath.item])
        }
    }
    
    func collectionView(_ collectionView: UICollectionView, 
                       cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
        // User scrolled away, cancel these operations
        
        for indexPath in indexPaths {
            imageLoader.cancelLoad(for: items[indexPath.item])
        }
    }
}
```

### Compositional Layout (Modern Approach)

```swift
// iOS 13+
func createLayout() -> UICollectionViewLayout {
    let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(0.5),
        heightDimension: .fractionalHeight(1.0)
    )
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
    item.contentInsets = NSDirectionalEdgeInsets(top: 5, leading: 5, bottom: 5, trailing: 5)
    
    let groupSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .absolute(200)
    )
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])
    
    let section = NSCollectionLayoutSection(group: group)
    section.contentInsets = NSDirectionalEdgeInsets(top: 10, leading: 10, bottom: 10, trailing: 10)
    
    return UICollectionViewCompositionalLayout(section: section)
}
```

### Diffable Data Source (Modern Approach)

```swift
// iOS 13+
enum Section {
    case main
}

typealias DataSource = UICollectionViewDiffableDataSource<Section, Item>
typealias Snapshot = NSDiffableDataSourceSnapshot<Section, Item>

class MyViewController: UIViewController {
    private var dataSource: DataSource!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        configureDataSource()
        applyInitialSnapshot()
    }
    
    func configureDataSource() {
        dataSource = DataSource(collectionView: collectionView) { 
            collectionView, indexPath, item in
            
            let cell = collectionView.dequeueReusableCell(
                withReuseIdentifier: "Cell",
                for: indexPath
            ) as! MyCell
            
            cell.configure(with: item)
            return cell
        }
    }
    
    func applyInitialSnapshot() {
        var snapshot = Snapshot()
        snapshot.appendSections([.main])
        snapshot.appendItems(items)
        
        dataSource.apply(snapshot, animatingDifferences: true)
    }
    
    func updateItems(_ newItems: [Item]) {
        var snapshot = Snapshot()
        snapshot.appendSections([.main])
        snapshot.appendItems(newItems)
        
        dataSource.apply(snapshot, animatingDifferences: true)
        // Automatically calculates diff and animates!
    }
}
```

---

## UITableView Optimization

### Cell Reuse Done Right

```swift
// ❌ Old way (string identifiers, casting)
let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath) as! MyCell

// ✅ Better way (type-safe)
class MyCell: UITableViewCell {
    static let reuseIdentifier = String(describing: self)
}

// Register
tableView.register(MyCell.self, forCellReuseIdentifier: MyCell.reuseIdentifier)

// Dequeue
let cell = tableView.dequeueReusableCell(
    withIdentifier: MyCell.reuseIdentifier, 
    for: indexPath
) as! MyCell
```

### Automatic Height Calculation

```swift
// Self-sizing cells
tableView.rowHeight = UITableView.automaticDimension
tableView.estimatedRowHeight = 100  // Important for performance!

// In your cell
class MyCell: UITableViewCell {
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        
        // Setup constraints
        // Make sure top and bottom constraints are set!
        NSLayoutConstraint.activate([
            label.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 8),
            label.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            label.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),
            label.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -8)
            // ↑ This bottom constraint is CRITICAL for auto height
        ])
    }
}
```

### Performance Optimization

```swift
// 1. Use estimated heights (prevents upfront calculation of all rows)
tableView.estimatedRowHeight = 100
tableView.estimatedSectionHeaderHeight = 40
tableView.estimatedSectionFooterHeight = 0

// 2. Implement height caching for complex calculations
private var heightCache: [IndexPath: CGFloat] = [:]

func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    if let cached = heightCache[indexPath] {
        return cached
    }
    
    let height = calculateHeight(for: indexPath)
    heightCache[indexPath] = height
    return height
}

// 3. Avoid expensive operations in cellForRowAt
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    
    // ❌ Don't do this
    cell.imageView?.image = processImage(largeImage)  // Expensive!
    
    // ✅ Do this
    cell.imageView?.image = cachedImages[indexPath.item]
    
    return cell
}

// 4. Load images asynchronously
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath) as! ImageCell
    
    let item = items[indexPath.row]
    cell.configure(with: item)
    
    // Load image async
    imageLoader.loadImage(for: item.imageURL) { [weak cell] image in
        // Check if cell is still for this item (reuse protection)
        guard cell?.imageURL == item.imageURL else { return }
        cell?.imageView.image = image
    }
    
    return cell
}
```

### Cell Reuse Pitfalls

```swift
class MyCell: UITableViewCell {
    private var imageLoadTask: URLSessionDataTask?
    
    override func prepareForReuse() {
        super.prepareForReuse()
        
        // ✅ CRITICAL: Clean up when cell is reused
        imageView.image = nil
        imageLoadTask?.cancel()
        imageLoadTask = nil
        
        // Reset any cell state
        titleLabel.text = nil
        isExpanded = false
    }
}
```

### Batch Updates

```swift
// Update multiple rows efficiently
tableView.performBatchUpdates({
    tableView.insertRows(at: [indexPath1, indexPath2], with: .automatic)
    tableView.deleteRows(at: [indexPath3], with: .fade)
    tableView.reloadRows(at: [indexPath4], with: .none)
}, completion: nil)

// Or use the block-based API
tableView.beginUpdates()
tableView.insertRows(at: [indexPath], with: .automatic)
tableView.endUpdates()
```

---

## Memory Management

### Strong, Weak, Unowned

```swift
// Strong (default)
class Parent {
    var child: Child?  // Strong reference
}

class Child {
    var parent: Parent?  // ❌ Retain cycle!
}

// ✅ Fix with weak
class Child {
    weak var parent: Parent?  // Weak reference (optional)
}

// Unowned (use when reference is never nil)
class Child {
    unowned var parent: Parent  // Unowned reference (non-optional)
    // ⚠️ Crashes if parent is deallocated
}
```

### Closure Capture Lists

```swift
// ❌ Retain cycle
class MyViewController: UIViewController {
    var name = "Test"
    
    func setupClosure() {
        someAsyncOperation { result in
            self.name = result  // Strong capture of self
        }
    }
}

// ✅ Fix with weak
func setupClosure() {
    someAsyncOperation { [weak self] result in
        guard let self = self else { return }
        self.name = result
    }
}

// ✅ Or use unowned (if you're certain self won't be nil)
func setupClosure() {
    someAsyncOperation { [unowned self] result in
        self.name = result  // Crashes if self is deallocated
    }
}

// Complex capture list
func setupClosure() {
    someAsyncOperation { [weak self, weak delegate = self.delegate, strongModel = model] in
        guard let self = self else { return }
        // self is weak
        // delegate is weak
        // strongModel is strong
    }
}
```

### Delegate Pattern Memory Management

```swift
// ✅ Always use weak for delegates
protocol MyDelegate: AnyObject {  // AnyObject = class-only protocol
    func didComplete()
}

class MyClass {
    weak var delegate: MyDelegate?  // Weak to prevent retain cycle
}
```

### Common Retain Cycles

```swift
// 1. Timer retain cycle
class MyViewController: UIViewController {
    var timer: Timer?
    
    func startTimer() {
        // ❌ Timer retains target (self)
        timer = Timer.scheduledTimer(
            timeInterval: 1.0,
            target: self,  // Strong reference!
            selector: #selector(timerFired),
            userInfo: nil,
            repeats: true
        )
    }
    
    deinit {
        timer?.invalidate()  // Won't be called if retained!
    }
}

// ✅ Fix: Use block-based timer (iOS 10+)
func startTimer() {
    timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] timer in
        self?.timerFired()
    }
}

// 2. Notification observer retain cycle (old API)
override func viewDidLoad() {
    super.viewDidLoad()
    
    // Pre-iOS 9: NotificationCenter retained observers
    // ✅ iOS 9+: No longer retains
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(handleNotification),
        name: .someNotification,
        object: nil
    )
}

deinit {
    // Still good practice to remove
    NotificationCenter.default.removeObserver(self)
}

// ✅ Better: Block-based with weak self
var observer: NSObjectProtocol?

override func viewDidLoad() {
    super.viewDidLoad()
    
    observer = NotificationCenter.default.addObserver(
        forName: .someNotification,
        object: nil,
        queue: .main
    ) { [weak self] notification in
        self?.handleNotification(notification)
    }
}

deinit {
    if let observer = observer {
        NotificationCenter.default.removeObserver(observer)
    }
}
```

### Memory Warnings

```swift
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    
    // Clear caches
    imageCache.removeAll()
    
    // Release non-critical resources
    heavyData = nil
}

// Also implement in AppDelegate
func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
    // Clear app-level caches
    URLCache.shared.removeAllCachedResponses()
}
```

---

## Responder Chain & Touch Handling

### The Responder Chain

```
Touch Event
    ↓
UIView (hit-tested view)
    ↓
Superview
    ↓
Superview
    ↓
UIViewController
    ↓
UIWindow
    ↓
UIApplication
    ↓
UIApplicationDelegate
```

### Hit Testing

```swift
// How iOS finds which view was touched
func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    // 1. Check if can receive touches
    guard isUserInteractionEnabled,
          !isHidden,
          alpha > 0.01,
          self.point(inside: point, with: event) else {
        return nil
    }
    
    // 2. Check subviews (reverse order - top to bottom)
    for subview in subviews.reversed() {
        let convertedPoint = convert(point, to: subview)
        if let hitView = subview.hitTest(convertedPoint, with: event) {
            return hitView
        }
    }
    
    // 3. Return self if no subview handles it
    return self
}

// Override to extend touch area
class BiggerTouchButton: UIButton {
    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        // Extend touch area by 20 points in all directions
        let biggerBounds = bounds.insetBy(dx: -20, dy: -20)
        return biggerBounds.contains(point)
    }
}

// Override to pass touches through transparent areas
class PassThroughView: UIView {
    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let hitView = super.hitTest(point, with: event)
        
        // Pass through if the hit view is self
        return hitView == self ? nil : hitView
    }
}
```

### Touch Handling

```swift
class CustomView: UIView {
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let touch = touches.first else { return }
        let location = touch.location(in: self)
        
        // Handle touch start
        print("Touch began at: \(location)")
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let touch = touches.first else { return }
        let location = touch.location(in: self)
        
        // Handle touch move (dragging)
        print("Touch moved to: \(location)")
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let touch = touches.first else { return }
        let location = touch.location(in: self)
        
        // Handle touch end
        print("Touch ended at: \(location)")
    }
    
    override func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?) {
        // Handle cancellation (phone call, notification, etc.)
        print("Touch cancelled")
    }
}
```

### Gesture Recognizers

```swift
// Basic usage
let tap = UITapGestureRecognizer(target: self, action: #selector(handleTap))
view.addGestureRecognizer(tap)

// Gesture recognizer delegate
extension MyViewController: UIGestureRecognizerDelegate {
    // Allow simultaneous recognition
    func gestureRecognizer(
        _ gestureRecognizer: UIGestureRecognizer,
        shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
    ) -> Bool {
        return true  // Allow both pan and pinch at same time
    }
    
    // Control whether gesture should begin
    func gestureRecognizerShouldBegin(_ gestureRecognizer: UIGestureRecognizer) -> Bool {
        // Custom logic
        if gestureRecognizer is UIPanGestureRecognizer {
            let pan = gestureRecognizer as! UIPanGestureRecognizer
            let velocity = pan.velocity(in: view)
            // Only recognize horizontal pans
            return abs(velocity.x) > abs(velocity.y)
        }
        return true
    }
}
```

---

## Animations & CALayer

### UIView Animations

```swift
// Basic animation
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0.5
    self.view.frame.origin.y += 100
}

// With completion
UIView.animate(withDuration: 0.3, animations: {
    self.view.alpha = 0
}) { finished in
    self.view.removeFromSuperview()
}

// With options
UIView.animate(
    withDuration: 0.3,
    delay: 0.1,
    options: [.curveEaseInOut, .allowUserInteraction],
    animations: {
        self.view.transform = CGAffineTransform(scaleX: 1.5, y: 1.5)
    }
)

// Spring animation
UIView.animate(
    withDuration: 0.5,
    delay: 0,
    usingSpringWithDamping: 0.7,  // 0-1, lower = more bounce
    initialSpringVelocity: 0.5,
    options: [],
    animations: {
        self.view.center = targetPoint
    }
)

// Keyframe animations
UIView.animateKeyframes(
    withDuration: 2.0,
    delay: 0,
    options: [],
    animations: {
        UIView.addKeyframe(withRelativeStartTime: 0.0, relativeDuration: 0.25) {
            self.view.alpha = 0.5
        }
        UIView.addKeyframe(withRelativeStartTime: 0.25, relativeDuration: 0.25) {
            self.view.alpha = 1.0
        }
        UIView.addKeyframe(withRelativeStartTime: 0.5, relativeDuration: 0.5) {
            self.view.transform = CGAffineTransform(rotationAngle: .pi)
        }
    }
)
```

### Animatable Properties

```swift
// ✅ Animatable
view.frame
view.bounds
view.center
view.transform
view.alpha
view.backgroundColor

// ❌ Not animatable with UIView.animate
view.layer.cornerRadius
view.layer.shadowOpacity
view.layer.contents (image)

// Use CABasicAnimation for layer properties
```

### CALayer Deep Dive

```swift
// Every UIView has a layer
let view = UIView()
view.layer.cornerRadius = 10
view.layer.masksToBounds = true  // Needed for cornerRadius to clip

// Shadow (don't use masksToBounds with shadow!)
view.layer.shadowColor = UIColor.black.cgColor
view.layer.shadowOpacity = 0.5
view.layer.shadowOffset = CGSize(width: 0, height: 2)
view.layer.shadowRadius = 4

// Border
view.layer.borderWidth = 2
view.layer.borderColor = UIColor.red.cgColor

// Rounding specific corners
let path = UIBezierPath(
    roundedRect: view.bounds,
    byRoundingCorners: [.topLeft, .topRight],
    cornerRadii: CGSize(width: 10, height: 10)
)
let mask = CAShapeLayer()
mask.path = path.cgPath
view.layer.mask = mask
```

### CABasicAnimation

```swift
// Animate layer properties
let animation = CABasicAnimation(keyPath: "cornerRadius")
animation.fromValue = 0
animation.toValue = 20
animation.duration = 0.3
view.layer.add(animation, forKey: "cornerRadius")

// ⚠️ Animation doesn't actually change the property!
view.layer.cornerRadius = 20  // Must set final value

// To make animation persist
animation.fillMode = .forwards
animation.isRemovedOnCompletion = false
// ⚠️ Still doesn't change underlying value - set it manually
```

### Gradient Layers

```swift
let gradientLayer = CAGradientLayer()
gradientLayer.frame = view.bounds
gradientLayer.colors = [
    UIColor.red.cgColor,
    UIColor.blue.cgColor
]
gradientLayer.startPoint = CGPoint(x: 0, y: 0)
gradientLayer.endPoint = CGPoint(x: 1, y: 1)
view.layer.insertSublayer(gradientLayer, at: 0)

// ⚠️ Update frame in layoutSubviews
override func layoutSubviews() {
    super.layoutSubviews()
    gradientLayer.frame = bounds
}
```

---

## UIKit → SwiftUI Bridge

### UIViewRepresentable

```swift
// Wrap UIKit view for use in SwiftUI
struct ActivityIndicator: UIViewRepresentable {
    @Binding var isAnimating: Bool
    let style: UIActivityIndicatorView.Style
    
    func makeUIView(context: Context) -> UIActivityIndicatorView {
        let indicator = UIActivityIndicatorView(style: style)
        return indicator
    }
    
    func updateUIView(_ uiView: UIActivityIndicatorView, context: Context) {
        if isAnimating {
            uiView.startAnimating()
        } else {
            uiView.stopAnimating()
        }
    }
}

// Usage in SwiftUI
struct ContentView: View {
    @State private var isLoading = true
    
    var body: some View {
        ActivityIndicator(isAnimating: $isLoading, style: .large)
    }
}
```

### UIViewControllerRepresentable

```swift
// Wrap UIKit view controller for SwiftUI
struct ImagePicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.presentationMode) var presentationMode
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {
        // Update if needed
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePicker
        
        init(_ parent: ImagePicker) {
            self.parent = parent
        }
        
        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.image = image
            }
            parent.presentationMode.wrappedValue.dismiss()
        }
    }
}
```

### Hosting SwiftUI in UIKit

```swift
// Embed SwiftUI view in UIKit
import SwiftUI

class MyViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Create SwiftUI view
        let swiftUIView = MySwiftUIView()
        
        // Wrap in hosting controller
        let hostingController = UIHostingController(rootView: swiftUIView)
        
        // Add as child
        addChild(hostingController)
        view.addSubview(hostingController.view)
        
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
        
        hostingController.didMove(toParent: self)
    }
}
```

---

## Common Gotchas & Interview Questions

### Q: "Why does my view controller not deallocate?"

**Common causes:**
1. Retain cycle with closure
2. Timer with target: self
3. Notification observer (pre-iOS 9)
4. Delegate not marked weak
5. Still in navigation stack

```swift
// Debug deallocation
deinit {
    print("MyViewController deallocated")
}
```

### Q: "Why is my collection view cell showing wrong data?"

**Cause:** Cell reuse not handled properly

```swift
// ✅ Always reset in prepareForReuse
override func prepareForReuse() {
    super.prepareForReuse()
    imageView.image = nil
    titleLabel.text = nil
}
```

### Q: "Why are my Auto Layout animations not working?"

```swift
// ❌ Wrong
UIView.animate(withDuration: 0.3) {
    self.constraint.constant = 100
}

// ✅ Correct
self.constraint.constant = 100
UIView.animate(withDuration: 0.3) {
    self.view.layoutIfNeeded()
}
```

### Q: "What's the difference between frame and bounds?"

- **frame**: Position and size in superview's coordinate system
- **bounds**: Position (usually 0,0) and size in own coordinate system

```swift
let view = UIView(frame: CGRect(x: 50, y: 50, width: 100, height: 100))
// frame = (50, 50, 100, 100)
// bounds = (0, 0, 100, 100)

view.bounds.origin = CGPoint(x: 10, y: 10)
// frame = (50, 50, 100, 100) - unchanged
// bounds = (10, 10, 100, 100)
// Content shifts, but view position doesn't change
```

### Q: "When should you use weak vs unowned?"

- **weak**: Reference might become nil (optional)
- **unowned**: Reference will never be nil (crashes if wrong)

```swift
// Use weak when unsure
class MyViewController {
    weak var delegate: MyDelegate?
}

// Use unowned for guaranteed lifetime relationships
class Child {
    unowned let parent: Parent  // Child can't exist without parent
    
    init(parent: Parent) {
        self.parent = parent
    }
}
```

### Q: "What's the difference between layoutIfNeeded and setNeedsLayout?"

- **setNeedsLayout()**: Marks view as needing layout, happens on next update cycle (async)
- **layoutIfNeeded()**: Forces immediate layout if view is marked as needing layout (sync)

### Q: "How do you prevent retain cycles in NotificationCenter?"

```swift
// Modern API (iOS 9+) - doesn't retain observer
NotificationCenter.default.addObserver(
    self,
    selector: #selector(handleNotification),
    name: .myNotification,
    object: nil
)

// Block-based - use weak self
observer = NotificationCenter.default.addObserver(
    forName: .myNotification,
    object: nil,
    queue: .main
) { [weak self] notification in
    self?.handle(notification)
}
```

### Q: "What's the difference between viewWillLayoutSubviews and layoutSubviews?"

- **viewWillLayoutSubviews**: View controller method, called before layout
- **layoutSubviews**: UIView method, called when view actually lays out

---

## Quick Reference

### Must-Know Methods

```swift
// View Controller
viewDidLoad()
viewWillAppear(_:)
viewDidAppear(_:)
viewWillDisappear(_:)
viewDidDisappear(_:)

// View
layoutSubviews()
draw(_:)
hitTest(_:with:)
point(inside:with:)

// Auto Layout
updateConstraints()
setNeedsLayout()
layoutIfNeeded()

// Cell
prepareForReuse()
```

### Performance Checklist

- [ ] Use cell reuse identifiers correctly
- [ ] Implement prepareForReuse()
- [ ] Set estimatedRowHeight
- [ ] Cache expensive calculations
- [ ] Load images asynchronously
- [ ] Use prefetching for collection/table views
- [ ] Profile with Instruments
- [ ] Check for retain cycles with Memory Graph
- [ ] Minimize work in cellForRowAt
- [ ] Use shadow path for shadows (performance)

---

This covers the deep UIKit knowledge expected at a senior level. Focus on the "why" behind each concept - that's what separates senior engineers from mid-level.
