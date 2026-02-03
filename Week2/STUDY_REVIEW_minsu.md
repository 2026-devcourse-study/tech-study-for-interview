## 📌 주제별 학습 정리 (개인 작성)

---

### 📍 Topic A – 트랜잭션 / AOP

---

### 1️⃣ 핵심 개념 한 줄
> **트랜잭션은 데이터베이스 작업의 논리적 단위로 ACID 특성을 보장하며, Spring의 @Transactional은 프록시 AOP를 통해 선언적으로 트랜잭션 경계를 설정하여 자동으로 커밋/롤백을 처리하는 구조입니다.**

---

### 2️⃣ 동작 원리 요약

- **왜 필요한가?**
  - 여러 DB 작업이 하나의 논리적 단위로 수행되어야 할 때 중간 실패 시 데이터 불일치 발생
  - 예: 계좌 이체 시 출금은 성공했지만 입금 실패 → 돈이 사라지는 문제
  - 👉 트랜잭션은 **All or Nothing** 원칙으로 데이터 일관성을 보장

- **어떻게 동작하는가?**
  - Spring은 **@Transactional** 어노테이션으로 트랜잭션 경계 설정
  - **AOP Proxy**가 메서드 호출을 가로채서 트랜잭션 시작/커밋/롤백 처리
  - **PlatformTransactionManager**가 실제 트랜잭션 관리
  - 기본 동작: 메서드 시작 시 트랜잭션 시작 → 성공 시 커밋, **RuntimeException/Error** 발생 시 롤백
  - 👉 개발자는 비즈니스 로직에만 집중, 트랜잭션 처리는 Spring이 자동화

**[AOP 프록시 동작 흐름]**
```
1. 외부에서 @Transactional 메서드 호출
2. AOP Proxy가 호출을 가로채기 (프록시 객체가 먼저 받음)
3. TransactionManager.getTransaction() - 트랜잭션 시작
4. 실제 Target 객체의 메서드 실행
5. 성공 → commit() / RuntimeException/Error → rollback()
6. 트랜잭션 종료 및 결과 반환
```

---

### 3️⃣ 핵심 키워드

- **`ACID`**
  - **Atomicity(원자성)**: 트랜잭션 내 모든 작업은 전부 성공 또는 전부 실패 (All or Nothing)
  - **Consistency(일관성)**: 트랜잭션 전후로 데이터 무결성 제약 조건 유지
  - **Isolation(격리성)**: 동시 실행되는 트랜잭션들이 서로 간섭하지 않음
  - **Durability(지속성)**: 커밋된 데이터는 시스템 장애가 발생해도 영구 보존

- **`@Transactional`** - 선언적 트랜잭션 설정 (클래스/메서드 레벨)

- **`AOP Proxy`** - @Transactional 구현 방식
  - **JDK Dynamic Proxy**: 인터페이스 기반
  - **CGLIB**: 클래스 기반 (Spring Boot 기본값)

- **`Propagation (전파)`** - 트랜잭션 경계 설정 방식
  - **REQUIRED (기본)**: 기존 트랜잭션이 있으면 참여, 없으면 새로 생성
  - **REQUIRES_NEW**: 항상 새 트랜잭션 생성 (기존 트랜잭션 일시 중단)

- **`rollbackFor`** - RuntimeException 외에 추가로 롤백할 예외 지정

---

### 4️⃣ 주의 포인트

- ❗ **프록시 방식의 한계 - @Transactional이 동작하지 않는 경우**

  **1) 같은 클래스 내부 메서드 호출 시 동작 안 함**
  ```java
  @Service
  public class UserService {
      // 외부에서 호출 시 트랜잭션 동작 ✅
      public void externalMethod() {
          internalMethod(); // ❌ 내부 호출: 프록시를 거치지 않아 트랜잭션 안 됨
      }

      @Transactional
      public void internalMethod() {
          // ...
      }
  }
  ```
  - 이유: 내부 호출은 `this.internalMethod()` 형태로 **실제 객체(Target)를 직접 호출**
  - **프록시를 거치지 않아** AOP가 적용되지 않음
  - 해결: 별도 클래스로 분리 (다른 Bean 주입받아 호출)

  **2) private 메서드에 @Transactional 동작 안 함**
  ```java
  @Transactional
  private void privateMethod() { } // ❌ 프록시가 접근 불가
  ```
  - 이유: **프록시는 상속(CGLIB) 또는 인터페이스(JDK Proxy) 기반**
  - private 메서드는 **오버라이딩/구현 불가능**하여 프록시가 가로챌 수 없음
  - 해결: public 또는 protected로 변경

  **3) final 클래스/메서드 (CGLIB 사용 시)**
  - CGLIB는 **상속 기반** 프록시 → final은 상속 불가
  - 해결: final 제거 또는 인터페이스 사용

- ⚠️ **RuntimeException만 기본 롤백 대상**
  - **Checked Exception**은 롤백 안 됨
  - 이유: Spring은 Checked Exception을 **복구 가능한 예외**로 간주
  - 해결: `@Transactional(rollbackFor = Exception.class)` 명시

- ⚠️ **전파 타입 선택 주의**
  - **REQUIRED**: 하나의 트랜잭션으로 묶고 싶을 때 (기본값)
  - **REQUIRES_NEW**: 독립적으로 커밋/롤백이 필요할 때 (예: 로그 저장)
    - 주의: 별도 트랜잭션이라 성능 저하 가능

---

### 5️⃣ 자주 나오는 면접 질문

- Q. **트랜잭션이란?**
  - A. **데이터베이스 작업의 논리적 단위**로, ACID 특성을 보장하여 All or Nothing 원칙으로 데이터 일관성을 유지합니다.

- Q. **ACID가 뭔가요?**
  - A. **Atomicity(원자성)**: 전부 성공 또는 전부 실패 / **Consistency(일관성)**: 무결성 제약 조건 유지 / **Isolation(격리성)**: 트랜잭션 간 간섭 방지 / **Durability(지속성)**: 커밋된 데이터는 영구 보존

- Q. **@Transactional 동작 원리는?**
  - A. **AOP 프록시 기반**으로 동작합니다. 메서드 호출 시 **프록시가 가로채서** 트랜잭션을 시작하고, 메서드 종료 시 성공이면 커밋, RuntimeException 발생 시 롤백합니다.

- Q. **왜 private 메서드에 @Transactional이 안 먹나요?**
  - A. 프록시는 **상속 또는 인터페이스 기반**으로 동작하는데, private 메서드는 **오버라이딩/구현이 불가능**하여 프록시가 가로챌 수 없기 때문입니다.

- Q. **트랜잭션이 적용 안 되는 경우는?**
  - A. 1) **같은 클래스 내부 메서드 호출** (프록시를 거치지 않음) 2) **private 메서드** (프록시가 접근 불가) 3) **final 클래스/메서드** (CGLIB 상속 불가) 4) **Checked Exception 발생** (기본은 롤백 안 됨)

- Q. **내부 호출 시 트랜잭션이 안 되는 이유는?**
  - A. 같은 클래스 내부 호출은 `this.method()` 형태로 **실제 객체를 직접 호출**합니다. **프록시를 거치지 않아** AOP가 적용되지 않습니다. 해결하려면 별도 클래스로 분리해야 합니다.

- Q. **Propagation 옵션 설명해보세요.**
  - A. **REQUIRED (기본)**: 기존 트랜잭션이 있으면 참여, 없으면 새로 생성. 하나의 트랜잭션으로 묶임 / **REQUIRES_NEW**: 항상 새 트랜잭션 생성, 기존 트랜잭션 일시 중단. 독립적으로 커밋/롤백 가능

---

### 6️⃣ 꼬리 질문 & 대응 포인트

- **꼬리 질문**: "프록시가 정확히 무엇인가요?"
  - **대응 논리**: 실제 객체(Target)를 감싸는 **대리 객체**입니다. 클라이언트는 프록시를 호출하고, 프록시가 **부가 기능(트랜잭션 등)을 실행한 후** 실제 객체의 메서드를 호출합니다. Spring은 Bean 생성 시 조건에 따라 프록시 객체를 생성합니다.

- **꼬리 질문**: "JDK Dynamic Proxy vs CGLIB 차이는?"
  - **대응 논리**: **JDK Dynamic Proxy**는 **인터페이스 기반**, 인터페이스가 없으면 사용 불가. **CGLIB**는 **클래스 상속 기반**, 인터페이스 없어도 되지만 final 클래스/메서드는 불가. Spring Boot는 **CGLIB를 기본**으로 사용합니다.

- **꼬리 질문**: "@Transactional을 Service 계층에 두는 이유는?"
  - **대응 논리**: 비즈니스 로직의 **논리적 경계**가 Service 계층에 있기 때문입니다. 여러 Repository 호출을 **하나의 트랜잭션으로 묶어야** 할 때가 많습니다. Controller는 웹 계층, Repository는 단순 데이터 접근 역할만 합니다.

- **꼬리 질문**: "REQUIRES_NEW는 언제 쓰나요?"
  - **대응 논리**: 메인 작업 실패 시에도 **독립적으로 커밋해야 할 작업**이 있을 때. 예: 에러 로그는 **메인 트랜잭션 롤백과 무관하게 반드시 저장**해야 하는 경우. 하지만 별도 커넥션을 사용하므로 **성능 저하** 주의

---

### 7️⃣ 실무 적용 or 가상 시나리오

- **실제/가정 상황**:
  - 전자상거래 주문 처리: 주문 생성 → 재고 차감 → 포인트 차감
  - 중간에 하나라도 실패하면 전체 롤백 필요

- **선택 이유**:
  - `@Transactional(propagation = Propagation.REQUIRED)` 사용
  - 모든 작업이 **하나의 트랜잭션**으로 묶여서 일관성 보장
  - 재고 차감 실패 시 → 주문 생성도 함께 롤백
  - 별도의 에러 로그 저장은 `REQUIRES_NEW`로 분리하여 **메인 실패와 무관하게 저장**

