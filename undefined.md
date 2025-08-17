# 루트 계정 로그인 알림

### **\[ 시나리오 안내 ]**&#x20;

<table data-header-hidden><thead><tr><th width="213.20001220703125"></th><th></th></tr></thead><tbody><tr><td><strong>내용</strong></td><td><strong>루트 계정은 AWS 계정 생성 시에만 활용되고 IAM User등으로 관리되어 로그인 횟수가 적어집니다. 또한 모든 권한을 가지고 있기 때문에 실제 운영 환경에서 사용이 최소화되어야 합니다. 루트 계정으로 콘솔 로그인이 발생한 경우 실시간 탐지를 통해 이상 행위 여부를 판단하는 시나리오를 구현해 봅니다.</strong></td></tr><tr><td>사용 서비스</td><td>CloudTrail, CloudWatch, EventBridge Rule, SNS, Lambda</td></tr><tr><td>탐지 조건</td><td>웹 콘솔에 로그인하는 이벤트 중 루트 사용자인 경우를 탐지합니다.</td></tr><tr><td>알림 방식</td><td>SNS를 통해 Email로 알림을 전송하고, 동시에 Lambda 함수를 활용해 Webhook을 통해 Discord로도 알림을 전송합니다.</td></tr><tr><td>대응</td><td>알림으로 내용을 확인하고 사용자가 대처합니다.</td></tr></tbody></table>

***

### **\[ 시나리오 전체적인 흐름 ]**

<figure><img src=".gitbook/assets/image (33).png" alt="" width="375"><figcaption></figcaption></figure>

| **AWS Service**                       | **Service Purpose**                    | **Workbook Usage**                                   |
| ------------------------------------- | -------------------------------------- | ---------------------------------------------------- |
| **CloudTrail**                        | AWS 계정 내 API 호출 및 로그인 활동을 기록하는 서비스입니다. | 루트 계정의 콘솔 로그인 이벤트를 감지하기 위한 로그 데이터 를수집합니다.            |
| **EventBridge**                       | 이벤트 패턴에 따라 자동으로 트리거를 발생시키는 서비스입니다.     | 루트 로그인 이벤트가 감지되면 지정된 Lambda와 SNS를 호출하도록 이벤트 룰 설정합니다. |
| **SNS (Simple Notification Service)** | 메시지를 여러 구독자에게 전달하는 메시징 서비스입니다.         | 루트 로그인 알림을 Email로 전송합니다.                             |
| **Lambda**                            | 서버 없이 코드를 실행할 수 있는 이벤트 기반 컴퓨팅 서비스입니다.  | 루트 로그인 이벤트 발생 시 Discord Webhook으로 메시지 전송합니다.         |
| **S3**                                | 객체 기반 스토리지 서비스입니다.                     | CloudTrail 로그 저장소로 활용합니다.                            |

**실습 개요**

* 이  워크북에서는 루트 계정의 AWS 콘솔 로그인 이벤트를 실시간으로 탐지하고, 이를 이메일(SNS)과 Discord(Webhook)를 통해 즉시 알림으로 전송하는 시나리오를 구현합니다.
* CloudTrail과 EventBridge를 활용해 이벤트를 감지하고, Lambda를 통해 다중 채널로 알림을 전달합니다.

#### **학습 목표**

* 루트 계정 로그인 탐지 시나리오 설계를 통해 보안 사고 대응 체계 학습합니다.
* AWS의 Event 기반 자동화 구조 (CloudTrail → EventBridge → Lambda/SNS) 이해합니다.
* 실시간 알림 시스템 구축 및 외부 채널(Discord) 연동 방법 습득합니다.

***

**참고 사항**

* 루트 로그인은 글로벌 이벤트 → EventBridge는 us-east-1에서만 감지 가능합니다.
* 실습 시 EventBridge와 Lambda를 us-east-1에 구성해야 알림이 정상 작동합니다.
* CloudTrail은 서울에 두어도 무방하나, 글로벌 이벤트 포함 옵션이 켜져 있어야 합니다.
* “루트 계정 로그인 알림 시나리오”실습 시, region을 서울로 하면 안 되고 반드시 버지니아 북부(미국 동부, us-east-1)로 설정해주어야  합니다.

\
&#xNAN;**\[ 콘솔 리소스명 ]**

<table data-header-hidden><thead><tr><th width="157.79998779296875"></th><th></th><th></th></tr></thead><tbody><tr><td><strong>리소스 종류</strong></td><td><strong>리소스명 설정</strong></td><td><strong>목적</strong></td></tr><tr><td>S3 Bucket</td><td><strong><code>s3-root-login-detect</code></strong></td><td>CloudTrail 로그 저장소로 사용하여 root 로그인 이벤트 기록</td></tr><tr><td>Lambda Function</td><td><strong><code>lambda-root-login</code></strong></td><td>EventBridge 이벤트 트리거 시 root 로그인 이벤트 처리 및 SNS/Discord 전송</td></tr><tr><td>CloudTrail</td><td><strong><code>ct-trail-monitor</code></strong></td><td>AWS 계정의 root 로그인 이벤트를 감지하기 위한 로그 수집 설정</td></tr><tr><td>EventBridge</td><td><strong><code>eventbridge-root-login-pattern</code></strong></td><td>root 로그인 이벤트 패턴 감지를 위한 이벤트 규칙 설정</td></tr><tr><td>SNS Topic</td><td><strong><code>sns-root-login-alarm</code></strong></td><td>이메일 등의 경고 채널로 root 로그인 알림 전송</td></tr><tr><td>Discord 채널</td><td><strong><code>root-login-alarm</code></strong></td><td>Lambda에서 전송한 root 로그인 알림 수신용 채널</td></tr></tbody></table>

***

### **\[ 시나리오 상세 구현 과정 ]**

