# Jetpack Compose phases

[Jetpack Compose phases  |  Android Developers](https://developer.android.com/develop/ui/compose/phases)

다른 대부분의 UI toolkit처럼, Compose도 몇 가지 단계를 거쳐 프레임 단위로 렌더링된다. Android View System을 살펴보면 measure → layout → drawing 이 세 단계로 이루어져 있는데, Compose도 매우 유사하지만 시작 단계에 **Composition**이라는 아주 중요한 단계가 하나 추가된다.

Composition은 [Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model), [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state) 와 같은 컴포즈 문서 곳곳에 잘 설명되어 있다.

## The three phases of a frame

Compose에는 세 가지 단계가 있다.

1. **Composition** : 어떤 UI를 보여줄지 결정한다. (**What** UI to show). Compose는 composable 함수들을 실행하고 상세 UI를 구성한다.
2. **Layout** : 어디에 UI를 놓을 것인지 결정한다. (**Where** to place UI). 해당 단계는 measurement와 placement 두 단계로 구성되어 있다. 자식 계층을 포함한 자기 스스로 길이를 측정하고 2D 좌표 위 어느 곳에 배치할지 결정한다. 이 단계는 Layout 트리의 각 노드에 대해 수행된다.
3. **Drawing** : 어떻게 UI를 그릴지 결정한다. UI는 일반적으로 Canvas에 의해 그려진다.

![사진 1. Compose가 데이터를 UI로 변환하는 세 가지 단계](https://developer.android.com/static/develop/ui/compose/images/compose-phases.png)

사진 1. Compose가 데이터를 UI로 변환하는 세 가지 단계

일반적으로는 위 세 단계의 순서를 따른다. 데이터는 Composition 부터 Layout, Drawing 단계까지 단방향 데이터 흐름으로 이동하여 프레임을 생성한다. 예외적으로 `BoxWithConstraints` , `LazyColumn` , `LazyRow` 는 부모의 Layout 단계에 따라 자식의 Composition 단계의 순서가 달라진다.

이러한 세 단계가 거의 모든 프레임에서 발생한다고 생각하면 되지만, 성능 상의 이유로 Compose는 동일한 Input으로 동일한 결과가 계산되는 작업을 반복적으로 작업하지 않기 위해 노력한다. Compose는 이전 결과를 재사용할 수 있는 경우 Composable 함수를 실행하지 않고, 필요하지 않다면 Layout과 Drawing 단계를 다시 반복하지 않는다. Compose는 UI를 업데이트하는 데 필요한 최소한의 작업만 수행한다. 이와 같은 최적화가 가능한 이유는 Compose가 단계마다 상태를 추적하기 때문이다.

### Understand the phases

**Composition**

Composition 단계에서 Compose는 런타임에 Composable 함수를 실행하고 UI 트리를 도출한다. 이 UI 트리는 layout 노드들로 구성되어 있고, 이 노드들은 다음 단계를 위해 모든 정보를 담고 있다.

[영상 2. Composition 단계에서 생성하는 UI 트리](https://developer.android.com/static/develop/ui/compose/images/composition.mp4)

영상 2. Composition 단계에서 생성하는 UI 트리

위 트리 중 일부분만 상세히 그린다면 다음과 같을 것이다.

![사진 3. 코드에 해당하는 UI 트리의 하위 섹션](https://developer.android.com/static/develop/ui/compose/images/code-subsection.png)

사진 3. 코드에 해당하는 UI 트리의 하위 섹션

위 예제에서는 Composable 함수들이 하나의 단일 노드로 맵핑되어 있는 것을 볼 수 있다. 더 복잡한 예제에서는 Composable이 플로우를 조작하고 상태에 따른 또 다른 트리를 생산하는 등의 복잡한 로직이 포함될 것이다.

**Layout**

Layout 단계에서 Compose는 Composition 단계에서 생산한 UI 트리를 인풋으로 사용한다. Layout 노드에는 각 노드의 크기와 2D 공간에서의 위치를 결정하는 데 필요한 모든 정보가 포함되어 있다.

[영상 4. Layout 단계에서 각 노드의 measurement & placement](https://developer.android.com/static/develop/ui/compose/images/layout.mp4)

영상 4. Layout 단계에서 각 노드의 measurement & placement

Layout 단계가 진행되는 동안, 세 단계에 걸친 알고리즘을 통해 트리가 순회된다.

1. **Measure children** : 자식 노드가 있다면 자식 노드를 측정한다.
2. **Decide own size** : 1의 결과를 토대로 부모 노드의 크기를 결정한다.
3. **Place children** : 각 자식 노드는 부모 노드의 위치에 따라 배치된다.

Layout 단계가 마무리됐을 때, 각 노드는 다음 정보를 알게 된다.

- 지정된 width(너비) & height(높이)
- 레이아웃이 그려질 x, y 좌표

![https://developer.android.com/static/develop/ui/compose/images/code-subsection.png](https://developer.android.com/static/develop/ui/compose/images/code-subsection.png)

위 트리에서 다음과 같은 순서로 알고리즘이 동작한다.

1. `Row { … }` 는 자식 Layout인 `Image` 와 `Column` 을 측정한다.
2. `Image` 을 측정한다. `Image` 는 자식 레이아웃이 없으므로, 자기 자신의 크기를 결정하고 이는 `Row` 돌아온 뒤 보고된다.
3. `Column` 을 측정한다. 두 개의 자식 `Text` Composable이 있으므로 먼저 자식을 측정한다.
4. 첫 번째 `Text` 를 측정한다. 자식 레이아웃이 없으므로, 자기 자신의 크기를 결정하고 `Column` 으로 돌아온 뒤 보고된다.
    1. 두 번째 `Text` 를 측정한다. 자식 레이아웃이 없으므로 자기 자신의 크기를 결정하고 `Column` 으로 돌아온 뒤 보고된다.
5. `Column` 은 자식 레이아웃들이 측정한 값을 사용하여 자기 자신의 크기를 결정하고, 자식 레이아웃의 최대 너비와 높이의 합을 구한다.
6. `Column` 은 자식 레이아웃들을 수직으로 하나씩 배치시킨다.
7. `Row` 는 자식 레이아웃들이 측정한 값을 사용하여 자기 자신의 크기를 결정하고 자식 레이아웃의 최대 너비와 높이의 합을 구한다. 그 후, 자식 레이아웃들을 배치시킨다.

각 노드는 오직 한 번씩만 방문되었음을 참고해두자. Compose 런타임은 성능 개선을 위해 모든 노드를 측정하고 배치할 때 UI 트리를 한 번만 순회한다. 트리의 노드가 증가할 때 마다, 트리를 순회하는 시간이 선형적으로 증가한다. 반대로 각 노드가 여러 번 방문한 경우, 순회 시간이 기하급수적으로 증가한다.

**Drawing**

Drawing 단계에서 UI 트리의 상위 노드부터 하위 노드까지 다시 순회가 이루어지고 각 노드가 화면에 자기 자신의 레이아웃을 그린다.

[영상 5. Drawing 단계에서 픽셀을 화면 위에 나타내는 과정](https://developer.android.com/static/develop/ui/compose/images/drawing.mp4)

영상 5. Drawing 단계에서 픽셀을 화면 위에 나타내는 과정

위의 예제를 가지고 이어서 이야기하면, 트리의 내용은 다음과 같은 순서로 그려진다.

1. `Row` 는 배경 색깔을 포함한 내용들을 그린다.
2. `Image` 가 그려진다.
3. `Column` 이 그려진다.
4. 두 `Text` 가 각각 그려진다.

[영상 6. UI 트리와 나타내어진 UI](https://developer.android.com/static/develop/ui/compose/images/drawing-ui-tree.mp4)

영상 6. UI 트리와 나타내어진 UI

## State reads

위에서 언급한 대로 Compose는 세 가지 주요 단계가 있고 각 단계 내에서 어떤 State를 구독하는 지 추적한다. State의 변경을 추적함에 따라 UI 요소의 상태가 변경되었을 때, 해당 요소에 영향을 미치는 작업을 수행할 필요가 있는 단계에만 알려주게 된다.

<aside>
⭐ 참고 : State의 인스턴스가 생성되고 보관되는 곳 보다는 언제 어디에서 State 값을 **참조(read)**하는지가 중요하다.

</aside>

State 값이 참조될 때 각 단계 별로 어떤 일이 일어나는지 자세히 알아보자.

### Phase 1: Composition

`@Composable` 함수 혹은 람다 블록 안에서 읽는 State는 Composition 단계에 영향을 주고, 그 이후의 단계에까지 영향을 줄 수도 있다. State의 값이 변경될 때, Recomposer는 해당 State를 구독하는 모든 Composable 함수들을 재실행하도록 스케쥴링한다. 만약 이전과 입력이 같다면 런타임에서 이를 스킵할 수 있음을 주의하자. 관련하여 더 많은 정보를 얻고 싶다면 [이곳](https://developer.android.com/jetpack/compose/lifecycle#skipping)에서 확인해보자.

Composition의 결과에 따라, Compose UI는 layout 단계와 drawing 단계를 실행한다. 만약 컴포저블 안의 내용과 크기가 변하지 않았다면, 이 과정을 스킵할 수도 있다.

```kotlin
var padding by remember { mutableStateOf(8.dp) }
Text(
    text = "Hello",
    // The `padding` state is read in the composition phase
    // when the modifier is constructed.
    // Changes in `padding` will invoke recomposition.
    modifier = Modifier.padding(padding)
)
```

### Phase 2: Layout

Layout 단계는 **측정(measurement)**과 **배치(placement)** 두 가지 단계로 나뉜다. 측정 단계에서는 `Layout` Composable에 전달된 measure 연산을 수행하는 람다 블록, `LayoutModifier` 인터페이스와 같은 `MeasureScope.measure` 메소드 등이 실행된다. 배치 단계에서는 `layout` 함수의 placement 블록, `Modifier.offset { … }` 람다 블록 등이 실행된다.

이 두 단계에서 읽은 State는 Layout 단계에 영향을 미치며, 잠재적으로 Drawing 단계까지 영향을 미칠 수 있다. State 값이 변경되면, Compose UI는 Layout 단계를 수행하고, 크기나 위치까지 변경되었다면 Drawing 단계도 수행하도록 스케쥴링한다.

더 정확하게 말하면, 측정 단계와 배치 단계는 각각 재시작 스코프를 갖고 있으며 배치 단계에서 읽은 상태를 측정 단계에서 재사용하지 않는다. 하지만 이 두 단계는 서로 얽혀있는 관계이기 때문에, 배치 단계에서 읽은 상태가 측정 상태의 재시작 스코프에 영향을 미칠 수 있다.

```kotlin
var offsetX by remember { mutableStateOf(8.dp) }
Text(
    text = "Hello",
    modifier = Modifier.offset {
        // The `offsetX` state is read in the placement step
        // of the layout phase when the offset is calculated.
        // Changes in `offsetX` restart the layout.
        IntOffset(offsetX.roundToPx(), 0)
    }
)
```

### Phase 3: Drawing

그리기 동작을 수행하는 코드에서 읽은 상태는 Drawing 단계에 영향을 미친다. 일반적으로 `Canvas()`,  `Modifier.drawBehind` 그리고 `Modifier.drawWithContent` 가 그 예시이다. 상태 값이 변경되면, Compose UI는 그리기 단계만 수행하게 된다.

```kotlin
var color by remember { mutableStateOf(Color.Red) }
Canvas(modifier = modifier) {
    // The `color` state is read in the drawing phase
    // when the canvas is rendered.
    // Changes in `color` restart the drawing.
    drawRect(color)
}
```

![https://developer.android.com/static/develop/ui/compose/images/phases-state-read-draw.svg](https://developer.android.com/static/develop/ui/compose/images/phases-state-read-draw.svg)

## Optimizing state reads

Compose는 상태를 읽기 위한 추적하는 행위를 함에 따라, 적절한 단계에 상태를 읽을 수 있도록 우리가 이 행위를 최소화할 수 있다.

다음 예시를 살펴보자. 유저가 스크롤할 때 parallax effect를 표현하기 위해 offset modifier를 사용하여 최종 레이아웃 위치 값을 가져오는 `Image()` 가 있다.

```kotlin
Box {
    val listState = rememberLazyListState()

    Image(
        // ...
        // Non-optimal implementation!
        Modifier.offset(
            with(LocalDensity.current) {
                // State read of firstVisibleItemScrollOffset in composition
                (listState.firstVisibleItemScrollOffset / 2).toDp()
            }
        )
    )

    LazyColumn(state = listState) {
        // ...
    }
}
```

이 코드는 작동은 하지만 최적의 성능을 내지는 못한다. 현재 코드는 `firstVisibleItemScrollOffset` 상태 값을 읽어 `Modifier.offset(offset: Dp)` 함수에 전달되며, 사용자가 스크롤할 때마다 `firstVisibleItemScrollOffset` 값이 변경된다. Compose는 이러한 상태를 추적하여 위 예시 코드 중 Box의 내용과 같은 상태를 가진 코드를 재시작하게 된다.

위는 **구성(Composition)** 단계에서 상태를 읽는 예시이다. Composition 단계에서 상태를 읽는다는 것은 기본적으로 나쁜 것이 아니라 데이터가 변경되어 새로운 UI를 방출하는 recomposition의 기본이 된다.

하지만 이 예에서는 최적화된 코드가 아닌데, 왜냐하면 스크롤 이벤트가 발생할 때마다 전체 composable 내용에 대한 평가 및 측정, 그리기 단계가 다시 수행되기 때문이다. 현재는 화면에서 보여지는 내용이 변경되지 않았음에도 매 스크롤 이벤트가 발생할 때마다 매 단계를 트리거하고 있지만, 매 스크롤마다 **무엇**을 보여줄 것인지에 대한 대상은 변하지 않고 **어디**에 보여지는지만 변경되기 때문에 Layout 단계만 다시 트리거하도록 최적화할 수 있다.

offset modifier를 적용하는 코드로 또 다른 친구가 있다. → `Modifier.offset(offset: Density.() → IntOffset)`

이 코드는 람다 블록의 매개변수로 offset을 적용하여 반환되는 offset으로 적용한다. 적용해보자.

```kotlin
Box {
    val listState = rememberLazyListState()

    Image(
        // ...
        Modifier.offset {
            // State read of firstVisibleItemScrollOffset in Layout
            IntOffset(x = 0, y = listState.firstVisibleItemScrollOffset / 2)
        }
    )

    LazyColumn(state = listState) {
        // ...
    }
}
```

이 코드가 왜 더 효율적인 성능을 낼까? modifier에 부여한 람다 블록은 **Layout** 단계(특히 Layout 단계 중 배치 단계에서)에서 접근하며, 이는 `firstVisibleItemScrollOffset` ****상태 값을 더 이상 composition 단계에서 읽지 않음을 의미한다. 따라서 `firstVisibleItemScrollOffset` 상태 값이 변경되면 Compose는 Layout과 Drawing 단계만 재시작할 수 있게 된다.

<aside>
⭐ **참고** : 람다 파라미터를 넘겨주는 행위가 단순히 값을 참조하는 것과 비교하여 부가적인 비용이 발생할 수 있다고 생각할 수 있다. 이는 맞는 말이지만, 위의 케이스에서는 Layout 단계에만 상태를 읽을 수 있도록 제한하는 것이 더 많은 이점을 가져다 준다. 스크롤할 때 매 프레임마다 **firstVisibleItemScrollOffset** 값이 변경되고, Layout 단계로 상태를 읽는 행위를 미룸으로써 우리는 불필요한 recomposition을 줄일 수 있다.

</aside>

이 예시에서 두 종류의 offset modifier에 의존하여 서로 다른 최적화 결과를 나타냈듯이, 상태를 읽는 것을 가장 낮은 단계에 두어 Compose가 수행하는 일을 최소화는 것이 일반적이다.

물론 Composition 단계에서도 상태를 읽도록 해야할 때가 있지만 상태 변경에 대한 필터링을 해줌으로써 다수의 불필요한 recomposition을 최소화하는 케이스도 있을 것이다. 이와 관련된 내용을 보고 싶다면, 이 페이지([derivedStateOf: 한개 이상의 State를 또 다른 State로 변환한다](https://www.notion.so/Side-effects-68bca357203d4b9baf90a7d73500d966?pvs=21))를 보아라.

## Recomposition loop (cyclic phase dependency)

앞서 Compose가 수행하는 단계들은 항상 같은 순서로 호출되고, 같은 프레임에 있는 동안에는 뒤의 단계로 이동할 방법이 없다고 설명했지만 다른 프레임에서 Compoisition 루프에 들어갈 수 없는 것은 아니다. 다음 예시를 살펴보자.

```kotlin
Box {
    var imageHeightPx by remember { mutableStateOf(0) }

    Image(
        painter = painterResource(R.drawable.rectangle),
        contentDescription = "I'm above the text",
        modifier = Modifier
            .fillMaxWidth()
            .onSizeChanged { size ->
                // Don't do this
                imageHeightPx = size.height
            }
    )

    Text(
        text = "I'm below the image",
        modifier = Modifier.padding(
            top = with(LocalDensity.current) { imageHeightPx.toDp() }
        )
    )
}
```

여기 그닥 좋지 않은 방법으로 구현된 `Image` 가 있고 그 아래 `Text` 가 있는 `Column` 이 있다. `Modifier.onSizeChanged()` 를 사용하여 이미지 크기를 파악하고 Text에서 `Modifier.padding()` 을 사용하여 파악된 크기만큼 이미지를 아래로 이동하고 있다. `Px` 에서 `Dp` 로의 부자연스러운 변환은 이미 이 코드에 문제가 있음을 나타내고 있다.

이 예제에서의 문제점은 단일 프레임 내에 “최종” 레이아웃 단계에까지 도달하지 않는 것이다. 이 코드는 불필요한 작업을 수행하는 여러 프레임에 의존하여 사용자의 화면에서 UI가 이리저리 이동하게 된다.

각 프레임을 단계별로 살펴보며 어떤 일이 일어나는지 알아보자.

첫 번째 프레임의 Composition 단계에서, `imageHeightPx` 값은 0이고 텍스트에는 `Modifier.padding(top = 0)` 이 제공된다. 그런 다음 Layout 단계가 수행되고 `onSizeChanged` modifier의 콜백이 호출된다. 이때 `imageHeightPx` 가 이미지의 실제 높이로 업데이트된다. Compose는 이 때 다른 프레임에 리컴포지션을 예약한다. Drawing 단계에서는 아직 업데이트된 값이 반영되지 않았기 때문에 텍스트의 패딩 값이 0이 적용된 채로 렌더링된다.

그런 다음 Compose는 `imageHeightPx` 값이 변경됨에 따라 예약된 두 번째 프레임을 시작한다. 컴포지션 단계에서 Box의 content 블록에서 상태를 읽고 이를 호출하는데, 이때 Text에는 이미지 높이에 해당하는 패딩이 적용된다. Layout 단계에서 코드는 `imageHeightPx` 의 값을 다시 설정하지만 값이 동일하기 때문에 recomposition을 예약하지는 않는다.

최종적으로 Text에 원하는 패딩을 적용했지만 패딩 값을 다른 단계에 전달하기 위해 중복된 값을 사용하는 추가적인 프레임이 생성되었고, 이는 바람직하지 않다.

![https://developer.android.com/static/develop/ui/compose/images/phases-recomp-loop.svg](https://developer.android.com/static/develop/ui/compose/images/phases-recomp-loop.svg)

이 예시는 다소 작위적으로 보일 수 있지만, 이와 같이 일반적으로 사용될 수 있는 패턴을 조심해야 한다.

- `Modifier.onSizeChanged()` , `onGloballyPositioned()` , 혹은 layout 연산 함수들
- State(상태)를 업데이트하는 행위
- 위의 State(상태)를 layout modifier에 삽입하는 행위 (`padding()` , `height()` 와 같은 함수들)
- 잠재적으로 재귀가 발생되는 행위

위 예제는 적절한 layout primitives를 사용하여 문제를 해결할 수 있다. 예제는 `Column()` 함수만 사용해서 구현할 수 있었지만, 커스텀 레이아웃을 작성해야 하는 더욱 복잡한 경우가 생길 수도 있다. 해당 [페이지](https://developer.android.com/develop/ui/compose/layouts/custom?_gl=1*1634y8k*_up*MQ..*_ga*OTU2NjYzMzkyLjE3MjA0MjI3OTE.*_ga_6HH9YJMN9M*MTcyMDQyMjc5MS4xLjAuMTcyMDQyMjc5MS4wLjAuMA..)에서 커스텀 레이아웃과 관련된 좀 더 자세한 내용을 확인할 수 있다.

해당 페이지의 궁극적인 내용은 측정되고 배치되어야 하는 여러 UI 요소들에 대해서 단일 진실 공급원(SSOT: Single Source Of Truth)을 가져야 한다는 것이다. 적절한 layout primitive를 사용하거나 커스텀한 레이아웃을 생성한다는 것은 최소한의 공유 부모가 여러 UI 요소 간의 관계를 조정하는 진실 공급원의 역할을 한다는 것을 의미한다. 하지만 동적 상태를 도입하면 이 원칙이 깨질 수 있다.