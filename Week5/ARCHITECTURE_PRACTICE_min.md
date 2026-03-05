# Part A. 서버 설계

## 핵심 요구사항

**기존**: 모든 조직이 같은 메시지 업체 이용 </br>
**변경**: 조직별로 메시지 업체 선택해서 발송하게 하자
<br></br>
## 1. 작업 단계

### 단계 1: 요구사항 분석
- 목표: 조직별 메시지 업체 설정 기능 필요, 기존 발송 기능 유지
- 산출물: 요구사항 정리, 시스템 가정 리스트

### 단계 2: 데이터 모델 수정
메시지 발송 업체가 기존의 단일 업체가 아닌 새로운 업체가 추가되므로, api 키나 설정 등이 달라짐으로 인해 새로운 테이블이 추가적으로 필요할 것으로 판단.
- 목표: 조직별 메시지 업체 설정 정보 저장을 위한 추가 테이블 설계 (API KEY, 시크릿 키 등..)
- 산출물: 변경된 ERD 다이어그램, 데이터 스키마
  - **OrgMessageConfig 테이블 추가** (Organization 테이블과는 1:1 연관관계를 가짐)
  - organization_id, api_key, secret_key 등의 설정 정보 컬럼 추가

### 단계 3: 메시지 발송 구조 변경
- 목표: 조직별 메시지 발송 업체(MessageSender) 선택 가능
- 산출물: MessageSender 인터페이스 정의, Sender 구현체 추가 (Strategy 패턴 적용)
  - CloudMessageSender
  - SendTalkSender
 
**기존 구조**</br>
```java
MessageService -> CloudMessageClient 호출

public void sendMessage(MessageRequest req) {
    cloudMessageClient.send(req);
    messageHistoryRepository.save(req);
}

```


**변경 구조**</br>
현재는 클라우드메시지 외에 센드톡 업체를 추가한 구조이지만, 향후 신규 업체 추가 가능성을 고려하여 OCP 적용

> 전략 패턴 적용 이유 : 호출하는 객체의 변경없이(예: MessageService) 유연하게 확장하기 위해 전략 패턴을 적용하였다.</br>
> MessageService는 호출하는 메시지 발송 업체가 무엇이든 신경쓰지 않고 MessageSender 인터페이스만 호출해서 메시지를 발송하게 할 수 있다.</br>
> CloudMessageSender, SendTalkSender 모두 같은 인터페이스를 구현 -> MessageService 코드 수정 필요 없음
```
<<interface>>
MessageSender
+ send(request: MessageRequest): MessageResponse

CloudMessageSender        SendtalkSender
implements MessageSender  implements MessageSender
```

MessageService는 MessageSender 인터페이스만 알고 있음. </br>
MessageService 내부에서 cloudMessageClient.send(...) 같은 직접 호출은 사라짐

대신 MessageSender 구현체 내부에서 CloudMessageClient를 호출
```java
MessageService
  - MessageSender (인터페이스)
  - CloudMessageSender (구현체)
  - SendtalkSender (구현체)
```



### 단계 4: MessageService 수정
- 목표: 메시지 발송 업체별(Sender) 호출 구조 변경
- 산출물: send() 로직 수정
  - 조직별 Sender 선택
  - MessageSenderFactory 구현
    - 조직 ID를 매개로 받아서 Sender 객체 반환 (클라우드메시지인지, 센드톡인지)
  - 발송 후 MessageHistory 기록

```java
interface MessageSender {
    void send(MessageRequest req);
}

class CloudMessageSender implements MessageSender { ... }

class SendTalkSender implements MessageSender { ... }

class MessageSenderFactory {
    MessageSender getSender(Organization org) { ... }
}


@Service
public class MessageService {

    private final MessageSenderFactory senderFactory;

    public MessageService(MessageSenderFactory senderFactory) {
        this.senderFactory = senderFactory;
    }

    public void sendMessage(MessageRequest request) {
        // CloudMessageSender, SendTalkSender 중 선택
        MessageSender sender = senderFactory.getSender(request.getOrganizationId());
        // MessageService는 구체 구현 몰라도 됨
        sender.send(request);
        // 전송 정보 기록
        messageHistoryRepository.save(request);
    }
}
```

### 단계 5: 예외 처리
- 목표: 장애/설정 누락 대비
- 산출물:
  - Null 처리 -> 예외 처리 응답 반환
  - API 장애 -> retry, circuit breaker (재시도 및 서킷 브레이커로 일정 시간 차단 로직)
  - 이중 발송 방지 로직
  - 업체 변경 시 실시간 발송 중인 메시지의 누락 가능성
    - 메시지 발송을 비동기 큐(예: Kafka, RabbitMQ)로 처리 


## 2. 기술적 결정 사항 : 모델(스키마), API 등


목표 : 조직별 메시지 업체 설정 정보 저장을 하려고 한다. </br>
**대안 비교** </br>
- 1. Organization 테이블에 컬럼 추가 (API KEY, 시크릿 키 컬럼 추가) : 업체별로 필요한 설정값이 달라질 경우, 테이블 관리가 복잡해짐</br>
- 2. **OrgMessageConfig 테이블 분리** : 업체별 설정 정보가 다를 경우, 확장하기 편함

```sql
변경된 스키마
-- 기존
Organization (id, name, ...)

-- 신규
OrganizationSenderConfig (
  id,
  organization_id  (FK) -> Organization,
  message_type  ENUM(SMS, MMS, KAKAO)
  provider  ENUM(CLOUD_MESSAGE, SENDTALK),
  config    JSON, {"api_key": "...", ...} => 메시지 업체별 설정 정보 규격이 맞지 않는 경우가 많으니 JSON으로 저장하여 쓰는 방법도 있다고 합니다.
  created_at,
  updated_at
)
```

## 3. 리스크 및 대응



# Part B. 화면 설계



## 1. 화면 흐름



## 2. UI 결정 사항


## 3. 예외/엣지 케이스

