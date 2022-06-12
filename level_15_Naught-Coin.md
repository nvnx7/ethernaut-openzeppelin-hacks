# Level 15: Naught Coin

This is the level 15 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) spec
- OpenZeppelin's [ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol) token codebase

## Hack
Given contract:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  }
} 
```

`player` is given `totalSupply` of a ERC20 Token - Naught Coin, but cannot transact them before 10 year lockout period. `player` has to get its token balance to 0.


The trick here is that `transfer` is not the only method in `ERC20`(and hence, in `NaughtCoin` too) that includes code for transfer of tokens between addresses.

According to `ERC20` spec, there's also an allowance mechanism that allows anyone to authorize someone else (the `spender`) to spend their tokens! This is exactly what `allowance(address owner, address spender)` method is for, in the `ERC20` contract. The allowance can then be transacted using the `transferFrom(address sender, address recipient, uint256 amount)` method.

Apart from `player` create another account named - `spender` in your wallet (MetaMask or some other wallet).

Get the `player`'s total balance by:
```javascript
totalBalance = await contract.balanceOf(player).then(v => v.toString())

// Output: '1000000000000000000000000'
```

Make the `player` approve `spender` for all of it's tokens:
```javascript
await contract.approve(spender, totalBalance)
```

Make the `spender` to transfer all of it's allowance (which is equal to all of the tokens of `player`) to `spender` itself.

I used MetaMask wallet and it connects only a single account at a time to an application. But we need both `player` and `spender` connected.

 For this, in a new browser window, go to Remix, and connect only the `spender` account to it. (Make sure `player` is disconnected from Remix and `spender` is disconnected from Ethernaut. For me, switching accounts in one window caused switch in other window too, otherwise).

Create an interface to `NaughtCoin`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface INaughtCoin {
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);
}
```

And load instance of `NaughtCoin` using **At Address** button with given instance address of `NaughtCoin`.

Call the `transferFrom` method with params - `player` address as sender, `spender` address as recipient and `1000000000000000000000000` as amount.

And that's it.

Level cleared!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

