---
name: android-guidelines
description: Android development guidelines (Jetpack Compose, MVI, Kotlin). Use when developing Android applications or reviewing Android code to ensure adherence to modern Android best practices including: (1) MVI (Model-View-Intent) architecture with unidirectional data flow, (2) Clean Architecture with clear layer separation, (3) Jetpack Compose UI patterns and state management, (4) Dependency injection with Hilt, (5) Kotlin coroutines and Flow for async operations, (6) Navigation component integration, (7) Error handling patterns, (8) Testing strategies. Apply these guidelines when creating new features, refactoring existing code, or when asked about Android architecture decisions.
---

# Android Development Guidelines

## Core Architecture: MVI + Clean Architecture

Follow MVI (Model-View-Intent) pattern with Clean Architecture layers:

**MVI Components:**
- **Model (State)**: Immutable data class representing UI state
- **View (Composable)**: Renders UI based on state, emits user intents
- **Intent**: User actions/events that trigger state changes

**Clean Architecture Layers:**
- **Presentation Layer**: UI (Compose) + ViewModel (State + Intent handling)
- **Domain Layer**: Use cases, business logic, repository interfaces
- **Data Layer**: Repository implementations, data sources (API, Database)

**Unidirectional Data Flow:**
```
User Action → Intent → ViewModel → Use Case → Repository → State Update → UI Render
```

## Project Structure

```
app/
├── data/
│   ├── remote/          # API services, DTOs
│   ├── local/           # Room database, DataStore
│   ├── repository/      # Repository implementations
│   └── mapper/          # Data layer mappers
├── domain/
│   ├── model/           # Business/Domain models
│   ├── repository/      # Repository interfaces
│   └── usecase/         # Business logic (use cases)
├── presentation/
│   ├── screens/         # Screen composables + ViewModels
│   │   └── login/
│   │       ├── LoginScreen.kt
│   │       ├── LoginViewModel.kt
│   │       ├── LoginIntent.kt
│   │       └── LoginState.kt
│   ├── components/      # Reusable UI components
│   ├── theme/           # Theme, colors, typography
│   └── navigation/      # Navigation setup
└── di/                  # Hilt modules
```

## MVI Pattern Implementation

### 1. Define Intent (User Actions)

```kotlin
sealed interface LoginIntent {
    data class EmailChanged(val email: String) : LoginIntent
    data class PasswordChanged(val password: String) : LoginIntent
    object LoginClicked : LoginIntent
    object ErrorDismissed : LoginIntent
}
```

### 2. Define State (UI State)

```kotlin
data class LoginState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null,
    val isLoginSuccess: Boolean = false
)
```

### 3. ViewModel (Intent Handler + State Manager)

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(LoginState())
    val state = _state.asStateFlow()

    fun handleIntent(intent: LoginIntent) {
        when (intent) {
            is LoginIntent.EmailChanged -> {
                _state.update { it.copy(email = intent.email, error = null) }
            }

            is LoginIntent.PasswordChanged -> {
                _state.update { it.copy(password = intent.password, error = null) }
            }

            is LoginIntent.LoginClicked -> {
                login()
            }

            is LoginIntent.ErrorDismissed -> {
                _state.update { it.copy(error = null) }
            }
        }
    }

    private fun login() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }

            loginUseCase(
                email = _state.value.email,
                password = _state.value.password
            ).fold(
                onSuccess = { user ->
                    _state.update {
                        it.copy(
                            isLoading = false,
                            isLoginSuccess = true
                        )
                    }
                },
                onFailure = { error ->
                    _state.update {
                        it.copy(
                            isLoading = false,
                            error = error.message ?: "Unknown error occurred"
                        )
                    }
                }
            )
        }
    }
}
```

### 4. View (Composable)

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateToHome: () -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // Handle navigation side effect
    LaunchedEffect(state.isLoginSuccess) {
        if (state.isLoginSuccess) {
            onNavigateToHome()
        }
    }

    LoginContent(
        state = state,
        onIntent = viewModel::handleIntent
    )
}

@Composable
private fun LoginContent(
    state: LoginState,
    onIntent: (LoginIntent) -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.Center
    ) {
        TextField(
            value = state.email,
            onValueChange = { onIntent(LoginIntent.EmailChanged(it)) },
            label = { Text("Email") },
            enabled = !state.isLoading,
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        TextField(
            value = state.password,
            onValueChange = { onIntent(LoginIntent.PasswordChanged(it)) },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation(),
            enabled = !state.isLoading,
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = { onIntent(LoginIntent.LoginClicked) },
            enabled = !state.isLoading && state.email.isNotEmpty() && state.password.isNotEmpty(),
            modifier = Modifier.fillMaxWidth()
        ) {
            if (state.isLoading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(24.dp),
                    color = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("Login")
            }
        }

        state.error?.let { error ->
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = error,
                color = MaterialTheme.colorScheme.error,
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

**Key MVI Principles:**
- Single immutable state per screen
- All user actions are explicit intents
- State updates only through intent handling
- Unidirectional data flow
- Composables are pure functions of state

## Clean Architecture Layers

### Domain Layer

#### Use Case

```kotlin
class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(
        email: String,
        password: String
    ): Result<User> {
        // Validation logic
        if (email.isBlank()) {
            return Result.failure(Exception("Email cannot be empty"))
        }
        if (password.length < 6) {
            return Result.failure(Exception("Password must be at least 6 characters"))
        }

        // Delegate to repository
        return authRepository.login(email, password)
    }
}
```

#### Repository Interface

```kotlin
interface AuthRepository {
    suspend fun login(email: String, password: String): Result<User>
    suspend fun logout(): Result<Unit>
    fun getCurrentUser(): Flow<User?>
}
```

#### Domain Model

```kotlin
data class User(
    val id: String,
    val email: String,
    val name: String,
    val avatarUrl: String?
)
```

### Data Layer

#### Repository Implementation

```kotlin
class AuthRepositoryImpl @Inject constructor(
    private val authApi: AuthApi,
    private val userDao: UserDao,
    private val authDataStore: AuthDataStore,
    private val userMapper: UserMapper
) : AuthRepository {

    override suspend fun login(
        email: String,
        password: String
    ): Result<User> = withContext(Dispatchers.IO) {
        try {
            // API call
            val response = authApi.login(
                LoginRequestDto(email = email, password = password)
            )

            // Map to domain model
            val user = userMapper.toDomain(response.user)

            // Cache locally
            userDao.insertUser(userMapper.toEntity(user))
            authDataStore.saveAuthToken(response.token)

            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override fun getCurrentUser(): Flow<User?> {
        return userDao.getUserFlow()
            .map { entity -> entity?.let { userMapper.toDomain(it) } }
    }
}
```

#### Data Transfer Object (DTO)

```kotlin
data class LoginRequestDto(
    @SerializedName("email")
    val email: String,
    @SerializedName("password")
    val password: String
)

data class LoginResponseDto(
    @SerializedName("token")
    val token: String,
    @SerializedName("user")
    val user: UserDto
)

data class UserDto(
    @SerializedName("id")
    val id: String,
    @SerializedName("email")
    val email: String,
    @SerializedName("name")
    val name: String,
    @SerializedName("avatar_url")
    val avatarUrl: String?
)
```

#### Mapper

```kotlin
class UserMapper @Inject constructor() {

    fun toDomain(dto: UserDto): User {
        return User(
            id = dto.id,
            email = dto.email,
            name = dto.name,
            avatarUrl = dto.avatarUrl
        )
    }

    fun toEntity(domain: User): UserEntity {
        return UserEntity(
            id = domain.id,
            email = domain.email,
            name = domain.name,
            avatarUrl = domain.avatarUrl
        )
    }

    fun toDomain(entity: UserEntity): User {
        return User(
            id = entity.id,
            email = entity.email,
            name = entity.name,
            avatarUrl = entity.avatarUrl
        )
    }
}
```

## Advanced MVI Patterns

### Side Effects (One-time Events)

For one-time events like navigation or showing snackbars, use a separate effect channel:

```kotlin
sealed interface LoginEffect {
    object NavigateToHome : LoginEffect
    data class ShowSnackbar(val message: String) : LoginEffect
}

@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(LoginState())
    val state = _state.asStateFlow()

    private val _effect = Channel<LoginEffect>(Channel.BUFFERED)
    val effect = _effect.receiveAsFlow()

    private fun login() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }

            loginUseCase(
                email = _state.value.email,
                password = _state.value.password
            ).fold(
                onSuccess = {
                    _state.update { it.copy(isLoading = false) }
                    _effect.send(LoginEffect.NavigateToHome)
                },
                onFailure = { error ->
                    _state.update { it.copy(isLoading = false) }
                    _effect.send(LoginEffect.ShowSnackbar(error.message ?: "Error"))
                }
            )
        }
    }
}

@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateToHome: () -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    // Collect effects
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is LoginEffect.NavigateToHome -> onNavigateToHome()
                is LoginEffect.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(effect.message)
                }
            }
        }
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        LoginContent(
            state = state,
            onIntent = viewModel::handleIntent,
            modifier = Modifier.padding(padding)
        )
    }
}
```

### Complex State Management

For screens with multiple data sources:

```kotlin
data class HomeState(
    val userState: DataState<User> = DataState.Loading,
    val postsState: DataState<List<Post>> = DataState.Loading,
    val isRefreshing: Boolean = false
)

sealed interface DataState<out T> {
    object Loading : DataState<Nothing>
    data class Success<T>(val data: T) : DataState<T>
    data class Error(val message: String) : DataState<Nothing>
}

sealed interface HomeIntent {
    object LoadData : HomeIntent
    object Refresh : HomeIntent
    data class PostClicked(val postId: String) : HomeIntent
}

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val getPostsUseCase: GetPostsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(HomeState())
    val state = _state.asStateFlow()

    fun handleIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.LoadData -> loadData()
            is HomeIntent.Refresh -> refresh()
            is HomeIntent.PostClicked -> handlePostClick(intent.postId)
        }
    }

    private fun loadData() {
        viewModelScope.launch {
            // Load user and posts in parallel
            launch { loadUser() }
            launch { loadPosts() }
        }
    }

    private suspend fun loadUser() {
        _state.update { it.copy(userState = DataState.Loading) }

        getUserUseCase().fold(
            onSuccess = { user ->
                _state.update { it.copy(userState = DataState.Success(user)) }
            },
            onFailure = { error ->
                _state.update {
                    it.copy(userState = DataState.Error(error.message ?: "Error"))
                }
            }
        )
    }

    private suspend fun loadPosts() {
        _state.update { it.copy(postsState = DataState.Loading) }

        getPostsUseCase().fold(
            onSuccess = { posts ->
                _state.update { it.copy(postsState = DataState.Success(posts)) }
            },
            onFailure = { error ->
                _state.update {
                    it.copy(postsState = DataState.Error(error.message ?: "Error"))
                }
            }
        )
    }
}
```

## Dependency Injection (Hilt)

### Application Setup

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

### Modules

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideAuthApi(retrofit: Retrofit): AuthApi {
        return retrofit.create(AuthApi::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindAuthRepository(
        impl: AuthRepositoryImpl
    ): AuthRepository
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context
    ): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}
```

## Jetpack Compose Best Practices

### State Hoisting

```kotlin
@Composable
fun SearchScreen(
    viewModel: SearchViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    SearchContent(
        state = state,
        onIntent = viewModel::handleIntent
    )
}

@Composable
private fun SearchContent(
    state: SearchState,
    onIntent: (SearchIntent) -> Unit
) {
    Column {
        SearchBar(
            query = state.query,
            onQueryChange = { onIntent(SearchIntent.QueryChanged(it)) },
            onSearch = { onIntent(SearchIntent.Search) }
        )

        when (state.results) {
            is DataState.Loading -> LoadingIndicator()
            is DataState.Success -> ResultsList(state.results.data)
            is DataState.Error -> ErrorMessage(state.results.message)
        }
    }
}

@Composable
private fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit
) {
    TextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = Modifier.fillMaxWidth(),
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
        keyboardActions = KeyboardActions(onSearch = { onSearch() })
    )
}
```

### LazyColumn with Pagination

```kotlin
@Composable
fun PostsList(
    posts: List<Post>,
    isLoadingMore: Boolean,
    onLoadMore: () -> Unit,
    onPostClick: (String) -> Unit
) {
    val listState = rememberLazyListState()

    // Detect when user scrolls to bottom
    LaunchedEffect(listState) {
        snapshotFlow { listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index }
            .collect { lastVisibleIndex ->
                if (lastVisibleIndex != null &&
                    lastVisibleIndex >= posts.size - 3 &&
                    !isLoadingMore) {
                    onLoadMore()
                }
            }
    }

    LazyColumn(state = listState) {
        items(
            items = posts,
            key = { it.id }
        ) { post ->
            PostItem(
                post = post,
                onClick = { onPostClick(post.id) }
            )
        }

        if (isLoadingMore) {
            item {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

## Navigation

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Screen.Login.route
    ) {
        composable(Screen.Login.route) {
            LoginScreen(
                onNavigateToHome = {
                    navController.navigate(Screen.Home.route) {
                        popUpTo(Screen.Login.route) { inclusive = true }
                    }
                }
            )
        }

        composable(Screen.Home.route) {
            HomeScreen(
                onNavigateToDetail = { postId ->
                    navController.navigate(Screen.PostDetail.createRoute(postId))
                },
                onLogout = {
                    navController.navigate(Screen.Login.route) {
                        popUpTo(0) { inclusive = true }
                    }
                }
            )
        }

        composable(
            route = Screen.PostDetail.route,
            arguments = listOf(
                navArgument("postId") { type = NavType.StringType }
            )
        ) { backStackEntry ->
            val postId = backStackEntry.arguments?.getString("postId") ?: return@composable
            PostDetailScreen(
                postId = postId,
                onBack = { navController.popBackStack() }
            )
        }
    }
}

sealed class Screen(val route: String) {
    object Login : Screen("login")
    object Home : Screen("home")
    object PostDetail : Screen("post/{postId}") {
        fun createRoute(postId: String) = "post/$postId"
    }
}
```

## Error Handling

```kotlin
sealed class AppError : Exception() {
    data class NetworkError(override val message: String) : AppError()
    data class ApiError(val code: Int, override val message: String) : AppError()
    object NoInternetConnection : AppError()
    object Unauthorized : AppError()
    data class Unknown(override val message: String) : AppError()
}

suspend fun <T> safeApiCall(
    apiCall: suspend () -> T
): Result<T> = withContext(Dispatchers.IO) {
    try {
        Result.success(apiCall())
    } catch (e: HttpException) {
        val error = when (e.code()) {
            401 -> AppError.Unauthorized
            in 400..499 -> AppError.ApiError(e.code(), e.message())
            in 500..599 -> AppError.ApiError(e.code(), "Server error")
            else -> AppError.Unknown(e.message())
        }
        Result.failure(error)
    } catch (e: IOException) {
        Result.failure(AppError.NetworkError(e.message ?: "Network error"))
    } catch (e: Exception) {
        Result.failure(AppError.Unknown(e.message ?: "Unknown error"))
    }
}
```

## Testing

### ViewModel Testing (MVI)

```kotlin
@OptIn(ExperimentalCoroutinesTest::class)
class LoginViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: LoginViewModel
    private lateinit var loginUseCase: LoginUseCase

    @Before
    fun setup() {
        loginUseCase = mockk()
        viewModel = LoginViewModel(loginUseCase)
    }

    @Test
    fun `email changed intent updates state`() = runTest {
        // When
        viewModel.handleIntent(LoginIntent.EmailChanged("test@example.com"))

        // Then
        assertEquals("test@example.com", viewModel.state.value.email)
    }

    @Test
    fun `login success updates state and emits navigation effect`() = runTest {
        // Given
        val user = User(id = "1", email = "test@example.com", name = "Test")
        coEvery { loginUseCase(any(), any()) } returns Result.success(user)

        viewModel.handleIntent(LoginIntent.EmailChanged("test@example.com"))
        viewModel.handleIntent(LoginIntent.PasswordChanged("password123"))

        // When
        viewModel.handleIntent(LoginIntent.LoginClicked)
        advanceUntilIdle()

        // Then
        val state = viewModel.state.value
        assertFalse(state.isLoading)
        assertTrue(state.isLoginSuccess)
        assertNull(state.error)
    }

    @Test
    fun `login failure updates state with error`() = runTest {
        // Given
        coEvery { loginUseCase(any(), any()) } returns
            Result.failure(Exception("Invalid credentials"))

        viewModel.handleIntent(LoginIntent.EmailChanged("test@example.com"))
        viewModel.handleIntent(LoginIntent.PasswordChanged("wrong"))

        // When
        viewModel.handleIntent(LoginIntent.LoginClicked)
        advanceUntilIdle()

        // Then
        val state = viewModel.state.value
        assertFalse(state.isLoading)
        assertFalse(state.isLoginSuccess)
        assertEquals("Invalid credentials", state.error)
    }
}
```

### Use Case Testing

```kotlin
@OptIn(ExperimentalCoroutinesTest::class)
class LoginUseCaseTest {

    private lateinit var useCase: LoginUseCase
    private lateinit var repository: AuthRepository

    @Before
    fun setup() {
        repository = mockk()
        useCase = LoginUseCase(repository)
    }

    @Test
    fun `empty email returns failure`() = runTest {
        // When
        val result = useCase("", "password123")

        // Then
        assertTrue(result.isFailure)
        assertEquals("Email cannot be empty", result.exceptionOrNull()?.message)
    }

    @Test
    fun `short password returns failure`() = runTest {
        // When
        val result = useCase("test@example.com", "123")

        // Then
        assertTrue(result.isFailure)
        assertTrue(result.exceptionOrNull()?.message?.contains("6 characters") == true)
    }

    @Test
    fun `valid credentials calls repository`() = runTest {
        // Given
        val user = User(id = "1", email = "test@example.com", name = "Test")
        coEvery { repository.login(any(), any()) } returns Result.success(user)

        // When
        val result = useCase("test@example.com", "password123")

        // Then
        assertTrue(result.isSuccess)
        assertEquals(user, result.getOrNull())
        coVerify { repository.login("test@example.com", "password123") }
    }
}
```

### Composable Testing

```kotlin
class LoginScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `typing email emits EmailChanged intent`() {
        var lastIntent: LoginIntent? = null

        composeTestRule.setContent {
            LoginContent(
                state = LoginState(),
                onIntent = { lastIntent = it }
            )
        }

        composeTestRule
            .onNodeWithText("Email")
            .performTextInput("test@example.com")

        val intent = lastIntent as? LoginIntent.EmailChanged
        assertNotNull(intent)
        assertEquals("test@example.com", intent?.email)
    }

    @Test
    fun `login button disabled when loading`() {
        composeTestRule.setContent {
            LoginContent(
                state = LoginState(isLoading = true),
                onIntent = {}
            )
        }

        composeTestRule
            .onNodeWithText("Login")
            .assertIsNotEnabled()
    }

    @Test
    fun `error message displayed when error exists`() {
        composeTestRule.setContent {
            LoginContent(
                state = LoginState(error = "Invalid credentials"),
                onIntent = {}
            )
        }

        composeTestRule
            .onNodeWithText("Invalid credentials")
            .assertIsDisplayed()
    }
}
```

## Kotlin Best Practices

### Extension Functions

```kotlin
// String extensions
fun String.isValidEmail(): Boolean {
    return android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()
}

fun String.toUser(): User? {
    return try {
        Gson().fromJson(this, User::class.java)
    } catch (e: Exception) {
        null
    }
}

// Flow extensions
fun <T> Flow<Result<T>>.asDataState(): Flow<DataState<T>> {
    return map { result ->
        result.fold(
            onSuccess = { DataState.Success(it) },
            onFailure = { DataState.Error(it.message ?: "Unknown error") }
        )
    }
}
```

### Coroutines Patterns

```kotlin
// Combine multiple flows
combine(
    userRepository.getUserFlow(),
    settingsRepository.getSettingsFlow(),
    notificationsRepository.getUnreadCount()
) { user, settings, unreadCount ->
    HomeData(user, settings, unreadCount)
}.collectLatest { data ->
    // Update state
}

// Debounce search queries
searchQueryFlow
    .debounce(300)
    .distinctUntilChanged()
    .collectLatest { query ->
        searchProducts(query)
    }

// Retry with exponential backoff
flow {
    emit(api.getData())
}.retry(retries = 3) { cause ->
    (cause is IOException).also { shouldRetry ->
        if (shouldRetry) delay(1000)
    }
}
```

## MVI Architecture Summary

**Key Principles:**
1. **Single State**: One immutable state data class per screen
2. **Explicit Intents**: All user actions are sealed interface members
3. **Unidirectional Flow**: Intent → Process → State → View
4. **State Updates**: Only through intent handler in ViewModel
5. **Side Effects**: Use separate Channel for one-time events
6. **Testability**: Easy to test state transitions and intent handling

**Benefits:**
- Predictable state management
- Easy to debug (log intents and states)
- Time-travel debugging possible
- Clear separation of concerns
- Scalable for complex UIs

## Quick Checklist

- [ ] Using MVI with Intent, State, ViewModel pattern
- [ ] Clean Architecture layer separation (Presentation, Domain, Data)
- [ ] Single immutable state per screen (StateFlow)
- [ ] All user actions as explicit intents (sealed interface)
- [ ] Use cases for business logic
- [ ] Repository pattern with interfaces in domain layer
- [ ] Hilt for dependency injection
- [ ] Proper error handling with Result<T> or sealed classes
- [ ] Coroutines with appropriate dispatchers
- [ ] Mappers between layers (DTO ↔ Domain ↔ Entity)
- [ ] Compose best practices (state hoisting, lifecycle awareness)
- [ ] Navigation component for screen transitions
- [ ] Unit tests for ViewModels, Use Cases, and Repositories
- [ ] UI tests for Composables
