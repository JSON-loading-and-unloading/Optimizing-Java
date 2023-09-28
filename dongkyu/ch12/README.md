# Chapter 12 동시 성능 기법

- 최신 하드웨어는 멀티코어 프로세서가 일반적이라 잘 만든 애플리케이션은 멀티코어에 부하를 고루 분산시킨다.
- JVM은 여러 프로세서 코어를 십분 활용 가능 VM 스레드 덕에 애플리케이션 스레드가 하나뿐이라도 멀티코어의 혜택을 누린다.
- 자바 개발자는 최신 하드웨어를 오롯이 활용하려면 동시성이 무엇인지, 애플리케이션 성능에 끼치는 영향을 무엇인지 알아야 한다.

## 12.1 병렬성이란

- 암달의 법칙에 의해 프로세서 수를 늘려도 순차 작업 시간 이상 총 소요 시간을 줄일 수 없다.
    - 병렬 태스크와 순차 태스크 간 소통의 필요가 전혀 없다면 이론적으론 속도를 무한히 높일 순 있다고 한다.
- 보통은 데이터 공유 없이 워크로드를 나누어 여러 스레드에 분산시킨다.
- 스레드끼리 상태나 데이터를 공유하기 시작하면 워크로드가 복잡해지고 일부 테스크는 순차 처리하게 된다.
    - 통신 오버헤드 발생
- 상태를 공유하는 워크로드는 무조건 정교한 보호/제어 장치가 필요하다.

### 12.1.1 자바 동시성 기초

- 아래 카운터를 락으로 적절히 보호하지 않은 상태로 멀티스레드 환경에서 실행하면 다른 스레드가 수정한 결과가 소실될 수도 있다.

    ```java
    public class Counter {
        private int i = 0;
    
        public int increment() {
            return i = i + 1;
        }
    }
    ```

    - 스레드는 메서드 개별 진입 시 각자 전용 평가 스택을 소유
    - 객체 필드는 힙에 위치하기에 모든 스레드가 공유 가능하기 때문
    - `synchronized` 키워드로 해결할 수 있고 자바 5 이전엔 이 방법이 유일했다.
- 동시 프로그램은 버그를 재연하기도 어렵다.
- 동기화를 사용할 땐 신중해야 하는데 `synchronized` 때문에 프로그램이 더 느려질 수도 있기 때문
    - 처리율 향상은 동시성과 상충되는 목표
    - 성능 테스트가 수반되어야 함

## 12.2 JMM의 이해

- JMM은 다음 질문에 답을 찾는 모델이다.
    - 두 코어가 같은 데이터를 엑세스하면 어떻게 되는가?
    - 언제 두 코어가 같은 데이터를 바라본다고 장담할 수 있는가?
    - 메모리 캐시는 위 두 질문에 답에 어떤 영향을 미치는가?
- 자바 플랫폼은 공유 상태를 어디서 엑세스하든 JMM이 약속한 내용을 반드시 이행한다.
    - 순서에 관한 보장
    - 여러 스레드에 대한 업데이트 가시성 보장

### 강한 메모리 모델과 약한 메모리 모델

- 고수준에서 JMM 같은 메모리 모델은 두 가지 방식으로 접근한다.
    - 강한 메모리 모델: 전체 코어가 항상 같은 값을 바라본다.
    - 약한 메모리 모델: 코어마다 다른 값을 바라볼 수 있고 그 시점을 제어하는 특별한 캐시 규칙이 있다.
- 최신 멀티 CPU 시스템에서 강한 메모리 모델의 문제
    - 사실상 메모리를 후기록(write-back)하는 것과 같다.
    - 캐시 무효화 알림이 메모리 버스를 잠식
    - 실제 메인 메모리 전송률 급락
    - 코어 수를 늘리는 건 상황을 악화시킬 뿐
- 강한 메모리 모델은 자바의 아키텍처 독립적 환경으로 설계된 플랫폼이라는 취지에도 맞지 않는다.
    - 네이티브 수준에서 강한 메모리 모델을 지원하지 않는다면 별도의 구현 작업이 필요해진다.
- JMM은 아주 약한 메모리 모델이기에 실제 CPU 아키텍처 추세와 잘 어울린다.
    - 또 JMM은 보장하는 내용이 거의 없어 이식 작업이 쉽다.
- 강한 메모리 모델 기반 플랫폼에서 개발한 애플리케이션은 동시성 버그를 갖고 있을 가능성이 있다.
    - 하드웨어가 보장해주는 탓에 동시성 버그가 드러나지 않음
    - 약한 메모리 모델에 배포하면 동시성 문제가 터질 수도 있는 것
    - 개발자가 그릇된 보안 감각에 빠지기 쉽다.

### JMM 기본 개념

- JMM은 다음 기본개념을 기반으로 애플리케이션을 보호한다.
    - Happens-Before: 한 이벤트는 무조건 다른 이벤트보다 먼저 발생한다.
    - Synchronizes-With: 이벤트가 객체 뷰를 메인 메모리와 동기화시킨다.
    - As-If-Serial: 실행 스레드 밖에서는 명령어가 순차 실행되는 것처럼 보인다.
    - Release-Before-acquire: 한 스레드에 걸린 락을 다른 스레드가 그 락을 획득하기 전에 해제한다.
- 자바에서 스레드는 객체 상태 정보를 스스로 들고 다닌다.
    - 스레드가 변경 내용은 메모리로 곧장 반영된다.
    - 같은 데이터를 엑세스하는 다른 스레드가 반영된 데이터를 다시 읽는 구조
- 위와 같은 맥락에서 `sychronized` 키워드가 나타내는 의미
    - ‘모니터를 장악한 스레드의 로컬 뷰가 메인 메모리와 동기화(Sychronizes-With)되었다.’
- 동기화 메서드, 동기화 블록은
    - 스레드가 반드시 동기를 맞춰야할 접점
    - 다른 동기화 메서드/블록이 시작되기 전 반드시 완료되어야 할 코드 블록
- JMM은 동기화되지 않은 엑세스에 대해선 동시성을 전혀 보장하지 않는다.
    - 쓰기에 그런 보장이 필요하면 동기화 블록으로 감싸 캐시된 값을 메모리에 후기록해야 한다.
    - 읽기 엑세스도 동기화 블록 내부에 넣어 강제로 메모리를 읽도록 해야 한다.

### synchronized 락의 한계

- 락에 걸린 객체에서 일어나는 동기화 작업은 모두 균등하게 취급된다.
    - 때문에 쓰기 작업에만 `synchronized`를 적용하면 소실된 업데이트 현상이 나타난다.
- 락 획득/해제는 반드시 메서드 수준이나 메서드 내부 동기화 블록 안에서 이루어져야 한다.
- 락을 얻지 못한 스레드는 블로킹되어 락을 얻지 못하면 실행이 불가능하다.

## 12.3 동시성 라이브러리 구축

- JMM은 성공적인 작품이긴 하지만 이해가 어렵고 응용은 더 어렵다.
    - 인트린직 락킹의 유연성도 흠이다.
- 자바 5부터 고급 동시성 라이브러리와 툴을 지원하기 시작했다.
- `java.util.concurrent` 패키지는 멀티스레드 애플리케이션을 더 쉽게 개발할 수 있게 설계된 라이브러리다.
    - 락, 세마포어(semaphore)
    - 아토믹스(atomics)
    - 블로킹 큐
    - 래치
    - 실행자(executor)
- 일반적으로 라이브러리는 OS 품에서 벗어나 유저 공간에서 더 많은 일을 하려고 한다.
    - 플랫폼 독립적으로 전역 범위에서 일관성을 보장한다는 측면에서 중요
- 일부 라이브러리는 CAS(compare and swap) 기법을 구현하기 위해 저수준 프로세서 명령어 및 OS 특성을 활용한다.
    - 락, 아토믹스 등
    - CAS: 예상되는 현재 값을 메모리 위치의 콘텐츠와 비교한 후 일치하면 현재 값을 원하는 새 값으로 교체하는 아토믹 유닛

### 12.3.1 Unsafe

- CAS 하드웨어는 `sun.misc.Unsafe` 클래스를 통해 엑세스한다.
    - 표준 자바 플랫폼 API는 아니다.
    - 개발자가 직접 사용할 일은 거의 없다.
    - 이 클래스는 핫스팟 VM에 직접 연결되고 깨질 우려가 높다.
- JVM 표준 로직을 무너뜨리는 `Unsafe`는 거의 모든 주요 프레임워크의 구현 핵심부가 됐다.
    - 객체는 할당하지만 생성자는 아직 실행하지 않는다.
    - 원메모리에 엑세스하고 포인터 수준의 연산을 수행
    - 프로세서별 하드웨어 특성(CAS)를 이용한다.
- 덕분에 같은 고수준의 프레임워크 기능을 구현 가능
    - 신속한 (역)직렬화
    - thread-safe한 네이티브 메모리 엑세스
    - 아토믹 메모리 연산
    - 효율적인 객체/메모리 레이아웃
    - 커스텀 메모리 펜스
    - 네이티브 코드와의 신속한 상호작용
    - JNI에 관한 다중 운영체제 대체물
    - 배열 원소에 volatile하게 엑세스
- 그러나 위와 같은 것들은 자바 9부터 영향을 받게 되어 자바 버전 몇 개에 걸쳐 크게 변할 가능성이 크다.

### 12.3.2 아토믹스와 CAS

- 아토믹스는 값을 더하고 증감하는 복합 연산을 통해 `get()`으로 계산한 결과를 돌려 받는다.
    - 아토믹 변수는 `volatile` 확장판이라고 할 수 있으며 더 유연해서 상태 의존적 업데이트가 가능
- 아토믹스는 자신이 감싸는 베이스 타입을 상속하지 않는다.
    - ex) `AtomicInteger`는 `Integer`를 상속하지 않음
- `Unsafe`를 이용한 아토믹 구현 원리

    ```java
    public class AtomicIntegerExample extends Number {
    
        private volatile int value;
    
        // Unsafe.compareAndSwapInt로 업데이트하기 위해 설정
        private static final Unsafe unsafe = Unsafe.getUnsafe();
        private static final long valueOffset;
    
        static {
            try {
                valueOffset = unsafe.objectFieldOffset(
                    AtomicIntegerExample.class.getDeclaredField("value"));
            } catch (Exception ex) {
                throw new Error(ex);
            }
        }
    
        public final int get() {
            return value;
        }
    
        public final void set(int newValue) {
            value = newValue;
        }
    
        public final int getAndSet(int newValue) {
            return unsafe.getAndSetInt(this, valueOffset, newValue);
        }
        // 생략
    }
    ```

    - `Unsafe`의 `getAndSetInt()` 메서드를 사용했고 JVM을 호출하는 네이티브 코드가 핵심

    ```java
    public final int getAndSetInt(Object o, long offset, int newValue) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, newValue));
        return v;
    }
    
    public native int getIntVolatile(Object o, long offset);
    
    public final native boolean compareAndSwapInt(Object o, long offset, 
                                                  int expected, int x);
    ```

    - `Unsafe` 내부에서 루프를 이용해 CAS를 반복 시도한다.
    - 루프를 이용한 아토믹 증분 연산이 `Unsafe`에 이미 존재하는 것
- 아토믹은 락-프리하므로 데드락은 있을 수 없다.
- 변수 업데이트를 위해 여러 차례 재시도를 하면 횟수만큼 성능이 나빠진다.
    - 처리율을 높게 유지하려면 경합 수준을 잘 모니터링 해야 한다.

### 12.3.3 락과 스핀락

- 인트린직 락은 OS를 이용해 스레드가 신호를 줄 때까지 기다리기 만들기 때문에 막대한 오버헤드를 유발할 수 있다.
- 스핀락(spinlock)은 완전히 상호 베타적인 락보다 가볍게 쓸 수 있는 락이다.
    - 블로킹된 스레드를 CPU에 활성 상태로 놔두고 아무 일도 시키지 않은 채 락을 얻을 때까지 ‘CPU를 태워가며’ 재시도하게 만드는 락
    - 최신 시스템은 CAS로 스핀락을 구현한다.
- 스핀락을 구현하는 코드는 CPU마다 다르겠지만 핵심 개념은 동일하다.
    - ‘테스트하고 세팅’하는 작업은 반드시 아토믹해야 한다.
    - 스핀락에 경합이 발생하면 대기 중인 프로세서는 tight loop를 실행하게 된다. (명령어는 별로 없는데 아주 많이 반복되는 루프)
- CAS는 값이 정확하다면 한 명령어로 값을 안전하게 업데이트하며 락 구성요소를 형성한다.

## 12.4 동시 라이브러리 정리

### 12.4.1 java.until.concurrent 락

- 자바 5부터 개변된 락 인터페이스 `java.util.concurrent.locks.Lock`이 추가되었다.
- 인트린직 락보다 더 많은 일을 할 수 있다.
- `lock()`
    - 기존 방식처럼 락을 획득하고, 사용할 수 있을 때까지 블로킹
- `newCondition()`
    - 락 주위에 조건을 설정해 더 유연하게 락을 활용
    - 락 내부에서 관심사를 분리할 수 있다. (읽기와 쓰기)
- `tryLock()`
    - 락을 획득하려고 시도하지만 타임아웃 옵션 설정 가능
    - 스레드가 락 획득에 실패해도 계속 처리를 진행할 수 있다.
- `unlock()`
    - `lock()`에 대응되는 후속 호출로 락을 해제한다.
- `ReentrantLock`
    - `Lock`의 주요 구현체로 내부적으로 `int` 값으로 `compareAndSwap()`을 한다.
    - 경합이 없다면 락을 획득하는 과정이 락-프리하다.
    - 경합이 거의 없는 시스템에선 성능이 매우 좋아지고 다양한 락킹 정책을 적용가능하다.
- `compareAndSet()`을 호출하고 `Unsafe`를 사용하는 `Sync`는 스레드를 파킹 및 재개하는 메서드가 구현된 `LockSupport` 클래스를 사용한다.
    - `Sync`는`AbstractQueuedSynchronizer`를 확장한 정적 서브 클래스
    - `LockSupport` 클래스는 스레드에게 퍼밋(허가증)을 발급한다. (퍼밋이 없으면 스레드는 기다려야 함)
    - LockSupport는 오직 한 가지 퍼밋만 발급 (바이너리 세마포어)

      ```java
      while (!canProceed()) { ... LockSupport.park(this); }}
      ```


### 12.4.2 읽기/쓰기 락

- 대부분 읽기와 쓰기 작업 횟수는 많이 차이가 난다.
    - 읽기는 상태를 바꾸지 않음
- 기존 `synchronized`나 (조건 없는) `ReentrantLock`을 쓰면 한 가지 락 정책을 따를 수밖에 없다.
    - 어느 한 읽기 스레드로 인해 나머지 읽기 스레드가 불필요하게 블로킹되어 시간 허비
- `ReentrantReadWriteLock` 클래스를 활용하면 여러 읽기 스레드 작업 중 다른 읽기 스레드를 블로킹하지 않을 수 있다.

    ```java
    public class AgeCache {
    
        private final ReentratReadWriteLock rwl = new ReentrantReadWriteLock();
        private final Lock readLock = rwl.readLock();
        private final Lock writeLock = rwl.writeLock();
        private Map<String, Integer> ageCache = new HashMap<>();
    
        public Integer getAge(String name) {
            readLock.lock();
            try {
                return ageCache.get(name);
            } finally {
                readLock.unLock();
            }
        }
    
        public void updateAge(String name, int newAge) {
            writeLock.lock();
            try {
                ageCache.put(name, newAge);
            } finally {
                writeLock.unLock();
            }
        }
    }
    ```


### 12.4.3 세마포어

- 세마포어는 풀 스레드나 DB 커넥션 등 여러 리소스의 엑세스를 허용하는 독특한 기술을 제공
    - ‘최대 O개 객체까지만 엑세스를 허용’

      ```java
      // 퍼밋은 2개, 공정 모드로 설정된 세마포어 생성
      private Semaphore poolPermits = new Semaphore(2, true);
      ```

- `Semaphore` 클래스의 `acquire()` 메서드로 사용 가능한 퍼밋 수를 하나씩 줄인다.
    - 퍼밋이 소진되면 블로킹
- `release()` 메서드는 퍼밋을 반납하고 대기 중인 스레드 중 하나에게 퍼밋을 전달
- 퍼밋이 하나뿐인 세마포어는 뮤텍스와 동등
    - 그러나 뮤텍스는 뮤텍스가 걸린 스레드만 해제 가능
    - 세마포어는 비소유 스레드도 해제할 수 있다.

### 12.4.4 동시 컬렉션

- `Map` 구현체 (`ConcurrentHashMap`)
    - 버킷 또는 세그먼트로 분할된 구조를 최대한 활용
    - 각 세그먼트가 자체 락킹 정책, 즉 자신만의 락 세트를 가진다.
    - 여러 읽기 스레드가 `Map` 곳곳을 읽는 동안 쓰기가 필요할 경우 한 세그먼트에만 락을 거는 것도 가능
- 이터레이터 (및 병렬 스트림용 스플릿터레이터)
    - 일종의 스냅샷으로 획득하기에 `ConcurrentModiFicationException`이 거의 발생하지 않는다.
    - 충돌이 많을 경우 테이블이 동적으로 팽창하여 비용이 많이 들기에 크기를 미리 지정하는 편이 좋다.
- `CopyOnWriteArrayList`, `CopyOnWriteArraySet`
    - 자료 구조를 변경하면 배킹 배열 사본이 하나 더 생성
    - 덕분에 예전 배열을 계속 탐색할 수 있고 레퍼런스가 없게 되면 이전 배열 사본은 GC 대상이 된다.
    - 스냅샷 스타일이기에 `ConcurrentModiFicationException이` 발생하지 않는다.

### 12.4.5 래치와 배리어

- 래치와 배리어는 스레드 세트의 실행을 제어하는 유용한 기법이다.
- 모든 스레드가 테스크1 → 테스크2 → 테스크3로 진행되는 것이 이상적이라면 래치를 쓰기 좋은 경우다.

    ```java
    public class CountdownLatchEx implements Runnable {
    
        private final CountDownLatch latch;
    
        public CountdownLatchEx(CountDownLatch latch) {
            this.latch = latch;
        }
    
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " api call done");
            try {
                latch.countDown();
                latch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " continue processing");
        }
    
        public static void main(String[] args) throws InterruptedException {
            CountDownLatch apiLatch = new CountDownLatch(5);
    
            ExecutorService pool = Executors.newFixedThreadPool(5);
            for (int i = 0; i < 5; i++) {
                pool.submit(new CountdownLatchEx(apiLatch));
            }
    
            System.out.println(Thread.currentThread().getName() + " await main");
            apiLatch.await();
            pool.shutdown();
            try {
                pool.awaitTermination(5, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " all processing complete");
        }
    }
    ```

    - 래치 카운트를 5로 세팅하고 각 스레드가 `countdown()`을 호출할 때마다 카운트 값이 1씩 감소한다.
    - 카운트가 0이 되면 래치가 열리고 `await()`에 매여 있든 스레드가 해제된다.

        ```
        pool-1-thread-5 api call done
        main await main
        pool-1-thread-3 api call done
        pool-1-thread-1 api call done
        pool-1-thread-4 api call done
        pool-1-thread-2 api call done
        pool-1-thread-2 continue processing
        pool-1-thread-5 continue processing
        pool-1-thread-3 continue processing
        pool-1-thread-1 continue processing
        pool-1-thread-4 continue processing
        main all processing complete
        ```

## 12.5 실행자와 태스크 추상화

- 저수준 스레드 문제를 직접 처리하기 보단 `java.util.concurrent` 패키지의 적절히 추상화된 기능을 골라 쓰는 편이 낫다.
- 스레딩 문제가 거의 없는 추상화 수준은 동시 테스크로 기술할 수 있다.
    - 현재 실행 컨텍스트 내에서 동시 실행해야 할 코드나 작업 단위
    - 일의 단위를 태스크로 바라보면 동시 프로그래밍을 단순화할 수 있다.
    - 태스크를 실행하는 실제 스레드 수명 주기를 개발자가 신경 쓸 필요가 없기 때문

### 12.5.1 비동기 실행이란?

### Callable

- `Callable` : 자바에서 태스크를 추상화하는 방법
    - `Callable<V>`은 `call()` 메서드만 존재하는 제네릭 인터페이스
    - `V` 타입을 반환하되 결과값을 계산할 수 없으면 예외를 던진다.
    - `Runnable`은 결과값과 예외를 반환하지 않는다는 점이 `Callable`과 다르다.

> 자바 스레드는 OS 수준 프로세스와 동등하며 생성 비용이 비싼 경우도 있다. `Runnable`에서 결과를 가져오는 방식은 다른 스레드를 상대로 실행 반환을 조정해야 하기에 복잡도가 가증될 수 있다.
>

### ExecutorService

- `ExecutorService`: 관리되는 스레드 풀에서 태스크 실행 메커니즘을 규정한 인터페이스
    - `submit()` 메서드를 통해 `Runnable` 또는 `Callable` 객체를 받는다.
    - 태스크가 제출되면 비동기로 처리된다.
    - `submit()` 메서드가 반환하는 `Future<V>` 객체의 `get()` 메서드를 통해 결과를 받을 때까지 블로킹할 수 있고 `isDone()`으로 논블로킹 호출할 수도 있다.
- `Executors`: 로직에 따라 서비스 및 기반 스레드 풀(`ExecutorService`)을 생성하는 `new*` 팩토리 메서드를 제공하는 헬퍼 클래스
    - `newFixtedThreadPool(int nThreads)`
        - 크기가 고정된 스레드 풀을 가짐.
        - 풀 내부의 스레드는 재사용되며 여러 테스크를 실행
        - 스레드가 전부 사용 중일 경우 태스크는 큐에 보관
    - `newCachedThreadPool()`
        - 필요한 만큼 스레드를 생성하지만 가급적 재사용
        - 생성된 스레드는 60초간 유지되며 이후엔 캐시에서 삭제된다.
        - 소규모 비동기 태스크 성능을 향상시킬 수 있다.
    - `newSingleThreadExecutor()`
        - 스레드 하나만 가동되는 스레드 풀
        - 새 태스크는 스레드를 이용할 수 있을 때까지 큐에서 대기
        - 동시 실행 테스크 개수를 제한할 경우 유용
    - `newScheduledThreadPool(int corePoolSize)`
        - 미래 특정 시점에 태스크를 실행시킬 수 있다..
        - `Callabler`과 지연 시간을 전달 받는 메서드가 존재

### 12.5.2 ExecutorService 선택하기

- 올바른 `ExecutorService` 선택의 장점
    - 비동기 프로세스를 잘 제어할 수 있다.
    - 풀 스레드 개수를 잘 정하면 성능이 향상된다.
- 직접 ExecutorService를 작성할 일은 별로 없다.
    - `ThreadFactory`를 이용해 커스텀 스레드 생성기를 작성할 수는 있다.
- 전체 애플리케이션 설정에 따라 ExecutorService를 경험적으로 튜닝해야 할 경우도 있다.
    - 흔히 사용되는 지표는 코어 수 대비 풀 스레드 수
    - 코어 개수보다 너무 높게 잡으면 경합이 발생하는 문제가 발생 (잦은 컨텍스트 스위칭)

### 12.5.3 포크/조인

- `ForkJoinPool`
    - `ExecutorService` 구현체
    - 자바 7부터 등장한 포크(분기)/조인(병합) 프레임워크
    - 멀티 프로세서 환경에서 효율적으로 동작하는 새로운 API를 제공
- `ForkJoinPool`의 두 가지 특성
    - 하위 분할 테스크(subdivided task)를 효율적으로 처리 가능
    - 작업 빼앗기(work-stealing) 알고리즘을 구현
- 하위 분할 테스크: 표준 자바 스레드보다 가벼운 스레드와 비슷한 엔티티로 `ForkJoinTask` 클래스가 지원
    - 적은 수의 실제 스레드가 아주 많은 태스크/서브태스크를 담당해야 하는 유스케이스에 주로 사용
- 작업 빼앗기 알고리즘은 태스크 하위 분할과 독립적으로 응용할 수 있다.
    - ex) 어느 스레드가 자신의 작업을 모두 마쳤는데 다른 스레드에 아직 백로그(큐에 쌓인 작업들)가 남아 있으면 바쁜 스레드 큐에서 작업을 몰래 가져와 실행할 수 있다.
- `ForkJoinPool`에 있는 `commonPool()` 정적 메서드로 전체 시스템 풀의 레퍼런스를 반환할 수 있다.
    - 개발자가 직접 자체 풀을 생성해 공유할 필요가 없다.
    - 공용 풀은 필요한 시점에 지연 초기화된다.
    - 풀 크기는 `Runtime.getRuntime().availableProcessors() - 1`로 정해지지만 항상 기대한 값을 반환하지는 않는다.
- 자바 8 이후 `parallelStream()` 때문에 포크/조인 활용 범위는 크게 확대되었다.

## 12.6 최신 자바 동시성

- 스레드는 현대 애플리케이션 개발에 있어 훨씬 더 저수준으로 추상화한 산물이 되었다.
- 현대 자바는 표준 라이브러리에 내장된 추상화를 이용해 성능을 높일 수 있는 환경을 제공한다.

### 12.6.1 스트림과 병렬 스트림

- 자바 8의 가장 큰 변경 사항은 람다와 스트림이다.
    - 자바 개발자도 함수형 프로그래밍의 혜택을 누릴 수 있게 되었다.
- 자바 스트림은 데이터 소스에서 원소를 퍼 나르는 불변 데이터 시퀸스이다.
    - 모든 타입의 데이터 소스(컬렉션, I/O)에서 추출할 수 있다.
    - 기존 루프를 내부 이터레이션(스트림)으로 변형했기에 데이터를 병렬화하거나 복잡한 표현식의 평가를 지연시킬 수도 있다.
- `Collections` 인터페이스의 `stream()` 메서드로 스트림을 생성할 수 있다.
    - 내부에서 `ReferencePipeline`을 생성
- `parallelStream()`을 사용하면 병렬로 데이터를 작업 후 결과를 재조립할 수 있다.
    - 내부족으로 `Spliterator`를 써서 작업을 분할하고 공용 포크/조인 풀에서 연산을 수행
- 성능 향상을 꾀하고자 병렬 스트림으로 바꿀 경우 측정한 뒤에 바꿔야 한다.
    - 컬렉션이 작을수록 병렬보다 직렬 연산이 빠르다.
    - 태스크를 찢어 스레드에 분배하고 다시 취합하는 일을 피할 수 없기 때문

### 12.6.2 락-프리 기법

- 락-프리하지 않은 블로킹은 처리율에 악영향을 미치고 성능을 저하시킬 수 있다.
    - 스레드를 컨텍스트 교환할 기회가 있다는 사실을 OS에 의지한다.
    - 중단/재개시키는 과정에서 많은 시간이 소요될 수 있기에 락-프리한 기법보다 훨씬 느리다.
- 락-프리 기법은 CPU 코어를 계속 스피닝하여 컨텍스트 교환 없이 해당 코어에서 작업할 준비를 한다는 뜻이다.

    ```java
    private volatile int proceedValue;
    
    while (i != proceedValue) {
        // 작업량이 많은 루프
    }
    ```

- 락-프리 기법 역시 대가는 따른다.
    - CPU 코어를 차지하면서 사용률, 전력 소비 측면에서 비용이 든다.

### 12.6.3 액터 기반 기법

- 액터(actor, 실행기) 패러다임도 `ForkJoinTask`처럼 태스크를 스레드 하나보다 더 작게 나타내는 접근 방식이다.
- 액터는 그 자체로 고유한 상태와 로직을 갖는다.
    - 다른 액터와 소통하는 메일박스 체계를 갖춘 작고 독립적인 단위
    - 가변 상태는 일체 공유하지 않고 불변 메시지를 통해서만 상호 통신
    - 액터 간 통신은 비동기적이며 메시지 수신에 반응하여 정해진 일을 한다.
- 액터는 병렬 시스템 내부에서 하나의 네트워크를 생성하고 각자 작업을 수행함으로써 하부 동시 모델을 완전히 추상화한다.
- 액터는 동일한 프로세스 내부에서 존재하지만 멀티 머신에 걸쳐 있는 상태로도 작동이 가능하다.
    - 액터 기반 시스템은 멀티 머신과 클러스터링 덕분에 어느 정도 내고장성이 필요한 상황에서 효과적이다.
    - 대부분 액터를 잘 작동시키기 위해 페일-패스트 전략을 구한다. (실패 가능성 및 조건을 즉시 보고)
- JVM 계열 언어에선 스칼라 언어로 작성된 아카(Akka) 프레임워크가 유명한데 (자바 API도 제공) 전톡적 락킹보다 아카를 쓰는 것이 더 좋은 3가지 이유가 공식 문서에 있다.
    - 도메인 모델 내의 가변 상태를 캡슐화하는 것은 까다로운데 객체 내부 요소를 가리키는 레퍼런스가 언제든 제어권 밖으로 벗어날 수 있기 때문이다.
    - 상태를 락으로 보호하면 처리율이 크게 떨어질 수 있다.
    - 락을 쓰면 데드락을 비롯한 별별 문제가 유발된다.
    - 그 외 공유 메모리를 정확히 사용하기 어려운 점, CPU 캐시 라인을 공유하게 함으로 발생할 수도 있는 성능 문제도 있다.
- 액터 모델이 유용한 툴이긴하지만 다른 기법 전체를 대체할 수 있는 범용 툴은 아니다.
    - 액터 방식은 불변 메시지의 비동기 전송, 가변 상태 공유 금지, 각 메시지가 제한된 시간 동안 실행 등
    - 설계 요건상 요청-응답의 비동기 처리, 가변 상태 공유, 무제한 실행 등을 고려해야 한다면 다른 방법을 찾아야 한다.
