# Terraform

***

#### 6. Terraform 구현

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.

**참고**

실습 전 [IaC 실습 전 환경 구성](https://app.gitbook.com/s/OxBH8cyDzBTzXyUOUzPl/ "mention")을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.



**\[ Terraform 파일 구조 ]**

```
Terraform-code/
├── main.tf                # 모든 주요 Terraform 리소스 정의 
├── variables.tf           # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars       # 민감 데이터 정의 (Email, Webhook URL 등)
├── lambda.zip             # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda.py          # Lambda 함수 원본 코드
```



<details>

<summary>main.tf</summary>

```hcl
#----------------------------------------------------------------------------
# Terraform 제공자 설정: AWS 제공자와 버전 명시
#----------------------------------------------------------------------------
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"     # AWS 제공자를 HashiCorp로부터 사용
      version = "~> 5.0"            # 5.x 버전대 사용
    }
  }
  required_version = ">= 1.3.0"     # Terraform 최소 버전 지정
}

#----------------------------------------------------------------------------
# AWS 리전 설정: 서울 리전 (ap-northeast-2)
#----------------------------------------------------------------------------
provider "aws" {
  region = "ap-northeast-2"
}

#----------------------------------------------------------------------------
# S3 버킷 이름에 랜덤한 접미사 추가를 위한 ID 생성
#----------------------------------------------------------------------------
resource "random_id" "bucket_suffix" {
  byte_length = 4                   # 4바이트 길이의 랜덤 ID 생성
}

#----------------------------------------------------------------------------
# CloudTrail 로그를 저장할 S3 버킷 생성
#----------------------------------------------------------------------------
resource "aws_s3_bucket" "cloudtrail_bucket" {
  bucket        = "ct-securitygroup-bucket-${random_id.bucket_suffix.hex}"  # 랜덤 접미사 포함한 버킷 이름
  force_destroy = true                    # 버킷 안의 객체까지 삭제 허용
}

#----------------------------------------------------------------------------
# S3 버킷 정책: CloudTrail이 로그를 업로드할 수 있도록 허용
#----------------------------------------------------------------------------
resource "aws_s3_bucket_policy" "cloudtrail_policy" {
  bucket = aws_s3_bucket.cloudtrail_bucket.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"       # CloudTrail 서비스에 권한 부여
        },
        Action = "s3:PutObject",                     # 객체 업로드 허용
        Resource = "${aws_s3_bucket.cloudtrail_bucket.arn}/*",
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"  # 버킷 소유자에게 전체 제어 권한 부여
          }
        }
      },
      {
        Effect = "Allow",
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        },
        Action = [
          "s3:GetBucketAcl",                         # 버킷 ACL 확인
          "s3:ListBucket"                            # 버킷 목록 확인
        ],
        Resource = aws_s3_bucket.cloudtrail_bucket.arn
      }
    ]
  })
}

#----------------------------------------------------------------------------
# CloudTrail 생성: API 이벤트를 S3에 기록
#----------------------------------------------------------------------------
resource "aws_cloudtrail" "ct_securitygroup" {
  name                          = "ct-securitygroup"  # CloudTrail 이름
  s3_bucket_name                = aws_s3_bucket.cloudtrail_bucket.id # 로그 저장 S3 버킷
  include_global_service_events = true                # 글로벌 서비스 이벤트 포함
  is_multi_region_trail         = true                # 다중 리전 지원
  enable_log_file_validation    = true                # 로그 무결성 검증 사용
  event_selector {
    read_write_type           = "All"                # 읽기/쓰기 이벤트 모두 기록
    include_management_events = true                 # 관리 이벤트 포함
  }
  depends_on = [aws_s3_bucket_policy.cloudtrail_policy] # 버킷 정책 적용 후 실행
}

#----------------------------------------------------------------------------
# SNS 토픽 생성: 알림 전송용
#----------------------------------------------------------------------------
resource "aws_sns_topic" "sns_securitygroup_alarm" {
  name = "sns-securitygroup-alarm"                   # SNS 토픽 이름
}

#----------------------------------------------------------------------------
# 이메일 구독자 추가: 알림을 이메일로 전송
#----------------------------------------------------------------------------
resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.sns_securitygroup_alarm.arn # SNS 토픽 ARN
  protocol  = "email"                                   # 이메일 프로토콜 사용
  endpoint  = var.alert_email                           # 수신할 이메일 주소 (변수)
}

#----------------------------------------------------------------------------
# Lambda 함수 구독자 추가: SNS 이벤트 발생 시 Lambda 호출
#----------------------------------------------------------------------------
resource "aws_sns_topic_subscription" "lambda_sub" {
  topic_arn = aws_sns_topic.sns_securitygroup_alarm.arn
  protocol  = "lambda"                                 # Lambda 호출용 프로토콜
  endpoint  = aws_lambda_function.lambda_securitygroup_alarm.arn # Lambda 함수 ARN
}

#----------------------------------------------------------------------------
# Lambda 실행 역할 생성: Lambda가 AWS 리소스 접근 가능하게 함
#----------------------------------------------------------------------------
resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"                           # 역할 이름
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Principal = {
        Service = "lambda.amazonaws.com"              # Lambda 서비스가 이 역할을 맡을 수 있도록 허용
      },
      Effect = "Allow",
      Sid    = ""
    }]
  })
}

#----------------------------------------------------------------------------
# Lambda에 기본 실행 정책 부여: CloudWatch 로그 접근 허용
#----------------------------------------------------------------------------
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

#----------------------------------------------------------------------------
# Lambda 함수 생성: 보안 그룹 변경 탐지 후 Discord로 전송
#----------------------------------------------------------------------------
resource "aws_lambda_function" "lambda_securitygroup_alarm" {
  function_name = "lambda-securitygroup-alarm"            # Lambda 함수 이름
  role          = aws_iam_role.lambda_exec.arn            # 실행 역할 ARN
  handler       = "lambda.lambda_handler"                 # Lambda 진입점
  runtime       = "python3.13"                            # 실행 환경
  filename      = "lambda.zip"                            # 업로드할 zip 파일
  environment {
    variables = {
      DISCORD_WEBHOOK_URL = var.discord_webhook_url       # Discord Webhook URL
    }
  }
  depends_on = [aws_iam_role_policy_attachment.lambda_basic_execution]
}

#----------------------------------------------------------------------------
# Lambda가 SNS로부터 호출될 수 있도록 권한 부여
#----------------------------------------------------------------------------
resource "aws_lambda_permission" "allow_sns" {
  statement_id  = "AllowExecutionFromSNS"
  action        = "lambda:InvokeFunction"                 # Lambda 호출 권한
  function_name = aws_lambda_function.lambda_securitygroup_alarm.function_name
  principal     = "sns.amazonaws.com"                     # SNS 서비스가 호출함
  source_arn    = aws_sns_topic.sns_securitygroup_alarm.arn
}

#----------------------------------------------------------------------------
# EventBridge 규칙 생성: 보안 그룹 변경 이벤트 감지
#----------------------------------------------------------------------------
resource "aws_cloudwatch_event_rule" "eventbridge_securitygroup_changerule" {
  name        = "eventbridge-securitygroup-changerule"
  description = "Detects changes to Security Groups"      # 설명
  event_pattern = jsonencode({
    source = ["aws.ec2"],                                  # EC2 서비스
    "detail-type" = ["AWS API Call via CloudTrail"],       # CloudTrail을 통한 API 호출
    detail = {
      eventSource = ["ec2.amazonaws.com"],
      eventName = [                                        # 감지할 이벤트 목록
        "AuthorizeSecurityGroupIngress",
        "AuthorizeSecurityGroupEgress",
        "RevokeSecurityGroupIngress",
        "RevokeSecurityGroupEgress",
        "DeleteSecurityGroup"
      ]
    }
  })
}

#----------------------------------------------------------------------------
# EventBridge 규칙 트리거 대상 지정: SNS 토픽으로 연결
#----------------------------------------------------------------------------
resource "aws_cloudwatch_event_target" "send_to_sns" {
  rule = aws_cloudwatch_event_rule.eventbridge_securitygroup_changerule.name
  arn  = aws_sns_topic.sns_securitygroup_alarm.arn
}

```

</details>

<details>

<summary>variables.tf</summary>



</details>

<details>

<summary>terraform.tfvars</summary>



</details>

<details>

<summary>lambda.zip (lambda.py)</summary>



</details>

<details>

<summary>Terraform 실행</summary>



</details>

