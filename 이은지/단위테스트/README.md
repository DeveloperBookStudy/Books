# 단위 테스트

## 책 정보
- 제목: 단위 테스트
- 저자: 블라디미르 코리코프 저 / 역자: 임준혁
- 링크: https://www.yes24.com/Product/Goods/104084175

## 읽기 전 생각해 볼 점
- 단위 테스트란
- 단위 테스트의 단위는 어느 정도의 단위를 말하는 걸까
- 단위 테스트를 작성해야 하는 이유
- 좋은 단위 테스트를 작성하는 방법
- private 메서드는 어떻게 하지

## 읽고 난 후 느낀 점
테스트를 작성하다 보면 막히는 부분이 생긴다. 분명 좋은 테스트의 기준을 알고 있음에도 불구하고, 그 기준에 맞춰 작성하는 게 쉽지 않을 때가 있다. 그렇다면 이건 기존 코드에 문제가 있을 수 있다는 신호가 아닐까? 이걸 잘 캐치할 수 있다면, 기존 코드를 리팩터링할 기회를 얻을 수 있다.

기존 코드에 어떤 문제가 있을 수 있을까? 예를 들어, 단위 테스트의 대상이 너무 많은 책임을 수행하고 있다면, 그 책임들을 적절하게 분리해 볼 수 있다. 그리고 테스트하려는 동작이 명확하게 드러나지 않는다면, 캡슐화를 통해 개선해볼 수 있을 것이다. 또한, 모듈 간 또는 외부 시스템 간의 결합이 너무 강한 경우에는 결합을 느슨하게 리팩터링하는 게 도움이 될 수 있다.

이렇게 단위 테스트를 작성하면서 만나는 어려움들은 사실 기존 코드의 개선점을 드러내는 신호가 되기도 한다. 테스트를 통해 기존 코드의 리팩터링이 이루어지고, 그 결과로 테스트를 작성하기 쉬운 코드가 만들어지는 어떤 선순환이 이뤄질 수 있다는 점이 크게 와닿았다.

## 이 책을 통해 얻은 것들
- 회사 과제 구현에 헥사고날 아키텍처를 적용하고 도메인, 애플리케이션, 인프라스트럭처. 각 레이어에 대한 단위 테스트를 작성했다.
  - 헥사고날 아키텍처에 대한 모호함이 어느정도 해소되었다.
  - 헥사고날 아키텍처 적용으로 각 레이어에 대한 경계가 명확해져서, 단위 테스트를 작성할 때 의존성에 대한 고민을 많이 하지 않아도 됐다. 따라서 단위 테스트 작성에 들이는 시간과 노력의 비용이 많이 줄었다.
  - 좋은 테스트와 좋은 설계 그리고 좋은 코드는 각각 별개로 진행되는 것이 아닌, 하나의 유기적인 프로세스로 연결되어 함께 발전하는 것이구나라는 것을 깨달았다.

- 단위 테스트 관련 블로그 글을 1개 작성했다.
  - 좀 더 내용을 보완하기 위해 현재는 비공개 상태이지만, 보완하여 추후 링크를 업데이트할 예정이다.(강제성 부여를 위함..)
  - 업데이트가 늦어진다면 스터디 팀원분들께 재촉을 부탁드립니다..
  - 블로그 글 링크: (업데이트 예정.!!)

## 참고
- https://shoulditestprivatemethods.com/
- https://tidyfirst.substack.com/p/canon-tdd