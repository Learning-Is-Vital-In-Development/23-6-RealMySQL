# 4장 MySQL 아키텍처

# 4.1 MySQL 엔진 아키텍처

MySQL 구조는 다른 DBMS에 비해 구조가 상당히 독특하다. 이로 인해 엄청난 혜택을 누릴 수 있으며, 반대로 문제되지 않을 것들이 가끔 문제가 되기도 한다.

## 4.1.1 MySQL의 전체 구조

![image](https://user-images.githubusercontent.com/61923768/232522643-6f5c8af5-c01e-4fa1-876d-7bd125579a8b.png)

MySQL 구조 (출처 : [MySQL 공식 개발자 문서](https://dev.mysql.com/doc/refman/8.0/en/pluggable-storage-overview.html))

MySQL은 대부분의 프로그래밍 언어로부터 접근 방법을 모두 지원한다.
MySQL 서버는 크게 `MySQL Engine` 과 `Storage Engines` 으로 구분할 수 있다.
이 책에서 Optimizer와 Parser 등의 기능은 Storage Engine으로 구분한다.

### 4.1.1.1 MySQL 엔진

주요 구성 요소

- 커넥션 핸들러: MySQL 엔진은 클라이언트로부터의 접속및 쿼리 요청을 처리
- SQL Parser
- 전처리기
- 옵티마이저: 쿼리 최적화된 실행

* 표준 SQL(ANSI SQL)을 지원

### 4.1.1.2 스토리지 엔진

실제 데이터를 디스크 스토리지에 저장하거나 읽어오는 부분을 전담

MySQL 엔진은 하나지만 Storage 엔진은 여러 개를 동시에 사용 가능

스토리지 엔진을 지정하면 이후 해당 테이블의 모든 읽기 작업이나 변경 작업은 정의된 스토리지 엔진이 처리한다.

스토리지 엔진은 성능 향상을 위해 키 캐시(MyISAM Storage Engine)나 InnoDB BufferPool(InnoDB Storage Engine) 같은 기능을 내장하고 있다.

```sql
-- ENGINE=INNODB not needed unless you have set a different
-- default storage engine.
CREATE TABLE t1 (i INT) ENGINE = INNODB;
-- Simple table definitions can be switched from one to another.
CREATE TABLE t2 (i INT) ENGINE = CSV;
CREATE TABLE t3 (i INT) ENGINE = MEMORY;

-- set default storage engine
SET default_storage_engine=NDBCLUSTER;

-- convert a table from one storage engine to another
ALTER TABLE t ENGINE = InnoDB;
```

ref: [https://dev.mysql.com/doc/refman/8.0/en/storage-engine-setting.html](https://dev.mysql.com/doc/refman/8.0/en/storage-engine-setting.html)

지원하는 Storage Engines 공식문서 참조

ref: [https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)


### 4.1.1.3 핸들러 API

MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리지 엔진에 요청한다. 이러한 요청을 핸들러 요청이라고 하고, 이때 사용되는 API를 핸들러 API라 한다.

이 핸들러 API를 통해 얼마나 많은 데이터(레코드) 작업이 있었는 지 아래 명령으로 확인할 수 있다.

```sql
mysql> SHOW GLOBAL STATUS LIKE 'Handler%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Handler_commit             | 591   |
| Handler_delete             | 8     |
| Handler_discover           | 0     |
| Handler_external_lock      | 6309  |
| Handler_mrr_init           | 0     |
| Handler_prepare            | 0     |
| Handler_read_first         | 41    |
| Handler_read_key           | 1728  |
| Handler_read_last          | 0     |
| Handler_read_next          | 4045  |
| Handler_read_prev          | 0     |
| Handler_read_rnd           | 0     |
| Handler_read_rnd_next      | 680   |
| Handler_rollback           | 0     |
| Handler_savepoint          | 0     |
| Handler_savepoint_rollback | 0     |
| Handler_update             | 331   |
| Handler_write              | 8     |
+----------------------------+-------+
18 rows in set (0.02 sec)
```

## 4.1.2 MySQL 스레딩 구조

MySQL 서버는 프로세스 기반이 아니라 쓰레드 기반으로 동작하며, 크게 Foreground 쓰레드와 Background 쓰레드로 구분할 수 있다.

실행 중인 쓰레드 목록

```sql
mysql> SELECT thread_id, name, type, processlist_user, processlist_host
    -> FROM performance_schema.threads ORDER BY type, thread_id;
+-----------+---------------------------------------------+------------+------------------+------------------+
| thread_id | name                                        | type       | processlist_user | processlist_host |
+-----------+---------------------------------------------+------------+------------------+------------------+
|         1 | thread/sql/main                             | BACKGROUND | NULL             | NULL             |
|         3 | thread/innodb/io_ibuf_thread                | BACKGROUND | NULL             | NULL             |
|         4 | thread/innodb/io_log_thread                 | BACKGROUND | NULL             | NULL             |
|         5 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         6 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         7 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         8 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         9 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        10 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        11 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        12 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        13 | thread/innodb/page_flush_coordinator_thread | BACKGROUND | NULL             | NULL             |
|        14 | thread/innodb/log_checkpointer_thread       | BACKGROUND | NULL             | NULL             |
|        15 | thread/innodb/log_flush_notifier_thread     | BACKGROUND | NULL             | NULL             |
|        16 | thread/innodb/log_flusher_thread            | BACKGROUND | NULL             | NULL             |
|        17 | thread/innodb/log_write_notifier_thread     | BACKGROUND | NULL             | NULL             |
|        18 | thread/innodb/log_writer_thread             | BACKGROUND | NULL             | NULL             |
|        19 | thread/innodb/log_files_governor_thread     | BACKGROUND | NULL             | NULL             |
|        24 | thread/innodb/srv_lock_timeout_thread       | BACKGROUND | NULL             | NULL             |
|        25 | thread/innodb/srv_error_monitor_thread      | BACKGROUND | NULL             | NULL             |
|        26 | thread/innodb/srv_monitor_thread            | BACKGROUND | NULL             | NULL             |
|        27 | thread/innodb/buf_resize_thread             | BACKGROUND | NULL             | NULL             |
|        28 | thread/innodb/srv_master_thread             | BACKGROUND | NULL             | NULL             |
|        29 | thread/innodb/dict_stats_thread             | BACKGROUND | NULL             | NULL             |
|        30 | thread/innodb/fts_optimize_thread           | BACKGROUND | NULL             | NULL             |
|        31 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
|        32 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
|        33 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
|        37 | thread/innodb/buf_dump_thread               | BACKGROUND | NULL             | NULL             |
|        38 | thread/innodb/clone_gtid_thread             | BACKGROUND | NULL             | NULL             |
|        39 | thread/innodb/srv_purge_thread              | BACKGROUND | NULL             | NULL             |
|        40 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        41 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        42 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        44 | thread/sql/signal_handler                   | BACKGROUND | NULL             | NULL             |
|        45 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
|        43 | thread/sql/event_scheduler                  | FOREGROUND | event_scheduler  | localhost        |
|        47 | thread/sql/compress_gtid_table              | FOREGROUND | NULL             | NULL             |
|        49 | thread/sql/one_connection                   | FOREGROUND | root             | localhost        |
+-----------+---------------------------------------------+------------+------------------+------------------+
39 rows in set (0.02 sec)
```

이 중 마지막 `thread/sql/one_connection` 스레드 만 실제 사용자의 요청을 처리하는 FOREGROUND 스레드이다.

### 4.1.2.1 포그라운드 쓰레드

포그라운드 쓰레드는 최소한 접속된 클라이언트의 수 만큼 존재하며, 주로 각 클라이언트 사용자가 요청한 쿼리 문장을 처리한다.

클라이언트 사용자가 작업을 마치고 커넥션을 종료하면 해당 커넥션을 담당하는 쓰레드는 다시 쓰레드 캐시로 되돌아간다. 이때 캐시에 일정 개수 이상의 대기 중이 쓰레드가 있다면 쓰레드를 캐시에 넣지 않고 종료시켜 일정 갯수를 유지한다. 이때 쓰레드 캐시에 유지할 수 있는 쓰레드 수는 `thread_cache_size` 시스템 변수로 설정한다.

```sql
mysql> SHOW VARIABLES LIKE 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 9     |
+-------------------+-------+
1 row in set (0.02 sec)
```

포그라운드 쓰레드는 데이터를 MySQL 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우에는 직접 `디스크 쓰기 작업까지 포그라운드 쓰레드가 처리`한다. 하지만 `InnoDB 테이블`은 데이터 버퍼나 캐시까지만 포그라운드 쓰레드가 처리하고 나머지 `버퍼로부터 디스크 기록까지 작업은 백그라운드 쓰레드가 처리`한다.

### 4.1.2.2 백그라운드 쓰레드

InnoDB는 다음과 같이 여러 작업이 백그라운드 쓰레드에서 처리된다.

- Main thread: Insert Buffer를 병합하는 쓰레드(정확히는 change buffer?)
    - The `InnoDB` `main thread` merges buffered changes when the server is nearly idle, and during a [slow shutdown](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_slow_shutdown)
      ref: [https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html)
- Log thread: 로그를 디스크로 기록하는 쓰레드
- Write thread: InnoDB Buffer Pool의 데이터를 디스크에 기록하는 쓰레드
- Read thread: 데이터를 Buffer로 읽어 오는 쓰레드
- Monitor thread: 잠금이나 데드락을 모니터링하는 쓰레드

이 중 가장 중요한 것은 로그 쓰레드와 쓰기 쓰레드일 것이다.

쓰기와 읽기 쓰레드는 시스템 변수로 개수를 설정한다.

```sql
mysql> SHOW VARIABLES LIKE '%_io_threads';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_read_io_threads  | 4     |
| innodb_write_io_threads | 4     |
+-------------------------+-------+
2 rows in set (0.02 sec)
```

- 읽기 작업은 주로 클라이언트 쓰레드에서 처리되기 때문에 많이 설정할 필요가 없다.
- 쓰기 쓰레드는 아주 많은 작업을 백그라운드로 처리하기 때문에
  → 일반적인 내장 디스크를 사용할 때는 2~4
  → DAS나 SAN 같은 스토리지를 사용할 때는 디스크를 최적으로 사용할 수 있을 만큼 충분히 설정하는 것이 좋다.

  > `DAS(Direct Attached Storage)`는 단일 서버 또는 컴퓨터에 직접 연결된 스토리지 시스템 유형입니다. 외장 하드 드라이브, USB 드라이브 및 내장 하드 드라이브를 포함한 다양한 장치와 함께 사용할 수 있는 간단하고 직접적인 스토리지 솔루션입니다. DAS는 여러 서버에서 공유할 수 없기 때문에 일반적으로 소규모 스토리지 요구 사항에 사용됩니다. 또한 확장성이 높지 않으며 수동 관리가 필요합니다.
  >
  >
  > `SAN(Storage Area Network)`은 여러 서버에 공유 스토리지를 제공하도록 설계된 보다 복잡한 스토리지 솔루션입니다. SAN은 전용 네트워크 패브릭에 연결된 스토리지 장치의 네트워크로 구성됩니다. 이를 통해 여러 서버가 동일한 스토리지 리소스에 동시에 액세스할 수 있으므로 효율성과 확장성이 향상됩니다. SAN은 일반적으로 중복성 및 재해 복구 기능이 있는 고성능 스토리지가 필요한 대기업에서 사용됩니다.
  >
  > 요약하면 DAS는 단일 서버에 직접 연결된 단순한 스토리지 솔루션인 반면 SAN은 전용 네트워크를 통해 여러 서버에 공유 스토리지를 제공하는 복합 스토리지 시스템입니다. DAS와 SAN 사이의 선택은 기업의 특정 스토리지 요구 사항에 따라 다릅니다.
  > By ChatGPT
>

사용자의 요청을 처리하는 도중 데이터 쓰기 작업은 지연(버퍼링)되어 처리할 수 있지만, 읽기 작업은 절대 지연될 수 없다. 일반적인 상용 DBMS는 대부분 쓰기 작업을 버퍼링해서 일괄 처리하는 기능이 탑재돼 있다.

InnoDB 또한 이러한 방식으로 처리한다. 하지만 MyISAM은 그렇지 않고 사용자 쓰레드가 쓰기 작업까지 함께 처리하도록 설계돼 있다.

## 4.1.3 메모리 할당과 사용 구조

### 4.1.3.1 글로벌 메모리 영역

클라이언트 쓰레드 수와 무관하게 하나의 메모리 공간만 할당된다. 필요에 따라 2개 이상 메모리 공간을 할당 받을 수 있지만 모든 쓰레드에 의해 공유된다.

대표적인 메모리 영역은 다음과 같다.

- Table Cache
- InnoDB Buffer Pool
- InnoDB Adaptive hash Index
- InnoDB Redo Log Buffer

### 4.1.3.2 로컬 메모리 영역

세션 메모리 영역이라고도 표현하면, 클라이언트 쓰레드가 쿼리를 처리하는 데 사용하는 메모리 영역이다.

대표적인 메모리 영역으로 `Connection Buffer`와 `Sort Buffer` 등이 있다.

클라이언트 커넥션으로부터 요청을 처리하기 위해 쓰레드를 하나씩 할당하게 되는 데, 이때 사용되는 메모리 공간이라고 해서 클라이언트 메모리 영역이라고 한다. 클라이언트와 MySQL 서버와의 커넥션을 세션이라고 하기 때문에 세션 메모리 영역이라고도 표현한다.

각 클라이언트 쓰레드 별로 독립적으로 할당되며, 절대 공유되어 사용되지 않는다. 쿼리의 용도별로 필요할 때만 할당되고 필요하지 않는 경우 메모리 공간을 할당조차 하지 않을 수 있다. 대표적으로 `sort buffer`와 `join buffer`가 그러하다.

커넥션이 열려있는 동안 계속 할당되는 상태로 있는 커넥션 버퍼나 결과 버퍼가 있고, 쿼리를 실행하는 순간에만 할당 되었다가 해제되는 정렬 버퍼나 조인 버퍼도 있다.

## 4.1.4 플러그인 스토리지 엔진 모델

MySQL의 독특한 구조 중 대표적인 것이 플러그인 모델이다. 기본적으로 제공되는 스토리지 엔진 이외에 부가적인 기능을 더 제공하는 스터리지 엔진이 필요할 수 있으며, 다른 전문 개발 회사 또는 사용자가 직접 스토리지 엔진을 개발하여 사용할 수 있다.

MySQL에서 쿼리가 실해되는 과정을 크게 나눈다면 거의 대부분의 작업이 MySQL 엔진에서 처리되고 마지막 `데이터 읽기/쓰기`만 스토리지 엔진 의해 처리된다.

`데이터 읽기/쓰기` 작업은 대부분 1건의 레코드 단위로 처리된다. 다른 스토리지 엔진을 사용하는 테이블에 대해 쿼리를 실행하더라도 MySQL의 처리 내용은 대부분 동일하며, 단순히 `데이터 읽기/쓰기 영역`만 차이가 있을 뿐이다. 실질적인 `GROUP BY` 나 `ORDER BY` 등 복잡한 처리는 스토리지 엔진 영역이 아니라 MySQL 엔진의 처리 영역인 쿼리 실행기에서 처리된다.

하나의 쿼리 작업은 다양한 하위 작업으로 이루어 지는 데, 각각의 하위 작업이 MySQL 엔진과 스토리지 엔진 중 어느 곳에서 처리되는 지 구분할 줄 알아야 한다.

```sql
-- 지원되는 스토리지 엔진 확인해보기
mysql> SHOW ENGINES;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
```

Suppor 컬럼은 총 4가지 상태를 표현할 수 있다.

- `YES`: MySQL 서버에 해당 스토리지 엔진이 포함되어 있고, 사용 가능상태로 활성화된 상태
- `DEFAULT`: `YES`와 동일한 상태지만, 필수 스토리지 엔진을 의미
- `NO`: 현재 MySQL 서버에는 포함되어 있지 않음을 의미
- `DISABLED`: 현재 MySQL 서버에는 포함되었지만 파라미터에 의해 비활성화된 상태

플러그인 형태로 빌드된 스토리지 엔진 라이브러리를 다운로드해서 끼워넣기만 하면 사용할 수 있다.

스토리지 엔진뿐만 아니라 모든 플러그인의 내용은 다음과 같이 확인할 수 있다.

```sql
mysql> SHOW PLUGINS;
+---------------------------------+----------+--------------------+---------+---------+
| Name                            | Status   | Type               | Library | License |
+---------------------------------+----------+--------------------+---------+---------+
| binlog                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| mysql_native_password           | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
| sha256_password                 | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
| caching_sha2_password           | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
| sha2_cache_cleaner              | ACTIVE   | AUDIT              | NULL    | GPL     |
| daemon_keyring_proxy_plugin     | ACTIVE   | DAEMON             | NULL    | GPL     |
| CSV                             | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| MEMORY                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| InnoDB                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| INNODB_TRX                      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMP                      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMP_RESET                | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMPMEM                   | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMPMEM_RESET             | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMP_PER_INDEX            | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CMP_PER_INDEX_RESET      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_BUFFER_PAGE              | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_BUFFER_PAGE_LRU          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_BUFFER_POOL_STATS        | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_TEMP_TABLE_INFO          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_METRICS                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_DEFAULT_STOPWORD      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_DELETED               | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_BEING_DELETED         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_CONFIG                | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_INDEX_CACHE           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_FT_INDEX_TABLE           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_TABLES                   | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_TABLESTATS               | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_INDEXES                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_TABLESPACES              | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_COLUMNS                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_VIRTUAL                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_CACHED_INDEXES           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| INNODB_SESSION_TEMP_TABLESPACES | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
| MyISAM                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| MRG_MYISAM                      | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| PERFORMANCE_SCHEMA              | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| TempTable                       | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| ARCHIVE                         | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| BLACKHOLE                       | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
| FEDERATED                       | DISABLED | STORAGE ENGINE     | NULL    | GPL     |
| ngram                           | ACTIVE   | FTPARSER           | NULL    | GPL     |
| mysqlx_cache_cleaner            | ACTIVE   | AUDIT              | NULL    | GPL     |
| mysqlx                          | ACTIVE   | DAEMON             | NULL    | GPL     |
+---------------------------------+----------+--------------------+---------+---------+
45 rows in set (0.02 sec)
```

다양한 기능을 플러그인 형태로 지원하는 데, 인증이나 전문 검색 파서, 쿼리 재작성과 같은 플러그인이 있고 비밀번호 검증과 커넥션 제어 등의 플러그인도 제공된다. API가 매뉴얼에 공개돼 있으므로 플러그인을 이용해 구현할 수도 있다.

## 4.1.5 컴포넌트

MySQL 8.0부터는 기존의 플러그인 아키텍처를 대체하기 위해 컴포넌트 아키텍처가 지원된다.

기존 플러그인을 아래와 같은 단점이 있었다.

- 플러그인은 오직 MySQL 서버와 인터페이스 할 수 있고, 플러그인간 통신을 할 수 없음.
- 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않다.(캡슐화 안됨)
- 플러그인은 상호 의존 관계를 설정할 수 없어서 초기화가 어려움.

MySQL 8.0의 비밀번호 검증 기능은 컴포넌트로 개선되었다. 간단한 사용법을 해당 컴포넌트를 통해 살펴보자.

```sql
mysql> INSTALL COMPONENT 'file://component_validate_password';
Query OK, 0 rows affected (0.02 sec)

mysql> SELECT * FROM mysql.component;
+--------------+--------------------+------------------------------------+
| component_id | component_group_id | component_urn                      |
+--------------+--------------------+------------------------------------+
|            1 |                  1 | file://component_validate_password |
+--------------+--------------------+------------------------------------+
1 row in set (0.01 sec)
```

## 4.1.6 쿼리 실행 구조

### 4.1.6.1 쿼리 파서

사용자 요청으로 들어온 쿼리 문장을 토큰(MySQL이 인식할 수 있는 최소 단위의 어휘나 기호)으로 분리해 트리 형태의 구조로 만드는 과정. 기본 문법 오류는 이 과정에서 발견되고 사용자에게 오류 메시지를 전달하게 된다.

### 4.1.6.2 전처리기

파서를 통해 만들어진 파서 트리를 기반으로 문장에 구조적인 문제가 있는 지 확인한다. 테이블 이름이나 칼럼 이름, 내장 함수와 같은 개체를 매핑해 존재 여부와 접근 권한 등을 확인하는 과정이 수행된다. 존재하지 않거나 권한상 사용할 수 없는 개체의 토큰은 이 단계에서 걸러진다.

### 4.1.6.3 옵티마이저

쿼리 문자을 저렴한 비용으로 가장 빠르게 처리할 지를 결정하는 역할을 담당하며, DBMS의 두뇌에 해당한다고 볼 수 있다. 옵티마이저가 선택하는 내용을 설명하고 더 나은 선택을 할 수 있도록 유도하는 가를 알려준ㄹ

### 4.1.6.4 실행 엔진

실행 엔진은 만들어진 계획대로 핸들러(스토리지 엔진)에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행한다.

### 4.1.6.5 핸들러(스토리지 엔진)

데이터를 디스크에 저장하고 읽어오는 역할을 한다.

## 4.1.7 복제

MySQL 서버에서 복제(Replication)은 매우 중요한 역할을 담당하며, 많은 발전을 거듭해왔다. 별도의 장에서 따로 다루도록 한다.

## 4.1.8 쿼리 캐시

MySQL 서버에서 쿼리 캐시(Query Cache)는 빠른 응답을 필요로 하는 응용 프로그램에서 매우 중요한 역할을 담당했다. 실행 결과를 메모리에 캐시하고, 동일한 SQL 쿼리가 실행되면 테이블을 읽지 않고 즉시 결과를 반환하기 떄문에 매우 빠른 성능을 보였따. 하지만 쿼리 캐시는 테이블의 데이터가 변경되면 캐시 저장된 결과 중에서 변경된 테이블과 관련된 것들을 모두 삭제(Invalidate)해야 했다. 이는 동시 처리 성능 저하를 유발한다.

MySQL 8.0으로 올라오면서 쿼리 캐시는 MySQL 서버의 기능에서 완전히 제거되고 관련된 시스템 변수도 제거 됐다.

## 4.1.9 쓰레드 풀

MySQL 엔터프라이즈 에디션은 쓰레드 풀(Thread Pool) 기능을 제공하지만 커뮤니티 에디션은 제공하지 않는다.  여기서는 `Percona Server`에서 제공하는 쓰레드 풀 기능을 살펴보고자 한다.

`Percona Server`에서 쓰레드 풀 플러그인 라이브러리를 설치해서 사용하면 된다. 쓰레드 풀은 내부적으로 사용자의 요청을 처리하는 쓰레드 개수를 줄여서 동시 처리되는 요청이 많다 하더라도 MySQL 서버의 CPU가 제한된 개수의 쓰레드 처리에만 집중할 수 있도록 하여 서버의 자원 소모를 줄이는 것이 목적이다.

쓰레드 풀은 앞서 설명한 것철머 동시에 실행 중인 쓰레드들을 CPU가 최대한 잘 처리해낼 수 있는 수준으로 줄여서 빨리 처리하게 하는 기능이기 때문에 스케쥴링 과정에서 CPU 시간을 제대로 확보하지 못하는 경우 쿼리 처리가 더 느려지는 사례가 발생할 수도 있따.

`Percona Server`의 스레드 풀은 기본적으로 CPU 코어의 개수만큼 쓰레드 그룹을 생성하는 데, 쓰레드 그룹의 개수는 `thread_pool_size` 시스템 변수를 변경해서 조정할 수 있다. 일반적으로는 코어의 개수와 맞추는 것이 CPU 프로세서 친화도를 높이는 데 좋다.

쓰레드 풀의 타이머 쓰레드는 주기적으로 쓰레드 그룹의 상태를 체크해서 `thread_poll_stall_limit` 시스템 변수에 정의된 밀리초만큼 작업 스레드가 지금 처리 중인 작업을 끝내지 못하면 새러온 쓰레드를 생성해 스레드 그룹에 추가한다. 이때 전체 쓰레드 풀의 쓰레드 수는 `tread_pool_max_threads` 를 넘을 수 없다.

즉, 모든 쓰레드 그룹은 `thread_poll_stall_limit` 시간을 기다려야 새로운 요청을 처리할 수 있다는 걸 의미한다. 따라서 응답시간이 아주 민감하다면 `thread_pool_stall_limit` 시스템 변수를 낮춰서 설정해야 한다.

## 4.1.10 트랜잭션 지원 메타데이터

데이터베이스 서버에서 테이블의 구조 정보와 스토어드 프로그램 등의 정보를 메타데이터라고 한다.

MySQL 5.7까지는 테이블 구조를 FRM 파일에 저장하고 일부 스토어드 프로그램 또한 파일 기반으로 관리했다. 하지만 이런 파일 기반의 메타데이터는 생성 및 변경 작업이 트랜잭션을 지원하지 않기 때문에 비정상적인 종료 시에 일관되지 않는 상태로 남는 문제가 있었다.

MySQL 8.0에서는 이러한 문제를 해결하기 위해 모든 메타데이터를 InnoDB의 테이블에 저장하도록 개선됐다. MySQL 서버가 작동하기 위해 기본적으로 필요한 테이블들을 묶어서 시스템 테이블이라고 한다. 이러한 시스템 테이블에는 대표적으로 사용자 인증과 권한에 관련된 테이블이 있다. 이런 시스템 테이블을 모두 InnoDB 스토리지 엔진을 사용하도록 개선했으며, 시스템 테이블과 메타 데이터 정보를 모두 모아서 `mysql DB`에 저장하고 있다. `mysql DB`는 통째로 `mysql.ibd` 라는 이름의 테이블 스페이스에 저장된다.

```bash
ls var/lib/mysql/mysql.ibd -al          
# -rw-r----- 1 mysql mysql 31457280 Apr 17 21:37 var/lib/mysql/mysql.ibd
```

MySQL 서버에 InnoDB 스토리지 엔진을 사용하는 테이블은 메타 정보가 저장되지만, MyISAM이나 CSV 등과 같은 스토리지 엔진의 메타 정보는 여전히 저정할 공간이 필요하다. 이를 위해 InnoDB 스토리지 엔진 외 다른 엔진을 사용하는 테이블들은 SDI(Serialized Dictionary Information) 파일을 사용한다. `*.sdi` 파일이 존재하며 기존의 `*.FRM` 파일과 동일한 역할을 한다.

# 4.2 InnoDB 스토리지 엔진 아키텍처

# 4.3 MyiSAM 스토리지 아키텍처

# 4.4 MySQL 로그 파일
