# WINGS API Specification and Guide to Custom ICO Contracts Integration

WINGS is a platform for curating, evaluating, valuing and conducting token generation events (aka ICO, token sale or crowdsale) based on Ethereum smart contracts and IPFS. 

With WINGS any proposal that passes a community-based evaluation procedure known as WINGS Curation & Valuation Forecasting can launch a token sales contract via the WINGS ICO constructor, or by writing its own crowdsale contract and integrating it with WINGS. Participating WINGS community members (token holders) who evaluate a project are paid via a smart contracts-based reward algorithm based on the accuracy of their valuation forecast, their WINGS forecast reputation score, how many WINGS they lock into the evaluation and other factors. Importantly, WINGS enables self-regulation for the broader crypto-community of project funders who can be assured that WINGS project evaluators are kept honest via the WINGS protocol, and who therefore can eschew ICOs that have not earned their wings.  

To get more details read our [whitepaper](https://wingsfoundation.ch/docs/WINGS_Whitepaper_V1.1.2_en.pdf), visit [site](https://wings.ai), or join our [chat](https://telegram.me/wingschat).

## Content

- [Security](#security)
- [Help](#help)
- [Overview](#overview)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Integration](#integration)
- [Specification](#specification)
- [Step By Step Integration Guide](#step-by-step)
- [Tests](#tests)
- [Contributors](#contributors)
- [License](#license)

## Security

This project is still a work in progress. Remember that you are dealing with real virtual money and any issue can potentially lead to financial losses.

We strongly recommend for you to assure your integration and implementation of the crowdsale contract with tests verifying that your code funcations as intended.

We take no responsibility for your implementation decisions and any security problem you might experience during integration.

If you encounter any security issues or otherwise related issues, please, contact us via our [telegram chat](https://t.me/wingschat) or send us an email: [support@wings.ai](mailto:support@wings.ai), we have generous bounties, read more [here](https://blog.wings.ai).

## Help

If you need help during integration, we strongly recommend to contact us by email at [support@wings.ai](mailto:support@wings.ai), by [telegram chat](https://telegram.me/wingschat), or by directly contacting [contributors](#contributors).

## Overview

For an Ethereum-based crowdsale or ICO to participate in the WINGS ecosystem with a project that is going to create its own crowdsale smart contract use this documentation as a starting point of integrating with the WINGS valuation forecasting rewards contracts.

Integration can be split into the following steps:

1. Development of a crowdsale smart contract
2. Integration with WINGS smart contracts
3. Creation of a project with custom crowdsale contract on [WINGS Beta Testnet](https://testnet.wings.ai) (via UI or manually)
4. After forecasting has successfully concluded, initiating your custom crowdsale contract and crowdsale on [WINGS Beta Testnet](https://testnet.wings.ai) (via UI or manually)

This documentation assumes that you have already taken step 1, it describes step 2 in depth (integration), for steps 3-4 please visit our blog [post](https://blog.wings.ai).

During integration the developer of the smart contract should understand their dependencies and follow the rules described in the current document to be absolutely sure their smart contract works correctly.

If the developer does not follow our specification it can lead to issues which may include financial losses.

## Requirements

- Node.js v8
- truffle 4.0.6
- testrpc

## Getting Started

To get started install our package using npm:

    npm install wings-integration --save

Once the package has been installed, you can import our BasicCrowdsale.sol contract to your smart contract.

Like so:

```cs
import 'wings-integration/contracts/BasicCrowdsale.sol';
```

And then set the inheritance of BasicCrowdsale to your contract:

```cs
contract MyCrowdsale is BasicCrowdsale {
    ...
}
```

Now read [specification](#integration) before starting the implementation of your crowdsale smart contract.

In addition to this specification, we also offer a [step by step](#step-by-step) integration guide.

## Integration

Project owner should provide 2 contracts:
+ Token contract that complies to the ERC20 specification. Note that during the crowdfunding process token values should be produced ("minted" as in the example herein) or transferred ("sold") to the buyer's account from some special account;
+ Custom `Crowdsale` contract which derives from `BasicCrowdsale` contract (see [BasicCrowdsale.sol](https://github.com/WingsDao/3rd-party-integration/blob/master/BasicCrowdsale.sol))


Example contracts (see [example](https://github.com/WingsDao/wings-integration/tree/master/contracts/example) directory):
+ [Token example with minting funciton](https://github.com/WingsDao/wings-integration/blob/master/contracts/example/CustomTokenExample.sol)
+ [Token example which is compatible to Bancor's smart token interface](https://github.com/WingsDao/wings-integration/blob/master/contracts/example/BancorCompatibleTokenExample.sol), see specifications for [Bancor protocol](https://github.com/bancorprotocol/contracts#the-smart-token-standard)
+ [Custom crowdsale contract](https://github.com/WingsDao/wings-integration/blob/master/contracts/example/CustomCrowdsale.sol);

During its lifetime, the `Crowdsale` contract may reside in the following states:
+ Initial state: crowdsale has not yet started;
+ Active state: when everyone is able to buy the project’s tokens (after crowdsale has started);
+ Successful state: when the crowdsale succeeded either by achieving its hard cap or after the crowdsale period ends, with its  collected value being equal or above its minimal goal;
+ Failed state: when the crowdsale period finishes, but the minimal goal has not been reached *OR* if the project's owner did not manage to start the crowdfunding process within a 30 days period after the forecasting stage finished;
+ Stopped by owner: when the project owner cancels a crowdsale.

`Crowdsale` instance is managed by `CrowdsaleController` instance, which is a part of the WINGS contracts infrastructure.


It’s necessary for custom crowdsale contract to keep in its actual state all public fields of [ICrowdsaleProcessor](https://github.com/WingsDao/3rd-party-integration/blob/master/interfaces/ICrowdsaleProcessor.sol) observing the following rules:

+ `totalCollected` and `totalSold` increase only during active state;
+ `minimalGoal` and `hardCap` can be changed any number of times before `start()` is called, but not after that;
+ `duration`, timestamps (`startTimestamp` and `endTimestamp`), `fundingAddress` are set in `start()` function.

In addition, be sure, that:
+ Crowdsale contract must start within 30 days after forecasting has ended
+ Crowdsale contract must have a minimal goal and a hard cap
+ Crowdsale contract can be stopped by the crowdsale contract creator
+ Crowdsale contract must be able to distribute rewards after a successful crowdsale is over (see **mintETHRewards** and **mintTokensRewards** functions)

For more detailed requirements, see the custom crowdsale review checklist:

    While writing or reviewing custom Crowdsale contracts, please meet the following requirements:

    1. The contract has to be written in the Solidity language and derive directly from BasicCrowdsale;
    2. If stop() is overriden by derived contract then super.stop(...) (i.e BasicCrowdsale.stop()) must be called inside;
    3. If start() is overriden by derived contract then super.start(...) (i.e BasicCrowdsale.start()) must be called inside;
    4. Timestamps, as well as minimal goal and hard cap cannot be modified after BasicCrowdsale.start() is called;
    5. totalCollected and totalSold fields increase during every token sale for right amounts;
    6. Contract cannot sell tokens before startTimestamp, after endTimestamp;
    7. Contract cannot sell tokens above hard cap;
    8. Project owner must be able to withdraw collected ETH only after the crowdsale completed successfully and only to the fundingAddress provided in start();
    9. If the crowdsale failed or is cancelled, all participants should have an ability to refund their ETH;


### Specification

Custom crowdsale contract **must** be derived from `BasicCrowdsale` contract which, in turn, is derived from `ICrowdsaleProcessor` which, in turn, is derived from `Ownable` (from zeppelin-solidity library) and `HasManager` contracts.

**Difference between owner and manager:** Owner is typically the address of contract creator's account, manager is the address of the contract to which some actions are delegated. So effectively crowdsale contracts have 2 owners with different access rights.

`BasicCrowdsale` ([see src](https://github.com/WingsDao/wings-integration/blob/master/contracts/BasicCrowdsale.sol)) implements default behavior of custom crowdsale, but BasicCrowdsale.start(...) and BasicCrowdsale.stop() functions **must** be called (via `super` mechanism) if appropriate methods are overriden in derived contracts.

#### Ownable fields

**owner**
```cs
address public owner;
```
Owner's address. Allows for certain methods to be callbable by owner only (via `onlyOwner` modifier)
<br>
<br>
<br>

#### HasManager fields

**manager**
```cs
address public manager;
```
Manager's address. Allows for certain methods to be callbable by manager only (via `onlyManager` modifier)
<br>
<br>
<br>

#### ICrowdsaleProcessor fields

**started**
```cs
bool public started;
```
Becomes true when timeframe is assigned
<br>
<br>
<br>
**stopped**
```cs
bool public stopped;
```
Becomes true if cancelled by owner
<br>
<br>
<br>
**totalCollected**
```cs
uint256 public totalCollected;
```
Total collected Ethereum: must be updated every time tokens has been sold
<br>
<br>
<br>
**totalSold**
```cs
uint256 public totalSold;
```
Total amount of project's token sold: must be updated every time tokens has been sold
<br>
<br>
<br>
**minimalGoal**
```cs
uint256 public minimalGoal;
```
Crowdsale minimal goal, must be greater or equal to Forecasting min amount
<br>
<br>
<br>
**hardCap**
```cs
uint256 public hardCap;
```
Crowdsale hard cap, must be less than or equal to Forecasting max amount
<br>
<br>
<br>
**duration**
```cs
uint256 public duration;
```
Crowdsale duration in seconds. Accepted range is `MIN_CROWDSALE_TIME..MAX_CROWDSALE_TIME`.
<br>
<br>
<br>
**startTimestamp**
```cs
uint256 public startTimestamp;
```
Start timestamp of crowdsale, absolute UTC time
<br>
<br>
<br>
**endTimestamp**
```cs
uint256 public endTimestamp;
```
End timestamp of crowdsale, absolute UTC time
<br>
<br>
<br>

#### Ownable methods

**transferOwnership**
```cs
function transferOwnership(address newOwner) public onlyOwner;
```
Allows the current owner to transfer control of the contract to a newOwner.
<br>
<br>
<br>

#### HasManager methods

**transferManager**
```cs
function transferManager(address _newManager) public onlyManager();
```
New manager transfers its functions to new address.
<br>
<br>
<br>

#### ICrowdsaleProcessor methods

**deposit**
```cs
function deposit() public payable;
```
Allows to transfer some ETH into the contract without selling tokens.
<br>
<br>
<br>
**getToken**
```cs
function getToken() public returns(address);
```
Returns address of crowdsale token.
<br>
<br>
<br>
**mintETHRewards**
```cs
function mintETHRewards(address _contract, uint256 _amount) public onlyManager();
```
Transfers ETH rewards amount (if ETH rewards is configured) to Forecasting contract.
<br>
<br>
<br>
**mintTokenRewards**
```cs
function mintTokenRewards(address _contract, uint256 _amount) public onlyManager();
```
Mints token Rewards to Forecasting contract
<br>
<br>
<br>
**releaseTokens**
```cs
function releaseTokens() public onlyManager() hasntStopped() whenCrowdsaleSuccessful();
```
Releases tokens (transfers crowdsale token from mintable to a transferrable state).
<br>
<br>
<br>
**stop**
```cs
function stop() public onlyManager() hasntStopped();
```
Stops crowdsale. Called by CrowdsaleController, the latter is called by project's owner. Crowdsale may be stopped any time before it finishes.
<br>
<br>
<br>
**start**
```cs
function start(uint256 _startTimestamp, uint256 _endTimestamp, address _fundingAddress) public onlyManager() hasntStarted() hasntStopped();
```
Validates parameters and starts crowdsale.
<br>
<br>
<br>
**isFailed**
```cs
function isFailed() public constant returns (bool);
```
Is crowdsale failed (completed, but minimal goal was not reached).
<br>
<br>
<br>
**isActive**
```cs
function isActive() public constant returns (bool);
```
Is crowdsale active (i.e. the token can be sold).
<br>
<br>
<br>
**isSuccessful**
```cs
function isSuccessful() public constant returns (bool);
```
Is crowdsale completed successfully.
<br>
<br>
<br>

If you don't feel this provides enough functionality, you can override functions, for example,
overriding of isFailed/isActive/isSuccessful could be a good idea in cases where you have other
standards of states of your ICO.

For complex changes we strongly recommend covering your project with tests,
and contacting the WINGS team for [help](#help).

## Step By Step

Let's do a custom crowdsale contract step by step and integrate it with WINGS.

We will do a Crowdsale contract that works only with whitelisted addresses (or addresses
that passed KYC), as WINGS by default does not support such functionality natively.

First we will prepare a contract that will contain all whitelisted addresses added by the contract
owner.

So we create another [truffle](https://github.com/trufflesuite/truffle) project, adding there
`wings-integration` and [zeppelin-solidity](https://github.com/OpenZeppelin/zeppelin-solidity).

```cs
npm i wings-integration --save
npm i zeppelin-solidity --save
```

Now let's code the whitelist contract.

```cs
pragma solidity 0.4.18;

import 'zeppelin-solidity/contracts/ownership/Ownable.sol';

contract Whitelist is Ownable {
  event APPROVE(address indexed approved);
  event DECLINE(address indexed declined);

  mapping(address => bool) list;

  function Whitelist() {
    owner = msg.sender;
  }

  function addAddress(address _participant) public onlyOwner {
    require(!list[_participant]);

    list[_participant] = true;
    APPROVE(_participant);
  }

  function declineAddress(address _participant) public onlyOwner {
    require(list[_participant]);

    list[_participant] = false;
    DECLINE(_participant);
  }

  function isApproved(address _participant) public view returns (bool) {
    return list[_participant];
  }
}
```

And we need to make a token contract that can issue new tokens.
Let's take one from the example directory:

```cs
pragma solidity 0.4.18;

import "zeppelin-solidity/contracts/token/ERC20/StandardToken.sol";
import "zeppelin-solidity/contracts/ownership/Ownable.sol";

// Minimal crowdsale token for custom contracts
contract Token is Ownable, StandardToken {
    // ERC20 requirements
    string public name;
    string public symbol;
    uint8 public decimals;

    // how many tokens was created (i.e. minted)
    uint256 public totalSupply;

    // here are 2 states: mintable (initial) and transferrable
    bool public releasedForTransfer;

    // Ctor. Hardcodes names in this example
    function Token() public {
        name = "CustomTokenExample";
        symbol = "CTE";
        decimals = 18;
    }

// override these 2 functions to prevent from transferring tokens before it was released

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(releasedForTransfer);
        return super.transfer(_to, _value);
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
        require(releasedForTransfer);
        return super.transferFrom(_from, _to, _value);
    }

    // transfer the state from intable to transferrable
    function release() public
        onlyOwner() // only owner can do it
    {
        releasedForTransfer = true;
    }

    // creates new amount of the token from a thin air
    function issue(address _recepient, uint256 _amount) public
        onlyOwner() // only owner can do it
    {
        // the owner can mint until released
        require (!releasedForTransfer);

        // total token supply increases here.
        // Note that the recepient is not able to transfer anything until release() is called
        balances[_recepient] += _amount;
        totalSupply += _amount;
    }
}
```

So we have 18 decimals, `issue` function that allows us to issue new tokens, `release` function that
allows us to move tokens after the crowdsale.

And now lets start our crowdsale contract where we will have a fixed price of 100 tokens per 1 ETH, a 100 ETH minimal goal, and a 1000 ETH hard cap, which means our crowdsale is successful if we collect 100 ETH.
If we collect 1000 ETH, the crowdsale is closed by hard cap.

```cs
pragma solidity 0.4.18;

import 'wings-integration/contracts/BasicCrowdsale.sol';
import './Whitelist.sol';
import './Token.sol';

contract Crowdsale is BasicCrowdsale {
    ...
}
```

Let's add token and whitelist instances:


```cs
Whitelist public whitelist; // initialize whitelist
Token public crowdsaleToken; // initial token

uint256 public price = 100; // price for one ETH

mapping(address => uint256) participants; // list of participants
```

And make constructor:

```cs
function Crowdsale(address _whitelist, address _fundingAddress) BasicCrowdsale(msg.sender, msg.sender) {
    minimalGoal = 100 ether; // minimal goal
    hardCap = 1000 ether; // hard cap

    whitelist = Whitelist(_whitelist); // initialize whitelist by address
    crowdsaleToken = new Token(); // create token

    fundingAddress = _fundingAddress; // address where to withdraw ETH
}
```

So as you see we initialize the whitelist by address because a whitelist can have been deployed in the past 
and is not managed by our crowdsale contract.

At the same time, a token is issued by our Crowdsale contract, so that our Crowdsale contract is able to issue new tokens to
participants.

Do not forget about the funding address, where you can withdraw ETH later, where you can change the funding address by calling the `start` method of the smart contract.

Now let's create functions that allow us to exchange ETH sent to the contract for tokens:

```cs
// accept ETH by this contract
function() payable {
    require(msg.value > 0);
    participate(msg.value, msg.sender); // issue tokens
}

function participate(uint256 _value, address _recepient) internal
    hasBeenStarted() hasntStopped() whenCrowdsaleAlive()  // check crowdsale started, hasnt stopped and alive
{
    require(whitelist.isApproved(_recepient)); // check whitelist

    uint256 newTotalCollected = totalCollected + _value;

   if (hardCap < newTotalCollected) {
     // don't sell anything above the hard cap

     uint256 refund = newTotalCollected - hardCap;
     uint256 diff = _value - refund;

     // send the ETH part which exceeds the hard cap back to the buyer
     _recepient.transfer(refund);
     _value = diff;
   }

   // token amount as per price (fixed in this example)
   uint256 tokensSold = _value * price;

   // create new tokens for this buyer
   crowdsaleToken.issue(_recepient, tokensSold);

   // remember the buyer so he/she/it may refund its ETH if crowdsale failed
   participants[_recepient] += _value;

   // update total ETH collected
   totalCollected += _value;

   // update totel tokens sold
   totalSold += tokensSold;
}
```

Notice how we have added a check if the participant address was whitelisted, and the importance of updating tokensSold and totalCollected.

It is important to keep the state of the contracts correct, so that it is possible to participate in
the crowdsale only if the crowdsale has started, hasn't stopped, and is alive (see modifiers for participate function).

We still need to implement a few more functions to have a working contract, such as:

- Function for getting the token address
- Minting of token rewards to token contracts
- Releasing token after crowdsale

```cs
// returns address of crowdsale token. The token must be ERC20-compliant
function getToken() public returns(address)
{
    return crowdsaleToken;
}

// called by CrowdsaleController to transfer reward part of
// tokens sold by successful crowdsale to Forecasting contract.
// This call is made upon closing successful crowdfunding process.
function mintTokenRewards(
    address _contract,  // Forecasting contract
    uint256 _amount     // agreed part of totalSold which is intended for rewards
)
    public
    onlyManager() // manager is CrowdsaleController instance
{
    // crowdsale token is mintable in this example, tokens are created here
    crowdsaleToken.issue(_contract, _amount);
}


// transfers crowdsale token from mintable to transferrable state
function releaseTokens()
    public
    onlyManager()             // manager is CrowdsaleController instance
    hasntStopped()            // crowdsale wasn't cancelled
    whenCrowdsaleSuccessful() // crowdsale was successful
{
    // see token example
    crowdsaleToken.release();
}
```

The previous functions are allow you to mint token rewards for the WINGS community forecaster rewards, get the token
address, and allows you to release tokens to make them transferable.

**What is important**, is to keep the collected ETH in the crowdfunding contract and to make it possible
to withdraw forecaster rewards (this is done automatically). 
If the developer prefers to keep ETH on another address, and transfer the ETH once the contract receives it, it is still possible to deposit the contract with ETH via `deposit` function.

Finally - withdraw the ETH (in case of our contract) and refund (in case of failed crowdsale):

```cs
// project's owner withdraws ETH funds to the funding address upon successful crowdsale
function withdraw(
    uint256 _amount // can be done partially
)
    public
    onlyOwner() // project's owner
    hasntStopped()  // crowdsale wasn't cancelled
    whenCrowdsaleSuccessful() // crowdsale completed successfully
{
    require(_amount <= this.balance);
    fundingAddress.transfer(_amount);
}

  // backers refund their ETH if the crowdsale was cancelled or has failed
function refund()
    public
{
    // either cancelled or failed
    require(stopped || isFailed());

    uint256 amount = participants[msg.sender];

    // prevent from doing it twice
    require(amount > 0);
    participants[msg.sender] = 0;

    msg.sender.transfer(amount);
}
```
Then we transfer the collected ETH to funding address by calling the `withdraw` function.

Refund is implemented via the `refund` function, that anyone can call if the crowdsale has stopped or failed.

The last thing we have to do is:

- Deploy the crowdsale
- Create a project on [WINGS](https://wings.ai) and provide the address of the crowdsale contract
- After the forecasting period ends, transfer manager from the account you used to deploy the crowdsale contract
to the DAO contract address (which was created by [WINGS](https://wings.ai))

For more detailed instruction about project creation and other iterations of the Platform/UI see on our [blog](https://blog.wings.ai).

## Tests

// TODO: move part of test to current repository

## Contributors

+ [Artem Gorbachev](mailto:artem@wings.ai)
+ [Boris Povod](mailto:boris@wings.ai)

## Running migrations

Only example of deployment of custom crowdsale and custom token from examples folder:

    CROWDSALE=1 truffle migrate -f 1

Write you own for your custom crowdsale/token.

## License

[GPL v3.0 License](LICENSE).

Copyright 2018 © Wings Stiftung. All rights reserved.
