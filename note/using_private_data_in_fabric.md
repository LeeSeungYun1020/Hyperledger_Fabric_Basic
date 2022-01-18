# 패브릭에서 개인(비밀) 데이터 사용

## 개요

튜토리얼에서는 PDC(개인 데이터 컬렉션)를 사용하여 승인된 조직의 승인된 피어를 위해
블록체인 네트워크에서 개인 데이터를 저장하고 검색하는 방법을 설명한다.
컬렉션은 해당 컬렉션을 관리하는 정책이 포함된 컬렉션 정의 파일을 사용하여 지정된다.

이 튜토리얼의 정보는 개인 데티어 저장소와 해당 사용 사례에 대한 배경 지식을 갖추고 있다고 생각하고 진행되므로
사전에 개인 데이터 저장소와 그 예를 살펴보고 아래 내용을 진행하는 것이 추천된다.

이 튜토리얼에서는 개인 데이터 저장소와 그 사용 사례에 대한 배경 지식을 갖추고 있다고 생각하고 진행된다.
개인 데이터 저장소와 예에 대해 알고 있다면 바로 [튜토리얼](#에셋-전송-개인-데이터-샘플-사용-사례)을 진행하자.

## 배경 지식

### 개인 데이터
채널에 있는 조직 그룹이 해당 채널의 다른 조직으로부터 데이터를 비밀로 관리하고 싶은 경우,
해당 데이터에 접근할 필요가 있는 조직으로 구성된 새 채널을 만드는 방법을 생각할 수 있다.
그러나 이러한 경우마다 별도의 채널을 만드는 것은 추가적인 관리 오버헤드(체인코드 버젼, 방침, MSP 관리 등)를 발생시킨다.
또한 데이터 일부를 비공개로 유지하면서 모든 채널 참가자가 트랜잭션을 보도록 하는 경우를 허용하지 않는다.

이것이 패브릭이 개인 데이터 컬렉션(PDC)을 만드는 기능을 제공하는 이유다.
개인 데이터 컬렉션은 채널에 조직의 하위 집합을 정의하도록 허용하여
별도의 채널을 만들지 않고도 개인 데이터를 보증하고 커밋하고 쿼리할 수 있도록 한다.

개인 데이터 컬렉션은 체인코드 정의를 통해 명시적으로 정의할 수 있다.
추가적으로 모든 체인코드에는 조직별 개인 데이터를 위한 암시적인 개인 데이터 이름공간(namespace)이 예약되어 있다.
이 암시적 조직별 개인 데이터 컬렉션은 각 조직의 개인 데이터를 저장하는데 사용된다.
이는 조직이 소유한 에셋에 대한 세부 정보와 체인코드로 구현된 다자간 비즈니스 프로세스 단계에서
조직의 승인과 같은 단일 조직과 관련된 개인 데이터를 저장하려는 경우 유용하다.

### 개인 데이터 컬렉션
컬렉션은 아래 두 요소로 구성된다.
1. 실제 개인 데이터  
개인 데이터는 가십 프로토콜(gossip protocol)을 통해 볼 수 있는 권한이 있는 조직에만 P2P(peer to peer)로 전송된다.
이 데이터는 승인된 조직의 개인 상태(private state) 데이터베이스에 저장된다.
데이터베이스에는 이 허가된 피어에서만 체인코드로 접근할 수 있다.
오더링 서비스는 여기에 관여하지 않으며 개인 데이터를 볼 수 없다.
가십은 승인된 조직 전체에 개인 데이터를 P2P로 배포한다.
그러므로 채널에 앵커(anchor) 피어 설정이 필요하며
조직간 커뮤니케이션을 부트스트랩하기 위해 각 피어에 `CORE_PEER_GOSSIP_EXTERNALENDPOINT`를 구성해야 한다.  
(* 부트스트랩: 한 번 시작되면 알아서 진행되는 일련의 과정)

2. 데이터의 해시  
채널의 모든 피어에서 원장에 보증된, 주문된, 작성된 해당 데이터의 해시(hash).
해시는 트랜잭션의 증거로 제공되며 상태(state) 확인과 감사의 목적으로 사용될 수 있다.

컬렉션 멤버는 분쟁이 발생하거나 에셋을 제3자에게 양도하려는 경우에서와 같이
개인 데이터를 다른 구성원과 공유하도록 결정할 수 있다.
제3자는 개인 데이터의 해시를 계산하여 채널 원장의 상태와 일치하는지 확인하여
특정한 시점에 컬렉션 멤버 사이에 상태가 존재하였음을 증명할 수 있다.

경우에 따라 각각 단일 조직으로 구성된 컬렉션 집합을 갖도록 결정할 수 있다.
일례로 조직이 나중에 다른 채널의 메법와 공유하고 체인코드 트랜잭션에서 참조 가능하도록 
개인 데이터를 자체 컬렉션에 기록할 수 있다. 아래 [개인 데이터 공유](#개인-데이터-공유)에서 이 예를 살펴보자.

#### 채널 내에서 컬렉션 사용하기 VS 별도의 채널에서 컬렉션 사용하기
* 채널 사용: 전체 트랜잭션(과 원장)이 채널의 멤버인 조직 집합 내에서 비밀로 유지되어야 하는 경우
* 컬렉션 사용: 트랜잭션(과 원장)이 조직 집합 간에 공유되어야 하나 일부 조직만이 트랜잭션 중에 데이터에 접근해야 하는 경우
추가적으로 개인 데이터는 블록을 통하지 않고 P2P로 전송되기 때문에
오더링 서비스 노드로부터 트랜잭션 데이터를 기밀로 유지해야 하는 겨우 개인 데이터 컬렉션을 사용한다.

### 컬렉션 설명을 위한 예제
농산물 거래 채널의 5개 조직이 있다고 가정하자.
* 농부: 상품 해외 판매
* 유통 업체: 상품 해외 수출
* 운송 업체: 당사자간 물품 이동
* 도매 업체: 유통 업체에서 상품 구매
* 소매 업체: 하역 업체와 도매 업체에서 상품 구매

유통 업체는 도매 업체와 소매 업체에게 거래 조건을 비밀로 유지하기 위해
농부와 운송 업체 사이에 개인 트랜잭션을 만들고 싶을 수 있다.

유통 업체는 소매 업체 보다 도매 업체에 낮은 가격을 청구하기 때문에
도매 업체와 별도의 개인 데이터 관계를 원할 수 있다.

도매 업체는 소매 업체와 유통 업체 사이에 비밀 데이터 관계를 원할 수 있다.

이러한 관계에 대해 소규모 채널을 계속 정의하는 것보다
다중 개인 데이터 컬렉션(multiple private data collections)을 정의하는 것이 좋다.

1. PDC1: 농부, 유통 업체, 운송 업체
2. PDC2: 유통 업체, 도매 업체
3. PDC3: 운송 업체, 도매 업체, 소매 업체

이 예시에서는 유통 업체가 소유한 피어는 원장 내부에 PDC1, PDC2 관계의
개인 데이터를 포함하는 여러 개인 데이터베이스가 있다.

### 개인 데이터 트랜잭션 흐름

개인 데이터 컬렉션이 체인코드에서 참조될 때
트랜잭션이 제안되고, 승인되며, 원장에 커밋될 때
개인 데티어의 기밀성을 보호하기 위해
트랜잭션 흐름이 약간 다르다.

1. 클라이언트 어플리케이션이 클라이언트 대신 트랜잭션 제출을 관리할 대상 피어에
체인코드 기능(개인 데이터 읽기/쓰기)을 호출하기 위한 제안 요청을 제출한다.
클라이언트 어플리케이션은 어떤 조직이 제안 요청을 보증해야하는지 명시하거나
보증 선택 로직(endorser selection logic)을 대상 피어의 게이트웨이 서비스에 위임할 수 있다.
보증 선택 로직을 위임한 경우, 게이트웨이는 체인코드의 영향을 받는 컬렉션의
승인된 조직 중 일부인 보증 피어(endorsing peer) 세트를 선택하려고 한다.
개인 데이터 또는 체인코드에서 개인 데이터를 만들기 위해 사용되는 데이터는
제안의 `transient` 필드로 보내진다.
2. 보증 피어는 트랜잭션을 시뮬레이션하고 개인 데이터를
`transient data store`(피어별 로컬 임시 저장소)에 저장한다.
컬렉션 방침에 따라 승인되 피어에 가십(gossip)을 통해 개인 데이터를 분배한다.
3. 보증 피어는 제안 응답을 대상 피어로 다시 보낸다.
제안 응답은 공개 데이터와 개인 데이터 키, 값의 해시가 포함된
보증된 읽기/쓰기 세트가 포함된다.
개인 데이터는 대상 피어나 클라이언트에 재전송되지 않는다.
4. 대상 피어는 보증을 트랜잭션에 결합하기 전에
서명을 위해 클라이언트로 다시 보낸 제안 요청을 검증한다.
대상 피어는 트랜잭션을 오더링 서비스로 전송한다.
개인 데이터 해시가 포함된 트랜잭션은 정상적으로 블록에 포함된다.
이러한 방법으로 채널의 모든 피어가 실제 개인 데이터를 몰라도
일관된 방법으로 개인 데이터의 해시를 이용해 트랜잭션을 검증할 수 있다.
5. 블록이 커밋될 때, 승인된 피어는 개인 데이터에 접근하도록 허가되었는지 
확인하기 위해 켤렉션 방침을 사용한다.
먼저 로컬 `transient data store`(임시 데이터 저장소)를 확인하여 체인코드 승인 시점에 개인 데이터를 이미 수신하였는지 확인한다.
수신하지 않았다면, 다른 승인된 피어에서 개인 데이터를 가져온다.
가져온 개인 데이터는 공개 블록의 해시에 대해 개인 데이터의 유효성을 검사하고 트랜잭션과 블록을 커밋한다.
유효성 검사/커밋시 개인 데이터는 개인 상태 데이터베이스 및 개인 쓰기셋 저장소의 복사본으로 이동한다.
개인 데이터는 `transient data store`에서 삭제된다.

### 개인 데이터 공유

많은 시나리오에서 컬렉션의 개인 데이터 키/값의 다른 채널의 멤버 또는 컬렉션과 공유해야 하는 경우가 있다.
기존 개인 데이터 컬렉션에 포함되지 않은 채널 멤버 또는 채널 멤버 그룹과 개인 데이터를 트랜잭션해야 할 수 있다.
수신한 당사자는 일반적으로 거래의 일부로 체인 상(on-chain)의 해시에 대해 개인 데이터를 확인하기를 원할 것이다.

개인 데이터의 공유와 검증을 가능하게 하는 개인 데이터 수집의 여러 방향이 있다.
* 보증 정책이 충족된다면 컬렉션의 키에 쓰려면 컬렉션 멤버가 아니어도 된다.
보증 정책은 체인코드 수준, 키 수준(상태 기반 보증), 컬렉션 수준(v2 이상)에서 정의 가능하다.
* 체인코드 API 중 `GetPrivateDataHash()`는 멤버가 아닌 피어의 체인코드가
개인 키의 해시 값을 읽을 수 있도록 해준다.(v1.4.2 이상) 
체인코드가 이전 트랜잭션의 개인 데이터에서 생성된 체인 상의 해시에 대해
개인 데이터를 확인할 수 있도록 하기 때문에 중요한 기능이다.

개인 데이터 공유와 확인 기능은
어플리케이션과 관련된 개인 데이터 컬렉션을 설계할 때 고려되어야 한다.
채널 멤버의 다양한 조합 간에 데이터를 공유하기 위해
다자간 개인 데이터 컬렉션 셋을 명확하게 생성할 수 있지만
이러한 접근 방식을 사용하면 정의해야 하는 컬렉션이 많을 수 있다.
그 대신 적은 수의 개인 데이터 컬렉션을 사용(ex 조직당 컬렉션 하나, 조직 쌍당 컬렉션 하나)하고
필요에 따라 다른 채널 멤버 또는 다른 컬렉션과 개인 데이터를 공유하는 것을 생각해 볼 수 있다.
패브릭 v2.0 부터 모든 체인코드에서 암시적 조직별 컬렉션을 사용할 수 있다.
암시적 체인코드를 이용하여 체인코드 배포시 조직별 컬렉션을 정의할 필요가 없어졌다.

#### 개인 데이터 공유 패턴

조직별로 개인 데이터를 모델링할 경우
많은 다자간 컬렉션을 정의하는 오버헤드 없이
개인 데이터를 공유하거나 전송하는데 여러가지 패턴을 사용할 수 있다.
다음은 체인코드 어플리케이션에서 활용 가능한 몇 가지 공유 패턴이다.
* 공개 상태 추적을 위해 해당 공개키 사용
* 체인코드 접근 제어
* 외부(다른 조직)에 개인 데이터 공유
* 다른 컬렉션과 개인 데이터 공유
* 다른 컬렉션에 개인 데이터 전송
* 개인 데이터를 트랜잭션 승인에 사용
* 거래 당사자를 비공개로 유지

위의 패턴과 함께 개인 데이터가 있는 트랜잭션은 일반적인 채널 상태 데이터와
동일한 조건으로 바인딩될 수 있다. 언급하자면 아래와 같다.
* 키 수준에서 트랜잭션 접근 제어
* 키 수준 보증 정책

#### 개인 데이터 컬렉션을 이용한 에셋 전송 시나리오

개인 데이터 공유 패턴을 결합하여 강력한 체인코드 기반 어플리케이션을 사용할 수 있다.
그 예로, 조직별 개인 데이터 컬렉션으로 자산 전송 시나리오를 구현하는 방법을 살펴보자.

* 에셋은 공용 체인코드 상태의 UUID 키로 추적 가능하다.
에셋의 소유권만 기록된다.
* 체인코드는 모든 전송 요청이 소유한 클라이언트에서 시작할 것을 요구한다.
키는 소유한 조직과 규제 조직의 피어가 모든 전송 요청을 승인할 것을 요구하는
상태 기반 보증에 종속된다.
* 에셋 소유자의 개인 데이터 컬렉션은 에셋에 대한 비밀 정보를 포함하고 있다.
데이터는 UUID의 해시를 키로 한다.
다른 조직과 오더링 서비스는 에셋 세부 정보의 해시만 확인할 수 있다.
* 규제 조직이 각 컬렉션의 멤버이기도 하므로 개인 데이터를 유지한다고 가정해보자.
  (반드시 그럴 필요는 없음)

에셋 트랜잭션은 아래와 같이 전개될 것이다.

1. 체인 밖에서 소유자와 잠재적 구매자가 자산을 특정 가격으로 거래하기 위한 거래를 체결한다.
2. 판매자는 대역 외로 개인 세부 정보를 전달하거나 구매자에게 노드 또는 규제자의 노드에서
개인 데티어를 쿼리할 수 있는 자격 증명을 제공하여 소유권을 증명한다.
3. 구매자는 개인 세부 정보의 해시가 체인 상의 공개 해시와 알치하는지 확인한다.
4. 구매자는 자신의 개인 데이터 컬렉션에 입찰 세부 정보를 기록하기 위해 체인코드를 호출한다.
체인코드는 구매자의 피어에서 호출되며 수집 보증 정책에서 요구하는 경우 잠재적으로 규제자의 피어에서 호출된다.
5. 현재 소유자는 개인 세부 정보와 입찰 정보를 전달하여 에셋을 판매 및 이전하기 위해 체인코드를 호출한다.
체인코드는 공개 키의 보증 정책과 구매자와 판매자 개인 데이터 컬렉션의 보증 정책을 충족하기 위해
판매자, 구매자, 규제자의 피어에서 호출된다.
6. 체인코드는 제출한 클라이언트가 소유자인지 확인하고
판매자 컬렉션의 해시에 대해 입찰 세부 정보를 확인한다.
이후 체인코드는 공개 키에 대해 제안된 업데이트를 작성한다.
   (구매자에 소유권을 설정하고 보증 정책을 구매 조직과 규제자로 설정한다.)
체인코드는 개인 비밀 정보를 구매자의 개인 데이터 컬렉션에 쓰고
잠재적으로 판매자 컬렉션에서 개인 세부 정보를 삭제한다.
최종 승인 전에 승인하는 피어는 개인 데이터가 판매자와 규제자의
다른 승인된 피어에 배포되도록 한다.
7. 판매자는 주문을 위해 공개 데이터와 개인 데이터의 해시를 트랜잭션으로 제출하고
블록의 모든 채널 피어에 배포된다.
8. 각 피어의 블록 검증 로직은 지속적으로 보증 정책이 만족하는지 확인한다.
체인코드에서 읽은 공개 및 개인 상태가 체인코드 실행 이후 다른 트랜잭션에 의해 수정되지 않았는지 확인한다.
9. 모든 피어가 유효성 검사를 거쳐 트랜잭션을 유효하다고 커밋한다.
구매자 피어와 규제자 피어는 승인 시점에 수신하지 못한 다른 승인된 피어에서
개인 데이터를 검색하고 개인 데이터 상태 데이터베이스에 개인 데이터를 유지한다.
10. 트랜잭션이 완료되어 에셋이 전송되었다.
에셋에 관심이 있는 다른 채널 멤버는 출처를 파악하기 위해 공개 키의 기록을 쿼리할 수 있다.
그러나 소유자가 공유하지 않는 개인 세부 정보에는 접근할 수 없다.

기본적인 에셋 전송 시나리오는 다른 고려사항에 따라 확장될 수 있다.
전송 체인코드가 지불 대 배송 요구사항을 만족하는지 지불 기록을 사용하여 확인할 수 있다.
전송 체인코드 실행 전에 은행이 신용장을 제출하였는지(결제가 완료되었는지) 확인할 수도 있다.
거래자가 피어를 직접 호스팅하지 않고 피어를 실행하는 관리 조직을 통해 거래할 수도 있다.

### 개인 데이터 삭제

데이터 고유 당사자가 삭제하기 원하거나 정부 또는 법령에 따라 삭제가 필요한
매우 민감한 데이터의 경우 주기적인 제거가 필요할 수 있다.
이러한 경우 개인 데이터의 변경 불가능한 증거로 사용하기 위해 블록체인에 데이터 해시를 남길 수 있다.

개인 데이터는 피어의 블록체인 외부 데이터베이스에 복제될 수 있을 때까지 피어의 개인 데이터베이스에만 있으면 된다.
데이터는 체인코드 비즈니스 과정이 완료될 때까지만 피어에 있어야 할 수도 있다.(거래 정산, 계약 이행시까지 등)

이러한 사용 사례 지원을 위해 구성 가능한 블록 수에 대해 개인 데이터가 수정되지 않은 경우 개인 데이터를 제거할 수 있다.
제거된 개인 데이터는 체인코드에서 쿼리할 수 없으며 다른 요청 피어에서 사용할 수 없다.

## 에셋 전송 개인 데이터 샘플 사용 사례

## 컬렉션 정의 JSON 파일 빌드

## 체인코드 API를 이용한 개인 데이터 읽기/쓰기

## 네트워크 시작

## 개인 데이터 스마트 컨트랙트를 채널에 배포

## 신원(ID) 등록

## 개인 데이터에 에셋 생성

## 승인된 피어로 개인 데이터 쿼리

## 승인되지 않은 피어로 개인 데이터 쿼리

## 에셋 전송

## 개인 데이터 삭제

## 개인 데이터와 인덱스 사용

## 정리하기