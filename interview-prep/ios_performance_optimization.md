---
layout: page
title: "iOS Performance & Optimization Guide"
subtitle: "Complete Reference for Senior iOS Engineers"
---

# iOS Performance & Optimization Guide
## Complete Reference for Senior iOS Engineers

---

## Table of Contents
1. [Profiling with Instruments](#profiling-with-instruments)
2. [Memory Management & Leaks](#memory-management--leaks)
3. [Scroll Performance Optimization](#scroll-performance-optimization)
4. [Rendering & Drawing Optimization](#rendering--drawing-optimization)
5. [App Launch Time Optimization](#app-launch-time-optimization)
6. [Threading & Concurrency](#threading--concurrency)
7. [Image Optimization](#image-optimization)
8. [Battery & CPU Optimization](#battery--cpu-optimization)
9. [Network Optimization](#network-optimization)
10. [Code-Level Optimizations](#code-level-optimizations)
11. [Lazy Loading & Pagination](#lazy-loading--pagination)
12. [Interview Questions](#interview-questions)

---

## Profiling with Instruments

### Essential Instruments Tools

```
Key Instruments for iOS Performance:
├── Time Profiler      - CPU usage, hot spots
├── Allocations        - Memory allocations, object lifecycle
├── Leaks             - Memory leak detection
├── Zombies           - Deallocated object access
├── Network           - HTTP requests, data transfer
├── Core Animation    - FPS, frame drops
├── Energy Log        - Battery usage
└── File Activity     - Disk I/O operations
```

### Using Time Profiler

```swift
// Find CPU hot spots

// 1. Run your app with Time Profiler
// 2. Perform the slow action
// 3. Stop recording
// 4. Look at "Heaviest Stack Trace"

// Example findings:
// - JSONDecoder taking 800ms (do on background thread)
// - Image resizing taking 300ms (cache resized images)
// - viewDidLayoutSubviews called 100x (constraint issue)

// Before optimization
func processData() {
    let json = loadLargeJSON()  // 1.2s
    let parsed = parse(json)     // 0.8s
    updateUI(parsed)             // 0.3s
    // Total: 2.3s on main thread ❌
}

// After optimization
func processData() {
    Task.detached {
        let json = await loadLargeJSON()     // Background
        let parsed = await parse(json)        // Background
        
        await MainActor.run {
            updateUI(parsed)                   // Main thread only
        }
    }
    // Main thread blocked: 0.3s ✅
}
```

### Using Allocations

```swift
// Find memory issues

// Red flags in Allocations:
// 1. "Persistent" memory growing continuously (leak)
// 2. Large spikes in "Transient" memory (optimization opportunity)
// 3. Same object type being allocated thousands of times (reuse opportunity)

// Example: Finding cell reuse issue
// Allocations shows 1000 MyCustomCell instances
// But only 10 visible on screen
// → Cells aren't being reused properly ❌

// Before
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = MyCustomCell()  // New instance every time ❌
    return cell
}

// After
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "MyCell", for: indexPath)
    return cell  // ✅ Reused
}
```

### Using Leaks Instrument

```swift
// Leaks shows retain cycles

// Common leak pattern found:
class ViewController: UIViewController {
    var timer: Timer?
    
    func startTimer() {
        // ❌ Leak: Timer retains self strongly
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            self.updateUI()  // Strong reference to self
        }
    }
    
    deinit {
        print("Dealloc")  // Never called ❌
    }
}

// Fixed version:
class ViewController: UIViewController {
    var timer: Timer?
    
    func startTimer() {
        // ✅ Use weak self
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            self?.updateUI()
        }
    }
    
    deinit {
        timer?.invalidate()
        print("Dealloc")  // Now called ✅
    }
}
```

### Core Animation Instrument

```swift
// Measure FPS and identify dropped frames

// Target: 60 FPS (16.67ms per frame)
// Warning: 50-59 FPS (noticeable stuttering)
// Critical: <50 FPS (obviously janky)

// Common issues found:
// 1. "Color Blended Layers" - Too much alpha blending
// 2. "Color Offscreen-Rendered" - Shadow/mask performance hit
// 3. "Color Misaligned Images" - Image scaling on GPU

// Before (drops to 45 FPS)
view.layer.shadowOpacity = 0.5
view.layer.shadowRadius = 10
// ❌ Shadow calculated every frame

// After (60 FPS)
view.layer.shadowOpacity = 0.5
view.layer.shadowRadius = 10
view.layer.shadowPath = UIBezierPath(rect: view.bounds).cgPath
// ✅ Shadow path pre-calculated
```

### Debug View Hierarchy

```swift
// Xcode Debug View Hierarchy (Cmd + Shift + 6)

// Issues you can find:
// 1. Hidden views still in hierarchy
// 2. Overdraw (too many layers)
// 3. Misaligned views
// 4. Constraint conflicts

// Example: Found 50 invisible views in hierarchy
// Solution:
func removeInvisibleViews() {
    for subview in view.subviews where subview.isHidden {
        subview.removeFromSuperview()  // Remove instead of hide
    }
}
```

---

## Memory Management & Leaks

### Finding Retain Cycles

```swift
// Use Memory Graph Debugger (Debug → Memory Graph)
// Purple ! icon indicates potential leak

// Common Pattern 1: Closure Capture
class DataManager {
    var completionHandler: (() -> Void)?
    
    func startOperation() {
        // ❌ Self retains completionHandler, handler captures self
        completionHandler = {
            self.finishOperation()
        }
    }
    
    // ✅ Fixed
    func startOperationFixed() {
        completionHandler = { [weak self] in
            self?.finishOperation()
        }
    }
}

// Common Pattern 2: Delegate Cycle
protocol MyDelegate: AnyObject {
    func didComplete()
}

class Manager {
    weak var delegate: MyDelegate?  // ✅ Always weak
}

// Common Pattern 3: Parent-Child
class Parent {
    var child: Child?  // Strong ✅
}

class Child {
    weak var parent: Parent?  // Weak ✅
}

// Common Pattern 4: Notification Observers (Pre-iOS 9)
class MyViewController: UIViewController {
    var observer: NSObjectProtocol?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // iOS 9+ doesn't retain, but good practice to clean up
        observer = NotificationCenter.default.addObserver(
            forName: .someNotification,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            self?.handleNotification()
        }
    }
    
    deinit {
        if let observer = observer {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

// Common Pattern 5: Timer Cycle
class MyViewController: UIViewController {
    var timer: Timer?
    
    // ❌ Timer retains target
    func startTimer() {
        timer = Timer.scheduledTimer(
            timeInterval: 1.0,
            target: self,
            selector: #selector(tick),
            userInfo: nil,
            repeats: true
        )
    }
    
    // ✅ Fixed with block-based API
    func startTimerFixed() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            self?.tick()
        }
    }
    
    deinit {
        timer?.invalidate()  // Must invalidate!
    }
    
    @objc func tick() {
        print("Tick")
    }
}
```

### Memory Warnings

```swift
class MyViewController: UIViewController {
    private var imageCache: [String: UIImage] = [:]
    private var dataCache: [String: Data] = [:]
    
    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        
        print("⚠️ Memory warning received")
        
        // Clear caches
        imageCache.removeAll()
        dataCache.removeAll()
        
        // Clear image views not on screen
        for cell in tableView.visibleCells {
            // Keep visible
        }
        
        // Notify user if needed
        showLowMemoryAlert()
    }
}

// App-level memory warnings
class AppDelegate: UIApplicationDelegate {
    func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
        // Clear app-level caches
        URLCache.shared.removeAllCachedResponses()
        ImageCache.shared.clearCache()
        
        // Post notification for view controllers to clean up
        NotificationCenter.default.post(name: .memoryWarning, object: nil)
    }
}
```

### Memory Footprint Optimization

```swift
// Reduce memory usage

// ❌ Loading full image into memory
func loadImage(named: String) -> UIImage? {
    return UIImage(named: named)  // Full resolution in memory
}

// ✅ Downsample large images
func downsampleImage(at url: URL, to size: CGSize) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else {
        return nil
    }
    
    let maxDimensionInPixels = max(size.width, size.height) * UIScreen.main.scale
    
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary
    
    guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
        return nil
    }
    
    return UIImage(cgImage: downsampledImage)
}

// Usage
let thumbnail = downsampleImage(at: url, to: CGSize(width: 200, height: 200))
// Uses ~10x less memory than full image ✅
```

### Autoreleasepool

```swift
// Use autoreleasepool for large batch operations

// ❌ Memory spikes
func processManyImages() {
    for i in 0..<10000 {
        let image = generateImage(index: i)
        process(image)
        // Memory keeps growing until end of scope
    }
}

// ✅ Release memory incrementally
func processManyImagesOptimized() {
    for i in 0..<10000 {
        autoreleasepool {
            let image = generateImage(index: i)
            process(image)
            // Memory released after each iteration
        }
    }
}
```

---

## Scroll Performance Optimization

### UITableView/UICollectionView Best Practices

```swift
// Golden Rules for Smooth Scrolling:
// 1. Keep cellForRowAt fast (<16ms for 60fps)
// 2. Use cell reuse
// 3. Avoid layout calculations in cellForRowAt
// 4. Load images asynchronously
// 5. Cache heights

class OptimizedViewController: UIViewController {
    
    // 1. Register cells properly
    override func viewDidLoad() {
        super.viewDidLoad()
        
        tableView.register(MyCell.self, forCellReuseIdentifier: "MyCell")
        
        // Enable self-sizing cells
        tableView.rowHeight = UITableView.automaticDimension
        tableView.estimatedRowHeight = 100  // ✅ Important for performance
        
        // Prefetching
        tableView.prefetchDataSource = self
    }
    
    // 2. Fast cell configuration
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "MyCell", for: indexPath) as! MyCell
        let item = items[indexPath.row]
        
        // ✅ Fast operations only
        cell.titleLabel.text = item.title
        cell.subtitleLabel.text = item.subtitle
        
        // ❌ Don't do this in cellForRowAt:
        // - Network requests
        // - Heavy computations
        // - Image processing
        // - Layout calculations
        
        // ✅ Load image asynchronously
        cell.loadImage(from: item.imageURL)
        
        return cell
    }
}

// 3. Proper cell reuse
class MyCell: UITableViewCell {
    private var imageLoadTask: Task<Void, Never>?
    
    func loadImage(from url: URL) {
        // Cancel previous load
        imageLoadTask?.cancel()
        
        // Reset
        imageView?.image = placeholderImage
        
        imageLoadTask = Task {
            if let image = try? await ImageLoader.shared.loadImage(from: url) {
                guard !Task.isCancelled else { return }
                
                await MainActor.run {
                    self.imageView?.image = image
                }
            }
        }
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        
        // ✅ CRITICAL: Clean up
        imageLoadTask?.cancel()
        imageView?.image = nil
        titleLabel.text = nil
        subtitleLabel.text = nil
    }
}

// 4. Height caching
extension OptimizedViewController {
    private var heightCache: [IndexPath: CGFloat] = [:]
    
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        // Return cached if available
        if let height = heightCache[indexPath] {
            return height
        }
        
        // Calculate and cache
        let height = calculateHeight(for: indexPath)
        heightCache[indexPath] = height
        return height
    }
    
    func invalidateHeightCache() {
        heightCache.removeAll()
    }
}

// 5. Prefetching
extension OptimizedViewController: UITableViewDataSourcePrefetching {
    func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
        // Start loading data/images for cells about to appear
        for indexPath in indexPaths {
            let item = items[indexPath.row]
            ImageLoader.shared.prefetch(url: item.imageURL)
        }
    }
    
    func tableView(_ tableView: UITableView, cancelPrefetchingForRowsAt indexPaths: [IndexPath]) {
        // Cancel prefetch if user scrolled away
        for indexPath in indexPaths {
            let item = items[indexPath.row]
            ImageLoader.shared.cancelPrefetch(url: item.imageURL)
        }
    }
}
```

### Optimizing Cell Layout

```swift
// ❌ Expensive layout in cellForRowAt
class SlowCell: UITableViewCell {
    override func layoutSubviews() {
        super.layoutSubviews()
        
        // Recalculating on every layout ❌
        let size = calculateComplexLayout()
        imageView?.frame = CGRect(x: 0, y: 0, width: size.width, height: size.height)
    }
}

// ✅ Use Auto Layout properly
class FastCell: UITableViewCell {
    private var didSetupConstraints = false
    
    override func updateConstraints() {
        if !didSetupConstraints {
            // Setup constraints once ✅
            NSLayoutConstraint.activate([
                imageView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
                imageView.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 8),
                imageView.widthAnchor.constraint(equalToConstant: 60),
                imageView.heightAnchor.constraint(equalToConstant: 60)
            ])
            didSetupConstraints = true
        }
        super.updateConstraints()
    }
}

// ✅ Or even better: Setup in init
class BetterCell: UITableViewCell {
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupConstraints()  // Once ✅
    }
    
    private func setupConstraints() {
        // All constraint setup here
    }
}
```

### Avoiding Main Thread Blocks

```swift
// ❌ Blocking main thread
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    
    let data = items[indexPath.row]
    
    // Synchronous JSON parsing ❌
    let json = try? JSONDecoder().decode(ComplexModel.self, from: data)
    
    // Synchronous image processing ❌
    let resizedImage = resizeImage(largeImage, to: thumbnailSize)
    
    // Heavy computation ❌
    let result = performExpensiveCalculation(data)
    
    return cell
}

// ✅ Do work on background, update on main
class SmartCell: UITableViewCell {
    func configure(with data: Data) {
        Task.detached {
            // Background thread ✅
            let json = try? JSONDecoder().decode(ComplexModel.self, from: data)
            let processed = processData(json)
            
            await MainActor.run {
                // Main thread for UI ✅
                self.updateUI(with: processed)
            }
        }
    }
}
```

---

## Rendering & Drawing Optimization

### Layer Optimization

```swift
class OptimizedView: UIView {
    override func layoutSubviews() {
        super.layoutSubviews()
        
        // ❌ Expensive operations
        layer.shadowOpacity = 0.5
        layer.shadowRadius = 10
        // Shadow calculated every frame, kills performance
        
        // ✅ Pre-calculate shadow path
        layer.shadowPath = UIBezierPath(roundedRect: bounds, cornerRadius: 8).cgPath
        // Shadow rendered once, massive performance gain
    }
}

// Rounded corners optimization
class RoundedView: UIView {
    // ❌ Slow approach
    func makeRoundedSlow() {
        layer.cornerRadius = 8
        layer.masksToBounds = true
        // Triggers offscreen rendering
    }
    
    // ✅ Fast approach for simple views
    func makeRoundedFast() {
        layer.cornerRadius = 8
        clipsToBounds = true
        
        // Or even better: use corner curve
        layer.cornerCurve = .continuous
    }
    
    // ✅ Best for complex rounded shapes
    func makeRoundedOptimal() {
        let path = UIBezierPath(roundedRect: bounds, cornerRadius: 8)
        let mask = CAShapeLayer()
        mask.path = path.cgPath
        layer.mask = mask
    }
}

// Opacity optimization
class TransparentView: UIView {
    // ❌ Causes blending
    override init(frame: CGRect) {
        super.init(frame: frame)
        backgroundColor = UIColor.black.withAlphaComponent(0.5)
        // Every subview blended with this alpha
    }
    
    // ✅ Avoid transparency when possible
    func optimized() {
        backgroundColor = .black
        alpha = 1.0  // No blending
        isOpaque = true  // Tell system we're opaque
    }
}
```

### Reducing Overdraw

```swift
// Overdraw = Drawing multiple layers on top of each other

// ❌ High overdraw
class OverdrawnView: UIView {
    override func draw(_ rect: CGRect) {
        // Drawing full background
        UIColor.white.setFill()
        UIBezierPath(rect: bounds).fill()
        
        // Drawing another full background
        UIColor.blue.setFill()
        UIBezierPath(rect: bounds).fill()
        
        // Both get drawn, waste of GPU ❌
    }
}

// ✅ Minimal overdraw
class OptimizedView: UIView {
    override func draw(_ rect: CGRect) {
        // Only draw what's needed
        UIColor.blue.setFill()
        UIBezierPath(rect: bounds).fill()
        // Single draw call ✅
    }
}

// ✅ Use opaque property
view.isOpaque = true  // System knows not to draw behind
view.backgroundColor = .white  // Solid color

// ✅ Remove hidden views from hierarchy
func cleanupHiddenViews() {
    for subview in view.subviews where subview.isHidden {
        subview.removeFromSuperview()
    }
}
```

### Rasterization

```swift
// For complex layer hierarchies that don't change

class ComplexView: UIView {
    func setupComplexLayers() {
        // Multiple sublayers, shadows, masks, etc.
        
        // ✅ Rasterize to bitmap
        layer.shouldRasterize = true
        layer.rasterizationScale = UIScreen.main.scale
        
        // Good for:
        // - Static content
        // - Complex layer trees
        // - Content that animates as a whole
        
        // Bad for:
        // - Frequently changing content
        // - Different scale factors
    }
}
```

### Avoiding Expensive Blending

```swift
// Debug blending with Simulator → Debug → Color Blended Layers

// Red = Blending occurring (bad)
// Green = No blending (good)

// ❌ Causes blending
let label = UILabel()
label.backgroundColor = .clear  // Transparent
label.text = "Hello"
// Text blends with background ❌

// ✅ Avoid blending
let optimizedLabel = UILabel()
optimizedLabel.backgroundColor = .white  // Opaque
optimizedLabel.isOpaque = true
optimizedLabel.text = "Hello"
// No blending needed ✅

// Images with alpha
// ❌ PNG with alpha channel
imageView.image = UIImage(named: "logo.png")  // Has transparency

// ✅ JPEG without alpha (if possible)
imageView.image = UIImage(named: "logo.jpg")  // No alpha channel

// ✅ Or tell system image is opaque
imageView.isOpaque = true
imageView.backgroundColor = .white
```

---

## App Launch Time Optimization

### Launch Time Breakdown

```
Cold Launch Timeline:
├── 0-400ms:   System initialization, dyld loading
├── 400-600ms: UIApplicationMain, App Delegate
├── 600-800ms: Scene/Window setup
├── 800-1000ms: First view controller
└── 1000ms+:   First meaningful content

Target: < 400ms for optimal UX
Warning: > 1000ms feels slow
Critical: > 2000ms users may leave
```

### Optimizing didFinishLaunching

```swift
// ❌ Slow app launch
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    // Synchronous network call ❌
    let config = fetchConfiguration()
    
    // Heavy SDK initialization ❌
    Analytics.shared.initialize()
    CrashReporter.shared.setup()
    ThirdPartySDK.configure()
    
    // Complex UI setup ❌
    setupComplexRootViewController()
    
    // Database migration ❌
    Database.shared.migrate()
    
    return true
    // Total: 3000ms ❌
}

// ✅ Fast app launch
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    // Only critical initialization
    setupWindow()
    
    // Defer non-critical work
    DispatchQueue.main.async {
        self.initializeSDKs()
    }
    
    Task.detached {
        await self.fetchConfiguration()
    }
    
    return true
    // Total: 200ms ✅
}

private func initializeSDKs() {
    // Initialize one at a time
    Analytics.shared.initialize()
    
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
        CrashReporter.shared.setup()
    }
    
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
        ThirdPartySDK.configure()
    }
}
```

### Lazy Initialization

```swift
// ❌ Eager initialization
class AppDelegate: UIApplicationDelegate {
    let analytics = Analytics()           // Created at app launch
    let database = Database()             // Created at app launch
    let networkManager = NetworkManager() // Created at app launch
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // All initialized before this runs ❌
        return true
    }
}

// ✅ Lazy initialization
class AppDelegate: UIApplicationDelegate {
    lazy var analytics = Analytics()           // Created when first accessed
    lazy var database = Database()             // Created when first accessed
    lazy var networkManager = NetworkManager() // Created when first accessed
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Nothing initialized yet ✅
        return true
    }
}
```

### Measuring Launch Time

```swift
// Add to AppDelegate
private var launchStartTime: CFAbsoluteTime = 0

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    launchStartTime = CFAbsoluteTimeGetCurrent()
    
    // Your initialization...
    
    return true
}

func applicationDidBecomeActive(_ application: UIApplication) {
    let launchTime = CFAbsoluteTimeGetCurrent() - launchStartTime
    print("🚀 Launch time: \(launchTime * 1000)ms")
    
    // Log to analytics
    Analytics.log(event: "app_launch_time", value: launchTime)
}
```

### Reducing Framework Loading

```swift
// Check what frameworks are loaded:
// Product → Scheme → Edit Scheme → Run → Arguments
// Add: DYLD_PRINT_STATISTICS

// Results show:
// - How many dylibs loaded
// - Time spent loading
// - Time spent binding

// Optimization strategies:
// 1. Avoid unnecessary frameworks
// 2. Use static linking when possible
// 3. Merge frameworks
// 4. Lazy load frameworks

// Lazy framework loading
class FeatureManager {
    private var heavyFramework: HeavyFramework?
    
    func useHeavyFeature() {
        if heavyFramework == nil {
            // Load framework only when needed
            heavyFramework = HeavyFramework()
        }
        heavyFramework?.doWork()
    }
}
```

---

## Threading & Concurrency

### GCD (Grand Central Dispatch)

```swift
// Background work
DispatchQueue.global(qos: .userInitiated).async {
    // Heavy computation
    let result = self.processData()
    
    DispatchQueue.main.async {
        // Update UI
        self.updateUI(with: result)
    }
}

// Quality of Service levels (priority)
DispatchQueue.global(qos: .userInteractive).async {
    // Highest priority - user waiting
    // Example: Animation calculations
}

DispatchQueue.global(qos: .userInitiated).async {
    // High priority - user initiated
    // Example: Loading data user requested
}

DispatchQueue.global(qos: .utility).async {
    // Medium priority - long running
    // Example: Downloading files
}

DispatchQueue.global(qos: .background).async {
    // Lowest priority - not user visible
    // Example: Syncing data, cleanup
}
```

### async/await Best Practices

```swift
// ❌ Blocking main thread
func loadData() {
    Task {
        let data = try await fetchData()  // Main thread blocked ❌
        updateUI(data)
    }
}

// ✅ Don't block main thread
func loadDataOptimized() {
    Task.detached {
        let data = try await fetchData()  // Background ✅
        
        await MainActor.run {
            updateUI(data)  // Main thread ✅
        }
    }
}

// ✅ Mark ViewModel methods as @MainActor
@MainActor
class ViewModel {
    @Published var data: [Item] = []
    
    func loadData() async {
        // Automatically on main thread
        let items = try? await repository.fetchItems()
        self.data = items ?? []
    }
}
```

### Avoiding Race Conditions

```swift
// ❌ Race condition
class Counter {
    var count = 0
    
    func increment() {
        DispatchQueue.global().async {
            self.count += 1  // ❌ Not thread-safe
        }
    }
}

// ✅ Serial queue for synchronization
class SafeCounter {
    private var count = 0
    private let queue = DispatchQueue(label: "com.app.counter")
    
    func increment() {
        queue.async {
            self.count += 1  // ✅ Thread-safe
        }
    }
    
    func getCount() -> Int {
        return queue.sync {
            return count
        }
    }
}

// ✅ Actor for thread-safety (modern approach)
actor ActorCounter {
    private var count = 0
    
    func increment() {
        count += 1  // ✅ Thread-safe automatically
    }
    
    func getCount() -> Int {
        return count
    }
}

// Usage
let counter = ActorCounter()
Task {
    await counter.increment()
    let value = await counter.getCount()
}
```

### OperationQueue

```swift
// For complex dependencies and cancellation

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 3  // Limit concurrent ops

// Add operations
let op1 = BlockOperation {
    // Do work
}

let op2 = BlockOperation {
    // Do work
}

// Dependencies
op2.addDependency(op1)  // op2 waits for op1

// Add to queue
queue.addOperation(op1)
queue.addOperation(op2)

// Cancel all
queue.cancelAllOperations()

// Wait for completion
queue.waitUntilAllOperationsAreFinished()
```

---

## Image Optimization

### Downsampling Large Images

```swift
// ❌ Loading full resolution (1000x1000 image)
let image = UIImage(named: "large-photo")  // 4MB in memory
imageView.image = image
// ImageView is 100x100, but full image in memory ❌

// ✅ Downsample to required size
func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat = UIScreen.main.scale) -> UIImage? {
    
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else {
        return nil
    }
    
    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary
    
    guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
        return nil
    }
    
    return UIImage(cgImage: downsampledImage)
}

// Usage
let thumbnail = downsample(
    imageAt: imageURL,
    to: CGSize(width: 100, height: 100)
)
// Only ~40KB in memory instead of 4MB ✅
```

### Image Decoding

```swift
// Images are decoded lazily (when displayed)
// Decoding can cause frame drops

// ❌ Decode on main thread
imageView.image = UIImage(named: "large")  // Decoded when displayed ❌

// ✅ Pre-decode on background thread
func prepareImage(_ image: UIImage) -> UIImage? {
    guard let cgImage = image.cgImage else { return nil }
    
    let colorSpace = CGColorSpaceCreateDeviceRGB()
    let bitmapInfo = CGImageAlphaInfo.premultipliedFirst.rawValue
    
    guard let context = CGContext(
        data: nil,
        width: cgImage.width,
        height: cgImage.height,
        bitsPerComponent: 8,
        bytesPerRow: 0,
        space: colorSpace,
        bitmapInfo: bitmapInfo
    ) else {
        return nil
    }
    
    let rect = CGRect(x: 0, y: 0, width: cgImage.width, height: cgImage.height)
    context.draw(cgImage, in: rect)
    
    guard let decodedCGImage = context.makeImage() else {
        return nil
    }
    
    return UIImage(cgImage: decodedCGImage, scale: image.scale, orientation: image.imageOrientation)
}

// Usage
Task.detached {
    let image = UIImage(named: "large")
    let decoded = prepareImage(image)  // Decode on background
    
    await MainActor.run {
        imageView.image = decoded  // No decoding needed ✅
    }
}
```

### Image Format Selection

```swift
// JPEG vs PNG vs HEIC

// JPEG:
// - Lossy compression
// - No transparency
// - Smaller file size
// - Best for photos

// PNG:
// - Lossless compression
// - Supports transparency
// - Larger file size
// - Best for graphics, icons

// HEIC (iOS 11+):
// - Better compression than JPEG
// - Supports transparency
// - Smaller file size
// - Best overall choice

// Convert to HEIC
func saveAsHEIC(image: UIImage, quality: CGFloat = 0.8) -> Data? {
    return image.heicData(compressionQuality: quality)
}

extension UIImage {
    func heicData(compressionQuality: CGFloat) -> Data? {
        guard let mutableData = CFDataCreateMutable(nil, 0),
              let destination = CGImageDestinationCreateWithData(
                mutableData,
                "public.heic" as CFString,
                1,
                nil
              ),
              let cgImage = self.cgImage else {
            return nil
        }
        
        let options = [kCGImageDestinationLossyCompressionQuality: compressionQuality] as CFDictionary
        CGImageDestinationAddImage(destination, cgImage, options)
        CGImageDestinationFinalize(destination)
        
        return mutableData as Data
    }
}
```

---

## Battery & CPU Optimization

### Reducing CPU Usage

```swift
// Check CPU usage in Xcode → Debug Navigator → CPU

// ❌ High CPU usage
class AnimationViewController: UIViewController {
    private var displayLink: CADisplayLink?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        displayLink = CADisplayLink(target: self, selector: #selector(update))
        displayLink?.add(to: .main, forMode: .common)
        // Runs 60 times per second, even when not animating ❌
    }
    
    @objc func update() {
        // Update animation
    }
}

// ✅ Stop when not needed
class OptimizedAnimationViewController: UIViewController {
    private var displayLink: CADisplayLink?
    
    func startAnimation() {
        displayLink = CADisplayLink(target: self, selector: #selector(update))
        displayLink?.add(to: .main, forMode: .common)
    }
    
    func stopAnimation() {
        displayLink?.invalidate()
        displayLink = nil
    }
    
    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        stopAnimation()  // ✅ Stop when not visible
    }
}
```

### Location Services Optimization

```swift
import CoreLocation

class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    
    // ❌ Battery drain
    func startTrackingContinuously() {
        manager.desiredAccuracy = kCLLocationAccuracyBest  // Most power
        manager.allowsBackgroundLocationUpdates = true
        manager.startUpdatingLocation()
        // Constant GPS usage ❌
    }
    
    // ✅ Optimized
    func startTrackingOptimized() {
        // Use lowest accuracy needed
        manager.desiredAccuracy = kCLLocationAccuracyHundredMeters
        
        // Use significant location changes (cell tower)
        manager.startMonitoringSignificantLocationChanges()
        // Only updates on major location changes ✅
        
        // Or defer updates
        manager.allowDeferredLocationUpdates(
            untilTraveled: 1000,  // 1km
            timeout: 300           // 5 minutes
        )
    }
    
    func stopTracking() {
        manager.stopUpdatingLocation()
        manager.stopMonitoringSignificantLocationChanges()
    }
}
```

### Background Tasks

```swift
import BackgroundTasks

// Register background tasks
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    // Background fetch (periodic)
    BGTaskScheduler.shared.register(
        forTaskWithIdentifier: "com.app.refresh",
        using: nil
    ) { task in
        self.handleAppRefresh(task: task as! BGAppRefreshTask)
    }
    
    return true
}

func handleAppRefresh(task: BGAppRefreshTask) {
    // Schedule next refresh
    scheduleAppRefresh()
    
    let queue = OperationQueue()
    queue.maxConcurrentOperationCount = 1
    
    // Do work
    task.expirationHandler = {
        queue.cancelAllOperations()
    }
    
    let operation = BlockOperation {
        // Sync data, cleanup, etc.
        // Keep under 30 seconds ✅
    }
    
    operation.completionBlock = {
        task.setTaskCompleted(success: !operation.isCancelled)
    }
    
    queue.addOperation(operation)
}

func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)  // 15 min
    
    try? BGTaskScheduler.shared.submit(request)
}
```

---

## Network Optimization

### Request Batching

```swift
// ❌ Many small requests
func loadUserData() async {
    let profile = try? await fetchProfile()       // Request 1
    let settings = try? await fetchSettings()     // Request 2
    let friends = try? await fetchFriends()       // Request 3
    let posts = try? await fetchPosts()           // Request 4
    // 4 round trips ❌
}

// ✅ Single batched request
func loadUserDataOptimized() async {
    let userData = try? await fetchAllUserData()  // Request 1
    // 1 round trip ✅
}

struct UserDataResponse: Codable {
    let profile: Profile
    let settings: Settings
    let friends: [Friend]
    let posts: [Post]
}
```

### Response Compression

```swift
// Enable gzip compression
var request = URLRequest(url: url)
request.setValue("gzip, deflate", forHTTPHeaderField: "Accept-Encoding")

// URLSession automatically decompresses
```

### Caching Strategy

```swift
// Cache policy selection
var request = URLRequest(url: url)

// Always fetch from network
request.cachePolicy = .reloadIgnoringLocalCacheData

// Use cache if available, else network (default)
request.cachePolicy = .returnCacheDataElseLoad

// Only use cache (fail if not cached)
request.cachePolicy = .returnCacheDataDontLoad

// Respect HTTP cache headers
request.cachePolicy = .useProtocolCachePolicy  // Default ✅
```

### HTTP/2 & HTTP/3

```swift
// URLSession automatically uses HTTP/2 when available
// No code changes needed

// Benefits:
// - Multiplexing (multiple requests over one connection)
// - Header compression
// - Server push
// - Lower latency

// iOS 15+: HTTP/3 support (QUIC)
// Automatically negotiated
```

---

## Code-Level Optimizations

### String Concatenation

```swift
// ❌ Slow
var result = ""
for i in 0..<1000 {
    result += "Item \(i)\n"  // Creates new string each time
}

// ✅ Fast
var result = ""
result.reserveCapacity(10000)  // Pre-allocate
for i in 0..<1000 {
    result += "Item \(i)\n"
}

// ✅ Fastest
let result = (0..<1000).map { "Item \($0)" }.joined(separator: "\n")
```

### Array Operations

```swift
// ❌ Slow iteration
for i in 0..<array.count {
    let item = array[i]  // Bounds checking each time
    process(item)
}

// ✅ Faster
for item in array {
    process(item)  // Optimized by compiler
}

// ❌ Slow append in loop
var result: [Int] = []
for i in 0..<10000 {
    result.append(i)  // May need reallocation
}

// ✅ Pre-allocate
var result: [Int] = []
result.reserveCapacity(10000)  // Allocate once
for i in 0..<10000 {
    result.append(i)
}
```

### Struct vs Class Performance

```swift
// Structs are generally faster for small data
struct Point {  // ✅ Stack allocated, no ARC
    let x: Double
    let y: Double
}

class PointClass {  // Heap allocated, ARC overhead
    let x: Double
    let y: Double
    init(x: Double, y: Double) {
        self.x = x
        self.y = y
    }
}

// Benchmark:
// Creating 1M structs: ~50ms
// Creating 1M classes: ~200ms

// But be careful with large structs (copying overhead)
struct LargeStruct {
    var data: [Int] = Array(repeating: 0, count: 10000)
    // Copying this struct is expensive ❌
}
```

---

## Lazy Loading & Pagination

### Infinite Scroll

```swift
class InfiniteScrollViewController: UIViewController, UITableViewDelegate, UITableViewDataSource {
    
    private var items: [Item] = []
    private var currentPage = 0
    private var isLoading = false
    private var hasMorePages = true
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return items.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        cell.textLabel?.text = items[indexPath.row].title
        return cell
    }
    
    func scrollViewDidScroll(_ scrollView: UIScrollView) {
        let offsetY = scrollView.contentOffset.y
        let contentHeight = scrollView.contentSize.height
        let height = scrollView.frame.size.height
        
        // Load more when near bottom
        if offsetY > contentHeight - height - 100 {
            loadMoreIfNeeded()
        }
    }
    
    private func loadMoreIfNeeded() {
        guard !isLoading && hasMorePages else { return }
        
        isLoading = true
        currentPage += 1
        
        Task {
            do {
                let newItems = try await fetchItems(page: currentPage)
                
                if newItems.isEmpty {
                    hasMorePages = false
                } else {
                    items.append(contentsOf: newItems)
                    tableView.reloadData()
                }
                
                isLoading = false
            } catch {
                isLoading = false
                currentPage -= 1
            }
        }
    }
}
```

### Lazy View Loading

```swift
// Don't create all views upfront

// ❌ Create all tabs immediately
class TabBarController: UITabBarController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let vc1 = FirstViewController()
        let vc2 = SecondViewController()
        let vc3 = ThirdViewController()
        let vc4 = FourthViewController()
        
        viewControllers = [vc1, vc2, vc3, vc4]
        // All 4 VCs created even though only 1 visible ❌
    }
}

// ✅ Create tabs lazily
class LazyTabBarController: UITabBarController, UITabBarControllerDelegate {
    private var loadedViewControllers: [Int: UIViewController] = [:]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        delegate = self
        
        // Create placeholders
        viewControllers = [
            PlaceholderViewController(),
            PlaceholderViewController(),
            PlaceholderViewController(),
            PlaceholderViewController()
        ]
    }
    
    func tabBarController(_ tabBarController: UITabBarController, shouldSelect viewController: UIViewController) -> Bool {
        guard let index = tabBarController.viewControllers?.firstIndex(of: viewController) else {
            return true
        }
        
        // Load real VC if not loaded
        if loadedViewControllers[index] == nil {
            let realVC = createViewController(for: index)
            loadedViewControllers[index] = realVC
            tabBarController.viewControllers?[index] = realVC
        }
        
        return true
    }
    
    private func createViewController(for index: Int) -> UIViewController {
        switch index {
        case 0: return FirstViewController()
        case 1: return SecondViewController()
        case 2: return ThirdViewController()
        case 3: return FourthViewController()
        default: return UIViewController()
        }
    }
}
```

---

## Interview Questions

### Q: "How do you identify and fix performance issues?"

**A:** Systematic approach:
1. **Profile with Instruments**: Time Profiler for CPU, Allocations for memory
2. **Identify bottlenecks**: Find hot spots (functions taking most time)
3. **Measure**: Get baseline metrics before optimizing
4. **Optimize**: Make targeted improvements
5. **Verify**: Measure again to confirm improvement
6. **Monitor**: Add analytics to track in production

### Q: "What causes scroll lag and how do you fix it?"

**A:** Common causes:
1. **Heavy cellForRowAt**: Move work to background threads
2. **Image loading**: Load/decode asynchronously
3. **No cell reuse**: Implement proper reuse identifiers
4. **Complex layout**: Simplify or cache calculations
5. **Shadow rendering**: Use shadowPath
6. **Large images**: Downsample to display size

### Q: "How do you optimize app launch time?"

**A:**
1. **Defer initialization**: Only critical setup in didFinishLaunching
2. **Lazy loading**: Create objects when needed, not upfront
3. **Reduce frameworks**: Fewer dylibs to load
4. **Measure**: Use Instruments or DYLD_PRINT_STATISTICS
5. **Async work**: Move non-critical work to background

### Q: "Explain memory leaks and how to find them"

**A:** Memory leak = object not deallocated when no longer needed.

**Finding:**
1. Memory Graph Debugger (purple ! = leak)
2. Leaks instrument
3. Check deinit is called

**Common causes:**
1. Retain cycles in closures (missing [weak self])
2. Timer with target: self
3. Delegate not marked weak
4. Notification observers (pre-iOS 9)

### Q: "What's the difference between weak and unowned?"

**A:**
- **weak**: Optional, becomes nil when deallocated (safe)
- **unowned**: Non-optional, crashes if accessed after deallocation (fast but risky)

Use weak by default, unowned only when you're certain the relationship guarantees the reference won't be nil.

### Q: "How do you optimize battery usage?"

**A:**
1. **Location**: Use lowest accuracy needed, stop when not needed
2. **Networking**: Batch requests, use cache
3. **Background**: Minimize background work
4. **CPU**: Stop animations when off-screen
5. **Display**: Reduce brightness, use dark mode

---

This covers all the performance topics you'll need. Focus on profiling with Instruments and scroll optimization - these are most commonly asked in interviews.
