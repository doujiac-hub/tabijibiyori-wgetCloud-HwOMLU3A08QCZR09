
# 项目介绍


Pendle 官方网站：[https://www.pendle.finance/](https://github.com)
Pendle Finance 是一种无需许可的收益交易协议，用户可以在其中执行各种收益管理策略，通过将 DeFi 上的生息资产拆分成 PT（本金代币）和 YT（收益代币）来实现上述功能。


将 1 ETH 质押成 1 stETH，年利率为 5%，那么到期后 1 stETH 就能收回 1 ETH（本金）\+ 0\.05 ETH（收益）。而 Pendle 所做的就是把 1 stETH 代表的生息代币（**SY**），拆分成了本金（**Principal**）和收益（**Yield**）两部分，使其可以分别流通于市场上。


YT 和 PT 的定义：


1. YT (Yield Token) \-\> 代表该仓位的应收收益：1 YT 使您有权在到期前获得 1 单位标的资产（例如 1 ETH、1 DAI、1 USDe 等）的收益，并可实时索取。
2. PT（Principal Token）\-\>代表本金金额：1 PT 赋予您在到期时赎回 1 单位标的资产（例如 1 ETH、1 DAI、1 USDe 等）的权利。


用户可以将生息代币（1 stETH）拆分成 PT 和 YT ，然后使其在市场上流通。也可以通过存入等量的 PT 和 YT 来赎回标的资产。到期后，PT 可以赎回其标的资产（1 ETH），无需对应的 YT（这是因为到期的 YT 价值为 0，因为它们不再产生收益）。



> 本篇文章分析时用的代码版本是：[https://github.com/pendle\-finance/pendle\-core\-v2\-public/tree/260e8d3a807ae2bd195a77cdefb869f494c53ebb](https://github.com)


# 流动性池


在项目介绍章节，有提到过 PT 和 YT 可以分别流通于市场上，也可以存入等量的 PT 和 YT 兑换成 SY。为了实现上面提到的功能，Pendle 实现了一个基于 \[SY, PT] 的 AMM Pool，用户可以通过它来兑换 SY，PT，YT 三种资产（很奇妙吧，一个 pair 兑换三种资产）。



> 由于到期后 PT 的价值和 SY 的价值是相等的，所以在 \[SY, PT] Pool 中添加流动性可以被认为在到期后不会产生无常损失。


Pendle 收益代币化的核心是将收益代币拆分为 PT 和 YT。两种代币的价格相加就是基础资产的价格。因此，PT和YT的价格一定是**负相关的**——YT价格越高，PT价格越低，反之亦然。


那么，到底是怎么样通过 \[SY, PT] Pool 来进行 PT/SY → YT 的兑换呢？官方文档给出了下面的描述：


1. 买方将 PT/SY 发送到 Router 中
2. Router 从池中闪电贷出 SY
3. 将这笔 SY 铸造成 PT \+ YT
4. 将 YT 发送给买家
5. 将 PT 出售兑换成 SY，以返还步骤 2 中的闪电贷


![image](https://img2024.cnblogs.com/blog/1483609/202412/1483609-20241202222246336-1278063365.png)


转换成公式表达



```
Input:      
					= x * PT
Flashloan SY: 
					= x * PT + y * SY
					= x * PT + y * (PT + YT)
					= (x + y) * PT + y * YT
Sell to repay flashloan:
					= y * YT
Output:
					= y * YT

===================================

In flashloan:
					y * SY = (x + y) * PT

```

基于上面的公式，x 作为输入的 PT 数量，计算出 YT 的输出数量 y。


那么这个计算过程是怎么通过实现的？在 Pendle 的代码实现中没有通过公式计算，而是采用了二分查找法直接找出 y 的值（没想到吧）。


PT → YT 的合约入口：



> contracts/router/ActionSwapYTV3\.sol



```
// ------------------ SWAP PT FOR YT ------------------

    /// @notice For details on the parameters (input, guessPtSwapToSy, limit, etc.), please refer to IPAllActionTypeV3.
    function swapExactPtForYt(
        address receiver,
        address market,
        uint256 exactPtIn,   **// @note `x` amount**
        uint256 minYtOut,
        ApproxParams calldata guessTotalPtToSwap
    ) external returns (uint256 netYtOut, uint256 netSyFee) {
        (, IPPrincipalToken PT, IPYieldToken YT) = IPMarket(market).readTokens();

        uint256 totalPtToSwap;
        
        **// @note `y` == netYtOut, `x + y` == totalPtToSwap**
        (netYtOut, totalPtToSwap, netSyFee) = _readMarket(market).approxSwapExactPtForYt(
            YT.newIndex(),
            exactPtIn,
            block.timestamp,
            guessTotalPtToSwap
        );

        _transferFrom(IERC20(PT), msg.sender, market, exactPtIn);

				**// @note swap PT to `receiver`**
        IPMarket(market).swapExactPtForSy(
            address(YT),
            totalPtToSwap,
            _encodeSwapExactPtForYt(receiver, exactPtIn, minYtOut, YT)
        );

        emit SwapPtAndYt(msg.sender, market, receiver, exactPtIn.neg(), netYtOut.Int());
    }

```


> contracts/router/math/MarketApproxLib.sol



```
/**
 * @dev algorithm:
 *     - Bin search the amount of PT to swap to SY
 *     - Flashswap the corresponding amount of SY out
 *     - Tokenize all the SY into PT + YT
 *     - PT to repay the flashswap, YT transferred to user
 *     - Stop when the additional amount of PT to pull to repay the loan approx the exactPtIn
 *     - guess & approx is for totalPtToSwap
 */
function approxSwapExactPtForYt(
    MarketState memory market,
    PYIndex index,
    uint256 exactPtIn,
    uint256 blockTime,
    ApproxParams memory approx
) internal pure returns (uint256, /*netYtOut*/ uint256, /*totalPtToSwap*/ uint256 /*netSyFee*/) {
    MarketPreCompute memory comp = market.getMarketPreCompute(index, blockTime);
    
    **// @note `guessOffchain` is the value of `y` calculated offline**
    **// @note `approx` is the upper and lower limits of the binary search**
    if (approx.guessOffchain == 0) {
        approx.guessMin = PMath.max(approx.guessMin, exactPtIn);
        approx.guessMax = PMath.min(approx.guessMax, calcMaxPtIn(market, comp));
        validateApprox(approx);
    }

		**// @note binary search**
    for (uint256 iter = 0; iter < approx.maxIteration; ++iter) {
        uint256 guess = nextGuess(approx, iter);    **// @note `guess` == (x + y) ?**
			  
			  **// @note (x + y) * PT -> y * SY**
	      (uint256 netSyOut, uint256 netSyFee, ) = calcSyOut(market, comp, index, guess);  
				
				**// @note calculate the actual vaule of SY (becasuse its value will change with the change of `index`)
				// @note `netAssetOut` == y ?**
        uint256 netAssetOut = index.syToAsset(netSyOut);

        // guess >= netAssetOut since we are swapping PT to SY
        uint256 netPtToPull = guess - netAssetOut;  **// @note `netPtToPull` == x ?**

        if (netPtToPull <= exactPtIn) {
		        **// @note if the gap of `netPtToPull` and `x` is acceptable** 
            if (PMath.isASmallerApproxB(netPtToPull, exactPtIn, approx.eps)) {
		            **// @note `y` == `netAssetOut`, `x + y` == `guess`**  
                return (netAssetOut, guess, netSyFee);
            }
            approx.guessMin = guess;
        } else {
            approx.guessMax = guess - 1;
        }
    }
    revert("Slippage: APPROX_EXHAUSTED");
}

```

当然除了 PT → YT，Pendle 还支持 SY，PT，YT 三种代币两两互相交换，在 contracts/router 目录下可以找到每种兑换方式的实现。读者感兴趣可以自行了解。


**为什么采用 `netAssetOut` 来计算 PT 的数量，而不是直接采用 SY 来计算？**


在 `approxSwapExactPtForYt` 函数中，通过 `uint256 netAssetOut = index.syToAsset(netSyOut);` 将 SY 的数量 `netSyOut` 转换成了 Asset 的数量 `netAssetOut` ，然后再计算出所需要的 PT 数量 `netPtToPull`。


PT 是代表到期赎回本金权利的代币，到期后按照 PT : Asset \= 1 : 1 的比例进行赎回。而 SY 包含了收益部分，其价值会随着时间的增加而增加。所以在计算 PT 数量的时候选择将 SY 换算成 Asset 进行计算，是为了统一计算单位。


官方例子：[https://docs.pendle.finance/Developers/HighLevelArchitecture\#principal\-token\-pt](https://github.com)



> At redemption, `1 PT = X SY`, where `X` satisfies the condition that `X SY = 1 Asset`. For example, assuming `1 wstETH = 1.2 stETH` on 1/1/2024, `1 PT-wstETH-01JAN2024` will be redeemable to `0.8928 wstETH` at maturity.


用户可以向流动性池子添加流动性，以通过多种途径获得收益：


1. 基础资产的协议收益/奖励
2. 池中 PT 部分的固定收益（添加流动性时将以折扣价格买入 PT）
3. Swap fees
4. $PENDLE 激励


![image](https://img2024.cnblogs.com/blog/1483609/202412/1483609-20241202222351338-2112024809.png)


# 经济模型


用户可以将添加流动性或其他渠道获得的 PENDLE 代币进行锁定，得到 vePENDLE 代币，而这个代币将在 Pendle 的治理与收益分配中起到重要作用。整个 Pendle 的经济模型如下图所示。


![image](https://img2024.cnblogs.com/blog/1483609/202412/1483609-20241202222405376-1015351146.png)


用户可以将手里的 PENDLE 通过 VotingEscrowPendleMainchain 合约进行锁定，合约会将用户的锁定时间和代币数量记录到 position 中。vePENDLE 不具有流动性，这在锁定期到期之前它无法转移。



> contracts/LiquidityMining/VotingEscrow/VotingEscrowPendleMainchain.sol


![image](https://img2024.cnblogs.com/blog/1483609/202412/1483609-20241202222424208-993096912.png)


在获得 vePENDLE 后，用户可以通过 PendleVotingControllerUpg 合约向池子进行投票。



> contracts/LiquidityMining/VotingController/PendleVotingControllerUpg.sol


![image](https://img2024.cnblogs.com/blog/1483609/202412/1483609-20241202222448772-750518029.png)


使用 vePENDLE 进行投票将获得的收益：


1. 将 $PENDLE 激励投入流动性池
2. 获得投票池 swap fee：
	1. vePENDLE 投票者能够从投票池中获得 80% 的 swap fee
3. 接收基础APY
	1. Pendle 从 YT 累积的所有收益（包括积分）中收取 3% 的费用。目前，该费用 100% 分配给 vePENDLE 持有者
	2. 到期未赎回 PT 的部分收益也将按比例分配给 vePENDLE 持有者。
4. 增加 LP 奖励（高达 250%）：
	1. 如果您在持有 vePENDLE 的同时将 LP 加入到池中，那么您所有 LP 的 PENDLE 激励和奖励也将进一步增加，根据您的 vePENDLE 价值最多可增加 250%。


由于 vePENDLE 投票可以给投票用户带来收益，那么为了将这个收益最大化，需要有组织地对 vePENDLE 代币进行投票操作。由此衍生出了一批在 Pendle 基础上建立起来的 DeFi 协议：Penpie、Equilibria 和 StakeDAO 等。


简单介绍一下其中一个平台 Penpie


官方文档：[https://docs.penpiexyz.io/penpie\-ecosystem/introduction](https://github.com)



> Penpie 是 Magpie 在 Pendle Finance 基础上打造的一个为 Pendle 用户提供**收益提升服务**的 DeFi 平台。
> Penpie 为 PENDLE 持有者提供通过 PENDLE 转换为 mPENDLE 来赚取可观收益。用户可以通过 Penpie 质押 mPENDLE，以获得平台上增强的活跃用户参与奖励。当用户将其 PENDLE 转换为 mPENDLE 时，Penpie 会自动将转换后的 PENDLE 在 Pendle Finance 中锁定为 vePENDLE。这一机制使 PENDLE 持有者能够通过 mPENDLE 获得更大的奖励，同时还授予 Penpie 治理权，并作为 Pendle Finance 的流动性提供者获得更高的年化利率 (APR%)。由于 Penpie 持有 vePENDLE，流动性提供者可以将资产存入平台并赚取提升的 PENDLE，而无需自己将任何 PENDLE 锁定为 vePENDLE。


Penpie 平台在为用户带来更高收益的同时，也遭受了项目攻击的风险。详情可见：[【漏洞分析】Penpie 攻击事件：重入攻击构造奖励金额](https://github.com)


这种在 DeFi 上建立 DeFi 的做法无疑是为用户带来了更高的收益，但是由于用户资金在投资过程中涉及的项目增加，遭受合约漏洞攻击而产生资金损失的风险也随之增大。在投资过程中选择经过专业安全审计机构审计过的项目十分重要。那么安全审计哪家强？!【此处招商，价格公道，童叟无欺！】


  * [项目介绍](#%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D):[Flowercloud 机场订阅加速](https://flowercloud6.com)
* [流动性池](#%E6%B5%81%E5%8A%A8%E6%80%A7%E6%B1%A0)
* [经济模型](#%E7%BB%8F%E6%B5%8E%E6%A8%A1%E5%9E%8B)

   \_\_EOF\_\_

   https://github.com/ACaiGarden/p/18582899  - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
