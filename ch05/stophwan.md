## 5.1 트랜잭션
트랜잭션은 논리적인 작업 셋이 모두 적용되거나(COMMIT) 아무것도 적용되지 않거나(ROLLBACK) 를 보장함으로서 일부만 적용되는 현상이 발생되지 않도록 하는 기능이다.  

### 5.1.2 주의사항
프로그램 코드에서 트랜잭션의 범위를 최소화 해야한다.
- 데이터베이트의 커넥션 개수는 제한 점이므로, 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질 수록 사용가능한 커넥션의 개수는 줄어든다.
- 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 DBMS에 트랜잭션 내에서 제거하는 것이 좋다. 네트워크 에러가 웹 서버 뿐 아니라 DBMS 서버에 영향을 미칠 수 있다.
- 업무 요건에 따라 하나의 트랜잭션으로 묶거나 별도로 트랜잭션을 사용하는 것을 구분해야한다. 또한 단순 조회에 대해서는 트랜잭션에 포함할 필요는 없다.

## 5.2 MySQL 엔진의 잠금
MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다.

### 5.2.1 글로벌 락
MySQL 서버에 존재하는 모든 테이블을 닫고 잠금을 건다.
- 읽기 잠금을 걸기 전에 먼저 FLUSH를 해야하기 때문에, 테이블에 실행 중인 모든 쿼리가 완료 되어야한다. 
- 락을 획득하지 않은 세션에서 SELECT를 제외환 대부분의 DDL 문장이나 DML 문장을 실행하는 경우 대기 상태로 남는다.
- InnoDB는 트랜잭션을 지원하기 때문에, 일관된 데이터 상태를 위해 모든 데이터 변경 작업을 멈출 필요가 없어 백업 락이 도입되었다.

### 5.2.2 테이블 락
개별 테이블 단위로 설정되는 잠금, 명시적 또는 묵시적으로 특정 테이블의 락을 획득 할 수 있다.
- 명시적 테이블 락   
  - LOCK TALBES table_name [READ | WRITE] 명령으로 특 정 테이블의 락 획득 가능. 많은 영향을 미치기 때문에 특별한 상황이 아니면 사용할 필요가 없다.
- 묵시적 테이블 락  - 쿼리가 실행되는 동안 자동으로 획득됐다가 쿼리가 완료된 후 자동으로 해제된다.  
  - MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리를 실행하면 발생.
  - InnoDB 테이블의 경우 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 단순 데이터 변경 쿼리(DML)로 인해 묵시적인 테이블 락이 설정되지 않는다. 
  
### 5.2.3 네임드 락
GET_LOCK() 함수를 이용해 임의의 문자열에 대해 잠금을 설정하는 락이다.  
테이블이나 레코드, AUTO_INCREMENT 와 같은 데이터베이스 객체가 아니라 단순히 사용자가 지정한 문자열(String)에 대해 획득과 반납이 이루어지는 잠금이다.  
여러 클라이언트가 상호 동기화를 처리해야 할때 네임드 락을 이용한다.(5대의 웹서버 1대의 데이터베이스 서버일때 웹서들이 어떤 정보를 동기화 할때)  

### 5.2.4 메타데이터 락
데이터 베이스 객체(테이블, 뷰)의 이름이나 구조를 변경하는 경우 획득하는 잠금. 테이블의 이름을 변경하는 경우 자동으로 획득하는 잠금

## InnoDB 스토리지 엔진 잠금
InnoDB 스토리지 엔진은 레코드 기반의 잠금 기능을 제공한다. 이는 뛰어난 동시성 처리를 제공할 수 있다.  
과거에는 MySQL 명령을 이용해 스토리지 엔진에서 사용되는 잠금에 대한 정보에 접근하는 것이 까다로웠지만, 최근에는 이를 조회하는 방법이 도입되었다.

### 5.3.1.1 레코드 락레코드 자체만을 잠그는 것.  
InnoDB 스토리지 엔진진은 레코드 자체가 아니라 인덱스의 레코를 잠근 다는 점에서 다른 상용 DBMS 레코드 락과 차이가 있다.  
인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정.  
PK 혹은 유니크 인덱스에 의한 변경작업에는 레코드 자체에 대해서만 락(갭락에 대해서는 잠그지 않음).  
보조 인덱스를 이용한 변경작업에는 넥스트 키락 혹은 갭 락 사용.  

### 5.3.1.2 갭락
레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠근을 것을 의미한다. 
레코드와 레코드 사이의 간격에 새로운 레코드가 INSERT 되는 것을 제어한다.

### 5.3.1.3 넥스트 키 락
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금  
갭락과 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행 될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다.  
하지만 넥스트 키 락이나 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다.  
가능하면 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락이나 갭 락을 줄이는 것이 좋다.

### 5.3.1.4 자동 증가 락
- MySQL 5.0 이하 버전   
  - 내부적으로 AUTO_INCREMENT 락 이라고 하는 테이블 수준의 잠금을 사용.
  - 트랜잭션과 관계 없이, INSERT나 REPLACE 문장에서 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다.
- 5.1 이상  
  - innodb_autoinc_lock_mode라는 시스템 변수를 이용해 자동 증가 락의 작동 방식을 변경할 수 있다. 
  - 8.0부터 기본값은 2(자동 증가 락 x, 경량화된 래치 사용)

### 5.3.2 인덱스와 잠금
InnoDB의 레코드 락은 인덱스를 잠그는 방식으로 처리. UPDATE 쿼리를 통해 특정 레코드를 변경해야 한다면, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드에 모두 락을 걸어야한다.  
만약 테이블에 인덱스가 하나도 없다면? 테이블을 풀 스캔해야한다.

### 5.3.3 레코드 수준의 잠금 확인 및 해제
MySQL 8.0 버전에서는 performance_schema의 data_locks와 data_locks_waits 테이블을 이용해 잠금과 잠금 대기 순서를 확인할 수 있다.  
KILL 스레드번호 명령어를 통해 스레드를 강제 종료해 잠금 경합을 끝낼 수 있다.

## 5.4 MySQL 의 격리 수준
트랜잭션의 격리수준이란? 여러 트랜잭션이 동시에 처리도리 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것  
- "READ UNCOMMITTED", "READ COMMITTED", "REPEATABLE READ", "SERIALIZABLE" 4가지로 나뉜다.
- 격리 수준에서 뒤로 갈 수록 각 트랜잭션 간의 데이터 격리 정도가 높아지며, 동시 처리 성능도 떨어진다.
- 격리 수준의 레벨에 따라 DIRTY READ, NON-REPEATABLE READ, PHANTOM READ 세가지 부정합의 문제점이 발생할 수 있다.
- 오라클은 주로 READ-COMMITTED 수준을 많이 사용하며, MySQL 에서는 REPEATABLE READ 를 주로 사용한다.

### 5.4.1 READ UNCOMMITTED
각 트랜잭션에서의 변경 내용이 COMMIT, ROLLBACK 여부에 상관 없이 다른 트랜잭션에서 보인다. 
Dirty Read가 허용되는 격리수준  Dirty read: 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상.

### 5.4.2 READ COMMITTED
어떤 트랜잭션에서 변경한 내용이 커밋되기 전까지는 다른 트랜잭션에서 그러한 변경 내용을 조회할 수 없게 하는 격리 수준.  
COMMIT이 완료되기 전까지는 변경이 일어난 레코드를 조회하는 것이 아닌 언두 로그에 백업된 레코드를 조회
한 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때 항상 같은 결과를 가져오지 못하는 NON-REPEATABLE READ의 문제가 있다.   

### 5.4.3 REPEATABLE READ
MySQL의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준. 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.  
REPEATABLE READ 격리 수준에눈 MVCC를 보장하기 위해 실행 중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 이전 번호 언두 영역의 데이터는 삭제 할 수 없다.   
SELECT ... FOR UPDATE, SELECT .. LOCK IN SHARE MODE 같이 레코드에 쓰기 잠금을 걸어야하는 명령어는 언두 레코드에는 잠금을 걸 수 었어 언두 영역의 변경 전 데이터를 가져오는 것이 아니라 현재 레코드의 값을 가져오게 된다. 
이로 인해 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안보였다 하는 현상인 PHANTOM READ가 발샐할 수 있다.  
### 5.4.4 SERIALIZABLE
한 트랜잭션에서 읽고 쓰는 레코드는 다른 트랜잭션에서 절대 접근할 수 없다.  
InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키락 덕분에 REPEATABLE READ 격리 수준에서도 이미 "PHANTOM READ"가 발생하지 않기 때문에 SERIALIZABLE을 사용할 필요성이 없다.
