# Test network 실행

## 설치 전 필요한 프로그램

### Git
64비트 버젼 Git 설치
```git
git config --global core.autocrlf false
git config --global core.longpaths true
```

### cURL
최신 버젼의 [cURL](https://curl.haxx.se/download.html) 설치

### Docker
최신 버젼의 [Docker](https://docs.docker.com/get-docker/) 설치  
wsl2 integration 설정 필요  
[wsl2 설정 변경](https://docs.microsoft.com/ko-kr/windows/wsl/install#upgrade-version-from-wsl-1-to-wsl-2)

### Go
최신 버젼의 [Go](https://golang.org/doc/install) 설치

### JQ
최신 버젼의 [jq](https://stedolan.github.io/jq/download/) 설치

## Fabric과 Fabric sample 설치

### 설치 경로
```text
C:\Users\fabi8\go\src\github.com\LeeSeungYun1020
```
### 다운로드
최신 버젼의 Fabric sample과 docker image, binary 다운로드
```text
cd C:\Users\fabi8\go\src\github.com\LeeSeungYun1020
curl -sSL https://bit.ly/2ysbOFE | bash -s
```
wsl2의 경우
```text
cd /mnt/c/Users/fabi8/go/src/github.com/LeeSeungYun1020
curl -sSL https://bit.ly/2ysbOFE | bash -s
```

## Fabric test 네트워크 사용

### 테스트 네트워크 가져오기
```text
cd fabric-samples/test-network
./network.sh down
./network.sh up
```
두 개의 피어 노드와 오더링 노드로 구성된 패브릭 네트워크를 만든다.
```text
docker ps -a
```
위 명령어로 도커에서 실행 중인 컨테이너를 확인할 수 있다.

### 채널 생성
```text
./network.sh createChannel
```
mychannel이라는 이름을 가진 채널이 생성된다.  
이름을 지정하기 위해서는 c 옵션을 통해 이름을 파라미터로 전달해야한다.
```text
./network.sh createChannel -c channel1
```

### 채널에서 체인코드 실행
```text
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
```
wsl2에서 실행할 경우 go language를 ubuntu에 [설치](https://go.dev/doc/install) 후 path를 추가해야한다.
```text
export PATH=$PATH:/usr/local/go/bin
```

### 네트워크와 상호작용
test 네트워크를 가져온 뒤에는 CLI를 통해 네트워크와 상호작용할 수 있다.  
peer CLI로 배포된 스마트 계약을 호출하거나 채널을 업데이트하거나 CLI에서 새 스마트 계약을 설치 및 배포할 수 있다.  
test-network 디렉토리에서 아래 코드를 입력하여 환경변수에 추가한다.
```text
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051
```

체인코드의 InitLedger 함수를 호출하여 원장에 초기 에셋 목록을 집어넣을 수 있다.
`{"function":"InitLedger","Args":[]}`
```text
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C mychannel -n basic --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" -c '{"function":"InitLedger","Args":[]}'
```

GetAllAssets 함수를 호출하여 내 채널의 원장에 포함된 에셋을 조회할 수 있다.
```text
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```
```text
[
    {"AppraisedValue":300,"Color":"blue","ID":"asset1","Owner":"Tomoko","Size":5},
    {"AppraisedValue":400,"Color":"red","ID":"asset2","Owner":"Brad","Size":5},
    {"AppraisedValue":500,"Color":"green","ID":"asset3","Owner":"Jin Soo","Size":10},
    {"AppraisedValue":600,"Color":"yellow","ID":"asset4","Owner":"Max","Size":10},
    {"AppraisedValue":700,"Color":"black","ID":"asset5","Owner":"Adriana","Size":15},
    {"AppraisedValue":800,"Color":"white","ID":"asset6","Owner":"Michel","Size":15}
]
```

네트워크의 멤버가 원장에 기록된 에셋을 전송하거나 변경하고 싶다면 asset-transfer 체인코드를 호출해야한다.
`{"function":"TransferAsset","Args":["asset6","Christopher"]}`
```text
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C mychannel -n basic --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" -c '{"function":"TransferAsset","Args":["asset6","Christopher"]}'
```
asset-transfer 코드 보증 정책은 트랜잭션이 Org1과 Org2 모두에서 서명되어야 하므로 --peerAddresses 플래그를 사용하여 둘 다 대상으로 지정하였다.  
넽으워크에 대한 TLS가 활성화되어 있기 때문에 --tlsRootCertFiles 태그를 이용하여 각 피어에 대한 TLS 인증서를 참조한다.  

다시 GetAllAssets 함수를 호출하여 asset6의 소유자가 Michel에서 Christopher로 변경된 것을 확인할 수 있다.
```text
[
    {"AppraisedValue":300,"Color":"blue","ID":"asset1","Owner":"Tomoko","Size":5},
    {"AppraisedValue":400,"Color":"red","ID":"asset2","Owner":"Brad","Size":5},
    {"AppraisedValue":500,"Color":"green","ID":"asset3","Owner":"Jin Soo","Size":10},
    {"AppraisedValue":600,"Color":"yellow","ID":"asset4","Owner":"Max","Size":10},
    {"AppraisedValue":700,"Color":"black","ID":"asset5","Owner":"Adriana","Size":15},
    {"AppraisedValue":800,"Color":"white","ID":"asset6","Owner":"Christopher","Size":15}
]
```
Org2에서도 에셋 변경이 잘 적용되었는지 확인해보기 위해 환경 변수를 일부 수정한다.
```text
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
```
ReadAsset 함수를 호출하여 Org2에서도 소유자가 크리스토퍼로 변경된 것을 확인할 수 있다.
```text
peer chaincode query -C mychannel -n basic -c '{"Args":["ReadAsset","asset6"]}'
```
```text
{"AppraisedValue":800,"Color":"white","ID":"asset6","Owner":"Christopher","Size":15}
```

### 네트워크 중단
```text
./network.sh down
```
노드와 체인코드 컨테이너를 중지, 제거하고 조직 암호화 자료를 삭제한다.
Docker 레지스트리에서 체인코드 이미지도 제거된다.  
채널 아티팩트와 도커 볼륨도 제거되므로 문제가 발생할 경우 이 명령을 수행하여 해결할 수 있다.

## 추가 자료

### 인증 기관(CA)을 통한 네트워크 연결
하이퍼레저 패브릭은 public key infrastructure를 사용하여 네트워크 참여자의 행동을 검증한다.  
모든 노드와 네트워크 관리자, 트랜잭션을 제출할 사용자는 신원 확인을 위해 공개 인증서(public certificate)와 개인 키(private key)가 있어야 한다.  
인증서가 네트워크의 멤버인 기관에서 발급되었음을 확인하는 유효한 신뢰 루트가 필요하다.  
network.sh 스크립트는 피어와 오더링 노드를 만들기 전에 네트워크를 배포하고 운영하는데 필요한 모든 암호화 자료를 만든다.  

기본적으로 스크립트는 crytogen 도구를 사용하여 인증서와 키를 만든다.  
도구는 개발과 테스트를 위해 제공되며 유효한 신뢰 루트가 있는 패브릭 조직에 필요한 암호화 자료를 빠르게 생성할 수 있다.  
./network.sh up을 실행하면 cryptogen 도구가 Org1, Org2, Orderer Org에 맞는 인증서와 키를 생성한다.  

스크립트는 인증 기관을 이용하여 네트워크를 가져오는 옵션도 제공한다.  
프로덕션 네트워크에서 각 조직은 조직에 속한 신원을 만드는 CA를 운영한다.  
조직에서 운영하는 CA에서 생성한 모든 신원은 동일한 신뢰 루트를 공유한다.  
cryptogen 도구를 사용하는 것보다 시간이 오래걸리지만 
CA를 사용하여 테스트 네트워크를 불러오면 네트워크가 프로덕션 환경에서 배포되는 방법에 대한 소개를 얻을 수 있다.  

```text
./network.sh down
./network.sh up -ca
```

테스트 네트워크는 패브릭 CA 클라이언트를 사용하여 각 조직의 CA에 노드 및 사용자 신원을 등록한다.  
그런 다음 스크립트는 등록 명령을 사용하여 각 신원에 대한 MSP 폴더를 생성한다.  
MSP 폴더에는 각 신원에 대한 인증서와 개인 키가 포함되어 있으며 CA를 운영하는 조직에서 신원의 역할과 멤버 자격을 설정한다.  
아래 명령으로 Org1 admin 사용자의 MSP 폴더를 검사해보자.
```text
tree organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/
```
```text
organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/
└── msp
    ├── IssuerPublicKey
    ├── IssuerRevocationPublicKey
    ├── cacerts
    │   └── localhost-7054-ca-org1.pem
    ├── config.yaml
    ├── keystore
    │   └── 3944f779ffd02730cce42d2d81d25d79847ac9365ad866b4e5f9a4bbb8430723_sk
    ├── signcerts
    │   └── cert.pem
    └── user
```
admin 사용자의 인증서는 signcerts 폴더에 개인 키는 keystore 폴더에서 찾을 수 있다.  

cryptogen과 패브릭 CA 모두 각 조직의 organizations 폴더에 암호화 자료를 생성한다.  
organizations/fabric-ca 디렉토리의 registerEnroll.sh 스크립트에서 네트워크를 설정하는 명령을 발견할 수 있다.

### 네트워크를 켤 때 일어나는 일

./network.sh up 명령을 내리면 어떤 일이 일어날까?
* ./network.sh는 피어 조직과 오더러 조직의 인증서와 키를 만든다.  
기본값으로 cryptogen이 사용되며 -ca 옵션을 주면 패브릭 CA를 사용한다.
* 조직의 암호화 자료가 생성되면 network.sh가 네트워크의 노드를 가져올 수 있다.  
스크립트는 피어와 오더러 노드 생성을 위해 docker 폴더의 docker-compose-test-net.yaml 파일을 사용한다.  
docker 폴더에는 docker-composer-e2e.yaml 파일도 있는데 이는 패브릭 CA로 네트워크 노드를 생성할 때 사용된다.
* createChannel 서브명령을 사용하면 ./network.sh는 script 폴더에 있는 createChannel.sh 스크립트를 실행한다.  
createChannel 스크립트는 기본 또는 인자로 제공된 이름의 채널을 만든다.  
configtxgen 도구를 사용하여 채널의 제네시스 블록을 만드는데 
configtx/configtx.yaml 파일에 정의된 TwoOrgsApplicationGenesis 채널 프로파일을 이용한다.
* deployCC 명령어를 사용하면 ./network.sh는 deployCC.sh 스크립트를 실행하여 
두 피어 모두에 asset-transfer 체인코드를 설치한 뒤 채널에 체인코드를 정의한다.  
한 번 체인코드 정의가 채널에 커밋되면 피어 cli는 Init을 사용하여 체인코드를 초기화하고 
원장에 초기 데이터를 넣기 위해서 체인코드가 호출된다.

