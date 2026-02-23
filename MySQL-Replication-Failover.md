# MySQL Replication Failover 재현 실습

> **목표:** 동일한 장애 상황을 File/Position과 GTID 두 방식으로 각각 재현하여,  
> GTID 방식의 Failover가 얼마나 간편한지 직접 체감한다.

---

## 실습 환경 구성

```
토폴로지:  Source(A) ──→ Replica(B)
                    └──→ Replica(C)

시나리오:  Source(A) 장애 → Replica(B)를 새 Source로 승격
           → Replica(C)가 새 Source(B)를 바라보도록 전환
```

---

## Part 1. File/Position 기반 Failover 재현

### 1-1. 기존 컨테이너 정리

```bash
docker rm -f mysql-source-a mysql-replica-b mysql-replica-c 2>/dev/null
```

### 1-2. Docker 네트워크 생성

```bash
docker network create fp-net
```

### 1-3. Source(A) 생성 및 설정

```bash
docker run -d --name mysql-source-a --network fp-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-source-a bash
```

```bash
microdnf install vim -y
```

```bash
vim /etc/my.cnf
```

아래 내용을 `[mysqld]` 섹션에 추가:

```ini
[mysqld]
log-bin=mysql-bin
server-id=1
```

```bash
exit
```

```bash
docker restart mysql-source-a
```

### 1-4. Replica(B) 생성 및 설정

```bash
docker run -d --name mysql-replica-b --network fp-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-replica-b bash
microdnf install vim -y
vim /etc/my.cnf
```

```ini
[mysqld]
log-bin=mysql-bin
server-id=2
```

```bash
exit
docker restart mysql-replica-b
```

### 1-5. Replica(C) 생성 및 설정

```bash
docker run -d --name mysql-replica-c --network fp-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-replica-c bash
microdnf install vim -y
vim /etc/my.cnf
```

```ini
[mysqld]
log-bin=mysql-bin
server-id=3
```

```bash
exit
docker restart mysql-replica-c
```

### 1-6. Source(A)에서 복제 계정 + 테스트 데이터 생성

```bash
docker exec -it mysql-source-a mysql -u root -p1234
```

```sql
-- 복제 전용 계정 생성
CREATE USER 'repl'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 테스트 데이터
CREATE DATABASE shopdb;
USE shopdb;
CREATE TABLE products (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO products (name) VALUES ('apple'), ('banana'), ('cherry');

-- 현재 binlog 좌표 확인 (기록해두기!)
SHOW BINARY LOG STATUS\G
```

출력 예시 (값을 메모):

```
File: mysql-bin.000001
Position: 1234
```

```sql
exit
```

### 1-7. 초기 데이터 동기화 (dump)

```bash
docker exec mysql-source-a mysqldump -u root -p1234 --all-databases --source-data > /tmp/fp-dump.sql
docker cp mysql-source-a:/tmp/fp-dump.sql /tmp/fp-dump.sql 2>/dev/null || true

# 직접 dump 파일 생성
docker exec mysql-source-a bash -c "mysqldump -u root -p1234 --all-databases --source-data > /tmp/dump.sql"

# Replica(B)에 복사 및 적용
docker cp mysql-source-a:/tmp/dump.sql /tmp/fp-dump.sql
docker cp /tmp/fp-dump.sql mysql-replica-b:/tmp/dump.sql
docker exec mysql-replica-b bash -c "mysql -u root -p1234 < /tmp/dump.sql"

# Replica(C)에 복사 및 적용
docker cp /tmp/fp-dump.sql mysql-replica-c:/tmp/dump.sql
docker exec mysql-replica-c bash -c "mysql -u root -p1234 < /tmp/dump.sql"
```

### 1-8. Replica(B)와 Replica(C)에서 복제 시작

> ⚠️ `SOURCE_LOG_FILE`과 `SOURCE_LOG_POS`는 1-6에서 메모한 값으로 대체

**Replica(B) 설정:**

```bash
docker exec -it mysql-replica-b mysql -u root -p1234
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source-a',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_LOG_FILE='mysql-bin.000001',   -- 메모한 값
  SOURCE_LOG_POS=1234;                  -- 메모한 값

START REPLICA;
SHOW REPLICA STATUS\G
-- Replica_IO_Running: Yes, Replica_SQL_Running: Yes 확인
exit
```

**Replica(C) 설정:**

```bash
docker exec -it mysql-replica-c mysql -u root -p1234
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source-a',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_LOG_FILE='mysql-bin.000001',   -- 메모한 값
  SOURCE_LOG_POS=1234;                  -- 메모한 값

START REPLICA;
SHOW REPLICA STATUS\G
exit
```

### 1-9. 정상 복제 확인

Source(A)에서 데이터 추가:

```bash
docker exec -it mysql-source-a mysql -u root -p1234 -e \
  "INSERT INTO shopdb.products (name) VALUES ('date'), ('elderberry');"
```

Replica(B), (C)에서 확인:

```bash
docker exec mysql-replica-b mysql -u root -p1234 -e "SELECT * FROM shopdb.products;"
docker exec mysql-replica-c mysql -u root -p1234 -e "SELECT * FROM shopdb.products;"
```

> 양쪽 모두 date, elderberry가 보이면 정상

### 1-10. 💥 Source(A) 장애 발생!

```bash
docker stop mysql-source-a
```

### 1-11. Replica(B)를 새 Source로 승격

```bash
docker exec -it mysql-replica-b mysql -u root -p1234
```

```sql
-- 복제 중지
STOP REPLICA;
RESET REPLICA ALL;

-- B에서의 현재 binlog 좌표 확인
SHOW BINARY LOG STATUS\G
```

출력 예시:

```
File: mysql-bin.000001
Position: 876        ← A의 Position과 완전히 다른 값!
```

```sql
-- B에 복제 계정 생성 (새 Source 역할)
CREATE USER 'repl'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

exit
```

### 1-12. 🚨 Replica(C)를 새 Source(B)로 전환 — 여기가 문제!

```bash
docker exec -it mysql-replica-c mysql -u root -p1234
```

```sql
-- 현재 복제 상태 확인
SHOW REPLICA STATUS\G
```

이 시점에서 확인할 값들:

```
Source_Log_File: mysql-bin.000001      ← A의 binlog 파일명
Read_Source_Log_Pos: 1890              ← A의 binlog에서 읽은 마지막 위치
Exec_Source_Log_Pos: 1890              ← A의 binlog에서 실행한 마지막 위치
```

**핵심 문제 발생:**

```
A의 마지막 Position:  1890  (← C가 알고 있는 좌표)
B의 현재 Position:     876  (← B의 binlog에서의 좌표)

→ 1890 ≠ 876
→ A의 pos:1890이 B에서 어디에 해당하는지 알 수 없음!
→ 잘못된 Position을 지정하면 데이터 누락 또는 중복 발생
```

**수동 해결 시도 (실무에서는 매우 고통스러운 과정):**

```sql
-- 일단 복제 중지
STOP REPLICA;

-- B의 binlog 좌표를 수동으로 지정해야 함
-- B에서 SHOW BINARY LOG STATUS로 확인한 Position 사용
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-replica-b',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_LOG_FILE='mysql-bin.000001',   -- B의 파일명
  SOURCE_LOG_POS=876;                   -- B의 Position (정확할까?)

START REPLICA;
SHOW REPLICA STATUS\G
```

> ⚠️ **실무에서는** B의 Position 876이 C가 이미 실행한 트랜잭션 이후의 지점인지 확신할 수 없습니다.  
> `mysqlbinlog` 도구로 A와 B의 binlog를 직접 파싱하여 이벤트를 대조해야 하며,  
> 이 과정에서 **15~30분 이상** 소요되고 인적 오류 위험이 높습니다.

```sql
exit
```

### 1-13. 검증 — 새 Source(B)에서 데이터 추가

```bash
docker exec mysql-replica-b mysql -u root -p1234 -e \
  "INSERT INTO shopdb.products (name) VALUES ('fig');"
```

```bash
docker exec mysql-replica-c mysql -u root -p1234 -e \
  "SELECT * FROM shopdb.products;"
```

> fig가 보이면 복제 전환 성공 (하지만 정확한 Position을 찾는 과정이 매우 고통스러웠음)

### 1-14. 정리

```bash
docker rm -f mysql-source-a mysql-replica-b mysql-replica-c
docker network rm fp-net
```

---

## Part 2. GTID 기반 Failover 재현

### 2-1. Docker 네트워크 생성

```bash
docker network create gtid-net
```

### 2-2. Source(A) 생성 및 설정

```bash
docker run -d --name mysql-source-a --network gtid-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-source-a bash
microdnf install vim -y
vim /etc/my.cnf
```

```ini
[mysqld]
log-bin=mysql-bin
server-id=1
gtid_mode=ON
enforce_gtid_consistency=ON
```

```bash
exit
docker restart mysql-source-a
```

### 2-3. Replica(B) 생성 및 설정

```bash
docker run -d --name mysql-replica-b --network gtid-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-replica-b bash
microdnf install vim -y
vim /etc/my.cnf
```

```ini
[mysqld]
log-bin=mysql-bin
server-id=2
gtid_mode=ON
enforce_gtid_consistency=ON
```

```bash
exit
docker restart mysql-replica-b
```

### 2-4. Replica(C) 생성 및 설정

```bash
docker run -d --name mysql-replica-c --network gtid-net \
  -e MYSQL_ROOT_PASSWORD=1234 mysql
```

```bash
docker exec -it mysql-replica-c bash
microdnf install vim -y
vim /etc/my.cnf
```

```ini
[mysqld]
log-bin=mysql-bin
server-id=3
gtid_mode=ON
enforce_gtid_consistency=ON
```

```bash
exit
docker restart mysql-replica-c
```

### 2-5. Source(A)에서 복제 계정 + 테스트 데이터 생성

```bash
docker exec -it mysql-source-a mysql -u root -p1234
```

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

CREATE DATABASE shopdb;
USE shopdb;
CREATE TABLE products (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO products (name) VALUES ('apple'), ('banana'), ('cherry');

-- GTID 상태 확인
SHOW BINARY LOG STATUS\G
```

출력 예시:

```
File: mysql-bin.000001
Position: 1234
Executed_Gtid_Set: aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:1-8
```

> File/Position과 달리 **Executed_Gtid_Set을 메모할 필요 없음** (자동 관리)

```sql
exit
```

### 2-6. 초기 데이터 동기화 (dump)

```bash
docker exec mysql-source-a bash -c \
  "mysqldump -u root -p1234 --all-databases --triggers --routines --events --set-gtid-purged=ON > /tmp/dump.sql"

docker cp mysql-source-a:/tmp/dump.sql /tmp/gtid-dump.sql

# Replica(B)에 적용
docker cp /tmp/gtid-dump.sql mysql-replica-b:/tmp/dump.sql
docker exec mysql-replica-b bash -c "mysql -u root -p1234 < /tmp/dump.sql"

# Replica(C)에 적용
docker cp /tmp/gtid-dump.sql mysql-replica-c:/tmp/dump.sql
docker exec mysql-replica-c bash -c "mysql -u root -p1234 < /tmp/dump.sql"
```

> `--set-gtid-purged=ON`이 핵심! dump에 GTID 메타데이터를 포함시킨다.

### 2-7. Replica(B)와 Replica(C)에서 복제 시작

**Replica(B):**

```bash
docker exec -it mysql-replica-b mysql -u root -p1234
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source-a',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_AUTO_POSITION=1;          -- ← 이것만 지정!

START REPLICA;
SHOW REPLICA STATUS\G
exit
```

> `SOURCE_LOG_FILE`이나 `SOURCE_LOG_POS`를 지정하지 않는다!

**Replica(C):**

```bash
docker exec -it mysql-replica-c mysql -u root -p1234
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source-a',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_AUTO_POSITION=1;

START REPLICA;
SHOW REPLICA STATUS\G
exit
```

### 2-8. 정상 복제 확인

```bash
docker exec mysql-source-a mysql -u root -p1234 -e \
  "INSERT INTO shopdb.products (name) VALUES ('date'), ('elderberry');"
```

```bash
docker exec mysql-replica-b mysql -u root -p1234 -e "SELECT * FROM shopdb.products;"
docker exec mysql-replica-c mysql -u root -p1234 -e "SELECT * FROM shopdb.products;"
```

GTID 상태도 확인:

```bash
docker exec mysql-replica-c mysql -u root -p1234 -e "SHOW BINARY LOG STATUS\G"
```

> `Executed_Gtid_Set`에 Source(A)의 UUID와 트랜잭션 범위가 표시됨

### 2-9. 💥 Source(A) 장애 발생!

```bash
docker stop mysql-source-a
```

### 2-10. Replica(B)를 새 Source로 승격

```bash
docker exec -it mysql-replica-b mysql -u root -p1234
```

```sql
STOP REPLICA;
RESET REPLICA ALL;

-- 복제 계정 생성
CREATE USER 'repl'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 현재 GTID 상태 확인
SHOW BINARY LOG STATUS\G
exit
```

### 2-11. ⚡ Replica(C)를 새 Source(B)로 전환 — 단 3줄!

```bash
docker exec -it mysql-replica-c mysql -u root -p1234
```

```sql
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-replica-b',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='1234',
  SOURCE_AUTO_POSITION=1;              -- ← 좌표 지정 없음!

START REPLICA;
SHOW REPLICA STATUS\G
```

**확인 포인트:**

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

> 🎉 **끝입니다.** Position을 계산할 필요가 전혀 없었습니다.  
> Replica(C)는 자신의 `Executed_Gtid_Set`을 B에게 보내고,  
> B는 C에 없는 GTID만 자동으로 전송합니다.

```sql
exit
```

### 2-12. 검증 — 새 Source(B)에서 데이터 추가

```bash
docker exec mysql-replica-b mysql -u root -p1234 -e \
  "INSERT INTO shopdb.products (name) VALUES ('fig');"
```

```bash
docker exec mysql-replica-c mysql -u root -p1234 -e \
  "SELECT * FROM shopdb.products;"
```

GTID가 자동 갱신되었는지 확인:

```bash
docker exec mysql-replica-c mysql -u root -p1234 -e "SHOW BINARY LOG STATUS\G"
```

> `Executed_Gtid_Set`에 B의 UUID도 추가되어 있음:
>
> ```
> aaaaaaaa-...:1-10,    ← Source(A)에서 받은 트랜잭션
> bbbbbbbb-...:1-1      ← 새 Source(B)에서 받은 트랜잭션
> ```

### 2-13. 정리

```bash
docker rm -f mysql-source-a mysql-replica-b mysql-replica-c
docker network rm gtid-net
```

---

## 결과 비교 요약

| 항목                        | File/Position                         | GTID                                    |
| --------------------------- | ------------------------------------- | --------------------------------------- |
| **Failover 시 핵심 명령어** | `CHANGE SOURCE TO ... FILE=?, POS=?;` | `CHANGE SOURCE TO ... AUTO_POSITION=1;` |
| **좌표 계산**               | B의 Position을 수동으로 계산해야 함   | 자동 (Executed_Gtid_Set 기반)           |
| **데이터 누락/중복 위험**   | Position이 틀리면 발생                | GTID가 정확성을 보장                    |
| **필요한 지식**             | mysqlbinlog 파싱, 이벤트 대조         | 호스트명만 변경하면 됨                  |
| **실제 소요 시간 (체감)**   | 복제 전환에 많은 시간 소요            | 즉시 전환 가능                          |

---

## 핵심 체감 포인트

Part 1(File/Position)의 **Step 1-12**에서 겪은 상황을 떠올려 보세요:

```
A의 Position 1890  →  B에서 몇 번?  →  알 수 없음!
```

Part 2(GTID)의 **Step 2-11**에서는 이런 고민이 전혀 없었습니다:

```sql
-- C가 B에게 "나는 여기까지 실행했어" 라고 알려주면 끝
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-replica-b',       -- 호스트만 변경
  SOURCE_AUTO_POSITION=1;              -- 나머지는 자동
```

이것이 GTID가 실무에서 사실상 표준이 된 이유입니다.
