
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



