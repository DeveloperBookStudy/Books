## 리팩터링-긴 매개변수 목록

매개변수 목록이 길어지면 이해하기 어려워진다.

1. 다른 매개변수에서 얻어올 수 있는 매개변수가 있을 경우 → **매개변수를 질의 함수로 바꾸기(11.5)**
2. 데이터 구조에서 값들을 뽑아 각각 별개의 매개변수로 전달한다면 → **객체 통째로 넘기기(11.4)**
3. 항상 함께 전달되는 매개변수 → **매개변수 객체 만들기(6.8)**
4. 함수의 동작 방식을 결정하는 플래그 역할의 매개변수 → **플래그 인수제거하기(11.3)**
5. 여러 개의 함수가 특정 매개변수들의 값을 공통으로 사용할 때 → **여러 함수를 클래스로 묶기(6.9)**로 공통 값들을 클래스의 필드로 정의
