# Subscription management

To simplify the money flow during request creation and fulfillment, Quex introduces **subscriptions** - pre-deposited native tokens managed by the `IDepositManager` contract, which handle covering all necessary fees when fulfilling requests.

```solidity
interface IDepositManager {
    function createSubscription() external returns (uint256);
    function setOwner(uint256 subscriptionId, address owner) external;
    function deposit(uint256 subscriptionId) external payable;
    function withdraw(uint256 subscriptionId, address receiver) external;
    function addConsumer(uint256 subscriptionId, address consumer) external;
    function removeConsumer(uint256 subscriptionId, address consumer) external;

    function balance(uint256 subscriptionId) external view returns (uint256);
    function withdrawableBalance(uint256 subscriptionId) external view returns (uint256);
    function hasAccessToSubscription(uint256 subscriptionId, address consumer) external view returns (bool);
}
```

In most cases, you’d use our [Request management](https://github.com/quex-tech/quex-v1-interfaces/pull/6/files) tool which includes a built-in `createSubscriotion()` method. However, in this manual, we’ll give a more detailed breakdown of what happens during subscription setup.

To start using a subscription, a client should first register it and save its ID:

```solidity
IDepositManager depositManager = IDepositManager(quexCoreAddress);
uint256 subscriptionId = depositManager.createSubscription();
```

The obtained ID is used later to manage deposits and withdrawals, and also to make requests. The party creating a subscription becomes its owner and can add consumers or withdraw funds. You can change the owner at any time using the `setOwner` method. After creating the subscription, deposit native tokens into it to activate it:

```solidity
depositManager.deposit{value: depositValue}(subscriptionId);
```

To make your subscription functional, add one or more consumers:

```solidity
depositManager.addConsumer(subscriptionId, consumerAddress);
```

Here, `consumerAddress` is the address initiating requests, typically the smart contract receiving the data. However, you have flexibility in choosing different initiating addresses. Once these steps are completed, the subscription becomes fully functional, allowing you to securely bring Quex-certified data on-chain.

Finally, if the subscription is no longer needed, an owner can withdraw funds using the `withdraw` method, or remove an individual consumer using `removeConsumer` if other consumers still depend on the subscription.
