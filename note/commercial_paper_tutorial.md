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



## 참고 자료
[1] Hyperledger, 12 06 2021. [온라인]. Available: https://hyperledger-fabric.readthedocs.io/en/latest/tutorial/commercial_paper.html. [액세스: 10 01 2022].