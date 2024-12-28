# 5장 목과 테스트 취약성
> 테스트에서 목을 사용하는 것은 논란의 여지가 있는 주제다. ~

- 목은 언제 사용하는 게 좋을까?
- 먼저, 목과 스텁을 구분해야한다.
- 그래서 이런 경우에는 이런 식으로 테스트를 작성하는 게 좋다
- 그러다보면 자연스럽게 이런 아키텍처가 된다.
---
## 목과 스텁 구분
- 테스트 대역이란?
  - 테스트 대역은 모든 유형의 가짜 의존성을 설명하는 포괄적인 용어로, 영화 산업의 '스턴트 대역'이라는 개념에서 비롯됐다.

  - 테스트 대역의 주 용도는 테스트를 편리하게 하는 것이다.

- 테스트 대역의 두 가지 유형
  ![IMG_5_1](./images/IMG_5_1.jpg)
  - 테스트 대역에는 더미, 스텁, 스파이, 목, 페이크 등의 다섯 가지 변형이 있는데, 이는 다시 목과 스텁이라는 두 가지 유형으로 분류할 수 있다.

  - 같은 유형 내에서는 각각의 구현 세부 사항에서 미미한 차이가 있다.
    - 따라서 스파이는 기능적으로 목과 같고, 더미와 페이크는 스텁과 같은 역할을 한다.

- 목과 스텁의 차이점
  ![IMG_5_2](./images/IMG_5_2.jpg)
  - 목은 외부로 나가는 상호 작용을 모방하고 검사하는 데 도움이 되는 테스트 대역이다.
    - SUT가 의존성의 상태를 변경하기 위해 의존성을 호출하는 것에 해당된다.

  - 스텁은 내부로 들어오는 상호 작용을 모방하는 데 도움이 되는 테스트 대역이다.
    - SUT가 의존성을 호출해 입력 데이터를 가져오는 것에 해당된다.

  - 목은 SUT와 관련된 의존성 간의 상호 작용을 모방하고 검사하는 반면, 스텁은 모방만 한다.

- 도구로서의 목과 테스트 대역으로서의 목
  - 도구로서의 목(Mock)은 테스트 대역으로서의 목이나 스텁을 만드는 데 사용하는 목 라이브러리의 클래스이다. 도구로서의 목을 사용해 목과 스텁, 이 두 가지 유형의 테스트 대역을 생성할 수 있기 때문에 도구로서의 목과 테스트 대역으로서의 목을 혼동하지 않아야 한다.

  - 목 라이브러리에서 Mock 클래스를 사용해 목을 생성
    ```java
    @Test
    public void sendingGreetingsEmail() {
      IEmailGateway mock = Mockito.mock(IEmailGateway.class); // Mock(도구로서의 목)으로 mock(테스트 대역으로서의 목) 생성
      Controller sut = new Controller(mock);

      sut.greetUser("user@email.com");

      verify(mock, times(1)).sendGreetingsEmail("user@email.com"); // 테스트 대역으로 하는 SUT의 호출을 검사
    }
    ```

  - Mock 클래스를 사용해 스텁을 생성
    ```java
    @Test
    public void creatingAReport() {
      IDatabase stub = Mockito.mock(IDatabase.class); // Mock(도구로서의 목)으로 스텁 생성
      when(stub.getNumberOfUsers()).thenReturn(10); // 준비한 응답 설정
      Controller sut = new Controller(stub);

      Report report = sut.createReport();

      assertEquals(10, report.getNumberOfUsers());
    }
    ```
  
- 스텁과의 상호 작용을 검증하지 말라
  >  스텁과의 상호 작용을 검증하는 것은 취약한 테스트를 야기하는 일반적인 안티 패턴이다.

  - SUT에서 스텁으로의 호출은 SUT가 생성하는 최종 결과가 아니다. 이러한 호출은 최종 결과를 산출하기 위한 수단일 뿐이다. 즉, 스텁은 SUT가 출력을 생성하도록 입력을 제공한다.
  
  - 스텁으로 상호 작용 검증(깨지기 쉬운 테스트의 예)
    ```java
    @Test
    public void creatingAReportWithOverSpecification() {
      IDatabase stub = Mockito.mock(IDatabase.class);
      when(stub.getNumberOfUsers()).thenReturn(10);
      Controller sut = new Controller(stub);

      Report report = sut.createReport();

      assertEquals(10, report.getNumberOfUsers());
      verify(stub, times(1)).getNumberOfUsers(); // 스텁으로 상호 작용 검증(과잉 명세)
    }
    ```
  - 4장에서 살펴봤듯이, 테스트에서 거짓 양성을 피하고 리팩터링 내성을 향상시키는 방법은 구현 세부 사항이 아니라 최종 결과(이상적으로는 비개발자들에게 의미가 있어야 함)를 검증하는 것이였다.  
  `verify(mock, times(1)).sendGreetingsEmail("user@email.com");`
    - 인사 메일을 보내는 것은 비즈니스 담당자가 시스템에 하길 원하는 결과에 부합한다.

  - `getNumberOfUsers()`
    - 해당 메서드를 호출하는 것은 보고서 작성에 필요한 데이터를 수집하는 방법에 대한 내부 구현 세부 사항이지 전혀 결과가 아니다.

- 목과 스텁 함께 쓰기
  > 때로는 목과 스텁의 특성을 모두 나타내는 테스트 대역을 만들 필요가 있다.

  - 목이자 스텁인 `storeMock`
    ```java
    @Test
    public void purchaseFailsWhenNotEnoughInventory() {
      IStore storeMock = Mockito.mock(IStore.class);
      when(storeMock.hasEnoughInventory(Product.SHAMPOO, 5)).thenReturn(false);
      Customer sut = new Customer();

      boolean success = sut.purchase(storeMock, Product.SHAMPOO, 5);
      
      assertFalse(success);
      verify(storeMock, never()).removeInventory(Product.SHAMPOO, 5); // SUT에서 수행한 호출을 검사
    }
    ```
    - 준비된 응답을 반환하는 `storeMock`

    - SUT에서 수행한 메서드 호출을 검증하는 `storeMock`
      - 스텁과의 상호 작용을 검증하는 것이 아니다!

## 식별할 수 있는 동작과 구현 세부 사항
- 이상적으로 시스템의 공개 API는 식별할 수 있는 동작과 일치해야 하며, 모든 구현 세부 사항은 클라이언트 눈에 보이지 않아야 한다. 이러한 시스템은 API 설계가 잘돼 있다.

  - 코드가 식별할 수 있는 동작인지 여부는 해당 클라이언트가 누구인지, 그리고 해당 클라이언트의 목표가 무엇인지에 달려 있다.
  
  - 식별할 수 있는 동작이 되려면 코드가 이러한 목표 중 하나에라도 직접적인 관계가 있어야 한다.

  - 클라이언트라는 단어는 코드가 있는 위치에 따라 다른 것을 의미할 수 있다. 흔한 예로 동일한 코드베이스, 외부 애플리케이션, 또는 사용자 인터페이스 등의 클라이언트 코드가 있다.

  - 구현 세부 사항을 유출하는 User 클래스
    ```java
    public class User {
      private String name;

      public String getName() {
        return name;
      }

      public void setName(String name) {
        this.name = name;
      }

      public String normalizeName(String name) {
        String result = (name == null ? "" : name).trim();

        if (result.length() > 50) {
          return result.substring(0, 50);
        }

        return result;
      }

      // 클라이언트 코드
      public class UserController {

        // 이 메서드의 목표는 사용자 이름을 변경하는 것이다.
        public void renameUser(int userId, string newName) {
          User user = getUserFromDatabase(userId);

          String normalizedName = user.normalizeName(newName);
          user.setName(normalizedName);

          saveUserToDatabase(user);
        }
      }
    }
    ```
    - `User` 클래스는 `name` 속성의 `getter`와 `setter`, 그리고 `normalizeName()` 메서드로 구성된 공개 API가 있다.

    - `UserController`는 `User` 클래스의 클라이언트 코드로서 `renameUser()` 메서드의 목표는 사용자 이름을 변경하는 것이다.

    - `User` 클래스의 `setter`만 이 요구 사항을 충족한다.
    
    - `normalizeName` 메서드도 작업이지만, 클라이언트의 목표에 직결되지는 않는다.
      - `UserController`가 `normalizeName` 메서드를 호출하는 이유는 `User`의 불변 속성을 만족시키기 위함이다. 따라서 `normalizeName`은 클래스의 공개 API로 유출되는 구현 세부사항이다.

    - API를 잘 설계하려면, User 클래스는 `normalizeName()` 메서드를 숨기고 속성 세터를 클라이언트 코드에 의존하지 않으면서 내부적으로 호출해야 한다.

  - API가 잘 설계된 User 클래스
    ```java
    public class User {
      private String name;

      public String getName() {
        return name;
      }

      public void setName(String name) {
        this.name = normalizeName(name);
      }

      private String normalizeName(String name) {
        String result = (name == null ? "" : name).trim();

        if (result.length() > 50) {
          return result.substring(0, 50);
        }

        return result;
      }
    }

    public class UserController {
      public void renameUser(int userId, String new Name) {
        User user = getUserFromDatabase(userId);
        user.setName(newName);
        saveUserToDatabase(user);
      }
    }
    ```
    - 식별할 수 있는 동작 `setName()`만 공개되어 있고, 구현 세부사항 `normalizeName()`은 비공개 API 뒤에 숨겨져 있다.
  
  - 클래스가 구현 세부 사항을 유출하는지 판단하는 데 도움이 되는 유용한 규칙
  
    - 단일한 목표를 달성하고자 클래스에서 호출해야 하는 연산의 수가 1보다 크면 해당 클래스에서 구현 세부 사항을 유출할 가능성이 있다. 이상적으로는 단일 연산으로 개별 목표를 달성해야 한다.

    - 위의 예제에서 `UserController`는 `User`의 두 가지 작업을 사용해야 했다.
      ``` java
      String normalizedName = user.normalizeName(newName);
      user.setName(normalizedName);
      ```
    
    - 리팩터링 후에 연산 수가 1로 감소했다.
      ``` java
      user.setName(newName);
      ```
  
  - 잘 설계된 API와 캡슐화
    - 캡슐화는 불변성 위반이라고도 하는 모순을 방지하는 조치다. 불변성은 항상 참이어야 하는 조건이다.
      - 위 예제의 `User` 클래스에서는 사용자 이름이 50자를 초과하면 안 된다는 불변성이 있었다.
      - 구현 세부 사항을 노출하면 불변성 위반을 가져온다. 원래 버전의 `User`는 구현 세부 사항을 유출할 뿐만 아니라 캡슐화를 제대로 유지하지 못했다. 클라이언트는 불변성을 우회해서 이름을 먼저 정규화하지 않고 새로운 이름을 할당할 수 있었다. 즉, 구현 세부사항을 노출하면 불변성 위반을 가져온다.

    - 계속해서 증가하는 코드 복잡도에 대처할 수 있는 방법은 캡슐화 말고는 없다.

    - 캡슐화를 올바르게 유지해서 개발자가 실수할 여지를 최대한 없게 하는 것이 좋다.

    - 캡슐화는 궁극적으로 단위 테스트와 동일한 목표를 달성한다. 즉, 소프트웨어 프로젝트의 지속적인 성장을 가능하게 하는 것이다.

  - 모든 구현 세부사항을 비공개로 하면 테스트가 식별할 수 있는 동작을 검증하는 것 외에는 다른 선택지가 없으며, 이로 인해 리팩터링 내성도 자동으로 좋아진다.

  - API를 잘 설계하면 단위 테스트도 자동으로 좋아진다.
    - 연산과 상태를 최소한으로 노출해야 한다.
    
    - 클라이언트가 목표를 달성하는 데 직접적으로 도움이 되는 코드만 공개해야 하며, 다른 모든 것은 구현 세부 사항이므로 비공개 API 뒤에 숨겨야 한다.

## 목과 테스트 취약성 간의 관계
> 육각형 아키텍처(hexagonal architecture), 내부 통신과 외부 통신의 차이점  
> 그리고 목과 테스트 취약성 간의 연관성. 목이 테스트에 취약한 이유?... 취약성을 일으키는 원인!

- 육각형 아키텍처
  ![IMG_5_3](./images/IMG_5_3.jpg)
  - 대표적인 애플리케이션은 도메인 계층과 애플리게이션 서비스 계층으로 구성된다.
  
  - 도메인 계층
    - 애플리케이션의 비즈니스 로직이 있다.
  
  - 애플리케이션 서비스
    - 도메인 계층 위에 있으며 도메인 클래스와 프로세스 외부 의존성 간의 작업을 조정한다.
      
      - 데이터베이스를 조회하고 해당 데이터로 도메인 클래스 인스턴스 구체화
      
      - 해당 인스턴스에 연산 호출
      
      - 결과를 데이터베이스에 다시 저장

- 육각형 아키텍처의 목적 세 가지
  - 도메인 계층과 애플리케이션 서비스 계층 간의 관심사 분리
    - 비즈니스 로직은 애플리케이션의 가장 중요한 부분이므로, 도메인 계층은 해당 비즈니스 로직에 대해서만 책임을 져야 하며, 다른 모든 책임에서는 제외돼야 한다. 외부 애플리케이션과 통신하거나 데이터베이스에서 데이터를 검색하는 것과 같은 책임은 애플리케이션 서비스에 귀속돼야 한다.

    - 반대로 애플리케이션 서비스에는 어떤 비즈니스 로직도 있으면 안 된다. 요청이 들어오면 도메인 클래스의 연산으로 변환한 다음 결과를 저장하거나 호출자에게 다시 반환해서 도메인 계층으로 변환하는 책임이 있다.

    - 도메인 계층을 애플리케이션 도메인 지식(사용 방법) 모음으로, 애플리케이션 서비스 계층을 일련의 비즈니스 유스케이스(사용 대상)로 볼 수 있다.

  - 애플리케이션 내부 통신
    - 육각형 아키텍처는 애플리케이션 서비스 계층에서 도메인 계층으로 흐르는 단방향 의존성 흐름을 규정한다. 도메인 계층의 내부 클래스는 도메인 계층 내부 클래스끼리 서로 의존하고, 애플리케이션 서비스 계층의 클래스에 의존하지 않는다.

  - 애플리케이션 간의 통신
    - 외부 애플리케이션은 애플리케이션 서베스 계층에 있는 공통 인터페이스를 통해 해당 애플리케이션에 연결된다. 아무도 도메인 계층에 직접 접근할 수 없다.

- 잘 설계된 API의 원칙에는 프랙탈(fractal) 특성이 있는데, 이는 전체 계층만큼 크게도, 단일 클래스만큼 작게도 똑같이 적용되는 것이다. 각 계층의 API를 잘 설계하면(즉, 구현 세부 사항을 숨기면) 테스트도 프랙탈 구조를 갖기 시작한다. 즉, 달성하는 목표는 같지만 서로 다른 수준에서 동작을 검증한다.
  ![IMG_5_4](./images/IMG_5_4.jpg)
  
  - 애플리케이션의 각 계층은 식별할 수 있는 동작을 나타내며 해당 구현 세부 사항을 포함하고 있다.
    
    - 예를 들어, 도메인 계층의 식별할 수 있는 동작은 이 계층의 연산과 상태이고, 연산과 상태는 애플리케이션 서비스 계층이 적어도 하나의 목표를 달성하는 데 도움이 된다.

  - 이전 장에서 어떤 테스트든 비즈니스 요구 사항으로 거슬러 올라갈 수 있어야 한다고 했다. 각 테스트는 도메인 전문가에게 의미 있는 이야기를 전달해야 하며, 그렇지 않으면 테스트가 구현 세부 사항과 결합돼 있으므로 불안정하다는 강한 암시다. 이제 그 이유를 알 수 있길 바란다.
  
- 시스템 내부 통신과 시스템과 통신
  - 시스템 내부 통신
    - 애플리케이션 내 클래스 간의 통신이다.

  - 시스템 간 통신
    - 애플리케이션이 다른 애플리케이션과 통신하는 것을 말한다.

  - 시스템 내부 통신은 구현 세부 사항이고, 시스템 간 통신은 그렇지 않다.
    - 연산을 수행하기 위한 도메인 클래스 간의 협력은 식별할 수 있는 동작이 아니므로 구현 세부 사항에 해당한다. 이러한 협력은 클라이언트의 목표가 직접적인 관계가 없다. 따라서 이러한 렵력(구현 세부 사항)과 결합하면 테스트가 취약해진다.

    - 시스템 외부 환경과 통신하는 방식은 전체적으로 해당 시스템의 식별할 수 있는 동작을 나타낸다. 이는 애플리케이션에 항상 있어야 하는 계약이다. 외부 애플리케이션과 통신할 때 사용하는 통신 패턴은 항상 외부 애플리케이션이 이해할 수 있도록 유지해야 한다. 시스템 내부에서 하는 리팩터링과는 다르다.

  - 목을 사용하면 시스템과 외부 애플리케이션 간의 통신 패턴을 확인할 때 좋다. 반대로 시스템 내 클래스 간의 통신을 검증하는 데 사용하면 테스트가 구현 세부 사항과 결합된다.

- 단위 테스트의 고전파와 런던파 재고

  - 런던파
    - 불변 의존성 외 모든 의존성에 대해 테스트 대역을 사용한다.

    - 시스템 내 통신과 시스템 간 통신을 구분하지 않는다. 그 결과, 테스트는 애플리케이션과 외부 시스템 간의 통신을 확인하는 것처럼 클래스 간 통신도 확인한다.
    - 런던파를 따라 목을 무분별하게 사용하면 종종 구현 세부 사항에 결합돼 테스트에 리팩터링 내성이 없게 된다.
    - 4장에서 살펴봤듯이, 리팩터링 내성은 대부분 이진 선택이다. 리팩터링 내성이 저하되면 테스트는 가치가 없어진다.
  
  - 고전파
    - 공유 의존성에 대해 테스트 대역을 사용한다.

    - 테스트 간에 공유하는 의존성(대부분이 SMTP 서비스나 메시지 버스등 프로세스 외부 의존성에 해당)만 교체하자고 한다.
    - 2장에서 책의 저자가 런던파보다 고전파를 더 선호한다고 했는데, 그 이유를 알 수 있는 부분이다.

- 그러나 모든 프로세스 외부 의존성을 목으로 해야 하는 것은 아니다.
  
  - 프로세스 외부 의존성이 애플리케이션을 통해서만 접근할 수 있으면, 이러한 의존성과의 통신은 시스템에서 식별할 수 있는 동작이 아니다. 실제로 외부에서 관찰할 수 없는 프로세스 외부 의존성은 애플리케이션의 일부로 작용한다.

    - 좋은 예로 애플리케이션에서만 사용되는 데이터베이스가 있다. 어떤 외부 시스템도 이 데이터베이스에 접근할 수 없다. 따라서 기존 기능을 손상시키지 않는 한 시스템과 애플리케이션 데이터베이스 간의 통신 패턴을 원하는 대로 수정할 수 있다.

    - 완전히 통제권을 가진 프로세스 외부 의존성에 목을 사용하면 깨지기 쉬운 테스트로 이어진다. 데이터베이스에서 테이블을 분할하거나 저장 프로시저에서 매개변수 타입을 변경할 때마다 테스트가 빨간색이 되는 것을 아무도 원하지 않는다. 데이터베이스와 애플리케이션은 하나의 시스템으로 취급해야 한다.
   