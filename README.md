# blur-analysis

- [blur-analysis](#blur-analysis)
  - [Marketplace: BlurSwap](#marketplace-blurswap)
  - [Marketplace 2: BlurExchange](#marketplace-2-blurexchange)
    - [代码地址](#代码地址)
    - [整体架构](#整体架构)
    - [(1) BlurExchange](#1-blurexchange)
      - [订单的定义](#订单的定义)
      - [execute](#execute)
    - [(2) MatchingPolicy](#2-matchingpolicy)
      - [PolicyManager](#policymanager)
      - [MatchingPolicy](#matchingpolicy)
    - [(3) ExecutionDelegate](#3-executiondelegate)
    - [(4) Signature Authentication](#4-signature-authentication)
      - [用户签名](#用户签名)
        - [批量 Listing](#批量-listing)
        - [\_validateUserAuthorization()](#_validateuserauthorization)
      - [Oracle 签名](#oracle-签名)
        - [\_validateOracleAuthorization()](#_validateoracleauthorization)
  - [总结](#总结)

## Marketplace: BlurSwap

https://etherscan.io/address/0x39da41747a83aee658334415666f3ef92dd0d541

BlurSwap 合约， fork 自 GemSwap。二者代码一致。

主要用于处理聚合交易相关逻辑。

## Marketplace 2: BlurExchange

https://etherscan.io/address/0x000000000000ad05ccc4f10045630fb830b95127

BlurExchange 合约。Blur 自建的交易市场合约。

BlurExchange 的交易模型于 Opensea 一样是中央订单簿的交易模型，都是由链下的中心化的订单簿和链上的交易组成。

其中链下的订单簿负责存储用户的挂单信息，并对订单进行撮合。最终的成交和转移 NFT 是由 BlurExchange 来负责的。

### 代码地址

Blur 官方没有给出具体的代码仓库地址。不过我在 GitHub 上找到了下面这个代码仓库，应该是之前提交审计的时候留下来的。

https://github.com/code-423n4/2022-10-blur

### 整体架构

![](exchange_architecture.png)

https://etherscan.io/viewsvg?t=1&a=0x39da41747a83aeE658334415666f3EF92DD0D541

![](mainnet-0x39da41747a83aee658334415666f3ef92dd0d541.svg)

按照模块可以分为一下几类：

1. BlurExchange：主合约，负责交易的执行。
2. MatchingPolicy：订单交易策略，负责判断买单、买单是否可以匹配。
3. PolicyManager：订单交易策略管理者
4. ExecutionDelegate：负责具体的转移代币的逻辑。

### (1) BlurExchange

#### 订单的定义

```solidity
struct Order {
    address trader; // 订单创建者
    Side side; // 交易方向
    address matchingPolicy; // 交易策略
    address collection;
    uint256 tokenId;
    uint256 amount;
    address paymentToken;
    uint256 price;
    uint256 listingTime;
    /* Order expiration timestamp - 0 for oracle cancellations. */
    uint256 expirationTime;
    Fee[] fees;
    uint256 salt;
    bytes extraParams;
}

enum Side { Buy, Sell }
enum SignatureVersion { Single, Bulk }

```

#### execute

订单的匹配通过 `execute()` 进行匹配。

```solidity
struct Input {
    Order order;
    uint8 v;
    bytes32 r;
    bytes32 s;
    bytes extraSignature;
    SignatureVersion signatureVersion;
    uint256 blockNumber;
}
```

```solidity
function execute(Input calldata sell, Input calldata buy)
        external
        payable
        reentrancyGuard
        whenOpen
    {
        require(sell.order.side == Side.Sell);

        // 计算订单 hash
        bytes32 sellHash = _hashOrder(sell.order, nonces[sell.order.trader]);
        bytes32 buyHash = _hashOrder(buy.order, nonces[buy.order.trader]);

        // 校验订单参数
        require(_validateOrderParameters(sell.order, sellHash), "Sell has invalid parameters");
        require(_validateOrderParameters(buy.order, buyHash), "Buy has invalid parameters");

        // 校验签名，order.trader == msg.sender 则不去校验直接返回 true
        require(_validateSignatures(sell, sellHash), "Sell failed authorization");
        require(_validateSignatures(buy, buyHash), "Buy failed authorization");

        // 校验买卖订单是否能匹配
        (uint256 price, uint256 tokenId, uint256 amount, AssetType assetType) = _canMatchOrders(sell.order, buy.order);

        // 执行资产转移
        _executeFundsTransfer(
            sell.order.trader,
            buy.order.trader,
            sell.order.paymentToken,
            sell.order.fees,
            price
        );

        // 执行 NFT 转移
        _executeTokenTransfer(
            sell.order.collection,
            sell.order.trader,
            buy.order.trader,
            tokenId,
            amount,
            assetType
        );

        /* Mark orders as filled. */
        // 存储订单状态
        cancelledOrFilled[sellHash] = true;
        cancelledOrFilled[buyHash] = true;

        // 发出事件
        emit OrdersMatched(
            // 买单时间大，表明此次是由买家触发，事件中的 maker 是买家。相反的 maker 是卖家表明是有卖家触发的订单。
            sell.order.listingTime <= buy.order.listingTime ? sell.order.trader : buy.order.trader, 
            sell.order.listingTime > buy.order.listingTime ? sell.order.trader : buy.order.trader,
            sell.order,
            sellHash,
            buy.order,
            buyHash
        );
    }
```

订单完成后发出 `OrdersMatched` 事件。

```solidity
    event OrdersMatched(
        address indexed maker,
        address indexed taker,
        Order sell,
        bytes32 sellHash,
        Order buy,
        bytes32 buyHash
    );
```

### (2) MatchingPolicy

在上面撮合订单的方法中会进行校验买卖单能否成交。

```solidity
function _canMatchOrders(Order calldata sell, Order calldata buy)
        internal
        view
        returns (uint256 price, uint256 tokenId, uint256 amount, AssetType assetType)
    {
        bool canMatch;
        if (sell.listingTime <= buy.listingTime) {
            /* Seller is maker. */
            // 校验订单的成交策略是否在白名单中
            require(policyManager.isPolicyWhitelisted(sell.matchingPolicy), "Policy is not whitelisted");
            // 调用具体的校验方法进行校验
            (canMatch, price, tokenId, amount, assetType) = IMatchingPolicy(sell.matchingPolicy).canMatchMakerAsk(sell, buy);
        } else {
            /* Buyer is maker. */
            require(policyManager.isPolicyWhitelisted(buy.matchingPolicy), "Policy is not whitelisted");
            (canMatch, price, tokenId, amount, assetType) = IMatchingPolicy(buy.matchingPolicy).canMatchMakerBid(buy, sell);
        }
        require(canMatch, "Orders cannot be matched");

        return (price, tokenId, amount, assetType);
    }
```

#### PolicyManager

https://etherscan.io/address/0x3a35A3102b5c6bD1e4d3237248Be071EF53C8331

用于管理所有的交易策略。

包括交易策略的添加，移除和查看等功能。

#### MatchingPolicy

目前官方代码里有两类限价单的策略，但是 ERC1155 策略的合约还没有设置进白名单中。

1. StandardPolicyERC721：https://etherscan.io/address/0x00000000006411739DA1c40B106F8511de5D1FAC#code
2. StandardPolicyERC1155

```solidity
function canMatchMakerAsk(Order calldata makerAsk, Order calldata takerBid)
        external
        pure
        override
        returns (
            bool,
            uint256,
            uint256,
            uint256,
            AssetType
        )
    {
        return (
            // 订单方向不能一样
            (makerAsk.side != takerBid.side) && 
            // 支付token相同
            (makerAsk.paymentToken == takerBid.paymentToken) &&
            (makerAsk.collection == takerBid.collection) &&
            (makerAsk.tokenId == takerBid.tokenId) &&
            (makerAsk.matchingPolicy == takerBid.matchingPolicy) &&
            (makerAsk.price == takerBid.price),
            makerAsk.price,
            makerAsk.tokenId,
            1,
            AssetType.ERC721
        );
    }
```

### (3) ExecutionDelegate

买卖双方在授权代币的时候会向 ExecutionDelegate 授权。然后在订单成交的时候由 ExecutionDelegate 负责具体的转移代币的逻辑。

```solidity
// BlurExchange 中的转移方法
// ETH or WETH 的转移
function _transferTo(
        address paymentToken,
        address from,
        address to,
        uint256 amount
    ) internal {
        if (amount == 0) {
            return;
        }

        if (paymentToken == address(0)) {
            /* Transfer funds in ETH. */
            payable(to).transfer(amount);
        } else if (paymentToken == weth) {
            /* Transfer funds in WETH. */
            executionDelegate.transferERC20(weth, from, to, amount);
        } else {
            revert("Invalid payment token");
        }
    }
```

```solidity
// BlurExchange 中的转移方法
// NFT 的转移
function _executeTokenTransfer(
        address collection,
        address from,
        address to,
        uint256 tokenId,
        uint256 amount,
        AssetType assetType
    ) internal {
        /* Assert collection exists. */
        require(_exists(collection), "Collection does not exist");

        /* Call execution delegate. */
        if (assetType == AssetType.ERC721) {
            executionDelegate.transferERC721(collection, from, to, tokenId);
        } else if (assetType == AssetType.ERC1155) {
            executionDelegate.transferERC1155(collection, from, to, tokenId, amount);
        }
    }
```

### (4) Signature Authentication

#### 用户签名

交易所接受由 signatureVersion 参数确定的两种类型的签名认证 - 单一或批量。单个 listing 通过订单哈希的签名进行身份验证。

##### 批量 Listing

要进行批量 listing 的时候，用户需要从订单哈希生成默克尔树并签署树根。最后将订单各自的默克尔路径打包在 extraSignature 中，然后成交的时候再从订单和默克尔路径重构默克尔根，并验证签名。

##### _validateUserAuthorization()

```solidity
function _validateUserAuthorization(
        bytes32 orderHash,
        address trader,
        uint8 v,
        bytes32 r,
        bytes32 s,
        SignatureVersion signatureVersion,
        bytes calldata extraSignature
    ) internal view returns (bool) {
        bytes32 hashToSign;
        if (signatureVersion == SignatureVersion.Single) { // 单个签名
            /* Single-listing authentication: Order signed by trader */
            hashToSign = _hashToSign(orderHash);
        } else if (signatureVersion == SignatureVersion.Bulk) { // 批量签名
            /* Bulk-listing authentication: Merkle root of orders signed by trader */
            // 从 extraSignature 中解出 merkle tree 的路径
            (bytes32[] memory merklePath) = abi.decode(extraSignature, (bytes32[]));
            // 计算 merkle tree 的 root 节点
            bytes32 computedRoot = MerkleVerifier._computeRoot(orderHash, merklePath);
            hashToSign = _hashToSignRoot(computedRoot);
        }
        // 校验签名
        return _recover(hashToSign, v, r, s) == trader;
    }
```

#### Oracle 签名

此功能允许用户事前明确同意，要求使用最近的区块号对订单进行授权的预言机签名。这启用了一种链下取消方法，预言机可以继续向潜在的接受者提供签名，直到用户请求预言机停止。一段时间后，旧的预言机签名将过期。

要选择事前明确同意，用户必须将 expireTime 设置为 0。为了履行订单，预言机签名必须打包在 extraSignature 中，并且 blockNumber 设置为预言机签名的内容。

##### _validateOracleAuthorization()

```solidity
function _validateOracleAuthorization(
        bytes32 orderHash,
        SignatureVersion signatureVersion,
        bytes calldata extraSignature,
        uint256 blockNumber
    ) internal view returns (bool) {
        bytes32 oracleHash = _hashToSignOracle(orderHash, blockNumber);

        uint8 v; bytes32 r; bytes32 s;
        if (signatureVersion == SignatureVersion.Single) {
            (v, r, s) = abi.decode(extraSignature, (uint8, bytes32, bytes32));
        } else if (signatureVersion == SignatureVersion.Bulk) {
            /* If the signature was a bulk listing the merkle path musted be unpacked before the oracle signature. */
            (bytes32[] memory merklePath, uint8 _v, bytes32 _r, bytes32 _s) = abi.decode(extraSignature, (bytes32[], uint8, bytes32, bytes32));
            v = _v; r = _r; s = _s;
        }

        return _recover(oracleHash, v, r, s) == oracle;
    }
```

## 总结

整体看来 Blur 给人的感觉还是挺简洁的。

批量签名、预言机签名这些新功能会有有很大的应用空间。

目前 Blur 应该还是在很早期的阶段，毕竟他只支持了 ERC721 的限价单的交易方式。不支持 ERC1155 的交易，也不支持拍卖功能。当然这些应该都在他们的开发计划中。通过 MatchingPolicy 可以很方便的添加新的交易策略。这一点跟 BendDao 的 Execution Strategy 很像。猜测大概率是借鉴过来的。（关于 BendDao 更多的信息可以查看我的另一篇文章：[BendDAO-analysis](https://github.com/cryptochou/BendDAO-analysis#execution-strategy%E6%89%A7%E8%A1%8C%E7%AD%96%E7%95%A5)）

虽然是很早起的阶段，但是 Blur 目前的交易量大有赶超 Opensea 之势。应该是明确的空投预期起到了很大的作用。毕竟天下苦 Opensea 久矣。🤣

