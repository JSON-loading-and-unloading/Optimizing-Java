# Chapter 1 : 성능과 최적화


# 1.1 자바 성능 : 잘못된 방법

우수한 성능 목표를 달성하기 위해 필요한 여러가지 단면을 종합적으로 집중 조명하고자 한다.
  - 전체 소프트웨어 수명주기의 성능 방법론
  - 성능과 연계된 테스트 이론
  - 측정, 통계, 툴링(도구 선정)
  - (시스템 + 데이터) 분석 스킬
  - 하부 기술과 메커니즘(장치, 수단)

**❗️ 모든 최적화 기법에는 개발자가 사용하기 전에 알아아 두어야 할 함정과 트레이드오프가 도사리고 있으니 조심❗️**


# 1.2 자바 성능 개요

> 자바는 블루 칼라 언어입니다. 박사 학위 논문 주제가 아니라 일을 하려고 만든 언어죠.
> 
> _(제임스 고슬링)_

⭐️ 즉, 자바는 **실용성**을 추구한다.


### 관리되는 서브 시스템의 장단점
- 장점: 개발자가 일일이 용량을 세세하게 관리하는 부담을 덜어준다.
- 단점: 대신 저수준으로 제어 가능한 일부 기능을 포기하며 JVM 전반에 걸쳐 등장하는 관리하는 서브시스템은 그 존재 자체로 런타임 동작에 복잡도를 유발한다.
  

### 성능 측정값의 특징
1. 정규 분포를 따르지 않는 경우가 많아 기초 통계 기법(ex. 표준편차, 분산)만 갖고는 측정 결과를 제대로 처리하지 못한다.
2. 환경이 복잡해질수록 시스템을 개별적으로 떼어내 생각하기 어렵기 때문에 조심해야 한다.
3. 측정하는 행위 자체도 오버헤드를 일으킨다.


# 1.3 성능은 실험과학이다

셩능은 다음과 같은 활동을 하면서 원하는 결과를 얻기 위한, 일종의 실험과학이다.
  - 원하는 결과를 정의한다.
  - 기존 시스템을 측정한다.
  - 요건을 충족시키려면 무슨 일을 해야 할지 정한다.
  - 개선 활동을 추진한다.
  - 다시 테스트한다.
  - 목표가 달성됐는지 판단한다.



# 1.4 성능 분류

성능 지표는 성능 분석의 어휘집이자, 튜닝 프로젝트의 목표를 정량적인 단위로 표현한 기준


## 1.4.1 처리율
**시스템이 수행 가능한 작업 비율을 나타낸 지표** 
보통 일정 시간동안 완료한 작업 단위 수로 표시

조건
1. 수치를 얻은 기준 플랫폼에 대해서도 내용을 기술 
2. 트랜잭션은 테스트할때마다 동일 
3. 처리율을 테스트할 때 실행 간 워크로드 역시 일정하게 유지 


## 1.4.2 지연
**하나의 트랜잭션을 처리하고 그 결과가가 나올 때까지 소요된 시간** 
종단시간이라고도 한다.


## 1.4.3 용량
**시스템이 동시 처리 가능한 트랜잭션 개수**  
시스템에 동시 부하가 증가할수록 처리율, 지연도 영향을 받기에 
용량은 어떤 처리율 또는 지연 값을 전제로 가능한 처리량으로 표시


## 1.4.4 사용률
**시스템 리소스를 얼마나 효율적으로 활용하는지**  
사용률은 워크로드에 따라서 리소스별로 들쑥날쑥할 수 있다.


## 1.4.5 효율
**처리율/리스소 사용률** 
대형 시스템에서는 원가 회계 형태로 효율을 측정하기도 한다.  
(처리율이 동일하다면 A 솔루션의 총소유비용이 B 솔루션의 2배라면 당연히 효율은 절반)


## 1.4.6 확장성
시스템 확장성은 궁극적으로는 정확히 리소스를 투입한 만큼 처리율이 변경되는 '선형 확장' 형태를 지향한다.  
하지만 보통 시스템 확장성은 하나의 단순한 상수 인자가 아나리,   
여러 가지 인자들의 영향을 받기 때문에 선형 확장을 달성하기는 매우 어렵다. 
리소스를 어느 정도까지 늘리면 거의 선형적으로 확장되지만, 대부분 부하가 높아지면 완벽한 확장을 저해하는 한계점에 봉착한다. 


## 1.4.7 저하
요청 개수가 증거하하거나 요청 접수 속도가 증가하는 것과 같이  
변화는 사용률에 따라 다르다. 
시스템을 덜 사용하고 있으면 측정값이 느슨하게 변하지만 
시스템이 풀 가동된 상태면 처리율이 더는 늘어나지 않는, 즉 지연이 증가하는 양상을 띤다 -> 부하 증가에 따른 저하 


## 1.4.8 측정값 사이의 연관 관계 
다양한 성능 측정값은 어떤 식으로든 서로 연결돼 있다.  


# 1.5 성능 그래프 읽기
### 성능 엘보
아래 그래프는 부하가 증가하면서 예기치 않게 저하(여기서는 지연)가 발생한 그래프로, 이런 형태를 성능 엘보라고 한다.  
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/c930a421-5dd6-4462-80d2-0537fb5647ce)

### 준-선형적 확장
이와 반대로, 아래 그래프는 클러스터에 장비를 추가함에 따라 거의 선형적으로 처리율이 확장되는 경우이다.  
이렇게 이상적인 모습에 가까운 결과는 환경이 극단적으로 순조로울 때나 가능하다.  
<img width="283" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/e20e6c3f-a779-4eb4-b6c7-2effcaec0b5e">


### 암달의 법칙

> ‘암달의 법칙(Amdahl's law)’은 컴퓨터 프로그램은 프로세서를 아무리 병렬화 시켜도  
병렬처리가 가능한 부분(전체 처리량의 약 5%)과 불가능한(순차 처리) 부분으로 구성되므로  
더 이상 성능이 향상되지 않는 한계가 존재한다는 법칙이다. 이 때문에 일명 ‘암달의 저주’라고 불리기도 한다. 

근본적으로 확장성에는 제약이 따른다. 
아래 그래프는 태스크를 처리할 때 프로세서 개수를 늘렸을 때 실행 속도를 최대 어느 정도까지 높일 수 있는지를 나타낸 그래프이다.  
![image](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/177ecf82-df47-4316-8cd4-79538b50ef42)

하부 태스크를 75%. 90%. 95% 세 가지 다른 비율로 병렬화했는데 그래프를 보면 워크로드에 반드시 순차 실행되어야 할 작업 조각이 하나라도 있으면 
선형 확장은 불가하며 확장 가능한 한계점도 뚜렷하다는 사실을 알 수 있다.  

### 건강한 메모리 사용 현황
JVM 가비지 수집 서브시스템의 메모리 사용 패턴은 그 하부 기술 때문에 부하가 별로 없는 건강한 애플리케이션도 '톱니' 모양을 나타낸다.  
<img width="668" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/27460d25-abf9-484c-82b1-dc17dfbd44da">

### 문제가 있는 할당률 분포
아래 그래프는 피보나치 수열을 계산하는 애플리케이션을 실행하여 얻은 그래프로, 애플리케이션에서 메모리 할당류를 성능 튜팅할 때 아주 중요한 메모리 그래프이다. 
90초 부근에서 갑자기 할당률이 급격히 떨어지고 있고 그 이유는 바로 이 저지점에서 애플리케이션에 심각한 가비지 수집 문제가 발생했고  
가비지 수집 스레드들이 서로 CPU 경합을 벌인 탓에 메모리를 충분히 할당받지 못했기 때문이다.   
<img width="692" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/5e80aa31-ea2d-4660-b270-fd07ad7b02a8">

### 부하가 높을 때 상당한 지연 발생
시스템 리소스가 누수될 때 흔히 나타나는 징후 
부하가 증가하면서 지연이 차츰 악화되다가 결국 시스템 성능이 급락하는 변곡점에 이르게 된다.  
<img width="686" alt="image" src="https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/86006389/070b5a7b-1068-4753-b439-321d5e827c3b">


