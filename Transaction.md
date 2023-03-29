### Transaction

- 외부 API POST 요청과 DB 저장이 하나의 트랜잭션으로 묶였을 때
    - 외부 API는 어차피 롤백이 안 된다. 실패해서 예외가 발생해서 다시 그걸 취소하는 요청 보내는 건 의미가 있나?
        - 이게 의문이 드는 점은 1) 외부 API POST -> 2) 외부 API GET -> 3) DB 저장 일 경우, 3에서 예외가 발생하면 그렇다 치는데 2번에서 예외가 발생했다면 어차피 못 받지 않나?
        - 물론 그래도 각각의 예외는 구분해서 로그를 남겨야 하지 싶음
    - 로그를 남기던, 자동 재시도를 구현하던 해야하지 않을까? (`@Retryable` 같은 걸 쓰고 maxAttempts 를 정해준다거나 훔)
    - 개인적으로는 SAGA 패턴을 적용하는게 좋지 않을까? 싶음
        - 전체 작업 중에서 일부에서 실패가 발생하면 앞서 수행된 모든 로컬 트랜잭션을 롤백하는 목적의 보상 트랜잭션을 실행
            - 외부 API POST 요청 수행하고 OK -> Created Event pub
            - Created Event 수신 후 GET 요청 수행하고 OK -> Retrieved Event pub
            - Retrieved Event 수신 후 DB 저장 수행 -> 성공하면 DBStored Event pub
            - 각각의 과정에서 Retrieved Failed Event 나 DBStored Failed Event 가 pub 된다면, 보상 트랜잭션으로 앞서 외부 API POST 요청을 취소 


- 고민을 했던 거는 이 지점인데
    - 외부 API POST가 먼저 진행 되고 201 Created 된 상태에서 중간에 있는 작업이 실패
        - 앞서 있던 POST를 취소하기 위해 DELETE 요청 보내기
           - 장점: 데이터 정합성을 즉시 보장, 롤백 과정에서 완전히 삭제되므로 관리가 용이
           - 단점: 롤백 과정에서 추가적인 네트워크 요청이 필요, DELETE 요청 자체가 실패할 수도 있음.
       - 실패를 기록한 후 다시 DB 저장
            - 장점 : 일시적인 오류가 발생했을 때 시스템이 자동으로 복구할 수 있으며, 롤백 과정에서 추가적인 네트워크 요청이 필요하지 않을 수 있음(경우에 따라?)
            - 단점: 데이터 정합성을 완전히 보장하기 어렵고, 재시도 로직이 복잡해질 수 있음. 또한, 재시도에 실패할 경우 추가적인 관리가 필요할 수 있음
    - 결론은 관리 포인트를 늘리지 않기 위해 DELETE를 보내기로 했다. 땅땅!!      
     


<details>
    <summary> 개념 정리 </summary>
    
- 트랜잭션은 보통 다중 객체에 대해 다중 연산을 하나의 실행 단위로 묶는 메커니즘으로 이해
- 단일 객체 연산과 다중 객체 연산
    - 다중 객체 연산은 어떤 읽기 연산과 쓰기 연산이 동일한 트랜잭션에 속하는지 알아낼 수단이 있어야 한다.
        - RDB에서 보통 클라이언트와 DB 서버 사이의 TCP 연결을 기반으로 한다.
        - 반면 비관계형 DB는 이런 식으로 연산을 묶는 방법이 없는 경우가 많다. 다중 객체 API가 있더라도(예를 들어, 키-값 저장소는 한 연산 내에서 여러 키를 갱신하는 다중 put(multi-put) 연산을 제공할 수 있다) 반드시 트랜잭션 시맨틱을 뜻하지 않는다. ← 일부가 실패할 수 있음
    - 대부분 보편적으로 한 노드에 존재하는 단일 객체 수준에서 원자성과 격리성 제공을 목표로 한다.
        - 원자성은 장애 복구용 로그를 써서 구현할 수 있고 격리성은 각 객체에 잠금을 사용해 구현할 수 있다.
        - DB에 따라 더 복잡한 원자적 연산을 제공하기도 한다. 비슷하게 CAS 연산을 제공하기도 함
        - 단일 객체 연산은 갱신 손실(lost update)을 방지해 유용하나 일반적으로 쓰이지는 않음

1. ACID
    - Atomicity
        - Abortability : 특정 트랜잭션을 abort하고 해당 트랜잭션에서 기록한 모든 내용을 취소하는 능력
            - 즉, 트랜잭션이 abort 됐다면 애플리케이션에서 이 트랜잭션이 어떤 것도 변경하지 않았음을 알 수 있고 이를 보장하는게 원자성
        - 동시성과는 무관한 개념. 동시성은 여러 프로세스가 동시에 같은 데이터에 접근할 때 발생하는 문제로 ACID에선 Isolation 개념에 가까움
    - Consistency
        - 데이터에 관한 어떤 불변하는(invariant) 선언이 지켜지는 것을 일관성 있다고 표현
            - 예를 들어, 회계에서 대변과 차변의 합은 항상 같아야 하는 것 같은 게 불변식에 해당
        - 즉, 트랜잭션이 커밋 됐다고 하면 해당 불변식을 항상 만족한다고 확신할 수 있음
        - Atomicity, Isolation, Durability 같은 경우 데이터베이스의 속성이지만 Consistency는 사실상 애플리케이션의 속성이다. 물론 데이터베이스에서도 외래키 같은 제약 조건이 있긴 하나 그 외 비즈니스 로직 상 불변식은 애플리케이션에서 다뤄진다. 이 때문에 C는 실제로 ACID에 속하지 않는다.
    - Isolation
        - 동시에 실행되는 트랜잭션은 서로 격리된다. 트랜잭션은 다른 트랜잭션을 방해할 수 없다.
        - 직렬성이라고 표현되기도 하나 직렬성 격리는 성능 손해를 동반되어 현실에선 거의 사용 되지 않고 그 보다 약한 snapshot isolation이 사용된다.
    - Durability
        - 트랜잭션이 성공적으로 커밋 됐다면 하드웨어 결함이 발생하거나 DB가 죽더라도 트랜잭션에서 기록한 모든 데이터는 손실되지 않는다는 보장
        - 복제 기능이 있는 DB에서 Durability는 데이터가 성공적으로 다른 노드 몇 개에 복사 됐다는 것을 의미한다. 지속성을 보장하려면 DB는 트랜잭션이 성공적으로 커밋 했다고 말하기 전에 복제가 완료될 때까지 기다려야 한다.
2. Isolation Level
3. Serialization
4. Spring의 `@Transactional`
    - proxy 생성 
        - AOP를 사용해 `@Transactional`를 붙인 메소드를 찾아 프록시를 적용
        - JDK dynamic proxy나 CGLIB를 사용
    - 메소드 호출 감지
        - 프록시 객체는 실제 객체와 동일한 인터페이스를 구현
        - 클라이언트 코드에서는 프록시를 사용해 메소드 호출할 때 특별한 코드 변경이 필요하지 않음
        - 프록시는 호출을 감지하고 트랜잭션 관리 코드를 실행한 다음 실제 메소드를 호출
    - 트랜잭션 시작
        - TransactionManager를 사용해 트랜잭션 시작. 
        - TransactionManager는 DataSourceTransactionManager, HibernateTransactionManager 등 다양한 구현체를 제공하며, 각각 DB 연결 또는 Hibernate 세션과 같은 리소스와 연계해 트랜잭션을 처리
    - 메소드 실행 : 트랜잭션 시작 시 프록시는 실제 메소드를 호출하고 결과 반환
    - 트랜잭션 커밋/롤백 
        - 메소드 실행이 성공적으로 완료되면 프록시는 TransactionManager를 사용해 트랜잭션을 커밋함. 만약 메소드 실행 중 예외가 발생하면, 프록시는 TransactionManager를 사용해 롤백함.
    = 결과 반환 : 프록시는 메소드 호출 결과를 클라이언트 코드에 반환


    - TransactionInterceptor:
        - TransactionInterceptor는 AOP 어드바이스(advice)로 메소드 호출을 가로채서 트랜잭션 로직을 적용 
        - @Transactional이 붙은 메소드가 호출되면, TransactionInterceptor는 트랜잭션 시작, 커밋, 롤백 등의 작업을 수행

    - TransactionAspectSupport
        - TransactionInterceptor는 내부적으로 TransactionAspectSupport 클래스를 사용. 
        - 실제 트랜잭션 처리와 관련된 작업을 수행

    - TransactionAttributeSource
        - TransactionAttributeSource는 `@Transactional` 어노테이션에 지정된 속성값을 저장함. 
        - 예를 들어, 트랜잭션 전파 속성, 격리 수준, 읽기 전용 여부, 롤백 규칙 등과 같은 트랜잭션 관련 설정

    - PlatformTransactionManager:
        - PlatformTransactionManager는 실제 트랜잭션 처리를 담당하는 인터페이스
        - 데이터베이스와의 트랜잭션을 처리하는데 필요한 동작을 정의 
        - 예를 들어, DataSourceTransactionManager, HibernateTransactionManager, JpaTransactionManager 등

    - AOP 프록시 생성:
        - Spring AOP는 프록시를 사용하여 메소드 호출을 가로채서 `@Transactional`이 붙은 메소드가 있는 클래스에 대한 프록시를 생성함 
        - 이 프록시는 원래의 메소드 호출 전후에 트랜잭션 처리를 수행할 수 있도록 TransactionInterceptor를 적용함. Spring은 기본적으로 JDK 동적 프록시를 사용하나 CGLIB와 같은 다른 프록시 생성 라이브러리를 사용할 수도 있음


</details>
