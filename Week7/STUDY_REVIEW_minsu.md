## 📌 주제별 학습 정리 (개인 작성)

---

### 📍 Topic - Kafka

---

### 1️⃣ 핵심 개념 한 줄
Kafka는 대용량 이벤트 데이터를 분산 환경에서 안정적으로 저장하고 전달하는 메시지 스트리밍 플랫폼이다.

---

### 2️⃣ 동작 원리 요약
1. Producer가 Topic에 메시지를 발행
2. Topic은 여러 Partition으로 나뉘어 병렬 처리됨
3. 각 메시지는 Offset으로 순서 관리됨
4. Consumer는 Consumer Group 단위로 메시지를 가져감
5. 같은 그룹 내 Consumer들은 Partition을 나눠 처리
6. 메시지는 즉시 삭제되지 않고 일정 기간 저장됨 (로그 기반)

---

### 3️⃣ 핵심 키워드
- Topic / Partition / Offset
- Consumer Group
- At least once / Exactly once
- 메시지 순서 보장 (Partition 단위)
- 이벤트 기반 아키텍처 (EDA)
- Producer / Consumer
- Outbox Pattern
- 비동기 처리
- High Throughput / Scalability

---

### 4️⃣ 주의 포인트
- Partition 단위로만 순서 보장됨 (전체 순서 ❌)
- At least once로 인해 메시지 중복 처리 필요 (idempotent)
- 트랜잭션(DB)과 함께 사용할 때 데이터 불일치 발생 가능 → Outbox 패턴 필요
- 운영 복잡도 높음 (모니터링, 장애 대응)
- Consumer offset 관리 실패 시 메시지 유실/중복 발생 가능

---

### 5️⃣ 자주 나오는 면접 질문 (선택)
- Kafka를 사용하는 이유는 무엇인가요?
- Partition을 나누는 이유는 무엇인가요?
- 메시지 순서는 어떻게 보장되나요?
- 메시지 중복은 어떻게 처리하나요?
- Kafka의 단점은 무엇인가요?
- RabbitMQ와 차이점은 무엇인가요?
- 결제 시스템에서 Kafka를 사용할 수 있나요?
- 이벤트 유실이 발생하면 어떻게 대응하나요?

---
