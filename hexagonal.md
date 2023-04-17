### 헥사고날 아키텍처

- 재고 관리 프로그램을 만든다고 하고 패키지 구조를 잡아보자.
- 패키지 구조

```sql
com.example.inventory
    ├── application
    │   ├── ports
    │   │   ├── in
    │   │   │   └── ManageInventoryUseCase.java
    │   │   └── out
    │   │       └── ProductRepositoryPort.java
    │   └── services
    │       └── InventoryService.java
    ├── config
    │  
    ├── domain
    │   ├── entities
    │   │   └── Product.java
    │   └── repositories
    │       └── ProductRepository.java
    ├── infrastructure
    │   ├── adapters
    │   │   └── ProductRepositoryAdapter.java
    │   └── persistence
    │       └── jpa
    │           └── JpaProductRepository.java
    └── presentation
        └── rest
            └── ProductController.java
```

