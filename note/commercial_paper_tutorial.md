# 기업 어음 튜토리얼

## 개요

기업 어음 어플리케이션과 스마트 컨트랙트를 설치하고 사용하는 방법을 살펴본다.
이번 예제는 작업 중심의 주제로 개념보다 절차를 강조하여 설명한다.

![그림1. 개요](img/commercial_paper.diagram_1.png)

이번 예제에서는 두 조직 Magneto Corp와 DigiBank가 존재하며
하이퍼레저 패브릭 블록체인 네트워크인 PaperNet을 통해 기업 어음을 발행하여 거래한다.

먼저 테스트 네트워크를 MagnetoCorp의 직원 Isabella로 설정하여 어음을 발행할 것이다.
다음으로 Digibank의 Balaji로 전환하여 어음을 거래한다.
기업 어음을 일정 기간 보유했다가 MagnetoCorp와 교환하여 소량의 이익을 얻는다.

서로 다른 조직에서 개발자, 최종 사용자, 관리자 역할을 수행하여
하이퍼 패브릭 네트워크에서 상호 합의된 규칙에 따라 독립적으로 작업하는
두 조직이 어떻게 협업하는지 이해할 수 있다.

- 머신 설정, 샘플 다운로드
- 네트워크 생성
- 기업 어음 스마트 컨트랙트 검토
- MagnetoCorp와 DigiBank가 체인코드 정의를 승인하여 채널에 스마트 컨트랙트 배포
- MagnetoCorp 어플리케이션 구조 이해
- 지갑과 신원(ID) 사용
- MagnetoCorp 어플리케이션으로 기업 어음 발행
- DigiBank가 어플리케이션에서 스마트 컨트랙트를 사용하는 방법 이해
- DigiBank로 어플리케이션을 실행하고 어음을 사고 팔기

## 사전 준비 사항

Node.js 설치
[샘플 다운로드](https://github.com/hyperledger/fabric-samples)
```text
cd fabric-samples
또는
cd /mnt/c/Users/fabi8/go/src/github.com/LeeSeungYun1020/fabric-samples
ls
```
commercial-paper 디렉토리에 이번 예제가 위치한다.
다른 사용자 및 구성 요소에 대해 여러 명령 창을 사용한다.
- 피어, 오더러, CA 로그 출력
- MagnetoCorp, DigiBank 각각 관리자로 체인코드 승인
- Isabella, Balaji로 스마트 컨트랙트 실행하여 어음 거래

혼동의 여지가 있으므로 아래와 같이 코드 앞에 어떤 창에서 명령어를 실행해야하는지 명기한다.
```text
(isabella) ls
```

## 네트워크 생성

패브릭 테스트 네트워크를 이용하여 스마트 컨트랙트를 배포할 예정이다.
테스트 네트워크는 두 개의 피어 조직과 하나의 오더링 조직으로 구성된다.
두 피어 조직은 각각 하나의 피어를 운영하며
오더링 조직은 싱글 노드 래프트 오더링 서비스를 운영한다.
테스트 네트워크를 사용하여 mychannel 채널을 생성하고 두 조직 모두 채널의 멤버로 추가한다.

![그림2. 테스트 네트워크](img/commercial_paper.diagram.testnet.png)

패브릭 테스트 네트워크는 두 개의 피어 조직 Org1과 Org2, 하나의 오더링 조직으로 구성된다.
각 구성 요소는 Docker 컨테이너로 실행된다.

각각의 조직은 각각이 인증 기관(CA)을 이용한다.
두 피어와 상태 데이터베이스, 오더링 서비스 노드, 각 조직의 CA는 각각 Docker 컨테이너에서 실행된다.
프로덕션 환경에서 일반적으로 조직은 패브릭 네트워크 전용 CA를 사용하기 보다
다른 시스템에서 기존에 사용하던 CA를 사용한다.

테스트 네트워크의 두 조직을 통해 블록체인 원장과 상호작용할 수 있다.
이번 예제에서 Org1을 DigiBank로 Org2를 MagnetoCorp로 생각하고 동작시켜 본다.

기업 어음 디렉토리로 이동하여 네트워크를 시작하는 스크립트를 실행하자.
```text
cd commercial-paper
./network-starter.sh
```

스크립트 작동 후에는 `docker ps` 명령으로 실행 중인 패브릭 노드를 확인해보자.
```text
docker ps
```

생성된 컨테이너는 fabric_test라는 Docker 네트워크를 형성한다.
`docker network` 명령으로 확인해보자.
```text
docker network inspect fabric_test
```

같은 하나의 Docker 네트워크이지만 각 컨테이너는 다른 IP 주소를 사용한다.

이는 Digibank와 MagnetoCorp로 테스트 네트워크를 운영하기 때문에
peer0.org1.example.com은 DigiBank 조직, peer0.org2.example.com은 MagnetoCorp 조직에 속한다.
테스트 네트워크가 실행되고 있으므로 이 시점부터 테스트 네트워크를 PaperNet이라고 하자.
이제 어음 발행 및 거래를 원하는 MagnetoCorp 역할을 수행해보자.

## MagnetoCorp로 네트워크 모니터링

기업 어음 튜토리얼은 DigiBank와 MagnetoCorp로 별도 폴더를 제공하여 두 조직으로 동작하도록 한다.
두 폴더에는 각각의 스마트 컨트랙트와 어플리케이션 파일을 포함한다.
이는 두 조직은 기업 어음 거래에서 서로 다른 역할을 수행하기 때문이다.
`fabric-samples` 레포지토리를 새 창에서 열고 MagnetoCorp 디텍토리로 이동한다.
```text
cd organization/magnetocorp
```

처음으로 하려는 것은 MagnetoCorp로 PaperNet의 컴포넌트를 모니터링하는 것이다.
관리자는 `logspout` 도구를 사용하여 여러 도커 컨테이너에서 나오는 출력을 볼 수 있다.
도구로 다른 출력 스트림을 한 곳으로 모아 한 화면에서 쉽게 어떤 일들이 벌어지는지 확인할 수 있다.
관리자가 스마트 컨트랙트를 설치하거나 개발자가 스마트 컨트랙트를 호출할 때 매우 도움이 된다.

MagnetoCorp 디렉토리에서 아래 명령을 실행하여 `monitordocker.sh` 스크립트를 실행하고 
`fabric_test`에서 실행되는 PaperNet과 관련된 컨테이너에 대한 `logspout` 도구를 시작한다.
```text
(magnetocorp admin)
./configuration/cli/monitordocker.sh fabric_test
```

`monitordocker.sh`에 기본으로 정의된 포트가 사용 중일 경우 포트 번호를 명시할 수 있다.
```text
(magnetocorp admin)
./monitordocker.sh fabric_test <port_number>
```

이 창에서 남은 튜토리얼 동안 도커 컨테이너에서 나오는 출력을 확인할 수 있다.
다른 창을 열고 계속해서 명령을 입력하자.
이제부터는 MagnetoCorp가 기업 어음을 발행할 때 사용하는 스마트 컨트랙트를 살펴보자.


## 기업 어음 스마트 컨트랙트 분석

`issue`, `buy`, `redeem` 이 세 함수가 기업 어음 스마트 컨트랙트의 핵심이다.
이 함수들은 어플리케이션에서 원장에 기업 어음을 발행, 구매, 상환하는 트랜잭션을 제출하는데 사용된다.
이 스마트 컨트랙트를 조사해 보자.

터미널을 새로 열고 `fabric-samples` 디렉토리로 이동하여 MagnetoCorp 개발자 역할을 수행하자.
```text
cd /mnt/c/Users/fabi8/go/src/github.com/LeeSeungYun1020/fabric-samples
cd commercial-paper/organization/magnetocorp
```
`contract` 디렉토리에서 코드를 확인할 수 있다.
visual studio code가 설치되어 있지 않다면
[여기](https://docs.microsoft.com/ko-kr/windows/wsl/tutorials/wsl-vscode)에서 설치 방법을 확인할 수 있다.
```text
(magnetocorp developer)
code contract
```

폴더의 lib 디렉토리에서 `papercontract.js` 파일을 확인할 수 있다.
이 파일에 기업 어음 스마트 컨트랙트가 포함되어 있다.
![visual studio code 작동 화면](img/commercial_paper.diagram.code.png)

`papercontract.js`는 Node.js 환경에서 동작하는 자바스크립트 프로그램이다.
중요한 부분을 잠깐 살펴보면
* `const { Contract, Context } = require('fabric-contract-api');`<br>
스마트 컨트랙트에서 광범위하게 사용될 2가지 주요 하이퍼레저 패브릭 클래스를 가져온다.
* `class CommercialPaperContract extends Contract`<br>
기존 패브릭 `Contract` 클래스를 상속받아 새 스마트 컨트랙트 클래스를 정의한다.
이 클래스에서 기업 어음을 발행(issue), 구매(buy), 상환(redeem)하는 주요 트랜잭션이 정의된다.
* `async issue(ctx, issuer, paperNumber, issueDateTime, maturityDateTime, faceValue)`
이 메소드는 PaperNet에 대한 기업 어음 발행 트랜잭션을 정의한다.
파라미터로 전달된 값을 이용하여 새로운 어음을 발행한다.
* `let paper = CommercialPaper.createInstance(issuer, paperNumber, issueDateTime, maturityDateTime, parseInt(faceValue));`
`issue` 트랜잭션 내에서 이 statement는 `CommercialPaper` 클래스를 이용하여 새 기업 어음을 메모리에 생성한다.
`buy`와 `redeem` 메소드에서도 `CommericalPaper` 클래스를 유사하게 활용한다.
* `await ctx.paperList.addPaper(paper);`
`ctx.paperList`를 사용해 새 기업 어음을 원장에 추가한다.
`ctx.paperList`는 스마트 컨트랙트 컨텍스트인 `CommercialPaperContext`가 초기화될 떄 만들어진 `PaperList` 클래스의 인스턴스이다.
  `buy`와 `redeem` 메소드에서도 `ctx.paperList`를 유사하게 활용한다.
* `return paper;`
트랜잭션 caller의 처리를 위해 `issue` 트랜잭션에 대한 응답으로 바이너리 버퍼를 반환한다.

스마트 계약이 어떻게 작동하는지 이해하기 위해 `contract` 디렉토리의 다른 파일을 자유롭게 검토해보자.

## 채널에 스마트 컨트랙트 배포

어플리케이션에서 `papercontract`가 호출되기 전에 테스트 네트워크의 적절한 피어 노드에 설치되고
패브릭 체인코드 생명주기를 이용하여 채널에 정의한다.
패브릭 체인코드 생명주기를 통해 여러 조직은 체인코드가 채널에 배포되기 전에 체인코드의 파라미터에 동의할 수 있다.
결과적으로 우리는 MagnetoCorp와 DigiBank 관리자로 체인코드를 설치하고 승인해야 한다.

스마트 컨트랙트에 어플리케이션 개발의 초점이 맞추어져 있으며
스마트 컨트랙트는 체인코드라 불리는 하이퍼 패브릭 아티팩트에 포함되어 있다.
하나 이상의 스마트 컨트랙트가 단일 체인 코드 내에 정의될 수 있으며
체인코드를 설치하면 PaperNet의 다른 조직에서 사용할 수 있게 된다.
정리하자면 관리자만 체인코드에 대해 걱정할 필요가 있다는 것을 의미한다.
다른 모든 사용자는 스마트 컨트랙트의 관점에서 생각할 수 있다.

### MagnetoCorp에서 스마트 컨트랙트 설치 및 승인

먼저 MagnetoCorp 관리자로 스마트 컨트랙트를 설치하고 승인해보자.
magnetocorp 폴더에서 명령을 실행해야 한다.
```text
cd commercial-paper/organization/magnetocorp
```

MagnetoCorp 관리자는 peer CLI를 통해 PaperNet과 상호작용할 수 있다.
관리자는 특정 환경 변수를 명령 창에서 설정해야 한다.
환경 변수를 설정해야 관리자가 올바른 피어 바이너리 세트를 사용하며
MagnetoCorp 피어의 주소로 명령을 보내며, 올바른 암호화 자료로 요청에 서명할 수 있다.

스크립트 파일을 이용하여 환경 변수를 자동으로 설정한다.
```text
(magnetocorp admin으로 만드려는 명령 창에서)
source magnetocorp.sh
```
스크립트를 실행하면 창에 설정한 환경 변수 목록이 표시된다.
이 명령 창에서 MagnetoCorp 관리자로 PaperNet과 상호작용할 수 있게 되었다.

첫번째 단께는 `papercontract` 스마트 컨트랙트를 설치하는 것이다.
스마트 컨트랙트는 `peer lifecycle chaincode package` 명령을 사용하여 체인코드로 패키징할 수 있다.
MagnetoCorp 관리자 명령 창에서 아래 명령을 실행하여 체인코드 패키지를 만들어보자.
```text
(magnetocorp admin)
peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```
이제 MagnetoCorp 관리자는 `peer lifecycle chaincode install` 명령으로 MagnetoCorp의 피어에 체인코드를 설치할 수 있다.

```text
(magnetocorp admin)
peer lifecycle chaincode install cp.tar.gz
```

```text
(result)
2022-01-11 17:37:48.546 KST 0001 INFO [cli.lifecycle.chaincode] submitInstallProposal -> Installed remotely: response:<status:200 payload:"\nEcp_0:df23cfaa2d118a48df5002d95521e14df1070e44d7858da6d2da2d9df3f138ce\022\004cp_0" >
2022-01-11 17:37:48.554 KST 0002 INFO [cli.lifecycle.chaincode] submitInstallProposal -> Chaincode code package identifier: cp_0:df23cfaa2d118a48df5002d95521e14df1070e44d7858da6d2da2d9df3f138ce
```

앞서 스크립트 실행으로
MagnetoCorp 관리자가 `CORE_PEER_ADDRESS="localhost:9051"`로 `peer0.org2.example.com`를 목표로 하기 때문에
`INFO 001...`은 `papercontract`이 해당 피어에 성공적으로 설치되었음을 나타낸다.

스마트 컨트랙트 설치 후에는 MagnetoCorp로 `papercontract` 체인코드 정의를 승인해야 한다.
첫 단계는 피어에 설치한 체인코드의 패키지 ID를 찾아야 한다.
`peer lifecycle chaincode queryinstalled` 명령을 통해 패키지 ID를 쿼리할 수 있다.
```text
peer lifecycle chaincode queryinstalled
```

이 명령은 설치 명령과 동일한 패키지 ID를 반환한다. 결과는 아래와 같다.

```text
(result)
Installed chaincodes on peer:
Package ID: cp_0:df23cfaa2d118a48df5002d95521e14df1070e44d7858da6d2da2d9df3f138ce, Label: cp_0
```

다음 단계에서 패키지 ID가 필요하므로 환경 변수로 저장한다.
```text
export PACKAGE_ID=cp_0:df23cfaa2d118a48df5002d95521e14df1070e44d7858da6d2da2d9df3f138ce
```

`peer lifecycle chaincode approveformyorg` 명령으로 MagnetoCorp 관리자는 체인코드 정의를 승인할 수 있다.
```text
(magnetocorp admin)
peer lifecycle chaincode approveformyorg --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name papercontract -v 0 --package-id $PACKAGE_ID --sequence 1 --tls --cafile $ORDERER_CA
```

체인코드 보증 정책은 채널 멤버가 체인코드 정의를 사용하기 위해 동의해야 하는 가장 중요한 체인코드 매개변수 중 하나이다.
보증 정책은 유효한 트랜잭션으로 결정되기 전에 트랜잭션을 보증(실행, 서명)해야 하는 조직 집합을 설명한다.
`papercontract` 체인코드를 `--policy` 플래그 없이 승인하였으므로 MagnetoCorp 관리자는
채널의 기본 보증 정책을 사용하게 된다.
채널의 기본 보증 정책은 과반수가 넘는 조직이 트랜잭션을 보증이 필요하다.
모든 트랙잭션은 유효하든 유효하지 않든 간에 레저 블록체인에 기록되나
유효한 트랜잭션만이 world state(원장의 현재 상태를 나타냄)를 업데이트한다.


### DigiBank에서 스마트 컨트랙트 설치 및 승인

`mychannel`, `LifecycleEndorsement` 정책에 기반하여, 패브릭 체인코드 생명주기는 체인코드가 채널에 커밋되기 전에
체인코드 정의에 조직의 과반수 이상이 동의해야 한다.
다시말해 과반수를 넘기 위해 `paperNet` 체인코드를 MagnetoCorp와 DigiBank 모두에서 승인해야 한다.
`fabric-samples` 폴더에서 새 터미널 창을 열고 DigiBank 스마트 컨트랙트와 어플리케이션 폴더로 이동한다.
```text
(digibank admin으로 만드려는 명령 창에서)
cd commercial-paper/organization/digibank/
```

DigiBank 폴더의 스크립트를 이용하여 환경 변수를 설정하여 DigiBank 관리자 역할을 수행할 수 있다.
```text
(digibank admin으로 만드려는 명령 창에서)
source digibank.sh
```

이제 papercontract를 DigiBank에 설치하고 승인할 수 있다.
체인코드 패키징을 위해 아래 명령을 수행해보자.
```text
(digibank admin)
peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```

관리자는 DigiBank 피어에 체인코드를 설치할 수 있다.
```text
(digibank admin)
peer lifecycle chaincode install cp.tar.gz
```

방금 설치된 체인코드의 패키지 ID를 쿼리하여 저장한다.
```text
(digibank admin)
peer lifecycle chaincode queryinstalled
```

환경 변수로 패키지 ID를 저장하자.
```text
export PACKAGE_ID=cp_0:f2316bf21cdc9aa6c3650c815228f88b3ae20cd2a74f30a1db76f02be2c4d599
```

DigiBank 관리자는 `papercontract` 체인코드 정의를 승인할 수 있다.
```text
(digibank admin)
peer lifecycle chaincode approveformyorg --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name papercontract -v 0 --package-id $PACKAGE_ID --sequence 1 --tls --cafile $ORDERER_CA
```

### 채널에 체인코드 정의 커밋
DigiBank와 MagnetoCorp 모두 `papernet` 체인코드를 승인하였으므로 체인코드 정의를 채널에 커밋하기 위해
필요한 과반수 이상 조건을 만족하였다.
한 번 체인코드가 채널에 성공적으로 정의되면 `papercontract` 체인코드 안의 `CommercialPaper` 스마트 컨트랙트가
채널의 클라이언트 어플리케이션에서 호출될 수 있다.
DigiBank와 MagnetoCorp 모두에서 채널에 체인코드를 커밋할 수 있으므로
계속해서 DigiBank 관리자로 커밋을 진행한다.

DigiBank 관리자가 `papercontract` 체인코드 정의를 채널에 커밋하면
양 PaperNet 피어에 `papercontact`를 실행하는 새로운 도커 체인코드 컨테이너가 생성된다.

DigiBank 관리자는 `peer lifecycle chaincode commit` 명령을 이용하여 `mychannel`에
`papercontract` 체인코드 정의를 커밋할 수 있다.
```text
(digibank admin)
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} --channelID mychannel --name papercontract -v 0 --sequence 1 --tls --cafile $ORDERER_CA --waitForEvent
```

체인코드 컨테이너는 체인코드 정의가 채널에 정의된 후에 시작될 것이다.
`docker ps` 명령을 사용하여 실제로 `papercontract` 컨테이너에서 두 피어가 시작되었는지 확인해보자.

```text
(digibank admin)
docker ps
```

컨테이너 이름에는 시작한 피어와 `papercontract` 버젼 0을 실행하고 있다는 것이 나타난다.

지금까지 `papercontract` 체인코드를 채널에 배포하였다.
이제부터는 MagnetoCorp 어플리케이션을 사용하여 상업 어음을 발행할 수 있다.
그 전에 잠깐 응용 프로그램 구조를 살펴보자.

## 어플리케이션 구조

`papercontract`에 포함된 스마트 컨트랙트는 MagnetoCorp 어플리케이션의 issue.js에서 호출된다.
Isabella는 어플리케이션을 사용하여 상업 어음 `00001`을 발행하는 트랜잭션을 원장에 제출한다.
빠르게 `issue` 어플리케이션이 어떻게 작동하는지 살펴보자.
![issue application](img/commercial_paper.diagram_issue.png)
게이트웨이는 어플리케이션이 트랜잭션 생성, 제출, 응답에만 포커스를 맞출 수 있도록 한다.
이는 서로 다른 네트워크 구성요소 간의 트랜잭션 제안, 오더링, 알림 처리를 조정한다.

`issue` 어플리케이션이 Isabella 대신 트랜잭션을 제출하므로
첫번째 단계로 Isabella의 X.509 인증서를 지갑에서 검색한다.
이 인증서는 로컬 파일 시스템이나 하드웨어 보안 모듈(HSM)에 저장되어 있을 수 있다.
그 다음으로 `issue` 어플리케이션은 게이트웨이를 이용하여 채널에 트랜잭션을 제출할 수 있다.
하이퍼레저 패브릭 SDK는 게이트웨이 추상화를 제공한다.
게이트웨이 추상화로 게이트웨이에 네트워크 상호작용을 위임하여 어플리케이션은 로직에만 집중할 수 있다.
게이트웨이와 지갑을 사용하여 하이퍼레저 패브릭 어플리케이션을 간단하게 작성할 수 있다.

Isabella가 사용하려고 하는 `issue` 어플리케이션을 살펴보자.
별도로 터미널 창을 열고 MagnetoCorp의 `/application` 폴더로 이동하자.
```text
(isabella)
cd commercial-paper/organization/magnetocorp/application/
ls
(result)
addToWallet.js  cpListener.js  enrollUser.js  issue.js  package.json  transfer.js
```
`addToWallet.js`는 Isabella가 자신의 ID를 지갑에 로드할 때 사용할 프로그램이다.
`issue.js`는 이 ID를 사용하여 `papercontract`를 호출하여 MagnetoCorp 대신 상업 어음 `00001`을 생성한다.

MagnetoCorp의 어플리케이션 `issue.js` 사본이 있는 디렉토리로 가서 코드 편집기(vs code)를 사용하여 코드를 살펴보자.
```text
(isabella)
(앞서 이동하지 않은 경우)
cd commercial-paper/organization/magnetocorp/application
code issue.js
```

디렉토리에는 `issue` 어플리케이션과 디펜던시가 존재한다.
![issue application code](img/commercial_paper.diagram.issue_code.png)

프로그램의 주요 부분을 따라가보자.
* `const { Wallets, Gateway } = require('fabric-network');` <br>
하이퍼레저 패브릭 SDK 클래스 - `Wallets`와 `Gateway`를 불러온다.
* `const wallet = await Wallets.newFileSystemWallet('../identity/user/isabella/wallet');`
어플리케이션이 `isabella` 지갑을 사용하여 블록체인 네트워크 채널에 연결할 것임을 명시한다.
Isabella의 X.509 인증서는 로컬 파일 시스템에 있기 때문에
어플리케이션은 새로운 `FileSystemWallet`을 생성한다.
어플리케이션은 `isabella` 지갑 내에서 특정 ID를 선택한다.
* `await gateway.connect(connectionProfile, connectionOptions);`
`ConnectionOptions`에서 참조하는 ID를 이용하여
`connectionProfile`로 식별된 게이트웨이로 네트워크에 연결한다.
`../gateway/networkConnection.yaml`과 `User1@Org1.example.com`에서 이 값을 각각 어떻게 사용하는지 확인할 수 있다.
* `const network = await gateway.getNetwork('mychannel');`
어플리케이션을 `mychannel` 네트워크에 연결한다. `mychannel` 네트워크는 이전에 `papercontract`를 배포했던 채널이다.
* `const contract = await network.getContract('papercontract');`
어플리케이션이 `papercontract` 체인코드에 접근할 수 있도록 한다.
한 번 어플리케이션이 getContract를 발행하면 체인코드 내에서 구현된 모든 스마트 컨트랙트 트랜잭션을 제출할 수 있다.
* `const issueResponse = await contract.submitTransaction('issue', 'MagnetoCorp', '00001', ...);`
스마트 컨트랙트 내에 정의한 `issue` 트랜잭션으로 네트워크에 트랜잭션을 제출한다.
`MagnetoCorp`, `00001`과 같은 파라미터는 'issue' 트랜잭션이 새로운 상업 어음을 생성하는데 필요한 값이다.
* `let paper = CommercialPaper.fromBuffer(issueResponse);`
`issue` 트랜잭션에 대한 응답을 처리한다.
어플리케이션에서 바르게 해석될 수 있도록 응답은 버퍼에서 `CommercoalPaper` 객체인 `paper`로 타입 변환되어야 한다.

`issue.js`가 어떻게 작동하는지 알기 위해 자유롭게 `/application` 디렉토리의 다른 파일을 살펴봐도 좋다.

## 어플리케이션 디펜던시

`issue.js` 어플리케이션은 JavaScript로 작성되었으며 Node.js 환경에서 작동하도록 설계된
PaperNet 네트워크 클라이언트 어플리케이션이다.

> npm과 node 패키지에 대해 알고 있다면 바로 `npm install` 명령을 실행하고 다음 문단으로 넘어가자.

MagnetoCorp 어플리케이션은 여러 외부 node 패키지를 사용한다.
`issue.js`는 YAML 게이트웨이 연결 프로필을 처리하기 위한 `js-yaml` 패키지,
`Gateway`와 `Wallet` 클래스에 접근하기 위한 `fabric-network` 패키지를 포함한다.
```javascript
const yaml = require('js-yaml');
const { Wallets, Gateway } = require('fabric-network');
```

`npm install` 명령으로 npm으로 로컬 파일 시스템에 패키지를 다운로드할 수 있다.
런타임에 사용하기 위해 (관행적으로) 어플리케이션 관련 패키지는 `/node_modules` 디렉토리 내에 위치해야 한다.

`package.json` 파일에서 다운로드할 패키지와 버젼을 확인할 수 있다.
이제 `npm install` 명령으로 필요한 패키지를 설치하자.
```text
(isabella)
npm install
ls
```

`node_modules` 디렉토리에서 설치된 패키지를 확인할 수 있다.
`js-yaml`과 `fabric-network`도 다른 npm 패키지를 사용하여 만들어졌기 때문에 많은 수의 패키지가 설치된다.
`package-lock.json` 파일에 설치된 정확한 버젼을 식별하므로
환경을 정확하게 재현(테스트, 문제 진단, 어플리케이션 제공)할 때 유용하다.

## 지갑

Isabella가 `issue.js`를 실행하여 MagnetoCorp 상업 어음 `00001`을 발행할 준비가 거의 완료되었다.
수행할 작업이 하나가 남아 있는데
`issue.js`는 Isabella 즉 MagnetoCorp로 행동하므로 이러한 사실을 반영하는 지갑의 ID를 사용한다.
따라서 Isabella의 지갑에 적절한 X.509 자격 증명을 생성하는 일회성 활동을 수행해야 한다.

네트워크를 배포할 때 인증 기관으로 등록한 어플리케이션 사용자가 있다.
MagnetoCorp의 인증 기관은 PaperNet의 ca_org2이다.
Isabella는 ID 이름과 비밀번호로 `issue.js` 어플리케이션에 대한 X.509 암호화 자료를 생성할 수 있다.
CA를 사용하여 클라이언트 측 암호화 자료를 생성하는 과정을 등록(enrollment)이라고 한다.
실사용시 네트워크 운영자가 CA에 등록된 클라이언트 ID와 비밀번호를 개발자에게 제공할 것이다.
개발자는 해당 자격증명을 통해 어플리케이션에 등록하고 네트워크와 상호작용해야 한다.

`enrollUser.js` 프로그램은 `fabric-ca-client` 클래스를 사용하여 공용키와 비밀키 쌍을 만들고
CA에 인증서 서명 요청(Certificate Signing Request)을 발행한다.
Isabella가 제출한 ID 이름과 비밀번호가 CA에 등록된 자격 증명과 일치하는 경우 CA는 공개키를 인코딩하는 인증서를 발습하고
서명하여 Isabella가 MagnetoCorp에 속해 있음을 확인해준다.
서명 요청이 완료되면, `enrollUser.js`는 비밀키와 서명된 인증서를 Isabella의 지갑에 저장한다.
Node SDK가 `fabric-ca-client` 클래스를 사용하여 작업을 완료하는 방법을 자세히 알아보려면
`enrollUser.js` 파일을 검토해보자.

Isabella의 터미널 창에서 `enrollUser.js` 프로그램을 실행하여 신원 정보를 지갑에 저장하자.
```text
(isabella)
node enrollUser.js
(result)
Wallet path: /mnt/c/Users/fabi8/go/src/github.com/LeeSeungYun1020/fabric-samples/commercial-paper/organization/magnetocorp/identity/user/isabella/wallet
Successfully enrolled client user "isabella" and imported it into the wallet
```

프로그램 실행 결과 생성된 PaperNet에 트랜잭션을 제출하는데 사용될 지갑의 내용을 살펴보자.
```text
(isabella)
ls ../identity/user/isabella/wallet/
(result)
isabella.id
```

Isabella는 여러 개의 신원을 지갑에 저장할 수 있다. (현재 예제에서는 하나만 사용한다.)
`wallet` 폴더에 Isabella가 네트워크에 연결하는데 필요한 정보가 저장된 `isabella.id` 파일이 있다.
Isabella가 사용하는 다른 신원이 있다면 별도의 파일로 저장될 것이다.
파일을 열어 JSON 형태로 저장된 ID 정보를 확인할 수 있다.
```text
(isabella)
cat ../identity/user/isabella/wallet/isabella.id
(result)
{
    "credentials":{
        "certificate":"-----BEGIN CERTIFICATE-----\n ... -----END CERTIFICATE-----\n",
        "privateKey":"-----BEGIN PRIVATE KEY-----\r\n ... -----END PRIVATE KEY-----\r\n"
    },
    "mspId":"Org2MSP",
    "type":"X.509",
    "version":1
}
```

파일에서 알 수 있는 것은
* `privateKey`는 Isabella로 트랜잭션에 서명할 때 사용된다. 단 Isabella의 즉각적인 통제 범위를 벗어나 배포되지는 않는다.
* `certificate`는 인증서 생성시 인증 기관에서 추가한 Isabella의 공용키와 X.509 속성이 포함되어 있다.
이 인증서는 네트워크에 배포되어 다른 행위자가 다른 시간에 Isabella의 개인키로 생성된 정보를 암호학적으로 검증할 수 있다.

인증서 파일에는 Isabella의 조직(Organization)과 역할 패브릭과 관련된 메타 데이터가 포함될 수 있다.

## Issue(발행) 어플리케이션

Isabella는 `issue.js`를 사용하여 MagnetoCorp 상업 어음 `00001`을 발행하는 트랜잭션을 제출할 수 있다.
```text
(isabella)
node issue.js
(result)
Connect to Fabric gateway.
Use network channel: mychannel.
Use org.papernet.commercialpaper smart contract.
Submit commercial paper issue transaction.
Process issue transaction response.
{
    "class":"org.papernet.commercialpaper",
    "currentState":1,
    "issuer":"MagnetoCorp",
    "paperNumber":"00001",
    "issueDateTime":"2020-05-31",
    "maturityDateTime":"2020-11-30",
    "faceValue":5000000,
    "mspid":"Org2MSP",
    "owner":"MagnetoCorp"
}
MagnetoCorp commercial paper : 00001 successfully issued for value 5000000
Transaction complete.
Disconnect from Fabric gateway.
Issue program complete.
```
프로그램 실행 결과, MagnetoCorp 상업 어음 00001이 액면가 5M USD로 발행된 것을 확인할 수 있다.
앞서 본 것처럼 이를 달성하기 위해 어플리케이션은 `papercontract.js`에서 `CommercialPaper` 스마트 컨트랙트에 정의된 `issue` 트랜잭션을 호출한다.
스마트 컨트랙트는 패브릭 API를 통해 원장과 상호작용한다.
특히 `putState()`와 `getState()`는 새로운 상업 어음을 world state 내에 벡터 상태로 나타낸다.
이러한 벡터 상태가 스마트 컨트랙트 내에서 정의된 `buy`와 `redeem` 트랜잭션에 의해 어떻게 조작될 수 있을지 살펴보자.

패브릭 SDK는 항상 트랜잭션 보증, 오더링, 알림 프로세스를 처리하여 어플리케이션 로직을 간단하게 만든다.
SDK는 게이트웨이를 사용하여 네트워크 상세 구현을 추상화하고
`connectionOptions`를 사용하여 트랜잭션 재시도 같은 고급 처리 전략을 선언할 수 있다.

이제부터는 DigiBank 어플리케이션을 사용하여 상업 어음을 구매할 DigiBank 직원 Balaji로 전환하여
MagnetoCorp `00001`의 수명주기를 계속 따라가 보자.

## DigiBank 어플리케이션

Balaji는 DigiBank의 `buy` 어플리케이션을 사용하여 MagnetoCorp에서 DigiBank로
상업 어음 00001의 소유권을 이전하는 트랜잭션을 원장에 제출한다.
`CommerialPaper` 스마트 컨트랙트는 MagnetoCorp 어플리케이션에서 사용한 것과 동일하다.
그러나 트랜잭션은 동일하지 않다. - `issue`가 아니라 `buy`가 사용될 것이다.
그럼 DigiBank 어플리케이션이 어떻게 동작하는지 살펴보자.

Balaji를 위한 별도의 터미널 창을 열자.
`fabric-samples`를 열고 `buy.js` 어플리케이션이 저장된 DigiBank 어플리케이션 디렉토리로 이동하여
코드 에디터로 열자.
```text
(balaji)
cd commercial-paper/organization/digibank/application/
code buy.js
```
해당 디렉토리에는 `buy.js` 뿐만 아니라 `redeem.js` 어플리케이션도 있다.
Balaji는 `buy`와 `redeem` 어플리케이션을 사용한다.

Digibank의 `buy.js` 어플리케이션은 MagnetoCorp의 `issue.js`와 몹시 비슷하나 2가지 중요한 차이점이 존재한다.
* 신원: 사용자가 MagnetoCorp의 `Isabella`가 아니라 DigiBank의 `Balaji`이다.
```javascript
const wallet = await Wallets.newFileSystemWallet('../identity/user/balaji/wallet');
```
PaperNet 네트워크 채널에 연결할 때 어플리케이션이 어떻게 `balaji` 지갑을 사용하는지 살펴보자.
`buy.js`는 `balaji` 지갑에서 특정 ID를 선택한다.

* 트랜잭션: 호출되는 트랜잭션이 `issue`가 아닌 `buy`이다.
```javascript
const buyResponse = await contract.submitTransaction('buy', 'MagnetoCorp', '00001', ...);
```
`buy` 트랜잭션은 `MagnetoCorp`, `00001`, ... 같은 값을 함께 전송하는데
이는 `CommercialPaper` 스마트 컨트랙트 클래스에서 상업 어음 `00001`의 소유권을 DigiBank 변경하는데 사용할 값이다.

자유롭게 `application` 디렉토리를 검토하여 어플리케이션 작동 방법을 이해해 보자.

## DigiBank로 실행

DigiBank 어플리케이션은 상업 어음을 구매한고 반환한다.
구매와 반환은 MagnetoCorp의 발행 어플리케이션과 매우 유사한 구조를 가진다.
마찬가지로 `npm install` 명령으로 필요한 어플리케이션 패키지를 설치하자.
```text
(digibank admin)
cd application
또는 cd commercial-paper/organization/digibank/application/
npm install
```

Balaji 명령 창에서 `enrollUser.js` 프로그램을 실행하여 지갑에 인증서와 비밀키를 생성하자.
```text
(balaji)
node enrollUser.js
(result)
Wallet path: /mnt/c/Users/fabi8/go/src/github.com/LeeSeungYun1020/fabric-samples/commercial-paper/organization/digibank/identity/user/balaji/wallet
Successfully enrolled client user "balaji" and imported it into the wallet
```
`addToWallet.js` 프로그램은 `balaji`의 신원 정보를 `balaji`의 지갑에 추가한다.
`balaji`의 지갑은 `buy.js`와 `redeem.js`에서 `PaperNet`으로 트랜잭션을 제출할 때 사용된다.

Isabella처럼 Balaji도 여러 개의 ID 정보를 저장할 수 있다. (이번 예제에서는 하나의 ID만 사용한다.)
해당 ID 파일은 `digibank/identity/user/balaji/wallet/balaji.id`로 저장된다.

## Buy(구매) 어플리케이션

Balaji는 이제 `buy.js`로 트랜잭션을 전송하여 MagnetoCorp의 기업 어음 00001을 DigiBank로 소유권을 이전할 수 있다.

Balaji 창에서 `buy` 어플리케이션을 실행해보자.
```text
(balaji)
node buy.js
```
프로그램 결과 출력을 통해 MagnetoCorp 기업 어음 00001이 성공적으로 구매된 것을 확인할 수 있다.
`buy.js`는 `CommercialPaper` 스마트 컨트랙트에 정의되어 있는 `buy` 트랜잭션을 호출한다.
`CommercialPape` 스마트 컨트랙트는 `putState()`와 `getState()` 패브릭 API를 사용하여 world state 내의 기업 업음 00001을 업데이트 한다.
살펴본 바와 같이 기업 어음 구매와 발행 어플리케이션 로직은 스마트 컨트랙트 로직과 매우 유사하다.


## Redeem(상환) 어플리케이션

기업 어음 00001의 수명 주기에서 최종 트랜잭션은 DigiBank가 MagnetoCorp로 이를 상환하는 것이다.
Balaji는 `redeem.js`를 사용하여 스마트 컨트랙트 내에서 상환 로직을 수행하기 위해 트랜잭션을 제출한다.

```text
(balaji)
node redeem.js
```

`redeem.js`가 `CommercialPaper`에 정의된 `redeem` 트랜잭션을 호출하여 기업 어음 00001을 성공적으로 상환하였음을 확인할 수 있다.
소유권이 어음 발행자인 MagnetoCorp에 반환되었음을 반영하기 위해 world state 내 기업 어음 00001을 업데이트 한다.

## 정리하기

기업 어음 튜토리얼이 완료되었다. 스크립트를 사용하여 환경을 정리할 수 있다.
기업 어음 예제 디렉토리로 이동하여 스크립트를 실행하자.
```text
cd fabric-samples/commercial-paper
./network-clean.sh
```

이 명령은 네트워크의 피어와 couch DB 컨테이너, 오더링 노드, logspout 도구를 중단한다.
Isabella와 Balaji에 대해 생성한 ID도 제거된다.
원장의 데이터 또한 모두 삭제됨을 주의해야 한다.
튜토리얼을 다시 진행하려면 위 명령으로 깨끗하게 초기화한 상태에서 진행해야 한다.

## 참고 자료
[1] Hyperledger, 12 06 2021. [온라인]. Available: https://hyperledger-fabric.readthedocs.io/en/latest/tutorial/commercial_paper.html. [액세스: 10 01 2022]. 