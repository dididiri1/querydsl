# 강의 들으면서 외적인 내용 정리하기

## 생성자 어노테이션

- @AllArgsConstructor
- @RequiredArgsConstructor
- @NoArgsConstructor

### @NoArgsConstructor(기본 생성자)


> 참고: 엔티티는 반드시 NoArgsConstructor를 가져야 합니다.

- 컴파일 오류
 - 컴파일 시점에 에러 발생

- 런타임 오류
  - 사용자가 이벤트 클릭시 에러 발생

