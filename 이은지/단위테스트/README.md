# 단위 테스트

## 책 정보
- 제목: 단위 테스트
- 저자: 블라디미르 코리코프 저 / 역자: 임준혁
- 링크: https://www.yes24.com/Product/Goods/104084175

## 읽기 전 생각해볼 점
- 단위 테스트란
- 단위 테스트의 단위는 어느 정도의 단위를 말하는 걸까
- 단위 테스트를 작성해야 하는 이유
- 좋은 단위 테스트를 작성하는 방법
- private 메서드는 어떻게 하지

## 읽고 난 후 느낀 점
테스트를 작성하다 보면 막히는 부분이 생긴다. 분명 좋은 테스트의 기준을 알고 있음에도 불구하고, 그 기준에 맞춰 작성하는 게 쉽지 않을 때가 있다. 그렇다면 이건 기존 코드에 문제가 있을 수 있다는 신호가 아닐까? 이걸 잘 캐치할 수 있다면, 기존 코드를 리팩터링할 기회를 얻을 수 있다.

기존 코드에 어떤 문제가 있을 수 있을까? 예를 들어, 단위 테스트의 대상이 너무 많은 책임을 수행하고 있다면, 그 책임들을 적절하게 분리해볼 수 있다. 그리고 테스트하려는 동작이 명확하게 드러나지 않는다면, 캡슐화를 통해 개선해볼 수 있을 것이다. 또한, 모듈 간 또는 외부 시스템 간의 결합이 너무 강한 경우에는 결합을 느슨하게 리팩터링하는 게 도움이 될 수 있다.

이렇게 단위 테스트를 작성하면서 만나는 어려움들은 사실 기존 코드의 개선점을 드러내는 신호가 되기도 한다. 테스트를 통해 기존 코드의 리팩터링이 이루어지고, 그 결과로 테스트를 작성하기 쉬운 코드가 만들어지는 어떤 선순환이 이뤄질 수 있다는 점이 크게 와닿았다.

## 참고
- https://shoulditestprivatemethods.com/
- https://tidyfirst.substack.com/p/canon-tdd