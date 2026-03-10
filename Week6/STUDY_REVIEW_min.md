## 📌 주제별 학습 정리 (개인 작성)

---

### 📍 Topic A – JOIN 전략

---

### 1️⃣ 핵심 개념 한 줄
> JOIN: JOIN은 두 개 이상의 테이블을 결합하여 데이터를 조회하는 SQL 연산이다. 관계형 데이터베이스에서 데이터는 정규화를 통해 여러 테이블로 분산되어 저장되므로, 의미 있는 정보를 얻기 위해서는 JOIN이 필수적이다. <br></br>
> INNER JOIN: 두 테이블 모두에 조인 조건에 맞는 값이 있는 행만 반환하는 교집합 방식 <br></br>
> LEFT OUTER JOIN: 왼쪽(기준) 테이블의 모든 행을 포함하고, 오른쪽 테이블에서 매칭되는 데이터가 있으면 결합하고, 없으면 NULL로 채운다. <br></br>

> RIGHT OUTER JOIN: LEFT JOIN과 비슷, 오른쪽 테이블 기준 모두 유지. 왼쪽에 매칭 데이터가 없으면 NULL <br></br>
> FULL OUTER JOIN: 양쪽 테이블 모두 유지, 매칭 안 되는 데이터는 NULL<br></br>


---

### 2️⃣ 동작 원리 요약

- **왜 필요한가?**
  - 연관 데이터를 기준으로 필터링/분석 위해 (단순 조회뿐 아니라, 관계 데이터를 기준으로 분석 가능)
    - 댓글이 있는 게시글만 보고 싶을 때 -> INNER JOIN
    - 모든 게시글과 댓글을 함께 보고 싶을 때 -> LEFT JOIN
  - 데이터 무결성을 유지하면서 효율적 조회
  - 관계형 DB는 데이터 중복을 최소화하기 위해 정보를 여러 테이블로 나눠 저장한다. 
  - JOIN은 관계형 데이터베이스에서 여러 테이블에 분리된 데이터를 논리적으로 결합하여 의미 있는 정보를 얻고, 효율적 조회와 무결성을 유지하기 위해 필요하다.
  ```
  예시:
  user 테이블: 사용자 정보(id, name)
  order 테이블: 주문 정보(order_id, user_id, product)
  
  한 테이블에서만 보면 사용자의 주문 내역을 바로 볼 수 없음.
  JOIN을 통해 두 테이블을 연결해야 “사용자 이름 + 주문 정보” 같이 의미 있는 데이터를 얻을 수 있음.
  ```
  
- **어떻게 동작하는가?**
  - INNER JOIN
    ```
    왼쪽 테이블의 각 행을 순회
    오른쪽 테이블에서 ON 조건과 일치하는 행 탐색
    일치하면 결과에 추가, 없으면 제외
  ```
  - LEFT OUTER JOIN
    ```
    왼쪽 테이블의 각 행을 순회
    오른쪽 테이블에서 ON 조건과 일치하는 행 탐색
    일치하면 결합, 없으면 오른쪽 컬럼에 NULL 채워서 결과 추가
    ```

---

### 3️⃣ 핵심 키워드

- `JOIN`
- `INNER JOIN`
- `LEFT OUTER JOIN`
- `JOIN 순서`
- `1:N Multiplication`

---

### 4️⃣ 주의 포인트

- ❗ NULL과 JOIN 조건
    ```
    INNER JOIN: NULL 값은 매칭되지 않아 자동 제외

    LEFT JOIN: NULL 값 포함 → 필터링 시 주의 필요

    오해 포인트: NULL을 단순히 비교(=)하면 의도와 다르게 걸러질 수 있음

    해결: IS NULL / IS NOT NULL / COALESCE() 사용
    ```
    
    
- ⚠️ LEFT JOIN + WHERE 조건
    ```sql
    흔히 하는 실수: LEFT JOIN 후 WHERE에서 오른쪽 컬럼 NULL 필터링
    
    SELECT p.title, c.text
    FROM post p
    LEFT JOIN comment c ON p.id = c.post_id
    WHERE c.text IS NOT NULL;  -> 잘못됨!!!


    문제: LEFT JOIN 효과 사라지고 결과가 INNER JOIN과 동일

    해결: 필터를 ON 절로 이동
    SELECT p.title, c.text
    FROM post p
    LEFT JOIN comment c ON p.id = c.post_id AND c.text IS NOT NULL;  -- 올바른 사례

    ```

### 5️⃣ 자주 나오는 면접 질문 (선택)

    
- Q. INNER JOIN과 LEFT OUTER JOIN의 차이를 설명해주세요.
    - A : 
    ```
    두 조인의 가장 큰 차이는 매칭되지 않는 데이터의 포함 여부입니다. 
    INNER JOIN은 두 테이블 모두에 조인 조건에 맞는 값이 있는 행만 반환하는 교집합 방식입니다. 

    반면 LEFT OUTER JOIN은 왼쪽 테이블의 모든 행을 유지하며, 오른쪽 테이블에 매칭되는 데이터가 없으면 NULL로 채워서 반환합니다.
    ```
    
    
- Q. 게시글 목록을 보여줄 때, 댓글이 하나도 없는 게시글도 함께 보여주려면 어떤 JOIN을 써야 할까요?
    - A : 
    ```
   게시글이 댓글과 관계없이 모두 조회되어야 하므로 LEFT JOIN을 사용합니다.

    댓글이 없으면 NULL로 채워지며, 쿼리 예시는 다음과 같습니다:

    SELECT p.title, c.text
    FROM post p
    LEFT JOIN comment c
    ON [p.id](http://p.id/) = c.post_id;

    게시글 테이블을 왼쪽에 두고 댓글 테이블을 `LEFT JOIN`하면, 댓글이 없는 게시글(오른쪽 테이블 데이터가 없는 경우)도 누락되지 않고 결과를 반환할 수 있습니다. 

    만약 `INNER JOIN`을 쓴다면 댓글이 없는 게시글은 결과에서 모두 사라지게 됩니다.
    ```
    
    
- Q. LEFT JOIN 후 WHERE 절에서 오른쪽 테이블의 컬럼을 조건으로 사용하면 어떤 문제가 발생하나요?
    - A : 
    ```
    LEFT JOIN 효과가 사라지고 INNER JOIN과 동일해짐.
    이럴 때는 조인 조건인 ON 절에 해당 조건을 포함시켜야 합니다.
    ```
    
    
- Q. OUTER JOIN은 성능상 INNER JOIN보다 항상 느린가요? 성능 차이가 발생하는 이유는?
    - A : 
    ```
    이론적으로 OUTER JOIN이 조금 더 느릴 수 있습니다. 
    연관 데이터가 없어도 조회해야 하는 경우에 필요합니다.
    다만 결과 데이터가 많아질 수 있고 불필요하게 쓰면 성능 저하가 발생할 수 있으므로, 실제 필요할 때만 사용해야 합니다.
    
    INNER JOIN은 매칭되는 결과가 없으면 즉시 건너뛰는 최적화가 가능하지만,
    OUTER JOIN은 매칭되는 데이터가 없더라도 왼쪽 테이블의 레코드를 위해 NULL을 생성하는 추가 작업을 수행해야 하기 때문입니다. 하지만 가장 큰 성능 차이는 인덱스 활용 여부와 드라이빙 테이블(기준 테이블)의 크기에서 발생합니다.
    ```

### 6️⃣ 꼬리 질문 & 대응 포인트 (선택)

- 꼬리 질문:
- 대응 논리:

### 7️⃣ 실무 적용 or 가상 시나리오 (선택)

- 실제/가정 상황:
- 선택 이유:

