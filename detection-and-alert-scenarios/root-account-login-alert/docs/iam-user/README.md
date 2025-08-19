# 새로운 IAM User의 생성, 삭제 탐지

**\[ 목차 ]**

[#undefined](./#undefined "mention")

[#undefined-1](./#undefined-1 "mention")

[#undefined-2](./#undefined-2 "mention")

[#undefined-3](./#undefined-3 "mention")

[#undefined-4](./#undefined-4 "mention")

[#undefined-5](./#undefined-5 "mention")

[#id-1.cloudtrail](./#id-1.cloudtrail "mention")

[#id-2.lambda-discord](./#id-2.lambda-discord "mention")

[#id-3.sns](./#id-3.sns "mention")

[#id-4.eventbridge](./#id-4.eventbridge "mention")

[#id-5](./#id-5 "mention")

***

### **\[ 시나리오 안내 ]**

<table data-header-hidden><thead><tr><th width="278.99993896484375"></th><th></th></tr></thead><tbody><tr><td>내용</td><td>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나<br>공격자가 콘솔 접근을 위해서도 사용할 수 있습니다.<br>사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고<br>변경 이력을 검토하는 시나리오를 구현 해봅니다.</td></tr><tr><td>사용 서비스</td><td>Lambda, CloudTrail, SNS, EventBridge</td></tr><tr><td>탐지 조건</td><td>IAM User의 생성, 삭제를 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td>알림을 확인하고 사용자가 대처한다.</td></tr></tbody></table>

***

### **\[ 시나리오 전체적인 흐름 ]**

<figure><img src="../.gitbook/assets/image (195).png" alt=""><figcaption></figcaption></figure>

| **AWS Service** | **Service Purpose**                                                                                                                                                              | **Workbook Usage**                                                                                |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| IAM             | AWS 리소스에 접근할 수 있는 사용자(User), 그들의 권한(Policy), 인증 정보(Access Key 등)를 관리하는 서비스입니다\*\*.\*\* IAM을 통해 개별 사용자 계정을 생성하고, 사용자 별로 권한을 부여할 수 있으며, 사용자 생성/삭제 이력에 대한 감사 추적이 가능합니다.             | IAM 사용자가 생성 및 삭제될 때 CloudTrail이 이를 기록하고, EventBridge를 통해 탐지되며, 알림이 SNS를 통해 전달되는 구조의 핵심 리소스 대상입니다. |
| CloudTrail      | <p>AWS 계정에서 발생하는 모든 사용자 활동과 API 호출을 자동으로 기록하고, 이를 로그로 저장하는 서비스입니다.<br>기본적으로는 최근 90일간의 이벤트만 콘솔에서 조회할 수 있지만,<br>추적을 생성하면 지정한 S3 버킷에 이벤트 로그 파일이 지속적으로 저장되어 장기 보관 및 분석이 가능합니다.</p> | EventBridge에서 CreateUser, DeleteUser로그를 참조할 수 있도록 추적을 생성합니다.                                      |
| EventBridge     | AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.                                                        | CloudTrail에 기록된 로그 중 `CreateUser`, `DeleteUser`를 탐지하고 SNS로 이벤트를 전송합니다.                            |
| SNS             | 발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.                                                                                         | Eventbridge로 부터 이벤트를 수신하면 SNS 주제를 구독하고 있는 엔드포인트(이메일, Lambda)로 이벤트 이력을 전송합니다.                      |
| Lambda          | 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.                                                        | SNS로부터 호출되면 사전 지정한 외부 Webhook(Discord)으로 메시지를 전달합니다.                                              |

***

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

<details>

<summary>1.CloudTrail 추적 생성</summary>

STEP 1) CloudTrail 검색

<figure><img src="../.gitbook/assets/image (9) (1) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (10) (1) (1).png" alt=""><figcaption></figcaption></figure>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 버지니아 리전 선택 후 CloudTrail서비스로 이동한다.\
해당 리전에 생성된 trail이 있을 경우, 추가 생성 없이 2번 단계 으로 넘어간다.



STEP 2 ) CloudTrail 생성

<figure><img src="../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

Create trail 버튼을 클릭해 사용할 추적을 생성한다.



\[ 추적 속성 선택 ]

<figure><img src="../.gitbook/assets/image (92).png" alt=""><figcaption></figcaption></figure>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

**Log file SSE-KMS encryption은 S3 버킷에 로그가 업로드 될 때마다 알림**을 SNS로 보내는 용도이므로 체크 해제한다.

* **Trail name** : ct-trail-monitor
* **Storage location :** Create new S3 bucket
* **Additional settings**
  * **Log file validation :** Enabled 해제



\[ 로그 이벤트 선택 ]

<figure><img src="../.gitbook/assets/image (93).png" alt=""><figcaption></figcaption></figure>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

* **Events** : Management events, Insights events
* **Management events - API activity :** Read, Write 체크



\[ 검토 및 생성 ]

<figure><img src="../.gitbook/assets/image (94).png" alt=""><figcaption></figcaption></figure>

각 단계 검토 후 Create trail 버튼을 클릭하면 추적이 생성된다.



STEP 3) 추적 생성 확인

<figure><img src="../.gitbook/assets/image (95).png" alt=""><figcaption></figcaption></figure>

대시보드에서 정상적으로 추적이 생성되었는지 확인한다.

</details>

<details>

<summary>2.Lambda 함수 생성 및 Discord 연동</summary>

STEP 1) Discord 채널 생성 및 WebHook 설정

\[ 채널 만들기 ]

<figure><img src="../.gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

IAM 사용자 생성/삭제 이벤트 알림을 수신할 채널을 만들어준다.

EventBridge 서비스 화면 오른쪽 상단의 EventBridge Rule을 선택하고 Create rule버튼을 클릭한다.

* **채널 이름** : **`iam-alarm`**



\[ 채널 편집 ]

<figure><img src="../.gitbook/assets/image (97).png" alt=""><figcaption></figcaption></figure>

위와 같이 생성된 채널에서 채널 편집을 클릭한다.



\[ 웹후크 연동 ]

<figure><img src="../.gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>

왼쪽 상단의 설정 목록에서 연동 → 웹후크 만들기를 클릭하여 웹후크 봇을 만들어 준다.



<figure><img src="../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 복사한다.

* **이름** : WEBHOOK\_URL
* **채널** : **`#iam-alarm`** (앞서 생성한 채널 이름 선택)



STEP 2) Lambda 함수 생성

<figure><img src="../.gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

알람을 발송할 함수를 만들기 위해 AWS 콘솔에서 Lambda서비스로 이동한다.



<figure><img src="../.gitbook/assets/image (101).png" alt=""><figcaption></figcaption></figure>

Lambda 서비스 화면 오른쪽 상단의 Create a function 버튼을 클릭한다.



\[ 함수 생성 ]

<figure><img src="../.gitbook/assets/image (102).png" alt=""><figcaption></figcaption></figure>

함수 이름, 런타임 및 아키텍처를 지정하고 **Next**버튼을 클릭한다.

* **Author from scratch** 선택
* **Function name** : **`lambda-iam-user-alarm`**
* **Runtime** : Python 3.13
* **Architecture** : x86\_64



\[ 생성된 함수 확인 ]

<figure><img src="../.gitbook/assets/image (103).png" alt=""><figcaption></figcaption></figure>

정상적으로 Lambda함수가 생성되었는지 확인해준다.



STEP 3) 환경 변수 편집

<figure><img src="../.gitbook/assets/image (104).png" alt=""><figcaption></figcaption></figure>

이후 Configuration → Environment variables로 들어가서 Edit 버튼을 클릭한다.



\[ 환경 변수 추가 ]

<figure><img src="../.gitbook/assets/image (105).png" alt=""><figcaption></figcaption></figure>

Edit environment variables로 이동하여 Add environment variables 버튼을 클릭한다.



\[ 환경 변수에 키와 값 추가 ]

<figure><img src="../.gitbook/assets/image (106).png" alt=""><figcaption></figcaption></figure>

**Key, Value**를 다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

* **Key, Value는 표를 참고**

| Key                   | **용도/설명**            | Value                                                                                           |
| --------------------- | -------------------- | ----------------------------------------------------------------------------------------------- |
| DISCORD\_WEBHOOK\_URL | 디스코드 알림용 Webhook URL | [https://discord.com/api/webhooks/\~\~\~](https://discord.com/api/webhooks/~~~) (알림 받을 웹후크 url) |



STEP 4) Lambda 코드 소스 편집

<figure><img src="../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

Code탭에서 Lambda python 코드를 작성 후 Deploy버튼을 클릭하여 배포해 준다.

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

</details>

<details>

<summary>3.SNS 주제 생성 및 구독 설정</summary>

STEP 1) SNS 검색

<figure><img src="../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

알람을 전송 받을 주제 및 구독을 생성하기 위해 SNS 서비스로 이동한다.



STEP 2) 주제 생성

<figure><img src="../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

좌측 탭에서 Topic으로 이동 후 Create topic 버튼을 클릭한다.



<figure><img src="../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

* **Type** : Standard(표준)
* **Name** : **`sns-iam-user-alarm`**



STEP 3) Email 구독 생성

<figure><img src="../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

생성된 주제 확인 후 Create subscription 버튼을 클릭한다.



\[ 구독 생성 - 세부사항 ]

<figure><img src="../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

* **Protocol** : email
* **Endpoint** : 알람 받을 이메일 주소



STEP 4) 구독한 이메일 인증

<figure><img src="../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

생성된 구독을 확인하면 Status가 Pending Confirmation(확인 대기 중)이다.

입력한 메일 주소로 온 확인 메일을 통해 인증을 진행한다.



\[ 이메일 인증 ]

<figure><img src="../.gitbook/assets/image (114).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

Subscription Confirmation 메일의 Confirm subscription 하이퍼링크를 눌러 접속하면 SNS 구독 등록이 완료된다.



STEP 5) Lambda 구독 생성

<figure><img src="../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

디스코드로 알림을 보내기 위해 위에서 만든 Lambda 구독을 추가 생성한다.



<figure><img src="../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

* **Protocol** : AWS Lambda
* **Endpoint** : 위에서 생성한 Lambda (**`lambda-iam-user-alarm`**) 선택

</details>

<details>

<summary>4.EventBridge 규칙 생성</summary>

STEP 1) EventBridge 검색

<figure><img src="../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

Lambda 함수를 주기적으로 실행하기 위해 EventBridge 서비스로 이동한다.



STEP 2) EventBridge 생성

<figure><img src="../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

EventBridge 서비스 화면 오른쪽 상단의 EventBridge Rule을 선택하고 Create rule버튼을 클릭한다.



\[ 상세 규칙 설정 ]

<figure><img src="../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

규칙 이름, 설명, Event bus 종류, 규칙 유형(이벤트 패턴 기반 or 스케줄 기반) 설정 후 **Next버튼**을 클릭한다.

* **Name** : **`eventbridge-iam-user-change`**
* **Event Bus** : default
* **Rule Type** : Rule with an event pattern (이벤트 패턴이 있는 규칙)



\[ 이벤트 패턴 작성 ]

<figure><img src="../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

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



\[ 대상 선택 ]

<figure><img src="../.gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

* **Target Types** : AWS service
* **Select a target** : SNS topic
* **Target location** : Target in this account
* **Topic** : 앞서 생성한 sns topic 선택(**`sns-iam-user-alarm`**)
* **Execution role** : Create a new role for this specific resources (이 특정 리소스에 대해 역할 생성)
* **Role name** : 자동 할당



\[ 태그 구성 (선택) ]

<figure><img src="../.gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

태그 구성은 선택 사항이므로 Next버튼을 클릭한다.



\[ 검토 및 생성 ]

<figure><img src="../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인



STEP 3) 생성된 규칙 확인

<figure><img src="../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

규칙이 정상적으로 생성되었는지 확인한다.

</details>

<details>

<summary>5.테스트</summary>

> IAM 콘솔에서 사용자를 생성하고 삭제하면서 이벤트를 발생시킨다.

**\[ 탐지 이벤트 안내 ]**

| 이벤트 이름           | 설명         | **탐지 목적**                      |
| ---------------- | ---------- | ------------------------------ |
| **`CreateUser`** | IAM 사용자 생성 | **생성 탐지** - 권한 남용, 내부 위협행위 식별  |
| **`DeleteUser`** | IAM 사용자 삭제 | **삭제 탐지** - 사용자를 없애려는 공격 행위 식별 |



<figure><img src="../.gitbook/assets/image (4) (1) (1).png" alt=""><figcaption></figcaption></figure>

이벤트 테스트를 위해 IAM 콘솔로 이동한다.



\[CreateUser이벤트 발생 ]

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

좌측 탭에서 Users를 선택 후 Create User 버튼을 클릭한다.



<figure><img src="../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

아래 사항 외에는 기본값 그대로 진행한 후 **Create User버튼**을 눌러 테스트 사용자를 생성한다.

* Name : **`IAM-test-user`**



\[DeleteUser이벤트 발생 ]

<figure><img src="../.gitbook/assets/image (4) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

앞서 만든 사용자를 체크한 후 Delete버튼을 클릭해 삭제한다.



\[ CloudTrail에서 이벤트 발생 기록 확인 ]

<figure><img src="../.gitbook/assets/image (5) (1) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (6) (1) (1).png" alt=""><figcaption></figcaption></figure>

IAM 사용자를 생성 하고 삭제하면서 CloudTrail에 CreaterUser, DeleteUser 이벤트가 기록된 것을 확인할 수 있다.



\[ Email 알림 확인 ]

<figure><img src="../.gitbook/assets/image (7) (1) (1).png" alt=""><figcaption></figcaption></figure>



\[ Discord 알림 확인 ]

<figure><img src="../.gitbook/assets/image (8) (1) (1).png" alt=""><figcaption></figcaption></figure>



</details>

