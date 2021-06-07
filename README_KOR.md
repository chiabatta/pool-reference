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

The pool periodically searches the blockchain for new pool rewards (1.75 XCH) that go to the various
`p2_singleton_puzzle_hashes` of each of the farmers. These coins are locked, and can only be spent if they are spent
along with the singleton that they correspond to. The singleton is also locked to a `target_puzzle_hash`, which in
this diagram is the red pool address. Anyone can spend the singleton and the `p2_singleton_puzzle_hash` coin, as 
long as it's a block reward, and all the conditions are met. Some of these conditions require that the singleton
always create exactly 1 new child singleton with the same launcher id, and that the coinbase funds are sent to the 
`target_puzzle_hash`.

### Calculating farmer rewards

Periodically (for example once a day), the pool executes the code in `create_payment_loop`. This first sums up all the 
confirmed funds in the pool that have a certain number of confirmations.

Then, the pool divides the total amount by the points of all pool members, to obtain the `mojo_per_point` (minus the pool fee
and the blockchain fee). A new coin gets created for each pool member (and for the pool), and the payments are added
to the pending_payments list. Note that since blocks have a maximum size, we have to limit the size of each transaction.
There is a configurable parameter: `max_additions_per_transaction`. After adding the payments to the pending list,
the pool members' points are all reset to zero. This logic can be customized.


### Difficulty adjustment algorithm
This is a simple difficulty adjustment algorithm executed by the pool. The pool can also improve this or change it 
however they wish. The farmer can provide their own `suggested_difficulty`, and the pool can decide whether or not
to update that farmer's difficulty. Be careful to only accept the latest authentication_public_key when setting
difficulty or pool payout info. The initial reference client and pool do not use the `suggested_difficulty`.

- Obtain the last successful partial for this launcher id
- If > 3 hours, divide difficulty by 5
- If > 45 minutes < 6 hours, divide difficulty by 1.5
- If < 45 minutes:
   - If have < 300 partials at this difficulty, maintain same difficulty
   - Else, multiply the difficulty by (24 * 3600 / (time taken for 300 partials))
  
The 6 hours is used to handle rare cases where a farmer's storage drops dramatically. The 45 minutes is similar, but
for less extreme cases. Finally, the last case of < 45 minutes should properly handle users with increasing space,
or slightly decreasing space. This targets 300 partials per day, but different numbers can be used based on
performance and user preference.

### Making payments
Note that the payout info is provided with each partial. The user can choose where rewards are paid out to, and this
does not have to be an XCH address. The pool should ONLY update the payout info for successful partials with the
latest seen authentication key for that launcher_id.


### Install and run (Testnet)
To run a pool, you must use this along with a branch of `chia-blockchain`.

1. Checkout the `pools.2021-may-25` branch of `chia-blockchain`, and install it. Checkout this repo in another
directory next to (not inside) `chia-blockchain`. Make sure to be on testnet by doing `export CHIA_ROOT=".chia/testnet7"` and `chia configure --testnet true`.

2. Create two keys, one which will be used for the block rewards from the blockchain, and the other
which will receive the pool fee that is kept by the pool.

3. Change the `wallet_fingerprint` and `wallet_id` in the `pool.py` constructor, using the information from the first
key you created in step 2. These can be obtained by doing 'chia wallet show'.

4. Do `chia keys show` and get the first address for each of the keys created in step 2. Put these into the `pool.py`
file in `default_target_puzzle_hash` and `pool_fee_puzzle_hash` respectively.
   
5. Change the pool_url in pool.py to point to your external ip or hostname.

6. Start the node using `chia start farmer`, and log in to a different key (not the two keys created for the pool). 
This will be referred to as the farmer's key here. Sync up your wallet on testnet for the farmer key. 

7. Create a venv (different from chia-blockchain) and start the pool server using the following commands:

```
cd pool-reference
python3 -m venv ./venv
source ./venv/bin/activate
pip install ../chia-blockchain/ 
sudo CHIA_ROOT="/your/home/dir/.chia/testnet7" ./venv/bin/python pool/pool_server.py
```

8. You should see something like this when starting, but no errors:
```
INFO:root:Logging in: {'fingerprint': 2164248527, 'success': True}
INFO:root:Obtaining balance: {'confirmed_wallet_balance': 0, 'max_send_amount': 0, 'pending_change': 0, 'pending_coin_removal_count': 0, 'spendable_balance': 0, 'unconfirmed_wallet_balance': 0, 'unspent_coin_count': 0, 'wallet_id': 1}
```

9. Create a pool nft (on the farmer key) by doing `chia poolnft create -u 127.0.0.1:80`, or whatever host:port you want
to use for your pool. Approve it and wait for transaction confirmation.
   
10. Do `chia poolnft show` to ensure that your poolnft is created. Now start making some plots for this pool nft.
You can make plots by specifying the -c argument in `chia plots create`. Make sure to *not* use the `-p` argument. The 
    value you should use for -c is the `P2 singleton address` from `chia poolnft show` output.
 You can start with small k25 plots and see if partials are submitted from the farmer to the pool server. The output
will be the following in the pool if everything is working:
```
INFO:root:Returning {'points_balance': 82629918227, 'current_difficulty': 1963211364}, time: 0.017535686492919922 singleton: 0x1f8dab79a614a82f9834c8f395f5fe195ae020807169b71a10218b9788a7a573
```
    
Note that claiming rewards and switching pools are still not enabled, but these will be added very shortly. Please
send a message to @sorgente711 on keybase if you have questions about the 10 steps explained above. All other questions
should be send to the #pools channel in keybase. Note that there will probably be breaking changes soon which will
require re-plotting and re-running all the steps above.


