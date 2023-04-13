### AWS RDS MySQL DB Table 날려 먹은 썰

- 인텔리제이 얼티밋 쓰면서 좋다고만 생각했지 이렇게 어이 없는 실수를 하다니
- 도커로 로컬에 띄운 디비 테이블을 날린다는게 dev 에 있던 테이블을 날렸다.
- 사고난 직후 바로 옆자리 동료에게 1분만에 알렸고, 3분만에 바로 데브옵스 시니어 분께 구두로 현 상태에 대해 전달 드렸다.
  - 해결은 사실상 시니어 분이 다 해주시고 옆에서 조마조마 지켜보다가 복원 후 API 동작에 문제 없는지 체크했다.


### 해결 방법

- AWS RDS MySQL의 경우 '특정 시점으로 복원' 기능을 제공함
- 사고 난 직후, 바로 특정 시간을 기준으로 특정 시점 복원 시도하심
  - 해당 시점의 DB snapshot 생성(생성 요청 -> 생성 시작 -> 생성 완료까지 대략 15분이 안 걸렸다)
  - 특정 시점 복원(새로운 클러스터, DB 생성됨. default는 현재 클러스터, DB와 동일 사양)
    - 복원 요청 -> 복원 시작 -> 복원 완료까지 20분이 안 걸림
  - 복원 후 DB는 public access config를 추가로 함

- 복원된 DB에 접근해 DB Export 후 복원 시도
  - 대략 MySQL workbench에 있는 export 기능을 사용해 복원 시도함 <- 그냥 하면 권한 에러 발생함
  - 다행이 테이블이 많지 않은 편이라 스키마에서 테이블 생성만 하고 data 쪽에서 insert query 실행