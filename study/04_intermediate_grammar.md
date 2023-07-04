# 중급 문법

### 프로젝션과 결과 반환 - 기본

- 프로젝션: select 대상 지정
    - 프로젝션 대상이 하나면 타입을 명확하게 지정할 수 있음
    - 프로젝션 대상이 둘 이상이면 튜플이나 DTO로 조회

``` java
List<String> result = queryFactory
        .select(member.username)
        .from(member)
        .fetch();
``` 

#### 튜플 조회

``` java
@Test
public void tupleProjection() throws Exception {
    List<Tuple> result = queryFactory
            .select(member.username, member.age)
            .from(member)
            .fetch();
            
    for (Tuple tuple : result) {
        String username = tuple.get(member.username);
        Integer age = tuple.get(member.age);
        
        System.out.println("username = " + username);
        System.out.println("age = " + age);
    }
}
``` 

### 프로젝션과 결과 반환 - DTO 조회

#### 순수 JPA에서 DTO 조회(new 오퍼레이션)

``` java
/**
 * new 오퍼레이션 방법
 */
@Test
public void findDtoByJPQL() throws Exception {

    List<MemberDto> resultList = em.createQuery("select new study.querydsl.dto.MemberDto(m.username, m.age) from Member m", MemberDto.class)
            .getResultList();
            
    for (MemberDto memberDto : resultList) {
        System.out.println("memberDto = " + memberDto);
    }
}
``` 
- 순수 JPA에서 DTO를 조회할 때는 new 명령어를 사용해야함
- DTO의 package이름을 다 적어줘야해서 지저분함
- 생성자 방식만 지원함(생성자 없으면 문법 오류남)

### Querydsl 빈 생성(Bean population)
결과를 DTO 반환할 때 사용
다음 3가지 방법 지원

- 프로퍼티 접근
- 필드 직접 접근
- 생성자 사용

#### 프로퍼티 접근 - Setter
``` java
@Test
public void findDtoBySetter() throws Exception {

    List<MemberDto> result = queryFactory
            .select(Projections.bean(MemberDto.class,
                    member.username, member.age))
            .from(member)
            .fetch();
            
    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
``` 
- Setter가 필수다 없으면 값이 Null

#### 필드 직접 접근
``` java
@Test
public void findDtoByFiled() throws Exception {

    List<MemberDto> result = queryFactory
            .select(Projections.fields(MemberDto.class,
                    member.username, member.age)) // 생성자 타입이 안맞으면 에러남.
            .from(member)
            .fetch();
            
    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
``` 

#### 별칭이 다를 때
``` java
@Data
@NoArgsConstructor
public class UserDto {

    private String name;

    private int age;
}
``` 

``` java
@Test
public void findUserDto() throws Exception {

    QMember memberSub = new QMember("memberSub");
    
    List<UserDto> result = queryFactory
            .select(Projections.fields(UserDto.class,
                    member.username.as("name"),
                    //ExpressionUtils.as(member.username,"name"), 위에꺼랑 똑같지만 지저분함.
                    ExpressionUtils.as(
                            JPAExpressions
                                    .select(memberSub.age.max())
                                    .from(memberSub), "age"
                    )))
            .from(member)
            .fetch();
            
    for (UserDto userDto : result) {
        System.out.println("userDto = " + userDto);
    }
}
``` 
- 프로퍼티나, 필드 접근 생성 방식에서 이름이 다를 때 해결 방안
- ExpressionUtils.as(source,alias) : 필드나, 서브 쿼리에 별칭 적용
- username.as("memberName") : 필드에 별칭 적용

#### 생성자 사용
``` java
@Test
public void findDtoByConstructor() throws Exception {

    List<MemberDto> result = queryFactory
            .select(Projections.constructor(MemberDto.class,
                    member.username, member.age))
            .from(member)
            .fetch();
            
    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
``` 
- 생성자가 필수이며, 매개변수 타입이 순서대로 맞아야됨.

### 프로젝션과 결과 반환 - @QueryProjection

#### DTO 생성자에 @QueryProjection 어노테이션 추가
``` java
@Data
@NoArgsConstructor
public class MemberDto {

    private String username;

    private int age;

    @QueryProjection
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }
    
    ...
}
``` 
- ./gradlew compileQuerydsl
- QMemberDto 생성 확인
``` java
@Test
public void findDtoByQueryProjection() throws Exception {
    List<MemberDto> result = queryFactory
            .select(new QMemberDto(member.username, member.age))
            .from(member)
            .fetch();
    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```
#### 장점
- 이 방법은 컴파일러로 타입을 체크할 수 있으므로 가장 안전한 방법이다.
- 위에 다른 방식은 살행은 정상 작동하지만, 런타임 오류 발생, 필드 잘못 작성시 오류 찾기 힘듬.
#### 단점
- DTO가 querydsl 어노테이션 인해서 의존성이 생김
- 보통 DTO 경우에는 repository, service, controller 여러 레이어에 걸쳐서 쓰이는데. 의존적으로 설계 되어 있기 떄문
- Q 파일을 생성해야 한다.

### distinct
``` java
List<String> result = queryFactory
         .select(member.username).distinct()
         .from(member)
         .fetch();
``` 

> 참고: distinct는 JPQL의 distinct와 같다.


### 동적 쿼리 - BooleanBuilder 사용

#### 동적 쿼리를 해결하는 두가지 방식
- BooleanBuilder
- Where 다중 파라미터 사용

``` java
@Test
public void dynamicQuery_BooleanBuilder() throws Exception {
    String usernameParam = "member1";
    Integer ageParam = 10;
    
    List<Member> result = searchMember1(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {

    BooleanBuilder builder = new BooleanBuilder();
    //BooleanBuilder builder = new BooleanBuilder(member.username.eq(usernameCond)); // 값을 미리 셋팅할 수 있음
    
    if (usernameCond != null) {
        builder.and(member.username.eq(usernameCond));
    }
    
    if (ageCond != null) {
        builder.and(member.age.eq(ageCond));
    }
    
    List<Member> result = queryFactory
            .selectFrom(member)
            .where(builder)
            .fetch();
            
    return result;
}
``` 

### 동적 쿼리 - Where 다중 파라미터 사용(실무에서 많이씀!)

``` java
@Test
public void dynamicQuery_WhereParam() throws Exception {
    String usernameParam = "member1";
    Integer ageParam = 10;
    
    List<Member> result = searchMember2(usernameParam, ageParam);
    
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {
    return queryFactory
            .selectFrom(member)
            .where(usernameEq(usernameCond), ageEq(ageCond))
            //.where(allEq(usernameCond, ageCond))
            .fetch();
}

// Predicate --> BooleanExpression 반환 타입으로 해놓는게 낫음.
private BooleanExpression usernameEq(String usernameCond) {
    return usernameCond != null ? member.username.eq(usernameCond) : null; // 삼항 연산자
}
private BooleanExpression ageEq(Integer ageCond) {
    if (ageCond == null) {
        return null;
    }
    
    return member.age.eq(ageCond);
}
``` 
- where 조건에 null 값은 무시된다.
- 메서드를 다른 쿼리에서도 재활용 할 수 있다.
- 쿼리 자체의 가독성이 높아진다.

#### 조합 가능
``` java
private BooleanExpression allEq(String usernameCond, Integer ageCond) {
    return usernameEq(usernameCond).and(ageEq(ageCond));
}
``` 
- 위에서 정의했던 usernameEq, ageEq함수의 반환 값을 .and()로 묶은 조건식을 반환할 수 있음.
- null 체크는 주의해서 처리해야함

### 수정, 삭제 벌크 연산

#### 쿼리 한번으로 대량 데이터 수정
``` java
@Test
public void bulkUpdate() throws Exception {

    //member1 = 10 -> 비회원
    //member2 = 20 -> 비회원
    //member3 = 30 -> 유지
    //member4 = 40 -> 유지
    
    long count = queryFactory
            .update(member)
            .set(member.username, "비회원")
            .where(member.age.lt(28))
            .execute();
            
    // 벌크 연산 이후, 영속성 컨텍스트 초기화
    em.flush();
    em.clear();
    
    List<Member> result = queryFactory
            .selectFrom(member)
            .fetch();
            
    for (Member member1 : result) {
        System.out.println("member1 = " + member1);
    }
}    
``` 

#### 쿼리 한번으로 대량 데이터 연산
``` java
@Test
public void bulkAdd() throws Exception {
    queryFactory
            .update(member)
            //.set(member.age, member.age.add(1)) // 더하기
            .set(member.age, member.age.add(-1)) // 빼기
            //.set(member.age, member.age.multiply(1)) // 곱하기
            .execute();

}
``` 

#### 쿼리 한번으로 대량 데이터 삭제
``` java
@Test
public void bulkDelete() throws Exception {
    queryFactory
            .delete(member)
            .where(member.age.gt(18))
            .execute();

}
``` 

> 주의: JPQL 배치와 마찬가지로, 영속성 컨텍스트에 있는 엔티티를 무시하고 실행되기 때문에 배치 쿼리를  
> 실행하고 나면 영속성 컨텍스트를 초기화 하는 것이 안전하다.

### SQL function 호출하기

SQL function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있다

##### member M으로 변경하는 replace 함수 사용

``` java
@Test
public void sqlFunction() throws Exception {
    List<String> result = queryFactory
            .select(
                    Expressions.stringTemplate(
                            "function('replace', {0}, {1}, {2})",
                            member.username, "member", "M"))
            .from(member)
            .fetch();
            
    for (String s : result) {
        System.out.println("s = " + s);
    }
}
```

#### 소문자로 변경
``` java
@Test
public void sqlFunction2() throws Exception {
    List<String> result = queryFactory
            .select(member.username)
            .from(member)
//                .where(member.username.eq(
//                        Expressions.stringTemplate(
//                                "function('lower', {0})", member.username)))
            .where(member.username.eq(member.username.lower()))
            .fetch();

    for (String s : result) {
        System.out.println("s = " + s);
    }
}
``` 

> 참고 lower 같은 ansi 표준 함수들은 querydsl이 상당부분 내장하고 있다. 따라서 다음과 같이   
> 처리해도 결과는 같다.