# Console

***

### **\[ 시나리오 상세 구현 과정 ]**

[#id-1.-cloudtrail](console.md#id-1.-cloudtrail "mention")

[#id-2.-lambda](console.md#id-2.-lambda "mention")

[#id-3.-sns](console.md#id-3.-sns "mention")

[#id-4.-eventbridge](console.md#id-4.-eventbridge "mention")

[#id-5](console.md#id-5 "mention")

***

#### 1. CloudTrail 추적 생성

**STEP 1) CloudTrail 검색**

![image.png](attachment:a52b6887-b9b6-4a0c-bb06-126d718f4cd2:image.png)

\<aside>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 **CloudTrail서비스**로 이동한다.

\</aside>

**STEP 2) CloudTrail 생성 (이미 생성된 추적 있을 경우 생략)**

![image.png](attachment:c5fa04fe-fb8b-4914-8e7a-6d412fd061f2:image.png)

\<aside>

**Create trail** 버튼을 클릭해 사용할 추적을 생성한다.

\</aside>

**\[ 추적 속성 선택 ]**

![스크린샷 2025-07-23 오후 11.57.27.png](attachment:cc9b50f6-40b9-4244-b9c2-541de8cd0a55:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.57.27.png)

\<aside>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

**Log file SSE-KMS encryption은 S3 버킷에 로그가 업로드 될 때마다 알림**을 SNS로 보내는 용도이므로 체크 해제한다.

* **Trail name** : **`ct-securitygroup`**
* **Storage location :** Create new S3 bucket
* **Additional settings - Log file validation :** Enabled 해제 \</aside>

**\[ 로그 이벤트 선택 ]**

![스크린샷 2025-07-24 오전 12.00.56.png](attachment:a5c4d20c-248d-4315-bad6-302c7cd68c0d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.00.56.png)

\<aside>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

* **Events** : Management events, Insights events
* **Management events - API activity :** Read, Write 체크 \</aside>

**\[ 검토 및 생성 ]**

![스크린샷 2025-07-24 오전 12.02.26.png](attachment:faf3cfe3-a5a6-40ae-bcdc-03740d673088:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.02.26.png)

\<aside>

각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.

\</aside>

**STEP 3) 추적 생성 확인**

![스크린샷 2025-07-24 오전 12.04.12.png](attachment:f879d735-c6a7-4a39-834b-59127bb82fe3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.04.12.png)

\<aside>

이후 정상적으로 추적이 생성되었는지 확인한다.

\</aside>

#### 2. Lambda 함수 생성

**STEP 1) Discord 채널 생성 및 WebHook 설정**

**\[ 채널 만들기 ]**

![스크린샷 2025-07-24 오전 12.05.59.png](attachment:785fb509-8d85-4abc-8d0d-aeac1635161d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.05.59.png)

\<aside>

이벤트에 관한 알림을 수신 할 채널을 만들어준다.

* **채널 이름** **: `securitygroup-alarm`** \</aside>

**\[ 채널 편집 ]**

![스크린샷 2025-07-24 오전 12.10.04.png](attachment:fd4e8654-e249-4ec1-b780-92a50c61b39d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.10.04.png)

\<aside>

위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.

\</aside>

**\[ 웹후크 연동 ]**

![image.png](attachment:c477fac1-826d-4b23-be21-2409bcfc8bdb:image.png)

\<aside>

왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.

\</aside>

**\[ 웹후크 URL 복사 ]**

![스크린샷 2025-07-24 오전 12.12.28.png](attachment:d5023c67-6523-45c6-b99c-71881e4834d4:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.12.28.png)

\<aside>

왼쪽 상단의 설정 목록에서 연동 → 웹후크 만들기를 클릭하여 웹후크 봇을 만들어 준다. 이후 아래의 ‘**웹후크 URL 복사**’ 버튼을 클릭해 Lambda에서 사용할 URL을 가져온다.

* **이름** : 원하는 이름으로 작성
* **채널** : #securitygroup\_alarm (앞서 생성한 채널 이름 선택) \</aside>

**STEP 2) Lambda 함수 생성**

![스크린샷 2025-07-24 오전 12.13.55.png](attachment:b577f050-6491-4993-a04a-b1205fb2f7b3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.13.55.png)

\<aside>

알람을 발송할 함수를 만들기 위해 AWS 콘솔에서 **Lambda서비스**로 이동한다.

\</aside>

![image.png](attachment:f1eebfa6-6882-4e1c-b3b2-81751385349b:image.png)

\<aside>

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.

\</aside>

**\[ 함수 생성 ]**

![스크린샷 2025-07-24 오전 12.16.20.png](attachment:db288bce-0853-41ec-8faa-4ebb98b86624:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.16.20.png)

\<aside>

함수 이름, 런타임 및 아키텍처를 지정하고 **Next**버튼을 클릭한다.

* **Create function :** Author from scratch 선택
* **Function name** **: `lambda-securitygroup-alarm`**
* **Runtime :** Python 3.13
* **Architecture :** x86\_64 \</aside>

**\[ 생성된 함수 확인 ]**

![스크린샷 2025-07-24 오전 12.18.21.png](attachment:6b55edde-d102-4628-9127-98c895bae15f:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.18.21.png)

\<aside>

정상적으로 Lambda함수가 생성되었는지 확인해준다.

\</aside>

**STEP 3) 환경 변수 편집**

![스크린샷 2025-07-24 오전 12.19.57.png](attachment:f7ccd746-3965-44a1-91c9-e06bd94e90ab:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.19.57.png)

\<aside>

이후 Configuration → Environment variables로 들어가서 **Edit** 버튼을 클릭한다.

\</aside>

**\[ 환경 변수 추가 ]**

![image.png](attachment:efc2a473-f143-4377-8257-7298a5136e3a:image.png)

\<aside>

Edit environment variables로 이동하여 **Add environment variables** 버튼을 클릭한다.

\</aside>

**\[ 환경 변수에 키와 값 추가 ]**

![image.png](attachment:7d1ca46a-00c3-41d0-9b36-065877dc8668:image.png)

\<aside>

**Key, Value**를 \*\*\*\*다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

* **Key, Value는 표를 참고**

| Key                   | **용도/설명**            | Value                                                                                           |
| --------------------- | -------------------- | ----------------------------------------------------------------------------------------------- |
| DISCORD\_WEBHOOK\_URL | 디스코드 알림용 Webhook URL | [https://discord.com/api/webhooks/\~\~\~](https://discord.com/api/webhooks/~~~) (알림 받을 웹후크 url) |
| \</aside>             |                      |                                                                                                 |

**STEP 4) Lambda 코드 소스 편집**

![스크린샷 2025-07-24 오전 12.26.10.png](attachment:5dbddf66-ddcd-4fab-aa54-106e67886313:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.26.10.png)

\<aside>

Code탭에서 **Lambda python 코드**를 작성 후 **Deploy**버튼을 클릭하여 배포한다.

\</aside>

```python
import os          
import json       
import urllib3     

# HTTP 요청 객체
http = urllib3.PoolManager()

# 환경 변수에서 Discord Webhook URL 불러오기
WEBHOOK = os.environ["DISCORD_WEBHOOK_URL"] # 위에서 정의한 key와 동일하게 작성

# Lambda의 메인 핸들러 함수 정의
def lambda_handler(event, context):
    for rec in event["Records"]:
        msg = json.loads(rec["Sns"]["Message"])
        detail = msg.get("detail", {})

        # 이벤트 이름
        event_name = detail.get("eventName", "N/A")

        # 이벤트 발생 시간
        event_time_utc = msg.get("time", "N/A")
        event_time_kst = event_time_utc.replace("T", " ").replace("Z", "") if event_time_utc != "N/A" else "N/A"

        # 사용자 정보
        user_arn = detail.get("userIdentity", {}).get("arn", "N/A")

        # API 호출자의 IP 주소
        source_ip = detail.get("sourceIPAddress", "N/A")

        # 리전 정보
        aws_region = msg.get("region", "N/A")

        # 계정 ID
        account_id = msg.get("account", "N/A")

        # 보안 그룹 ID
        sg_id = "N/A"
        params = detail.get("requestParameters", {})
        if "groupId" in params:
            sg_id = params["groupId"]
        elif "groupIds" in params and isinstance(params["groupIds"], list):
            sg_id = ", ".join(params["groupIds"])

        # Discord 메시지 구성
        content = (
            f"**[Security Group 변경 감지]**\\n"
            f"• 이벤트 이름: `{event_name}`\\n"
            f"• 보안 그룹 ID: `{sg_id}`\\n"
            f"• 발생 시간(KST): `{event_time_kst}`\\n"
            f"• 사용자 ARN: `{user_arn}`\\n"
            f"• 소스 IP: `{source_ip}`\\n"
            f"• 리전: `{aws_region}`\\n"
            f"• 계정 ID: `{account_id}`"
        )

        # Discord Webhook으로 메시지 전송
        http.request(
            "POST", WEBHOOK,
            body=json.dumps({"content": content}).encode(),
            headers={"Content-Type": "application/json"}
        )

http.request(
            "POST", WEBHOOK,
            body=json.dumps({"content": content}).encode(),
            headers={"Content-Type": "application/json"}
        )
```

#### 3. SNS 주제 생성

**STEP 1) SNS 검색**

![image.png](attachment:fe6edf73-93fc-4fa8-b07f-9e3c1b2c2066:image.png)

\<aside>

알람을 전송 받을 주제 및 구독을 생성하기 위해 **SNS 서비스**로 이동한다.

\</aside>

**STEP 2) 주제 생성**

![image.png](attachment:2217bf7e-d2a1-4db2-bcb4-31d20655c005:image.png)

\<aside>

좌측 탭에서 Topic으로 이동 후 **Create** topic 버튼을 클릭한다.

\</aside>

![스크린샷 2025-07-24 오전 12.31.02.png](attachment:f97c3cce-8ce4-4022-b3a0-8a27533b47c1:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.31.02.png)

\<aside>

아래와 같이 작성한 후 Create Topic을 클릭한다.

* **Details :** Standard
* **Name :** **`sns-securitygroup-alarm`** \</aside>

**STEP 3 ) 구독 생성 - Email**

![스크린샷 2025-07-24 오전 12.32.35.png](attachment:75cf107a-861f-4497-a16a-b06573b44f89:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.32.35.png)

\<aside>

생성된 주제 확인 후 **Create subscription**을 누른다.

\</aside>

**\[ 구독 생성 - 세부사항 ]**

![스크린샷 2025-07-24 오전 12.34.06.png](attachment:c58d724c-7cb4-4b7c-befc-e58cb15b67e9:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.34.06.png)

\<aside>

생성된 주제 확인 후 **Create subscription**을 클릭한다.

* **Protocol** : Email
* **Endpoint** : 알람 받을 이메일 주소 \</aside>

**STEP 4 ) 구독한 이메일 인증**

![스크린샷 2025-07-24 오전 12.36.34.png](attachment:7afcaf85-c002-419f-8389-4bef9f24cdc3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.36.34.png)

\<aside>

설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다. 이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.

\</aside>

![image.png](attachment:a1640aac-6890-49a3-be3c-0b368fd08cee:image.png)

\<aside>

**Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.

\</aside>

**STEP 5 ) 구독 생성 - Lambda**

![스크린샷 2025-07-24 오전 12.41.38.png](attachment:e6067d34-d71c-49a2-86a6-f518fe3a31b5:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.41.38.png)

\<aside>

디스코드로 알림을 보내기 위해 위에서 만든 Lambda 구독을 추가 생성한다.

\</aside>

**\[ 구독 생성 - 세부사항 ]**

![스크린샷 2025-07-24 오전 12.44.18.png](attachment:b3729999-e81c-4afb-a063-e777d5809e95:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.44.18.png)

\<aside>

* **Protocol** : AWS Lambda
* **Endpoint** : 앞서 생성한 Lambda 선택 \</aside>

**\[ 생성된 구독 확인 ]**

![스크린샷 2025-07-24 오전 12.47.34.png](attachment:9fe9df50-cf9e-4ea5-9cdd-94949896e127:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.47.34.png)

#### 4. EventBridge 규칙 생성

**STEP 1) EventBridge 검색**

![image.png](attachment:b8e6ca7a-b26d-4fae-92a1-a8dc5bb0d022:image.png)

\<aside>

Lambda 함수를 주기적으로 실행하기 위해 AWS 콘솔에서 **EventBridge 서비스**로 이동한다.

\</aside>

**STEP 2) EventBridge 규칙 생성**

![image.png](attachment:c9822578-a0f8-492e-9e34-25ececdaf47d:image.png)

\<aside>

**Create rule** 버튼을 클릭해서 새 EventBridge 규칙을 생성한다.

\</aside>

**\[ 규칙 세부 정보 정의 ]**

![스크린샷 2025-07-24 오전 12.51.44.png](attachment:e612efcb-0da9-4895-9182-99ff9a2fec05:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.51.44.png)

\<aside>

* **Name** : **`eventbridge-securitygroup-changerule`**
* **Description** : (옵션)
* **Event bus :** default
* **Rule type** : Rule with an event pattern \</aside>

**\[ 이벤트 패턴 작성 ]**

![스크린샷 2025-07-26 오전 4.34.19.png](attachment:fe0b389d-b9c4-487a-97ca-1c6759ea453b:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_4.34.19.png)

\<aside>

* **Event Source :** Other
* **Event pattern** : Custom pattern (JSON editor)

```json
{
  "source": ["aws.ec2"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": [
      "AuthorizeSecurityGroupIngress",
      "AuthorizeSecurityGroupEgress",
      "RevokeSecurityGroupIngress",
      "RevokeSecurityGroupEgress",
      "DeleteSecurityGroup"
    ]
  }
}
```

\</aside>

**\[ 대상 선택 ]**

![스크린샷 2025-07-24 오전 12.57.19.png](attachment:00397719-07e6-412f-81f0-c57b1cfd3210:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.57.19.png)

\<aside>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

* **Target Types** : AWS service
* **Select a target** : SNS topic
* **Target location** : Target in this account
* **Topic** : 미리 만들어 둔 sns topic 선택 \</aside>

**\[ 태그 구성 (선택) ]**

![image.png](attachment:e5371439-fac3-4a7d-8a35-7fff304748aa:image.png)

\<aside>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.

\</aside>

**\[ 검토 및 생성 ]**

![스크린샷 2025-07-24 오전 12.59.14.png](attachment:7d1cc016-8caf-4d39-8534-c526e884d40c:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.59.14.png)

\<aside>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인 \</aside>

**STEP 3) 생성된 규칙 확인**

![스크린샷 2025-07-24 오전 1.01.09.png](attachment:413b6777-50f7-4e6a-9d31-d39a0c98df07:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.01.09.png)

\<aside>

규칙이 정상적으로 생성되었는지 확인해준다.

\</aside>

#### 5. 테스트

> 테스트를 위한 EC2를 생성하여 보안그룹 자체를 삭제하거나 인바운드 / 아웃바운드 규칙을 추가하고 삭제 할 수 있다.

**\[ 탐지 이벤트 안내 ]**

| 이벤트 이름                          | 설명                           | 탐지목적                                                                  |
| ------------------------------- | ---------------------------- | --------------------------------------------------------------------- |
| `AuthorizeSecurityGroupIngress` | Security group에 인바운드 규칙 추가   | **비인가 외부 접근 허용 탐지** – 외부에서 EC2 인스턴스로의 접근을 허용하는 행위 감지                  |
| `RevokeSecurityGroupIngress`    | Security group에서 인바운드 규칙 제거  | **접근 차단 또는 흔적 제거 시도 탐지** – 기존 접속 규칙을 제거함으로써 로그 감시 우회를 시도하는 행위 식별      |
| `AuthorizeSecurityGroupEgress`  | Security group에 아웃바운드 규칙 추가  | **외부로의 데이터 유출 통로 생성 탐지** – EC2 등 내부 자원에서 외부로 통신할 수 있는 경로를 설정한 행위 감시   |
| `RevokeSecurityGroupEgress`     | Security group에서 아웃바운드 규칙 제거 | **유출 차단 흔적 삭제 또는 탐지 회피 시도 탐지** – 기존 유출 경로를 감추기 위한 행위 식별               |
| `DeleteSecurityGroup`           | 기존 Security Group 삭제         | **보안 정책 우회 또는 탐지 회피 시도 탐지** – 모니터링 대상이던 보안 그룹을 삭제함으로써 감시를 회피하려는 행위 감지 |

**\[ EC2 인스턴스 생성 ]**

![스크린샷 2025-07-22 오전 2.11.28.png](attachment:6b79b214-36f3-4638-8aef-825f660affe6:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.11.28.png)

\<aside>

이벤트 테스트를 위해 EC2로 이동한다.

\</aside>

![스크린샷 2025-07-22 오전 2.21.00.png](attachment:48a3d310-e95d-4dfa-b7f2-06b629cadd25:f383a41c-4bdc-45b1-ab8c-3b71404b3f39.png)

**\[ Instance 시작 ]**

![스크린샷 2025-07-24 오전 1.11.12.png](attachment:e0b405de-c033-46e4-8ed8-f1b880c0c4d0:ec0fa719-1944-4fa5-bc14-09b287783931.png)

\<aside>

* **Name** **: `ec2-securitygroup-test`**
* **Amazon Machine Image (AMI)** : Amazon Linux 2 AMI \</aside>

**\[ Network 설정 ]**

![스크린샷 2025-07-24 오전 1.17.42.png](attachment:45fd31a4-33df-4a7c-b82e-f6c412e7911d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.17.42.png)

\<aside>

테스트용 EC2 인스턴스를 생성해 테스트를 진행한다. 아래 사항 외에는 기본값 그대로 진행한다.

**Network Settings**

* **VPC** : default
* **Firewall (security groups)** : Create security group
* **Security group name** : **`securitygroup-test`**
* **Type, Source type** : ssh - My IP / Custom TCP - Anywhere

입력 후 **Launch Instance**를 클릭한다.

\</aside>

**\[ Key pair 설정]**

![스크린샷 2025-07-22 오전 2.17.58.png](attachment:fc18d063-30a6-4c77-b2b2-8d14ad2af177:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.17.58.png)

\<aside>

테스트를 위해 임의로 만든 EC2이므로 따로 키 페어 없이 진행한다.

\</aside>

**\[ 인스턴스 생성 확인 ]**

![스크린샷 2025-07-24 오전 1.22.05.png](attachment:13659b76-e6f9-4255-b778-270b39c63af4:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.22.05.png)

**\[ `AuthorizeSecurityGroupIngress` 이벤트 발생 ]**

![스크린샷 2025-07-24 오전 1.26.07.png](attachment:0aff6da3-7ae9-4269-84cc-8be173d72a42:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.26.07.png)

\<aside>

생성한 EC2 인스턴스의 **Security** 탭에서 앞서 생성한 **Inbound rule**의 **Security Group** 설정에 접속한다

\</aside>

![스크린샷 2025-07-24 오전 1.28.53.png](attachment:cda27b0d-1242-44b8-93ce-28d30153640e:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.28.53.png)

\<aside>

**Edit Inbound rules**를 클릭한다.

\</aside>

**\[ 보안 그룹 인바운드 규칙 추가 ]**

![스크린샷 2025-07-24 오전 1.32.28.png](attachment:d3131e47-56cc-468f-beb7-2607cbc691e3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.32.28.png)

\<aside>

SSH의 source를 **Anywhere-IPv4**로 바꿔 저장한다.

\</aside>

**\[ `RevokeSecurityGroupIngress` 이벤트 발생 ]**

![스크린샷 2025-07-24 오전 1.26.07.png](attachment:0aff6da3-7ae9-4269-84cc-8be173d72a42:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.26.07.png)

\<aside>

생성한 EC2 인스턴스의 **Security** 탭에서 앞서 생성한 **Inbound rules**의 **Security Group** 설정에 접속한다

\</aside>

![스크린샷 2025-07-24 오전 1.28.53.png](attachment:425b0e2e-b7e5-4e2e-8ffd-9f5b13e75768:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-24_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.28.53.png)

\<aside>

**Edit Inbound rules**를 클릭한다.

\</aside>

**\[ 보안 그룹 인바운드 규칙 제거 ]**

![스크린샷 2025-07-25 오후 9.47.36.png](attachment:1f9e4ab9-cd6c-4b52-bfc9-67aa3fcaf230:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-25_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.47.36.png)

\<aside>

보안 그룹에서 생성한 TCP 규칙을 **Delete**을 클릭해 삭제한다.

삭제 후 **Save rules**를 클릭해 변경 사항을 저장해준다.

\</aside>

**\[ `AuthorizeSecurityGroupEgress` 이벤트 발생 ]**

![스크린샷 2025-07-26 오전 1.57.36.png](attachment:d952fc02-fe81-40bc-bc98-c056e788a345:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.57.36.png)

\<aside>

생성한 EC2 인스턴스의 **Security** 탭에서 앞서 생성한 **Outbound rules**의 **Security Group** 설정에 접속한다

\</aside>

![스크린샷 2025-07-26 오전 2.00.13.png](attachment:9b755336-3280-4409-9500-d282a754cbce:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.00.13.png)

\<aside>

**Edit Inbound rules**를 클릭한다.

\</aside>

**\[ 보안 그룹 아웃바운드 규칙 추가 ]**

![스크린샷 2025-07-26 오전 2.05.44.png](attachment:aeac3605-70d7-4bc3-ab15-afbb8412e819:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.05.44.png)

\<aside>

편집 페이지에서 **Add rule**을 클릭한다.

\</aside>

![스크린샷 2025-07-26 오전 2.07.46.png](attachment:df9dcf83-726e-41c1-89e7-8ca7952ae5ef:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.07.46.png)

\<aside>

아웃바운드 규칙 추가 이벤트를 유발하기 위해 새로운 규칙을 생성한다.

* **Type** : HTTP
* **Destination** : Anywhere-IPv4

설정 후 **Save rules**를 클릭한다.

\</aside>

**\[ `RevokeSecurityGroupEgress` 이벤트 발생 ]**

![스크린샷 2025-07-26 오전 1.57.36.png](attachment:d952fc02-fe81-40bc-bc98-c056e788a345:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.57.36.png)

\<aside>

생성한 EC2 인스턴스의 **Security** 탭에서 앞서 생성한 **Outbound rules**의 **Security Group** 설정에 접속한다

\</aside>

![스크린샷 2025-07-26 오전 2.00.13.png](attachment:9b755336-3280-4409-9500-d282a754cbce:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.00.13.png)

\<aside>

**Edit Inbound rules**를 클릭한다.

\</aside>

**\[ 보안 그룹 아웃바운드 규칙 제거 ]**

![스크린샷 2025-07-26 오전 2.18.25.png](attachment:ae94f50d-ace6-4432-b409-72fc2b3b9ccc:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.18.25.png)

\<aside>

보안 그룹에서 앞서 생성한 HTTP 규칙을 **Delete**을 클릭해 삭제한다.

삭제 후 **Save rules**를 클릭해 변경 사항을 저장해준다.

\</aside>

**\[ `DeleteSecurityGroup` 이벤트 발생 ]**

![스크린샷 2025-07-26 오전 3.50.06.png](attachment:acf8beb3-e434-4633-afcd-4fea79c77bcf:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.50.06.png)

\<aside>

보안 그룹이 인스턴스에 연결되어 있는 상태에서는 삭제가 불가능하기 때문에 **인스턴스를 먼저 제거**해야 한다.

\</aside>

**\[ 보안 그룹 삭제 ]**

![스크린샷 2025-07-26 오전 3.53.39.png](attachment:4c20bf7a-a126-491b-b1f9-9347dedf580b:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.53.39.png)

\<aside>

**EC2 → Security Groups**에 접속해 앞서 생성한 보안 그룹을 선택한다. 선택 후 **Actions → Delete security groups**을 클릭하여 보안 그룹을 삭제해 이벤트를 유발한다.

\</aside>

**\[ Email 알림 확인 ]**

![스크린샷 2025-07-26 오전 3.58.18.png](attachment:e749f3cc-c2ac-47aa-ac8a-7552bdb66b80:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.58.18.png)

**\[ Discord 알림 확인 ]**

![스크린샷 2025-07-26 오전 4.07.35.png](attachment:3dc0832d-2b3c-46af-a820-d6d6ca6025ba:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-26_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_4.07.35.png)
