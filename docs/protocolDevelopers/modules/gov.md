---
sidebar_position: 9
---

# Gov

## Introduction

gov 模块负责链上资产抵押、委托等操作的处理，这些操作通过消息MsgCreateValidator、MsgEditValidator、MsgDelegate、MsgBeginRedelegate完成。

* MsgCreateValidator: 抵押链上资产并创建验证者。
* MsgEditValidator: 修改验证者的参数。
* MsgDelegate: 将链上资产委托给某个验证者。
* MsgBeginRedelegate: 重新委托。
* MsgUndelegate: 撤回委托。

gov模块的Keeper为上述5个消息分别定义了相应的处理函数。而staking模块的所有逻辑都围绕验证者结构体Validator和委托结构体Delegation展开。前者记录验证者信息，后者记录委托操作信息。

```golang
type Validator struct {
	OperatorAddress string                                  //验证者节点地址
	ConsensusPubkey *types1.Any                             //验证者的共识公钥
	Jailed bool                                             //验证者是否处于监禁惩罚
	Status BondStatus                                       //验证者状态
	Tokens github_com_cosmos_cosmos_sdk_types.Int           //质押链上资产数量
	DelegatorShares github_com_cosmos_cosmos_sdk_types.Dec  //分配给验证者的委托人份额总量
	Description Description                                 //验证者描述信息
	UnbondingHeight int64                                   //验证者开始解绑周期的区块高度
	UnbondingTime time.Time                                 //验证者完成解绑周期的最早时间
	Commission Commission                                   //验证者的佣金
	MinSelfDelegation github_com_cosmos_cosmos_sdk_types.Int//验证者声明的最小自抵押量
	TatTokens github_com_cosmos_cosmos_sdk_types.Int        //质押TAT的链上资产
	NewTokens github_com_cosmos_cosmos_sdk_types.Int        //验证者TAT和UNIT的资产总量
	TatPower github_com_cosmos_cosmos_sdk_types.Int         //分配给验证者TAT的委托人份额总量
	NewUnitPower github_com_cosmos_cosmos_sdk_types.Int     //分配给验证者(TAT+UNIT)的委托人份额总量
}
```

```golang
type Delegation struct {
	DelegatorAddress string                                 //委托者的地址
	ValidatorAddress string                                 //验证者的地址
	Shares github_com_cosmos_cosmos_sdk_types.Dec           //收到的委托份额(UNIT)
	TatShares github_com_cosmos_cosmos_sdk_types.Dec        //收到的委托份额(TAT)
}
```

根据Cosmos Hub中POS机制，链上资产委托人与验证者节点运营方共同承担收益和风险。收益分发与提取的逻辑有distribution模块处理(参见[distribution模块](./distribution.md)),以链上资产的具体数量为指标进行计算;而链上惩罚是直接扣除固定比例的抵押链上资产。由此验证者代理的链上资产总量以及相关的委托操作涉及的链上资产数量会随着惩罚事件的发生而减少。


## 重新委托与撤回奖励

重新委托操作会涉及三方: 委托人(delegator)、源验证者(source validator)和目标验证者(destination validator)。
撤回委托操作涉及两方: 委托人和验证者

这两项操作不是立即完成的，都需要时间。撤回委托操作涉及的链上资产在等待成熟期间没有任何收益，而重新委托所涉及的链上资产即使相应操作没有成熟也可以参与目标验证者的收益分成，目标验证者的投票权重会立即增加，委托人也会立即有收益。
之所以需要时间，原因在于作恶行为发生与执行链上惩罚之间存在时间差。如果允许两个操作立即完成，并且该操作发生在验证者作恶和惩罚之前，则这些曾经赋予作恶验证者投票权重的抵押链上资产可以逃避惩罚。(参见[slashing模块](./slashing.md))

撤回委托操作的成熟时间与验证者的状态无关，无论验证者处于什么状态，撤回委托操作都需要等待完整的解绑周期才可以成熟。如果重新委托操作正在等待成熟，则不允许将相关的链上资产再次委托。重新委托操作的成熟时间与原验证者的状态有关。

* 对Bonded状态的源验证者发起重新委托，成熟时间为完整的解绑周期。
* 对Unbonding状态的源验证者发起重新委托，成熟时间为源验证者的解绑周期结束时间。
* 对Unbonded状态的源验证者发起重新委托，无需等待立即成熟。

## 验证者状态切换

Treasurenetd中的Validator可以有三种状态Unbonded、Unbonding和Bonded。
通过Msg-CreateValidator消息创建的新验证者被初始化成Unbonded状态，并被设置份额以及投票权重。staking模块的EndBlocker()会统计本区块验证者状态的变化

* 新创建的Validator的投票权重排名进入前100名:状态从Unbonded变成Bonded。
* 新创建的Validator的投票权重排名没有进入前100名:Unbonded状态维持不变。
* 投票权重增加且投票权重排名进入前100名时的状态切换
  - Unbonded --> Bonded: 初次成为活跃验证者。
  - Unbonding --> Bonded: 再次成为活跃验证者。
  - Bonded维持不变: 已经是活跃验证者。

```sequence
节点->Unbonded: 发送交易创建验证者
Unbonded->Bonded: 投票权重排名上升
Bonded->Unbonding: 投票权重排名下降
Unbonding->Unbonded: 解绑周期结束
Bonded->UnbondingJailed: 可用性差或者自抵押链上资产太少
UnbondingJailed->Bonded: 禁闭时间结束或具有足量自抵押的链上资产
Unbonding->UnbongdingJailed: 可用性差或者自抵押的链上资产太少
UnbongdingJailed->Unbonding: 禁闭时间结束或具有足量的自抵押链上资产
```

## parameter

* unbonding_time: 解绑的持续时间;
* max_validators: 验证者的最大数量;
* max_entries: 解除绑定委托或重新委托的最大条目数;
* historical_entries: 保留的历史条目数;
* bond_denom: 质押硬币的面额。

## Validator
Validators are responsible for signing or proposing a block at each consensus round. It is important that the validators maintain excellent availability and network connectivity to perform their tasks. To incentivise the validator nodes to run the network, rewards are distributed to the validators according to their performance and amount of staked tokens (see [distribution](./distribution.md) and [mint](./mint.md)). On the other hand, a penalty should be imposed on validators' misbehaviour (see [slashing](./slashing.md)).

## Delegator
The staking module enables CRO owners to delegate their tokens to active validators and share part of the reward obtained by the validator during the proof of stake protocol(see [distribution module](./distribution.md)). Specifically, it allows token owners to take part in the consensus process without running a validator themselves.

It is important to point out that the delegator and the validator are on the same boat: they share the reward and the risk. In particular, part of their delegated token could be slashed due to validator's misbehaviour (see [slashing](./slashing.md)). Therefore, it is very important to choose a reliable validator to delegate to. Kindly refer to this link for detailed specification and state transitions of delegation.

## Transactions and Queries

### Transactions

   > treasurenetd tx staking create-validator - Create new validator initialized with a self-delegation

```sh
$ treasurenetd tx staking create-validator \
--from=[name_of_your_key] \
--amount=[staking_amount] \
--pubkey=[treasurenetpub...]  \
--moniker="[moniker_id_of_your_node]" \
--security-contact="[security contact email/contact method]" \
--chain-id="[chain-id]" \
--commission-rate="[commission_rate]" \
--commission-max-rate="[maximum_commission_rate]" \
--commission-max-change-rate="[maximum_rate_of_change_of_commission]" \
--min-self-delegation="[min_self_delegation_amount]"
--keyring-backend test

## Transactions payload##
{"body":{"messages":[{"@type":"/cosmos.staking.v1beta1.MsgCreateValidator"...}
confirm transaction before signing and broadcasting [y/N]: y
```

   > treasurenetd tx staking delegate [validator-address] [amount] - Delegate liquid tokens to a validator

```sh
treasurenetd tx staking delegate treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq 10unit
--from treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg 
--home (defaule:"/root/.treasurenet/")  
--fees 1unit 
--gas auto
--keyring-backend test

## Transactions payload##
{"body":{"messages":[{"@type":"/cosmos.staking.v1beta1.MsgDelegate","delegator_address":"treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg","validator_address":"treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq","amount":{"denom":"aunit","amount":"10000000000000000000"}}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extension_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[{"denom":"aunit","amount":"1000000000000000000"}],"gas_limit":"214201","payer":"","granter":""}},"signatures":[]}

confirm transaction before signing and broadcasting [y/N]: y
```

   >  treasurenetd tx staking unbond [validator-address] [amount] -  Unbond shares from a validator
这里需要注意的是，在解绑委托后，不会立即生效，因为我们有个参数unbonding_time(解绑时间)，资金只有在unbonding_time通过后才能生效

```sh
treasurenetd tx staking unbond treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq 10unit 
--home (defaule:"/root/.treasurenet/")
--from treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg
--chain-id treasurenet_9000-1
--fees 1unit
--gas auto
--keyring-backend test

## Transactions payload##
{"body":{"messages":[{"@type":"/cosmos.staking.v1beta1.MsgUnDelegate","delegator_address":"treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg","validator_address":"treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq","amount":{"denom":"aunit","amount":"10000000000000000000"}}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extension_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[{"denom":"aunit","amount":"1000000000000000000"}],"gas_limit":"214201","payer":"","granter":""}},"signatures":[]}

confirm transaction before signing and broadcasting [y/N]: y
```

   > treasurenetd tx staking redelegate [validator-address] [validator-address2] [amount] - 将代币从一个验证者重新委托给另一个验证者
重新绑定操作需要注意几个方面:
1. 当同意用户在解绑过程中，有重新进行了委托绑定，需要等该账户解绑结束后才能进行委托绑定
2. 重新授权期间没有unbonding time ,所以不会错过奖励，但是每个验证者只能重新委托一次，直到unbonding time 结束才能进行新的重新委托
3. Max_entries表示解除绑定委托活重新委托的最大条目，我们重新委托需要在这个参数范围内，如果请求量过大会报错 “too many unbonding delegation entries in this delegator/validator duo, please wait for some entries to mature”

```sh
treasurenetd tx staking redelegate treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq treasurenetvaloper2as78dmzhesjndy3v6wsdxjfqnmwnyy2gzs32qq 10unit 
--home (defaule:"/root/.treasurenet/")
--from treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg
--chain-id treasurenet_9000-1
--fees 1unit
--gas auto
--keyring-backend test

## Transactions payload##
{"body":{"messages":[{"@type":"/cosmos.staking.v1beta1.MsgBeginRedelegate","delegator_address":"treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg","validator_src_address":"treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq","validator_dst_address":"treasurenetvaloper2as78dmzhesjndy3v6wsdxjfqnmwnyy2gzs32qq","amount":{"denom":"aunit","amount":"10000000000000000000"}}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extension_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[{"denom":"aunit","amount":"1000000000000000000"}],"gas_limit":"214201","payer":"","granter":""}},"signatures":[]}

confirm transaction before signing and broadcasting [y/N]: y
```

### Queries

   > treasurenetd query staking validators --home --output json | jq - 查询所有验证者

```json
{
  "validators": [
    {
      "operator_address": "treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq",
      "consensus_pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "dGvx6FL1zdjKsmzZ7R/2EBfCgJcsneP0rUpMkxs9Si8="
      },
      "jailed": false,
      "status": "BOND_STATUS_BONDED",
      "tokens": "268000000000000000000",
      "delegator_shares": "268000000000000000000.000000000000000000",
      "description": {
        "moniker": "localtestnet",
        "identity": "",
        "website": "",
        "security_contact": "",
        "details": ""
      },
      "unbonding_height": "0",
      "unbonding_time": "1970-01-01T00:00:00Z",
      "commission": {
        "commission_rates": {
          "rate": "0.100000000000000000",
          "max_rate": "0.200000000000000000",
          "max_change_rate": "0.010000000000000000"
        },
        "update_time": "2023-02-02T10:48:47.611931848Z"
      },
      "min_self_delegation": "158000000000000000000",
      "tat_tokens": "0",
      "new_tokens": "0",
      "tat_power": "0",
      "newunit_power": "0"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "0"
  }
}

```

   > treasurenetd query staking validator [validator-address] --home --output json | jq - 查看质押validator的情况

```json
{
  "operator_address": "treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq",
  "consensus_pubkey": {
    "@type": "/cosmos.crypto.ed25519.PubKey",
    "key": "dGvx6FL1zdjKsmzZ7R/2EBfCgJcsneP0rUpMkxs9Si8="
  },
  "jailed": false,
  "status": "BOND_STATUS_BONDED",
  "tokens": "268000000000000000000",
  "delegator_shares": "268000000000000000000.000000000000000000",
  "description": {
    "moniker": "localtestnet",
    "identity": "",
    "website": "",
    "security_contact": "",
    "details": ""
  },
  "unbonding_height": "0",
  "unbonding_time": "1970-01-01T00:00:00Z",
  "commission": {
    "commission_rates": {
      "rate": "0.100000000000000000",
      "max_rate": "0.200000000000000000",
      "max_change_rate": "0.010000000000000000"
    },
    "update_time": "2023-02-02T10:48:47.611931848Z"
  },
  "min_self_delegation": "158000000000000000000",
  "tat_tokens": "0",
  "new_tokens": "0",
  "tat_power": "0",
  "newunit_power": "0"
}

```

   > treasurenetd query staking delegations [delegator-address] --home --output json | jq - 根据地址查询委托详情

```json
{
  "delegation_responses": [
    {
      "delegation": {
        "delegator_address": "treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg",
        "validator_address": "treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq",
        "shares": "268000000000000000000.000000000000000000",
        "tat_shares": "0"
      },
      "balance": {
        "denom": "aunit",
        "amount": "268000000000000000000"
      }
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "0"
  }
}

```

   > treasurenetd query staking delegations-to [validator-address] --home --output json | jq - 查询一个验证者的所有委托

```json
{
  "delegation_responses": [
    {
      "delegation": {
        "delegator_address": "treasurenet1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2grwxmrg",
        "validator_address": "treasurenetvaloper1wf78qmzhfsjndy3v6wsdxjfqnmwnyy2gzs46zq",
        "shares": "268000000000000000000.000000000000000000",
        "tat_shares": "0"
      },
      "balance": {
        "denom": "aunit",
        "amount": "268000000000000000000"
      }
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "0"
  }
}
```