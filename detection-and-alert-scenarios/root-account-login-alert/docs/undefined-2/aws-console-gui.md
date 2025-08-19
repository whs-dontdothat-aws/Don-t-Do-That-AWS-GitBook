# AWS Console (GUI)

### **\[ 시나리오 상세 구현 과정 ]**

<details>

<summary><strong>1. S3 버킷 및 CloudTrail 추적 생성</strong></summary>

**STEP 1) CloudTrail 검색**

<figure><img src="../.gitbook/assets/image (196).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
AWS 계정 내에서 발생하는 API 호출 및 활동 내역을 자동으로 기록하고 추적하기 위해 **CloudTrail 서비스**로 이동한다.
{% endhint %}



**STEP 2) CloudTrail 생성**

<figure><img src="../.gitbook/assets/image (198).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**Create trail** 버튼을 클릭해 사용할 추적을 생성한다.
{% endhint %}



**\[ 추적 속성 선택 ]**

<figure><img src="../.gitbook/assets/image (200).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
CloudTrail 트레일(추적)의 기본 설정을 지정 후 **Next**버튼을 클릭한다.

**Log file SSE-KMS encryption은 S3 버킷에 로그가 업로드 될 때마다 알림**을 SNS로 보내는 용도이므로 체크 해제한다.



* **Trail name** : <mark style="color:$danger;">**`ct-snapshot-monitor`**</mark>
* **Storage location :** Create new S3 bucket
* **Additional settings - Log file validation :** Enabled
{% endhint %}



**\[ 로그 이벤트 선택 ]**

<figure><img src="../.gitbook/assets/image (201).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
로그 이벤트, 이벤트 관리 옵션 선택 후 **Next**버튼을 클릭한다.



* **Events** : Management events, Insights events
* **Management events - API activity :** Read, Write 체크&#x20;
{% endhint %}



**\[** **검토 및 생성 ]**

<figure><img src="../.gitbook/assets/image (202).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
각 단계 검토 후 **Create trail** 버튼을 클릭하면 추적이 생성된다.
{% endhint %}



**STEP 3) 추적 생성 확인**

<figure><img src="../.gitbook/assets/image (203).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
대시보드에서 정상적으로 추적이 생성되었는지 확인한다.
{% endhint %}

</details>

<details>

<summary>2. SNS 주제 생성 및 구독 설정</summary>

**STEP 1) SNS 검색**

<figure><img src="../.gitbook/assets/image (204).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
알람을 전송 받을 주제 및 구독을 생성하기 위해 AWS 콘솔에서 **SNS 서비스**로 이동한다.
{% endhint %}



**STEP 2) 주제 생성**

<figure><img src="../.gitbook/assets/image (205).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
좌측 탭에서 Topic으로 이동 후 **Create** topic 버튼을 클릭한다.
{% endhint %}



**\[ 주제 생성 - 세부 사항 ]**

<figure><img src="../.gitbook/assets/image (206).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
* **Type** : Standard
* **Name** : <mark style="color:$danger;">`sns-snapshot-alarm`</mark>
{% endhint %}



**STEP 3 ) 구독 생성**

<figure><img src="../.gitbook/assets/image (207).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
생성된 주제 확인 후 **Create subscription**을 누른다.
{% endhint %}



**\[ 구독 생성 - 세부사항 ]**

<figure><img src="../.gitbook/assets/image (208).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
* **Protocol** : email
* **Endpoint** : 알람 받을 이메일 주소
{% endhint %}



**STEP 4 ) 구독한 이메일 인증**

**\[ 이메일 인증 ]**

<figure><img src="../.gitbook/assets/image (209).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
알림 수신을 설정한 Email에 Subscription Confirmation 메일을 전송 받고 \*\*\*\*생성된 구독 확인 후 메일 인증을 해야 한다.
{% endhint %}

<figure><img src="../.gitbook/assets/image (210).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**Confirm subscription**을 눌러 접속하면 정상적으로 SNS 구독 등록이 된 것이다.
{% endhint %}

</details>

<details>

<summary>3. Lambda 함수 생성 및 Discord 연동</summary>

**STEP 1) Discord 채널 생성 및 WebHook 설정**

**\[ 채널 만들기 ]**

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
이벤트에 관한 알림을 수신 할 채널을 만들어준다.



* **채널 이름** : <mark style="color:$danger;">**`ebs-snapshot-alarm`**</mark>&#x20;
{% endhint %}



**\[ 채널 편집 ]**

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
위와 같이 생성된 채널에서 **채널 편집**을 클릭한다.
{% endhint %}



**\[ 웹후크 연동 ]**

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
왼쪽 상단의 설정 목록에서 **연동 → 웹후크 만들기**를 클릭하여 웹후크 봇을 만들어 준다.
{% endhint %}



**\[ 웹후크 URL 복사 ]**

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**웹후크 URL 복사** 버튼을 클릭해 Lambda에서 사용할 URL을 복사한다.



* **이름** : WEBHOOK\_URL
* **채널** : # (앞서 생성한 채널 이름 선택)
{% endhint %}



**STEP 2) Lambda 함수 생성**



</details>

알람을 발송할 함수를 만들기 위해 AWS 콘솔에서 **Lambda 서비스**로 이동한다.

\</aside>

![image.png](attachment:f1eebfa6-6882-4e1c-b3b2-81751385349b:image.png)

\<aside>

Lambda 서비스 화면 오른쪽 상단의 **Create a function** 버튼을 클릭한다.

\</aside>

**\[ 함수 생성 ]**

![스크린샷 2025-07-23 오전 5.16.48.png](attachment:332fdb0b-e32b-4a7e-82c0-8a992265d111:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.16.48.png)

\<aside>

함수 이름, 런타임 및 아키텍처를 지정하고 **Create function** 버튼을 클릭한다.

* **Author from scratch** 선택
* **Function name** : **`lambda-ebs-snapshot-alarm`**
* **Runtime** : Python 3.13
* **Architecture** : x86\_64 \</aside>

**\[ 생성한 함수 확인 ]**

![스크린샷 2025-07-23 오전 5.20.27.png](attachment:06886af7-a3cd-45c2-bff0-f1b3f02f7cd9:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.20.27.png)

\<aside>

정상적으로 Lambda함수가 생성되었는지 확인해준다.

\</aside>

**STEP 3) 환경 변수 편집**

![image.png](attachment:63da8f49-6998-4271-8eba-cca6ae0054cf:09cf90a2-36a6-4dde-8cdc-a4b633e0670f.png)

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

![스크린샷 2025-07-23 오전 5.27.49.png](attachment:6739cb4e-2686-49f8-83c8-74bcd9020aa1:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.27.49.png)

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
            user_arn = detail.get('userIdentity', {}).get('arn', 'Unknown')
            source_ip = detail.get('sourceIPAddress', 'Unknown')
            aws_region = outer_msg.get('region', 'Unknown')
            account_id = outer_msg.get('account', 'Unknown')
            event_time_utc = detail.get('eventTime', '')[:19]

            # 시간 변환 (UTC → KST)
            try:
                event_time_kst = datetime.strptime(event_time_utc, '%Y-%m-%dT%H:%M:%S') + timedelta(hours=9)
                time_str = event_time_kst.strftime('%Y-%m-%d %H:%M:%S') + " (KST)"
            except:
                time_str = 'Unknown'

            # Discord 메시지 구성
            discord_msg = {
                "content": f"**[ EBS 스냅샷 이벤트 감지됨 ]**\\n"
                           f"• 이벤트 이름: `{event_name}`\\n"
                           f"• 발생 시간: `{time_str}`\\n"
                           f"• 사용자 ARN: `{user_arn}`\\n"
                           f"• 소스 IP: `{source_ip}`\\n"
                           f"• 리전: `{aws_region}`\\n"
                           f"• 계정 ID: `{account_id}`"
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

**STEP 5) lambda 트리거 추가**

![스크린샷 2025-07-23 오전 5.30.56.png](attachment:3455d52b-4877-4cdc-a00d-7df57ed4669b:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.30.56.png)

\<aside>

생성한 Lambda함수의 다이어그램 왼쪽 하단의 **Add trigger**버튼을 클릭한다.

\</aside>

**\[ Lambda 트리거 - SNS ]**

![스크린샷 2025-07-23 오전 5.32.03.png](attachment:778e1305-1429-4679-81cd-458a9eb32944:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.32.03.png)

\<aside>

트리거 구성, sns 주제를 지정하고 **Add**버튼을 클릭한다.

* **Trigger configuration** : SNS
* **SNS topic** : 앞서 생성한 SNS 주제 선택 \</aside>

**STEP 6) 추가된 트리거 확인**

![스크린샷 2025-07-23 오전 5.33.46.png](attachment:d083f07f-85fb-4302-9748-c5fda6e84278:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.33.46.png)

\<aside>

SNS가 정상적으로 트리거링 되었고 Discord에 알림을 보내기 위한 설정을 마쳤다.

\</aside>

#### **4. EventBridge 규칙 생성**

**STEP 1) EventBridge 검색**

![스크린샷 2025-07-22 오전 1.14.15.png](attachment:1fc38a91-71ec-4057-8ecc-1122932a677a:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.14.15.png)

\<aside>

\*\*\*\*Lambda 함수를 주기적으로 실행하기 위해 AWS 콘솔에서 **EventBridge 서비스**로 이동한다.

\</aside>

**STEP 2) EventBridge 생성**

![image.png](attachment:c9822578-a0f8-492e-9e34-25ececdaf47d:image.png)

\<aside>

**Create rule** 버튼을 클릭해서 새 EventBridge 규칙을 생성한다.

\</aside>

**\[ 규칙 세부 정보 정의 ]**

![스크린샷 2025-07-23 오전 5.39.23.png](attachment:2e2a0e48-0948-4dbc-81a0-3252337d6d68:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.39.23.png)

\<aside>

* **Name** : **`eventbridge-ebs-detect-snapshot`**
* **Description** : (옵션)
* **Event bus :** default
* **Rule type** : Rule with an event pattern \</aside>

**\[** **이벤트 패턴 작성 ]**

![스크린샷 2025-07-23 오전 5.42.54.png](attachment:7ed19555-676c-4a8c-9e10-53b5588bc4f6:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.42.54.png)

\<aside>

탐지할 이벤트 조건을 설정을 설정하고 **Next**버튼을 클릭한다.

**Events :** Other

**Event pattern** : Custom pattern (JSON editor)

| 이벤트 이름                    | 설명                   |
| ------------------------- | -------------------- |
| `CreateSnapshot`          | EBS 볼륨 스냅샷 생성 API    |
| `DeleteSnapshot`          | 기존 스냅샷 삭제 API        |
| `ModifySnapshotAttribute` | EBS 스냅샷 공유 설정 변경 API |

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": [
      "CreateSnapshot",
      "DeleteSnapshot",
      "ModifySnapshotAttribute"
    ]
  }
}
```

\</aside>

**\[ 대상 선택 - SNS topic ]**

![스크린샷 2025-07-23 오전 5.48.50.png](attachment:a6c46a39-156e-4ec3-b8b1-0696b0ad8ff4:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.48.50.png)

\<aside>

* **Target types :** AWS service
* **Select a target :** SNS topic
* **Target location :** Target in this account
* **Topic :** 미리 만들어 둔 sns topic 선택

SNS 주제 선택 후 lambda 함수를 대상으로 추가하기 위해 **Add another target**을 클릭한다.

\</aside>

**\[ 대상 선택 - Lambda function ]**

![스크린샷 2025-07-23 오전 5.51.35.png](attachment:39102862-6687-4091-94fd-26f27fc124b5:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.51.35.png)

\<aside>

* **Target types :** AWS service
* **Select a target :** Lambda function
* **Target location :** Target in this account
* **Fuction :** 미리 만들어 둔 lambda function 선택

이벤트가 감지되었을 때 실행할 대상들을 지정하고 **Next**버튼을 클릭한다.

\</aside>

![스크린샷 2025-07-22 오전 1.59.01.png](attachment:9869130e-41d1-4edc-b4f5-a0f2beb4ff0f:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.59.01.png)

\<aside>

태그 구성은 선택 사항이므로 **Next**버튼을 클릭한다.

\</aside>

**\[** **검토** **및 생성 ]**

![스크린샷 2025-07-23 오전 5.54.14.png](attachment:8078676d-0512-47e1-afd6-b34b0744ac2c:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.54.14.png)

\<aside>

설정 내용 최종 확인 후 **Create rule**버튼을 클릭한다.

* status - **enabled** 확인 \</aside>

**STEP 3)** **생성된 규칙 확인**

![스크린샷 2025-07-23 오전 5.57.36.png](attachment:8bdf2863-022d-4b0d-95ed-785a98d981c5:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_5.57.36.png)

\<aside>

규칙이 정상적으로 생성되었는지 확인해준다.

\</aside>

#### 5. 테스트

> 테스트를 위한 EC2를 생성하여 인스턴스를 생성, 공유, 삭제 할 수 있다.

**\[ 탐지 이벤트 안내 ]**

| 이벤트 이름                    | 설명                   | 탐지목적                                                     |
| ------------------------- | -------------------- | -------------------------------------------------------- |
| `CreateSnapshot`          | EBS 볼륨 스냅샷 생성 API    | **데이터 무단 복제 시도 탐지** – 민감 정보 포함 볼륨을 외부로 유출하기 위한 스냅샷 생성 식별 |
| `DeleteSnapshot`          | 기존 스냅샷 삭제 API        | **증거 삭제 시도 탐지** – 기존 백업 또는 유출 흔적을 지우려는 행위 식별             |
| `ModifySnapshotAttribute` | EBS 스냅샷 공유 설정 변경 API | **스냅샷 외부 공유 시도 탐지** – 타 계정 혹은 공개 설정을 통한 데이터 유출 가능성 식별    |

**\[ EC2 인스턴스 생성 ]**

![스크린샷 2025-07-22 오전 2.11.28.png](attachment:6b79b214-36f3-4638-8aef-825f660affe6:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.11.28.png)

\<aside>

이벤트 테스트를 위해 EC2로 이동한다.

\</aside>

![스크린샷 2025-07-22 오전 2.21.00.png](attachment:48a3d310-e95d-4dfa-b7f2-06b629cadd25:f383a41c-4bdc-45b1-ab8c-3b71404b3f39.png)

![스크린샷 2025-07-23 오전 6.05.00.png](attachment:7cbb6822-bca9-4d2c-b786-e290c11c57b2:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_6.05.00.png)

\<aside>

임의의 EC2 인스턴스를 생성해 테스트를 진행한다. 아래 사항 외에는 기본값 그대로 진행한다.

**Name** : `ec2-ebs-test`

\</aside>

![스크린샷 2025-07-22 오전 2.17.58.png](attachment:fc18d063-30a6-4c77-b2b2-8d14ad2af177:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.17.58.png)

\<aside>

키페어 없이 진행한다.

\</aside>

**\[ 인스턴스 생성 확인 ]**

![스크린샷 2025-07-22 오전 2.19.38.png](attachment:a67f6ee6-709a-4417-8245-7b1f5d7ddfa3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.19.38.png)

**\[** `CreateSnapshot`**이벤트 발생 ]**

![스크린샷 2025-07-22 오전 2.22.19.png](attachment:c92dda6b-763b-4a7d-847c-7fa301650aad:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.22.19.png)

![스크린샷 2025-07-22 오전 2.24.07.png](attachment:761550bd-f41e-47c8-819a-cb44ef5b81d7:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.24.07.png)

\<aside>

EC2 → Elastic Block Store → Snapshots → Create snapshot

스냅샷을 생성하기 위해 **Create snapshot**을 클릭한다.

\</aside>

**\[ 세부 설정 ]**

![스크린샷 2025-07-22 오전 2.26.12.png](attachment:de3f7836-712e-4431-8dfb-504184a21dbb:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.26.12.png)

\<aside>

* **Source** : Resource type - Volume
* **Volume ID** : EC2 인스턴스 생성할 때 만들어진 볼륨 선택 \</aside>

**\[** `ModifySnapshotAttribute`**이벤트 발생 ]**

**STEP 1) 스냅샷 선택**

![스크린샷 2025-07-22 오전 2.37.36.png](attachment:ed8366b6-bac7-432f-868f-cd5f240a44d6:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.37.36.png)

\<aside>

생성한 **스냅샷**을 클릭한다.

\</aside>

**STEP 2) 권한 공유**

![스크린샷 2025-07-22 오전 2.49.35.png](attachment:1eece7d4-b344-4527-a7f1-a65f386c31eb:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.49.35.png)

\<aside>

**Modify permission** 버튼을 클릭한다.

**Add account ID**도 선택 가능하다.

\</aside>

**STEP 3) 권한 수정 - 계정 추가**

![스크린샷 2025-07-22 오전 2.53.10.png](attachment:91039694-426c-449d-bc1c-923461062be9:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.53.10.png)

\<aside>

권한 수정에서 **Add account** 버튼을 클릭한다.

\</aside>

![스크린샷 2025-07-22 오전 2.54.37.png](attachment:f4cbb27e-c333-4042-a58e-7d8ccbe0698d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.54.37.png)

\<aside>

**AWS 계정 ID**는 **12자리** 아무 숫자를 넣어준다.

\</aside>

![스크린샷 2025-07-22 오전 2.55.21.png](attachment:353feb42-44e4-4c71-8aec-8b3dc48ac4d3:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_2.55.21.png)

\<aside>

**Modify permission**을 누르면 공유 계정 추가 된다. 외부 계정 공유 시, Discord와 이메일로 알림이 간다.

\</aside>

**\[** `DeleteSnapshot`**이벤트 발생 ]**

![스크린샷 2025-07-22 오전 3.01.03.png](attachment:4fea690f-429b-45d1-81c2-8faa7de6d541:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.01.03.png)

![스크린샷 2025-07-22 오전 3.03.24.png](attachment:4683b39f-8a13-415a-b8be-54a2b4c3db3d:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-22_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.03.24.png)

\<aside>

삭제할 스냅샷을 선택하고, **Action**을 누른 다음 **Delete snapshot**을 클릭한다. 삭제 시, Discord와 이메일로 알림이 간다.

\</aside>

***

**\[ Email 알림 확인 ]**

![스크린샷 2025-07-23 오전 6.37.11.png](attachment:21c0155f-0d0b-43b5-88a9-fcf17c4c742e:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_6.37.11.png)

**\[ Discord 알림 확인 ]**

![스크린샷 2025-07-23 오전 6.50.06.png](attachment:d614e5c5-11a9-423a-b379-af9c93deb9b0:%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-07-23_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_6.50.06.png)
