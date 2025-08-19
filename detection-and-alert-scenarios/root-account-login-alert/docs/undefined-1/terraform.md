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
```



</details>

<details>

<summary>variables.tf</summary>

```hcl
```



</details>

<details>

<summary>terraform.tfvars</summary>

```hcl
```

</details>

<details>

<summary><strong>lambda.zip (lambda_function.py)</strong></summary>

**의존성 없이 zip 만들기**

```bash
zip lambda.zip lambda_function.py
```

```python
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



**\[ 전체 코드 압축 파일 ]**

