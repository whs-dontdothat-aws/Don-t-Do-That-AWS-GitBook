# 새로운 IAM User의 생성, 삭제 탐지

**\[ 목차 ]**

[#undefined](iam-user.md#undefined "mention")

[#undefined-4](iam-user.md#undefined-4 "mention")

[#undefined-1](iam-user.md#undefined-1 "mention")

[#undefined-2](iam-user.md#undefined-2 "mention")

[#undefined-3](iam-user.md#undefined-3 "mention")

[#undefined-5](iam-user.md#undefined-5 "mention")

[#undefined-6](iam-user.md#undefined-6 "mention")

[#id-1.-cloudtrail](iam-user.md#id-1.-cloudtrail "mention")

[#id-2.-lambda-discord](iam-user.md#id-2.-lambda-discord "mention")

[#id-3.-sns](iam-user.md#id-3.-sns "mention")

[#id-4.-eventbridge](iam-user.md#id-4.-eventbridge "mention")

***

### **\[ 시나리오 안내 ]**

<table><thead><tr><th width="124.77783203125">내용</th><th>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나 공격자가 콘솔 접근을 위해서도 사용할 수 있습니다. 사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고 변경 이력을 검토하는 시나리오를 구현 해봅니다.</th></tr></thead><tbody><tr><td>사용 서비스</td><td>Lambda, CloudTrail, SNS, EventBridge</td></tr><tr><td>탐지 조건</td><td>IAM User의 생성, 삭제를 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td>알림을 확인하고 사용자가 대처한다.</td></tr><tr><td></td><td>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나 공격자가 콘솔 접근을 위해서도 사용할 수 있습니다. 사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고 변경 이력을 검토하는 시나리오를 구현 해봅니다.</td></tr></tbody></table>

<table data-header-hidden><thead><tr><th width="278.99993896484375"></th><th></th></tr></thead><tbody><tr><td>내용</td><td>새로운 임직원이 입사하거나 AWS 접근이 필요한 경우 IAM 사용자가 신규 생성될 수 있으나<br>공격자가 콘솔 접근을 위해서도 사용할 수 있습니다.<br>사용자 계정 정책에 영향을 줄 수 있는 행위를 탐지하고<br>변경 이력을 검토하는 시나리오를 구현 해봅니다.</td></tr><tr><td>사용 서비스</td><td>Lambda, CloudTrail, SNS, EventBridge</td></tr><tr><td>탐지 조건</td><td>IAM User의 생성, 삭제를 탐지하는 조건 확인</td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr><tr><td>대응</td><td>알림을 확인하고 사용자가 대처한다.</td></tr></tbody></table>

***

### **\[ 시나리오 전체적인 흐름 ]**

| IAM         | AWS 리소스에 접근할 수 있는 사용자(User), 그들의 권한(Policy), 인증 정보(Access Key 등)를 관리하는 서비스입니다. IAM을 통해 개별 사용자 계정을 생성하고, 사용자 별로 권한을 부여할 수 있으며, 사용자 생성/삭제 이력에 대한 감사 추적이 가능합니다.                     | IIAM 사용자가 생성 및 삭제될 때 CloudTrail이 이를 기록하고, EventBridge를 통해 탐지되며, 알림이 SNS를 통해 전달되는 구조의 핵심 리소스 대상입니다. |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| CloudTrail  | <p>AWS 계정에서 발생하는 모든 사용자 활동과 API 호출을 자동으로 기록하고, 이를 로그로 저장하는 서비스입니다.<br>기본적으로는 최근 90일간의 이벤트만 콘솔에서 조회할 수 있지만,<br>추적을 생성하면 지정한 S3 버킷에 이벤트 로그 파일이 지속적으로 저장되어 장기 보관 및 분석이 가능합니다.</p> | EventBridge에서 CreateUser, DeleteUser로그를 참조할 수 있도록 추적을 생성합니다.                                       |
| EventBridge | AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.                                                        | EventBridge에서 CreateUser, DeleteUser로그를 참조할 수 있도록 추적을 생성합니다.                                       |
| SNS         | 발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.                                                                                         | Eventbridge로 부터 이벤트를 수신하면 SNS 주제를 구독하고 있는 엔드포인트(이메일, Lambda)로 이벤트 이력을 전송합니다.                       |
| Lambda      | 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.                                                        | SNS로부터 호출되면 사전 지정한 외부 Webhook(Discord)으로 메시지를 전달합니다.                                               |

#### 실습 개요

* 이 워크북에서는 IAM 사용자가 생성, 삭제되었을 때 이를 탐지하는 방법을 학습합니다.
* IAM 사용자 생성 및 삭제 탐지는 AWS 계정 보안의 핵심 지표로, 권한 남용, 내부 위협, 외부 침해 행위를 조기에 식별하고 대응하기 위해 반드시 구현되어야 할 보안 모니터링 항목입니다.
* 본 워크북에서는 CloudTrail과 EventBridge, SNS 등을 활용해 IAM 사용자 생성/삭제 이벤트 발생 시 이메일 및 디스코드 알림을 받도록 구현하는 방법을 실습합니다.

#### 학습 목표

* AWS에서 IAM 사용자의 생성 및 삭제 이벤트를 탐지하는 방법을 학습합니다.
* CloudTrail과 EventBridge를 활용해 IAM 사용자 생성/삭제 이벤트를 수집하고, 이를 SNS와 연동하여 이메일 및 디스코드 알림을 전송하는 실습을 진행합니다.
* 생성/삭제 이벤트에 대한 탐지 조건을 코드로 정의하여 실시간 대응 체계를 구축하는 과정을 이해합니다.
* 실제로 IAM 사용자를 생성 및 삭제하여 이벤트 탐지가 정상적으로 동작하는지 실습을 통해 검증하는 경험을 쌓습니다.

***

#### 참고 사항

* **IAM**은 AWS **전역(Global) 서비스**로, 그 이벤트는 **버지니아 북부(us-east-1)** 리전의 CloudTrail과EventBridge를 통해서만 탐지할 수 있기 때문에 본 실습은 \*\*“us-east-1(Virginia)”\*\*리전에서 진행됩니다.
* 본 워크북에서 생성되는 리소스 중 일부는 Terraform code를 통해 제공되고 있습니다.
* 주요 리소스에 대한 사전 구성을 필요로 하는 경우 하단의 “xx. Terraform으로 리소스 구현” 에서 내용을 참고하실 수 있습니다.
* 해당 시나리오에 맞게 리소스명은 임의로 설정하였으며 사용자가 원하는 이름으로 바꿔도 무방합니다.



**\[ 콘솔 리소스명 ]**

| **리소스 종류**       | **리소스명 설정**                 | **목적**                                             |
| ---------------- | --------------------------- | -------------------------------------------------- |
| CloudTrail Trail | **`ct-trail-monitor`**      | AWS 계정 내 API 활동을 감시하고 로그를 S3로 저장                   |
| Discord 채널       | **`iam-alarm`**             | SNS에 연동된 Lambda 함수를 통해 알림 메시지를 수신하고 확인할 수 있는 알림 채널 |
| Lambda Function  | **`lambda-iam-user-alarm`** | SNS 메시지를 파싱하여 Discord Webhook으로 이상 행위 알림 전송        |
| SNS Topic        | **`sns-iam-user-alarm`**    | 이벤트 발생 시 이메일 및 Lambda로 알림을 전송하는 주제                 |
| EventBridge Rule | **`eventbridge-ct-detect`** | 특정 CloudTrail 이벤트 발생 시 SNS로 이벤트 전송하는 트리거 역할        |
| IAM User         | **`IAM-test-user`**         | IAM 사용자 생성 및 삭제 이벤트가 발생하는지 테스트하기 위해 생성하는 테스트 사용자   |

***

### **\[ 시나리오 상세 구현 과정 ]**

<details>

<summary></summary>



</details>

<details>

<summary></summary>



</details>

