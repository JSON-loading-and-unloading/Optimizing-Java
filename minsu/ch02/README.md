# 2 JVM 이야기
# 2.1 인터프리팅과 클래스로딩 (Interpreting and Classloading)

- JVM은 스택 기반의 해석 머신이다.
- register 는 없지만 일부 결과를 실행 스택에 보관하며, 이 스택의 가장 위에 쌓인 값을 가져와 계산한다.
- JVM interpreter 는 while 루프 안의 switch 문 으로 이해할 수 있다.
 ``` Java
while(조건) {
		switch(대상) {
				case 1:

				case 2:

				default:	
		}
}
 ```
## ```java HelloWorld``` 라는 명령을 실행하면  
1. OS는 가상 머신 프로세스(java binary)를 구동한다.
2. 자바 가상 환경이 구성된다.
3. stack 머신이 초기화 된다.
4. 실제로 유저가 작성한 HelloWorld 클래스가 실행된다.  
여기서 application 의 entry point는 HelloWorld.class 의 main() method 이다.  
제어권을 이 클래스로 넘기기 위해 가상 머신(이하 VM) 이 실행되기 전에 이 클래스를 load 해야한다.  


## classloading 매커니즘  
자바 process 가 초기화되면 사슬처럼 연결된 클래스 로더가 차례차례 작동한다.

1. bootstrap class 실행  
부트스크랩 클래스로더의 주임무는 다른 클래스로더가 나머지 시스템에 필요한 크래스를 로드할 수 있게 최소한의 필수 클래스(```java.lang.Object```,  ```Class```,  ```Cloassload```) 만 로드하는 것이다.

  
2. 자바 8 이전까지는 rt.jar 에서 런타임 코어 클래스를 로드한다.  
   자바 9 이후부터는 런타임이 모듈화 되고 클래스 로딩 개념 자체가 달라졌다.

3. 확장 클래스로더  
부트스크랩 클래스로더를 자기 부모로 설정하고 필요할 때 클래스로딩 작업을 부모에게 넘긴다.  
특정한 OS나 플랫폼에 native code 를 제공하고 기본 환경을 오버라이드할 수 있다.

4. 애플리케이션 클래스로더
끝으로 애플리케이션 클래스로더가 생성되고 지정된 클래스패스에 위치한 유저 클래스를 로드한다.

❗️ 애플리케이션 클래스로더를 '시스템' 클래스로더라고 부르는 글도 있는데 시스템 클래스는 로드하지 않기 때문에 가급적 사용하지 말아야 한다

이는 모두 상속 관계이다.
> bootstrap (최상단 부모) → extension classload → application classloader (최하단 자식) 

java program 을 실행하다가 새 클래스를 발견하여 찾지 못하면 한 단계씩 상위로 올려 lookup 을 하고,
만약 찾지 못한다면  ```ClassNotFoundException``` 을 일으킨다.



# 2.2 바이트코드 실행 (Executing Bytecode)
자바 소스코드는 실행되기까지 꽤 많은 변환 과정을 거친다.
첫 단계는 javac를 이용해 컴파일하는 것이다. 
> javac가 하는 일은 자바 소스 코드를 바이트코드로 가득 찬  ```.class```파일로 바꾸는 것이다. 컴파일하는 동안 최적화는 거의 하지 않기 때문에 생성된 바이트코드는 쉽게 해독할 수 있다.
>
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/cbf3a0f9-379f-4d45-8e3c-2c6ba2de2f38)

## 바이트코드
바이트코드는 컴퓨터의 아키텍쳐에 특정되지 않는 중간 표현형(Intermediate Representation, IR) 이다.  
이식성이 좋고, JVM 이 지원되는 플랫폼 어디서든 실행할 수 있으며, java 에 대해서 추상화 되어있다.  
JVM이 코드를 실행하는 원리를 이해하는데 중요하다.  
```
JVM 에서 J 는 오해의 소지가 있는 naming 이다.  
JVM 규격에 맞춰 만들어진 모든 언어가 JVM 에서 실행될 수 있다.(scala, kotlin)
```


## 클래스 파일 해부도 (spec)
### 1. 매직 넘버 : 0xCAFEEBABE
- 모든 클래스 파일은 0xCAFEBABE라는 매직 넘버, 즉 이 파일이 클래스 파일임을 나타내는 4바이트 16진수로 시작한다.  
- 자바 9부터는 0xCAFEDADA  

### 2. 클래스 파일 포맷 버전: 클래스 파일의 메이저/ 마이너 버전
- 매직 넘버 이후의 4 바이트  
- 클래스로더의 호환성 보장을 위해 검사  
- 호환되지 않는 버전의 클래스 파일을 만나면 ```UnSupportedClassVersionError``` 예외 발생

### 3. 상수 풀: 클래스 상수들이 모여 있는 위치 
- 상숫값 (ex. 클래스명, 인터페이스명, 필드명)
- JVM은 코드를 실행할 때 런타임에 배치된 메모리 대신, 이 상수 풀 테이블을 찾아보고 필요한 값을 참조

### 4. 액세스 플래그: 추상 클래스, 정적 클래스 등 클래스 종류를 표시
- 클래스에 적용한 수정자를 결정  
- 플래그 첫 부분에는 public 클래스인지, final 클래스인지 나타낸다.  
- 인터페이스인지, 추상 클래스인지도 나타낸다.  
- 플래그 끝 부분에는 합성 클래스인지, 애너테이션 타입인지, enum인자를 각각 나타낸다.

### 5. this 클래스 : 현재 클래스명, 슈퍼클래스: 슈퍼클래스명, 인터페이스: 클래스가 구현한 모든 인터페이스
- 클래스에 포함된 타입 계층을 나타낸다.
- 각각 상수 풀을 가리키는 인덱스를 표시한다.


### 6. 필드 : 클래스에 들어 있는 모든 필드, 메서드 : 클래스에 들어 있는 모든 메서드
- 시그니처 비슷한 구조를 정의하고 여기에 수정자도 포함되어 있다.

### 7. 속성 : 클래스가 지닌 모든 속성
- 더 복잡하고 크기가 고정되지 않은 구조를 나타내는 데 쓰인다.
- 메서드는 Code 속성으로 특정 메서드와 연관된 바이트코드를 나타낸다.

암기팁 !
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/dad8eddb-d83a-40c0-8660-1262e1a06221)

다음과 같이 간단한 HelloWorld 클래스를 javac로 컴파일하면 무슨 일이 일어날까? 
```Java
class HelloWorld2 {
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            System.out.println("Hello World");
        }
    }
}
```
이것을 javap -c HelloWorld 명령을 실행하면 HelloWorld 클래스 파일을 들여다볼수 있다.

```Java
public class HelloWolrd
  public HelloWorld();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
  public static void main(java.lang.String[]);
    Code:
       0: iconst_0
       1: istore_1
       2: iload_1
       3: bipush        10
       5: if_icmpge     22
       8: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: ldc           #3                  // String Hello World
      13: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      16: iinc          1, 1
      19: goto          2
      22: return
}
```

<details open>
  <summary>자세한 설명 </summary>
  생성자에서  

0. this 레퍼런스를 스택 상단에 올려놓는 aload_0명령이 실행된다.
1. invokespecial 명령이 호출되면 슈퍼생성자들을 호출하고 객체를 생서하는 등 특정 작업을 담당하는 인스턴스 메서드를 실행한다.

main 메서드에서  

0. iconst_0으로 정수형 상수 0을 평가 스택에 푸시한다.
1. istore_1로 이 상숫값을 오프셋 1에 위치한 지역 변수에 스토어한다.

2~16 과정은 if_icmpage 테스트가 성공할때까지 반복되다가 마지막에 22번 명령으로 제어권이 넘어가 메서드가 반환  

　2\. 오프셋 1의 변수를 스택으로 다시 로드한다.  
　3\. 상수 10을 푸시한다.   
　5\. if_icpge로 둘을 비교한다. (정숫값이 10보다 같거나 큰가? )  
　처음 몇 차례는 비교 테스트가 실패할테니 8로 간다.  
　8\. 정적 메서드를 해석한다.  
　11\. 상수 풀에서 "Hello World"라는 스트링을 로드한다.  
　13\. 이 클래스에 속한 인스턴스 메서드를 실행한다.  
　16\. 정숫값이 하나 증가되고 goto를 만나 다시 2번 명령으로 돌아간다 (반복)  

</details>

# 2.3 핫스팟 입문

1999년 4월 자바 성능의 가장 큰 변화를 가져온 요체가 hotspot 이다.

핫스팟 JVM
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/3d8486ff-c108-46c1-bc43-e965edc2286c)  
zero cost(비용이 들지 않는) 추상화 사상에 근거하여 '기계에 가까운' 언어와 그리고 개발자의 생산성을 염두한 엄격한 저수준 제어 이 두가지 사이에서 언어는 갈등을 겪고 있다.
> C++ 코드는 제로-오버헤드 원칙을 준수합니다.  
> 사용하지 않는 것에는 대가를 치르지 않습니다.  
> 즉, 여러분이 사용하는 코드보다 더 나은 코드를 건네줄 수는 없습니다  
> 
>  -비야네 스트롭스트룹

제로-오버헤드 원칙은 결국 컴퓨터와 OS가 실제로 어떻게 작동해야 하는지 개발자가 아주 세세한 저수준까지 일러주어야 한다는 것이다.  
또한, 개발자는 결코 자동화 시스템보다 더 나은 코드를 작성할 수 없을 거라는 전제도 깔려 있다.  

Java 는 이러한 zero-overhead 추상화 원칙을 동조하지 않는다.
hotspot은 runtime 에 동작을 분석하고, 성능에 가장 유리한 방향으로 그때그때 최적화를 적용하는 가상머신이다.  
즉, 개발자가 VM 틀에 맞게 욱여넣는 대신, 자바 코드로 작성하고 바람직한 설계 원리를 따르도록 한다.  

## 2.3.1 JIT 컴파일러란?
자바 프로그램은 bytecoe interpreter 가 가상화한 stack 머신에서 실행하며 시작된다.  
CPU 를 추상화 한 구조이기 때문에 다른 platform 에서 class 파일을 문제 없이 실행할 수 있지만, 성능을 최대로 내려면 native 기능을 활용해 CPU 에서 직접 실행시켜야 한다.  
hotspot 은 이를 위해 interpreted bytecode 에서 native 코드로 컴파일 하는데 이것을 JIT(Just-In-Time) 이라 한다. 애플리케이션을 모니터링하면서 자주 실행되는 코드 파트를 찾아내 JIT 컴파일을 수행한다.
이러한 코드는 최적화된 코드이다.  
컴파일러가 해석 단계에서 수집한 정보를 근거로 최적화를 결정한다.  
따라서 최신 성능 최적화의 덕을 보려면 hotspot 최신 버젼에서 application을 실행하는 것이 좋다.  
java는 profile 기반 최적화(Profile-guided optimization PGO)를 응용하는 환경에서 대부분의 AOT 로 불가능한 방식을 선택해 runtime 정보를 활용하며, CPU 타입을 정확히 감지해 해당 프로세서의 기능에 맞게 최적화한다.  

# 2.4 JVM 메모리 관리

C, C++ 은 메모리를 개발자가 직접 관리한다.
- 장점: 조금 더 확정적인 성능을 낼 수 있다.
- 단점: 개발자가 메모리를 정확하게 계산해서 처리해야 하는 막중한 책임감이 수반된다.

자바는 Garbage Collection(이하 GC) 라는 프로세를 이용해 heap memory 를 자동으로 관리한다.
> GC는 JVM이 더 많은 메모리를 할당해야 할 때 불필요한 메모리를 회수하거나 재사용하는 불확정적 프로세스이다.
> GC가 실행되면 그 동안 다른 애플리케이션은 모두 중단되고 하던일을 멈춰야 하는데 이것은 애플리케이션의 부하가 늘어날 수록 무시할 수 없는 시간이 된다.


# 2.5 스레딩과 자바 메모리 모델
java 는 1.0 부터 멀티스레드 프로그래밍을 지원하였고 다음과 같이 실행 thread 를 새로 만들 수 있다
```Java
Thread t = new Thread(() -> {
		System.out.println("Hello World");
});
```
- java 환경 자체가 멀티스레드 기반이기 때문에 복잡해졌고 작업하기 힘들어졌다.
  
주류 JVM 구현체에서 자바 애플리케이션 스레드는 는 각각 하나의 전용 OS thread 에 대응한다.
공유 스레드 풀을 이용해 전체 자바 애플리케이션 풀을 실행하는(green thread) 방안도 있지만 복잡도만 증가시킬 뿐 성능적인 측면에서 이득이 없는것으로 밝혀졌다.

1990년대 후반부터 자바 멀티스레드 방식은 다음의 3가지 설계 원칙에 기반한다.
- 자바 프로세스의 모든 스레드는 가바지가 수집되는 하나의 공용 힙을 갖는다.
- 한 스레드가 생성한 객체는 그 객체를 참조하는 다른 스레드가 액세스할 수 있다.
- 기본적으로 객체는 final 키워드로 불변 표시하지 않는 한 변경이 가능하다.

# 2.6 JVM 구현체 종류

### openJDK
- 자바 기준 구현체를 제공하는 특별한 오픈소 소스이다.
- 오라클이 직접 주관/지원하며 자바 릴리즈 기준을 발표한다

### oracle Java
- 가장 널리 알려진 구현체로 OpenJDK 기반이지만, 오라클의 상용 라이센스이다.
- 오라클 자바에 변경된 내용은 거의 전부 OpenJDK 공개 저장소에 커밋된다.


## 2.6.1 JVM 라이선스
JVM 구현체는 거의 다 오픈 소스이고, IBM J9와 사용 제품인 아줄 징을 제외하면 대부분 핫스팟에서 비롯된 제품이다.  
오라클 자바의 기반은 OpenJDK 코드 베이스이지만 오픈 소스가 아닌, 상용 제품이다.  
오라클 JDK와 OpenJDK는 라이선스 외에는 아무런 차이가 없다.  

개발자가 조심해야 할 몇 가지 조항 !
- 회사 밖으로 오라클 바이너리를 재배포하는 행위는 허용되지 않는다.
- 사전 동의 없이 오라클 바이너리를 함부로 패치하면 안 된다.

# 2.7 JVM 모니터링과 툴링
JVM은 성숙한 실행 플랫폼으로, 실행 중인 애플리케이션을 instrumentation, 모니터링, 관측하는 다양한 기술을 제공한다. 

## Java Management Extensions (JMX)
- JVM과 그 위에서 동작하는 애플리케이션을 제어하고 모니터링하는 범용 툴dlek.
- 메서드를 호출하고 매개변수를 바꿀 수 있다. 
- JVM을 관리하는 기본 수단이다.
  
## Java agent
- 자바 언어로 작성된 툴 컴포넌트로, java.lang.instrument 인터페이스로 메서드 바이트코드 조작  
> javaagent: <agent jar 경로>=<옵션>
>
- 에이전트 JAR 파일에서 매니페스트(manifest, manifest.mf 파일)를 반드시 지정해야 한다. 
- 등록 hook 역할을 하는 pbulic static premain()메서드를 구현해야 한다.

## JVM Toll Interface (JVMTI)
- java instrument API 로 부족할 때 사용한다
- C/C++ 과 같은 native compile 언어로 작성해야 한다
- 설치하는 플래그는 다음과 같다
> agentlib:<라이브러리명>=<옵션>

또는

> agentpath:<에이전트 경로>=<옵션>

되도록 JVMTI 보다 java agent 를 사용하는 게 낫다.

## Serviceability Agent (SA)
- 자바 객체, 핫스팟 자료 구조 모두 표출 가능한 API 툴을 모아 놓은 것이다.
- JVM에서 코드를 실행할 필요가 없다.
- symbol lookup과 같은 기본형을 이용하거나 프로세스 메모리를 읽는 방식으로 디버깅

## 2.7.1 VisualVM
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/7196a8d1-5c9b-4fdd-9931-c68da5ae518c)
- Attach machanism을 이용해 실행 프로세스를 실시간으로 모니터링 한다.
- 프로세스가 local, remote 에 따라 작동 방식이 다르다
  - 개요 : 자바 프로세스에 관한 요약 정보
  - 모니터: CPU, heap 사용량 등 JVM 을 고수준에서 원격 측정한 값 제공
  - 스레드: 실행 중인 애플리케이션 각 스레드가 시간대별로 표시
  - 샘플러 및 프로파일러: CPU 및 메모리 사용률에 관한 단순 샘플링 결과
