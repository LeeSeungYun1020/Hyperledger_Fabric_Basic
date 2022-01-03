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
