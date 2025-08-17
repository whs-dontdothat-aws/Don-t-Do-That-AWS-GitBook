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

*   **main.tf**

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



*   **variables.tf**

    ```hcl
    # 해당 파일에서는 변수를 여기다가 모두 선언
    # terraform 실행 시 terraform.tfvars에 선언된 값을 바탕으로 값이 들어감

    variable "discord_webhook_url" {
      description = "Discord Webhook URL"
      type        = string
      sensitive   = true
    }

    variable "alert_email" {
      description = "Email for SNS alerts"
      type        = string
    }
    ```



*   **terraform.tfvars**

    ```hcl
    discord_webhook_url = "<https://discord.com/api/webhooks/웹> 훅 복사"
    alert_email         = "알림받을 이메일"
    ```



*   **lambda.zip (lambda.py)**

    ```python
    # lambda_function.zip 안에 있는 lambda.py
    import os
    import json
    import urllib3

    # HTTP 요청을 위한 PoolManager 객체 생성
    http = urllib3.PoolManager()
    # Lambda 환경 변수에서 Discord Webhook URL을 가져옴 (main.tf에서 DISCORD_WEBHOOK_URL로 지정)
    WEBHOOK = os.environ["DISCORD_WEBHOOK_URL"]

    def lambda_handler(event, context):
        # SNS 메시지는 여러 개의 Records로 들어올 수 있으므로 반복 처리
        for rec in event["Records"]:
            try:
                # SNS 메시지의 JSON 문자열을 파싱하여 Python 객체로 변환
                msg = json.loads(rec["Sns"]["Message"])
            except Exception as e:
                # 파싱 실패 시 에러 로그 출력 후 다음 메시지로 넘어감
                print("SNS 메시지 파싱 실패:", str(e))
                continue

            # CloudTrail 이벤트의 상세 정보 추출
            detail = msg.get("detail", {})

            # 주요 정보들 추출
            event_name = detail.get("eventName", "N/A")                         # API 호출 이벤트 이름
            event_time_utc = msg.get("time", "N/A")                            # UTC 기준 발생 시간
            event_time_kst = event_time_utc.replace("T", " ").replace("Z", "") if event_time_utc != "N/A" else "N/A"  # KST 포맷으로 변환

            user_arn = detail.get("userIdentity", {}).get("arn", "N/A")       # 이벤트 발생 주체 (ARN)
            source_ip = detail.get("sourceIPAddress", "N/A")                  # 요청 발생 IP 주소
            aws_region = msg.get("region", "N/A")                             # AWS 리전
            account_id = msg.get("account", "N/A")                            # AWS 계정 ID

            # 보안 그룹 ID 추출 (groupId 또는 groupIds 필드에서 가져옴)
            sg_id = "N/A"
            params = detail.get("requestParameters", {})
            if "groupId" in params:
                sg_id = params["groupId"]
            elif "groupIds" in params and isinstance(params["groupIds"], list):
                sg_id = ", ".join(params["groupIds"])

            # Discord로 보낼 메시지 구성 (Markdown 스타일 포함)
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

            # Discord Webhook으로 HTTP POST 요청 전송
            http.request(
                "POST", WEBHOOK,
                body=json.dumps({"content": content}).encode(),               # 메시지를 JSON 문자열로 인코딩
                headers={"Content-Type": "application/json"}                 # 헤더에 JSON 콘텐츠 타입 명시
            )

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

