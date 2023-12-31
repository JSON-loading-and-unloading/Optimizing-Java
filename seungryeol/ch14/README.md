<h1>고성능 로깅 및 메시징</h1>
자바는 개발자가 신경 써서 처리해야 하는 저수준 관심사를 대폭 줄임으로써 생산성을 엄청나게 끌어올렸다.</br>
고수준 언어로 추상화해서 개발자 생산성은 향상시켰지만, 대신 저수준 제어를 포기하고 성능 자체를 직접 다룰 수 없는 단점을 남겼다.</br></br>


저지연, 고성능 시스템에서 핵심적인 고려 사항 두 가지는 로깅, 메시징이다.</br>

<h2>로깅</h2>

로거 중 바람직하지 않는 안티패턴

<h4>10년짜리 로거</h4>
누군가 이미 로거를 잘설정해놨다. 뭐 하러 다시 만드나? 그냥 편하게 갖다 쓰면 되지.

<h4>프로젝트 전체 로거</h4>
누군가 프로젝트 각 파트마다 따로 로거를 재구성하지 않아도 되게끔 로거를 감싸놓았다.

<h4>전사 로거</h4>
누군가 전사적으로 사용 가능한 로거를 만들었다.

<h3>로깅 벤치마크</h3>

<h4>로깅 없음</h4>
현재 로거가 켜져 있고 어떤 한계치 이하로 메시지가 로깅되고 있는 상태에서 무동작로그의 비용을 측정하는 벤치마크 테스트이다.</br>

<h4>Logback 포맷</h4>
```
14:18:17.635 [name of thread] INFO c.e.NameOfLogger - Log message
```
</br>
<h4>java.util.logging 포맷</h4>
```
Feb 08, 2017 2:09:19 PM com.example.NameOfLogger nameOfMethod
INFO: Log message
```
</br>
<h4>Log4j 포맷</h4>
```
2017-02-08 14:16:29,651 [Name Of Thread] INFO com.example.NameOfLogger - message
```
</br>
<h4>측정</h4>

이미지
</br>
Logback이 Log4j보다 약간 더 빠르게 나왔다.</br>

<h4>로거 결과</h4>
실행 시간 측면에서 대체로 Logback 성능이 가장 좋고, 자바 유틸 로거가 제일 나빴다..</br></br>

로깅 프레임워크가 생성하는 엄청난 양의 가비지도 잘 따져보아야 한다.</br>
로깅하느라 소비한 cpu 시간만큼 핵심 업무를 병렬 처리할 기회를 잃어버리기 때문이다.</br>
로깅 라이브러리의 설계와 작동 원리 역시 직선적인 마이크로벤치마크 실행 결과만큼 중요하다.</br>

<h2>성능에 영향이 적은 로거 설계하기</h2>
로깅은 모든 애플리케이션의 필수 컴포넌트이지만, 저지연 애플리케이션에서 로거는 비즈니스 로직 성능에 병목 현상을 초래해선 안된다.</br></br>

Log4j 2.6으로 실행하면 같은 시간 동안 GC사이클이 한 번도 발생하지 않았다.</br>
Log4j 2.6에서 성능이 향상된 비결은, 각 로그 메시지마다 임시 객체를 생성했던 로직을 객체를 재사용하는 방향으로 수정한 것이다.</br>
객체 풀 패턴을 실천한 전형적인 사례이다.</br>
Log4j 2.6은 ThreadLocal 필드를 이용해 스트링 -> 바이트 변환 시 버퍼를 재사용하는 식으로 객체를 재사용한다.</br></br>

Logj4는 임시 배열을 생성해 로그문에 전달되는 매개변수를 담고 가변인수를 사용해 할당 횟수를 줄인다.</br>
Log4j를 SLF4J로 감싸면 퍼사드가 매개변수를 2개만 지원하기 때문에, 가비지-프리한 방식을 응용하거나 Log4j2 라이브러리를 직접 사용해서 코드 베이스를 리팩터링할 필요가 없다.</br>

<h2>리얼 로직 라이브러리를 이용해 지연 줄이기</h2>

로얼 로직은 저수준 세부의 이해가 고성능 설계에 영향을 미친다는 기계 공감 접근 방식을 주장한 마틴 톰슨이 설립한 영구 회사이다?</br>

<h2>아그로나</h2>
아그로나 프로젝트는 저지연 애플리케이션 전용 구성 요소를 담아놓은 라이브러리이다.</br></br>

아그로나는 진정한 저지연 애플리케이션 라이브러리 세트를 제공한다.</br>
=> 표준 라이브러리만으로 유스케이스를 충족시키기 어렵다는 사실이 밝혀졌다면 아그로나 라이브러리를 한 번 검토해보자!</br>

<h4>버퍼</h4>
자바에는 다이렉트/논다이렉트 버퍼를 추상화한 ByteBuffer클래스가 있다.</br>
다이렉트 버퍼는 자바 힙 밖에 있기 때문에 온-힙 버퍼보다 할당/해제율은 낮은  편이다.</br>
다이렉트 버퍼의 장점은 중단 단계의 매핑 없이 직접 구조체에 명령어를 실행하는 것이다</br></br>

ByteBuffer는 일반화한 유스케이스가 가장 큰 문제로, 버퍼 타입별로 최적화를 적용할 수 없다.</br>
가령, ByteBuffer는 아토믹 연산을 지원하지 않으므로 생산자/소비자 방식의 버퍼를 구축할 때 제약이 따른다.</br>
또 ByteBuffer를 사용하려면 매번 다른 구조체를 감쌀 때마다 하부 버퍼를 새로 할당해야한다.</br>
아그로나는 복사를 지양하며 저마다 독특한 특정을 지닌 버퍼를 네 가지 지원한다.</br></br>

 - DirectBuffer 인터페이스 : 버퍼에서 읽기만 가능하며 최상위 상속 계층에 위치한다.
 - MutableDirectBuffer 인터페이스 : DirectBuffer를 상속하며 버퍼 쓰기도 가능하다.
 - AtomicBuffer 인터페이스 : MutableDirectBuffer를 상속하며 메모리 액세스 순서까지 보장한다.
 - UnsafeBuffer 클래스 : Unsafe를 이용해 AtomicBuffer를 구현한 클래스이다.

아그로나 버퍼 클래스의 상속 계층도</br>

이미지
</br>
아그로나 버퍼를 이용하면 다양한 get메서드를 통해 하부 데이터를 가져올 수 있다.</br>
put메서드를 이용하면 버퍼의 특정 위치에 long값을 넣을 수 있다.</br>
경계 검사 기능은 설정/해제가 가능하므로 불필요한 코드는 JIT 컴파일러로 최적화하면서 들어낼 수 있다.</br>

<h4>리스트,맵,세트</h4>
아그로나는 int 또는 long형 배열에 기반한 리스트 구현체를 여럿 제공한다.</br>
아그로나 ArrayListUtil을 이용하면 리스트 순서는 안 맞지만 ArrayList에서 신속하게 원소를 제거할 수 있다.</br>
아그로나 맵, 세트 구현체 키/값을 해시 테이블 자료 구조에 나란히 저장된다.</br>
키와 충돌하면 다음 값은 해시 테이블의 해당 위치 바로 다음에 저장된다.</br>
(동일한 캐시 라인에 있는 기본형 매핑을 재빠르게 액세스할 때 좋은 자료구조이다.)</br>

<h4>큐</h4>

아그로나의 동시성 패키지에는 큐, 링 버퍼를 비롯해 쓸만한 자료 구조 및 동시성 유틸리티가 있다.</br></br>

아그로나 큐는 표준 java.util.Queue 인터페이스를 준수하므로 표준 큐 구현체 대신 쓸 수 있고, 순차 처리용 컨테이너 지원 기능이 부각된</br>
org.agrona.concurrent.Pipe인터페이스도 함께 구현되어 있다. </br>
특히, Pipe는 원소를 카운팅하고, 수용 가능한 최대 원소 개수를 반환하고,</br>
원소를 비우는 작업을 지원하므로 큐를 소비하는 코드와 원활하게 상호작용할 수 있다.</br>
큐는 모두 락-프리하고 Unsafe를 사용하므로 저지연 시스템에 안성맞춤이다!!!</br></br>


큐 분리 구현체 종류</br>

<h5>OneToOneConcurrentArrayQueue</h5>
헤드는 큐에서 poll() 또는 drain()할 때에만, 테일은 put()할 때에만 업데이트할 수 있다.</br>
이 모드를 선택하면 나머지 두 가지 큐에서 꼭 필요한, 부수적인 조정 체크를 하느라 쓸데없이 성능 누수를 유발할 일이 없다.</br>

<h5>ManyToManyConcurrentArrayQueue</h5>
생산자가 다수일 경우에는 테일 위치를 업데이트할 때 부가적인 제어 로직이 필요하다.</br>
while루프에서 Unsafe.compareAndSwapLong를 사용하면 꼬리가 업데이트될 때까지 큐 테일을 안전하게, 락-프리하게 업데이트할 수 있다.</br>

<h5>ManyToOneConcurrentArrayQueue</h5>
생성자, 소비자가 모두 다수일 경우, 머리/테일 양쪽을 업데이트해야 한다.</br>
이 정도 수준으로 조정/제어하려면 compareAndSwap을 감싼 while 루프가 필요하다.</br>

<h4>링 버퍼</h4>

아그로나가 제공하는 org.agrona.concurrent.RingBuffer는 프로세스 간 통신용 바이너리 인코딩 메세지를 교환하는 인터페이스이다.</br>
RingBuffer는 DirectBuffer를 이용해 메시지 오프-힙 저장소를 관리한다.</br>
소스 코드에 주석으로 포함된 다음 아스키 아트 덕분에 메시지가 RecordDescriptor 자료 구조에 저장된다는 사실을 알 수 있다.</br></br>

아그로나에 내장된 링 버퍼 구현체는 OneToOneConcurrentArrayQueue, ManyToOneConcurrentArrayQueue 두 가지이다.</br>
쓰기 작업은 소스 버퍼를 전달받아 그 메시지를 별도의 버퍼에 써넣는 반면, 읽기 작업은 메시지 핸들러의 onMessage() 메서드로 콜백된다.</br>
ManyToOneConcurrentArrayQueue에서 여러 생산자가 쓰기하고 있는 상황에서 Unsafe.storeFence() 메서드를 호출하면 수동으로 메모리 동기화를 통제할 수 있다.</br>

<h2>단순 바이너리 인코딩</h2>

단순 바이너리 인코딩(SBE)는 저지연 성능에 알맞게 개발된 바이너리 인코딩 방식으로, 금융시스템에서 쓰이는 FIX 프로토콜에 특화되어 있다.</br></br>

버퍼는 아그로나에서 빌려쓴다.</br>
SBE는 GC를 유발하지 않고 메모리 액세스 같은 문제를 최적화하지 않고도 효율적인 자료 구조를 통해 저지연 메시지를 전달할 수 있다.</br></br>

저지연 애플리캐이션의 목표는 수단과 방법을 가리지 않고 애플리케이션 성능을 최대한 짜내는 것이다.</br>
그래서 거래 애플리케이션의 임계 경로를 거치는 지연을 줄이고자 경쟁 거래사 간에 군비 경쟁을 방불케 하는 노력이 치열하다.</br></br>

<h5>카피-프리, 네이티브 타입 매핑</h5>
복사는 비용이 든다. 조그마한 객체는 별로 안비싸지만, 크기가 커질수록 복사 비용도 함께 증가한다.</br>
SBE의 카피-프리 기술은 중간 버퍼를 쓰지 않고 메시지를 인코딩/디코딩하도록 설계됐다.</br>

하지만, 하부 버퍼에 직접 쓰는 작업은 설계 비용이 든다. 버퍼에 집어넣지 못할 정도로 큰 메시지를 지원할 수 없기 때문에 메시지를 조각조각 나누어 다시 조립하는 프로토콜을 구축해야</br>
이런 메시지도 지원 가능하다.</br>

<h5>정상 상태 할당</h5>
저지연 애플리케이션 설계 시 자바의 객체 할당 방식은 문제가 된다.</br>
할당 작업 자체도 cpu사이클을 소모하지만 사용을 마친 후 객체를 지우는 것도 문제이다</br>
GC는 STW, 즉 중단을 자주 일으키다.</br>
SBE는 하부 버퍼에 플라이트웨이트 패턴을 사용하므로 할당-프리한다.</br>

<h5>스트리밍/단어 정렬 액세스</h5>
자바에서 메모리 액세스는 범접할 수 없는 대상이다.</br>
자바 배열은 보통 레퍼런스 배열 형태라서 메모리 순차 읽기는 불가능하다.</br>
SBE는 메시지를 진행 방향으로 인코딩/디코딩하도록 설계되어 있어서 정확하게 단어를 정렬할 수 있는 틀이 잡혀있다.</br>

<h2>에어론</h2>
에어론은 SBE, 아그로나에 기반한 툴이다.</br>
에어론은자바 및 C++용도로 개발된, UDP 유니캐스트, 멀티캐스트, IPC 메시지를 전송하는 수단이다.</br></br>

에어론은 같은 머신에서, 또는 네트워크를 넘나들며 IPC를 통해 서로 소통할 수 있게 해주는 것들을 망라한, 일반적인 메시지 프로토콜이다.</br>
최고의 처리율을 지향하는 에어론은 지연을 예측 가능한 방향으로 가장 낮게 유지하는 것을 목표한다.</br>

<h4>발행자</h4>

![KakaoTalk_20231011_195836703_02](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/e82befa3-64f1-4058-8762-66a5f22d61d6)

- 미디어 : 에어론이 통신하는 매개체이다.</br>
- 미디어 드라이버 : 미디어와 에어론 사이의 연결 통로이다. 원하는 전송 구성을 세팅해 통신할 수 있다.</br>
- 감독자 : 전체 흐름을 관장한다. 버퍼를 설정하거나 새 구독자/발행자 요청을 리스닝하는 일을 한다. 또, NAK를 감지해 재전송을 준비하기도 한다.</br>
- 송신자 : 생산자로부터 데이터를 읽어 소켓으로 전송한다.</br>
- 수신자 : 소켓에서 데이터를 읽고 해당 채널/세션으로 내보낸다.</br></br>

  <h5>구독자</h5>

  미디어 드라이버를 가져와 에어론 클라이언트로 접속하는 과정은 발행자와 비슷하다</br>
  위 코드는 메시지 수신 시 작동시킬 콜백 함수를 동록한다.</br>

 
![KakaoTalk_20231011_195836703_01](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/438ba1c7-f3ce-4e89-8509-0edd32bf75b5)

  <h3>에어론의 설계 개념</h3>

  <h4>전송 요건</h4>
  에어론은 OSI 4 전송 계층에서 메시징을 하므로 반드시 준수해야 할 요건들이 있다.</br>
  - 정렬</br>
  - 신뢰성</br>
  - 배압</br>
  - 혼잡</br>
  - 다중화</br>

  <h4>지연 및 애플리케이션 원칙</h4>
  - 정상 상태에서 가비지-프리 실현</br>
  - 메시지 경로에 스마트 배칭 적용</br>
  - 메시지 경로의 락-프리 알고리즘</br>
  - 메시지 경로의 논블로킹 I/O</br>
  - 메시지 경로의 비예외 케이스</br>
  - 단일 출력기 원칙을 적용</br>
  - 공유 안 하는 상태가 더 좋다</br>
  - 쓸데없이 데이터를 복사하지 말라</br></br>

  
  <h4>내부 작동 원리</h4>

에어론은 최대한 깔끔하고 단순한 방식으로 자료 구조에 메시지 시퀀스를 생성한다.</br>


![KakaoTalk_20231011_195836703](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/80222797-a50b-4db8-959f-d0f43750965d)

테일 포인터는 최종 메시지가 쓰인 지점을 찾아가는 용도로 쓰인다.</br>
테일 포인터는 파일 내부에 메시지 공간을 예약한다.</br>
테일 증분 작업은 아토믹하므로 출력기는 자기 영역의 처음과 끝이 어디인지 잘 알고 있다.</br>
=> 다중 출력기가 락-프리하게 파일을 업데이트할 수 있고 파일 쓰기 프로토콜을 아주 효율적으로 작동시킬 수 있다.</br></br>

헤더는 제일 마지막으로 아토믹하게 파일에 출력되므로 그 존재 여부로 메시지가 완성됐음을 아므로 작업이 다 끝났는 지 알 수 있다.</br></br>


파일이 한 없이 커지는 것을 막기 위해,</br>
액티브, 더티, 클린 세 파일로 두어 해결한다.</br>
액티브는 현재 쓰고 있는 파일, </br>
더티는 이전에 쓰인 파일,</br>
클린은 바로 다음에 쓸 파일이다,</br>
큰 파일때문에 지연되지 않도록 계속 파일을 순환시킨다.</br></br>

테일이 액티브 파일 끝에서 밀려나면, 삽입 프로세스는 파일의 나머지 부분을 채우고</br>
메세지를 클린 파일에 쓴다.</br>
트랜잭션 로그는 더티 파일에서 꺼내 아카이빙할 수 있다.</br>



   
  



