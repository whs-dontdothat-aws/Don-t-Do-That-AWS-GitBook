# 로그 그룹 삭제 또는 변경 탐지

**\[ 목차 ]**

[#undefined](./#undefined "mention")

[#undefined-1](./#undefined-1 "mention")

[#undefined-2](./#undefined-2 "mention")

[#undefined-3](./#undefined-3 "mention")

[#undefined-5](./#undefined-5 "mention")

[#id-1.-cloudtrail](./#id-1.-cloudtrail "mention")

[#id-2.-lambda-discord](./#id-2.-lambda-discord "mention")

[#id-3.-sns](./#id-3.-sns "mention")

[#id-4.-eventbridge](./#id-4.-eventbridge "mention")

[#id-5](./#id-5 "mention")

***

### **\[ 시나리오 안내 ]**

<table><thead><tr><th width="148"></th><th></th></tr></thead><tbody><tr><td>내용</td><td>공격자가 분석 회피를 위해 로그 그룹을 삭제하거나 로그 수집을 변경하는 시도 탐지합니다.</td></tr><tr><td>사용 서비스</td><td>CloudTrail, EventBridge, SNS</td></tr><tr><td>탐지 조건</td><td>1. 로그 그룹에 대해서 정책을 삭제하거나 구성을 변경하려는 시도를 탐지합니다.<br>2. DeleteLogGroup, PutRetentionPolicy 등 다양한 조건을 사전 학습하여 어떤 행위가 로그 그룹에 영향을 미칠 수 있는지 검토합니다.<br>3. 해당 조건을 토대로 로그 그룹에 대한 알람을 진행합니다.</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr></tbody></table>

#### 실습 개요

* 이 워크북에서는 로그 그룹이 삭제되거나 구성이 변경 되었을 때 자동 탐지하고 알림을 받는 방법을 학습합니다.
* 로그 그룹은 CloudWatch Log를 그룹화해 관리하는 논리적 컨테이너입니다. 삭제 시 분석 회피를 위한 악성 행위로 간주할 수 있습니다.
* 본 워크북에서는 CloudTrail과 EventBridge, SNS 등을 활용해 로그 그룹 삭제/변경 이벤트 발생 시 이메일 및 디스코드 알림을 받도록 구현하는 방법을 실습합니다.

#### 학습 목표

* AWS에서 로그 그룹의 삭제 및 변경 이벤트를 탐지하는 방법을 학습합니다.
* CloudTrail과 EventBridge를 활용해 이벤트를 수집하고, 이를 SNS와 연동하여 이메일 및 디스코드 알림을 전송하는 실습을 진행합니다.
* 이벤트에 대한 탐지 조건을 JSON 코드로 정의하여 실시간 대응 체계를 구축하는 과정을 이해합니다.
* 실제로 로그 그룹을 생성한 뒤 삭제하거나 변경하여 이벤트 탐지가 정상적으로 동작하는지 실습을 통해 검증합니다.

***

### **\[ 시나리오 전체적인 흐름 ]**

<figure><img src="../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

| **AWS Service** | **Service Purpose**                                                                                                            | **Workbook Usage**                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| CloudTrail      | AWS 계정 내에서 발생하는 모든 API 호출 이벤트를 기록하는 서비스입니다. 사용자의 콘솔 작업, AWS CLI, SDK, 서비스 간 호출 등 모든 요청을 JSON 로그 형식으로 저장하며, 보안 및 감사 목적으로 활용됩니다. | 로그 그룹을 변경하거나 삭제하는 API 이벤트인 `DeleteLogGroup`, `PutRetentionPolicy` 를 기록합니다.                                               |
| EventBridge     | AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.      | CloudTrail이 기록한 이벤트 로그 중 작성한 이벤트 패턴과 일치하는 이벤트를 **실시간으로 탐지**하기 위해 사용합니다.                                                  |
| SNS             | 발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.                                       | CloudWatch 삭제, 보존 기간 변경 등 이벤트가 발생했을 때, 이를 **Email로 관리자에게 자동 통보**하기 위해 SNS 주제를 생성하고, 알림을 전송 받기 위해 해당 주제를 구독한 Email을 인증합니다 |
| Lambda          | 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.      | SNS의 주제와 연동하여 이벤트가 발생하면 작성된 Lambda 함수를 통해 Discord로 알림을 보내주도록 설정합니다.                                                      |
| LogGroup        | CloudTrail 이벤트 로그를 저장하고 모니터링하는 CloudWatch Log를 그룹화해 관리하는 논리적 컨테이너입니다. 모니터링, 접근제어 등 설정을 공유할 수 있습니다.                             | 공격자가 분석 회피를 위해 로그 그룹을 삭제하거나 로그 보존 기간을 변경하는 대상입니다.                                                                        |

#### 참고 사항

* 본 워크북에서 생성되는 리소스 중 일부는 Terraform code를 통해 제공되고 있습니다.
* 주요 리소스에 대한 사전 구성을 필요로 하는 경우 하단의 “xx. Terraform으로 리소스 구현” 에서 내용을 참고하실 수 있습니다.
* 본 서비스의 주요 리소스는 “ap-northeast-2(Seoul)”에서 리전에서 진행됩니다. 주요 서비스 및 기능은 제공되는 서비스 리전에 따라 다를 수 있습니다.
* 리소스명은 해당 시나리오에 맞게 임의로 설정하였으며 사용자가 원하는 이름으로 바꿔도 무방합니다.



**\[ 콘솔 리소스명 ]**

<table data-header-hidden><thead><tr><th width="203">리소스 종류</th><th width="250">리소스명 설정</th><th>목적</th></tr></thead><tbody><tr><td><strong>리소스 종류</strong></td><td><strong>리소스명 설정</strong></td><td><strong>목적</strong></td></tr><tr><td>CloudTrail Trail</td><td><strong><code>ct-trail-monitor</code></strong></td><td>AWS 계정 내 API 활동을 감시하고 로그를 S3로 저장</td></tr><tr><td>Discord 채널</td><td><strong><code>log-group-alarm</code></strong></td><td>SNS에 연동된 Lambda 함수를 통해 알림 메시지를 수신하고 확인할 수 있는 알림 채널</td></tr><tr><td>Lambda Function</td><td><strong><code>lambda-loggroup-alarm</code></strong></td><td>SNS 메시지를 파싱하여 Discord Webhook으로 이상 행위 알림 전송</td></tr><tr><td>SNS Topic</td><td><strong><code>sns-loggroup-alarm</code></strong></td><td>이벤트 발생 시 이메일 및 Lambda로 알림을 전송하는 주제</td></tr><tr><td>EventBridge Rule</td><td><strong><code>eventbridge-loggroup-change</code></strong></td><td>특정 CloudTrail 이벤트 발생 시 SNS로 이벤트 전송하는 트리거 역할</td></tr><tr><td>CloudWatch Log Group</td><td><strong><code>loggroup-test</code></strong></td><td>로그 그룹 삭제 및 변경 시 알림이 발생하는지 확인하기 위해 생성하는 테스트리소스</td></tr></tbody></table>

***

### **\[ 시나리오 상세 구현 과정 ]**

<details>

<summary>1. CloudTrail 추적 생성</summary>

**STEP 1) CloudTrail 검색**

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 **CloudTrail서비스**로 이동한다. 리전에 이미 생성된 trail이 있을 경우, 추가 생성 없이 **2번 단계** 으로 넘어간다.



**STEP 2 ) CloudTrail 생성**

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Create trail 버튼을 클릭해 사용할 추적을 생성한다.



**\[ 추적 속성 선택 ]**

<figure><img src="../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>예시 사진</p></figcaption></figure>

CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

**Log file SSE-KMS encryption은 S3 버킷에 로그가 업로드 될 때마다 알림**을 SNS로 보내는 용도이므로 체크 해제한다.

* **Trail name** : ct-trail-monitor
* **Storage location :** Create new S3 bucket
* **Additional settings - Log file validation :** Enabled 해제



**\[ 로그 이벤트 선택 ]**

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption><p>예시 사진</p></figcaption></figure>

로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.

* **Events** : Management events, Insights events
* **Management events - API activity :** Read, Write 체크



**\[ 검토 및 생성 ]**

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption><p>예시 사진</p></figcaption></figure>

각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.



**STEP 3) 추적 생성 확인**

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption><p>예시 사진</p></figcaption></figure>

대시보드에서 정상적으로 추적이 생성되었는지 확인한다.

</details>

<details>

<summary>2. Lambda 함수 생성 및 Discord 연동</summary>

**STEP 1) Discord 채널 생성 및 WebHook 설정**

**\[ 채널 만들기 ]**

<div align="left"><figure><img src="../.gitbook/assets/image (6).png" alt="" width="467"><figcaption></figcaption></figure></div>

로그 그룹의 변경/삭제 이벤트 알림을 수신 할 채널을 만들어준다.

* **채널 이름** : **`log-group-alarm`**



**\[ 채널 편집 ]**

<div align="left"><figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure></div>

위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.



**\[ 웹후크 연동 ]**

<div align="left"><figure><img src="../.gitbook/assets/image (8).png" alt="" width="563"><figcaption></figcaption></figure></div>

왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.



<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>



**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 복사한다.

* **이름** : WEBHOOK\_URL
* **채널** : **`#log-group-alarm`** (앞서 생성한 채널 이름 선택)



**STEP 2) Lambda 함수 생성**

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

알람을 발송할 함수를 만들기 위해 AWS 콘솔에서 **Lambda 서비스**로 이동한다.



<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.



**\[ 함수 생성 ]**

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

함수 이름, 런타임 및 아키텍처를 지정하고 **Next**버튼을 클릭한다.

* **Author from scratch** 선택
* **Function name** : **`lambda-loggroup-alarm`**
* **Runtime** : Python 3.13
* **Architecture** : x86\_64



**\[ 생성된 함수 확인 ]**

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

생성된 Lambda 함수를 확인할 수 있다.



**STEP 3) 환경 변수 편집**

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

이후 Configuration → Environment variables로 들어가서 **Edit** 버튼을 클릭한다.





**\[ 환경 변수에 키와 값 추가 ]**

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

**Key, Value**를 \*\*\*\*다음과 같이 추가한 이후 **Save**버튼을 눌러 환경 변수를 추가해 준다.

* **Key, Value는 표를 참고**

| Key                   | **용도/설명**            | Value                                                                                           |
| --------------------- | -------------------- | ----------------------------------------------------------------------------------------------- |
| DISCORD\_WEBHOOK\_URL | 디스코드 알림용 Webhook URL | [https://discord.com/api/webhooks/\~\~\~](https://discord.com/api/webhooks/~~~) (알림 받을 웹후크 url) |



**STEP 4) Lambda 코드 소스 편집**

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

Code탭에서 Lambda python 코드를 작성 후 Deploy버튼을 클릭하여 배포한다.

```python
import os, json, urllib3

http = urllib3.PoolManager()
WEBHOOK = os.environ['DISCORD_WEBHOOK_URL'] # 위에서 정의한 key와 동일하게 작성

def lambda_handler(event, context):
    for rec in event["Records"]:
        msg = json.loads(rec["Sns"]["Message"])
        detail = msg.get("detail", {})
        
        # Discord로 보낼 메시지 내용 포맷팅
        content = (
            f"**[ 로그 그룹 변경 이벤트 탐지 ]**\n"
            f"- 이벤트 이름: `{detail.get('eventName')}`\n"
            f"- 로그 그룹: `{detail.get('requestParameters', {}).get('logGroupName', 'N/A')}`\n"
            f"- 사용자: `{detail.get('userIdentity', {}).get('arn', 'Unknown')}`\n"
            f"- 시간: {msg.get('time')}"
        )
        # 웹훅을 통해 Discord로 HTTP POST 요청(알림 전송)
        http.request(
            "POST", WEBHOOK,
            body=json.dumps({"content": content}).encode(),
            headers={"Content-Type": "application/json"}
        )
```

</details>

<details>

<summary>3. SNS 주제 생성 및 구독 설정</summary>

**STEP 1) SNS 검색**

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

알람을 전송 받을 주제 및 구독을 생성하기 위해 **SNS 서비스**로 이동한다.



STEP 2) 주제 생성

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

좌측 탭에서 Topic으로 이동 후 **Create topic 버튼**을 클릭한다.



<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

* **Type** : Standard(표준)
* **Name** : **`sns-loggroup-alarm`**



**STEP 3) Email 구독 생성**

<div align="left"><figure><img src="../.gitbook/assets/image (20).png" alt="" width="563"><figcaption></figcaption></figure></div>

생성된 주제 확인 후 **Create subscription 버튼**을 클릭한다.



**\[ 구독 생성 - 세부사항 ]**

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

* **Protocol** : Email
* **Endpoint** : 이메일 주소



**STEP 4) 구독한 이메일 인증**

<figure><img src="../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

생성된 구독을 확인하면 Status가 Pending Confirmation(확인 대기 중)이다.

입력한 메일 주소로 온 확인 메일을 통해 인증을 진행한다.



**\[ 이메일 인증 ]**

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure></div>

Subscription Confirmation 메일의 **Confirm subscription 하이퍼링크**를 눌러 접속하면 SNS 구독 등록이 완료된다.



**STEP 5) Lambda 구독 생성**

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

디스코드로 알림을 보내기 위해 위에서 만든 Lambda 구독을 추가 생성한다.



<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

* **Protocol** : AWS Lambda
* **Endpoint** : 위에서 생성한 Lambda **(`lambda-loggroup-alarm`)** 선택



**\[ Email, Lambda 구독 확인 ]**

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

최종적으로 위와 같이 Email, Lambda가 구독 설정되어 SNS주제와 연동된 모습을 확인 할 수 있다.

</details>

<details>

<summary>4. EventBridge 규칙 생성</summary>

**STEP 1) EventBridge 검색**

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Lambda 함수를 주기적으로 실행하기 위해 AWS 콘솔에서 **EventBridge 서비스**로 이동한다.



**STEP 2) EventBridge 생성**

<figure><img src="../.gitbook/assets/image (1) (2).png" alt=""><figcaption></figcaption></figure>

**EventBridge** 서비스 화면 오른쪽 상단의 **EventBridge Rule**을 선택하고 **Create rule**버튼을 클릭한다.



**\[ 상세 규칙 설정 ]**

<figure><img src="../.gitbook/assets/image (2) (2).png" alt=""><figcaption></figcaption></figure>

규칙 이름, 설명, Event bus 종류, 규칙 유형(이벤트 패턴 기반 or 스케줄 기반) 설정 후 **Next버튼**을 클릭한다.

* **Name** : **`eventbridge-loggroup-change`**
* **Event Bus** : default
* **Rule Type** : Rule with an event pattern (이벤트 패턴이 있는 규칙)



**\[ 이벤트 패턴 작성 ]**

<figure><img src="../.gitbook/assets/image (3) (2).png" alt=""><figcaption></figcaption></figure>

탐지할 이벤트 조건을 설정을 설정하고 **Next**버튼을 클릭한다.

* **Event Source :** Other
*   **Event pattern** : Custom pattern (JSON editor)

    사용자가 원하는 조건만 감지할 수 있도록 JSON으로 직접 작성
*   로그 그룹 변경 또는 삭제 시도를 탐지하는 JSON 코드

    ```json
    {
      "detail-type": ["AWS API Call via CloudTrail"],
      "source": ["aws.logs"],
      "detail": {
        "eventName": [
          "DeleteLogGroup",         
          "PutRetentionPolicy",       
          "PutSubscriptionFilter",   
          "DeleteSubscriptionFilter",
          "PutResourcePolicy",      
          "DeleteResourcePolicy"    
        ]
      }
    }
    ```

**\[ 설정한 이벤트 안내 ]**

<table><thead><tr><th width="203">이벤트 이름</th><th width="166">설명</th><th>탐지 목적</th></tr></thead><tbody><tr><td><strong><code>DeleteLogGroup</code></strong></td><td>로그 그룹 삭제</td><td>중요 로그 삭제로 인한 데이터 유실 및 보안 사고 방지</td></tr><tr><td><strong><code>PutRetentionPolicy</code></strong></td><td>로그 보존 기간 변경</td><td>로그 보존 정책 조작 시도 감지로 컴플라이언스 및 감사 추적 유지</td></tr><tr><td><strong><code>PutSubscriptionFilter</code></strong></td><td>로그 필터 추가</td><td>외부로의 로그 데이터 전달/유출 및 무단 전달 경로 추가 탐지</td></tr><tr><td><strong><code>DeleteSubscriptionFilter</code></strong></td><td>로그 필터 제거</td><td>필터 삭제를 통한 로그 유출 탐지 차단 등 보안 설정 변경 감지</td></tr><tr><td><strong><code>PutResourcePolicy</code></strong></td><td>리소스 정책 설정</td><td>로그 그룹 정책 오남용 방지 및 인증 관리</td></tr><tr><td><strong><code>DeleteResourcePolicy</code></strong></td><td>리소스 정책 제거</td><td>정책 삭제를 통한 무단 접근 위험 증대 및 보안 정책 훼손 탐지</td></tr></tbody></table>



{% hint style="success" %}
**이벤트 추가 설명과 해당 이벤트 발생시키는 CLI 정리**



1. 로그 그룹 삭제 (`DeleteLogGroup`) 로그 그룹 삭제는 AWS에서 `DeleteLogGroup` API 호출을 통해 로그 그룹 전체와 그 안의 로그를 모두 제거하는 정책이다.

```bash
aws logs delete-log-group --log-group-name "test" --region ap-northeast-2
```

로그 그룹 삭제가 중요한 이유는 다음과 같다.

* 공격자가 로그 그룹 자체를 삭제하면 **과거 모든 로그를 완전히 제거**하여 추적이 불가능하게 된다.
* 이와 같은 행동은 **증거 은폐 시도**로 간주될 수 있으며, 반드시 탐지되어야 한다.

따라서 로그 그룹 삭제는 고위험 이벤트이며 실시간 알림 및 분석이 필요하다.

2. 보존 설정 변경 (`PutRetentionPolicy`) 보존 설정 변경은 AWS에서 `PutRetentionPolicy` 로 보존 기간을 설정하는 정책이다. 해당 정책은 지정된 이후의 로그는 자동으로 삭제된다.

```bash
aws logs put-retention-policy --log-group-name "test" --retention-in-days 7 --region ap-northeast-2
```

보존 기간 설정이 중요한 이유는 다음과 같다.

* 공격자가 보존 기간을 짧게 설정하면 과거 로그가 자동 삭제되어 추적이 불가하다.
* 공격자가 보존 정책을 아예 없애거나 변경하면 흔적 지우는 시도가 있었음을 알 수 있다.

결국 해당 정책은 보안 관점에서 이상 행위 시그널로 판단할 수 있다.

3. 로그 필터 추가 (`PutSubscriptionFilter`) 로그 필터 추가는 AWS에서 `PutSubscriptionFilter` 를 통해 로그 그룹의 로그를 외부 대상(Kinesis, Lambda, Firehose 등)으로 실시간 전송하는 정책이다.

```bash
aws logs put-subscription-filter \\
  --log-group-name "test" \\
  --filter-name "filter" \\
  --filter-pattern "" \\
  --destination-arn "arn:aws:lambda:ap-northeast-2:500113025916:function:lambda-loggroup-alarm"
```

로그 필터 설정이 중요한 이유는 다음과 같다.

* 공격자가 로그를 특정 외부 대상(Lambda 등)으로 전송하게 하여, **민감 정보 유출**이나 **비인가 분석**을 시도할 수 있다.
* 필터가 갑작스레 추가될 경우, **로그 흐름 변경** 여부를 반드시 확인해야 한다.

이런 설정은 외부 유출 또는 우회 경로로 악용될 수 있으므로 주의 깊게 감시해야 한다.

4. 로그 필터 제거 (`DeleteSubscriptionFilter`) 로그 필터 제거는 AWS에서 `DeleteSubscriptionFilter` 를 통해 로그 그룹에서 설정된 로그 전송 필터를 삭제하는 정책이다.

```bash
aws logs delete-subscription-filter \\
  --log-group-name "test" \\
  --filter-name "filter"
```

* 필터가 제거되면 **로그가 더 이상 외부 시스템(SIEM, Lambda 등)으로 전달되지 않는다**.
* 이는 탐지를 피하기 위한 공격자의 **감시 우회** 시도일 수 있다.

따라서 기존 필터가 삭제되는 이벤트는 이상 징후로 판단하고 대응해야 한다.

5. 리소스 정책 설정 (`PutResourcePolicy`) 리소스 정책 설정은 AWS에서 `PutResourcePolicy` 를 통해 CloudWatch Logs 서비스에 대한 IAM 리소스 기반 권한 정책을 설정하는 정책이다.

```bash
aws logs put-resource-policy \\
  --policy-name "test-policy" \\
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": "logs:CreateLogStream",
        "Resource": "*"
      }
    ]
  }'
```

리소스 정책 설정이 중요한 이유는 다음과 같다.

* 공격자가 외부 계정 또는 자신에게 로그 그룹 접근 권한을 부여할 수 있다.
* 로그에 대한 **비인가 접근 가능성**이 증가하여, 보안 위협이 된다.

결국 정상적인 연동이 아닌 경우, 반드시 의심 행위로 간주되어야 한다.

6. 리소스 정책 제거 (`DeleteResourcePolicy`) 리소스 정책 제거는 AWS에서 `DeleteResourcePolicy` 를 통해 CloudWatch 로그의 IAM 정책을 삭제하는 정책이다.

```bash
aws logs delete-resource-policy --policy-name test-policy
```

리소스 정책 제거가 중요한 이유는 다음과 같다.

* 기존에 설정되어 있던 **접근 제어가 무력화**될 수 있다.
* 공격자는 접근 제한을 제거함으로써 **제한 없는 로그 조회나 수정**을 시도할 수 있다.

정책이 삭제되면 누구든 로그 리소스에 접근할 수 있으므로, **즉각적인 탐지와 대응이 필수**이다.
{% endhint %}



**\[ 대상 선택 ]**

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

이벤트가 감지되었을 때 실행할 대상 지정하고 **Next**버튼을 클릭한다.

* **Target Types** : AWS service
* **Select a target** : SNS topic
* **Target location** : Target in this account
* **Topic** : 앞서 생성한 sns topic 선택(**`sns-loggroup-alarm`**)
* **Execution role** : Create a new role for this specific resources (이 특정 리소스에 대해 역할 생성)
* **Role name** : 자동 할당



\[ 태그 구성 (선택) ]

<figure><img src="../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.



**\[ 검토 및 생성 ]**

<figure><img src="../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인



**STEP 3) 생성된 규칙 확인**

<figure><img src="../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

규칙이 정상적으로 생성되었는지 확인한다.

</details>

<details>

<summary>5. 테스트</summary>

> Cloudwatch 콘솔과 CLI로 이벤트를 발생시킨다.



**\[ 탐지 이벤트 안내 ]**

| 이벤트 이름                         | 설명          | 탐지 목적                                |
| ------------------------------ | ----------- | ------------------------------------ |
| **`DeleteLogGroup`**           | 로그 그룹 삭제    | 중요 로그 삭제로 인한 데이터 유실 및 보안 사고 방지       |
| **`PutRetentionPolicy`**       | 로그 보존 기간 변경 | 로그 보존 정책 조작 시도 감지로 컴플라이언스 및 감사 추적 유지 |
| **`PutSubscriptionFilter`**    | 로그 필터 추가    | 외부로의 로그 데이터 전달/유출 및 무단 전달 경로 추가 탐지   |
| **`DeleteSubscriptionFilter`** | 로그 필터 제거    | 필터 삭제를 통한 로그 유출 탐지 차단 등 보안 설정 변경 감지  |
| **`PutResourcePolicy`**        | 리소스 정책 설정   | 로그 그룹 정책 오남용 방지 및 인증 관리              |
| **`DeleteResourcePolicy`**     | 리소스 정책 제거   | 정책 삭제를 통한 무단 접근 위험 증대 및 보안 정책 훼손 탐지  |



1. **콘솔 테스트**

<figure><img src="../.gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

이벤트 테스트를 위해 CloudWatch로 이동한다



<figure><img src="../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (74).png" alt=""><figcaption></figcaption></figure>

아래와 같이 입력하여 로그 그룹을 생성해준다.

* **Log group name**: **`loggroup-test`**
* **Retention setting**: Never expire (만기 없음)
* **Log class**: standard



<figure><img src="../.gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

만들어진 로그 그룹을 선택하고 Actions를 눌러 로그 그룹 삭제, 로그 필터 추가, 리소스 정책 생성을 테스트해본다.



2. **AWS CLI를 통한 테스트**

[#id-4.-eventbridge](./#id-4.-eventbridge "mention")에서 **이벤트 추가 설명과 해당 이벤트 발생시키는 CLI 정리**를 참조하여 CLI로 이벤트를 발생시켜 테스트를 진행한다.



**\[ 결과 확인 ]**



1. **`DeletelogGroup`**

<figure><img src="../.gitbook/assets/image (78).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure></div>



2. **`PutRetentionPolicy`**

<figure><img src="../.gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure></div>



3. **`PutSubscriptionFilter`**

<figure><img src="../.gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure></div>



4. **`DeleteSubscriptionFilter`**

<figure><img src="../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure></div>



5. **`PutResourcePolicy`**

<figure><img src="../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure></div>

이때 알림을 확인해 보면 LogGroup에 N/A로 표시되는 것을 알 수 있는데 이는 CLI 명령에서 다수의 로그 그룹을 대상으로 지정하여 다음과 같이 표시되는 것이다.



6. **`DeleteResourcePolicy`**

<figure><img src="../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

<div align="left"><figure><img src="../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure></div>

</details>

