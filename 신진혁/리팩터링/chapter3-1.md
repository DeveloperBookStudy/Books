# 코드에서 나는 악취

## 기이한 이름 (Strange Names)

코드를 명료하게 표현하는 데 가장 중요한 요소 중 하나는 “**이름(name)**”입니다.

- **이름만 보고**도 해당 코드가 어떤 일을 하고, 어떻게 사용해야 하는지 **명확히** 파악할 수 있어야 합니다.
- 좋은 이름을 짓는 것은 쉽지 않지만, **이름을 잘못 지은 것을 발견하면 즉시 변경**하는 것이 좋습니다.
- 만약 **마땅한 이름이 떠오르지 않는다면**, 더 근본적인 설계 문제를 의심해볼 수 있습니다.

리팩터링에서 가장 자주 등장하는 기법으로 **함수 선언 바꾸기**, **변수 이름 바꾸기**, **필드 이름 바꾸기** 등이 있습니다.

이를 통해 코드 가독성과 유지보수성을 높일 수 있습니다.

---

## 함수 선언 바꾸기 (Changing Function Declaration)

**함수**는 소프트웨어를 조립하는 **접합부(연결부)** 역할을 합니다.

이 과정에서 가장 중요한 것은 **함수의 이름과 매개변수**입니다.

- **함수 이름**이 적절하면, 구현 코드를 열어보지 않고도 함수의 목적과 사용 방법을 빠르게 이해할 수 있습니다.
- **매개변수**는 함수가 외부 세계와 **어우러지는 방식**을 정의합니다.
    - 매개변수의 추상화 수준(구체화 vs 캡슐화 수준)에 따라 결합도와 재사용성이 달라지며, 어떤 매개변수를 제공해야 할지는 절대적인 정답이 없습니다.

### 함수 선언 바꾸기: 마이그레이션 절차

### 1) 함수 이름 바꾸기

아래 예시는 *둘레를 구하는 함수*를 예시로 든 것입니다.

```kotlin
fun circum(radius: Double): Double {
    return 2 * Math.PI * radius
}

```

이 함수를 `circumference`라는 더 명확한 이름으로 바꾸고자 한다면,

1. **새 함수를 추출**하여 기존 로직을 옮깁니다.
    
    ```kotlin
    fun circum(radius: Double): Double {
        return circumference(radius)
    }
    
    fun circumference(radius: Double): Double {
        return 2 * Math.PI * radius
    }
    
    ```
    
2. 테스트 후, 기존 함수를 **인라인**하거나 `@Deprecated`로 표시하여 사용을 줄입니다.
    
    ```kotlin
    @Deprecated(
        message = "Use 'circumference()' instead",
        replaceWith = ReplaceWith("circumference(radius)")
    )
    fun circum(radius: Double): Double {
        return circumference(radius)
    }
    
    ```
    
3. 외부에서 쓰던 코드를 모두 `circum` → `circumference`로 변경합니다.

### 2) 매개변수 추가하기

```kotlin
fun addReservation(customer: Customer) {
    this._reservation.add(customer)
}

```

‘우선순위 여부(isPriority)’라는 새로운 매개변수를 추가하고 싶다면:

1. **본문**을 새 함수로 추출(임시 이름 사용)
    
    ```kotlin
    fun addReservation(customer: Customer) {
        this.zz_addReservation(customer)
    }
    
    fun zz_addReservation(customer: Customer) {
        this._reservation.add(customer)
    }
    
    ```
    
2. 새 매개변수를 추가
    
    ```kotlin
    fun addReservation(customer: Customer) {
        this.zz_addReservation(customer, false)
    }
    
    fun zz_addReservation(customer: Customer, isPriority: Boolean) {
        this._reservation.add(customer)
    }
    
    ```
    
3. 기존 함수를 **인라인**하고, 새 함수를 **원래 이름**으로 바꿉니다.

```kotlin
fun addReservation(customer: Customer, isPriority: Boolean) {
    this._reservation.add(customer)
}

```

> Tip: 코틀린에서는 isPriority가 nullable이라면 require(isPriority != null) 같은 방어 코드를 통해 안전하게 처리할 수 있습니다.
> 

### 3) 매개변수를 속성으로 바꾸기

다음 예시는 특정 고객이 **미국 뉴잉글랜드(NE) 지역**에 속하는지를 판별하는 함수입니다.

```kotlin
fun inNewEngland(aCustomer: Customer): Boolean {
    return listOf("MA", "CT", "ME", "VT", "NH", "RI").contains(aCustomer.address.state)
}

```

이 로직에서 `aCustomer.address.state`를 **매개변수**로 직접 받도록 리팩터링하려면,

1. **변수 추출** ( `stateCode` )
    
    ```kotlin
    fun inNewEngland(aCustomer: Customer): Boolean {
        val stateCode = aCustomer.address.state
        return listOf("MA", "CT", "ME", "VT", "NH", "RI").contains(stateCode)
    }
    
    ```
    
2. **함수 추출** (새로운 이름 사용)
    
    ```kotlin
    fun inNewEngland(aCustomer: Customer): Boolean {
        val stateCode = aCustomer.address.state
        return xxNewInNewEngland(stateCode)
    }
    
    fun xxNewInNewEngland(stateCode: String): Boolean {
        return listOf("MA", "CT", "ME", "VT", "NH", "RI").contains(stateCode)
    }
    
    ```
    
3. **변수 인라인**하기
    
    ```kotlin
    fun inNewEngland(aCustomer: Customer): Boolean {
        return xxNewInNewEngland(aCustomer.address.state)
    }
    
    ```
    
4. **기존 함수 호출문**을 새 함수 호출로 변경하고, 최종적으로 새 함수 이름을 `inNewEngland`로 바꿉니다.

---

## 변수 이름 바꾸기 (Rename Variable)

변수의 이름은 해당 변수가 사용되는 **범위**와 **영속성**에 큰 영향을 미칩니다.

특히, **필드 수준의 변수(프로퍼티)**는 오랫동안 유지되므로 좋은 이름이 매우 중요합니다.

```kotlin
var tpHd = "untitled"

// 읽기
val result = "test" + tpHd
// 수정
tpHd = "temp"

```

이 `tpHd`의 이름을 `title`로 바꾸고 싶다면,

1. **변수 캡슐화**를 먼저 진행
    
    ```kotlin
    var tpHd = "untitled"
    
    fun title(): String = tpHd
    fun setTitle(title: String) {
        tpHd = title
    }
    
    ```
    
2. 호출부를 모두 `title()`, `setTitle()`로 변경한 후, 내부 변수를 `_title`로 바꾼다.
    
    ```kotlin
    var _title = "untitled"
    
    fun title(): String = _title
    fun setTitle(title: String) {
        _title = title
    }
    
    ```
    

> 코틀린 팁:
> 
> 
> 실제로는 아래처럼 **프로퍼티**와 **커스텀 getter/setter**를 바로 사용하는 편이 좋습니다.
> 
> ```kotlin
> var title: String = "untitled"
>     private set
> 
> ```
> 
> 혹은
> 
> ```kotlin
> private var _title: String = "untitled"
> var title: String
>     get() = _title
>     set(value) { _title = value }
> 
> ```
> 

### 상수 이름 바꾸기

상수(예: `const val`)의 경우에도 비슷한 방식으로 이름을 교체할 수 있습니다.

한 번에 모든 곳을 수정하기 부담스럽다면, **복제**를 만들어 점진적으로 교체하는 방법이 있습니다.

```kotlin
const val cpyNm: String = "애크미 구스베리"
```

1. 복제본 생성
    
    ```kotlin
    const val companyName: String = "애크미 구스베리"
    const val cpyNm: String = companyName
    ```
    
2. 점차 `cpyNm` 대신 `companyName`을 사용하도록 수정 후, 마지막에 `cpyNm` 삭제.

---

## 필드 이름 바꾸기 (Rename Field)

데이터 구조(테이블, 레코드)는 **프로그램 이해**에 큰 영향을 미칩니다.

**레코드 구조체의 필드 이름**이 중요하다는 것은, 코틀린에서라면 곧 **데이터 클래스의 프로퍼티**가 중요함을 뜻합니다.

```kotlin
data class Organization(
    val name: String,
    val country: String
)
```

### 캡슐화 후 이름 변경

1. **Organization**을 **캡슐화**하는 클래스로 감싸거나, 내부 필드를 **private**로 두어 접근자를 제공
    
    ```kotlin
    class OrganizationData(data: Organization) {
        private var _name: String = data.name
        private var _country: String = data.country
    
        var name: String
            get() = _name
            set(value) {
                _name = value
            }
    
        var country: String
            get() = _country
            set(value) {
                _country = value
            }
    }
    ```
    
2. 필드(`_name`)를 `_title`로 바꾸고, **public 프로퍼티**를 유지
    
    ```kotlin
    class OrganizationData(data: Organization) {
        private var _title: String = data.name
        private var _country: String = data.country
    
        var name: String
            get() = _title
            set(value) { _title = value }
    
        var country: String
            get() = _country
            set(value) { _country = value }
    }
    ```
    
3. 점차 **생성자**와 **호출부**도 `title`을 쓸 수 있도록 수정
    
    ```kotlin
    data class OrganizationData(
        val title: String? = null,  // 새 필드
        val name: String,           // 기존 필드
        val country: String
    )
    
    class Organization(data: OrganizationData) {
        private var _title: String = data.title ?: data.name
        private var _country: String = data.country
    
        val name: String
            get() = _title
    
        val country: String
            get() = _country
    }
    ```
    
4. 기존 `name` 대신 `title`을 사용하는 코드를 늘려가고, 필요 없다면 최종적으로 `name`을 제거 가능

이 과정을 통해 **필드 이름 변경**과 함께 점진적으로 호출부에 영향이 확산되지 않도록 캡슐화할 수 있습니다.

> 참고:
> 
> - 코틀린에서는 처음부터 `data class Organization(val title: String, val country: String)`로 바로 정의하는 것도 가능합니다. 다만 *“점진적 리팩터링”* 시나리오를 보여주기 위해 책에서는 이러한 단계를 예시로 들고 있습니다.
> - 목적에 따라 `data class`(불변 모델)와 일반 `class`(동작, 로직 포함)를 구분하여 사용하는 편이 좋습니다.
