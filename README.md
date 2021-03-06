# Legal and smart contracts framework to implement Employee Stock Options Plan

There is a lot of stuff below on what ESOP is, how vesting works etc. If you are just interested in smart contract info go here, for info on testing and deployment go here.

## What is ESOP and why we do it?
ESOP stands for Employees Stock Options Plan. Many companies decide to involve their employees in company's success by offering them shares. Shares are typically available in form of options (mostly due to tax reasons) and are converted directly into cash when company has an IPO or gets acquired. There is a lot of interesting reasoning behind various ESOP's structures and a lot of discussions when it works best and when not. Here is a nice introduction:
https://www.accion.org/sites/default/files/Accion%20Venture%20Lab%20-%20ESOP%20Best%20Practices.pdf

Neufund preaches what it prays and offers its employees ESOP via a smart contract where options are represented as Ethereum tokens. On the other side employees are still provided with ESOP terms in readable English (we call it *legal wrapper*) which are generated from before mentioned smart contract. Such construct replaces paper agreement employee signs and adds many interesting things on top.

1. Process of assigning options, vesting and converting is immutable and transparent, all the nice things you get by using smart contracts are here.
2. It is enforceable in off-chain court like standard paper agreement, *however* as smart contracts are self-enforcing probability of such occurrence is much lower!
3. Typical criticism of ESOP is that you need to wait till the exit or IPO to get your shares and money. This is too long for being a real incentive. **This is not the case with tokenized options.** Being an Ethereum token extends ways in which you can put your options to use. For example **you can convert them into ERC20 compliant tokens when company is doing its ICO** or **make options directly trade-able** (via migration mechanism described later).
4. Smart contracts are self-enforcing and do their own book keeping. They are very cheap once written and tested. Together with companys/employee dapp (http://github.com/Neufund/...) [...]

## ESOP Algorithm
### ESOP Roles and Lifecycles
There are 3 main roles in ESOP project:
1. `owner` which represents company's sysadmin - deploys and upgrades smart contracts.
2. `company` which represents company management. Transactions signed by this role are deemed to be executed by Neufund (in case of our ESOP deployment). There are several reasons to have a separate account for this, but primarily we do not want `owner` to execute any ESOP logic, instead we provide nice D-app which our CEO can directly use.
3. `employee` which corresponds to employee receiving and exercising options.

Employee life within ESOP starts when company offers him/her options. Employee should sign the offer within provided deadline. This starts his/her employment period (counted from so called `issue date`) If employee leaves company then he goes to terminated state which also stops the vesting. Employee may be also fired in which case s/he is simply removed from employees list. Finally when exit/ICO etc. happens, conversion offer is made by company to employee which when accepted puts employee in converted state.
Check `ESOPTypes.sol` for employees possible states and properties.

ESOP itself has simple lifecycle. When deployed, it is in `new` state. It is opened by providing configuration parameters by `company` role. At that point all functions involving employees may happen. When there is conversion event like exit, `company` will switch ESOP into `conversion` state in which options may be exercised by employees.

### Assigning new options
Options are assigned to employees in two ways.

1. A method preferred by us that rewards risk taken. It simply gives more options to employees that came to work for us earlier, nothing else matters. It works as follows: there is a pool of 1 000 000 options of which employee 1 gets 10% which is 100 000. Employee 2 gets 10% of what remains in the pool (that is 900 000 options) which equals 90 000 and so on and so on. We call options distributed in this way `pool options`.
2. A method where company decides how much option it gives to employee. We call this options `extra options`.

Both methods can be combined. `pool options + extra options == issued options`

### Employee's options over time
Employee will not get all his/her options at the moment they are issued (however you can configure our smart contract to act in such way). Smart contract will release options up to all `issued options` with an algorithm called *vesting*.
[image in vesting]
As you can see there is a period of time called `cliff period` (for example 1 year, configurable, may be 0) during which employee does not get any options.

Then during the `vesting period` (period configurable, also may be 0), number of options increases up until `issued options`. In case of exit, IPO, ICO etc. additional `bonus options` (for example 20%, configurable) are added on top of issued options.

Please note that if exit, ICO etc. happens before `vesting period` is over, employee gets all `issued options` + `bonus` (we call this case `accelerated vesting.`).

### What happens when employee stops working for a company?
Sometimes people leave and ESOP smart contract handles that as well.

1. Company may remove employee from ESOP smart contract and cancel all his/her options. This is called `bad leaver event` in legal wrapper and may happen when for example employee breaks the law and needs to be fired. As you can expect such event cannot be defined in smart contract (there's no proper oracle yet in court system ;>) so this definition remains in legal wrapper.
2. Employee may just leave company and go working somewhere else. This case is more complicated.

[img termination]

As you can see, vesting stops at the moment employee stops working at the company and all vested options are issued to employee. From that time amount of `vested options` is slowly decreasing down to `residual amount` (value configurable). Such process is called `fadeout` and fadeout period equals period of time employee worked at company. Smart contract may be configured for full fadeout and to not do fadeout at all.

Terminated employee has no rights to `accelerated vesting` and no rights to `bonus options`.

### What is options conversion?
Here is the most interesting thing about ESOP smart contract. At some point in time (called `conversion event`), if company is successful it gets acquired or does an IPO and shareholders make a lot of money. This is what happens classically and we fully support it. We, however, extend this definition to any ICO, tokenization event or even further by allowing direct trade of vested options.

As you could expect there is no oracle for conversion events so those are defined in legal wrapper (see chapter 3). When such event happens employee may convert his/her options into shares, tokens or directly into EUR/ETH/BTC according to a Options Conversion Smart Contract (we have a few examples later) which will be provided by company when conversion event hapens.

## Procedures and Security
Here's how Neufund handles security when issuing options.

1. All our employees and company's dog get basic training in blockchain and security. We've published our training material here and here. May be pretty useful!
2. All our employees get Nano Ledger and store their private keys in there. We encourage employees to store their backup codes in some safe place (like notary).
3. Backup codes of Neufund admin's Nano Ledger which is used to deploy smart contract and company management Nano Ledger are kept in safe at notary office.
4. Options are offered via subscription forms implemented as d-apps where we enforce usage of Nano Ledger for our employees (however, we support Metamask and other web3 providers).

Also it is clear for everyone that if you loose your private key you will loose all your options.

## Smart Contracts
### `ESOP` contract
`ESOP` smart contracts handles employees' lifecycle, manages options' pool and handles conversion offer via calling provided implementation of options conversion contracts. Implementation is pretty straightforward and functions more or less correspond to provisions in legal wrapper. Terminology is also preserved.
All non-const functions return "businees logic" errors via return codes and throw only in case generic problems like no permission to call function or invalid state of smart contracts. Return codes correspond to `ReturnCode` event, in case of `OK` return code, specific event is logged (like `EmployeeSignedToESOP`). I hope `revert` opcode gets implemented soon!
`ESOP` aggregates the following contracts:
* `OptionsCalculator` which handles all options calculations like computing vesting, fadeout etc. and after configuring provides just a set of public constant methods.
* `EmployeesList` that contains iterable mapping of employees. Please note that `ESOP` is the sole writer to `EmployeesList` instance.

`ESOP` inherits from following contracts (I skip some obvious things like `Ownable`)
* `TimeSource` provides unified time source for all smart contracts in this project. It allows to mock-up time when not on mainnet.
* `ESOPTypes` defines `Employee` struct that is used everywhere in this project (I know, it should be a library)
* `CodeUpdateable` provides method to lock ESOP contract state when migrating to updated code (see later)

Ownership structure is worth mentioning. In `ESOP` and `OptionsCalculator` we have two owners:
* `owner` which deploys and upgrades code
* `company` that represents company's management

Please note that `owner` cannot execute any ESOP logic. S/he can deploy contracts and upgrade code but cannot open (activate) ESOP nor add employees.

#### ESOP configuration examples

|Description|Cliff Period|Vesting Period|Residual Amount %|Bonus Options %|New Employee Pool %|
|-----|----|----|----|----|----|
|Neufund configuration|1 year|4 years|20%|20%|10%|
|No cliff|0|4 years|20%|20%|10%|
|No fadeout|1 year|4 years|100%|20%|10%|
|Full fadeout|1 year|4 years|0|20%|10%|
|Disable pool options, only extra|1 year|4 years|20%|10%|0|
|Disable vesting, fadeout and bonus|0|2 weeks|100%|0|10%|

Neufund configuration sets options pool size to 1000080 and options per share to 360.

### Root of Trust
`RoT` is an immutable, deployed once contract that points to other contracts that are deemed currently "supported/official/endorsed by company" etc. At this moment `RoT` points to current ESOP implementation (see code update and migration procedures). Our d-app uses `RoT` address (which never changes) to infer all other addresses it needs.
The right to change ESOP implementation and current owner `RoT` is with `company` not `owner`.

### Option Converters

Options converter implementation will correspond to given conversion scenario (for example cash payout after exit is different from getting tokens in ICO). However, each such contract must derive from `BaseOptionsConverter` which is provided to `ESOP` smart contract by `company`.
At minimum `exerciseOptions` function must be implemented (except two getters) that will be called by `ESOP` smart contract on behalf of employee when s/he decides to execute options. Please note that employee can burn his/her options - see comments in base class for details.
W provide two implementations of `BaseOptionsConverter`:
* `ERC20OptionsConverter` which converts options into ERC20 token.
* 'ProceedsOptionsConverter' that adds proceeds payouts via withdrawal pattern with several payouts made by company. Token trading is still enabled.

Both example converters are nicely tested but they are not considered production grade so beware.

### Code Update
Code update is strictly defined in legal wrapper chapter 8 and is reserved for bugfixing, optimization etc. where "spirit of the agreement" is not changed.

Code update starts via calling methods defined in `CodeUpdateable` base class from which `ESOP` contract derives. During code update only constant method of this contract are available so state cannot be changed. Whole procedure ends with replacing ESOP contract address in `RoT` contract instance.

There is a nice test/example of the whole procedure in `js_tests/test.CodeUpdate.js`, example updated ESOP is provided in `js_tests/UpdatedESOP.sol` and concept of data structure migration (`EmployeesList`) is demonstrated by `js_tests/UpdatedESOP.sol`.

Please note that all old ESOP contracts (including `ESOP`, `EmployeesList` and `OptionsCalculator`) selfdestruct at the end.

### Migration to new ESOP
Employee and company may agree to migrate to ESOP with different set of rules. This happens via `ESOPMigration` smart contract that is provided by company and then accepted by employee. Please check comments around `allowEmployeeMigration` and `employeeMigratesToNewESOP` functions in `ESOP` for many interesting details.
Migration process is strictly defined in the legal wrapper.

### TODO
* `ESOPTypes` asks to be an library that defines `Employee` type. Update is simple however `dapple` test framework does not appear to support libraries anymore and currently I will not port all the test to truffle (it lacks many useful features - see later).
* Options pool management functions (fadeout and options re-distribution, see `removeEmployeesWithExpiredSignaturesAndReturnFadeout`) should go to separate library. Why not now? See above.
* Port all solidity tests from `dapple` (which is sadly discontinues) to `truffle`.

## Legal Wrapper
Legal wrapper establishes ESOP and accompanying smart contracts as legally binding in off-chain court system. Please read the source document [here], it is really interesting!

A fundamental problem we had to solve is **which contract form should prevail in case of conflict: computer code vs. terms in legal wrapper**. It's a story similar to multi-lingual legal documents (like you sign your ESOP with Chinese company which is originally in Chinese but translated to English).

* In case of ESOP agreement we have an asymmetric situation in terms of information and power. Employees are clearly at disadvantage in both. In such case, to make agreement legally binding, **it must be proven that employee could understand what s/he was signing**. Thus we had to make **legal wrapper to prevail over smart contract** as it is extremely easy for any employee to prove s/he could not understood the Solidity code and make court take his/her side. If there are any conflicts, English is the ground truth. This unfortunate situation will persist until there is a smart contract language that ordinary people can understand or they all learn Solidity. Whichever happens first.
* In case of b2b agreements like when Limited Partner joins VC Fund, there is a symmetry both in information and power. This allows a smart contract to be a source of truth (if you are rich guy you can always ask your developer buddies to check the contract for you - it may be much cheaper than to ask lawyers to do the same). In case of b2b the legal wrapper may be limited to just the most important business terms and still hold in court.

Other fundamental problem is a possible conflict of spirit and letter of the law (remember TheDAO?). Can we have bugs in the code or all code is law? We are clearly on the side of spirit of law prevailing and our legal wrapper (and corresponding smart contract code!) contains **bug fixing provision** and **provision to change ESOP rules** in chapter 8.

The same chapter defines a few other blockchain-related provisions like what we do in case of fork and what should happen when employee looses his/her private key (in short: s/he looses all their options so keep your keys safe).

Technically, wrapper is just a text document stored in ipfs, whose hash is added to ESOP smart contract. This document is filled with employee-specific variables and may be printed for reference. As you could expect we do not translate EVM assembly to English.

## UI/D-APP
It's here.
Please note that options subscription form has terminology and content defined in legal wrapper and d-app UI conforms to that.

## Development
We used solc 0.4.8
install binary packages cpp solc from solc repo http://solidity.readthedocs.io/en/develop/installing-solidity.html to use with dapple
upgrade truffle by modifying package.js of truffle and putting right solc version like
```
cd /usr/lib/node_modules/truffle
atom package.js <- change solc version
npm update
```

### Running unit (solidity) tests
Solidity tests are run with dapple. This is unfortunate as dapple is discontinued and does not support libraries

### Running integration (js) tests

RoT.at(RoT.address).ESOPAddress()
ESOP.at(ESOP.address).rootOfTrust()

running testrpc with lower block gas limit ~ mainnet limit
`testrpc --gasLimit=4000000  -i=192837991`
`truffle deploy --network deployment`

### setting up dev chain on parity and get some eth
parity --chain dev --jsonrpc-port 8444 ui
https://github.com/paritytech/parity/wiki/Private-development-chain

## Steps to reproduce and verify bytecode deployed on mainnet


--------------------
fineprints:
I hereby subscribe for the Issued Options for shares in {company} under the terms and conditions as set out in the ESOP Smart Contract at address {sc-address} and made available to me in [title of legal wrapper].
