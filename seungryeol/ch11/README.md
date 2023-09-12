<h1>자바 언어의 성능 향상</h1>

<h2>컬렉션 최적화</h2>

- 순차 컨테이너 : 수치 인덱스로 표기한 특정 위치에 객체를 저장한다.
- 연관 컨테이너 : 객체 자체를 이용해 컬렉션 내부에 저장할 위치를 결정한다.

컨테이너에서 메서드가 정확히 작동하려면 저장할 객체가 호환성과 동등성 개념을 지니고 있어야한다.</br>
모든 객체가 반드시 hashcode()및 equals() 메서드를 구현해야 한다고 표현한다.</br>

참조형 필드는 힙에 레퍼런스로 저장된다. 객체가 순차적으로 저장된다고 대충 말하지만, 사실 컨테이너에 저장되는 건 객체 자신이 아니라, 객체ㅔ를 가리키는 레퍼런스이다.</br></br>

자바는 메모리 서브시스템이 알아서 가비지 수집을 해주는 대신, 저수준의 메모리 제어를 포기할 수 밖에 없다. 메모리 수동 할당/해제는 물론, 저수준 메모리 레이아웃 제어까지 단념해야한다.</br>


![KakaoTalk_20230912_113924840_02](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/c0e4b5ed-a36c-4be7-8f93-c3dc048b33ec)

<h2>LIST 최적화</h2>

리스트는 ArrayList와 LinkedList 두 가지로 나뒨다.(Vector는 더 이상 쓰이지 않음)</br>

<h4>ArrayList</h4>

배킹 배열의 최대 크기만큼 원소를 추가할 수 있고 이 배열이 꽉 차면 더 큰 배열을 새로 할당한 다음 기존 값을 복사한다.</br>
따라서 크기 조정 작업 비용과 유연성을 잘 저울질해야한다.</br></br>

밴치마킹에 의하면 처음 리스트의 크기를 정해놓고 작업을 돌리는것이 정하지 않고 돌리는 것보다 성은이 더 뛰어난다.</br>

<h4>LinkedList</h4>

LinkedList는 동적으로 증가하는 리스트이다.(이중 연결 리스트)</br>

![KakaoTalk_20230912_113924840_01](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/ed997a35-44bb-400b-94f1-2e3505f223e1)


<h4>ArrayList vs LinkedList</h4>

둘 중 어느 것을 쓸지는 데이터 접근/수정 패턴에 따라 다르다.</br>

그러나 ArrayList의 특정 인덱스에 원소를 추가하려면 다른 원소들을 모두 한 칸씩 우측으로 이동시켜야한다.</br>
반면, LinkedList는 삽입 지점을 찾기 위해 노드 레퍼런스를 죽 따라가는 수고는 있지만, 삽입 작업은 노드 하나 생성한 다음 두 레퍼런스(first,next)를 세팅하면 간단히 끝난다.</br></br>

리스트를 주로 랜덤 엑세스하는 경우라면 ArrayList가 정답이다.</br>
=> LinkedList는 처음부터 인덱스 카운트만큼 원소를 방문해야한다.</br>

벤치마크에 따르면, 랜덤 엑세스가 필요한 알고리즘을 구사할 때는 ArrayList를 권장한다. ArrayList는 가급적 미리 크기를 지정해서 중간에 다시 조정하는 일이 없도록 하는게 좋다.</br>


<h2>Map 최적화</h2>

자바는 java.util.Map<K,V> 인터페이스를 제공하면 키/값 모두 반드시 참조형이어야 한다.</br>

<h4>HashMap</h4>


![KakaoTalk_20230912_113924840](https://github.com/JSON-loading-and-unloading/Optimizing-Java/assets/106163272/04aecc95-3cf0-4cb4-8c26-1123349efe81)

처음에는 버킷 엔트리를 리스트에 저장한다. 값을 찾으려면 키 해시값을 계산하고 equals()메서드로 리스트에서 해당 키를 찾는다. 키를 해시하고 동등성을 기준으로 리스트에서 값을 찾는 메커니즘이므로 키 중복은
허용되지 않는다. </br>

자바 최근 버전에서는 HashMap이 키 해시값을 계산할 때 상위비트를 무조건 반영하도록 설계한 것이다.</br>
=> 인덱스 계산 시 상위비트가 누락될 수 있기 때문에 여러가지 문제가 생김</br>

HashMap 생성자에 전달하는 initialCapacity와 loadFactor 두 매개변수는 HashMap의 성능에 큰 영향을 미친다. HashMap 용량은 현재 생성된 버킷 개수를, loadFactor는 버킷 용량을 자동 증가(2배)시키는 한계치이다.
용량을 2배 늘리고 저장된 데이터를 다시 배치한 다음, 해시를 다시 계산하는 과정을 재해시라고 한다.</br>

intialCapacity와 loadFactor를 높게 잡으면 순회 시 성능에 상당한 영향을 받는다.</br></br>

initialCapacity는 HashMap이 내부 해시 테이블을 생성할 때 초기로 할당되는 버킷 (해시 버킷)의 수를 나타냅니다.</br>
loadFactor는 해시 맵이 언제 내부 해시 테이블을 재조정할지를 결정하는 요소입니다.</br>

ex)</br>
HashMap<String, Integer> hashMap = new HashMap<>(16, 0.75f);</br>
initialCapacity는 16이고 loadFactor는 0.75로 설정되었습니다. 이러한 매개변수를 조절하여 HashMap의 성능과 메모리 사용량을 조절할 수 있습니다.</br>

최근 HashMap에는  새로운 장치가 달려 있어서 비용이 크기에 비례하여 늘지 않습니다.</br>
하나의 버킷에 TREEIFY_THRESHOLD에 설정한 개수만큼 키/값 쌍이 모이면 버킷을 TreeNode로 바꿔버린다.</br>

※아예 처음부터 바꿔버려?</br>
TreeNode는 리스트 노드보다 약 2배 더 커서 그만큼 공간을 더 차지한다.</br>

<h6>LinkedHashMap</h6>

LinkedHashMapdms HashMap의 서브클래스로, 이중 연결 리스트를 사용해 원소의 삽입 순서를 관리한다.</br>
(LinkeHashMap의 기본 관리 모드는 삽입 순서이지만, 액세스 순서모드로 바꿀 수 있다.)</br>
</br>
LinkedHashMap은 순서가 중요한 코드에서 많이 쓰이지만 TreeMap처럼 비용이 많이 들지는 않는다.</br>

<h5>TreeMap</h5>

TreeMap은 다양한 키가 필요할 때 아주 유용하며, 서브맵에 신속히 접근할 수 있다.</br>
TreeMap이 제공하는 get(), put(), containsKey(), remove() 메서드는 log(n) 작업 성능을 보장한다.</br>
(HahsMap만으로도 충분하지만, 스트림이나 람다로 Map일부를 처리해야 할 때 쓰인다.)</br>
