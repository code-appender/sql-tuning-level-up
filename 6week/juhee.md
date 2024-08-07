# 소트 튜닝
## 1️⃣ 소트 연산에 대한 이해

소트는 기본적으로 PGA 에 할당된 SortArea 에서 이루어 진다. 메모리 공간이 다 차면 디스크 공간을 활용한다.

메모리 소트와 디스크 소트로 구분된다.

소트연산 : CPU 집약 + 메모리 집약  (소트가 일어나는경우 메모리 내에서 완료할 수 있도록 해야한다)

소트 오퍼레이션

1. Sort Aggregate
2. Sort Order By
3. Sort Group By
4. Sort Unique
    - 옵티마이저가 서브쿼리를 풀어 일반 조인문으로 변환하는 것을 ‘서브쿼리 Unnesting’이라 한다.
    - 이때 메인 쿼리와 조인하기 전에 중복 레코드를 제거하기 위해 Sort Unique 오퍼레이션이 동작한다.(pk,unique 가 보장되면 진행되지 않는다)
5. Sort Join
    1. 소트머지조인 실행시 발생
6. Window Sort
    1. 윈도우 함수를 수행할때 나타난다.

## 2️⃣ 소트가 발생하지 않도록 SQL 작성

1. Union vs Union All
    1. Union All은 두 집합간 중복제거를 위해 소트작업을 수행 하기 떄문에 될 수 있으면 UnionALL을 수행하는것이 낫다.
2. Exists
    1. Distinct사용시 중복제거를 위해 소트를 진행하여야 한다. 또한 모든 데이터를 읽어야 해 부분 처리가 불가능하고 많은 I/O작업이 수행된다.
3. 조인 방식 변경
    1. /*+ leading(c) use_nl(p)*/ 와 같이 힌트를 제공하여 조인 방식을 변경하면 소트 연산을 생략 할 수 있다.


## 3️⃣ 인덱스를 이용한 소트연산 생략

1. 인덱스 선두 컬럼을 활용하여 Sort Order By를 생략할 수 있다.
2. TopN쿼리
    1. 부분 범위 처리는 3-tier에서도 유효하다. 여기서 포인트가 TopN쿼리이다.
        1. ex ) FETCH FIRST 10 ROWS ONLY ;
        2. ex ) where rownum ≤ 10
    2. 옵티마이저 결과 확인시에 Count(stopkey)가 있다 일르 TOP N Stopkey알고리즘이라고 부른다.
    3. 이를 활용하여 페이징 처리를 진행할 수 있다.
        1. 페이징 처리 ANTI패턴
            1. 간결하게 처리하기 위해 RowNum을 제거하면 stopkey가 동작하지 않아 전체범위를 스캔하기때문에 주의 해야한다.
    4. 인덱스 컬럼을 가공하면 First Row StopKey는 동작하지 않는다.
3. 최소값 최댓갑 구하기
    1. 이런경우 인덱스가 있어도 정렬을 진행한다. 이를 위해 조건절 컬럼과 min/max가 모두 인덱스에 포함되어있어야 정렬을 진행한다.
    2. 여기서도 TOP N 쿼리를 이용해 최소/최대값 구하기
        1. Order By 를 진행하고 Where ROWNUM ≤1 과 같이 이용하여 TopN을 사용할 수 있다.
4. 이력 조회
    1. 이해가 아이 값니다..
5. Sort Group By 생략
    1. Sort Group By Nosort
        1. 이 방식은 인덱스를 이용해 NoSort방식으로 Group BY를 처리하여 부분범위 처리가 가능해진다.

## 4️⃣ Sort Area를 적게 사용하도록 SQL작성

1. 소트 데이터 줄이기
    1. 레코드의 개수를 줄여서 정렬을 줄인다. (서브 쿼리 사용 등..)
2. Top N 쿼리의 소트 부하 경감 원리
    1. RowNum을 지정하여 일부만 취하고 버리도록 한다.(메모리 최적화 )
3. Top N쿼리가 아닐때 발생하는 소트 부하 (안티 패턴 예시)
4. 분석 함수에서의 Top N 소트