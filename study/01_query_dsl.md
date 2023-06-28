
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
