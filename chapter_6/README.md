# 데이터 압축

데이터의 크기는 쿼리의 처리 성능 및 백업 및 복구 시간과 연관이 있다.
이를 줄이기 위해서 데이터 압축을 사용하는데,
MySQL 서버에서 사용 가능한 압축 방식은 크게 테이블 압축과 페이지 압축의 두 가지 종류로 구분할 수 있다.

## 페이지 압축

페이지 압축은 "Transparent Page Compression"이라고도 불리는데, MySQL 서버가 디스크에 저장하는 시점에
데이터 페이지가 압축되어 저장되고, 반대로 MySQL 서버가 디스크에서 데이터 페이지를 읽어올 때 압축이
해제되기 때문이다. 즉 버퍼 풀에서는 압축이 해제된 상태로만 데이터 페이지를 관리한다.

한편 여기서 발생하는 문제점은 16KB 데이터 페이지를 압축한 결과가 용량이 얼마나 될지 예측이 불가능한데 
적어도 하나의 테이블은 동일한 크기의 페이지(블록)로 통일돼야 한다는 것이다.

그래서 페이지 압축 기능은 운영체제별로 특정 버전의 파일 시스템에서만 지원되는 펀치 홀(Punch hole)이라는 
기능을 사용한다. [https://stackoverflow.com/questions/13982478/what-is-file-hole-and-how-can-it-be-used](https://stackoverflow.com/questions/13982478/what-is-file-hole-and-how-can-it-be-used)
>```punch a hole : file에서 사용되지 않는 부분을 표시하는 기능으로 실제로 해당 부분의 데이터를 사용하지 않는다.```

- 16KB 페이지를 압축 (압축 결과를 7KB로 가정)
- MySQL 서버는 디스크에 압축된 결과 7KB를 기록 (이때 MySQL 서버는 압축 데이터 7KB에 9KB 의 빈 데이터를 기록)
- 디스크에 데이터를 기록한 후, 7KB 이후의 공간 9KB에 대해 펀치 홀(Punch-hole)을 생성
- 파일 시스템은 7KB만 남기고 나머지 디스크의 9KB 공간은 다시 운영체제로 반납

하지만 다음과 같은 문제점이 존재한다.


- 펀치 홀 기능은 운영체제뿐만 아니라 하드웨어 자체에서도 해당 기능을 지원해야 사용 가능하다
- 파일 시스템 관련 명령어가 펀치 홀을 지원하지 못한다.

이런 이유로 실제 페이지 압축은 많이 사용되지 않는다. 만약 사용하고 싶다면 다음과 같은 COMPRESSION 옵션을 설정하면 된다.

``` SQL
-- // 테이블 생성 시
mysql> CREATE TABLE t1 (c1 INT) COMPRESSION="zlib";
-- // 테이블 변경 시
mysql> ALTER TABLE t1 COMPRESSION="zlib";
mysql> OPTIMIZE TABLE t1;
```

## 테이블 압축

테이블 압축은 운영체제나 하드웨어에 대한 제약 없이 사용할 수 있다.
하지만 다음과 같은 단점이 존재한다

- 버퍼 풀 공간 활용률이 낮음
- 쿼리 처리 성능이 낮음
- 빈번한 데이터 변경 시 압축률이 떨어짐

### 압축 테이블 생성

테이블 압축을 사용하기 위한 전제 조건으로 압축을 사용하려는 테이블이 별도의 테이블 스페이스를 사용해야 한다.
이를 위해서는 innodb_file_per_table 시스템 변수가 ON으로 설정된 상태에서 테이블이 생성돼야 한다.
또한 테이블 압축을 사용하는 테이블은 테이블을 생성할 때 ROW_FORMAT=COMPRESSED 옵션을 명시해야 한다.
추가로 KEY_BLOCK_SIZE 옵션을 이용해 압축된 페이지의 타깃 크기(목표 크기)를 명시하는데 2n(n 값은 2 이상)으로만 설정할 수 있다.
InnoDB 스토리지 엔진의 페이지 크기가 16KB라면 KEY_BLOCK_SIZE는 4KB 또는 8KB만 설정할 수 있다.
그리고 페이지 크기가 32KB 또는 64KB인 경우에는 테이블 압축을 적용할 수 없다.

```SQL
mysql> SET GLOBAL innodb_file_per_table=ON;

-- // ROW_FORMAT 옵션과 KEY_BLOCK_SIZE 옵션을 모두 명시
mysql> CREATE TABLE compressed_table(
        c1 INT PRIMARY KEY
       )
       ROW_FORMAT=COMPRESSED
       KEY_BLOCK_SIZE=8;

-- // KEY_BLOCK_SIZE 옵션만 명시
mysql> CREATE TABLE compressed_table(
        c1 INT PRIMARY KEY
       )
       KEY_BLOCK_SIZE=8;
```

두 번째 테이블 생성 구문에서와같이 ROW_FORMAT 옵션이 생략되면 자동으로 ROW_FORMAT=COMPRESSION 옵션이 추가되어 생성된다.

>``` innodb_file_per_table 시스템 변수가 0인 상태에서 제너럴 테이블스페이스에 생성되는 테이블도 테이블 압축을 사용할 수 있다.
하지만 제약 사항을 검토할 필요가 있다. ```

한편 데이터 페이지를 압축한 용량이 얼마가 되는지 알 수 없는데 어떻게 KEY_BLOCK_SIZE를 미리 설정할 수 있을까?
InnoDB 스토리지 엔진이 압축을 적용하는 방법은 다음과 같다.

```
1. 16KB의 데이터 페이지를 압축
  1.1 압축된 결과가 8KB 이하이면 그대로 디스크에 저장(압축 완료)
  1.2 압축된 결과가 8KB를 초과하면 원본 페이지를 스플릿(split)해서 2개의 페이지에 8KB씩 저장
2. 나뉜 페이지 각각에 대해 KEY_BLOCK_SIZE에 도달 할때까지 "1"번 단계를 반복 실행
```

### KEY_BLOCK_SIZE 결정

테이블 압축에서 가장 중요한 부분은 압축된 결과가 어느 정도가 될지를 예측해서 KEY_BLOCK_SIZE를 결정하는 것이다.
따라서 최소한 10개 이상의 샘플 데이터를 이용하여 적절히 판단하는 것이 좋다.

```SQL
mysql> USE employees;

-- // employees 테이블과 동일한 구조로, 테이블 압축을 사용하는 예제 테이블을 생성
mysql> CREATE TABLE employees_comp4k (
        emp_no int NOT NULL,
        birth_date date NOT NULL,
        first_name varchar(14) NOT NULL,
        last_name varchar(16) NOT NULL,
        gender enum('M', 'F') NOT NULL,
        hire_date date NOT NULL,
        PRIMARY KEY (emp_no),
        KEY ix_firstname (first_name),
        KEY ix_hiredate (hire_date)
      ) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

-- // 테스트를 실행하기 전에 innodb_cmp_per_index_enabled  시스템 변수를 ON으로 변경해야
-- // 인덱스별로 압축 실행 횟수와 성공 횟수가 기록된다.
mysql> SET GLOBAL innodb_cmp_per_index_enabled=ON;

-- // employees 테이블의 데이터를 그대로 압축 테스트 테이블로 저장
mysql> INSERT INTO employees_comp4k SELECT * FROM employees;

-- // 인덱스별로 압축 횟수와 성공 횟수, 압축 실패율을 조회
mysql> SELECT
        table_name, index_name, compress_ops, compress_ops_ok,
        (compress_ops-compress_ops_ok) / compress_ops * 100 as compression_failure_pct
      FROM information_schema.INNODB_CMP_PER_INDEX;
+------------------+--------------+--------------+-----------------+-------------------------+
| table_name       | index_name   | compress_ops | compress_ops_ok | compression_failure_pct |
+------------------+--------------+--------------+-----------------+-------------------------+
| employees_comp4k | PRIMARY      | 18635        | 13478           | 27.6737                 |
| employees_comp4k | ix_firstname | 8320         | 7653            | 8.0168                  |
| employees_comp4k | ix_hiredate  | 7756         | 6721            | 13.4561                 |
+------------------+--------------+--------------+-----------------+-------------------------+
```

위의 결과를 해석해보면 PRIMARY 키는 전체 18635번 압축을 실행했는데 그중에서 13478번 성공했다.
즉 (18635-13478)번 압축했는데 압축의 결과가 4KB를 초과해서 데이터 페이지를 스플릿해서 다시 압축을 실행했다는 의미이다.
일반적으로 압축 실패율은 3~5% 미만으로 유지할 수 있게 KEY_BLOCK_SIZE를 선택하는 것이 좋다.

한편 주의해야할 점은, 무조건 압축 실패율이 높다고 해서 압축을 사용하지 말아야 한다는 것을 의미하지는 않는다는 것이다.
예를 들어 INSERT만 되는 경우에는 전체적으로 데이터 파일의 크기가 큰 폭으로 줄기만 한다면 큰 손해가 아니다.
한편 반대로 압축 실패율이 그다지 높지 않은 경우라도 테이블의 데이터가 매우 빈번하게 조회된다면 압축을 고려히자 않는 것도 좋다.
압축은 생각보다 많은 CPU 자원을 소모한다.

### 압축된 페이지의 버퍼 풀 적재 및 사용

InnoDB 스토리지 엔진은 압축된 테이블의 데이터 페이지를 버퍼 풀에 적재하면 압축된 상태와 압축이 해제된 상태 2개 버전을 관리한다.
그래서 InnoDB 스토리지 인젠은 디스크에서 읽은 상태 그대로의 데이터 페이지 목록을 관리하는 LRU 리스트와 압축된 페이지들의
압축 해제 버전인 Unzip_LRU 리스트를 별도로 관리하게 된다.

MySQL 서버에서는 압축된 테이블과 압축되지 않은 테이블이 공존하므로 결국 LRU 리스트는 다음과 같이 압축된 페이지와 압축되지
않은 페이지를 모두 가질 수 있다.

- 압축이 적용되지 않은 테이블의 데이터 페이지
- 압축이 적용된 테이블의 압축된 데이터 페이지

Unzip_LRU 리스트는 압축이 적용되지 않은 테이블의 데이터 페이지는 가지지 않으며, 압축이 적용된 테이블에서 읽은 데이터 페이지만 관리한다.
이러한 방식은 다음과 같은 단점을 가진다.

- 버퍼 풀의 공간을 이중으로 사용함으로써 메모리가 낭비되는 효과
- 압축된 페이지에서 데이터를 읽거나 변경하기 위해서는 압축을 해제해야 한다.

이러한 두 가지 단점을 보환하기 위해서 Unzip_LRU 리스트를 별도로 관리하고 있다가 MySQL 서버로 유입되는 요청 패턴에 따라서 적절히
다음과 같은 처리를 수행한다.

- InnoDB 버퍼 풀의 공간이 필요한 경우에는 LRU 리스트에서 원본 데이터 페이지(압축된 형태)는 유지하고, Unzip_LRU 리스트에서
압축 해제된 버전은 제거해서 버퍼 풀의 공간을 확보한다.
- 압축된 데이터 페이지가 자주 사용되는 경우에는 Unzip_LRU 리스트에 압축 해제된 페이지를 계속 유지하면서 압축 및 압축 해제 작업을
최소화 한다.
- 압축된 데이터 페이지가 사용되지 않아서 LRU 리스트에서 제거되는 경우에는 Unzip_LRU 리스트에서도 함께 제거된다.

InnoDB 스토리지 엔진은 버퍼 풀에서 압축 해제된 버전의 데이터 페이지를 적절한 수준으로 유지하기 위해 다음과 같은 어댑티브 알고리즘을
사용한다.

- CPU 사용량이 높은 서버에서는 가능하면 압축과 압축 해제를 피하기 위해 Unzip_LRU의 비율을 높여서 유지한다
- Desk IO 사용량이 높은 서버에서는 가능하면 Unzip_LRU 리스트의 비율을 낮춰서 InnoDB 버퍼 풀의 공간을 더 확보하도록 작동한다.

### 테이블 압축 관련 설정

테이블 압축ㅇ과 관련된 시스템 변수

- innodb_cmp_per_index_enabled :
테이블 압축이 사용된 테이블의 모든 인덱스별로 압축 성공 및 압축 실행 횟수를 수집하도록 설정
- innodb_compression_level :
압축률을 설정 0~9의 값을 사용할 수 있다.
- innodb_compression_failure_threshold_pct, innodb_compression_pad_pct_max
테이블 단위로 압축 실패율이 innodb_compression_failure_threshold_pct 값보다 커지면 압축을 실행하기 전
원본 데이터 페이지의 끝에 의도적으로 일정 크기의 빈 공간(padding)을 추가한다. 추가 가능한 패딩 공간의
최대 크기는 innodb_compression_pad_pct_max 시스템 설정값 이상을 넘을 수 없다.
- innodb_log_compressed_pages :
MySQL 서버가 비정상적으로 종료됐다가 다시 시작되는 경우 압축 알고리즘의 버전 차이가 있더라도 복구 과정이
실패하지 않도록 InnoDB 스토리지 엔진은 압축된 데이터 페이지를 그대로 리두 로그에 기록한다. 이는 압축 알고리즘을
업그레이드할 때 도움이 되지만, 데이터 페이지를 통째로 리두로그에 저장하는 것은 리두 로그의 증가량에 상당한 영향을
미칠 수도 있다. 압축을 적용한 후 리두 로그 용량이 매우 빠르게 증가한다거나 버퍼 풀로부터 더티 페이지가 한꺼번에
많이 기록되는 패턴으로 바뀌었다면 innodb_log_compressed_pages 시스템 변수를 OFF로 설정한 후 모니터링해보는 것이 좋다.
innodb_log_compressed_pages 시스템 변수의 기본값은 ON인데, 가능하면 기본값인 ON 상태를 유지하자.
