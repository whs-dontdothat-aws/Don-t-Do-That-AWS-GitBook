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

</details>

<details>

<summary>variables.tf</summary>

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

</details>

<details>

<summary>terraform.tfvars</summary>

```hcl
# 알림을 받을 디스코드 Webhook URL로 설정
# 해당 URL을 통해 Lambda 함수가 알림을 디스코드 채널로 전송
discord_webhook_url = "discord web hook url"

# 알림을 받을 이메일 주소로 설정
# 보안 이벤트가 발생 시 SNS를 통해 해당 이메일로 알람 전송
notification_email  = "이메일 주소"
```

</details>

<details>

<summary>lambda.zip (lambda_function.py)</summary>

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

</details>

<details>

<summary>코드 실행</summary>

\[ Terraform 실행 코드 ]

```bash
terraform init # 초기화
terraform plan # 설정 검증
terraform apply # 적용 (실행)
-------------------------------------------------------
terraform destroy # 실습 완료 후, 리소스 정리
```



\[ init ]

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



\[ plan ]

```bash
terraform plan
```

Terraform 코드 적용 시, 인프라에 어떤 변경이 발생할지 미리 확인할 수 있는 실행 계획을 보여주는 명령어이다.



```bash
Plan: 24 to add, 0 to change, 0 to destroy.
```

총 24개의 리소스가 새로 생성될 예정이며, 실행 계획이 정상적으로 생성된 상태이다. 이 출력 결과는 적용해도 문제가 없는 준비 완료 상태임을 의미한다.



\[ apply ]

```bash
terraform apply
```

위와 같은 메시지가 출력되면, 모든 리소스가 정상적으로 생성되었거나 업데이트되어 인프라 상태가 원하는 대로 적용된 것이다.



\[ 이메일 인증 ]

<figure><img src="../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다.\
이메일을 열어 Confirm subscription 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.



<figure><img src="../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

Confirm subscription를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.



\[ 테스트 진행 ]

5. [테스트](./#id-5)

인증 후 위를 참고하여 테스트를 진행하면 된다.



\[ destroy ]

```bash
terraform destroy
```

Terraform으로 생성된 모든 인프라 리소스를 자동으로 삭제하는 명령어이다.\
실습 완료 후에는 해당 명령어로 불필요한 리소스를 정리할 수 있다.



```bash
Destroy complete! Resources: 0 destroyed.
```

위와 같은 메시지가 출력되면, 모든 리소스가 성공적으로 정리되었음을 의미한다.

</details>



**\[ 전체 코드 압축 파일 ]**

{% file src="../.gitbook/assets/Terraform-code.zip" %}

