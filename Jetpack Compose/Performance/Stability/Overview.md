# Stability in Compose

Compose는 타입을 안정하거나(stable) 불안정한(unstable) 타입 두 가지 중 하나로 간주한다. 불변한(immutable) 타입이거나, Compose가 리컴포지션 중 값이 바뀌었다는 것을 알 수 있을 때 안정적인 타입으로 간주한다. Compose가 리컴포지션 중 값이 바뀌었다는 것을 알 수 없을 때 불안정한 타입으로 간주한다.

Compose는 리컴포지션 도중 스킵 가능한 composable인지 결정하기 위해 composable의 파라미터를 통해 안정성(stability)을 확인한다:

- **안정된 파라미터들(Stable parameters)**: composable이 변경되지 않은 안정된 파라미터를 가지는 경우, Compose는 리컴포지션 중 해당 composable을 스킵한다.
- **불안정한 파라미터들(Unstable parameters):** composable이 불안정한 파라미터를 가지는 경우, Compose는 항상 해당 컴포넌트의 부모가 리컴포지션이 일어날 때 해당 컴포넌트의 리컴포지션을 발생시킨다.

만약 앱이 리컴포지션을 발생시키는 불필요한 불안정 컴포넌트들이 다수 존재한다면, 성능 이슈 혹은 다른 문제가 발견될 수도 있다.

해당 문서는 성능과 유저 경험을 개선하기 위한 앱의 안정성(stability)를 증가시키는 방법에 대해서 설명한다.

## Immutable objects

아래의 코드들은 안전성(stability)과 리컴포지션의 원칙을 나타낸다.

`Contact` 클래스는 불변한 data class인데, 모든 파라미터가 `val` 키워드로 정의된 윈시 타입이기 때문이다. `Contact` 인스턴스를 만들게 되면, 해당 객체의 프로퍼티 값을 변경할 수 없다. 그렇게 하기 위해서는 새로운 객체를 만들어야 할 것이다.

```kotlin
data class Contact(val name: String, val number: String)
```

컴포저블 `ContactRow` 이 `Contact` 타입을 파라미터로 가진다고 가정하자.

```kotlin
@Composable
fun ContactRow(contact: Contact, modifier: Modifier = Modifier) {
   var selected by remember { mutableStateOf(false) }

   Row(modifier) {
      ContactDetails(contact)
      ToggleButton(selected, onToggled = { selected = !selected })
   }
}
```

유저가 토글 버튼을 클릭하고 이에 따라 `selected` 상태가 변경되면 어떤 일이 일어나는지 생각해보자:

1. Compose는 `ContactRow` 안의 코드를 리컴포지션 해야 하는지 평가한다.
2. Compose는 `ContactDetails` 의 유일한 인자가 `Contact` 타입임을 확인한다.
3. `Contact` 는 불변 데이터 클래스이기 때문에, Compose는 `ContactDetails` 의 파라미터가 변경되지 않았음을 확신한다.
4. 따라서 Compose는 `ContactDetails` 를 건너뛰고 리컴포지션하지 않는다.
5. 반면에 `ToggleButton` 의 인자는 변경되었으므로 Compose는 해당 컴포넌트를 다시 재구성한다.

### Mutable objects

앞의 예시는 불변(immutable) 객체를 사용했지만, 가변(mutable) 객체를 사용하는 것도 가능하다. 아래의 코드로 고려해보자.

```kotlin
data class Contact(var name: String, var number: String)
```

`Contact` 의 각 파라미터가 이제는 `var` 로 변경되어 해당 클래스는 더이상 불변성을 지니고 있지 않다. 프로퍼티가 변경된다고 하더라도 Compose는 알아챌 수 없게 되는데, Compose는 오로지 Compose의 [State 객체](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState)의 변화까지만 추적하기 때문이다.

Compose는  Contact와 같은 클래스를 불안정하다고 생각한다. Compose는 리컴포지션 도중 이러한 불안정한 클래스들을 스킵하지 않는다. 따라서, `Contact` 가 이런 식으로 정의되었다면, 앞선 예제의 `ContactRow` 는 `selected` 값이 변경될 때마다 리컴포지션된다.

## Implementation in Compose

중요하지는 않지만 Compose가 리컴포지션의 과정 속에서 어떤 함수를 스킵할지 결정하는 방법에 대해 고려하는 것이 도움될 수 있다.

Compose 컴파일러가 코드를 실행할 때, 각 함수와 타입에 여러가지 태그를 부여하고, 이 태그들은 리컴포지션 과정 중 Compose과 해당 함수과 타입을 어떻게 다룰 것인지 그 역할을 가진다.

<aside>
⭐ **참고:** 이 태그들은 해당 문서의 앞부분에서 설명한 바와 같이 리컴포지션과 안정성을 이해하는 데 꼭 필요하는 것은 아니지만, 안정성 문제를 디버깅하는 데에 있어 매우 유용하다.

</aside>

### Functions

Compose는 함수에 `skippable` 혹은 `restartable` 태그를 적용한다. 태그는 아래의 두 개가 모두 적용될 수도, 단 한 개도 적용되지 않을 수도 있다.

- **Skippable** : 컴파일러가 composable에 skippable 태그를 적용했다면, Compose는 리컴포지션 과정 중 인자의 값이 이전 값 그대로라면 스킵할 수 있다.
- **Restartable** : restartable이 적용된 composable은 리컴포지션이 시작되는 “범위(scope)”를 부여받는다. 다시 말해서, restartable 함수는 상태가 변경된 후 리컴포지션을 위해 Compose가 코드에 다시 접근하는 시작점이 될 수 있다.

### Types

Compose는 타입에 `immutable` 혹은 `stable` 둘 중 하나의 태그를 적용한다.

- **Immutable** : Compose는 프로퍼티가 절대 변경되지 않고 모든 메소드가 참조 투명한 경우 태그가 부여된다.
    - `String`, `Int` 그리고 `Float` 을 포함한 모든 원시 타입들은 immutable로 적용된다는 것을 알아두자.
- **Stable** : 초기화 이후 프로퍼티가 변경될 수 있음을 나타낸다. 혹시라도 프로퍼티들이 런타임이 변경된다면, Compose는 변화를 감지할 수 있다.

<aside>
⭐ **참고:** Compose가 스킵할 수 있도록 Composable의 파라미터들이 꼭 불변(immutable)이어야 할 필요는 없다. 매개 변수가 가변적이더라도 Compose 런타임에 모든 변경 사항이 알려지면 된다. 대부분 이러한 계약을 유지하기는 어렵지만, Compose는 이러한 계약을 유지할 수 있도록 **MutableState**, **SnapshotStateMap** 그리고 **SnapshotStateList** 등의 가변 클래스를 제공한다.

</aside>

## Debug stability

매개 변수가 변경되지 않았음에도 composable이 리컴포지션 되고 있다면, 가장 먼저 매개 변수들이 명확히 mutable로 정의되어 있는지 확인해보자. Compose는 프로퍼티가 `var` 로 정의되어 있거나, 불안정한(unstable) `val` 타입이라면 컴포넌트를 항상 리컴포지션시킨다.

Compose에서 안정성과 관련하여 보다 자세하게 이슈의 원인을 진단하는 방법을 알고 싶다면, [Debug stability 문서](https://developer.android.com/develop/ui/compose/performance/stability/diagnose)를 보아라.

## Fix stability issues

Compose를 구현하는 데에 있어 안전성(stability)을 제공하는 방법에 대해 알고 싶다면, [Fix stability issues 문서](https://developer.android.com/develop/ui/compose/performance/stability/fix)를 보아라.

## Summary

정리하자면, 아래의 사항들에 주목해야 한다.

- **매개 변수(Parameters):** Compose는 리컴포지션 과정 중 스킵할 composable을 결정하기 위해 각 composable들의 매개 변수의 안전성(stability)를 결정한다.
- **즉각적인 수정(Immediate fixes):** composable이 스킵되지 않고 성능 문제를 일으키는 경우, `var` 매개 변수와 같은 명백하게 불안정성을 제공하는 원인을 먼저 확인해야 한다.
- **컴파일러 보고서(Compiler reports):** 클래스에 대해 추론된 안전성을 확인하기 위해 [컴파일러 보고서](https://developer.android.com/develop/ui/compose/performance/stability/diagnose#compose-compiler)를 사용할 수 있다.
- **컬렉션(Collections):** Compose는 `List` , `Set` , `Map` 과 같은 컬렉션 클래스를 항상 불안정한 것으로 간주하는데, 불변성이 보장되지 않기 때문이다. 이를 위해 [Kotlinx 불변 컬렉션](https://developer.android.com/develop/ui/compose/performance/stability/fix?_gl=1*oswko5*_up*MQ..*_ga*NTkzMTY3MDc1LjE3MjE5MDgwODg.*_ga_6HH9YJMN9M*MTcyMTkwODA4Ny4xLjAuMTcyMTkwODA4Ny4wLjAuMA..#immutable-collections)을 사용하거나 클래스에 `@Immutable` 또는 `@Stable` 어노테이션을 사용할 수 있다.
- **다른 모듈(Other modules):** Compose는 Compose 컴파일러가 실행하지 않는 모듈의 클래스를 항상 불안정한 것으로 간주한다. 필요한 경우 클래스를 UI 모델의 클래스로 귀속시키자.