> JVM의 구조와 프로그램 실행 과정에 대해 알아보자

## JVM 구조
- Class Loader
    - .class 파일들을 Runtime Data Area에 배치한다.
        - Loading - 메모리에 로드
        - Linking - 검증, 할당 등
        - Initialization - 클래스 변수 초기화
- Execute Engine
- Garbage Collector
- Runtime Data Area (JVM 메모리 영역 구조)
    - METHOD AREA
    - HEAP AREA
        - YOUNG (Minor GC)
            - Eden
                - 새로 생성된 객체
                - 주기적인 GC 후, 살아남은 객체들을 Survivor1으로 보냄
            - Survivor0, Survivor1
                - 최소 1번 이상의 GC 이후 살아남은 객체
                - Survivor0과 Survivor1 중 둘 중 하나는 반드시 비어 있어야 함
                    - Survivo0과 1중 하나에 살아남은 객체를 몰아놓고 GC 수행하기 때문
        - Old (Major GC)
    - JVM Language Stacks
    - PC Registers
    - Native Method Stack

---

## 프로그램 실행 과정
1. javac(컴파일러)가 .java 파일을 .class(바이트 코드)로 변환한다.
2. JVM의 Class Loader가 .class를 JVM 메모리에 적재한다.
3. Execute Engine이 .class 파일들을 실행시킨다.
   - Interpreter
     - .class 파일을 하나씩 읽으면서 실행. 파일 개별의 실행은 빠르지만, 전체적인 속도는 느리다.
   - JIT (Just-In-Time Compiler)
     - .class 파일 전체를 컴파일하여 실행. 전체적인 속도가 빠르다.

---

### JVM 옵션
- 튜닝
  - TPS
  - 응답시간 (네트워크 간 연결 시간 + 트랜잭션 시간)


- 성능 튜닝의 목적
  - 2가지 관점에서 튜닝을 진행한다.
    - Throughput
      - 일정 시간 내에 최대한 많은 요청을 처리하는 것을 목적으로 한다.
    - Response
      - 요청에 대한 응답속도를 높이는 것을 목적으로 한다.

- GC 알고리즘
  - Parallel GC - 처리량이 중요할때 고려
  - CMS GC - 응답시간이 중요할때 고려
  - G1 GC - JAVA 9 이후, Default GC 방식

- Memory 튜닝에 대한 영역별 값
  - Heap (-Xmx)
    - (-Xms : Eden / To / From / Tenured 영역) 
    - Young 영역 (-XX:MaxNewSize)
      - Reserved
      - Eden (-XX:NewSize : Eden / To / From 영역)
      - To
      - From
    - Old 영역
      - Tenured
      - Reserved
  - MetaSpace ( XX:MaxMetaspaceSize )
    - Metaspace ( XX:MetaspaceSize )
    - Reserved

- XX:NewSize
  - Young(New) 영역의 크기 설정을 위한 파라미터
- XX:MaxNewSize
  - 최대 Young(New) 영역의 크기 설정
- XX:NewRatio
  - 전체 Heap 크기 중 Young과 Old 영역의 비율 설정 