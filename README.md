# cna-bookstore

## User Scenario
```
* 고객은 주문과 취소만 가능하며, 주문의 수정은 할 수 없다.
* 고객이 주문을 생성할 때 고객정보와 Book 정보가 있어야 한다.
  - Order -> Customer 동기호출
  - Order -> BookInventory 동기호출
* 주문 시에 재고가 없더라도 주문이 가능하다.
  - 주문 상태는 “Ordered”
* 주문 취소는 Delivery가 시작되기 이전에만 가능하다.
* 배송을 시행하는 외부 시스템(물류 회사 시스템)이 배송 단계별로 상태는 변경한다.
* 주문 취소되더라도 고객은 MyPage에서 주문 이력을 모두 조회할 수 있다.
```

## Cloud Native Application Model
![Alt text](cna-bookstore.PNG?raw=true "Optional Title")
