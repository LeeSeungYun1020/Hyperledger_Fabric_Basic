# 패브릭 어플리케이션 실행

## 개요
배포된 블록체인 네트워크오 패브릭 어플리케이션이 어떻게 상호작용하는지 살펴보려 한다.
패브릭 게이트웨이 어플리케이션 API를 이용하여 스마트 컨트랙트를 호출한다.
스마트 컨트랙트를 통해 원장을 쿼리하고 수정할 수 있다.

### Asset transfer
Asset transfer 샘플을 통해 에셋을 어떻게 만들고 수정하고 쿼리하는지 설명한다.
1. 샘플 어플리케이션: 블록체인 네트워크를 호출하여 스마트 컨트랙트에 구현된 트랙잭션을 호출한다.
2. 스마트 컨트랙트: 원장과 상호작용하기 위해 구현한 트랙잭션이다. 스마트 컨트랙트는 fabric-samples 디렉토리에 존재한다.

튜토리얼은 두 부분으로 구성되어 있다.
1. 블록체인 네트워크 설정
   - 어플리케이션과 상호작용할 블록체인 네트워크가 필요하므로 기본 네트워크를 시작하고 스마트 컨트랙트를 배포한다.
2. 스마트 컨트랙트와 상호작용할 샘플 어플리케이션 실행
   - 원장에서 에셋을 만들고 쿼리하고 수정하기 위해서 assetTransfer 스마트 컨트랙트를 사용한다.
   - 에셋 초기화, 쿼리, 생성, 수정(이전)하는 어플리케이션과 트랜잭션 코드를 단계적으로 살펴본다.

패브릭 어플리케이션과 스마트 계약이 함께 작동하여
블록체인 네트워크의 분산된 원장에 있는 데이터를 관리하는 방법에 대해 이해할 수 있다.

## 시작하기 전에

[패브릭 테스트 네트워크 실행](/note/test_network.md)의 사전요구사항을 충족하고
build-essential을 설치해야 한다.
```text
sudo apt install build-essential
```

## 블록체인 네트워크 설정

### 블록체인 네트워크 시작

기존 네트워크를 종료한다.
```text
cd fabric-samples/test-network
./network.sh down
```
network.sh 스크립트를 사용하여 테스트 네트워크를 시작한다.
```text
./network.sh up createChannel -c mychannel -ca
```
두 개의 피어, 오더링 서비스, 3개의 인증 기관이 있는 패브릭 테스트 네트워크를 배포한다.
cryptogen 도구를 사용하는 대신 인증 기관을 사용하기 위해 -ca 플래그를 추가하였다.
org admin 사용자 등록은 인증 기관이 시작될 때 알아서 진행된다.

### 스마트 컨트랙트 배포

앞 예제에서 Go와 JavaScript를 사용하였으므로 이번에는 TypeScript를 사용해 본다.
```text
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-typescript/ -ccl typescript
```
스크립트는 체인코드 수명주기를 사용하여 설치된 체인코드를 패키징, 설치, 쿼리하고
Org1, Org2 모두에 대해 체인코드를 승인하고 마지막으로 체인코드를 커밋한다.

## 샘플 어플리케이션 준비

배포된 스마트 컨트랙트와 상호작용할 샘플 에셋 교환 TypeScript 어플리케이션을 준비해보자.
먼저 application-gateway-typescript 디렉토리로 이동한다.
```text
cd ../asset-transfer-basic/application-gateway-typescript
```

디렉토리에는 Node용 패브릭 게이트웨이 어플리케이션 API로 개발된 샘플 어플리케이션이 있다.
npm install로 디펜던시를 설치한다.
```text
npm install
```
package.json에 정의된 어플리케이션 디펜던시를 설치한다.
가장 중요한 패키지는 @hyperledger/fabric-gateway로
패브릭 게이트웨이 연결, 클라이언트 ID를 사용한 트랜잭션 제출과 평가, 이벤트 수신에
사용되는 패브릭 게이트웨이 어플리케이션 API를 제공한다.

src 디렉토리에 클라이언트 어플리케이션 소스 코드가 있다.
설치 과정에서 생성된 JavaScript 출력은 dist 디렉토리에 있다.

## 샘플 어플리케이션 실행

패브릭 테스트 네트워크를 시작할 때, 인증 기관을 통해 신원(ID)을 만들었다.
물론 여기에는 각 조긱의 사용자 신원이 포함된다.
어플리케이션은 이러한 사용지 신원 중 하나의 자격 증명으로 블록체인 네트워크와 거래한다.

어플리케이션을 실행하고 스마트 계약 기능과 각 상호작용을 단계별로 살펴보겠습니다.
asset-transfer-basic/application-gateway-typescript 디렉토리에서 아래 명령을 실행한다.
```text
npm start
```

### 1. 게이트웨이에 gRPC 연결 설정

클라이언트 어플리케이션은 블록체인 네트워크를 거래하는데 사용할 
패브릭 게이트웨이 서비스에 gRPC 연결을 설정한다.
패브릭 게이트웨이의 엔드포인트 주소와 적절한 TLS 인증서(TLS를 사용하도록 구성된 경우)가 필요하다.
여기에서는 게이트웨이 엔드포인트 주소는 패브릭 게이트웨이 서비스를 제공할 피어의 주소로 한다.


> gRPC 연결 설정에는 상당한 오버헤드가 발생한다. 따라서 어플리케이션이 이 연결을 유지하고 
> 패브릭 게이트웨이와 모든 상호 작용에서 사용되어야 한다.

> 트랜잭션에 사용될 비밀 데이터의 보안을 유지하기 위해
> 어플리케이션은 클라이언트 신원과 동일한 조직에 대한 패브릭 게이트웨이에 연결되어야 한다.
> 클라이언트 신원의 조직이 게이트웨이를 호스팅하지 않는 경우
> 다른 조직의 신뢰할 수 있는 게이트웨이를 사용해야 한다.

TypeScript 어플리케이션은 게이트웨이의 TLS 인증서의 진위를 확인할 수 있도록
서명 인증 기관의 TLS 인증서를 사용하여 gRPC 연결을 생성한다.

TLS 연결이 성공적으로 생성되면 클라이언트가 사용할
엔드포인트 주소는 게이트웨이의 TLS 인증서와 주소가 일치해야 한다.
클라이언트는 localhost 주소에서 게이트웨이의 Docker 컨테이너에 액세스하므로
엔드포인트 주소가 게이트웨이의 구성된 호스트 이름으로 해석되도록 gRPC 옵션이 지정된다.

```typescript
const peerEndpoint = 'localhost:7051';

async function newGrpcConnection(): Promise<grpc.Client> {
    const tlsRootCert = await fs.readFile(tlsCertPath);
    const tlsCredentials = grpc.credentials.createSsl(tlsRootCert);
    return new grpc.Client(peerEndpoint, tlsCredentials, {
        'grpc.ssl_target_name_override': 'peer0.org1.example.com',
    });
}
```

### 2. 게이트웨이 연결 생성
어플리케이션은 패브릭 게이트웨이에 접근 가능한 모든 네트워크에 엑세스하는데 사용하는 게이트웨이 연결을 생성하고
해당 네트워크에 배포된 스마트 계약을 생성한다. 게이트웨이 연결은 3가지 요구사항을 만족해야 한다.

1. 패브릭 게이트웨이에 대한 gRPC 연결
2. 네트워크 트랜잭션에 사용될 클라이언트 신원(ID)
3. 클라이언트 ID에 대한 디지털 서명을 생성하는데 사용되는 서명 구현

샘플 어플리케이션은 Org1 사용자의 X.509 인증서를 클라이언트 신원으로 사용하며 사용자의 개인 키에 기반한 서명 구현을 사용한다.
```typescript
const client = await newGrpcConnection();

const gateway = connect({
    client,
    identity: await newIdentity(),
    signer: await newSigner(),
});

async function newIdentity(): Promise<Identity> {
    const credentials = await fs.readFile(certPath);
    return { mspId: 'Org1MSP', credentials };
}

async function newSigner(): Promise<Signer> {
    const privateKeyPem = await fs.readFile(keyPath);
    const privateKey = crypto.createPrivateKey(privateKeyPem);
    return signers.newPrivateKeySigner(privateKey);
}
```

### 3. 호출할 스마트 컨트랙트에 접근(access)
샘플 어플리케이션은 Gateway 연결을 사용하여 Network에 대한 참조를 가져오고
해당 네트워크에 배포된 체인코드 내의 기본 Contract를 가져온다.
```typescript
const channelName = 'mychannel';
const chaincodeName = 'basic';

const network = gateway.getNetwork(channelName);
const contract = network.getContract(chaincodeName);
```
체인코드 패키지에 여러 스마트 계약이 포함된 경우 getContract() 호출에 대한 인수로
체인코드 이름과 특정 스마트 계약의 이름을 모두 제공할 수 있다.
```typescript
const contract = network.getContract('chaincodeName', 'smartContractName');
```

### 4. 샘플 에셋으로 원장 채우기
체인코드 패키지를 처음으로 배포하면 원장은 비어있다.
어플리케이션은 submitTransaction() 함수를 사용하여 InitLedger 트랜잭션 함수를 호출한다.
InitLedger 트랜잭션 함수는 샘플 에셋으로 원장을 채운다.
submitTransaction() 함수가 패브릭 게이트웨이를 사용하는 과정은

1. 트랜잭션 재안을 보증
2. 보증 트랜잭션을 오더링 서비스에 제출
3. 트랜잭션이 커밋될 때까지 대기, 원장 상태 업데이트

```typescript
await contract.submitTransaction('InitLedger');
```

### 5. 에셋 읽기/쓰기 트랜잭션 함수 호출
이제 스마트 컨트랙트에서 트랜잭션 기능을 호출하여 원장의 에셋을 쿼리하고
에셋을 추가하고 수정하는 비즈니스 로직을 실행할 준비가 완료되었다.

#### 모든 에셋 쿼리
evaluateTransaction() 함수는 읽기 전용 트랜잭션을 발생시켜 원장을 쿼리한다.
이 때, 트랜잭션 함수 호출과 결과 반환에는 패브릭 게이트웨이가 사용된다.
트랜잭션은 오더링 서비스에 전송되지 않으며 원장에는 어떠한 변화도 생기지 않는다.

GetAllAssets를 호출하여 앞에서 초기화한 원장의 모든 에셋을 가져와보자.
```typescript
const resultBytes = await contract.evaluateTransaction('GetAllAssets');

const resultJson = utf8Decoder.decode(resultBytes);
const result = JSON.parse(resultJson);
console.log('*** Result:', result);
```

> 트랜잭션 함수의 결과는 항상 바이트로 반환된다. 이는 모든 데이터 타입이 트랜잭션 함수의 결과로 반환될 수 있기 때문이다.
> 예를 들어 위에서 GetAllAssets는 UTF-8 문자열로 JSON 데이터를 반환하였다.
> 어플리케이션은 결과 바이트의 타입을 올바르게 해석해야 한다.

#### 새로운 에셋 추가
샘플 어플리케이션은 CreateAsset을 호출하여 새로운 에셋을 만드는 트랜잭션을 제출한다.
```typescript
const assetId = `asset${Date.now()}`;

await contract.submitTransaction(
    'CreateAsset',
    assetId,
    'yellow',
    '5',
    'Tom',
    '1300',
);
```

> 트랜잭션은 체인코드가 예상하는 것과 동일한 타입, 같은 수의 인자를 제출해야 한다.  
> CreateAsset 트랜잭션은 ID, Color, Size, Owner, AppraisedValue 순으로 인자를 받으므로
> assetId, "yellow", "5", "Tom", "1300"를 인자로 전달하였다.
 
#### 에셋 수정
샘플 어플리케이션은 새로 만든 에셋의 소유권을 변경하는 트랜잭션을 제출한다.
트랜잭션은 submitAsync() 함수로 호출되며 승인된 트랜잭션을 오더링 서비스에 제출 성공한 이후 반환된다.
(* 트랜잭션이 원장에 커밋될 때까지 기다리지 않는다.)
어플리케이션은 커밋을 기다리는 동안 트랜잭션 결과를 이용하여 다른 작업을 수행할 수 있다.
```typescript
const commit = await contract.submitAsync('TransferAsset', {
    arguments: [assetId, 'Saptha'],
});
const oldOwner = utf8Decoder.decode(commit.getResult());

console.log(`*** Successfully submitted transaction to transfer ownership from ${oldOwner} to Saptha`);
console.log('*** Waiting for transaction commit');

const status = await commit.getStatus();
if (!status.successful) {
    throw new Error(`Transaction ${status.transactionId} failed to commit with status code ${status.code}`);
}

console.log('*** Transaction committed successfully');
```

#### 수정한 에셋 쿼리하기
샘플 어플리케이션은 전송된 에셋에 대한 쿼리를 비교하여 에셋이 설명된 속성으로 생성되었으며
이후 새 소유자에게 전송(변경)되었음을 보여준다.
```typescript
const resultBytes = await contract.evaluateTransaction('ReadAsset', assetId);

const resultJson = utf8Decoder.decode(resultBytes);
const result = JSON.parse(resultJson);
console.log('*** Result:', result);
```

#### 트랜잭션 오류 처리하기
시퀀스의 마지막 부분에는 트랜잭션 제출 오류가 나타난다.
샘플 어플리케이션은 존재하지 않는 에셋 ID를 넣어 UpdateAsset 트랜잭션을 제출하려 한다.
트랜잭션 함수에서 에러 응답을 반환하고 submitTransaction() 함수 호출에 실패한다.

submitTransaction() 함수의 실패는 제출 흐름 상에서 오류가 발생한 곳을 표시하고
어플리케이션이 적절하게 응답할 수 있도록 하는 추가 정보를 포함한 여러 유형의 오류를 생성할 수 있다.
발생할 수 있는 에러 타입은 API 문서를 참고해야 한다.
```typescript
try {
    await contract.submitTransaction(
        'UpdateAsset',
        'asset70',
        'blue',
        '5',
        'Tomoko',
        '300',
    );
    console.log('******** FAILED to return an error');
} catch (error) {
    console.log('*** Successfully caught the error: \n', error);
}
```

EndorseError 타입은 보증 과정에서 실패가 발생하였음을 의미한다.
gRPC 상태 코드 ABORTED는 어플리케이션이 패브릭 게이트웨이를 성공적으로 호출했지만 보증 과정에서 실패가 발생하였음을 나타낸다.
gRPC 상태 코드 UNAVAILABLE, DEADLINE_EXCEEDED는 패브릭 게이트웨이에 연결할 수 없거나
시간 내에 응답을 받지 못하였으므로 작업을 재시도하는 것이 적절할 수 있음을 나타낸다.

## 정리하기
```text
cd ../../test-network
./network.sh down
```
asset-transfer 샘플 사용이 끝났으므로 테스트 네트워크를 중단한다.

## 정리
이번 예제에서는 테스트 네트워크를 시작하고 스마트 컨트랙트를 배포하여 블록체인 네트워크를 설정하는 법을 살펴보았다.
클라이언트 어플리케이션을 실행한 뒤 어플리케이션의 코드를 살펴보았다.
패브릭 게이트웨이 API를 사용하여 패브릭 게이트웨이에 연결하고 배포된 스마트 컨트랙트에서 트랜잭션 기능을 호출하여
원장을 쿼리, 추가, 수정하는 방법을 알 수 있었다.