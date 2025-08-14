# 루트 계정 로그인 알림

---

**[ 목차 ]**

---

## **[ 시나리오 안내 ]**

| 내용 | 루트 계정은 AWS 계정 생성 시에만 활용되고 
IAM User등으로 관리되어 로그인 횟수가 적어집니다.
또한 모든 권한을 가지고 있기 때문에 실제 운영 환경에서 사용이 최소화되어야 합니다.
루트 계정으로 콘솔 로그인이 발생한 경우 실시간 탐지를 통해
이상 행위 여부를 판단하는 시나리오를 구현해 봅니다. |
| --- | --- |
| 사용 서비스 | CloudTrail, CloudWatch, EventBridge Rule, SNS, Lambda |
| 탐지 조건 | 웹 콘솔에 로그인하는 이벤트 중 루트 사용자인 경우를 탐지한다. |
| 알림 방식 | SNS를 통해 Email로 알림을 전송하고,
동시에 Lambda 함수를 활용해 Webhook을 통해 Discord로도 알림을 전송한다. |
| 대응 | 알림으로 내용을 확인하고 사용자가 대처한다. |

### **실습 개요**

본 실습은 루트 계정의 AWS 콘솔 로그인 이벤트를 실시간으로 탐지하고, 이를 이메일(SNS)과 Discord(Webhook)를 통해 즉시 알림으로 전송하는 시나리오를 구현합니다. CloudTrail과 EventBridge를 활용해 이벤트를 감지하고, Lambda를 통해 다중 채널로 알림을 전달합니다.

### **학습 목표**

- 루트 계정 로그인 탐지 시나리오 설계를 통해 보안 사고 대응 체계 학습한다.
- AWS의 Event 기반 자동화 구조 (CloudTrail → EventBridge → Lambda/SNS) 이해한다.
- 실시간 알림 시스템 구축 및 외부 채널(Discord) 연동 방법 습득한다.

## **[ 시나리오 전체적인 흐름 ]**

![image.png](image.png)

| **AWS Service** | **Service Purpose** | **Workbook Usage** |
| --- | --- | --- |
| **CloudTrail** | AWS 계정 내 API 호출 및 로그인 활동을 기록하는 서비스 | 루트 계정의 콘솔 로그인 이벤트를 감지하기 위한 로그 데이터 수집 |
| **EventBridge** | 이벤트 패턴에 따라 자동으로 트리거를 발생시키는 서비스 | 루트 로그인 이벤트가 감지되면 지정된 Lambda와 SNS를 호출하도록 이벤트 룰 설정 |
| **SNS (Simple Notification Service)** | 메시지를 여러 구독자에게 전달하는 메시징 서비스 | 루트 로그인 알림을 Email로 전송 |
| **Lambda** | 서버 없이 코드를 실행할 수 있는 이벤트 기반 컴퓨팅 서비스 | 루트 로그인 이벤트 발생 시 Discord Webhook으로 메시지 전송 |
| **S3** | 객체 기반 스토리지 서비스 | CloudTrail 로그 저장소로 활용 |

---

### 참고 사항

<aside>

**< 주의사항 >** 

“루트 계정 로그인 알림 시나리오”실습 시, region을 **서울로 하면 안 되고 반드시 버지니아 북부(미국 동부, `us-east-1`)로** 설정해주어야 한다. 아래의 AWS의 내부 동작 원리와 글로벌 이벤트 처리 방식을 통해 이유를 확인해 보자.

---

**< 상세 설명 >**

루트 계정 로그인 이벤트는 "글로벌 서비스 이벤트"로 분류되며,
EventBridge에서 이를 감지하려면 반드시 `us-east-1` region에서 규칙을 만들어야 하기 때문

1. 루트 로그인 이벤트는 **글로벌 서비스 이벤트**

- `signin.amazonaws.com` → 전 세계 공통의 로그인 서비스
- 즉, 로그인 이벤트는 **특정 region에 귀속되지 않음** (서울에서 로그인해도, region은 글로벌)

2. EventBridge는 글로벌 이벤트를 **특정 region에서만** 수신 가능

- AWS 내부적으로 **글로벌 이벤트는 `us-east-1(버지니아 북부)`에서만 EventBridge를 통한 규칙 매칭이 가능**
- 서울 region (ap-northeast-2)에서 EventBridge 규칙을 만들어도 **`signin.amazonaws.com` 글로벌 이벤트는 매칭되지 않음** → **실패 원인**

3. CloudTrail은 서울에서도 글로벌 이벤트를 수집할 수는 있음

- CloudTrail에서 **"글로벌 서비스 이벤트 포함"** 옵션이 활성화되어 있다면,
- 서울 region에서도 S3로 로그인 이벤트 기록은 남길 수 있음
- 하지만 **EventBridge 규칙이 글로벌 이벤트와 매칭 되지 않으면, Lambda 트리거로 설정 X**

---

**< 요약 >**

| **설정 항목** | **올바른 값** | **이유** |
| --- | --- | --- |
| EventBridge 규칙 생성 region | us-east-1 (버지니아) | 글로벌 이벤트 수신 가능 |
| CloudTrail region | 서울도 가능 | 단, 글로벌 이벤트 포함 체크 필수 |
| Lambda 함수 region | us-east-1 (EventBridge와 동일 리전) | 트리거 연결 위해 동일 리전 필수 |
1. CloudTrail → 서울 리전 + 글로벌 이벤트 포함
2. EventBridge → **us-east-1**에서 `"userIdentity.type": "Root"` 조건의 규칙 생성
3. Lambda → us-east-1에 배포 + 해당 규칙과 연결
4. 루트 계정 로그인 → 이벤트 발생 → us-east-1의 EventBridge가 감지 → Lambda 트리거됨 → Discord알림
5. 루트 계정 로그인 → 이벤트 발생 → us-east-1의 EventBridge가 감지 → SNS 서비스로 연결 → 설정된 이메일로 알림

---

**< 결론 >**

- 루트 로그인은 글로벌 이벤트 → EventBridge는 **us-east-1에서만 감지 가능**
- 실습 시 **EventBridge와 Lambda를 us-east-1에 구성해야 알림이 정상 작동**
- CloudTrail은 서울에 두어도 무방하나, 글로벌 이벤트 포함 옵션이 켜져 있어야 함
</aside>

**[ 콘솔 리소스명 ]**

| **리소스 종류** | **리소스명 설정** | **목적** |
| --- | --- | --- |
| **S3 Bucket** | **`s3-root-login-detect`** | CloudTrail 로그 저장소로 사용하여 root 로그인 이벤트 기록 |
| **Lambda Function** | **`lambda-root-login`** | EventBridge 이벤트 트리거 시 root 로그인 이벤트 처리 및 SNS/Discord 전송 |
| **CloudTrail** | **`ct-trail-monitor`** | AWS 계정의 root 로그인 이벤트를 감지하기 위한 로그 수집 설정 |
| **EventBridge** | **`eventbridge-root-login-pattern`** | root 로그인 이벤트 패턴 감지를 위한 이벤트 규칙 설정 |
| **SNS Topic** | **`sns-root-login-alarm`** | 이메일 등의 경고 채널로 root 로그인 알림 전송 |
| **Discord 채널** | **`root-login-alarm`** | Lambda에서 전송한 root 로그인 알림 수신용 채널 |

## **[ 시나리오 상세 구현 과정 ]**

### **1. Lambda 함수 생성**

**STEP 2) Lambda 함수 생성**

![image.png](image%201.png)

<aside>

서버 없이 이벤트 발생 시 자동으로 코드를 실행하기 위해 AWS 콘솔에서 **Lambda서비스**로 이동한다.

</aside>

![image.png](image%202.png)

<aside>

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.

</aside>

**[ 함수 생성 ]**

![image.png](image%203.png)

<aside>

- **Author from scratch** 선택
- **Function name** : `lamda-root-login`
- **Runtime** : Python 3.13
- **Architecture** : x86_64
</aside>

**[ 생성된 함수 확인 ]**

![image.png](image%204.png)

<aside>

정상적으로 Lambda함수가 생성되었는지 확인해준다.

</aside>

### **2. S3 버킷 및 CloudTrail 추적 생성**

**STEP 1) S3 검색**

![image.png](698faf0f-1b59-4343-a75f-392bdd8b0d08.png)

<aside>

Cloudtrail 로그를 저장할 버킷을 만들기 위해 **S3 서비스로 이동**한다.

</aside>

**STEP 2) S3 bucket 생성**

[ **S3 bucket 생성 ]**

![image.png](image%205.png)

<aside>

**S3** 서비스 화면 오른쪽 상단의 **Create a bucket**버튼을 클릭한다.

</aside>

**[ bucket 속성 선택 ]**

![image.png](image%206.png)

![스크린샷 2025-06-30 163654.png](%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2025-06-30_163654.png)

![스크린샷 2025-06-30 163705.png](%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2025-06-30_163705.png)

<aside>

- **Bucket name :**  **`s3-root-login-detect`**
- **Object Ownership :** ACLs disabled (recommended)
- **Block Public Access settings for this bucket :** Block all public access
- **Bucket Versioning :** Enable
- **Encryption type :** Server-side encryption with Amazon S3 managed keys (SSE-S3)
</aside>

**STEP 3) CloudTrail 검색**

![image.png](image%207.png)

<aside>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 **CloudTrail서비스**로 이동한다. 

</aside>

**STEP 4) CloudTrail 생성**

![image.png](image%208.png)

<aside>

**Create trail**버튼을 클릭해 사용할 추적을 생성한다.

</aside>

**[ 추적 속성 선택 ]**

![image.png](image%209.png)

<aside>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다. 

- **Trail name** : **`ct-trail-monitor`**
- **Storage location :** Use existing S3 bucket (**Browe**를 클릭해 앞서 생성한 버킷 선택)
- **Additional settings**
    - **Log file validation :** Enabled
</aside>

**[ 로그 이벤트 선택 ]**

![image.png](image%2010.png)

<aside>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

- **Events** : Management events
- **Management events - API activity :** Read, Write
</aside>

**[** **검토 및 생성 ]**

![image.png](image%2011.png)

<aside>

각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.

</aside>

**STEP 3) 추적 생성 확인**

![image.png](image%2012.png)

<aside>

대시보드에서 정상적으로 추적이 생성되었는지 확인한다.

</aside>

### **3.** EventBridge 규칙 생성

**STEP 1) EventBridge 검색**

![image.png](image%2013.png)

<aside>

Lambda 함수를 주기적으로 실행하기 위해 AWS 콘솔에서 **EventBridge 서비스**로 이동한다.

</aside>

**STEP 2) EventBridge 생성**

![image.png](image%2014.png)

<aside>

**EventBridge** 서비스 화면 오른쪽 상단의 **EventBridge Rule**을 선택하고 **Create rule**버튼을 클릭한다.

</aside>

**[ 상세 규칙 설정 ]** 

![image.png](image%2015.png)

<aside>

- **Name** : **`eventbridge-root-login-pattern`**
- **Event bus :** default
- **Rule type** : Rule with an event pattern
</aside>

**[ 이벤트 패턴 작성 ]**

![image.png](image%2016.png)

<aside>

 탐지할 이벤트 조건을 설정을 설정하고 **Next**버튼을 클릭한다.

- **Events :** Other
- **Event pattern** : Custom pattern (JSON editor)

```json
{
    "detail-type": ["AWS Console Sign In via CloudTrail"],
    "detail": {
      "userIdentity": {
        "type": ["Root"]
      },
      "eventName": ["ConsoleLogin"]
    }
}
```

**[ 설정한 이벤트 안내 ]**

| **이벤트 이름** | **설명** | **탐지 목적** |
| --- | --- | --- |
| **`ConsoleLogin`** |  |  |
</aside>

**[ 대상 선택 ]**

![image.png](image%2017.png)

<aside>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

- **Target types:** AWS service
- **Select a target:** Lambda function
- **Target location:** Target in this account
- **Topic:**  앞서 생성한 Lambda function 선택
</aside>

**[ 태그 구성 (선택) ]**

![image.png](image%2018.png)

<aside>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.

</aside>

**[ 검토 및 생성 ]**

![image.png](image%2019.png)

<aside>

설정 내용 최종 확인 후  **Create rule**버튼을 클릭한다.

- status - **enabled** 확인
</aside>

**STEP 3) 생성된 규칙 확인**

![image.png](image%2020.png)

<aside>

규칙이 정상적으로 생성되었는지 확인해준다.

</aside>

### 4. SNS 주제 생성 및 구독 설정

**STEP 1) SNS 검색**

![image.png](image%2021.png)

<aside>

알람을 전송 받을 주제 및 구독을 생성하기 위해 **SNS 서비스**로 이동한다.

</aside>

**STEP 2) 주제 생성**

![image.png](image%2022.png)

<aside>

좌측 탭에서 Topic으로 이동 후 **Create** topic 버튼을 클릭한다.

</aside>

![image.png](image%2023.png)

<aside>

- **Type** : Standard
- **Name** : **`sns-root-login-alarm`**
</aside>

**STEP 3 ) 구독 생성**

![image.png](image%2024.png)

<aside>

생성된 주제 확인 후 **Create subscription**을 누른다.

</aside>

**[ 구독 생성 - 세부사항 ]**

![image.png](image%2025.png)

<aside>

- **Protocol** : email
- **Endpoint** : 알람 받을 이메일 주소
</aside>

**STEP 4 ) 구독한 이메일 인증**

![image.png](image%2026.png)

<aside>

설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다.
이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.

</aside>

![image.png](image%2027.png)

<aside>

**Confirm subscription**을 눌러 접속하면 정상적으로 SNS 구독 등록이 된 것이다.

</aside>

### **5. Discord  연결**

**STEP 1) Discord 채널 생성 및 WebHook 설정**

**[ 채널 만들기 ]**

![image.png](image%2028.png)

<aside>

이벤트에 관한 알림을 수신 할 **채널**을 **생성**해준다. 

- **채널 이름** : root-login-alarm
</aside>

**[ 채널 편집 ]**

![image.png](image%2029.png)

<aside>

위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.

</aside>

**[ 웹후크 연동 ]**

![image.png](image%2030.png)

<aside>

왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.

</aside>

**[ 웹후크 URL 복사 ]**

![image.png](image%2031.png)

<aside>

**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 가져온다.

- **이름** : WEBHOOK_URL
- **채널** : # (앞서 생성한 채널 이름 선택)
</aside>

### 6. Lambda 추가 설정

**STEP 1) 환경 변수 편집** 

![image.png](09cf90a2-36a6-4dde-8cdc-a4b633e0670f.png)

<aside>

이후 Configuration → Environment variables로 들어가서 **Edit** 버튼을 클릭한다.

</aside>

**[ 환경 변수 추가 ]**

![image.png](image%2032.png)

<aside>

Edit environment variables로 이동하여 **Add environment variables** 버튼을 클릭한다.

</aside>

**[ 환경 변수에 키와 값 추가 ]**

![image.png](image%2033.png)

<aside>

**Key, Value**를 ****다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

이렇게 환경 변수 지정을 통해 Discord Hook 봇을 **호출할 수 있고** 환경 변수를 사용함으로써 소스 코드에서 URL의 **노출을 방지**할 수 있다.

- **Key, Value는 표를 참고**

| Key | **용도/설명** | Value |
| --- | --- | --- |
| DISCORD_WEBHOOK_URL | 디스코드 알림용 Webhook URL | https://discord.com/api/webhooks/ |
</aside>

**STEP 2) Lambda 소스 코드 편집**

![image.png](image%2034.png)

<aside>

Code탭에서 **Lambda python 코드**를 작성 후 **Deploy**버튼을 클릭하여 배포해 준다. 

</aside>

```python
import json
import logging
import os
from datetime import datetime, timedelta
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError

# 환경변수에서 Discord 웹훅 URL 가져오기
webhook_url = os.environ.get('DISCORD_WEBHOOK_URL')
if not webhook_url:
    print("환경변수 DISCORD_WEBHOOK_URL이 설정되어 있지 않습니다.")
    return {'statusCode': 500, 'body': 'Webhook URL not set'}

# 로깅 설정
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info("Received event: " + json.dumps(event))

    try:
        data = event['detail']

        # 사용자 구분
        user_type = data['userIdentity']['type']
        user = "Root" if user_type == "Root" else data['userIdentity'].get('userName', 'Unknown')

        # 로그인 시간 처리 (UTC → KST)
        login_time_utc = data['eventTime'][:19]
        login_time_kst = datetime.strptime(login_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)

        ip = data['sourceIPAddress']
        mfa = data['additionalEventData'].get('MFAUsed', 'Unknown')
        result = data['responseElements'].get('ConsoleLogin', 'Unknown')

        # 디스코드 메시지
        discord_message = {
            "content": f" **[{user_type} AWS 콘솔 로그인 탐지 ]**\n"
                       f" 사용자: {user}\n"
                       f" 시간: {login_time_kst.strftime('%Y-%m-%d %H:%M:%S')} (KST)\n"
                       f" IP: {ip}\n"
                       f" MFA 사용: {mfa}\n"
                       f" 로그인 결과: {result}"
        }

        # Discord Webhook 요청 (403 우회를 위한 User-Agent 추가)
        req = Request(
            HOOK_URL,
            data=json.dumps(discord_message).encode('utf-8'),
            headers={
                'Content-Type': 'application/json',
                'User-Agent': 'Mozilla/5.0 (Lambda Discord Notification)'
            }
        )

        response = urlopen(req)
        response.read()
        logger.info("Discord 메시지 전송 성공")

    except HTTPError as e:
        logger.error(f"HTTP 오류: {e.code} - {e.reason}")
    except URLError as e:
        logger.error(f"연결 실패: {e.reason}")
    except Exception as e:
        logger.error(f"예외 발생: {str(e)}")

```

**STEP 3) Lambda 트리거 추가**

**[ Lambda 트리거 - EventBridge(CloudWatch Events) ]**

![image.png](image%2035.png)

<aside>

생성한 Lambda함수의 다이어그램 왼쪽 하단의 **Add trigger**버튼을 클릭한다.

</aside>

![image.png](image%2036.png)

<aside>

트리거 구성, **EventBridge(CloudWatch Events)**를 지정하고 **Add**버튼을 클릭한다. 

- **Trigger configuration** : **EventBridge(CloudWatch Events)**
- **Rule** : Existing rules
- **기존 규칙** : 앞서 생성한 규칙 선택
</aside>

**STEP 4) 추가된 트리거 확인**

![image.png](image%2037.png)

<aside>

EventBridge가 정상적으로 트리거링 되었고 Discord에 알림을 보내기 위한 설정을 마쳤다.

</aside>

### 테스트

### Terraform 코드

> **IaC를 사용하는 이유**
> 
> 
> ---
> 
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.
> 

**참고**

실습 전 [IaC 실습 전 환경 구성](https://www.notion.so/IaC-236b5a2aa9af80528203d0ee4c07992d?pvs=21) 을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.

**[ Terraform 파일 구조 ]**

```hcl
Terraform-code/
├── main.tf                # 모든 주요 Terraform 리소스 정의 
├── variables.tf           # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars       # 민감 데이터 정의 (Email, Webhook URL 등)
├── lambda.zip             # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda_function.py # Lambda 함수 원본 코드

```

**[ Terraform 구현 코드 ]**

![image.png](image%2038.png)

- **main.tf**
    
    ```hcl
     
    ```
    
- **variables.tf**
    
    ```hcl
    # Discord Webhook 주소를 입력받기 위한 변수
    variable "discord_webhook_url" {
      description = "Discord Webhook URL for alerts"  # 알림을 보낼 Discord Webhook URL
      type        = string
    }
    
    # 이메일 수신자를 입력받기 위한 변수
    variable "email_address" {
      description = "Email address to receive SNS alerts"  # SNS 알림을 받을 이메일 주소
      type        = string
    }
    
    ```
    
- **terraform.tfvars**
    
    ```hcl
    # 알림을 받을 이메일 주소로 설정
    # 보안 이벤트가 발생 시 SNS를 통해 해당 이메일로 알람 전송
    email_address      = "your-email@example.com"
    
    # 알림을 받을 디스코드 Webhook URL로 설정
    # 해당 URL을 통해 Lambda 함수가 알림을 디스코드 채널로 전송
    discord_webhook_url = "https://discord.com/api/webhooks/xxxx/yyyy"
    ```
    
- **lambda.zip (lambda_function.py)**
    
    **의존성 없이 zip 만들기**
    
    ```bash
    zip lambda.zip lambda_function.py
    ```
    
    ```python
    import json
    import logging
    import os
    from datetime import datetime, timedelta
    from urllib.request import Request, urlopen
    from urllib.error import HTTPError, URLError
    
    # 환경변수에서 Discord 웹훅 URL 가져오기
    webhook_url = os.environ.get('DISCORD_WEBHOOK_URL')
    if not webhook_url:
        print("환경변수 DISCORD_WEBHOOK_URL이 설정되어 있지 않습니다.")
        raise Exception("Webhook URL not set")  # return 제거하고 예외 처리로
    
    # 로깅 설정
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    
    def lambda_handler(event, context):
        logger.info("Received event: " + json.dumps(event))
    
        try:
            data = event['detail']
    
            # 사용자 구분
            user_type = data['userIdentity']['type']
            user = "Root" if user_type == "Root" else data['userIdentity'].get('userName', 'Unknown')
    
            # 로그인 시간 처리 (UTC → KST)
            login_time_utc = data['eventTime'][:19]
            login_time_kst = datetime.strptime(login_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)
    
            ip = data['sourceIPAddress']
            mfa = data['additionalEventData'].get('MFAUsed', 'Unknown')
            result = data['responseElements'].get('ConsoleLogin', 'Unknown')
    
            # 디스코드 메시지
            discord_message = {
                "content": f"**[{user_type} AWS 콘솔 로그인 탐지]**\n"
                           f"사용자: {user}\n"
                           f"시간: {login_time_kst.strftime('%Y-%m-%d %H:%M:%S')} (KST)\n"
                           f"IP: {ip}\n"
                           f"MFA 사용: {mfa}\n"
                           f"로그인 결과: {result}"
            }
    
            # Discord Webhook 요청
            req = Request(
                webhook_url,  # 여기 수정됨
                data=json.dumps(discord_message).encode('utf-8'),
                headers={
                    'Content-Type': 'application/json',
                    'User-Agent': 'Mozilla/5.0 (Lambda Discord Notification)'
                }
            )
    
            response = urlopen(req)
            response.read()
            logger.info("Discord 메시지 전송 성공")
    
        except HTTPError as e:
            logger.error(f"HTTP 오류: {e.code} - {e.reason}")
        except URLError as e:
            logger.error(f"연결 실패: {e.reason}")
        except Exception as e:
            logger.error(f"예외 발생: {str(e)}")
    
    ```
    
- **Terraform 실행**
    
    **[ Terraform 실행 코드 ]**
    
    ```bash
    terraform init # 초기화
    terraform plan # 설정 검증
    terraform apply # 적용 (실행)
    -------------------------------------------------------
    terraform destroy # 실습 완료 후, 리소스 정리
    ```
    
    **[ init ]**
    
    ```bash
    terraform init
    ```
    
    <aside>
    
    Terraform 프로젝트를 처음 시작할 때 실행하는 명령어로, 작업 디렉토리를 초기화하고 필요한 설정 파일과 실행에 필요한 구성 요소들을 준비해준다. 이후 plan, apply 등의 명령을 정상적으로 사용할 수 있는 상태로 만든다.
    
    </aside>
    
    ```bash
    Terraform has been successfully initialized!
    
    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.
    
    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
    ```
    
    <aside>
    
    위와 같은 메시지가 출력되면, 프로젝트가 초기화되어 Terraform 명령어를 사용할 수 있는 준비가 완료된 것이다.
    
    </aside>
    
    **[ plan ]**
    
    ```bash
    terraform plan
    ```
    
    <aside>
    
    Terraform 코드 적용 시, 인프라에 어떤 변경이 발생할지 미리 확인할 수 있는 실행 계획을 보여주는 명령어이다.
    
    </aside>
    
    ```bash
    Plan: 24 to add, 0 to change, 0 to destroy.
    ```
    
    <aside>
    
    총 24개의 리소스가 새로 생성될 예정이며, 실행 계획이 정상적으로 생성된 상태이다. 이 출력 결과는 적용해도 문제가 없는 준비 완료 상태임을 의미한다.
    
    </aside>
    
    **[ apply ]**
    
    ```bash
    terraform apply
    ```
    
    <aside>
    
    terraform apply 명령어는 실행 계획(plan)에 따라 실제로 클라우드 인프라를 생성, 변경, 삭제하는 작업을 수행한다. Plan 단계에서 검토한 내용을 기반으로 실제 인프라에 반영하고자 할 때 사용한다.
    
    </aside>
    
    ```bash
    Apply complete! Resources: 24 added, 0 changed, 0 destroyed.
    ```
    
    <aside>
    
    위와 같은 메시지가 출력되면, 모든 리소스가 정상적으로 생성되었거나 업데이트되어 인프라 상태가 원하는 대로 적용된 것이다.
    
    </aside>
    
    **[ 이메일 인증 ]**
    
    ![image.png](ceb3bc68-5166-4e17-967d-f7e0a721307a.png)
    
    <aside>
    
    terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다.
    이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.
    
    </aside>
    
    ![image.png](image%2039.png)
    
    <aside>
    
    **Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.
    
    </aside>
    
    [테스트 ](https://www.notion.so/238b5a2aa9af80a6b190e10656c71307?pvs=21) 
    
    <aside>
    
    인증 후 위를 참고하여 테스트를 진행하면 된다.
    
    </aside>
    
    **[ destroy ]**
    
    ```bash
    terraform destroy
    ```
    
    <aside>
    
    Terraform으로 생성된 모든 인프라 리소스를 자동으로 삭제하는 명령어이다.
    **실습 완료 후**에는 해당 명령어로 불필요한 리소스를 정리할 수 있다.
    
    </aside>
    
    ```bash
    Destroy complete! Resources: 0 destroyed.
    ```
    
    <aside>
    
    위와 같은 메시지가 출력되면, 모든 리소스가 성공적으로 정리되었음을 의미한다.
    
    </aside>
    

**[ 전체 코드 압축 파일 ]**

---

## [ Scenario에 사용한 Terraform 소스 코드 ]

- **main.tf**
    
    [main.tf](main.tf)
    
    ```json
    # main.tf
    
    provider "aws" {
      region = "us-east-1"
    }
    
    # S3 Bucket for CloudTrail logs
    resource "aws_s3_bucket" "cloudtrail_bucket" {
      bucket = "root-login-alert-trail-bucket"
    }
    
    resource "aws_s3_bucket_public_access_block" "block_public_access" {
      bucket = aws_s3_bucket.cloudtrail_bucket.id
    
      block_public_acls       = true
      block_public_policy     = true
      ignore_public_acls      = true
      restrict_public_buckets = true
    }
    
    resource "aws_s3_bucket_policy" "cloudtrail_bucket_policy" { 
      bucket = aws_s3_bucket.cloudtrail_bucket.id
    
      policy = jsonencode({
        Version = "2012-10-17",
        Statement = [
          {
            Sid       = "AWSCloudTrailAclCheck",
            Effect    = "Allow",
            Principal = { Service = "cloudtrail.amazonaws.com" },
            Action    = "s3:GetBucketAcl",
            Resource  = "arn:aws:s3:::${aws_s3_bucket.cloudtrail_bucket.id}"
          },
          
          # CloudTrail 로그는 AWSLogs/<account_id>/ 경로에 저장되므로,
          # 계정 ID를 포함해 명확히 지정하는 것이 권장된다.
          # 아래의 caller_identity 데이터 소스를 사용하면 
          # account_id를 동적으로 참조할 수 있다.
    
          {
            Sid       = "AWSCloudTrailWrite",
            Effect    = "Allow",
            Principal = { Service = "cloudtrail.amazonaws.com" },
            Action    = "s3:PutObject",
            Resource  = "arn:aws:s3:::${aws_s3_bucket.cloudtrail_bucket.id}/AWSLogs/${data.aws_caller_identity.current.account_id}/*",
            Condition = {
              StringEquals = {
                "s3:x-amz-acl" = "bucket-owner-full-control"
              }
            }
          }
        ]
      })
    }
    
    # 현재 Terraform을 실행 중인 AWS 계정 정보를 가져오기 위한 데이터 소스
    # → 주로 S3 경로에 account_id를 포함시킬 때 사용된다.
    data "aws_caller_identity" "current" {}
    
    # CloudTrail 설정
    resource "aws_cloudtrail" "root_login_trail" {
      name                          = "root-console-login-alarm"
      s3_bucket_name                = aws_s3_bucket.cloudtrail_bucket.id
      include_global_service_events = true
      is_multi_region_trail         = true
      enable_logging                = true
    }
    
    # Lambda 실행 역할
    resource "aws_iam_role" "lambda_exec_role" {
      name = "lambda_root_login_exec_role"
      assume_role_policy = jsonencode({
        Version = "2012-10-17",
        Statement = [{
          Action = "sts:AssumeRole",
          Effect = "Allow",
          Principal = {
            Service = "lambda.amazonaws.com"
          }
        }]
      })
    }
    
    resource "aws_iam_role_policy_attachment" "lambda_basic" {
      role       = aws_iam_role.lambda_exec_role.name
      policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
    }
    
    # Lambda 함수 정의
    resource "aws_lambda_function" "discord_alert" {
      filename         = "lambda.zip"
      function_name    = "root_login_notify_discord"
      role             = aws_iam_role.lambda_exec_role.arn
      handler          = "lambda_function.lambda_handler"
      runtime          = "python3.13"
      source_code_hash = filebase64sha256("lambda.zip")
      environment {
        variables = {
          HOOK_URL = var.discord_webhook_url
        }
      }
    }
    
    # SNS Topic & Email 구독
    resource "aws_sns_topic" "root_login_email_topic" {
      name = "root-console-login-email"
    }
    
    resource "aws_sns_topic_subscription" "email_subscriber" {
      topic_arn = aws_sns_topic.root_login_email_topic.arn
      protocol  = "email"
      endpoint  = var.notification_email
    }
    
    # EventBridge 규칙
    resource "aws_cloudwatch_event_rule" "root_login_rule" {
      name        = "root-login-pattern"
      description = "Detect root login and trigger Lambda & SNS"
      event_pattern = jsonencode({
        "detail-type": ["AWS Console Sign In via CloudTrail"],
        "detail": {
          "userIdentity": {
            "type": ["Root"]
          },
          "eventName": ["ConsoleLogin"]
        }
      })
    }
    
    # Lambda와 연결
    resource "aws_cloudwatch_event_target" "send_to_lambda" {
      rule      = aws_cloudwatch_event_rule.root_login_rule.name
      target_id = "sendToLambda"
      arn       = aws_lambda_function.discord_alert.arn
    }
    
    resource "aws_lambda_permission" "allow_eventbridge" {
      statement_id  = "AllowExecutionFromEventBridge"
      action        = "lambda:InvokeFunction"
      function_name = aws_lambda_function.discord_alert.function_name
      principal     = "events.amazonaws.com"
      source_arn    = aws_cloudwatch_event_rule.root_login_rule.arn
    }
    
    # SNS와 연결
    resource "aws_cloudwatch_event_target" "send_to_sns" {
      rule      = aws_cloudwatch_event_rule.root_login_rule.name
      target_id = "sendToSNS"
      arn       = aws_sns_topic.root_login_email_topic.arn
    }
    
    resource "aws_sns_topic_policy" "allow_eventbridge" {
      arn    = aws_sns_topic.root_login_email_topic.arn
      policy = jsonencode({
        Version = "2012-10-17",
        Statement = [
          {
            Sid       = "AllowEventBridgePublish",
            Effect    = "Allow",
            Principal = { Service = "events.amazonaws.com" },
            Action    = "sns:Publish",
            Resource  = aws_sns_topic.root_login_email_topic.arn
          }
        ]
      })
    }
    
    ```
    
- **variables.tf**
    - 변수 정의 코드
    
    [variables.tf](variables.tf)
    
    ```
    # variables.tf
    
    variable "discord_webhook_url" {
      description = "Discord Webhook URL"
      type        = string
    }
    
    variable "notification_email" {
      description = "Email address for alerts"
      type        = string
    }
    ```
    
- **terraform.tfvars**
    - 변수 값 설정 (웹후크 URL, 이메일 주소)
    
    [terraform.tfvars](terraform.tfvars)
    
    ```json
    # terraform.tfvars
    
    discord_webhook_url = "Discord WebHook URL입력"
    notification_email  = "Email입력"
    ```
    
- **lambda.zip (lambda_function.py)**
    - 루트 로그인 이벤트 발생 시 Discord로 메시지 보내는 Lambda코드
    (zip 그대로 사용, 압축 해제X)
    
    > **Lambda는 함수 코드를 압축한 `.zip` 파일만 인식하고 실행할 수 있도록 설계되어 있기 때문**이다.
    > 
    > 
    > Terraform이나 AWS CLI, 콘솔에서 Lambda 코드를 업로드할 때 `.zip`은 사실상 **표준 형식**이다.
    > 
    
    [lambda.zip](lambda.zip)
    
    ```python
    # lambda_function.zip 안에 있는 lambda_function.py
    import json
    import logging
    import os
    from datetime import datetime, timedelta
    from urllib.request import Request, urlopen
    from urllib.error import HTTPError, URLError
    
    # Discord Webhook 환경 변수
    HOOK_URL = os.environ['HOOK_URL']
    
    # 로깅 설정
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    
    def lambda_handler(event, context):
        logger.info("Received event: " + json.dumps(event))
    
        try:
            data = event['detail']
    
            # 사용자 구분
            user_type = data['userIdentity']['type']
            user = "Root" if user_type == "Root" else data['userIdentity'].get('userName', 'Unknown')
    
            # 로그인 시간 처리 (UTC → KST)
            login_time_utc = data['eventTime'][:19]
            login_time_kst = datetime.strptime(login_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)
    
            ip = data['sourceIPAddress']
            mfa = data['additionalEventData'].get('MFAUsed', 'Unknown')
            result = data['responseElements'].get('ConsoleLogin', 'Unknown')
    
            # 디스코드 메시지
            discord_message = {
                "content": f" **[{user_type}] AWS 콘솔 로그인 탐지**\n"
                           f" 사용자: {user}\n"
                           f" 시간: {login_time_kst.strftime('%Y-%m-%d %H:%M:%S')} (KST)\n"
                           f" IP: {ip}\n"
                           f" MFA 사용: {mfa}\n"
                           f" 로그인 결과: {result}"
            }
    
            # Discord Webhook 요청 (403 우회를 위한 User-Agent 추가)
            req = Request(
                HOOK_URL,
                data=json.dumps(discord_message).encode('utf-8'),
                headers={
                    'Content-Type': 'application/json',
                    'User-Agent': 'Mozilla/5.0 (Lambda Discord Notification)'
                }
            )
    
            response = urlopen(req)
            response.read()
            logger.info("✅ Discord 메시지 전송 성공")
    
        except HTTPError as e:
            logger.error(f"❌ HTTP 오류: {e.code} - {e.reason}")
        except URLError as e:
            logger.error(f"❌ 연결 실패: {e.reason}")
        except Exception as e:
            logger.error(f"❌ 예외 발생: {str(e)}")
    ```
    
- **[옵션 코드] output.tf**
    - Console창에서 결과 확인을 위한 코드 (debug, 선택 사항)
    
    [output.tf](output.tf)
    
    ```json
    # output.tf
    
    output "s3_bucket_name" {
      description = "CloudTrail 로그가 저장되는 S3 버킷 이름"
      value       = aws_s3_bucket.cloudtrail_bucket.bucket
    }
    
    output "cloudtrail_name" {
      description = "생성된 CloudTrail 이름"
      value       = aws_cloudtrail.root_login_trail.name
    }
    
    output "lambda_function_name" {
      description = "루트 로그인 알림용 Lambda 함수 이름"
      value       = aws_lambda_function.discord_alert.function_name
    }
    
    output "sns_topic_arn" {
      description = "Email 알림을 위한 SNS Topic ARN"
      value       = aws_sns_topic.root_login_email_topic.arn
    }
    
    output "eventbridge_rule_name" {
      description = "EventBridge 규칙 이름"
      value       = aws_cloudwatch_event_rule.root_login_rule.name
    }
    ```
    

## [ Terraform 구현 ]

- AWS CLI 설치
    
    https://awscli.amazonaws.com/AWSCLIV2.msi
    
    ![image.png](image%2040.png)
    
    ![image.png](image%2041.png)
    
    <aside>
    
    다운로드 후 해당 파일을 설치해준다.
    설치가 잘 됐는지 **`aws --verison`**으로 확인할 수 있다.
    → 테라폼과 연동해서 쓸 때 필요하므로 설치했다.
    
    </aside>
    

- 액세스 키 생성
    
    ![image.png](image%2042.png)
    
    <aside>
    
    다음과 같이 사용자 → 개인 IAM계정 클릭 이후 다음과 같이 액세스 키 생성에 들어간다.
    
    </aside>
    
    ![image.png](image%2043.png)
    
    ![image.png](image%2044.png)
    
    <aside>
    
    **다음 버튼을 클릭** 한 뒤 태그는 생략하면, 키 생성이 완료가 된다.
    
    </aside>
    

- AWS 계정 연결
    
    ![image.png](image%2045.png)
    
    <aside>
    
    **`aws configure`** AWS CLI가 사용할 인증 정보(AWS 자격 증명)를 로컬 환경에 저장한다.
    
    **AWS Access Key ID** : 위에서 생성한 액세스 키
    
    **AWS Secret Access Key** : 위에서 생성한 비밀 액세스 키
    
    **Default region name** : us-east-1(버지니아 북부)
    
    **Default output format** : json 
    
    </aside>
    
    ```bash
    terraform init # 테라폼 초기화
    terraform plan # 테라폼 적용 전 어떤 거를 수행할 지 보여줌
    terraform apply # 테라폼 적용
    terraform destroy # 리소스 정리
    
    # [옵션] 테라폼 상태 리셋 코드 -> init, plan, apply각 단계에서 에러 발생하는 경우 사용
    rm .terraform  # terraform init 시 생성된 디렉터리 및 파일 삭제 이후 'A'옵션 입력으로 삭제
    rm terraform.tfstate* # 테라폼 현재 상태 삭제
    terraform init # 다시 초기화
    -------------------------------------------------------------------------------
    
    # [옵션] AWS 콘솔을 통해 만든 리소스가 존재하는 경우, apply 하기 이전에 실행해야할 명령어
    
    # 이미 만들어진 Lambda IAM Role 가져오기
    terraform import aws_iam_role.lambda_exec_role lambda_root_login_exec_role
    
    # 이미 만들어진 Cloudtrail S3 Bucket 가져오기
    terraform import aws_s3_bucket.cloudtrail_bucket root-login-alert-trail-bucket
    
    # 이미 만들어진 Cloudtrail 가져오기
    terraform import aws_cloudtrail.root_login_trail root-console-login-alarm
    
    # 이후 실행
    terraform plan
    terraform apply
    ```
    

- 오류 발생
    
    ![image.png](image%2046.png)
    
    <aside>
    
    해당 오류가 뜨면 테라폼 상태 리셋 코드 실행해야 한다.
    
    </aside>
    
    ![image.png](image%2047.png)
    
    <aside>
    
    **[ 옵션 ]**  apply후 잘못된 경우 사용하는 **테라폼 상태 리셋 코드** 실행
    
    </aside>
    
    ![image.png](image%2048.png)
    
    ![image.png](image%2049.png)
    
    ![image.png](image%2050.png)
    
    <aside>
    
    만들어지는 과정에서 무한 로딩이 될 경우, lambda에 이미 함수가 존재해서 그런 것이므로
    콘솔에 들어가서 중복되는 함수를 지우고 다시 plan, apply 해주면 정상적으로 생성이 된다.
    
    </aside>