# 17강 — `JOIN` 기본 (`INNER JOIN`, `LEFT JOIN`)

> **목표:** 두 테이블을 묶어서 한 번에 조회한다. PK/FK가 진짜 일하는 순간.
> **선수 지식:** 10~16강

---

## 1. 왜 JOIN이 필요한가?

16강에서 게임 DB를 PK/FK로 깔끔하게 분리해 두었다.

```text
guilds              users                   items
┌────┬────────┐    ┌────┬──────┬────────┐  ┌────┬─────────┬──────────┐
│ id │ name   │    │ id │ name │guild_id│  │ id │ user_id │ item_name│
├────┼────────┤    ├────┼──────┼────────┤  ├────┼─────────┼──────────┤
│  1 │흑사자단│←───┤  1 │김철수│   1    │←─┤  1 │    1    │ 회복포션 │
│  2 │ 백호단 │    │  2 │이영희│   1    │  │  2 │    1    │ 강철검   │
└────┴────────┘    │  3 │박무소│  NULL  │  │  3 │    2    │ 마법서   │
                   └────┴──────┴────────┘  └────┴─────────┴──────────┘
```

문제는, 화면에는 보통 **"김철수 — 흑사자단 — 회복포션"** 처럼 합쳐서 보여줘야 한다는 점. 따로 떨어진 테이블을 **연결해서 한 행으로 만드는 연산**이 필요하다. 그게 `JOIN`이다.

C++로 비유하면:

```cpp
for (const FUser& U : Users)
    for (const FGuild& G : Guilds)
        if (U.GuildId == G.Id)
            Print(U.Name, G.Name);
```

이중 루프로 매칭하는 걸 SQL은 `JOIN ... ON ...`으로 선언적으로 표현한다.

---

## 2. `INNER JOIN` — 양쪽 모두 매칭되는 행

### 문법

```sql
SELECT 열목록
FROM   왼쪽테이블
INNER JOIN 오른쪽테이블
       ON 매칭조건;
```

### 첫 예시

```sql
SELECT users.name, guilds.guild_name
FROM   users
INNER JOIN guilds
       ON users.guild_id = guilds.id;
```

결과:

```text
   name   | guild_name
----------+------------
 김철수   | 흑사자단
 이영희   | 흑사자단
```

`박무소속`(guild_id=NULL)은 **빠졌다.** `INNER JOIN`은 **양쪽에서 매칭되는 행만** 결과에 포함하기 때문.

### `INNER`는 생략 가능

```sql
SELECT users.name, guilds.guild_name
FROM users
JOIN guilds ON users.guild_id = guilds.id;   -- INNER 생략 = INNER JOIN
```

`JOIN`만 쓰면 기본이 `INNER JOIN`이다. 실무에선 `INNER`를 거의 생략한다.

---

## 3. `ON` 절 — 매칭 조건

JOIN의 핵심. 보통 **자식의 FK = 부모의 PK** 형태가 가장 흔하다.

```sql
ON users.guild_id = guilds.id     -- 가장 흔한 형태 (FK = PK)
ON items.user_id  = users.id      -- 같은 형태
```

조건은 한 개일 필요 없다. `AND`로 여러 조건도 가능.

```sql
SELECT ...
FROM orders o
JOIN order_items oi
  ON o.id = oi.order_id
 AND o.company_id = oi.company_id;   -- 복합키 매칭
```

`=`이 아닌 비교도 가능하지만 (`<`, `BETWEEN` 등) — 그건 18강에서.

---

## 4. 테이블 별칭 (alias) — `AS`

테이블 이름이 길거나 `JOIN` 여러 개를 쓸 때 매번 풀이름을 적으면 코드가 지저분해진다. **짧은 별칭**을 붙이는 게 관습.

```sql
SELECT u.name, g.guild_name
FROM   users  AS u
JOIN   guilds AS g  ON u.guild_id = g.id;
```

`AS`도 생략 가능하다.

```sql
SELECT u.name, g.guild_name
FROM   users  u
JOIN   guilds g  ON u.guild_id = g.id;
```

### 별칭 관례

| 테이블 | 자주 쓰는 별칭 |
|--------|---------------|
| `users` | `u` |
| `guilds` | `g` |
| `items` | `i` |
| `orders` | `o` |
| `order_items` | `oi` |

**첫 글자 + 명사 약자.** 가독성이 떨어지면 짧은 단어(`usr`, `gld`)도 OK.

> 💡 별칭을 한 번이라도 선언하면 그 뒤로는 **별칭만 써야** 한다. `users u` 라고 해놓고 `users.name`이라고 쓰면 에러.

---

## 5. `LEFT JOIN` — 왼쪽은 다 살리고 매칭

`INNER JOIN`에서 `박무소속`이 빠졌다. 하지만 "**모든 유저 보여줘. 길드 없으면 빈칸으로**" 라는 요구가 훨씬 흔하다. 그게 `LEFT JOIN`.

```sql
SELECT u.name, g.guild_name
FROM   users  u
LEFT JOIN guilds g  ON u.guild_id = g.id;
```

결과:

```text
   name   | guild_name
----------+------------
 김철수   | 흑사자단
 이영희   | 흑사자단
 박무소속 |   NULL        ← 매칭 안 됐어도 살아남음, 우측은 NULL
```

**왼쪽(`users`)은 무조건 다 나오고**, 오른쪽(`guilds`)에 매칭되는 게 없으면 그쪽 컬럼만 NULL이 된다.

### `INNER` vs `LEFT` — 한눈에

```text
INNER JOIN:  A ∩ B  (양쪽 모두 매칭)
LEFT  JOIN:  A     (왼쪽 전부 + 오른쪽 매칭되는 것)
```

| 상황 | 어느 쪽 |
|------|--------|
| "길드에 소속된 유저만" | `INNER JOIN` |
| "모든 유저, 길드 정보는 있으면 같이" | `LEFT JOIN` |
| "주문 + 결제(있다면)" | `LEFT JOIN` |
| "주문 + 그 주문의 상품 (둘 다 있어야)" | `INNER JOIN` |

> 💡 실무에서 `LEFT JOIN`이 압도적으로 많다. "정보가 없을 수도 있는 부가 데이터를 합치는" 패턴이 흔해서.

---

## 6. 컬럼 모호성 — `qualified` 이름

두 테이블에 같은 이름의 컬럼(예: `id`)이 있으면 그냥 `id`라고 쓰면 에러.

```sql
SELECT id, name        -- ❌ id가 users.id인지 guilds.id인지 모름
FROM users u
JOIN guilds g ON u.guild_id = g.id;
```

```text
ERROR: column reference "id" is ambiguous
```

해결: **`테이블별칭.컬럼` 형태**로 명시.

```sql
SELECT u.id, u.name, g.id AS guild_id, g.guild_name
FROM users u
JOIN guilds g ON u.guild_id = g.id;
```

> 💡 **습관적으로 모든 컬럼에 별칭을 붙이자.** 지금은 안 헷갈려도 JOIN이 3~4개가 되면 헷갈린다. 처음부터 `u.id`, `g.name`처럼 일관되게.

---

## 7. `JOIN` + `WHERE`

JOIN으로 합친 결과에 `WHERE`로 추가 필터를 거는 건 매우 흔하다.

```sql
-- 흑사자단 소속이면서 레벨 30 이상인 유저
SELECT u.name, u.level, g.guild_name
FROM   users  u
JOIN   guilds g  ON u.guild_id = g.id
WHERE  g.guild_name = '흑사자단'
  AND  u.level >= 30;
```

실행 순서(개념적): `FROM` → `JOIN` → `WHERE` → `SELECT`. 즉 **합친 다음 거른다**.

### ⚠️ `LEFT JOIN`에서의 `WHERE` 함정

```sql
-- 의도: "모든 유저, 흑사자단 정보 있으면 같이"
SELECT u.name, g.guild_name
FROM users u
LEFT JOIN guilds g  ON u.guild_id = g.id
WHERE g.guild_name = '흑사자단';
```

결과:

```text
   name   | guild_name
----------+------------
 김철수   | 흑사자단
 이영희   | 흑사자단     ← 박무소속 사라짐!
```

왜? `박무소속`은 `LEFT JOIN`까지는 살아남았지만(`g.guild_name = NULL`), `WHERE g.guild_name = '흑사자단'`에서 NULL이라 탈락했기 때문.

**`LEFT JOIN`을 유지하고 싶다면 우측 테이블 조건은 `ON`에 붙이자.**

```sql
SELECT u.name, g.guild_name
FROM users u
LEFT JOIN guilds g
  ON u.guild_id = g.id
 AND g.guild_name = '흑사자단';   -- 조건이 ON으로
```

이제 `박무소속`은 결과에 살아남고, `g.guild_name`만 NULL이 된다.

> 💡 **규칙:** `LEFT JOIN`의 **왼쪽** 테이블 조건은 `WHERE`, **오른쪽** 테이블 조건은 `ON`에. 헷갈리면 결과가 조용히 잘못 나온다 — 흔한 실무 버그.

---

## 8. 실습 — 게임 DB로 JOIN 연습

16강 실습 테이블 그대로 사용한다. 데이터를 약간 보강:

```sql
-- 깨끗하게 다시
DROP TABLE IF EXISTS items;
DROP TABLE IF EXISTS users;
DROP TABLE IF EXISTS guilds;

CREATE TABLE guilds (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    guild_name TEXT NOT NULL UNIQUE
);

CREATE TABLE users (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    level INTEGER NOT NULL DEFAULT 1,
    guild_id BIGINT,
    CONSTRAINT fk_users_guild FOREIGN KEY (guild_id) REFERENCES guilds(id)
);

CREATE TABLE items (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL,
    item_name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT fk_items_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

INSERT INTO guilds (guild_name) VALUES ('흑사자단'), ('백호단'), ('청룡회');
INSERT INTO users (name, level, guild_id) VALUES
  ('김철수', 30, 1),
  ('이영희', 45, 1),
  ('박무소속', 12, NULL),
  ('최백호', 50, 2);
INSERT INTO items (user_id, item_name, quantity) VALUES
  (1, '회복포션', 5),
  (1, '강철검', 1),
  (2, '마법서', 3),
  (3, '나무막대', 1);
```

```sql
-- 1) 모든 유저와 그 길드 이름 (길드 없으면 NULL)
SELECT u.name, g.guild_name
FROM users u
LEFT JOIN guilds g ON u.guild_id = g.id;

-- 2) 길드 가입자만
SELECT u.name, g.guild_name
FROM users u
JOIN guilds g ON u.guild_id = g.id;

-- 3) 모든 유저와 그들이 가진 아이템들 (아이템 없는 유저는?)
SELECT u.name, i.item_name, i.quantity
FROM users u
LEFT JOIN items i ON u.id = i.user_id;
-- 최백호는 아이템이 없으니 item_name이 NULL인 한 줄로 나옴

-- 4) 흑사자단 소속 유저의 아이템 목록 — 세 테이블 JOIN (다음 강의 예고)
SELECT u.name, g.guild_name, i.item_name
FROM users u
JOIN guilds g ON u.guild_id = g.id
JOIN items i  ON i.user_id = u.id
WHERE g.guild_name = '흑사자단';

-- 5) ⚠️ 함정 — LEFT JOIN인데 WHERE에 우측 조건
SELECT u.name, g.guild_name
FROM users u
LEFT JOIN guilds g ON u.guild_id = g.id
WHERE g.guild_name = '흑사자단';
-- '박무소속'이 사라진다 → 사실상 INNER JOIN이 돼버림

-- 6) ✅ 올바른 방법 — 우측 조건은 ON에
SELECT u.name, g.guild_name
FROM users u
LEFT JOIN guilds g
  ON u.guild_id = g.id
 AND g.guild_name = '흑사자단';
-- '박무소속'은 살아남고 g.guild_name이 NULL
```

---

## 9. 이번 강의에서 기억할 것

1. **`JOIN`은 두 테이블을 매칭 조건으로 합치는 연산.** PK/FK가 진가를 발휘.
2. **`INNER JOIN` = 양쪽 매칭만**, **`LEFT JOIN` = 왼쪽 전부 + 매칭되는 오른쪽**.
3. **`ON`이 매칭 조건**, 보통 `FK = PK` 형태.
4. **테이블 별칭** (`users u`) — 무조건 쓰는 습관. 컬럼은 `u.id`처럼 한정.
5. **`LEFT JOIN`의 우측 테이블 조건은 `ON`에, 좌측 조건만 `WHERE`에.** 이걸 헷갈리면 LEFT가 사실상 INNER로 둔갑한다.
6. 실무에서 `LEFT JOIN`이 훨씬 자주 보인다.

---

## 10. 다음 강의 예고

**18강 — `JOIN` 심화**
`RIGHT JOIN`, `FULL OUTER JOIN`, `CROSS JOIN`, **`SELF JOIN`**(자기 자신을 JOIN), `USING` 절, 그리고 **3개 이상 테이블을 한 번에 JOIN**하는 방법까지. 이번 강의의 ⚠️ 함정 이슈도 한 번 더 짚는다.
