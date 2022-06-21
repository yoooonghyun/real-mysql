# 5. 트랜잭션과 잠금


#### 트랜잭션 (Transaction)
- 데이터 정합성을 보장하기 위한 기능.
- 논리적인 작업 셋을 묶어 처리하지 못할 경우에 원 상태로 복구.
- Partial update를 막아 작업의 완전성 보장.

#### 잠금 (Lock)
- 동시성을 제어하기 위한 기능.
- 여러 커넥션에서 동시에 동일한 자원을 요청할 경우, 한 시점에는 하나의 커넥션만 변경.
- 트랜잭션 내 / 트랜잭션 간의 작업 내용을 어떻게 공유 / 차단할 지 결정.

## 5.1 트랜잭션

### 5.1.1 MySQL에서의 트랜잭션



- 하나의 논리적인 작업 셋이 ```COMMIT```되어 100% 적용되거나 ```ROLLBACK```되어 아무것도 적용되지 않음을 보장.
- 

#### ex. 5.1.1
innoDB 테이블과 MyISAM 테이블의 동작 차이

```
// MyISAM

mysql> CREATE TABLE tab_mysam ( fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=MyISAM;
mysql> INSERT INTO tab_mysam (fdpk) VALUSE (3);

// AUTO-COMMIT 활성화
mysql> SET autocommit=ON;
mysql> INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);

mysql> SELECT * FROM tab_myisam
```
|fdpk|
|----|
|1   |
|2   |
|3   |

- 1, 2를 저장하고 3을 저장하다 실패.
- ```INSERT```가 실패했음에도 전체가 ```ROLLBACK``` 되지 않고 partial update가 일어남.
- Memory storage engine을 사용하더라도 이와 같이 동작.

```
// InnoDB

mysql> CREATE TABLE tab_mysam ( fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=MyISAM;
mysql> INSERT INTO tab_mysam (fdpk) VALUSE (3);

// AUTO-COMMIT 활성화
mysql> SET autocommit=ON;
mysql> INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);

mysql> SELECT * FROM tab_innodb
```
|fdpk|
|----|
|3   |

- 1, 2를 저장하고 3을 저장하다 실패.
- ```INSERT```가 실패로 인해 전체 transaction ```ROLLBACK```.


### 5.1.2 주의사항

Transation을 오래 점유하게 되면 성능 저하를 발생.
- Transaction과 관련 없는 작업은 transaction이 시작하기 전 처리.
- Transaction안에 network I/O 지양.
- 단순 조회 작업은 transaction에 포함 X.

## 5.2 MySQL 엔진의 잠금

#### 스토리지 엔진 레벨 잠금

#### MySQL 엔지 레벨 잠금
- 모든 스토리지 엔진에 영향.
- Metadata lock
- Named lock

### 5.2.1 글로벌 락

- ```FLUSH TABLES WITH READ LOCK``` 명령어를 통한 획득
- MySQL에서 제공하는 잠금 중 가장 큰 범위.
- 한 세션에서 글로벌 락을 실행하게 되면 SELECT를 제외한 DDL 문장이나 DML 문장은 대기 상태.
- 영향 범위는 MySQL 서버 전체.
- MyISAM이나 MEMORY 테이블에 대해 mysqldump로 일관된 백업을 받아야할 때 사용.

```
DDL (Data definition language)
- CREATE
- ALTER
- DROP
- TRUNCATE
- RENAME

DML(Data manipulation language)
- INSERT
- UPDATE
- DELETE
```

### 백업락

- InnoDB는 transaction을 지원하기 때문에 일관된 데이터 상태를 위해 모든 데이터 변경 작업을 멈출 필요 X.
- 좀더 가벼운 글로벌 락의 필요.
- Xtrabackup / Enterprise backup 같은 백업 툴들의 안정성을 위한 도입.


백업락 획득 시 제한 
- 테이블의 스키마 / 사용자 인증 관련 정보 변경 불가.
- 데이터베이스 및 테이블 등 모든 객체 생성 / 변경 / 삭제
- REPAIR TABLE과 OPTIMIZE TABLE 명령
- 사용자 관리 및 비밀번호 변경

기존 MySQL 복제 시 문제
- Global lock 이용 시 Replica server의 복제는 백업 시간만큼 지연.
- 백업 도중 스키마 변경 시 실패.
- 백업 락 도입 이후에는 복제 도중 DDL 명령시 일시 중지.

### 5.2.2 테이블 락

- 테이블 단위로 설정되는 잠금


획득: ```LOCK TABLES table_name [ READ | WRITE ]```

반환: ```UNLOCK TABLES```
