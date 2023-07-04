# 실무 활용 - 순수 JPA와 Querydsl
- 순수 JPA 리포지토리와 Querydsl
- 동적쿼리 Builder 적용
- 동적쿼리 Where 적용
- 조회 API 컨트롤러 개발

### 순수 JPA 리포지토리와 Queryds

### JPAQueryFactory 스프링 빈 등록
#### 다음과 같이 JPAQueryFactory 를 스프링 빈으로 등록해서 주입받아 사용해도 된다.
``` java
@SpringBootApplication
public class QuerydslApplication {

	public static void main(String[] args) {
		SpringApplication.run(QuerydslApplication.class, args);
	}

	@Bean
	JPAQueryFactory jpaQueryFactory(EntityManager em) {
		return new JPAQueryFactory(em);
	}
}
``` 
> 참고: 동시성 문제는 걱정하지 않아도 된다. 왜냐하면 여기서 스프링이 주입해주는 엔티티 매니저는 실제 동작 시점에  
> 진짜 엔티티 매니저를 찾아주는 프록시용 가짜 엔티티 매니저이다. 이 가짜 엔티티 매니저는 실제  
> 사용 시점에 트랜잭션 단위로 실제 엔티티 매니저(영속성 컨텍스트)를 할당해준다.  
> 자세한 내용은 자바 ORM 표준 JPA 책 13.1 트랜잭션 범위의 영속성 컨텍스트를 참고하자.

### 동적 쿼리와 성능 최적화 조회 - Builder 사용

#### MemberTeamDto - 조회 최적화용 DTO 추가
``` java
@Data
public class MemberTeamDto {

    private Long id;
    private String username;
    private int age;
    private Long teamId;
    private String teamName;

    @QueryProjection
    public MemberTeamDto(Long id, String username, int age, Long teamId, String teamName) {
        this.id = id;
        this.username = username;
        this.age = age;
        this.teamId = teamId;
        this.teamName = teamName;
    }
}
``` 
- @QueryProjection 을 추가했다. QMemberTeamDto 를 생성하기 위해 ./gradlew compileQuerydsl 을 한번 실행하자.

> 참고: @QueryProjection 을 사용하면 해당 DTO가 Querydsl을 의존하게 된다. 이런 의존이 싫으면,  
> 해당 에노테이션을 제거하고, Projection.bean(), fields(), constructor() 을 사용하면 된다.

#### 회원 검색 조건
``` java
@Data
public class MemberSearchCondition {

    private String username;
    private String teamName;
    private Integer ageGoe;
    private Integer ageLoe;

}
```
- 이름이 너무 길면 MemberCond 등으로 줄여 사용해도 된다

### 동적쿼리 - Builder 사용

#### Builder를 사용한 예제
``` java
public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {

    BooleanBuilder builder = new BooleanBuilder();
    
    if (hasText(condition.getUsername())) {
        builder.and(member.username.eq(condition.getUsername()));
    }
    
    if (hasText(condition.getTeamName())) {
        builder.and(team.name.eq(condition.getTeamName()));
    }
    
    if (condition.getAgeGoe() != null) {
        builder.and(member.age.goe(condition.getAgeGoe()));
    }
    
    if (condition.getAgeLoe() != null) {
        builder.and(member.age.loe(condition.getAgeLoe()));
    }       
    
    return queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(builder)
            .fetch();
}
```

#### 조회 예제 테스트
``` java
@Test
public void searchTest() throws Exception {
    
    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(35);
    condition.setAgeLoe(40);
    condition.setTeamName("teamB");
    
    List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);
    
    for (MemberTeamDto memberTeamDto : result) {
        System.out.println("memberTeamDto = " + memberTeamDto);
    }
    
    assertThat(result).extracting("username").containsExactly("member4");
}
``` 
> 오류 정정  
> 강의 영상에서는 member.id.as("memberId") 라고 적었는데, QMemberTeamDto 는 생성자를 사용하기  
> 때문에 필드 이름을 맞추지 않아도 된다. 따라서 member.id 만 적으면 된다.

### 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용

#### Where절에 파라미터를 사용한 예제
``` java
public List<MemberTeamDto> search(MemberSearchCondition condition) {
    return queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(
                    usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe())
            )
            .fetch();
}

private BooleanExpression usernameEq(String username) {
    return hasText(username) ? member.username.eq(username) : null;
}

private BooleanExpression teamNameEq(String teamName) {
    return hasText(teamName) ? null : team.name.eq(teamName);
}

private BooleanExpression ageGoe(Integer ageGoe) {
    return ageGoe != null ? member.age.goe(ageGoe) : null;
}

private BooleanExpression ageLoe(Integer ageLoe) {
    return ageLoe != null ? member.age.loe(ageLoe) : null;
}
```
- where 절에 파리미터 방식을 사용하면 조건 재사용 가능