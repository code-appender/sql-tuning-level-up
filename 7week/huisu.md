# DML 튜닝

## 기본 DML 튜닝

### DML 성능에 영향을 미치는 요소

**인덱스**

- 인덱스를 위한 데이터를 생성하거나 삭제하는 작업은 DML에 영향
- INSERT, DELETE는 인덱스 조작을 한 번만 수행
- UPDATE는 인덱스 조작을 두 번 수행 (DELETE → INSERT)
- 시스템마다 다르지만
    - 인덱스 1개에 100만 건 데이터를 넣을 때는 5초
    - 인덱스 3개에 100만 건 데이터를 넣을 때는 40초

**무결성 제약**

- PK, FK, Check, Not Null 같은 제약도 DML 성능에 영향을 미침
- 시스템마다 다르지만
    - PK가 없으면 100만 건 데이터를 넣을 때는 1.3초
    - PK가 있으면 100만 건 데이터를 넣을 때는 4.95초

**조건절 & 서브쿼리**

- 조건절을 확인하기 위해 혹은 서브쿼리를 실행하기 위해 SELECT 진행
- 따라서 SELECT할 때와 동일하게 인덱스를 잘 활용해야 함

**Redo 로깅**

- Redo: 모든 변경 사항을 기록하는 로그
    - 물리적으로 디스크가 깨질 경우, 데이터를 복구하기 위해
    - 정전 등으로 버퍼 캐시가 사라질 경우, 캐시를 복구하기 위해
    - 커밋은 느리기 때문에 혹시 모를 상황을 대비해 트랜잭션에 변경 사항 저장
- DML을 수행할 때마다 로그를 쌓기 때문에 성능에 영향

**Undo 로깅**

- Undo: Rollback
- 변경된 블록을 이전 상태로 되돌리는 데 필요한 정보를 로깅해야 하므로 성능에 영향

**Lock**

- Lock을 길게 잡으면 DML 성능이 안 좋아지고, 짧게 잡으면 데이터 품질이 안 좋아짐

**커밋**

- Lock과 함께 커밋도 DML 성능에 간접적으로 영향
- 커밋의 내부 매커니즘
    - DML 문을 수행하면 버퍼 캐시가 정전 등으로 사라질 수 있기에 Redo 로깅을 먼저 해 놓음
    - Redo 로그 파일에 기록하기 전에 로그 버퍼에 변경 사항 먼저 기록
    - 버퍼 캐시에 변경된 블록을 기록
    - 커밋 후 LGWR 프로세스가 Redo 로그 버퍼 내용을 Redo 로그 파일에 저장, DBWR 프로세스가 버퍼 캐시에 변경된 블록을 데이터 파일에 저장

### 데이터베이스 Call과 성능

- Parse Call: 파싱과 최적화 수행
- Execute Call: 실제로 SQL 수행
- Fetch Call: SELECT 문에서 사용자에게 결과를 전송하는 단계
- User Call: DBMS 외부로부터 인입되는 Call
    - WAS가 앞단에 있다면 WAS가 DBMS를 호출할 때 발생
- Recursive Call: DBMS 내부에서 발생하는 Call
    - 함수, 프로시저, 트리거에 내장된 SQL을 실행할 때 발생
- Call은 성능에 영향을 주며, 네트워크를 경유하는 User Call이 성능에 미치는 영향이 큼
    - 100만 건 기준으로 Recursive Call만 발생하는 for 문의 insert문을 짠 PL/SQL은 30초지만 User Call은 220초
    - 하나의 SQL인 Insert Into Select는 한 번의 Call만 발생하므로 One SQL을 사용하는 것이 좋음

### Array Processing 활용

- One SQL로 작성하기 힘들 때는 Array Processing을 활용

### 인덱스 및 제약 해제를 통한 대량 DML 튜닝

- 인덱스와 무결성 제약은 DML 성능에 영향을 줌
- 1000만 건의 데이터를 입력하게 되면
    - PK 인덱스 + 일반 인덱스 → 1분 19초
    - 인덱스 및 제한 사항 없음 → 5.8초
- 애플리케이션이 실행 중에 접근이 자주 발생하는 테이블에 대한 인덱스 및 제약을 잠시 동안 해제하기는 어려움
- 대량 데이터를 적재하기 위한 배치 프로그램에서는 이들 기능을 해제함으로 DML 성능을 높일 . 수있음
- 해당 DML이 테이블의 데이터를 5%이상 수정할 경우 추천

### 수정 가능 조인 뷰

- 뷰: 하나 이상의 테이블이나 다른 뷰의 데이터를 볼 수 있게 하는 데이터베이스 객체로 실제로 존재하는 테이블은 아니지만 논리적 데이터
- 조인 뷰: FROM 절에 두 개 이상의 테이블을 가진 뷰
- 수정 가능 조인 뷰: 입력, 수정, 삭제가 허용되는 조인 뷰
    - 키 보존 테이블 설정을 해야만 가능

## Direct Path I/O 활용

### Direct Path I/O

- 버퍼 캐시의 장점
    - 일반적으로 SQL문을 실행하면 버퍼 캐시를 확인해 본 뒤 작업 진행
    - 반복적으로 동일한 블록을 찾는 경우에 버퍼 캐시는 성능을 높여 주는 아주 좋은 기회
- 버퍼 캐시의 단점
    - 버퍼 캐시를 탐색하는 것도 락에 의해 느릴 수 있음
    - 반복적으로 동일한 블록을 찾는 경우가 없다면 버퍼 캐시를 경유하는 것이 오히려 나쁜 영향
- 오라클은 버퍼 캐시를 사용할 필요가 없을 경우를 위해 버퍼 캐시를 경유하지 않는 Direct Path I/O를 제공

### Direct Path Insert

- 일반적인 Insert vs Direct Path Insert


    | 일반적인 Insert | Direct Path Insert |
    | --- | --- |
    | FreeList에서 데이터를 입력할 수 있는 블록을 찾음 | FreeList를 찾아보지 않고 맨 뒤에 순차적으로 쌓음 |
    | FreeList에서 할당받은 블록을 버퍼 캐시에서 찾음 | 블록을 버퍼 캐시에서 탐색하지 않음 |
    | 버퍼 캐시에 없으면 데이터 파일에서 읽어 버퍼 캐시에 적재 | 버퍼 캐시에 적재하지 않고 데이터 파일에 직접 기록 |
    | Undo 기록 | Undo 기록하지 않음 |
    | Redo 기록 | Redo를 하지 않도록 할 수 있음 |
- 유도 방법
    - INSERT /*+ append */ INTO SELECT ... 처럼 append 힌트로 유도한 경우
    - INSERT /*+ parallel(C 4) */ INTO 고객 C 처럼 parallel 힌트로 유도한 경우
    - CREATE TABLE ... AS SELECT 문을 사용할 경우
    - 데이터를 적재할 수 있는 툴인 SQL*LOADER를 사용할 때 direct 옵션을 true로 준 경우
- 주의점
    - Direct Path Insert는 항상 맨 뒤에 Insert를 하므로 사이즈가 줄지 않고 계속 늘어남

### 병렬 DML

- Direct Path Insert는 INSERT에서 사용하는 방법이기에 UPDATE/DELETE에서는 사용할 . 수없다
- 다만 병렬 DML로 UPDATE와 DELETE를 수행할 경우 Direct Path Insert를 사용할 수 있다
- 병렬 DML을 사용할 경우 Exclusive 모드 TM Lock이 걸리는 점을 유의해야 함