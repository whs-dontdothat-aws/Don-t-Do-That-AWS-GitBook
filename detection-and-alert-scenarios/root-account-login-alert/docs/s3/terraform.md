# Terraform

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.

**참고**

실습 전 [IaC 실습 전 환경 구성](https://app.gitbook.com/s/OxBH8cyDzBTzXyUOUzPl/ "mention")을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.

{% embed url="https://app.gitbook.com/o/Ovrk2sznEJVoEThBhvLL/s/OxBH8cyDzBTzXyUOUzPl/" %}



**\[ terraform 파일 구조 ]**

```hcl
Terraform-code/
 ├── main.tf                      # 모든 주요 Terraform 리소스 정의
 ├── variables.tf                 # 입력 변수 정의 (Email, Discord Webhook 등)
 └── lambda_function.zip          # 디스코드 알림을 전송하는 패키지된 Lambda 코드
      └── lambda_function.py      # Lambda 함수 원본 코드
```



**\[ Terraform 구현 코드 ]**

* 리소스명

| 리소스 종류               | 리소스명 설정                         | 목적                                |
| -------------------- | ------------------------------- | --------------------------------- |
| **S3 Bucket**        | **`s3-config-logbucket`**       | AWS Config 로그 저장용 버킷              |
| **AWS Config Role**  | **`iam-config-role`**           | Config 서비스 실행 권한 역할               |
| **SNS Topic**        | **`sns-config-alert`**          | 퍼블릭 탐지 시 이메일 알림 SNS 토픽            |
| **Lambda 함수**        | **`lambda-config-function`**    | Config 이벤트 감지 후 Discord 알림 Lambda |
| **Lambda Role**      | **`iam-lambda-config`**         | Lambda 실행용 역할                     |
| **EventBridge Rule** | **`eventbridge-config-public`** | 퍼블릭 상태 감지 시 Lambda/SNS 실행 규칙      |
| **Lambda Policy**    | **`lambda-public-block`**       | 퍼블릭 상태 감지 시 S3 버킷을 자동으로 비공개 처리    |



<details>

<summary>main.tf</summary>

```hcl
#----------------------------------------------------------------------------
#  PROVIDER ─ AWS 리전 설정
#----------------------------------------------------------------------------
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

#----------------------------------------------------------------------------
#  현재 AWS 계정 정보
#----------------------------------------------------------------------------
data "aws_caller_identity" "current" {}


#----------------------------------------------------------------------------
#  S3 버킷 ─ AWS Config 로그 저장
#----------------------------------------------------------------------------
resource "aws_s3_bucket" "config_bucket" {
  bucket = "s3-public-bucket-config"
}

resource "aws_s3_bucket_public_access_block" "config_bucket_block" {
  bucket = aws_s3_bucket.config_bucket.id

  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_policy" "config_bucket_policy" {
  bucket = aws_s3_bucket.config_bucket.id

  policy = jsonencode({
    Version   = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSConfigBucketPermissionsCheck"
        Effect = "Allow"
        Principal = { Service = "config.amazonaws.com" }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.config_bucket.arn
        Condition = {
          StringEquals = {
            "AWS:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      },
      {
        Sid    = "AWSConfigBucketExistenceCheck"
        Effect = "Allow"
        Principal = { Service = "config.amazonaws.com" }
        Action   = "s3:ListBucket"
        Resource = aws_s3_bucket.config_bucket.arn
        Condition = {
          StringEquals = {
            "AWS:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      },
      {
        Sid    = "AWSConfigBucketDelivery"
        Effect = "Allow"
        Principal = { Service = "config.amazonaws.com" }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.config_bucket.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl"      = "bucket-owner-full-control"
            "AWS:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      },
      {
        Sid    = "AWSConfigBucketGetObject"
        Effect = "Allow"
        Principal = { Service = "config.amazonaws.com" }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.config_bucket.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })

  depends_on = [aws_s3_bucket_public_access_block.config_bucket_block]
}

#----------------------------------------------------------------------------
#  IAM 역할 ─ AWS Config Recorder
#----------------------------------------------------------------------------
resource "aws_iam_role" "config_service_role" {
  name = "AWSConfigServiceRole"

  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = { Service = "config.amazonaws.com" },
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "config_policy_attach" {
  role       = aws_iam_role.config_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

#----------------------------------------------------------------------------
#  AWS Config ─ Recorder / Delivery Channel
#----------------------------------------------------------------------------
resource "aws_config_configuration_recorder" "recorder" {
  name     = "default"
  role_arn = aws_iam_role.config_service_role.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }

  depends_on = [aws_s3_bucket_policy.config_bucket_policy]
}

resource "aws_config_delivery_channel" "main" {
  name           = "default"
  s3_bucket_name = aws_s3_bucket.config_bucket.id

  depends_on = [aws_config_configuration_recorder.recorder]
}

resource "aws_config_configuration_recorder_status" "recorder_status" {
  name       = aws_config_configuration_recorder.recorder.name
  is_enabled = true

  depends_on = [aws_config_delivery_channel.main]
}

#----------------------------------------------------------------------------
#  AWS Config 규칙 ─ 퍼블릭 읽기/쓰기 금지
#----------------------------------------------------------------------------
resource "aws_config_config_rule" "s3_public_read_prohibited" {
  name        = "s3-bucket-public-read-prohibited"
  description = "S3 버킷 퍼블릭 읽기 권한 금지"

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder.recorder]
}

resource "aws_config_config_rule" "s3_public_write_prohibited" {
  name        = "s3-bucket-public-write-prohibited"
  description = "S3 버킷 퍼블릭 쓰기 권한 금지"

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_WRITE_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder.recorder]
}

#----------------------------------------------------------------------------
#  IAM 역할 ─ Lambda 실행
#----------------------------------------------------------------------------
resource "aws_iam_role" "lambda_exec_role" {
  name = "ConfigRuleNotifierLambdaRole"

  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = { Service = "lambda.amazonaws.com" },
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_logs_policy" {
  role       = aws_iam_role.lambda_exec_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

#----------------------------------------------------------------------------
#  Lambda 함수 ─ Discord 웹훅
#----------------------------------------------------------------------------
resource "aws_lambda_function" "discord_notifier" {
  function_name    = "Discord"
  filename         = "function.zip"
  source_code_hash = filebase64sha256("function.zip")
  runtime          = "python3.13"
  handler          = "lambda_function.lambda_handler"
  role             = aws_iam_role.lambda_exec_role.arn

  environment {
    variables = {
      DISCORD_WEBHOOK_URL = var.discord_webhook_url
    }
  }
}

#----------------------------------------------------------------------------
#  EventBridge 규칙 ─ NON_COMPLIANT
#----------------------------------------------------------------------------
resource "aws_cloudwatch_event_rule" "config_noncompliant_rule" {
  name        = "public-event-rule"
  description = "AWS Config 규칙 NON_COMPLIANT 시 트리거"

  event_pattern = jsonencode({
    source        = ["aws.config"],
    "detail-type" = ["Config Rules Compliance Change"],
    detail        = {
      configRuleName = [
        aws_config_config_rule.s3_public_read_prohibited.name,
        aws_config_config_rule.s3_public_write_prohibited.name
      ],
      messageType         = ["ComplianceChangeNotification"],
      newEvaluationResult = { complianceType = ["NON_COMPLIANT"] }
    }
  })
}

#----------------------------------------------------------------------------
#  EventBridge 대상 ─ Lambda (Discord)
#----------------------------------------------------------------------------
resource "aws_cloudwatch_event_target" "config_rule_lambda_target" {
  rule      = aws_cloudwatch_event_rule.config_noncompliant_rule.name
  target_id = "DiscordNotifierLambda"
  arn       = aws_lambda_function.discord_notifier.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  function_name = aws_lambda_function.discord_notifier.function_name
  action        = "lambda:InvokeFunction"
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.config_noncompliant_rule.arn
}

#----------------------------------------------------------------------------
#  SNS ─ 이메일 알림 + EventBridge 권한
#----------------------------------------------------------------------------
resource "aws_sns_topic" "config_alert_email" {
  name         = "Email"
  display_name = "AWS Config Alerts Email"
}

# EventBridge가 토픽으로 Publish 할 수 있도록 정책 부여
resource "aws_sns_topic_policy" "allow_eventbridge_publish" {
  arn = aws_sns_topic.config_alert_email.arn

  policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [
      {
        Sid       = "AllowEventBridgePublish"
        Effect    = "Allow"
        Principal = { Service = "events.amazonaws.com" }
        Action    = "sns:Publish"
        Resource  = aws_sns_topic.config_alert_email.arn
      }
    ]
  })
}

resource "aws_sns_topic_subscription" "config_alert_email_sub" {
  topic_arn = aws_sns_topic.config_alert_email.arn
  protocol  = "email"
  endpoint  = var.notification_email
}

resource "aws_cloudwatch_event_target" "config_rule_sns_target" {
  rule      = aws_cloudwatch_event_rule.config_noncompliant_rule.name
  target_id = "EmailAlertTopic"
  arn       = aws_sns_topic.config_alert_email.arn
}
```

</details>

<details>

<summary>variables.tf</summary>

```hcl
variable "discord_webhook_url" {
description = "Discord Webhook URL (예: https://discord.com/api/webhooks/...)"
type        = string
}

variable "notification_email" {
description = "Config 알림 이메일 주소"
type        = string
}
```

</details>

<details>

<summary>lambda_function.zip (lambda_function.py)</summary>

```hcl
# lambda_function.py
import json
import urllib3
import os

http = urllib3.PoolManager()
DISCORD_WEBHOOK_URL = os.environ.get("DISCORD_WEBHOOK_URL", "")

def lambda_handler(event, context):
    try:
        detail = event.get("detail", {})
    except:
        return {"statusCode": 400, "body": "Invalid EventBridge message"}

    bucket_name = detail.get("resourceId", "Unknown")
    compliance = detail.get("newEvaluationResult", {}).get("complianceType", "UNKNOWN")
    annotation = detail.get("newEvaluationResult", {}).get("annotation", "No annotation")

    message = {
        "content": (
            f"S3 Public Access Detected\n"
            f"Bucket: `{bucket_name}`\n"
            f"Compliance Status: `{compliance}`\n"
            f"Reason: {annotation}"
        )
    }

    try:
        http.request(
            "POST",
            DISCORD_WEBHOOK_URL,
            body=json.dumps(message),
            headers={"Content-Type": "application/json"}
        )
    except:
        return {"statusCode": 500, "body": "Failed to send Discord message"}

    return {"statusCode": 200, "body": "Alert sent"}

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

<details>

<summary>오류 발생 시 확인</summary>

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

```bash
terraform import aws_iam_role.config_service_role AWSConfigServiceRole
```

AWS에 **이미 존재하는 Role**을 Terraform state에 가져오기 위해서 해당 명령어를 입력한다.

</details>



**\[ 전체 코드 압축 파일 ]**

{% file src="../.gitbook/assets/terraform code.zip" %}
