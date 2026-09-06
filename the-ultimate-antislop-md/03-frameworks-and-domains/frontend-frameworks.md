# Frontend Frameworks & Web Accessibility

## React

### Hooks Discipline

- **DO:** Call hooks only at the top level of a function component or custom hook, never inside conditionals, loops, or nested functions. React matches hook state to call order across renders, so any conditional call shifts every hook after it and corrupts state silently.
```jsx
// BAD — hook called conditionally
function Profile({ userId }) {
  if (userId) {
    const [name, setName] = useState('');
  }
  // ...
}

// GOOD — hook always called, condition moved inside
function Profile({ userId }) {
  const [name, setName] = useState('');
  if (!userId) return null;
  // ...
}
```
- **DO:** Extract repeated stateful logic into a custom hook once it appears in two or more components, and name it with a `use` prefix. This keeps components declarative and lets the linter verify hook rules for the extracted logic.
- **DON'T:** Name a function `useFoo` unless it actually calls other hooks internally. A `use`-prefixed name that doesn't call hooks misleads readers and the `eslint-plugin-react-hooks` rules, which special-case anything matching that naming pattern; use a plain verb name (`getFoo`, `formatFoo`) for non-hook helpers instead.
- **DO:** Keep custom hooks focused on one concern (one piece of state, one subscription, one side effect family) rather than bundling unrelated state into a single mega-hook. A hook called `useDashboardEverything` that manages filters, websockets, and pagination together forces every consumer to opt into all of it and makes the hook untestable in isolation; split it into `useFilters`, `useLiveUpdates`, and `usePagination`.
- **DON'T:** Reach for `useState` to store a value that can be derived from existing props or state during render. Derived values duplicated into state drift out of sync and require extra effects to keep them updated; compute the value inline during render (optionally memoized with `useMemo` if the computation is expensive) instead.
```jsx
// BAD — fullName is derived state that can go stale
const [fullName, setFullName] = useState(`${first} ${last}`);
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// GOOD — computed directly during render
const fullName = `${first} ${last}`;
```
- **DO:** Use the lazy initializer form of `useState` (`useState(() => expensiveInit())`) when the initial value requires real computation. Passing a plain expensive call directly runs it on every render even though only the first result is used.
- **DON'T:** Store objects or arrays in state and mutate them in place before calling the setter. React compares state by reference for `useState`/`useReducer`, so mutating and then passing the same reference back skips the re-render entirely; always create a new object/array (spread, `map`, `filter`, `concat`) when updating.
```jsx
// BAD — mutates in place, React sees the same reference
function addItem(item) {
  items.push(item);
  setItems(items); // no re-render, reference unchanged
}

// GOOD — new array reference
function addItem(item) {
  setItems(prev => [...prev, item]);
}
```
- **DO:** Prefer the updater-function form of `setState` (`setCount(c => c + 1)`) whenever the next value depends on the previous one, especially inside effects, event handlers that may batch, or closures captured by timers. Reading the outer variable directly risks acting on a stale snapshot when multiple updates are queued in the same tick.
- **DON'T:** Split one logical piece of state into many separate `useState` calls that must always change together. Several independently-updated booleans and strings that represent one status (`isLoading`, `isError`, `data`, `errorMessage`) invite impossible combinations (loading and error both true); model them as one `useReducer` or a single discriminated-union state value instead.
```jsx
// BAD — four independent booleans/values that can desync
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [data, setData] = useState(null);
const [error, setError] = useState(null);

// GOOD — one state shape, impossible states are unrepresentable
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Data }
  | { status: 'error'; error: string };
const [state, dispatch] = useReducer(reducer, { status: 'idle' });
```
- **DO:** Use `useReducer` instead of a pile of `useState` calls once a component has several related state transitions with non-trivial logic (multi-step forms, undo/redo, state machines). Centralizing the transition logic in a reducer function makes the valid transitions explicit and testable outside of React.
- **DON'T:** Call `useContext` for state that only one or two components actually need. Wrapping a value in context "for later" adds an indirection layer and a re-render surface that outlives its justification; pass props directly until multiple, non-adjacent components genuinely need the same value.
- **DO:** Split a single large context into several smaller, purpose-specific contexts (e.g., `AuthContext`, `ThemeContext`, `CartContext`) rather than one `AppContext` object holding everything. Every consumer of a context re-renders on any change to that context's value, so a monolithic context makes an unrelated theme toggle re-render the entire authenticated app tree.
- **DON'T:** Put a context provider's value inline as a new object literal on every render (`<Ctx.Provider value={{ user, setUser }}>`) without memoizing it. A fresh object identity on every render defeats `React.memo` on every consumer and forces them all to re-render even when the actual data didn't change; wrap the value in `useMemo`.
```jsx
// BAD — new object every render, breaks memoized consumers
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

// GOOD — stable reference unless user actually changes
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```
- **DO:** Use `useRef` for values that must persist across renders but should not trigger a re-render when they change (DOM node handles, previous-value tracking, interval/timeout IDs, mutable instance variables). Using `useState` for these forces unnecessary render cycles for bookkeeping that has no visual effect.
- **DON'T:** Read or write `ref.current` during render to drive rendering logic. Refs are explicitly exempt from React's render-consistency guarantees (their mutation doesn't schedule a re-render and can violate Strict Mode's double-invocation assumptions); read refs only in effects and event handlers, never in the render body, to decide what to display.
- **DO:** Follow the Rules of Hooks with the official ESLint plugin (`eslint-plugin-react-hooks`) enabled and treated as a build-blocking error, not a warning to be silenced. Hand-verifying hook order and dependency correctness on every review does not scale and the plugin catches the exact classes of bugs described above.
- **DON'T:** Wrap `useState`/`useReducer` setters or context values in `useCallback`/`useMemo` "just in case" when nothing downstream is memoized or expensive. Setter functions from `useState` are already stable across renders, and wrapping cheap values in memoization hooks adds a dependency-array maintenance burden and a small overhead for zero benefit; memoize only what actually needs a stable identity or avoids real recomputation cost.
- **DO:** Colocate state as close as possible to the components that use it, and lift it only when a sibling or ancestor genuinely needs to read or write it. State declared at the top of the tree "to be safe" causes every child down to the leaf to re-render on every update, even ones that never read that state.
- **DON'T:** Build custom hooks that silently swallow errors from async work inside a `try { } catch {}` with no rethrow, no error state, and no logging. A hook that eats exceptions makes failures invisible to both the UI and to error-tracking tools; surface the error through returned state (e.g., `{ data, error, isLoading }`) or rethrow it for an error boundary to catch.

### Avoiding Unnecessary Re-Renders

- **DO:** Understand that a re-render is triggered by a state change, a parent re-render, or a context value change — not by a prop merely "looking similar." Every child of a re-rendered parent re-renders by default unless it is wrapped in `React.memo` and its props are referentially stable.
- **DON'T:** Reach for `React.memo`, `useMemo`, and `useCallback` reflexively on every component and value before measuring anything. Memoization has its own cost (extra comparisons, extra memory, extra code to maintain) and on cheap components it can make things slower, not faster; profile with React DevTools first and memoize the components and values that actually show up as expensive or that actually break prop-identity checks downstream.
- **DO:** Memoize a component with `React.memo` when it is expensive to render and its parent re-renders often for reasons unrelated to that component's props. This skips the child's render entirely when its props are shallow-equal to the previous render.
```jsx
// Only worth doing if ExpensiveChart is costly to render
const ExpensiveChart = React.memo(function ExpensiveChart({ data }) {
  return <svg>{/* heavy drawing work */}</svg>;
});
```
- **DON'T:** Wrap a component in `React.memo` and then pass it inline arrow functions or object/array literals as props from the parent on every render. `React.memo`'s shallow comparison sees a new function/object identity every time and re-renders anyway, so the memoization is dead weight; stabilize those props with `useCallback`/`useMemo` in the parent, or restructure so the memoized component doesn't need them.
```jsx
// BAD — memo is defeated by a fresh inline handler every render
<MemoizedRow onSelect={() => selectRow(row.id)} />

// GOOD — stable callback identity
const handleSelect = useCallback((id) => selectRow(id), [selectRow]);
<MemoizedRow onSelect={handleSelect} rowId={row.id} />
```
- **DO:** Push state down into the smallest subtree that needs it instead of hoisting it to a shared ancestor. Moving a text-input's local state out of a giant form component and into a small `SearchBox` child means keystrokes only re-render `SearchBox`, not the entire page.
- **DON'T:** Put fast-changing state (scroll position, mouse coordinates, animation frame values, input-in-progress text) in a context or a high-level store when only a small, localized part of the UI needs it. Broadcasting high-frequency updates through context re-renders every consumer on every tick; keep such state local to the component that owns the visual feedback, or use a ref plus imperative DOM updates for animation-grade frequency.
- **DO:** Use the `children` prop (composition) to keep a slow-changing wrapper component from re-rendering fast-changing content, and vice versa. Content passed as `children` is created by the parent that owns it and is not re-created just because the wrapping component re-renders for its own reasons.
```jsx
// Layout re-renders on its own state changes, but `children`
// was created by App and is untouched by Layout's re-render.
function Layout({ children }) {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  return <div className={sidebarOpen ? 'with-sidebar' : ''}>{children}</div>;
}
```
- **DO:** Use the React DevTools Profiler (or the "Highlight updates when components render" setting) to confirm which components actually re-render before optimizing, rather than guessing from reading the code. Intuition about render behavior is frequently wrong, especially around context and memo boundaries.
- **DON'T:** Assume `useMemo`/`useCallback` guarantee referential stability across renders in all React versions and configurations. They are performance hints, not semantic guarantees; React may in some modes discard a cached value and recompute it (for example under experimental memory-pressure invalidation), so code must remain correct even if the memoized value is recomputed, never rely on it for correctness (like skipping a required side effect).
- **DO:** Move expensive, purely-computational work (large array transforms, aggregations, formatting of big datasets) into `useMemo` keyed on its actual inputs, once profiling shows it's a bottleneck. This avoids repeating O(n) or worse work on every render that doesn't change the inputs.
- **DON'T:** Give `useMemo`/`useCallback` an incomplete or dishonest dependency array to "keep the value stable" while covering up that its inputs really did change. This produces stale closures and stale computed values that silently diverge from the current props/state; if a value must stay stable across an input change, restructure the state (e.g., a ref for values that intentionally shouldn't trigger recomputation) rather than lying to the dependency array.
- **DO:** Consider the `key` prop as a re-mounting tool: changing a component's `key` forces React to discard its entire state and DOM and mount a fresh instance. Use this deliberately when you want a full reset (e.g., resetting a form when navigating between different records) instead of writing an effect to manually reset each field.
```jsx
// GOOD — changing the key forces a clean remount per user
<EditProfileForm key={user.id} user={user} />
```
- **DON'T:** Split state across too many sibling components purely to "optimize re-renders" when it fragments a cohesive interaction into hard-to-follow pieces. Rendering performance work should follow a demonstrated problem (jank, dropped frames, slow interactions); premature fragmentation trades a small, hypothetical render-time saving for real, ongoing readability and maintenance cost.

### Key Prop Misuse

- **DO:** Use a stable, unique identifier from the data itself (a database id, a UUID, a slug) as the `key` for items in a list. Keys let React match array items across renders so it can preserve state, DOM nodes, and avoid needless remounts.
- **DON'T:** Use the array index as `key` for a list that can be reordered, filtered, inserted into, or removed from. Index keys tie each key to a position, not to an item, so React attaches the wrong internal state (uncontrolled input values, animation state, focus) to the wrong data when the list changes shape.
```jsx
// BAD — index key breaks when items are removed/reordered
{todos.map((todo, i) => <TodoRow key={i} todo={todo} />)}

// GOOD — stable identity key
{todos.map((todo) => <TodoRow key={todo.id} todo={todo} />)}
```
- **DO:** Accept index keys only for lists that are provably static for the lifetime of the component — never filtered, sorted, reordered, spliced, or fed by paginated/streamed data — and say so in a comment, because the exception is easy to invalidate later by an innocuous feature change.
- **DON'T:** Generate a new random key on every render (`key={Math.random()}` or `key={crypto.randomUUID()}` computed inline during render). A key that changes every render forces React to unmount and remount every list item on every render, destroying all internal state, losing focus, and re-triggering mount effects — this is strictly worse than no memoization at all.
```jsx
// BAD — new key every render forces full remount of every row
{items.map((item) => <Row key={Math.random()} item={item} />)}
```
- **DO:** Derive keys from content-independent identity, not from data that can collide (e.g., don't key by `item.name` when two items can share a name; prefer `item.id`). Duplicate keys make React fall back to unpredictable matching and log a warning that is easy to miss in noisy console output.
- **DON'T:** Ignore the "Each child in a list should have a unique key prop" console warning as cosmetic. It signals a real correctness risk (state bleeding between rows, wrong elements animating, incorrect focus retention) that often only manifests visibly under specific reorder/filter interactions that may not be covered by manual testing.
- **DO:** Put the `key` on the outermost element returned from the `.map()` callback (or on the component invocation itself), not on some nested child. React only reads `key` at the position where it iterates the array of elements; a key placed deeper has no effect on reconciliation of the list.
- **DON'T:** Use `key` as a general-purpose prop to pass data into the component. `key` is stripped by React before the component receives its props (it is not accessible via `props.key`); if the child needs that value, pass it again under a different prop name.

### useEffect Dependency Array Pitfalls

- **DO:** Include every reactive value used inside an effect (props, state, and any value derived from them, including functions and objects from the surrounding scope) in the dependency array. The dependency array tells React when the effect's closure has gone stale; omitting a used value means the effect keeps operating on an old snapshot of it.
```jsx
// BAD — `userId` is used but missing from deps: effect runs once
// with the userId from the first render only.
useEffect(() => {
  fetchUser(userId).then(setUser);
}, []);

// GOOD — effect re-runs whenever userId changes
useEffect(() => {
  fetchUser(userId).then(setUser);
}, [userId]);
```
- **DON'T:** Add a dependency array item purely to silence the `exhaustive-deps` ESLint rule without understanding why the value is needed, and without checking whether adding it changes how often the effect runs in a way that breaks behavior (e.g., an object recreated every render now causes the effect to run every render). Fix the actual cause — stabilize the value with `useMemo`/`useCallback`, move it outside the component, or restructure the effect — rather than reflexively appending to the array.
- **DO:** Enable `eslint-plugin-react-hooks`'s `exhaustive-deps` rule and treat its warnings as bugs to fix, not suggestions to override. Manually auditing every effect's closure for missed dependencies does not scale across a codebase and this is exactly the failure mode that produces stale-closure bugs in production.
- **DON'T:** Disable `exhaustive-deps` with an inline `// eslint-disable-next-line` as a default habit to "run this effect only once." An effect that intentionally must skip a dependency is a signal that the logic doesn't actually belong in that effect shape; use a ref to read the latest value without re-triggering the effect, split the effect, or move the logic to an event handler instead of suppressing the lint rule.
```jsx
// BAD — silences the warning instead of fixing the design
useEffect(() => {
  logPageView(pageName, user);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);

// GOOD — read the latest value via a ref without re-running on every change
const userRef = useRef(user);
useEffect(() => { userRef.current = user; });
useEffect(() => {
  logPageView(pageName, userRef.current);
}, [pageName]);
```
- **DO:** Return a cleanup function from any effect that subscribes to something external (event listeners, WebSocket connections, timers, observers, third-party library instances). Without cleanup, every re-run of the effect adds another subscription on top of the previous one, leaking listeners and eventually firing callbacks multiple times per event.
```jsx
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```
- **DON'T:** Assume an effect runs only once just because its dependency array is `[]`. In React 18+ Strict Mode (development only), effects with mount/cleanup are intentionally invoked twice (mount, cleanup, mount again) to surface exactly these missing-cleanup bugs; treat a double-invocation warning as a real bug in the effect, not a Strict Mode quirk to ignore.
- **DO:** Split one effect that does two unrelated things into two separate effects, each with its own accurate dependency array. A single effect that both syncs a document title and subscribes to a WebSocket forces one shared dependency list that is wrong for at least one of the two responsibilities.
- **DON'T:** Use an object or array literal created inline as a dependency without memoizing it, then wonder why the effect re-runs on every render. `[]`-created or `{}`-created values are a new reference every render, so a dependency array containing one always fails the `Object.is` comparison and re-triggers the effect every time; memoize the value or depend on its primitive fields instead.
```jsx
// BAD — options object is new every render, effect fires every render
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal, ...options });
  return () => controller.abort();
}, [url, options]); // options = {} literal from parent, new identity each time

// GOOD — depend on primitive fields instead of the object identity
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal, method: options.method });
  return () => controller.abort();
}, [url, options.method]);
```
- **DO:** Guard async work inside effects against race conditions when the effect can re-run before the previous async call resolves (e.g., rapid prop changes triggering repeated fetches). Use a cancellation flag, `AbortController`, or check that the effect instance is still the latest before applying the result.
```jsx
useEffect(() => {
  let cancelled = false;
  fetchResults(query).then((data) => {
    if (!cancelled) setResults(data);
  });
  return () => { cancelled = true; };
}, [query]);
```
- **DON'T:** Write an async function directly as the effect callback (`useEffect(async () => {...}, [])`). `useEffect`'s callback must return either `undefined` or a cleanup function; an `async` function returns a Promise instead, and React will log a warning and never treat the Promise as cleanup. Define the async logic as an inner function and invoke it, or use a separate named async helper.
```jsx
// BAD — effect callback returns a Promise, not a cleanup function
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// GOOD — async logic wrapped inside the effect
useEffect(() => {
  let cancelled = false;
  (async () => {
    const data = await fetchData();
    if (!cancelled) setData(data);
  })();
  return () => { cancelled = true; };
}, []);
```
- **DO:** Ask, before writing any `useEffect`, whether the logic actually needs to happen in response to a commit to the DOM, or whether it's really just derived state or an event response that belongs in the render body or an event handler. A large share of `useEffect` calls in AI-generated and junior code are effects that shouldn't exist at all — the React docs' "You Might Not Need an Effect" guidance is the right first filter.
- **DON'T:** Use an effect to update one piece of state in response to another piece of state changing in the same component, when the second value could instead be computed directly during render. This adds an extra render pass (state changes, effect fires, state changes again, re-render) for something that could have been done in zero extra renders.
```jsx
// BAD — an effect just to recompute a derived value
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);
useEffect(() => {
  setTotal(items.reduce((sum, i) => sum + i.price, 0));
}, [items]);

// GOOD — computed inline, no effect, no extra render
const [items, setItems] = useState([]);
const total = items.reduce((sum, i) => sum + i.price, 0);
```
- **DO:** Reset component state in response to a prop change by changing the component's `key` (forcing a remount) instead of writing an effect that manually resets every piece of state when the prop changes. The `key` approach is fewer moving parts and cannot miss a field.
- **DON'T:** Chain multiple effects that each set state to trigger the next effect, forming an implicit waterfall (effect A sets state X, which triggers effect B, which sets state Y, which triggers effect C). This is hard to trace, causes multiple unnecessary render passes, and usually indicates the whole sequence should be one event handler or one effect with an inlined async function instead.
- **DO:** Put logic that only needs to happen in response to a specific user action (form submission, button click, item selection) directly in the event handler, not in an effect that watches for the resulting state change. Event handlers know exactly why they're running; effects that infer "the user probably just did X" from a state diff are indirect and fragile, and can also misfire on unrelated causes of the same state change (e.g., a page refresh or programmatic reset).

### State Management Choices (React)

- **DO:** Default to local component state (`useState`/`useReducer`) and lift it only as far as the nearest common ancestor that actually needs it, before reaching for a global store. Most state in a typical UI is local — a dropdown's open/closed flag, a form field's current value, a tooltip's visibility — and promoting it to global state adds indirection with no benefit.
- **DON'T:** Put server-derived data (API responses, anything fetched over the network) into the same global client-state store as UI state (theme, sidebar open, active tab). Server data has fundamentally different needs — caching, staleness, revalidation, request deduplication, retries — that a generic client-state library doesn't provide out of the box; use a dedicated data-fetching/caching library (e.g., React Query / TanStack Query, SWR, or RTK Query) for server state and a lightweight store only for genuine client/UI state.
- **DO:** Choose a state-management approach based on the shape of the problem: Context for rarely-changing, broadly-shared values (theme, locale, auth session); a dedicated store library (Zustand, Redux Toolkit, Jotai, Valtio) for frequently-updated state shared across distant parts of the tree; and a server-state library for anything that originates from the network. Picking one tool for every kind of state (e.g., forcing all server data through Redux with hand-written thunks) reproduces problems those specialized libraries already solved.
- **DON'T:** Introduce Redux (or any global store) for a small app or a single feature with no cross-cutting state-sharing need, purely out of habit or resume-driven development. The boilerplate (actions, reducers, selectors, middleware wiring) is a real ongoing cost that should be justified by an actual sharing/consistency problem, not adopted preemptively.
- **DO:** Normalize collections of entities (by id, in a flat map) when the same entity can be read or updated from multiple places in the store, rather than duplicating copies of the same entity nested inside different arrays. Duplicated, un-normalized entities drift out of sync the moment one copy is updated and the others aren't.
- **DON'T:** Store values in global state that are trivially derivable from other state already in the store (a filtered list, a count, a sum). Store the source data and compute derived values with a selector (`useMemo`, `reselect`, or the store library's own selector mechanism); storing derived values invites the same "forgot to update all copies" bug as duplicated entities.
- **DO:** Use selectors (or per-slice subscriptions in libraries that support them) so components subscribe only to the specific store fields they use, not the entire store object. Subscribing to the whole store means any unrelated change anywhere re-renders the component.
```jsx
// BAD — re-renders on any store change, not just cartTotal
const store = useStore();
const total = store.cart.total;

// GOOD — component only re-renders when cartTotal actually changes
const total = useStore((state) => state.cart.total);
```
- **DON'T:** Mix state-management paradigms inconsistently within the same feature — some data in Context, some in Redux, some in a third ad-hoc singleton module with module-level `let` variables, all managing overlapping concerns. Pick one primary approach per category of state (local, shared client, server) and apply it consistently so a new contributor can predict where a given piece of state lives.
- **DO:** Prefer colocated state (context or a store scoped to a route/feature, created and torn down with it) over one application-wide global store when the state is genuinely feature-scoped (a multi-step wizard's in-progress data, a data-grid's column configuration). Scoped state avoids leaking memory and stale data once the user navigates away, and keeps the feature independently testable.
- **DON'T:** Reach for `useSyncExternalStore` or hand-rolled subscription mechanisms to work around a state library's limitations before checking whether the library already exposes the primitive needed (fine-grained selectors, `useShallow`, computed/derived atoms). Duplicate, hand-rolled synchronization logic tends to miss edge cases (tearing during concurrent rendering) that the library's own primitives were built to handle.

### Component Composition vs Prop Drilling

- **DO:** Recognize prop drilling — passing a prop down through three or more intermediate components that don't use it themselves, just to forward it — as a design smell, not a fact of life to always tolerate. It couples every intermediate component to a prop it doesn't care about and makes refactoring the shape of that data a change that touches many unrelated files.
```jsx
// BAD — `user` threads through Layout and Sidebar, unused by either,
// only to be read by ProfileCard three levels down.
<Layout user={user}>
  <Sidebar user={user}>
    <ProfileCard user={user} />
  </Sidebar>
</Layout>
```
- **DO:** Use the `children` prop (or other "slot"-style props) to let a component receive fully-formed subtrees instead of receiving raw data and rendering it itself. This removes the need for intermediate wrapper components to know about props they only need to relay to something further down.
```jsx
// GOOD — Layout and Sidebar know nothing about `user`
<Layout>
  <Sidebar>
    <ProfileCard user={user} />
  </Sidebar>
</Layout>
```
- **DO:** Reach for Context specifically when a value is needed by many components at different depths within one bounded subtree (a form's shared validation state, a modal's shared open/close controller), not as a blanket replacement for props at the first sign of two or three levels of drilling. Two or three levels of an explicit, typed prop is often more readable and traceable than an implicit context lookup.
- **DON'T:** Default to Context for every case of prop drilling without considering composition first. Composition (passing components as props/children) keeps data flow explicit and traceable through JSX structure; Context makes the data source implicit, which is harder to trace from a consuming component back to its origin, and re-renders every consumer on value change unless carefully split and memoized.
- **DO:** Design components to accept a small number of well-named, cohesive props (or a single well-typed options object for genuinely related options) rather than an ever-growing list of loosely related boolean flags and one-off overrides. A component with fifteen independent boolean props has an exponential number of untested visual/behavioral states.
```tsx
// BAD — flag soup, most combinations never tested
<Button primary large disabled loading outline rounded compact iconOnly />

// GOOD — a small set of intentional, mutually-exclusive variants
<Button variant="primary" size="lg" state="loading" />
```
- **DON'T:** Build a "god component" that owns layout, data fetching, business logic, and presentation all in one file for an entire page or feature. Split it into a container that owns data/state and presentational components that only receive props and render UI, so each piece can be tested, reused, and reasoned about independently.
- **DO:** Use render props or component-as-prop patterns (or hooks, in modern React) to share cross-cutting *behavior* (not just markup) between components, when the behavior needs to control what gets rendered based on internal state the consumer doesn't have direct access to. In current React, a custom hook is usually the more idiomatic and less deeply-nested way to share this kind of logic than a render-prop wrapper.
- **DON'T:** Nest multiple render-prop or higher-order-component wrappers (`withAuth(withTheme(withRouter(Component)))`) when the same cross-cutting concerns could be exposed as hooks (`useAuth()`, `useTheme()`, `useRouter()`) consumed directly inside the component. HOC-wrapper chains obscure a component's actual prop shape ("wrapper hell") and make displayed component names in DevTools hard to trace back to the source; hooks keep the composed logic visible at the call site.
- **DO:** Compose small, single-purpose components together to build complex UI (a `Modal` built from `Modal.Root`, `Modal.Header`, `Modal.Body`, `Modal.Footer` compound components) rather than one component with many props controlling which internal sections render. Compound components let the consumer control structure and content directly through JSX, which composes more naturally than an ever-expanding props API.

### Server Components vs Client Components

- **DO:** Default new components in a framework that supports React Server Components (e.g., Next.js App Router) to Server Components, and add the `'use client'` directive only to the specific leaf components that actually need interactivity, browser-only APIs, or React state/effects. This keeps more of the bundle off the client and lets data fetching happen closer to the data source without shipping fetch/serialization code to the browser.
- **DON'T:** Add `'use client'` to a component reflexively "to be safe" or because an example online had it, without checking whether the component actually uses hooks, event handlers, or browser APIs. Marking a component (and everything it imports) as a Client Component pulls it and its entire subtree into the client JavaScript bundle, increasing bundle size and hydration cost for no functional reason.
- **DO:** Push `'use client'` boundaries as far down the tree as possible — wrap only the small interactive island (a like button, a dropdown, a form) in a Client Component, and keep the surrounding layout, headings, and static content as Server Components. This "leaf-level" client boundary pattern minimizes the JavaScript shipped to the browser.
```tsx
// GOOD — only the interactive button is a Client Component;
// the page and its static content stay server-rendered.
// app/product/[id]/page.tsx (Server Component, no directive needed)
export default async function ProductPage({ params }) {
  const product = await getProduct(params.id);
  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} />
    </article>
  );
}

// components/AddToCartButton.tsx
'use client';
export function AddToCartButton({ productId }) {
  const [pending, setPending] = useState(false);
  return <button onClick={() => addToCart(productId, setPending)}>Add to cart</button>;
}
```
- **DON'T:** Pass non-serializable values (functions, class instances, Symbols, React elements created inside a Server Component that close over server-only state) as props from a Server Component into a Client Component. Only serializable data crosses that boundary; passing a function reference either throws or silently fails, and passing a database client or secret leaks server-only capabilities into a boundary that isn't meant to carry them.
- **DO:** Fetch data directly inside Server Components with `async`/`await`, colocated with the component that needs it, instead of routing every request through a client-side data-fetching hook when the data is available at request/render time on the server. This removes a client-server round trip and the associated loading-spinner flash for data that could have been rendered on the first response.
- **DON'T:** Treat Server Components as a place for interactive state or effects — they cannot use `useState`, `useEffect`, `useContext`, or browser event handlers at all, because they never run in the browser. Reaching for these inside a Server Component is a build error in frameworks that enforce the boundary, or dead code in ones that don't; move the stateful piece into a Client Component and pass only the data it needs as props.
- **DO:** Keep environment secrets, API keys, and privileged database/service credentials exclusively inside Server Components and server-only modules. Because Client Components ship their code to the browser, any secret referenced from a Client Component (or from a shared module imported by one) is exposed to anyone who opens dev tools.
- **DON'T:** Assume "Server Components" and "Server-Side Rendering (SSR)" are the same thing. SSR renders a Client Component's output to HTML on the server for the first paint, but the component still hydrates and re-runs its full JavaScript in the browser afterward; a true Server Component never ships its component code to the client at all and never hydrates — conflating the two leads to wrong assumptions about what gets sent over the wire.
- **DO:** Use Server Actions (or the framework's equivalent server-mutation mechanism) for form submissions and mutations that don't need optimistic, highly interactive client feedback, instead of building a client-side fetch call to a hand-written API route for every mutation. This removes an entire API-route layer for simple CRUD operations tied to a specific UI.
- **DON'T:** Wrap an entire page or layout in a single top-level `'use client'` directive just because one small piece of it needs interactivity. That single directive converts every component imported beneath it into client code, defeating the purpose of the server/client split; isolate the interactive piece into its own small Client Component and import that into the still-server-rendered page instead.

### Suspense and Error Boundaries

- **DO:** Wrap data-fetching or lazily-loaded parts of the tree in a `Suspense` boundary with a meaningful `fallback`, and wrap the same regions (or a slightly larger enclosing region) in an error boundary. These two mechanisms are complementary: `Suspense` handles the "not ready yet" state declaratively, while an error boundary handles the "this threw" state — a component tree with data fetching but neither is missing two of its three fundamental states (loading, error, success).
- **DON'T:** Rely on a single top-level `Suspense`/error boundary wrapping the entire application as the only boundary anywhere in the tree. One global boundary means any single failing or slow component blanks out or freezes the *entire* app behind one fallback screen; place boundaries around independent, meaningfully-sized regions (a page section, a widget, a route) so a failure or a slow load in one area doesn't take down unrelated, already-loaded UI elsewhere on the page.
```jsx
// GOOD — an isolated widget's failure or loading state doesn't
// blank out the rest of an already-rendered dashboard
<Dashboard>
  <ErrorBoundary fallback={<WidgetError />}>
    <Suspense fallback={<WidgetSkeleton />}>
      <RevenueWidget />
    </Suspense>
  </ErrorBoundary>
  <ErrorBoundary fallback={<WidgetError />}>
    <Suspense fallback={<WidgetSkeleton />}>
      <ActivityFeedWidget />
    </Suspense>
  </ErrorBoundary>
</Dashboard>
```
- **DO:** Implement error boundaries as class components using `static getDerivedStateFromError` and `componentDidCatch` (React has no Hook-based error boundary API as of the versions in common use), or use a well-maintained library (`react-error-boundary`) instead of hand-rolling one from scratch in every project. Error boundaries have a small, fixed, easy-to-get-wrong implementation surface, and a shared implementation avoids repeating subtle mistakes (forgetting to reset the boundary's state when the underlying error condition is fixed and retried).
- **DON'T:** Assume an error boundary catches errors thrown from event handlers, asynchronous code (a rejected Promise inside a `setTimeout` or a fetch callback), server-side rendering, or errors thrown in the boundary component itself. React error boundaries only catch errors thrown during rendering, in lifecycle methods, and in constructors of the tree below them; handle event-handler and async errors with ordinary `try`/`catch` and explicit error state instead, and don't treat the presence of an error boundary as a substitute for that handling.
- **DO:** Give an error boundary's fallback UI a way to retry (a "Try again" button that resets the boundary's error state and re-attempts rendering the child tree) rather than leaving the user stuck on a dead-end error screen with no path forward except a full page reload.

### Controlled vs Uncontrolled Components

- **DO:** Decide deliberately whether a form input is controlled (its value lives in React state and is driven via `value`/`onChange`) or uncontrolled (its value lives in the DOM itself, read via a `ref` when needed), and be consistent about which one a given input is for its entire lifetime. Controlled inputs give React authority over the value (useful for live validation, formatting-as-you-type, or syncing to other UI); uncontrolled inputs are simpler and cheaper when the value is only needed at submit time (e.g., wired up via an uncontrolled form library or plain `FormData`).
- **DON'T:** Switch an input between controlled and uncontrolled across renders — most commonly, initializing a controlled input's state as `undefined`/`null` and only assigning it a string value later. React logs a warning for this and the input's behavior around cursor position and typed characters can become inconsistent; always initialize controlled state to a defined value of the correct type (`''` for a text input, not `undefined`).
```jsx
// BAD — value starts undefined (uncontrolled), then becomes a string
// (controlled) once data loads — React warns about switching modes.
const [name, setName] = useState(); // undefined initial value
<input value={name} onChange={(e) => setName(e.target.value)} />

// GOOD — always controlled, from a defined initial value
const [name, setName] = useState('');
<input value={name} onChange={(e) => setName(e.target.value)} />
```
- **DO:** Prefer uncontrolled inputs (or a form library built around them, like React Hook Form) for large forms with many fields, when per-keystroke React state updates and re-renders for every field aren't actually needed. Controlling dozens of fields through individual `useState` calls means every keystroke in any field re-renders the whole form component; an uncontrolled approach with validation triggered on blur/submit avoids that render cost entirely.

### Forms and Validation

- **DO:** Use a dedicated form library (React Hook Form, Formik, or a framework's built-in forms system — Angular's Reactive Forms, VeeValidate for Vue) once a form has more than a handful of fields or needs non-trivial validation, instead of hand-rolling validation state and error-message wiring field by field. These libraries have already solved cross-field validation, submission state, dirty/touched tracking, and schema-based validation (often via Zod/Yup) in a tested, well-documented way.
- **DON'T:** Validate only on the client and skip server-side validation of the same data, or the reverse. Client-side validation is for immediate user feedback and is trivially bypassable (a modified request, a disabled JS environment, a non-browser API client); the server must independently validate and reject invalid data regardless of what the client already checked, and the client validation exists purely to make the common, honest-user path pleasant, not to serve as the actual security/data-integrity boundary.
- **DO:** Validate against a single shared schema (Zod, Yup, or similar) reused on both the client (for immediate feedback) and the server (for authoritative validation) when the stack allows sharing code between them (e.g., a full-stack TypeScript app). A shared schema guarantees the client and server can never validate the same field differently, which is a common source of "the form said this was fine, but the server rejected it" bugs when the two validation rule sets are maintained by hand in two places.
- **DON'T:** Disable a submit button for the entire duration a form is "invalid" in a way that gives the user no indication of *why* — a grayed-out button with no visible validation messages leaves the user guessing which field is wrong. Show field-level error messages (associated via `aria-describedby`, per the accessibility section above) as soon as it's helpful (typically on blur, not on every keystroke, to avoid nagging a user who hasn't finished typing yet) rather than only revealing that something is wrong at submit time with no detail.

### Testing Considerations

- **DO:** Write component tests that assert on rendered output and user-facing behavior (what's on the screen, what happens when a user clicks/types) using a library like React Testing Library / Vue Testing Library, rather than asserting on a component's internal implementation details (state variable names, instance methods, internal prop names passed to child components). Tests coupled to implementation details break on every internal refactor even when the actual user-facing behavior hasn't changed, which trains a team to distrust and eventually ignore test failures.
```jsx
// BAD — tests implementation details, breaks on internal refactors
expect(wrapper.state('isOpen')).toBe(true);

// GOOD — tests what a user actually sees/does
await userEvent.click(screen.getByRole('button', { name: /open menu/i }));
expect(screen.getByRole('menu')).toBeVisible();
```
- **DON'T:** Query the DOM in tests by CSS class name or a test-only `data-testid` as the first choice when an accessible query (`getByRole`, `getByLabelText`, `getByText`) would also work. Querying by role/label doubles as an accessibility check — if `getByRole('button', { name: 'Submit' })` can't find the element, it usually means a real user relying on assistive technology couldn't find it either; reserve `data-testid` for elements with no reasonable accessible way to query them (e.g., a purely decorative container).
- **DO:** Test custom hooks in isolation with a dedicated testing utility (`renderHook` from React Testing Library) when a hook has meaningful branching logic of its own, rather than only ever testing it indirectly through every component that happens to use it. Isolated hook tests are faster to write, faster to run, and pinpoint failures directly in the hook rather than requiring a failure to be traced back from a component-level test.
- **DON'T:** Mock so much of a component's dependencies (every child component, every hook, every context) that the resulting test only verifies the mocks were called correctly, rather than verifying anything about real component behavior. Over-mocked tests pass even when the real integration between the pieces is broken; prefer testing a reasonably-sized subtree with its real child components and only mocking true external boundaries (network requests, browser APIs, timers).
- **DO:** Include at least one test per interactive component that exercises it via keyboard only (`userEvent.tab()`, `userEvent.keyboard('{Enter}')`) alongside the mouse-driven tests, catching keyboard-accessibility regressions automatically instead of relying on someone remembering to manually re-test with a keyboard after every change.

### Concurrent Features and Rendering Timing

- **DO:** Reach for `useTransition`/`startTransition` to mark a state update as non-urgent (e.g., re-filtering a large list as the user types in a search box) so React can keep the input itself responsive by interrupting the slower re-render if a newer update comes in, rather than blocking the input on every keystroke while a big list re-renders synchronously.
```jsx
const [isPending, startTransition] = useTransition();
function handleChange(e) {
  setQuery(e.target.value); // urgent: keep the input responsive
  startTransition(() => {
    setFilteredResults(filterBigList(e.target.value)); // non-urgent
  });
}
```
- **DON'T:** Wrap genuinely urgent updates (the value the user is actively typing into a controlled input) inside `startTransition`. Marking the input's own state update as non-urgent can make the input itself lag behind the user's keystrokes, which is the opposite of the desired effect; only the expensive, secondary re-render triggered by that input should be marked as a transition, not the input's own value update.
- **DO:** Use `useDeferredValue` when a value comes from a prop or another source the component doesn't directly control the update of, but a downstream expensive render based on that value should be allowed to lag behind slightly rather than block the UI. It's the "receive a value, defer using it" counterpart to `startTransition`'s "mark my own update as low priority."
- **DON'T:** Reach immediately for `useLayoutEffect` in place of `useEffect` without a specific reason. `useLayoutEffect` runs synchronously after DOM mutations but before the browser paints, blocking visual updates until it completes — genuinely necessary for effects that must measure or mutate the DOM before the user sees a flicker (e.g., measuring an element and repositioning it before paint), but a needless default choice for anything else, since it removes the async-scheduling benefits `useEffect` provides.
- **DO:** Use `useLayoutEffect` specifically for the narrow case of reading a layout value (like an element's measured size) and synchronously applying a DOM change based on it before the browser paints, to avoid a visible one-frame flash of the unadjusted layout. Outside of measurement-and-adjust cases like this, default to `useEffect`.

### Portals and Refs

- **DO:** Use `createPortal` to render content (modals, tooltips, dropdowns that must escape an `overflow: hidden`/`z-index`-stacking-context parent) into a different part of the DOM tree than its logical component position, while keeping it part of the same React tree for event bubbling, context, and state purposes. This solves the common CSS stacking/clipping problem of a modal rendered inside a component with `overflow: hidden` without giving up React's component-tree semantics.
- **DON'T:** Forget that a portaled component still participates in React's context tree and event bubbling based on its logical (not DOM) position — code that assumes a portaled modal is "detached" from its parent's context or event handling will be surprised when context values and bubbled synthetic events still flow through normally.
- **DO:** Use `forwardRef` combined with `useImperativeHandle` specifically to expose a deliberately narrow, controlled imperative API from a component (e.g., an `inputRef.current.focus()` method) rather than exposing the component's entire internal DOM node or internal state through the ref. `useImperativeHandle` lets a component keep encapsulation while still supporting the rare cases where imperative control (focus, scroll-into-view, an imperative animation trigger) is the more natural API than a purely prop-driven one.
- **DON'T:** Reach for refs and imperative method calls as a general substitute for props and state simply because it's more familiar from imperative DOM programming. Declarative props/state should be the default communication mechanism between components; imperative refs are the exception, reserved for the specific cases (focus management, media playback control, triggering an animation, integrating a non-React/imperative third-party library) where there's no good declarative equivalent.

## Vue

### Composition API vs Options API

- **DO:** Pick one primary API style — Composition API with `<script setup>` for new Vue 3 projects, or Options API for a codebase that has already standardized on it — and apply it consistently within a project. Mixing both styles heavily across the same codebase forces every contributor to hold two different mental models of "where does this component's logic live."
- **DON'T:** Mix Composition API and Options API within the *same* component (e.g., a `setup()` function alongside `data()`, `methods`, and `computed` in the same file) except for the narrow, well-understood migration case of gradually introducing composables into legacy Options components. Vue does support both coexisting, but a component that uses both makes it unclear which state is reactive through which system and complicates debugging.
- **DO:** Prefer the Composition API (`<script setup>`) for new components, especially ones with non-trivial logic reuse needs. It gives better TypeScript inference, lets logic be extracted into composables without the pitfalls of Options-API mixins (implicit property merging, unclear origin of injected properties), and colocates related reactive state instead of splitting it across `data`, `computed`, and `methods` blocks.
```vue
<!-- GOOD — related state and logic colocated -->
<script setup>
import { ref, computed } from 'vue';
const count = ref(0);
const doubled = computed(() => count.value * 2);
function increment() { count.value++; }
</script>
```
- **DON'T:** Reach for the Options API's `mixins` to share logic across components in new code. Mixins merge properties into the component implicitly, so a consumer of the component cannot tell, just by reading the component, which properties came from which mixin, and name collisions between mixins fail silently or overwrite each other; use composables (Composition API) instead, which make every imported piece of state and every function explicit at the call site.
- **DO:** Extract reusable stateful logic into composables (functions prefixed `use`, following the same convention as React hooks) once it's needed in more than one component. A composable that returns `{ data, error, isLoading, refetch }` from a single `useFetch(url)` call keeps data-fetching logic out of every component that needs it.
- **DON'T:** Call composables conditionally or inside loops in a way that changes how many reactive refs get created across invocations, mirroring the Rules of Hooks concern in React. Although Vue's reactivity system doesn't rely on call order the way React's hooks do, conditionally skipping a composable that registers a lifecycle hook (`onMounted`, `onUnmounted`) or a watcher can leave a component in an inconsistent subscription state; keep composable calls unconditional at the top of `setup()` and let the composable itself branch on its arguments if needed.

### Reactivity Pitfalls

- **DO:** Understand that `ref()` wraps a value in an object with a `.value` property that must be unwrapped in JavaScript code, while `reactive()` returns a deeply reactive proxy of an object whose properties are accessed directly (no `.value`). Confusing the two idioms is the single most common source of "why isn't this updating" bugs in Composition API code.
```js
// ref: unwrap with .value in script, auto-unwrapped in template
const count = ref(0);
count.value++;

// reactive: access properties directly, no .value
const state = reactive({ count: 0 });
state.count++;
```
- **DON'T:** Destructure a `reactive()` object's properties into plain local variables and expect them to stay reactive. Destructuring reads the current primitive value once and breaks the reactive connection to the source object; use `toRefs()` (or `toRef()` for a single property) when destructuring is needed, which preserves reactivity by returning refs bound back to the source.
```js
// BAD — count is a plain number now, disconnected from state.count
const { count } = reactive({ count: 0 });

// GOOD — toRefs preserves the reactive link
const state = reactive({ count: 0 });
const { count } = toRefs(state);
```
- **DON'T:** Reassign a `reactive()` object wholesale (`state = newObject`) expecting the template to pick up the new object. `reactive()` returns a Proxy tied to the original target; overwriting the variable that held it breaks every existing binding to that proxy. Either mutate the existing reactive object's properties in place, or use `ref()` around the object if wholesale replacement is actually the intended operation.
- **DO:** Use `ref()` for primitive values (and as the general-purpose default, since it also works for objects) and reserve `reactive()` for cases where an object is genuinely always used as a whole and never reassigned or destructured. This avoids the destructuring pitfall entirely by making `.value` unwrapping the one consistent rule to remember.
- **DON'T:** Mutate a prop directly inside a child component. Props flow one-way from parent to child; mutating them directly works in some cases due to JavaScript's pass-by-reference semantics for objects but produces a Vue warning and creates a state source that's ambiguous (does the parent's or the child's value win?) — emit an event asking the parent to update its own state instead, or copy the prop into local reactive state if the child needs an independent editable copy.
```vue
<!-- BAD — mutating a prop directly -->
<script setup>
const props = defineProps(['modelValue']);
function toggle() { props.modelValue = !props.modelValue; } // warns, wrong direction
</script>

<!-- GOOD — emit the change, let the parent own the source of truth -->
<script setup>
const props = defineProps(['modelValue']);
const emit = defineEmits(['update:modelValue']);
function toggle() { emit('update:modelValue', !props.modelValue); }
</script>
```
- **DO:** Use `v-model` (and `defineModel()` in Vue 3.4+) for two-way-bound form inputs and custom components instead of manually wiring a prop and an `@input`/`@change` listener by hand. It's the idiomatic, well-tested pattern and keeps the parent/child contract for editable state explicit and consistent across the codebase.
- **DON'T:** Watch a whole reactive object or array with `watch()` without `{ deep: true }` when the mutation happens on a nested property, and expect the callback to fire. By default, `watch()` on a reactive source only triggers on reference changes at the watched path, not on deep nested mutations; pass `{ deep: true }` explicitly when deep mutations must be observed, understanding that deep watching has a real performance cost on large objects.
- **DO:** Prefer `computed()` over a `watch()` + separate ref combo whenever a value can be derived purely from other reactive state. `computed()` is cached, lazily re-evaluated only when its dependencies change, and eliminates an entire class of bugs where a `watch` callback that's supposed to keep a derived value in sync gets out of step with its source.
```js
// BAD — a watcher just to keep a derived value updated
const price = ref(10);
const qty = ref(2);
const total = ref(20);
watch([price, qty], ([p, q]) => { total.value = p * q; });

// GOOD — computed handles the dependency tracking automatically
const price = ref(10);
const qty = ref(2);
const total = computed(() => price.value * qty.value);
```
- **DON'T:** Perform side effects with lasting external impact (network requests that mutate server state, writes to `localStorage` outside of persistence-specific composables) inside a `computed()` getter. Computed getters are expected to be pure and can be recomputed multiple times, or not at all if unused, purely based on Vue's internal caching decisions; use `watch()` or `watchEffect()` for anything that needs to reliably run exactly once per meaningful change as a side effect.
- **DO:** Clean up any subscriptions, timers, or event listeners created in a composable's `onMounted` inside a matching `onUnmounted`, exactly as an effect's cleanup function is required in React. A composable used in many components that leaks one listener per mount accumulates a listener per navigation and degrades performance over a session.

### Single-File Component Conventions

- **DO:** Keep the standard `<script setup>` / `<template>` / `<style scoped>` block order in `.vue` files and keep components single-purpose — one component, one clear responsibility, matching its filename. Consistent block order and one-component-per-file makes files predictable to navigate across a team.
- **DON'T:** Write global, unscoped CSS inside a component's `<style>` block without either the `scoped` attribute or a CSS Modules / naming convention (BEM, utility classes) that prevents collisions. Unscoped component styles leak into the global stylesheet cascade and can unpredictably affect unrelated components that happen to share a class or element selector.
- **DO:** Name components with multi-word PascalCase names (`UserProfileCard`, not `Card` or `profile`) in both the filename and the component registration. Multi-word names avoid collisions with existing and future native HTML elements and make a component's purpose identifiable at a glance in DevTools and imports.
- **DON'T:** Put non-trivial business logic (data transformation, validation rules, API calls not related to fetching this component's own display data) directly inline in a large `<script setup>` block that also handles template-facing reactive state. Extract it into a composable or a plain utility module so the component's script block stays focused on wiring reactive state to the template, and the logic itself becomes unit-testable without mounting a component.
- **DO:** Declare props with explicit types and, where meaningful, runtime validators via `defineProps` (using TypeScript generics in `<script setup lang="ts">`, or the object syntax with `type`/`required`/`validator` in JS). Untyped, unvalidated props push bugs from "component receives wrong shape" at development time out to "silently renders wrong thing" at runtime.
```vue
<script setup lang="ts">
interface Props {
  title: string;
  count?: number;
}
const props = withDefaults(defineProps<Props>(), { count: 0 });
</script>
```
- **DON'T:** Emit event names that don't match what the component actually communicates (a generic `change` for what's semantically a `submit`, or emitting inconsistent casing like `updateValue` in one component and `update-value` in another). Vue recommends kebab-case event names in templates; standardize the naming convention for emits across the codebase so parent components can predict what to listen for.
- **DO:** Use slots (default, named, and scoped slots) to let a parent customize a child's rendered content, the same way `children` is used for composition in React. A `<Card>` component that accepts a `#header` and `#footer` named slot is more flexible than one with `headerText`/`footerText` string props when the content needs richer markup.
- **DON'T:** Nest deeply-coupled sibling components that reach into each other via `$parent`/`$refs` chains to call methods or read state across the tree. This creates an implicit, untyped coupling that's invisible from either component's declared props/emits interface; pass data down through props and communicate up through emits (or a shared composable/store for non-adjacent components), keeping the parent-child contract explicit.

### State Management with Pinia

- **DO:** Use Pinia (the current officially-recommended Vue state-management library, superseding Vuex) for shared application state, structured as multiple small, focused stores (a `useCartStore`, a `useAuthStore`) rather than one monolithic store holding all application state. Small, focused stores mirror the same "split large contexts into small ones" guidance given for React and keep each store's actions and getters legible.
```js
// stores/cart.js
export const useCartStore = defineStore('cart', {
  state: () => ({ items: [] }),
  getters: {
    total: (state) => state.items.reduce((sum, i) => sum + i.price * i.qty, 0),
  },
  actions: {
    addItem(item) { this.items.push(item); },
  },
});
```
- **DON'T:** Mutate a Pinia store's state directly from many unrelated components scattered across the codebase, bypassing the store's own defined actions. While Pinia does allow direct state mutation (unlike Vuex, which required mutations), doing so from arbitrary call sites makes every possible state change untraceable to a single place; define actions for meaningful state transitions and call those from components, reserving direct state access for genuinely simple, one-off cases.
- **DO:** Use Pinia's getters for derived/computed store values (mirroring the `computed()` guidance above) instead of storing a derived value as separate state that has to be kept in sync by hand on every relevant action.
- **DON'T:** Reach for Pinia (or any shared store) for state that's genuinely local to one component or one small, non-shared subtree. The same locality-first principle from the general State Management section applies inside Vue specifically — component-local `ref`/`reactive` state remains the right default, promoted to a Pinia store only once multiple, non-adjacent components need it.

### provide/inject Pitfalls

- **DO:** Use `provide`/`inject` for the same category of problem Context solves in React — values needed by many components at varying depths within a bounded subtree (a form's shared validation context, a themed component library's internal theme tokens) — and prefer typed injection keys (`InjectionKey<T>` in TypeScript) over plain string keys, which give no compile-time guarantee that the provided and injected types actually match.
- **DON'T:** Provide a mutable reactive object via `provide()` without also providing a controlled way to update it (paralleling the "expose named update functions, not a raw mutable store" guidance above). A consumer component that can freely mutate an injected reactive object makes it impossible to trace, from the providing ancestor's code alone, every place that object's value can change.
- **DO:** Make an injected value's optionality explicit and handle the "not provided" case deliberately — `inject(key, defaultValue)` with an explicit default, or an explicit check/throw when a value is genuinely required. Injecting a value with no default and no injected-value guard, in a component that gets rendered somewhere its expected ancestor `provide()` call is missing, fails silently (`undefined`) rather than with a clear error pointing at the actual problem.
- **DON'T:** Chain `provide`/`inject` across many unrelated levels of nesting as a substitute for simply passing a prop down one or two levels. Exactly as with React Context, one or two levels of an explicit, typed prop is often more traceable than an implicit injected value whose provider could be anywhere above the current component in the tree.

### Teleport, watchEffect, and Script Setup Macros

- **DO:** Use `<Teleport>` to render a component's markup into a different part of the DOM (commonly `body`) for modals, tooltips, and toasts — the same DOM-escaping problem `createPortal` solves in React — while keeping the teleported content part of the same logical Vue component tree for props, events, and reactivity.
- **DON'T:** Forget to condition a `<Teleport>`'s target existence, especially during server-side rendering (where the target DOM node may not exist at the time of render) — guard with `<ClientOnly>`/an `onMounted`-gated render or verify the framework's SSR-safe teleport handling, since teleporting to a nonexistent target fails.
- **DO:** Reach for `watchEffect()` instead of `watch()` when an effect should automatically track whatever reactive values it reads, without needing to declare them as an explicit source list — it runs once immediately and then re-runs whenever any reactive dependency it read on its last run changes. Use `watch()` instead when the effect needs the previous value alongside the new one, needs to *not* run immediately on setup, or needs precise, explicit control over exactly which sources trigger it (rather than "whatever this function happens to read").
```js
// watchEffect — implicit dependency tracking, runs immediately
watchEffect(() => {
  document.title = `${unreadCount.value} unread`;
});

// watch — explicit source, gives access to both the old and new value
watch(unreadCount, (newVal, oldVal) => {
  if (newVal > oldVal) notifyUser();
});
```
- **DON'T:** Use `defineExpose()` to expose a component's entire internal state/methods to any parent holding a template ref. `<script setup>` components are closed by default (a parent template ref only gets `undefined` unless the child explicitly exposes something), which is the correct default for encapsulation; expose only the specific, deliberate imperative API a parent genuinely needs (mirroring the `useImperativeHandle` guidance above), not the component's full internals.
- **DO:** Use `<script setup>`'s compiler macros (`defineProps`, `defineEmits`, `defineModel`, `defineExpose`) as the standard way to declare a component's public interface in new Vue 3 components, since they're compiled away and require no explicit import, and give strong TypeScript inference when typed with generics.

### Testing Vue Components

- **DO:** Use Vue Testing Library (built on the same philosophy as React Testing Library) to test components through their rendered, user-facing output and accessible queries, rather than reaching into a component's internal instance (`wrapper.vm.someInternalMethod()`) via Vue Test Utils' lower-level API. The same implementation-detail-coupling concerns from the React testing guidance apply equally here.
- **DON'T:** Shallow-render every component test by default (stubbing out all child components) as a blanket policy. Shallow rendering can hide real integration bugs between a component and its children (a prop name mismatch, a slot that's never actually filled correctly) that only a more complete render would catch; reserve shallow rendering for cases where a child is genuinely expensive or irrelevant to the behavior under test, and prefer fuller rendering by default.

## Angular

### Module and Standalone Component Conventions

- **DO:** Default to standalone components, directives, and pipes (the default since Angular 14+, and the default project structure since Angular 17+) for new code, and set `standalone: true` explicitly on older Angular versions where it isn't yet the default. Standalone components declare their own imports directly, removing the indirection of tracing a component's dependencies through an `NgModule`'s `declarations`/`imports` arrays.
```ts
// GOOD — standalone component declares its own dependencies directly
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './user-card.component.html',
})
export class UserCardComponent {}
```
- **DON'T:** Introduce a new `NgModule` for a greenfield Angular 17+ project just because older tutorials or existing internal examples still use them. `NgModule`s add an extra layer of bookkeeping (which module declares this component, which module exports it, which module needs to import that module) that standalone components eliminate; reserve `NgModule` usage for maintaining an existing module-based codebase or for third-party libraries that still ship module-based APIs.
- **DO:** Migrate an existing module-based codebase to standalone incrementally (Angular's own schematics, `ng generate @angular/core:standalone`, support this) rather than attempting a single big-bang rewrite. Standalone components can be imported into and used from existing `NgModule`-based components during the transition, so the migration doesn't have to be all-or-nothing.
- **DON'T:** Register a component, directive, or pipe in more than one `NgModule`'s `declarations` array (in module-based code) — Angular will throw a compile-time error, but the underlying design smell (a shared, reusable piece of UI declared inside a feature module) is worth avoiding: put genuinely shared declarables in a dedicated shared/UI module (or, in standalone-first code, just import the standalone component directly wherever it's used).
- **DO:** Keep routing configuration declarative and lazy by feature: use `loadComponent`/`loadChildren` with dynamic `import()` in the route config for standalone components, so each feature's code is code-split automatically rather than bundled into the initial chunk.
```ts
export const routes: Routes = [
  {
    path: 'settings',
    loadComponent: () =>
      import('./settings/settings.component').then((m) => m.SettingsComponent),
  },
];
```
- **DON'T:** Import an entire feature module or an unrelated shared module into a component just to get access to one pipe or directive it uses. This bloats the standalone component's dependency graph and pulls in code the component never actually uses at runtime; import only the specific standalone pipes/directives/components actually referenced in the template.
- **DO:** Use Angular's `inject()` function inside standalone components, functional route guards, and functional interceptors, instead of constructor-parameter dependency injection, when writing functional-style code (guards, resolvers, interceptors defined as plain functions rather than injectable classes). This is the idiomatic pattern for the functional APIs Angular has moved toward and avoids needing a class wrapper purely to get constructor injection.

### RxJS Discipline

- **DO:** Unsubscribe from every manually-created `Observable` subscription when a component is destroyed, using `takeUntilDestroyed()` (Angular 16+, the current idiomatic approach), the `async` pipe (which handles subscription/unsubscription automatically), or an explicit `Subscription` collected and unsubscribed in `ngOnDestroy`. An un-cleaned subscription keeps a reference to the destroyed component alive and keeps firing its callback against a component that's no longer in the DOM, which is both a memory leak and a source of "cannot read property of undefined" errors.
```ts
// GOOD — subscription automatically cleaned up on component destroy
export class UserListComponent {
  private destroyRef = inject(DestroyRef);
  constructor(private userService: UserService) {
    this.userService.users$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((users) => (this.users = users));
  }
}
```
- **DO:** Prefer the `async` pipe in templates over manually subscribing in the component class whenever the observable's value is only needed for display. The `async` pipe subscribes on component init, unsubscribes on destroy, and triggers change detection automatically — eliminating an entire category of manual-subscription-management bugs.
```html
<!-- GOOD — no manual subscribe/unsubscribe needed -->
<ul>
  <li *ngFor="let user of users$ | async">{{ user.name }}</li>
</ul>
```
- **DON'T:** Subscribe to an observable inside another observable's `subscribe()` callback ("nested subscribes" / the "subscribe pyramid") to sequence dependent async calls. Nested subscriptions are hard to read, hard to cancel correctly, and usually indicate the code should use a flattening operator (`switchMap`, `mergeMap`, `concatMap`, `exhaustMap`) to compose the two observables into one pipeline instead.
```ts
// BAD — nested subscribe, no automatic cancellation of the inner call
this.userService.getUser(id).subscribe((user) => {
  this.orderService.getOrders(user.id).subscribe((orders) => {
    this.orders = orders;
  });
});

// GOOD — flattened with switchMap, inner call auto-cancelled on new id
this.userId$.pipe(
  switchMap((id) => this.userService.getUser(id)),
  switchMap((user) => this.orderService.getOrders(user.id)),
).subscribe((orders) => (this.orders = orders));
```
- **DO:** Choose the flattening operator deliberately based on the desired cancellation/concurrency behavior: `switchMap` to cancel the previous inner observable when a new source value arrives (typeahead search, route-param-driven fetches), `mergeMap` to run all inner observables concurrently (independent parallel requests), `concatMap` to queue them strictly in order (sequential writes that must not race), and `exhaustMap` to ignore new source values while an inner observable is still in flight (preventing double-submit on a button click). Using the wrong one is a common source of subtle race conditions — e.g., `mergeMap` on a search box lets stale, slower responses overwrite fresher ones.
- **DON'T:** Use `mergeMap` for a search-as-you-type input and expect results to always reflect the latest query. Because `mergeMap` runs every inner request concurrently and emits results as they resolve, a slow response to an earlier keystroke can arrive after a faster response to a later keystroke and overwrite it with stale results; use `switchMap` there specifically because it cancels the in-flight previous request when a new one starts.
- **DO:** Handle errors within the RxJS pipeline itself using `catchError`, positioned at the point where the specific failure should be handled, rather than letting an unhandled error propagate up and silently terminate the entire observable chain (including any downstream operators and the subscription itself). An uncaught error in an Observable pipeline unsubscribes the whole chain — a single failed request can silently kill a long-lived stream (e.g., a shared, app-wide observable) for every consumer.
```ts
this.searchTerm$.pipe(
  switchMap((term) =>
    this.api.search(term).pipe(
      catchError((err) => {
        this.errorService.report(err);
        return of([]); // fall back to an empty result set, keep the stream alive
      })
    )
  )
).subscribe((results) => (this.results = results));
```
- **DON'T:** Create a fresh, uncached `Observable` from a cold HTTP call every time multiple subscribers need the same data, when a single shared request would do. Without `shareReplay` (or an equivalent multicasting operator), each `subscribe()` call to a cold observable made from `HttpClient` triggers its own separate HTTP request — three components displaying the same "current user" data can silently fire three identical network requests.
- **DO:** Use `shareReplay({ bufferSize: 1, refCount: true })` (favoring the object-config overload with `refCount: true` over the deprecated bare-number overload) when multiple subscribers should share one underlying source and late subscribers should receive the most recent emitted value. The `refCount: true` option additionally ensures the underlying subscription is torn down when the last subscriber unsubscribes, rather than staying open forever.
- **DON'T:** Chain long sequences of RxJS operators inline in a template-binding-adjacent property or a deeply nested subscribe callback without extracting the pipeline into a clearly named, well-typed observable property or a service method. A ten-operator pipe chain inlined where it's consumed is unreadable and untestable in isolation; give the composed pipeline its own name (`filteredResults$`) so its purpose is legible without mentally executing every operator.
- **DO:** Prefer Angular Signals (Angular 16+) for simple, synchronous component state — especially state that doesn't represent an asynchronous stream of events — over wrapping it in RxJS `BehaviorSubject`s purely out of habit. Signals have a simpler mental model (a value you read and write, with automatic dependency tracking for `computed()`) and integrate with the newer, more granular change-detection model; reserve RxJS for what it's actually suited for — composing and transforming genuinely asynchronous event streams (HTTP, WebSocket messages, user input events over time, complex cancellation/retry logic).

### Change Detection Pitfalls

- **DO:** Set `changeDetection: ChangeDetectionStrategy.OnPush` on presentational/leaf components whose inputs are treated as immutable, and pair it with immutable data updates (create new object/array references on change, mirroring the React state-immutability rule). `OnPush` skips a component's change-detection check entirely unless one of its `@Input()` references changes, an event originates from inside it, or an observable bound via the `async` pipe emits — this is one of the highest-leverage Angular performance techniques available.
```ts
@Component({
  selector: 'app-price-tag',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<span>{{ price | currency }}</span>`,
})
export class PriceTagComponent {
  @Input() price!: number;
}
```
- **DON'T:** Mutate an `@Input()`-bound object or array in place on a component using `OnPush`. Because `OnPush` compares input references, not deep contents, mutating the same object and passing it back leaves the reference unchanged and the component silently fails to re-render, exactly like the analogous React `useState` mutation bug.
- **DO:** Call methods used inside a template with the understanding that Angular's default change detection re-invokes template expressions on every change-detection cycle. A method call in a template (`{{ getFormattedPrice() }}`) that does non-trivial work runs on every single change-detection pass across the whole app (potentially many times per user interaction), not once; move the computation to a `computed()` signal, a memoized getter, or a pipe (pure pipes are only re-evaluated when their inputs change by reference) instead of an unmemoized method call.
```html
<!-- BAD — re-runs the formatting logic on every CD cycle -->
<span>{{ getFormattedPrice() }}</span>

<!-- GOOD — pure pipe only re-evaluates when `price` changes -->
<span>{{ price | currency:'USD' }}</span>
```
- **DON'T:** Write a custom Angular pipe without marking it `pure: true` (the default) unless it genuinely needs to react to internal mutable state changes that aren't reflected in its input reference. An impure pipe (`pure: false`) re-executes on every change-detection cycle regardless of whether its arguments changed, which defeats the whole performance benefit pipes are meant to provide and can be a hidden source of severe slowdowns in large templates.
- **DO:** Run code that triggers frequent updates outside the Angular zone with `NgZone.runOutsideAngular()` when those updates don't need to trigger Angular's change detection (e.g., a high-frequency canvas animation loop, a raw WebSocket message handler feeding a chart that manages its own rendering). Angular's zone-based change detection, by default, triggers a full check on essentially every async event (timers, DOM events, XHR/fetch callbacks, promises); leaving high-frequency work inside the zone runs full change-detection passes far more often than necessary.
- **DON'T:** Assume `ChangeDetectorRef.detectChanges()` or `markForCheck()` calls sprinkled through a component are a substitute for correctly modeling `OnPush` inputs and signals. Manually forcing change detection to "fix" a component that isn't updating is treating the symptom; find the actual broken reactivity link (a mutated input, a missed signal update, an observable not going through the `async` pipe) and fix that instead, since scattered manual `detectChanges()` calls make change-detection timing unpredictable and hard to reason about.
- **DO:** Use Angular Signals for component and shared state now that they're a first-class reactivity primitive, since signal reads inside templates are automatically tracked and only the specific bindings that depend on a changed signal are re-evaluated — a more granular and closer-to-zero-effort optimization than manually tuning `OnPush` and immutability discipline everywhere.
```ts
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);
  increment() { this.count.update((c) => c + 1); }
}
```
- **DON'T:** Read a signal's value by calling it as a function (`count()`) outside of a reactive context (a template, `computed()`, or `effect()`) when the intent was to track that value reactively — e.g., reading it once in a constructor and expecting later updates to somehow propagate to a plain variable it was assigned to. A plain variable assigned from a signal read is a one-time snapshot, not a live binding; keep the signal itself as the source of truth and read it again wherever the current value is needed, or derive a `computed()` if a transformed live view is needed.
- **DO:** Understand `zoneless` change detection (an increasingly supported opt-in mode in recent Angular versions) as the direction the framework is heading — a model where signals, not Zone.js monkey-patching of every async API, are what schedule change detection. Writing signal-first, `OnPush`-everywhere code today keeps a codebase compatible with that migration path with minimal rework later.

### Dependency Injection Patterns

- **DO:** Scope injectable services deliberately using Angular's hierarchical injector: `providedIn: 'root'` for genuine app-wide singletons (an `AuthService`, a `ConfigService`), and component- or route-level providers for services that should be scoped to a feature or have a fresh instance per component instance (a wizard's in-progress-state service that should reset every time the wizard is opened). Registering everything as a root singleton by default when a service actually needs per-feature isolation causes state to leak between unrelated usages of the same feature.
- **DON'T:** Reach into the injector manually via `Injector.get()`/`EnvironmentInjector.get()` inside ordinary application code as a substitute for constructor injection or `inject()`, except in the narrow cases that genuinely require it (a dynamically-created component, a library's low-level integration point). Manual injector lookups bypass Angular's static analyzability of a class's dependencies and make it harder to see, just by reading a class's constructor or field declarations, everything it depends on.
- **DO:** Define an `InjectionToken` (with a clear, descriptive description string) for any non-class dependency injected via DI (a configuration object, a primitive value, a function) rather than relying on ambiguous string tokens or injecting primitive types directly, which Angular's DI system can't disambiguate on their own. A well-named, typed `InjectionToken` also gives useful debugging output when Angular reports an injection error.
- **DON'T:** Create tightly-coupled services that directly instantiate their own dependencies with `new` instead of receiving them via DI. Directly instantiated dependencies can't be swapped out for a test double in unit tests and can't be scoped/overridden per module or route the way an injected dependency can; let Angular's DI container own the construction and lifecycle of any dependency that has its own dependencies, side effects, or that a test might need to mock.

### Reactive Forms vs Template-Driven Forms

- **DO:** Prefer Reactive Forms (`FormGroup`, `FormControl`, `FormBuilder`) for any form with non-trivial validation, dynamic fields, or programmatic value manipulation, since the form's structure and validation rules are defined explicitly in the component class as testable, synchronous, immutable data — no template parsing or two-way-binding indirection required to reason about the form's current state.
```ts
// GOOD — form structure and validation defined explicitly, testable in isolation
this.form = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(8)]],
});
```
- **DON'T:** Default to Template-Driven Forms (`ngModel`-based, with validation directives sprinkled through the template) for anything beyond a genuinely simple form (a two-field contact form) where the light-weight, less-explicit template syntax is a reasonable tradeoff. Template-driven forms push validation logic and form structure into the template itself, which becomes hard to unit-test and hard to reason about once the form grows dynamic fields, cross-field validation, or conditional field visibility.
- **DO:** Implement cross-field validation (password confirmation matching, a date range where "end" must be after "start") as a custom `ValidatorFn` applied at the `FormGroup` level, not by manually comparing two `FormControl` values inside a template expression or a component method triggered on every keystroke. A `FormGroup`-level validator is Angular's idiomatic mechanism for exactly this kind of validation and integrates correctly with the rest of the form's validity/error state.
- **DON'T:** Leave a `FormControl`'s validation errors undisplayed or displayed with no association to the actual invalid field (mirroring the general forms/accessibility guidance above) — bind an error message's visibility to the specific control's `invalid`/`touched`/`dirty` state and associate it via `aria-describedby`, rather than showing one generic "form has errors" message with no indication of which field is the problem.

### Content Projection and Lifecycle Hooks

- **DO:** Use `<ng-content>` (with named `select` slots for multiple projection points) to let a parent project arbitrary markup into specific regions of a child component, mirroring the composition/slots guidance already given for React's `children` and Vue's slots. This keeps a wrapper component (a `Card`, a `Panel`) from needing string/template-string props for content that's more naturally expressed as real markup.
- **DON'T:** Implement `ngOnChanges` to react to every `@Input()` change generically without checking `changes[propertyName]` for the specific input actually being reacted to, and without checking `.firstChange` when the initial-set case should be treated differently from a later update. A generic `ngOnChanges` that reacts the same way to any input change, including the very first one during initialization, can trigger unwanted side effects (like an unnecessary initial API call already handled by `ngOnInit`) on component creation.
- **DO:** Put component initialization logic that depends on `@Input()` values already being set in `ngOnInit`, not the constructor. Angular sets `@Input()`-bound properties after the constructor runs but before `ngOnInit` is called, so code in the constructor that reads an `@Input()` value sees it as still `undefined`; reserve the constructor for dependency injection only.
- **DON'T:** Leave unremoved manually-attached DOM event listeners, `setInterval`/`setTimeout` timers, or third-party library instances (charts, maps) uncleaned in `ngOnDestroy`. Angular does not automatically tear these down when a component is destroyed (only its own template bindings and `@Output()` subscriptions set up through Angular's own mechanisms are handled automatically); anything set up manually in `ngOnInit`/`ngAfterViewInit` needs matching manual teardown in `ngOnDestroy`.

### Testing with TestBed

- **DO:** Use `TestBed.configureTestingModule` to construct a minimal, purpose-built testing module per test suite — providing only the specific dependencies (real, or mocked/stubbed) a component or service actually needs — rather than importing an entire application module (with all of its unrelated providers and declarations) into every test. A minimal testing module makes it clear, from the test file alone, exactly what a component depends on, and keeps test setup fast.
- **DON'T:** Write component tests that assert on a component's internal TypeScript properties (`component.someInternalFlag`) instead of on the rendered DOM output, mirroring the same implementation-detail-coupling concern raised for React/Vue testing above. Query the component's rendered template (via `fixture.debugElement.query`, or Angular Testing Library's accessible queries) and assert on what's actually displayed and how it responds to simulated user interaction.
- **DO:** Call `fixture.detectChanges()` deliberately at the points in a test where Angular would normally run change detection (after setting an input, after a simulated user event) rather than either never calling it (leaving the template out of sync with the component's actual current state) or calling it reflexively after every single line without understanding why it's needed at that point.

## Svelte/SvelteKit

### Reactivity Model

- **DO:** Understand Svelte's compiler-driven reactivity model: in Svelte 4 and earlier, a top-level `let` variable reassignment triggers reactivity, and `$:` labeled statements re-run when the values they reference change; in Svelte 5, runes (`$state`, `$derived`, `$effect`) make reactivity explicit and consistent between `.svelte` files and plain `.svelte.js`/`.svelte.ts` modules. Know which model a given codebase uses before writing new code — the two are not source-compatible without migration.
```svelte
<!-- Svelte 5 runes -->
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  function increment() { count += 1; }
</script>
<button onclick={increment}>{count} (doubled: {doubled})</button>
```
- **DON'T:** Mutate a Svelte 4 top-level variable through a method call that doesn't itself involve an assignment recognized by the compiler (e.g., `items.push(x)` alone, with no follow-up assignment). Svelte 4's reactivity is triggered by assignment expressions the compiler can statically see, not by arbitrary mutation, so `array.push()`/`.splice()`/`.sort()` alone do not trigger a UI update; either reassign the variable afterward (`items = items`) or, preferably, use a non-mutating operation that produces a new array/object.
```svelte
<script>
  let items = [1, 2, 3];
  function addBad(x) {
    items.push(x); // compiler doesn't see an assignment — UI won't update
  }
  function addGood(x) {
    items = [...items, x]; // assignment triggers reactivity
  }
</script>
```
- **DO:** Prefer Svelte 5 runes (`$state`, `$derived`) for new projects/components where available, since they make reactive mutation tracking explicit (a `$state` array's mutating methods are tracked correctly, e.g., `.push()` on a `$state([])` array does trigger updates) and work identically inside and outside `.svelte` files, unlike the `$:` label syntax which only has meaning inside a component's `<script>` block.
- **DON'T:** Overuse `$:` reactive statements (Svelte 4) for logic that has side effects with dependencies the compiler can't fully infer (accessing a value only inside a conditional branch, or through a function call whose internal reads aren't visible to the compiler's static dependency analysis). The compiler determines a `$:` block's dependencies by statically scanning which variables it references at the top level of the statement; logic hidden behind indirection can silently fail to re-run when its true dependencies change. Reference the dependent variables directly and explicitly in the reactive statement.
- **DO:** Reach for `$effect` (Svelte 5) or `$:` (Svelte 4) only for genuine side effects (DOM manipulation outside Svelte's control, subscribing to an external non-store data source, logging, syncing to `localStorage`) — not for computing a derived value, which belongs in `$derived`/a plain `$:` assignment instead. This mirrors the React guidance against using `useEffect` to compute values that could be derived directly.
- **DON'T:** Create infinite reactive loops by having a `$effect`/`$:` block both read and write the same reactive variable it's supposed to be reacting to, without a guard. An effect that sets state based on that same state's own value re-triggers itself every time it runs, either causing a runaway loop or (in Svelte 5, which has some infinite-loop protection) a silently dropped update; keep effects one-directional — read some values, write a different one.

### Store Usage

- **DO:** Use Svelte's built-in store contract (`writable`, `readable`, `derived`) for state that's shared across components that aren't in a direct parent-child relationship, and access a store's current value in a component with the `$store` auto-subscription syntax rather than manually calling `.subscribe()`. Auto-subscription handles subscribing on component init and unsubscribing on destroy automatically, exactly like the `async` pipe does in Angular templates.
```svelte
<!-- store.js -->
<script context="module">
  import { writable } from 'svelte/store';
  export const cart = writable([]);
</script>

<!-- Component.svelte -->
<script>
  import { cart } from './store.js';
</script>
<p>{$cart.length} items in cart</p>
```
- **DON'T:** Manually call `store.subscribe(callback)` inside a component without storing and later calling the returned unsubscribe function (or without using the `$store` syntax, which handles this automatically). A manual subscription left uncleaned keeps the component's callback registered on the store after the component is destroyed, leaking memory and potentially updating state on an unmounted component, mirroring the same class of bug as an uncleaned React effect subscription or Angular `Observable` subscription.
```svelte
<script>
  import { onDestroy } from 'svelte';
  import { cart } from './store.js';
  let items;
  const unsubscribe = cart.subscribe((value) => (items = value));
  onDestroy(unsubscribe); // required when not using $cart auto-subscription
</script>
```
- **DO:** Derive computed store values with `derived()` instead of manually subscribing to a source store and writing the derived value into a second `writable` store by hand. `derived()` handles subscription lifecycle and recomputation automatically and communicates intent (this store's value depends on that one) directly in the store's definition.
- **DON'T:** Put every piece of shared state into global Svelte stores by default, including state that's really scoped to one route or one feature and doesn't need to survive navigation away from it. A module-level store persists for the lifetime of the page/app unless explicitly reset; feature-scoped state that's never cleared can leak stale data into a feature the next time the user visits it. Scope stores to the component tree (via Svelte's context API, `setContext`/`getContext`) or reset them explicitly on teardown when they shouldn't be truly global.
- **DO:** Choose between Svelte 5 runes-based shared state (an exported `$state` object/class from a `.svelte.js` module) and the classic store contract deliberately: runes-based shared state is simpler for plain synchronous shared state in a Svelte-5-only codebase, while the store contract remains the more portable, framework-documented option and is still necessary for constructs like `readable` stores wrapping external event sources. Don't mix both patterns for the same piece of state within one feature — pick one so consumers know whether to read `$state` directly or subscribe with `$`.
- **DON'T:** Export a mutable store and let any component that imports it call arbitrary mutating methods on its contents, with no controlled update API. This makes it impossible to trace, from the store's definition alone, every way its value can change; expose specific update functions alongside (or instead of) the raw store (`addItem`, `removeItem`, `clearCart`) so all mutations are named, discoverable, and easy to grep for.

### SvelteKit Conventions

- **DO:** Use SvelteKit's `load` functions (in `+page.js`/`+page.server.js`/`+layout.js`) to fetch data needed for a route before the page renders, taking advantage of automatic request deduplication, streaming, and the framework's built-in handling of loading/error states at the routing layer, instead of fetching data ad hoc inside `onMount` in the page component.
- **DON'T:** Fetch data that's needed for the initial render inside a component's `onMount` when a `load` function would do it server-side (or at least before the component mounts). `onMount`-based fetching means the user sees an empty/loading UI on first paint even when the framework could have fetched and rendered the data as part of the initial response, and it forfeits SvelteKit's built-in loading and error boundary conventions (`+page.ts`'s errors flowing to `+error.svelte`).
- **DO:** Put secrets and privileged logic exclusively in `+page.server.js`/`+layout.server.js`/`+server.js` files (which never ship to the client), and keep `+page.js`/`+layout.js` (universal, run on both server and client) free of anything that must not be exposed in the browser bundle. SvelteKit's naming convention encodes exactly where a given piece of code executes; using the wrong file suffix for privileged logic exposes it to the client.
- **DON'T:** Perform data mutations from the client with hand-rolled `fetch()` calls to custom API endpoints when SvelteKit's form actions (`+page.server.js`'s `actions` export, invoked via a plain `<form>` with progressive enhancement through `use:enhance`) already provide a mutation pathway that works without client-side JavaScript and integrates with SvelteKit's validation/error-return conventions. Reach for hand-rolled `fetch` to an API route for cases actions genuinely don't fit (e.g., a third-party webhook receiver, a non-form-triggered background sync).

### hooks.server.js and Server-Side Concerns

- **DO:** Put cross-cutting server-side logic that needs to run on every request — authentication/session parsing, request logging, response header injection — in `hooks.server.js`'s `handle` function, rather than duplicating that logic at the top of every individual `+page.server.js`/`+server.js` route. Centralizing it in `hooks.server.js` guarantees it actually runs consistently for every request, instead of depending on every route author remembering to call it.
- **DON'T:** Perform expensive, blocking work inside `handle` for routes that don't actually need it (e.g., a full session/user lookup against the database on every single request, including requests for static assets or public pages that don't require authentication). Guard expensive hook logic with a check on the request path/route so it only runs where it's actually needed, since `handle` sits in the critical path of every single request the server processes.
- **DO:** Use `event.locals` to pass data computed in `hooks.server.js` (like the resolved authenticated user) down to `load` functions and route handlers, giving them typed, request-scoped access to that data without needing to recompute or re-fetch it themselves.

### Actions, Transitions, and Snippets

- **DO:** Use Svelte actions (`use:action`) to encapsulate reusable, imperative DOM-interaction logic (attaching a third-party library to a node, a custom click-outside detector, a tooltip-positioning library) that doesn't map cleanly onto Svelte's declarative component model. Actions give a clean lifecycle (an initial call, an optional `update` when parameters change, and a `destroy` for cleanup) for exactly this category of imperative DOM work, mirroring what a `useEffect` with cleanup or an Angular directive does in their respective frameworks.
```svelte
<script>
  function clickOutside(node, callback) {
    function handleClick(e) {
      if (!node.contains(e.target)) callback();
    }
    document.addEventListener('click', handleClick, true);
    return { destroy() { document.removeEventListener('click', handleClick, true); } };
  }
</script>
<div use:clickOutside={closeMenu}>...</div>
```
- **DON'T:** Write imperative DOM manipulation directly inside component logic (querying and mutating nodes by hand outside of Svelte's reactive markup) when an action, a bound element reference (`bind:this`), or ordinary reactive markup would achieve the same result more declaratively. Reserve manual DOM manipulation for the specific cases (non-Svelte-aware third-party libraries, imperative browser APIs with no declarative equivalent) an action pattern exists to encapsulate.
- **DO:** Use Svelte's built-in transition directives (`transition:`, `in:`, `out:`) for entering/leaving DOM elements instead of hand-rolling CSS class toggling plus manually-timed `setTimeout` calls to remove elements after an animation completes. Svelte's transitions are integrated with the compiler's understanding of when an element is actually being added or removed from the DOM, which correctly handles interruption (an element that starts leaving and then re-enters mid-animation) in a way manual timing code easily gets wrong.
- **DON'T:** Apply layout-affecting transition properties (animating `height`/`width`/`margin` directly) for frequent or performance-sensitive transitions, for the same reason raised in the general Performance section — prefer `transform`/`opacity`-based transitions (Svelte's built-in `fade`, `fly`, `scale` transitions are compositor-friendly by default) unless a layout-affecting transition (like the built-in `slide`) is specifically needed and its cost is acceptable for the frequency it runs at.
- **DO:** Use Svelte 5 snippets (`{#snippet}`/`{@render}`) as the modern replacement for slots when passing reusable, parameterized chunks of markup into a component, since they're more explicit about their parameters and composable as first-class values (can be passed around, stored, and rendered conditionally) compared to the more implicit, magic-named slot props of Svelte 4.

### Testing Svelte Components

- **DO:** Use `@testing-library/svelte` to test Svelte components through rendered output and accessible queries, following the exact same "test behavior, not implementation" principle already covered for React and Vue testing above — the underlying philosophy is framework-agnostic even though the specific testing library differs.
- **DON'T:** Test a component's reactive `$:` statements or `$state`/`$derived` values by reaching into the component instance's internals. Trigger the same interactions a real user would (simulated input, simulated clicks) and assert on the resulting rendered DOM, which validates the actual reactive chain end-to-end rather than just one internal link in it.

## State Management

### Selection Criteria

- **DO:** Classify state before choosing a tool for it. Four categories cover most UI state and each has a different natural home: **local/UI state** (a dropdown's open flag, a hovered row, an input's draft value — component-local `useState`/`ref`/component fields), **shared client state** (theme, authenticated user, feature flags, a shopping cart — a lightweight store or context), **server/remote state** (anything fetched from an API — a dedicated caching library), and **URL state** (the current page, filters, sort order, a selected tab that should survive a refresh or be shareable via link — the URL/router itself, not a JS store). Picking a state category correctly upfront avoids most of the "why is my state duplicated/stale/out of sync" bugs that follow.
- **DON'T:** Default to the most powerful, most global tool available (a full app-wide store, or Redux with every possible middleware) for state that's actually local or server-derived, just because it's already wired up elsewhere in the app. The cost of an oversized tool isn't hypothetical — it shows up as unnecessary re-renders, stale-cache bugs the tool wasn't designed to solve, and a steeper onboarding curve for every new contributor who has to learn the global store's shape before touching an isolated feature.
- **DO:** Put anything that should survive a page refresh, be bookmarkable, or be shareable via a copy-pasted link — active filters, the open tab, a search query, pagination — into the URL (query params or route segments) rather than into in-memory JS state alone. In-memory-only state for these values means a refresh silently discards the user's context and a shared link doesn't reproduce what the sender was looking at.
- **DON'T:** Duplicate server-fetched data into a separate global client store "for convenience" once a dedicated server-state library (React Query/TanStack Query, SWR, Apollo Client, RTK Query, VueQuery) is already fetching and caching it. Two sources of truth for the same server data (the cache's copy and a manually-synced store copy) inevitably drift, especially across cache invalidation/refetch events the manual copy doesn't know about; read directly from the query cache/hook wherever the data is needed, or use the query library's own derived-selector mechanism.
- **DO:** Weigh a state library primarily on: how granular its subscriptions are (does a component re-render only when the specific slice it reads changes, or on any store change), how well it's typed end-to-end, how much boilerplate a typical read/write requires, and whether it has first-class devtools for time-travel/inspection during debugging. These four factors predict day-to-day developer experience far better than raw popularity or benchmark throughput numbers.
- **DON'T:** Introduce a second, competing state-management library into a codebase that already has one doing the same job, just because a new contributor prefers it or a tutorial used it. Two libraries solving the same problem (e.g., both Redux and Zustand both holding pieces of client UI state) forces every future contributor to learn both and to guess which one a given piece of state lives in; migrate deliberately and completely, or don't migrate at all.
- **DO:** Favor atomic/fine-grained state primitives (Jotai atoms, Recoil-style, Vue's `ref`/`computed`, Svelte 5 runes, Solid's signals) for UI-heavy apps with many independently-updating small pieces of state, since they let each consuming component subscribe to exactly the atoms it reads, without needing manual selector functions to avoid over-subscribing to a shared store object.
- **DON'T:** Treat "we might need to scale this later" as sufficient justification for adopting the heaviest available state architecture on day one of a small project. Start with the simplest tool that solves the actual current problem (often just component state plus a data-fetching library); introduce a heavier shared-state tool when a real cross-cutting sharing need appears, since retrofitting is a well-understood, incremental refactor, while premature complexity is a sunk cost paid on every subsequent feature regardless of whether it was ever needed.

### Choosing Among Popular Libraries

- **DO:** Reach for Redux Toolkit (not legacy hand-written Redux with manual action-type constants and switch-based reducers) when a project genuinely needs Redux's specific strengths: a large team that benefits from a single, strict, well-documented convention for every state change; deep, reliable time-travel debugging; or an existing large Redux codebase that a new feature needs to remain consistent with. Redux Toolkit's `createSlice` eliminates almost all of legacy Redux's boilerplate (action creators, action type strings, manual immutability) while keeping its core meaningful guarantees.
- **DON'T:** Adopt Redux (in any form) as a default for a new, small-to-medium project without another Redux-specific requirement driving the choice. Its explicit action/reducer/dispatch cycle is valuable specifically for the debugging and predictability guarantees it buys, but that structure is pure overhead relative to a simpler tool for a project that doesn't need those specific guarantees.
- **DO:** Reach for Zustand (or a similarly minimal store library) when a project wants a small, un-opinionated global store with fine-grained selector-based subscriptions and almost no boilerplate — a good default for small-to-medium apps that need some shared client state but don't want Redux's ceremony or Context's re-render-granularity problems.
- **DO:** Reach for an atomic state library (Jotai, Recoil-style) when a UI is composed of many small, independently-updating pieces of state that naturally decompose into individual "atoms" rather than one shared object tree — a good fit for highly interactive, widget-dense UIs (a spreadsheet, a form builder, a complex settings page) where forcing every piece of state into one shared store object creates awkward, overly-coupled selectors.
- **DON'T:** Use a general-purpose state store (Redux, Zustand, Context) to model a UI flow that is fundamentally a finite state machine — a multi-step wizard, a media player's play/pause/buffering/error states, an upload's pending/uploading/success/failed states. Modeling these as a loose bag of independent booleans/strings in a general store reproduces the "impossible state combinations" problem raised earlier; a dedicated state-machine library (XState, or a hand-rolled discriminated-union reducer for simpler cases) makes the valid states and valid transitions between them explicit and prevents invalid combinations from being representable at all.
```ts
// GOOD — impossible combinations (e.g., "uploading" and "failed" at once)
// are unrepresentable by construction
type UploadState =
  | { status: 'idle' }
  | { status: 'uploading'; progress: number }
  | { status: 'success'; url: string }
  | { status: 'failed'; error: string };
```
- **DO:** Choose a dedicated form-state library (React Hook Form, Formik, VeeValidate, Angular's Reactive Forms) rather than modeling per-field form state by hand in a general-purpose global store. Form state has its own specific needs — per-field dirty/touched tracking, validation timing, submission state, array-field manipulation — that a generic store doesn't provide out of the box and that a form-specific library has already solved.
- **DON'T:** Put form-in-progress state (the current, unsaved value of every field in a multi-step form) into the same global store as durable application data, without at least a clear boundary and an explicit "commit" step. Blurring in-progress, potentially-invalid draft state together with confirmed, saved application state makes it easy to accidentally treat a draft value as if it were already saved, or to have partially-typed invalid data leak into parts of the UI that assume only valid data lives in that store.

### Optimistic Updates and Undo/Redo

- **DO:** Implement optimistic updates (showing the result of a mutation immediately, before the server confirms it) for interactions where a fast perceived response matters and failures are rare and recoverable (liking a post, checking off a to-do item, reordering a list) — most server-state libraries (React Query, SWR, RTK Query) have first-class support for this pattern including automatic rollback on failure.
- **DON'T:** Apply optimistic updates to mutations where failure is common, where the consequence of a visible rollback would be jarring or confusing (a payment submission, an irreversible destructive action), or where the update can't be cleanly rolled back if the server rejects it. For these cases, show a clear pending/loading state and wait for actual server confirmation before updating the UI, rather than showing a result the user might see reversed a moment later with no clear explanation why.
- **DO:** Always implement the rollback path for an optimistic update — reverting the UI to its prior state and surfacing an error — with the same care as the optimistic-apply path itself. An optimistic update that always assumes success and has no tested failure/rollback behavior leaves the UI in a permanently incorrect state (out of sync with the actual server data) the first time a mutation genuinely fails.
- **DO:** Model undo/redo functionality as an explicit history of state snapshots or, for more complex state, an explicit list of invertible actions/commands, rather than trying to bolt undo onto an ad hoc mutable store after the fact. Undo/redo is much easier to build correctly when it's designed into the state architecture from the start (e.g., a reducer whose actions are all easily invertible) than retrofitted onto state that's been freely, directly mutated all over the codebase.

## Component Architecture

### Component Design Principles

- **DO:** Give every component a single, clearly nameable responsibility — the classic single-responsibility principle applied to UI. A component named `UserCard` should render a user card; if it also fetches its own data, manages a modal's global visibility, and formats currency inline, those are three additional responsibilities that belong in separate, composable pieces.
- **DON'T:** Let a component's prop list, internal state, or file length grow unbounded as "just one more flag" gets added every sprint. A component that started as a simple `Button` and has accumulated fifteen boolean props, three render-prop slots, and four different internal state machines over a year of feature requests should be recognized as a signal to refactor into variants or sub-components, not as normal growth to keep patching.
- **DO:** Separate "container" (or "smart"/"connected") components that own data fetching, state, and business logic from "presentational" (or "dumb") components that only receive data via props and render UI, with no knowledge of where that data came from. This separation makes presentational components trivially reusable and testable (render with props, assert output — no mocking a store or API), while containers stay focused on wiring, without also encoding markup and styling decisions.
```jsx
// Presentational — no idea where `user` comes from, pure function of props
function UserCard({ user, onFollow }) {
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <button onClick={() => onFollow(user.id)}>Follow</button>
    </div>
  );
}

// Container — owns the data, delegates rendering
function UserCardContainer({ userId }) {
  const { data: user } = useUser(userId);
  const follow = useFollowMutation();
  if (!user) return <UserCardSkeleton />;
  return <UserCard user={user} onFollow={follow.mutate} />;
}
```
- **DON'T:** Build a component's public API (props/inputs) around its current single call site's exact needs, encoding that call site's specific conditional logic as special-case props (`showExtraButtonForCheckoutPageOnly`). Design the component's contract around the general concept it represents; if a second call site needs different behavior, extend the API with a general-purpose mechanism (a slot, a variant, a composable sub-part), not a single-purpose flag named after the page that requested it.
- **DO:** Make invalid states unrepresentable in a component's prop types wherever practical — e.g., type a component's status as a discriminated union (`{status: 'loading'} | {status: 'error', message: string} | {status: 'success', data: T}`) rather than accepting independent `isLoading`, `error`, and `data` props that can be inconsistently combined (both `isLoading` and `data` truthy at once). A well-typed prop shape turns a class of runtime bugs into compile-time type errors.
- **DON'T:** Reach immediately for a new abstraction (a generic `<DataTable>` that tries to handle every table in the app, a `renderItem` prop threading through five layers) the very first time similar-looking markup appears twice. Two similar-but-not-identical usages are often cheaper to leave as two separate, slightly-duplicated components until a third and fourth usage reveal the actual shared shape; a premature abstraction built from a sample size of two tends to need invasive prop-API surgery by the third real use case.
- **DO:** Keep components pure with respect to rendering — the same props (and state) should produce the same rendered output, with side effects isolated to explicit lifecycle hooks/effects and event handlers, not scattered into the render body. A render function that mutates a module-level variable or calls `Math.random()`/`Date.now()` directly in its body produces output that can't be reasoned about, tested predictably, or safely re-rendered by concurrent rendering features.

### Folder Structure Conventions

- **DO:** Organize a non-trivial frontend codebase by feature/domain (a `features/checkout/`, `features/user-profile/` structure, each folder self-contained with its own components, hooks, API calls, and types) rather than by technical layer alone (`components/`, `hooks/`, `services/`, `types/` as top-level folders holding every feature's files interleaved). Feature-based structure means a contributor working on checkout only needs to look inside `features/checkout/`, while a layer-based structure scatters one feature's code across four or five top-level directories that must all be found and cross-referenced.
```
// GOOD — feature-first structure
src/
  features/
    checkout/
      components/
      hooks/
      api.ts
      types.ts
    user-profile/
      components/
      hooks/
      api.ts
  shared/
    components/   (genuinely cross-feature UI primitives)
    hooks/
    lib/
```
- **DON'T:** Let "shared"/"common"/"utils" folders become an unstructured dumping ground for anything that doesn't obviously belong elsewhere. An unbounded `utils/` folder with dozens of unrelated one-off functions is effectively unsearchable and tends to accumulate near-duplicate helpers because nobody can find the existing one before writing a new one; subdivide by actual purpose (`lib/formatting/`, `lib/validation/`) or, better, colocate a helper with the single feature that uses it until a second feature genuinely needs it too.
- **DO:** Colocate a component's styles, tests, and Storybook stories with the component itself (`Button.tsx`, `Button.test.tsx`, `Button.stories.tsx`, `Button.module.css` in the same folder) rather than mirroring the source tree in parallel `tests/` and `styles/` directories. Colocation means deleting or moving a component naturally deletes or moves everything associated with it, and a contributor editing a component sees its tests and styles right next to it in the file tree.
- **DON'T:** Create deeply nested folder hierarchies (five or six directory levels before reaching an actual file) purely to mirror an org chart or an over-decomposed taxonomy. Excessive nesting makes import paths long and fragile to refactor and adds cognitive overhead to "where does this go" decisions with no compensating benefit; prefer a flatter structure with clear, few top-level categories (feature, shared, app-shell/routing) and let file naming do more of the organizational work than folder depth.
- **DO:** Establish and document one barrel-export (`index.ts` re-export) convention — either consistently used at each feature/module boundary to define its public API, or consistently avoided in favor of direct imports — rather than mixing both approaches unpredictably across the codebase. Inconsistent use of barrel files makes it unclear, without checking, whether a given import path is meant to be "the module's public API" or an internal implementation detail a caller shouldn't reach into directly.
- **DON'T:** Reach into another feature folder's internal files directly (`import { formatCheckoutDate } from '../checkout/components/internal/utils'`) bypassing its intended public entry point. Deep imports into another feature's internals create a hidden coupling that breaks the moment that feature's internal structure is refactored, even though its public API didn't change; either promote the needed utility to a shared module both features can depend on, or expose it through the owning feature's public export surface.

### Avoiding God-Components

- **DO:** Watch for the concrete warning signs of a god-component: a file exceeding a few hundred lines with no clear internal sectioning, more than eight or so distinct pieces of `useState`/reactive state that don't obviously group together, a component that both fetches data and deeply understands multiple unrelated child components' internal prop shapes, or a component whose diff touches something in nearly every PR because everyone's feature needs to add "just one more thing" to it.
- **DON'T:** Let a top-level page or route component accumulate all of a page's data fetching, all of its local UI state, and all of its layout markup in one file as the app grows past its initial prototype. Break it into a thin page shell that composes purpose-specific child components (`<Header>`, `<FilterBar>`, `<ResultsList>`, `<Pagination>`), each owning its own relevant slice of state and markup, so the page component itself stays a readable table of contents for the feature.
- **DO:** Extract a section of a large component into its own named component as soon as that section has an internally cohesive block of state/logic that doesn't interact with the rest of the parent beyond a couple of shared values passed as props. A `useReducer` with five action types dedicated to a filter panel embedded inside a 600-line page component is a strong signal that the filter panel should be its own component with its own reducer.
- **DON'T:** Solve a god-component by mechanically splitting its JSX into smaller files while still threading all of its original state and callbacks through props from the original monolithic parent (extracting markup without extracting responsibility). This often just relocates the same tight coupling one level deeper and adds prop-drilling on top of it; extract logic and its owning state together with the JSX that uses it, or lift only the state that genuinely needs to be shared back up.
- **DO:** Give a large feature a small number of well-defined seams — a container that owns cross-cutting state and orchestration, and leaf presentational components with narrow, well-typed prop interfaces — so that no single file needs to be understood in full to make a typical change. A contributor fixing a bug in the results-list rendering shouldn't need to read the data-fetching and filter-state logic to do it safely.

### CSS-in-JS vs Utility CSS

- **DO:** Pick a styling approach based on the team's actual constraints — runtime performance sensitivity, design-system consistency needs, TypeScript integration, and how much the team values colocated styles versus a constrained utility vocabulary — rather than by current trend alone. CSS-in-JS (styled-components, Emotion), utility-first CSS (Tailwind), CSS Modules, and vanilla-extract/zero-runtime CSS-in-JS each trade off differently on runtime cost, authoring ergonomics, and how much they constrain arbitrary one-off styling.
- **DON'T:** Adopt a runtime CSS-in-JS library (one that generates styles via JavaScript executing in the browser, as opposed to build-time-extracted CSS) for a performance-sensitive app without accounting for its runtime cost: style injection and serialization on every render of a dynamically-styled component adds real CPU work, and can be a measurable contributor to slow first paint and interaction latency on lower-end devices, especially at scale. If runtime CSS-in-JS is used, keep dynamically-computed styles to a minimum and prefer static, class-based composition for anything that doesn't need to change based on props/state.
- **DO:** Constrain a utility-CSS (Tailwind-style) design surface with a shared config (a fixed spacing scale, a fixed color palette, fixed breakpoints) and enforce it, rather than letting arbitrary-value utility classes (`w-[437px]`, `text-[#3a7bd5]`) proliferate throughout the codebase. Unconstrained arbitrary values defeat the main benefit of a utility system — a consistent, small, reusable design vocabulary — and reproduce the same inconsistency problem hand-written CSS had, just in class-attribute form instead of stylesheet form.
```jsx
// BAD — arbitrary one-off values bypass the design system's scale
<div className="mt-[13px] text-[15.5px] text-[#3b82f680]">

// GOOD — values drawn from the shared design tokens/scale
<div className="mt-3 text-sm text-primary/50">
```
- **DON'T:** Let a long, unreadable string of utility classes stand in for what should be a named, reusable component when the same combination of utility classes is copy-pasted across many call sites. Repeated utility-class strings are exactly the kind of duplication a component (or a framework-level style-composition helper, e.g., a `cva`/class-variance-authority variant definition) exists to eliminate; extract a component once a class combination is copied more than a couple of times, so a future design change is a one-line edit instead of a find-and-replace across the codebase.
- **DO:** Keep component-scoped styles (CSS Modules, `<style scoped>`, styled-components) the default for one-off, component-specific visual details, and reserve global stylesheets for genuinely global concerns (CSS resets/normalize, design tokens as CSS custom properties, base typography). Mixing arbitrary global selectors freely with component-scoped styles reintroduces the specificity and cascade conflicts that scoped styling approaches exist to prevent.
- **DON'T:** Hand-roll one-off color, spacing, or typography values (raw hex codes, raw pixel values) inline in component styles when a design-token system (CSS custom properties, a Tailwind theme config, a design-system package's exported tokens) already defines the approved values. Ad hoc values drift the UI away from the design system incrementally, one component at a time, until the app's actual rendered colors/spacing no longer match what the design system claims to define.

### Bundle Size Discipline

- **DO:** Check a new dependency's bundle-size cost (via a tool like Bundlephobia, or the bundler's own analyzer) before adding it, and prefer smaller, more focused libraries or a hand-written utility over pulling in a large general-purpose library for one narrow use. Importing an entire 80KB date library to format one timestamp, when the app already ships a smaller utility or the native `Intl.DateTimeFormat` would do, is paid by every user on every page load that touches the bundle.
- **DON'T:** Import an entire library's default export when only a handful of its functions are actually used, on a library that doesn't tree-shake cleanly (older CommonJS-style utility libraries, some icon packs). `import _ from 'lodash'` can pull in the whole library even if only `_.debounce` is called; import the specific function directly (`import debounce from 'lodash/debounce'`, or better, a tree-shakeable ESM-only alternative) so bundlers can eliminate the unused code.
- **DO:** Run a bundle analyzer (webpack-bundle-analyzer, Vite's `rollup-plugin-visualizer`, Next.js's built-in analyzer) periodically, and especially before and after adding a heavy dependency, to catch unexpectedly large additions before they ship. Bundle size regressions are easy to introduce silently (a transitive dependency update pulling in a much larger version) and easy to catch early with a visual breakdown, but expensive to notice after the fact once users are already downloading the bloat.
- **DON'T:** Import a large icon library's entire icon set (`import * as Icons from 'huge-icon-library'`) when only a handful of specific icons are used. Import each icon individually from its specific module path, or use an SVG-sprite/inline-SVG approach for a small fixed icon set, so unused icons don't ship to every user.
- **DO:** Set an actual bundle-size budget (a maximum JS payload for the initial route, enforced in CI via a tool like `bundlesize`, `size-limit`, or the framework's built-in budget config) once an app matters to real users on real networks. A budget makes bundle growth a visible, reviewed decision at PR time instead of an invisible creep that's only noticed once load times have already degraded for months.
- **DON'T:** Ship a polyfill bundle unconditionally to every browser, including modern evergreen browsers that don't need it. Use differential/targeted polyfilling (serving polyfills only to browsers that lack the feature, via `browserslist`-aware tooling or `<script type="module">`/`nomodule` splitting) so the majority of users on modern browsers aren't paying download and parse cost for compatibility code they'll never execute.

### Design Tokens and Theming

- **DO:** Define design tokens (color, spacing, typography scale, radii, shadow, motion-duration values) as a single source of truth — CSS custom properties, a theme object, or a design-system package's exported constants — and have every component reference tokens rather than hardcoded values. A token layer means a single rebrand/redesign edit (changing the primary color token) propagates everywhere automatically instead of requiring a find-and-replace across hundreds of hardcoded hex codes.
```css
/* GOOD — components reference tokens, never raw values */
:root {
  --color-primary: #3b82f6;
  --space-sm: 0.5rem;
  --radius-md: 0.375rem;
}
.button { background: var(--color-primary); padding: var(--space-sm); border-radius: var(--radius-md); }
```
- **DON'T:** Implement dark mode (or any alternate theme) as a parallel, hand-duplicated set of component styles maintained alongside the light-mode styles. Duplicated per-theme stylesheets drift out of sync as components change over time; implement theming by swapping token *values* (e.g., redefining the same custom properties under a `[data-theme="dark"]` selector or `prefers-color-scheme` media query) while component styles themselves reference the tokens once and never change between themes.
- **DO:** Verify color contrast independently for every theme a design system ships (light, dark, high-contrast) rather than assuming a contrast ratio validated in light mode automatically holds after color values are swapped for dark mode. Token swaps for dark mode are a common place for contrast regressions to sneak in unnoticed, since visual review in one theme doesn't catch failures that only manifest in the other.
- **DON'T:** Let component-level styling silently drift from the design system's declared tokens over time (a developer eyeballing "close enough" spacing instead of using the actual scale value). Lint for hardcoded color/spacing values where a token exists (a stylelint rule or a custom lint check) so drift is caught in code review rather than accumulating invisibly release after release.

### Storybook and Component Documentation

- **DO:** Maintain a living component catalog (Storybook, or an equivalent tool) for a shared component library, with stories covering each component's meaningful states (default, loading, error, empty, disabled, various sizes/variants) — not just its default happy-path appearance. A catalog that only shows the default state doesn't actually help a consumer discover what states exist or verify a component looks right in the state their feature needs.
- **DON'T:** Let a component library's Storybook stories drift out of sync with the actual component implementation (stories referencing removed props, or missing stories for props added since the catalog was last updated). A stale catalog is worse than no catalog for a new contributor, since it actively teaches an incorrect API; treat updating a component's stories as part of the same PR that changes its props/behavior, not a separate, easily-deferred chore.
- **DO:** Use a component catalog as a tool for isolated accessibility and visual-regression testing (many Storybook addons integrate axe-core accessibility checks and visual diffing directly into the catalog) in addition to its documentation role, catching component-level regressions earlier and more cheaply than only ever testing components in the context of a full page.

### Internationalization Considerations

- **DO:** Route all user-facing text through an internationalization (i18n) library (react-i18next, vue-i18n, Angular's i18n tooling, or a similar message-catalog system) from the start of a project expected to support more than one language, rather than hardcoding strings directly in JSX/templates and retrofitting i18n later. Retrofitting i18n onto a codebase with hundreds of hardcoded strings scattered through markup is a large, error-prone, easy-to-undercount migration; starting with an i18n layer costs little extra for a single-locale app and avoids that migration entirely if a second locale is ever needed.
- **DON'T:** Concatenate translated string fragments together in code to build a sentence (`t('you_have') + count + t('items')`), since word order, pluralization rules, and grammatical gender vary by language in ways that make fragment concatenation produce grammatically broken or nonsensical output in many target languages. Use the i18n library's interpolation and pluralization features (ICU MessageFormat or equivalent) so an entire sentence is one translatable unit with named placeholders, not several string fragments stitched together in source-language order.
```js
// BAD — assumes source-language word order and pluralization
const msg = t('you_have') + ' ' + count + ' ' + t(count === 1 ? 'item' : 'items');

// GOOD — one full translatable string with proper pluralization support
const msg = t('itemCount', { count }); // catalog: "You have {{count}} item(s)" with plural rules
```
- **DO:** Design layouts to tolerate significant text-length variation between languages (German and Finnish strings routinely run 30–40% longer than their English equivalents; some UI labels in Asian languages run shorter but with different line-wrapping behavior) rather than sizing containers tightly around the English string's exact pixel width. A button or label that only fits its English text overflows or truncates unpredictably once translated.
- **DON'T:** Assume left-to-right reading order and layout direction as a given if the product needs to support right-to-left (RTL) languages (Arabic, Hebrew). Use logical CSS properties (`margin-inline-start` instead of `margin-left`, `padding-inline-end` instead of `padding-right`) and the `dir` attribute rather than hardcoded physical-direction properties, so the layout mirrors correctly under RTL without needing a parallel, hand-maintained RTL stylesheet.
- **DO:** Format dates, numbers, currencies, and pluralization using locale-aware APIs (`Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.PluralRules`, or the i18n library's equivalents) instead of hand-formatting them with string concatenation and hardcoded separators. Manually formatted numbers/dates encode source-locale conventions (comma as a thousands separator, MM/DD/YYYY date order) that are simply wrong in many other locales.

### Responsive Design Strategy

- **DO:** Design and build mobile-first — writing base styles for the smallest supported viewport and layering on progressively larger-viewport overrides via `min-width` media queries — rather than desktop-first with `max-width` overrides stripping features away for small screens. Mobile-first tends to produce simpler, more resilient CSS because base styles handle the more constrained case, and enhancements are additive rather than subtractive.
- **DON'T:** Hardcode pixel-based breakpoints inconsistently across a codebase (one component breaking at 768px, another at 767px, another at 800px for what's meant to be the same "tablet" breakpoint). Define a shared, named breakpoint scale (in the design tokens/theme config) and have every responsive style reference those shared values, so the app's responsive behavior is consistent and a single breakpoint adjustment updates the whole app at once.
- **DO:** Use relative, content-based responsive techniques (CSS Grid's `auto-fit`/`auto-fill` with `minmax()`, Flexbox wrapping, container queries) where they solve the actual layout need, rather than reaching for a fixed breakpoint media query as the default tool for every responsive decision. A card grid that reflows its column count based on available width via `auto-fit`/`minmax()` adapts smoothly to any container size, including ones a fixed set of breakpoints wouldn't have anticipated (a component embedded in a narrow sidebar, a resizable panel).
- **DON'T:** Design a responsive layout using only viewport-based media queries when a component's actual layout should really respond to the size of its own containing element, not the whole browser viewport (a card component that needs to look different in a wide two-column layout versus a narrow sidebar, regardless of the overall viewport size). Use container queries (`@container`) for this case — a component that only has viewport-based responsive logic behaves wrong the moment it's reused in a differently-sized container than the one it was originally designed inside of.

### Error Boundary and Failure-Domain Placement

- **DO:** Think of a page as a set of independent failure domains — each self-contained widget, panel, or section should be able to fail without taking down unrelated, already-working parts of the same page. This principle applies across frameworks (React error boundaries, Angular's `ErrorHandler` combined with per-feature try/catch at data-loading boundaries, a Vue `errorCaptured` hook scoped to a specific subtree) even though the specific mechanism differs.
- **DON'T:** Let one non-critical, decorative, or supplementary widget's failure (a "related articles" recommendation panel, a third-party embedded widget, an analytics/tracking script) take down or blank out the primary content of the page it's embedded in. Isolate genuinely non-critical embedded content behind its own failure boundary so its failure degrades gracefully (that section disappears or shows a small inline error) rather than cascading.
- **DO:** Report caught errors (from error boundaries, global error handlers, or unhandled promise rejection listeners) to an error-tracking/observability tool with enough context (the route, the relevant component, a user/session identifier where privacy policy allows it) to actually debug the failure later, rather than only showing a fallback UI to the user with no server-side record that the failure happened at all.

### Code-Splitting and Lazy Loading

- **DO:** Split the app's JavaScript by route at minimum, so a user visiting one page doesn't download the code for every other page in the app upfront. Every major framework has a supported mechanism for this — dynamic `import()` combined with `React.lazy`/`Suspense`, Vue Router's lazy route components, Angular's `loadComponent`/`loadChildren`, SvelteKit's automatic per-route splitting — and it's usually close to a "just do this by default" decision with little downside for a multi-page app.
```jsx
// GOOD — the settings page's code loads only when a user navigates there
const SettingsPage = React.lazy(() => import('./pages/SettingsPage'));
<Suspense fallback={<PageSkeleton />}>
  <SettingsPage />
</Suspense>
```
- **DON'T:** Lazy-load content that's needed immediately for the first meaningful paint (above-the-fold hero content, the primary navigation) just to chase a smaller initial-bundle number on a size dashboard. Deferring critical content behind a dynamic import adds a network round trip and a loading flash to the part of the page users see first, which usually costs more in perceived performance than it saves in raw bytes; reserve lazy-loading for content that's genuinely below the fold, behind an interaction, or on a route the user hasn't navigated to yet.
- **DO:** Lazy-load heavy, conditionally-rendered UI — modals, rich-text editors, charting libraries, complex data-grids, anything gated behind a feature flag or a permission check that many users never trigger — so their (often large) dependencies aren't included in the main bundle for users who never open them.
- **DON'T:** Split code so finely (a separate dynamic import for every small component) that the number of small network requests and the fixed per-chunk overhead (module wrapper code, extra round trips, waterfalls of dependent chunk loads) outweighs the benefit of the smaller individual chunks. Chunk granularity is a real tuning knob with a sweet spot, not "smaller is always better" — group logically-related, typically-co-used components into shared chunks rather than code-splitting at the level of every single leaf component.
- **DO:** Preload (not just lazy-load) a route's or feature's code when there's a strong signal the user is about to need it — hovering over a link, focusing an input, or completing a step in a wizard right before the next step's code is needed. This hides the network fetch latency that a naive on-click lazy load would otherwise expose as a visible delay.
- **DON'T:** Forget to provide a loading fallback (a skeleton, a spinner, or at minimum a non-jarring placeholder) for lazy-loaded content, leaving a blank gap or a layout shift while the chunk downloads. An un-fallback'd `Suspense` boundary (or equivalent) that just renders nothing until the chunk resolves reads as a bug to users, especially on slower connections where the gap is visible for a noticeable duration.

## Performance

### Avoiding Layout Thrash

- **DO:** Understand layout thrash (also called "forced synchronous layout" or "layout thrashing") as the pattern of interleaving DOM writes and DOM reads in a loop, forcing the browser to recalculate layout synchronously on every read instead of batching it once per frame. Reading a layout-dependent property (`offsetHeight`, `offsetWidth`, `getBoundingClientRect()`, `scrollTop`, computed styles) immediately after a write forces the browser to flush any pending layout changes early to answer the read accurately.
```js
// BAD — read-write-read-write interleaved, forces layout on every iteration
elements.forEach((el) => {
  const height = el.offsetHeight; // read (forces layout flush)
  el.style.height = `${height * 2}px`; // write
}); // next iteration's read forces layout again, repeated N times

// GOOD — batch all reads first, then all writes
const heights = elements.map((el) => el.offsetHeight); // all reads
elements.forEach((el, i) => { el.style.height = `${heights[i] * 2}px`; }); // all writes
```
- **DON'T:** Read layout-dependent DOM properties inside a loop that also writes to the DOM in the same iteration. Batch all the reads into one pass (into a plain array), then perform all the writes in a second pass, so the browser only needs to compute layout once instead of once per element.
- **DO:** Use the `ResizeObserver` and `IntersectionObserver` APIs to react to size and visibility changes, instead of polling `getBoundingClientRect()`/`offsetWidth` on a `scroll` or `resize` event handler at high frequency. These observer APIs are implemented to batch and schedule their callbacks efficiently around the browser's own layout pipeline, rather than forcing synchronous layout computation on every scroll pixel.
- **DON'T:** Trigger layout-affecting style changes (changing `width`, `height`, `top`, `left`, margin/padding, or anything that isn't purely compositor-friendly) inside a `scroll` or `mousemove` event handler without throttling/debouncing or batching the update via `requestAnimationFrame`. Unthrottled layout-affecting writes on a high-frequency event can trigger many forced layout recalculations per second, visibly dropping frames; prefer animating `transform` and `opacity` (which the browser can typically handle on the compositor thread without triggering layout) and schedule any layout-affecting updates inside `requestAnimationFrame`.
- **DO:** Prefer CSS transitions/animations driven by `transform` and `opacity` over animating layout properties (`top`/`left`/`width`/`height`) directly, since transform/opacity changes typically skip layout and paint entirely and run on the compositor thread, giving smoother animation with less main-thread work.
```css
/* BAD — animating top/left triggers layout on every frame */
.panel { transition: left 200ms, top 200ms; }

/* GOOD — animating transform is compositor-only */
.panel { transition: transform 200ms; }
.panel.open { transform: translateX(0); }
.panel.closed { transform: translateX(-100%); }
```
- **DON'T:** Force layout recalculation just to read a value that could instead be cached or computed without touching the DOM (e.g., re-measuring an element's width on every render when it hasn't actually changed size, or measuring inside a loop when the same measurement is reused for every iteration). Cache layout reads that don't need to be re-taken, and invalidate the cache only on the specific events that could actually change the measurement (resize, content change).
- **DO:** Batch multiple DOM writes together (e.g., by toggling a single class that applies several style changes at once, or by writing to a detached DOM fragment / off-screen clone before reattaching) rather than performing many separate style mutations one property at a time. Batched style changes let the browser compute layout once for the whole batch instead of potentially once per mutation.

### Memoization Discipline

- **DO:** Treat memoization (`React.memo`, `useMemo`, `useCallback`, Vue's `computed`, Angular pure pipes, `createSelector`/reselect-style memoized selectors) as an optimization applied in response to a measured problem, not a default coding style applied everywhere preemptively. Every memoization has a real cost — extra memory to hold the cached value/dependencies, extra comparison work on every render, and extra code for future maintainers to keep the dependency list correct — so it should be justified by an actual expensive computation or an actual re-render problem observed in profiling.
- **DON'T:** Wrap trivial, cheap computations (string concatenation, simple arithmetic, a short array `.filter()` on a handful of items) in `useMemo` reflexively. The overhead of the memoization machinery itself (creating the dependency array, running the comparison, storing the cached value) can exceed the cost of just recomputing a cheap value directly; reserve memoization for computations that are actually expensive relative to a render (large data transforms, complex derived aggregations, expensive formatting over large datasets).
- **DO:** Memoize a value or callback specifically when (a) it's passed to a memoized child component (`React.memo`) where a stable reference is required for the memoization to have any effect, (b) it's a dependency of another hook (another `useEffect`/`useMemo`) where an unstable reference would cause that hook to re-run unnecessarily, or (c) the computation itself is measurably expensive. Outside of these three cases, memoization is very often solving a problem that doesn't exist.
- **DON'T:** Nest memoization so deeply that the dependency arrays themselves become a maintenance burden bigger than the code they're protecting — e.g., memoizing a value, then memoizing a callback that depends on that memoized value, then memoizing an object built from both, several layers deep, for a component that renders forty times a session total. Step back and ask whether the entire subtree even needs this level of render optimization, or whether the component is simply cheap enough that none of this matters.
- **DO:** Verify a memoization actually achieves its intended effect after adding it, using the profiler (React DevTools "why did this render" / Profiler tab, Vue Devtools' component inspection, Angular's change-detection debugging tools) rather than assuming it worked because the code compiles. It's common to add `React.memo` to a component while still passing it an inline object or arrow-function prop from the parent, which silently defeats the memoization while looking correct in code review.
- **DON'T:** Memoize a selector or computed value with dependencies that are broader than necessary (e.g., depending on an entire large object when only one field of it is actually used in the computation). An overly broad dependency invalidates the cache more often than needed, recomputing the "memoized" value on changes that shouldn't have affected it, and disguises what the computation actually depends on to future readers.
- **DO:** Use `useMemo`/`computed` for values that are genuinely expensive to recompute AND whose inputs change less often than the component re-renders for other reasons. If a computation's inputs change on essentially every render anyway, memoizing it saves nothing — the cache is invalidated as often as it would have simply recomputed.

### Virtualizing Long Lists

- **DO:** Virtualize (render only the visible subset of rows, plus a small overscan buffer, recycling DOM nodes as the user scrolls) any list or table that can realistically grow past a few hundred rendered rows — a chat history, a spreadsheet-like grid, a large search-results list, an infinite feed. Rendering thousands of full DOM nodes at once (even off-screen ones) creates a large, slow-to-update DOM tree, high memory usage, and slow initial mount time, none of which improve the user's actual experience since they can only see a handful of rows at a time.
- **DON'T:** Reach for full-list virtualization for lists that are reliably small (a settings page with a dozen options, a dropdown with twenty entries). Virtualization adds real implementation complexity (careful height management, harder-to-implement find-in-page/browser-native scroll-to-anchor behavior, extra work to keep accessibility semantics correct with recycled DOM nodes) that isn't worth paying for a list that will never be large enough to cause a real performance problem.
- **DO:** Use an established virtualization library (`react-window`/`react-virtual`/`@tanstack/virtual`, `vue-virtual-scroller`, Angular CDK's `cdk-virtual-scroll-viewport`) rather than hand-rolling scroll-position-based show/hide logic from scratch. These libraries have already solved the hard edge cases — variable row heights, scroll-anchor preservation during data updates, keyboard navigation, and correct behavior on resize — that a first attempt at hand-rolled virtualization typically gets wrong.
```jsx
// GOOD — only visible rows (plus overscan) are actually mounted
import { useVirtualizer } from '@tanstack/react-virtual';

function BigList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 8,
  });
  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((row) => (
          <div key={row.key} style={{ position: 'absolute', top: row.start, height: row.size }}>
            {items[row.index].label}
          </div>
        ))}
      </div>
    </div>
  );
}
```
- **DON'T:** Pair virtualization with `key={index}` on the virtualized rows, or with any assumption that a given DOM node is permanently bound to a given data item. Virtualized lists recycle their limited set of mounted DOM nodes across different underlying data items as the user scrolls, specifically to avoid the cost of constant mount/unmount; combine this with an index-based key (which is exactly what index keys were warned against above) and rows can display stale content or lose input focus/state as they're recycled onto different data.
- **DO:** Pair virtualization with pagination or infinite-scroll data fetching (fetch a window of data around the current scroll position, not the entire dataset up front) for lists backed by very large or unbounded server-side datasets. Virtualizing the rendering of ten thousand already-in-memory rows solves the DOM-size problem but not the "we downloaded and parsed ten thousand rows of JSON to display forty of them" problem; combine windowed rendering with windowed/paginated fetching for datasets too large to reasonably hold entirely in memory.
- **DON'T:** Virtualize a list whose items have wildly unpredictable, dynamically-measured heights without using a virtualization approach that explicitly supports dynamic sizing (measuring rendered rows and adjusting scroll-position math accordingly). Assuming a fixed row height when heights actually vary produces visibly jumpy scrollbars and incorrect scroll-position jumps as items are measured and re-measured; use the library's dynamic-size / auto-measuring mode, or normalize row heights in the design if scroll stability matters more than variable content height.

### General Performance Discipline

- **DO:** Measure before optimizing, using real tooling (Lighthouse, WebPageTest, the browser's own Performance panel, React/Vue/Angular DevTools profilers, Core Web Vitals field data from real users via a RUM tool) rather than optimizing based on source-reading intuition about what "feels slow." Source code that looks inefficient sometimes isn't a bottleneck at all in practice, and code that looks fine can hide the actual bottleneck; profiling data, not intuition, should drive where optimization effort goes.
- **DON'T:** Chase a synthetic benchmark score or a micro-benchmark improvement that doesn't correspond to any real, user-perceivable difference (shaving milliseconds off a computation that already completes well within a single frame, or optimizing a code path that runs once per session). Time spent optimizing something a user will never perceive is time not spent on something they would; prioritize interaction responsiveness and perceived loading speed on the actual critical user paths first.
- **DO:** Pay particular attention to the Core Web Vitals as a baseline, framework-agnostic performance target: Largest Contentful Paint (main content visible quickly), Interaction to Next Paint (the UI responds quickly to input), and Cumulative Layout Shift (content doesn't visually jump around as it loads). These map closely to what users actually perceive as "this feels fast" versus "this feels janky," and are also used as real search-ranking signals, giving them a business-visible reason to matter beyond internal engineering pride.
- **DON'T:** Ship an image at its full original resolution/format when a browser-appropriate responsive size and a modern compressed format (`srcset`/`sizes`, WebP/AVIF, lazy-loading via `loading="lazy"` for below-the-fold images) would serve the same visual result at a fraction of the bytes. Unoptimized images are consistently one of the largest contributors to slow page loads and poor Largest Contentful Paint scores, and are also one of the cheapest problems to fix with existing, well-supported browser and build-tool features.
- **DO:** Debounce or throttle event handlers bound to high-frequency events (`scroll`, `resize`, `mousemove`, `input` on a search-as-you-type field triggering a network request) so expensive work runs at a bounded rate instead of on every single event firing. A search input that fires a network request on every keystroke, with no debounce, both wastes bandwidth/server load and risks the exact out-of-order-response race condition discussed under RxJS's `switchMap` guidance above.

### Images and Web Fonts

- **DO:** Serve images at a size appropriate to their actual rendered dimensions and device pixel ratio using `srcset`/`sizes` (or a framework's built-in image component — Next.js's `<Image>`, Nuxt Image, Angular's `NgOptimizedImage`), rather than shipping one large source image and letting CSS scale it down in the browser. A 4000px-wide source image displayed in a 300px-wide card wastes the vast majority of its downloaded bytes on pixel detail the user never sees rendered.
- **DON'T:** Omit explicit `width`/`height` attributes (or an equivalent aspect-ratio reservation) on images. Without a reserved space, the browser doesn't know how much room to leave for the image before it loads, causing surrounding content to visibly shift down once the image arrives — a direct, common cause of poor Cumulative Layout Shift scores and a jarring experience for anyone reading content as a page loads.
```html
<!-- BAD — no reserved space, causes layout shift when the image loads -->
<img src="hero.jpg" alt="Product hero shot" />

<!-- GOOD — space reserved via explicit dimensions, no shift on load -->
<img src="hero.jpg" alt="Product hero shot" width="1200" height="600" loading="lazy" />
```
- **DO:** Load web fonts with `font-display: swap` (or `optional`, depending on how tolerant the design is of a fallback-to-webfont visual jump) so text remains visible in a fallback font while the custom font downloads, rather than blocking text rendering entirely until the font arrives (`font-display: block`'s default-like behavior), which can leave users staring at invisible text on a slow connection.
- **DON'T:** Load an entire web font family's full character set and every weight/style variant when only a couple of specific weights are actually used in the design. Subset fonts to the character ranges actually needed (especially significant for non-Latin scripts) and load only the weights/styles the design actually calls for, rather than the full family by default.
- **DO:** Preconnect to (or preload) critical third-party origins and above-the-fold critical assets (a hero image, the primary web font) using `<link rel="preconnect">`/`<link rel="preload">` in the document head, to shave the connection-setup and asset-discovery latency off the critical rendering path for resources known in advance to be needed immediately.

### Prefetching and Service Workers

- **DO:** Prefetch the JavaScript and data for a route the user is very likely to navigate to next — most reliably signaled by a link entering the viewport (many frameworks' link components do this automatically) or a hover/focus on a link — so the navigation itself feels instantaneous when it happens, since the work was already done ahead of time during idle moments.
- **DON'T:** Prefetch indiscriminately, every link on every page, regardless of viewport visibility, network conditions, or actual likelihood of navigation. Blanket prefetching wastes bandwidth and can measurably compete with the current page's own more important requests for network/CPU resources, especially harmful on metered or slow connections; respect `navigator.connection.saveData`/effective connection type where available, and prefer a framework's built-in, viewport/interaction-aware prefetching heuristics over a naive "prefetch everything" approach.
- **DO:** Use a service worker (via Workbox or a framework's built-in PWA tooling) to cache static assets and, where appropriate, API responses, enabling faster repeat visits and offline/flaky-network resilience — genuinely valuable for an app that benefits from installability or offline use.
- **DON'T:** Adopt a service worker with an aggressive caching strategy (cache-first for everything, including HTML/API responses that change frequently) without a clear cache-invalidation and versioning plan. A poorly-configured service worker is one of the few frontend mistakes capable of trapping a user on a stale, broken version of the app indefinitely — since the service worker itself controls how updates are fetched, a bad caching decision can make the app unable to self-heal even after the bug is fixed and redeployed, unlike almost any other kind of frontend bug.

### Offloading Work: Web Workers and Idle Scheduling

- **DO:** Move genuinely expensive, synchronous CPU-bound computation (parsing/transforming a large dataset, complex client-side image/video processing, running a heavy client-side search index) off the main thread into a Web Worker, so it doesn't block user input handling, animation, and rendering while it runs. Main-thread-blocking work is one of the most direct causes of poor Interaction to Next Paint scores and visibly "frozen" UI during the computation.
- **DON'T:** Move trivial or already-fast computations into a Web Worker "for performance," when the overhead of serializing data across the worker boundary (`postMessage` structured-clone cost) and the added architectural complexity (async message-passing instead of a direct function call) exceeds any benefit for work that was already fast enough to not visibly block the main thread.
- **DO:** Use `requestIdleCallback` (where supported, with a `setTimeout` fallback for browsers/contexts that lack it) to schedule genuinely low-priority work — non-essential analytics/logging, speculative prefetching, non-visible background computation — during browser idle periods, rather than competing with rendering and user-input handling for the main thread's immediate attention.
- **DON'T:** Schedule anything user-visible or time-sensitive via `requestIdleCallback`, since idle callbacks can be deferred arbitrarily long (or never fire at all, e.g., under continuous user activity) — it has no guaranteed execution deadline suitable for anything the user is actively waiting to see.

### SSR, Hydration, and Streaming

- **DO:** Understand hydration cost as a real, often-underestimated performance factor for server-rendered apps: the browser must download, parse, and execute the full client-side JavaScript bundle and re-run component logic to attach event handlers to server-rendered HTML before the page becomes fully interactive, and a large hydration payload can leave a page visually complete (good Largest Contentful Paint) but unresponsive to input for a noticeable window (poor Interaction to Next Paint) — the "uncanny valley" of a page that looks ready but isn't.
- **DON'T:** Ship the same amount of client-side JavaScript for hydration regardless of how much of the page is actually interactive. Where the framework supports it, use partial/selective hydration or islands-architecture patterns (React Server Components' client-boundary isolation, Astro's islands, Qwik's resumability model) to hydrate only the genuinely interactive fragments of a mostly-static page, rather than hydrating and re-executing component logic for content that never needed any client-side behavior in the first place.
- **DO:** Use streaming SSR (progressively flushing HTML to the browser as it becomes ready, rather than waiting for the entire page's data to resolve before sending any response) for pages with slow, independent data-fetching needs across different sections, so the fast parts of the page (navigation, layout, above-the-fold content) reach the browser and start rendering immediately instead of being held up behind the page's single slowest data dependency.
- **DON'T:** Introduce a hydration mismatch by rendering different output on the server than the client will render on its first client-side pass — the most common causes are reading browser-only values (`window`, `localStorage`, the current time, `Math.random()`) directly during the initial render instead of gating them behind a mount-effect check, or letting server and client clocks/locales disagree on formatted output. A hydration mismatch typically either throws a visible error, forces the framework to discard and fully client-re-render the mismatched subtree (losing the performance benefit SSR was providing), or silently displays incorrect content depending on the framework's mismatch-handling behavior.
```jsx
// BAD — window/localStorage read directly during render; undefined
// on the server, so server and client output disagree
function ThemeToggle() {
  const stored = window.localStorage.getItem('theme'); // throws/mismatches on server
  return <button>{stored ?? 'light'}</button>;
}

// GOOD — browser-only read deferred to after mount, server renders a stable default
function ThemeToggle() {
  const [theme, setTheme] = useState('light');
  useEffect(() => {
    setTheme(window.localStorage.getItem('theme') ?? 'light');
  }, []);
  return <button>{theme}</button>;
}
```

### Memory Leak Prevention in Long-Lived SPAs

- **DO:** Treat a single-page application as a long-lived process that must actively clean up after itself on every navigation and every component unmount — event listeners, timers/intervals, WebSocket connections, observer instances (`ResizeObserver`, `IntersectionObserver`, `MutationObserver`), and third-party library instances (map/chart libraries with their own `.destroy()` method) — since none of this is automatically garbage-collected just because a component's DOM node was removed, if something else (a global listener, a closure captured by a still-active timer) still holds a reference to it.
- **DON'T:** Assume a memory leak in a client-rendered SPA will be caught quickly through ordinary manual testing. Because a single component-level leak is small, it typically only becomes visible as degraded performance or an eventual crash after a user has navigated through the app for an extended session — exactly the kind of usage pattern normal QA testing (a handful of page loads, not an extended session) often doesn't reproduce; use the browser's memory profiler (heap snapshots taken before and after repeated mount/unmount cycles of a component) to proactively check for leaks in components that create subscriptions, timers, or hold large data references.
- **DO:** Pay particular attention to closures captured by long-lived callbacks (an event listener, an interval callback, a subscription callback) that reference component state or props — if that closure is never cleaned up, it keeps the entire referenced component instance (and everything it in turn references) alive in memory long after the component should have been eligible for garbage collection.

## Accessibility

### WCAG Basics for Interactive Components

- **DO:** Treat WCAG 2.1/2.2 Level AA as the practical baseline for any production interactive component, not an optional nice-to-have addressed at the end of a project. Accessibility defects are functionally equivalent to correctness bugs for the population of users who rely on assistive technology or keyboard-only navigation — a button a screen-reader user can't discover is, for that user, a button that doesn't exist.
- **DON'T:** Treat accessibility as something to bolt on after the visual design and interaction are finalized. Retrofitting keyboard support, focus management, and ARIA semantics onto a component built without them typically requires larger structural changes than building them in from the start (correct semantic elements, a sane DOM order, and keyboard handlers designed alongside the mouse/touch interaction).
- **DO:** Reach for the native HTML element that already has the required semantics and behavior built in — `<button>` for anything clickable that performs an action, `<a href>` for anything that navigates, `<input>`/`<select>`/`<textarea>` for form controls, `<dialog>` for modal dialogs — before building a custom component from a `<div>`. Native elements come with correct default keyboard behavior, focus handling, and accessibility-tree roles for free; a hand-rolled `<div onClick>` reproduces none of that automatically and requires manually re-implementing everything the native element already provided.
```jsx
// BAD — a div "button" with none of a real button's built-in behavior
<div className="btn" onClick={submit}>Submit</div>

// GOOD — native semantics, keyboard support, and focusability for free
<button type="button" className="btn" onClick={submit}>Submit</button>
```
- **DON'T:** Add a `role="button"` and an `onClick` handler to a non-interactive element (`<div>`, `<span>`) as a substitute for using an actual `<button>`, unless there's a genuinely unavoidable constraint (e.g., nesting restrictions) that prevents using the native element. A `role` attribute alone only changes what's announced to assistive technology — it does not add keyboard focusability, `Enter`/`Space` key activation, or any of the other behaviors a real `<button>` gets natively; all of that has to be reimplemented by hand (`tabIndex="0"`, an `onKeyDown` handler checking for Enter and Space) and it's easy to miss a case.
- **DO:** Ensure text has sufficient color contrast against its background — WCAG AA requires at least 4.5:1 for normal text and 3:1 for large text (approximately 18pt+/24px+, or 14pt+/18.5px+ bold), and 3:1 for the visual boundaries of UI components and graphical objects that convey meaning (icon-only buttons, input borders, focus indicators). Verify contrast with an actual contrast-checking tool against the real rendered colors, not by eye, since perceived contrast is a poor substitute for the calculated ratio.
- **DON'T:** Convey meaning, state, or required information through color alone (a red border meaning "invalid" with no accompanying icon or text, a green dot meaning "online" with no text label reachable by assistive tech). Color-only signaling is invisible to colorblind users and to anyone using a screen reader; pair color with a text label, icon, or pattern that communicates the same information non-visually.
- **DO:** Ensure every interactive element has a large enough touch/click target — WCAG 2.2's target-size guidance recommends at least 24x24 CSS pixels (44x44 is the more conservative, widely-cited mobile-usability recommendation) — with adequate spacing between adjacent targets. Small, tightly-packed tap targets are a common source of mis-taps for users with motor impairments, and simply frustrating for everyone on a touchscreen.
- **DON'T:** Set `outline: none` (or `outline: 0`) on a focusable element's CSS without providing a clearly visible replacement focus style. Removing the focus outline with no substitute makes it impossible for a keyboard user to see which element currently has focus, which breaks keyboard navigation entirely for that element even though it may still technically be focusable.
```css
/* BAD — focus indicator removed with nothing to replace it */
button:focus { outline: none; }

/* GOOD — a clear, visible custom focus style replaces the default */
button:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
}
```
- **DO:** Use `:focus-visible` rather than `:focus` when customizing focus styles, so the visible focus ring appears for keyboard navigation (where it's essential) without necessarily appearing on every mouse click (where many designs prefer to omit it). This matches the behavior users already expect from native browser focus styling.
- **DON'T:** Assume WCAG compliance based only on running a single automated scanning tool (axe, Lighthouse's accessibility audit, WAVE) and treating a clean report as proof the component is accessible. Automated tools reliably catch a meaningful subset of issues (missing alt text, insufficient contrast, missing form labels, invalid ARIA) but structurally cannot detect a large share of real accessibility problems — illogical reading order, a keyboard trap, a misleading ARIA label, whether an interaction actually makes sense when only heard, not seen. Automated scanning is a floor, not a substitute for manual keyboard testing and, ideally, real screen-reader testing.

### Focus Management

- **DO:** Move focus explicitly to a sensible location when a significant UI change happens that isn't the direct result of a focused element's own action being logically continued — opening a modal (focus moves into the modal), navigating to a new route in a single-page app (focus moves to the new page's main heading or content region), or a form submission producing a validation error (focus moves to the first invalid field, or to an error summary). Without deliberate focus management, focus silently stays wherever it was (often on a now-hidden or now-irrelevant element, or reset to the `<body>`), leaving keyboard and screen-reader users with no indication that anything changed.
```jsx
// GOOD — focus moves into the dialog on open and returns to the trigger on close
function Dialog({ isOpen, onClose, triggerRef, children }) {
  const dialogRef = useRef(null);
  useEffect(() => {
    if (isOpen) {
      dialogRef.current?.focus();
    } else {
      triggerRef.current?.focus();
    }
  }, [isOpen]);
  return isOpen ? (
    <div role="dialog" aria-modal="true" ref={dialogRef} tabIndex={-1}>
      {children}
    </div>
  ) : null;
}
```
- **DON'T:** Open a modal dialog without trapping keyboard focus inside it. If `Tab` is allowed to cycle focus out to elements behind the modal (which are visually obscured or entirely hidden), a keyboard user can end up interacting with content they can't see, and a screen-reader user loses the modal's context entirely; trap `Tab`/`Shift+Tab` cycling within the modal's focusable elements for as long as it's open, and restore focus to the element that opened it once it closes.
- **DO:** Use a well-tested focus-trap implementation (a library like `focus-trap`, or the framework/design-system's built-in dialog primitive) rather than hand-rolling `Tab` key interception logic from scratch. Correctly handling every edge case — dynamically added/removed focusable children, `Shift+Tab` wrapping backward from the first element, elements that become focusable/unfocusable while the trap is active — is easy to get subtly wrong by hand, and a broken hand-rolled trap can itself become a keyboard trap that strands the user.
- **DON'T:** Create an unintentional keyboard trap — any point in the UI where a keyboard user can tab into a region but has no way to tab back out of it (a broken focus trap that doesn't release on `Escape`, an embedded third-party widget that intercepts and swallows `Tab` presses). WCAG treats keyboard traps as a Level A (baseline, non-negotiable) failure precisely because they can leave a keyboard-only user completely stuck, unable to reach any other part of the page.
- **DO:** Set focus to a logical, predictable target after route navigation in a client-rendered single-page app — typically the page's `<h1>` (given `tabIndex={-1}` so it's programmatically focusable despite not being a natively-focusable element) or a designated "skip to main content" landing point — since browsers don't do this automatically for JS-driven route changes the way they do for full page loads.
- **DON'T:** Manage focus purely by calling `.focus()` imperatively wherever convenient in event handlers without also announcing the resulting change to screen-reader users when the visible content update wouldn't otherwise be obvious from focus alone (e.g., an inline status message that appears without any element gaining focus). Pair silent DOM updates that carry important information with an `aria-live` region (see below) when moving focus to the changed content itself isn't appropriate.
- **DO:** Preserve a logical, visually-matching DOM order for focusable elements — the order elements receive focus via sequential `Tab` presses should match their visual left-to-right, top-to-bottom reading order. A component visually reordered with CSS (`order`, absolute positioning, grid placement) while its DOM/tab order stays unchanged creates a confusing mismatch between what a sighted keyboard user sees happening visually and what a screen-reader user hears announced.
- **DON'T:** Use positive `tabIndex` values (`tabIndex="1"`, `tabIndex="2"`, etc.) to manually control tab order. Positive tabindex values create a separate, higher-priority tab sequence that overrides the natural DOM order in ways that are extremely easy to get inconsistent as the page evolves, and that behave surprisingly once combined with any other positive-tabindex element elsewhere on the page; use `tabIndex="0"` (join the natural DOM order) or `tabIndex="-1"` (programmatically focusable but not in the Tab sequence), and fix ordering issues by changing the actual DOM order instead.

### ARIA Roles Used Correctly, Not as a Crutch

- **DO:** Follow the "first rule of ARIA": don't use ARIA if a native HTML element or attribute already provides the required semantics and behavior. ARIA attributes only affect what's exposed to the accessibility tree (what screen readers announce) — they add no built-in keyboard behavior, no built-in focus management, and no visual styling, so using ARIA to fake a native element's semantics still leaves all of that native behavior to be hand-built and easy to get wrong.
- **DON'T:** Add ARIA attributes speculatively "to be safe" or "because a linter flagged something" without understanding what each attribute actually communicates to assistive technology. An incorrect or contradictory ARIA role/attribute is frequently worse than no ARIA at all — it overrides the element's native semantics with (potentially wrong) explicit ones, actively misleading screen-reader users rather than simply leaving them with default (and often already-correct) behavior.
```jsx
// BAD — role="button" on a link changes its announced semantics
// and its expected keyboard behavior (Enter only, not Space+scroll),
// with no actual reason to override the native <a> semantics here.
<a href="/settings" role="button">Settings</a>

// GOOD — leave the native anchor semantics alone
<a href="/settings">Settings</a>
```
- **DO:** Use `aria-label` or `aria-labelledby` to give an accessible name to interactive elements that have no visible text label — most commonly icon-only buttons. Without an accessible name, a screen reader announces the element only as its generic role ("button") with no indication of what it does.
```jsx
// BAD — screen reader announces only "button", no indication of purpose
<button onClick={closeModal}><XIcon /></button>

// GOOD — accessible name announced alongside the role
<button onClick={closeModal} aria-label="Close dialog"><XIcon /></button>
```
- **DON'T:** Put an `aria-label` on an element that already has adequate visible text content, especially one that doesn't match that visible text. `aria-label` fully overrides the accessible name computation, including hiding the element's actual visible text from assistive technology entirely; if visible text already serves as a good accessible name, leave it as the (already correct) accessible name rather than redundantly or incorrectly overriding it.
- **DO:** Use `aria-live` regions (`aria-live="polite"` for non-urgent updates like a "saved" confirmation, `aria-live="assertive"` sparingly for urgent, must-interrupt updates like a session-expiring warning) to announce dynamic content changes that occur without a corresponding focus change — toast notifications, inline validation messages that appear after a field loses focus, a live search-results count. Without a live region, a screen-reader user has no way to know that content changed unless they happen to be reading that exact part of the page at that exact moment.
```jsx
// GOOD — screen readers announce the count update without needing focus to move
<div aria-live="polite" className="sr-only-visually-but-announced">
  {resultCount} results found
</div>
```
- **DON'T:** Overuse `aria-live="assertive"` for routine, non-urgent updates. Assertive live regions interrupt whatever the screen reader is currently announcing, which is jarring and disorienting when used for anything other than genuinely time-critical information; default to `polite` (which waits for the current speech to finish) and reserve `assertive` for alerts that truly need to interrupt.
- **DO:** Keep custom ARIA-role-based widgets (a `role="combobox"`, `role="tablist"`/`role="tab"`/`role="tabpanel"`, `role="menu"`/`role="menuitem"`) fully conformant with the specific keyboard interaction pattern and required/expected ARIA attribute set that role implies, per the WAI-ARIA Authoring Practices Guide (APG). Applying a role like `tablist` without implementing its expected arrow-key navigation and `aria-selected` state management gives screen-reader users a set of expectations (based on the announced role) that the actual component then fails to meet, which is often more confusing than not using that role at all.
- **DON'T:** Reach for a generic, complex ARIA widget pattern (a full custom combobox, a custom listbox) when a simpler native element (a native `<select>`, or a well-tested existing design-system component that already implements the pattern correctly) would satisfy the actual requirement. Hand-building a WAI-ARIA APG-conformant combobox from scratch is one of the most failure-prone accessibility tasks in frontend development — subtle mistakes in state synchronization between the input, the listbox, and the announced `aria-activedescendant` are common and easy to miss without dedicated screen-reader testing.
- **DO:** Set `aria-hidden="true"` on purely decorative content (a decorative icon next to text that already conveys the icon's meaning, a background illustration) so screen readers skip it, while making sure `aria-hidden` is never applied to an ancestor of an element that itself needs to remain focusable/announced — a focusable descendant of an `aria-hidden` element becomes unreachable to assistive tech while remaining visually and keyboard-focusably present, an inconsistent and confusing state.
- **DON'T:** Use `aria-hidden="true"` on the entire rest of the page/app without correctly restoring it when a modal closes, when implementing the "hide everything outside the modal" pattern for screen readers. A modal implementation that sets `aria-hidden` on background content on open but has a bug in its cleanup path (e.g., an early return, an unmount that skips the cleanup effect) can leave the rest of the app permanently hidden from assistive technology after the modal is dismissed; use a maintained dialog/modal library or the native `<dialog>` element (whose `showModal()` handles this automatically) rather than hand-rolling this bookkeeping.

### Keyboard Navigation

- **DO:** Verify every interactive element and every complete user flow is fully operable using only the keyboard — no mouse, no touch — as a baseline test before considering a feature done. This single check surfaces a large share of accessibility defects immediately: elements that are visually clickable but not focusable, custom widgets missing arrow-key/Enter/Space handling, and modals or menus that can be opened but never closed via keyboard.
- **DON'T:** Ship a custom dropdown, menu, tab set, carousel, or any other interactive widget that only responds to mouse/pointer events (`onClick`, `onMouseEnter`) with no keyboard event handling at all. A component that opens on hover or click alone, with no `onKeyDown` handling for the keys a user would expect (Enter/Space to activate, Escape to close, arrow keys to move within a menu/tablist/listbox), is entirely unusable for anyone navigating by keyboard.
- **DO:** Implement the conventional key bindings users and assistive technology expect for a given widget pattern: `Escape` closes a modal/dropdown/popover and typically returns focus to its trigger; `Enter`/`Space` activate a button-like control; arrow keys move selection within a composite widget (menu, tablist, listbox, radio group) with roving `tabIndex` (only the current item is in the Tab sequence; arrow keys move an internal "current item" pointer); `Home`/`End` jump to the first/last item in a list-like widget. Deviating from these conventions without a strong reason surprises users who've built muscle memory around how these patterns are supposed to behave everywhere else on the web.
```jsx
// GOOD — Escape closes the menu and returns focus to the trigger,
// matching the behavior users already expect from every native menu.
function DropdownMenu({ isOpen, close, triggerRef }) {
  useEffect(() => {
    function onKeyDown(e) {
      if (e.key === 'Escape' && isOpen) {
        close();
        triggerRef.current?.focus();
      }
    }
    document.addEventListener('keydown', onKeyDown);
    return () => document.removeEventListener('keydown', onKeyDown);
  }, [isOpen]);
  // ...
}
```
- **DON'T:** Implement "roving tabindex" incorrectly by leaving every item in a composite widget (every tab, every menu item, every option) individually focusable via sequential `Tab`. This forces a keyboard user to press `Tab` once per item to get through the whole widget instead of tabbing to it once and using arrow keys to move within it; only the currently-active item should have `tabIndex="0"`, with every other item at `tabIndex="-1"`, updated as arrow-key navigation moves the active item.
- **DO:** Provide a visible "skip to main content" link as the very first focusable element on the page (visually hidden until it receives focus) so keyboard users can bypass a long repeated navigation menu and jump straight to the page's main content on every page load. Without it, a keyboard user has to tab through the entire navigation on every single page before reaching the actual content, on every single page visit.
- **DON'T:** Rely on `onClick` handlers attached to non-interactive elements without also verifying (and testing) that the corresponding `onKeyDown` handling was actually added, and works — it's a common and easy-to-miss gap for a component to gain a keyboard handler during initial development but silently lose keyboard support during a later refactor (e.g., a click handler moved to a wrapping div during a redesign) that never gets caught because manual QA defaults to mouse/touch testing.
- **DO:** Test drag-and-drop interactions (reordering a list, moving a card between columns) with a keyboard-accessible alternative — many drag-and-drop implementations are pointer-only by default, and a fully mouse/touch-only reordering mechanism has no keyboard equivalent at all unless one is deliberately built (e.g., a "move up"/"move down" button pair, or arrow-key-based reordering when a draggable item has focus). Drag-and-drop is one of the interaction patterns most commonly shipped with zero keyboard support, since it doesn't obviously fail during typical development (mouse-driven) testing.

### Screen-Reader Testing

- **DO:** Perform real manual testing with at least one actual screen reader (VoiceOver on macOS/iOS, NVDA or JAWS on Windows, TalkBack on Android) for any genuinely custom or complex interactive component, rather than relying solely on automated tooling or on reading the rendered accessibility tree in browser DevTools. Automated accessibility checkers verify structural rules (valid ARIA, presence of labels, contrast ratios) but cannot judge whether the actual experience of listening to the component makes sense, is efficiently navigable, or announces state changes at the right moments.
- **DON'T:** Assume a component "must be accessible" simply because it passes an automated Lighthouse/axe scan with zero errors. Automated tools structurally cannot catch a large class of real defects — a misleading accessible name, a live region that fires so often it becomes noise, a logical-but-wrong reading order, redundant or missing state announcements — because these require judging the actual experience against user intent, which only a human (ideally with screen-reader fluency, or an actual assistive-technology user) testing the real flow can evaluate.
- **DO:** Test the full, realistic user flow with the screen reader — not just an isolated component in a sandbox — since context matters: how a component is announced when reached by continuous-reading, by heading navigation, by form-field navigation, and by tab order can each surface different issues, and a component that seems fine when tested in isolation can behave confusingly within the actual surrounding page.
- **DON'T:** Rely exclusively on developers testing their own work with a screen reader as the final accessibility signoff, without ever involving people who use assistive technology daily as their primary means of access. A sighted developer using a screen reader occasionally, without the fluency of a daily user, tends to miss efficiency and clarity problems that only become apparent to someone who relies on the tool as their sole way of using the web; where possible, include accessibility specialists or actual assistive-technology users in review for significant new interactive features.
- **DO:** Verify that dynamically loaded content (infinite scroll appending new items, a live-updating dashboard, a chat message stream) is announced sensibly to screen-reader users, and that new content doesn't silently arrive with no indication at all, or arrive so frequently that live-region announcements become an unusable wall of noise. Tune the frequency and granularity of live-region announcements deliberately (e.g., announcing "5 new messages" as a batched summary rather than reading each of five rapid messages individually).
- **DON'T:** Ship a form where required fields, validation errors, and field constraints (format requirements, character limits) are conveyed only through visual styling (a red asterisk, a red border) with no programmatic association a screen reader can pick up. Associate required-state via the native `required` attribute or `aria-required`, associate error messages with their field via `aria-describedby`, and use `aria-invalid="true"` on fields currently failing validation, so a screen-reader user gets the same information a sighted user gets from the visual styling alone.
```html
<!-- GOOD — error text is programmatically tied to the field -->
<label for="email">Email</label>
<input id="email" type="email" required aria-invalid="true" aria-describedby="email-error" />
<p id="email-error" role="alert">Enter a valid email address.</p>
```
- **DO:** Check heading structure (`<h1>` through `<h6>`) forms a logical, non-skipping outline of the page's content, since screen-reader users very commonly navigate a page by jumping between headings rather than reading linearly — a broken or skipped heading hierarchy (jumping from `<h1>` straight to `<h4>` because it happened to match a desired font size) actively harms this common navigation strategy. Choose heading levels based on document structure/semantics, and control visual size with CSS, not by picking whichever heading tag happens to render at the desired font size.

### Semantic Landmarks and Document Structure

- **DO:** Structure a page using semantic landmark elements (`<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`) rather than an undifferentiated tree of `<div>`s with visual-only styling distinguishing them. Landmarks let screen-reader users jump directly between major page regions (skip straight to `<main>`, or straight to `<nav>`) instead of having to read or tab through the entire page linearly to find the region they want.
- **DON'T:** Use more than one `<main>` landmark on a single page, or nest a `<main>` inside another landmark. There should be exactly one `<main>` per page, containing the page's actual primary content, so that "jump to main content" navigation has one unambiguous, correct target.
- **DO:** Set the `lang` attribute on the `<html>` element to the page's primary language (and on any specific element whose content is in a different language than the surrounding page, via that element's own `lang` attribute). Screen readers use the `lang` attribute to select the correct pronunciation/voice profile — an incorrect or missing `lang` attribute causes the screen reader to read content aloud with the wrong language's pronunciation rules, often producing unintelligible output.
- **DON'T:** Rely on visual grouping alone (whitespace, a border, a background color change) to communicate that a set of elements forms a logical group, when that grouping carries meaning a screen-reader user also needs. Use the appropriate grouping element or attribute (`<fieldset>`/`<legend>` for a related set of form controls, `role="group"` with `aria-labelledby` for a general grouped region) so the relationship is exposed programmatically, not just visually.

### Tables, Images, and Media

- **DO:** Use `<th>` (with an appropriate `scope="col"`/`scope="row"` attribute, or `id`/`headers` associations for complex tables with multi-level headers) for actual table header cells, and reserve `<table>` markup for genuinely tabular data — never for layout purposes. A screen reader announces a `<th>`'s association with its data cells automatically when `<th>`/`scope` are used correctly, letting a user navigating cell-by-cell hear which column/row they're in; a table built entirely from `<td>`s (or from non-table `<div>`s styled to look like a table) provides none of that structural information.
- **DON'T:** Leave a data table with a caption-worthy purpose lacking a `<caption>` element, forcing a screen-reader user to infer the table's purpose purely from its contents. A concise `<caption>` announced before the table's content gives immediate context ("Quarterly revenue by region") that would otherwise have to be inferred cell by cell.
- **DO:** Write meaningful, specific `alt` text for every informative image, describing the image's actual content or function in context — not the literal filename, not "image," not an empty restatement of adjacent visible text. For a purely decorative image that conveys no information (a background flourish, a duplicate of information already available as text nearby), use an empty `alt=""` (not an omitted `alt` attribute) so screen readers correctly skip it as decorative rather than reading the filename or a fallback description.
```html
<!-- BAD — meaningless alt text, or worse, none at all -->
<img src="chart_final_v3.png">

<!-- GOOD — alt text describes what the image conveys in context -->
<img src="chart_final_v3.png" alt="Bar chart showing revenue grew 34% year over year, from $2.1M to $2.8M">

<!-- GOOD — purely decorative image explicitly marked to be skipped -->
<img src="divider-flourish.svg" alt="">
```
- **DON'T:** Ship video or audio content with no captions, transcript, or audio description track. Captions serve deaf and hard-of-hearing users (and, incidentally, anyone watching with sound off); a transcript additionally serves users who prefer to read, and search/discoverability; audio description (a narrated description of important visual content) serves blind and low-vision users for video where visual information isn't otherwise conveyed through dialogue/audio. Treat captions as a required deliverable for any video content shipped in a product, not an optional add-on.
- **DO:** Give complex, information-dense images (a diagram, an infographic, a data visualization) a long-form text alternative in addition to a short `alt` attribute, since a single `alt` string is rarely sufficient to convey everything a complex image communicates. Provide the fuller description as adjacent visible text, a linked long-description page, or via `aria-describedby` pointing at a fuller text block.

### Accessible Drag-and-Drop and Data Visualization

- **DO:** Build a genuine keyboard-operable alternative for any drag-and-drop interaction, since drag-and-drop is inherently a pointer-driven gesture with no built-in keyboard equivalent — a reorderable list needs a keyboard path (arrow keys while an item has focus, or explicit "move up"/"move down" controls) that accomplishes the same reordering a mouse-drag would, and the interaction should announce the resulting position change via a live region so a screen-reader user gets confirmation of what just happened.
- **DON'T:** Assume a drag-and-drop library's default behavior is accessible out of the box without checking — many popular drag-and-drop libraries are pointer/touch-only unless the specific library's accessibility-focused APIs (some, like `@dnd-kit`, ship built-in keyboard sensor support and live-region announcements) are deliberately enabled and configured.
- **DO:** Provide a non-visual way to access the underlying data behind any chart or data visualization — a paired data table (visually hidden if needed, but present in the DOM and reachable), a text summary of the key takeaway, or `aria-label`/`aria-describedby` content summarizing what the chart shows — since the visual chart itself (typically rendered as an SVG or canvas with little to no inherent semantic structure) is usually opaque to a screen reader by default.
- **DON'T:** Rely on hover-triggered tooltips as the only way to access a data point's exact value in a chart, with no keyboard-accessible or screen-reader-accessible equivalent. A chart interaction that only reveals precise values on mouse hover excludes keyboard users and screen-reader users from that information entirely; ensure focusable data points expose the same detail on focus that they show on hover, and that the detail is announced, not just visually displayed.

### Motion, Forms, and Error Prevention

- **DO:** Respect the `prefers-reduced-motion` media query by disabling or substantially reducing non-essential animations (parallax effects, large sliding/zooming transitions, auto-playing decorative motion) for users who've indicated a preference for reduced motion at the OS level. Excessive motion can trigger real physical symptoms (nausea, dizziness, vestibular disorder reactions) for some users — this isn't a cosmetic preference to treat as optional polish, but a documented accessibility need.
```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```
- **DON'T:** Autoplay video, audio, or a looping animation with no easily discoverable, keyboard-accessible way to pause, stop, or hide it. Autoplaying, moving, or blinking content that can't be paused is both a distraction/accessibility problem (for users with attention-related conditions, vestibular disorders, or who use screen readers, where competing audio interferes with the screen reader's own speech output) and a widely-cited usability annoyance for everyone.
- **DO:** Use appropriate `autocomplete` attribute values (`autocomplete="email"`, `autocomplete="street-address"`, `autocomplete="cc-number"`, etc.) on form inputs collecting standard categories of personal information. Correct `autocomplete` values let both browsers and assistive technology help users fill forms faster and more accurately — genuinely important for users with motor or cognitive disabilities for whom retyping the same information repeatedly is a significant burden, not just a minor convenience.
- **DON'T:** Let a user submit a form that performs an irreversible, significant, or destructive action (a payment, a permanent deletion, an account cancellation) with a single, easily-mis-clicked action and no confirmation step, no review-before-submit summary, or no ability to correct an error before the action is final. WCAG's error-prevention guidance for this category of action specifically calls for a way to reverse, check-and-correct, or confirm the action before it's committed — this benefits every user, but is especially important for users for whom a small motor slip (an accidental double-click, a mis-tap) is more likely.
- **DO:** Warn users clearly, with enough time to respond, before a session times out and discards unsaved work, and provide a way to extend the session rather than silently expiring it. An unannounced session timeout that discards a partially-completed form is frustrating for any user, but disproportionately affects users who need more time to complete an interaction due to a motor, cognitive, or reading-related disability — WCAG explicitly requires that time limits be adjustable, extendable, or removable except in a narrow set of justified real-time-exception cases (e.g., a live auction).

## Common AI-Assistant Mistakes in Frontend Code

- **DON'T:** Invent hooks, lifecycle methods, component APIs, or framework functions that sound plausible but don't actually exist in the target framework/version (a hallucinated `useAsyncEffect`, a `useDebounce` assumed to be a React built-in when it isn't, a nonexistent Vue lifecycle hook, an Angular decorator that was renamed or removed in a later version). This is a distinctly LLM-shaped failure mode — code that looks fluent and idiomatic but references an API that simply isn't there, which fails at build time (best case) or is silently a no-op / type error depending on the framework's strictness. Verify any less-common API against the current, version-matched official documentation before using it, and prefer well-known, actually-existing primitives over a confident-sounding invention.
```jsx
// BAD — useAsyncEffect is not a real React hook; this doesn't exist
useAsyncEffect(async () => {
  const data = await fetchData();
  setData(data);
}, [id]);

// GOOD — the actual React primitive for this: useEffect with an
// inner async function, exactly as covered above
useEffect(() => {
  let cancelled = false;
  (async () => {
    const data = await fetchData();
    if (!cancelled) setData(data);
  })();
  return () => { cancelled = true; };
}, [id]);
```
- **DON'T:** Generate a `useEffect` (or `watch`/`$:` reactive block) with a dependency array that's incomplete, guessed, or copied from a superficially similar example without actually tracing which values the effect body reads. This is one of the highest-frequency AI-generated-code defects in React specifically, because the dependency array is easy to produce syntactically plausible output for without correctly reasoning about closures — always cross-check every dependency array against the actual variables referenced inside the effect body, not against what "looks about right."
- **DON'T:** Use the array index as the `key` prop for a dynamically-orderable, filterable, or mutable list, out of habit from countless training examples that use `.map((item, i) => <Row key={i} />)` for illustrative simplicity. This is one of the most consistently over-generated antipatterns in AI-produced React/Vue/Svelte code specifically because index-as-key appears constantly in simplified tutorial snippets where the list never actually changes shape; default to a stable identity field (`item.id`) as the key for any list backed by real, mutable data, and only use an index when the list is provably static.
- **DON'T:** Mix state-management paradigms inconsistently within a single generated feature — e.g., producing one component that reads global state from Redux, a sibling component for the same feature that reads what should be the same data from local `useState` fetched independently, and a third that introduces a brand-new Context for overlapping data. This tends to happen when a model pattern-matches each component in isolation against a different training example rather than reasoning about the feature's state architecture holistically; before generating multiple components for one feature, decide the state-ownership plan once, and generate every component in that feature against that single, consistent plan.
- **DON'T:** Generate a data-fetching component that only handles the "happy path" success render, with no explicit loading state, no error state, and no empty-result state. A component that assumes `data` is always a populated array/object the instant it renders will crash or render nonsense during the actual, common real-world states of "still loading," "the request failed," and "the request succeeded but returned zero results" — all three of which occur routinely in production and need their own deliberate UI, not an implicit fallthrough.
```jsx
// BAD — assumes data is always already there and non-empty
function UserList({ users }) {
  return (
    <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>
  );
}

// GOOD — loading, error, and empty states are handled explicitly
function UserList({ status, users, error }) {
  if (status === 'loading') return <UserListSkeleton />;
  if (status === 'error') return <ErrorMessage message={error} />;
  if (users.length === 0) return <EmptyState text="No users found." />;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```
- **DON'T:** Ignore an existing design system's components and hand-roll a fresh, one-off `<button>`/modal/dropdown/input from raw HTML elements and inline styles when a `Button`/`Modal`/`Dropdown`/`Input` component already exists in the project's component library. Reinventing a primitive that already exists produces visual and behavioral drift from the rest of the app (subtly different padding, a missing focus style, a missing loading-state affordance the design-system version already has), and duplicates accessibility work the design-system component may have already solved correctly; search the codebase's existing component library first, and only build a new primitive when the design system genuinely doesn't offer one.
- **DON'T:** Assume a package, hook, or utility is available in the project without checking its `package.json`/imports first, and then write code that imports it as though it were already installed. Generating `import { useDebounce } from 'react-use'` (or any other convenience library) in a codebase that has never depended on that package produces code that fails to build the moment it's actually run; check existing dependencies (or explicitly flag the new dependency and its cost, per the bundle-size guidance above) before assuming an import will resolve.
- **DON'T:** Generate a component that fetches data or manages significant client state but omits any cancellation/race-condition handling, silently reproducing the exact stale-response bug the `useEffect`/RxJS sections above warn against. This is a common omission specifically because the "happy path" fetch-then-setState code looks complete and correct on a first read, and the race-condition failure mode only manifests under specific timing (fast prop changes, slow/variable network conditions) that isn't obvious from reading the code once; default to including cancellation (an `AbortController`, a cancelled-flag, or the equivalent flattening operator like `switchMap`) any time a fetch is triggered by something that can change again before the fetch resolves.
- **DON'T:** Produce a form or interactive component with no accessibility semantics at all (missing `<label>`/label association, a clickable `<div>` instead of a `<button>`, no keyboard handling, no focus management on open/close) when explicitly asked for a production-quality component, and treat accessibility as something to add only if the user separately asks for it. Given how much publicly available frontend example code (blog posts, quick demos, Stack Overflow snippets used implicitly as training signal) skips accessibility for brevity, generated code has a systematic bias toward the same omission; bake in semantic HTML, labels, and keyboard support by default rather than only on request, the same way error handling and null checks are expected by default in backend code.
- **DON'T:** Write a `key` prop, a `useMemo`/`useCallback` dependency array, or an RxJS operator chain that merely looks locally plausible around the line it's attached to, without tracing the actual data flow through the component to verify correctness. Superficial pattern-matching (this looks like a list-rendering block, so it probably needs a `key`; this looks like a hook, so it probably needs a dependency array with the obviously-referenced variables) produces code that passes a cursory glance but fails on the specific edge cases the pattern exists to protect against (reordering, closures over stale values, race conditions); when generating this class of code, explicitly enumerate what the mechanism (key, dependency array, operator choice) needs to guarantee, and verify the generated code actually guarantees it.
- **DON'T:** Silently invent a prop, a slot name, or an event name on an existing shared component (assuming `<Modal onDismiss={...}>` exists when the actual component's prop is named `onClose`) instead of checking that component's actual defined interface first. This produces code that fails type-checking (if typed) or silently does nothing (if the framework doesn't validate unknown props strictly) and is functionally indistinguishable, from a training-data-pattern-matching perspective, from hallucinating a framework API — the fix is the same: check the actual, current definition of any existing component before assuming its interface.
- **DON'T:** Generate deeply nested conditional rendering or a large `switch` on a "type" field directly inline in JSX/template markup as a substitute for actual component composition, especially when asked for something "quick." Sprawling inline conditionals are exactly the kind of code an LLM can produce fluently and that looks locally reasonable line by line, while accumulating into an unmaintainable god-component; when a component's rendering logic branches on more than two or three conditions, extract named sub-components or a lookup/dispatch table instead of one more nested ternary or `if`.
- **DON'T:** Default to `any`/untyped props, loosely-typed event handlers, or a generic `object`/`unknown` prop type in a TypeScript codebase just to make a generated component compile quickly. Untyped props defeat the entire purpose of using TypeScript in the first place — catching prop-shape mismatches at compile time — and are a common shortcut in generated code specifically because a fully-inferred, precisely-typed prop interface takes more deliberate reasoning than a fast, permissive escape hatch; type props as precisely as the actual data allows, and treat `any` as something to justify explicitly, not a default.
- **DON'T:** Regenerate an entire large component from scratch in response to a small, localized change request (e.g., "add a loading spinner to this button") when a small, targeted edit would do. Wholesale regeneration risks silently dropping existing behavior, accessibility attributes, edge-case handling, or styling that isn't obviously connected to the requested change but was there for a reason; make the smallest edit that satisfies the request, and preserve everything else in the component exactly as it was.
- **DON'T:** Invent a Tailwind (or other utility-CSS framework) class name that sounds plausible but doesn't exist in that version's actual utility set, or mix class-name conventions from a different major version of the framework (v2-style class names in a v4 project, or vice versa). Utility-CSS frameworks version their exact class vocabulary, and a generated class that doesn't exist in the project's installed version silently does nothing (no build error, just an unstyled element), which is a failure mode that's easy to miss in a quick visual check and confusing to debug later; check the actual utility class names against the project's installed framework version rather than recalling a generically "Tailwind-shaped" class name from memory.
- **DON'T:** Generate code written against a different major version of the framework than the one actually installed in the project — React class-component lifecycle patterns in a hooks-only modern codebase, Vue 2 Options API idioms (`this.$set`, filters) in a Vue 3 project, Angular `NgModule`-based patterns in a project that has fully migrated to standalone components, or outdated RxJS operator import paths (`rxjs/operators` patterns that predate a later RxJS major version's flattened imports). Check the project's actual `package.json` version for the framework before generating code, and match idioms to that specific version rather than defaulting to whichever version's patterns are most represented in general training data.
- **DON'T:** Reimplement a utility function, a formatting helper, or a component that already exists somewhere in the project, simply because the existing one wasn't discovered before writing new code. Duplicated near-identical helpers (two slightly different date-formatting functions, two slightly different debounce implementations) are a direct source of inconsistent behavior across the app and unnecessary bundle bloat; search the codebase for an existing implementation of a needed utility before writing a new one, and prefer extending or reusing what's already there.
- **DON'T:** Add a new dependency to solve a problem the project's existing dependencies (or a few lines of native code) already solve, without flagging that a new dependency is being introduced and why. Silently adding a new package to `package.json` changes the project's install size, its security-audit surface, and its long-term maintenance burden — all real, ongoing costs that deserve an explicit decision, not an implicit one made mid-generation because it was the fastest way to satisfy the immediate request.
- **DON'T:** Generate code that stylistically clashes with the surrounding codebase's established conventions (a different quote style, a different import-ordering convention, `function` declarations dropped into a codebase that consistently uses arrow-function components, default exports introduced into a codebase that consistently uses named exports, or vice versa). Match the file's and the project's existing conventions — check a couple of neighboring files before introducing a new component — rather than defaulting to whatever style happens to be most familiar or most common in training data generally; a codebase with inconsistent style across files is harder for every future contributor (human or AI) to pattern-match against correctly.
- **DON'T:** Present generated frontend code as finished and correct without having actually verified it against the real, current API surface it depends on — an existing component's actual prop names, an actual installed library version's actual exported function signatures, or the project's actual TypeScript types. A confident, fluent-sounding explanation of what the code does is not evidence that it was checked against the real, current interfaces it calls into; read the relevant existing files (component definitions, type declarations, the installed package's actual exports) before asserting the generated code is correct, rather than asserting confidence based on how plausible the code looks.
- **DON'T:** Assume a browser API is universally available without a feature check or fallback, when the API in question has known compatibility gaps (a newer CSS property, a newer JS API like `structuredClone` or the View Transitions API, a permissions-gated API like clipboard access) relevant to the project's actual supported-browser matrix. Check the project's declared browser support target (a `browserslist` config, a stated minimum-browser-version requirement) before assuming an API's availability, and provide a graceful fallback or a feature-detection guard (`if ('structuredClone' in window)`) for anything not universally supported within that target matrix.
- **DON'T:** Over-engineer a response to a simple, narrowly-scoped request by introducing unrequested abstraction layers — a generic factory for a component that only ever needs one concrete instance, a configurable plugin system for a feature with exactly one use case, a new custom hook or utility module extracted for logic used in exactly one place. Matching the complexity of the solution to the actual complexity of the request is itself a form of correctness; unrequested abstraction adds surface area to review, maintain, and reason about, in exchange for flexibility nobody asked for and may never use.
- **DON'T:** Ignore server-rendering/hydration implications when generating code for a framework that does SSR (Next.js, Nuxt, SvelteKit, Angular Universal) by defaulting to patterns that only work in a pure client-side-rendered context — reading `window`/`document`/`localStorage` directly at the top of a component body, assuming `useEffect`-gated browser-only code is unnecessary, or forgetting that a value randomly generated per-render (an id, a random key) must be stable between the server-rendered and client-hydrated output. Default to writing SSR-safe code (guarding browser-only APIs behind a mount check, using the framework's provided stable-id primitives like React's `useId`) whenever the target framework is known to render on the server, rather than only writing for the simpler client-only case.

## Quick Checklist
- Call hooks only at the top level, never conditionally, in loops, or after an early return; name a function `useX` only if it actually calls other hooks.
- Don't store derived values in state; compute them inline or with `computed`/`useMemo`/`$derived`. Never mutate state/props in place — always create new references.
- Use the updater-function form of a setter when the next value depends on the previous one; reach for `useReducer`/a state machine once transitions get non-trivial.
- Split large contexts into small, purpose-specific ones; memoize provider values, and don't reach for Context/a global store before local state is proven insufficient.
- Read/write refs only in effects and event handlers, never during render, to drive rendering logic.
- Enable and respect exhaustive-deps/hook lint rules; don't silence a warning without fixing the root cause.
- Profile before adding `React.memo`/`useMemo`/`useCallback`; verify the memoization isn't defeated by an inline object/array/function prop from the parent.
- Use `children`/slots/composition to keep fast- and slow-changing parts of the tree from re-rendering each other, and to avoid prop-drilling.
- Use a stable, unique data identity (`item.id`) as list `key`; never use an array index or a freshly generated random value for a mutable/reorderable list.
- Include every reactive value an effect actually reads in its dependency array; return a cleanup function from any effect that subscribes to something external.
- Guard async effects/fetches against race conditions (`AbortController`, a cancelled flag, or `switchMap`); never pass an `async` function directly as an effect callback.
- Ask whether an effect is needed at all before writing one — derived state and event-driven logic often don't need one.
- Wrap data-fetching/lazy subtrees in Suspense-and-error-boundary pairs sized to independent failure domains, not one global boundary for the whole app.
- Default Server-Component-capable pages to server rendering; push `'use client'` to interactive leaves only, and never pass non-serializable values or secrets across that boundary.
- Pick one Vue API style (Composition or Options) per project and per component; know `ref()` needs `.value` while `reactive()` doesn't, and never destructure a `reactive()` object without `toRefs`.
- Never mutate a Vue prop directly in a child — emit an event instead; prefer `computed()`/`watchEffect` over a manual watch-and-set-a-second-ref combo.
- Scope component styles (`scoped`, CSS Modules) and default new Angular code to standalone components; reserve `NgModule` for existing module-based code.
- Unsubscribe every manual RxJS subscription; never nest `subscribe()` calls — flatten with `switchMap`/`mergeMap`/`concatMap`/`exhaustMap`; catch pipeline errors with `catchError`.
- Use `OnPush` with immutable input updates on Angular components; never mutate an `@Input()`-bound object in place; avoid expensive calls directly in templates.
- In Svelte 4, reassign (don't just mutate) top-level variables to trigger reactivity; prefer Svelte 5 runes for new code, and always clean up store subscriptions.
- Prefer framework data loaders (SvelteKit `load`, RSC async components, Angular resolvers) over client-only `onMount` fetches for a route's initial data.
- Classify state as local/shared-client/server/URL before picking a tool; don't duplicate server-fetched data into a separately-synced store; put shareable state in the URL.
- Choose a state library (Redux Toolkit, Zustand, Jotai, a state machine) to match the actual shape of the problem, not by default habit; model finite flows as explicit state machines.
- Implement optimistic updates only where failures are rare/recoverable, and always build the rollback path alongside the optimistic-apply path.
- Separate container (data/state) components from presentational (props-only) ones; extract a component on real duplication, not preemptively from a sample size of two.
- Organize by feature/domain, colocate tests/styles/stories with their component, and watch for god-component warning signs (huge files, unrelated state, constant churn).
- Reference design tokens instead of hardcoded colors/spacing; verify contrast independently per theme; constrain utility CSS to the design system's scale.
- Route all user-facing text through an i18n layer with real pluralization/interpolation; use logical CSS properties and locale-aware `Intl` formatting.
- Check a new dependency's bundle-size cost before adding it; import specific functions, not whole libraries; code-split by route and lazy-load heavy conditional UI with a loading fallback.
- Batch DOM reads separately from writes to avoid layout thrash; animate `transform`/`opacity`, not layout properties.
- Virtualize lists that can grow past a few hundred rows using a maintained library; never combine virtualization with index-based keys.
- Reserve image dimensions and use `srcset`/lazy-loading; load fonts with `font-display: swap`; prefetch likely-next routes, not every link indiscriminately.
- Move genuinely expensive synchronous work off the main thread; don't leak listeners, timers, or observers — clean them up on unmount/navigation.
- Guard browser-only APIs (`window`, `localStorage`, random/time-based values) behind a mount check in SSR frameworks to avoid hydration mismatches.
- Prefer real `<button>`/`<a>`/form elements over a `<div onClick>` with a bolted-on role; never remove a focus outline without a visible `:focus-visible` replacement.
- Trap and restore keyboard focus correctly for modals; never create a keyboard trap; give every icon-only control an accessible name.
- Use ARIA only when native HTML can't provide the needed semantics; implement full keyboard operability (Tab/Enter/Space/Escape/arrows) for custom widgets.
- Use `aria-live="polite"` (not `assertive`) for routine dynamic announcements; associate form errors/required state programmatically, not just visually.
- Use semantic landmarks and correct heading hierarchy; set `lang`; write real `alt` text (or `alt=""` for decorative images); caption video/audio.
- Respect `prefers-reduced-motion`; never let autoplaying/moving content be unpausable; confirm before irreversible/destructive actions; warn before session timeout.
- Manually test complex interactive components with a real screen reader — don't rely on automated scans alone.
- Never invent a hook, API, prop, package, or CSS/utility class that hasn't been verified to exist in the project's actual dependencies and version.
- Always include explicit loading, error, and empty states in any component that renders fetched data.
- Check the existing design system for a component before hand-rolling a new button/modal/input from scratch; search for existing utilities before writing new ones.
- Match the surrounding codebase's conventions (style, exports, framework version/idioms) rather than defaulting to generic training-data patterns.
- Don't add an unrequested dependency or unrequested abstraction layer to satisfy a narrowly-scoped request.
- Make the smallest targeted edit for a small requested change — don't regenerate an entire component from scratch.
