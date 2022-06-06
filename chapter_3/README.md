# 3. 사용자 및 권한

- MySQL 8.0부터는 권한을 묶어서 관리하는 Role 도입 

## 3.1 사용자 식별

MySQL의 사용자 계정은 아이디와 IP를 확인

ID와 IP는 역따옴표 (`) 혹은 홑따옴표 (')를 사용하여 식별

#### ex. 3.1.1
- ID: svc_id
- IP: localhost
```
'svc_id'@'127.0.0.1'
```

모든 PC에서 접속 가능한 계정을 생성하고 싶다면 IP주소를 %로 등록

접속 시에는 권한의 범위가 좁은 것부터 확인
#### ex. 3.1.2
- ID는 동일, IP에 따라 다른 PW
- 192.168.0.10에서 접속 시 PW는 123
```
'svc_id'@'192.168.0.10' (PW: 123)
'svc_id'@'%' (PW: abc)
```

## 3.2 사용자 계정 관리

### 3.2.1 시스템 계정과 일반 계정

MySQL 8.0부터는 **SYSTEM_USER** 권한에따라 **System Account**와 **Regular Account**로 구분.

- System Account의 주요 권한
    - 계정 관리 (계정 생성 / 삭제, 권한 부여 / 제거)
    - 다른 세션 (Connection), 세션에서 실행 중인 쿼리를 강제 종료
    - Stored 프로그램 생성 시 DEFILNER를 타 사용자로 설정
    
- MySQL 서버의 내장 계정
    - 'root'@'localhost' 외의 3개의 내장 계정이 존재
    - **Account lock**이 걸려 있어, 의도적으로 풀지 않는 이상 보안 이슈 발생 X

| User             | host          |account_locked | 목적          |
| ---------------- | ------------- | ------------- | ------------- | 
| msysql.infoschema| localhost     |Y              |기본으로 내장된 sys 스키마의 객체들의 DEFINER로 사용
| mysql.session    | localhost     |Y              |MySQL 플러그인이 서버로 접근할 때 사용
| mysql.sys        | localhost     |Y              |information_schema에 정의된 뷰의 DEFINER로 사용

### 3.2.2 계정 생성
계정 생성은 ```CREAT USER```, 권한 부여는 ```GRANT```로 분리하여 사용

- 계정 생성 옵션
    - 계정 인증 방식과 비밀번호
    - 비밀번호 관련 옵션 (유효 기간 / 이력 개수 / 재사용 불가 기간)
    - 기본 역할 (Role)
    - SSL 옵션
    - 계정 잠금 여부
    
#### ex. 3.2.1
- 계정 생성 예시
```
mysql > CREATE USER 'user'@'%'
        IDENTIFIED WITH 'mysql_native_password' BY 'password'
        REQUIRE NONE
        PASSWORD EXPIRE INTERVAL 30 DAY
        ACCOUNT UNLOCK
        PASSWORD HISTORY DEFAULT
        PASSWORD REUSE INTERVAL DEFAULT
        PASSWORD REQUIRE CURRENT DEFAULT;
```

**Reference**: https://dev.mysql.com/doc/refman/8.0/en/create-user.html
### 3.2.2.1 IDENTIFIED WITH
사용자의 인증 방식과 비밀번호를 설정

MySQL 서버에서는 4가지 인증 방식을 플러그인 형태로 제공

MySQL 8.0의 기본 인증 방식은 Caching SHA-2 Pluggable Authentication

기본 인증 방식을 변경하고자 한다면 MySQL 명령어를 실행하거나, mu.cnf 설정 파일에 추가
#### ex. 3.2.2
```
mysql > SET GLOBAL default_authentication_plugin="mysql_native_password"
```

- Native Pluggable Authentication
    - MySQL 5.7까지 제공하던 방식
    - 비밀번호에 대한 해시 값 저장 (SHA-1)
- Caching SHA-2 Pluggable Authentication
    - MySQL 5.6에서 도입, MySQL 8.0에서 보완
    - 해시 값 생성을 위해 SHA-2 알고리즘 사용
    - 내부적으로 salt를 사용하며 동일 key에 대한 결과가 달라짐
    - 연산량이 많기 때문에 caching을 통해 이를 보완
    - 인증방식 이용을 위해 SSL/TLS 혹은 RSA 키페어를 사용해야함
- PAM Pluggable Authentication
    - MySQL 엔터프라이즈에서만 사용 가능
    - 유닉스나 리눅스 패스워드 또는 LDAP (Lightweight directory Access Protocol)같은 외부 인증을 사용
- LDAP Pluggable Authentication
    - MySQL 엔터프라이즈에서만 사용 가능
    - LDAP를 이용한 외부 인증 사용

### 3.2.2.2 REQUIRE

접속 시 SSL/TLS의 사용여부
#### ex. 3.2.3
- 다양한 암호화 옵션 선택 가능

```
REQUIRE NONE;

REQUIRE SSL;

REQUIRE X509;

REQUIRE ISSUER '/C=SE/ST=Stockholm/L=Stockholm/
    O=MySQL/CN=CA/emailAddress=ca@example.com';

REQUIRE SUBJECT '/C=SE/ST=Stockholm/L=Stockholm/
    O=MySQL demo client certificate/
    CN=client/emailAddress=client@example.com';
    
REQUIRE CIPHER 'EDH-RSA-DES-CBC3-SHA';

REQUIRE SSL AND X509;
```

### 3.2.2.3 PASSWORD EXPIRE

비밀번호의 유효 기간 설정

미입력 시 ```default_password_lifetime``` 시스템 변수에 저장된 기간으로 설정

응용 프로그램 접속용 계정에는 유효기간 X

- PASSWORD EXPIRE절 옵션
    - ```PASSWORD EXPIRE```: 계정 생성과 동시에 만료
    - ```PASSWORD EXPIRE NEVER```: 만료기간 없음
    - ```PASSWORD EXPIRE DEFAULT```: 기본 값으로 설정
    - ```PASSWORD EXPIRE INTERVAL n DAY```: 비밀번호 유효기간을 n일로 설정
    
### 3.2.2.4 PASSWORD HISTORY

이전에 사용했던 비밀번호를 재사용하지 못하게 설정

기본 값은 password_history 시스템 변수에 저장

- PASSWORD HISTORY절 옵션
    - ```PASSWORD HISTORY DEFAULT```: 기본 값 만큼 이력 저장
    - ```PASSWORD HISTORY n```: 최근 n개의 비밀번호 이력 저장
    
### 3.2.2.5 PASSWORD REUSE INTERVAL

이전에 사용한 비밀번호의 재사용 금지 기간을 설정

기본 값은 pssword_reuse_interval 시스템 변수에 저장
 - PASSWORD REUSE INTERVAL절 옵션
    - ```PASSWORD REUSE INTERVAL DEFAULT```: 기본 값으로 설정
    - ```PASSWORD RESUE INTERVAL n DAY```: n일 이후 비밀번호를 재사용 할 수 있게 설정
    
### 3.2.2.6 PASSWORD REQUIRE

비밀번호 만료 후 비밀번호를 변경할 때, 변경 전 비밀번호를 결정

기본 값은 password_require_current 시스템 변수에 저장

- PASSWORD REQUIRE절 옵션
    - ```PASSWORD REQUIRE CURRENT```: 비밀번호 변경 시 현재 비밀번호를 입력
    - ```PASSWORD REQUIRE OPTIONAL```: 비밀번호 변경 시 현재 비밀번호 미입력
    - ```PASSWORD REQUIRE DEFAULT```: 기본 값으로 설정 

### 3.2.2.7 ACCOUNT LOCK / UNLOCK

계정 생성 시, 혹은 ALTER USER를 통한 계정정보 변경 시 계정을 사용 못하도록 잠글지 결정

- 계정 잠금 설정
    - ```ACCOUNT LOCK```: 계정 잠금
    - ```ACCOUNT UNLOCK```: 계정 잠금 해제
    
    
## 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

- 유효기간이나 이력 관리를 통한 재사용 금지
- 글자의 조합을 강제, 금칙어 설정

유효성 체크 규칙 적용을 위해 ```valicate_password```를 설치 / 사용
```
mysql> INSTALL COMPONET 'file://component_validate_password';
mysql> SELECT * FROM mysql.component;

```
| component_id| component_group_id |component_urn                     |
|-------------|--------------------|----------------------------------|
|            1|                   1|file://component_validate_password|

```validate_password```에서 제공하는 시스템 변수 확인
```
mysql> SHOW GLOBAL VARIABLES LIKE 'validate_password%';
```
|Variable_name                       |Value                    |                                             목적|
|------------------------------------|-------------------------|-------------------------------------|
|   validate_password.check_user_name|             **ON** / OFF|                          user name과 같은지 검사|
|   validate_password.dictionary_file|       [금칙어 파일 경로]|                                      금칙어 파일|    
|            validate_password.length|          **8** [Integer]|                               비밀번호 길이 검사|
|  validate_password.mixed_case_count|          **2** [Integer]|         해당 값 이상의 대문자 & 소문자를 사용|
|      validate_password.number_count|          **2** [Integer]|                       해당 값 이상의 숫자를 사용|
|            validate_password.policy|LOW / **MEDIUM** / STRONG|                               비밀번호 검사 정책|
|validate_password.special_char_count|          **2** [Integer]|                  해당 값 이상의 특수 문자를 사용|

**Reference**: https://dev.mysql.com/doc/refman/8.0/en/validate-password-options-variables.html

- 비밀번호 정책
    - **LOW**: 비밀번호의 길이 검사
    - **MEDIUM**: LOW 포함 / 숫자, 대소문자, 특수문자의 배합을 검사 
    - **STRONG**: MEDIUM 포함 / 금칙어가 포함되었는지 여부 검사

### 3.3.2 이중 비밀번호

DB는 web application과 연결되어있어 계정정보 (비밀번호)를 변경하기는 어려움.

비밀번호를 2개 사용함으로써 서비스를 멈추지 않으면서 비밀번호 변경 가능.

#### ex. 3.3.1
```
// 비밀번호 변경
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

// 비밀번호를 변경하면서 기존 비밀번호를 secondary로 설정
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;;

// Secondary 비밀번호 삭제
mysql> ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD;
```

## 3.4 권한 (Privilege)
- 글로벌 권한
    - 데이터베이스나 테이블 이외의 객체에 적용되는 권한
- 객체 권한
    - 데이터베이스나 테이블에 적용되는 권한
    - ```GRANT``` 명령어를 통해 권한을 부여할 때, 특정 객체를 명시
- 동적 권한
    - 컴포넌트나 플러그인의 설치에 따라 등록
    - MySQL 8.0부터 지원

권한 테이블은 **Appendix.A** 참고 

#### 권한 부여

권한 부여는 다음과 같은 명령어를 통해서 부여 가능하다.

```
mysql> GRANT [privilege list] ON [database].[table] TO [user]@[host];
```

- 사용자를 생성하지 않은 상태로 권한 부여를 하면 오류 발생
- ```GRANT OPTION``` 권한은 ```GRANT [privilege list] ON [database].[table] TO [user]@[host] WITH GRANT OPTION```과 같이 실행
- 권한 종류에 따라 ON에 범위가 달라짐

#### ex. 3.4.1

- 글로벌 권한은 특정 객체에 부여되지 않기 때문에 ON절에 *.*으로 기입
    
```
mysql> GRANT SUPER ON *.* TO 'user'@'localhost';
```

#### ex. 3.4.2

- DB 권한은 특정 DB에 대해서 권한을 부여하거나 서버에 존재하는 모든 DB에 권한 부여
- ON절에 *.* 혹은 [database].*으로 기입

```
mysql> GRANT EVENT ON *.* 'user'@'localhost';
mysql> GRANT EVENT ON employees.* 'user'@'localhost';
```

#### ex. 3.4.3

- 테이블 권한은 권한 부여 범위에 따라 ON절에 *.* / [database].* / [database].[table]으로 기입

```
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON *.* 'user'@'localhost';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* 'user'@'localhost';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON employees.department 'user'@'localhost';
```

#### ex. 3.4.4

- 테이블에 특정 칼럼에 대해서만 권한을 부여 가능
- INSERT, UPDATE, SELECT 권한에 대해 동작

```
// INSERT는 dept_id에 대해서만 동작
// UPDATE는 dept_name에 대해서만 동작
// SELECT는 모든 컬럼에 대해 수행

mysql> GRANT SELECT, INSERT(dept_id), UPDATE(dept_name) ON employees.department 'user'@'localhost'; 
```

컬럼 단위의 설정은 나머지 모든 컬럼에 대해서도 검사하여 성능에 영향

VIEW를 만들어서 VIEW에 대한 권한으로 관리

## 3.5 역할 (Role)

MySQL 8.0부터 권한을 묶어서 Role로 사용

## Apendix

### A. 권한 목록

    
#### 글로벌 권한
**Reference**: https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html

| Privilege                     | Grant Table Column           | Context                               |
|-------------------------------|------------------------------|---------------------------------------|
| ```ALL [PRIVILEGES]```        | Synonym for “all privileges” | Server administration                 |
| ```ALTER```                   | Alter_priv                   | Tables                                |
| ```CREATE ROLE```             | Create_role_priv             | Server administration                 |
| ```CREATE TABLESPACE```       | Create_tablespace_priv       | Server administration                 |
| ```CREATE USER```             | Create_user_priv             | Server administration                 |
| ```DROP ROLE```               | Drop_role_priv               | Server administration                 |
| ```FILE```                    | File_priv                    | File access on server host            |
| ```PROCESS```                 | Process_priv                 | Server administration                 |
| ```PROXY```                   | See proxies_priv table       | Server administration                 |
| ```RELOAD```                  | Reload_priv                  | Server administration                 |
| ```REPLICATION CLIENT```      | Repl_client_priv             | Server administration                 |
| ```REPLICATION SLAVE```       | Repl_slave_priv              | Server administration                 |
| ```SHOW DATABASES```          | Show_db_priv                 | Server administration                 |
| ```SHUTDOWN```                | Shutdown_priv                | Server administration                 |
| ```SUPER```                   | Super_priv                   | Server administration                 |
| ```USAGE```                   | Synonym for “no privileges”  | Server administration                 |

#### 객체 권한
| Privilege                     | Grant Table Column           | Context                               |
|-------------------------------|------------------------------|---------------------------------------|
| ```ALL [PRIVILEGES]```        | Synonym for “all privileges” | Server administration                 |
| ```ALTER```                   | Alter_priv                   | Tables                                |
| ```ALTER ROUTINE```           | Alter_routine_priv           | Stored routines                       |
| ```CREATE```                  | Create_priv                  | Databases, tables, or indexes         |
| ```CREATE ROUTINE```          | Create_routine_priv          | Stored routines                       |
| ```CREATE TEMPORARY TABLES``` | Create_tmp_table_priv        | Tables                                |
| ```CREATE VIEW```             | Create_view_priv             | Views                                 |
| ```DELETE```                  | Delete_priv                  | Tables                                |
| ```DROP```                    | Drop_priv                    | Databases, tables, or views           |
| ```EVENT```                   | Event_priv                   | Databases                             |
| ```EXECUTE```                 | Execute_priv                 | Stored routines                       |
| ```GRANT OPTION```            | Grant_priv                   | Databases, tables, or stored routines |
| ```INDEX```                   | Index_priv                   | Tables                                |
| ```INSERT```                  | Insert_priv                  | Tables or columns                     |
| ```LOCK TABLES```             | Lock_tables_priv             | Databases                             |
| ```REFERENCES```              | References_priv              | Databases or tables                   |
| ```SELECT```                  | Select_priv                  | Tables or columns                     |
| ```SHOW VIEW```               | Show_view_priv               | Views                                 |
| ```TRIGGER```                 | Trigger_priv                 | Tables                                |
| ```UPDATE```                  | Update_priv                  | Tables or columns                     |

#### 동적 권한
| Privilege                          | Context                                         |
|------------------------------------|-------------------------------------------------|
| ```APPLICATION_PASSWORD_ADMIN```   | Dual password administration                    |
| ```AUDIT_ABORT_EXEMPT```           | Allow queries blocked by audit log filter       |
| ```AUDIT_ADMIN```                  | Audit log administration                        |
| ```AUTHENTICATION_POLICY_ADMIN```  | Authentication administration                   |
| ```BACKUP_ADMIN```                 | Backup administration                           |
| ```BINLOG_ADMIN```                 | Backup and Replication administration           |
| ```BINLOG_ENCRYPTION_ADMIN```      | Backup and Replication administration           |
| ```CLONE_ADMIN```                  | Clone administration                            |
| ```CONNECTION_ADMIN```             | Server administration                           |
| ```ENCRYPTION_KEY_ADMIN```         | Server administration                           |
| ```FIREWALL_ADMIN```               | Firewall administration                         |
| ```FIREWALL_EXEMPT```              | Firewall administration                         |
| ```FIREWALL_USER```                | Firewall administration                         |
| ```FLUSH_OPTIMIZER_COSTS```        | Server administration                           |
| ```FLUSH_STATUS```                 | Server administration                           |
| ```FLUSH_TABLES```                 | Server administration                           |
| ```FLUSH_USER_RESOURCES```         | Server administration                           |
| ```GROUP_REPLICATION_ADMIN```      | Replication administration                      |
| ```GROUP_REPLICATION_STREAM```     | Replication administration                      |
| ```INNODB_REDO_LOG_ARCHIVE```      | Redo log archiving administration               |
| ```NDB_STORED_USER```              | NDB Cluster                                     |
| ```PASSWORDLESS_USER_ADMIN```      | Authentication administration                   |
| ```PERSIST_RO_VARIABLES_ADMIN```   | Server administration                           |
| ```REPLICATION_APPLIER```          | PRIVILEGE_CHECKS_USER for a replication channel |
| ```REPLICATION_SLAVE_ADMIN```      | Replication administration                      |
| ```RESOURCE_GROUP_ADMIN```         | Resource group administration                   |
| ```RESOURCE_GROUP_USER```          | Resource group administration                   |
| ```ROLE_ADMIN```                   | Server administration                           |
| ```SENSITIVE_VARIABLES_OBSERVER``` | Server administration                           |
| ```SESSION_VARIABLES_ADMIN```      | Server administration                           |
| ```SET_USER_ID```                  | Server administration                           |
| ```SHOW_ROUTINE```                 | Server administration                           |
| ```SYSTEM_USER```                  | Server administration                           |
| ```SYSTEM_VARIABLES_ADMIN```       | Server administration                           |
| ```TABLE_ENCRYPTION_ADMIN```       | Server administration                           |
| ```VERSION_TOKEN_ADMIN```          | Server administration                           |
| ```XA_RECOVER_ADMIN```             | Server administration                           |
