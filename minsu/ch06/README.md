# 6 가비지 수집 기초
# 6.1 마크 앤 스위프
## GC의 알고리즘인 마크 앤 스위프 알고리즘은?
1. 할당 리스트를 순회하면서 mark bit를 지운다.
2. GC 루트부터 살아 있는 객체를 찾는다.
3. 이렇게 찾은 객체마다 mark bit를 세팅한다.
4. 할당 리스트를 순회하면서 mark bit가 세팅되지 않은 객체를 찾는다.
  - 힙에서 메모리를 회수해 free list (동적 메모리 할당을 위해서 계획적으로 사용된 자료 구조)
  - 할당 리스트에서 객체를 삭제한다.

살아 있는 객체는 대부분 DFS로 찾는다. -> 생성된 객체 그래프를 live object graph라고 하며, transitive closure of reachable objects라고 한다.

> 힙은 VisualVM전용 플러그인을 쓰면 시시각각 변하는 모습을 지켜볼 수 있지만, 그때그때 힙 모습만 봐서는 정확한 분석을 할 수 없으니 GC로그를 이용해야 한다.

## 6.1.1 가비지 수집 용어
### STW
- STW는 Stop The World의 약자로 GC 사이클이 발생하여 가비지를 수집하는 동안에는 모든 애플리케이션 스레드가 중단된다.

### 동시
- GC 스레드는 애플리케이션 스레드와 동시 실행될 수 있다.
- 하지만 이는 계산 비용 면에서 아주 어렵고 비싸며 100% 동시 실행을 보장하는 알고리즘은 없다.

### 병렬
- 여러 스레드를 동원해서 가비지 수집을 한다.

### 정확
- 정확한 GC 스킴은 전체 가비지를 한방에 수집할 수 있게 힙 상태에 관한 충분한 타입 정보를 지니고 있다.
- 대략 int와 포인터의 차이점을 언제나 분간할 수 있는 속성을 지닌 스킴이 정확한 것이다.

### 보수
- 보수적인 스킴은 정확한 스킴의 정보가 없다.
- 그래서 리소스를 낭비하는 일이 잦고 근본적으로 타입 체계를 무시하기 때문에 훨씬 비효율적이다.

### 이동
- 이동 수집기에서 객체는 메모리를 여기저기 오갈 수 있다. 즉, 객체 주소가 고정된게 아니다.

### 압착
- 할당된 메모리는 GC 사이클 마지막에 연속된 단일 영역으로 배열되며, 객체 쓰기가 가능한 여백의 시작점을 가리키는 포인터가 있다.
- 압착 수집기는 메모리 단편화를 방지한다.

### 방출
- 수집 사이클 마지막에 할당된 영역을 완전히 비우고 살아남은 객체는 모두 다른 메모리 영역으로 이동한다.


# 6.2 핫스팟 런타임 개요
- 가비지 수집의 작동 원리르 온전히 이해하기 위해서는 학스팟 내부도 어느 정도 알아야 한다.
- 자바 언어에서는 1. 기본형 (byte, int 등), 2. 객체 래퍼런스 만 사용한다.  
-> 이때 자바는 C++과 달리 주소를 역참조하는 일반적인 메커니즘이 없고, 오직 오프셋 연산자 (. 연산자)만으로 필드에 액세스하거나 객체 레퍼런스의 메서드를 호출할 수 있다. 또 자바는 오직 '값으로 인한 호출' 방식으로만 메서드를 호출한다.

## 6.2.1 객체를 런타임에 표현하는 방법
- 핫스팟은 런타임에 oop(ordinary object pointer)라는 구조체로 자바 객체를 나타낸다.
- oop는 참조형 지역 변수 안에 위치하며 자바 메서드의 스택 프레임으로부터 자바 힙을 구성하는 메모리 내부 영역을 가리킨다.  

### instanceOop
- instanceOop는 자바 클래스의 인스턴스를 나타낸다.
- 모든 객체에 대해 기계어 워드 2개로 구성된 헤더로 시작된다.
    - Mark 워드: 인스턴스 관련 메타데이터를 가리키는 포인터
    - Klass 워드: 클래스 메타데이터를 가리키는 포인터
      -> 자바 8부터는 Klass가 자바 힙의 주 영역 밖으로 빠지게 되어 최신 버전의 자바는 Klass워드가 자바 힙 밖을 가리키므로 객체 헤더가 필요 없다.

  klassOop에는 클래스용 가상 함수 테이블(vtable)이 존재하지만, Class객체에는 리플렉션으로 호출할 method객체의 레퍼런스 배열이 담겨있다.
  ![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/9c1ba985-97ad-4fd4-a6ad-ab20761625fe)

oop에서는 메모리를 절약할 수 있게 압축 oop라는 기법을 제공한다. 

```
-XX:+UseCompressedOops
```

위의 옵션을 주면 힙에 있는 다음 oop가 압축된다.
- 힙에 있는 모든 객체의 Klass 워드
- 참조형 인스턴스 필드
- 객체 배열의 각 원소

핫스팟 객체 헤더는 다음과 같이 구성된다.
- Mark 워드 (32비트 환경은 4바이트, 64비트 환경은 8바이트)
- Klass워드(압축됐을 수도 있다)
- 객체가 배열이면 length 워드 (항상 32비트)
- 32비트 여백(정렬 규칙 때문에 필요할 경우)

JVM 환경에서 자바 레퍼런스는 instanceOop를 제외한 어떤 것도 가리킬 수 없다.
- 자바 값은 기본형 값 또는 instanceOop 주소(레퍼런스)에 대응되는 비트 패턴이다.
- 모든 자바 레퍼런스는 자바 힙의 주 영역에 있는 주소를 가리키는 포인터이다.
- 자바 레퍼런스가 가리키는 주소에는 Mark 워드와 Klass 워드가 들어있다.
- klassOop와 Class<?> 의 인스턴스는 다르며 klassOop(힙의 메타데이터 영역에 있음)을 자바 변수에 넣을 수 없다.

핫스팟의 oop 체계를 까보면 다음과 같다. (hotspot/src/share/vm/oops )
```
oop (추상 베이스)
 instanceOop (인스턴스 객체)
 methodOop (메서드 표현형)
 arrayOop (배열 추상 베이스)
 symbolOop (내부 심볼 / 스트링 클래스)
 klassOop (Klass 헤더) (자바 7 이전만 해당)
 markOop
```

## 6.2.2 GC 루트 및 아레나
GC 루트는 메모리의 고정점(앵커 포인트, anchor point)으로 메모리 풀 외부에서 내부를 가리키는 포인터이다. 즉, 메모리 풀 내부에서 같은 메모리 풀 내부의 다른 메모리 위치를 가리키는 내부 포인터(internal pointer)와 정반대인 외부 포인터이다.

다음과 같은 종류가 있다.
- 스택 프레임(stack frame)
- JNI
- 레지스터(호이스트된 변수)
- 코드 루트
- 전역 객체
- 로드된 클래스의 메타데이터

핫스팟 GC는 아레나라는 메모리 영역에서 작동한다. **핫스팟은 자바 힙을 관리할 때 시스템 콜을 하지 않는다.**


## 6.3 할당과 수명
자바 애플리케이션에서 가비지 수집이 일어나는 주된 원인은 다음 두가지이다.
### 할당률
- 일정 기간 동안 새로 생성된 객체가 사용한 메모리량
- 비교적 쉽게 측정이 가능하고 센섬 같은 툴을 통해서 구할 수 있다.

### 객체 수명
- 측정하기가 어렵다.
- 가비지 수집은 '메모리를 회수해 재사용'하는 일이기에 가비지 수집기는 할당 및 수명과 연관되어 있다.

## 6.3.1 약한 세대별 가설
소프트웨어 시스템의 런타임 작용을 관찰한 결과 알게 된 경험 지식이며, JVM 메모리 관리의 이론적 근간을 형성한다.

```
JVM 및 유사 소프트웨어 시스템에서 객체 수명은 이원적 분포 양상을 보인다.  
거의 대부분의 객체는 아주 짧은 시장만 살아 있지만, 나머지 객체는 기대 수명이 크다.
```
-> 결론: 가비지를 수집하는 힙은, 단명 객체를 쉽고 빠르게 수집할 수 있게 설계해야 하며, 장수 객체과 단명 객체를 완전히 떼어놓는게 가장 좋다.

핫스팟은 몇 가지 매커니즘을 응용하여 약한 세대별 가설을 활용한다. 
- 객체마다 세대 카운트(객체가 지금까지 무사 통과한 가비지 수집 횟수)를 센다.
- 큰 객체를 제외한 나머지 객체는 에덴 공간에 생성한다. 여기서 살아남은 객체는 다른 곳으로 옮긴다.
- 장수했다고 할 정도로 충분히 오래 살아남은 객체들은 별도의 메모리 영역(올드 또는 테뉴어드)에 보관한다.
  
<img src = "https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/944e48ea-35d2-4cb0-8fbb-579b154797f7" width="100%">


핫스팟은 카드 테이블이라는 자료 구조에 늙은 객체가 젊은 객체를 참조하는 정보를 기록한다.
자바 수집기는 힙을 영/올드 영역으로 나누어서 관리해왔는데, 자바 8u40 버전부터 새로운 수집기의 품질이 완성 단계에 이르렀다. 

# 6.4 핫스팟의 가비지 수집
자바는 C/C++와 달리 OS를 이용해 동적으로 메모리를 관리하지 않는다. 대신, 일단 프로세스가 시작하면 JVM은 메모리를 할당하고 유저 공간에서 연속된 단일 메모리 풀을 관리한다.  

이 메모리 풀은 각자의 목적에 따라 서로 다른 영역으로 구성되며 객체는 보통 에덴 영역에 생성된다. 수집기는 객체를 이동시키는데 객체가 차지한 주소는 대부분 시간이 흐르면서 아주 빈번하게 바뀐다. 이처럼 객체를 이동시키는 것은 '방출'이라고 하며, 핫스팟 수집기는 대부분 방출 수집기이다.

## 6.4.1 스레드 로컬 할당
JVM은 성능을 강화해서 에덴을 관리하며, 에덴은 대부분의 객체가 탄생하는 장소이다. 특히 수명이 짧은 객체는 다른 곳에는 위치할 수 없으므로 특별히 관리를 잘해야한다. 

JVM은 에덴을 여러 버퍼로 나누어 각 애플리케이션 스레드가 새 객체를 할당하는 구역으로 활용하도록 배포한다. 이 때 이 구역을 스레드 로컬 할당 버퍼(TLAB, Thread-Local Allocation Buffer)라고 한다.

<img src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/ba77f900-3f11-4ed2-9ae9-af6fb7cd0bc3" width="100%">

각 애플리케이션 스레드가 새 객체를 할당할 버퍼를 갖고 있으며 애플리케이션 스레드가 버퍼를 다 채우면 JVM은 새 에덴 영역을 가리키는 포인터를 내준다.


## 6.4.2 반구형 수집
반구형 수집기는 두 공간을 사용하는 독특한 방출 수집기로 오래 살지 못한 객체를 임시 수용소에 담아 두는 아이디어이다. 이 공간은 두가지의 기본 특징을 가진다.

- 수집기가 라이브 반구를 수집할 때 객체들은 다른 반구로 압착시켜 옮기고 수집된 반구는 비워서 재사용한다.
- 절반의 공간은 항상 완전히 비운다.

핫스팟은 이 반구형 기법과 에덴 공간을 접목시켜서 영 세대 수집을 한다. 핫스팟에서는 영 힙의 반구부를 서바이버(survivor) 공간이라고 한다.


# 6.5 병렬 수집기
자바 병렬 수집기도 여러개가 있.

### Parallel GC
가장 단순한 영 세대용 병렬 수집기

### ParNew GC
CMS 수집기와 함께 사용할 수 있게 Parallel GC를 조금 변형한 것이다.

###ParallelOld GC
올드 세대용 병렬 수집기이다.

## 6.5.1 영 세대 병렬 수집
- 영세대 수집은 가장 흔한 가비지 수집 형태이다.
- 스레드가 에덴에 객체를 할당하려는데 자신이 할당받은 TLAB 공간은 부족하고 JVM은 새 TLAB을 할당할 수 없을 때 영 세대 수집이 발생한다.
- 영 세대 수집이 일어나면 JVM은 전체 애플리케이션 스레드를 중단시킨다.

![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/c3f8d892-49b6-4b88-92fe-887be03596e1)
전체 애플리케이션 스레드가 중단되면 핫스팟은 영 세대를 뒤져 가비지 아닌 객체를 골라낸다.

![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/833016c6-7a95-4a8d-89cd-c087f6a21758)
Parallel GC는 살아남은 객체를 현재 비어 있는 서바이버 공간으로 모두 방출한 후, 세대 카운트를 늘려 한 차례 이동했음을 기록한다. 
마지막으로, 에덴과 이제 막 객체들을 방출시킨 서바이버 공간을 재사용 가능한 빈 공간으로 표시하고, 애플리케이션 스레들르 재시작해 TLAB를 애플리케이션 스레드에 배포하는 프로세스를 재개한다.

## 6.5.2 올드 세대 병렬 수집
올드 세대에 더 이상 방출할 공간이 없으면 병렬 수집기는 올드 세대 내부에서 객체들을 재배치해서 늙은 객체가 죽고 빠져 버려진 공간을 회수하려고 한다.
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/59d1de52-a294-462b-b39e-4c390e7a759d)


올드 공간은 크게 눈에 띄는 변화가 없다. 때때로 큰 객체가 테뉴어드 세대에 직접 생성되는 경우도 있지만, 그 외에는 영 세대 객체가 승격되거나 올드/풀 수집이 일어나 객체를 재탐색 후 다시 패치하는 등의 수집이 일어날 때만 변한다.


# 6.5.3 병렬 수집기의 한계
병렬 수집기는 세대 전체 콘텐츠를 대상으로 한번에 가능한 한 효율적으로 가비지를 수집한다. 다만 이러한 설계에도 단점이 있다. 

### 풀 STW
- 힙 크기가 커질수록 느려진다. (비례한다)
- 영역 내 살아 있는 객체 수만큼 마킹 시간이 늘어난다.

# 6.6 할당의 역할
- 자바의 가비지 수집 프로세스는 보통 유입된 메모리 할당 요청을 수용하기에 메모리가 부족할 때 작동하여 필요한 만큼 메모리를 공급한다. 즉, GC 사이클은 어떤 고정된 예측 가능한 일정에 맞춰 발생하는 것이 아니라 순전히 필요로 할 때 발생한다. (불확정적+불규칙)

- GC가 발생하면 모든 애플리케이션 스레드가 멈춘다. (객체를 생성할 수 없으므로 오래 실행될 자바 코드가 사실상 없다.) JVM은 모든 코어를 총동원해 가비지를 수집하고 메모리를 회수한 후, 애플리케이션 스레드를 재개한다.

- 가비지 수집은 일정한 주기마다 실행되는 것이 아니라 필요에 따라 그때마다 실행된다. 따라서 할당률이 높을수록 GC는 더 자주 발생합한다. 할당률이 너무 높은 경우에는 테뉴어드로 곧장 승격이 되는데 이를 조기 승격이라고 한다.,

