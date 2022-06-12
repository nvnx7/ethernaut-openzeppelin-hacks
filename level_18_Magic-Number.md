# Level 18: Magic Number

This is the level 18 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- Solidity [bytecode and opcode](https://medium.com/@blockchain101/solidity-bytecode-and-opcode-basics-672e9b1a88c2) basics
- [Destructuring Solidity Contract](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-i-introduction-832efd2d7737/) series
- EVM opcodes [reference](https://ethereum.org/en/developers/docs/evm/opcodes)

## Hack

Given contract:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

`player` has to make a tiny contract (`Solver`) in size (10 opcodes at most) and set it's address in `MagicNum`.

There's a tight restriction on size of the `Solver`
contract - 10 opcodes or less. Because each opcode is 1 byte, the bytecode of the solver must be 10 bytes at max.

Writing high-level solidity would yield the size much greater than just 10 bytes, so we turn to writing raw EVM bytes corresponding to contract opcodes.

We need to write two sections of opcodes:
- **Initialization Opcodes** which EVM uses to create the contract by replicating the runtime opcodes and returning them to EVM to save in storage.

- **Runtime Opcodes** which contains the execution logic of the contract.

Alright, so let's figure out runtime opcode first.

### Runtime Opcode

The code needs to return the 32 byte magic number - 42 or `0x2a` (in hex).

The corresponding opcode is `RETURN`. But, `RETURN` takes two arguments - the location of value in memory and the size of this value to be returned. That means the `0x2a` needs to be stored in memory first - which `MSTORE` facilitates. But `MSTORE` itself takes two arguments - the location of value in stack and its size. So, we need push the value and size params into stack first using `PUSH1` opcode.

Lookup the opcodes to be used in opcode reference to get corresponding hex codes:
```
OPCODE       NAME
------------------
 0x60        PUSH1
 0x52        MSTORE
 0xf3        RETURN
```

Let's write corresponding opcodes:
```
OPCODE   DETAIL
------------------------------------------------
602a     Push 0x2a in stack. 
         Value (v) param to MSTORE(0x60)

6050     Push 0x50 in stack. 
         Position (p) param to MSTORE

52       Store value,v=0x2a at position, p=0x50 in memory

6020     Push 0x20 (32 bytes, size of v) in stack. 
         Size (s) param to RETURN(0xf3)

6050     Push 0x50 (slot at which v=0x42 was stored). 
         Position (p) param to RETURN

f3      RETURN value, v=0x42 of size, s=0x20 (32 bytes)
```

Concatenate the opcodes and we get the bytecode:
```
602a60505260206050f3
```
which is exactly 10 bytes, the max limit allowed by the level!

### Initialization opcode

The initialization opcodes need to come before the runtime opcode. These opcodes need to load runtime opcodes into memory and return the same to EVM.

To `CODECOPY` opcode can be used to copy the runtime opcodes. It takes three arguments - the destination position of copied code in memory, current position of runtime opcode in the bytecode and size of the code in bytes.

Following opcodes is needed for the above purpose:
```
OPCODE       NAME
------------------
 0x60        PUSH1
 0x52        MSTORE
 0xf3        RETURN
 0x39        CODECOPY
```

But we don't know the position of runtime opcode in final bytecode (since init. opcode comes before runtime opcode). Let's omit it using `--` for now and calculate the init. opcodes:
```
OPCODE   DETAIL
-----------------------------------------
600a     Push 0x0a (size of runtime opcode i.e. 10 bytes)
         in stack.
         Size (s) param to COPYCODE (0x39)

60--     Push -- (unknown) in stack 
         Position (p) param to COPYCODE

6000     Push 0x00 (chosen destination in memory) in stack
         Destination (d) param to COPYCODE

39       Copy code of size, s at position, p
         to destination, d in memory

600a     Push 0x0a (size of runtime opcode i.e. 10 bytes)
         in stack.
         Size (s) param to RETURN (0xf3)

6000     Push 0x00 (location of value in memory) in stack.
         Position (p) param to RETURN

f3      Return value of size, s at position, p
```
So the initialization opcode is:
```
600a60--600039600a6000f3
```
which is 12 bytes in total.

And hence runtime opcodes start at index 12 or position `0x0c`.

Therefore initialization opcode must be:
```
600a600c600039600a6000f3
```

### Final Opcode

Alright we have initialization as well as runtime opcodes now. Concatenate them to get final opcode:
```
    initialization opcode + runtime opcode

=   600a600c600039600a6000f3 + 602a60505260206050f3

=   600a600c600039600a6000f3602a60505260206050f3
```

We can now create the contract by noting the fact that a transaction sent to zero address (`0x0`) with some data is interpreted as Contract Creation by the EVM.

```javascript
bytecode = '600a600c600039600a6000f3602a60505260206050f3'

txn = await web3.eth.sendTransaction({from: player, data: bytecode})
```

After deploying get the contract address from returned transaction receipt:
```javascript
solverAddr = txn.contractAddress
```

Set the address `Solver` address in `MagicNum`:
```javascript
await contract.setSolver(solverAddr)
```

Submit instance.

Done!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

