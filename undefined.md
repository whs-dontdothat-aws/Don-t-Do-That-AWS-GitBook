# root-login

***

### **\[ 시나리오 안내 ]**&#x20;

<table data-header-hidden><thead><tr><th width="213.20001220703125"></th><th></th></tr></thead><tbody><tr><td><strong>내용</strong></td><td><strong>루트 계정은 AWS 계정 생성 시에만 활용되고 IAM User등으로 관리되어 로그인 횟수가 적어집니다. 또한 모든 권한을 가지고 있기 때문에 실제 운영 환경에서 사용이 최소화되어야 합니다. 루트 계정으로 콘솔 로그인이 발생한 경우 실시간 탐지를 통해 이상 행위 여부를 판단하는 시나리오를 구현해 봅니다.</strong></td></tr><tr><td>사용 서비스</td><td>CloudTrail, CloudWatch, EventBridge Rule, SNS, Lambda</td></tr><tr><td>탐지 조건</td><td>웹 콘솔에 로그인하는 이벤트 중 루트 사용자인 경우를 탐지합니다.</td></tr><tr><td>알림 방식</td><td>SNS를 통해 Email로 알림을 전송하고, 동시에 Lambda 함수를 활용해 Webhook을 통해 Discord로도 알림을 전송합니다.</td></tr><tr><td>대응</td><td>알림으로 내용을 확인하고 사용자가 대처합니다.</td></tr></tbody></table>

***

### **\[ 시나리오 전체적인 흐름 ]**

<figure><img src=".gitbook/assets/image (33).png" alt="" width="375"><figcaption></figcaption></figure>

| **AWS Service**                       | **Service Purpose**                    | **Workbook Usage**                                   |
| ------------------------------------- | -------------------------------------- | ---------------------------------------------------- |
| **CloudTrail**                        | AWS 계정 내 API 호출 및 로그인 활동을 기록하는 서비스입니다. | 루트 계정의 콘솔 로그인 이벤트를 감지하기 위한 로그 데이터 를수집합니다.            |
| **EventBridge**                       | 이벤트 패턴에 따라 자동으로 트리거를 발생시키는 서비스입니다.     | 루트 로그인 이벤트가 감지되면 지정된 Lambda와 SNS를 호출하도록 이벤트 룰 설정합니다. |
| **SNS (Simple Notification Service)** | 메시지를 여러 구독자에게 전달하는 메시징 서비스입니다.         | 루트 로그인 알림을 Email로 전송합니다.                             |
| **Lambda**                            | 서버 없이 코드를 실행할 수 있는 이벤트 기반 컴퓨팅 서비스입니다.  | 루트 로그인 이벤트 발생 시 Discord Webhook으로 메시지 전송합니다.         |
| **S3**                                | 객체 기반 스토리지 서비스입니다.                     | CloudTrail 로그 저장소로 활용합니다.                            |

**실습 개요**

* 이  워크북에서는 루트 계정의 AWS 콘솔 로그인 이벤트를 실시간으로 탐지하고, 이를 이메일(SNS)과 Discord(Webhook)를 통해 즉시 알림으로 전송하는 시나리오를 구현합니다.
* CloudTrail과 EventBridge를 활용해 이벤트를 감지하고, Lambda를 통해 다중 채널로 알림을 전달합니다.

#### **학습 목표**

* 루트 계정 로그인 탐지 시나리오 설계를 통해 보안 사고 대응 체계 학습합니다.
* AWS의 Event 기반 자동화 구조 (CloudTrail → EventBridge → Lambda/SNS) 이해합니다.
* 실시간 알림 시스템 구축 및 외부 채널(Discord) 연동 방법 습득합니다.

***

**참고 사항**

* 루트 로그인은 글로벌 이벤트 → EventBridge는 us-east-1에서만 감지 가능합니다.
* 실습 시 EventBridge와 Lambda를 us-east-1에 구성해야 알림이 정상 작동합니다.
* CloudTrail은 서울에 두어도 무방하나, 글로벌 이벤트 포함 옵션이 켜져 있어야 합니다.
* “루트 계정 로그인 알림 시나리오”실습 시, region을 서울로 하면 안 되고 반드시 버지니아 북부(미국 동부, us-east-1)로 설정해주어야  합니다.

\
&#xNAN;**\[ 콘솔 리소스명 ]**

<table data-header-hidden><thead><tr><th width="157.79998779296875"></th><th></th><th></th></tr></thead><tbody><tr><td><strong>리소스 종류</strong></td><td><strong>리소스명 설정</strong></td><td><strong>목적</strong></td></tr><tr><td>S3 Bucket</td><td><strong><code>s3-root-login-detect</code></strong></td><td>CloudTrail 로그 저장소로 사용하여 root 로그인 이벤트 기록</td></tr><tr><td>Lambda Function</td><td><strong><code>lambda-root-login</code></strong></td><td>EventBridge 이벤트 트리거 시 root 로그인 이벤트 처리 및 SNS/Discord 전송</td></tr><tr><td>CloudTrail</td><td><strong><code>ct-trail-monitor</code></strong></td><td>AWS 계정의 root 로그인 이벤트를 감지하기 위한 로그 수집 설정</td></tr><tr><td>EventBridge</td><td><strong><code>eventbridge-root-login-pattern</code></strong></td><td>root 로그인 이벤트 패턴 감지를 위한 이벤트 규칙 설정</td></tr><tr><td>SNS Topic</td><td><strong><code>sns-root-login-alarm</code></strong></td><td>이메일 등의 경고 채널로 root 로그인 알림 전송</td></tr><tr><td>Discord 채널</td><td><strong><code>root-login-alarm</code></strong></td><td>Lambda에서 전송한 root 로그인 알림 수신용 채널</td></tr></tbody></table>

***

### **\[ 시나리오 상세 구현 과정 ]**

<details>

<summary>1.Lambda 함수 생성 및 Discord 연동</summary>



**STEP 1) Discord 채널 생성 및 WebHook 설정**

**\[ 채널 만들기 ]**

<figure><img src=".gitbook/assets/image (63).png" alt="" width="375"><figcaption></figcaption></figure>

이벤트에 관한 알림을 수신 할 채널을 생성해준다.

* **채널 이름** : `root-login-alarm`&#x20;

**\[ 채널 편집 ]**

<figure><img src=".gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.

**\[ 웹후크 연동 ]**

<figure><img src=".gitbook/assets/image (65).png" alt="" width="375"><figcaption></figcaption></figure>

왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.

**\[ 웹후크 URL 복사 ]**

<figure><img src=".gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 가져온다.

* **이름** : WEBHOOK\_URL
* **채널** : `#root-login-alarm` (앞서 생성한 채널 이름 선택)&#x20;

**STEP 2) Lambda 검색**&#x20;

![](<.gitbook/assets/image (34).png>)\
서버 없이 이벤트 발생 시 자동으로 코드를 실행하기 위해 AWS 콘솔에서 Lambda서비스로 이동한다.

**STEP 3) Lambda 함수 생성**

![](<.gitbook/assets/image (35).png>)&#x20;

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.

**\[ 함수 생성 ]**

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

* **Author from scratch** 선택
* **Function name** : `lamda-root-login`
* **Runtime** : Python 3.13
* **Architecture** : x86\_64

**\[ 생성된 함수 확인 ]**

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

정상적으로 Lambda함수가 생성되었는지 확인해준다.

**STEP 4) 환경 변수 편집**

<figure><img src=".gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

이후 Configuration → Environment variables로 들어가서 **Edit** 버튼을 클릭한다.

**\[ 환경 변수 추가 ]**

<figure><img src=".gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

Edit environment variables로 이동하여 **Add environment variables** 버튼을 클릭한다.

**\[ 환경 변수에 키와 값 추가 ]**

<figure><img src=".gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

**Key, Value**를 다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

이렇게 환경 변수 지정을 통해 Discord Hook 봇을 **호출할 수 있고** 환경 변수를 사용함으로써 소스 코드에서 URL의 **노출을 방지**할 수 있다.

* **Key, Value는 표를 참고**

| Key                   | 용도/설명                | Value                             |
| --------------------- | -------------------- | --------------------------------- |
| DISCORD\_WEBHOOK\_URL | 디스코드 알림용 Webhook URL | https://discord.com/api/webhooks/ |



**STEP 5) Lambda 소스 코드 편집**

<figure><img src=".gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

Code탭에서 Lambda python 코드를 작성 후 Deploy버튼을 클릭하여 배포해 준다.

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



**STEP 6) Lambda 트리거 추가**

**\[ Lambda 트리거 - EventBridge(CloudWatch Events) ]**

<figure><img src=".gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

생성한 Lambda함수의 다이어그램 왼쪽 하단의 **Add trigger**버튼을 클릭한다.



<figure><img src=".gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

트리거 구성, EventBridge(CloudWatch Events)를 지정하고 **Add**버튼을 클릭한다.

* **Trigger configuration** : **EventBridge(CloudWatch Events)**
* **Rule** : Existing rules
* **기존 규칙** : 앞서 생성한 규칙 선택

**STEP 7) 추가된 트리거 확인**

<figure><img src=".gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

EventBridge가 정상적으로 트리거링 되었고 Discord에 알림을 보내기 위한 설정을 마쳤다.

</details>

<details>

<summary>2.S3 버킷 및 CloudTrail 추적 생성</summary>

**STEP 1) S3 검색**&#x20;

<figure><img src=".gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

Cloudtrail 로그를 저장할 버킷을 만들기 위해 S3 서비스로 이동한다.

**STEP 2) S3 bucket 생성**

**\[ S3 bucket 생성]**

<figure><img src=".gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

S3 서비스 화면 오른쪽 상단의 **Create a bucket**버튼을 클릭한다.

**\[ bucket 속성 선택 ]**&#x20;

<figure><img src=".gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/스크린샷 2025-06-30 163654.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/스크린샷 2025-06-30 163705.png" alt=""><figcaption></figcaption></figure>

* **Bucket name :** **`s3-root-login-detect`**
* **Object Ownership :** ACLs disabled (recommended)
* **Block Public Access settings for this bucket :** Block all public access
* **Bucket Versioning :** Enable
* **Encryption type :** Server-side encryption with Amazon S3 managed keys (SSE-S3)

**STEP 3) CloudTrail 검색**&#x20;

<figure><img src=".gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 CloudTrail서비스로 이동한다.&#x20;

**STEP 4) CloudTrail 생성**

<figure><img src=".gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

**Create trail**버튼을 클릭해 사용할 추적을 생성한다.

**\[ 추적 속성 선택 ]**

<figure><img src=".gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

* **Trail name** : **`ct-trail-monitor`**
* **Storage location :** Use existing S3 bucket (**Browe**를 클릭해 앞서 생성한 버킷 선택)
* **Additional settings**
  * **Log file validation :** Enabled

**\[ 로그 이벤트 선택 ]**

<figure><img src=".gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

* **Events** : Management events
* **Management events - API activity :** Read, Write

**\[ 검토 및 생성 ]**

<figure><img src=".gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.

**STEP 5) 추적 생성 확인**

<figure><img src=".gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

대시보드에서 정상적으로 추적이 생성되었는지 확인한다.

</details>

<details>

<summary>3.EventBridge 규칙 생성</summary>

**STEP 1) EventBridge 검색**

<figure><img src=".gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

Lambda 함수를 주기적으로 실행하기 위해 AWS 콘솔에서 **EventBridge 서비스**로 이동한다.

**STEP 2) EventBridge 생성**

<figure><img src=".gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

EventBridge 서비스 화면 오른쪽 상단의 EventBridge Rule을 선택하고 **Create rule**버튼을 클릭한다.

**\[ 상세 규칙 설정 ]**

<figure><img src=".gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

* **Name** : **`eventbridge-root-login-pattern`**
* **Event bus :** default
* **Rule type** : Rule with an event pattern

**\[ 이벤트 패턴 작성 ]**

<figure><img src=".gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

탐지할 이벤트 조건을 설정을 설정하고 **Next**버튼을 클릭한다.

* **Events :** Other
* **Event pattern** : Custom pattern (JSON editor)

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

**\[ 설정한 이벤트 안내 ]**

| 이벤트 이름         | 설명            | 탐지 목적                                          |
| -------------- | ------------- | ---------------------------------------------- |
| `ConsoleLogin` | 웹 콘솔에 로그인 이벤트 | **이상 행위 여부 판단** - 모든 권한을 가진 루트 계정으로 로그인할 경우 식별 |

**\[ 대상 선택 ]**

<figure><img src=".gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

* **Target types:** AWS service
* **Select a target:** Lambda function
* **Target location:** Target in this account
* **Topic:** 앞서 생성한 Lambda function 선택

**\[ 태그 구성 (선택) ]**

<figure><img src=".gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.

**\[ 검토 및 생성 ]**

<figure><img src=".gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인

**STEP 3) 생성된 규칙 확인**

<figure><img src=".gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

규칙이 정상적으로 생성되었는지 확인해준다.

</details>

<details>

<summary>4.SNS 주제 생성 및 구독 설정</summary>

**STEP 1) SNS 검색**

<figure><img src=".gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

알람을 전송 받을 주제 및 구독을 생성하기 위해 **SNS 서비스**로 이동한다.

**STEP 2) 주제 생성**

<figure><img src=".gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

좌측 탭에서 Topic으로 이동 후 **Create topic** 버튼을 클릭한다.



<figure><img src=".gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

* **Type** : Standard
* **Name** : **`sns-root-login-alarm`**

**STEP 3 ) 구독 생성**

<figure><img src=".gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

생성된 주제 확인 후 **Create subscription**을 누른다.

**\[ 구독 생성 - 세부사항 ]**

<figure><img src=".gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

* **Protocol** : email
* **Endpoint** : 알람 받을 이메일 주소

**STEP 4 ) 구독한 이메일 인증**

<figure><img src=".gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다.\
이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.

<figure><img src=".gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

**Confirm subscription**을 눌러 접속하면 정상적으로 SNS 구독 등록이 된 것이다.

</details>

<details>

<summary>5.테스트</summary>



</details>
