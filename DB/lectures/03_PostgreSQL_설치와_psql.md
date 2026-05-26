# 3강 — PostgreSQL 설치 및 `psql` 사용법

> **목표:** 내 컴퓨터에 PostgreSQL을 설치하고, 첫 SQL을 직접 쳐본다.
> **선수 지식:** 1~2강

---

## 1. 무엇을 설치하나?

PostgreSQL을 깔면 보통 세 가지가 같이 들어온다.

| 이름 | 역할 | 비유 |
|------|------|------|
| **PostgreSQL 서버** | 실제로 데이터를 저장·관리하는 본체 | 게임 서버 |
| **`psql`** | 명령창에서 DB와 대화하는 도구 | 콘솔 클라이언트 |
| **pgAdmin** | DB를 GUI로 다루는 프로그램 | 관리자 웹페이지 |

이번 강의에선 **서버 + `psql`** 위주로 쓴다. `psql`이 좀 투박해 보여도 **SQL 문법을 익히는 데는 GUI보다 훨씬 좋다.** (헷갈릴 게 없어서)

---

## 2. Windows에 설치하기

### 2-1. 설치 파일 다운로드

- 공식 사이트: <https://www.postgresql.org/download/windows/>
- "Download the installer" 클릭 → EnterpriseDB 설치 파일 다운로드
- **최신 안정 버전**(이 강의 작성 시점 기준 17.x 이상)을 받으면 된다.

### 2-2. 설치 마법사 진행

설치 중 물어보는 것들:

| 항목 | 권장 입력 |
|------|---------|
| Installation Directory | 기본값 유지 |
| Select Components | 전부 체크 (Server, pgAdmin 4, Command Line Tools 필수) |
| Data Directory | 기본값 유지 |
| **Password** | **반드시 메모해둘 것** (관리자 `postgres` 계정 비밀번호) |
| Port | `5432` (기본값) |
| Locale | `Default locale` |

> ⚠️ **비밀번호는 절대 잊지 말 것.** 까먹으면 재설정이 꽤 귀찮다. 처음엔 `1234` 같은 단순한 거 써도 OK (로컬 학습용이니까).

설치 완료 후 마지막에 "Stack Builder를 실행하겠냐"고 묻는데, **체크 해제하고 종료** 해도 된다.

---

## 3. 설치 확인하기

### 3-1. `psql` 실행

윈도우 검색창에 **"SQL Shell (psql)"** 검색 → 실행.

검은 창이 뜨면서 이렇게 물어본다.

```text
Server [localhost]:        ← 그냥 Enter
Database [postgres]:       ← 그냥 Enter
Port [5432]:               ← 그냥 Enter
Username [postgres]:       ← 그냥 Enter
Password for user postgres: ← 설치 때 정한 비밀번호 입력 (안 보임)
```

성공하면 이렇게 나온다.

```text
psql (17.x)
Type "help" for help.

postgres=#
```

`postgres=#` 이게 **프롬프트**다. 여기에 SQL을 치면 된다.

> 💡 비밀번호 칠 때 화면에 `*`도 안 찍힌다. 보안상 정상. 그냥 치고 Enter.

---

## 4. 첫 SQL — 버전 확인

```sql
SELECT version();
```

세미콜론(`;`)을 꼭 붙이고 Enter. 결과:

```text
                          version
------------------------------------------------------------
 PostgreSQL 17.x on x86_64-windows, compiled by ...
(1 row)
```

축하한다. **방금 첫 SQL을 실행했다.** `SELECT`로 뭔가를 "조회"한 것이다.

> ⚠️ **세미콜론(`;`)을 안 붙이면 명령이 안 끝난 걸로 인식**하고 `postgres-#` 로 프롬프트가 바뀐다. 당황하지 말고 `;` 치고 Enter 하면 된다.

---

## 5. 실습용 DB 만들기

지금 우리는 `postgres`라는 기본 DB에 접속해 있다. 학습용으로 별도의 DB를 하나 만들자.

```sql
CREATE DATABASE study;
```

성공하면:

```text
CREATE DATABASE
```

만든 DB로 접속을 바꿔보자.

```text
postgres=# \c study
You are now connected to database "study" as user "postgres".
study=#
```

프롬프트가 `study=#` 로 바뀌었다. **이제 `study` DB 안에서 작업한다.**

> `\c` 는 SQL이 아니라 **`psql` 자체의 명령어**다. 백슬래시로 시작하는 명령은 모두 `psql` 전용 (메타 명령어). 자세한 건 다음 절에서.

---

## 6. 알아두면 편한 `psql` 메타 명령어

`\` 로 시작하는 명령어들. 세미콜론 필요 없음.

| 명령어 | 설명 |
|--------|------|
| `\l`   | 모든 DB 목록 보기 |
| `\c <DB이름>` | 다른 DB로 접속 변경 |
| `\dt`  | 현재 DB의 테이블 목록 |
| `\d <테이블이름>` | 테이블 구조(스키마) 보기 |
| `\?`   | 메타 명령어 도움말 전체 |
| `\h <SQL>` | SQL 문법 도움말 (`\h SELECT` 등) |
| `\q`   | psql 종료 |

지금은 다 외울 필요 없다. **자주 쓰는 건 `\l`, `\c`, `\dt`, `\q`** 정도.

---

## 7. 미니 실습 — 끝까지 한 번 해보기

```sql
-- 1. 테이블 만들기 (스키마 정의)
CREATE TABLE test_users (
    id   INTEGER,
    name TEXT
);

-- 2. 데이터 한 건 넣기
INSERT INTO test_users (id, name) VALUES (1, '김철수');

-- 3. 조회해보기
SELECT * FROM test_users;
```

마지막 결과:

```text
 id |  name
----+--------
  1 | 김철수
(1 row)
```

🎉 **DB 라이프 시작.**

> 문법은 다음 강의들에서 본격적으로 다루니까, 지금은 "이런 식으로 동작하는구나" 정도면 충분하다.

치고 나서 `\dt` 도 한 번 쳐보자. `test_users` 테이블이 목록에 보일 것이다.

---

## 8. 자주 만나는 문제 (트러블슈팅)

### "psql을 찾을 수 없다"고 나올 때

- 환경 변수 PATH에 PostgreSQL의 `bin` 폴더가 빠졌을 수 있다.
- 보통 경로: `C:\Program Files\PostgreSQL\17\bin`
- 시스템 환경 변수 → `Path`에 추가 → 새 명령창 열어서 다시 시도.

### "비밀번호 인증 실패"

- 설치 때 정한 `postgres` 계정 비밀번호를 정확히 입력했는지 확인.
- 까먹었다면 `pg_hba.conf` 수정으로 재설정 가능 (구글에 "postgres password reset" 검색).

### 한글이 깨질 때

- `psql`에서 한글이 깨지면 명령창에서 먼저 `chcp 65001` 입력 후 `psql` 실행.
- 또는 pgAdmin GUI를 쓰는 것도 방법.

---

## 9. 이번 강의에서 기억할 것

1. PostgreSQL 설치 시 **비밀번호 메모 필수.**
2. `psql`은 DB와 대화하는 **명령창 도구.**
3. **SQL 문장 끝엔 항상 세미콜론(`;`)** — 안 붙이면 안 실행된다.
4. `\` 로 시작하는 건 SQL이 아니라 **`psql` 메타 명령어** (`\l`, `\c`, `\dt`, `\q`).
5. 학습은 `study` DB에서 진행.

---

## 10. 다음 강의 예고

**4강 — 데이터베이스 / 스키마 / 테이블의 계층 구조**
"DB 안에 스키마, 스키마 안에 테이블"이라는 PostgreSQL의 3단 구조를 정리한다. 이걸 알아야 5강에서 `CREATE TABLE`을 할 때 "어디에" 만드는지 헷갈리지 않는다.
