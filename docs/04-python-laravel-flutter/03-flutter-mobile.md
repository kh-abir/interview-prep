# 03 — Flutter & Dart: Mobile Architecture, State Engineering & Native Systems

> **Context**: Mobile Application Engineering & Cross-Platform Systems (JD-CRITICAL). Comprehensive coverage of the Dart runtime, Skia/Impeller rendering pipeline, the Three Trees architecture, state management (Provider, Riverpod, Bloc), deep routing via GoRouter, resilient networking with Dio, and native platform channels.

---

## 1. Dart Language Internals & Asynchronous Runtime

### 1.1 Sound Null Safety, Type Promotion & Memory Semantics

#### 1. Definition & Core Concept
Dart features **Sound Null Safety**. In Dart, types are non-nullable by default (`String` cannot be `null`). Nullability is explicitly marked with `?` (`String?`). "Soundness" means that if the static type system determines an expression is not null, it **can never be null at runtime**, allowing the Dart AOT (Ahead-of-Time) compiler to emit smaller, faster machine code without redundant defensive null-checks.

#### 2. Internal Mechanics
1. **Flow Analysis & Type Promotion**:
   The Dart compiler analyzes control flow. If a nullable variable is checked for null inside a condition (`if (x != null)`), the compiler **promotes** the variable from `T?` to `T` within that scope.
   - *Limitation*: Flow analysis does not promote public class fields or getters because another thread, getter mutation, or subclass override could theoretically return null on subsequent reads.
2. **The `late` Keyword**:
   - Deferred Assignment: Marks a non-nullable variable that is initialized *after* object construction.
   - Lazy Initialization: If a `late` variable has an initializer (`late final client = initClient()`), the expression is evaluated only when the variable is **first accessed**, not during class instantiation.
   - *Trap*: Accessing an uninitialized `late` variable throws a runtime `LateInitializationError`.

```
Nullable Type Hierarchy:
               Object?  (Top Type)
              /       \
          Object       Null (Only instance is null)
         /   |  \
      int  String  List<T>
         \   |   /
           Never        (Bottom Type: functions that always throw / loop)
```

#### 3. Production Code & Real-World Usage

```dart
class AccountProfile {
  final String id;
  final String email;
  String? avatarUrl; // Nullable: may not exist

  // Lazy expensive resource: only instantiated if accessed
  late final String profileChecksum = _computeHeavyHash();

  AccountProfile({
    required this.id,
    required this.email,
    this.avatarUrl,
  });

  String _computeHeavyHash() {
    return 'checksum_${id}_${email.hashCode}';
  }

  // Safe flow-analyzed promotion pattern
  String getDisplayName(String? fallbackNickname) {
    // Local copy enables reliable flow analysis promotion
    final localAvatar = avatarUrl;
    if (localAvatar != null) {
      return 'User with avatar: $localAvatar'; // localAvatar promoted to String
    }

    // Null-coalescing assignment and fallback
    return fallbackNickname ?? email.split('@').first;
  }
}
```

#### 4. Production Pitfalls & Debugging
- **Overusing the Null Assertion Operator (`!`)**: The bang operator (`!`) forcefully casts `T?` to `T`, bypassing compile-time safety. If the value is `null`, it throws `NullCheckError` at runtime, causing an immediate application crash.
- **Uninitialized `late` Variables**: Relying on `late` for asynchronous dependency injection often results in race conditions where widgets access the field before `initState` completes, crashing with `LateInitializationError: Field 'user' has not been initialized`.

#### 5. Trade-offs & Decision Matrix

| Mechanism | Safety Level | Evaluation Timing | Memory Overhead | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **`T?` (Nullable)** | Compile-time Enforced | Eager | Minimal pointer overhead | Optional data, API responses, nullable states |
| **`late` (Deferred)** | Runtime Checked | Deferred | Extra internal sentinel check flag | Dependency injection before usage, lazy initialization |
| **`late final` (Lazy)** | Runtime Checked | On First Access | Single evaluation cache flag | Expensive computations, heavy singleton instances |

#### 6. Senior Interview Q&A
- **Q**: *Why does Dart not allow type promotion on class member fields (e.g., `if (this.name != null) return this.name.length;`)?*
- **A**: Class getters can be overridden by subclasses or can contain custom logic that returns `null` on subsequent calls. Furthermore, field accessors cannot be statically guaranteed to be immutable across asynchronous gaps (`await` boundaries). To promote a class field, assign it to a local variable first: `final name = this.name; if (name != null) return name.length;`.

---

### 1.2 Dart Event Loop: Microtasks vs. Event Queue

#### 1. Definition & Core Concept
Dart is a **single-threaded**, event-driven programming language. Concurrency is achieved via an event loop backed by two distinct queues: the **Microtask Queue** and the **Event Queue**.

#### 2. Internal Mechanics

```
Dart Single-Thread Execution:
┌─────────────────────────────────────────────────────────────┐
│                   Main Synchronous Stack                    │
└──────────────────────────────┬──────────────────────────────┘
                               │ (Sync code completes)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      Microtask Queue                        │
│   (scheduleMicrotask, Future.microtask, stream controllers) │
└──────────────────────────────┬──────────────────────────────┘
                               │ (Must be COMPLETELY EMPTY)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                        Event Queue                          │
│   (I/O, Timers, HTTP requests, Gestures, Native Channels)   │
└─────────────────────────────────────────────────────────────┘
```

1. **Microtask Queue**: Has absolute execution priority over the Event Queue. Microtasks are used for internal bookkeeping and operations that must complete before control returns to the event loop.
2. **Event Queue**: Handles external events: UI touch gestures, network socket data, timers (`Timer.periodic`), and native platform channel messages.
3. **Starvation Hazard**: If a recursive or continuous microtask is scheduled (`scheduleMicrotask(run)`), the Event Queue will **never process**. UI rendering freezes, gesture events are dropped, and the app exhibits an Application Not Responding (ANR) lockup.

#### 3. Production Code & Real-World Usage

```dart
import 'dart:async';

void demonstrateEventLoopOrder() {
  print('1. Synchronous Start');

  Timer.run(() {
    print('5. Event Queue: Timer callback executed');
  });

  scheduleMicrotask(() {
    print('3. Microtask Queue: Immediate microtask executed');
  });

  Future.microtask(() {
    print('4. Microtask Queue: Future.microtask executed');
  });

  print('2. Synchronous End');

  // Output:
  // 1. Synchronous Start
  // 2. Synchronous End
  // 3. Microtask Queue: Immediate microtask executed
  // 4. Microtask Queue: Future.microtask executed
  // 5. Event Queue: Timer callback executed
}
```

---

### 1.3 Reactive Streams & StreamControllers

#### 1. Definition & Core Concept
A `Stream` is an asynchronous sequence of events. A `StreamController` provides a controller to inject data, errors, and done events into a stream.

#### 2. Internal Mechanics
- **Single-Subscription Stream**: Allows only one listener over its entire lifetime. If a second listener attempts to listen, a `StateError: Bad state: Stream has already been listened to` is thrown.
- **Broadcast Stream**: Allows multiple concurrent listeners. Events are fired as they occur; listeners that subscribe late miss prior events.

#### 3. Production Code & Real-World Usage

```dart
import 'dart:async';

class NetworkConnectivityEngine {
  // Broadcast controller allows multiple widgets to listen to connection changes
  final _controller = StreamController<bool>.broadcast();

  Stream<bool> get onConnectivityChanged => _controller.stream;

  void emitNetworkStatus(bool isConnected) {
    if (!_controller.isClosed) {
      _controller.add(isConnected);
    }
  }

  // Generator-based Stream using async*
  Stream<int> countdownTimer(int start) async* {
    for (var i = start; i >= 0; i--) {
      await Future.delayed(const Duration(seconds: 1));
      yield i; // Lazily emits value to listener
    }
  }

  void dispose() {
    _controller.close();
  }
}
```

---

## 2. Flutter Rendering Pipeline & The Three Trees Architecture

### 2.1 The Three Trees: Widget, Element & RenderObject

#### 1. Definition & Core Concept
Flutter does not wrap native Android/iOS UI controls. It paints pixels directly onto the screen canvas using its own rendering engine (**Impeller** on modern iOS/Android, previously **Skia**). Flutter maintains **Three Parallel Trees** during runtime:

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   Widget Tree   │       │   Element Tree  │       │ RenderObjectTree│
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ • Immutable     │ Creates│ • Mutable       │ Creates│ • Layout        │
│ • Ephemeral     ├──────►│ • Persistent    ├──────►│ • Paint         │
│ • Configuration │       │ • State Holder  │       │ • Hit-Testing   │
└─────────────────┘       └─────────────────┘       └─────────────────┘
```

#### 2. Internal Mechanics
1. **Widget Tree**: Lightweight, immutable declarative configurations. When `setState()` is called, new widget instances are instantiated rapidly. Because they are plain data transfer objects, creating them costs almost nothing on the Dart heap.
2. **Element Tree**: The structural backbone of Flutter. Elements are mutable, persistent objects that manage the lifecycle of widgets and hold onto state (`StatefulElement` holds the `State` instance).
   - *Reconciliation*: When a widget rebuilds, Flutter invokes `Widget.canUpdate(oldWidget, newWidget)`:
     ```dart
     static bool canUpdate(Widget oldWidget, Widget newWidget) {
       return oldWidget.runtimeType == newWidget.runtimeType
           && oldWidget.key == newWidget.key;
     }
     ```
   - If `canUpdate` returns `true`, the existing `Element` is retained, updates its reference to the new widget, and updates the `RenderObject`. No expensive layout/paint reallocations occur!
   - If `false`, the entire sub-tree is unmounted, disposed, and recreated from scratch.
3. **RenderObject Tree**: Responsible for geometry, computing box layout constraints, hit-testing, and drawing pixels via `PaintingContext`.

---

### 2.2 StatefulWidget Lifecycle Deep Dive

#### 1. Definition & Core Concept
A `StatefulWidget` binds an immutable widget configuration to a persistent, mutable `State` instance retained across rebuild passes.

#### 2. Internal Mechanics

```
State Lifecycle Sequence:
┌─────────────────────────────────────────────────────────────┐
│ 1. createState()           -> Element instantiates State    │
│ 2. initState()             -> Exactly ONCE: Allocations     │
│ 3. didChangeDependencies() -> When InheritedWidget updates  │
│ 4. build()                 -> Returns Widget tree           │
│ 5. didUpdateWidget()       -> Parent rebuilt with new props │
│ 6. deactivate()            -> Removed from tree (temp)      │
│ 7. dispose()               -> Permanently destroyed: Teardown│
└─────────────────────────────────────────────────────────────┘
```

- `initState()`: Called once when the Element is inserted into the tree. Ideal for initializing controllers, animations, and subscribing to streams. `BuildContext` is not fully initialized for inherited widgets here.
- `didChangeDependencies()`: Called immediately after `initState()`, and whenever an `InheritedWidget` that this state depends on (`MediaQuery`, `Theme`, `Provider`) mutates.
- `didUpdateWidget(covariant T oldWidget)`: Called whenever the parent widget rebuilds and provides a new widget instance of the same `runtimeType` and `Key`. Perfect for comparing `widget.id != oldWidget.id` to re-fetch data.
- `dispose()`: Permanent destruction. **Must** unsubscribe from streams, cancel timers, and dispose `TextEditingController` / `AnimationController` to prevent memory leaks.

#### 3. Production Code & Real-World Usage

```dart
import 'package:flutter/material.dart';

class RealtimeStockWatcher extends StatefulWidget {
  final String stockSymbol;

  const RealtimeStockWatcher({
    super.key,
    required this.stockSymbol,
  });

  @override
  State<RealtimeStockWatcher> createState() => _RealtimeStockWatcherState();
}

class _RealtimeStockWatcherState extends State<RealtimeStockWatcher> {
  late TextEditingController _notesController;

  @override
  void initState() {
    super.initState();
    _notesController = TextEditingController();
    _subscribeToTicker(widget.stockSymbol);
  }

  @override
  void didUpdateWidget(covariant RealtimeStockWatcher oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Compare new widget property with previous configuration
    if (widget.stockSymbol != oldWidget.stockSymbol) {
      _unsubscribeFromTicker(oldWidget.stockSymbol);
      _subscribeToTicker(widget.stockSymbol);
    }
  }

  void _subscribeToTicker(String symbol) {
    // Socket or stream subscription logic
  }

  void _unsubscribeFromTicker(String symbol) {
    // Teardown subscription
  }

  @override
  void dispose() {
    _unsubscribeFromTicker(widget.stockSymbol);
    _notesController.dispose(); // Prevent native text controller memory leak
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16.0),
      child: Text('Tracking: ${widget.stockSymbol}'),
    );
  }
}
```

---

## 3. Flutter Layout System: Rules, Constraints & Overflows

### 3.1 The Golden Rule of Flutter Layout

> **"Constraints go down. Sizes go up. Parent sets position."**

```
                  ┌─────────────────────────────────┐
                  │ 1. Parent passes down           │
                  │    BoxConstraints (min/max W, H)│
                  └────────────────┬────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────┐
                  │ 2. Child determines its own     │
                  │    Size (width x height)        │
                  └────────────────┬────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────┐
                  │ 3. Parent sets Child position   │
                  │    (Offset dx, dy) in its canvas│
                  └─────────────────────────────────┘
```

- **Tight Constraints**: `minWidth == maxWidth` and `minHeight == maxHeight`. The child has no freedom and is forced to take the exact size (e.g. `SizedBox.expand()`).
- **Loose Constraints**: `minWidth == 0` and `minHeight == 0`. The child can choose any size up to the maximum width/height.
- **Unbounded Constraints**: `maxHeight == double.infinity` (occurs inside `Column`, `ListView`, or `SingleChildScrollView`). Placing a child that attempts to expand to infinity inside an unbounded parent triggers a runtime crash: `Vertical viewport was given unbounded height`.

---

### 3.2 Flex, Expanded, Flexible & Overflow Resolution

#### 1. Definition & Core Concept
`Row` and `Column` extend `Flex`. They lay out children along a **Main Axis** and a **Cross Axis**.
- `Expanded`: Enforces `FlexFit.tight`. The child **must** fill all available space allocated to its flex factor.
- `Flexible`: Enforces `FlexFit.loose` by default. The child can take **up to** its allocated flex space, but can shrink if its content is smaller.

#### 2. Production Code & Real-World Usage

```dart
import 'package:flutter/material.dart';

class OrderItemCard extends StatelessWidget {
  final String title;
  final String description;
  final String price;

  const OrderItemCard({
    super.key,
    required this.title,
    required this.description,
    required this.price,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Fixed avatar size
            const CircleAvatar(radius: 24, child: Icon(Icons.shopping_cart)),
            const SizedBox(width: 12),
            
            // Expanded forces column to take all remaining horizontal space,
            // preventing "RenderFlex overflowed" when text is excessively long.
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    title,
                    style: Theme.of(context).textTheme.titleMedium,
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const SizedBox(height: 4),
                  Text(
                    description,
                    style: Theme.of(context).textTheme.bodySmall,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                ],
              ),
            ),
            const SizedBox(width: 8),
            
            // Text retains its intrinsic content size
            Text(
              price,
              style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### 3. Production Pitfalls & Debugging
- **RenderFlex Overflow Errors**: When children of a `Row` or `Column` exceed available physical pixels along the main axis, Flutter renders yellow-and-black caution stripes and logs:
  `A RenderFlex overflowed by 48 pixels on the right.`
  - *Fix*: Wrap flex children in `Expanded`, `Flexible`, or replace the `Row`/`Column` with `Wrap` or `SingleChildScrollView`.
- **`MediaQuery.of(context)` Rebuild Cascades**: Calling `MediaQuery.of(context).size` subscribes the entire widget to **all** MediaQuery changes, including keyboard appearance (`viewInsets`) and orientation changes.
  - *Fix*: Use `MediaQuery.sizeOf(context)` or `MediaQuery.paddingOf(context)` (introduced in Flutter 3.10) to subscribe only to specific property changes, eliminating redundant builds.

---

## 4. Modern Navigation: Declarative Routing with GoRouter

### 4.1 Imperative (Navigator 1.0) vs. Declarative (GoRouter / Navigator 2.0)

#### 1. Definition & Core Concept
- **Navigator 1.0**: Imperative stack manipulation (`push()`, `pop()`). Fragile for deep-linking, complex nested routing, and web browser history synchronization.
- **GoRouter**: A declarative routing package built on top of Flutter’s Router API (Navigator 2.0). It defines application navigation via uniform URL paths, supporting path parameters, query parameters, shell routes (persistent bottom navigation), and asynchronous authentication redirection guards.

#### 2. Production Code & Real-World Usage

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// Simulated Auth State
class AuthRepository extends ChangeNotifier {
  bool _isAuthenticated = false;
  bool get isAuthenticated => _isAuthenticated;

  void login() {
    _isAuthenticated = true;
    notifyListeners();
  }

  void logout() {
    _isAuthenticated = false;
    notifyListeners();
  }
}

final authRepo = AuthRepository();

// Declarative GoRouter Configuration
final goRouterConfig = GoRouter(
  initialLocation: '/dashboard',
  refreshListenable: authRepo, // Automatically re-evaluates redirect guards on auth change
  redirect: (BuildContext context, GoRouterState state) {
    final loggedIn = authRepo.isAuthenticated;
    final isLoggingIn = state.matchedLocation == '/login';

    if (!loggedIn && !isLoggingIn) {
      return '/login'; // Force unauthenticated users to login
    }
    if (loggedIn && isLoggingIn) {
      return '/dashboard'; // Redirect authenticated users away from login
    }
    return null; // Allow navigation
  },
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => const Scaffold(body: Center(child: Text('Login Screen'))),
    ),
    // ShellRoute for persistent BottomNavigationBar across tabs
    ShellRoute(
      builder: (context, state, child) {
        return Scaffold(
          body: child,
          bottomNavigationBar: BottomNavigationBar(
            currentIndex: state.matchedLocation.startsWith('/dashboard') ? 0 : 1,
            onTap: (index) {
              if (index == 0) context.go('/dashboard');
              if (index == 1) context.go('/settings');
            },
            items: const [
              BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Dashboard'),
              BottomNavigationBarItem(icon: Icon(Icons.settings), label: 'Settings'),
            ],
          ),
        );
      },
      routes: [
        GoRoute(
          path: '/dashboard',
          builder: (context, state) => const Center(child: Text('Dashboard View')),
          routes: [
            // Nested Deep-link Route with Path Parameter
            GoRoute(
              path: 'order/:orderId',
              builder: (context, state) {
                final orderId = state.pathParameters['orderId']!;
                return Center(child: Text('Order Details: $orderId'));
              },
            ),
          ],
        ),
        GoRoute(
          path: '/settings',
          builder: (context, state) => const Center(child: Text('Settings View')),
        ),
      ],
    ),
  ],
);
```

---

## 5. State Management Paradigms: Provider, Riverpod & Bloc

### 5.1 Architecture Comparison Matrix

| State Solution | Architecture Paradigm | BuildContext Required? | Testability | Boilerplate | Best Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Provider** | `InheritedWidget` Wrapper | Yes | High | Low | Small-to-medium apps, simple shared UI state |
| **Riverpod** | Global Compile-Safe Providers | **No** | Maximum (overridable) | Low to Medium | High-scale modern apps, async caching, DI replacement |
| **Bloc / Cubit** | Event-Driven Reactive Streams | No (via BlocProvider) | Maximum | Moderate to High | Strict enterprise apps, financial transactions, audit logs |

---

### 5.2 Riverpod: Modern Compile-Safe State Architecture

#### 1. Definition & Core Concept
Riverpod is a rewrite of Provider that is completely decoupled from the Flutter widget tree. It does not rely on `BuildContext`, catches provider errors at compile-time instead of runtime, and natively handles asynchronous caching via `AsyncNotifierProvider`.

#### 2. Production Code & Real-World Usage

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// 1. Immutable Model
class AccountBalance {
  final double balance;
  final String currency;
  const AccountBalance({required this.balance, required this.currency});
}

// 2. AsyncNotifier for state handling with caching & mutations
class AccountBalanceNotifier extends AutoDisposeAsyncNotifier<AccountBalance> {
  @override
  Future<AccountBalance> build() async {
    // Fetches initial data asynchronously on first listen
    return _fetchBalanceFromApi();
  }

  Future<AccountBalance> _fetchBalanceFromApi() async {
    await Future.delayed(const Duration(seconds: 1)); // Simulated network latency
    return const AccountBalance(balance: 14500.50, currency: 'USD');
  }

  Future<void> refreshBalance() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _fetchBalanceFromApi());
  }
}

// Provider Declaration
final balanceProvider = AsyncNotifierProvider.autoDispose<AccountBalanceNotifier, AccountBalance>(
  AccountBalanceNotifier.new,
);

// 3. ConsumerWidget UI Implementation
class AccountOverviewCard extends ConsumerWidget {
  const AccountOverviewCard({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch automatically triggers rebuild when AsyncValue state changes
    final balanceAsync = ref.watch(balanceProvider);

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: balanceAsync.when(
          data: (account) => Column(
            children: [
              Text('Available Balance: ${account.currency} ${account.balance}'),
              ElevatedButton(
                onPressed: () => ref.read(balanceProvider.notifier).refreshBalance(),
                child: const Text('Refresh'),
              ),
            ],
          ),
          loading: () => const CircularProgressIndicator(),
          error: (err, stack) => Text('Error loading balance: $err'),
        ),
      ),
    );
  }
}
```

---

### 5.3 Bloc / Cubit: Enterprise Event-Driven Reactive State

#### 1. Definition & Core Concept
The **BLoC (Business Logic Component)** pattern enforces strict separation between presentation and business logic.
- **Cubit**: Method-driven state emission (`emit(NewState)`).
- **Bloc**: Formal Event-driven state machine. Events are dispatched into an incoming sink, transformed (debounced, throttled), and converted into States emitted via streams.

#### 2. Production Code & Real-World Usage

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:equatable/equatable.dart';

// 1. Events
abstract class AuthEvent extends Equatable {
  const AuthEvent();
  @override
  List<Object?> get props => [];
}

class LoginSubmittedEvent extends AuthEvent {
  final String email;
  final String password;
  const LoginSubmittedEvent(this.email, this.password);
  @override
  List<Object?> get props => [email, password];
}

// 2. States
abstract class AuthState extends Equatable {
  const AuthState();
  @override
  List<Object?> get props => [];
}

class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState {
  final String userId;
  const AuthAuthenticated(this.userId);
  @override
  List<Object?> get props => [userId];
}
class AuthFailure extends AuthState {
  final String message;
  const AuthFailure(this.message);
  @override
  List<Object?> get props => [message];
}

// 3. Bloc Implementation
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc() : super(AuthInitial()) {
    // Event Handler registration
    on<LoginSubmittedEvent>((event, emit) async {
      emit(AuthLoading());
      try {
        if (event.password.length < 6) {
          emit(const AuthFailure('Password must be at least 6 characters'));
          return;
        }
        await Future.delayed(const Duration(seconds: 1)); // Network auth
        emit(const AuthAuthenticated('usr_9912'));
      } catch (e) {
        emit(AuthFailure(e.toString()));
      }
    });
  }
}
```

---

## 6. Network Engineering with Dio & Secure Native Platform Channels

### 6.1 Enterprise Networking with Dio: Refresh Queuing & Mutex

#### 1. Definition & Core Concept
In production mobile apps, access tokens expire while users are actively navigating. When multiple API requests fail simultaneously with `401 Unauthorized`, naive implementations trigger multiple concurrent token refresh calls. A production **Dio Interceptor** uses a lock/queue pattern to serialize refresh requests.

#### 2. Production Code & Real-World Usage

```dart
import 'package:dio/dio.dart';
import 'dart:async';

class ResilientDioClient {
  late final Dio dio;
  bool _isRefreshing = false;
  final List<Completer<String>> _refreshQueue = [];

  ResilientDioClient() {
    dio = Dio(BaseOptions(
      baseUrl: 'https://api.vivasoft.com/v1',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 15),
      headers: {'Accept': 'application/json'},
    ));

    dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) async {
          final token = await _getStoredAccessToken();
          if (token != null) {
            options.headers['Authorization'] = 'Bearer $token';
          }
          return handler.next(options);
        },
        onError: (DioException error, handler) async {
          if (error.response?.statusCode == 401) {
            final options = error.requestOptions;

            // Prevent infinite refresh loop if refresh endpoint itself failed with 401
            if (options.path.contains('/auth/refresh')) {
              return handler.next(error);
            }

            try {
              final newToken = await _synchronizedTokenRefresh();
              
              // Retry original failed request with updated token
              options.headers['Authorization'] = 'Bearer $newToken';
              final retryResponse = await dio.fetch(options);
              return handler.resolve(retryResponse);
            } catch (refreshErr) {
              return handler.next(error);
            }
          }
          return handler.next(error);
        },
      ),
    );
  }

  Future<String> _synchronizedTokenRefresh() async {
    if (_isRefreshing) {
      // Queue subsequent failing requests until active refresh completes
      final completer = Completer<String>();
      _refreshQueue.add(completer);
      return completer.future;
    }

    _isRefreshing = true;

    try {
      final refreshToken = await _getStoredRefreshToken();
      final response = await dio.post('/auth/refresh', data: {'refresh_token': refreshToken});
      final newAccessToken = response.data['access_token'] as String;

      await _saveAccessToken(newAccessToken);

      // Release queued callers
      for (final completer in _refreshQueue) {
        completer.complete(newAccessToken);
      }
      _refreshQueue.clear();

      return newAccessToken;
    } catch (e) {
      for (final completer in _refreshQueue) {
        completer.completeError(e);
      }
      _refreshQueue.clear();
      rethrow;
    } finally {
      _isRefreshing = false;
    }
  }

  Future<String?> _getStoredAccessToken() async => 'access_tok_xyz';
  Future<String?> _getStoredRefreshToken() async => 'refresh_tok_abc';
  Future<void> _saveAccessToken(String token) async {}
}
```

---

### 6.2 Platform Channels: Native Hardware Communication

#### 1. Definition & Core Concept
Flutter uses **Platform Channels** to communicate with native platform code (Android Kotlin/Java and iOS Swift/Objective-C). Communication is asynchronous and serialized through binary messaging codecs.
- `MethodChannel`: Named channel for invoking platform methods and receiving responses.
- `EventChannel`: Channel for streaming continuous events (e.g. accelerometer, battery percentage).

#### 2. Internal Mechanics

```
Dart UI Thread (Flutter)
           │
           ▼
MethodChannel.invokeMethod('getBatteryLevel')
           │
           ├── Serialized via StandardMethodCodec into binary buffer
           └── Transferred across C++ engine boundary to Platform Thread
                                  │
                                  ▼
Android (Kotlin) / iOS (Swift) UI Thread
           ├── Decodes message arguments
           ├── Executes native hardware API (e.g. BatteryManager)
           └── Sends binary reply back across platform channel
```

#### 3. Production Code & Real-World Usage

**Flutter Dart Side:**
```dart
import 'package:flutter/services.dart';

class NativeDeviceBridge {
  static const MethodChannel _channel = MethodChannel('com.vivasoft.app/device');

  static Future<int> getBatteryLevel() async {
    try {
      final int batteryLevel = await _channel.invokeMethod('getBatteryLevel');
      return batteryLevel;
    } on PlatformException catch (e) {
      throw Exception('Failed to read battery level: ${e.message}');
    }
  }
}
```

**Android Native Side (Kotlin `MainActivity.kt`):**
```kotlin
package com.vivasoft.app

import android.content.Context
import android.os.BatteryManager
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity: FlutterActivity() {
    private val CHANNEL = "com.vivasoft.app/device"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL).setMethodCallHandler { call, result ->
            if (call.method == "getBatteryLevel") {
                val batteryManager = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
                val batteryLevel = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
                if (batteryLevel != -1) {
                    result.success(batteryLevel)
                } else {
                    result.error("UNAVAILABLE", "Battery level not accessible", null)
                }
            } else {
                result.notImplemented()
            }
        }
    }
}
```

---

### 6.3 Application Lifecycle Management

#### 1. Definition & Core Concept
`WidgetsBindingObserver` tracks transitions between operating system states (`resumed`, `inactive`, `paused`, `detached`, `hidden`).

#### 2. Production Code & Real-World Usage

```dart
import 'package:flutter/material.dart';

class LifecycleAwareEngine extends StatefulWidget {
  final Widget child;
  const LifecycleAwareEngine({super.key, required this.child});

  @override
  State<LifecycleAwareEngine> createState() => _LifecycleAwareEngineState();
}

class _LifecycleAwareEngineState extends State<LifecycleAwareEngine> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    super.didChangeAppLifecycleState(state);
    switch (state) {
      case AppLifecycleState.resumed:
        // App returned to foreground: Re-authenticate biometric session, reconnect WebSockets
        print('App in foreground: Syncing pending transactions...');
        break;
      case AppLifecycleState.inactive:
        // Transitory state: incoming phone call or native permission prompt
        break;
      case AppLifecycleState.paused:
        // App placed in background: Pause heavy animations, release camera/audio resources
        print('App backgrounded: Saving local draft state...');
        break;
      case AppLifecycleState.detached:
        // Engine is terminating
        break;
      case AppLifecycleState.hidden:
        // Flutter 3.13+: Window minimized or hidden behind another application
        break;
    }
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

---

## 7. Senior Interview Q&A for Flutter & Mobile Systems

### Q1: What causes UI jank in Flutter, and how do you diagnose frame drops below 60fps / 120fps?
**Answer**:
UI jank occurs when a frame takes longer than 16.6ms (for 60Hz displays) or 8.3ms (for 120Hz ProMotion displays) to compute and render. Jank stems from two primary threads:
1. **UI Thread Bottlenecks**: Heavy synchronous Dart computation (JSON parsing of 10MB payloads, heavy regex, sorting massive arrays) inside `build()` or event callbacks. Diagnose using the **Flutter DevTools CPU Profiler** to inspect flame charts. Offload CPU-heavy processing to background worker isolates using `compute()` or `Isolate.spawn()`.
2. **Raster Thread Bottlenecks**: Excessive shader compilation or complex rendering commands (e.g. `BackdropFilter`, excessive saveLayers, un-cached clipping paths). Diagnose using **DevTools Performance Overlay**. The modern **Impeller** rendering engine eliminates runtime shader compilation jank by precompiling shaders ahead-of-time during Flutter engine build.

### Q2: Why is `const` constructor usage so heavily recommended across Flutter widget definitions?
**Answer**:
When a widget is instantiated with `const`, the Dart compiler allocates a single canonical instance in memory at compile-time. During the element tree reconciliation pass (`Widget.canUpdate`), Flutter performs an identity check (`identical(oldWidget, newWidget)`). If the two widget references point to the exact same compile-time `const` instance, Flutter skips the reconciliation and build traversal for that entire widget sub-tree immediately, drastically reducing memory allocation pressure and garbage collection pauses.

### Q3: How do you implement offline-first database synchronization in a high-scale Flutter app?
**Answer**:
1. **Local Persistent Storage**: Use embedded high-performance databases like **Isar** or **Drift** (SQLite wrapper) with reactive stream queries (`watch()`).
2. **Optimistic UI Updates**: When the user performs an action (e.g., creating an order), insert the record locally with a `sync_status = PENDING` flag and update the UI instantly via reactive streams.
3. **Outbox Synchronization Queue**: Store mutations as pending actions in an Outbox table. A background synchronization worker (using `dio` and connectivity observers) processes mutations in strict chronological order with idempotency keys. On successful server commit, mark `sync_status = SYNCHRONIZED`.
4. **Conflict Resolution**: Implement Last-Write-Wins (LWW) using server timestamps, or return a 409 Conflict status forcing the client to merge remote state.
