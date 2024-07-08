# Side-effects

[Side-effects in Compose  |  Jetpack Compose  |  Android Developers](https://developer.android.com/jetpack/compose/side-effects)

**Side-effect**란, `Composable` 함수의 scope 밖에서 일어나는 앱 상태의 변화를 의미한다. `Composable` 은 자체적으로 가지고 있는 생명주기가 있고 언제든지 recomposition이 발생할 수 있으며 발생하는 순서를 예측하기 어려운 상황도 있기 때문에, `Composable` 은 [이상적으로는 Side-effect로부터 자유로워야 한다](https://developer.android.com/jetpack/compose/mental-model).

그러나 가끔은 **Side-effect**가 필요하기는 하다. 예를 들어 스낵 바를 표시하거나, 특정 상태 조건에 따라 다른 화면으로 이동하는 등의 일회성 이벤트를 트리거해야 할 때가 있다. 이럴 때는 `Composable` 이 생명주기를 알 수 있는 환경에서 호출되어야 한다. 이 페이지에서는 Jetpack Compose가 제공하는 다양한 **Side-effect** API를 소개한다.

## State and effect use cases

[Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model) 문서에서 소개했듯이, `Composable` 은 **Side-effect**로부터 자유로워야 한다. [해당 문서](https://developer.android.com/jetpack/compose/state)에서 설명하고 있는 앱의 상태를 변경해야 할 때, Effect API를 사용하여 **Side-effect**가 의도한 방향으로 동작할 수 있도록 해야 한다.

<aside>
💬 **Effect**란, UI를 방출하지 않고 Composition이 완료된 후에 Side-effect가 호출될 수 있도록 해주는 Composable 함수이다.

</aside>

**Effect** API가 제공하는 다양한 가능성들로 인해서 API들이 남용되기 쉽다. [해당 문서](https://developer.android.com/jetpack/compose/state)에서 소개하고 있는 것과 같이 앱의 상태 변경이 필요한 경우, **Effect**가 수행하는 작업들이 UI와 관련이 있으면서, `단방향 데이터 흐름(UDF)` 을 깨뜨리지 않도록 주의해야 한다.

<aside>
⭐ **참고** : 반응형 UI는 본질적으로 비동기적이다. Jetpack Compose는 이를 callback 대신 API 수준에서 코루틴을 사용하여 해결한다.

</aside>

### LaunchedEffect: Composable scope 내에서 실행하는 중단 함수(Suspend function)

중단 함수(Suspend function)를 `Composable` 안에서 안전하게 호출하려면 **LaunchedEffect** Composable을 사용해야 한다. Composable이 Composition 단계에 진입하면, **LaunchedEffect**는 매개 변수로 전달된 코루틴 코드 블록을 시작한다. LaunchedEffect가 Composition을 떠나면 코루틴은 취소된다. 만약 **LaunchedEffect**가 다른 key를 가지고 Recomposition이 발생하면, 기존의 코루틴을 취소되고 새로운 중단 함수가 새로운 코루틴 안에서 시작된다.

예를 들어, Scaffold에서 SnackBar를 표시하는 경우, 중단 함수인 `SnackbarHostState.showSnackbar()` 를 호출한다.

```kotlin
@Composable
fun MyScreen(
    state: UiState<List<Movie>>,
    snackbarHostState: SnackbarHostState
) {

    // If the UI state contains an error, show snackbar
    if (state.hasError) {

        // `LaunchedEffect` will cancel and re-launch if
        // `scaffoldState.snackbarHostState` changes
        LaunchedEffect(snackbarHostState) {
            // Show snackbar using a coroutine, when the coroutine is cancelled the
            // snackbar will automatically dismiss. This coroutine will cancel whenever
            // `state.hasError` is false, and only start when `state.hasError` is true
            // (due to the above if-check), or if `scaffoldState.snackbarHostState` changes.
            snackbarHostState.showSnackbar(
                message = "Error message",
                actionLabel = "Retry message"
            )
        }
    }

    Scaffold(
        snackbarHost = {
            SnackbarHost(hostState = snackbarHostState)
        }
    ) { contentPadding ->
        // ...
    }
}
```

위 코드에서는 상태에 에러가 포함되어 있을 경우 코루틴이 트리거되고, 오류가 없을 때는 코루틴이 취소될 것이다. **LaunchedEffect**의 호출 위치가 if문 안에 있을 경우, 조건문이 false일 때 **LaunchedEffect**가 Composition 단계에 있다면 **LaunchedEffect**가 제거되면서 이미 시작됐던 코루틴도 함께 취소된다.

### rememberCoroutineScope: Composable 외부에서 코루틴을 실행하기 위한 Composable의 생명주기를 따라가는 Scope를 만든다

LaunchedEffect는 Composable 함수로써, 다른 Composable 함수 내에서만 사용할 수 있다. Composable의 scope를 가지지만 Composable의 바깥에서 코루틴을 실행하면서 Composition을 떠날 때 자동으로 코루틴이 종료되기를 희망한다면, **rememberCoroutineScope**를 사용하라. 또한 사용자 이벤트가 발생했을 때 애니메이션을 취소하는 등, 하나 이상의 코루틴 생명주기를 다루어야 할 때도 사용할 수 있다.

**rememberCoroutineScope**는 Composable 함수로, Composition이 발생한 시점에 바인딩된 `CoroutineScope` 를 반환한다. 이 Scope는 Composition을 떠날 때 취소된다.

위의 예제에 이어, 유저가 Button을 탭했을 때 SnackBar를 보여주는 코드를 다음과 같이 작성할 수 있다.

```kotlin
@Composable
fun MoviesScreen(snackbarHostState: SnackbarHostState) {

    // Creates a CoroutineScope bound to the MoviesScreen's lifecycle
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = {
            SnackbarHost(hostState = snackbarHostState)
        }
    ) { contentPadding ->
        Column(Modifier.padding(contentPadding)) {
            Button(
                onClick = {
                    // Create a new coroutine in the event handler to show a snackbar
                    scope.launch {
                        snackbarHostState.showSnackbar("Something happened!")
                    }
                }
            ) {
                Text("Press me")
            }
        }
    }
}
```

### rememberUpdateState: 값이 변경되어도 다시 시작되지 않는 Effect를 참조한다

`LaunchedEffect` 는 매개변수의 키 값이 변경될 때 마다 재시작된다. 하지만 어떤 상황에서는 Effect가 참조하고 있는 값이 변경되더라도 다시 Effect를 시작하고 싶지 않을 수 있다. 이럴 때 **rememberUpdatedState**를 사용한다. 비용이 많이 들거나, 다시 생성하고 다시 시작하기 어려운 장기간 지속되어야 하는 작업을 포함하고 있는 Effect에 유용하다.

예를 들어, 앱에 일정 시간 후에 사라지는 LandingScreen이 있다고 가정하자. LandingScreen이 recompose 되더라도, Effect는 재시작되지 않고 일정 시간이 지난 후에 시간이 지났음을 알린다.

```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {

    // This will always refer to the latest onTimeout function that
    // LandingScreen was recomposed with
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    // Create an effect that matches the lifecycle of LandingScreen.
    // If LandingScreen recomposes, the delay shouldn't start again.
    LaunchedEffect(true) {
        delay(SplashWaitTimeMillis)
        currentOnTimeout()
    }

    /* Landing screen content */
}
```

호출 지점의 생명주기와 일치하는 Effect를 생성하려면 `Unit` 이나 `true` 와 같이 절대 변하지 않는 상수를 매개변수로 사용하면 된다. 위의 코드에서는 `LaunchedEffect(true)` 가 사용되었다. LandingScreen이 recompose 될 때마다 항상 최신의 onTimeout 람다 값을 가지도록 하려면, onTimeout을 rememberUpdatedState 함수로 감싸야 한다. 위 코드에서 State를 반환하는 `currentOnTimeout` 는 Effect 안에서 사용되어야 한다.

<aside>
⚠️ 주의 : `LaunchedEffect(true)` 는 `while(true)` 과 같이 코드를 사용함에 있어 주의를 기울여야 한다. 이것을 사용하기에 적합한 케이스라 하더라도 정말 사용이 필요할지 다시 한 번 생각해보자.

</aside>

### DisposableEffect: Dispose(해지)가 가능한 Effect

Key가 변경되거나 Composition을 벗어날 때, 더 이상 특정 코드가 호출되지 않도록 사용을 해지하고 새로운 Effect를 정의하고자 한다면 **DisposableEffect**를 사용하자. **DisposableEffect**의 키가 변경될 때, dispose할 현재의 Effect와 새로운 Effect를 정의하면 된다.

예를 들어 Jetpack Compose에서 `LifecycleObserver` 를 활용하여 Lifecycle 이벤트를 기반으로 Analytics 이벤트를 보내야 할 경우, DisposableEffect를 활용하여 Observer를 구독하고 해지할 수 있다.

```kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit, // Send the 'started' analytics event
    onStop: () -> Unit // Send the 'stopped' analytics event
) {
    // Safely update the current lambdas when a new one is provided
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    // If `lifecycleOwner` changes, dispose and reset the effect
    DisposableEffect(lifecycleOwner) {
        // Create an observer that triggers our remembered callbacks
        // for sending analytics events
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_START) {
                currentOnStart()
            } else if (event == Lifecycle.Event.ON_STOP) {
                currentOnStop()
            }
        }

        // Add the observer to the lifecycle
        lifecycleOwner.lifecycle.addObserver(observer)

        // When the effect leaves the Composition, remove the observer
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }

    /* Home screen content */
}
```

위의 코드에서는 `lifecycleOwner` 에 `observer` 를 추가하고 있다. 만약 lifecycleOwner가 변경된다면, Effect는 dispose되고 새로운 lifecycleOwner와 함께 Effect가 리셋될 것이다.

**DisposableEffect**는 반드시 코드 블락 최하단에 `onDispose {…}` 를 포함해야 한다. 그렇지 않으면 IDE에서 빌드 타임 에러가 발생한다.

<aside>
⭐ **참고** : `onDispose` 안에 아무것도 정의하지 않는 것은 좋은 경험이 아니다. **DisposableEffect**을 사용하는 것보다 더 적합한 Effect API가 있는지 다시 생각해보자.

</aside>

### SideEffect: Compose가 아닌 코드에 Compose State를 사용한다

Compose에 의해 관리되지 않는 객체에 Compose의 상태를 공유하고자 한다면, **SideEffect**를 사용하라. **SideEffect**는 Recomposition이 일어날 때 마다 계속 호출될 것이다.

Analytics 라이브러리로 매 이벤트마다 유저 프로퍼티를 추가하여 유저 통계를 관찰한다고 가정해보자. 현재 유저의 프로퍼티를 라이브러리에 전달하려면, **SideEffect**를 사용하여 그 값을 전달하면 된다.

```kotlin
@Composable
fun rememberFirebaseAnalytics(user: User): FirebaseAnalytics {
    val analytics: FirebaseAnalytics = remember {
        FirebaseAnalytics()
    }

    // On every successful composition, update FirebaseAnalytics with
    // the userType from the current User, ensuring that future analytics
    // events have this metadata attached
    SideEffect {
        analytics.setUserProperty("userType", user.userType)
    }
    return analytics
}
```

### produceState: Compose state가 아닌 값을 Compose state로 변환한다

**produceState**는 Composition 단계 범위 내에서 실행되는 코루틴을 발행하고, 해당 코루틴이 반환하는 값을 State로 변환함으로써 Compose state가 아닌 값을 Compose state로 변환해주는 역할을 한다. `Flow` , `LiveData` 또는 `RxJava` 와 같은 외부 구독 기반의 상태를 Composition 단계로 가져올 때 사용할 수 있다.

**produceState**가 Composition 단계에 진입할 때 producer를 발행하고, Composition을 벗어날 때 취소된다. 반환된 State의 값이 변경되지 않는다면 Recomposition이 일어나지 않는다.

produceState가 코루틴을 발행하지만, 중단 함수를 사용하지 않는 datasource들을 구독할 때도 사용할 수 있다. 해당 구독을 제거하려면 `awaitDispose` 를 사용하면 된다.

아래 예시는 네트워크를 통해 이미지를 로드할 때 어떻게 **produceState**를 사용할 수 있는지 그 예시를 보여준다. `loadNetworkImage` Composable 함수는 다른 Composable에서도 사용할 수 있는 State를 반환한다.

```kotlin
@Composable
fun loadNetworkImage(
    url: String,
    imageRepository: ImageRepository = ImageRepository()
): State<Result<Image>> {

    // Creates a State<T> with Result.Loading as initial value
    // If either `url` or `imageRepository` changes, the running producer
    // will cancel and will be re-launched with the new inputs.
    return produceState<Result<Image>>(initialValue = Result.Loading, url, imageRepository) {

        // In a coroutine, can make suspend calls
        val image = imageRepository.load(url)

        // Update State with either an Error or Success result.
        // This will trigger a recomposition where this State is read
        value = if (image == null) {
            Result.Error
        } else {
            Result.Success(image)
        }
    }
}
```

<aside>
⭐ **참고** : 리턴 값이 있는 Composable 함수의 네이밍은 소문자로 시작하는 일반적인 코틀린 함수를 선언하듯이 작성해야 한다.

</aside>

<aside>
💡 **핵심 포인트** : **produceState**는 내부적으로 또 다른 Effect를 사용한다! `remember { mutableStateOf(initivalValue) }` 를 사용하여 Result를 가지고 있으며, LaunchedEffect 안의 producer 블록을 트리거한다. producer 블록 안의 값이 변경될 때 마다, Result State는 새로운 값으로 변경된다.

**produceState**처럼 이미 존재하는 API를 사용하여 쉽게 자신만의 Effect를 만들 수 있다.

</aside>

### derivedStateOf: 한개 이상의 State를 또 다른 State로 변환한다

Compose에서 구독 중인 State나 Composable input이 변경될 때 Recomposition이 매번 발생한다. State 혹은 input은 정말로 UI를 업데이트하는데 필요한 횟수보다 더 자주 변경될 수 있으며 이로 인하여 불필요한 Recomposition이 발생할 수 있다.

위와 같이 Recomposition이 필요한 상황보다 Composable input이 더 자주 변경된다면 **derivedStateOf**를 사용해야 한다. 스크롤 포지션과 같이 값의 변경이 자주 일어나지만, 사전에 정의된 threshold를 넘겼을 때만 UI를 업데이트하는 등의 반응이 필요할 때 사용할 수 있다. derivedStateOf는 필요한 만큼의 State만 사용하여 새로운 State를 만들 수 있다. 이 방식은 Kotlin Flow의 `distinctUntilChanged()` 연산자와 동작이 유사하다.

<aside>
⚠️ 주의 : derivedStateOf는 cost가 높은 연산자이기 때문에, 정말 state가 변하지 않아 불필요한 recomposition을 피해야 할 때만 사용해야 한다.

</aside>

**올바른 사용법**

아래의 코드는 **derivedStateOf**를 사용하기에 적절한 케이스를 보여준다.

```kotlin
@Composable
// When the messages parameter changes, the MessageList
// composable recomposes. derivedStateOf does not
// affect this recomposition.
fun MessageList(messages: List<Message>) {
    Box {
        val listState = rememberLazyListState()

        LazyColumn(state = listState) {
            // ...
        }

        // Show the button if the first visible item is past
        // the first item. We use a remembered derived state to
        // minimize unnecessary compositions
        val showButton by remember {
            derivedStateOf {
                listState.firstVisibleItemIndex > 0
            }
        }

        AnimatedVisibility(visible = showButton) {
            ScrollToTopButton()
        }
    }
}
```

`firstVisibleItemIndex` 는 첫 번째로 보이는 item의 변경이 일어날 때 마다 언제든 값이 변경될 수 있다. 스크롤을 하면 0, 1, 2, 3, 4, 5와 같이 변경되지만, recomposition은 값이 0보다 클 때만 발생해야 한다. 스크롤 포지션과 같이 값이 변경되는 빈도 수와 recomposition이 일어나야 하는 빈도 수가 맞지 않는 위와 같은 상황에서 **derivedStateOf**를 사용하기에 아주 적절하다.

**올바르지 못한 사용법**

흔한 실수 중 하나는, 두 State를 조합하여 새로운 State를 만들기 위해 **derivedStateOf**를 사용하는 것이다. 그러나 다음과 같은 사용법은 순전히 오버헤드를 발생시킬 뿐이지 올바른 사용법이 아니다.

<aside>
⚠️ 아래의 코드는 derivedStateOf의 올바르지 못한 사용법을 나타낸 것으로 이렇게 사용해서는 안된다.

</aside>

```kotlin
// DO NOT USE. Incorrect usage of derivedStateOf.
var firstName by remember { mutableStateOf("") }
var lastName by remember { mutableStateOf("") }

val fullNameBad by remember { derivedStateOf { "$firstName $lastName" } } // This is bad!!!
val fullNameCorrect = "$firstName $lastName" // This is correct
```

`fullName` 은 그저 `firstName` 과 `lastName` 이 변경될 때 마다 값의 업데이트가 필요할 뿐 recomposition을 발생시킬 이유가 없으므로 **derivedStateOf**의 사용이 필요하지는 않다.

### snapshotFlow: Compose 상태를 Flow로 변환한다

`State<T>` 를 cold Flow로 변환하고 싶다면 **snapshotFlow**를 사용하자. **snapshotFlow**의 코드 블록은 collect될 때 실행되고, 그 안에서 읽은 State 객체의 결과를 발행한다. **snapshotFlow**의 코드 블록 내에서 읽은 State 객체들 중 하나가 변경되었을 때 새 값이 이전에 발행된 값과 다르다면, Flow는 새 값을 구독자에게 발행한다. (이 동작은 `Flow.distinctUntilChanged` 와 유사하다.)

다음 예제는 사용자가 List의 첫 번째 아이템에서 스크롤했을 때마다 Analytics 이벤트를 전송하는 Side-Effect를 보여준다.

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```

`listState.firstVisibleItemIndex` 가 Flow로 변환되어 Flow API에서 제공하는 연산자를 활용하는 모습을 볼 수 있다.

## Restarting effects

`LaunchedEffect` , `produceState` , `DisposableEffect` 와 같은 Effect API는 매개변수로 여러 개의 key를 가질 수 있고 이는 실행 중인 Effect를 취소하고 새로운 key 값과 함께 새로운 Effect를 실행할 때 사용된다.

key는 전형적으로 아래와 같은 형태로 구성되어 있다.

```kotlin
EffectName(restartIfThisKeyChanges, orThisKey, orThisKey, ...) { block }
```

위의 동작과 같은 세세한 부분으로 인해, Effect를 다시 시작하기 위해 사용된 매개변수가 올바르게 사용되지 않았다면 문제가 발생할 수 있다.

- 의도한 것보다 적게 Effect를 재시작하면 버그가 발생할 수 있다.
- 의도한 것보다 많이 Effect를 재시작하면 비효율적일 수 있다.

일반적으로 Effect를 사용하는 코드 블록의 매개변수로 가변/불변(mutable/immutable) 변수들이 추가되어야 한다. 이 외에도, Effect 재시작을 위해 하나만이 아닌 더 많은 파라미터를 추가할 수 있다. 변수의 값 변경으로 인해 Effect의 재시작을 막아야 하는 경우, 변수를 `rememberUpdatedState` 로 감싸야 한다. key가 없는 remember로 래핑되어 변경이 일어나지 않을 경우, Effect의 key로 전달할 필요가 없다.

<aside>
💡 핵심 포인트 : Effect에 사용되는 변수들은 Effect의 파라미터로 추가하거나 `rememberUpdatedState` 를 사용해야 한다.

</aside>

아래 코드에서 `DisposableEffect` 를 보면, 값의 변경에 따른 Effect의 재시작을 막기 위해서 lifecycleOwner를 key로 취하고 있는 것을 볼 수 있다.

```kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit, // Send the 'started' analytics event
    onStop: () -> Unit // Send the 'stopped' analytics event
) {
    // These values never change in Composition
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            /* ... */
        }

        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

`currentOnStart` 와 `currentOnStop` 은 `rememberUpdatedState` 로 인하여 Composition 단계에 값의 변경이 일어나지 않으므로 `DisposableEffect` 의 key로 사용될 필요가 없다. `DisposableEffect` 의 파라미터로 lifecycleOwner를 추가하지 않고 값의 변경이 일어나게 된다면 `HomeScreen` 은 Recomposition이 발생하고, `DisposableEffect` 의 안의 코드 블록이 dispose되지 않은 채로 재시작된다. 이는 잘못된 lifecyleOwner를 레퍼런스하고 있을 것이기 때문에 문제를 일으킬 것이다.

### Constants as keys

`true` 와 같은 상수를 Effect key로 사용하여 Effect를 호출하는 곳의 생명주기를 따라가도록 만들 수 있다. 이 방법을 적용하기 전에, 정말 필요한 방법인지 다시 한 번 생각해보고 사용하도록 하자.