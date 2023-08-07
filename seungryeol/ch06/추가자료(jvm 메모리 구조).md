<h1>JVM 메모리  구조</h1>

![jvm 메모리 구조](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/bc42c225-51d6-4c8c-84d4-5a9f21bf86ac)



(1) Class Loader</br>

JVM 내로 클래스 파일을 로드하고, 링크를 통해 배치하는 작업을 수행하는 모듈입니다. 런타임 시에 동적으로 클래스를 로드합니다.</br>

(2) Execution Engine</br>

클래스 로더를 통해 JVM 내의 Runtime Data Area에 배치된 바이트 코드들을 명렁어 단위로 읽어서 실행합니다. 최초 JVM이 나왔을 당시에는 인터프리터 방식이었기때문에 속도가 느리다는 단점이 있었지만</br> JIT 컴파일러 방식을 통해 이 점을 보완하였습니다.</br>
JIT는 바이트 코드를 어셈블러 같은 네이티브 코드로 바꿈으로써 실행이 빠르지만 역시 변환하는데 비용이 발생하였습니다. </br>
이 같은 이유로 JVM은 모든 코드를 JIT 컴파일러 방식으로 실행하지 않고, 인터프리터 방식을 사용하다가 일정한 기준이 넘어가면 JIT 컴파일러 방식으로 실행합니다.</br>


(3) Garbage Collector</br>

Garbage Collector(GC)는 힙 메모리 영역에 생성된 객체들 중에서 참조되지 않은 객체들을 탐색 후 제거하는 역할을 합니다. 이때, GC가 역할을 하는 시간은 언제인지 정확히 알 수 없습니다.</br>

(4) Runtime Data Area</br>

JVM의 메모리 영역으로 자바 애플리케이션을 실행할 때 사용되는 데이터들을 적재하는 영역입니다. 이 영역은 크게 Method Area, Heap Area, Stack Area, PC Register, Native Method Stack로 나눌 수 있습니다.</br>

<h4>Runtime Data Area</h4>

(1) Method area </br>

모든 쓰레드가 공유하는 메모리 영역입니다. 메소드 영역은 클래스, 인터페이스, 메소드, 필드, Static 변수 등의 바이트 코드를 보관합니다.</br>
메서드 영역은 JVM이 시작될 때 생성되는 공간으로 바이트 코드(.class)를 처음 메모리 공간에 올릴 때 초기화되는 대상을 저장하기 위한 메모리 공간이다.</br>
JVM이 동작하고 클래스가 로드될 때 적재되서 프로그램이 종료될 때까지 저장 된다.</br>

2. Heap area</br>

![KakaoTalk_20230806_214524915](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/38e65cb8-9dc0-4fbc-bf1a-ba3ee916abc9)</br>


모든 쓰레드가 공유하며, new 키워드로 생성된 객체와 배열이 생성되는 영역입니다. 또한, 메소드 영역에 로드된 클래스만 생성이 가능하고 Garbage Collector가 참조되지 않는 메모리를 확인하고 제거하는 영역입니다.</br>
힙 영역은 메서드 영역와 함께 모든 쓰레드가 공유하며, JVM이 관리하는 프로그램 상에서 데이터를 저장하기 위해 런타임 시 동적으로 할당하여 사용하는 영역이다.</br>
즉, new 연산자로 생성되는 클래스와 인스턴스 변수, 배열 타입 등 Reference Type이 저장되는 곳이다.</br>

![캡처3](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/280c48d7-3c42-47fb-9e10-5c31ada96c71)



3. Stack area </br>

![스택 영역](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/30813718-b260-44b8-b0bd-40dd1cf51df8)</br>


메서드 호출 시마다 각각의 스택 프레임(그 메서드만을 위한 공간)이 생성합니다. 그리고 메서드 안에서 사용되는 값들을 저장하고, 호출된 메서드의 매개변수, 지역변수, 리턴 값 및 연산 시 일어나는 값들을 임시로 저장합니다. 마지막으로, 메서드 수행이 끝나면 프레임별로 삭제합니다.</br>



4. PC Register</br>

쓰레드가 시작될 때 생성되며, 생성될 때마다 생성되는 공간으로 쓰레드마다 하나씩 존재합니다. 쓰레드가 어떤 부분을 무슨 명령으로 실행해야할 지에 대한 기록을 하는 부분으로 현재 수행중인 JVM 명령의 주소를 갖습니다.</br>

일반적으로 프로그램의 실행은 CPU에서 명령어(Instruction)을 수행하는 과정으로 이루어진다.</br>


5. Native method stack</br>

네이티브 메서드 스택는 자바 코드가 컴파일되어 생성되는 바이트 코드가 아닌 실제 실행할 수 있는 기계어로 작성된 프로그램을 실행시키는 영역이다.</br>
JIT 컴파일러에 의해 변환된 Native Code 역시 여기에서 실행이 된다. </br>
일반적으로 메소드를 실행하는 경우 JVM 스택에 쌓이다가 해당 메소드 내부에 네이티브 방식을 사용하는 메소드가 있다면 해당 메소드는 네이티브 스택에 쌓인다.</br>
그리고 네이티브 메소드가 수행이 끝나면 다시 자바 스택으로 돌아와 다시 작업을 수행한다.</br>
그래서 네이티브 코드로 되어 있는 함수의 호출을 자바 프로그램 내에서도 직접 수행할 수 있고 그 결과를 받아올 수도 있는 것이다.</br>

 



![캡처1](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/9d24a645-5319-4cdd-a6fe-912b9130bc6f)</br>

</br>
 Method Area, Heap Area 는 모든 쓰레드(Thread)가 공유하는 영역이고, 나머지 Stack Area, PC Register, Native Method Stack 은 각 쓰레드 마다 생성되는 개별 영역이다.</br>

![캡처2](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/7d90c600-0d40-4abb-8a60-de3d348baebc)




