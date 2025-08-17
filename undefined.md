# 루트 계정 로그인 알림



### **\[ 시나리오 안내 ]**&#x20;

<table data-header-hidden><thead><tr><th width="213.20001220703125"></th><th></th></tr></thead><tbody><tr><td><strong>내용</strong></td><td><strong>루트 계정은 AWS 계정 생성 시에만 활용되고 IAM User등으로 관리되어 로그인 횟수가 적어집니다. 또한 모든 권한을 가지고 있기 때문에 실제 운영 환경에서 사용이 최소화되어야 합니다. 루트 계정으로 콘솔 로그인이 발생한 경우 실시간 탐지를 통해 이상 행위 여부를 판단하는 시나리오를 구현해 봅니다.</strong></td></tr><tr><td>사용 서비스</td><td>CloudTrail, CloudWatch, EventBridge Rule, SNS, Lambda</td></tr><tr><td>탐지 조건</td><td>웹 콘솔에 로그인하는 이벤트 중 루트 사용자인 경우를 탐지한다.</td></tr><tr><td>알림 방식</td><td>SNS를 통해 Email로 알림을 전송하고, 동시에 Lambda 함수를 활용해 Webhook을 통해 Discord로도 알림을 전송한다.</td></tr><tr><td>대응</td><td>알림으로 내용을 확인하고 사용자가 대처한다.</td></tr></tbody></table>







