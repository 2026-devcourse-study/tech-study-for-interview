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
> 각 Sender 내부에서 업체별 Client를 호출한다.
 

### 단계 4: MessageService 수정
- 목표: 메시지 발송 업체별(Sender) 호출 구조 변경
- 산출물: send() 로직 수정
  - 조직별 Sender 선택
  - MessageSenderFactory 구현
    - 조직 ID를 매개로 받아서 Sender 객체 반환 (클라우드메시지인지, 센드톡인지)
  - 발송 후 MessageHistory 기록


### 단계 5: 예외 처리
- 목표: 장애/설정 누락 대비
- 산출물:
  - Null 처리 -> 예외 처리 응답 반환
  - API 장애 -> retry, circuit breaker (재시도 및 서킷 브레이커로 일정 시간 차단 로직)
  - 이중 발송 방지 로직
  - 업체 변경 시 실시간 발송 중인 메시지의 누락 가능성
    - 메시지 발송을 비동기 큐(예: Kafka, RabbitMQ)로 처리 

<br></br>
## 2. 기술적 결정 사항 : 모델(스키마), API 등

### 1. 모델(스키마) 결정 사항
목표 : 조직별 메시지 업체 설정 정보 저장을 하려고 한다. </br>
**대안 비교** </br>
- 1번 - Organization 테이블에 컬럼 추가 (API KEY, 시크릿 키 컬럼 추가) : 업체별로 필요한 설정값이 달라질 경우, 테이블 관리가 복잡해짐</br>
- 2번 - **OrgMessageConfig 테이블 분리** : 업체별 설정 정보가 다를 경우, 확장하기 편함

**2번 대안 선택**

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
### ❗중요: 클라우드메시지와 센드톡 API 문서의 차이 보완
> config 컬럼의 경우, 업체별로 필요한 인증 정보 값과 개수가 다르기 때문에 고정 컬럼보다는 스키마 변경 없이 확장할 수 있도록 JSON 컬럼이 적합하다고 판단.<br>
> CloudMessage용: {"api_key": "cloud_sk_..."}<br>
> SendTalk용: {"api_token": "sendtalk_at_...", "secret_key": "..."}<br>
> 클라우드메시지(Header 인증)와 센드톡(Body 인증)의 API 명세 차이가 있음. <br>

<br></br>
**MessageHistory 이력 저장용 테이블에도 provider_type 컬럼 추가**
```sql
MessageHistory(
  id,
  organization_id (FK) -> Organization,
  message_type  ENUM(SMS, MMS, KAKAO),
  provider_type ENUM(CLOUD_MESSAGE, SENDTALK),
  message_content,
  status,
  created_at
)
```

<br></br>

### 2. 전송 구조 결정 사항

MessageService 수정 없이 신규 업체 추가가 가능하도록, OCP를 적용하여 구현

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
<br></br>



MessageService는 MessageSender 인터페이스만 알고 있음. </br>
MessageService 내부에서 cloudMessageClient.send(...) 같은 직접 호출은 사라짐

대신 MessageSender 구현체 내부에서 CloudMessageClient를 호출
```java
MessageService
  - MessageSender (인터페이스)
  - CloudMessageSender (구현체)
  - SendtalkSender (구현체)
```


<img width="355" height="440" alt="스크린샷 2026-03-05 오후 6 46 06" src="https://github.com/user-attachments/assets/f14ffab2-194e-4b68-a56d-2f9746e772d5" />




<br></br>
### 3. 업체별 메시지 전송 로직 호출 (대안 비교)
- 1번 - MessageService에서 직접 호출
- 2번 - 각 Sender 구현체 내부에서 Client 호출

**2번 선택**

**1번 경우**
```java
MessageService -> CloudMessageClient/SendTalkClient 직접 호출하는 방식
```
로직이 간단하는 장점이 있으나, 아래와 같은 단점으로 인해 유지보수가 어려워짐.
- MessageService가 모든 업체 API를 알아야 함으로 결합도가 높아짐
- 새로운 업체 추가 시 MessageService 코드 수정 필요 -> OCP 위반

**2번 경우**
```java
MessageService -> MessageSender 인터페이스 호출 -> Sender 구현체 내부에서 Client 호출
```
- 전략 패턴을 적용하여 MessageService는 인터페이스만 의존하게 됨
- 새로운 업체 추가 시 MessageService 수정 불필요
- OCP 준수: 확장에는 열려 있고, 변경에는 닫혀 있음

MessageService는 인터페이스만 의존, 확장성/유지보수성 확보

> 그런데, 이러한 설계는 메시지 전송 업체가 확장될때마다 Client 코드를 새로 만들어야 하는 구조이다.<br>
> 이러한 비효율을 개선하기 위해 추후에는 추상 클래스를 활용한 템플릿 메서드를 도입하여 공통 로직을 분리하는 것이 유지보수 측면에서 좋을 것으로 예상한다.

<br></br>

### 4. 메시지 전송 업체 선택하는 로직 (Factory 기반)
- 1번 - if-else 기반
- 2번 - Factory 구현, Map/Bean 기반 자동 등록

#### Factory를 적용하기로 한 이유
- Factory : 객체 생성 책임을 한 곳으로 모아두는 패턴
- 누가 어떤 Sender를 쓸지 결정하고 객체를 반환하는 생성 책임을 Factory에서 담당 (조직별로 어떤 Sender를 선택해야 하는지 결정하기 위해)
- 장점: 객체 생성 로직을 분리 -> MessageService 같은 호출자는 생성 방식에 신경 쓸 필요가 없다.

**2번 선택**

**기존 1번의 경우**
```java
if (org.getProviderType() == CLOUD) {
    cloudMessageClient.send(msg);
} else if (org.getProviderType() == SENDTALK) {
    sendTalkClient.send(msg);
}
```
- Factory에서 Sender를 if-else로 반환하면, 새로운 업체마다 Factory 코드 수정 필요
- 조직이 많아지면 if-else문이 더 복잡해짐

**2번**

1. 업체별 구현체 등록 **(동적 빈(Bean) 주입 방식)**
각 구현체에 @Component 이름을 명시적으로 부여하여 빈 등록
```java
// 인터페이스
public interface MessageSender {
    void send(MessageRequest request, OrganizationMessageConfig config);
    boolean supports(ProviderType providerType); // 어떤 업체를 담당하는지 반환
}

// 클라우드메시지 구현체
@Component("CLOUD_MESSAGE")
public class CloudMessageSender implements MessageSender { ... }

// 센드톡 구현체
@Component("SENDTALK")
public class SendTalkSender implements MessageSender { ... }
```
<br></br>
2. 전송 전략 팩토리
MessageService가 직접 고민하게 하지 말고, MessageSender 업체를 찾아주는 함수 생성

```java
@Component
public class MessageSenderFactory {
    // Spring이 모든 MessageSender 구현체를 Map으로 주입해줌 (Key: Bean Name)
    private final Map<String, MessageSender> senders;

    public MessageSender getSender(ProviderType providerType) {
        MessageSender sender = senders.get(providerType.name());
        if (sender == null) {
            throw new IllegalArgumentException("지원하지 않는 업체입니다.");
        }
        return sender;
    }
}
```

3. MessageService에서의 활용
이제 MessageService는 어떤 업체가 있는지 몰라도 됨
```java
@Service
@RequiredArgsConstructor
public class MessageService {
    private final OrganizationMessageConfigRepository configRepository;
    private final MessageSenderFactory senderFactory;

    public void sendMessage(Long orgId, MessageRequest request) {
        // 해당 조직의 설정 정보를 DB에서 조회
        OrganizationMessageConfig config = configRepository.findByOrganizationId(orgId);
        
        // 설정에 명시된 provider에 맞는 구현체를 팩토리에서 가져옴
        MessageSender sender = senderFactory.getSender(config.getProvider());
        
        // 발송 (구현체가 무엇이든 send()만 호출하면 됨)
        sender.send(request, config);
    }
}
```
- Spring Bean 이름 자동 주입
- Map<String, MessageSender> 등록 후 key로 선택 -> Factory 수정 없이 확장 가능
- 새로운 업체가 계속 추가되어도 Factory나 MessageService 코드 수정 필요없이 MessageSender 구현체 추가만 하면 됨 -> OCP 준수


<br></br>



### 5. 메시지 업체 API의 인증 방식과 API 요청 스펙이 다를 경우
> 외부 메시지 업체 API의 요청 스펙이 서로 다르므로<br>
> 내부 공통 DTO(MessageRequest)를 정의하고,<br>
> 각 Sender 구현체에서 업체별 API 요청 DTO로 변환하도록 설계<br>

### 내부 공통 DTO
```java
class MessageRequest {
    String phoneNumber;
    String message;
}
```
### 업체별 요청 DTO
```java
class CloudMessageRequest {
    String phone;
    String message;
    String apiKey;
}
```
```java
class SendTalkRequest {
    String receiver;
    String text;
    String accessToken;
}
```

따라서 업체별 MessageSender 구현체에서는 아래와 같이 구현된다.
#### CloudMessageSender (Header 인증)
#### SendTalkSender (Body 인증)
```java
@Component
public class CloudMessageSender implements MessageSender {

    private final CloudMessageClient client;

    public CloudMessageSender(CloudMessageClient client) {
        this.client = client;
    }

    @Override
    public MessageResponse send(MessageRequest request,
                                ProviderConfig config) {

        CloudMessageApiRequest apiRequest =
                new CloudMessageApiRequest(
                        request.getPhoneNumber(),
                        request.getMessage()
                );

        return client.send(
                apiRequest,
                config.getApiKey(),
                config.getSecretKey()
        );
    }
}
```

<br></br>

## 3. 리스크 및 대응

### 1. API 장애 (업체 서버 다운, 네트워크 문제)
영향: 모든 조직/사용자에게 메시지 발송 실패
대응 방안:
  - 재시도 전략
  - Circuit breaker 패턴 적용 :
    - 특정 업체의 에러율이 임계치를 넘으면 'Open' 상태가 되어 해당 업체로의 요청을 즉시 차단하고 에러를 반환하거나(사용자 경험 위해 '서비스 점검 중' 문구 표시),Fallback으로 전환하여 예외 처리하도록 설계
    -  API Timeout
      - 외부 메시지 API 응답 지연 시 타임아웃 설정


### 2. 응답 지연 / 타임아웃
영향: 실시간 메시지 발송 지연
- 타임아웃 설정
- 비동기 큐로 발송 처리 (Kafka, RabbitMQ)
- 타임아웃 시 재시도 (비동기 처리와 DLQ)
  - DLQ의 역할: 재시도 횟수를 초과한 메시지는 버리지 않고 'DLQ'라는 별도의 저장소로 보내어 수동으로 다시 처리하거나 고객에게 실패 알림 표시

### 3. 메시지 업체 변경 직후 발송 중 메시지 누락
영향: 변경 시점의 발송 메시지 일부 누락
문제 상황 : 
```
관리자가 설정을 '업체 A'에서 '업체 B'로 바꾸는 0.1초 사이에 대량의 메시지 발송 요청이 들어온다면? 
큐에 쌓인 메시지들이 처리될 때 DB를 다시 조회하면, 어떤 건 A로, 어떤 건 B로 발송되는 혼란이 생길 수 있음.
```
- 변경 트랜잭션 적용
- 메시지 발송을 비동기 큐(예: Kafka, RabbitMQ)로 처리

**상세 해결 방안**: 비동기 메시지 큐를 사용한다고 가정(Kafka, RabbitMQ 등)
1. 메시지 발행(Producer) 시점: DB에서 현재 조직의 OrgMessageConfig를 조회
2. 페이로드(Payload) 구성: 메시지 본문뿐만 아니라, 사용할 Sender 이름과 API Key 등을 포함한 전송 객체 생성
3. 큐 저장: 이 객체를 Kafka/RabbitMQ에 전달
4. 컨슈머(Consumer) 실행: 큐에서 메시지를 꺼낸 워커는 다시 DB를 조회하지 않고, 메시지 객체 안에 들어있는 설정 정보를 그대로 사용하여 발송 API를 호출
=> 설정이 변경되더라도 이미 큐에 들어간 작업들은 발행 당시의 유효한 설정으로 안전하게 끝까지 처리된다.

<br></br>



## 결론
```
외부 메시지 업체는 인증 방식(Header, Body)과 요청 스펙이 서로 다르므로,
내부 공통 DTO(MessageRequest)와 인터페이스(MessageSender)를 정의하고 업체별 Sender 구현체에서 API 요청 변환 및 인증 처리를 담당하도록 설계
```

## ERD 정리
![JPEG 이미지-45A6-85F7-DE-0](https://github.com/user-attachments/assets/94d9b116-29e4-433c-bd46-db12240d86bd)



# Part B. 화면 설계

[대량 문자 발송 서비스 - SOLAPI](https://console.solapi.com/purplebook) 를 참고하였습니다.
## 1. 화면 흐름

1. 메시지 전송 관리자 화면 진입
   - 현재 설정된 업체 정보 및 발송 통계 요약 노출
   - 발송량 통계 노출

2. 사용자(관리자)가 메시지 발송 업체 등록
   - 제휴된 업체(클라우드메시지/센드톡) 중에서 선택
   - 서비스 제공 업체에서 필요로 하는 값 입력 (API Key, Secret 키, 발신번호 등)
   - 유효성 검증
   - 등록 성공 시 확인 팝업 표시
   
5. 사용자(관리자)가 메시지 발송 업체 선택
   - 이용할 업체(클라우드메시지/센드톡) 선택
   - SMS/MMS/알림톡(카카오) 메시지 유형 선택
   - 메시지 수신번호 입력 (주소록, Excel, 직접 입력 등 선택지 버튼 제공)
   
6. 메시지 입력
   - 유효성 검증 및 가이드 텍스트 제공
   - 6-1. SMS/MMS 선택
   - 제목, 메시지 입력 후
   - 6-2. 알림톡(카카오) 선택
   - 연동된 카카오 채널 선택
   - 알림톡 템플릿 추가 or 선택
   - 내용 입력
     
7. 연동 테스트 확인
  - '연동 테스트' 버튼 선택
  - 최초에 등록한 서비스 제공 업체 키로 실제 업체 API를 호출하여 유효성 즉시 확인

8. 메시지 전송
  - 전송 처리 직전, 주의사항(예약 메시지 처리 등)을 담은 확인 팝업 노출
  - 메시지 전송 완료 알림 및 감사 이력(Audit Log) 기록


### 고려 사항
고객 조직이 업체 계정을 갖는 모델이 실무에서 더 많이 적용된다고 알고 있습니다.

> 예: 각 회사가 자기 SendTalk 계정을 가지고 있음 <br>
> 우리 서비스는 그 계정을 연결만 해주는 방식. 그래서 최초에 사용자가 해당 메시지 업체의 API Key 정보를 입력하는 것
<br></br>

## 2. UI 결정 사항

1. 업체 선택/등록 방식
   - 드롭다운 vs **카드형 UI**
   - **카드형 UI**의 경우, 각 업체의 특징(아이콘, 단가, 주요 기능)을 한눈에 비교하고 명확하게 선택할 수 있는 장점이 있음.
  
2. 전체 필드 노출 vs 동적 필드 노출
   - **동적 필드 노출** : 업체마다 요구하는 값(어떤 곳은 Secret Key가 필수, 어떤 곳은 아님)이 다르므로, 혼란을 방지하기 위해 선택한 업체의 필드만 노출
   - **유효성 검사** : 각 입력 필드에 해당하는 유효성 검사를 충족하지 못하면, 경고 표시글과 함께 유효성 오류가 발생했음을 UI로 표시
   - API Key는 민감 정보이므로 기본 마스킹 처리. 필요시 보게끔 마스킹 해제 아이콘 추가

## 3. 예외/엣지 케이스

1. 유효하지 않은 API Key 입력
   - 메시지 업체 등록 실패 시, 에러 메시지(예: "인증 실패 - 인증 정보가 일치하지 않습니다.") 에러 메시지와 함께 해당 필드를 붉은색으로 강조
  
2. 네트워크 타임아웃
   - 업체 API 응답 기다리는 동안, 로딩 화면 표시
   - 응답 실패 시, "현재 메시지 업체 서버가 응답하지 않습니다. 잠시 후 다시 시도해주세요."라는 토스트 메시지 노출

3. 업체 변경 시 예약 메시지 존재
   - "현재 n건의 예약 메시지가 있습니다. 업체 변경 후에도 기존 업체로 발송됩니다."라는 안내 팝업 제공

4. 전송 시 필수 정보 확인사항
   - 발신번호 미등록
   - 충전된 이용 요금 잔액 부족 확인
   - API Key 등록 여부
  
5. 관리자 권한 없는 사용자가 변경 시도
   - 예를 들어, 일반 사용자가 업체 설정 변경 시도한다면 "조직 관리자만 사용할 수 있습니다."라는 토스트 메시지 표시
  
6. 메시지 서비스 업체 API 등록 시
   - API 등록 이후, 연동 테스트 버튼을 통해 "연결 테스트" 하도록 유도
   - "연결 테스트"가 완료되어야 해당 메시지 서비스 업체를 등록할 수 있도록 제한
