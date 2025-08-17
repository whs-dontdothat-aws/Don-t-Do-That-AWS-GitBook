# Security group의 정책 변경 탐지

***

### **\[ 시나리오 안내 ]**

<table data-header-hidden><thead><tr><th width="114.4000244140625"></th><th valign="middle"></th></tr></thead><tbody><tr><td>내용</td><td valign="middle">인스턴스에 적용된 보안 그룹은 간혹 정해진 환경에서만 접근 가능하도록 인바운드 포트를 Open하여 운영합니다. 만약 공격자가 이를 변경하는 경우 외부로부터의 불필요한 접근 허용하게 됩니다. 이를 시나리오로 탐지하고 대응하는 과정을 구현 해 봅니다.</td></tr><tr><td>사용 서비스</td><td valign="middle">CloudTrail, EventBridge, SNS, Lambda</td></tr><tr><td>탐지 조건</td><td valign="middle">SecurityGroup의 정책 변경을 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td valign="middle">SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td valign="middle">관리자가 직접 대응</td></tr></tbody></table>

#### 실습 개요

* 이 워크북에서는 AWS 보안 그룹(Security Group)의 인바운드 및 아웃바운드 규칙 변경을 실시간으로 탐지하고 알림을 받는 방법을 학습합니다.
* Security Group은 EC2 인스턴스를 포함한 다양한 AWS 리소스에 대한 네트워크 접근을 제어하는 가상 방화벽 역할을 합니다. 그러나 잘못 구성된 규칙은 외부 공격자에게 서비스 접근을 허용하여 보안 사고로 이어질 수 있습니다.
* 본 워크북에서는 AWS CloudTrail과 EventBridge를 활용하여 보안 그룹 변경 이벤트를 감지하고, Amazon SNS를 통해 이메일 및 Discord로 알림을 전송하는 탐지 및 대응 시나리오를 실습합니다.
* 실습을 통해 보안 그룹의 변경 이벤트가 실제로 발생했을 때 이를 자동으로 탐지하고 경고를 전달하는 일련의 경보 시스템을 구성하는 방법을 익힐 수 있습니다.

#### 학습 목표

* AWS CloudTrail을 활용하여 보안 그룹(Security Group)에 대한 변경 사항(API 호출 기록)을 추적하는 방법을 학습합니다.
* EventBridge를 사용하여 보안 그룹 인바운드/아웃바운드 규칙 변경 이벤트를 실시간으로 탐지하는 방법을 이해합니다.
* Amazon SNS를 설정하고, 이메일 및 Lambda를 활용한 Discord 알림 전송 방식에 대해 실습을 통해 익힙니다.
* 보안 그룹 규칙 변경 시 어떤 API가 호출되는지(`AuthorizeSecurityGroupIngress`, `RevokeSecurityGroupEgress` 등)를 식별하고, 이를 기반으로 탐지 조건을 설계할 수 있습니다.
* 실습 환경에서 실제로 보안 그룹의 인바운드 규칙을 변경하여 탐지 알림 시스템이 정상적으로 동작하는지를 검증하는 방법을 학습합니다.
* Lambda 함수를 활용해 CloudTrail 이벤트를 외부 시스템(Discord)으로 연동하는 과정을 체험합니다.

***

### \[ 시나리오 전체적인 흐름 ]

<figure><img src="../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

| **AWS Service** | **Service Purpose**                                                                                                                      | **Workbook Usage**                                                                                                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CloudTrail      | AWS 계정 내에서 발생하는 모든 API 호출 이벤트를 기록하는 서비스입니다. 사용자의 콘솔 작업, AWS CLI, SDK, 서비스 간 호출 등 모든 요청을 JSON 로그 형식으로 저장하며, 보안 및 감사 목적으로 활용됩니다.           | 보안 그룹(Security Group)의 인바운드/아웃바운드 정책이 변경될 때 해당 API 호출 이벤트 (`AuthorizeSecurityGroupIngress`, `RevokeSecurityGroupEgress` 등)를 기록하고, 이후 EventBridge가 이를 탐지할 수 있도록 기반 로그 데이터를 제공합니다. |
| EventBridge     | AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.                | Cloudtrail에서 발생한 이벤트들 중, 작성한 이벤트 패턴에 맞는 이벤트가 발생했을 때 SNS로 전달합니다.                                                                                                                  |
| SNS             | 발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.                                                 | Eventbridge로 부터 수신한 이벤트 이력을 확인하고 SNS 주제를 구독하고 있는 엔드포인트로 이벤트 이력을 전송합니다.                                                                                                           |
| Lambda          | 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.                | 사전 지정한 외부 Webhook(Slack, Discord 등)으로 메시지를 전달합니다.                                                                                                                                |
| EC2             | 클라우드 상에서 가상 서버(인스턴스)를 실행할 수 있도록 지원하는 AWS의 핵심 서비스입니다. EC2 인스턴스는 네트워크 트래픽 제어를 위해 보안 그룹(Security Group)을 할당받으며, 이는 인바운드/아웃바운드 접근 규칙을 정의합니다. | EC2 인스턴스에 적용된 Security Group이 변경될 때(예: 외부 접근 허용) CloudTrail이 이를 기록하고, EventBridge를 통해 탐지되며, 알림이 SNS를 통해 전달되는 구조의 핵심 리소스 대상입니다.                                                   |

***

#### 참고 사항

* 리소스명은 해당 시나리오에 맞게 임의로 설정하였으며 사용자가 원하는 이름으로 바꿔도 무방합니다.
* 본 워크북에서 생성되는 리소스 중 일부는 Terraform code를 통해 제공되고 있습니다.
* 주요 리소스에 대한 사전 구성을 필요로 하는 경우 하단의 “xx. Terraform으로 리소스 구현” 에서 내용을 참고하실 수 있습니다.
* 본 서비스의 주요 리소스는 “ap-northeast-2(Seoul)”에서 리전에서 진행됩니다. 주요 서비스 및 기능은 제공되는 서비스 리전에 따라 다를 수 있습니다.

**\[ 콘솔 리소스명 ]**

| **리소스 종류**       | **리소스명 설정**                                | **목적**                                                 |
| ---------------- | ------------------------------------------ | ------------------------------------------------------ |
| CloudTrail Trail | **`ct-securitygroup`**                     | AWS 계정 내 API 활동을 감시하고 로그를 S3로 저장                       |
| SNS Topic        | **`sns-securitygroup-alarm`**              | 이벤트 발생 시 이메일 및 Lambda로 알림을 전송하는 주제                     |
| Lambda Function  | **`lambda-securitygroup-alarm`**           | SNS 메시지를 파싱하여 Discord Webhook으로 이상 행위 알림 전송            |
| EventBridge Rule | **`eventbridge-securitygroup-changerule`** | 특정 CloudTrail 이벤트 발생 시 SNS로 이벤트 전송하는 트리거 역할            |
| EC2 instance     | **`ec2-securitygroup-test`**               | 보안 그룹 변경 이벤트 발생을 유도하기 위한 테스트용 EC2 인스턴스                 |
| Security Group   | **`securitygroup-test`**                   | 인바운드/아웃바운드 규칙을 변경하여 CloudTrail 이벤트 발생을 유도하는 테스트용 보안 그룹 |
| Discord 채널       | **`securitygroup-alarm`**                  | CloudTrail 알림 메시지를 수신하고 확인할 수 있는 알림 채널                 |

