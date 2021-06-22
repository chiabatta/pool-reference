이 프로토 타입은 아직 지원되지 않으며 아직 개발 중입니다. 
이 코드는 Apache 2.0 라이선스에 따라 제공됩니다. 
참고 : 초안 사양은 SPECIFICATION.md 파일에 있습니다.

### Customizing
커스터마이징 가능한 것들
* 풀에서 나가기위한 제한 시간
* 난이도 조정 방법
* 취할 수수료 및 블록 체인 수수료로 지불 할 금액
* 지불시 farmer의 포인트가 계산되는 방법 (PPS, PPLNS 등)
* farmer가 지불금을받는 방법 (XCH, BTC, ETH 등) 및 빈도

그러나 일부는 변경할 수 없습니다. 이는 SPECIFICATION.md에 설명되어 있으며 주로 유효성 검사, 프로토콜 및 스마트 코인의 싱글톤 형식과 관련이 있습니다.

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
풀을 돌리기 위해선 반드시 `chia-blockchain`과 더불어 아래의 branch들을 사용해야 합니다.

1. Checkout the `pools.dev` branch of `chia-blockchain`, and install it. Checkout this repo in another
directory next to (not inside) `chia-blockchain`. Make sure to be on testnet by doing `export CHIA_ROOT=".chia/testnet7"` and `chia configure --testnet true`.

2. Create three keys, one which will be used for the block rewards from the blockchain, one to receive the pool fee that is kept by the pool, and the third to be a wallet that acts as a test user.

3. Change the `wallet_fingerprint` and `wallet_id` in the `config.yaml` constructor, using the information from the first
key you created in step 2. These can be obtained by doing `chia wallet show`.

4. Do `chia keys show` and get the first address for each of the keys created in step 2. Put these into the `config.yaml`
config file in `default_target_address` and `pool_fee_address` respectively.
   
5. Change the `pool_url` in `config.yaml` to point to your external ip or hostname. 
   This must match exactly with what the user enters into their UI or CLI, and must start with https://. For now
   http:// can also be used.
   
6. Start the node using `chia start farmer`, and log in to a different key (not the two keys created for the pool). 
This will be referred to as the farmer's key here. Sync up your wallet on testnet for the farmer key. 
You can log in to a key by running `chia wallet show` and then choosing each wallet in turn, to make them start syncing.

7. Create a venv (different from chia-blockchain) and start the pool server using the following commands:

```
cd pool-reference
python3 -m venv ./venv
source ./venv/bin/activate
pip install ../chia-blockchain/ 
sudo CHIA_ROOT="/your/home/dir/.chia/testnet7" ./venv/bin/python pool/pool_server.py
```

You should see something like this when starting, but no errors:
```
INFO:root:Logging in: {'fingerprint': 2164248527, 'success': True}
INFO:root:Obtaining balance: {'confirmed_wallet_balance': 0, 'max_send_amount': 0, 'pending_change': 0, 'pending_coin_removal_count': 0, 'spendable_balance': 0, 'unconfirmed_wallet_balance': 0, 'unspent_coin_count': 0, 'wallet_id': 1}
```

8. Create a pool nft (on the farmer key) by doing `chia plotnft create -u http://127.0.0.1:80`, or whatever host:port you want
to use for your pool. Approve it and wait for transaction confirmation. This url must match *exactly* with what the 
   pool uses.
   
9. Do `chia plotnft show` to ensure that your plotnft is created. Now start making some plots for this pool nft.
You can make plots by specifying the -c argument in `chia plots create`. Make sure to *not* use the `-p` argument. The 
    value you should use for -c is the `P2 singleton address` from `chia plotnft show` output.
 You can start with small k25 plots and see if partials are submitted from the farmer to the pool server. The output
will be the following in the pool if everything is working:
```
INFO:root:Returning {'current_difficulty': 1963211364}, time: 0.017535686492919922 singleton: 0x1f8dab79a614a82f9834c8f395f5fe195ae020807169b71a10218b9788a7a573
```
    
위에 설명 된 9 단계에 대한 질문이있는 경우 keybase의 @ sorgente711로 메시지를 보내주십시오. 기타 모든 질문
keybase의 #pools 채널로 보내야합니다.