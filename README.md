# Rubic-hacked

### 漏洞:

項目方誤將 USDC 的合約加入到 proxy 的白名單中, 駭客調用 proxy 去呼叫 USDC 合約的 transferFrom 把受害者的 USDC 轉走

### 駭客 tx:

0x9a97d85642f956ad7a6b852cf7bed6f9669e2c2815f3279855acf7f1328e7d46

---

我們先去思考項目方的業務邏輯, 原本理想上是統一透過 proxy 去收手續費:

### 用戶 ----> Proxy Contract ----> Rubic Router

所以用戶在使用 Rubic router 的功能做轉帳或是跨鏈橋之前, 都必須透過這個 proxy 合約.

產生漏洞的是 proxy 下面這個 function:

```solidity
function routerCallNative(BaseCrossChainParams calldata _params, bytes calldata _data)
    external
    payable
    nonReentrant
    whenNotPaused
    eventEmitter(_params)
{
    if (!availableRouters.contains(_params.router)) {
        revert RouterNotAvailable();
    }
    IntegratorFeeInfo memory _info = integratorToFeeInf[_params.integrator];
    uint256 _amountIn = accrueTokenFees(
        _params.integrator,
        _info,
        accrueFixedCryptoFee(_params.integrator, _info),
        0,
        address(0)
    );
    AddressUpgradeable.functionCallWithValue(_paramsrouter, _data, _amountIn);
}
```

首先它是一個 external 合約, 任何人都可以呼叫他, 他唯一限制只有:

```solidity
if (!availableRouters.contains(_params.router)) {
    revert RouterNotAvailable();
}
```

如果用戶要呼叫的`_params.router`在白名單內, 那才可以繼續往下

下面有一個`AddressUpgradeable.functionCallWithValue`, 這是一個基本的 call 方法.

項目方先前誤將 USDC 加到白名單, 所以黑客調用方式變成:

### 駭客 ----> Proxy Contract ----> USDC contract

要滿足這個攻擊還有一個要條件就是---**用戶 approve proxy contract 使用 USDC**

所以當駭客透過 proxy 調用 USDC contract 時, 對 USDC 來說, msg.sender 是 proxy(有被 user approve), 那駭客便可以利用`functionCallWithValue`去 call `transferFrom`, 進而把用戶的 USDC 轉走

---

這個 proxy 設計缺陷在於, 他擁有的權限太大了, 讓用戶 approve 權限給一個不安全的 proxy 合約.
