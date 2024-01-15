> 3 way와 4 way handshake 동작에 대해 알아보자

### 3 WAY Handshake
- 3 Way
    1. SYN → (CLIENT : CLOSED → SYN_SENT)
        - seq = x 값을 담는 연결 요청을 보낸다
    2. SYN + ACK ← (SERVER : LISTEN → SYN_RCV)
        - seq = y, ack = x+1 값을 담는 SYN + ACK 패킷을 보냄
    3. ACK → (CLIENT : SYN_SENT → ESTABLISHED)
        - ack = y+1 값을 서버로 다시 보냄 (SERVER는 ack를 받으면 ESTABLISHED 상태가 됨)

### 4 WAY Handshake
- 4 Way
    1. FIN(+ACK) → (CLIENT : ESTABLISHED → FIN_WAIT_1)
        - ACK를 포함한 FIN 패킷을 통해 close() 요청 호출하여 접속 끊는 요청
    2. ACK ← (SERVER : ESTABLISHED → CLOSE_WAIT)
        - CLOSE_WAIT 상태에서 잔여 미전송 데이터가 있다면 마무리 후 close() 호출
        - CLIENT는 ack를 받으면 FIN_WAIT_2 상태가 됨
    3. FIN ← (SERVER : CLOSE_WAIT → LAST_ACK)
        - 미전송 데이터 전송 완료 후, FIN 패킷을 발송하여 최종 CLOSED를 기다리는 LAST_ACK 상태가 됨
    4. ACK → (CLIENT : FIN_WAIT_2 → TIME_WAIT → CLOSED)
        - TIME_WAIT이란 close() 이후에도 지연, 재전송 등의 이슈 해결을 위해 (default: 4분) 정도 응답 대기
            - TIME_WAIT이 짧으면, 미처리되는 패킷 발생 가능성 증가, 데드락 가능성 존재
        - SERVER는 ACK를 받아서 CLOSED 상태로 바뀜
        - CLIENT는 TIME_WAIT 이후 CLOSED 상태로 바뀜