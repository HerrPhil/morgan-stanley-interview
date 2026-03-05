# morgan-stanley-interview
My retrospective of the Morgan Stanley technical interview.



Morgan Stanley Interview

They expect the candidate to talk in code.

Effectively, they will ask how to get data from a network call.
When there are screen changes configuration, when does the app get next network call happen?

They want to say what dependencies to add.

dependencies {
    implementation "com.squareup.retrofit2:retrofit:2.9.0"
    implementation "com.squareup.retrofit2:converter-gson:2.9.0"
    implementation "com.google.dagger:hilt-android:2.38.1"
    kapt "com.google.dagger:hilt-compiler:2.38.1"
}   

They want to describe what an api interface looks like.

interface ApiService {
    @GET("items")
    suspend fun getItems(): Response<List<Item>>
}

They want to see the lines of code to register an api service.

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideApiService(): ApiService {
        val retrofit = Retrofit.Builder()
            .baseUrl("https://example.com/api/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        return retrofit.create(ApiService::class.java)
    }
}

They want to see line-by-line the code to
- inject the service in your view model
- the uistate representation of endpoint loading
- channel to fetch items.
- a fetch items function to transform api response state to ui state, as well as convert response exceptions into ui state.
- use viewModelScope to launch the back-end call
- get the data as LaunchedEffecct to trigger the intent to delegate to the fetch items function.

@HiltViewModel
class MainViewModel @Inject constructor(
    private val apiService: ApiService
) : ViewModel() {

    sealed class UserIntent {
        object FetchItems : UserIntent()
    }

    sealed class UiState {
        object Loading : UiState()
        data class Success(val data: List<Item>) : UiState()
        data class Error(val exception: Exception) : UiState()
    }

    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state: StateFlow<UiState> = _state

    private val userIntents = Channel<UserIntent>(Channel.UNLIMITED)

    init {
        handleIntents()
    }

    private fun handleIntents() {
        viewModelScope.launch {
            userIntents.consumeAsFlow().collect { intent ->
                when (intent) {
                    is UserIntent.FetchItems -> fetchItems()
                }
            }
        }
    }

    fun fetchItems() {
        viewModelScope.launch {
            _state.value = UiState.Loading
            try {
                val response = apiService.getItems()
                if (response.isSuccessful) {
                    _state.value = UiState.Success(response.body() ?: emptyList())
                } else {
                    _state.value = UiState.Error(Exception("HTTP ${response.code()}"))
                }
            } catch (e: Exception) {
                _state.value = UiState.Error(e)
            }
        }
    }

    fun onIntentReceived(intent: UserIntent) {
        viewModelScope.launch {
            userIntents.send(intent)
        }
    }
}   

@HiltViewModel
class MainViewModel @Inject constructor(
    private val apiService: ApiService
) : ViewModel() {

    sealed class UserIntent {
        object FetchItems : UserIntent()
    }

    sealed class UiState {
        object Loading : UiState()
        data class Success(val data: List<Item>) : UiState()
        data class Error(val exception: Exception) : UiState()
    }

    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state: StateFlow<UiState> = _state

    private val userIntents = Channel<UserIntent>(Channel.UNLIMITED)

    init {
        handleIntents()
    }

    private fun handleIntents() {
        viewModelScope.launch {
            userIntents.consumeAsFlow().collect { intent ->
                when (intent) {
                    is UserIntent.FetchItems -> fetchItems()
                }
            }
        }
    }

    fun fetchItems() {
        viewModelScope.launch {
            _state.value = UiState.Loading
            try {
                val response = apiService.getItems()
                if (response.isSuccessful) {
                    _state.value = UiState.Success(response.body() ?: emptyList())
                } else {
                    _state.value = UiState.Error(Exception("HTTP ${response.code()}"))
                }
            } catch (e: Exception) {
                _state.value = UiState.Error(e)
            }
        }
    }

    fun onIntentReceived(intent: UserIntent) {
        viewModelScope.launch {
            userIntents.send(intent)
        }
    }
}

@Composable
fun ItemsScreen() {
    val viewModel: MainViewModel = hiltViewModel()
    val state by viewModel.state.collectAsState()

    LaunchedEffect(Unit) {
        viewModel.onIntentReceived(MainViewModel.UserIntent.FetchItems)
    }

    when (state) {
        is MainViewModel.UiState.Loading -> {
            Text("Loading...")
        }
        is MainViewModel.UiState.Success -> {
            LazyColumn {
                items(state.data) { item ->
                    Text(text = item.name)
                }
            }
        }
        is MainViewModel.UiState.Error -> {
            Text(text = "Error: ${state.exception.message}")
        }
    }
}



Technically, this is valid.

However, this does not fit my learning style.
I know what concepts are in play.
Then I seek out the specific constructs and best practices to implement this.
Then I write code like this.

As well, their interview style is the dungeons and dragons style.
If you don't know the secret password, then the master does not mention the next keyword or key topic to trigger a memory.
This style never works for me. I only function well in mature, adult conversations of how to solve problems, not what memorized solution is sought.

Effectively, they only want developers with work experience.
I get it.
After two years at my past role, neuroplasticity of the brain created memories from doing the same thing day-in and day-out.
Basically, Morgan Stanley is a past neuroplasticity harvester.
Get two years of workplace coding elsewhere, then, maybe, you have a shot.
