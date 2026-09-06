# Mobile App Architecture (iOS, Android, React Native & Flutter)

## iOS

### Choosing an App Architecture Pattern (MVC / MVVM / TCA)

- **DO:** Default to MVVM (Model-View-ViewModel) for most SwiftUI apps of small-to-medium size, letting the View bind to an `ObservableObject`/`@Observable` ViewModel that exposes state and intents. MVVM maps naturally onto SwiftUI's declarative, data-driven rendering model and keeps business logic testable without needing a view controller or live UI to exercise it.
```swift
@Observable
final class ProfileViewModel {
    private(set) var user: User?
    private(set) var isLoading = false

    func load(id: String) async {
        isLoading = true
        defer { isLoading = false }
        user = try? await userRepository.fetch(id: id)
    }
}
```
- **DON'T:** Reach for a heavyweight unidirectional-data-flow framework (The Composable Architecture, Redux-style stores, or a hand-rolled equivalent) on a small app or a single-screen prototype just because it's trendy. These frameworks add real ceremony — actions, reducers, effects, dependency injection graphs — that pays off on large, long-lived, multi-contributor codebases with complex cross-screen state, but is pure overhead on a five-screen utility app.
- **DO:** Reserve MVC (Model-View-Controller, i.e. classic `UIViewController`-owns-logic) for UIKit codebases where it's already the established pattern, or for very simple screens where a dedicated ViewModel layer would be pure indirection. UIKit's own APIs (delegate methods, `IBAction`s) are shaped around the controller owning coordination, so fighting that with a forced MVVM layer on every single screen can add boilerplate without a real testability win if the controller has no meaningful logic to extract.
- **DON'T:** Let a `UIViewController` or SwiftUI `View` accumulate networking calls, persistence code, business rules, and formatting logic directly in its body — the so-called "Massive View Controller" anti-pattern. A view/controller that owns twenty responsibilities is nearly impossible to unit test, and every unrelated change risks breaking unrelated UI, because there's no seam between "how it's displayed" and "what it means."
- **DO:** Adopt TCA (The Composable Architecture) or a similar unidirectional-flow framework deliberately, when the app has genuinely complex, deeply nested state that many screens need to observe and mutate consistently (e.g., a shared cart, a multi-step wizard with branching, deep undo/redo). Its strengths — a single source of truth, exhaustive testing of state transitions, explicit effect management — are proportional to how much state-sharing complexity the app actually has.
- **DON'T:** Mix architecture patterns inconsistently screen-by-screen without a documented convention (one screen in MVVM, the next hand-rolling Combine pipelines, a third using TCA) inside the same app without a clear boundary or migration plan. Inconsistent architecture forces every new contributor to relearn the rules per-screen and makes it hard to share reusable pieces (navigation, error handling, loading state) across the app.

### SwiftUI vs. UIKit Decision-Making

- **DO:** Default new iOS apps and new screens in existing apps to SwiftUI when the minimum deployment target and required capabilities support it (SwiftUI has matured significantly since iOS 13, and most CRUD-shaped, form-shaped, and list-shaped screens are faster to build and maintain in SwiftUI's declarative model with far less boilerplate than the UIKit equivalent).
- **DON'T:** Force a UIKit-only capability (fine-grained custom collection view layouts and drag interactions, precise `UIScrollView` delegate control, certain camera/AR compositing pipelines, or deep integration with a legacy UIKit-based SDK) into SwiftUI through awkward workarounds when `UIViewRepresentable`/`UIViewControllerRepresentable` bridging — or simply keeping that one screen in UIKit — is the straightforward, well-supported path. Fighting the framework to avoid a bridge usually produces more fragile code than the bridge itself.
```swift
struct LegacyMapView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> LegacyMapViewController {
        LegacyMapViewController()
    }
    func updateUIViewController(_ vc: LegacyMapViewController, context: Context) {
        vc.refresh()
    }
}
```
- **DO:** Keep a UIKit codebase in UIKit rather than doing a risky full rewrite to SwiftUI purely for its own sake. Incrementally introducing SwiftUI for new screens (via `UIHostingController`) while leaving stable, working UIKit screens alone is almost always lower-risk than a big-bang rewrite, and lets the team learn SwiftUI's idioms gradually.
- **DON'T:** Assume every UIKit API, gesture recognizer, or animation curve has a drop-in SwiftUI equivalent as of any given SDK version. SwiftUI's API surface still has real gaps versus UIKit for certain fine-grained control (precise text selection handling, some `UICollectionViewCompositionalLayout` capabilities, certain low-level `CATransaction` timing control) — verify the specific SwiftUI API actually exists and behaves as expected for the target OS version before committing a screen to it, rather than assuming feature parity.
- **DO:** Interoperate deliberately at the boundary: use `UIHostingController` to embed SwiftUI in a UIKit navigation stack, and `UIViewRepresentable`/`UIViewControllerRepresentable` to embed UIKit views inside SwiftUI, with clear data flow (bindings, closures) crossing that boundary rather than shared mutable global state.

### App Lifecycle and Scene Management

- **DO:** Use the `App`/`Scene`/`WindowGroup` lifecycle (`SwiftUI App` protocol) for new apps, and handle multi-window/multi-scene support explicitly on iPadOS if the app should support it — each `UIScene` (or SwiftUI `Scene`) instance has its own independent lifecycle and state, which matters for Slide Over, Split View, and Stage Manager on iPad.
- **DON'T:** Assume there is exactly one "app-wide" foreground/background state via the old `UIApplicationDelegate` lifecycle callbacks (`applicationDidBecomeActive`, etc.) in a scene-based app — on iPadOS, multiple scenes can be active, backgrounded, or suspended independently, and code that only reacts to app-level delegate callbacks will miss per-scene transitions (e.g. one window backgrounding while another stays foreground).
- **DO:** Persist and restore meaningful UI state (scroll position, selected tab, in-progress form input) across app termination and relaunch, using `NSUserActivity`/state restoration APIs or your own lightweight persisted state, matching what users expect from Apple's platform conventions: an app that reopens exactly where they left it.
- **DON'T:** Perform expensive, long-running work (network calls, heavy computation, disk I/O) directly inside lifecycle callbacks like `scenePhase` changes or `applicationDidBecomeActive` without dispatching it appropriately — these callbacks are on the main thread and expected to return quickly; blocking them delays the system's ability to consider your app fully launched/foregrounded and can produce visible hitches or watchdog termination.
```swift
.onChange(of: scenePhase) { _, newPhase in
    if newPhase == .active {
        Task { await syncService.refreshIfStale() } // don't block here
    }
}
```

### Navigation Patterns

- **DO:** Use `NavigationStack` with a typed, `Hashable`/`Codable` path (`NavigationPath` or a typed enum-based path) for programmatic, deep-linkable navigation in SwiftUI apps targeting iOS 16+. A typed path makes navigation state serializable (for state restoration and deep links) and testable, unlike relying purely on chained `NavigationLink` destinations scattered across views.
```swift
enum Route: Hashable { case profile(id: String), settings }

NavigationStack(path: $router.path) {
    HomeView()
        .navigationDestination(for: Route.self) { route in
            switch route {
            case .profile(let id): ProfileView(id: id)
            case .settings: SettingsView()
            }
        }
}
```
- **DON'T:** Keep using the deprecated `NavigationView` with implicit, view-embedded `NavigationLink(destination:)` for new code on iOS 16+, and don't build ad-hoc navigation state with a tangle of `@State` booleans (`isShowingDetail`, `isShowingSettings`, `isShowingProfile`) driving separate `.sheet`/`.fullScreenCover`/`NavigationLink` modifiers — this doesn't scale past a couple of destinations, can't easily be serialized for deep linking, and tends to produce bugs where two presentations trigger simultaneously.
- **DO:** Centralize navigation logic in a coordinator/router object (whether a plain `ObservableObject` holding a `NavigationPath`, or a full Coordinator pattern in UIKit) rather than letting every screen decide independently how to push, present, or dismiss the next screen. Centralizing navigation makes deep linking, analytics on navigation events, and flow-level testing dramatically simpler.
- **DON'T:** Hardcode navigation transitions that ignore accessibility settings — e.g., a custom push transition that only works with a specific screen size or that breaks under Larger Text/Dynamic Type by clipping content. Verify custom transitions and modal presentations at multiple Dynamic Type sizes and both size classes.
- **DO:** Distinguish sheet vs. full-screen-cover vs. push intentionally, following Apple's Human Interface Guidelines: use a push (`NavigationStack`) for drilling into a hierarchy the user should be able to back out of step by step, a sheet for a self-contained, dismissible task, and a full-screen cover sparingly, for content that needs full attention (e.g., camera, onboarding). Misusing presentation styles (e.g., pushing every screen, including modal-shaped tasks) creates a navigation stack that doesn't match the user's mental model of "where am I and how do I get back."

### Local Persistence: Core Data, SwiftData, and UserDefaults

- **DO:** Use `UserDefaults` only for small, simple, non-sensitive key-value settings (a theme preference, a "has seen onboarding" flag, a last-selected tab) — not as a general-purpose data store. `UserDefaults` is a plist loaded entirely into memory; storing large object graphs, arrays of records, or anything resembling structured relational data in it degrades performance and isn't designed for querying.
```swift
// GOOD: a small preference flag
UserDefaults.standard.set(true, forKey: "hasCompletedOnboarding")

// BAD: don't store a growing array of user records here
UserDefaults.standard.set(try? JSONEncoder().encode(allOrders), forKey: "orders")
```
- **DO:** Use SwiftData (iOS 17+) for new apps' primary structured local persistence when the deployment target allows it — it gives Core Data-equivalent capability (relationships, queries, migrations) with a much smaller, more Swift-native API surface and tighter SwiftUI integration via `@Query` and `@Model`.
- **DON'T:** Introduce SwiftData into a codebase whose minimum deployment target is below iOS 17, or into an app with an existing, working Core Data stack, purely to chase the newer API. SwiftData isn't available pre-iOS 17, and a mid-project migration from a stable Core Data store carries real data-migration risk that should be weighed against the actual benefit, not adopted reflexively.
- **DO:** Use Core Data (still fully supported) for apps that need its more mature ecosystem — `NSFetchedResultsController`-driven UIKit lists, complex migrations with mapping models, or the wider tooling and StackOverflow-era knowledge base — especially in an existing large app already built on it.
- **DON'T:** Perform Core Data fetches, saves, or heavy predicate-based queries on the main thread/main queue context for anything beyond trivial reads. Use a background `NSManagedObjectContext` (or SwiftData's `ModelActor`) for writes and heavy queries, and only touch the main-queue context for what's actually bound to UI, or the UI will stutter under load.
- **DON'T:** Store sensitive data (auth tokens, passwords, personally identifiable financial or health data) in `UserDefaults`, a plain Core Data/SwiftData store, or an unencrypted file — `UserDefaults` and an unprotected SQLite store are not secure storage. See the Keychain guidance under Cross-Cutting Mobile Concerns.

### Background Execution Limits

- **DO:** Understand and design around the specific background execution mechanisms iOS actually offers — background tasks via `BGTaskScheduler` (`BGAppRefreshTask`, `BGProcessingTask`) for deferred, opportunistic work; background URL sessions for large uploads/downloads that should continue after the app is suspended; and the small time window granted by `beginBackgroundTask` for finishing brief, already-in-progress work. Don't assume arbitrary background execution is available — iOS aggressively suspends apps to preserve battery and only wakes them under system-scheduled, budgeted conditions.
```swift
BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.app.refresh", using: nil) { task in
    handleAppRefresh(task: task as! BGAppRefreshTask)
}
```
- **DON'T:** Assume `BGAppRefreshTask` will run at a specific time, or run at all, on any predictable schedule — the system decides when (and whether) to grant background execution based on usage patterns, battery level, and Low Power Mode, and a background refresh task can be starved for days on a rarely-opened app. Design features so they degrade gracefully (refresh on next foreground) rather than depending on background refresh actually firing.
- **DO:** Use a background `URLSession` configuration (`URLSessionConfiguration.background(withIdentifier:)`) for uploads/downloads that must survive app suspension or termination, and implement the app delegate's background-session-completion handler correctly, rather than trying to keep a foreground task alive indefinitely with `beginBackgroundTask`, which is time-limited (on the order of tens of seconds, not minutes).
- **DON'T:** Rely on background execution for anything the user would consider mission-critical without also handling the failure/never-ran case in the foreground UI — e.g., don't assume a background upload silently succeeded; surface upload state and retry from the UI when the user returns to the app.

### Push Notifications

- **DO:** Request notification authorization (`UNUserNotificationCenter.requestAuthorization`) contextually, ideally right before or right after the moment the user would understand why notifications are useful (e.g., after they enable a feature that depends on them), rather than immediately on first app launch before any context has been established. A cold, unexplained system permission prompt on first launch gets reflexively denied far more often than one shown with context, and a denial is very hard to recover from without sending the user to Settings.
- **DON'T:** Assume a push notification will always be delivered promptly, or at all — APNs is best-effort, and delivery can be delayed or coalesced under poor connectivity, Low Power Mode, or system throttling. Don't build a feature whose correctness depends on a push always arriving within a specific time window; treat push as a hint to refresh, backed by a pull-based fallback (foreground refresh, background refresh) for state that must eventually be accurate.
- **DO:** Distinguish and correctly configure the two notification categories: user-visible alerts (an `aps` payload with `alert`/`sound`/`badge`, requiring user permission) versus silent background/content-available pushes (`content-available: 1`, for triggering a background fetch without alerting the user, subject to `BGTaskScheduler`-style throttling and no permission prompt). Sending an alert-shaped payload when you meant a silent update-trigger (or vice versa) either surprises the user with an unwanted alert or silently fails to wake the app.
- **DON'T:** Register for and request push permission for a feature the user hasn't opted into or doesn't know exists — every unnecessary permission prompt spends trust the app may need later for a more important prompt (e.g., location, camera).
- **DO:** Handle notification actions, deep links embedded in the payload, and the app's cold-launch-from-notification path explicitly and test it — tapping a push should take the user to the specific, relevant content it references, not just to the app's default launch screen.

### Permissions Handling: Privacy Prompts and Info.plist Usage Descriptions

- **DO:** Add a specific, human-readable usage description string for every privacy-sensitive API the app actually uses (`NSCameraUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSMicrophoneUsageDescription`, `NSContactsUsageDescription`, `NSHealthShareUsageDescription`, `NSUserTrackingUsageDescription`, etc.) to `Info.plist`, and write the description to explain *why*, in plain language, from the user's perspective — not just what the permission is. A vague description ("This app needs your location") is a common App Store rejection reason and a poor user experience; a good one ("Used to show nearby stores and calculate delivery time") sets clear expectations.
```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>We use your location to show nearby stores and estimate delivery time.</string>
```
- **DON'T:** Call any privacy-sensitive API (camera, location, contacts, photo library, microphone, health data, tracking) without first adding its corresponding `Info.plist` usage-description key — the app will crash outright at runtime when the OS can't find the required description string for the permission prompt, which is a completely avoidable, easily-tested failure.
- **DO:** Request each permission at the specific moment its feature is used, not all up front at launch. Requesting camera, location, contacts, and notifications back-to-back on first launch overwhelms the user, produces reflexive denials, and doesn't match Apple's guidance of asking in context.
- **DON'T:** Request "Always" location access (`NSLocationAlwaysAndWhenInUseUsageDescription`) when "When In Use" is sufficient for the feature. Requesting the broader permission than the feature actually needs is both a common App Store review rejection and a real user-trust cost; request the minimum scope the feature requires, and only escalate to "Always" with a clear, separate justification if a specific background-location feature genuinely needs it.
- **DO:** Handle every permission state explicitly in the UI — not just "granted" — including "denied," "restricted" (e.g., by parental controls or MDM), and "not yet determined," and provide a clear, actionable path (e.g., a button linking to `UIApplication.openSettingsURLString`) when a denied permission blocks a feature the user is trying to use.
- **DON'T:** Silently fail or show a generic error when a permission is denied. Tell the user specifically what's blocked and how to fix it if they choose to (open Settings), rather than leaving them to guess why a feature isn't working.
- **DO:** Implement App Tracking Transparency (`ATTrackingManager.requestTrackingAuthorization`) correctly, with the required `NSUserTrackingUsageDescription`, whenever the app or any of its third-party SDKs tracks users or device data across apps/websites owned by other companies for advertising/measurement purposes — this is an App Store requirement, not optional based on whether you consider your own analytics "tracking" under Apple's definition.

### App Store Review Guideline Pitfalls

- **DO:** Provide complete, working demo credentials or a fully functional demo mode in App Review notes for any app that requires login, so reviewers can actually test the app's full functionality. A large share of rejections are simply "we couldn't get past your login screen" — entirely avoidable friction that delays every release.
- **DON'T:** Ship placeholder content, Lorem Ipsum text, broken links, or obviously non-functional buttons/features in a build submitted for review. Apple explicitly rejects apps that appear unfinished or contain filler content — every visible screen in a submitted build should be functionally complete, even if a full content catalog isn't populated yet.
- **DO:** Route any purchase of digital content or unlockable features/subscriptions consumed within the app through Apple's In-App Purchase system, per App Store Review Guideline 3.1.1, rather than linking out to an external website for a purely digital-goods purchase. Payment-flow rejections for bypassing IAP on digital goods are among the most common and most consistently enforced rejections.
- **DON'T:** Assume a workaround (a "read more on our website" link that happens to include a way to purchase digital content, a web view wrapping a payment page) will pass review — Apple actively looks for exactly this pattern. Physical goods/services consumed outside the app (ride-sharing, food delivery, hotel booking) are the legitimate exception to the IAP requirement, not digital content.
- **DO:** Include a functioning account-deletion flow directly within the app if the app supports account creation, per Apple's requirement that any app allowing account creation must also allow in-app account deletion — a "contact support to delete your account" flow alone is not sufficient.
- **DON'T:** Submit a build that crashes on launch, on a specific device/OS-version combination reviewers commonly test with, or under a specific permission-denial path (e.g., crashing when location is denied instead of degrading gracefully) — crash-on-launch and crash-under-common-conditions are automatic, fast rejections; test explicitly on the oldest supported OS version and with every permission denied before submitting.
- **DO:** Match the app's actual behavior to its App Store listing (screenshots, description, age rating) precisely — misleading screenshots, an inaccurate age rating for the actual content, or claimed functionality that doesn't exist in the submitted build are all distinct, well-documented rejection categories.
- **DON'T:** Use private/undocumented APIs, even indirectly through a third-party SDK that itself wraps one — Apple's static/dynamic analysis during review checks for known private API symbols, and apps (or SDKs) using them get rejected; verify any third-party SDK's API usage is App Store-compliant before shipping it.

### Performance: Main-Thread Blocking and Image Loading/Caching

- **DO:** Keep all UI-affecting work — layout, rendering, and anything that touches `UIView`/SwiftUI view state — on the main thread, and move everything else (networking, JSON/Codable decoding of large payloads, image decoding, file I/O, cryptography, heavy computation) off it, using `async`/`await` with a background executor, `Task.detached` where appropriate, or a dedicated `DispatchQueue`. Any main-thread work that takes more than roughly 16ms causes a dropped frame, and sustained blocking beyond a few seconds risks a watchdog termination.
```swift
// BAD: decoding a large JSON payload synchronously on the calling (often main) context
let data = try Data(contentsOf: url)
let result = try JSONDecoder().decode(BigResponse.self, from: data)

// GOOD: decode off the main thread, hop back only to update UI
let result = try await Task.detached(priority: .userInitiated) {
    let data = try Data(contentsOf: url)
    return try JSONDecoder().decode(BigResponse.self, from: data)
}.value
```
- **DON'T:** Perform synchronous network calls, unbounded disk reads, or image decoding directly inside a SwiftUI view's `body` computed property, a `UITableViewCell`/`UICollectionViewCell` `configure` method, or any other code path that runs during scrolling/layout — this is one of the most common sources of visible jank (dropped frames, stuttery scrolling) in both SwiftUI and UIKit apps.
- **DO:** Use an established image loading/caching library (Kingfisher, Nuke, SDWebImage) or, at minimum, implement your own combination of an in-memory `NSCache` and on-disk cache with proper eviction, for any list or grid that loads remote images — never re-download and re-decode the same image on every cell reuse.
- **DON'T:** Decode full-resolution images just to display a small thumbnail. Downsample images to the target display size (using `ImageIO`'s `CGImageSourceCreateThumbnailAtIndex` with `kCGImageSourceCreateThumbnailFromImageAlways`, or a caching library's built-in resizing) rather than loading a 4000×3000 photo into memory to render it at 80×80 — this is a common source of memory spikes and OOM terminations on lower-memory devices.
- **DO:** Profile with Instruments (Time Profiler for CPU, Allocations/Leaks for memory, Core Animation for frame rate) before optimizing, and specifically watch for main-thread hangs, retain cycles in closures capturing `self`, and view-hierarchy complexity that inflates layout time — don't guess at performance problems.

### Offline-First Patterns

- **DO:** Design the local data store as the source of truth the UI reads from, with network sync treated as a background process that updates the local store, rather than having the UI read directly from network responses. This pattern (sometimes called "single source of truth" or "offline-first") means the UI stays responsive and shows the best-known data even with no connectivity, and syncs opportunistically when connectivity returns.
- **DON'T:** Build a UI that shows a blank/error state the instant a network call fails, with no fallback to previously cached data. Even a simple "showing cached data from 3 minutes ago, pull to refresh" state is a dramatically better experience than an empty screen or an error the user can't act on.
- **DO:** Design an explicit conflict-resolution strategy (last-write-wins with timestamps, a merge strategy, or server-authoritative resolution with a clear notice to the user) for any data that can be edited both offline and, independently, from another device/session — silently discarding one side's changes without any strategy produces confusing, hard-to-reproduce data-loss bugs.
- **DON'T:** Queue offline writes/mutations without a durable, persisted queue and without idempotency — a write queued in memory is lost on app termination, and a non-idempotent retry (e.g., "create order" retried after a successful-but-unacknowledged first attempt) can create duplicate records on the server.
- **DO:** Surface connectivity state honestly in the UI (a banner, a disabled-but-visible send button with a "will send when back online" note) rather than letting actions silently fail or silently queue with no indication to the user about what's actually going to happen and when.

## Android

### App Architecture: MVVM with ViewModel, LiveData, and StateFlow

- **DO:** Follow Android's officially recommended app architecture: a UI layer (Composable functions or `Activity`/`Fragment` + XML) observing a `ViewModel`'s exposed state, backed by a repository layer that mediates between local (Room/DataStore) and remote (network) data sources. This layered separation survives configuration changes cleanly (the `ViewModel` outlives rotation) and keeps business logic testable independent of any `Activity`/`Fragment` lifecycle.
```kotlin
class ProfileViewModel(private val repo: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<ProfileUiState>(ProfileUiState.Loading)
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun load(id: String) = viewModelScope.launch {
        _uiState.value = ProfileUiState.Loading
        _uiState.value = runCatching { repo.fetch(id) }
            .fold(ProfileUiState::Success, ProfileUiState::Error)
    }
}
```
- **DON'T:** Put business logic, networking calls, or database access directly inside an `Activity` or `Fragment`. Activities and Fragments are UI controllers with a lifecycle the system can destroy and recreate at any time (rotation, low memory, process death) — logic living there is untestable without instrumentation and gets silently lost or duplicated across recreation.
- **DO:** Prefer `StateFlow`/`SharedFlow` (Kotlin Coroutines Flow) over `LiveData` for new code — `StateFlow` composes better with coroutines, supports operators (`map`, `combine`, `debounce`) more flexibly, and works uniformly in both Compose and non-Compose code, whereas `LiveData` is main-thread-only and Android-lifecycle-coupled by design.
- **DON'T:** Mix `LiveData` and `Flow` inconsistently within the same layer for no reason, or convert between them repeatedly (`Flow.asLiveData()` then back with `.asFlow()`) without a clear reason — pick one reactive stream type per architectural layer and stay consistent, converting only at genuine layer boundaries (e.g., exposing a `Flow` from a repository but converting to `LiveData` at the very edge if a legacy XML-based screen still needs it).
- **DO:** Give the `ViewModel` a single exposed UI state (a sealed class or data class representing the whole screen's state) rather than a dozen independent, uncoordinated `LiveData`/`StateFlow` properties. A single state object makes illegal state combinations (e.g., "loading" and "error" both true at once) representable-away and makes the UI layer's `collectAsState()`/`observe()` code simpler.
```kotlin
sealed interface ProfileUiState {
    object Loading : ProfileUiState
    data class Success(val user: User) : ProfileUiState
    data class Error(val message: String) : ProfileUiState
}
```
- **DON'T:** Pass a `Context` reference (especially an `Activity` context) into a `ViewModel` or hold one as a field — this is a classic memory leak, since the `ViewModel` can outlive the `Activity`/`Fragment` across configuration changes, keeping the destroyed `Activity` (and everything it references) alive in memory. Use `Application` context (via `AndroidViewModel` only when truly needed) or, better, avoid needing a `Context` in the `ViewModel` at all by injecting only what's needed (a repository, a use case).

### Jetpack Compose vs. XML Views

- **DO:** Default new screens and new Android projects to Jetpack Compose — it's Google's recommended modern UI toolkit, reduces boilerplate dramatically versus XML layouts + `findViewById`/view binding, and has reached parity or better for the vast majority of common UI needs as of current stable releases.
- **DON'T:** Rip out a large, stable, working XML-based screen and rewrite it in Compose purely for its own sake in a mature app under active feature development — as with SwiftUI/UIKit, incremental adoption (new screens in Compose, `ComposeView`/`AndroidView` interop at the boundary for existing XML screens) is almost always the lower-risk path than a wholesale rewrite.
- **DO:** Use `AndroidView` to embed a legacy `View`/XML-based component inside Compose, and `ComposeView` to embed Compose inside an XML-based `Fragment`/`Activity`, when a full migration isn't justified yet — Compose's interop APIs are specifically designed for this gradual-migration path.
- **DON'T:** Write Compose code that recomposes the entire screen on every state change because state was hoisted too high or a large composable wasn't broken into smaller, independently-recomposable pieces. Unnecessary broad recomposition is Compose's most common performance pitfall — see the performance section below for specifics.
- **DO:** Understand that Compose and XML views can coexist productively in the same app during a migration — pick a clear per-module or per-screen boundary (e.g., "all new feature modules are Compose-only; legacy modules stay XML until touched") rather than an ad-hoc, undocumented mix that confuses contributors about which pattern to follow where.

### Navigation (Navigation Component / Compose Navigation)

- **DO:** Use the Jetpack Navigation Component (`NavHost`/`NavController` for Compose, or the XML nav graph for View-based UI) rather than manually managing a back stack of `Fragment` transactions or Compose screen state by hand. It correctly integrates with the system back button/gesture, handles the up/back distinction, supports deep links declaratively, and gives you a visual/serializable graph of the app's flow.
```kotlin
NavHost(navController = navController, startDestination = "home") {
    composable("home") { HomeScreen(onProfileClick = { id -> navController.navigate("profile/$id") }) }
    composable("profile/{id}") { backStackEntry ->
        ProfileScreen(id = backStackEntry.arguments?.getString("id")!!)
    }
}
```
- **DON'T:** Pass complex objects (a full domain model, a `Parcelable` with nested data) directly as navigation arguments when a lightweight identifier (an ID string/long) that the destination screen re-fetches or looks up from a shared `ViewModel`/repository would do. Large navigation argument payloads bloat the saved-instance-state bundle and can trigger `TransactionTooLargeException` on process restoration.
- **DO:** Define and use type-safe navigation arguments (Navigation Component's Safe Args plugin for XML graphs, or a typed route/sealed-class approach for Compose Navigation) rather than raw string-keyed bundles pieced together by hand — type safety here catches a wrong-argument-name or wrong-type bug at compile time instead of a runtime crash deep in a screen nobody's actively testing.
- **DON'T:** Ignore the distinction between "back" (pop the current screen off this task's stack) and "up" (navigate to the logical parent screen per the app's hierarchy, which may differ from the literal previous screen on the stack, e.g. after a deep link). Configure `NavigationUI`/the nav graph's `AppBarConfiguration` correctly so the system back button and any in-app up button both behave per Android's navigation conventions.

### Local Persistence: Room, DataStore, and SharedPreferences

- **DO:** Use Room (Jetpack's SQLite abstraction) for any structured, relational, or queryable local data — anything beyond a handful of scalar preferences. Room gives compile-time-verified SQL queries, built-in support for `Flow`/coroutines-based observation of query results, and a real migration system for schema changes.
```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    fun observe(id: String): Flow<User?>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(user: User)
}
```
- **DO:** Use Jetpack DataStore (Preferences DataStore for simple key-value, Proto DataStore for typed structured data) for app settings and preferences in new code, rather than `SharedPreferences`. DataStore is fully asynchronous (no main-thread I/O risk, unlike `SharedPreferences.apply()`/`commit()` under certain conditions), transactionally consistent, and exposes updates as a `Flow` for reactive UI.
- **DON'T:** Continue using `SharedPreferences` in new code without a specific reason (e.g., a small, targeted addition to an already-`SharedPreferences`-based legacy screen where migrating is out of scope). `SharedPreferences.commit()` blocks the calling thread on disk I/O, and even `apply()` has known consistency edge cases DataStore was specifically built to avoid.
- **DON'T:** Run Room queries or any disk I/O on the main thread — Room enforces this at compile/runtime by default (throwing `IllegalStateException` for main-thread queries unless explicitly allowed), and that default should be respected, not disabled, except in narrow, deliberate cases.
- **DO:** Write and test real Room migrations (`Migration` objects with explicit `ALTER TABLE`/`CREATE TABLE` SQL) for every schema version bump, and never ship `fallbackToDestructiveMigration()` in production for an app already in users' hands — that silently wipes the local database on schema mismatch, which is real user data loss for anything not already synced to a server.

### Background Work: WorkManager vs. Services

- **DO:** Use WorkManager for deferrable, guaranteed background work — syncing data, uploading logs, periodic cleanup — especially work that should survive app restarts and process death, and that should respect system battery/network constraints (`Constraints.Builder().setRequiredNetworkType(...)`). WorkManager is Android's recommended API specifically because it transparently picks the right underlying mechanism (`JobScheduler`, `AlarmManager`, or a direct execution) based on OS version and constraints, so you don't have to branch on API level yourself.
```kotlin
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
    .build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync", ExistingPeriodicWorkPolicy.KEEP, syncRequest
)
```
- **DON'T:** Use a bare `Service` (or worse, a long-running background thread with no lifecycle awareness) for deferrable background work that doesn't need to run *right now* — modern Android (8.0+) aggressively restricts background services, and code written against pre-restriction assumptions about background execution will simply stop working (or get killed) on current OS versions.
- **DO:** Reserve a Foreground Service, with its required persistent notification, for work the user is actively aware of and that must keep running immediately and continuously while the app is backgrounded — active music playback, an ongoing navigation session, an active fitness-tracking session, or a large in-progress file transfer the user explicitly started. Declare the correct foreground service type (`dataSync`, `mediaPlayback`, `location`, etc.) as required by current Android versions, since an incorrectly-typed or undeclared foreground service can be killed by the system or rejected by Play policy.
- **DON'T:** Start a Foreground Service for background work the user isn't aware of or didn't initiate, just to dodge background execution limits — this is both a poor user experience (an unwanted persistent notification) and increasingly restricted/flagged by Play Store policy and by the OS itself.
- **DO:** Use `WorkManager`'s expedited work (`setExpedited()`) for background work that needs to start soon but doesn't need the always-on guarantee of a foreground service, as the modern middle ground between "deferrable, whenever" (regular WorkManager) and "must run now, continuously, with user visibility" (foreground service).

### Permissions Handling: Runtime Permissions

- **DO:** Request dangerous/runtime permissions (camera, location, contacts, storage, microphone, etc.) at the point of use, using the `ActivityResultContracts.RequestPermission()` (or `RequestMultiplePermissions()`) API, and explain *why* the permission is needed — either via a rationale dialog shown when `shouldShowRequestPermissionRationale()` returns true, or contextually before the system prompt appears at all.
```kotlin
val requestPermission = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    if (granted) startCamera() else showRationale()
}
```
- **DON'T:** Request every permission the app might ever need immediately on first launch, before the user has any context for why each one matters. Batched, context-free upfront permission requests measurably increase denial rates and can make an otherwise-usable app feel invasive on first impression.
- **DO:** Handle both the "denied" and "denied, don't ask again"/permanently-denied states distinctly — the latter (detectable via `shouldShowRequestPermissionRationale()` returning false after a prior denial) means the system will no longer show the prompt at all, so the only recourse is directing the user to the app's system Settings page.
- **DON'T:** Assume a permission granted at one point in the session remains granted for the rest of the app's lifetime — starting with Android 11, users can grant one-time permissions for location/camera/microphone that automatically revoke after the app is backgrounded, so re-check permission state when a feature that depends on it is used again, rather than caching a stale "granted" flag.
- **DO:** Declare the correct permission variant for the actual need — e.g., `ACCESS_COARSE_LOCATION` vs. `ACCESS_FINE_LOCATION`, and only request `ACCESS_BACKGROUND_LOCATION` (which requires a separate, additional runtime prompt on Android 10+) when foreground location genuinely isn't sufficient for the feature, since Play policy requires justifying background location access explicitly during app review.

### Play Store Policy Pitfalls

- **DO:** Target a current API level as required by Google Play's target API level policy (Play requires apps to target an API level within a defined recency window of the current Android release for new submissions and updates) — apps targeting an outdated API level are blocked from publishing updates until they raise `targetSdkVersion`.
- **DON'T:** Request the `QUERY_ALL_PACKAGES` permission, broad storage access (`MANAGE_EXTERNAL_STORAGE`), or other sensitive/restricted permissions without a use case that clearly falls into Google Play's documented allowed-use categories for that permission — these are among the most commonly rejected/pulled permission declarations, and Play increasingly requires an explicit declaration form justifying the specific use case during submission.
- **DO:** Provide a complete, accurate Data Safety section in the Play Console listing, describing exactly what data is collected, why, and whether it's shared with third parties — this must match the app's actual behavior (including third-party SDKs bundled in, like analytics or ad SDKs), not just first-party data collection, and mismatches are an active enforcement target.
- **DON'T:** Ship ad content, permission requests, or data collection inconsistent with the app's declared target-audience/content rating, particularly for apps in or resembling the Families/kids category — Play's policies for that category are significantly stricter (no behavioral ad targeting, no unnecessary permissions, restricted third-party SDKs).
- **DO:** Route digital-goods purchases through Google Play's Billing Library the same way iOS requires In-App Purchase for digital content — this is Play's equivalent policy requirement (with similar carve-outs for physical goods/services consumed outside the app), and non-compliant payment flows are a recurring rejection category.
- **DON'T:** Leave debug logging, hardcoded test/staging API endpoints, or verbose crash information enabled in a production release build — beyond the policy risk of leaking data, this is a straightforward security/quality issue reviewers and users alike may flag.

### Performance: Overdraw, RecyclerView, and LazyColumn

- **DO:** Use Android Studio's GPU rendering profiler / Layout Inspector to check for overdraw (the same pixel being drawn multiple times per frame due to overlapping opaque backgrounds) and reduce unnecessary background layers, especially deeply nested `ViewGroup` hierarchies with redundant backgrounds.
- **DON'T:** Nest `ViewGroup`s (deeply nested `LinearLayout`s, in particular) many levels deep for a layout that could be expressed flatly with `ConstraintLayout` (in the View system) or a single Compose layout composable. Deep view hierarchies increase measure/layout pass cost and are one of the most common, most fixable sources of janky scrolling in legacy XML-based screens.
- **DO:** In the View system, use `RecyclerView` (never a non-recycling `ListView`/manually-inflated views in a `ScrollView`) for any list that could grow beyond a small, fixed number of items, with a correctly implemented `DiffUtil`-backed adapter so item updates animate and update efficiently rather than triggering a full rebind of every visible row.
```kotlin
class ItemsAdapter : ListAdapter<Item, ItemViewHolder>(object : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(a: Item, b: Item) = a.id == b.id
    override fun areContentsTheSame(a: Item, b: Item) = a == b
})
```
- **DON'T:** Put a `LazyColumn`/`LazyRow` (or `RecyclerView`) inside a scrollable parent (another `verticalScroll`/`ScrollView`) without constraining its height — nesting two independently-scrolling lists either breaks scroll gesture handling or, if the inner list's height is unconstrained, forces it to measure and lay out its *entire* content up front, defeating the whole point of lazy loading.
- **DO:** Keep Compose composables small and give each list item a stable, unique `key` in `LazyColumn`/`LazyRow` (`items(list, key = { it.id })`) so Compose can correctly track item identity across insertions, removals, and reordering, and only recompose/re-animate the items that actually changed.
```kotlin
LazyColumn {
    items(items = users, key = { it.id }) { user -> UserRow(user) }
}
```
- **DON'T:** Read mutable state broadly at the top of a large composable when only a small nested part of the UI actually depends on it — this forces the whole composable (and everything it emits) to recompose on every change. Hoist state reads down to the smallest composable that actually needs them, and use `derivedStateOf` for values computed from frequently-changing state that itself changes less often.
- **DO:** Use the Compose compiler's stability/recomposition reports (or the Layout Inspector's recomposition counts) to find composables recomposing far more often than their visible output changes, and address unstable parameters (lambdas capturing changing state, unstable collection types) as the usual root cause.

### Offline-First Patterns

- **DO:** Apply the same "local database as single source of truth, network as background sync" pattern described for iOS — a repository exposes a `Flow` backed by Room that the UI observes, and a WorkManager-scheduled sync job (or a direct suspend function triggered on refresh) updates Room from the network, with the UI updating automatically via the `Flow` regardless of whether the change came from a network sync or a local user edit.
- **DON'T:** Have UI code call the network directly and hold the response in transient `ViewModel` state with no local persistence backing it — on Android in particular, process death (which happens far more readily than app "quitting" from the user's perspective, e.g. when the OS reclaims memory for another app) will silently wipe that transient state, and the user returns to what looks like data loss.
- **DO:** Combine `WorkManager`'s network-constrained periodic/one-time work with a manual "sync now"/pull-to-refresh trigger, so data both syncs opportunistically in the background and can be forced immediately when the user explicitly asks for fresh data.
- **DON'T:** Silently swallow a failed sync with no visible retry path or staleness indicator. Persist a "last successfully synced at" timestamp and surface it (or a simple "offline — showing saved data" banner) so the user understands why what they're looking at might not be current.

### Fragmentation Across API Levels and Devices

- **DO:** Explicitly decide and document the app's `minSdkVersion` based on real analytics of the actual target user base (Play Console's Android version distribution data for the specific app/region, not generic industry-wide charts), and use `Build.VERSION.SDK_INT` checks (or, better, AndroidX's backward-compatible APIs, which handle this internally) for any API introduced after `minSdkVersion`.
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    // API 33+ only behavior
} else {
    // fallback for older devices
}
```
- **DON'T:** Call an API gated to a higher OS version without a `SDK_INT` guard (or without confirming AndroidX already backports it) — this is a guaranteed crash (`NoSuchMethodError`/`ClassNotFoundException` or similar) on any device below that OS version still within the app's supported `minSdkVersion` range, and it's one of the most common defects in AI-generated Android code that doesn't check API-level availability.
- **DO:** Test on a real range of screen sizes and densities (small phones, tablets, foldables) using resizable emulator configurations or real devices, not just one reference phone size — Android's device fragmentation across manufacturers, screen sizes, aspect ratios, and notch/cutout shapes is far wider than iOS's, and layouts that assume one fixed screen size or aspect ratio break visibly on others.
- **DON'T:** Hardcode pixel dimensions (`px`) for UI sizing — use density-independent pixels (`dp`) for layout dimensions and scale-independent pixels (`sp`) for text, so the UI scales correctly across the actual density range of real Android devices, which spans a much wider range of DPI buckets than iOS's small, fixed set of device classes.
- **DO:** Account for manufacturer-specific behavior differences that go beyond stock Android — some OEM Android skins impose more aggressive background-process killing than stock AOSP/Pixel behavior, which can affect WorkManager/background service reliability in practice even when the code is technically correct per the official APIs; test on a couple of major OEM devices (not exclusively Pixel/emulator) before assuming background behavior generalizes.

## Cross-Platform (React Native & Flutter)

### When Cross-Platform Is (and Isn't) Appropriate

- **DO:** Choose a cross-platform framework (React Native, Flutter) when the team's priority is shipping and maintaining one codebase across iOS and Android efficiently, the app's UI is largely standard (forms, lists, navigation, media, moderate animation) without deep platform-specific capability requirements, and the team either already has strong web/JS or Dart expertise, or lacks separate native iOS and Android teams to maintain two codebases.
- **DON'T:** Choose a cross-platform framework for an app that fundamentally depends on deep platform-specific capability — heavy AR/ARKit-or-ARCore-specific work, complex custom camera/video pipelines, background audio processing with tight latency requirements, or an app whose primary value proposition *is* pixel-perfect adherence to one platform's native look, feel, and cutting-edge OS features on day one of a new OS release. In these cases, native development avoids a constant stream of bridging workarounds and lag behind new OS APIs.
- **DO:** Evaluate cross-platform against native per-project, considering team composition and long-term maintenance realistically, not just initial build speed — a cross-platform app still needs someone competent in the underlying native platform to debug native crashes, handle store-specific requirements, and write/vet any native module bridge code, so "we don't need native expertise" is rarely fully true in practice.
- **DON'T:** Assume "cross-platform" means "write once, never touch platform-specific code again." Both React Native and Flutter still frequently require platform-specific branches for permissions handling, certain native integrations, and adapting to each platform's UI conventions (see below) — budget for that ongoing platform-specific work rather than treating it as an edge case.

### Bridging Native Modules Safely

- **DO:** Wrap native SDK integrations (payment SDKs, native camera/AR libraries, platform-specific analytics) behind a well-defined native module with a narrow, well-typed interface — in React Native via the Native Modules/Turbo Modules API, in Flutter via platform channels (`MethodChannel`/`EventChannel`) — rather than scattering ad-hoc platform-specific calls throughout the shared codebase.
```dart
// Flutter side
static const _channel = MethodChannel('com.app/payments');
Future<String> startPayment(double amount) =>
    _channel.invokeMethod('startPayment', {'amount': amount});
```
- **DON'T:** Send large, complex, or high-frequency payloads across the JS-native bridge (React Native's legacy bridge) or a platform channel (Flutter) without considering serialization cost — every crossing serializes/deserializes data, and doing this at high frequency (e.g., streaming raw sensor or video frame data per-frame) is a well-known performance bottleneck; batch, throttle, or move that specific hot path to fully native code with only summarized results crossing the bridge.
- **DO:** Handle the native side's errors and edge cases explicitly and propagate them as typed, catchable errors on the cross-platform side (a rejected Promise with a defined error code in RN, a `PlatformException` with a defined code in Flutter) rather than letting a native crash silently kill the app or letting a native failure surface as an opaque, unhandled exception in JS/Dart.
- **DON'T:** Assume a community-maintained native-module package that hasn't been updated in a long time, or that lacks recent-OS-version compatibility reports, will keep working as-is on new OS releases without verification — third-party native bridges are a common source of app breakage after an iOS/Android major version update specifically because they depend on private/internal native behavior that a new OS release can change without notice.
- **DO:** Use React Native's New Architecture (Turbo Modules/Fabric) or its equivalent current recommended module system, and Flutter's current platform channel/plugin APIs, for new native bridge code rather than deprecated bridging mechanisms that are being phased out — check the framework's current stable documentation, since both ecosystems have moved through multiple generations of native interop APIs.

### Performance Pitfalls: Re-renders, JS Thread Blocking, Widget Rebuild Scope

- **DO:** Memoize React Native components and derived values appropriately (`React.memo`, `useMemo`, `useCallback`) to prevent unnecessary re-renders cascading through a component tree, and profile with React DevTools/Flipper before assuming a specific memoization will help — as with web React, misapplied memoization can add overhead without benefit if the underlying props/state genuinely change every render.
- **DON'T:** Perform expensive synchronous computation (heavy JSON parsing, large list transformations, image manipulation, cryptography) directly on React Native's JS thread during an interaction — the JS thread also drives touch handling and layout calculation triggers in the classic architecture, so blocking it causes the UI to visibly freeze and touch input to feel unresponsive, distinct from (but analogous to) blocking a native main thread.
```jsx
// BAD: heavy synchronous work blocks the JS thread during a user interaction
const onPress = () => {
  const result = expensiveTransform(hugeArray); // freezes UI
  setResult(result);
};

// GOOD: defer to InteractionManager or move to a native/worker thread
const onPress = () => {
  InteractionManager.runAfterInteractions(() => {
    const result = expensiveTransform(hugeArray);
    setResult(result);
  });
};
```
- **DO:** Use `FlatList`/`FlashList` (a community-maintained, better-performing virtualized list) for React Native lists, with a stable `keyExtractor`, `getItemLayout` where item sizes are known, and appropriate `windowSize`/`initialNumToRender` tuning — never render a large list with `.map()` inside a plain `ScrollView`, which mounts every item at once regardless of visibility.
- **DON'T:** Rebuild a large Flutter widget subtree unnecessarily because state was hoisted too high in the widget tree, or because a `StatefulWidget`'s `build()` method recomputes expensive derived values inline on every rebuild instead of caching them. Scope `setState`/state-management rebuilds to the smallest widget subtree that actually needs to change, using `const` constructors wherever a widget's inputs are genuinely constant so Flutter can skip rebuilding it entirely.
```dart
// BAD: entire large tree rebuilds when only the counter text changes
class MyScreen extends StatefulWidget {
  @override State<MyScreen> createState() => _MyScreenState();
}
// Counter lives in the same build() as an expensive, unrelated widget tree

// GOOD: isolate the changing part
class MyScreen extends StatelessWidget {
  const MyScreen({super.key});
  @override
  Widget build(BuildContext context) => Column(children: [
    const ExpensiveStaticHeader(), // const: never rebuilds
    const CounterWidget(),          // owns its own state, rebuilds alone
  ]);
}
```
- **DO:** Use Flutter's `const` constructors aggressively for widgets whose parameters don't change, and reach for `Consumer`/`Selector`-style scoped rebuilding (in Provider, Riverpod, Bloc, etc.) so a state-management update only rebuilds the specific widgets that actually read the changed slice of state, not the entire screen.
- **DON'T:** Use `setState()` at the top of a large screen-level `StatefulWidget` to update one small, deeply nested piece of UI — this is Flutter's most common performance anti-pattern for teams new to the framework, and it rebuilds far more of the widget tree than necessary; extract the changing part into its own smaller stateful widget, or use a state-management solution that supports scoped rebuilds.
- **DO:** Profile React Native with Flipper/the built-in Perf Monitor (checking both JS and UI thread frame rates separately) and Flutter with DevTools' widget rebuild and performance overlays before optimizing — both frameworks report JS/UI-thread and widget-rebuild information explicitly, so use it rather than guessing.

### Platform-Specific UI Adaptation (Not Just Stretching One Design to Both)

- **DO:** Adapt navigation chrome, iconography, and interaction conventions per platform even in a shared cross-platform codebase — a back button/back gesture behaves and is positioned differently on iOS (top-left, swipe-from-edge back gesture) versus Android (system back button/gesture, or an up-arrow in the app bar per Material conventions); tab bars sit at the bottom on iOS by convention while Android has historically favored a bottom nav bar too but with different elevation/ripple/selection-indicator conventions; date pickers, alerts, and action sheets have distinct native look-and-feel (`UIAlertController`-style vs. Material dialogs) that users on each platform recognize.
- **DON'T:** Ship one visual design, unmodified, on both iOS and Android and call it "cross-platform" — a Material-styled app with Android-style elevation shadows and ripple effects looks foreign on iOS, and an iOS-styled app with `SF Symbols`-esque icons and iOS-style switches looks foreign on Android. Both Apple's Human Interface Guidelines and Google's Material Design guidelines represent real, well-tested platform conventions users have learned to expect; ignoring them produces an app that feels subtly "off" on at least one platform even if it's functionally correct.
- **DO:** Use the frameworks' built-in platform-adaptive primitives where they exist — React Native's `Platform.select()`/`Platform.OS` checks and platform-specific file extensions (`Component.ios.js`/`Component.android.js`) for targeted differences; Flutter's `Platform.isIOS`/`Platform.isAndroid` checks combined with `Cupertino` widgets on iOS and `Material` widgets on Android where a platform-native look matters (e.g., `CupertinoSwitch` vs. `Switch`, `CupertinoActivityIndicator` vs. `CircularProgressIndicator`).
```jsx
const Button = Platform.select({
  ios: () => IOSStyledButton,
  android: () => MaterialStyledButton,
})();
```
- **DON'T:** Reach for full platform-specific branching on every single component when a shared, neutral design that doesn't strongly signal either platform's identity is a legitimate and simpler choice for a brand-driven app (many well-known consumer apps deliberately use one consistent branded look on both platforms). The mistake isn't "using one design" per se — it's doing so unintentionally, by default, without evaluating whether the app's specific screens (system dialogs, navigation chrome, form controls) benefit from platform-native conventions that users specifically rely on for those interaction patterns.
- **DO:** Respect each platform's system-level typography and spacing conventions (Dynamic Type on iOS, font scale on Android) even within a shared design system — a cross-platform app that hardcodes fixed font sizes ignoring the user's OS-level accessibility text-size setting fails accessibility on both platforms simultaneously.

### State Management Conventions

- **DO:** Pick a state management approach appropriate to the app's actual state complexity: React Context + hooks or a lightweight library (Zustand, Jotai) for small-to-medium React Native apps; Redux Toolkit or a similar structured store for apps with complex, deeply shared cross-screen state; Provider/Riverpod/Bloc for Flutter apps scaled similarly by complexity (Provider/Riverpod for simpler dependency-injection-plus-reactive-state needs, Bloc for teams wanting a strict, testable event-driven architecture).
- **DON'T:** Introduce a full unidirectional-flow state management library (Redux, Bloc with strict event/state separation) for a small app with mostly local, screen-scoped state — the ceremony (actions, reducers/blocs, boilerplate types) is disproportionate to the problem and slows down simple features without a corresponding maintainability win.
- **DO:** Keep server-derived/cached data in a dedicated data-fetching/caching layer (React Query/TanStack Query or RTK Query in React Native; `riverpod`'s `AsyncNotifier`/a dedicated repository pattern in Flutter) rather than manually managing loading/error/data state and re-fetch logic by hand for every screen that needs remote data — this avoids the extremely common bug class of stale, duplicated, or inconsistently-refreshed copies of the same server data living in multiple local state slices.
- **DON'T:** Store server-fetched data as plain component state with no caching/invalidation strategy across multiple screens that need the same data — this produces the same "N screens independently fetch and can disagree about the same underlying data" problem cross-platform frameworks are just as susceptible to as any other client architecture.

### App Store Submission for Both Platforms Simultaneously

- **DO:** Budget for genuinely separate submission processes and separate sets of platform-specific requirements even when shipping from one cross-platform codebase — Apple's Info.plist usage descriptions, ATT prompt, and App Store Review Guidelines apply independently from Google Play's Data Safety section, target API level requirement, and Play policy review, and passing one platform's review says nothing about the other's.
- **DON'T:** Assume a single generic privacy policy, a single generic set of App Store/Play Store screenshots, or a single generic app description will satisfy both stores' specific formatting and content requirements without platform-specific review — each store has its own listing requirements, required assets/sizes, and metadata review criteria.
- **DO:** Test the actual release build (not just a debug/simulator build) on both platforms before submission — a cross-platform framework's release-mode builds (Hermes-compiled JS in React Native's case, AOT-compiled Dart in Flutter's) can behave subtly differently from debug builds (different error visibility, different startup timing, different bundled-asset resolution), and issues specific to release mode are a common source of last-minute submission surprises.
- **DON'T:** Ship a bundled native module or third-party SDK on both platforms without independently verifying its App Store *and* Play Store compliance — an SDK compliant on one store isn't automatically compliant on the other (e.g., different tracking-disclosure requirements, different permission-declaration rules), and both stores review independently.
- **DO:** Version the app consistently across platforms in a way that maps clearly to the underlying codebase's version (matching semantic version numbers, with platform-specific build numbers incrementing independently per each store's requirements) so a bug report referencing "version 2.3.1" is unambiguous regardless of which store's build the user has.

## Cross-Cutting Mobile Concerns

### Responsive Layout Across Screen Sizes

- **DO:** Build layouts that adapt to a continuous range of screen sizes and aspect ratios using each platform's relative/constraint-based layout primitives (SwiftUI's `GeometryReader`/adaptive stacks and size classes, Auto Layout constraints in UIKit, `ConstraintLayout`/Compose's `Modifier.weight`/`BoxWithConstraints` in Android, Flutter's `LayoutBuilder`/`MediaQuery`/flexible widgets, React Native's Flexbox layout) rather than fixed pixel/point dimensions.
- **DON'T:** Hardcode absolute screen dimensions (assuming a specific device's width/height in points/dp) anywhere in layout code. The range of shipping screen sizes — iPhone SE through iPhone Pro Max, iPad in multiple sizes and orientations and Split View widths, the enormous range of Android phone/tablet/foldable sizes — makes any hardcoded dimension assumption break visibly on some real, currently-shipping device.
```swift
// BAD: assumes a specific screen width
.frame(width: 375)

// GOOD: relative to available space
.frame(maxWidth: .infinity)
```
- **DO:** Test explicitly on both the smallest and largest currently-supported device sizes, plus at least one tablet/large-screen form factor if the app supports tablets, plus both orientations if the app supports rotation — a layout that only gets checked on the developer's own device (often a mid-to-large, portrait-only phone) reliably misses truncation, overlap, and awkward whitespace issues on the extremes.
- **DON'T:** Ignore multi-window/split-screen/foldable states on Android and Stage Manager/Split View on iPadOS — an app that assumes it always owns the full screen can render badly or crash when resized into a partial-width window, which is an increasingly common real usage pattern on tablets and foldables specifically.
- **DO:** Design foldable-aware layouts (using Jetpack WindowManager's `WindowInfoTracker` on Android, or size-class-based adaptive layouts generally) for apps likely to run on foldable devices, accounting for the hinge/fold area and the transition between folded and unfolded states.

### Accessibility: VoiceOver/TalkBack, Dynamic Type/Font Scaling, Touch Targets

- **DO:** Give every interactive element and every meaningful image a correct accessibility label/description — `accessibilityLabel` in SwiftUI/UIKit, `contentDescription` in Android Views, `semantics`/`Semantics` widget properties in Flutter, `accessibilityLabel` in React Native — so VoiceOver (iOS) and TalkBack (Android) announce something meaningful rather than a raw resource ID, "button," or nothing at all.
```swift
Image(systemName: "trash")
    .accessibilityLabel("Delete item")
    .accessibilityHint("Removes this item from your list")
```
- **DON'T:** Leave decorative-only images unlabeled-but-still-focusable, or meaningful icon-only buttons with no label at all — a screen-reader user tabbing through an unlabeled icon button has no way to know what it does, and a decorative image that IS focusable just wastes a swipe/tab stop without conveying anything.
- **DO:** Support each platform's system text-scaling setting — Dynamic Type on iOS (using scalable text styles like `.body`/`.headline` rather than fixed point sizes, and testing at the largest accessibility text sizes) and font scale on Android (using `sp` units, never fixed `dp`/`px` for text, and testing with the system font scale set to its maximum). Layouts should reflow/wrap gracefully at large text sizes rather than truncating or overlapping.
- **DON'T:** Disable text scaling (`.dynamicTypeSize(.large...(.large))`-style clamping, or Android's `android:targetSdkVersion`-adjacent tricks that ignore the system font scale) without a specific, justified reason (e.g., a fixed-size logo lockup) — overriding the user's chosen text size defeats an accessibility setting they deliberately configured, often for a genuine visual-impairment need.
- **DO:** Size every touch target to at least the platform-recommended minimum — 44×44 points on iOS (per Apple's Human Interface Guidelines), 48×48 dp on Android (per Material Design guidelines) — even if the visible icon/glyph is smaller; use padding to expand the tappable area rather than shrinking the target to match a small visual asset.
- **DON'T:** Rely on color alone to convey state or meaning (a red vs. green indicator with no icon or text label, a form field that only turns red on error with no accompanying error message) — this fails for colorblind users and for anyone using a grayscale/reduced-color display accommodation; pair color with an icon, text, or shape change.
- **DO:** Test both platforms with their real screen readers turned on (VoiceOver via triple-click-side-button or Settings, TalkBack via Accessibility settings) navigating the actual critical flows (not just a cursory swipe-through) — automated accessibility scanners catch missing labels and contrast issues but miss focus-order problems, confusing grouping, and genuinely unusable custom-gesture interactions that only show up under real screen-reader navigation.
- **DON'T:** Build custom gesture-based interactions (custom swipe actions, custom drag-and-drop, a bespoke carousel) without an accessible alternative path — a screen-reader user or a switch-control user often can't perform an arbitrary custom gesture, so provide an equivalent action reachable via standard accessibility navigation (e.g., an accessibility action registered alongside a swipe-to-delete gesture).

### Deep Linking

- **DO:** Support standard deep links (custom URL schemes as a fallback, plus platform-verified universal links on iOS and App Links on Android) so links from outside the app — push notifications, emails, web pages, other apps — can route directly to specific in-app content, and implement the platform's domain-verification mechanism (`apple-app-site-association` file for iOS Universal Links, the Digital Asset Links `assetlinks.json` file for Android App Links) so the OS trusts the association without prompting a disambiguation dialog.
- **DON'T:** Rely solely on a custom URL scheme (`myapp://`) for links that matter for growth/sharing/marketing — custom schemes can be claimed by any app with no verification, are vulnerable to being hijacked/squatted by another app, and don't gracefully fall back to a web page or app-store listing when the app isn't installed, unlike properly configured universal links/App Links.
- **DO:** Handle a deep link that arrives while the app is already running (foreground), one that arrives while backgrounded, and one that cold-launches the app, testing each path — deep link handling code that only gets exercised via cold launch during development commonly breaks silently for the warm/foreground case, since the navigation stack is in a different state.
- **DON'T:** Route a deep link straight into arbitrary deep app state without validating that the referenced content still exists and that the user has the necessary auth/permission to see it (a deep link to an order that's been deleted, or a screen requiring login when the user's session has expired) — handle the "target no longer valid" and "not authenticated" cases explicitly rather than crashing or showing a blank/broken screen.
- **DO:** Design deep link routes as a first-class part of the app's navigation architecture (the same router/coordinator that handles in-app navigation should be the thing that resolves a deep link URL into a route), rather than a separate, parallel, ad-hoc URL-parsing code path that duplicates and can drift from the app's real navigation graph.

### Versioning and Staged Rollouts

- **DO:** Use staged/phased rollouts on both stores (Apple's phased release over a set number of days, Google Play's staged rollout percentage) for meaningful releases, monitoring crash rate and key metrics at each stage before expanding to 100% of users — this limits the blast radius of a release that turns out to have a serious regression that testing missed.
- **DON'T:** Ship every release to 100% of users immediately, especially for a release containing a risky change (a data migration, a major dependency upgrade, a significant refactor) — a staged rollout that's halted at 5-10% after a crash-rate spike affects a tiny fraction of the user base, versus the same bug shipped to everyone at once.
- **DO:** Maintain a clear, consistent versioning scheme (semantic versioning for the user-facing version, an always-incrementing build number per store's requirement) and keep meaningful, specific release notes rather than a perpetual "bug fixes and improvements" — specific release notes help support triage user reports against known issues in a given version, and vague notes actively hide that signal.
- **DON'T:** Force an immediate, blocking update for every release — reserve forced updates for genuinely breaking changes (an incompatible API change on the backend, a critical security fix) and let most releases roll out as optional/staged updates, since forced updates on every release train users to resent and delay updating, or in the worst case can strand users who can't update immediately (poor connectivity, storage constraints).
- **DO:** Support backend/API compatibility across at least the last few app versions still meaningfully in use (visible in your own analytics), since staged rollouts and users who don't auto-update mean multiple app versions are always live against the same backend simultaneously — a backend change that assumes only the newest app version exists breaks real users on older-but-still-supported versions.

### Crash Reporting and Analytics Integration Discipline

- **DO:** Integrate a crash reporting tool (Crashlytics, Sentry, or equivalent) from the start of a project and treat crash-free-user-rate as a tracked, reviewed metric — catching and fixing crashes before they compound across releases is dramatically cheaper than debugging a backlog of unreproducible crash reports months later.
- **DON'T:** Log personally identifiable information, auth tokens, payment details, or other sensitive data into crash reports or analytics events, even inadvertently via a full request/response dump attached as crash context. Crash/analytics data often flows to third-party services and is retained under different policies than the app's own data handling — scrub sensitive fields before attaching them as crash breadcrumbs or event properties.
- **DO:** Symbolicate crash reports properly (uploading dSYMs for iOS, ProGuard/R8 mapping files for Android) as part of the release process — an unsymbolicated crash report showing raw memory addresses or obfuscated method names is effectively useless for actually diagnosing the bug.
- **DON'T:** Instrument analytics events so densely and inconsistently that the event taxonomy becomes unmaintainable (duplicate events with slightly different names for the same user action, events fired multiple times per single logical action, no shared naming convention across features) — a sprawling, undocumented event taxonomy makes the resulting data unreliable exactly when it's needed for a real product decision.
- **DO:** Gate analytics/crash-reporting SDK initialization and data collection appropriately behind the user's actual consent state where required by applicable privacy regulation and by App Tracking Transparency/Play Data Safety disclosures — don't initialize a tracking-capable SDK and start sending events before consent has actually been obtained where consent is legally or policy-required.
- **DON'T:** Treat a spike in a specific crash signature as noise because "it's only a small percentage of sessions" without checking whether it's concentrated on a specific OS version, device model, or app version — a crash affecting 1% of *overall* sessions can be affecting 100% of users on one specific device/OS combination, which is a very different severity than a uniformly-distributed 1%.

### Secure Local Storage: Keychain/Keystore vs. Plain Storage

- **DO:** Store credentials, auth tokens, encryption keys, and any other sensitive secret in the iOS Keychain (via `Keychain Services` or a well-vetted wrapper like KeychainAccess) on iOS, and in the Android Keystore system (via `EncryptedSharedPreferences`/Jetpack Security, or directly via `KeyStore` for key material) on Android — both provide OS-level, hardware-backed-where-available secure storage specifically designed for this purpose, distinct from and stronger than the app's general-purpose data storage.
```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "authToken",
    kSecValueData as String: tokenData,
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
]
SecItemAdd(query as CFDictionary, nil)
```
- **DON'T:** Store an auth token, refresh token, API key, or password in `UserDefaults`, plain `SharedPreferences`, an unencrypted local database column, or a plain file on disk — none of these are encrypted at rest by default in a way appropriate for secrets, and on a rooted/jailbroken device (or via a backup-extraction attack in some configurations) that data is readily accessible.
- **DO:** Choose an appropriate Keychain accessibility level (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` is a reasonable default for most app secrets — accessible after first unlock, and excluded from device-to-device migrating backups) rather than defaulting to the most permissive option without considering the tradeoff between convenience (background access) and exposure.
- **DON'T:** Roll a custom, homegrown encryption scheme for local secure storage instead of using the platform's provided secure storage APIs — hand-rolled encryption is a well-documented source of subtle, exploitable mistakes (weak key derivation, a hardcoded or predictable key, an insecure IV/nonce reuse), and both platforms' built-in secure storage already solves this correctly.
- **DO:** In React Native and Flutter, use a library that wraps the native secure storage on each platform (`react-native-keychain`/`expo-secure-store` for RN, `flutter_secure_storage` for Flutter) rather than a cross-platform key-value package that only wraps plain, non-secure storage under a name that sounds secure — verify what a given package actually does on each platform before trusting it with secrets.

### Battery and Network Usage Discipline

- **DO:** Batch and coalesce network requests where possible, and use appropriate polling intervals (or better, push/webhook-driven updates instead of polling at all) rather than polling a server every few seconds for data that changes infrequently — frequent network activity is one of the largest drivers of both battery drain and cellular data usage, and both platforms' battery-usage breakdowns surface an app's network activity prominently to users.
- **DON'T:** Keep GPS/high-accuracy location updates running continuously in the background when a coarser accuracy or a significant-location-change/geofencing-based approach would satisfy the actual feature need — continuous high-accuracy GPS is one of the single largest battery drains an app can cause, and both platforms specifically call this out in their battery-usage attribution to users, who will notice and often uninstall the app.
- **DO:** Respect system-level power-saving states (Low Power Mode on iOS, Battery Saver/Doze on Android) by deferring non-critical background work when the system reports a reduced-power state (`ProcessInfo.processInfo.isLowPowerModeEnabled` on iOS; Doze-mode-aware `WorkManager` constraints on Android), rather than continuing to poll, sync, or run background work at full frequency regardless of device power state.
- **DON'T:** Download or sync large media/data payloads over cellular by default without either a user preference to control it ("Wi-Fi only" sync) or at least a size/frequency check appropriate to the content — an app that silently burns through a user's cellular data allowance is a well-documented driver of uninstalls and negative reviews.
- **DO:** Use each platform's network-condition APIs (`NWPathMonitor` on iOS, `ConnectivityManager`/`NetworkCallback` on Android, `NetInfo` in React Native, `connectivity_plus` in Flutter) to detect connection type and quality, and adapt behavior accordingly — deferring large downloads on a metered connection, reducing image quality on a slow connection, or queuing writes until connectivity improves.

## Common AI-Assistant Mistakes in Mobile Code

### Inventing SDK/API Methods That Don't Exist for a Given OS Version

- **DO:** Verify that a suggested API, method, or modifier actually exists in the target platform's SDK, and at the actual minimum-deployment-target/`minSdkVersion` the project supports, before generating code that calls it — check the platform's current official API reference or the project's actual imports/available symbols rather than recalling a plausible-sounding method name from general training pattern-matching.
- **DON'T:** Generate a SwiftUI modifier, a UIKit method, a Jetpack Compose function, or an Android API call that sounds plausible (matches the naming conventions of real APIs) but doesn't actually exist in the SDK, or that exists only in a newer OS version than the project's deployment target supports. This is one of the single most common and most damaging categories of AI-generated mobile code mistakes, because it compiles-looking-plausible in a description but fails at build time (or, worse, only at runtime for an `@available`/`SDK_INT`-unguarded call on an older device) — always cross-check the exact API surface for the exact SDK version in use, and flag explicitly when unsure rather than presenting invented API usage with confidence.
```swift
// Invented / unverified: don't assume a modifier like this exists without checking
.someModifierNameThatSoundsRight(true)

// Instead: verify against the actual current SwiftUI API reference for the target OS version,
// or check the project's own code for the real modifier already in use
```

### Ignoring Platform Human Interface Guidelines and Producing Identical UI on Both Platforms

- **DO:** Explicitly consider each platform's interaction and visual conventions (see the Platform-Specific UI Adaptation section above) when generating cross-platform UI code, rather than defaulting to whichever platform's conventions dominate general training data (often iOS-leaning, or a generic web-influenced "flat design" that matches neither platform well) and applying it uniformly to both.
- **DON'T:** Generate a settings screen, a form, an alert dialog, or a tab bar that looks identical on iOS and Android without at least flagging that platform-specific adaptation might be wanted — copy-pasting one platform's native-feeling component pattern onto the other platform is a frequent, easy-to-miss mistake specifically because the code "works" (renders, functions correctly) while still looking wrong to that platform's users.

### Forgetting Permission Usage Descriptions

- **DO:** Whenever generating code that calls a privacy-sensitive API (camera, location, contacts, photo library, microphone, Bluetooth, health data, tracking), also generate or explicitly call out the required `Info.plist` usage-description entry (iOS) or the required runtime-permission request flow with rationale (Android) in the same response — these two pieces (the API call and its permission declaration) are functionally paired and one without the other produces broken or crashing code.
- **DON'T:** Generate a call to `CLLocationManager`, `AVCaptureSession`, `PHPickerViewController`, Android's `LocationManager`/CameraX, or similar, without also mentioning the required permission entry/request — omitting it produces code that crashes at runtime (iOS, missing `Info.plist` key) or silently fails/denies (Android, missing runtime request), and a reviewer skimming the generated code for correctness may not notice the omission until it's actually run on a device.

### Blocking the Main/UI Thread

- **DO:** Default to placing network calls, file I/O, database queries, and any non-trivial computation off the main/UI thread in generated code, using each platform's standard async mechanism (Swift concurrency `async`/`await`, Kotlin coroutines, RN's inherently-async bridge calls handled correctly, Dart's `Future`/`async`/`await` with heavy work moved to an `Isolate` when appropriate) as the default pattern, not an afterthought added only when asked.
- **DON'T:** Generate a "quick example" that performs a synchronous network call or heavy computation directly in a UI event handler, a `build()`/`body` method, or a list-cell binding method, on the assumption that thread-safety concerns can be addressed later. This exact pattern — starting with a blocking, main-thread version "to keep it simple" — is disproportionately likely to be copied as-is into real production code without the follow-up fix, so generate the off-main-thread version by default.

### Not Handling Offline/Poor-Network States

- **DO:** Include explicit loading, error, empty, and offline/stale-data states any time generated code fetches remote data — the same discipline expected of well-written frontend code generally, applied to mobile's added dimension of genuinely frequent connectivity loss (subway commutes, elevators, rural coverage gaps, airplane mode) that a purely web-oriented mental model can under-account for.
- **DON'T:** Generate a data-fetching screen that only handles the success path (render the data) with no visible handling for a failed request, a timeout, or a completely offline device — an implicit assumption of reliable connectivity is a systematic bias in generated mobile code, and it's specifically mobile networking's much higher rate of real-world transience (versus a developer's usually-stable desktop/office network while writing and testing the code) that makes this gap surface in production far more than it does in a quick manual test.

### Hardcoding Screen Dimensions

- **DO:** Default generated layout code to relative, constraint-based, or flexible sizing (percentage/weight-based Flexbox in RN, `Expanded`/`Flexible`/`MediaQuery`-relative sizing in Flutter, `GeometryReader`/adaptive stacks in SwiftUI, `ConstraintLayout`/`Modifier.weight` in Compose) rather than a fixed pixel/point/dp value copied from whatever device happened to be used as a mental reference point while generating the example.
- **DON'T:** Hardcode a specific screen width/height (e.g., `width: 375` matching one specific iPhone model, or a fixed dp value assuming one Android reference device) into generated layout code presented as general-purpose — this produces UI that looks correct in whatever preview/simulator matches that specific assumption and breaks (overflows, clips, leaves excessive whitespace) on every other currently-shipping screen size, which is a large and clearly-defined set the code should be checked against, not assumed away.

## Quick Checklist
- Pick MVVM as the default iOS/Android architecture for most apps; reserve TCA/Redux-style unidirectional-flow frameworks for apps with genuinely complex, deeply shared cross-screen state.
- Don't let a `UIViewController`, SwiftUI `View`, `Activity`, or `Fragment` accumulate networking, persistence, and business logic directly — extract it into a ViewModel/repository layer.
- Default new iOS work to SwiftUI and new Android work to Compose; bridge to UIKit/XML views only for specific capability gaps, not wholesale rewrites of stable screens.
- Use `NavigationStack` with a typed path (iOS) and the Navigation Component/Compose Navigation (Android) instead of ad-hoc boolean-driven presentation state.
- Use `UserDefaults`/`SharedPreferences`/DataStore only for small preferences; use SwiftData/Core Data/Room for structured, queryable local data.
- Never store auth tokens, passwords, or secrets in plain `UserDefaults`/`SharedPreferences`/an unencrypted file — use Keychain (iOS) or Keystore-backed encrypted storage (Android).
- Don't run database queries, file I/O, or heavy computation on the main/UI thread on either platform.
- Design around each OS's real background-execution limits (`BGTaskScheduler`/background URL sessions on iOS; WorkManager, with Foreground Service reserved for user-visible ongoing work, on Android) — don't assume arbitrary background execution.
- Request notification/runtime permissions contextually, at point of use, with a clear rationale — never batch every permission request at first launch.
- Add every required `Info.plist` usage-description key before calling a privacy-sensitive API on iOS, or the app crashes at runtime.
- Handle every permission state explicitly (granted, denied, permanently denied, one-time-granted-then-revoked on Android 11+) with a path to Settings when needed.
- Provide working demo credentials/demo mode for App Review; never ship placeholder content, broken links, or Lorem Ipsum in a submitted build.
- Route digital-goods purchases through Apple IAP / Google Play Billing; don't link out to an external payment page for purely digital content.
- Support in-app account deletion if the app supports account creation (Apple requirement).
- Don't request `QUERY_ALL_PACKAGES`, `MANAGE_EXTERNAL_STORAGE`, or broad background location without a Play-policy-compliant justified use case.
- Fill out the Play Console Data Safety section accurately, including third-party SDK data collection, not just first-party.
- Downsample images to their display size and use a real caching library (Kingfisher/Nuke/SDWebImage, or Coil/Glide on Android) rather than re-decoding full-resolution images on every reuse.
- Use `RecyclerView`/`LazyColumn`/`FlatList`/`ListView.builder` with stable keys/DiffUtil for any list that can grow — never render an unbounded list inside a plain scroll container.
- Give `LazyColumn`/`LazyRow` items and RN `FlatList` a stable `key`/`keyExtractor`; never use an array index for a reorderable/mutable list.
- Scope Compose/Flutter state to the smallest subtree that needs it; use `const` constructors in Flutter and avoid hoisting `setState` above where it's needed.
- Treat the local database (SwiftData/Core Data/Room) as the single source of truth the UI reads from, syncing from the network in the background — don't read directly from transient network responses.
- Persist offline write queues durably and make them idempotent; never keep a pending-writes queue only in memory.
- Surface connectivity/staleness state honestly in the UI ("offline — showing saved data," a last-synced timestamp) rather than silently failing or showing a blank screen.
- Guard any API introduced after the app's `minSdkVersion`/deployment target with an explicit version check (`SDK_INT`, `@available`) or a backport library.
- Use `dp`/`sp` (Android) and points/Dynamic Type-relative sizing (iOS), never hardcoded pixel values, for layout and text.
- Test on the smallest and largest supported screen sizes, at least one tablet/large-screen and one foldable posture if supported, and both orientations.
- Wrap native SDK integrations behind a narrow, well-typed module boundary (Turbo/Native Modules in RN, platform channels in Flutter); don't scatter ad-hoc platform calls through shared code.
- Don't send large or high-frequency payloads across the JS-native bridge/platform channel; batch or move the hot path fully native.
- Adapt navigation chrome, dialogs, and control styling per platform (Cupertino vs. Material) rather than shipping one identical design on both.
- Choose a state-management approach proportional to actual app complexity; use a dedicated data-fetching/caching layer for server data instead of ad hoc per-screen fetch logic.
- Budget separate App Store and Play Store submission review cycles, listing requirements, and compliance checks — passing one says nothing about the other.
- Test actual release-mode builds (not just debug) on both platforms before submission.
- Use staged/phased rollouts for meaningful releases and watch crash rate before expanding to 100%.
- Reserve forced/blocking updates for genuinely breaking changes; keep specific, non-generic release notes.
- Integrate crash reporting from day one, symbolicate reports (dSYMs/ProGuard mapping), and treat crash-free rate as a tracked metric.
- Never log PII, tokens, or payment data into crash reports or analytics events.
- Watch for crash signatures concentrated on one device/OS/app-version combination, not just overall crash percentage.
- Give every interactive element a real accessibility label; never leave icon-only controls unlabeled.
- Support Dynamic Type / Android font scale; don't hardcode text sizes or disable system text scaling without strong justification.
- Size touch targets to at least 44×44pt (iOS) / 48×48dp (Android), even for visually small icons.
- Never convey state through color alone; pair it with icon/text/shape.
- Test both platforms with a real screen reader (VoiceOver/TalkBack) on critical flows, not just an automated scanner.
- Provide an accessible alternative for any custom-gesture-only interaction.
- Implement verified universal links/App Links (domain-association files), not just an unverifiable custom URL scheme, for links that matter.
- Handle deep links arriving via cold launch, warm foreground, and background resume — all three, not just cold launch.
- Validate deep-linked content still exists and the user is authorized before routing into it.
- Respect Low Power Mode / Battery Saver and avoid continuous high-accuracy GPS or tight polling loops when a coarser or event-driven approach suffices.
- Don't silently burn cellular data on large downloads/syncs without a Wi-Fi-only preference or size-awareness.
- Never hand-roll local encryption for secrets — use Keychain/Keystore-backed APIs (or `flutter_secure_storage`/`react-native-keychain`/`expo-secure-store` cross-platform).
- Verify every generated SDK/API call actually exists for the target OS version and deployment target before presenting it as correct — don't pattern-match a plausible-sounding method name.
- Pair every privacy-sensitive API call in generated code with its required permission declaration/request in the same response.
- Default generated data-fetching UI to include loading, error, empty, and offline states — never generate only the happy path.
- Default generated layout code to relative/flexible sizing; never hardcode a screen dimension from one reference device.
