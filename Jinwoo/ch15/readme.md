## 자바 9와 미래

자바 9는 '모듈과 그 나머지'이다.  자바 8이 람다와 그 결과물에 중점을 두었다면 자바 9는 거의 모듈에 관한 버전이다.

모듈이 실제 성능 측면에서 크게 중요한 비중을 차지하는 건 아니기 때문에 모듈에 초점을 맞추기보다 성능과 관련된 작은 변경 사항 위주로 설명한다.

## 15.1 자바 9에서 소소하게 개선된 성능

* 코드 캐시 세그먼트화
* 컴팩트 스트링
* 새로운 스트링 연결
* C2 컴파일러 개선
* G1 새 버전

### 15.1.1 코드 캐시 세그먼트화

자바 9부터는 코드 캐시 성능을 개선코자 다음 항목별 영역으로 분리했다.

* 인터프리터 등의 논메서드 코드
* 프로파일드 코드
* 논프로파일드 코드

덕분에 스위퍼 가동 시간이 짧아지고 풀 최적화 코드에 대한 코드 지역성이 향상된다. 다른 영역은 공간이 남아도는데 특정 영역이 꽉 채워질 수 잇다는 단점은 있다.

### 15.1.2 콤팩트 스트링

자바에서 스트링 콘텐츠는 char[] 타입으로 저장된다. 하지만 char는 16비트(2바이트) 타입이라서 ASCII 스트링(1바이트) 을 저장하면 필요 공간의 2배를 더 차지하는 공간 낭비가 이뤄진다.

자바 9 이후부터는 **콤팩트 스트링(compact string)** 으로 최적화한다. Latin-1 캐릭터로 표기 가능한 스트링은 byte 배열로 나타낼 수 있다. 기존 char 배열로 나타낼 때 0으로 채워졌던 바이트 개수만큼 절약할 수 있다.

### 15.1.3 새로운 스트링 연결

자바 5 이후로는 문자열을 붙일 때 **StringBuilder** 를 많이 사용했다.

자바 9부터는 **invokedynamic**을 활용해 **StringBuilder** 에 대한 바이트코드를 줄일 수 있다.

```javap -verbose``` 명령으로 살펴보면 상수 풀에 bootstrap메서드가 보인다. **StringConcatFactory.makeConcatWithConstants()** 라는 팩토리 메서드로 스트링 연결 레시피를 제공한다. 이 기법을 응용하면 새로운 커스텀 메서드용 바이트코드 작성 등 다양한 전략을 구사할 수 있다.

### 15.1.4 C2 컴파일러 개선

현대 CPU에 선보인 **SIMD(single instruction multiple data)** 확장 기능을 응용하면 C2의 성능 개선을 기대할 수 있다. 

* SIMD 
    * 하나의 명령어로 여려개의 데이터를 처리할 수 있는 인텔의 기술

다른 프로그래밍 환경에 비해 자바/JVM은 플랫폼 특성 덕분에 SIMD를 더 효과적으로 활용하기 좋은 조건을 가지고 있다.

성능 개선은 **VM 인트린직**으로 구현할 수 있다. 핫스팟 VM이 애너테이션을 붙인 메서드를 수동 작성된 어셈블리 그리고/또는 컴파일러 IR로 치환하면 해당 메서드는 인트린직 된다.

자바 9버전은 SIMD의 장점 및 그와 연관된 프로세서 특성을 더 잘 활용하고자 기존 인트린직을 개선하거나 새로운 인트린직을 탑재했다.

### 15.1.5 G1 새 버전

G1은 중단 시간을 더 쉽게 튜닝하고 더 효과적으로 제어하는 특성을 지닌, 여러 가지 문제를 한꺼번에 해결하고자 설계된 수집기이다. 자바 9부터는 디폴트 가비지 수집기가 되었다.

자바 9 G1와 자바 8 G1은 다르기 때문에 애플리케이션에 영향을 줄 수 있다. 자바 9로 이전한 애플리케이션이 별다른 문제가 없는지 꼼꼼히 테스트해야봐야 한다.

## 15.2 자바 10과 그 이후 버전

### 15.2.1 새로운 릴리즈 절차

자바 10 이전까지는 새로운 특성이 준비될 때까지 릴리즈를 미루는 경우가 있었는데, 자바 10 이후부터는 정확한 시기에 릴리즈되는 모델로 개편됐다.

### 15.2.2 자바 10

JVM의 새로운 특성 및 개선사항은 자바 개선 프로세스를 통해 관리하며, JDK 개선 제안서마다 관리 번호를 하나씩 매긴다.

* 286 : 지역 변수 타입 추론
    * 286이 구현되면 지역 변수를 선언할 때 반복되는 코드를 줄여쓸 수 있다.
        ```
        var list = new ArrayList<String>(); // ArrayList<String> 추론
        var stream = list.stream(); // Stream<String> 추론
        ```
    * for루프의 초기자, 지역 변수에만 쓸 수 있게 제한될 예정이다.
    * 소스 컴파일러에 구현될 로직이라서 바이트코드, 성능과는 전혀 무관하지만 와들러의 법칙으로 잘 알려진 언어 설계의 중요한 단면을 관찰할 수 있다.

* 296 : JDK 포레스트를 단일 리파지터리로 통합
    * 완전히 내부적인 문제이다.

* 304: 가비지 수집기 인터페이스
    * 서로 다른 가비지 수집기의 코드를 더 분리해서 동일한 JDK 빌드에서 가비지 수집기 인터페이스를 깔끔하게 하자는 아이디어

아래 3가지 특성은 미미하게나마 성능에 영향을 준다. 

* 307 : G1에서 풀 병렬 GC 구현
    * G1 수집기의 문제점을 해결한다.
    * 마크-스위프-컴팩트 알고리즘을 병렬화해서 G1에 풀 GC가 발생할 경우 같은 개수의 스레드를 그대로 동시 수집에 사용하자는 발상이다.

* 310 : 애플리케이션 클래스 데이터 공유

* 312 : 스레드 로컬 핸드쉐이크

## 15.3 자바 9 Unsafe 그 너머

sun.misc.Unsafe 기능을 대체할 기술이 없다면 주요 프레임워크와 라이브러리는 더 이상 제대로 작동하지 않을 것이다. 

자바 9에서는 ```--illegal-access``` 라는 스위치로 이 API에 대한 런타임 액세스를 조정할 수 있다. 하지만 자바 9 출시 전에는 이 작업을 마치지 못했다.

이들 API의 대안을 마련하는 과정에서 다른 성과도 있었다. getCallerClass() 메서드는 JEP 259에서 발의된 스택-워킹에서 사용할 수 있게 됐다.

### 15.3.1 자바 9의 VarHandle

메서드 핸들은 실행 가능한 메서드의 레퍼런스를 직접 조작할 수 있게 해주지만, 게터/세터 액세스만 지원했다. 

자바 9부터 메서드 핸들은 JEP 193 가변 핸들까지 포괄하도록 확장됐다. 이는 Unsafe에 있는 API 일부를 안전하게 대체하여 간극을 메우자는 의도이다.


## 15.4 발할라 프로젝트와 값 타입

프로젝트의 주요 목표

* JVM 메모리 레이아웃을 최신 하드웨어의 비용 모델에 맞게 조정한다.
* 제네릭스가 기본형, 값, 심지어 void까지 포함하도록 모든 타입에 추상화한다.
* 기존 라이브러리, 특히 JDK가 이러한 특성을 최대한 활용하는 방향으로 호환성을 유지하며 진화하도록 한다.

JVM에서 **값 타입**의 사용 가능성을 모색하는 것이 이 프로젝트에서 가장 중점을 둔 부분이다.

지금까지 자바의 메모리 레이아웃 패턴은 단순하다는 게 장점이지만, 객체 배열을 처리하려면 간접화가 불가피하며 캐시 미스가 수반되어 성능 감소로 이어지는 단점이 있었다.

이를 유사 구조체 배열로 구현할 수 있다면 단순 타입 객체를 메모리에 배치할 수 있기 때문에 훨씬 효율적일 것이다.

하지만, 자바 5부터 제네릭 타입이 등장하면서 난관에 부딪혔다. 자바 제네릭스는 'Object' 서브 타입인 참조형에만 적용된다. 값 타입은 반드시 제네릭스를 개선한 형태에서 타입 매개변수 값으로 유효하다는 전제하에 설계해야 한다.

## 15.5 그랄과 트러플

C2 컴파일러의 대체품을 찾는 연구 프로젝트로는 현재 그랄과 트러플 위주로 진행되고 있다.

### 그랄

* 특수한 JIT 컴파일러
* C2 컴파일러는 C++로 작성되어 있어 심각한 문제를 일으킬 소지가 있다. 
    * C++은 개발자가 수동으로 직접 메모리를 관리하는 안전하지 않은 언어라서 언제라도 C2코드가 VM을 멈추게 할 수 있다.
    * 관리/확장이 힘들다.
* 그랄은 JVM용 JIT컴파일러를 자바언어로 개발한다. JIT 컴파일러는 JVM 바이트코드를 받아 기계어를 생성하기만 하면 된다는 것이다.
    * java-in-java 방식은 단순함, 메모리 보안 등 여러 면에서 이점이 있다.
    * 그랄로 부분 탈출 분석 등의 새로운 최적화 기법을 구현하고 커스텀 인트리직 등을 구현할 수 있게 되었다.

### 트러플

* JVM 런타임에 호스팅된 언어에 대한 인터프리터 생성기
* 입력 언어에 대한 고성능 JIT 컴파일러를 인터프리터에서 자동 생성하는 라이브러리

## 15.7 동시성의 향후 발전 방향

자바가 일으킨 가장 큰 혁신은 자동 메모리 관리 기능이다.

초반 자바 스레딩 모델은 프로그래머가 모든 스레드를 직접 관리하고 가변 상태는 락으로 보호해야 했다. 나중에 등장한 자바 버전은 더 고수준의, 개발자 손이 덜 가고 일반적으로 더 안전하게, 사실상 런타임이 동시성을 관리하는 방향으로 진화했다.

룸 프로젝트는 그러한 노력의 일환으로, 동시성을 JVM 상에서 지원했던 것과 달리, 더 저수준에서 지원하는 방안을 모색한다.




