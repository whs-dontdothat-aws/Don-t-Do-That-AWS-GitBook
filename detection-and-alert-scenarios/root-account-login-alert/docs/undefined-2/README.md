# 스냅샷 / 자원 공유를 통한 은폐 및 유출 시도 시나리오

***

**\[ 목차 ]**

[#undefined](./#undefined "mention")

[#undefined-1](./#undefined-1 "mention")

[#undefined-2](./#undefined-2 "mention")

[#undefined-3](./#undefined-3 "mention")

[#undefined-4](./#undefined-4 "mention")

[#undefined-5](./#undefined-5 "mention")

[terraform-cli.md](terraform-cli.md "mention")

***

### **\[ 시나리오 안내 ]**

<table data-header-hidden data-full-width="false"><thead><tr><th width="137.5"></th><th></th></tr></thead><tbody><tr><td>내용</td><td>공격자가 EC2의 EBS 볼륨 스냅샷을 생성하고 외부 AWS 계정과 공유하여 민감 데이터를 유출하려는 시도를 탐지합니다.</td></tr><tr><td>사용 서비스</td><td>CloudTrail, EventBridge, SNS, EC2, EBS</td></tr><tr><td>탐지 조건</td><td><p></p><ol><li>AWS 계정에서 EBS 스냅샷을 생성합니다.</li><li>생성되어있는 스냅샷을 현재 계정이 아닌 다른 계정으로 공유하려는 시도를 검토합니다.</li><li>해당 행위가 파악되는 경우 알람을 발생시킵니다.</li><li>스냅샷 삭제까지 연관지어 알람을 발생시킬 수 있다면 더 완벽합니다.</li></ol></td></tr><tr><td>알림 방식</td><td>SNS + Email 및 Discord 전송</td></tr></tbody></table>

#### **실습 개요**

* 이 워크북에서는 AWS EC2 인스턴스의 EBS 볼륨 스냅샷이 외부 계정과 공유되는 행위를 모니터링하고 탐지하는 방법을 실습합니다.
* EBS 스냅샷은 EC2 인스턴스의 디스크 데이터를 백업하거나 마이그레이션할 때 사용되며, 스냅샷 내부에는 시스템 설정, 민감한 로그, 사용자 데이터 등이 포함될 수 있습니다. 이러한 스냅샷이 외부 계정과 공유될 경우, 의도치 않은 데이터 유출로 이어질 수 있습니다.
* 본 워크북에서는 CloudTrail, EventBridge, SNS, Lambda를 연동하여 스냅샷 공유 시도를 실시간으로 감지하고, 알림을 이메일 및 디스코드 채널로 전송하는 자동화된 경보 체계를 구성하는 실습을 진행합니다.

#### **학습 목표**

* CloudTrail을 활성화하고, EBS 스냅샷 생성 및 속성 변경 이벤트를 로깅하는 방법을 학습합니다.
* EventBridge 규칙을 생성하여 EBS 스냅샷이 외부 계정과 공유되는 API 호출을 실시간으로 탐지하는 방법을 이해합니다.
* Amazon SNS를 사용하여 경보 메시지를 이메일과 Lambda에 동시에 전달하는 멀티 알림 채널 구성을 실습합니다.
* Lambda 함수를 통해 공유 대상 계정 ID를 판별하고, Discord Webhook을 통해 실시간 경고 메시지를 전송하는 자동화 프로세스를 구현합니다.
* 테스트 이벤트를 직접 생성하고, 실제로 스냅샷을 공유하여 탐지 및 알림 흐름이 정상적으로 작동하는지 검증하는 방법을 학습합니다.

***

### **\[ 시나리오 전체적인 흐름 ]**

<figure><img src="../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

<table data-header-hidden data-full-width="true"><thead><tr><th width="127"></th><th width="283"></th><th></th></tr></thead><tbody><tr><td><strong>AWS Service</strong></td><td><strong>Service Purpose</strong></td><td><strong>Workbook Usage</strong></td></tr><tr><td>EC2</td><td>AWS 클라우드에서 확장 가능한 컴퓨팅 리소스를 제공하는 가상 서버 서비스입니다.</td><td>공격자가 접근한 인스턴스의 디스크 볼륨을 백업하기 위해 연결된 EBS 볼륨의 스냅샷을 생성하는 대상이 됩니다.</td></tr><tr><td>EBS</td><td>EC2 인스턴스에 연결되는 블록 스토리지입니다. 안정적이고 지속적인 데이터 저장이 가능하며, 스냅샷 기능을 제공하여 백업 및 복제를 지원합니다.</td><td>EC2 인스턴스에 연결된 EBS 볼륨의 스냅샷을 생성하고, 해당 스냅샷을 외부 계정과 공유하는 행위(ModifySnapshotAttribute)를 통해 유출 시도가 이루어집니다.</td></tr><tr><td>CloudTrail</td><td>AWS 계정에서 발생하는 모든 API 호출 이력 및 이벤트 로그를 캡처하여 보안 감사, 운영 분석, 컴플라이언스 검토를 가능하게 해주는 서비스입니다.</td><td>EBS 스냅샷 생성(<mark style="color:$danger;"><code>CreateSnapshot</code></mark>) 및 속성 수정(<mark style="color:$danger;"><code>ModifySnapshotAttribute</code></mark>) 등의 API 호출 이력을 수집하며, 이 기록을 기반으로 이상 행위를 탐지합니다.</td></tr><tr><td>EventBridge</td><td>AWS 서비스 및 애플리케이션에서 사용자가 정의한 이벤트가 발생하는 경우 다양한 대상 서비스(Lambda, SNS 등)로 전달하는 서비스 입니다.복잡한 이벤트 버스를 구현하거나 스케쥴링 기능으로도 활용할 수 있습니다.</td><td>CloudTrail에서 기록된 이벤트 중 EBS 스냅샷이 외부 계정과 공유되는 특정 패턴을 탐지하여, 해당 이벤트를 SNS와 Lambda 함수로 전달하는 역할을 수행합니다.</td></tr><tr><td>SNS</td><td>발행-구독 기반의 메시징 서비스 입니다.이벤트를 HTTP, HTTPS, Email, SMS, Lambda, SQS 등 다양한 엔드포인트로 전달할 수 있습니다.</td><td>EventBridge에서 탐지한 이벤트를 이메일 또는 Lambda로 전달하여 경고 알림을 전송합니다. 수신자는 이메일 또는 Discord 채널을 통해 실시간으로 이상 징후를 인지할 수 있습니다.</td></tr><tr><td>Lambda</td><td>서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다. 다양한 이벤트 소스(S3, Eventbridge, SNS 등)와 연동하여 이벤트에 대한 응답으로 코드를 실행할 수 있습니다.</td><td>SNS로부터 전달받은 이벤트 메시지를 가공하여 Discord Webhook 형식으로 변환하고, 보안 운영자가 실시간 알림을 받을 수 있도록 메시지를 전송합니다. 필요한 경우 메시지 내용을 분석해 유출 경로 및 대상 계정을 파악할 수 있습니다.</td></tr></tbody></table>

#### **참고 사항**

* 해당 시나리오에 맞게 리소스명은 임의로 설정하였으며 사용자가 원하는 이름으로 바꿔도 무방합니다.
* 본 워크북에서 생성되는 리소스 중 일부는 Terraform code를 통해 제공되고 있습니다.
* 주요 리소스에 대한 사전 구성을 필요로 하는 경우 하단의 “xx. Terraform으로 리소스 구현” 에서 내용을 참고하실 수 있습니다.
* 본 서비스의 주요 리소스는 “ap-northeast-2(Seoul)”에서 리전에서 진행됩니다. 주요 서비스 및 기능은 제공되는 서비스 리전에 따라 다를 수 있습니다.

#### **콘솔 변수 이름**

<table data-header-hidden data-full-width="false"><thead><tr><th width="160.33331298828125"></th><th width="279"></th><th></th></tr></thead><tbody><tr><td><strong>리소스 종류</strong></td><td><strong>리소스명 설정</strong></td><td><strong>목적</strong></td></tr><tr><td>CloudTrail Trail</td><td><strong><code>ct-snapshot-monitor</code></strong></td><td>AWS 계정 내 API 활동을 감시하고 로그를 S3로 저장</td></tr><tr><td>SNS Topic</td><td><strong><code>sns-snapshot-alarm</code></strong></td><td>이벤트 발생 시 이메일 및 Lambda로 알림을 전송하는 주제</td></tr><tr><td>Lambda Function</td><td><strong><code>lambda-ebs-snapshot-alarm</code></strong></td><td>SNS 메시지를 파싱하여 Discord Webhook으로 이상 행위 알림 전송</td></tr><tr><td>EventBridge Rule</td><td><strong><code>eventbridge-ebs-detect-snapshot</code></strong></td><td>특정 EBS 이벤트 발생 시 SNS로 이벤트 전송하는 트리거 역할</td></tr><tr><td>EC2</td><td><strong><code>ec2-ebs-test</code></strong></td><td>EBS 스냅샷 생성 및 삭제 등 테스트용 볼륨을 생성하여 보안 탐지 시나리오 검증</td></tr><tr><td>Discord Channel</td><td><strong><code>ebs-snapshot-alarm</code></strong></td><td>CloudTrail 알림 메시지를 수신하고 확인할 수 있는 알림 채널</td></tr></tbody></table>
