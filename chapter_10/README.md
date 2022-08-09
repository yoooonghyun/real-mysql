# 실행 계획

MySQL 서버의 실행 계획에 가장 큰 영향을 미치는 통계 정보에 대해 간략히 살펴보고,
MySQL 서버가 보여주는 실행 계획을 읽는 순서와 실행 계획에 출력되는 키워드,
그리고 알고리즘에 대해 살펴본다.

## 통계 정보

MySQL 8.0 버전부터 인덱스되지 않은 칼럼들에 대해서도 데이터 분포도를 수집해서 저장하는 히스토그램 정보됐다.

### 테이블 및 인덱스 통계 정보

비용 기반 최적화에서 가장 중요한 것은 통계 정보다.
통계 정보가 정확하지 않다면 전혀 엉뚱한 방향으로 쿼리를 실행할 수 있기 때문이다.
MySQL 또한 다른 DBMS와 같이 비용 기반의 최적화를 사용하지만,
다른 DBMS보다 통계 정보의 정확도가 높지 않고 통계 정보의 휘발성이 강했다.
그래서 MySQL 서버에서는 쿼리의 실행 계획을 수립할 때 실제 테이블의 데이터를 일부 분석해서 통계 정보를 보완해서 사용했다.
이러한 이유로 MySQL 5.6 버전부터는 통계 정보의 정확성을 높일 수 있는 방법이 제공되기 시작했지만 아직도 많은 사용자가 기존방식을 그대로
사용한다. 여기서는 MySQL 8.0 버전에서 통계 정보 관리가 어떻게 개선됐는지도 함께 살펴보겠다.

#### MySQL 서버의 통계 정보

MySQL 5.6 버전부터는 InnoDB 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적으로 관리할 수 있게 개선됐다.
MySQL 5.5 버전까지는 각 테이블의 통계 정보가 메모리에만 관리되고,
SHOW INDEX 명령으로만 테이블의 인덱스 칼럼의 분포도를 볼 수 있었다.
이처럼 통계 정보가 메모리에 관리될 경우 MySQL 서버가 재시작되면 지금까지 수집된 통계 정보가 모두 사라진다.
MySQL 5.6 버전부터는 각 테이블의 통계 정보를 mysql 데이터베이스의 innodb_index_stats 테이블과 innodb_table_stats
테이블로 관리할 수 있게 개선됐다.
이렇게 통계 정보를 테이블로 관리함으로써 MySQL 서버가 재시작돼도 기존의 통계 정보를 유지할 수 있게 됐다.

```SQL
mysql> SHOW TABLES LIKE '%_stats';
+---------------------------+
| Tables_in_mysql (%_stats) |
+---------------------------+
| innodb_index_stats        |
| innodb_table_stats        |
+---------------------------+
```

MySQL 5.6에서 테이블을 생성할 때는 STATS_PERSISTENT 옵션을 설정할 수 있는데, 이 설정값에 따라 테이블 단위로 영구적인
통계 정보를 보관할지 말지를 결정할 수 있다.

```SQL
mysql> CREATE TABLE tab_test (fd1 INT< fd2 VARCHAR(20), PRIMARY KEY(fd1))
       ENGINE=InnoDB
       STATS_PERSISTENT={ DEFAULT | 0 | 1 }
```

- STATS_PERSISTENT=0: 테이블의 통계 정보를 MySQL 5.5 이전의 방식대로 관리하며, mysql 데이터베이스의 innodb_index_stats와 
innodb_table_stats 테이블에 저장하지 않음

- STATS_PERSISTENT=1: 테이블의 통계 정보를 mysql 데이터베이스의 innodb_index_stats와 innodb_table_stats 테이블에 저장함

- STATS_PERSISTENT=DEFAULT: 테이블을 생성할 때 별도로 STATS_PERSISTENT 옵션을 설정하지 않은 것과 동일하며, 테이블의 통계를
영구적으로 관리할지 말지를 innodb_stats_persistent 시스템 변수의 값으로 결정한다.

innodb_stats_persistent 시스템 설정 변수는 기본적으로 ON으로 설정돼 있다.

STATS_PERSISTENT의 값은 ALTER TABLE 명령을 통해 변경 가능하다.

select 문을 통해 innodb_index_stats를 조회할 때 통계 정보의 각 칼럼은 다음과 같은 정보를 저장하고 있다.

- innodb_index_stats.stat_name='n_diff_pfx%': 인덱스가 가진 유니크한 값의 개수
- innodb_index_stats.stat_name='n_leaf_pages': 인덱스의 리프 노드 페이지 개수
- innodb_index_stats.stat_name='size': 인덱스 트리 전체 페이지 개수
- innodb_index_stats.n_rows:  테이블 전체 레코드 건수
- innodb_index_stats.clustered_index_size: 프라이머리 키의 크기(InnoDB 페이지 개수)
- innodb_index_stats.sum_of_other_index_sizes: 프라이머리 키를 제외한 인덱스의 크기(InnoDB 페이지 개수)

MySQL 5.5 버전까지 MySQL 서버 재시작 이외에 다음과 같은 이벤트가 발생하면 자동으로 통계 정보가 갱신됐다.

- 테이블이 새로 오픈되는 경우
- 테이블의 레코드가 대량으로 변경되는 경우
- ANALYZE TABLE 명령이 실행되는 경우
- SHOW TABLE STATUS 명령이나 SHOW INDEX FROM 명령이 실행되는 경우
- InnoDB 모니터가 활성화되는 경우
- innodb_stats_on_metadata 시스템 설정이 ON인 상태에서 SHOW TABLE STATUS 명령이 실행되는 경우

영구적인 통계 정보가 도입되면서 이렇게 의도하지 않은 통계 정보 변경을 막을 수 있게 됐다.
또한 innodb_stats_auto_recalc 시스템 설정 변수의 값을 OFF로 설정해서 통계 정보가 자동으로 갱신되는 것을 막을 수 있다.
통계 정보를 자동으로 수집할지 여부도 테이블을 생성할 때 STATS_AUTO_RECALC 옵션을 이용해 테이블 단위로 조정할 수 있다.

- STATS_AUTO_RECALC=1: 테이블의 통계 정보를 MySQL 5.5 이전의 방식대로 자동 수집한다.
- STATS_AUTO_RECALC=0: 테이블의 통계 정보는 ANALYZE TABLE 명령을 실행할 때만 수집된다.
- STATS_AUTO_RECALC=DEFAULT: 테이블을 생성할 때 별도로 STATS_AUTO_RECALC 옵션을 설정하지 않은 것과 동일하며,
테이블의 통계 정보 수집을 innodb_stats_auto_recalc 시스템 설정 변수의 값으로 결정한다.

MySQL 5.5 버전에서는 테이블의 통계 정보를 수집할 때 몇 개의 InnoDB 테이블 블록을 샘플링할지 결정하는 옵션으로 
innodb_stats_sample_pages 시스템 설정 변수가 제공되는데, MySQL 5.6 버전부터는 이 옵션은 없어졌다. 대신 이 시스템 변수가
innodb_stats_transient_sample_pages와 innodb_stats_persistent_sample_pages 시스템 변수 2개로 분리됐다.

- innodb_stats_transient_sample_pages
이 시스템 변수의 기본값은 8인데, 이는 자동으로 통계 정보 수집이 실행될 때 8개 페이지만 임의로 샘플링해서 분석하고 그 결과를
통계 정보로 활용함을 의미한다.

- innodb_stats_persistent_sample_pages
기본값은 20이며, ANALYZE TABLE 명령이 실행되면 임의로 20개 페이지만 샘플링해서 분석하고 그 결과를 영구적인 통계 정보 테이블에
저장하고 활용함을 의미한다.

### 히스토그램

MySQL 5.7 버전까지는 실행 계획을 수립할 때 실제 인덱스의 일부 페이지를 랜덤으로 가져와 참조.
8.0 버전으로 업그레이드 되면서 MySQL서버도 드디어 칼럼의 데이터 분포도를 참조할 수 있는 히스토그램 정보를 활용할 수 있게 됨.

#### 히스토그램 정보 수집 및 삭제

MySQL 8.0 버전에서 히스토그램 정보는 칼럼 단위로 관리되는데, 이는 자동으로 수집되지 않고 ANALYZE TABLE ... UPDATE HISTOGRAM 명령을
실행해 수동으로 수집 및 관리된다. 수집된 히스토그램 정보는 시스템 딕셔너리에 함께 저장되고, MySQL 서버가 시작될 때 딕셔너리의 히스토그램 
정보를 information_schema 데이터베이스의 column_statistics 테이블로 로드한다. 그래서 실제 히스토그램 정보를 조회하려면 column_statistics 테이블을
SELECT해서 참조할 수 있다.

MySQL 8.0 버전에서는 다음과 같이 2종류의 히스토그램 타입이 지원된다.

- Singleton(싱글톤 히스토그램): 칼럼값 개별로 레코드 건수를 관리하는 히스토그램으로, Value-Based 히스토그램 또는 도수 분포라고도 불린다.
- Equi-Height(높이 균형 히스토그램): 칼럼값의 범위를 균등한 개수로 구분해서 관리하는 히스토그램으로, Height-Balanced 히스토그램이라고도 불린다.

히스토그램은 버킷이라는 단위로 구분되어 레코드 건수나 칼럼값의 범위가 관리되는데, 싱글톤 히스토그램은 칼럼이 가지는 값별로 버킷이 할당되며 높이 균형
히스토그램에서는 개수가 균등한 칼럼값의 범위별로 하나의 버킷이 할당된다. 싱글톤 히스토그램은 각 버킷이 칼럼의 값과 발생 빈도의 비율의 2개의 값을 가진다.
반면 높이 균형 히스토그램은 각 버킷이 범위 시작 값과 마지막 값, 그리고 발생빈도율과 각 버킷에 포함된 유니크한 값의 개수 등 4개의 값을 가진다.


EXAMPLE
```JSON
Singleton: {"buckets": [
  [1, 0.5998529],
  [2, 1.0]
],
...
}

Equi-height: {"buckets": [
["1985-02-01", "1985-02-28", 0.009838277, 28],
["1985-03-01", "1985-03-28", 0.020159909, 28],
["1985-03-29", "1985-04-26", 0.030159305, 29],
...
["1998-08-07", "2000-01-06", 1.0, 233],
],
...
}
```

information_schema.column_statistics 테이블의 HISTOGRAM 칼럼이 가진 나머지 필드들은 다음과 같은 의미를 가지고 있다.

- sampling-rate: 히스토그램 정보를 수집하기 위해 스캔한 페이지의 비율을 저장한다.
- histogram-type: 히스토그램의 종류를 저장한다.
- number-of-buckets-specified: 히스토그램을 생성할 때 설정했던 버킷의 개수를 저장한다. 히스토그램을 생성할 때 별도로 
버킷의 개수를 지정하지 않았다면 기본으로 100개의 버킷이 사용된다. 

생성된 히스토그램은 다음과 같이 삭제할 수 있다.
```SQL
mysql> ANALYZE TABLE employees.employees
       DROP HISTOGRAM ON gender, hire_date;
```

히스토그램을 삭제하지 않고 MySQL 옵티마이저가 히스토그램을 사용하지 않게 하려면 다음과 같이 optimizer_switch 시스템 변수의 값을
변경하면 된다.
```SQL
mysql> SET GLOBAL optimizer_switch='condition_fanout_filter=off';
```
특정 커넥션 또는 특정 쿼리에서만 히스토그램을 사용하지 않고자 한다면 다음과 같은 방법을 사용하면 된다.
```SQL
-- // 현재 커넥션에서 실행되는 쿼리만 히스토그램을 사용하지 않게 설정
mysql> SET SESSION optimizer_switch='condition_fanout_filter=off';

-- // 현재 쿼리만 히스토그램을 사용하지 않게 설정
mysql> SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */ *
       FROM ...
```

#### 히스토그램의 용도

히스토그램 전에는 인덱스된 칼럼이 가지는 유니크한 값의 개수 정도만 통계 정보로 저장.
때문에 각 유니크한 값의 개수가 모두 동일한 분포를 가지고 있을 것으로 예상하여 실행계획을 설정함 => 히스토그램을 사용하면 분포에 맞춰서
실행 계획을 수정. 이러한 차이로 인해 쿼리의 성능은 10배 정도 차이를 보일 수 있으며, InnoDB 버퍼 풀에 데이터가 존재하지 않아서 디스크에서 
데이터를 읽어야 하는 경우라면 몇 배의 차이가 발생할 수도 있다.

#### 히스토그램과 인덱스

MySQL 서버에서는 쿼리의 실행 계획을 수립할 때 사용 가능한 인덱스들로부터 조건절에 일치하는 레코드 건수를 대략 파악하고 최종적으로 가장 나은 
실행 계획을 선택한다. 이때 조건절에 일치하는 레코드 건수를 예측하기 위해 옵티마이저는 실제 인덱스의 B-Tree 샘플링해서 살펴본다. 이 작업을 메뉴얼에서는 
"인덱스 다이브(Index Dive)"라고 표현한다.

MySQL 8.0 버전에서 히스토그램은 주로 인덱스되지 않은 칼럼에 대한 데이터 분포도를 참조하는 용도로 사용된다.
하지만 인덱스 다이브 작업은 어느 정도의 비용이 필요하며, 때로는 실행 계획 수립만으로도 상당한 인덱스 다이브를 실행하고 비용도 그만큼 커진다.
아마 조만칸 실제 인덱스 다이브를 실행하기보다는 히스토그램을 활용하는 최적화 기능도 MySQL 추가되지 않을까 생각된다.

### 코스트 모델(Cost Model)

MySQL 서버가 쿼리를 처리하려면 다음과 같은 다양한 작업을 필요로 한다.

- 디스크로부터 데이터 페이지 읽기
- 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
- 인덱스 키 비교
- 레코드 평가
- 메모리 임시 테이블 작업
- 디스크 임시 테이블 작업

MySQL 서버는 사용자의 쿼리에 대해 이러한 다양한 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과를 바탕으로 최적의 실행 계획을 찾는다.
이렇게 전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용을 코스트 모델(Cost Model)이라고 한다.
MySQL 5.7 이전 버전까지는 이런 작업들의 비용을 MySQL 서버 소스 코드에 상수화해서 사용했다.
이후 버전에서는 DBMS 관리자가 이를 조정할 수 있게 개선됐다.
MySQL 8.0 버전으로 업그레이드 되면서 칼럼의 데이터 분포를 위한 히스토그램과 각 인덱스별 메모리에 적재된 페이지 비율이 관리되고 옵티마이저의 실행 계획
수립에 사용되기 시작했다.

MySQL 8.0 서버의 코스트 모델은 다음 2개 테이블에 저장돼 있는 설정값을 사용하는데, 두 테이블 모두 mysql DB에 존재한다.
- server_cost: 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
- engine_cost: 레코드를 가진 데이터 페이지를 가져오는 데 필요한 비용 관리

server_cost 테이블과 engine_cost 테이블은 공통으로 다음 5개 의 칼럼을 가지고 있다.
- cost_name: 코스트 모델의 각 단위 작업
- default_value: 각 단위 작업의 비용
- cost_value: DBMS 관리자가 설정한 값
- last_updated: 단위 작업의 비용이 변경된 시점
- comment: 비용에 대한 추가 설명

engine_cost 테이블은 위의 5개 칼럼에 추가로 다음 2개 칼럼을 더 가지고 있다.
- engine_name: 비용이 적용된 스토리지 엔진
- device_type: 디스크 타입

engine_name 칼럼은 스토리지 엔진별로 각 단위 작업의 비용을 설정할 수 있는데, 기본값은 "default"다. 여기서 "default"는 특정 스토리지 엔진의 비용이 설정되지 않았다면
해당 스토리지 엔진의 비용으로 이 값을 적용한다는 의미다.

MySQL 8.0 버전의 코스트 모델에서 지원하는 단위 작업은 다음과 같이 8개다.

|             | cost_name                    | default_value | 설명 
|-------------|------------------------------|---------------|----------------------------------|
| engine_cost | io_block_read_cost           | 1.00          | 디스크 데이터 페이지 읽기        |
| engine_cost | memory_block_read_cost       | 0.25          | 메모리 데이터 페이지 읽기        |
| server_cost | disk_temptable_create_cost   | 20.00         | 디스크 임시 테이블 생성          |
| server_cost | dist_temptable_row_cost      | 0.50          | 디스크 임시 테이블의 레코드 읽기 |
| server_cost | key_compare_cost             | 0.05          | 인덱스 키 비교                   | 
| server_cost | memory_temptable_create_cost | 1.00          | 메모리 임시 테이블 생성          |
| server_cost | memory_temptable_row_cost    | 0.10          | 메모리 임시 테이블의 레코드 읽기 |
| server_cost | row_evaluate_cost            | 0.10          | 레코드 비교                      |

## 실행 계획 확인
MySQL 서버의 실행 계획은 DESC 또는 EXPLAIN 명령으로 확인할 수 있다.

### 실행 계획 출력 포맷

테이블 포맷 표시
```SQL
mysql> EXPLAIN  
       SELECT *
       FROM employees e
        INNER JOIN salaries s ON s.emp_no=e.emp_no
       WHERE first_name='ABC';
+----+-------------+-------+------------+------+-----------------------+--------------+---------+--------------------+-----+----------+-------+
| id | select_type | table | partitions | type | possible_keys         | key          | key_len | ref                |rows | filtered | Extra |
+----+-------------+-------+------------+------+-----------------------+--------------+---------+--------------------+-----+----------+-------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY, ix_firstname | ix_firstname | 58      | const              |   1 |   100.00 | NULL  |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY               | PRIMARY      | 4       | employees.e.emp_no |   1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+-----------------------+--------------+---------+--------------------+-----+----------+-------+
```

트리 포맷 표시
```SQL
mysql> EXPLAIN FORMAT=TREE
       SELECT *
       FROM emplyees e
          INNER JOIN salaries s ON s.emp_no=e.emp_no
       WHERE first_name='ABC'\G
****************************** 1. row *************************************
EXPLAIN: -> Nested loop inner join (cost=2.40 rows=10)
    -> Index lookup on e using ix_firstname (first_name='ABC') (cost=0.35 rows=1)
    -> Index lookup on s using PRIMARY (emp_no=e.emp_no) (cost=2.05 rows=10)
```

JSON 포맷 표시

```SQL
mysql> EXPLAIN FORMAT=JSON
       SELECT *
       FROM employees e
          INNER JOIN salaries s ON s.emp_no=e.emp_no
       WHERE first_name='ABC'\G
****************************** 1. row *************************************
EXPLAIN: {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "2.40"
    },
    "nested_loop": [
      {
        "table": {
          "table_name": "e",
          "access_type": "ref",
          "possible_keys": [
            "PRIMARY",
            "ix_firstname",
          ],
          "key": "ix_firstname",
          "used_key_parts": [
            "first_name",
          ],
...
```

### 쿼리의 실행 시간 확인

EXPLAIN ANALYZE를 통해서 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있다.

```SQL
mysql> EXPLAIN ANALYZE
       SELECT e.emp_no, avg(s.salary)
       FROM employees e
          INNER JOIN salaries s ON s.emp_no=e.emp_no
                     AND s.salary>50000
                     AND s.from_date<='1990-01-01'
                     AND s.to_date>'1990-01-01'
       WHERE e.first_name='Matt'
       GROUP BY e.hire_date \G

A) -> Table scan on <temporary> (actual time=0.001..0.004 rows=48 loops=1)
B)      -> Aggregate using temporary table (actual time=3.799..3.808 rows=48 loops=1)
C)          -> Nested loop inner join (cost=685.24 rows=135)
                            (actual time=0.367..3.602 rows=48 loops=1)
D)              -> Index lookup on e using ix_firstname (first_name='Matt') (cost=215.08 rows=233)
                            (actual time=0.348..1.046 rows=233 loops=1)
E)              -> Filter:  ((s.salary > 50000) and (s.from_date <= DATE'1990-01-01')
                                         and (s.to_date > DATE '1990-01-01')) (cost=0.98 rows=1)
                           (actual time=0.009..0.011 rows=0 loops=233)
F)                    -> Index lookup on s using PRIMARY (emp_no=e.emp_no) (cost=0.98 rows=10)
                           (actual time=0.007..0.009 rows=10 loops=233)
```

- 들여쓰기가 같은 레벨에서는 상단에 위치한 라인이 먼저 실행
- 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행

위 쿼리의 실행 계획은 다음의 실행 순서를 의미한다.

```SQL
1. D) Index lookup on e using ix_firstname
2. F) Index lookup on s using PRIMARY
3. E) Filter
4. C) Nested loop inner join
5. B) Aggregate using temporary table
6. A) Table scan on <temporary>
```

각 필드들의 의미는 다음과 같다 (F 기준)

- actual time=0.007..0.009: 레코드를 검색하는 데 걸린 시간(밀리초)를 의미한다. 첫 번째 숫자 값은 첫번째 레코드를 가져
오는 데 걸린 평균 시간(밀리초)을 의미한다. 두 번째 숫자 값은 마지막 레코드를 가져오는데 걸린 평균 시간(밀리초)을 의미한다.
- rows=10: employees 테이블에서 읽은 emp_no에 일치하는 salaries 테이블의 평균 레코드 건수를 의미한다.
- loops=233: employess 테이블에서 읽은 emp_no를 이용해 salaries 테이블의 레코드를 찾는 작업이 반복된 횟수를 의미한다.
결국 여기서는 employees 테이블에서 읽은 emp_no의 개수가 233개임을 의미한다.

EXPLAIN ANALYZE는 쿼리가 끝난 뒤에 수행되기 때문에 EXPLAIN 명령으로 먼저 실행 계획만 확인해서 어느 정도 튜닝한 후
EXPLAIN ANALYZE 명령을 실행하는 것이 좋다.

## 실행 계획 분석

EXPLAIN 명령을 통해 나타난 실행 계획에서 위에 있는 것은 쿼리의 바깐 부분이거나 먼저 접근한 테이블이고,
아래쪽에 출력된 결과일수록 쿼리의 안쪽 부분 또는 나중에 접근한 테이블에 해당한다.
이 장에서는 실행 계획에 표시되는 각 칼럼이 어떤 것을 의미하는지, 그리고 각 칼럼에 어떤 값들이 출력될 수 있는지 하나씩 살펴본다.

### id 칼럼

```SQL
mysql> SELECT ...
       FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
       WHERE tb1.id=tb2.id;
```

위의 쿼리 문장에 있는 각 SELECT를 다음과 같이 분리해서 생각해볼 수 있다.

```SQL
mysql> SELECT ... FROM tb_test1;
mysql> SELECT ... FROM tb1, tb_test2 tb2 WHERE tb1.id=tb2.id;
```

실행 계획에서 가장 왼쪽에 표시되는 id 칼럼은 단위 SELECT 쿼리별로 부여되는 식별자 값이다.
이 예제 쿼리의 경우 실행 계획에서 최소 2개의 id 값이 표시될 것이다.
다음 예제에서처럼 SELECT 문장은 하나인데, 여러 개의 테이블이 조인되는 경우에는 id 값이 증가하지 않고 같은 id 값이 부여된다.

```SQL
mysql> EXPLAIN
       SELECT e.emp_no, e.first_name, s.from_date, s.salary
       FROM employees e, salares s
       WHERE e.emp_no=s.emp_no LIMIT 10;

+----+-------------+-------+-------+--------------+--------------------+--------+-------------+
| id | select_type | table | type  | key          | ref                |rows    | Extra       |
+----+-------------+-------+-------+--------------+--------------------+--------+-------------+
|  1 | SIMPLE      | e     | index | ix_firstname | NULL               | 300252 | Using index |
|  1 | SIMPLE      | s     | ref   | PRIMARY      | employees.e.emp_no |     10 | NULL        |
+----+-------------+-------+-------+--------------+--------------------+--------+-------------+
```
```id 칼럼의 값이 테이블의 접근 순서를 의미하지는 않는다```

### select_type 칼럼

SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼이다. select_type 칼럼에 표시될 수 있는 값은 다음과 같다.

#### SIMPLE

UNION이나 서브 쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우 (쿼리에 조인이 포함된 경우에도 마찬가지이다.)
쿼리 문장이 아무리 복잡하더라도 실행 계획에서 select_type이 SIMPLE인 단위 쿼리는 하나만 존재한다.
일반적으로 제일 바깥 SELECT 쿼리의 select_type이 SIMPLE로 표시된다.

#### PRIMARY

UNION이나 서브쿼리를 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리는 select_type이 PRIMARY로 표시된다.
SIMPLE과 마찬가지로 select_type이 PRIMARY인 단위 SELECT 쿼리는 하나만 존재하며, 쿼리의 제일 바깥쪽에 있는 SELECT 단위 쿼리가 PRIMARY로 표시된다.

#### UNION

UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은
UNION으로 표시된다. UNION의 첫 번째 단위 SELECT는 select_type이 UNION이 아니라 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블이 select_type으로 표시된다.

```SQL
mysql> EXPLAIN
       SELECT * FROM(
          (SELECT emp_no FROM employees e1 LIMIT 10) UNION ALL
          (SELECT emp_no FROM employees e2 LIMIT 10) UNION ALL
          (SELECT emp_no FROM employees e3 LIMIT 10) ) tb;

+----+-------------+------------+-------+--------------+------+--------+-------------+
| id | select_type | table      | type  | key          | ref  |rows    | Extra       |
+----+-------------+------------+-------+--------------+------+--------+-------------+
|  1 | PRIMARY     | <derived2> | ALL   | NULL         | NULL |     30 | NULL        |
|  2 | DERIVED     | e1         | index | ix_hiredate  | NULL | 300252 | Using index |
|  3 | UNION       | e2         | index | ix_hiredate  | NULL | 300252 | Using index |
|  4 | UNION       | e3         | index | ix_hiredate  | NULL | 300252 | Using index |
+----+-------------+------------+-------+--------------+------+--------+-------------+
```

#### DEPENDENT UNION

DEPENDENT UNION 또한 UNION select_type과 같이 UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시된다.
그리고 여기서 DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.

```SQL
mysql> EXPLAIN
       SELECT *
       FROM employees e1 WHERE e1.emp_no IN (
          SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
          UNION
          SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'
        );

+----+--------------------+------------+--------+---------+------+--------+-----------------+
| id | select_type        | table      | type   | key     | ref  |rows    | Extra           |
+----+--------------------+------------+--------+---------+------+--------+-----------------+
|  1 | PRIMARY            | e1         | ALL    | NULL    | NULL | 300252 | Using where     |
|  2 | DEPENDENT SUBQUERY | e2         | eq_ref | PRIMARY | func |      1 | Using where     |
|  3 | DEPENDENT UNION    | e3         | eq_ref | PRIMARY | func |      1 | Using where     |
|NULL| UNION RESULT       | <union2,3> | index  | NULL    | NULL |   NULL | Using temporary |
+----+--------------------+------------+--------+---------+------+--------+-----------------+
```
#### UNION RESULT

UNION 결과를 담아두는 테이블을 의미한다. MySQL 8.0 버전부터는 UNION ALL의 경우 임시 테에블을 사용하지 않도록 기능이 개선됐지만
UNION은 여전히 임시 테이블에 결과를 버퍼링한다. 실행 계획상에서 이 임시 테이블을 가리키는 라인의 select_type이 UNION RESULT다.
별도의 id값은 부여되지 않는다.

```SQL
mysql> EXPLAIN
       SELECT *
       FROM employees e1 WHERE e1.emp_no IN (
          SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
          UNION
          SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'
        );

+----+--------------------+------------+--------+---------+------+--------+-----------------+
| id | select_type        | table      | type   | key     | ref  |rows    | Extra           |
+----+--------------------+------------+--------+---------+------+--------+-----------------+
|  1 | PRIMARY            | e1         | ALL    | NULL    | NULL | 300252 | Using where     |
|  2 | DEPENDENT SUBQUERY | e2         | eq_ref | PRIMARY | func |      1 | Using where     |
|  3 | DEPENDENT UNION    | e3         | eq_ref | PRIMARY | func |      1 | Using where     |
|NULL| UNION RESULT       | <union2,3> | index  | NULL    | NULL |   NULL | Using temporary |
+----+--------------------+------------+--------+---------+------+--------+-----------------+
```

#### SUBQUERY

일반적으로 서브쿼리라고 하면 여러 가지를 통틀어서 이야기할 때가 많은데, select_type의
SUBQUERY는 FROM 절 이외에서 사용되는 서브쿼리만을 의미한다.

```SQL
mysql> EXPLAIN
       SELECT e.first_Name,
              (SELECT COUNT(*)
               FROM dept_emp de, dept_manager dm
               WHERE dm.dept_no=de.dept_no) AS cnt
       FROM employees e WHERE e.emp_no=10001;

+----+--------------------+------------+-------+---------+-------+-----------------+
| id | select_type        | table      | type  | key     | rows  | Extra           |
+----+--------------------+------------+-------+---------+-------+-----------------+
|  1 | PRIMARY            | e1         | const | PRIMARY |     1 | UNULL           |
|  2 | SUBQUERY           | dm         | index | PRIMARY |    24 | Using index     |
|  3 | SUBQUERY           | de         | ref   | PRIMARY | 41392 | Using index     |
+----+--------------------+------------+-------+---------+-------+-----------------+
```

MySQL 서버의 실행 계획에서 FROM 절에 사용된 서브쿼리는 select_type이 DERIVED로 표시되고, 
그 밖의 위치에서 사용된 서브쿼리는 전부 SUBQUERY라고 표시된다.

서브쿼리는 사용하는 위치에 따라 각각 다른 이름을 지니고 있다.
- 중첩된 쿼리(Nested Query): SELECT되는 칼럼에 사용된 서브쿼리를 네스티드 쿼리라고 한다.
- 서브쿼리(Subquery): WHERE 절에 사용된 경우에는 일반적으로 그냥 서브쿼리라고 한다.
- 파생 테이블(Derived Table): FROM 절에 사용된 서브쿼리를 MySQL에서는 파생 테이블이라고 하며,
일반적으로 RDBMS에서는 인라인 뷰 또는 서브셀렉트라고 부른다.


또한 서브쿼리가 반환하는 값의 특성에 따라 다음과 같이 구분하기도 한다.
- 스칼라 서브쿼리(Scalar Subquery): 하나의 값만(칼럼이 단 하나인 레코드 1건만) 반환하는 쿼리
- 로우 서브쿼리(Row Subquery): 칼럼의 개수와 관계없이 하나의 레코드만 반환하는 쿼리

#### DEPENDENT SUBQUERY

서브쿼리가 바깥쪽 SELECT 쿼리에서 정의된 칼럼을 사용하는 경우, select_type에 DEPENDENT SUBQUERY라고
표시된다.

```SQL
mysql> EXPLAIN
       SELECT e.first_name,
              (SELECT COUNT(*)
               FROM dept_emp de, dept_manager dm
               WHERE dm.dept_no=de.dept_no AND de.emp_no=e.emp_no) AS Cnt
       FROM employees e
       WHERE e.first_name='Matt';
```
이럴 때는 안쪽의 서브쿼리 결과가 바깥쪽 SELECT 쿼리의 칼럼에 의존적이기 때문에 DEPENDENT라는 키워드가 붙는다.
또한 DEPENDENT UNION과 같이 DEPENDENT SUBQUERY 또한 외부 쿼리가 먼저 수행된 후 내부커리가 실행돼야 하므로 
일반 서브쿼리보다는 처리 속도가 느릴 때가 많다.

#### DERIVED

MySQL 5.5 버전까지는 서브쿼리가 FROM 절에 사용된 경우 항상 select_type이 DERIVED인 실행 계획을 만든다.
하지만 MySQL 5.6 버전부터는 옵티마이저 옵션에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화가 수행되기도 한다.
DERIVED는 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것을 의미한다.
select_type이 DERIVED인 경우에 생성되는 임시 테이블을 파생 테이블이라고도 한다.
MySQL 서버는 버전이 업그레이드되면서 조인 쿼리에 대한 최적화가 많이 좋아졌으므로 되도록이면 DERIVED 형태의 실행 계획을 조인으로
해결할 수 있게 쿼리를 바꿔주는 것이 좋다.

#### DEPENDENT DERIVED

MySQL 8.0 이전 버전에서는 FROM 절의 서브쿼리는 외부 칼럼을 사용할 수가 없었는데, MySQL 8.0 버전부터는 래터럴 조인 기능이 추가되면서
FROM 절의 서브쿼리에서도 외부 칼럼을 참조할 수 있게 됐다.
래터럴 조인의 경우에는 LATERAL 키워드를 사용해야 하며, LATERAL 키워드가 없는 서브쿼리에서 외부 칼럼을 참조하면 오류가 발생한다.
실행 계획에서 보다시피 select_type 칼럼의 DEPENDENT DERIVED 키워드는 해당 테이블이 래터럴 조인으로 사용된 것을 의미한다.

#### UNCACHEABLE SUBQUERY

조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 담아둔다.

- SUBQUERY는 바깥쪽의 영향을 받지 않으므로 처음 한 번만 실행해서 그 결과를 캐시하고 필요할 때 캐시된 결과를 이용한다.
- DEPENDENT SUBQUERY는 의존하는 바깥쪽 쿼리의 칼럼의 값 단위로 캐시해두고 사용한다.

select_type이 SUBQUERY인 경우와 UNCACHEABLE SUBQUERY는 이 캐시를 사용할 수 있느냐 없느냐의 차이가 있다.
서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능할 수가 있는데, 그럴 경우 select_type이 "UNCACHEABLE SUBQUERY"로 표시된다.
캐시를 사용하지 못하게 하는 요소로는 대표적으로 다음과 같은 것들이 있다.

- 사용자 변수가 서브쿼리에 사용된 경우
- NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
- UUID()나 RAND()와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우

```SQL
mysql> EXPLAIN
       SELECT *
       FROM employees e WHERE e.emp_no - (
          SELECT @status FROM dept_emp de WHERE de.dept_no='d005');

+----+----------------------+-------+-------+---------+--------+-----------------+
| id | select_type          | table | type  | key     | rows   | Extra           |
+----+----------------------+-------+-------+---------+--------+-----------------+
|  1 | PRIMARY              | e     | ALL   | NULL    | 300252 | Using where     |
|  2 | UNCACHEABLE SUBQUERY | de    | ref   | PRIMARY | 165571 | Using index     |
+----+----------------------+-------+-------+---------+--------+-----------------+
```

#### UNCACHEABLE

UNCACHEABLE과 UNION의 속성이 혼합된 select_type

#### MATERIALIZED

DERIVED와 비슷하게 쿼리의 내용을 임시 테이블로 생성한다.

### table 칼럼

MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다.
테이블의 이름에 별칭이 부여된 경우에는 별칭이 표시된다.
<derived N> 또는 <union M,N> 은 임시 테이블을 의미하며 "<>" 안에 항상 표시되는 숫자는 단위 SELECT 쿼리의 id 값을 지칭힌다.


### partitions 칼럼

옵티마이저가 쿼리 처리를 위해 필요한 파티션들의 목록만 모아서 실행 계획의 partitions 칼럼에 표시해준다.

### type 칼럼

MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타낸다.
다음과 같은 값들이 표시된다.
- system
- const
- eq_ref
- ref
- fulltext
- ref_or_null
- unique_subquery
- index_subquery
- range
- index_merge
- index
- ALL

ALL을 제외한 나머지는 모두 인덱스를 사용하는 접근 방법이다. ALL은 풀 테이블 스캔 접근 방법을 의미한다.
또한 index_merge를 제외한 나머지 접근 방법은 하나의 인덱스만 사용한다.

#### system

레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system이라고 한다.
InnoDB 이외에 MyISAM이나 MEMORY 테이블에서만 사용되는 접근 방법이다.

#### const

테이블의 레코드 건수와 관계 없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며,
반드시 1건을 반환하는 쿼리의 처리 방식을 const 라고 한다. 다른 DBMS에서는 이를 유니크 인덱스 스캔이라고도 표현한다.
const 경우 옵티마이저가 쿼리를 최적화하는 단계에서 쿼리를 먼저 실행해서 통째로 상수화한다.

#### eq_ref

eq_ref 접근 방법은 여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시된다.
조인에서 처음 읽은 테이블의 컬럼값을, 그다음 읽어야 할 테이블의 프라이머리 키나 유니크 키 칼럼의 검색 조건에 사용할 때를 가리켜
eq_ref라고 한다. 이때 두 번째 이후에 읽는 테이블의 type 칼럼에 eq_ref가 표시된다.
또한 두 번째 이후에 읽히는 테이블을 유니크 키로 검색할 때 그 유니크 인덱스는 NOT NULL이어야 하며,
다중 칼럼으로 만들어진 프라이머리 키나 유니크 인덱스라면 인덱스의 모든 칼럼이 비교 조건에 사용돼야만 eq_ref 접근 방법이 사용될 수 있다.
즉 조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 사용할 수 있는 접근 방법이다.

#### ref

ref 접근 방법은 eq_ref와는 달리 조인의 순서와 관계없이 사용되며, 또한 프라이머리 키나 유니크 키 등의 제약 조건도 없다.
인덱스의 종류와 관계없이 동등 조건으로 검색할 때는 ref 접근 방법이 사용된다.
ref 타입은 반환되는 레코드가 반드시 1건이라는 보장이 없으므로 const 나 eq_ref보다는 빠르지 않다.
하지만 동등한 조건으로만 비교되므로 매우 빠른 레코드 조회 방법의 하나다.

#### fulltext

fulltext 접근 방법은 MySQL 서버의 전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법을 의미한다.
전문 검색 인덱스는 통계 정보가 관리되지 않으며, 전문 검색 인덱스를 사용하려면 전혀 다른 SQL 문법을 사용해야한다.
MySQL 서버에서 전문 검색 조건은 우선순위가 상당히 높다. 쿼리에서 전문 인덱스를 사용하는 조건과 그 이외의 일반 인덱스를
사용하는 조건을 함께 사용하면 일반 인덱스의 접근 방법이 const나 eq_ref, ref가 아니면 일반적으로 MySQL은 전문 인덱스를 사용하는 조건을 선택
해서 처리한다. 전문 검색은 "MATCH (...) AGAINST (...)" 구문을 사용해서 실행하는데, 이때 반드시 해당 테이블에 전문 검색용 인덱스가 준비돼 있어
야만 한다.

```SQL
mysql> CREATE TABLE emplyee_name (
         emp_no int NOT NULL,
         first_name varchar(14) NOT NULL,
         last_name varchar(16) NOT NULL,
         PRIMARY KEY (emp_no),
         FULLTEXT KEY fx_name (first_name,last_name) WITH PARSER ngram
      ) ENGINE=InnoDB;

mysql> EXPLAIN
       SELECT *
       FROM emplyee_name
       WHERE emp_no=10001
       AND emp_no BETWEEN 10001 AND 10005
       AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);
```

위의 경우 const 조건인 WHERE emp_no=10001 을 사용한다.
두번째 조건과 세번째 조건이 같이 있는 경우에는 FULLTEXT조건을 사용하여 검색한다.
하지만 지금까지의 경험으로 보면 전문 검색 인덱스를 이용하는 fulltext보다 일반 인덱스를 이용하는 range 접근 방법이 더 빨리 처리되는 경우가 더 많았따.
따라서 전문 검색 쿼리를 사용할 때는 조건별로 성능을 확인해 보는 편이 좋다.

#### ref_or_null

이 접근 방법은 ref 접근 방법과 같은데, NULL 비교가 추가된 형태다.
접근 방법의 이름 그대로 ref 방식 또는 NULL 비교 접근 방법을 의미한다.
실제 업무에서 많이 활용되지 않지만, 만약 사용된다면 나쁘지 않은 접근 방법 정도로 기억해 두면 충분하다.

#### unique_subquery

WHERE 조건절에서 사용될 수 있는 IN(subquery) 형태의 쿼리를 위한 접근 방법이다. unique_subquery의 의미 그대로 서브 쿼리에서
중복되지 않은 유니크한 값만 반환할 때 이 접근 방법을 사용한다.

#### index_subquery

IN 연산자의 특성상 IN(subquery) 또는 IN(상수 나열) 형태의 조건은 괄호 안에 있는 값의 목록에서 중복된 값이
먼저 제거돼야 한다. 앞서 살펴본 unique_subquery 접근 방법은 IN(subquery) 조건의 subquery가 중복된 값을 만들어내
지 않는다는 보장이 있으므로 별도의 중복을 제거할 필요가 없었다. 한편 이 경우에는 서브 쿼리 결과의 중복된 값을 인덱스를
이용해서 제거할 수 있을 때 index_subquery 접근 방법이 사용된다.

#### range

range는 우리가 익히 알고 있는 인덱스 레인지 스캔 형태의 접근 방법이다.
range는 인덱스를 하나의 값이 아니라 범위로 검색하는 경우를 의미하는데, 주로 "<, >, IS NULL, BETWEEN, IN, LIKE" 등의 연산자를
이용해 인덱스를 검색할 때 사용된다. MySQL 서버가 가지오 있는 접근 방법 중에서 상당히 우선순위가 낮다.
얼마나 많은 레코드를 필요로 하느냐에 따라 차이는 있겠지만 range 접근 방법도 상당히 빠르며, 모든 쿼리가 이 접근
방법만 사용해도 최적의 성능이 보장된다고 볼 수 있다.

#### index_merge

지금까지 설명한 다른 접근 방법과는 달리 index_merge 접근 방법은 2개 이상의 인덱스를 이용해 각각의
검색 결과를 만들어낸 후, 그 결과를 병합해서 처리하는 방식이다. 하지만 index_merge 접근 방법이 사용되는 경우를 생각해보면
이름만큼 그렇게 효율적으로 작동하는 것은 아니다. index_merge 접근 방법에는 다음과 같은 특징이 있다.

- 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방법보다 효율성이 떨어진다.

- 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge가 적용되지 않는다.

- index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합, 또는 중복 제거와
같은 부가적인 작업이 더 필요하다.

MySQL 메뉴얼에서 index_merge 접근 방법의 우선순위는 ref_or_null 바로 다음에 있다. 하지만 실제로는 위 이유 때문에 책에서는 
range 다음의 우선 순위로 두었다.

#### index

index 접근 방법은 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미한다.
index 접근 방법은 다음 조건 가운데 1,2 조건을 충족하거나 1,3 조건을 충족하는 쿼리에서 사용되는 읽기 방식이다.

- range나 const, ref같은 접근 방법으로 인덱스를 사용하지 못하는 경우
- 인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우
- 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우 (즉, 별도의 정렬 작업을 피할 수 있는 경우)

ex)
```SQL
mysql> EXPLAIN
       SELECT * FROM departments ORDER BY dept_name DESC LIMIT 10;
```

#### ALL

풀 테이블 스캔을 의미하는 접근 방법이다. 가장 마지막에 선택하는 가장 비효율적인 방법이다.
다만 데이터 웨어하우스나 배치 프로그램처럼 대용량의 레코드를 처리하는 쿼리에서는 잘못 튜닝된 쿼리보다 더 나은
접근 방법이기도 하다.

일반적으로 index와 ALL 접근 방법은 작업 범위를 제한하는 조건이 아니므로 빠른 응답을 사용자에게 보내야 하는 웹 서비스 등과 같은
온라인 트랜잭션 처리 환경에는 적합하지 않다. 테이블이 매우 작지 않다면 실제로 테이블에 데이터를 어느 정도 저장한 상태에서 쿼리의
성능을 확인해 보고 적용하는 것이 좋다.

### possible_keys 칼럼

옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록일 뿐이다. 즉, 말그대로
"사용될 법했던 인덱스의 목록"이다. => 크게 도움되지 않는다

### key 칼럼

실행 계획에서 사용하는 인덱스를 의미한다. key 칼럼에 표시되는 값이 PRIMARY인 경우에는 프라이머리 키를 사용한다는 의미이며,
그 이외의 값은 모두 테이블이나 인덱스를 생성할 때 부여했던 고유 이름이다.

실행 계획의 type 칼럼이 index_merge가 아닌 경우에는 반드시 테이블 하나당 하나의 인덱스만 이용할 수 있다.
하지만 index_merge 실행 계획이 사용될 때는 2개 이상의 인덱스가 사용되는데, 이때는 key 칼럼에 여러 개의 인덱스가 ","로 구분되어
표시된다. 그리고 실행 계획의 type이 ALL일 때와 같이 인덱스를 전혀 사용하지 못하면 key 칼럼은 NULL로 표시된다.

### key_len 칼럼

실행 계획의 key_len 칼럼의 값은 쿼리를 처리하기 위해 다중 칼럼으로 구성된 인덱스에서 몇 개의 칼럼까지 사용됐는지 우리에게 알려준다.
정확하게는 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값이다.
그래서 다중 칼럼 인덱스뿐 아니라 단일 칼럼으로 만들어진 인덱스에서도 같은 지표를 제공한다. (+ NULL이 저장될 수 있는 칼럼을 사용하는
경우 + 1바이트가 추가된다.)

### ref 칼럼

접근 방법이 ref면 참조 조건으로 어떤 값이 제공됐는지 보여준다. 상수값을 지정했다면 ref 칼럼의 값은 const로 표시되고,
다른 테이블의 칼럼값이면 그 테이블명과 칼럼명이 표시된다. 콜레이션 변환 값이나 값 자체의 연산을 거쳐서 참조되면 func라고 표시된다.

### rows 칼럼

MySQL 옵티마이저는 각 조건에 대해 가능한 처리 방식을 나열하고, 각 처리 방식의 비용을 비교해 최종적으로 하나의 실행 계획을 수립한다.
이때 각 처리 방식이 얼마나 많은 레코드를 읽고 비교해야 하는지 예측해서 비용을 산정한다.
대상 테이블에 얼마나 많은 레코드가 포함돼 있는지 또는 각 인덱스 값의 분포도가 어떤지를 통계 정보를 기준으로 조사해서 예측한다.

MySQL 실행 계획의 rows 칼럼값은 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다.
이 값은 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이라서 정확하지는 않다.
또한 rows 칼럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미한다.
그래서 실행 계획의 rows 칼럼에 출력되는 값과 실제 쿼리 결과 반환된 레코드 건수는 일치하지 않는 경우가 많다.

### filtered 칼럼

옵티마이저는 각 테이블에서 일치하는 레코드 개수를 가능하면 정확히 파악해야 좀 더 효율적인 실행 계획을 수립할 수 있다.
실행 계획에서 rows 칼럼의 값은 인덱스를 사용하는 조건에만 일치하는 레코드 건수를 예측한 것이다.
하지만 대부분 쿼리에서 WHERE 절에 사용되는 조건이 모두 인덱스를 사용할 수 있는 것은 아니다.
특히 조인이 사용되는 경우에는 WHERE 절에서 인덱스를 사용할 수 있는 조건도 중요하지만 인덱스를 사용하지 못하는 조건에 일치하는 레코드 건수를
파악하는 것도 매우 중요하다. filtered 칼럼의 값은 필터링되어 버려지는 레코드의 비율이 아니라 필터링되고 남은 레코드의 비율을 의미한다.
이 값은 히스토그램을 통해 예측된다.

### Extra 칼럼

Extra 칼럼에는 주로 내부적인 처리 알고리즘에 대해 조금 더 깊이 있는 내용을 보여주는 경우가 많다.

#### const row not found

const 접근 방법으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 존재하지 않으면 Extra 칼럼에 이 내용이 표시된다.
Extra 칼럼에 이런 메시지가 표시되는 경우에는 테이블에 적절히 테스트용 데이터를 저장하고 다시 한번 쿼리의 실행 계획을 확인해 보는 것이 좋다.

#### Deleting all rows

WHERE 조건절이 없는 DELETE 문장의 실행 계획에서 자주 표시되며, 이 문구는 테이블의 모든 레코드를 삭제하는 핸들러 기능을 한번 호출함으로써
처리됐다는 것을 의미한다.
8.0 버전 이상에서는 InnoDB, MyISAM 모두 이 문구가 표시되지 않는다. => TRUNCATE TABLE 명령을 추천

#### Distinct

중복 없이 가져오는 경우 필요한 레코드만 읽은 것을 표현함

#### FirstMatch

함께 표시되는 테이블 명을 기준으로 첫 번째로 일치하는 한 건만 검색하는 것을 의미한다.

#### Full scan on NULL key

col1 IN (SELECT col2 FROM) 과 같은 쿼리에서 col1이 NULL이라면

- 서브쿼리가 1건이라도 결과 레코드를 가진다면 최종 비교 결과는 NULL
- 서브쿼리가 1건도 결과 레코드를 가지지 않는다면 최종 비교 결과는 FALSE

이 비교 과정에서 서브 쿼리에 사용된 테이블에 대해서 풀 테이블 스캔을 해야만 결과를 알아낼 수 있다.
이를 의미한다.

#### Impossible HAVING

쿼리에 사용된 HAVING 절의 조건을 만족하는 레코드가 없을 때 표시되는 키워드
=> 쿼리 내용을 다시 점검하는 것이 좋다

#### Impossible WHERE

WHERE 조건이 항상 FALSE가 될 수 없는 경우 표시

#### LooseScan

세미 조인 최적화 중에서 LooseScan 최적화 전략이 사용되는 경우

#### No matching min/max row

쿼리의 WHERE 조건절을 만족하는 레코드가 한 건도 없는 경우 중에서 MIN, MAX와 같은 집합 조건에 해당하는 코드가 한건도 없을
경우에는 이 값이 출력된다.

#### no matching row in const table

조인에 사용된 테이블에서 const 방법으로 접근할 때 일치하는 레코드가 없다면 해당 메시지를 출력한다.

#### No matching rows after partition pruning

해당 파티션에서 UPDATE하거나 DELETE할 대상 레코드가 없을 때 표시된다.

#### No tables used

FROM 절이 없는 쿼리 문장이나 "FROM DUAL" 형태의 쿼리 실행 계획에서는 해당 메시지가 출력된다.

#### Not exists

```SQL
mysql> EXPLAIN
       SELECT *
       FROM dept_emp de
          LET JOIN departments d ON de.dept_no=d.dept_no
       WHERE d.dept_no IS NULL;
```

아우터 조인을 이용해 안티-조인을 수행하는 쿼리에서 조인 조건에 일치하는 레코드가 여러 건 있다고 하더라도 딱 1건만 조회해보고 처리를 완료하는 최적화를 의미한다.

#### Plan isn't ready yet

아직 쿼리의 실행 계획을 수립하지 못한 상태에서 EXPLAIN FOR CONNECTION 명령이 실행된 것을 의미한다.

#### Range checked for each record(index map: N)

```SQL
mysql> EXPLAIN
       SELECT *
       FROM employees e1, emplyees e2
       WHERE e2.emp_no >= e1.emp_no;
```

해당과 같은 경우 레코드마다 인덱스 레인지 스캔을 체크한다 라는 해당 문구가 표시된다.
즉 e2를 e1을 무시하고 풀 스캔할지 혹은 테이블의 첫번째 인덱스를 사용할지를 레코드 단위로 결정하는 것이다.

#### Recursive

WITH 구문이 재귀를 사용하는 경우
```SQL
mysql> WITH RECURSIVE cte (n) AS
       (
          SELECT 1
          UNION ALL
          SELECT n + 1 FROM cte WHERE n < 5
       )
       SELECT * FROM cte;
```

#### Rematerialize

래터럴 조인에서 래터럴 조인되는 테이블은 선행 테이블의 레코드별로 서브쿼리를 실행해서 그 결과를 임시 테이블에 저장하는데, 이 과정을 
Rematerializing이라고 한다.

#### Select tables optimized away

MIN() 또는 MAX()만 SELECT 절에 사용되거나 GROUP BY로 MIN(), MAX()를 조회하는 쿼리가 인덱스를 오름차순 또는 내림차순으로 1건만 읽는 형태의
최적화가 적용된다면 이 문구가 표시된다. 또한 MyISAM테이블에 대해서는 GROUP BY 없이 COUNT(\*)만 SELECT 할 때도 이런 형태의 최적화가 적용된다.

#### Start temporary, End temporary

세미 조인 최적화 중에서 Duplicate Weed-out 최적화 전략이 사용되면 MySQL 옵티마이저는 Start temporary와 End temporary 문구를 표시하게 된다.
Duplicate Weed-out 최적화 전략은 불필요한 중복 건을 제거하기 위해서 내부 임시 테이블을 사용하는데,
이때 조인되어 내부 임시테이블에 저장되는 테이블을 식별할 수 있게 조인의 첫 번째 테이블에 "Start temporary" 문구를 보여주고 조인이 끝나는
부분에 "End temporary" 문구를 표시해준다.

#### unique row not found

두 개의 테이블이 각각 유니크 칼럼으로 아우터 조인을 수행하는 쿼리에서 아우터 테이블에 일치하는 레코드가 존재하지 않을 때
해당 문구가 표시된다.

#### Using filesort

ORDER BY를 처리하기 위해 인덱스를 사용하지 못할 때만 표시되는 코멘트로, 이는 조회된 레코드를 정렬용 메모리 버퍼에 복사해 퀵 소트 또는 힙 소트
알고리즘을 이용해 정렬을 수행하게 된다는 의미다.

#### Using index(커버링 인덱스)

데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을 때 표시된다.

#### Using index condition

MySQL 옵티마이저가 인덱스 컨디션 푸시 다운 최적화를 사용하면 해당 문구가 표시된다.

#### Using index for group-by

Group By를 사용할 때 인덱스를 사용하는 경우

##### 타이트 인덱스 스캔(인덱스 스캔)을 통한 GROUP BY 처리

AVG(), SUM(), COUNT()처럼 조회하려는 값이 모든 인덱스를 다 읽어야 할 때는 Using index for group-by 메시지가 출력되지 않는다.

##### 루스 인덱스 스캔을 통한 GROUP BY처리

단일 칼럼으로 구성된 인덱스에서는 그루핑 칼럼 말고는 아무것도 조회하지 않는 쿼리에서 루스 인덱스 스캔을 사용할 수 있다.
그리고 다중 칼럼으로 만들어진 인덱스에서는 GROUP BY 절이 인덱스를 사용할 수 있어야 함은 물론이고 MIN()이나 MAX()와 같이 조회
하는 값이 인덱스의 첫 번째 또는 마지막 레코드만 읽어도 되는 쿼리는 "루스 인덱스 스캔"이 사용될 수 있다.

#### Using index for skip scan

MySQL 옵티마이저가 인덱스 스킵 스캔 최적화를 사용하면 표시된다.

#### Using join buffer(Block Nested Loop), Using join buffer(Batched Key Access), Using join buffer(hash join)

join buffer와 사용된 알고리즘을 의미

#### Using MRR

Multi Range Read, MySQL 엔진이 여러 개의 키 값을 한 번에 스토리지 엔진으로 전달하고, 스토리지 엔진은 넘겨받은 키 값들을
정렬해서 최소한의 페이지 접근만으로 필요한 레코드를 읽을 수 있게 최적화 한다.

#### Using sort_union, Using union, Using intersect

쿼리가 index_merge 접근 방법으로 실행되는 경우에 두 인덱스로부터 읽은 결과를 어떻게 병합했는지 조금 더 상세하게 설명하기 위해
다음 3개 중에서 하나의 메시지를 선택적으로 출력한다.

- Using intersect: 각각의 인덱스를 사용할 수 있는 조건이 AND로 연결된 경우 각 처리 결과에서 교집합을 추출해내는 작업을 수행
했다는 의미다.

- Using union: 각 인덱스를 사용할 수 있는 조건이 OR로 연결된 경우 각 처리 결과에서 합집합을 추출해내는 작업을 수행했다는 의미다.

- Using sort_union: Using union과 같은 작업을 수행하지만 Using union으로 처리될 수 없는 경우(OR로 연결된 상대적으로 대량의 range
조건들) 이 방식으로 처리된다. Using sort_union과 Using union의 차이점은 Using sort_union은 프라이머리 키만 먼저 읽어서 정렬하고
병합합 이후 비로소 레코드를 읽어서 반환할 수 있다는 것이다.

#### Using temporary

임시테이블 생성 시 출력, 메모리에 생성됐는지 디스크에 생성됐는지는 알 수 없다.

#### Using where

MySQL 엔진 레이어에서 별도의 가공을 해서 필터링 작업을 처리한 경우에만 Extra 칼럼에 "Using where" 코멘트가 표시된다.

#### Zero limit

데이터 값이 아닌 쿼리 결과값의 메타데이터만 필요한 경우 출력

