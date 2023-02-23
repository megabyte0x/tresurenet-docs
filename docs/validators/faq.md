---
sidebar_position: 4
---

# Validator FAQ

:::note
Check the FAQ for running a validator on TreasureNet
:::

## Introduce

### What is a validator?

[TreasuerNet 基于Tendermint](https://docs.tendermint.com/main/introduction/what-is-tendermint.html)它依赖于一组负责在区块链中提交新块的验证器。这些验证者通过广播包含由每个验证者的私钥签名的加密签名的投票来参与共识协议。

验证者候选者可以绑定他们自己的 UNIT或者TAT，并让代币持有者将UNIT “委托”或质押给他们。TreasureNet 目前只允许100个validator参与共识，但随着时间的推移，可以通过治理提案增加验证器的数量。验证者由委托给他们的 UNIT代币总数和TAT一起决定([参见活跃验证者的选取规则](./faq.md))，投票权最高的前 100 名验证者候选人是当前的活跃验证者参与共识生成新的区块。

验证者及其委托人通过执行 Tendermint 共识协议赚取 Unit 作为区块条款和代币作为交易费用。请注意，验证者可以为其委托人收取的费用设置佣金百分比作为额外奖励。

如果验证者双重签名或长时间离线，他们质押的 UNIT（包括委托给他们的用户的 UNIT）可能会被罚没。处罚取决于违规的严重程度。

### Active Validator的选取规则

1. 每个节点要想成为Validator必须自抵押UNIT，且验证者的自我委托永远不能低于 min-self-delegation(最小自抵押,默认为158unit)。
2. 第一轮筛选满足min-self-delegation的validator，取权重最高的前400个validator进行选取。
   - 通过event监听bid操作，获取质押TAT的validator为list-supervalidator。
   - 第一步中没有质押TAT的validator称为list-validator。
3. 判断Active Validator(活跃验证者)的数量:
   - 如果list-supervalidator + list-validator >= 200,Active Validator的数量 num = 100。
   - 如果list-supervalidator + listvalidator < 200,Actibr Validator的数量 num = (list-supervalidator + listvalidator ) / 2。
4. 第二轮筛选
   - 当list-supervalidator < 100, 且listvalidator > 100, All-list-super = list-supervalidator (选取全部的list-supervalidator作为参考值), num-validator = 2 * num - num(All-list-super), 从list-validator中选取权重前num-validator，组成新的All-list-validator。
   - 当list-supervalidator >= 100 且 list-validator >= 100, 从list-supervalidator中选取前100组成新的All-list-super, 从list-validator中选取权重前100，组成新的All-list-validator。
   - list-supervalidator < 100 且 list-validator < 100, All-list-super = list-supervalidator, All-list-validator = list-validator。
   - 当list-supervalidator > 100, 且listvalidator < 100, All-list-validator = list-validator (选取全部的list-validator作为参考值), num-super = 2 * num - num(All-list-validator), 从list-supervalidator中选取权重前num-super，组成新的All-list-super。
5. 第三轮筛选，组成新的Active Validator进行接下来的pos运算
   - 将All-list-super和All-list-validator组成新的列表list，然后随机选取其中的num作为Active Validator

### What is "staking"?

TreasureNet is a public Proof-of-Stake (PoS) blockchain, meaning that validator's weight is determined by the amount of staking tokens (UNIT) bonded as collateral. These staking tokens can be staked directly by the validator or delegated to them by UNIT holders.

Any user in the system can declare its intention to become a validator by sending a [create-validator](./faq.md) transaction. From there, they become validators.

验证者的权重（即总权益或投票权）决定了它是否是一个活跃的验证者，也决定了该节点提出区块的频率以及它将获得多少收益。根据active Validator选取规则选出合适的validator作为活跃验证者。如果验证者双重签名或经常离线，他们将冒着被抵押的代币（包括用户委托的 UNIT）被协议“罚没”的风险，以惩罚疏忽和不当行为。

### What is a full node?
A full node is a program that fully validates transactions and blocks of a blockchain. It is distinct from a light client node that only processes block headers and a small subset of transactions. Running a full node requires more resources than a light client but is necessary in order to be a validator. In practice, running a full-node only implies running a non-compromised and up-to-date version of the software with low network latency and without downtime.

Of course, it is possible and encouraged for any user to run full nodes even if they do not plan to be validators.

### What is a delegator?

不能或不想操作验证节点的人仍然可以作为委托人参与质押过程。事实上，验证者的选择不是基于他们自己委托的股份，而是基于他们的总股份，即他们自己委托的股份和委托给他们的股份的总和。这是一个重要的属性，因为它使委托人可以防止验证者表现出不良行为。如果验证者行为不端，他们的委托人会将他们的 UNIT 移离他们，从而减少他们的股份。最终，通过[Active Validator的选取规则](./faq.md)，他们将退出验证者集。

**委托人分享他们验证者的收入，但他们也分担风险。** 在收入方面，验证人和委托人的不同之处在于，验证人可以对分配给委托人的收入收取佣金。该佣金事先为委托人所知，并且只能根据预定义的约束进行更改([请参阅staking](../protocolDevelopers/modules/staking.md))。就风险而言，如果验证者行为不当，委托人的 UNIT 可能会被削减。有关更多信息，请参阅[slashing](../protocolDevelopers/modules/slashing.md)。

要成为委托人，UNIT 持有者需要发送一个“委托交易”，在其中指定他们想要绑定多少个 UNIT 以及与哪个验证者绑定。候选验证者列表将显示在 TreasutreNet 浏览器中。之后，如果委托人想要解绑部分或全部股份，他们需要发送“解绑交易”。从那里开始，委托人将不得不等待 3 周才能取回他们的 UNIT。委托人还可以发送“重新绑定交易”以从一个验证器切换到另一个验证器，而无需经过 3 周的等待期([请参阅重新委托和解除绑定](../protocolDevelopers/modules/staking.md))。

### Becoming a Validator
#### How to become a validator?
Any participant in the network can signal their intent to become a validator by creating a validator and registering its validator profile. To do so, the candidate broadcasts a create-validator transaction, in which they must submit the following information:

* Validator's PubKey: Validator operators can have different accounts for validating and holding liquid funds. The PubKey submitted must be associated with the private key with which the validator intends to sign prevotes and precommits.
* Validator's Address: treasurenetvaloper1- address. This is the address used to identify your validator publicly. The private key associated with this address is used to bond, unbond, and claim rewards.
* Validator's name (also known as the moniker)
* Validator's website (optional)
* Validator's description (optional)
* Initial commission rate: The commission rate on block provisions, block rewards and fees charged to delegators.
* Maximum commission: The maximum commission rate which this validator will be allowed to charge.
* Commission change rate: The maximum daily increase of the validator commission.
* Minimum self-bond amount: Minimum amount of UNIT the validator needs to have bonded at all times. If the validator's self-bonded stake falls below this limit, its entire staking pool will be unbonded.
* Initial self-bond amount: Initial amount of UNIT the validator wants to self-bond.
```shell
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

:::caution
   ❗️ DANGER: Never create your mainnet validator keys using a test keying backend. Doing so might result in a loss of funds by making your funds remotely accessible via the eth_sendTransaction JSON-RPC endpoint.

   Ref: [Security Advisory: Insecurely configured geth can make funds remotely accessible](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

Once a validator is created and registered, EVMOS holders can delegate EVMOS to it, effectively adding stake to its pool. The total stake of a validator is the sum of the EVMOS self-bonded by the validator's operator and the EVMOS bonded by external delegators.

Only the top 150 validators with the most stake are considered the active validators, becoming bonded validators. If ever a validator's total stake dips below the top 150, the validator loses its validator privileges (meaning that it won't generate rewards) and no longer serves as part of the active set (i.e doesn't participate in consensus), entering unbonding mode and eventually becomes unbonded.

#Validator keys and states
#What are the different types of keys?
In short, there are two types of keys:

Tendermint Key: This is a unique key used to sign block hashes. It is associated with a public key evmosvalconspub.

Generated when the node is created with evmosd init.
Get this value with evmosd tendermint show-validator e.g. evmosvalconspub1zcjduc3qcyj09qc03elte23zwshdx92jm6ce88fgc90rtqhjx8v0608qh5ssp0w94c
Application keys: These keys are created from the application and used to sign transactions. As a validator, you will probably use one key to sign staking-related transactions, and another key to sign oracle-related transactions. Application keys are associated with a public key evmospub- and an address evmos-. Both are derived from account keys generated by evmosd keys add.

A validator's operator key is directly tied to an application key, but uses reserved prefixes solely for this purpose: evmosvaloper and evmosvaloperpub

#What are the different states a validator can be in?
After a validator is created with a create-validator transaction, it can be in three states:

bonded: Validator is in the active set and participates in consensus. Validator is earning rewards and can be slashed for misbehaviour.
unbonding: Validator is not in the active set and does not participate in consensus. Validator is not earning rewards, but can still be slashed for misbehaviour. This is a transition state from bonded to unbonded. If validator does not send a rebond transaction while in unbonding mode, it will take two weeks for the state transition to complete.
unbonded: Validator is not in the active set, and therefore not signing blocks. Unbonded validators cannot be slashed, but do not earn any rewards from their operation. It is still possible to delegate EVMOS to this validator. Un-delegating from an unbonded validator is immediate.
Delegators have the same state as their validator.

Delegations are not necessarily bonded. EVMOS can be delegated and bonded, delegated and unbonding, delegated and unbonded, or liquid.

#What is "self-bond"? How can I increase my "self-bond"?
The validator operator's "self-bond" refers to the amount of EVMOS stake delegated to itself. You can increase your self-bond by delegating more EVMOS to your validator account.

#Is there a testnet faucet?
If you want to obtain coins for the testnet, you can do so by using the faucet (opens new window).

#Is there a minimum amount of EVMOS that must be staked to be an active (bonded) validator?
There is no minimum. The top 150 validators with the highest total stake (where total stake = self-bonded stake + delegators stake) are the active validators.

#How will delegators choose their validators?
Delegators are free to choose validators according to their own subjective criteria. That said, criteria anticipated to be important include:

Amount of self-bonded EVMOS: Number of EVMOS a validator self-bonded to its staking pool. A validator with higher amount of self-bonded EVMOS has more skin in the game, making it more liable for its actions.

Amount of delegated EVMOS: Total number of EVMOS delegated to a validator. A high stake shows that the community trusts this validator, but it also means that this validator is a bigger target for hackers. Validators are expected to become less and less attractive as their amount of delegated EVMOS grows. Bigger validators also increase the centralization of the network.

Commission rate: Commission applied on revenue by validators before it is distributed to their delegators

Track record: Delegators will likely look at the track record of the validators they plan to delegate to. This includes seniority, past votes on proposals, historical average uptime and how often the node was compromised.

Apart from these criteria, there will be a possibility for validators to signal a website address to complete their resume. Validators will need to build reputation one way or another to attract delegators. For example, it would be a good practice for validators to have their setup audited by third parties. Note though, that the Evmos team will not approve or conduct any audit itself.

#Responsibilites
#Do validators need to be publicly identified?
No, they do not. Each delegator will value validators based on their own criteria. Validators will be able(and are advised) to register a website address when they nominate themselves so that they can advertise their operation as they see fit. Some delegators may prefer a website that clearly displays the team running the validator and their resume, while others might prefer anonymous validators with positive track records. Most likely both identified and anonymous validators will coexist in the validator set.

#What are the responsiblities of a validator?
Validators have three main responsibilities:

Be able to constantly run a correct version of the software: validators need to make sure that their servers are always online and their private keys are not compromised.

Provide oversight and feedback on correct deployment of community pool funds: the Evmos protocol includes the a governance system for proposals to the facilitate adoption of its currencies. Validators are expected to hold budget executors to account to provide transparency and efficient use of funds.

Additionally, validators are expected to be active members of the community. They should always be up-to-date with the current state of the ecosystem so that they can easily adapt to any change.

#What does staking imply?
Staking EVMOS can be thought of as a safety deposit on validation activities. When a validator or a delegator wants to retrieve part or all of their deposit, they send an unbonding transaction. Then, the deposit undergoes a two week unbonding period during which they are liable to being slashed for potential misbehaviors committed by the validator before the unbonding process started.

Validators, and by association delegators, receive block provisions, block rewards, and fee rewards. If a validator misbehaves, a certain portion of its total stake is slashed (the severity of the penalty depends on the type of misbehavior). This means that every user that bonded EVMOS to this validator gets penalized in proportion to its stake. Delegators are therefore incentivized to delegate to validators that they anticipate will function safely.

#Can a validator run away with its delegators' EVMOS?
By delegating to a validator, a user delegates staking power. The more staking power a validator has, the more weight it has in the consensus and processes. This does not mean that the validator has custody of its delegators' EVMOS. By no means can a validator run away with its delegator's funds.

Even though delegated funds cannot be stolen by their validators, delegators are still liable if their validators misbehave. In such case, each delegators' stake will be partially slashed in proportion to their relative stake.

#How often will a validator be chosen to propose the next block? Does it go up with the quantity of EVMOS staked?
The validator that is selected to mine the next block is called the proposer, the "leader" in the consensus for the round. Each proposer is selected deterministically, and the frequency of being chosen is equal to the relative total stake (where total stake = self-bonded stake + delegators stake) of the validator. For example, if the total bonded stake across all validators is 100 EVMOS, and a validator's total stake is 10 EVMOS, then this validator will be chosen 10% of the time as the proposer.

To understand more about the proposer selection process in Tendermint BFT consensus, read more in their official docs (opens new window).

#Incentives
#What is the incentive to stake?
Each member of a validator's staking pool earns different types of revenue:

Block rewards: Native tokens of applications run by validators (e.g. EVMOS on Evmos) are inflated to produce block provisions. These provisions exist to incentivize EVMOS holders to bond their stake, as non-bonded EVMOS will be diluted over time.
Transaction fees: Evmos maintains a whitelist of token that are accepted as fee payment. The initial fee token is the evmos.
This total revenue is divided among validators' staking pools according to each validator's weight. Then, within each validator's staking pool the revenue is divided among delegators in proportion to each delegator's stake. A commission on delegators' revenue is applied by the validator before it is distributed.

#What is the incentive to run a validator ?
Validators earn proportionally more revenue than their delegators because of commissions.

Validators also play a major role in governance. If a delegator does not vote, they inherit the vote from their validator. This gives validators a major responsibility in the ecosystem.

#What is a validator's commission?
Revenue received by a validator's pool is split between the validator and its delegators. The validator can apply a commission on the part of the revenue that goes to its delegators. This commission is set as a percentage. Each validator is free to set its initial commission, maximum daily commission change rate and maximum commission. Evmos enforces the parameter that each validator sets. These parameters can only be defined when initially declaring candidacy, and may only be constrained further after being declared.

#How are block provisions distributed?
Block provisions (rewards) are distributed proportionally to all validators relative to their total stake (voting power). This means that even though each validator gains EVMOS with each provision, all validators will still maintain equal weight.

Let us take an example where we have 10 validators with equal staking power and a commission rate of 1%. Let us also assume that the provision for a block is 1000 EVMOS and that each validator has 20% of self-bonded EVMOS. These tokens do not go directly to the proposer. Instead, they are evenly spread among validators. So now each validator's pool has 100 EVMOS. These 100 EVMOS will be distributed according to each participant's stake:

Commission: 100*80%*1% = 0.8 EVMOS
Validator gets: 100\*20% + Commission = 20.8 EVMOS
All delegators get: 100\*80% - Commission = 79.2 EVMOS
Then, each delegator can claim its part of the 79.2 EVMOS in proportion to their stake in the validator's staking pool. Note that the validator's commission is not applied on block provisions. Note that block rewards (paid in EVMOS) are distributed according to the same mechanism.

#How are fees distributed?
Fees are similarly distributed with the exception that the block proposer can get a bonus on the fees of the block it proposes if it includes more than the strict minimum of required precommits.

When a validator is selected to propose the next block, it must include at least ⅔ precommits for the previous block in the form of validator signatures. However, there is an incentive to include more than ⅔ precommits in the form of a bonus. The bonus is linear: it ranges from 1% if the proposer includes ⅔rd precommits (minimum for the block to be valid) to 5% if the proposer includes 100% precommits. Of course the proposer should not wait too long or other validators may timeout and move on to the next proposer. As such, validators have to find a balance between wait-time to get the most signatures and risk of losing out on proposing the next block. This mechanism aims to incentivize non-empty block proposals, better networking between validators as well as to mitigate censorship.

Let's take a concrete example to illustrate the aforementioned concept. In this example, there are 10 validators with equal stake. Each of them applies a 1% commission and has 20% of self-bonded EVMOS. Now comes a successful block that collects a total of 1005 EVMOS in fees. Let's assume that the proposer included 100% of the signatures in its block. It thus obtains the full bonus of 5%.

We have to solve this simple equation to find the reward 
�
R for each validator:

9
�
+
�
+
5
%
(
�
)
=
1
0
0
5
⇔
�
=
1
0
0
5
/
1
0
.
0
5
=
1
0
0
9R+R+5%(R)=1005⇔R=1005/10.05=100

For the proposer validator:

The pool obtains 
�
+
5
%
(
�
)
R+5%(R): 105 EVMOS

Commission: 
1
0
5
∗
8
0
%
∗
1
%
105∗80%∗1% = 0.84 EVMOS

Validator's reward: 
1
0
5
∗
2
0
%
+
�
�
�
�
�
�
�
�
�
�
105∗20%+Commission = 21.84 EVMOS

Delegators' rewards: 
1
0
5
∗
8
0
%
−
�
�
�
�
�
�
�
�
�
�
105∗80%−Commission = 83.16 EVMOS (each delegator will be able to claim its portion of these rewards in proportion to their stake)

The pool obtains 
�
R: 100 EVMOS

Commission: 
1
0
0
∗
8
0
%
∗
1
%
100∗80%∗1% = 0.8 EVMOS

Validator's reward: 
1
0
0
∗
2
0
%
+
�
�
�
�
�
�
�
�
�
�
100∗20%+Commission = 20.8 EVMOS

Delegators' rewards: 
1
0
0
∗
8
0
%
−
�
�
�
�
�
�
�
�
�
�
100∗80%−Commission = 79.2 EVMOS (each delegator will be able to claim its portion of these rewards in proportion to their stake)

#What are the slashing conditions?
If a validator misbehaves, its bonded stake along with its delegators' stake and will be slashed. The severity of the punishment depends on the type of fault. There are 3 main faults that can result in slashing of funds for a validator and its delegators:

Double-signing: If someone reports on chain A that a validator signed two blocks at the same height on chain A and chain B, and if chain A and chain B share a common ancestor, then this validator will get slashed on chain A. The penalty for double signing is 10.00% of total stake.

Downtime: If a validator misses more than 50% of the last 90.000 blocks, they will get slashed by 0.50%.

Unavailability: If a validator's signature has not been included in the last X blocks, the validator will get slashed by a marginal amount proportional to X. If X is above a certain limit Y, then the validator will get unbonded.

Note that even if a validator does not intentionally misbehave, it can still be slashed if its node crashes, loses connectivity, gets DDoSed, or if its private key is compromised.

#Do validators need to self-bond EVMOS
No, they do not. A validators total stake is equal to the sum of its own self-bonded stake and of its delegated stake. This means that a validator can compensate its low amount of self-bonded stake by attracting more delegators. This is why reputation is very important for validators.

Even though there is no obligation for validators to self-bond EVMOS, delegators should want their validator to have self-bonded EVMOS in their staking pool. In other words, validators should have skin-in-the-game.

In order for delegators to have some guarantee about how much skin-in-the-game their validator has, the latter can signal a minimum amount of self-bonded EVMOS. If a validator's self-bond goes below the limit that it predefined, this validator and all of its delegators will unbond.

#How to prevent concentration of stake in the hands of a few top validators?
For now the community is expected to behave in a smart and self-preserving way. When a mining pool in Bitcoin gets too much mining power the community usually stops contributing to that pool. Evmos will rely on the same effect initially. In the future, other mechanisms will be deployed to smoothen this process as much as possible:

Penalty-free re-delegation: This is to allow delegators to easily switch from one validator to another, in order to reduce validator stickiness.
UI warning: Wallets can implement warnings that will be displayed to users if they want to delegate to a validator that already has a significant amount of staking power.
#Technical Requirements
#What are hardware requirements?
Validators should expect to provision one or more data center locations with redundant power, networking, firewalls, HSMs and servers.

We expect that a modest level of hardware specifications will be needed initially and that they might rise as network use increases. Participating in the testnet is the best way to learn more.

#What are software requirements?
In addition to running an Evmos node, validators should develop monitoring, alerting and management solutions.

#What are bandwidth requirements?
Evmos has the capacity for very high throughput compared to chains like Ethereum or Bitcoin.

As such, we recommend that the data center nodes only connect to trusted full nodes in the cloud or other validators that know each other socially. This relieves the data center node from the burden of mitigating denial-of-service attacks.

Ultimately, as the network becomes more used, one can realistically expect daily bandwidth on the order of several gigabytes.

#What does running a validator imply in terms of logistics?
A successful validator operation will require the efforts of multiple highly skilled individuals and continuous operational attention. This will be considerably more involved than running a bitcoin miner for instance.

#How to handle key management?
Validators should expect to run an HSM that supports ed25519 keys. Here are potential options:

YubiHSM 2
Ledger Nano S
Ledger BOLOS SGX enclave
Thales nShield support
Strangelove Horcrux(opens new window)
The Evmos team does not recommend one solution above the other. The community is encouraged to bolster the effort to improve HSMs and the security of key management.

#What can validators expect in terms of operations?
Running effective operation is the key to avoiding unexpectedly unbonding or being slashed. This includes being able to respond to attacks, outages, as well as to maintain security and isolation in your data center.

#What are the maintenance requirements?
Validators should expect to perform regular software updates to accommodate upgrades and bug fixes. There will inevitably be issues with the network early in its bootstrapping phase that will require substantial vigilance.

#How can validators protect themselves from Denial-of-Service attacks?
Denial-of-service attacks occur when an attacker sends a flood of internet traffic to an IP address to prevent the server at the IP address from connecting to the internet.

An attacker scans the network, tries to learn the IP address of various validator nodes and disconnect them from communication by flooding them with traffic.

One recommended way to mitigate these risks is for validators to carefully structure their network topology in a so-called sentry node architecture.

Validator nodes should only connect to full-nodes they trust because they operate them themselves or are run by other validators they know socially. A validator node will typically run in a data center. Most data centers provide direct links the networks of major cloud providers. The validator can use those links to connect to sentry nodes in the cloud. This shifts the burden of denial-of-service from the validator's node directly to its sentry nodes, and may require new sentry nodes be spun up or activated to mitigate attacks on existing ones.

Sentry nodes can be quickly spun up or change their IP addresses. Because the links to the sentry nodes are in private IP space, an internet based attacked cannot disturb them directly. This will ensure validator block proposals and votes always make it to the rest of the network.

It is expected that good operating procedures on that part of validators will completely mitigate these threats.

For more on sentry node architecture, see this (opens new window).