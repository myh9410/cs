> Redis의 자료구조와 싱글스레드의 장,단점

### 자료구조
- Key, Value 형식으로 Value에 String, List, Set, Sorted Set 등을 사용할 수 있다.

### Single Thread로의 장,단점
- 장점 : Single Thread라서 Context-Switching이 발생하지 않는다. 이로 인한 DeadLock 발생이 없다.
- 단점 : Full Scan 동작 시 다른 명령어 처리가 불가능하여, 응답속도 저하로 이어진다.