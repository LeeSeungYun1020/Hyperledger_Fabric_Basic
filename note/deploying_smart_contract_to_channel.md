# 채널에 스마트 컨트랙트 배포하기

## 개요

최종 사용자는 스마트 컨트랙트를 통해 블록체인 원장과 상호작용한다.
하이퍼레저 패브릭에서는 체인코드라고 불리는 패키지로 스마트 컨트랙트가 배포된다.
조직이 트랜잭션을 검증하거나 원장에 쿼리하기 위해서는 피어에 체인코드를 설치해야한다.
체인코드가 채널에 합류한 피어에 설치된 뒤에 채널 맴버는 채널에 체인코드를 배포하고
채널 원장의 에셋을 생성/수정하는 체인코드에 탑재된 스마트 컨트랙트를 사용할 수 있다.

페브릭 체인코드 수명주기(lifecycle)의 과정을 거쳐 체인코드가 채널에 배포된다.
패브릭 체인코드 수명주기를 통해 여러 조직에서 트랜색션을 생성하기 이전에 어떻게 체인코드가 동작할지(운영 방식에) 동의할 수 있다.
예를 들어 보증 정책이 트랜잭션을 검증하기 위해 체인코드를 실행해야 하는 조직을 지정하면
채널 구성원은 패브릭 체인코드 수명주기를 사용하여 체인코드 보증 정책에 동의해야 한다.

## 네트워크 시작하기

[패브릭 테스트 네트워크 실행](/note/test_network.md)의 사전요구사항을 충족해야한다.
```text
cd fabric-samples/test-network
```
```text
./network.sh down
./network.sh up createChannel
```
앞선 예제에서 살펴본 것과 같이 createChannel은 mychannel 채널과 Org1, Org2 조직을 만든다.

이제 피어 CLI를 사용하여 asset-transfer 체인코드를 배포할 수 있다.
1. [스마트 컨트랙트 패키징](#스마트-컨트랙트-패키징)
2. [체인코드 패키지 설치](#체인코드-패키지-설치)
3. [체인코드 정의 승인](#체인코드-정의-승인)

## 로그 라우터 Logspout 설정 (선택)

스마트 컨트랙트 관련 도커 컨테이너의 로그를 쉽게 보기 위해 Logspout 도구를 사용한다.
Logspout은 도커 컨테이너의 출력을 모아서 보여주어 디버그를 쉽게 한다.
특히 일부 컨테이너의 경우 스마트 계약만을 목적으로 생성되어 짧은 시간 동안만 존재하기 때문에 네트워크에서 모든 로그를 수집하는 것이 좋다.

Logspout을 설치/설정하기 위해 commercial-paper에 포함된 monitordocker.sh 스크립트를 이용한다.
```text
cp ../commercial-paper/organization/digibank/configuration/cli/monitordocker.sh .
find . -name monitordocker.sh
./monitordocker.sh fabric_test
```

##스마트 컨트랙트 패키징

피어에 설치하기 전에 체인코드 패키징이 필요하다.

### Go
체인코드 패키징을 위해 체인코드 디펜던시를 설치해야한다.
아래 폴더의 go.mod 파일에 디펜던시가 작성되어있다.
```text
cd fabric-samples/asset-transfer-basic/chaincode-go
cat go.mod
```
asset-transfer-basic/chaincode-go/chaincode/smartcontract.go 파일에서 
SmartContract 타입을 어떻게 정의하였는지 확인할 수 있다.
SmartContract 타입은 블록체인 원장에서 데이터를 읽고 쓰는 스마트 컨트랙트 내에 정의된
기능에 대한 트랜잭션 컨텍스트를 만드는데 사용된다.
```text
cat chaincode/smartcontract.go
```
```text
vi chaincode/smartcontract.go
```
이제 스마트 컨트랙트 디펜던시를 설치하자.  
아래 코드는 asset-transfer-basic/chaincode-go 디렉토리에서 실행해야 한다.
```text
GO111MODULE=on go mod vendor
```

필요한 디펜던시를 준비 완료했으므로 체인코드 패키지를 생성할 수 있게 되었다.
peer CLI를 이용하여 요구되는 포맷에 맞는 체인코드 패키지를 생성할 수 있다.
피어 바이너리는 bin 폴더에 있으므로 환경 변수에 bin 폴더를 추가한다.
또한 core.yaml 파일 경로를 FABRIC_CFG_PATH로 선언한다.
```text
cd ../../test-network
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
```
체인코드 패키지를 peer lifecycle chaincode package 명령을 이용하여 생성한다.
아래 명령을 실행하면 basic.tar.gz 파일이 생성된다.
```text
peer lifecycle chaincode package basic.tar.gz --path ../asset-transfer-basic/chaincode-go/ --lang golang --label basic_1.0
```
--lang은 체인코드의 언어, --path는 스마트 컨트랙트 코드 경로,
--label은 체인코드 설치 후 구분하는 라벨을 받는 플래그이다.
라벨에는 체인코드 이름과 버젼을 명시하는 것이 권장된다.  
이제 테스트 네트워크 피어에 체인코드를 설치할 수 있다.

### JavaScript
JavaScript를 사용할 경우에도 Go와 유사하게 패키징할 수 있다.
```text
cd fabric-samples/asset-transfer-basic/chaincode-javascript
cat package.json
npm install
```
```text
cd ../../test-network
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
peer lifecycle chaincode package basic.tar.gz --path ../asset-transfer-basic/chaincode-javascript/ --lang node --label basic_1.0
```

##체인코드 패키지 설치

asset-transfer 스마트 컨트랙트를 패키징한 뒤에는 피어에 체인코드를 설치할 수 있다.
체인코드는 트랜색션을 보증할 모든 피어에 설치해야한다.  
Org1과 Org2 모두에서 보증을 요구하도록 보증 정책을 설정하려한다.
이를 위해 두 조직에서 운영하는 피어에 체인코드를 설치해야한다.

먼저 Org1 피어에 체인코드를 설치하자.
```text
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051
peer lifecycle chaincode install basic.tar.gz
```
같은 방법으로 Org2 피어에 체인코드를 설치하자.
```text
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
peer lifecycle chaincode install basic.tar.gz
```

##체인코드 정의 승인

체인코드 패키지를 설치한 뒤, 조직에 체인코드 정의를 승인해야 한다.
정의에는 이름, 버젼, 체인코드 보증 정책 같이 체인코드 거버넌스의 중요한 매개변수를 포함한다.

체인코드를 배포되기 이전에 승인을 받아야 하는 채널 멤버 집합은 /Channel/Application/LifecycleEndorsement 방침을 적용받는다.
기본적으로 이 정책은 채널 구성원의 대다수가 채널에서 체인코드를 사용하기 전에 승인할 것을 요구한다.
우리는 채널에 조직이 2개 뿐이므로 과반수는 2이다.
따라서 asset-transfer 체인코드 정의를 위해 Org1과 Org2의 승인이 필요하다.  

조직이 피어에 체인코드를 설치하면 조직에서 승인한 체인코드 정의에 패키지 ID를 포함해야 한다.
패키지 ID는 피어에 설치된 체인코드를 승인된 체인코드 정의와 연결할 때 사용되며
조직이 체인코드를 사용하여 트랜잭션을 승인할 수 있도록 한다.
체인코드의 패키지 ID는 peer lifecycle chaincode queryinstalled 명령을 통해 찾을 수 있다.
```text
peer lifecycle chaincode queryinstalled
```
```text
Installed chaincodes on peer:
Package ID: basic_1.0:dee2d612e15f5059478b9048fa4b3c9f792096554841d642b9b59099fa0e04a4, Label: basic_1.0
```
체인코드 승인에 패키지 ID를 사용하기 위해 얻은 패키지 ID를 변수로 지정한다.  
패키지 ID는 다를 수 있기 때문에 위 명령을 통해 얻은 패키지 ID로 변경하여 사용하길 바란다.
```text
export CC_PACKAGE_ID=basic_1.0:dee2d612e15f5059478b9048fa4b3c9f792096554841d642b9b59099fa0e04a4
```
환경 변수를 Org2 Admin의 피어 CLI로 작동되도록 설정했었기 때문에
Org2로 asset-transfer 체인코드 정의를 승인할 수 있다.
체인코드는 조직 레벨에서 승인되며 명령은 하나의 피어만 대상으로 하면 된다.
승인은 가십(gossip)을 사용하여 조직 내의 다른 동료에게 배포된다.
peer lifecycle chaincode approveformyorg 명령을 통해 체인코드 정의를 승인할 수 있다.
```text
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name basic --version 1.0 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```
--package-id 플래그는 체인코드 정의에 패키지 식별자를 넣는다.  
--sequence 파라미터는 체인코드가 정의/업데이트된 횟수를 나타내는 정수이다.
여기에서는 체인코드가 처음 배포되므로 횟수는 1이다.
체인코드가 업그레이드되면 2로 횟수가 증가될 것이다.  
저수준 API인 체인코드 Shim API를 사용한다면 --init-required 플래그를 통해
체인코드를 초기화하기 위한 Init 함수 실행을 요청할 수 있다.
체인코드가 처음으로 호출된 경우 Init 함수를 대상으로 한다.
--isInit 플래그를 통해 체인코드의 다른 함수를 이용하여 원장에서 상호작용할 수 있다.

approveformyorg 명령에는 체인코드 보증 정책을 명시하기 위해
--signature-policy나 --channel-config-policy 인자를 전달할 수 있다.
보증 정책은 서로 다른 채널 멤버에 속한 피어가 주어진 체인코드에 대해 트랜잭션을 검증해야 하는 수를 지정한다.
여기서는 정책을 지정하지 않았으므로, asset-transfer는 기본 보증 정책을 사용한다.
기본 보증 정책은 앞서 살펴보았듯 과반수가 넘는 채널 멤버가 트랜색션을 보증해야 한다.
채널에 새로운 조직이 추가되거나 삭제되면 보증 정책이 자동으로 업데이트되어 필요한 보증 수가 조정된다.
여기에서는 2개의 조직이 존재하므로 과반수는 2이다. 따라서 트랜색션은 Org1과 Org2의 피어 모두에서 보증되어야 한다.

체인코드 정의는 admin 권한을 가지고 있는 신원으로 승인해야 한다.
결과적으로 admin 신원이 저장된 MSP 폴더 경로를 가리키는 CORE_PEER_MSPCONFIGPATH 변수가 필요하다.
클라이언트 사용자로는 체인코드 정의를 승인할 수 없다.
승인을 오더링 서비스에 제출하면 admin의 서명을 확인한 다음 승인을 피어에 배포한다.  

Org1 admin으로 교체하여 체인코드 정의를 승인하자.
```text
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_ADDRESS=localhost:7051
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name basic --version 1.0 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```

이제 과반수가 asset-transfer 체인코드를 채널에 배포하는 것을 승인하였다.
과반수의 조직만 체인코드 정의를 승인하면 되지만 피어에서 체인코드를 시작하려면 해당 조직은 체인코드 정의를 승인해야 한다.
채널의 멤버가 체인코드를 승인하기 전에 정의를 커밋하면 조직은 트랜색션을 보증할 수 없다.
결과적으로 모든 채널의 멤버가 체인코드 정의를 커밋하기 전에 체인코드를 승인하는 것이 권장된다.

## 채널에 체인코드 정의 커밋하기

충분한 수의 조직이 체인코드 정의를 승인하면, 한 조직이 채널에 체인코드 정의를 커밋할 수 있다.
과반이 넘는 채널의 멤버가 정의를 승인하면 커밋 트랜색션은 성공하며
체인코드 정의에서 동의한 매개변수가 채널에서 구현된다.

peer lifecycle chaincode checkcommitreadiness 명령으로 채널의 멤버가 체인코드 정의를 승인하였는지 확인할 수 있다.
사용되는 플래그는 조직에서 체인코드를 승일할 때 사용하였던 것과 동일하나
--package-id 플래그는 넣지 않아도 된다.
```text
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name basic --version 1.0 --sequence 1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" --output json
```
```text
{
        "approvals": {
                "Org1MSP": true,
                "Org2MSP": true
        }
}
```

채널의 멤버인 두 조직이 동일한 파라미터로 승인하였기 때문에 체인코드 정의는 채널에 커밋할 준비가 완료되었다.
peer lifecycle chaincode commit 명령을 통해 채널에 체인코드 정의를 커밋할 수 있다.
커밋 명령은 조직 admin이 제출해야 한다.
```text
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name basic --version 1.0 --sequence 1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
```
--peerAddressed 플래그는 Org1의 peer0.org1.example.com과 Org2의 peer0.org2.example.com를 대상으로 지정한다.  
commit 트랜색션은 피어를 운영하는 조직에서 승인한 체인코드 정의를 쿼리하기 위해 채널에 가입한 피어에게 제출된다.
명령은 체인코드 보증 정책을 충족하기 위한 충분한 수의 조직의 피어를 대상으로 해야 한다.
승인은 각 조직 내에서 배포되므로 채널 멤버에 속한 모든 피어를 대상으로 지정할 수 있다.

채널 멤버에게 보증된 체인코드 정의는 블록에 추가되고 채널에 배포되기 위해 오더링 서비스에 제출된다.
채널의 피어는 충분한 수의 조직이 체인코드 정의를 승인하였는지 검증한다.
peer lifecycle chaincode commit 명령은 응답을 반환하기 전에 피어의 검증 검사를 기다린다.

peer lifecycle chaincode querycommitted 명령으로 체인코드 정의가 채널에 커밋되었는지 확인할 수 있다.
```text
peer lifecycle chaincode querycommitted --channelID mychannel --name basic --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```
체인코드가 채널에 성공적으로 커밋되었다면 아래처럼 체인코드 정의의 횟수와 버젼이 표시된다. 
```text
Committed chaincode definition for chaincode 'basic' on channel 'mychannel':
Version: 1.0, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc, Approvals: [Org1MSP: true, Org2MSP: true]
```

## 체인코드 호출

체인코드 정의가 채널에 커밋된 후에는 체인코드가 설치된 피어에서 체인코드가 시작된다.
asset-transfer 체인코드는 클라이어트 어플리케이션에서 호출될 준비가 완료되었다.
아래 명령을 이용해 원장에 초기 에셋을 생성할 수 있다.
호출 명령은 체인코드 보증 정책 충족을 위한 충분한 수의 피어를 대상으로 해야 한다.
(CLI는 패브릭 게이트웨이 피어에 엑세스하지 않으므로 각 보증 피어를 지정해야 한다.)
```text
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C mychannel -n basic --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" -c '{"function":"InitLedger","Args":[]}'
```
초기화가 잘 되었는지 확인해보자.
```text
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```
