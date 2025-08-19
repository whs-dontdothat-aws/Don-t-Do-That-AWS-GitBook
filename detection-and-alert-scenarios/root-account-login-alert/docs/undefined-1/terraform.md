# Terraform

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.



**참고**

실습 전 [IaC 실습 전 환경 구성](https://app.gitbook.com/o/Ovrk2sznEJVoEThBhvLL/s/OxBH8cyDzBTzXyUOUzPl/) 을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.

{% embed url="https://app.gitbook.com/o/Ovrk2sznEJVoEThBhvLL/s/OxBH8cyDzBTzXyUOUzPl/" %}



**\[ Terraform 파일 구조 ]**

```hcl
Terraform-code/
├── main.tf                # 모든 주요 Terraform 리소스 정의 
├── variables.tf           # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars       # 민감 데이터 정의 (Email, Webhook URL 등)
├── lambda.zip             # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda_function.py # Lambda 함수 원본 코드 

```



**\[ Terraform 구현 코드 ]**

<details>

<summary>main.tf</summary>

```hcl
#---------------------------------------------------------------------------
# 1. PROVIDER ─ AWS 리전 설정 (서울)
#--------------------------------------------------------------------------
provider "aws" {
  region = "ap-northeast-2" # 서울 리전 사용
}

#---------------------------------------------------------------------------
# 2. CloudTrail 설정(이미 리전에 추적 존재한다면 이 부분 생략)
#---------------------------------------------------------------------------

# 현재 AWS 계정 정보 조회 (account_id 등 활용 가능)
data "aws_caller_identity" "current" {}

# CloudTrail 로그를 저장할 S3 버킷 생성
resource "aws_s3_bucket" "trail_bucket" {
  bucket = "log-group-trail-bucket"
  force_destroy = true # 버킷 비워진 후 삭제 허용
}

# 생성한 S3 버킷에 대한 퍼블릭 액세스 차단
resource "aws_s3_bucket_public_access_block" "trail_bucket_block" {
  bucket                  = aws_s3_bucket.trail_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# CloudTrail 서비스가 로그 기록을 위한 S3 버킷에 접근 허용
resource "aws_s3_bucket_policy" "trail_bucket_policy" {
  bucket = aws_s3_bucket.trail_bucket.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck",
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        },
        Action   = "s3:GetBucketAcl",
        Resource = "arn:aws:s3:::${aws_s3_bucket.trail_bucket.id}"
      },
      {
        Sid    = "AWSCloudTrailWrite",
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        },
        Action   = "s3:PutObject",
        Resource = "arn:aws:s3:::${aws_s3_bucket.trail_bucket.id}/AWSLogs/${data.aws_caller_identity.current.account_id}/*",
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
          }
        }
      }
    ]
  })
}

# CloudTrail 트레일 생성 (모든 리전에서 이벤트 수집)
resource "aws_cloudtrail" "log_group_trail" {
  name                          = "log-group-monitor"
  s3_bucket_name                = aws_s3_bucket.trail_bucket.id
  include_global_service_events = true # 글로벌 서비스 이벤트 포함
  is_multi_region_trail         = true # 다중 리전 이벤트 포함
  enable_logging                = true # 로그 수집 활성화

  depends_on = [
    aws_s3_bucket_policy.trail_bucket_policy
  ]
}

#---------------------------------------------------------------------------
# 3. Lambda 설정
#---------------------------------------------------------------------------

# Lambda 실행을 위한 IAM 역할 생성
resource "aws_iam_role" "lambda_exec_role" {
  name = "log_group_alert_lambda_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Action = "sts:AssumeRole",
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}


# Lambda Role 에 AWSLambdaBasicExecutionRole 정책 연결 (CloudWatch 로그 기록 가능)
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}


# Discord Webhook으로 알림을 보내는 Lambda 함수 생성
resource "aws_lambda_function" "discord_alert" {
  filename         = "lambda.zip" # 패키징된 코드 zip 파일
  function_name    = "log_group_alert"
  role             = aws_iam_role.lambda_exec_role.arn
  handler          = "lambda_function.lambda_handler" # Python 핸들러 경로
  runtime          = "python3.12"
  source_code_hash = filebase64sha256("lambda.zip") # 코드 변경 감지용

  environment {
    variables = {
      HOOK_URL = var.discord_webhook_url # Discord Webhook URL 환경변수
    }
  }
}

#---------------------------------------------------------------------------
# 4. SNS Topic 및 Email, Lambda 구독 설정
#---------------------------------------------------------------------------

# SNS 생성
resource "aws_sns_topic" "log_group_topic" {
  name = "log_group_event"
}

# 이메일 구독 생성
resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.log_group_topic.arn
  protocol  = "email"
  endpoint  = var.notification_email
}

# Lambda 구독 생성
resource "aws_sns_topic_subscription" "lambda_sub" {
  topic_arn = aws_sns_topic.log_group_topic.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.discord_alert.arn
}

# SNS가 Lambda 호출할 수 있도록 권한 부여
resource "aws_lambda_permission" "allow_sns_invoke" {
  statement_id  = "AllowSNSInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.discord_alert.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.log_group_topic.arn
}

#---------------------------------------------------------------------------
# 5. EventBridge 설정
#---------------------------------------------------------------------------

# EventBridge 규칙 생성 : IAM User 생성, 삭제 탐지
resource "aws_sns_topic_policy" "allow_eventbridge_publish" {
  arn = aws_sns_topic.log_group_topic.arn

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Sid       = "AllowEventBridgePublish",
      Effect    = "Allow",
      Principal = { Service = "events.amazonaws.com" },
      Action    = "sns:Publish",
      Resource  = aws_sns_topic.log_group_topic.arn
    }]
  })
}

# EventBridge 규칙의 타겟으로 SNS 주 연결
resource "aws_cloudwatch_event_rule" "log_group_change_rule" {
  name        = "detect-log-group-change"
  description = "Detect CloudWatch LogGroup deletion or configuration changes"

  event_pattern = jsonencode({
    "source" : ["aws.logs"], # 로그 서비스에서 발생한 이벤트
    "detail-type" : ["AWS API Call via CloudTrail"], # API 호출 이벤트
    "detail" : {
      "eventSource" : ["logs.amazonaws.com"], # CloudWatch Logs API 호출
      "eventName" : [ # 감지할 API 이벤트 목록
        "DeleteLogGroup",           # 로그 그룹 삭제
        "PutRetentionPolicy",       # 보존 정책 변경
        "DeleteSubscriptionFilter", # Subscription 필터 삭제
        "PutSubscriptionFilter",    # Subscription 필터 추가/변경
        "DeleteResourcePolicy",     # 리소스 정책 삭제
        "PutResourcePolicy"         # 리소스 정책 추가/변경
      ]
    }
  })
}


# EventBridge 규칙의 타겟으로 SNS 주 연결
resource "aws_cloudwatch_event_target" "send_to_sns" {
  rule      = aws_cloudwatch_event_rule.log_group_change_rule.name
  target_id = "snsTarget"
  arn       = aws_sns_topic.log_group_topic.arn
}

```



</details>

<details>

<summary>variables.tf</summary>

```hcl
# 해당 파일에서는 변수를 여기다가 모두 선언
# terraform 실행 시 terraform.tfvars에 선언된 값을 바탕으로 값이 들어감

# Discord Webhook 주소를 입력받기 위한 변수
variable "discord_webhook_url" {
  description = "Discord webhook URL"
  type        = string
}

# 이메일 수신자를 입력받기 위한 변수
variable "notification_email" {
  description = "Email address to receive alerts"
  type        = string
}
```



</details>

<details>

<summary>terraform.tfvars</summary>

```hcl
# 알림을 받을 디스코드 Webhook URL로 설정
# 해당 URL을 통해 Lambda 함수가 알림을 디스코드 채널로 전송
discord_webhook_url = "discord web hook url 작성"

# 알림을 받을 이메일 주소로 설정
# 보안 이벤트가 발생 시 SNS를 통해 해당 이메일로 알람 전송
notification_email  = "이메일 주소"
```

</details>

<details>

<summary><strong>lambda.zip (lambda_function.py)</strong></summary>

**의존성 없이 zip 만들기**

```bash
zip lambda.zip lambda_function.py
```

```python
import os, json, urllib3

http = urllib3.PoolManager()
WEBHOOK = os.environ["HOOK_URL"]

def lambda_handler(event, context):
    for rec in event["Records"]:
        msg = json.loads(rec["Sns"]["Message"])
        detail = msg.get("detail", {})

        content = (
            "CloudWatch Logs 변경 탐지\n"
            f"Event: {detail.get('eventName')}\n"
            f"LogGroup: {detail.get('requestParameters', {}).get('logGroupName', 'N/A')}\n"
            f"User: {detail.get('userIdentity', {}).get('arn', 'Unknown')}\n"
            f"Time: {msg.get('time')}"
        )
        http.request(
            "POST", WEBHOOK,
            body=json.dumps({"content": content}).encode(),
            headers={"Content-Type": "application/json"}
        )
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

<figure><img src="../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다. 이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.



<figure><img src="../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

**Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.



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

{% file src="../.gitbook/assets/Terraform-code.zip" %}
