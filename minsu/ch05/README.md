# 5.1 자바 성능 측정 기초
벤치마킹을 이용해서 공정한 테스트를 수행하는 것이 목표이지만 자바 플랫폼을 벤치마크할 때는 런타임의 정교함 때문에 문제가 발생하곤 한다.  
예를 들어 아래의 ClassicSort라는 코드를 실행시키면 실제 성능 측정에 있어서 여러가지 문제점이 발생한다.  

```Java
public class ClassicSort {
  private static final int N = 1_000;
  private static final int I = 150_000;
  private static final List<Integer> testData = new ArrayList<>();

  public static void main(String[] args) {
    Random randomGenerator = new Random();
    for (int i = 0; i < N; i++) {
      testData.add(randomGenerator.nextInt(Integer.MAX_VALUE));
    }

    System.out.println("정렬 알고리즘 테스트");

    double startTime = System.nanoTime();

    for (int i = 0; i < I; i++) {
      List<Integer> copy = new ArrayList<>(testData);
      Collections.sort(copy);
    }

    double endTime = System.nanoTime();
    double timePerOperation = ((endTime - startTime) / (1_000_000_000L * I));
    System.out.println("결과: " + (1/timePerOperation) + "op/s");
  }
}
```
### 문제점
- JVM이 가동 준비를 하는 데에 시간이 소요
- 가비지 수집(GC)이 불확정적으로 발생하여 실제 성능 측정이 정확하지 않을 수 있다. 
- 최적화되지 않은 코드가 실행되어 실제 사용하지 않는 코드의 영향으로 정확한 벤치마크가 이루어지지 않을 수 있다. 
- 벤치마크의 수행 과정을 정확히 알 수 없어서 결과의 평균을 낼 수 없다.


### 해결하기 위한 방법
- 시스템 전체를 벤치마크-> 저수준 수치는 수집하지 않거나 무시한다.
- 연관된 저수준의 결과를 의미 있게 비교하기 위해 공통 프레임워크를 이용하여 문제를 처리-> JMH(Java Microbenchmark Harness) 같은 도구를 사용할 수 있다.

# 5.2 JMH 소개
## 5.2.1 될 수 있으면 마이크로벤치마크를 하지 않는다.
개발자는 너무 작은 코드를 세세히 뜯어보며 원인을 찾으려고 하면 '베어 트랩'에 빠질 수 있다.

## 5.2.2 휴리스틱 : 마이크로 벤치마킹은 언제 할까?
주로 유스케이는 다음과 같다.
```
- 사용 범위가 넓은 범용 라이브러리 코드를 개발한다. 
- OpenJDK 또는 다른 자바 플랫폼 구현체를 개발한다.
- 지연에 극도로 민감한 코드를 개발한다.
```
일반적으로 마이크로벤치마크는 가장 극단적인 애플리케이션에 한해서 사용하는 것이 좋다. 
```
- 총 코드 경로 실행 시간이 적어도 1밀리초, 실제로는 100마이크로초 보다 짧아야 한다.
- 메모리 할당률을 측정하는데 그 값은 1MB/s 미만 가급적 0에 가까워야 한다.
- 100%에 가깝게 CPU를 사용하여 시스템 이용률은 낮게 (10% 미만) 유지해야 한다.
- 실행 프로파일러로 CPU를 소비하는 메서드들의 분포를 이해해야 하며, 분포 그래프에서 지배적인 영향을 끼치는 메서드는 많아야 두세 개 정도다.
```

## 5.2.3 JMH 프레임워크
JMH는 앞의 이슈를 해결하기 위해 개발된 프레임워크이다.

JMH : 자바를 비롯해 JVM을 타깃으로 하는 언어로 작성된 벤치마크를 제작, 실행, 분석하는 자바 도구입니다.
프로엠워크는 컴파일 타임에 벤치마크 내용을 알 수 없으므로 동적이여야합니다.

## 5.2.4 벤치마크 실행
gradle 또는 maven을 이용해 환경설정을 한 뒤 벤치마크 메서드에는 @Benchmark 르 붙이고, 벤치마크 실행을 설정하는 매개변수는 명령줄에 넣거나, main() 메서드에 세팅한다. 
```java
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Random;

@State(Scope.Benchmark)
public class MyBenchmark {

    private static final int N = 1000;
    private static final int I = 150000;
    private static final List<Integer> testData = new ArrayList<>();

    public MyBenchmark() {
        // 무작위 정수 배열 생성
        Random randomGenerator = new Random();
        for (int i = 0; i < N; i++) {
            testData.add(randomGenerator.nextInt(Integer.MAX_VALUE));
        }
    }

    @Benchmark
    public void testMethod() {
        for (int i = 0; i < I; i++) {
            // 정렬을 위해 배열을 복사한 후 정렬
            List<Integer> copy = new ArrayList<>(testData);
            Collections.sort(copy);
        }
    }

    public static void main(String[] args) throws RunnerException {
        // JMH 설정
        Options opt = new OptionsBuilder()
                .include(MyBenchmark.class.getSimpleName())
                .warmupIterations(100)
                .measurementIterations(5)
                .forks(1)
                .jvmArgs("-server", "-Xms2048m", "-Xmx2048m")
                .build();

        new Runner(opt).run();
    }
}
```

- @State는 상태를 정의하는 애너테이션으로, Benchmark, Group, Thread 세 상태가 정의된 Scope enum을 받는다.
- @State를 붙인 객체는 벤치마크 도중에 액세스할 수 있다.
- 멀티스레드 코드 역시 상태를 제대로 관리되지 않아 벤치마크가 편향되지 않게 하려면 조심스럽게 다루어야 한다.

### JVM은 메서드 내에서 실행된 코드가 부수 효과를 전혀 일으키지 않고 그 결과를 사용하지 않을 경우 해당 메서드를 삭제 대상으로 삼기 때문에 JMH는 벤치마크 메서드가 변환한 단일 결괏값을 암묵적으로 `블랙홀`에 할당한다

## 블랙홀
블랙홀은 네 가지 장치를 이용해 벤치마크에 영향을 줄 수 있는 최적화로부터 보호한다.
- 런타임에 죽은 코드를 제거하는 최적화를 못하게 한다.
- 반복되는 계산을 상수 폴딩(컴파일러가 컴파일 타임에 미리 계산 가능한 표현식을 상수로 바꾸어 처리하는 최적화 과정)하지 않게 만든다.
- 값을 읽거나 쓰는 행위가 현재 캐시 라인에 영향을 끼치는 잘못된 공유 현상 방지
- 쓰기 장벽으로부터 보호한다.

`장벽` : 리소스가 포화돼 사실상 애플리케이션에 병목을 초래하는 지점

# 5.3 JVM 성능 통계
## 5.3.1 오차 유형
### 랜덤 오차
측정 오차 또는 무관계 요인이 어떤 상관관게 없이 결과에 영향을 미친다.
`정밀도`: 랜덤 오차를 나타내는 용어
  - 정밀도가 높으면 랜덤 오차가 높다.

### 계통 오차
원인을 알 수 없는 요인이 상관관계 있는 형태로 측정에 영향을 미친다.
`정확도`: 계통 오차 수준을 나타내는 용어
  - 정확도가 높으면 계통 오차가 낮다.

예를 들어 화살을 쐈을 때
- 과녁 중앙에 몰렸다 -> 정밀도, 정확도 모두 높다
- 화살이 모두 (가장자리) 한 쪽에 쏠려 있다 -> 계통 오차가 높다 (정밀도는 높으나 정확도는 낮다)
- 화살이 모두 과녁에 맞았지만 (6~10점 사이) 드문드문 흩어져 있다 -> 랜덤 오차가 높다. (정밀도는 낮으나 정확도가 높다)

### 허위 상관
- 상관은 인과를 나타내지 않는다.
- 따라서 JVM과 성능 분석 영역에서는 그럴싸해 보이는 연결고리와 상관관게만 보고 측정값 간의 인과관계를 넘겨짚지 않도록 조심해야 한다.

## 5.3.2 비정규 통계학
- 만족스러운 서비스를 받고 있는 대다수의 고객의 경험보다 특이점을 유발하는 이벤트가 더 중요한 관심사이다.
- 다음은 트랜잭션 시간 분포를 현실적으로 나타낸 그래프이다.
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/adb60362-ae96-4f93-b662-7bc47d9add0f)
- 우리가 JVM에 대해 직관적으로 알고 있는 것, 즉 모든 관련 코드가 이미 JIT 컴파일돼서 GC 사이클이 없는 핫 패스의 존재를 시사한다.

## 5.4 통계치 해석
웹 애플리케이션은 응답 유형마다 응답 시간 분포는 다르다.  

클라이언트 오류  
- 매핑되지 않은 URL로 클라이언트가 요청하면 웹 서버는 곧장 404 응답을 줄 것이고, 이를 그림으로 나타내면 다음과 같다.  
<img width="450" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/5c949453-bd1c-48fb-beed-40c4a710dd7a">  

서버 오류  
- 서버 에러는 장시간 요청을 처리하다가 발생하는 편이다.  
<img width="450" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/8bdccd84-3a05-4658-b78c-f6e9bf8deace">  


성공 요청  
- 성공한 요청은 긴 꼬리형 분포를 보이지만, 실제로는 극댓값이 여러개인 다봉분포를 나타낸다. 
<img width="450" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/66a24ced-895e-42ef-a39f-0e4675c8fe7c">  


- 유형이 다른 세 가지 응답 시간을 하나로 조합하니 다음과 같다.  


<img width="450" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/5cb7fa78-6f36-4c00-9dd5-548b5a44ef4d">



