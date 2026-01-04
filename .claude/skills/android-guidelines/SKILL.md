---
name: android-guidelines
description: Android development guidelines (Jetpack Compose, MVVM, Kotlin).
allowed-tools: Read, Edit
---

# Android Guidelines

## Jetpack Compose
```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    Column(modifier = Modifier.padding(16.dp)) {
        TextField(
            value = uiState.email,
            onValueChange = { viewModel.updateEmail(it) }
        )
        Button(onClick = { viewModel.login() }) {
            Text("Login")
        }
    }
}
```

## MVVM Pattern
```kotlin
class LoginViewModel(
    private val repository: AuthRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState = _uiState.asStateFlow()
    
    fun login() {
        viewModelScope.launch {
            _uiState.update { it.copy(loading = true) }
            try {
                repository.login(email, password)
                _uiState.update { it.copy(success = true) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message) }
            }
        }
    }
}
```

## Data Classes
```kotlin
data class User(
    @SerializedName("user_id")
    val userId: Int,
    @SerializedName("email")
    val email: String
)
```
