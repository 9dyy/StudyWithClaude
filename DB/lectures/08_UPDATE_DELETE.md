# 8강 — `UPDATE`, `DELETE`로 데이터 수정·삭제

> **목표:** 기존 데이터를 안전하게 바꾸고 지운다.
> **선수 지식:** 7강 (`INSERT`)

> ⚠️ 이번 강의는 **사고가 가장 많이 나는** 구간이다. `WHERE` 절을 빼먹으면 **테이블 전체가 한 방에 날아간다.** 집중!

---

## 1. `UPDATE` — 데이터 수정

### 기본 문법

```sql
UPDATE 테이블이름
SET 열1 = 값1, 열2 = 값2
WHERE 조건;
```

C++로 비유:

```cpp
for (FUser& U : Users)
{
    if (U.Id == 1)       // ← WHERE
    {
        U.Level = 20;    // ← SET
        U.Gold  = 1000;
    }
}
```

### 실습

```sql
-- 김철수(id=1)의 레벨과 골드 변경
UPDATE users
SET level = 20, gold = 1000
WHERE id = 1;

-- 확인
SELECT * FROM users WHERE id = 1;
```

결과:

```text
UPDATE 1
```

`UPDATE 1`은 **1개 행이 바뀌었다**는 뜻. 0이 나오면 조건에 맞는 게 없었다는 의미.

---

## 2. ⚠️ `WHERE` 없는 `UPDATE`의 공포

```sql
UPDATE users SET level = 1;   -- 모든 유저 레벨이 1로!
```

오타 한 줄로 **100만 유저가 전부 레벨 1**이 된다. 진짜다. 실무에서 가장 흔한 대형사고 중 하나.

### 예방법 3가지

1. **`WHERE` 먼저 쓰는 습관.** `WHERE` → `SET` 순서로 작성하면 잊을 일이 줄어듦.
2. **항상 `SELECT`로 먼저 확인.**
   ```sql
   SELECT * FROM users WHERE id = 1;   -- 이거 먼저!
   UPDATE users SET level = 20 WHERE id = 1;
   ```
3. **`BEGIN` … `COMMIT` 트랜잭션으로 감싸기** (28강 본격 학습).
   ```sql
   BEGIN;
   UPDATE users SET level = 20 WHERE id = 1;
   -- 확인
   SELECT * FROM users WHERE id = 1;
   -- 괜찮으면
   COMMIT;
   -- 아니면
   ROLLBACK;
   ```

> 💡 **`ROLLBACK` = "취소" 버튼.** 트랜잭션 안에서면 `UPDATE` 실수도 되돌릴 수 있다.

---

## 3. `UPDATE`의 다양한 패턴

### ① 기존 값 기준으로 변경

```sql
-- 모든 유저 골드 10% 증가
UPDATE users SET gold = gold * 1.1;

-- 김철수의 골드를 100 깎기
UPDATE users SET gold = gold - 100 WHERE id = 1;
```

`gold = gold * 1.1` 같은 표현이 가능. 오른쪽의 `gold`는 **현재 값**.

### ② 여러 조건

```sql
-- 길드1 소속 유저들 중 레벨 10 미만만 골드 +500
UPDATE users
SET gold = gold + 500
WHERE guild_id = 1 AND level < 10;
```

### ③ `RETURNING` (INSERT에서 본 그것)

```sql
UPDATE users
SET level = level + 1
WHERE id = 1
RETURNING id, name, level;
```

변경된 행을 바로 받아본다. **로깅이나 후속 처리에 유용.**

### ④ 다른 테이블 참조 — `UPDATE ... FROM`

```sql
-- 각 유저의 골드를 길드 보너스만큼 증가 (예시)
UPDATE users
SET gold = users.gold + 100
FROM guilds
WHERE users.guild_id = guilds.id
  AND guilds.guild_name = '흑사자단';
```

`UPDATE`에서 `JOIN` 비슷한 걸 할 수 있다. 16강 이후 JOIN 배우고 다시 보면 이해 쉬움.

---

## 4. `DELETE` — 데이터 삭제

### 기본 문법

```sql
DELETE FROM 테이블이름
WHERE 조건;
```

> ⚠️ `DELETE FROM users` ← **이렇게만 쓰면 모든 유저가 사라진다.** `UPDATE`와 똑같은 함정.

### 실습

```sql
-- 비활성화된 유저 삭제
DELETE FROM users WHERE is_active = FALSE;

-- 특정 유저 삭제
DELETE FROM users WHERE id = 6;
```

결과의 `DELETE 1`은 1행 삭제 의미.

---

## 5. `DELETE` vs `TRUNCATE` vs `DROP`

세 명령은 비슷해 보이지만 다르다. 헷갈리지 말자.

| 명령 | 지우는 것 | 속도 | 트랜잭션 롤백 | 자동증가 ID |
|------|----------|------|---------------|-------------|
| `DELETE FROM t WHERE ...` | 조건 맞는 행 | 느림 | 가능 | 유지 |
| `DELETE FROM t` | 모든 행 | 느림 | 가능 | 유지 |
| `TRUNCATE t` | 모든 행 (빠른 삭제) | 빠름 | 가능 (PG에선) | 보통 유지 (`RESTART IDENTITY`로 초기화 가능) |
| `DROP TABLE t` | **테이블 자체** | 빠름 | 가능 | (테이블이 사라지니까 N/A) |

비유:
- `DELETE` = 서랍 안 물건 하나씩 꺼내기
- `TRUNCATE` = 서랍 통째로 비우기
- `DROP` = 서랍장을 박살내기

---

## 6. `RETURNING`은 `DELETE`에도 쓸 수 있다

```sql
DELETE FROM items
WHERE quantity = 0
RETURNING *;
```

지워진 행들을 화면에 보여준다. **"내가 뭘 지웠지?"** 를 확인할 때 유용.

---

## 7. 외래 키와 삭제 — 미리보기

`users.guild_id`가 `guilds.id`를 가리킨다고 외래 키를 걸어두면, **참조되고 있는 길드는 못 지운다.**

```sql
DELETE FROM guilds WHERE id = 1;
-- ERROR: update or delete on table "guilds" violates foreign key constraint ...
```

대처법:

- 먼저 자식(users) 정리하고 부모(guilds) 삭제
- 또는 `ON DELETE CASCADE` 옵션으로 **자동 연쇄 삭제**
- 또는 `ON DELETE SET NULL`로 자식의 `guild_id`만 NULL로

자세한 건 23강에서 다룸. 지금은 **"외래 키 걸려 있으면 그냥은 못 지운다"** 만 기억.

---

## 8. 실습 — 시나리오로 익히기

7강에서 채운 데이터 기준.

```sql
-- 1) 박민수(id=3)가 레벨업: 레벨 +1, 골드 +200
UPDATE users
SET level = level + 1, gold = gold + 200
WHERE id = 3
RETURNING id, name, level, gold;

-- 2) 길드 없는 유저(홍길동)를 흑사자단(id=1) 가입시키기
UPDATE users SET guild_id = 1 WHERE guild_id IS NULL;

-- 3) 마나포션 다 떨어진 사람의 마나포션 삭제 (quantity = 0짜리 가정)
-- 일단 한 명에게 0짜리 만들어두고
UPDATE items SET quantity = 0 WHERE id = 4;   -- 이영희의 마나포션 0개

DELETE FROM items WHERE quantity = 0 RETURNING *;

-- 4) 전체 유저 골드의 1% 세금 차감
UPDATE users SET gold = gold - (gold * 0.01);

-- 5) 트랜잭션 안전 실습
BEGIN;
DELETE FROM users WHERE id = 99;   -- 없는 ID, 0건 삭제
ROLLBACK;                          -- 어차피 확인했으니 되돌림
```

---

## 9. 안전하게 쓰는 체크리스트

- [ ] `WHERE` 절이 있나?
- [ ] 같은 조건으로 `SELECT` 먼저 돌려봤나?
- [ ] 영향받는 행 수를 머릿속으로 예상했나?
- [ ] 실수 시 되돌릴 트랜잭션을 열어뒀나?
- [ ] 정말 지워도 되는 데이터인가? (백업 필요?)

이 다섯 줄이 평생 도움 된다.

---

## 10. 이번 강의에서 기억할 것

1. **`UPDATE 테이블 SET ... WHERE ...;`** — `WHERE` 없으면 전체 변경.
2. **`DELETE FROM 테이블 WHERE ...;`** — `WHERE` 없으면 전체 삭제.
3. **`SELECT`로 먼저 확인** + **`BEGIN`/`ROLLBACK`으로 안전망**.
4. `RETURNING`은 변경/삭제된 행을 즉시 돌려준다.
5. `DELETE` / `TRUNCATE` / `DROP`은 **지우는 단위와 속도**가 다르다.

---

## 11. 다음 강의 예고

**9강 — `ALTER TABLE`, `DROP TABLE`로 구조 바꾸기**
이미 만든 테이블에 열을 추가하거나, 타입을 바꾸거나, 테이블을 통째로 지우는 법. **DDL의 마지막 조각.** 이걸로 2단계가 끝난다.
