## 3.1 테이블 액세스 최소화

대량 데이터의 경우 인덱스가 한없이 느리다 (테이블 전체를 스캔하는 경우보다 느리다)

RowId 는 논리적 주소에 더 가깝다. 테이블 레코드를 찾아가기 위한 위치 정보를 담는다

메인 메모리 DB : 데이터를 모두 메모리로 로드해놓고 메모리를 통해서만 I/O를 수행하는 DB이다.

잘 튜닝된 OLTP성 데이터 베이스 시스템이라면 버퍼 캐시 히트율이 99%이상이다. ⇒ 디스크를 경유하지 않고 메모리에서 읽어오는 것이다. ⇒ 어떤 메인메모리의 경우 인스턴스를 기동하면 디스크에 저장된 데이터를 버퍼캐시로 로딩하고 이어서 인덱스를 생성한다. 이때 인덱스 주소가 아닌 메모링 상의 주소정보(포인터)를 갖는다.

이에 오라클은 테이블 블록이 수시로 버퍼캐시에 밀려났다 캐싱되어 그때마다 다른 공간에 캐싱되어 인덱스에서 포인터로 연결할 수 없다.  이 때문에 일반 DBMS에서 인덱스 ROWID를 이용한 테이블 액세스가 생각 만큼 빠르지 않은것이다.

DBA(= 데이터파일번호 + 블록번호)는 디 스크 상에서 블록을 찾기 위한 주소 정보다. 이때문에 버퍼캐시를 잘 활용해야한다.

인덱스 ROWID는 포인터가 아니다. 디스크 상에서 테이블 레 코드를 찾아가기 위한 논리적인 주소 정보다. ROWID가 가리키는 테이블 블록을 버퍼캐시 에서 먼저 찾아보고 못 찾을 때만 디스크에서 블록을 읽는다. 물론 버퍼캐시 에 적재한 후에 읽는다.

설령 모든 데이터가 캐싱돼 있더라도 테이블 레코드를 찾기 위해 매번 DBA 해싱과 래치 회 득 과정을 반복해야 한다. 동시 액세스가 심할 때는 캐시버퍼 체인 래치와 버퍼 Lock에 대한 경합까지 발생한다. 이처럼 인덱스 ROWID를 이용한 테이블 액세스는 생각보다 고비용 구조다.

오라클에서 하나의 레코드를 찾아가는 데 있어 가장 빠르다고 알려진 `ROWID에 의한 테이 블 액세스`는 고비용 연산이다.

인덱스 ROWID를 이용한 테이블 엑세스는 고비용 구조다

인덱스를 이용한 테이블 액세스가 Table Full Scan보다 더 느려지게 만드는 가장 핵심적인 두 가지 요인은 다음과 같다.

- Table Full Scan은 시퀀셜 액세스인 반면, 인덱스 ROWID를 이용한 테이블 액세스 는 랜덤 액세스 방식이다.
- Table Full Scan은 Multiblock I/0인 반면, 인덱스 ROWID를 이용한 테이블 액세 스는 Single Block I/O 방식이다.

즉, 테이블 스캔이 항상 나쁜 것은 아니며, 바꿔 말해 인덱스 스캔이 항상 좋은 것도 아니라는 사실을 설명하는 데 목적이 있다.

온라인 프로그램 튜닝 : 소량 데이터를 읽고 갱신하므로 인덱스를 효과적으로 활용하는 것이 무엇보다 중요하다.조인도 대부분 NL방식을 사용한다(인덱스 활용방식) ⇒ 소트 연산을 생략하고 온라인 환경에서 대량 데이터를 조회 할 떄도 아주 빠른 응답속드를 낼 수 있다.

배치 프로그램 : 대량 데이터 처리 , 항상 전체 범위의 처리 기준으로 튜닝해야 한다. 처리 대상 집합중 일부를 빠르게 처리하는 것이 아니라 전체를 빠르게 처리하는 것을 목표로 삼아야 한다. 대량 데이터를 빠르게 처리하려면 인덱스,NL 조인보다 FUllScan 과 HashJoin이 유리하다.

다시 강조하지만, 모든 성능 문제를 인덱스로 해결하려 해선 안 된다. 인덱스는 다양한 튜닝 도구 중 하나일 뿐이며, 큰 테이블에서 아주 적은 일부 데이터를 빨리 찾고자 할 때 주로 사용한다.

테이블 액세스 최소화를 위해 가장 일반적으로 사용하는 튜닝 기법은 인덱스에 컬럼을 추가 하는 것이다.

인덱스 자체를 추가하다보면 수십개의 인덱스가 달려 배보다 배꼽이 커진다.

이럴 때 기존 인덱스에 SQL 컬럼을 추가하여 효과를 얻을 수 있다. 인덱스 스캔량이 줄진 않지만 데이블 랜덤 엑세스 횟수는 줄여준다.

반드시 성능을 개선해야 한다면, 쿼리에 사용된 컬럼을 모두 인덱스에 추가해서 테이블 액세 스가 아에 발생하지 않게 하는 방법을 고려해 볼 수 있다. 참고로, 인덱스만 읽어서 처리하는 쿼리를 Cvered 쿼리 라고 부르며, 그 쿼리에 사용한 인덱스를 `Covered 인덱스`라고 부른다. ⇒ 추가해야할 컬럼이 많아진다는 단점이 있다.

랜덤 엑세스가 아예 발생하지 않도록 테이블을 인덱스 구조로 생성하는 방법 → IOT (인위적으로 클러스터링 팩터를 좋게 한다)

summary : table index가 성능에 미치는 영향, 그리고 그것을 최소화하기 위해 인덱스에 컬럼을 추가하고 테이블 저장 구조를 개선하는 방법

## 3.2 부분범위 처리 활용

부분 범위 처리 : 데이터를 전송할 때 일정량 씩 나누어 전송한다.

1. 정렬 조건이 있을때 부분범위 처리
2. ArraySize 조정을 통한 Fetch Call 최소화
3. 쿼리 툴에서 부분 범위 처리

   → 중간에 멈췄다가 사용자의 추가 요청이 있을 때마다 데이터를 가져오도록 구현