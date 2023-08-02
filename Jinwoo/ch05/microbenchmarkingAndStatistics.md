## 5.1 자바 성능 측정 기초

* 목표 : 벤치마크로 공정한 테스트를 하는 것. 가급적 시스템의 한 곳만 변경하고 다른 외부 요인은 통제하는 것이 이상적이다.
* 하지만, 자바 플랫폼을 벤치마크할 때는 런타임의 정교함이 문제이다. 자바 플랫폼은 다양한 최적화 기술과 자체적인 런타임 최적화를 수행하여 프로그램의 성능을 향상시키기 때문에, 벤치마크 결과에 런타임 최적화의 영향이 크게 나타날 수 있다.

다음은 100,000개 숫자를 정렬하는 벤치마크 코드이다. 아래의 코드 실행 시, 여러 문제점이 있다.

```java
public class ClassicSort {

    private static final int N = 1_000;
    private static final int I = 150_000;
    private static final List<Integer> testData = new ArrayList<>();
    public static void main(String[] args) {
        Random randomGenerator = new Random();
        for (int i=0; i<N; i++) {
            testData.add(randomGenerator.nextInt(Integer.MAX_VALUE));
        }

        System.out.println("정렬 알고리즘 테스트");

        double startTime = System.nanoTime();

        for(int i = 0; i < I; i++) {
            List<Integer> copy = new ArrayList<>(testData);
            Collections.sort(copy);
        }

        double endTime = System.nanoTime();
        double timePerOperation = ((endTime - startTime) / (1_000_000_000L * I));
        System.out.println("결과 : " + (1 / timePerOperation) + " op/s"); 
    }
}

```

1. JVM 웜업을 전혀 고려하지 않았다.

JIT 컴파일러는 JVM에 내장된 덕분에 인터프리티드 바이트코드를 고도로 최적화한 기계어로 변환한다. JVM 웜업이 없이 벤치마크를 실행하면, JIT 컴파일러가 최적화되지 않은 초기 코드를 사용하여 측정하게 된다. 이로 인해 실제로는 성능이 더 우수한 코드가 벤치마크 결과로부터 제외될 수 있다. 따라서 일반적으로 처음 몇 차례의 실행은 예상 성능에 미치지 못할 수 있으며, 실제 최적화된 결과를 얻기 위해서는 여러 번의 반복 실행과 웜업 과정이 필요하다.


2. 가비지 수집 
    
가비지 수집이 불확정적으로 발생하기 때문에 정확한 확인이 되지 않는다. 가비지 수집은 일시적인 지연, GC 오버헤드와 같은 영향을 끼치기 때문에 벤치마크를 정확하게 수행하기 위해서는 가비지 수집의 영향을 고려해 적절한 반복 실행과 결과 분석이 필요하다.
    
* 가비지 수집이 작동되는 모습은 다음 VM 플래그를 추가하면 볼 수 있다.
    * java -Xms2048m -Xmx2048m -verbose:gc ClassicSort

3. 테스트하려는 코드에서 생성된 결과를 실제로 사용하지 않는다.

JIT 컴파일러가 해당 코드를 죽은 코드로 간주하고 최적화 과정에서 제거할 수 있으며, 이로 인해 우리가 벤치마크하려던 코드의 성능 측정이 왜곡될 수 있다.

이를 해결하는 방법은 크게 두가지이다.

* 시스템 전체를 벤치마크한다. 수많은 개별 작용의 전체 결과를 평균내어 더 큰 규모에서 유의미한 결과를 얻는다.
* 연관된 저수준의 결과를 의미있게 비교하기 위해서는 앞서 언급한 문제들을 공통 프레임워크를 이용해서 처리한다.

JMH가 그런 툴이다.

## 5.2 JMH 소개

마이크로벤치마킹은 잘못되기 쉽기 때문에 가능하면 하지 않는게 좋다.

### 5.2.1 될 수 있으면 마이크로벤치마크하지 말지어다

자바 성능 문제에서 개발자는 큰 그림을 못 보고 자기 코드가 성능을 떨어뜨렸을 거란 강박에 사로잡히는 경우가 많다. 작은 코드를 세세히 뜯어보는 수준으로 벤치마킹하는 건 몹시 어렵고 ‘베어 트랩’에 빠질 위험도 있다.

### 5.2.2 휴리스틱 : 마이크로벤치마킹은 언제 하나 ?

주로 사용하는 경우는 다음과 같다.

* 사용 범위가 넓은 범용 라이브러리 코드를 개발한다.
* OpenJDK 또는 다른 자바 플랫폼 구현체를 개발한다.
* 지연에 극도로 민감한 코드를 개발한다.

일반적으로 마이크로벤치마크는 가장 극단적인 애플리케이션에 한해서 사용하는 것이 좋다. 대부분의 경우에는 거의 쓸일이 없지만 이론과 복잡성을 알고 있는 것이 좋다.

### 5.2.3 JMH 프레임워크
> JMH는 자바를 비롯해 JVM을 타깃으로 하는 언어로 작성된 나노/마이크로/밀리/매크로 벤치마크를 제작, 실행, 분석하는 자바 도구다.

* 특정 메소드의 성능을 측정하는 식으로 사용할 수 있고 실제 테스트하기전 워밍업 과정과 실제 측정 과정을 수행하는데 각 과정의 실행 수를 제어할 수 있고, 측정 후 결과로 나오는 시간의 단위를 지정하는 기능도 제공한다.

참고 : https://hyesun03.github.io/2019/08/27/how-to-benchmark-java/

### 5.2.4 벤치마크 실행

* 여러 설정 태스크를 마친 후 실행시킬 벤치마크 메서드에 @Benchmark 를 붙인다.

```java
    public class MyBenchmark {

    	@Benchmark
    	public void testMethod() {
    		// 코드 스텁
    	}

    }
```

* 벤치마크 실행을 설정하는 매개변수는 명령줄에 넣거나 `main()` 메서드에 세팅한다.

```java
    public class MyBenchmark {

    	public static void main(String[] args) throws RunnerException {
    		Options opt = new OptionsBuilder()
    				.include(SortBenchmark.class.getSimpleName())
    				.warnupIterations(100)
    				.mesurementIterations(5).forks(1)
    				.jvmArgs("-server", "-Xms2048m", "-Xmx2048m").build();

    		new Runner(opt).run();
    	}

    }
```
* JMH 프레임워크는 @State를 통해 상태를 제어하는 기능을 제공한다.
    * @State : Benchmark, Group, Thread의 세 상태가 있는 Scope enum을 받는다.
    * @State를 붙인 객체는 벤치마크 도중 엑세스할 수 있어 어떤 설정을 하는 용도로 쓸 수 있다.

* JMH는 벤치마크 코드에서 계산된 값을 반환하지 않아도 JVM에서 마치 사용하는 것처럼 인식하게 하는 Blackhole 방법을 제공한다.

출처: https://ysjee141.github.io/blog/quality/java-benchmark/ [happs's doodle]

### 실제 테스트
참고 : https://shirohoo.github.io/backend/java/2021-09-06-jmh-gradle/

* 플러그인 버전 : 0.7.1
* ./gradlew jmh
* 별도의 옵션 없이 돌리면 시간이 오래 걸려서 다음을 추가해준다.
```
jmh {
    fork = 1
    warmupIterations = 1
    iterations = 1
}
```

<img width="1280" alt="스크린샷 2023-08-03 오전 1 48 51" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/98975580/b2c40865-be5d-421d-a3d0-0f48faf2120a">
<img width="831" alt="스크린샷 2023-08-03 오전 1 49 20" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/98975580/28afc064-0525-4540-8a44-f51dd3d06a59">

## 5.3 JVM 성능 통계

모든 측정은 어느 정도의 오차를 가지고 있다.

### 5.3.1 오차유형

* 랜덤 오차 
    * 무관계 요인이 어떤 상관관계 없이 결과에 영향을 미친다.
    * 정밀도는 랜덤 오차를 나타내는 용어이다. 정밀도가 높으면 랜덤 오차가 낮다.
    * 랜덤 오차는 원인을 알 수 없는, 또는 예기치 못한 환경상의 변화 때문에 일어난다.

* 계통 오차
    * "원인을 알 수 없는 요인"이 상관관계 있는 형태로 측정에 영향을 미친다.
    * 정확도는 계통 오차 수준을 나타내는 용어로, 정확도가 높으면 계통 오차가 낮다.
![계통 오차](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/98975580/d0b46f73-22e9-423d-8c2a-cfc3519d93f7)
    * 선형 패턴으로 증가하는 그래프를 보니 어떤 한정된 서버 리소스가 조금씩 소모되고 있음을 알 수 있다.
        * 메모리 누수가 발생하거나, 어떤 스레드가 요청 처리 도중 다른 리소스를 점유하여 놔주질 않는 상황에서 발생하는 패턴이다.
    * 모든 api 응답 시간이 180밀리초로 일정한 양상을 보인다. 이는 테스트 대상 서버가 런던에 있는데 부하 테스트를 인도에서 했기 때문에 네트워크 지연 시간이 응답 시간에 포함되었기 때문이다. 120~150밀리초 범위의 지연 시간이 특이점을 제외한 절대 다수의 서비스 응답 시간을 차지했다.

* 허위 상관
    * "상관은 인과를 나타내지 않는다."
    * 두 변수가 비슷하게 움직인다고 해서 이들 사이에 연결고리가 있다고 볼 수 없다.
    * JVM과 성능 분석 영역에서는 연관있어보이는 연결고리와 상관관계만 보고 측정값 간의 인과 관계를 넘겨짚지 않도록 조심해야 한다.
![허위상관](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/98975580/6f4c7885-97e4-4d64-aba4-4b88dc7a3457)


### 5.3.2 비정규 통계학

정규 분포 통계학에 나오는 '기본 원칙'은 비정규 분포에는 잘 맞지 않는다. 특히, 표준편차/분산, 다른 고차적률 같은 개념은 기본적으로 쓸모가 없다.

자바 성능 튜닝 중 측정한 값은 대부분 통계적으로 심한 비정규 분포를 나타낸다. 이때 고도 동적 범위에 분포된 데이터셋을 처리하는 HdrHistogram 라는 공개 라이브러리를 이용하면 좀 더 정교한 분석이 가능하다.

## 5.4 통계치 해석

HTTP 요청-응답 시간을 측정한 히스토그램

* 일반적인 측정값을 보다 유의미한 하위 구성 요소들로 분해할 수 있다.

* 클라이언트 오류
: 매핑되지 않는 URL을 클라이언트가 요청하면 웹 서버는 곧장 응답을 준다.

* 서버 오류
: 장시간 처리하다가 발생하는 편이다.

* 성공 요청
: 긴 꼬리형 분포를 보이지만, 실제로는 극댓값이 여럿인 다봉분포를 나타낸다.

## 5.5 마치며

* 유스케이스를 확실히 모르는 상태에서 마이크로벤치마킹 하면 안됩니다.
* 그래도 해야 한다면 JMH를 이용하세요.
* 해야 한다면 가능한 한 많은 사람과 공유하고 회사 동료들과 함께 의논합니다.
* 항상 잘못될 가능성을 염두에 두고 여러분의 생각을 지속적으로 검증합니다.