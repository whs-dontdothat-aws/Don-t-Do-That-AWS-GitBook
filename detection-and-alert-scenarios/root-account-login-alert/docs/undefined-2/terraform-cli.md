# Terraform (CLI)

### **\[ 시나리오 상세 구현 과정 ]**

> **IaC를 사용하는 이유**
>
> ***
>
> 수동 설정은 실수 위험이 크고 반복이 어렵지만, Terraform을 사용하면 콘솔에서 구현한 시나리오를 코드로 재현할 수 있어 **버전 관리, 재사용, 자동화**에 유리하다.



**\[ 참고 ]**

실습 전 [IaC 실습 전 환경 구성](https://app.gitbook.com/o/Ovrk2sznEJVoEThBhvLL/s/OxBH8cyDzBTzXyUOUzPl/) 을 참고하여 환경 구성을 완료했다면 Terraform 코드를 실행하여 리소스를 자동으로 생성할 수 있다.



**\[ Terraform 파일 구조 ]**

```
Terraform-code/
├── main.tf                    # 모든 주요 Terraform 리소스 정의 
├── variables.tf               # 입력 변수 정의 (Email, Discord Webhook 등)
├── terraform.tfvars           # 민감 데이터 정의 (Email, Webhook URL 등)
├── lambda_function.zip        # 디스코드 알림을 전송하는 패키지된 Lambda 코드
│   └── lambda_function.py     # Lambda 함수 원본 코드
```



**\[ Terraform 구현 코드 ]**

{% tabs %}
{% tab title="main.tf" %}
```hcl
###############################################################################
# PROVIDER ─ AWS 리전 설정
###############################################################################
provider "aws" {
  region = "ap-northeast-2" # 서울 리전 사용
}

###############################################################################
# 기본 VPC, 서브넷, AMI 정보 정의
###############################################################################
data "aws_vpc" "default" {
  default = true # 기본 VPC 불러오기
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id] # 기본 VPC의 서브넷 ID 목록 가져오기
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"] # Amazon 공식 이미지

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"] # Amazon Linux 2 최신 AMI
  }
}

###############################################################################
# EC2 인스턴스 및 보안 그룹 구성
###############################################################################
resource "aws_security_group" "ec2_eg" {
  description = "Allow SSH and ICMP"
  vpc_id      = data.aws_vpc.default.id # 기본 VPC에 생성

  ingress {
    description = "Allow ping"
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"] # ICMP(ping) 전체 허용
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # SSH 전체 허용 (보안상 운영에는 부적절)
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] # 아웃바운드 전체 허용
  }

  tags = {
    Name = "ec2-sh" # 리소스 이름 태그
  }
}

resource "aws_instance" "ebs_snapshot_ec2" {
  ami                         = data.aws_ami.amazon_linux.id # 최신 Amazon Linux 2 사용
  instance_type               = "t3.micro" # 프리티어 해당 타입
  subnet_id                   = data.aws_subnets.default.ids[0] # 첫 번째 서브넷 사용
  vpc_security_group_ids      = [aws_security_group.ec2_eg.id] # 위에서 만든 보안그룹 연결
  associate_public_ip_address = true # 퍼블릭 IP 자동 할당

  tags = {
    Name = "ebs-snapshot-monitor-ec2"
  }
}

```
{% endtab %}

{% tab title="variable.tf" %}
```hcl
# 해당 파일에서는 변수를 여기다가 모두 선언
# terraform 실행 시 terraform.tfvars에 선언된 값을 바탕으로 값이 들어감

variable "discord_webhook_url" {
  description = "Discord Webhook URL"
  type        = string
}

variable "notification_email" {
  description = "Email address for alerts"
  type        = string
}
```
{% endtab %}

{% tab title="terraform.tfvars " %}
```hcl
discord_webhook_url = "discord web hook url"
notification_email  = "이메일 주소"
```
{% endtab %}

{% tab title="lambda_function.zip (lambda_function.py) " %}
```python
# lambda_function.zip 안에 있는 lambda_function.py
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
{% endtab %}

{% tab title="Terraform 실행" %}
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

{% hint style="info" %}
Terraform 프로젝트를 처음 시작할 때 실행하는 명령어로,\
작업 디렉토리를 초기화하고 필요한 설정 파일과 실행에 필요한 구성 요소들을 준비해준다.\
이후 plan, apply 등의 명령을 정상적으로 사용할 수 있는 상태로 만든다.
{% endhint %}

```bash
Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

{% hint style="info" %}
위와 같은 메시지가 출력되면,\
프로젝트가 초기화되어 Terraform 명령어를 사용할 수 있는 준비가 완료된 것이다.
{% endhint %}



**\[ plan ]**

```bash
terraform plan
```

{% hint style="info" %}
Terraform 코드 적용 시,\
인프라에 어떤 변경이 발생할지 미리 확인할 수 있는 실행 계획을 보여주는 명령어이다.
{% endhint %}

```bash
Plan: 24 to add, 0 to change, 0 to destroy.
```

{% hint style="info" %}
위와 같은 메시지가 출력되면, 24개의 리소스가 새로 생성될 예정이며 실행 계획이 정상적으로 생성된 상태를 의미한다. 이 출력 결과는 적용해도 문제가 없는 준비 완료 상태임을 의미한다.
{% endhint %}



**\[ apply ]**

```bash
terraform apply
```

{% hint style="info" %}
terraform apply 명령어는 실행 계획(plan)에 따라 실제로 클라우드 인프라를 생성, 변경, 삭제하는 작업을 수행한다. Plan 단계에서 검토한 내용을 기반으로 실제 인프라에 반영하고자 할 때 사용한다.
{% endhint %}

```bash
Apply complete! Resources: 24 added, 0 changed, 0 destroyed.
```

{% hint style="info" %}
위와 같은 메시지가 출력되면,\
모든 리소스가 정상적으로 생성되었거나 업데이트되어 인프라 상태가 원하는 대로 적용된 것이다.
{% endhint %}



**\[ 이메일 인증 ]**

<figure><img src="../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
terraform apply 이후, 설정한 이메일 주소로 SNS의 Subscription Confirmation 메일이 전송된다.\
이메일을 열어 **Confirm subscription** 버튼을 클릭해야 알림 수신이 정상적으로 설정된다.
{% endhint %}

<figure><img src="../.gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**Confirm subscription**를 눌러 인증을 완료하면, SNS 구독이 정상적으로 등록된 것이다.
{% endhint %}



**\[ 테스트 진행 ]**

{% embed url="https://app.gitbook.com/o/Ovrk2sznEJVoEThBhvLL/s/O10UPNyD9g2PLsfoMELd/undefined-2" %}

{% hint style="info" %}
인증 후 위를 참고하여 테스트를 진행하면 된다.
{% endhint %}



**\[ destroy ]**

```bash
terraform destroy
```

{% hint style="info" %}
Terraform으로 생성된 모든 인프라 리소스를 자동으로 삭제하는 명령어이다.\
**실습 완료 후**에는 해당 명령어로 불필요한 리소스를 정리할 수 있다.
{% endhint %}

```bash
Destroy complete! Resources: 0 destroyed.
```

{% hint style="info" %}
위와 같은 메시지가 출력되면, 모든 리소스가 성공적으로 정리되었음을 의미한다.
{% endhint %}
{% endtab %}
{% endtabs %}
