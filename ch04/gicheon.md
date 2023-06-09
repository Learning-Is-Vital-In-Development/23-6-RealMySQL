# CH 04. 아키텍처

# 4.1 MySQL 엔진 아키텍쳐

## 4.1.1 MySQL의 전체구조

MySQL 서버는 다른 DBMS에 비해 구조가 독특하다

MySQL 서버는 크게 2가지로 구분할 수 있다.

- MySQL 엔진
- 스토리지 엔진

### 4.1.1.1 MySQL 엔진

- 커넥션 핸들러
    - 클라이언트로부터의 접속 및 쿼리 요청을 처리
- SQL 파서
    - 요청의 의한 Query를 토큰으로 분리해 MySQL 이 인식할 수 있는 트리 형태의 구조로 만들고, 기본 문법 오류가 이 과정에서 발견됨
- 전처리기
    - 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인
- 옵티마이저
    - Query를 얼마나 낮은 비용으로 효율적으로 처리할지 결정하는 역할 수행
    - Query 재작성, 스캔 순서 조정 및 인덱스의 선택같은 작업 수행

이 4가지가 중심을 이룬다.

요청된 SQL 문장으 분석하거나 최적화 하는 등 DBMS의 두뇌에 해당하는 처리를 수행한다.

### 4.1.1.2 스토리지 엔진

데이터를 디스크 스토리지에 저장

디크스 스토리지로부터 데이터 읽어오기

- MySQL 서버에서
    - MySQL엔진은 1개
    - 스토리지 엔진은 여러개를 동시에 사용 가능

### 4.1.1.3 핸들러 API

- 핸들러
    - MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 읽어오는 역할 담당
- 핸들러 API
    - 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때 각 스토리지 엔진에 쓰기 또는 읽기를 요청할 때 사용되는 API

## 4.1.2 MySQL 스레딩 구조

MySQL 서버는 프로세스 기반이 아닌 스레드 기반으로 작동

- 포그라운드 스레드
- 백그라운드 스레드

### 4.1.2.1 포그라운드 스레드(클라이언트 스레드)

 MySQL 서버에 접속된 클라이언트 사용자가 요청하는 쿼리 문장을 처리

- 커넥션을 종료하면 해당 커넥션을 담당하던 스레드는 다시 스레드 캐시로 돌아가게 되는데, thread_cache_size 에 설정된 수에 따라 이미 스레드 캐시에 대기중이라면 스레드 캐시에 넣지 않고 스레드를 종료 시킨다.
- 데이터를 버퍼나 캐시로부터 가져온다. 버퍼나 캐시가 없는 경우 디스크의 파일로부터 읽어와서 처리한다.
- MyISAM 테이블은 디스크 쓰기 작업까지 포그라운드 스레드가 처리하지만 InnoDB 테이블은 버퍼나 캐시까지만 포그라운드 스레드가 처리하고, 디스크 기록하는 작업은 백그라운드 스레드가 처리한다.

### 4.1.2.2 백그라운드 스레드

MyISAM 은 해당사항이 별로 없지만 InnoDB는 여러 작업이 백그라운드로 처리된다

- 인서트 버퍼(Insert Buffer)를 병합하는 스레드
- 로그를 디스크로 기록하는 스레드
- InnoDB 버퍼 풀의 데이터를 디스크에 기록하는 스레드
- 데이터를 버퍼로 읽어 오는 스레드
- 잠금이나 데드락을 모니터링하는 스레드

데이터 쓰기 작업은 지연되도 읽기 작업은 절대 지연될 수 없다.

InnoDB는 쓰기 작업을 버퍼링해서 일괄 처리하는 기능이 있음

- INSERT, UPDATE, DELETE 쿼리로 데이터가 변경되는 경우 데이터가 디스크의 데이터 파일로 완전히 저장될 때까지 기다리지 않아도 된다.
- MyISAM 에서는 일박저인 쿼리는 쓰기 버퍼링 기능을 제공할 수 없다.

## 4.1.3 메모리 할당 및 사용 구조

- 글로리 메모리 영역
    - MySQL 서버를 실행할 때 운영체제로 부터 설정한 만큼 할당
    - 필요에 따라 2개 이상 할당받을 수 있지만 각 공간이 모든 스레드에 의해 공유됨
    
    - 테이블 캐시
    - InnoDB 버퍼 풀
    - InnoDB 어댑티브 해시 인덱스
    - InnoDB 리두 로그 버퍼
    
- 로컬 메모리 영역
    - 클라이언드 스레드가 쿼리를 처리하는데 사용하는 메모리 영역
    - 대표적으로 커넥션 버퍼와 정렬 버퍼가 있음
    - 각 클라리어트 스레드 별로 독립적으로 할당되며 절대 공유되지 않는다
    - 쿼리의 용도별로 필요할 때만 할당되고 필요하지 않은 경우 메모리 공간을 할당조차 아힞 ㅏ않을 수 있다 (대표적으로 소트 버퍼나 조인 버퍼)
    - 커넥션이 열려 있는 동안 계속 할당된 상태로 남아 있는 공간과
        - 커넥션 버퍼
        - 결과 버퍼
    - 쿼리를 실행하는 순사에만 할당했다가 해제하는 공간이 있다
        - 소트 버퍼
        - 조인 버퍼
    
    대표적인 로컬 메모리 영역은 다음과 같다.
    
    - 정렬 버퍼
    - 조인 버퍼
    - 바이너리 로그 캐시
    - 네트워크 버퍼

## 4.1.4 플러그인 스토리지 엔진 모델

MySQL 에서 독특한 구조 중 대표적인 것이 플러그인 모델이다

MySQL 서버에 추가적인 기능을 제공하는 외부 소프트웨어 모듈을 로드할 수 있는 방법을 제공하는 아키텍처이다

- MySQL 에서 쿼리가 실행되는 과정
    
    SQL 파서 ↔ SQL 옵티마이저 ↔ SQL 실행기 ↔ 데이터 읽기 쓰기 ↔ 디스크
    
1. 대부분 작업은 MySQL 엔진(사람 역할)에서 처리된다
2. 마지막 데이터 읽기/쓰기 작업만 스토리지 엔진(자동차 역할)에서 처리된다

MySQL 엔진이 각 스토리지 엔진에게 데이터를 읽어오거나 저장하도록 명령하려면 반드시 핸들러를 통해야 한다

GROUP BY, ORDER BY 등 복잡한 처리는 스토리지 엔진 영역이 아니라 MySQL 엔진의 처리 영역인 쿼리 실행기에 처리된다

하나의 쿼리 작업은 여러 하위 작업으로 나뉘는데 각 하위 작업이 MySQL 엔진 영역에서 처리되는지 아니면 스토리지 엔진 연역에서 처리되는지 구분할 줄 알아야 한다.

## 4.1.5 컴포넌트

MySQL 서버의 플러그인은 몇 가지 단점이 있다

- 플러그인은 오직 MySQL 서버와 인터페이스를 할 수 있고, 플러그인끼리 통신할 수 없음
- 플러그인은 MySQL 서버의 벼수나 함수를 직접 호출하기 때문에 안전하지 않음(캡슐화가 안 됨)
- 플러그인은 상호 의존 관계를 설정할 수  없어서 초기화가 어려움

이러한 단점들을 컴포넌트가 보안해준다

예시로 MySQL 5.7 까지는 비밀번호 검증 기능이 플러그인 형태로 제공됐지만, MySQL 8.0 부터는 컴포넌트로 개선되었다

## 4.1.6 쿼리 실행 구조


### 4.1.6.1 쿼리 파서

사용자 요청으로 들어온 쿼리 문장을 토큰으로 분리해 트리형태 구조로 만들어내는 작업

쿼리 문장의 기본 문법 오류는 이 과정에서 발견되고 오류 메시지를 전달한다.

### 4.1.6.2 전처리기

파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인한다

### 4.1.6.3 옵티마이저

사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할지를 결정하는 역할을 담당하며, DBMS의 두뇌에 해당한다

### 4.1.6.4 실행 엔진

실행 엔진은 만들어진 계획대로 핸들러(스토리지 엔진)에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행한다

### 4.1.6.5 핸들러 (스토리지 엔진)

데이터를 디스크로 저장하고 디스크로부터 읽어 오는 역하을 담당한다

핸들러는 스토리지 엔진을 의미하며

MyISAM 테이블을 조작하는 경우에는 핸들러가 MyISAM 스토리지 엔진이 되고 

InnoDB 테이블을 조작하는 경우 핸들러가 InnoDB 스토리지 엔진이 된다.

## 4.1.7 복제

MySQL 서버에서 복제는 중요한 역할을 담당한다 

## 4.1.8 쿼리 캐시

빠른 응답을 필요로 하는 웹 기반 응용 프로그램에서 매우 중요한 역할을 담당했다

쿼리 캐시는 SQL 실행 결과를 메모리에 캐시하고 동일한 SQL 쿼리가 실행되면 테이블을 읽지 않고 즉시 결과를 반환하기 때문이 빠른 성능을 보여준다

그러나 테이블의 데이터가 변경되면 캐시에 저장된 결과중에 관련있을 경우 삭제해야 했다

→ 심각한 동시 처리 성능 저하 유발

결국 MySQL8.0 부터 삭제됐다

## 4.1.9 스레드 풀

내부적으로 사용자의 요청을 처리하는 스레드 개수를 줄여서 동시 처리되는 요청이 많다하더라도 MySQL 서버의 CPU 제한된 개수의 스레드 처리에만 집중할 수 있게 해서 서버의 자원 소모를 줄이는 것이 목적이다.

스레드 풀이 실제 서비스에서 눈에 띄는 성능 향상을 보여주는 경우는 드물다.

## 4.1.10 트랜잭션 지원 메타데이터

MySQL 5.7 버전까지 메타 데이터를 다른 파일로 만들어 저장, 관리했으나,메타 데이터 파일 생성과 변경 작업은 트랜잭션을 지원하지 않기 때문에 테이블의 생성/변경 중에 서버가 비정상적으로 종료되면 데이터가 오염되었다.

MySQL 8.0 버전부터 문제 해결을 위해모든 메타 데이터와 사용자의 인증과 권한에 관련된 시스템 테이블을 InnoDB를 통해 테이블로 저장한다

하지만 InnoDB를 제외한 다른 스토리지 엔진들은 여전히 파일을 따로 저장해야 하기 때문에 데이터를 직렬화하여 sdi라는 파일로 저장하게 된다

# 4.2 InnoDB 스토리지 엔진 아키텍쳐

InnoDB는 MySQL 에서 사용할 수 있는 스토리지 엔진 중 거의 유일하게 레코드 기반의 잠금을 제공하며, 높은 동시성 처리가 가능하고 안정적이고 성능이 뛰어나다

## 4.2.1 프라이머리 키의 의한 클러스터링

InnoDB의 모든 테이블은 기본적으로 프라이머리 키를 기준으로 클러스러팅되어 저장된다

(순서대로 디스크에 저장된다)

모든 세컨더리 인덱스는 레코드의 주소 대신 프라이머리 키의 값을 논리적인 주소로 사용한다

프라이머리 키가 클러스터링 인덱스이기 때문에 프라이머리 키를 이용한 레인지 스캔이 빨리 처리된다 → 쿼리의 실행 계획에서 프라이머리 키는 기본적으로 다른 보조 인덱스의 비해 높게 설정된다.

MyISAM 엔진에서는 클러스터링 키가 지원하지 않는다 그래서 프라이머리 키와 세컨더리 인덱스는 구조적으로 아무런 차이가 없다

## 4.2.2 외래 키 지원

InnoDB 에서 외래 키는 부모 테이블과 자식 테이블 모두 해당 칼럼에 인덱스 생성이 필요하고 변경 시에는 반드시 부모 테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업이 필요하므로 잠금이 여러 테이블로 전파된다 → 그로 인해 데드락이 발생할 때가 많으니 개발할때도 외래 키의 존재에 주의하는 것이 좋다.

수동으로 데이터를 적재하거나 스키마 변경등에 작업에 실패할 때가 있다

이럴 경우 foreign_key_check 시스템 변수를 OFF 로 설정하면 외래 키 관계에 대한 체크 작업을 임시로 멈출 수 있다.

## 4.2.3 MVCC(Multi Version Concurrency Control)

레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능

잠금을 사용하지 않은 일관된 읽기를 제공하는 데에 있다.

InnoDB 같은 경우 Undo log 를 이용하여 이 기능을 구현한다.

MySQL 서버에 설정된 격리 수준에 따라 

- 격리 수준이 READ_UNCOMMITTED
    - InnoDB 버퍼 풀이나 데이터 파일로부터 변경되지 않은 데이터를 읽어서 반환
    - 커밋됐든 아니든 변경된 상태의 데이터를 반환한다.
- 그 이상 격리 수준(REPEATABLE_READ, SERIALIZABLE)
    - 아직 커밋되지 않았기 때문에 InnoDB 버퍼 풀이나 데이터파일에 있는 내용 대신 변경하기 이전의 내용을 보관하고 있는 언두 영역의 데이터를 반환한다

## 4.2.4 잠금 없는 일관된 읽기 (Non-Locking Consistent Read)

InnoDB 스토리지 엔진은 MVCC 기술을 이용해서 잠금을 걸지 않고 읽기 작업을 수행한다

→ 읽기 작업은 다른 트랜잭션이 가지고 있는 잠금을 기다리지 않고 읽기가 가능하다

오랜 시간 동안 활성 상태인 트랜잭션으로 인해 MySQL 서버가 느려지거나 문제가 발생할 때가 가끔 있는데, 이러한 일관된 읽기를 위헤 언두 로그를 삭제하지 못하고 계속 유지해야 하기 때문에 발생하는 문제다.

따라서 트랜잭션이 시작됐다면 가능한 한 빨리 롤백이나 커밋을 통해 트랜잭션을 완료하는 것이 좋다

## 4.2.5 자동 데드락 감지

InnoDB 스토리지 엔진은 내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프 형태로 관리한다

InnoDB 스토리지 엔진은 데드락 감지 스레드를 가지고 있어서 데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션들을 찾아서 그중 하나를 강제 종료 한다

종료를 판단하는 기준은 트랜잭션의 언두 로그 양이고 언두 로그 레코드를 더 적게 가진 트랜잭션이 일반적으로 롤백의 대상이 된다.

일반적으로 데드락 감지 스레드가 트랜잭션 잠금 목록을 검사해 데드락을 찾는 것은 부담되지 않는다

하지만 동시 처리 스레드가 많아지거나 각 트랜잭션의 잠금 갯수가 많아지면 느려지게 된다.

## 4.2.6 자동화된 장애 복구

InnoDB 스토리지 엔진은 매우 견고해서 데이터 파일이 손상되거나 MySQL 서버가 시작되지 못하는 경우는 거의 발생하지 않는다.

만약 디스크나 서버 하드웨어 이슈로 자동으로 복구하지 못한다면 복구하기 쉽지 않다.

이때는 MySQL 서버의 설정 파일에 innodb_force_recovery 시스템 변수를 설정해서 MySQL 서버를 재시작 해야된다.

만약 그래도 시작되지 않는다면 백업을 이용해서 다시 구축을 해야된다.

## 4.2.7 InnoDB 버퍼 풀

디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해두고 쓰기 작업을 일괄 처리할 수 있게 버퍼링 해주는 역할을 한다

### 4.2.7.1 버퍼 풀의 크기 결정

InnoDB 의 버퍼 풀은 innodb_buffer_pool_size 시스템 변수로 크기를 설정할 수 있고, 동적으로 확장할 수 있다

사이즈를 늘리는 것은 괜찮지만 줄일 경우 서비스에 영향이 매우 크니 주의하자

### 4.2.7.2 버퍼 풀의 구조

버퍼 풀이라는 거대한 메모리 공간을 페이지 크기의 조각으로 쪼개어,

InnoDB 스토리지 엔진이 데이터를 필요로 할 때 해당 데이터 페이지를 읽어서 각 조각에 저장한다.

LRU 리스트와 Flush 리스트, Free 리스트라는 3개의 자료 구조를 통해 버퍼 풀의 페이지 크기 조각을 관리한다.

- LRU 리스트를 관리하는 목적은 디스크로부터 한 번 읽어온 페이지를 최대한 오랫동안 InnoDB 버퍼 풀의 메모리에 유지해서 디스크 읽기를 최소화 하는 것이다.
- 자주 사용될 수록 MRU 영역에 계속 있고 읽히지 않으면 InnoDB 버퍼 풀에서 제거된다.
- Flush 리스트는 디스크로 동기화하지 않는 데이터를 가진 데이터 페이지의 변경 시점 기준의 페이지 목록을 관리한다
    - 데이터가 변경되면 InnoDB는 변경 내용을 리두 로그에 기록하고 버퍼 풀의 데이터 페이지에도 변경 내용을 반영한다.

### 4.2.7.3 버퍼 풀과 리두 로그

InnoDB 버퍼 풀은 서버의 메모리가 허용하는 만큼 설정하면 할수록 쿼리의 성능이 빨라진다

- 데이터 캐시
- 쓰기 버퍼링

2가지 용도가 있다.

버퍼 풀의 메모리 공간을 늘리는 거는 데이터 캐시 기능만 향상 시키는 것이다.

버퍼 풀은 디스크에 읽은 상태로 전혀 변경되지 않은 클린 페이지와 변경된 데이터를 가진 더티 페이지를 가지고 있다

데이터 변경이 계속 발생하면 리두 로그 파일에 기록됐던 로그 엔트리는 어느 순간 다시 새로운 로그 엔트리로 덮어 쓰인다

InnoDB 스토리지 엔진은 전체 리두 로그 파일에서 재사용한 공간과 재사용이 불가능한 공간으로 관리해야 된다

재사용이 불가능한 공간은 활성 리두 로그(Active Redo Log)라고 한다

- LSN(Log Sequence Number)
    
    리두 로그 파일의 공간은 계속 순한되어 재사용 되자만 매번 기록될 때마다 로그 포지션은 계속 증가된 값을 가진다
    

## 4.2.7.4. 버퍼 풀 플러쉬

InnoDB 스토리지 엔진은 버퍼 풀에서 아직 디스크로 기록되지 않은 더티 페이지들을 성능상의 악영향 없이 디스크에 동기화 하기 위해 

- 플러시 리스트 플러시
    - InnoDB 는 주기적으로 플러시 함수를 호출해서 동기화 작업을 한다
    - InnoDB 버퍼 풀에 더티 페이지가 많으면 디스크 쓰기 폭발 현상이 발생할 수도 있다.
- LRU 리스트 플러시
    - 사용 빈도가 낮은 데이터 페이지들을 제거해서 공간을 만든다

기능을 제공한다

## 4.2.8 Double Write Buffer

InnoDB 스토리지 엔진의 리두 로그는 리두 로그 공간의 낭비를 막기 위해 페이지의 변경된 내용만 기록한다.

- 더티 페이지를 디스크 파일로 플러시 할때 일부만 기록되는 문제가 있다
- 이러한 현상을 파셜 페이지, 톤 페이지 라고 한다

InnoDB 에서는 이러한 문제를 막기 위해 Double-Write 기법을 쓴다.

- 데이터 안정성을 위해 자수 사용된다
- 데이터 무결성이 매우 중요한 서비스에서는 Double-Write 활성화를 고려해야한다.

## 4.2.9 언두 로그

InnoDB 스토리지 엔진은 트랜잭션과 격리 수준을 보장하기 위해 DML 로 변경되지 전에 별도로 백업한다.

백업된 데이터를 Undo Log 라고 한다.

- 트랜잭션 보장
    - 롤백 대비용이다.
- 격리 수준 보장
    - 트랜잭션 격리 수준을 유지하면서 높은 동시성을 제공한다

언두 로그가 저장 되는 공간을 Undo TableSpace 라고 한다

## 4.2.10 체인지 버퍼

레코드가 인서트 되거나 업데이트 될떄

테이블에 포함된 인덱스를 업데이트 하는 작업도 필요하다

InnoDB에서 변경할 인덱스 페이지가 버퍼 풀에 없을 때 임시 공간에 저장해두는데 임시 공간 메모리를 체인지 버퍼라고 한다

체인지 버퍼는 기본적으로 InnoDB 버퍼 풀로 설정된 메모리 공간의 25% 까지 사용할 수 있도록 설정되있다

## 4.2.12 어댑티브 해시 인덱스

일반적으로 인덱스는 테이블에 사용자가 생성해 둔 B-Tree 인덱스를 의미한다.

어댑티브 해시 인덱스는 사용자가 수동으로 생성하는 인덱스가 아니라 InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터를 자동으로 생성해두는 인덱스 이다.

InnoDB 스토리지 엔진의 어댑티브 해시 인덱스는 성능상의 이점이 있지만

의도적으로 비활성화 하는경우도 많다

- 디스크 읽기가 많은 경우
- 특정 패턴의 쿼리가 많은 경우 (조인이나 Like)
- 매우 큰 데이터를 가진 테이블의 레코드를 폭넓게 이근 경우

그러나 이러한 경우에는 도움이 된다

- 디스크의 데이터가 InnoDB 버퍼 풀 크기와 비슷한 경우
- 동등 조건 검색이 많은 경우
- 쿼리가 데이터 중에서 일부 데이터에만 집중되는 경우

또한, 어댑티브 해시 인덱스는 삭제에도 영향을 많이 미친다

# 4.3 MyISAM 스토리지 엔진 아키텍쳐

MyISAM 스토리지 엔진의 성능에 영향을 미치는 요소는 아래 2가지 요소 이다

- 키 캐시
    - InnoDB 버퍼 풀과 비슷한 역하을 한다.
    - 인덱스만을 대상으로 작동한다
    - 인덱스의 디스크 쓰기 작업에 대해서만 부분적으로 버퍼링을 한다
- 운영체제의 캐시/버퍼
    - MyISAM 테이블의 읽기 쓰기 작업흔 운영체제의 디스크 읽기 또는 쓰기 작업으로 요청된다
    - InnoDB 처럼 전문적으로 캐시나 버퍼링을 못하지만 없는 것 보단 낫다

데이터 파일과 프라이머리 키(인덱스)구조

- InnoDB는 클러스터링에 저장되는 반면 MyISAM 테이블은 데이터 파일이 Heap 공간처럼 활용된다.
    - 프라이머리 키 값과 무관하게 INSERT 되는 순서대로 데이터 파일에 저장된다
    - MyISAM에 저장되는 레코드는 모두 ROWID 라는 물리적인 주솟값을 가진다
    - ROWID 는 가변길이과 고정길이 2가지 방법으로 저장될 수 있다.

# 4.4 로그 파일

로그 파일을 이용하면 MySQL 서버의 깊은 내부 지식이 없어도 상태나 부하를 일으키는 원인을 쉽게 찾아서 해결할 수 있다

- 에러 로그 파일
    - 설정파일에서 log_error 라는 파라미터로 정의된 경로에 생성된다
    - 또는 데이터 디렉토리에 .err 확장자가 붙은 파일로 생성된다
- 제네럴 쿼리 로그
    - 실행되는 쿼리로 어떤 것이 있는지 검토할 수 있다
    - 슬로우 쿼리 로그와는 다르게 MySQL 이 쿼리 요청을 받으면 바로 기록하기 떄문에 쿼리 실행 중에 에러가 발생해도 일단 로그파일에 기록된다
- 슬로우 쿼리 로그
    - 로그 파일에 시스템 변수에 설정한 시간 이상의 시간이 소요된 쿼리가 모두 기록된다
    - 반드시 쿼리가 정상적으로 실행이 완료돼야 슬로우 쿼리 로그에 기록될 수 있다.