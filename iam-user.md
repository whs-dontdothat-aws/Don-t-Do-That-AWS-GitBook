# 새로운 IAM User의 생성, 삭제 탐지

**\[ 목차 ]**

[#undefined](iam-user.md#undefined "mention")

[#undefined-4](iam-user.md#undefined-4 "mention")

[#undefined-1](iam-user.md#undefined-1 "mention")

[#undefined-2](iam-user.md#undefined-2 "mention")

[#undefined-3](iam-user.md#undefined-3 "mention")

[#undefined-5](iam-user.md#undefined-5 "mention")

[#undefined-6](iam-user.md#undefined-6 "mention")

[#id-1.-cloudtrail](iam-user.md#id-1.-cloudtrail "mention")

[#id-2.-lambda-discord](iam-user.md#id-2.-lambda-discord "mention")

[#id-3.-sns](iam-user.md#id-3.-sns "mention")

[#id-4.-eventbridge](iam-user.md#id-4.-eventbridge "mention")

***

### **\[ 시나리오 안내 ]**

<table><thead><tr><th width="124.77783203125">내용</th><th>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나 공격자가 콘솔 접근을 위해서도 사용할 수 있습니다. 사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고 변경 이력을 검토하는 시나리오를 구현 해봅니다.</th></tr></thead><tbody><tr><td>사용 서비스</td><td>Lambda, CloudTrail, SNS, EventBridge</td></tr><tr><td>탐지 조건</td><td>IAM User의 생성, 삭제를 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td>알림을 확인하고 사용자가 대처한다.</td></tr><tr><td></td><td>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나 공격자가 콘솔 접근을 위해서도 사용할 수 있습니다. 사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고 변경 이력을 검토하는 시나리오를 구현 해봅니다.</td></tr></tbody></table>

<table data-header-hidden><thead><tr><th width="278.99993896484375"></th><th></th></tr></thead><tbody><tr><td>내용</td><td>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나<br>공격자가 콘솔 접근을 위해서도 사용할 수 있습니다.<br>사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고<br>변경 이력을 검토하는 시나리오를 구현 해봅니다.</td></tr><tr><td>사용 서비스</td><td>Lambda, CloudTrail, SNS, EventBridge</td></tr><tr><td>탐지 조건</td><td>IAM User의 생성, 삭제를 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td>알림을 확인하고 사용자가 대처한다.</td></tr></tbody></table>

***

### **\[ 시나리오 전체적인 흐름 ]**

| **AWS Service**                                              | **Service Purpose**                                                                                                                                                  | **Workbook Usage**                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| IAM                                                          | AWS 리소스에 접근할 수 있는 사용자(User), 그들의 권한(Policy), 인증 정보(Access Key 등)를 관리하는 서비스입니다\*\*.\*\* IAM을 통해 개별 사용자 계정을 생성하고, 사용자 별로 권한을 부여할 수 있으며, 사용자 생성/삭제 이력에 대한 감사 추적이 가능합니다. | IAM 사용자가 생성 및 삭제될 때 CloudTrail이 이를 기록하고, EventBridge를 통해 탐지되며, 알림이 SNS를 통해 전달되는 구조의 핵심 리소스 대상입니다. |
| CloudTrail                                                   | AWS 계정에서 발생하는 모든 사용자 활동과 API 호출을 자동으로 기록하고, 이를 로그로 저장하는 서비스입니다.                                                                                                      |                                                                                                   |
| 기본적으로는 최근 90일간의 이벤트만 콘솔에서 조회할 수 있지만,                         |                                                                                                                                                                      |                                                                                                   |
| 추적을 생성하면 지정한 S3 버킷에 이벤트 로그 파일이 지속적으로 저장되어 장기 보관 및 분석이 가능합니다. | EventBridge에서 `CreateUser`, `DeleteUser`로그를 참조할 수 있도록 추적을 생성합니다.                                                                                                     |                                                                                                   |
| EventBridge                                                  | AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.                                            | CloudTrail에 기록된 로그 중 `CreateUser`, `DeleteUser`를 탐지하고 SNS로 이벤트를 전송합니다.                            |
| SNS                                                          | 발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.                                                                             | Eventbridge로 부터 이벤트를 수신하면 SNS 주제를 구독하고 있는 엔드포인트(이메일, Lambda)로 이벤트 이력을 전송합니다.                      |
| Lambda                                                       | 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.                                            | SNS로부터 호출되면 사전 지정한 외부 Webhook(Discord)으로 메시지를 전달합니다.                                              |

#### 실습 개요

* 이 워크북에서는 IAM 사용자가 생성, 삭제되었을 때 이를 탐지하는 방법을 학습합니다.
* IAM 사용자 생성 및 삭제 탐지는 AWS 계정 보안의 핵심 지표로, 권한 남용, 내부 위협, 외부 침해 행위를 조기에 식별하고 대응하기 위해 반드시 구현되어야 할 보안 모니터링 항목입니다.
* 본 워크북에서는 CloudTrail과 EventBridge, SNS 등을 활용해 IAM 사용자 생성/삭제 이벤트 발생 시 이메일 및 디스코드 알림을 받도록 구현하는 방법을 실습합니다.

#### 학습 목표

* AWS에서 IAM 사용자의 생성 및 삭제 이벤트를 탐지하는 방법을 학습합니다.
* CloudTrail과 EventBridge를 활용해 IAM 사용자 생성/삭제 이벤트를 수집하고, 이를 SNS와 연동하여 이메일 및 디스코드 알림을 전송하는 실습을 진행합니다.
* 생성/삭제 이벤트에 대한 탐지 조건을 코드로 정의하여 실시간 대응 체계를 구축하는 과정을 이해합니다.
* 실제로 IAM 사용자를 생성 및 삭제하여 이벤트 탐지가 정상적으로 동작하는지 실습을 통해 검증하는 경험을 쌓습니다.

***

#### 참고 사항

* **IAM**은 AWS **전역(Global) 서비스**로, 그 이벤트는 **버지니아 북부(us-east-1)** 리전의 CloudTrail과EventBridge를 통해서만 탐지할 수 있기 때문에 본 실습은 \*\*“us-east-1(Virginia)”\*\*리전에서 진행됩니다.
* 본 워크북에서 생성되는 리소스 중 일부는 Terraform code를 통해 제공되고 있습니다.
* 주요 리소스에 대한 사전 구성을 필요로 하는 경우 하단의 “xx. Terraform으로 리소스 구현” 에서 내용을 참고하실 수 있습니다.
* 해당 시나리오에 맞게 리소스명은 임의로 설정하였으며 사용자가 원하는 이름으로 바꿔도 무방합니다.



**\[ 콘솔 리소스명 ]**

| **리소스 종류**       | **리소스명 설정**                 | **목적**                                             |
| ---------------- | --------------------------- | -------------------------------------------------- |
| CloudTrail Trail | **`ct-trail-monitor`**      | AWS 계정 내 API 활동을 감시하고 로그를 S3로 저장                   |
| Discord 채널       | **`iam-alarm`**             | SNS에 연동된 Lambda 함수를 통해 알림 메시지를 수신하고 확인할 수 있는 알림 채널 |
| Lambda Function  | **`lambda-iam-user-alarm`** | SNS 메시지를 파싱하여 Discord Webhook으로 이상 행위 알림 전송        |
| SNS Topic        | **`sns-iam-user-alarm`**    | 이벤트 발생 시 이메일 및 Lambda로 알림을 전송하는 주제                 |
| EventBridge Rule | **`eventbridge-ct-detect`** | 특정 CloudTrail 이벤트 발생 시 SNS로 이벤트 전송하는 트리거 역할        |
| IAM User         | **`IAM-test-user`**         | IAM 사용자 생성 및 삭제 이벤트가 발생하는지 테스트하기 위해 생성하는 테스트 사용자   |

***

### **\[ 시나리오 상세 구현 과정 ]**

#### **1. CloudTrail 추적 생성**

**STEP 1) CloudTrail 검색**

![image.png](attachment:c111f85f-655c-4355-8c89-ad7ae897b8f8:image.png)

![image.png](attachment:a52b6887-b9b6-4a0c-bb06-126d718f4cd2:image.png)

\<aside>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 버지니아 리전 선택 후 **CloudTrail서비스**로 이동한다. 해당 리전에 생성된 trail이 있을 경우, 추가 생성 없이 [**2번 단계**](https://www.notion.so/IAM-User-1fcb5a2aa9af8097b7c0e37c535b603b?pvs=21) 으로 넘어간다.

\</aside>

**STEP 2 ) CloudTrail 생성**

![image.png](attachment:c5fa04fe-fb8b-4914-8e7a-6d412fd061f2:image.png)

\<aside>

**Create trail** 버튼을 클릭해 사용할 추적을 생성한다.

\</aside>

**\[ 추적 속성 선택 ]**

![image.png](attachment:7ebff021-323c-40f6-93d6-80c6ece63bd4:image.png)

<예시 사진>

\<aside>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

**Log file SSE-KMS encryption은 S3 버킷에 로그가 업로드 될 때마다 알림**을 SNS로 보내는 용도이므로 체크 해제한다.

* **Trail name** : ct-trail-monitor
* **Storage location :** Create new S3 bucket
* **Additional settings**
  * **Log file validation :** Enabled 해제 \</aside>

**\[ 로그 이벤트 선택 ]**

![image.png](attachment:938ed6a1-ff8e-47b1-b0ee-5637f0e48db7:image.png)

<예시 사진>

\<aside>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

* **Events** : Management events, Insights events
* **Management events - API activity :** Read, Write 체크 \</aside>

**\[** **검토 및 생성 ]**

![image.png](attachment:a95a3cd7-b1eb-43ff-ba1a-7d3c4960c168:image.png)

<예시 사진>

\<aside>

각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.

\</aside>

**STEP 3) 추적 생성 확인**

![image.png](attachment:b219cf81-66e3-47cc-b449-4ab80875dbb1:image.png)

<예시 사진>

\<aside>

대시보드에서 정상적으로 추적이 생성되었는지 확인한다.

\</aside>

#### 2. **Lambda 함수 생성 및 Discord 연동**

**STEP 1) Discord 채널 생성 및 WebHook 설정**

**\[ 채널 만들기 ]**

![image.png](attachment:a6593368-eccb-4924-9776-30ceedc9f77f:image.png)

\<aside>

IAM 사용자 생성/삭제 이벤트 알림을 수신할 채널을 만들어준다.

* **채널 이름** : **`iam-alarm`** \</aside>

**\[ 채널 편집 ]**

![image.png](attachment:e29c632f-57a3-4a05-9b11-4bf1c7d09dc6:image.png)

\<aside>

위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.

\</aside>

**\[ 웹후크 연동 ]**

![image.png](attachment:f86ebf3c-8688-435c-837c-12d54c82d567:image.png)

\<aside>

왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.

\</aside>

![image.png](attachment:82cb7478-dd85-431b-83af-a0bb38ae6298:image.png)

\<aside>

**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 복사한다.

* **이름** : WEBHOOK\_URL
* **채널** : **`#iam-alarm`** (앞서 생성한 채널 이름 선택) \</aside>

**STEP 2) Lambda 함수 생성**

![image.png](attachment:c7d406c9-1c91-4581-8a69-1168b29fa009:image.png)

\<aside>

알람을 발송할 함수를 만들기 위해 AWS 콘솔에서 **Lambda서비스**로 이동한다.

\</aside>

![image.png](attachment:f1eebfa6-6882-4e1c-b3b2-81751385349b:image.png)

\<aside>

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.

\</aside>

**\[ 함수 생성 ]**

![image.png](attachment:49dcdc44-c4ea-4399-9297-41b01f16c6b7:ebb0d972-1612-4049-b398-008994cbfb7f.png)

\<aside>

함수 이름, 런타임 및 아키텍처를 지정하고 **Next**버튼을 클릭한다.

* **Author from scratch** 선택
* **Function name** : **`lambda-iam-user-alarm`**
* **Runtime** : Python 3.13
* **Architecture** : x86\_64 \</aside>

**\[ 생성된 함수 확인 ]**

![image.png](attachment:b8ae8f03-906f-4aa1-b75d-1332d676d7c4:image.png)

\<aside>

정상적으로 Lambda함수가 생성되었는지 확인해준다.

\</aside>

**STEP 3) 환경 변수 편집**

![image.png](attachment:e343adad-9524-4bab-ad2b-f4c52d8d8adc:image.png)

\<aside>

이후 Configuration → Environment variables로 들어가서 **Edit** 버튼을 클릭한다.

\</aside>

**\[ 환경 변수 추가 ]**

![image.png](attachment:1d0a38ed-5d05-447d-b34a-282836b9d4ca:image.png)

\<aside>

Edit environment variables로 이동하여 **Add environment variables** 버튼을 클릭한다.

\</aside>

**\[ 환경 변수에 키와 값 추가 ]**

![image (1).png](attachment:71e48632-82d4-49c5-9506-9888a52b7564:image_\(1\).png)

\<aside>

**Key, Value**를 \*\*\*\*다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

* **Key, Value는 표를 참고**

| Key                   | **용도/설명**            | Value                                                                                           |
| --------------------- | -------------------- | ----------------------------------------------------------------------------------------------- |
| DISCORD\_WEBHOOK\_URL | 디스코드 알림용 Webhook URL | [https://discord.com/api/webhooks/\~\~\~](https://discord.com/api/webhooks/~~~) (알림 받을 웹후크 url) |
| \</aside>             |                      |                                                                                                 |

**STEP 4) Lambda 코드 소스 편집**

![image.png](attachment:fea7aa61-714a-4f93-9aee-025cad0a8d23:image.png)

\<aside>

Code탭에서 **Lambda python 코드**를 작성 후 **Deploy**버튼을 클릭하여 배포해 준다.

\</aside>

```python
import json
import urllib3
import os
from datetime import datetime, timedelta

# HTTP 요청을 위한 객체 생성
http = urllib3.PoolManager()

# 환경 변수에서 Discord Webhook URL 불러오기
HOOK_URL = os.environ['DISCORD_WEBHOOK_URL'] # 위에서 정의한 key와 동일하게 작성

def lambda_handler(event, context):
    try:
        for record in event['Records']:
            # SNS 메시지 파싱
            sns_message_str = record['Sns']['Message']
            outer_msg = json.loads(sns_message_str)

            # CloudTrail 이벤트의 실제 내용은 'detail' 내부에 존재
            detail = outer_msg.get('detail', {})

            # 이벤트 정보 추출
            event_name = detail.get('eventName', 'Unknown')
            user = detail.get('userIdentity', {}).get('userName', 'Unknown')
            source_ip = detail.get('sourceIPAddress', 'Unknown')
            event_time_utc = detail.get('eventTime', '')[:19]

            # 시간 변환 (UTC → KST)
            try:
                event_time_kst = datetime.strptime(event_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)
                time_str = event_time_kst.strftime('%Y-%m-%d %H:%M:%S') + " (KST)"
            except:
                time_str = 'Unknown'

            # Discord 메시지 구성
            discord_msg = {
                "content": f"**[ IAM 이벤트 탐지 ]**\\n"
                           f"- 이벤트 이름: `{event_name}`\\n"
                           f"- 사용자: `{user}`\\n"
                           f"- IP: `{source_ip}`\\n"
                           f"- 시간: `{time_str}`"
            }

            # 전송
            encoded_msg = json.dumps(discord_msg).encode("utf-8")
            response = http.request(
                "POST",
                HOOK_URL,
                body=encoded_msg,
                headers={"Content-Type": "application/json"}
            )

            print(f"Discord 응답 상태: {response.status}")
        
        return {"statusCode": 200, "body": "Success"}

    except Exception as e:
        print(f"에러 발생: {str(e)}")
        return {"statusCode": 500, "body": "Error"}

```

#### 3. SNS 주제 생성 및 구독 설정

**STEP 1) SNS 검색**

![image.png](attachment:8a97ce67-abe0-4e72-8a1a-cb6eb2327ecb:image.png)

\<aside>

알람을 전송 받을 주제 및 구독을 생성하기 위해 **SNS 서비스**로 이동한다.

\</aside>

**STEP 2) 주제 생성**

![image.png](attachment:57b9ede9-5fe9-4e4d-a139-1a53e9cfb66b:image.png)

\<aside>

좌측 탭에서 Topic으로 이동 후 **Create** topic 버튼을 클릭한다.

\</aside>

![image.png](attachment:a6301fd9-bb98-46c7-8981-035938bc8629:image.png)

\<aside>

* **Type** : Standard(표준)
* **Name** : **`sns-iam-user-alarm`** \</aside>

**STEP 3) Email 구독 생성**

![image.png](attachment:a9770000-a772-4346-83c8-f3297f4716a6:image.png)

\<aside>

생성된 주제 확인 후 **Create subscription 버튼**을 클릭한다.

\</aside>

**\[ 구독 생성 - 세부사항 ]**

![image.png](attachment:6870759a-ed7b-444e-8649-ba433d70105b:image.png)

\<aside>

* **Protocol** : email
* **Endpoint** : 알람 받을 이메일 주소 \</aside>

**STEP 4) 구독한 이메일 인증**

![image.png](attachment:280226fa-94d9-42cb-9358-df9c89ab5c8c:image.png)

\<aside>

생성된 구독을 확인하면 Status가 Pending Confirmation(확인 대기 중)이다.

입력한 메일 주소로 온 확인 메일을 통해 인증을 진행한다.

\</aside>

**\[ 이메일 인증 ]**

![image.png](attachment:c7232419-a7c5-4e67-9dbd-8ba70eb94155:image.png)

![image.png](attachment:08e19220-de39-43c6-8eb5-007eb582f85d:image.png)

![image.png](attachment:91e04a6c-3ef8-4d60-9aba-4ecb2d83fbab:image.png)

\<aside>

**Subscription Confirmation** 메일의 Confirm subscription 하이퍼링크를 눌러 접속하면 SNS 구독 등록이 완료된다.

\</aside>

**STEP 5) Lambda 구독 생성**

![image.png](attachment:0fe3e83f-21c9-4393-8aa4-607a98433c6c:image.png)

\<aside>

디스코드로 알림을 보내기 위해 위에서 만든 Lambda 구독을 추가 생성한다.

\</aside>

![image.png](attachment:275d27f5-7c90-4b2d-a753-725d30ae80be:image.png)

\<aside>

* **Protocol** : AWS Lambda
* **Endpoint** : 위에서 생성한 Lambda (**`lambda-iam-user-alarm`**) 선택 \</aside>

#### 4. EventBridge 규칙 생성

**STEP 1) EventBridge 검색**

![image.png](attachment:c0566cb9-8453-4e57-bbf9-e61d2893d2b7:image.png)

\<aside>

Lambda 함수를 주기적으로 실행하기 위해 **EventBridge 서비스**로 이동한다.

\</aside>

**STEP 2) EventBridge 생성**

![image.png](attachment:c9822578-a0f8-492e-9e34-25ececdaf47d:image.png)

\<aside>

**EventBridge** 서비스 화면 오른쪽 상단의 **EventBridge Rule**을 선택하고 **Create rule**버튼을 클릭한다.

\</aside>

**\[ 상세 규칙 설정 ]**

![image.png](attachment:830e7d13-f190-4313-8929-4a8cc857c09d:image.png)

\<aside>

규칙 이름, 설명, Event bus 종류, 규칙 유형(이벤트 패턴 기반 or 스케줄 기반) 설정 후 **Next버튼**을 클릭한다.

* **Name** : **`eventbridge-iam-user-change`**
* **Event Bus** : default
* **Rule Type** : Rule with an event pattern (이벤트 패턴이 있는 규칙) \</aside>

**\[ 이벤트 패턴 작성 ]**

![image.png](attachment:7b51bc59-a3d1-4dfb-8d91-033bc17b572f:image.png)

\<aside>

탐지할 이벤트 조건을 설정을 설정하고 **Next**버튼을 클릭한다.

* **Event Source :** Other
*   **Event pattern** : Custom pattern (JSON editor)

    사용자가 원하는 조건만 감지할 수 있도록 JSON으로 직접 작성
*   IAM 사용자 생성/삭제를 탐지하는 JSON 코드

    ```json
    {
      "source": ["aws.iam"],
      "detail-type": ["AWS API Call via CloudTrail"],
      "detail": {
        "eventSource": ["iam.amazonaws.com"],
        "eventName": ["CreateUser", "DeleteUser"]
      }
    }
    ```

**\[ 설정한 이벤트 안내 ]**

| 이벤트 이름           | 설명         | **탐지 목적**                      |
| ---------------- | ---------- | ------------------------------ |
| **`CreateUser`** | IAM 사용자 생성 | **생성 탐지** - 권한 남용, 내부 위협행위 식별  |
| **`DeleteUser`** | IAM 사용자 삭제 | **삭제 탐지** - 사용자를 없애려는 공격 행위 식별 |
| \</aside>        |            |                                |

**\[ 대상 선택 ]**

![image.png](attachment:a5047ce9-e86f-4561-9d6d-511a55f3e9d2:image.png)

\<aside>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

* **Target Types** : AWS service
* **Select a target** : SNS topic
* **Target location** : Target in this account
* **Topic** : 앞서 생성한 sns topic 선택(**`sns-iam-user-alarm`**)
* **Execution role** : Create a new role for this specific resources (이 특정 리소스에 대해 역할 생성)
* **Role name** : 자동 할당 \</aside>

**\[ 태그 구성 (선택) ]**

![image.png](attachment:e5371439-fac3-4a7d-8a35-7fff304748aa:image.png)

\<aside>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.

\</aside>

**\[ 검토 및 생성 ]**

![image.png](attachment:c46f9c6b-2039-4012-b597-b12d86f2a569:image.png)

![image.png](attachment:0d151e2e-5153-427d-bb9e-0796ae80ce43:9cc4b38a-eee1-45e6-aef6-6bf5e6c2143c.png)

\<aside>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인 \</aside>

**STEP 3) 생성된 규칙 확인**

![image.png](attachment:1ce37096-ffbb-4509-8640-0b6a687ea88f:image.png)

![image.png](attachment:9061ab21-076c-4df5-bcdf-7ddd2436e68c:image.png)

\<aside>

규칙이 정상적으로 생성되었는지 확인한다.

\</aside>

#### 5. 테스트

> IAM 콘솔에서 사용자를 생성하고 삭제하면서 이벤트를 발생시킨다.

**\[ 탐지 이벤트 안내 ]**

| 이벤트 이름           | 설명         | **탐지 목적**                      |
| ---------------- | ---------- | ------------------------------ |
| **`CreateUser`** | IAM 사용자 생성 | **생성 탐지** - 권한 남용, 내부 위협행위 식별  |
| **`DeleteUser`** | IAM 사용자 삭제 | **삭제 탐지** - 사용자를 없애려는 공격 행위 식별 |

![image.png](attachment:8dd27e1c-879b-43ec-b50d-b200015fae8c:image.png)

\<aside>

이벤트 테스트를 위해 IAM 콘솔로 이동한다.

\</aside>

**\[`CreateUser`이벤트 발생 ]**

![image.png](attachment:af336aff-bc03-4b78-a2f0-efd5d312caa5:image.png)

\<aside>

좌측 탭에서 **Users**를 선택 후 **Create User 버튼**을 클릭한다.

\</aside>

![image.png](attachment:44eeb056-eb80-4fac-8f07-7f88a113e55a:image.png)

![image.png](attachment:e4db739f-6104-4f2f-aa01-6c7fbfd3abb6:image.png)

\<aside>

아래 사항 외에는 기본값 그대로 진행한 후 **Create User버튼**을 눌러 테스트 사용자를 생성한다.

* Name : **`IAM-test-user`** \</aside>

**\[`DeleteUser`이벤트 발생 ]**

![image.png](attachment:2de44043-73dd-4a70-8a76-cc920cb65387:image.png)

\<aside>

앞서 만든 사용자를 체크한 후 **Delete버튼**을 클릭해 삭제한다.

\</aside>

**\[ CloudTrail에서 이벤트 발생 기록 확인 ]**

![image.png](attachment:ef2ce093-0b01-47c0-bd36-614c751c84c0:image.png)

![image.png](attachment:1bb4b4a1-81c7-4d7b-a641-3481158743ea:4c6d592d-d7e8-49fb-9c9d-7c4f2d32da49.png)

\<aside>

IAM 사용자를 생성 하고 삭제하면서 CloudTrail에 **`CreaterUser`**, **`DeleteUser`** 이벤트가 기록된 것을 확인할 수 있다.

\</aside>

**\[ Email 알림 확인 ]**

![image.png](attachment:4143c882-c0a9-4699-b0ba-14d6ba9c2f0e:image.png)

**\[ Discord 알림 확인 ]**

![image.png](attachment:a79c3643-99af-479c-97d0-8cbd704161be:image.png)

#### 6. Terraform 구현

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.

**참고**

실습 전 [IaC 실습 전 환경 구성](https://www.notion.so/IaC-236b5a2aa9af80528203d0ee4c07992d?pvs=21) 을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.

**\[ terraform 파일 구조 ]**

```hcl
Terraform-code/
├── main.tf                # 모든 주요 Terraform 리소스 정의 
├── variables.tf           # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars       # 민감 데이터 정의 (Email, Webhook URL 등)
├── lambda.zip             # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda_function.py # Lambda 함수 원본 코드
```

**\[ Terraform 구현 코드 ]**

*   [**main.tf**](http://main.tf)

    ```hcl
    #---------------------------------------------------------------------------
    # 1. PROVIDER ─ AWS 리전 설정 (버지니아)
    #---------------------------------------------------------------------------
    provider "aws" {
      region = "us-east-1"
    }

    #---------------------------------------------------------------------------
    # 2. CloudTrail 설정(이미 리전에 추적 존재한다면 이 부분 생략)
    #---------------------------------------------------------------------------

    # CloudTrail 로그를 저장할 S3 버킷 생성
    resource "aws_s3_bucket" "cloudtrail_bucket" {
      bucket = "iam-user-event-alarm-bucket" # 버킷 이름
      force_destroy = true # destroy 할 때 강제로 삭제하게함 (로그 비우고 삭제)
    }

    # 생성한 S3 버킷에 대한 퍼블릭 액세스 차단
    resource "aws_s3_bucket_public_access_block" "block_public_access" {
      bucket = aws_s3_bucket.cloudtrail_bucket.id

      block_public_acls       = true # 퍼블릭 ACL 지정 차단
      block_public_policy     = true # 퍼블릭한 버킷 정책 차단
      ignore_public_acls      = true # 퍼블릭 ACL 있어도 무시
      restrict_public_buckets = true # 퍼블릭 정책이 있어도 거부
    }

    # CloudTrail 서비스가 로그 기록을 위한 S3 버킷에 접근 허용
    resource "aws_s3_bucket_policy" "cloudtrail_bucket_policy" {
      bucket = aws_s3_bucket.cloudtrail_bucket.id

      policy = jsonencode({
        Version = "2012-10-17",
        Statement = [ # CloudTrail이 ACL을 읽는 권한 필요
          {
            Sid       = "AWSCloudTrailAclCheck",
            Effect    = "Allow",
            Principal = { Service = "cloudtrail.amazonaws.com" },
            Action    = "s3:GetBucketAcl",
            Resource  = "arn:aws:s3:::${aws_s3_bucket.cloudtrail_bucket.id}"
          },
          { # CloudTrail이 로그 객체를 업로드 할 수 있도록 설정
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

    # 현재 사용중인 계정 ID 가져옴
    data "aws_caller_identity" "current" {}

    # CloudTrail 추적 생성
    resource "aws_cloudtrail" "iam_event_trail" {
      name                          = aws_s3_bucket.cloudtrail_bucket.bucket # Trail 이름름
      s3_bucket_name                = aws_s3_bucket.cloudtrail_bucket.id
      include_global_service_events = true
      is_multi_region_trail         = true
      enable_logging                = true
    }

    #---------------------------------------------------------------------------
    # 3. Lambda 설정
    #---------------------------------------------------------------------------

    # Lambda 실행을 위한 IAM 역할 생성
    resource "aws_iam_role" "lambda_exec_role" {
      name = "lambda_iam_event_exec_role"
      assume_role_policy = jsonencode({
        Version = "2012-10-17",
        Statement = [{
          Action = "sts:AssumeRole", # 역할 위임받는 데 필요한 권한 부여
          Effect = "Allow",
          Principal = {
            Service = "lambda.amazonaws.com"
          }
        }]
      })
    }

    # Lambda 실행 IAM 역할에 AWSLambdaBasicExecutionRole 정책 연결
    resource "aws_iam_role_policy_attachment" "lambda_basic" {
      role       = aws_iam_role.lambda_exec_role.name
      policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
    }

    # Lambda 함수 정의
    resource "aws_lambda_function" "discord_alert" {
      filename         = "lambda_function.zip"
      function_name    = "iam_user_event_discord"
      role             = aws_iam_role.lambda_exec_role.arn
      handler          = "lambda_function.lambda_handler"
      runtime          = "python3.13"
      source_code_hash = filebase64sha256("lambda_function.zip")
      environment {
        variables = {
          HOOK_URL = var.discord_webhook_url
        }
      }
    }

    #---------------------------------------------------------------------------
    # 4. SNS Topic 및 Email, Lambda 구독 설정
    #---------------------------------------------------------------------------

    # SNS 생성
    resource "aws_sns_topic" "iam_user_event_topic" {
      name = "iam-user-event-topic"
    }

    # 이메일 구독 생성
    resource "aws_sns_topic_subscription" "email_subscriber" {
      topic_arn = aws_sns_topic.iam_user_event_topic.arn
      protocol  = "email"
      endpoint  = var.notification_email
    }

    # Lambda 구독 생성
    resource "aws_sns_topic_subscription" "lambda_subscriber" {
      topic_arn = aws_sns_topic.iam_user_event_topic.arn
      protocol  = "lambda"
      endpoint  = aws_lambda_function.discord_alert.arn
    }

    # SNS가 Lambda 호출할 수 있도록 권한 부여
    resource "aws_lambda_permission" "allow_sns_invoke" {
      statement_id = "AllowExecutionFromSNS"
      action = "lambda:InvokeFunction"
      function_name = aws_lambda_function.discord_alert.function_name
      principal = "sns.amazonaws.com"
      source_arn = aws_sns_topic.iam_user_event_topic.arn
    }

    #---------------------------------------------------------------------------
    # 5. EventBridge 설정
    #---------------------------------------------------------------------------

    # EventBridge 규칙 생성 : IAM User 생성, 삭제 탐지
    resource "aws_cloudwatch_event_rule" "iam_user_event_pattern" {
      name        = "iam-user-event-pattern"
      description = "Detect IAM User Event and trigger Lambda & SNS"
      event_pattern = jsonencode({
        "source": ["aws.iam"],
        "detail-type": ["AWS API Call via CloudTrail"],
        "detail": {
          "eventSource": ["iam.amazonaws.com"],
          "eventName": ["CreateUser", "DeleteUser"]
        }
      })
    }

    # EventBridge 규칙의 타겟으로 SNS 주 연결
    resource "aws_cloudwatch_event_target" "send_to_sns" {
      rule      = aws_cloudwatch_event_rule.iam_user_event_pattern.name
      target_id = "sendToSNS"
      arn       = aws_sns_topic.iam_user_event_topic.arn
    }

    # EventBridge에 SNS 호출 권한을 부여
    resource "aws_sns_topic_policy" "allow_eventbridge" {
      arn    = aws_sns_topic.iam_user_event_topic.arn
      policy = jsonencode({
        Version = "2012-10-17",
        Statement = [
          {
            Sid       = "AllowEventBridgePublish",
            Effect    = "Allow",
            Principal = { Service = "events.amazonaws.com" },
            Action    = "sns:Publish",
            Resource  = aws_sns_topic.iam_user_event_topic.arn
          }
        ]
      })
    }
    ```
*   [**variables.tf**](http://variables.tf)

    ```hcl
    # 해당 파일에서는 변수를 여기다가 모두 선언
    # terraform 실행 시 terraform.tfvars에 선언된 값을 바탕으로 값이 들어감

    # Discord Webhook 주소를 입력받기 위한 변수
    variable "discord_webhook_url" {
      description = "Discord Webhook URL"
      type        = string
    }

    # 이메일 수신자를 입력받기 위한 변수
    variable "notification_email" {
      description = "Email address for alerts"
      type        = string
    }
    ```
*   **terraform.tfvars**

    ```hcl
    # 알림을 받을 디스코드 Webhook URL로 설정
    # 해당 URL을 통해 Lambda 함수가 알림을 디스코드 채널로 전송
    discord_webhook_url = "discord web hook url"

    # 알림을 받을 이메일 주소로 설정
    # 보안 이벤트가 발생 시 SNS를 통해 해당 이메일로 알람 전송
    notification_email  = "이메일 주소"
    ```
*   **lambda.zip (lambda\_function.py)**

    ```python
    # lambda.zip 안에 있는 lambda_function.py
    import json
    import urllib3
    import os
    from datetime import datetime, timedelta

    # HTTP 요청을 위한 객체 생성
    http = urllib3.PoolManager()

    # 환경 변수에서 Discord Webhook URL 불러오기
    HOOK_URL = os.environ['HOOK_URL']

    def lambda_handler(event, context):
        try:
            for record in event['Records']:
                # SNS 메시지 파싱
                sns_message_str = record['Sns']['Message']
                outer_msg = json.loads(sns_message_str)

                # CloudTrail 이벤트의 실제 내용은 'detail' 내부에 존재
                detail = outer_msg.get('detail', {})

                # 이벤트 정보 추출
                event_name = detail.get('eventName', 'Unknown')
                user = detail.get('userIdentity', {}).get('userName', 'Unknown')
                source_ip = detail.get('sourceIPAddress', 'Unknown')
                event_time_utc = detail.get('eventTime', '')[:19]

                # 시간 변환 (UTC → KST)
                try:
                    event_time_kst = datetime.strptime(event_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)
                    time_str = event_time_kst.strftime('%Y-%m-%d %H:%M:%S') + " (KST)"
                except:
                    time_str = 'Unknown'

                # Discord 메시지 구성
                discord_msg = {
                    "content": f"**IAM 이벤트 감지됨**\\n"
                               f"- 이벤트: `{event_name}`\\n"
                               f"- 사용자: `{user}`\\n"
                               f"- IP: `{source_ip}`\\n"
                               f"- 시간: `{time_str}`"
                }

                # 전송
                encoded_msg = json.dumps(discord_msg).encode("utf-8")
                response = http.request(
                    "POST",
                    HOOK_URL,
                    body=encoded_msg,
                    headers={"Content-Type": "application/json"}
                )

                print(f"Discord 응답 상태: {response.status}")
            
            return {"statusCode": 200, "body": "Success"}

        except Exception as e:
            print(f"에러 발생: {str(e)}")
            return {"statusCode": 500, "body": "Error"}

    ```
*   **Terraform 실행**

    **\[ Terraform 실행 코드 ]**

    ```bash
    terraform init # 초기화
    terraform plan # 설정 검증
    terraform apply # 적용 (실행)
    -------------------------------------------------------
    terraform destroy # 실습 완료 후, 리소스 정리
    ```

    **\[ init ]**

    ```bash
    terraform init
    ```

    \<aside>

    Terraform 프로젝트를 처음 시작할 때 실행하는 명령어로, 작업 디렉토리를 초기화하고 필요한 설정 파일과 실행에 필요한 구성 요소들을 준비해준다. 이후 plan, apply 등의 명령을 정상적으로 사용할 수 있는 상태로 만든다.

    \</aside>

    ```bash
    Terraform has been successfully initialized!

    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.

    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
    ```

    \<aside>

    위와 같은 메시지가 출력되면, 프로젝트가 초기화되어 Terraform 명령어를 사용할 수 있는 준비가 완료된 것이다.

    \</aside>

    **\[ plan ]**

    ```bash
    terraform plan
    ```

    \<aside>

    Terraform 코드 적용 시, 인프라에 어떤 변경이 발생할지 미리 확인할 수 있는 실행 계획을 보여주는 명령어이다.

    \</aside>

    ```bash
    Plan: 24 to add, 0 to change, 0 to destroy.
    ```

    \<aside>

    총 24개의 리소스가 새로 생성될 예정이며, 실행 계획이 정상적으로 생성된 상태이다. 이 출력 결과는 적용해도 문제가 없는 준비 완료 상태임을 의미한다.

    \</aside>

    **\[ apply ]**

    ```bash
    terraform apply
    ```

    \<aside>

    terraform apply 명령어는 실행 계획(plan)에 따라 실제로 클라우드 인프라를 생성, 변경, 삭제하는 작업을 수행한다. Plan 단계에서 검토한 내용을 기반으로 실제 인프라에 반영하고자 할 때 사용한다.

    \</aside>

    ```bash
    Apply complete! Resources: 24 added, 0 changed, 0 destroyed.
    ```

    \<aside>

    위와 같은 메시지가 출력되면, 모든 리소스가 정상적으로 생성되었거나 업데이트되어 인프라 상태가 원하는 대로 적용된 것이다.

    \</aside>

    **\[ 이메일 인증 ]**

    ![image.png](attachment:d535f282-0114-4cc6-a9dd-d0763c795521:ceb3bc68-5166-4e17-967d-f7e0a721307a.png)

    \<aside>

    terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다. 이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.

    \</aside>

    ![image.png](attachment:a1640aac-6890-49a3-be3c-0b368fd08cee:image.png)

    \<aside>

    **Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.

    \</aside>

    ‣

    \<aside>

    인증 후 위를 참고하여 테스트를 진행하면 된다.

    \</aside>

    **\[ destroy ]**

    ```bash
    terraform destroy
    ```

    \<aside>

    Terraform으로 생성된 모든 인프라 리소스를 자동으로 삭제하는 명령어이다. **실습 완료 후**에는 해당 명령어로 불필요한 리소스를 정리할 수 있다.

    \</aside>

    ```bash
    Destroy complete! Resources: 0 destroyed.
    ```

    \<aside>

    위와 같은 메시지가 출력되면, 모든 리소스가 성공적으로 정리되었음을 의미한다.

    \</aside>

**\[ 전체 코드 압축 파일 ]**

[Terraform-code.zip](attachment:665ef56d-f846-4cc5-b876-eb61a7011f82:Terraform-code.zip)
