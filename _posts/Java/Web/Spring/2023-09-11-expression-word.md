---
title:  "[Web] Spring, Spring boot 공부하면서 나온 용어 정리"  
excerpt: "여러가지 잡다 용어"

categories: # 분류하고싶은 카테고리 입력
  - Spring
tags:     # 중요 키워드
  - [Term]

toc: true
toc_sticky: true
published: true

author: Hwet

date: 2023-09-11
last_modified_at: 2023-09-11
---

인터넷에서 검색하여 여러가지를 참고하여 정리하였습니다.  
[참고]()
{: .notice--warning}


## 📌 의존성 (Dependency)

의존성은 프로젝트가 필요로 하는 외부 라이브러리나 모듈을 나타낸다. 

스프링 부트에서는 주로 Maven이나 Gradle을 통해서 의존성을 관리한다.

### 의존성 관리 (Dependency Management)

컴파일 시점에서 의존성을 해결하고 라이브러리를 프로젝트에 추가하는 작업

Maven이나 Gradle에 외부 라이브러리를 설정하는 것을 `의존성 관리` 라고 부른다. 

이것은 특정 라이브러리나 모듈을 사용하려고 선언하는 것이며, 프로젝트 빌드 시 해당 의존성을 다운로드하고 사용 가능하게 만든다.


### 의존성 주입 (Dependency Injection)

런타임 시점에서 객체간의 의존성을 관리하고 주입하는 작업

코드에서 객체를 생성하거나 참조하는 곳에서 외부 의존성을 주입받는다는 것을 의미한다. 


## 📌 JPA (Java Persistence API)

자바 언어를 기반으로 한 데이터 베이스를 다루는 자바 ORM (Object-Relational Mapping) 기술

관계형 데이터베이스와 자바 객체 간의 매핑을 단순화하고 객체 지향 프로그래밍과 데이터베이스 간의 상호 작용을 효율적으로 처리하는데 사용된다.

### ORM (Object-Relational Mapping)

JPA는 관계형 데이터베이스와 객체 지향 프로그래밍 언어 간의 매핑을 제공합니다.

데이터베이스 테이블과 자바 객체 간의 대응 관계를 설정하고 객체를 데이터베이스에서 조회,저장 할 경우 SQL 쿼리문을 직접 작성하지 않고 사용이 가능하다.

```java 
public interface SomethingRepository extends JpaRepository<Something, Long> {
  // SELECT * FROM Something 과 동일한 기능을 하는 메서드 
  List<Something> findAll();
}
```

쿼리문을 직접 작성하지 않아도 SQL의 쿼리문이 작성된것처럼 작동이 가능하다.

### EntityManager 

EntityManager는 JPA에서 엔티티를 관리하고 데이터베이스와 상호 작용하는 주요 인터페이스이다. 

> 엔티티의 영속성 관리 

엔티티의 생명주기를 관리한다. 엔티티를 영속성 컨텍스트에 등록하고, 수정된 엔티티를 추적하며, 필요할 때 데이터베이스와의 동기화를 수행한다.

> 트랜잭션 관리 

JPA는 트랜잭션 내에서 엔티티를 변경하고 데이터베이스에 반영할 수 있도록 하는데, 이때 트랜잭션의 시작,커밋,롤백 등과 관련된 작업을 수행한다.

> EntityManager 사용 X

```java 
@Service
public class UserService {
    private final UserRepository userRepository;

    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUserById(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }

    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

> EntityManager 사용

```java 
@Service
public class UserService {
    private final EntityManager entityManager;

    @PersistenceContext
    public UserService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public User getUserById(Long userId) {
        return entityManager.find(User.class, userId);
    }

    public void saveUser(User user) {
        entityManager.persist(user);
    }
}
```

@PersistenceContext는 EntityManager를 주입하기 위해 사용된다.


#### EntityManager 장.단점

`장점`

> 직접적인 데이터베이스 제어 

데이터베이스에 대한 제어가 직접적입니다. SQL 쿼리를 직접 작성하고 실행 할 수 있으며, 복잡한 연산을 수행할 때 용이합니다.

> 고도로 사용자 지정된 쿼리 

Repository를 사용하는 것 보다 EntityManager를 사용하면, 쿼리를 더욱 고도로 사용자 지정할 수 있다.
복잡한 조인이나 저장 프로시저와 같은 작업을 수행할 때 더욱 많은 제어가 가능하다.

==> 복잡한 조건,로직, 쿼리문이라고 생각하면 될거같다.

> 특수한 경우 처리 

특수한 데이터베이스 엑세스 작업이나 특수한 작업을 수행할 때 유용할 수 있습니다.

`단점`

> 복잡성 증가

데이터베이스 엑세스 작업을 직접 처리하는 것이 복잡성을 증가시킬 수 있다.
특히 간단한 CRUD 연산을 자주 수행하는 경우, 불필요한 코드 작성이 필요할 수 있다.


❗ 일반적으로 Repository의 기능을 사용하며, 사용자 정의 레포지토리를 구현하고자 할때 EntityManager를 사용한다고 한다.
















*** 



***
<br>
    
    📢 개인 공부 및 정리용 블로그로, 틀리거나 주관적인 의견이 들어갈 수 있습니다.
    잘못된 부분이 있다면, 언제든지 댓글이나 메일로 알려주시면 참고하겠습니다. 🔔

[맨 위로 이동하기](#){: .btn .btn--primary }{: .align-right}