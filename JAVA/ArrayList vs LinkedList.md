> ArrayList와 LinkedList의 차이

## ArrayList
- index 기반

## LinkedList
- 각 Node의 head와 tail 기반 ( 각 원소마다 앞,뒤 원소의 위치값을 갖는다. )

## 조회, 삽입, 삭제 성능
- ArrayList
  - get() : O(1)
  - add() : O(1)
  - remove() : O(n)
- LinkedList
  - get() : O(n)
  - add() : O(1)
  - remove() : O(n)

- 조회의 경우, ArrayList가 성능이 더 뛰어나고,
- 중간 데이터에 대한 추가/ 삭제는 LinkedList를 사용하는 것이 좋다.