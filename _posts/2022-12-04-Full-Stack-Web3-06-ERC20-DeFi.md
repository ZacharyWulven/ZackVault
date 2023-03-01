---
layout: post
title: Full-Stack-Web3-06-ERC20-DeFi
date: 2022-12-04 16:45:30.000000000 +09:00
categories: [区块链, Chainlink 公开课]
tags: [区块链, Chainlink 公开课]
---

# Hardhat-Starter-Kit ( Lesson 11)

[smartcontractkit](https://github.com/smartcontractkit)

[hardhat-starter-kit](https://github.com/smartcontractkit/hardhat-starter-kit)
你可以以 `hardhat-starter-kit` 为模板创建 repo，进行智能合约开发。`hardhat-starter-kit` 里有一些例子可供参考


# Hardhat ERC20s ( Lesson 12)

EIP (Ethereum Improvement Proposal)
ERC (Ethereum Request for Comments)

[EIP20](https://eips.ethereum.org/EIPS/eip-20)

```
$ mkdir hardhat-erc20-fcc
$ cd hardhat-erc20-fcc
$ code .
$ yarn add --dev hardhat
$ yarn hardhat
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface tokenRecipient {
  function receiveApproval(
    address _from,
    uint256 _value,
    address _token,
    bytes calldata _extraData
  ) external;
}

contract TokenERC20 {
  // Public variables of the token
  string public name;
  string public symbol;
  uint8 public decimals = 18;
  // 18 decimals is the strongly suggested default, avoid changing it
  uint256 public totalSupply;

  // This creates an array with all balances
  mapping(address => uint256) public balanceOf;
  // 允许其他地址有权操作自己 token 数量
  mapping(address => mapping(address => uint256)) public allowance;

  // This generates a public event on the blockchain that will notify clients
  event Transfer(address indexed from, address indexed to, uint256 value);

  // This generates a public event on the blockchain that will notify clients
  event Approval(
    address indexed _owner,
    address indexed _spender,
    uint256 _value
  );

  // This notifies clients about the amount burnt
  event Burn(address indexed from, uint256 value);

  /**
   * Constructor function
   *
   * Initializes contract with initial supply tokens to the creator of the contract
   */
  constructor(
    uint256 initialSupply,
    string memory tokenName,
    string memory tokenSymbol
  ) {
    totalSupply = initialSupply * 10**uint256(decimals); // Update total supply with the decimal amount
    balanceOf[msg.sender] = totalSupply; // Give the creator all initial tokens
    name = tokenName; // Set the name for display purposes
    symbol = tokenSymbol; // Set the symbol for display purposes
  }

  /**
   * Internal transfer, only can be called by this contract
   */
  function _transfer(
    address _from,
    address _to,
    uint256 _value
  ) internal {
    // Prevent transfer to 0x0 address. Use burn() instead
    require(_to != address(0x0));
    // Check if the sender has enough
    require(balanceOf[_from] >= _value);
    // Check for overflows
    require(balanceOf[_to] + _value >= balanceOf[_to]);
    // Save this for an assertion in the future
    uint256 previousBalances = balanceOf[_from] + balanceOf[_to];
    // Subtract from the sender
    balanceOf[_from] -= _value;
    // Add the same to the recipient
    balanceOf[_to] += _value;
    emit Transfer(_from, _to, _value);
    // Asserts are used to use static analysis to find bugs in your code. They should never fail
    assert(balanceOf[_from] + balanceOf[_to] == previousBalances);
  }

  /**
   * Transfer tokens
   *
   * Send `_value` tokens to `_to` from your account
   *
   * @param _to The address of the recipient
   * @param _value the amount to send
   */
  function transfer(address _to, uint256 _value) public returns (bool success) {
    _transfer(msg.sender, _to, _value);
    return true;
  }

  /**
   * Transfer tokens from other address
   *
   * Send `_value` tokens to `_to` on behalf of `_from`
   *
   * @param _from The address of the sender
   * @param _to The address of the recipient
   * @param _value the amount to send
   */
  function transferFrom(
    address _from,
    address _to,
    uint256 _value
  ) public returns (bool success) {
    require(_value <= allowance[_from][msg.sender]); // Check allowance
    allowance[_from][msg.sender] -= _value;
    _transfer(_from, _to, _value);
    return true;
  }

  /**
   * Set allowance for other address
   *
   * Allows `_spender` to spend no more than `_value` tokens on your behalf
   *
   * @param _spender The address authorized to spend
   * @param _value the max amount they can spend
   */
  function approve(address _spender, uint256 _value)
    public
    returns (bool success)
  {
    allowance[msg.sender][_spender] = _value;
    emit Approval(msg.sender, _spender, _value);
    return true;
  }

  /**
   * Set allowance for other address and notify
   *
   * Allows `_spender` to spend no more than `_value` tokens on your behalf, and then ping the contract about it
   *
   * @param _spender The address authorized to spend
   * @param _value the max amount they can spend
   * @param _extraData some extra information to send to the approved contract
   */
  function approveAndCall(
    address _spender,
    uint256 _value,
    bytes memory _extraData
  ) public returns (bool success) {
    tokenRecipient spender = tokenRecipient(_spender);
    if (approve(_spender, _value)) {
      spender.receiveApproval(msg.sender, _value, address(this), _extraData);
      return true;
    }
  }

  /**
   * Destroy tokens
   *
   * Remove `_value` tokens from the system irreversibly
   *
   * @param _value the amount of money to burn
   */
  function burn(uint256 _value) public returns (bool success) {
    require(balanceOf[msg.sender] >= _value); // Check if the sender has enough
    balanceOf[msg.sender] -= _value; // Subtract from the sender
    totalSupply -= _value; // Updates totalSupply
    emit Burn(msg.sender, _value);
    return true;
  }

  /**
   * Destroy tokens from other account
   *
   * Remove `_value` tokens from the system irreversibly on behalf of `_from`.
   *
   * @param _from the address of the sender
   * @param _value the amount of money to burn
   */
  function burnFrom(address _from, uint256 _value)
    public
    returns (bool success)
  {
    require(balanceOf[_from] >= _value); // Check if the targeted balance is enough
    require(_value <= allowance[_from][msg.sender]); // Check allowance
    balanceOf[_from] -= _value; // Subtract from the targeted balance
    allowance[_from][msg.sender] -= _value; // Subtract from the sender's allowance
    totalSupply -= _value; // Update totalSupply
    emit Burn(_from, _value);
    return true;
  }
}
```

## 使用 Openzeppelin
```
$ yarn add --dev @openzeppelin/contracts
```

另一个类似 [openzeppelin](https://docs.openzeppelin.com/contracts/4.x/) 的库是 [solmate](https://github.com/transmissions11/solmate)，它们的目标都是 Solidity 标准库。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract OurToken is ERC20 {
    // initialSupply is 50 WEI
    // initialSupply 50e18, 50 * 10**18

    constructor(uint256 initialSupply) ERC20("DAI", "DAI") {
        _mint(msg.sender, initialSupply);
    }
}
```

# Hardhat DeFi & Aave ( Lesson 13)

[What is DeFi?](https://github.com/smartcontractkit/full-blockchain-solidity-course-js#lesson-13-hardhat-defi--aave)
[defillama](https://defillama.com/)

What is Aave?
Aave 是一个借贷协议，它允许我们借入借出加密货币，一切都是通过智能合约，

[测试网](https://staging.aave.com/)
[测试网](https://testnet.aave.com/market)

1:25:42
