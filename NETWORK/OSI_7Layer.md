> OSI 7 계층에 대해 알아보자

### Application Layer - HTTP, SMTP, FTP 등
> 응용 프로그램에 대한 계층

### Presentation Layer
> 데이터 표현에 대한 계층

Application Layer에서 받은 데이터/ 보낼 데이터를 변환/압축 처리

### Session Layer
> 세션에 대한 계층


세션 생성, 유지, 종료 등을 처리

### Transport Layer - TCP, UDP
> 송/수신자의 논리적 연결에 대한 계층

데이터 단위 : Segment  
신뢰성 있는 연결을 유지  
연결을 생성하고, 데이터를 받았는지 확인한다.

### Network Layer - 라우터

데이터 단위 : Packet  
IP를 활용하여 경로/목적지 설정

### DataLink Layer - 브릿지, 스위치
 
데이터 단위 : Frame  
오류에 대한 재전송이 가능 (오류 검출, 데이터 재전송)

### Physical Layer - 케이블, 허브 등
> 물리적으로 데이터가 전송되는 계층