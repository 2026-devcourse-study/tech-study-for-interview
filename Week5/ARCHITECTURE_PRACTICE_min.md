# Part A. 서버 설계

## 핵심 요구사항

**기존**: 모든 조직이 같은 메시지 업체 이용 </br>
**변경**: 조직별로 메시지 업체 선택 가능
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

```
<<interface>>
MessageSender
+ send(request: MessageRequest): MessageResponse

CloudMessageSender        SendtalkSender
implements MessageSender  implements MessageSender
```

### 단계 4: MessageService 수정
- 목표: Provider 호출 구조 변경
- 산출물: sendMessage() 로직 수정
  - 조직별 Provider 선택
  - 발송 후 MessageHistory 기록



## 2. 기술적 결정 사항 : 모델(스키마), API 등


목표 : 조직별 메시지 업체 설정 정보 저장을 하려고 한다. </br>
**대안 비교** </br>
- 1. Organization 테이블에 컬럼 추가 (API KEY, 시크릿 키 컬럼 추가) : 업체별로 필요한 설정값이 달라질 경우, 테이블 관리가 복잡해짐</br>
- 2. **OrgMessageConfig 테이블 분리** : 업체별 설정 정보가 다를 경우, 확장하기 편함

```
변경된 스키마
-- 기존
Organization (id, name, ...)

-- 신규
OrganizationProviderConfig (
  id,
  organization_id  FK -> Organization,
  provider         ENUM('CLOUD_MESSAGE', 'SENDTALK'),
  config           JSON,  -- {"api_key": "...", ...}
  created_at,
  updated_at
)
```

## 3. 리스크 및 대응



# Part B. 화면 설계



## 1. 화면 흐름



## 2. UI 결정 사항


## 3. 예외/엣지 케이스

