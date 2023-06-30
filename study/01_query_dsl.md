
# QueryDSL

## Querydsl 설정과 검증

- build.gradle에 주석을 참고해서 querydsl 설정 추가

``` build.gradle
buildscript {
	ext {
		queryDslVersion = "5.0.0"
	}
}


plugins {
	id 'java'
	id 'org.springframework.boot' version '2.7.14-SNAPSHOT'
	id 'io.spring.dependency-management' version '1.0.15.RELEASE'
	//querydsl 추가
	id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}

group = 'study'
version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '11'
}

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
	maven { url 'https://repo.spring.io/milestone' }
	maven { url 'https://repo.spring.io/snapshot' }
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'

	// querydsl 추가
	implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
	implementation "com.querydsl:querydsl-apt:${queryDslVersion}"

	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'com.h2database:h2'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}

def querydslDir = "$buildDir/generated/querydsl"

querydsl {
	jpa = true
	querydslSourcesDir = querydslDir
}
sourceSets {
	main.java.srcDir querydslDir
}
configurations {
	querydsl.extendsFrom compileClasspath
}

compileQuerydsl {
	options.annotationProcessorPath = configurations.querydsl
}
``` 

### Querydsl 환경설정 검증

#### 검증용 엔티티 생성
```java
@Entity
@Getter @Setter
public class Hello {

    @Id @GeneratedValue
    private Long id;
    
}
``` 

#### 검증용 Q 타입 생성
![](https://github.com/dididiri1/querydsl/blob/main/study/images/01_01.png?raw=true)

#### Gradle IntelliJ 사용법
- Gradle -> Tasks -> build -> clean
- Gradle -> Tasks -> other -> compileQuerydsl

#### Gradle 콘솔 사용법
- ./gradlew clean compileQuerydsl

#### Q 타입 생성 확인
- build -> generated -> querydsl
  - study.querydsl.entity.QHello.java 파일이 생성됨

> 참고: Q타입은 컴파일 시점에 자동 생성되므로 버전관리(GIT)에 포함하지 않는 것이 좋다.  
> 생성 위치를 gradle build 폴더 아래 생성되도록 했기 때문에 이 부분도 자연스럽게 해결된다.  
> (대부분 gradle build 폴더를 git에 포함하지 않는다.)


#### 테스트 케이스로 실행 검증
```java
@SpringBootTest
@Transactional
class QuerydslApplicationTests {

	@Autowired
	EntityManager em;

	@Test
	void contextLoads() {
		Hello hello = new Hello();
		em.persist(hello);

		JPAQueryFactory query = new JPAQueryFactory(em);
		QHello qHello = new QHello("h"); 

		Hello result = query
				.selectFrom(qHello)
				.fetchOne();

		assertThat(result).isEqualTo(hello);
        assertThat(result.getId()).isEqualTo(hello.getId());
	}

}
```

## 예제 도메인 모델

### 예제 도메인 모델과 동작확인

![](https://github.com/dididiri1/querydsl/blob/main/study/images/01_02.png?raw=true)


### Member 엔티티

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {"id", "username", "age"})
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String username;

    private int age;

    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    public Member(String username, int age, Team team) {
        this.username = username;
        this.age = age;
        if (team != null) {
            changeTeam(team);
        }
    }

    private void changeTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);
    }
    
}
```

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {"id", "name"})
public class Team {

    @Id @GeneratedValue
    @Column(name = "team_id")
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();

    public Team(String name) {
        this.name = name;
    }

}
```

### Querydsl vs JPQL

``` java
@Test
public void startJPQL() throws Exception {
    String qlString = "select m from Member m " +
                      "where username = :username";
    Member findMember = em.createQuery(qlString, Member.class)
                .setParameter("username", "member1")
                .getSingleResult();

    assertThat(findMember.getUsername()).isEqualTo("member1");

}
```

``` java
@Test
public void startQuerydsl() throws Exception {
    JPAQueryFactory queryFactory = new JPAQueryFactory(em);

    QMember m = new QMember("m"); // 식별자 아무거나 줘야됨.

    Member findMember = queryFactory
                .selectFrom(m)
                .from(m)
                .where(m.username.eq("member1"))
                .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");

}
```
- EntityManager 로 JPAQueryFactory 생성
- Querydsl은 JPQL 빌더
- JPQL: 문자(실행 시점 오류), Querydsl: 코드(컴파일 시점 오류)
- JPQL: 파라미터 바인딩 직접, Querydsl: 파라미터 바인딩 자동 처리

### 기본 Q-Type 활용

#### Q클래스 인스턴스를 사용하는 2가지 방법
``` java
QMember qMember = new QMember("m"); //별칭 직접 지정
QMember qMember = QMember.member; //기본 인스턴스 사용
``` 

#### 기본 인스턴스를 static import와 함께 사용
``` java
import static study.querydsl.entity.QMember.*;

@Test
public void startQuerydsl() {

    Member findMember = queryFactory
                             .select(member)
                             .from(member)
                             .where(member.username.eq("member1"))
                             .fetchOne();
                             
    assertThat(findMember.getUsername()).isEqualTo("member1");
}
``` 

#### 다음 설정을 추가하면 실행되는 JPQL을 볼 수 있다
``` yml
spring.jpa.properties.hibernate.use_sql_comments: true
``` 

> 참고: 같은 테이블을 조인해야 하는 경우가 아니면 기본 인스턴스를 사용하자

### 검색 조건 쿼리

#### 기본 검색 쿼리
``` java
@Test
public void search() throws Exception {
     Member findMember = queryFactory
                .selectFrom(member)
                .where(member.username.eq("member1")
                        .and(member.age.eq(10))
                ).fetchOne();

     assertThat(findMember.getUsername()).isEqualTo("member1");
}
``` 
- 검색 조건은 .and() , . or() 를 메서드 체인으로 연결할 수 있다.

> 참고: select , from 을 selectFrom 으로 합칠 수 있음

### JPQL이 제공하는 모든 검색 조건 제공
``` java
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'


member.username.isNotNull() //이름이 is not null


member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30


member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30


member.username.like("member%") //like 검색
member.username.contains("member") // like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색

``` 

### AND 조건을 파라미터로 처리
``` java
@Test
public void searchAndParam() throws Exception {
     
     Member findMember = queryFactory
                .selectFrom(member)
                .where(
                        member.username.eq("member1"),
                        member.age.eq(10)
                ).fetchOne();

     assertThat(findMember.getUsername()).isEqualTo("member1");

}
``` 
- where() 에 파라미터로 검색조건을 추가하면 AND 조건이 추가됨
- 이 경우 null 값은 무시됨. (메서드 추출을 활용해서 동적 쿼리를 깔끔하게 만들 수 있음)

### 결과 조회
- fetch() : 리스트 조회, 데이터 없으면 빈 리스트 반환
- fetchOne() : 단 건 조회
  - 결과가 없으면 : null
  - 결과가 둘 이상이면 : com.querydsl.core.NonUniqueResultException
- fetchFirst() : limit(1).fetchOne()
- fetchResults() : 페이징 정보 포함, total count 쿼리 추가 실행
- fetchCount() : count 쿼리로 변경해서 count 수 조회

``` java
@Test
public void resultFetch() throws Exception {

    List<Member> fetch = queryFactory
                .selectFrom(member)
                .fetch();

    Member fetchOne = queryFactory
                .selectFrom(member)
                .where(member.username.eq("member1"))
                .fetchOne();

    Member fetchFirst = queryFactory
                .selectFrom(member)
                .fetchFirst();

    QueryResults<Member> results = queryFactory
                .selectFrom(member)
                .fetchResults();

    results.getTotal();
    List<Member> content = results.getResults();

    //count 쿼리로 변경
    long fetchCount = queryFactory
                .selectFrom(member)
                .fetchCount();
}
```

### 정렬

``` java
/**
 * 회원 정렬 순서
 * 1. 회원 나이 내림차순(desc)
 * 2. 회원 이름 올림차순(asc)
 * 단 2에서 회원 이름이 없으면 마지막에 출력(nulls last)
 */
@Test
public void sort() throws Exception {
    em.persist(new Member(null, 100));
    em.persist(new Member("member5", 100));
    em.persist(new Member("member6", 100));
    
    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(100))
            .orderBy(member.age.desc(), member.username.asc().nullsLast())
            .fetch
    Member member5 = result.get(0);
    Member member6 = result.get(1);
    Member memberNull = result.get(2);
    
    assertThat(member5.getUsername()).isEqualTo("member5");
    assertThat(member6.getUsername()).isEqualTo("member6");
    assertThat(memberNull.getUsername()).isNull
}
``` 

- desc() , asc() : 일반 정렬
- nullsLast() , nullsFirst() : null 데이터 순서 부여

## 집합

### 집합 함수

``` java
@Test
public void aggregation() throws Exception {
     List<Tuple> result = queryFactory
                .select(
                        member.count(),
                        member.age.sum(),
                        member.age.avg(),
                        member.age.max(),
                        member.age.min()
                )
                .from(member)
                .fetch();

     Tuple tuple = result.get(0);
     
     assertThat(tuple.get(member.count())).isEqualTo(4);
     assertThat(tuple.get(member.age.sum())).isEqualTo(100);
     assertThat(tuple.get(member.age.avg())).isEqualTo(25);
     assertThat(tuple.get(member.age.max())).isEqualTo(40);
     assertThat(tuple.get(member.age.min())).isEqualTo(10);

    }
``` 
- JPQL이 제공하는 모든 집합 함수를 제공한다.
- tuple은 프로젝션과 결과반환에서 설명한다.
- **실무에는 tuple 반환보다는 dto로 변환해서 사용함.**

### GroupBy 사용
``` java
/**
 * 팀의 이름과 각 팀의 평균 연령을 구해라.
 *
 */
@Test
public void group() throws Exception {
    List<Tuple> result = queryFactory
            .select(team.name, member.age.avg())
            .from(member)
            .join(member.team, team)
            .groupBy(team.name)
            .fetch
            
    Tuple teamA = result.get(0);
    Tuple teamB = result.get(2);
    
    assertThat(teamA.get(team.name)).isEqualTo("teamA");
    assertThat(teamA.get(member.age.avg())).isEqualTo(1
    assertThat(teamB.get(team.name)).isEqualTo("teamB");
    assertThat(teamB.get(member.age.avg())).isEqualTo(3);
}
``` 

## 조인 - 기본 조인

- INNER JOIN(내부 조인)
  - 두 테이블을 조인할 때, 두 테이블에 모두 지정한 열의 데이터가 있어야 한다. 
- OUTER JOIN(외부 조인)
  - 두 테이블을 조인할 때, 1개의 테이블에만 데이터가 있어도 결과가 나온다.

### 기본 조인

조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭(alias)으로 사용할  
Q 타입을 지정하면 된다.

``` java
join(조인 대상, 별칭으로 사용할 Q타입)
``` 

``` java
/**
* 기본 조인
* 팀 A에 소속된 모든 회원을 찾아라.
*/
@Test
public void join() throws Exception {
    List<Member> result = queryFactory
            .selectFrom(member)
             //.join(member.team, team) 
            .leftJoin(member.team, team)
            .where(team.name.eq("teamA"))
            .fetch
    assertThat(result)
            .extracting("username")
            .containsExactly("member1", "member2");
}
``` 

- join() , innerJoin() : 내부 조인(inner join)
- leftJoin() : left 외부 조인(left outer join)
- rightJoin() : rigth 외부 조인(rigth outer join)
- JPQL의 on 과 성능 최적화를 위한 fetch 조인 제공 다음 on 절에서 설명

### 세타 조인
연관관계가 없는 필드로 조인
``` java
/**
* 세타 조인
* 회원의 이름이 팀 이름과 같은 회원 조회
*/
@Test
public void theta_join() throws Exception {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));
    em.persist(new Member("teamC"));
    
    List<Member> result = queryFactory
            .select(member)
            .from(member, team)
            .where(member.username.eq(team.name))
            .fetch();
            
    assertThat(result)
            .extracting("username")
            .containsExactly("teamA", "teamB인");
}
``` 
- from 절에 여러 엔티티를 선택해서 세타 조인
- 외부 조인 불가능 -> 다음에 설명할 조인 on을 사용하면 외부 조인 가능

## 조인 - on절
- ON절을 활용한 조인(JPA 2.1부터 지원)
  1. 조인 대상 필터링
  2. 연관관계 없는 엔티티 외부 조인

1. 조인 대상 필터링

### 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회 (3개)

#### 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회 (4개)

##### 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회 (5개)

###### 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회 (6개)

``` java
/**
 * 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
 * JPQL: select m, t from Member m left join m.taem t on t.name = 'teamA'
 * SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='teamA'
 */
@Test
public void join_on_filtering() throws Exception {
    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(member.team, team).on(team.name.eq("teamA"))
            .fetch();
            
    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }    
}
``` 

#### 결과

``` sql
*/ select
       member0_.member_id as member_i1_1_0_,
       team1_.team_id as team_id1_2_1_,
       member0_.age as age2_1_0_,
       member0_.team_id as team_id4_1_0_,
       member0_.username as username3_1_0_,
       team1_.name as name2_2_1_ 
   from
       member member0_ 
   left outer join
       team team1_ 
           on member0_.team_id=team1_.team_id 
           and (
               team1_.name=?
           )
``` 

``` log
tuple = [Member(id=3, username=member1, age=10), Team(id=1, name=teamA)]
tuple = [Member(id=4, username=member2, age=20), Team(id=1, name=teamA)]
tuple = [Member(id=5, username=member3, age=30), null]
tuple = [Member(id=6, username=member4, age=40), null]
``` 

> 참고: on 절을 활용해 조인 대상을 필터링 할 때, 외부조인이 아니라 내부조인(inner join)을 사용하면,  
> where 절에서 필터링 하는 것과 기능이 동일하다. 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때,  
> 내부조인 이면 익숙한 where 절로 해결하고, 정말 외부조인이 필요한 경우에만 이 기능을 사용하자

2. 연관관계 없는 엔티티 외부 조인

#### 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인

``` java  
/**
 * 연관관계가 없는 엔티티 외부 조인
  * 회원의 이름이 팀 이름과 같은 대상 외부 조인
 * JPQL: SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
 * SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name
*/
@Test
public void join_on_no_relation() throws Exception {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));
    em.persist(new Member("teamC"));
    
    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(team).on(member.username.eq(team.name))
            .fetch();
            
    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
``` 

``` sql
*/ select
        member0_.member_id as member_i1_1_0_,
        team1_.team_id as team_id1_2_1_,
        member0_.age as age2_1_0_,
        member0_.team_id as team_id4_1_0_,
        member0_.username as username3_1_0_,
        team1_.name as name2_2_1_ 
    from
        member member0_ 
    left outer join
        team team1_ 
            on (
                member0_.username=team1_.name
            ) 
``` 

``` log
tuple = [Member(id=3, username=member1, age=10), null]
tuple = [Member(id=4, username=member2, age=20), null]
tuple = [Member(id=5, username=member3, age=30), null]
tuple = [Member(id=6, username=member4, age=40), null]
tuple = [Member(id=7, username=teamA, age=0), Team(id=1, name=teamA)]
tuple = [Member(id=8, username=teamB, age=0), Team(id=2, name=teamB)]
tuple = [Member(id=9, username=teamC, age=0), null]
``` 

- 주의! 문법을 잘 봐야 한다. leftJoin() 부분에 일반 조인과 다르게 엔티티 하나만 들어간다.
  - 일반조인: leftJoin(member.team, team)
  - on조인: from(member).leftJoin(team).on(xxx)