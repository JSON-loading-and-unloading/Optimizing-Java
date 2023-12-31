<h1>JVM 이야기</h1>

<h2>인터프리팅과 클래스로딩</h2>
</br>
Jvm 인터프리터의 기본 로직 : 평가 스택을 이용해 중간값들을 담아두고 가장 마지막에 실행된 명령어와 독립적으로 프로그램을 구성하는 옵코드를 하나씩 순서대로 처리하는 'while 루프 안의 switch문'이다.
</br>
</br>
자바 실행 과정
</br>
자바 애플리케이션 실행(자바 클래스로딩 메커니즘 구동?) => 가상 머신 프로세스 구동 => 자바 가상 환경 구성 => 스택 머신 초기화 => 클래스파일 실행</br>
** 제어권을 실행하려는 클래스에 넘기려면 가상 머신이 실행이 시작되기 전에 실행하려는 클래스를 로드해야함.</br>
이를 <strong>자바 클래스 로딩 메커니즘</strong>이 관여함.

<h5>자바 클래스로딩 메커니즘</h5>
자바 프로세스 초기화 => 부트스트랩 클래스가 자바 런타임 코어 클래스를 로드 => 확장 클래스 로더가 생김 => 애플리케이션 클래스 로더가 생성<br><br>

#자바 런타임 코어 클래스 : 실행환경을 구성하는 핵심 클래스(ex object클래스, class클래스, system클래스) </br>
% 자바 8이전까지는 rt.jar 파일에서 가져오지만, 자바9 이후부터는 런타임이 모듈화되고 클래스로딩 개념 자체가 많이 달라짐.</br></br>

자바는 프로그램 실행 중 처음 보는 새 클래스를 디펜던시에 로드함.</br>
클래스를 찾지 못한 클래스 로더는 자신의 부모 클래스로더에게 대신 룩업을 넘김</br>
부모의 부로로 거슬러 올라가 결국 부트스트랩도 룩업을 못하면 ClassNotFoundException예외가 남.</br>
따라서, 빌드 프로세스 수립 시 운영 환경과 동일한 클래스패스를 컴파일하는 것이 좋다</br></br>

/***
<h4>자바(Java)런타임 과정</h4>

1.소스 코드 작성: 자바 프로그램을 개발하기 위해 텍스트 편집기 등을 사용하여 자바 소스 코드를 작성합니다. 자바는 객체 지향 프로그래밍 언어로서, 클래스와 메서드 등으로 구성된 소스 코드를 작성합니다.
</br></br>
2.컴파일(Compilation): 작성된 자바 소스 코드를 컴파일러를 통해 바이트 코드(Bytecode)로 변환합니다. 자바 컴파일러(javac)는 소스 코드를 읽어들여 문법 및 오류를 체크하고, 바이트 코드로 변환하여 클래스 파일(.class)을 생성합니다.
</br></br>
3.클래스 로딩(Class Loading): JVM은 실행을 위해 필요한 클래스 파일들을 동적으로 메모리에 로드합니다. 클래스 로더(Class Loader)는 클래스 파일을 찾아서 JVM의 메모리 영역인 메소드 영역(Method Area)에 로드하며, 필요한 클래스들을 연결 및 초기화합니다.
</br></br>
4.바이트 코드 검증(Verification): 클래스 로딩 후, JVM은 바이트 코드의 유효성을 검증합니다. 바이트 코드 검증 단계에서는 메모리 오버플로우, 타입 일치성, 스택 오버플로우 등의 보안 및 안전성을 위한 검사가 수행됩니다.
</br></br>
5.인터프리트(Interpretation) 또는 JIT 컴파일: 유효성이 확인된 바이트 코드는 인터프리터에 의해 한 줄씩 읽혀 실행될 수 있습니다. 또는, Just-In-Time(JIT) 컴파일러에 의해 바이트 코드가 기계어로 변환되어 실행될 수 있습니다. JIT 컴파일은 반복적으로 실행되는 코드를 동적으로 컴파일하여 성능을 향상시킵니다.
</br></br>
6.실행(Execution): 인터프리터 또는 JIT 컴파일에 의해 변환된 바이트 코드가 JVM 상에서 실행됩니다. 프로그램의 흐름에 따라 메서드들이 호출되고, 객체들이 생성되며, 연산이 수행됩니다.
</br></br>
7.가비지 컬렉션(Garbage Collection): JVM은 자동 메모리 관리를 수행하기 위해 가비지 컬렉션을 수행합니다. 가비지 컬렉션은 더 이상 사용되지 않는 객체들을 탐지하고, 해당 객체들을 자동으로 해제하여 메모리를 회수합니다.
</br></br>
8.종료: 프로그램이 실행을 마치거나, 명시적으로 종료되면 JVM은 실행을 종료하고 할당된 자원들을 해제합니다.
</br></br>
***/</br></br>

<h2>바이트코드 실행</h2>

자바 소스 코드  --------------------->   바이트 코드(.class파일)</br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  javac(컴파일)</br>

⁂바이트 코드는 컴퓨터 아키텍처의 지배를 받지 않으므로 이식성이 좋아 개발을 마친 소프트웨어는 JVM 지원 플랫폼 어디서건 실행할 수 있고 자바 언어에 대해서도 추상화 돼있다.</br></br>
![KakaoTalk_20230707_160101627](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/25ddcd97-16d5-4415-8adc-8e1ad760c40e)
(클래스 파일 해부도)

![KakaoTalk_20230707_160101627_01](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/9eaeea98-c855-4a7e-b325-638e99f19506)
(쉽게 암기하는 방법)


<h2>핫스팟 입문</h2>

자바 프로그램은 바이트코드 인터프리터가 가상화한 스택 머신에서 명령어를 실행하며 시작된다.</br>
⁂프로그램이 성능을 최대로 내려면 네이티브 기능을 활용해 CPU에서 직접 프로그램을 실행시켜야한다.</br>

핫스팟은 인터프리티드 바이트 코드에서 네이티브 코드로 컴파일한다.(JIT컴파일)</br>

<strong>핫스팟</strong>
 - 애플리케이션을 모니터링하면서 가장 자주 실행되는 코드 파트를 발견해 JIT 컴파일을 수행
 - 위를 수행하는 동안 미리 프로그래밍 한 추적 정보가 취합되면서 더 정교하게 최적화 할 수 있다.
 - 특정 메서드가 어느 한계치를 넘어가면 프로파일러가 특정 코드 섹션을 컴파일/최적화한다.
</br>
프로필 기반 최적화(PGO)를 응용하는 환경에서는 대부분의 AOT플랫폼에서 불가능한 방식으로 런타임 정보를 활용할 여지가 있으므로 동적 인라이닝 또는 가상 호출 등으로 성능을 개선할 수 있다.</br>

AOT : AOT(Ahead-of-Time) 컴파일은 프로그램을 실행하기 전에 전체 소스 코드를 기계어로 번역하는 컴파일 방식입니다.</br>
( 일반적으로 JIT 컴파일은 프로그램 실행 중에 코드를 동적으로 컴파일하여 실행하는 반면, AOT 컴파일은 미리 컴파일된 코드를 실행하기 때문에 실행 시간에 동적 컴파일 과정이 필요하지 않습니다. 이로 인해 초기 실행 시간이 짧아지고, 프로그램의 반응성이 향상될 수 있습니다.)</br>
PGO : PGO(Profile-Guided Optimization)는 프로그램의 성능을 향상시키기 위해 프로파일 정보를 활용하는 최적화 기술입니다.</br>

<h2>Jvm내부 인터프리터와 jit컴파일러</h2>

인터프리터는 바이트 코드를 읽고 해석하는 과정에서 성능상의 이점이 있는 JIT(Just-in-Time) 컴파일러로 대체될 수 있습니다. </br>
JIT 컴파일러는 프로그램의 실행 중에 바이트 코드를 네이티브 기계 코드로 컴파일하여 실행 속도를 향상시킵니다. </br>
JIT 컴파일러는 자주 실행되는 코드 또는 루프 등을 대상으로 최적화를 수행하며(캐싱),</br> 
이를 통해 인터프리터보다 더 빠른 실행이 가능합니다.</br>

<h2>AOP컴파일과 JIT컴파일 차이점</h2>
AOT 컴파일은 소스 코드를 기계어로 변환하는 과정을 사전에 수행하는 방식입니다. </br>
즉, 소스 코드를 컴파일하여 실행 파일을 생성하는 과정이 빌드 시간에 이미 이루어집니다.</br>
AOT 컴파일된 실행 파일은 컴퓨터 아키텍처와 운영체제에 특화되어 있어서 직접 실행할 수 있습니다.</br>

JIT 컴파일은 소스 코드를 중간 언어로 변환한 후, 실행 시간에 해당 언어를 기계어로 컴파일하는 방식입니다. </br>
즉, 소스 코드를 실행하기 전에 컴파일 과정이 필요하며, 이러한 컴파일은 실행 시간에 동적으로 이루어집니다. </br>
JIT 컴파일은 실행 시점에서 실제로 필요한 부분만을 컴파일하고 최적화하므로,</br>
실행 중에 동적으로 변화하는 상황에 적응하여 최적의 성능을 제공할 수 있습니다. </br>

간단히 말해서, AOT 컴파일은 사전에 컴파일하여 실행 파일을 생성하는 방식이고, JIT 컴파일은 실행 시점에서 필요한 부분을 컴파일하는 방식입니다.  </br>
AOT 컴파일은 실행 파일의 크기가 크고 빌드 시간이 오래 걸리지만 실행 속도가 빠르며,  </br>
JIT 컴파일은 실행 파일의 크기가 작고 빌드 시간이 짧지만 처음 실행 시에는 다소 느릴 수 있지만  </br>
나중에 최적화되어 빠른 실행을 제공할 수 있습니다. </br>


<h2>JVM 메모리 관리</h2>

자바는 <strong>가비지 수집</strong>이라는 프로세스를 이용해 힙 메모리를 자동 관리하는 방식으로 해결한다. <br>
가비지 수집 : jvm이 더 많은 메모리를 할당해야할 때 불필요한 메모리를 회수하거나 재사용하는 불확정적프로세스이다


<h2>스레딩과 자바 메모리 모델(JMM)</h2>

스레드 A와 스레드 B가 둘 다 객체 obj를 참조할 때 스레드 A가 obj값을 바꾸면 스레드 B는 무슨 값을 참조하게 될까? <br>
=> 상호 배타적 락 : 코드가 동시 실행되는 도중 객체가 손상되는 현상을 막을 수 있는 자바의 유일한 방어 장치

<h2>JVM 구현체 종류</h2>

<h6>OpenJDK</h6>
<h6>오라클 자바</h6>
<h6>줄루</h6>
<h6>아이스티</h6>
<h6>징</h6>
<h6>J9</h6>
<h6>애비안</h6>
<h6>안드로이드</h6>


<h2>JVM 모니터링과 툴링</h2>

Jvm은 실행 중인 애플리케이션을 인스트루먼테이션, 모니터링, 관측하는 다양한 기술을 제공.</br>
  - 자바 관리 확장 (JMX)
  - 자바 에이전트
  - JVM툴 인터페이스(JVMTI)
  - 서비서빌리티 에이전트(SA)

<strong>JMX</strong>는 JVM과 그 위에서 동작하는 애플리케이션을 제어하고 모니터링하는 범용 툴.</br>
<strong>자바 에이전트</strong>는 자바 언어로 작성된 툴 컴포넌트로, java.lang.instrument 인터페이스로 메서드 바이트코드를 조작.</br>
<strong>JVMTI</strong>는 JVM의 네이티브 인터페이스이다. JVMTI를 사용하는 에이전트는 필히 c/c++ 같은 네이티브 컴파일 언어로 작성.</br>
<strong>SA</strong>는 자바객체, 핫스팟 자료 구조 모두 표출 가능한 API와 툴을 모아놓은 것.</br>
 => 핫스팟 SA는 심볼 룩업 같은 기본형을 이용하거나 프로세스 메모리를 읽는 방식으로 디커깅을 함.
 => 코어파일 및 아직 생생한 자바 프로세스까지 디버깅할 수 있다.
<h5>VisulVM</h5>

VisulVM은 JVM 어태치 메커니즘을 이용해 실행 프로세스를 실시간모니터링.</br>
=> 윈도우 visualVM 설치 및 인텔리제이 연동</br>
https://haenny.tistory.com/410 (참고)

<h6>구성</h6>
- 개요
  - 자바 프로세스에 관한 요약 정보를 표시
- 모니터
  - cpu, 힙 사용량 등 JVM을 고수준에서 원격 측정한 값들이 표시
- 스레드
  - 각 스레드가 시간대별로 표시
- 샘플러 및 프로파일러
  - cpu 및 메모리  사용률에 관한 단순 샘플링 결과가 표시


![KakaoTalk_20230709_123254776](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/ebf5cbe2-2642-4879-8ac5-45d1e6fb2ace)




 
  ![KakaoTalk_20230709_123254776_01](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/5014f45c-577a-4d68-9a02-7fc5928b597a)



  ![KakaoTalk_20230709_123254776_02](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/5bcc4e77-c382-4439-afb3-fe8c0424e066)







