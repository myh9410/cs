> Multi-Thread 환경에서 데이터 동기화를 보장할 수 있는 방법에 대해 알아보자

### volatile - target: value
> java에서 변수는 CPU Cache와 Main Memory에 저장된다.  
> volatile은 Main Memory를 통해 변수의 값을 관리하는 방식이다.
- Multi Thread 환경에서 Cache영역의 공통변수를 참조하여 사용하면, 불일치할 수 있는 문제를 야기할 수 있다.
- Main Memory를 사용하는 것은 변수 접근의 속도가 상대적으로 느리다는 단점이 있지만, 데이터의 동기화를 보장할 수 있다는 장점이 있다.

### synchronized - target: method
> blocking을 사용하여, Multi Thread 환경에서 데이터 동기화를 보장한다.  
> 특정 Thread가 synchronized 영역 접근 시, 다른 Thread들은 block에 막혀 대기하게 되는데 이는 성능 이슈를 야기할 수 있다.


### Atomic - target: value
> Atomic은 CAS(Compare-And-Swap) 기반으로 non-blocking하며, 데이터의 동기화를 보장한다.
- CAS란?
  > 기존 값, 변경할 값 2개의 인자를 활용하여 기존값이 메모리의 값과 같다면 변경할 값을 반영하고  
  > 기존값이 메모리의 값과 다르다면 변경할 값을 반영하지 않는다 (이미 정상 반영된 것으로 보는 것 같음)