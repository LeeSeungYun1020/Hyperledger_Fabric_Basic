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


## 상업 어음 스마트 컨트랙트 분석

`issue`, `buy`, `redeem` 이 세 함수가 상업 어음 스마트 컨트랙트의 핵심이다.
이 함수들은 어플리케이션에서 원장에 상업 어음을 발행, 구매, 상환하는 트랜잭션을 제출하는데 사용된다.
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
이 파일에 상업 어음 스마트 컨트랙트가 포함되어 있다.
![visual studio code 작동 화면](img/commercial_paper.diagram.code.png)

`papercontract.js`는 Node.js 환경에서 동작하는 자바스크립트 프로그램이다.
중요한 부분을 잠깐 살펴보면
* `const { Contract, Context } = require('fabric-contract-api');`<br>
스마트 컨트랙트에서 광범위하게 사용될 2가지 주요 하이퍼레저 패브릭 클래스를 가져온다.
* `class CommercialPaperContract extends Contract`<br>
기존 패브릭 `Contract` 클래스를 상속받아 새 스마트 컨트랙트 클래스를 정의한다.
이 클래스에서 상업 어음을 발행(issue), 구매(buy), 상환(redeem)하는 주요 트랜잭션이 정의된다.
* `async issue(ctx, issuer, paperNumber, issueDateTime, maturityDateTime, faceValue)`
이 메소드는 PaperNet에 대한 상업 어음 발행 트랜잭션을 정의한다.
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


### DigiBank에서 스마트 컨트랙트 설치 및 승인


### 채널에 체인코드 정의 커밋


## 어플리케이션 구조


## 어플리케이션 디펜던시


## 지갑


## 발행(Issue) 어플리케이션


## DigiBank 어플리케이션


## DigiBank로 실행


## 구매(Buy) 어플리케이션


## 상환(Redeem) 어플리케이션


## 정리하기


## 참고 자료
[1] Hyperledger, 12 06 2021. [온라인]. Available: https://hyperledger-fabric.readthedocs.io/en/latest/tutorial/commercial_paper.html. [액세스: 10 01 2022].