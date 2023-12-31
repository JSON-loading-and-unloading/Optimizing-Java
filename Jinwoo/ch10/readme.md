## 10.1 JITWatch란?

> 실행 중인 자바 애플리케이션이 생성한 핫스팟 컴파일 상세 로그를 파싱/분석하여 그 결과를 자바FX GUI 형태로 보여준다.

* 애플리케이션 실행시 반드시 다음 플래그를 추가해줘야 한다.

    ```
    -XX:+UnlockDiagnosticVMOptions -XX:+TraceClassLoading -XX:+LogCompilation
    ```

* 반드시 '핫'패스에 있는 컴파일 대상 메서드를 분석 대상으로 삼아야 한다. 인터프리티드 메서드는 최적화 대상으로 적절치 않다.

### 10.1.1 기본적인 JITWatch 뷰

### 샌드박스

> JIT 작동을 시험해볼 수 있는 환경

* 작은 프로그램을 신속히 프로토타이핑하여 JVM이 어떤 JIT 결정을 내렸는지 확인할 수 있다.

* JIT 서브시스템을 조정하는 **VM 스위치** 를 시험해볼 수 있는 환경을 제공한다. 

    * 다음과 같이 JVM JIT 로직을 변경할 수 있다.

        * 역어셈블된 네이티브 메서드를 출력하고 어셈블리 구문을 선택한다.
        * JVM에서 단계별 컴파일에 맞게 설정된 디폴트를 오버라이드한다.
        * 압축 oop 사용을 오버라이드한다.
        * OSR 을 해제한다.
        * 인라이닝 디폴트 한계치를 오버라이드한다.

    <br>

    * 설정값을 바꾸면 JVM 성능에 심각한 영향을 미칠 수 있으니 극단적인 경우가 아니면 운영계에서는 권장하지 않는다.

### JITWatch 3단뷰

> 샌드박스보다 더 정교한 애플리케이션 컴파일 뷰로, 소스 코드가 바이트코드, 어셈블리 양쪽으로 어떻게 컴파일됐는지 알 수 있다.

### 10.1.2 디버그 JVM과 hsdis

* 디버그 JVM

    > 운영 JVM보다 더 상세한 디버깅 정보를 추출하는 가상머신

    (성능 희생은 감수해야 한다.)

* hsdis

    > JIT 컴파일러가 생성한, 역어셈블된 네이티브 코드를 살펴볼 수 있는 도구

## 10.2 JIT 컴파일 개요

핫스팟은 PGO(프로파일 기반 최적화)를 이용해 JIT 컴파일 여부 판단한다. 내부적으로 실행 프로그램 정보는 MDO(메서드 데이터 객체)라는 구조체에 저장한다.

핫스팟 JIT 컴파일러는 다양한 최신 컴파일러 최적화 기법을 동원한다.

* 인라이닝
* 루프 펼치기
* 탈출 분석
* 락 생략/확장
* 단일형 디스패치
* 인트린직

이러한 최적화 기법은 런타임 정보와 지원 여부에 따라 다소(완전히) 달라질 수 있다. 

핫스팟의 C1/C2 컴파일러 역시 이들 기법을 상이하게 조합해서 사용하며, 기본적으로 컴파일에 접근하는 철학이 다르다. C1은 추측성 최적화를 하지 않으나 C2는 공격적인 최적화기를 수행한다. 그래서 추측성 최적화를 하기 전에 항상 **가드** 라는 '타당성 검사'를 한다.

## 10.3 인라이닝

> 호출된 메서드(피호출부)의 콘텐츠를 호출한 지점(호출부)에 복사하는 것

예시

```
int result = add(a,b);

private int add(int x, int y) {
    return x + y;
}
```
인라이닝 최적화 후 add() 메서드 바디는 호출부에 합쳐진다.

```
int result = a + b;
```

다음과 같은 오버헤드를 제거할 수 있다.

* 전달할 매개변수 세팅
* 호출할 메서드를 정확하게 룩업
* 새 호출 프레임에 맞는 런타임 자료 구조 생성
* 새 메서드로 제어권 발송
* 호출부에 결과 반환


### 인라이닝 특징

* JIT 컴파일러가 제일 먼저 적용하는 최적화라서, **관문 최적화** 라고도 한다.
* 인라이닝 덕분에 잘 조직된, 재사용 가능한 코드를 작성할 수 있고, 무엇보다 손수 마이크로 최적화를 할 필요가 없어 좋다.
* 자동으로 통계치를 분석해 관련된 코드를 어느 시점에 하나로 모을지 결정한다. 그래서 다른 최적화의 범위를 확장시키는 역할을 한다.

### 10.3.1 인라이닝 제한

### 제약 조건
* JIT 컴파일러가 메서드를 최적화하는 데 소비하는 시간
* 생성된 네이티브 코드 크기

### 10.3.2 인라이닝 서브시스템 튜닝

### 인라이닝 서브시스템의 작동 방식을 제어하는 기본적인 JVM 스위치
|스위치|디폴트|설명|
|------|---|---|
|-XX:MaxInlineSize=<n>|35바이트의 바이트코드|메서드를 이 크기 이하로 인라이닝한다.|
|-XX:FreqInlineSize=<n>|325바이트의 바이트코드|'핫' 메서드를 이 크기 이하로 인라이닝한다.|
|-XX:InlineSmallCode=<n>|1000바이트의 네이티브 코드(단계없음) 2000바이트의 네이티브 코드(단계있음)|코드 캐시에 이 수치보다 더 많은 공간을 차지한 최종 단계 컴파일이 이미 존재할 경우 메서드를 인라이닝하지 않는다.|
|-XX:MaxInlineLevel=<n>|9|이 수준보다 더 깊이 호출 프레임을 인라이닝하지 않는다.|

중요 메서드가 인라이닝되지 않는 경우 ```-XX:MaxInlineSize=<n>``` 나 ```-XX:FreqInlineSize=<n>``` 값을 바꿔보면서 튜닝을 해볼 수 있다.

매개변수를 바꿔가며 튜닝할 때에는 반드시 측정 데이터를 근거로 삼아야 한다.


## 10.4 루프 펼치기

백 브랜치는 한번 순회를 마치고 다시 루프문 처음으로 돌아가는 것을 의미한다.

백 브랜치가 일어나면 그때마다 CPU는 유입된 명령어 파이프라인을 덤프하기 때문에 성능상 바람직하지 않다. 보통 루프 바디가 짧을수록 백 브랜치 비용이 상대적으로 높다.

### 루프 펼치기 여부 기준

* 루프 카운터 변수 유형
* 루프 보폭(한번 순회할 때마다 루프 카운터 값이 얼마나 바뀌는가)
* 루프 내부의 탈출 지점 개수(return 또는 break)

경계 검사 제거 ..?

### int형 카운터 vs. long형 카운터

int형 카운터 루프의 처리량이 약 64% 높다.

long형 카운터를 쓸 경우 루프 바디가 펼쳐지지 않고 루프안에 세이프포인트 폴이 박힌다는 사실을 알 수 있다. JIT 컴파일러는 컴파일된 코드가 너무 오랫동안 세이프포인트 플래그 체크 없이 실행되는 일이 없도록 세이프 포인트 검사 코드를 삽입한다.

### 10.4.1 루프 펼치기 정리

* 카운터가 int, short, char형일 경우 루프를 최적화한다.
* 루프 바디를 펼치고 세이프포인트 폴을 제거한다.
* 루프를 펼치면 백 브랜치 횟수가 줄고 그만큼 분기 예측 비용도 덜 든다.
* 세이프포인트 폴을 제거하면 루프를 순회할 때마다 하는 일이 줄어든다.

## 10.5 탈출 분석('범위 이탈 분석')

> 어떤 메서드가 내부에서 수행한 작업을 그 메서드 경계 밖에서도 볼 수 있는지, 또는 부수효과를 유발하지 않는지 판별하는 기법

탈출 분석 최적화는 반드시 인라이닝을 수행한 이후 시도해야 한다. 인라이닝해서 피호출부 메서드 바디를 호출부에 복사하면 호출부에 메서드 인수로 전달된 객체는 더 이상 탈출 객체로 표시되지 않기 때문이다.

### 잠재적 탈출 객체 유형

* NoEscape
    * 객체가 메서드/스레드를 탈출하지 않는다.
    * 호출 인수로 전달되지 않는다.
    * 스칼라로 대체 가능하다.

<br>

* ArgEscape
    * 객체가 메서드/스레드를 탈출하지 않는다.
    * 호출 인수로 전달되거나 레퍼런스로 참조된다.
    * 호출 도중에는 탈출하지 않는다.

<br>

* GlobalEscape
    * 객체가 메서드/스레드를 탈출한다.

### 10.5.1 힙 할당 제거

탈출 분석의 통해 힙 할당을 막을 수 있는지 추론한다.

루프 안에서 객체를 새로 만들면 그만큼 메모리 할당 서브시스템을 압박하게 되고, 단명 객체가 끊임없이 양산되며 이를 정리할 마이너 GC 이벤트가 자주 발생한다. 할당률이 너무 높아 영 세대가 꽉 차면 조기 승격이 일어날 가능성도 있다. 그러면 풀 GC 이벤트가 발생한다.

NoEscape 객체라면 VM은 **스칼라 치환** 최적화를 적용해 겍체 필드를 마치 처음부터 객체 필드가 아닌 지역 변수였던 것처럼 스칼라 값으로 바꾼다. 그런 다음 **레지스터 할당기** 라는 핫스팟 컴포넌트에 의해 CPU 레지스터 속으로 배치된다.

### 10.5.2 락과 탈출 분석

### 락 최적화의 핵심

* 비탈출 객체에 있는 락은 제거한다.(락 생략)
* 같은 락을 공유한, 락이 걸린 연속된 영역은 병합한다.(락 확장)
* 락을 해제하지 않고 같은 락을 반복 획득한 블록을 찾아낸다.(중첩 락)

단, 이 최적화는 인스린직 락에만 해당되며, java.util.concurrent 패키지에 있는 락에는 적용되지 않는다.

* 락 확장 최적화
    * 기본 활성화되어 있지만, VM 스위치 ```-XX:-EliminateLocks``` 로 해제하여 그 영향도를 살펴볼 수 있다.

<br>

* 중첩 락 최적화
    * 기본 활성화되어 있지만, VM 스위치 ```-XX:-EliminateNestedLocks``` 로 끌 수 있다.

핫스팟은 언제 락을 확장/제거하는 게 안전할지 자동으로 계산한다. JITWatch나 디버그 JVM을 사용해 그 내용을 살펴볼 수 있다.


### 10.5.3 탈출 분석의 한계

1. 기본적으로 원소가 64개 이상인 배열은 핫스팟에서 탈출 분석의 혜택을 볼 수 없다. 배열 길이가 64를 초과하면 (모든 배열 원소를 다 쓰지 않아도) 무조건 힙에 저장되고 이 코드의 할당률은 빠르게 상승할 수 있다.

    이 개수 제한은 다음 VM 스위치로 조정할 수 있다.

    ```
    -XX:EliminateAllocationArraySizeLimit=<n>
    ```

2. 부분 탈출 분석을 지원하지 않는다. 객체가 어느 분기점에서건 메서드 범위를 탈출하면 힙에 객체를 할당하지 않는 최적화는 적용되지 않는다.

    ```
    for(int i=0; i< 100_000_000; i++) {
        Object mightEscape = new Object(i);

        if(condition) {
            result += inlineableMethod(mightEscape);
        } else {
            result += tooBigToInline(mightEscape);
        }
    }
    ```

    따라서 위의 코드보다 아래코드처럼 비탈출 분기 조건 안에 객체 할당을 묶어두면 탈출 분석의 덕을 볼 수 있다.

    ```
    for(int i=0; i< 100_000_000; i++) {

        if(condition) {
            Object mightEscape = new Object(i);
            result += inlineableMethod(mightEscape);
        } else {
            Object mightEscape = new Object(i);
            result += tooBigToInline(mightEscape);
        }
    }
    ```

## 10.6 단형성 디스패치

단형성 디스패치는 아래의 가정으로부터 시작한다.

'어떤 객체에 있는 메서드를 호출할 때, 그 메서드를 최초로 호출한 객체의 런타임 타입을 알아내면 그 이후의 모든 호출도 동일한 타입일 가능성이 크다.'

항상 타입이 같다면 일단 호출 대상을 계산해서 invokevirtual 명령어를 퀵 타입 테스트(가드) 후 컴파일드 메서드 바디로 분기하는 코드로 치환하면 된다.

즉, klass 포인터 및 vtable을 통해 가상 룩업을 하고 에둘러 참조하는 일은 딱 한번만 하면 된다. 그 이후로 해당 호출부에서 메서드를 호출할 경우를 대비해 캐시하는 것이다.

핫스팟은 자주 쓰이지는 않지만 **이형성 디스패치** 최적화도 지원한다.


## 10.7 인트린직

> JIT 서브시스템이 동적 생성하기 이전에 JVM이 이미 알고 있는, 고도로 튜닝된 네이티브 메서드 구현체

주로 OS나 CPU 아키텍처의 특정 기능을 응용하는, 성능이 필수적인 코어 메서드에 쓰인다.

### 인트린직화한 메서드

|메서드|설명|
|------|---|
|java.lang.System.arraycopy()|CPU의 벡터 지원 기능으로 배열을 빨리 복사한다.|
|java.lang.System.currentTimeMillis()|대부분 OS가 제공하는 구현체가 빠르다.|
|java.lang.Math.min()|일부 CPU에서 분기 없이 연산 가능하다.|
|java.lang.Math 메서드|일부 CPU에서 직접 명령어를 지원한다.|
|암호화 함수(예 : AES)|하드웨어로 가속하면 성능이 매우 좋아진다.|

인트린직은 정말 자주 쓰이는 작업에 한해서만 성능에 큰 영향을 미칠 수 있다.

## 10.8 온-스택 치환

컴파일을 일으킬 정도로 호출 빈도가 높지는 않지만 메서드 내부에 핫 루프가 포함된 경우가 있다. 예를 들어 main() 메서드가 있다.

핫스팟은 이런 코드를 온-스택 치환(OSR)을 이용해 최적화한다.

## 10.9 세이프포인트 복습

### 세이프포인트에 걸리는 경우
* 메서드를 역최적화
* 힙 덤프를 생성
* 바이어스 락을 취소
* 클래스를 재정의

컴파일드 코드에서 세이프포인트 체크 발급은 JIT 컴파일러가 담당하며, 핫스팟에서는 다음 지점에서 세이프포인트 체크 코드를 넣는다.
* 루프 백 브랜치 지점
* 메서드 반환 지점

## 10.10 코어 라이브러리 메서드

### 10.10.1 인라이닝하기 적합한 메서드 크기 상한

인라이닝 여부는 메서드의 바이트코드 크기로 결정되므로 클래스 파일을 정적 분석하면 인라이닝을 하기에 지나치게 큰 메서드들을 골라낼 수 있다.

예를 들어, java.lang.String의 두 메서드 toUpperCase(), toLowerCase()가 있다.

두 메서드의 바이트코드가 커진 까닭은 대/소문자를 바꾸면 저장할 캐릭터 개수가 달라지는 **로케일** 이 있기 때문이다. 따라서 대/소문자 변환 메서드는 이런 경우를 알아서 감지해 배킹 배열의 크기를 재조정하고 복사해서 스트링을 표현해야 한다.

### 도메인에 특정한 메서드로 성능 개선

toUpperCase() 를 도메인에 특정한 메서드로 만들어 바이트코드 크기를 인라이닝 한계치 이하로 줄일 수 있다.

ASCII 전용 구현체를 만들어 컴파일하면 바이트코드가 69바이트밖에 안 된다.

### 메서드를 작게하면 좋은 점

메서드를 작게 유지하면 다양한 인라이닝 트리를 구축해서 핫 경로를 더욱 최적화할 수 있다.

### 10.10.2 컴파일하기 적합한 메서드 크기 상한

핫스팟에는 메서드 크기가 어느 이상 초과하면 컴파일되지 않는 한계치(8000바이트)가 있다.

운영계 JVM에서는 이 수치를 바꿀 수 없지만, 디버그 JVM에서는 ```-XX:HugeMethodLimit=<n> ``` 스위치로 컴파일 가능한 메서드 바이트코드의 최대 크기를 설정할 수 있다.

이런 메서드는 핫 코드에서 발견될 가능성은 크지 않다.

예제를 통해 확인하면, JIT 컴파일 안 된 메서드가 JIT 컴파일드 메서드보다 거의 2배 느리다. 큰 메서드를 생성하지 않는 이유는 여러가지겠지만(ex) 가독성, 유지 보수, 디버깅), 예제와 같은 경우도 있다.

