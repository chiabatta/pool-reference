## Pool Reference V1
이 코드는 Apache 2.0 라이선스에 따라 제공됩니다. 
참고 : 초안 사양은 SPECIFICATION.md 파일에 있습니다.

### Summary
이 저장소는 Chia Pool의 기반 역할을 하는 Python으로 작성된 샘플 서버를 제공합니다.
이것은 완전한 기능의 구현이지만 프로덕션에서 실행하려면 확장 성과 보안에 대한 약간의 작업이 필요합니다.
An FAQ is provided here: https://github.com/Chia-Network/chia-blockchain/wiki/Pooling-FAQ

### Customizing
커스터마이징 가능한 것들
* 풀에서 나가기위한 제한 시간
* 난이도 조정 방법
* 취할 수수료 및 블록 체인 수수료로 지불 할 금액
* 지불시 farmer의 포인트가 계산되는 방법 (PPS, PPLNS 등)
* farmer가 지불금을받는 방법 (XCH, BTC, ETH 등) 및 빈도
* 사용되는 저장소 (DB)-기본적으로 SQLite db입니다. 사용자는`AbstractPoolStore`를 기반으로하는 자체 스토어 구현을`pool_server.start_pool_server`에 제공하여 사용할 수 있습니다.

그러나 일부는 변경할 수 없습니다. 이는 SPECIFICATION.md에 설명되어 있으며 주로 유효성 검사, 프로토콜 및 스마트 코인의 싱글톤 형식과 관련이 있습니다.

### Pool Protocol Benefits
Chia 풀 프로토콜은 제3자, 폐쇄 된 코드 또는 신뢰할 수 있는 행동에 의존하지 않고 보안 및 탈 중앙화를 위해 설계되었습니다.

* farmer는 중복파밍으로 pool에서 정상적이지 않게 보상을 가져 갈 수 없습니다.
* farmer는 pool에 참여하기 위해 담보가 필요하지 않으며 싱글 톤을 만드는 데 몇 센트 만 필요합니다.
* farmer는 원하는 경우 쉽고 안전하게 풀을 변경할 수 있습니다.
* farmer는 풀 노드를 실행할 수 있습니다 (분권화 증가).
* farmer는 24 개의 단어만으로 다른 컴퓨터에 로그인 할 수 있으며, 중앙 서버 없이도 풀링 구성이 설정됩니다.

### Pool Protocol Summary
풀링하지 않을 때 농부는 9 초마다 전체 노드에서 간판 포인트를 받고이 간판 포인트를 수확기에 보냅니다. 각 사이 니지 포인트는 `sub_slot_iters`및 `difficulty`와 함께 전송되며, 매일 조정되는 두 개의 네트워크 전체 매개 변수 (4608 개 블록)입니다. `sub_slot_iters`는 네트워크에서 가장 빠른 VDF에 대해 10 분 동안 수행 된 VDF 반복 횟수입니다. 가장 빠른 타임로드의 속도가 증가하면 증가합니다. 난이도는 타임로드 속도의 영향을 받지만 (블록이 더 빨라지므로 타임로드 속도가 증가하면 증가합니다) 네트워크의 총 공간에도 영향을받습니다. 이 두 매개 변수는 블록을 "승리"하고 증명을 찾는 것이 얼마나 어려운지를 결정합니다.

Since only about 1 farmer wordwide finds a proof every 18.75 seconds, this means the chances of finding one are 
extremely small, with the default `difficulty` and `sub_slot_iters`. For pooling, what we do is we increase the 
`sub_slot_iters` to a constant, but very high number: 37600000000, and then we decrease the difficulty to an
artificially lower one, so that proofs can be found more frequently.

The farmer communicates with one or several pools through an HTTPS protocol, and sets their own local difficulty for
each pool. Then, when sending signage points to the harvester, the pool `difficulty` and `sub_slot_iters` are used. 
This means that the farmer can find proofs very often, perhaps every 10 minutes, even for small farmers. These proofs,
however, are not sent to the full node to create a block. They are instead only sent to the pool. This means that the 
other full nodes in the network do not have to see and validate everyone else's proofs, and the network can scale to
millions of farmers with no issue, as long as the pool scales properly. Since many farmers are part of the pool,
only 1 farmer needs to win a block, for the entire pool to be rewarded proportionally to their space.

The pool then keeps track of how many proofs (partials) each farmer sends, weighing them by difficulty. Occasionally 
(for example every 3 days), the pool can perform a payout to farmers based on how many partials they submitted. Farmers
with more space, and thus more points, will get linearly more rewards. 

Instead of farmers using a `pool_public_key` when plotting, they now use a puzzle hash, referred to as the 
`p2_singleton_puzzle_hash`, also known as the `pool_contract_address`. These values go into the plot itself, and 
cannot be changed after creating the plot, since they are hashed into the `plot_id`. The pool contract address is the
address of a chialisp contract called a singleton. The farmer must first create a singleton on the blockchain, which
stores the pool information of the pool that that singleton is assigned to. When making a plot, the address of that
singleton is used, and therefore that plot is tied to that singleton forever. When a block is found by the farmer, 
the pool portion of the block rewards (7/8, or 1.75XCH) go into the singleton, and when claimed, 
go directly to the pool's target address. 

The farmer can also configure their payout instructions, so that the pool knows where to send the occasional rewards
to.
### Receiving partials
partial은 특정 최소 난이도 요구 사항을 충족하는 농부의 추가 메타 데이터 및 인증 정보가 포함된 공간 증명입니다. 부분은 블록 체인 사이 니지 포인트에 응답하는 공간의 실제 증거 여야하며 블록 체인 시간 창 (사이 니지 포인트 후 28 초) 내에 제출해야합니다.

풀 서버는 사용자로부터 partial을 수신하고 사용자들이 옳고 블록 체인의 유효한 signage point에 해당하는지 확인한 다음 대기열에 추가하는 방식으로 작동합니다. 몇 분 후 풀이 대기열에서 가져와 해당 partial의 signage point가 여전히 블록 체인에 있는지 확인합니다. 모든 것이 양호하면 partial이 유효한 것으로 간주되고 해당 farmer에 대한 점수가 추가됩니다.


### Collecting pool rewards
![Pool absorbing rewards image](images/absorb.png?raw=true "Absorbing rewards")

풀은 주기적으로 블록체인에서 새로운 풀 보상(1.75XCH)을 찾습니다. 이렇게 찾은 새로운 풀 보상(1.75XCH)은 각 farmer의 다양한 `p2_singleton_puzzle_hashes` 로 갑니다.
이 코인은 잠겨 있으며 해당하는 싱글 톤과 함께 사용하는 경우에만 사용할 수 있습니다.
싱글 톤은이 다이어그램에서 빨간색 풀 주소 인`target_puzzle_hash`에도 잠겨 있습니다. 싱글 톤과`p2_singleton_puzzle_hash` 코인은 블록 보상이고 모든 조건이 충족되는 한 누구나 사용할 수 있습니다. 이러한 조건 중 일부는 싱글 톤이 항상 동일한 런처 ID로 정확히 1 개의 새로운 하위 싱글 톤을 생성하고 코인베이스 자금이 `target_puzzle_hash`로 전송되어야합니다.
### Calculating farmer rewards

주기적으로 (예 : 하루에 한 번) 풀은`create_payment_loop`의 코드를 실행합니다. 이것은 먼저 특정 수의 확인이있는 풀에서 확인 된 모든 자금을 합산합니다.

그런 다음 풀은 총 금액을 모든 풀 멤버의 포인트로 나누어`mojo_per_point` (풀 요금과 블록 체인 요금을 뺀)를 얻습니다. 각 풀 멤버 (및 풀)에 대해 새 코인이 생성되고 지불이 pending_payments 목록에 추가됩니다. 블록의 크기는 최대이므로 각 트랜잭션의 크기를 제한해야합니다.
구성 가능한 매개 변수 인`max_additions_per_transaction`이 있습니다. 보류 목록에 지불을 추가하면 풀 멤버의 포인트가 모두 0으로 재설정됩니다. 이 로직은 사용자 정의 할 수 있습니다.

### 1/8 vs 7/8
Note that the coinbase rewards in Chia are divided into two coins: the farmer coin and the pool coin. The farmer coin
(1/8) only goes to the puzzle hash signed by the farmer private key, while the pool coin (7/8) goes to the pool.
The user transaction fees on the blockchain are included in the farmer coin as well. This split of 7/8 1/8 exists
to prevent attacks where one pool tries to destroy another by farming partials, but never submitting winning blocks.

### Difficulty
The difficulty allows the pool operator to control how many partials per day they are receiving from each farmer.
The difficulty can be adjusted separately for each farmer. A reasonable target would be 300 partials per day,
to ensure frequent feedback to the farmer, and low variability.
A difficulty of 1 results in approximately 10 partials per day per k32 plot. This is the minimum difficulty that
the V1 of the protocol supports is 1. However, a pool may set a higher minimum difficulty for efficiency. When
calculating whether a proof is high quality enough for being awarded points, the pool should use
`sub_slot_iters=37600000000`.
If the farmer submits a proof that is not good enough for the current difficulty, the pool should respond by setting
the `current_difficulty` in the response.

### Points
X points are awarded for submitting a partial with difficulty X, which means that points scale linearly with difficulty.
For example, 100 TiB of space should yield approximately 10,000 points per day, whether the difficulty is set to
100 or 200. It should not matter what difficulty is set for a farmer, as long as they are consistently submitting partials.
The specification does not require pools to pay out proportionally by points, but the payout scheme should be clear to
farmers, and points should be acknowledged and accumulated points returned in the response.

### Difficulty adjustment algorithm
이것은 풀에서 실행하는 간단한 난이도 조정 알고리즘입니다. 풀은 또한이를 개선하거나 원하는대로 변경할 수 있습니다. 농부는 자신의 `suggested_difficulty`를 제공 할 수 있으며 풀은 해당 농부의 난이도 업데이트 여부를 결정할 수 있습니다. 난이도 또는 풀 지불 정보를 설정할 때 최신 authentication_public_key 만 허용하도록주의하십시오. 초기 참조 클라이언트 및 풀은`suggested_difficulty`를 사용하지 않습니다.

-이 런처 ID에 대해 마지막으로 성공한 부분 획득
-3 시간 이상이면 난이도를 5로 나눕니다.
-45 분 초과 6 시간 미만인 경우 난이도를 1.5로 나눕니다.
-45 분 미만인 경우 :
   -이 난이도에서 부분이 300 개 미만이면 동일한 난이도 유지
   -그렇지 않으면 난이도에 (24 * 3600 / (300 부분 소요 시간))을 곱하십시오.
  
6 시간은 농부의 저장고가 급격히 떨어지는 드문 경우를 처리하는 데 사용됩니다. 45 분은 비슷하지만 덜 극단적 인 경우입니다. 마지막으로 45 분 미만의 마지막 경우는 공간을 늘리거나 공간을 약간 줄인 사용자를 적절히 처리해야합니다. 이는 하루에 300 개의 부분을 대상으로하지만 성능 및 사용자 선호도에 따라 다른 숫자를 사용할 수 있습니다.
### Making payments
지급 정보는 각 부분과 함께 제공됩니다. 사용자는 보상이 지급되는 위치를 선택할 수 있으며 XCH 주소일 필요는 없습니다. 풀은 해당 launcher_id에 대해 가장 최근에 확인 된 인증 키로 성공적인 부분에 대한 지불 정보 만 업데이트해야합니다.
### Install and run (Testnet)
풀을 돌리기 위해선 반드시 `chia-blockchain`의 메인브랜치를 사용해야 합니다.

1. Checkout the `pools.dev` branch of `chia-blockchain`, and install it. Checkout this repo in another
directory next to (not inside) `chia-blockchain`. Make sure to be on testnet by doing `export CHIA_ROOT=".chia/testnet7"` and `chia configure --testnet true`.

2. Create three keys, one which will be used for the block rewards from the blockchain, one to receive the pool fee that is kept by the pool, and the third to be a wallet that acts as a test user.

3. Change the `wallet_fingerprint` and `wallet_id` in the `config.yaml` constructor, using the information from the first
key you created in step 2. These can be obtained by doing `chia wallet show`.

4. Do `chia keys show` and get the first address for each of the keys created in step 2. Put these into the `config.yaml`
config file in `default_target_address` and `pool_fee_address` respectively.
   
5. Change the `pool_url` in `config.yaml` to point to your external ip or hostname. 
   This must match exactly with what the user enters into their UI or CLI, and must start with https://.
   
6. Start the node using `chia start farmer`, and log in to a different key (not the two keys created for the pool). 
This will be referred to as the farmer's key here. Sync up your wallet on testnet for the farmer key. 
You can log in to a key by running `chia wallet show` and then choosing each wallet in turn, to make them start syncing.

7. Create a venv (different from chia-blockchain) and start the pool server using the following commands:

```
cd pool-reference
python3 -m venv ./venv
source ./venv/bin/activate
pip install ../chia-blockchain/ 
sudo CHIA_ROOT="/your/home/dir/.chia/testnet9" ./venv/bin/python -m pool
```

You should see something like this when starting, but no errors:
```
INFO:root:Logging in: {'fingerprint': 2164248527, 'success': True}
INFO:root:Obtaining balance: {'confirmed_wallet_balance': 0, 'max_send_amount': 0, 'pending_change': 0, 'pending_coin_removal_count': 0, 'spendable_balance': 0, 'unconfirmed_wallet_balance': 0, 'unspent_coin_count': 0, 'wallet_id': 1}
```

8. Create a pool nft (on the farmer key) by doing `chia plotnft create -s pool -u http://127.0.0.1:80`, or whatever host:port you want
to use for your pool. Approve it and wait for transaction confirmation. This url must match *exactly* with what the 
   pool uses.
   
9. Do `chia plotnft show` to ensure that your plotnft is created. Now start making some plots for this pool nft.
You can make plots by specifying the -c argument in `chia plots create`. Make sure to *not* use the `-p` argument. The 
    value you should use for -c is the `P2 singleton address` from `chia plotnft show` output.
 You can start with small k25 plots and see if partials are submitted from the farmer to the pool server. The output
will be the following in the pool if everything is working:
```
INFO:root:Returning {'new_difficulty': 1963211364}, time: 0.017535686492919922 singleton: 0x1f8dab79a614a82f9834c8f395f5fe195ae020807169b71a10218b9788a7a573
```
    
위에 설명 된 9 단계에 대한 질문이있는 경우 keybase의 @ sorgente711로 메시지를 보내주십시오. 기타 모든 질문
keybase의 #pools 채널로 보내야합니다.