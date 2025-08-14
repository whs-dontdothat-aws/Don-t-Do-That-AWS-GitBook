# Terraform

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.



**참고**

실습 전 [IaC 실습 전 환경 구성](https://www.notion.so/IaC-236b5a2aa9af80528203d0ee4c07992d?pvs=21) 을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.



**\[ Terraform 파일 구조 ]**

```hcl
Terraform-code/
├── main.tf                # 모든 주요 Terraform 리소스 정의 (CloudTrail, SNS, EventBridge, Lambda, IAM 포함)
├── variables.tf           # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars       # 민감 데이터 정의 (이메일 주소, Webhook URL 등)
├── lambda.zip             # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda_function.py # Lambda 함수 원본 코드

```



**\[ Terraform 구현 코드 ]**

*   리소스명

    | **리소스 종류**       | **Terraform 리소스명**                                   | **목적/설명**                     |
    | ---------------- | ---------------------------------------------------- | ----------------------------- |
    | S3 Bucket        | s3-cloudtrail-monitor                                | CloudTrail 로그 저장 버킷           |
    | S3 Bucket Policy | aws\_s3\_bucket\_policy.cloudtrail\_policy           | CloudTrail이 S3에 로그 저장 가능하게 허용 |
    | CloudTrail Trail | ct-trail-monitor                                     | AWS API 활동 감시 및 로그 저장         |
    | SNS Topic        | sns-cloudtrail-alarm                                 | 이메일 및 Lambda로 알림 전송하는 SNS 주제  |
    | SNS 이메일 구독       | aws\_sns\_topic\_subscription.email\_sub             | 알림 이메일 구독                     |
    | EventBridge Rule | eventbridge-ct-detect (aws\_cloudwatch\_event\_rule) | CloudTrail 이벤트 감지용 규칙         |
    | Lambda 함수        | lambda-ct-detect-alarm                               | SNS 이벤트를 디스코드 알림으로 전송         |
    | Lambda 환경 변수     | DISCORD\_WEBHOOK\_URL                                | 디스코드 웹훅 URL 저장                |
    | Discord 채널       | cloudtrail\_detect\_alarm                            | 디스코드 알림 수신 채널                 |



<details>

<summary>main.tf</summary>

```hcl
provider "aws" {
  region = "ap-northeast-2"  # 서울 리전
}

data "aws_caller_identity" "current" {}  # 현재 AWS 계정 정보 조회

#----------------------------------------------------------------------------
# 1. S3 Bucket (CloudTrail 로그 저장용)
#----------------------------------------------------------------------------
resource "aws_s3_bucket" "cloudtrail_logs" {
  bucket = "s3-cloudtrail-monitor"

  versioning {
    enabled = true  # 버전 관리 활성화
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"  # 서버 측 암호화 (기본)
      }
    }
  }

  tags = {
    Name = "CloudTrailLogs"
  }
}

resource "aws_s3_bucket_policy" "cloudtrail_policy" {
  bucket = aws_s3_bucket.cloudtrail_logs.id

  # CloudTrail이 S3에 로그를 쓸 수 있도록 허용
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck20150319",
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        },
        Action   = "s3:GetBucketAcl",
        Resource = aws_s3_bucket.cloudtrail_logs.arn
      },
      {
        Sid    = "AWSCloudTrailWrite20150319",
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        },
        Action   = "s3:PutObject",
        Resource = "${aws_s3_bucket.cloudtrail_logs.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*",
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" : "bucket-owner-full-control"
          }
        }
      }
    ]
  })
}

#----------------------------------------------------------------------------
# 2. CloudTrail Trail 생성
#----------------------------------------------------------------------------
resource "aws_cloudtrail" "ct_trail" {
  name                          = "ct-trail-monitor"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  include_global_service_events = true  # 글로벌 이벤트 포함
  is_multi_region_trail         = true  # 멀티 리전 지원
  enable_log_file_validation    = true  # 로그 검증 활성화

  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }

  tags = {
    Name = "ct-trail-monitor"
  }

  depends_on = [aws_s3_bucket_policy.cloudtrail_policy]  # 정책 적용 후 Trail 생성
}

#----------------------------------------------------------------------------
# 3. SNS Topic 및 Email 구독 설정
#----------------------------------------------------------------------------
resource "aws_sns_topic" "cloudtrail_alarm" {
  name = "sns-cloudtrail-alarm"
}

resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.cloudtrail_alarm.arn
  protocol  = "email"
  endpoint  = var.email_address  # 변수로 받은 이메일 주소
}

#----------------------------------------------------------------------------
# 4. EventBridge Rule → SNS 트리거
#----------------------------------------------------------------------------
resource "aws_cloudwatch_event_rule" "ct_eventbridge_rule" {
  name        = "eventbridge-ct-detect"
  description = "Detect CloudTrail Stop/Delete/Update events"

  # StopLogging, DeleteTrail 등의 이벤트를 감지
  event_pattern = jsonencode({
    source       = ["aws.cloudtrail"],
    "detail-type" = ["AWS API Call via CloudTrail"],
    detail       = {
      eventSource = ["clou]()

```



</details>

<details>

<summary>variables.tf</summary>

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



</details>

<details>

<summary>terraform.tfvars</summary>

```hcl
# 알림을 받을 이메일 주소로 설정
# 보안 이벤트가 발생 시 SNS를 통해 해당 이메일로 알람 전송
email_address      = "your-email@example.com"

# 알림을 받을 디스코드 Webhook URL로 설정
# 해당 URL을 통해 Lambda 함수가 알림을 디스코드 채널로 전송
discord_webhook_url = "<https://discord.com/api/webhooks/xxxx/yyyy>"
```

</details>

<details>

<summary><strong>lambda.zip (lambda_function.py)</strong></summary>

**의존성 없이 zip 만들기**

```bash
zip lambda.zip lambda_function.py
```

```python
import json
import urllib.request
import os
from datetime import datetime, timezone, timedelta

def lambda_handler(event, context):
    # 환경변수에서 Discord 웹훅 URL 가져오기
    webhook_url = os.environ.get('DISCORD_WEBHOOK_URL')
    if not webhook_url:
        print("환경변수 DISCORD_WEBHOOK_URL이 설정되어 있지 않습니다.")
        return {'statusCode': 500, 'body': 'Webhook URL not set'}

    try:
        # SNS 메시지 추출 및 파싱
        sns_message_str = event['Records'][0]['Sns']['Message']
        sns_message = json.loads(sns_message_str)

        # 이벤트 상세 정보 추출
        detail = sns_message.get("detail", {})
        event_name = detail.get("eventName", "Unknown Event")
        event_time_utc = detail.get("eventTime")
        user_identity = detail.get("userIdentity", {})
        user_arn = user_identity.get("arn", "Unknown ARN")
        source_ip = detail.get("sourceIPAddress", "Unknown IP")
        aws_region = detail.get("awsRegion", "Unknown Region")
        account_id = detail.get("recipientAccountId", "Unknown Account")

        # UTC 시간을 한국 시간(KST)으로 변환
        if event_time_utc:
            try:
                utc_dt = datetime.strptime(event_time_utc, "%Y-%m-%dT%H:%M:%SZ").replace(tzinfo=timezone.utc)
                kst_dt = utc_dt.astimezone(timezone(timedelta(hours=9)))
                event_time_kst = kst_dt.strftime("%Y-%m-%d %H:%M:%S (KST)")
            except Exception as e:
                print("시간 변환 실패:", e)
                event_time_kst = event_time_utc + " (UTC)"
        else:
            event_time_kst = "Unknown Time"

    except Exception as e:
        # 메시지 파싱 실패 시 기본 값 설정
        print("SNS 메시지 파싱 실패:", e)
        event_name = "Unknown Event"
        event_time_kst = "Unknown Time"
        user_arn = "Unknown ARN"
        source_ip = "Unknown IP"
        aws_region = "Unknown Region"
        account_id = "Unknown Account"

    # Discord로 전송할 메시지 구성
    content = (
        f"**[ CloudTrail 이벤트 탐지 ]**\\n"
        f"• 이벤트 이름: `{event_name}`\\n"
        f"• 발생 시간: `{event_time_kst}`\\n"
        f"• 사용자 ARN: `{user_arn}`\\n"
        f"• 소스 IP: `{source_ip}`\\n"
        f"• 리전: `{aws_region}`\\n"
        f"• 계정 ID: `{account_id}`"
    )

    payload = json.dumps({"content": content}).encode("utf-8")

    # Discord 웹훅 요청 생성 및 전송
    req = urllib.request.Request(
        webhook_url,
        data=payload,
        headers={
            "Content-Type": "application/json",
            "User-Agent": "Mozilla/5.0 (Lambda)"
        }
    )

    try:
        with urllib.request.urlopen(req) as response:
            resp_body = response.read()
            print("Discord 응답:", resp_body)
    except Exception as e:
        # 전송 실패 시 오류 출력
        print("Discord 전송 실패:", e)
        return {'statusCode': 500, 'body': 'Discord notification failed'}

    return {'statusCode': 200, 'body': 'Notification sent successfully'}

```



</details>

<details>

<summary>코드 실행</summary>

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

Terraform 프로젝트를 처음 시작할 때 실행하는 명령어로, 작업 디렉토리를 초기화하고 필요한 설정 파일과 실행에 필요한 구성 요소들을 준비해준다. 이후 plan, apply 등의 명령을 정상적으로 사용할 수 있는 상태로 만든다.



```bash
Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

위와 같은 메시지가 출력되면, 프로젝트가 초기화되어 Terraform 명령어를 사용할 수 있는 준비가 완료된 것이다.



**\[ plan ]**

```bash
terraform plan
```

Terraform 코드 적용 시, 인프라에 어떤 변경이 발생할지 미리 확인할 수 있는 실행 계획을 보여주는 명령어이다.



```bash
Plan: 24 to add, 0 to change, 0 to destroy.
```

총 24개의 리소스가 새로 생성될 예정이며, 실행 계획이 정상적으로 생성된 상태이다. 이 출력 결과는 적용해도 문제가 없는 준비 완료 상태임을 의미한다.



**\[ apply ]**

```bash
terraform apply
```

terraform apply 명령어는 실행 계획(plan)에 따라 실제로 클라우드 인프라를 생성, 변경, 삭제하는 작업을 수행한다. Plan 단계에서 검토한 내용을 기반으로 실제 인프라에 반영하고자 할 때 사용한다.



```bash
Apply complete! Resources: 24 added, 0 changed, 0 destroyed.
```

위와 같은 메시지가 출력되면, 모든 리소스가 정상적으로 생성되었거나 업데이트되어 인프라 상태가 원하는 대로 적용된 것이다.



**\[ 이메일 인증 ]**

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다. 이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.



<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

**Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.



[5. 테스트](https://www.notion.so/5-1feb5a2aa9af80e49d4fc2a03a45f6da?pvs=21)

인증 후 위를 참고하여 테스트를 진행하면 된다.



**\[ destroy ]**

```bash
terraform destroy
```

Terraform으로 생성된 모든 인프라 리소스를 자동으로 삭제하는 명령어이다. **실습 완료 후**에는 해당 명령어로 불필요한 리소스를 정리할 수 있다.



```bash
Destroy complete! Resources: 0 destroyed.
```

위와 같은 메시지가 출력되면, 모든 리소스가 성공적으로 정리되었음을 의미한다.

</details>



**\[ 전체 코드 압축 파일 ]**

git..
