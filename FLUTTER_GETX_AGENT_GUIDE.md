# Flutter + GetX Agentic Development Guide

> **Purpose:** This document is the engineering contract for an AI/agentic coding assistant working on this Flutter project.
>
> **Primary stack:** Flutter + Dart + GetX.
>
> **Architecture:** MVC-oriented feature structure with strict separation between View, Controller, Model, Repository, and Service responsibilities.
>
> **Core rule:** Prefer simple, explicit, reusable code over clever abstractions. Do not create abstractions unless they remove real duplication or isolate a real responsibility.

---

## 1. Non-Negotiable Rules

The agent MUST follow these rules for every implementation.

### 1.1 Use GetX correctly

Use the official GetX mechanisms rather than inventing a custom state-management layer.

- `GetxController` for screen/feature controllers.
- `.obs` / `Rx<T>` for genuinely reactive state.
- `Obx` for small, targeted reactive UI rebuilds.
- `GetBuilder` for explicit block-based updates when reactive state is unnecessary.
- `Bindings` / `BindingsBuilder` for dependency registration per route.
- `Get.find<T>()` to retrieve registered dependencies.
- `onInit()` for controller initialization.
- `onReady()` for work that should happen after the first frame when appropriate.
- `onClose()` for cleanup.
- `GetMaterialApp` when using GetX routing/dependency features.
- `GetPage` and route bindings for feature-level dependency setup.

Do NOT create a custom `ChangeNotifier`, Provider-like layer, or custom observable abstraction just to replace GetX.

Official GetX documentation describes `.obs`, `Obx`, `GetBuilder`, controller lifecycle methods, bindings, and the distinction between reactive and simple state management. citeturn1search0

---

## 2. State Management Policy

### 2.1 Default: Reactive state

Use reactive state when a value is independently observed by the UI.

```dart
class OrdersController extends GetxController {
  final isLoading = false.obs;
  final orders = <OrderModel>[].obs;
  final errorMessage = RxnString();
}
```

UI:

```dart
Obx(() {
  if (controller.isLoading.value) {
    return const LoadingWidget();
  }

  return OrderList(orders: controller.orders);
})
```

### 2.2 Use GetBuilder when a group of widgets should update together

```dart
class CartController extends GetxController {
  int itemCount = 0;

  void addItem() {
    itemCount++;
    update(['cart_summary']);
  }
}
```

```dart
GetBuilder<CartController>(
  id: 'cart_summary',
  builder: (controller) {
    return Text('${controller.itemCount}');
  },
)
```

Do not convert every variable into `.obs`.

### 2.3 Do not overuse Obx

Bad:

```dart
Obx(() => Scaffold(
  appBar: ...,
  body: ...,
  bottomNavigationBar: ...,
))
```

Better:

```dart
Scaffold(
  appBar: ...,
  body: Obx(() => OrderList(
    orders: controller.orders,
  )),
)
```

Keep reactive boundaries as small as practical.

### 2.4 Never put business logic inside Obx

Bad:

```dart
Obx(() {
  final total = controller.orders
      .where(...)
      .map(...)
      .reduce(...);

  return Text('$total');
})
```

Prefer:

```dart
final totalAmount = controller.totalAmount;
```

with the calculation owned by the controller/model layer.

---

# 3. MVC Project Structure

Use a feature-first MVC structure.

Recommended structure:

```text
lib/
├── app/
│   ├── bindings/
│   │   └── initial_binding.dart
│   ├── routes/
│   │   ├── app_pages.dart
│   │   └── app_routes.dart
│   └── theme/
│       ├── app_theme.dart
│       ├── app_colors.dart
│       ├── app_text_styles.dart
│       └── app_dimensions.dart
│
├── core/
│   ├── constants/
│   │   ├── api_constants.dart
│   │   ├── app_constants.dart
│   │   └── storage_keys.dart
│   ├── errors/
│   │   ├── app_exception.dart
│   │   ├── failure.dart
│   │   └── error_mapper.dart
│   ├── network/
│   │   ├── api_client.dart
│   │   ├── api_interceptors.dart
│   │   └── network_info.dart
│   ├── services/
│   │   ├── storage_service.dart
│   │   ├── auth_service.dart
│   │   └── connectivity_service.dart
│   ├── utils/
│   │   ├── validators.dart
│   │   ├── date_utils.dart
│   │   └── debounce_helper.dart
│   └── widgets/
│       ├── app_button.dart
│       ├── app_text_field.dart
│       ├── app_loader.dart
│       ├── app_error_view.dart
│       └── app_empty_view.dart
│
├── data/
│   ├── models/
│   ├── repositories/
│   └── services/
│
├── features/
│   ├── auth/
│   │   ├── bindings/
│   │   │   └── auth_binding.dart
│   │   ├── controllers/
│   │   │   └── login_controller.dart
│   │   ├── models/
│   │   │   └── login_request.dart
│   │   ├── repositories/
│   │   │   └── auth_repository.dart
│   │   ├── services/
│   │   │   └── auth_api_service.dart
│   │   └── views/
│   │       ├── login_view.dart
│   │       └── widgets/
│   │           └── login_form.dart
│   │
│   └── home/
│       ├── bindings/
│       ├── controllers/
│       ├── models/
│       ├── repositories/
│       ├── services/
│       └── views/
│
└── main.dart
```

### Important

The exact folders may be simplified for a very small feature.

Do not create empty architectural layers just to satisfy a diagram.

If a feature only needs a controller and repository, do not create five unused classes.

---

# 4. Responsibility of Each Layer

## View

Responsible only for:

- Layout.
- Widget composition.
- User interaction forwarding.
- Displaying controller state.
- Simple presentation-only conditions.
- Navigation triggers.

The View MUST NOT:

- Call Dio directly.
- Call APIs.
- Parse JSON.
- Contain business rules.
- Access repositories directly.
- Manage authentication logic.
- Perform database operations.
- Contain large calculations.

Flutter's architecture guidance also recommends keeping logic out of widgets and separating UI/data responsibilities. citeturn0search0turn0search1

---

## Controller

The Controller is the UI-facing state and orchestration layer.

Responsible for:

- UI state.
- User actions.
- Form state.
- Calling repositories.
- Converting repository results into UI state.
- Validation orchestration.
- Loading/error/success state.
- Pagination state.
- Search/filter/sort state.
- Navigation decisions where appropriate.

Controller MUST NOT:

- Perform raw HTTP calls.
- Know Dio implementation details.
- Parse API JSON manually.
- Contain reusable API infrastructure.
- Directly manipulate database internals.

Example:

```dart
class LoginController extends GetxController {
  LoginController(this._repository);

  final AuthRepository _repository;

  final isLoading = false.obs;
  final errorMessage = RxnString();

  Future<void> login() async {
    if (!_validateForm()) return;

    isLoading.value = true;
    errorMessage.value = null;

    final result = await _repository.login(
      username: usernameController.text.trim(),
      password: passwordController.text,
    );

    result.when(
      success: (_) => _handleLoginSuccess(),
      failure: (failure) => _handleFailure(failure),
    );

    isLoading.value = false;
  }

  bool _validateForm() {
    // validation
    return true;
  }

  void _handleLoginSuccess() {
    Get.offAllNamed(AppRoutes.home);
  }

  void _handleFailure(Failure failure) {
    errorMessage.value = failure.userMessage;
  }
}
```

---

# 5. Model Rules

Models represent application/API data.

Prefer immutable models where practical.

Example:

```dart
class UserModel {
  const UserModel({
    required this.id,
    required this.name,
  });

  final int id;
  final String name;

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as int,
      name: json['name'] as String? ?? '',
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
    };
  }
}
```

Rules:

- No API calls in models.
- No GetX dependency inside normal data models.
- `fromJson` belongs in the model or a dedicated mapper for complex APIs.
- Avoid dynamic fields when a real type can be used.
- Handle nullable backend values explicitly.
- Do not silently assume a backend field is always present.

For large applications, separate API DTOs from domain models when the API contract and UI/domain representation differ substantially.

---

# 6. Repository Rules

Repositories are the source of truth for feature data.

They sit between Controllers and Services.

Flow:

```text
View
  ↓
Controller
  ↓
Repository
  ↓
Service
  ↓
API / Database / Platform
```

Repository responsibilities:

- Coordinate one or more services.
- Convert service responses into models.
- Apply business/data rules.
- Cache data where required.
- Handle retry policy where appropriate.
- Coordinate local/remote data.
- Provide a stable interface to Controllers.

Flutter's architecture guidance specifically recommends repositories as the source of truth and services as external-data wrappers. citeturn0search1turn0search9

Prefer an abstract repository contract:

```dart
abstract class AuthRepository {
  Future<Result<UserModel>> login({
    required String username,
    required String password,
  });
}
```

Implementation:

```dart
class AuthRepositoryImpl implements AuthRepository {
  AuthRepositoryImpl(this._service);

  final AuthApiService _service;

  @override
  Future<Result<UserModel>> login({
    required String username,
    required String password,
  }) async {
    try {
      final response = await _service.login(
        username: username,
        password: password,
      );

      return Success(UserModel.fromJson(response.data));
    } catch (error) {
      return FailureResult(ErrorMapper.from(error));
    }
  }
}
```

Repositories should not depend on other repositories unless there is a strong architectural reason.

---

# 7. Service Rules

Services are responsible for external communication.

Examples:

- REST API service.
- Local storage service.
- Firebase service.
- Maps service.
- Device/platform service.

A service should:

- Perform the actual external operation.
- Return typed/raw response data.
- Know endpoint paths.
- Know request parameters.
- Know serialization requirements.
- NOT own UI state.

A service should NOT:

```dart
Get.snackbar(...)
```

or:

```dart
Get.toNamed(...)
```

or:

```dart
controller.isLoading.value = false;
```

Services must remain UI-agnostic.

---

# 8. API Architecture

Use one configured HTTP client.

Example:

```text
ApiClient
    ↓
Interceptors
    ↓
Feature API Service
    ↓
Repository
    ↓
Controller
```

Do not instantiate Dio repeatedly:

Bad:

```dart
final dio = Dio();
```

inside every service.

Preferred:

```dart
class ApiClient {
  ApiClient(this.dio);

  final Dio dio;
}
```

Register it once through GetX dependency injection.

---

## 8.1 API Client Responsibilities

The shared API client should handle:

- Base URL.
- Connect timeout.
- Receive timeout.
- Send timeout.
- Headers.
- Authorization.
- Logging in development.
- Response normalization where appropriate.
- Interceptors.
- Token refresh coordination.

Example:

```dart
class ApiClient {
  ApiClient() {
    dio = Dio(
      BaseOptions(
        baseUrl: ApiConstants.baseUrl,
        connectTimeout: const Duration(seconds: 20),
        receiveTimeout: const Duration(seconds: 30),
        sendTimeout: const Duration(seconds: 30),
        headers: {
          'Accept': 'application/json',
        },
      ),
    );

    _configureInterceptors();
  }

  late final Dio dio;

  void _configureInterceptors() {
    dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: _onRequest,
        onResponse: _onResponse,
        onError: _onError,
      ),
    );
  }

  void _onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) {
    handler.next(options);
  }

  void _onResponse(
    Response response,
    ResponseInterceptorHandler handler,
  ) {
    handler.next(response);
  }

  void _onError(
    DioException error,
    ErrorInterceptorHandler handler,
  ) {
    handler.next(error);
  }
}
```

Extract interceptor logic into separate classes when it becomes substantial.

---

# 9. Authentication and Token Refresh

Token refresh must be centralized.

Do NOT implement token refresh independently in every API service.

Recommended:

```text
API Request
   ↓
401
   ↓
Auth/Refresh Interceptor
   ↓
Refresh Token
   ↓
Retry original request
   ↓
Success / Logout
```

For concurrent 401 responses, use a single refresh operation and queue/wait for other failed requests rather than firing multiple refresh requests.

The refresh mechanism must prevent:

- Infinite refresh loops.
- Multiple simultaneous refresh calls.
- Retrying requests after refresh has definitively failed.

If refresh fails:

```text
Clear session
Clear protected local data
Navigate to login
```

The repository/controller should not need to know how token refresh works.

---

# 10. Error Handling

Never expose raw technical exceptions directly to users.

Bad:

```dart
Get.snackbar('Error', e.toString());
```

Prefer a centralized error mapping layer.

Example:

```dart
abstract class Failure {
  const Failure({
    required this.message,
    this.code,
  });

  final String message;
  final String? code;

  String get userMessage => message;
}

class NetworkFailure extends Failure {
  const NetworkFailure({
    required super.message,
    super.code,
  });
}

class ServerFailure extends Failure {
  const ServerFailure({
    required super.message,
    super.code,
  });
}

class UnauthorizedFailure extends Failure {
  const UnauthorizedFailure({
    required super.message,
    super.code,
  });
}

class UnknownFailure extends Failure {
  const UnknownFailure({
    required super.message,
    super.code,
  });
}
```

Central mapper:

```dart
class ErrorMapper {
  static Failure from(Object error) {
    if (error is DioException) {
      return _fromDio(error);
    }

    return UnknownFailure(
      message: 'Something went wrong. Please try again.',
    );
  }

  static Failure _fromDio(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return const NetworkFailure(
          message: 'Request timed out. Please try again.',
        );

      case DioExceptionType.connectionError:
        return const NetworkFailure(
          message: 'Unable to connect to the server.',
        );

      case DioExceptionType.badResponse:
        return _fromStatusCode(error.response?.statusCode);

      default:
        return const UnknownFailure(
          message: 'Something went wrong. Please try again.',
        );
    }
  }

  static Failure _fromStatusCode(int? statusCode) {
    switch (statusCode) {
      case 401:
        return const UnauthorizedFailure(
          message: 'Your session has expired.',
        );
      case 403:
        return const ServerFailure(
          message: 'You are not allowed to perform this action.',
        );
      case 404:
        return const ServerFailure(
          message: 'Requested resource was not found.',
        );
      default:
        return const ServerFailure(
          message: 'Server error. Please try again.',
        );
    }
  }
}
```

---

# 11. Result Pattern

For important repository operations, prefer explicit success/failure results instead of throwing arbitrary exceptions through every layer.

Example:

```dart
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  const Success(this.data);

  final T data;
}

class FailureResult<T> extends Result<T> {
  const FailureResult(this.failure);

  final Failure failure;
}
```

Then:

```dart
final result = await repository.fetchOrders();

switch (result) {
  case Success(data: final data):
    orders.assignAll(data);

  case FailureResult(failure: final failure):
    errorMessage.value = failure.userMessage;
}
```

Use ordinary exceptions where they are appropriate internally, but normalize errors at architectural boundaries.

Flutter's architecture documentation also presents Result-style error handling as a way to make errors explicit across layers. citeturn0search7

---

# 12. Loading, Empty, Error and Success States

Every API-driven screen should explicitly consider:

```text
Initial
Loading
Success with data
Success but empty
Error
Refreshing
Submitting
```

Do not use one global `isLoading` for unrelated operations.

Bad:

```dart
final isLoading = false.obs;
```

for:

- initial loading
- pagination
- refresh
- delete
- upload
- submit

Better:

```dart
final isInitialLoading = false.obs;
final isRefreshing = false.obs;
final isLoadingMore = false.obs;
final isSubmitting = false.obs;
final isDeleting = false.obs;
```

For a screen with a clear single resource lifecycle, GetX `StateMixin<T>` can be used:

```dart
class ProfileController extends GetxController
    with StateMixin<UserModel> {}
```

and:

```dart
change(
  user,
  status: RxStatus.success(),
);
```

Use `StateMixin` when it genuinely simplifies the screen. Do not combine multiple overlapping state systems without a reason.

---

# 13. Controller Lifecycle

Use GetX lifecycle methods correctly.

```dart
class HomeController extends GetxController {
  @override
  void onInit() {
    super.onInit();
    _setupListeners();
    _loadInitialData();
  }

  @override
  void onReady() {
    super.onReady();
    // Work that benefits from the first frame being rendered.
  }

  @override
  void onClose() {
    searchController.dispose();
    scrollController.dispose();
    super.onClose();
  }
}
```

Rules:

- `onInit()` → initialization and subscriptions.
- `onReady()` → post-render work when needed.
- `onClose()` → cleanup.
- Do not put `BuildContext` dependent initialization in controllers unless there is a strong reason.
- Controllers should not retain unnecessary references to Views.

---

# 14. Dependency Injection

Use GetX dependency injection.

Recommended route binding:

```dart
class HomeBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut<HomeRepository>(
      () => HomeRepositoryImpl(
        Get.find<HomeApiService>(),
      ),
    );

    Get.lazyPut<HomeController>(
      () => HomeController(
        Get.find<HomeRepository>(),
      ),
    );
  }
}
```

For application-wide dependencies:

```dart
class InitialBinding extends Bindings {
  @override
  void dependencies() {
    Get.put<ApiClient>(ApiClient(), permanent: true);

    Get.lazyPut<AuthService>(
      () => AuthService(Get.find<ApiClient>()),
      fenix: true,
    );
  }
}
```

Prefer constructor injection:

```dart
HomeController(this.repository);
```

over retrieving every dependency internally:

```dart
final repository = Get.find<HomeRepository>();
```

Constructor injection makes dependencies obvious and testing easier.

Use `Get.find()` primarily at composition/registration boundaries.

---

# 15. Routing

Centralize route names.

```dart
abstract class AppRoutes {
  static const login = '/login';
  static const home = '/home';
  static const profile = '/profile';
}
```

Centralize pages:

```dart
abstract class AppPages {
  static final pages = <GetPage>[
    GetPage(
      name: AppRoutes.login,
      page: () => const LoginView(),
      binding: LoginBinding(),
    ),
    GetPage(
      name: AppRoutes.home,
      page: () => const HomeView(),
      binding: HomeBinding(),
    ),
  ];
}
```

Avoid scattering string routes throughout the application.

Do not navigate from repositories or API services.

---

# 16. Theme Architecture

Theme values must not be duplicated throughout views.

Use:

```text
app/theme/
├── app_theme.dart
├── app_colors.dart
├── app_text_styles.dart
└── app_dimensions.dart
```

Example:

```dart
class AppColors {
  static const primary = Color(0xFF1565C0);
  static const background = Color(0xFFF7F8FA);
  static const error = Color(0xFFD32F2F);
}
```

Theme:

```dart
class AppTheme {
  static ThemeData light() {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: AppColors.primary,
      ),
      scaffoldBackgroundColor: AppColors.background,
    );
  }
}
```

Do not write arbitrary colors repeatedly:

Bad:

```dart
color: Color(0xFF1565C0)
```

Better:

```dart
color: AppColors.primary
```

Prefer `Theme.of(context).colorScheme` and `Theme.of(context).textTheme` for values that should respond to theme configuration.

If dark mode is required, define both themes centrally.

---

# 17. Reusability Rule

This is a critical rule for the agent.

## If code is reusable across files/features

Create a separate class/file.

Example:

Bad:

```dart
class LoginView extends StatelessWidget {
  Widget _buildButton(...) { ... }
}

class RegisterView extends StatelessWidget {
  Widget _buildButton(...) { ... }
}
```

If the button is genuinely shared, create:

```text
core/widgets/app_button.dart
```

and:

```dart
class AppButton extends StatelessWidget {
  ...
}
```

## If code is only a private implementation detail of the same file

Extract it into a private method.

Example:

```dart
Widget _buildHeader() {
  ...
}

Widget _buildForm() {
  ...
}

Widget _buildActions() {
  ...
}
```

### Decision rule

```text
Used by one file?
    ↓
Private method/function in that file.

Used by multiple files?
    ↓
Separate reusable class/helper in its own file.

Used by multiple features?
    ↓
Move to core/shared layer.

Used only by one feature?
    ↓
Keep inside that feature.
```

Do not create a new class/file for every 5-line widget.

---

# 18. Avoid Redundancy

Before creating code, the agent MUST search the project for:

- Existing widgets.
- Existing validators.
- Existing API methods.
- Existing models.
- Existing formatters.
- Existing error handlers.
- Existing dialogs.
- Existing loading widgets.
- Existing theme values.
- Existing repository methods.
- Existing services.

Never duplicate an existing implementation without a concrete reason.

Bad:

```text
UserApiService
UserService
UserNetworkService
UserHttpService
```

when one API service is sufficient.

Bad:

```text
CustomLoading
LoadingWidget
CommonLoader
AppLoader
```

when there is already one project-standard loading component.

---

# 19. Methods vs Classes

Within a file, extract repeated or logically distinct behavior into methods.

Example:

```dart
Future<void> submit() async {
  if (!_validate()) return;

  _setSubmitting(true);

  try {
    final request = _buildRequest();
    final result = await _repository.submit(request);

    _handleSubmitResult(result);
  } finally {
    _setSubmitting(false);
  }
}
```

Do not write one enormous method containing:

- validation
- request construction
- API call
- response parsing
- navigation
- snackbar logic
- UI state

Separate responsibilities into private methods.

---

# 20. API Service Naming

Use predictable names.

```text
AuthApiService
UserApiService
OrderApiService
VehicleApiService
TripApiService
```

Repositories:

```text
AuthRepository
UserRepository
OrderRepository
VehicleRepository
TripRepository
```

Controllers:

```text
LoginController
HomeController
OrderController
VehicleController
TripController
```

Views:

```text
LoginView
HomeView
OrderView
VehicleView
TripView
```

Models:

```text
UserModel
OrderModel
VehicleModel
TripModel
```

---

# 21. Pagination

Pagination belongs primarily to the Controller + Repository flow.

Controller owns:

```dart
final currentPage = 1.obs;
final hasMore = true.obs;
final isLoadingMore = false.obs;
```

Repository fetches data.

Controller decides when to request another page.

Do not put scroll-controller UI logic into the repository.

Prevent duplicate requests:

```dart
if (isLoadingMore.value || !hasMore.value) return;
```

Update pagination only after a successful response.

---

# 22. Search and Debouncing

Do not call the API on every keystroke.

Use a debounce mechanism.

Example:

```dart
Worker? _searchWorker;

@override
void onInit() {
  super.onInit();

  _searchWorker = debounce(
    searchQuery,
    (_) => _performSearch(),
    time: const Duration(milliseconds: 400),
  );
}

@override
void onClose() {
  _searchWorker?.dispose();
  super.onClose();
}
```

Use GetX workers where appropriate.

Do not create unnecessary streams just for basic debounce behavior.

---

# 23. Forms

Controllers should own form state when the form is feature-level state.

```dart
final formKey = GlobalKey<FormState>();

final usernameController = TextEditingController();
final passwordController = TextEditingController();
```

Cleanup:

```dart
@override
void onClose() {
  usernameController.dispose();
  passwordController.dispose();
  super.onClose();
}
```

Validation:

- Keep field-level validators close to the form field when they are purely presentation validation.
- Keep business validation in the Controller/Model/Repository layer.
- Never trust client-side validation as server validation.

---

# 24. Async Safety

Every async operation must consider:

- Duplicate requests.
- Controller disposal.
- Loading-state reset.
- Error state.
- Retry behavior.
- Cancellation where appropriate.

Prefer:

```dart
try {
  ...
} catch (e) {
  ...
} finally {
  isLoading.value = false;
}
```

rather than manually resetting loading state in multiple branches.

---

# 25. Offline and Caching

When offline support is required:

```text
Controller
    ↓
Repository
    ↓
Local + Remote Services
```

Repository decides:

- Local-first vs remote-first.
- Cache freshness.
- Sync.
- Retry.
- Conflict handling.

Do not put Hive/database code directly into Controllers.

For offline-first features, treat local storage as a real data source, not as a random cache scattered throughout UI code.

---

# 26. Logging

Use centralized logging.

Do not scatter:

```dart
print(...)
```

through production code.

Create a project logger:

```dart
class AppLogger {
  static void debug(String message) {
    if (kDebugMode) {
      debugPrint('[DEBUG] $message');
    }
  }

  static void error(
    String message, [
    Object? error,
    StackTrace? stackTrace,
  ]) {
    if (kDebugMode) {
      debugPrint('[ERROR] $message');
      debugPrint('$error');
      debugPrint('$stackTrace');
    }
  }
}
```

Never log:

- Passwords.
- Access tokens.
- Refresh tokens.
- Secrets.
- Sensitive personal information.

---

# 27. API Logging

Development logging should include useful diagnostic information:

```text
METHOD
URL
STATUS
REQUEST DURATION
SAFE REQUEST PARAMETERS
SAFE RESPONSE SUMMARY
```

Do not log sensitive headers.

Do not dump entire production responses unnecessarily.

Use Dio's logging interceptor only in development or configure filtering carefully.

---

# 28. UI Feedback

Controllers may trigger user-facing feedback when appropriate, but keep the mechanism consistent.

Prefer centralized helpers:

```dart
AppSnackbar.success('Saved successfully');
AppSnackbar.error('Unable to save changes');
```

rather than dozens of slightly different `Get.snackbar()` implementations.

However, do not make every repository/service responsible for UI feedback.

---

# 29. Dialogs and Bottom Sheets

Reusable dialogs should become shared widgets/classes.

Feature-specific dialogs should remain inside the feature.

Do not put large dialog widget trees directly into controller methods.

Bad:

```dart
Get.dialog(
  Dialog(
    child: Column(
      ...
    ),
  ),
);
```

for a complex dialog.

Prefer:

```dart
Get.dialog(
  const ConfirmationDialog(),
);
```

---

# 30. Widget Reuse

Create reusable widgets when:

- The same UI appears in multiple places.
- The widget has meaningful configuration.
- The widget represents a domain-neutral UI component.
- The widget is large enough to make the parent unreadable.

Do not create abstractions for every `Row`, `Padding`, or `Container`.

---

# 31. Constants

Centralize repeated constants.

```dart
class ApiConstants {
  static const baseUrl = '...';

  static const login = '/auth/login';
  static const refreshToken = '/auth/refresh';
}
```

Storage:

```dart
class StorageKeys {
  static const accessToken = 'access_token';
  static const refreshToken = 'refresh_token';
  static const user = 'user';
}
```

Avoid magic strings throughout the application.

---

# 32. Security

Never hardcode:

- Passwords.
- API secrets.
- Private keys.
- Production access tokens.

Do not commit `.env` secrets.

Use secure storage for sensitive tokens when required.

Remember:

> Anything shipped inside a mobile application can potentially be inspected.

Do not treat an embedded API key as a secret unless the platform's security model explicitly supports it.

---

# 33. Testing Strategy

At minimum, architecture should allow testing:

```text
Service
Repository
Controller
View
```

Test important logic separately.

Repository tests:

- Success response.
- API error.
- Network error.
- Empty response.
- Cache behavior.

Controller tests:

- Initial state.
- Loading state.
- Success.
- Failure.
- Validation.
- Pagination.
- Search.
- User actions.

Widget tests:

- Loading UI.
- Empty UI.
- Error UI.
- Success UI.
- User interaction.

Flutter recommends testing architectural components separately and together, including services, repositories, ViewModels/controllers, and views. citeturn0search0

---

# 34. Dependency Rules

Dependencies should flow in one direction:

```text
View
 ↓
Controller
 ↓
Repository
 ↓
Service
 ↓
External system
```

Allowed:

```text
Controller → Repository
Repository → Service
Service → ApiClient
View → Controller
```

Avoid:

```text
View → Repository
View → Service
Repository → Controller
Service → Controller
Service → View
Model → Controller
```

This keeps responsibilities predictable and testable.

---

# 35. Feature Implementation Workflow

When asked to implement a new feature, follow this sequence.

### Step 1 — Inspect existing code

Before writing code:

- Search existing architecture.
- Find similar features.
- Find existing API client.
- Find existing repository pattern.
- Find existing theme components.
- Find existing reusable widgets.
- Find existing error handling.
- Find existing route/binding patterns.

Do not create parallel architecture if the project already has a valid implementation.

### Step 2 — Understand the API

Identify:

```text
HTTP method
Endpoint
Headers
Path parameters
Query parameters
Request body
Response body
Pagination
Authentication requirement
Error responses
```

### Step 3 — Design the model

Create typed models.

### Step 4 — Add API service

Only the external communication.

### Step 5 — Add repository

Coordinate service + model + data rules.

### Step 6 — Add Controller

Own UI state and user actions.

### Step 7 — Add Binding

Register repository/controller dependencies.

### Step 8 — Add View

Keep it UI-focused.

### Step 9 — Add route

Register the feature.

### Step 10 — Test

Verify:

- Success.
- Empty.
- Error.
- Loading.
- Retry.
- Navigation.
- Rebuild behavior.

---

# 36. Agent Decision Tree

When generating code, use this decision tree.

```text
Do I need to call an API?
        ↓ yes
Is there an existing API client?
        ↓
Reuse it.
        ↓
Does the feature already have a service?
        ↓ yes
Extend/reuse it.
        ↓ no
Create FeatureApiService.

Do I need data/business coordination?
        ↓ yes
Use/create Repository.

Does UI need state?
        ↓ yes
Use GetxController.

Does the state need independent reactive updates?
        ↓ yes
Use Rx + Obx.

Does a group of widgets update together?
        ↓ yes
Consider GetBuilder.

Does the code repeat across files?
        ↓ yes
Extract reusable class into separate file.

Does the code only help this file?
        ↓ yes
Make it a private method/function.

Does a value appear in multiple UI components?
        ↓
Consider a shared model/state property rather than duplicating it.

Does a class have too many responsibilities?
        ↓
Split it.
```

---

# 37. Anti-Patterns

The agent MUST avoid these.

## God Controller

Bad:

```text
HomeController
 ├── API calls
 ├── Dio setup
 ├── JSON parsing
 ├── Hive operations
 ├── navigation
 ├── dialogs
 ├── theme
 ├── validation
 ├── calculations
 └── business rules
```

Split responsibilities.

## API calls inside View

Never:

```dart
onPressed: () async {
  final response = await Dio().get(...);
}
```

## API calls inside Model

Never:

```dart
class UserModel {
  Future<void> load() async {}
}
```

## GetX inside Service

Avoid:

```dart
class UserApiService {
  Future<void> login() async {
    Get.snackbar(...);
  }
}
```

## Global mutable state everywhere

Avoid arbitrary:

```dart
static var user = ...
```

Use repositories/services and GetX dependency management.

## Duplicate utilities

Search before creating.

## Over-abstraction

Do not create:

```text
BaseController
BaseRepository
BaseService
BaseResponse
BaseUseCase
BaseMapper
BaseManager
```

unless the project has a real, repeated need.

Abstraction must solve duplication or responsibility boundaries.

---

# 38. Naming Conventions

Use Dart/Flutter conventions.

Classes:

```text
PascalCase
```

Variables/methods:

```text
camelCase
```

Files:

```text
snake_case.dart
```

Private members:

```dart
final _repository;
```

Methods:

```dart
Future<void> _loadOrders()
```

Avoid vague names:

```text
Data
Helper
Manager
Handler
Utils
Common
Stuff
```

Prefer specific names:

```text
OrderRepository
TokenStorageService
ApiErrorMapper
DateFormatter
```

---

# 39. Code Quality Rules

Every generated implementation should prioritize:

1. Correctness.
2. Separation of concerns.
3. Reusability.
4. Readability.
5. Testability.
6. Performance.
7. Minimal duplication.

Do not optimize prematurely.

Do not make code shorter at the cost of readability.

Do not make code architecturally complex just to appear "clean".

---

# 40. Comments

Do not comment obvious code.

Bad:

```dart
// Increment count by one.
count++;
```

Good comments explain:

- Why something unusual exists.
- Why a workaround is necessary.
- Why a backend behavior requires special handling.
- Why a specific architecture decision was made.

---

# 41. Documentation Requirements for Agent

When implementing a non-trivial feature, briefly document:

```text
Feature
API endpoints
State variables
Repository responsibility
Service responsibility
Navigation
Special edge cases
```

Do not generate huge documentation for a trivial CRUD screen.

---

# 42. Refactoring Rules

When modifying existing code:

### Do not rewrite everything automatically.

First determine:

```text
What is broken?
What is duplicated?
What violates architecture?
What can be reused?
What must remain for compatibility?
```

Make the smallest safe architectural improvement.

If a refactor changes behavior, explicitly preserve existing behavior unless the user requested a behavior change.

---

# 43. Performance Rules

Avoid unnecessary rebuilds.

Prefer:

```dart
Obx(() => Text(controller.count.value.toString()))
```

over wrapping an entire screen in Obx.

For large lists:

- Use lazy builders.
- Avoid expensive work inside `build`.
- Avoid creating unnecessary Rx variables per list item.
- Use stable keys when appropriate.
- Paginate large remote datasets.
- Avoid repeatedly parsing the same data.

Do not use reactive state for values that never change.

---

# 44. Accessibility and UX

Generated UI should consider:

- Touch target size.
- Text readability.
- Loading feedback.
- Empty states.
- Error recovery.
- Keyboard behavior.
- Form validation.
- Safe areas.
- Different screen sizes.
- Dark mode if supported.
- Localization if supported.

Do not hardcode UI assumptions for one screen size.

---

# 45. Localization

Do not hardcode user-facing strings when the project has localization.

Bad:

```dart
Text('Submit')
```

Prefer the project's localization mechanism.

Keep translation keys organized by feature where the project supports feature-based localization.

Do not put localization logic into repositories/services.

---

# 46. Final Agent Checklist

Before considering an implementation complete, verify:

### Architecture

- [ ] View contains UI only.
- [ ] Controller contains UI state/actions.
- [ ] Repository owns data coordination.
- [ ] Service owns external communication.
- [ ] Models are typed.
- [ ] Dependencies are injected.

### GetX

- [ ] Correct GetX state mechanism selected.
- [ ] `Obx` boundaries are small.
- [ ] `GetBuilder` is used where explicit grouped updates make sense.
- [ ] Bindings register route dependencies.
- [ ] Lifecycle methods are used correctly.
- [ ] Controllers are not initialized repeatedly.

### API

- [ ] Shared API client is reused.
- [ ] No duplicated Dio instances.
- [ ] Errors are mapped centrally.
- [ ] Auth refresh is centralized.
- [ ] Loading states are accurate.
- [ ] Pagination prevents duplicate requests.

### Reusability

- [ ] Existing reusable code was searched for first.
- [ ] Repeated cross-file code was extracted.
- [ ] File-local helpers are private methods.
- [ ] No unnecessary abstraction was introduced.

### UI

- [ ] Loading state.
- [ ] Empty state.
- [ ] Error state.
- [ ] Success state.
- [ ] Retry/recovery.
- [ ] Responsive layout.
- [ ] Theme values are centralized.

### Safety

- [ ] No secrets committed.
- [ ] No passwords/tokens logged.
- [ ] No raw exceptions shown to users.
- [ ] No API calls from Views.

### Testing

- [ ] Repository logic can be unit tested.
- [ ] Controller logic can be unit tested.
- [ ] Important widgets can be widget tested.
- [ ] API failures are covered.

---

# 47. Architectural Philosophy

Use this hierarchy when making decisions:

```text
Simple
   ↓
Reusable
   ↓
Testable
   ↓
Maintainable
   ↓
Scalable
```

Do not start with maximum abstraction.

Start with clear responsibility boundaries.

The best implementation is not the one with the most classes.

The best implementation is the one where each class has one understandable responsibility and the code can be changed without creating unexpected side effects.

---

## Reference Basis

This guide follows the official GetX project's documented state-management/dependency-management patterns and Flutter's current architecture guidance around separation of concerns, repositories, services, dependency injection, testing, and unidirectional data flow.

References:

- GetX state management documentation: https://github.com/jonataslaw/getx/blob/master/documentation/en_US/state_management.md
- GetX repository/documentation: https://github.com/jonataslaw/getx
- Flutter architecture recommendations: https://docs.flutter.dev/app-architecture/recommendations
- Flutter architecture guide: https://docs.flutter.dev/app-architecture/guide
- Flutter architecture concepts: https://docs.flutter.dev/app-architecture/concepts
- Flutter Result/error handling pattern: https://docs.flutter.dev/app-architecture/design-patterns/result

**Important:** When the installed GetX version differs from the version assumed by this guide, inspect the version's official API/documentation before generating code. Do not invent APIs based on memory.
