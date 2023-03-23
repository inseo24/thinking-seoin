### Transaction

1. ACID
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




