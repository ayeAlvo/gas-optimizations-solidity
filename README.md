# GAS OPTIMIZATIONS

1. ## ++variable costs less gas than variable++, especially when it’s used in for-loops (—variable/variable— too)

_Prefix increments are cheaper than postfix increments._

Example wrong:

```java
 for (uint256 index = 0; index < epochs.length; i++) {
```

Change `i++` to `++i`

### 1.1 Add unchecked:

_`++i`/`i++` should be `unchecked{++i}`/`unchecked{i++}` when it is not possible for them to overflow, as is the case when used in for- and while-loops_

```java
for (uint256 i = 0; i < length;) {
    // ...
    unchecked { --i; }
}
```

Note though that this is safe to do only if you know that the post-iteration operation will never overflow the variable type of length.

The unchecked keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas per loop.

Solidity >= v0.8.19 option:

```java
pragma solidity >=0.8.19;

import { UC, uc } from "unchecked-counter/UC.sol";

function iterate(uint256[] memory arr) pure {
    uint256 length = arr.length;
    for (UC i = uc(0); i < uc(length); i = i + uc(1)) {
        uint256 element = arr[i.into()];
    }
}
```

See the [GitHub repo](https://github.com/PaulRBerg/unchecked-counter) for more details.

Increment:

-   i += 1 is the most expensive form
-   i++ costs 6 gas less than i += 1
-   ++i costs 5 gas less than i++ (11 gas less than i += 1)

Decrement:

-   i -= 1 is the most expensive form
-   i-- costs 11 gas less than i -= 1
-   --i costs 5 gas less than i-- (16 gas less than i -= 1)

Note that post-increments (or post-decrements) return the old value before incrementing or decrementing, hence the name post-increment. Consider using pre-increments and pre-decrements where they are logically relevant.

 <hr>
 <br>

2. ## Cheaper input valdiations should come before expensive operations

_Check @audit comment for details_

Example wrong and comments:

```java
925:        LockedBalance memory _locked = locked[_tokenId];
926:					//@audit this check should be before we read from storage
927:        require(_value > 0); // dev: need non-zero value
942:        uint256 unlock_time = ((block.timestamp + _lock_duration) / WEEK) * WEEK; // Locktime is rounded down to weeks
943:					//@audit this check should be before we do unlock_time evalutaion
944:        require(_value > 0); // dev: need non-zero value
977:        assert(_isApprovedOrOwner(msg.sender, _tokenId));
978:
979:        LockedBalance memory _locked = locked[_tokenId];
980:					//@audit this check should be first, before line 977 check
981:        assert(_value > 0); // dev: need non-zero value
```

 <hr>
 <br>

3.  ## Help the optimizer by saving a storage variable’s reference instead of repeatedly fetching it

_To help the optimizer, declare a `storage` type variable and use it instead of repeatedly fetching the reference in a map or an array. The effect can be quite significant.
As an example, instead of repeatedly calling `someMap[someIndex]`, save its reference like this: `SomeStruct storage someStruct = someMap[someIndex]` and use it._

Example wrong:

```java
138:        if (checkpoints[nftId][nCheckpoints - 1].fromBlock <= blockNumber) {
139:            return checkpoints[nftId][nCheckpoints - 1].delegatedTokenIds;
```

Declare `storage` variable and use it:

```java
Checkpoint storage checkpoint = checkpoints[nftId][nCheckpoints - 1];
if (checkpoint.fromBlock <= blockNumber) {
    return checkpoint.delegatedTokenIds;
}
```

 <hr>
 <br>

4.  ## `public` functions to `external`

_External call cost is less expensive than of public functions.
Contracts [are allowed](https://docs.soliditylang.org/en/latest/contracts.html#function-overriding) to override their parents’ functions and change the visibility from `external` to `public`.
The following functions could be set `external` to save gas and improve code quality._

Example wrong:

```java
  // Add fees contributed by the Seller of nft and the exchange/frontend that facilated the trade
98:  function addFee(address[2] memory addr, uint256 fee) public onlyTrader {

  // allows sellers of nft to claim there previous epoch rewards
141: function traderClaim(address addr, uint256[] memory epochs) public {

  // allows exchange that facilated the nft trades to claim there previous epoch rewards
155: function exchangeClaim(address addr, uint256[] memory epochs) public {

  // allows VeNFT holders to claim there token and eth rewards
172: function multiStakerClaim(uint256[] memory tokenids, uint256[] memory epochs) public {
```

<hr>
<br>

5. ## Using `calldata` instead of `memory` for read-only arguments in `external` functions saves gas.

_If you choose to make suggested above `public` functions as `external`, to continue gas optimizaton we can use `calldata` function arguments instead of `memory`._

_When a function with a `memory` array is called externally, the `abi.decode()` step has to use a for-loop to copy each index of the calldata to the memory index. Each iteration of this for-loop costs at least 60 gas (i.e. `60 * <mem_array>.length`). Using `calldata` directly, obliviates the need for such a loop in the contract code and runtime execution. Structs have the same overhead as an array of length one._

Example:

```java
98:  function addFee(address[2] memory addr, uint256 fee) public onlyTrader {
141: function traderClaim(address addr, uint256[] memory epochs) public {
155: function exchangeClaim(address addr, uint256[] memory epochs) public {
172: function multiStakerClaim(uint256[] memory tokenids, uint256[] memory epochs) public {
```

<hr>
<br>

6. ## Default value initialization

_If a variable is not set/initialized, it is assumed to have the default value (`0`, `false`, `0x0` etc depending on the data type).
Explicitly initializing it with its default value is an anti-pattern and wastes gas._

Example wrong:

```java
45:  uint256 public epoch = 0;
```

Example wrong:

```java
for (uint256 i = 0; i < num.length; ++i) {};
```

Change for:

```java
for (uint256 i; i < num.length; ++i) {};
```

Remove explicit initialization for default values.

### 6.1 Using delete instead of setting struct 0 save gas

Example wrong:

```java
  93:    liquidityPosition.long0Fees = 0;
  101:    liquidityPosition.long1Fees = 0;
  109:    liquidityPosition.shortFees = 0;
```

Change for:

```java
- 228:    pool.long0ProtocolFees = 0;
+ 228:    delete pool.long0ProtocolFees;
```

<hr>
<br>

7. ## Splitting `require()` statements that use `&&` saves gas

_Instead of using the `&&` operator in a single require statement to check multiple conditions, I suggest using multiple require statements with 1 condition per require statement (saving 3 gas per `&`).
See [this issue](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) which describes the fact that there is a larger deployment gas cost, but with enough runtime calls, the change ends up being cheaper._

Example:

```java
239: require(attachments[_tokenId] == 0 && !voted[_tokenId], 'attached');
```

### 7.1 Use nested if and, avoid multiple check combinations

_Using nested is cheaper than using && multiple check combinations. There are more advantages, such as easier to read code and better coverage reports._

Example:

```java
- 149:   if (param.long0FeesDesired == 0 && param.long1FeesDesired == 0 && param.shortFeesDesired == 0) Error.zeroInput();
+           if (param.long0FeesDesired == 0) {
+               if (param.long1FeesDesired == 0) {
+                   if (param.shortFeesDesired == 0) {
+                       Error.zeroInput();
+	 }
+               }
+           }
```

### 7.2 Sort Solidity operations using short-circuit mode

_Short-circuiting is a solidity contract development model that uses `OR`/`AND` logic to sequence different cost operations. It puts low gas cost operations in the front and high gas cost operations in the back, so that if the front is low, if the cost operation is feasible, you can skip (short-circuit) the subsequent high-cost Ethereum virtual machine operation._

Example:

```java
//f(x) is a low gas cost operation
//g(y) is a high gas cost operation

//Sort operations with different gas costs as follows
f(x) || g(y)
f(x) && g(y)
```

<hr>
<br>

8. ## `abi.encode()` is less efficient than `abi.encodePacked()`

_`abi.encode` will apply [ABI encoding rules](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html). Therefore all elementary types are padded to 32 bytes and dynamic arrays include their length. Therefore it is possible to also decode this data again (with `abi.decode`) when the type are known._

_`abi.encodePacked` will only use the minimal required memory to encode the data. E.g. an address will only use 20 bytes and for dynamic arrays only the elements will be stored without length. For more info see the [Solidity docs for packed mode](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html?highlight=encodepacked#non-standard-packed-mode)._

Example:

```java
102:  abi.encode(
115:	abi.encode(
130:	abi.encode(
414:  bytes32 computedHash = keccak256(abi.encode(leaf));
```

<hr>
<br>

9. ## State variables can be packed into fewer storage slots

_These state variables can be packed together to use less storage slots:_

Example:

```java
297: int128 internal constant iMAXTIME = 4 * 365 * 86400;
320: uint8 public constant decimals = 18;
347: bytes4 internal constant ERC165_INTERFACE_ID = 0x01ffc9a7;
350: bytes4 internal constant ERC721_INTERFACE_ID = 0x80ac58cd;
353: bytes4 internal constant ERC721_METADATA_INTERFACE_ID = 0x5b5e139f;
356: uint8 internal constant _not_entered = 1;
357: uint8 internal constant _entered = 2;
358: uint8 internal _entered_state = 1;
```

<hr>
<br>

10. ## `x +=` y costs more gas than `x = x + y` for state variables

_`x += y` costs more than `x = x + y`_

_same as `x -= y`_

Example:

```java
499: ownerToNFTokenCount[_to] += 1;
512: ownerToNFTokenCount[_from] -= 1;
```

Replace `x += y` and `x -= y` with `x = x + y` and `x = x - y`.

<hr>
<br>

11. ## Using `> 0` costs more gas than `!= 0` when used on a `uint` in a `require()` and `assert` statement and `condicional`

_`> 0` is less efficient than `!= 0` for unsigned integers._

_`!= 0` costs less gas compared to `> 0` for unsigned integers in `require` and `assert` statements with the optimizer enabled (6 gas)._

_To update this with using at least `0.8.6` there is no difference in gas usage with != 0 or > 0_

Example wrong:

```java
927: require(_value > 0); // dev: need non-zero value
944: require(_value > 0); // dev: need non-zero value
981: assert(_value > 0); // dev: need non-zero value
```

Replace `> 0` with `!= 0`
Or update `soldity` compiler to `>=0.8.13`

### 11.1 `>=` is cheaper than `>`

_Non-strict inequalities (`>=`) are cheaper than strict ones (`>`). This is due to some supplementary checks (ISZERO, 3 gas)._
_The compiler uses opcodes GT and ISZERO for solidity code that uses >, but only requires LT for >=, which saves 3 gas_

```java
uint256 public gas;
function checkStrict() external {
        gas = gasleft();
        require(999999999999999999 > 1); // gas 5017
        gas -= gasleft();
    }
function checkNonStrict() external {
        gas = gasleft();
        require(999999999999999999 >= 1); // gas 5006
        gas -= gasleft();
    }
```

Gas savings: non-strict inequalities will save you 15–20 gas.

### 11.2 `> 0` is cheaper than `!= 0` sometimes

_`!= 0` costs less gas compared to `> 0` for unsigned integers in require statements `with the optimizer enabled`. But `> 0` is cheaper than `!=`, with the optimizer enabled and outside a require statement. [See twwet](https://twitter.com/GalloDaSballo/status/1485430318706405378)_

```java

uint256 public gas;
function check1() external {
        gas = gasleft();
        require(99999999999999 != 0); // gas 22136 --disabled optimizer
        gas -= gasleft();
    }
function check2() external {
        gas = gasleft();
        require(99999999999999 > 0); // gas 22136 --disabled optimizer
        gas -= gasleft();
    }
function check3() external {
        gas = gasleft();
        if (99999999999999 != 0){ // 22149 gas --disabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
function check4() external {
        gas = gasleft();
        if (99999999999999 > 0){ // 22152 gas --disabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
```

```java
uint256 public gas;
function check1() external {
        gas = gasleft();
        require(99999999999999 != 0); // gas 22106 --enabled optimizer
        gas -= gasleft();
    }
function check2() external {
        gas = gasleft();
        require(99999999999999 > 0); // gas 22117 --enabled optimizer
        gas -= gasleft();
    }
function check3() external {
        gas = gasleft();
        if (99999999999999 != 0){ // 22106 gas --enabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
function check4() external {
        gas = gasleft();
        if (99999999999999 > 0){ // 22105 gas --enabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
```

### 11.2 `>=` is cheaper than `>`

_Non-strict inequalities (`>=`) are cheaper than strict ones (`>`). This is due to some supplementary checks (ISZERO, 3 gas))._

```java
uint256 public gas;
function checkStrict() external {
        gas = gasleft();
        require(999999999999999999 > 1); // gas 5017
        gas -= gasleft();
    }
function checkNonStrict() external {
        gas = gasleft();
        require(999999999999999999 >= 1); // gas 5006
        gas -= gasleft();
    }
```

<hr>
<br>

12. ## Usage of `uints`/`ints` smaller than 32 bytes (256 bits) incurs overhead

_When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size.
It is only beneficial to use reduced-size arguments if you are dealing with storage values because the compiler will pack multiple elements into one storage slot, and thus, combine multiple reads or writes into a single operation. When dealing with function arguments or memory values, there is no inherent benefit because the compiler does not pack these values. [Docs](https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html)_

Example:

```java
271: int128 amount; //@audit no storage slot saved
297: int128 internal constant iMAXTIME = 4 * 365 * 86400;
311: mapping(uint256 => int128) public slope_changes; // time -> signed slope change
320: uint8 public constant decimals = 18;
356:  uint8 internal constant _not_entered = 1;
357:	uint8 internal constant _entered = 2;
358:  uint8 internal _entered_state = 1;
697:  int128 old_dslope = 0;
698:  int128 new_dslope = 0;
749:  int128 d_slope = 0;
1169: int128 d_slope = 0;
```

Use a larger size then downcast where needed.

<hr>
<br>

13. ## Do not calculate constants

_Due to how constant variables are implemented (replacements at compile-time), an expression assigned to a constant variable is recomputed each time that the variable is used, which wastes some gas._

Example:

```java
48: uint256 constant dailyEmission = 600000 * 10**18;
57: uint256 constant secsInDay = 24 * 60 * 60;
296: uint256 internal constant MAXTIME = 4 * 365 * 86400;
297: int128 internal constant iMAXTIME = 4 * 365 * 86400;
```

<hr>
<br>

14. ## No need to evaluate all expressions to know if one of them is true

_When we have a code `expressionA || expressionB` if `expressionA` is true then `expressionB` will not be evaluated and gas saved._

Example wrong:

```java
650: bool senderIsOwner = (idToOwner[_tokenId] == msg.sender);
651: bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
652: require(senderIsOwner || senderIsApprovedForAll);
```

Variables bool `senderIsOwner` and bool `senderIsApprovedForAll` just add more work, it should be:

```java
require((idToOwner[_tokenId] == msg.sender) || (ownerToOperators[owner])[msg.sender];
```

so if `(idToOwner[_tokenId] == msg.sender)` is `true` we will not do SLOAD to get `(ownerToOperators[owner])[msg.sender]`; saving runtime and deployment gas.

<hr>
<br>

15. ## No need to read `tokenId` second time

Example wrong:

```java
948:        ++tokenId;
949:        uint256 _tokenId = tokenId;
```

Change it to `uint256 _tokenId = ++tokenId`; and 92 gas is saved this way

<hr>
<br>

16. ## `<array>.length` should not be looked up in every loop of a for-loop

_Reading array length at each iteration of the loop consumes more gas than necessary.
In the best-case scenario (length read on a memory variable), caching the array length in the stack saves around 3 gas per iteration. In the worst-case scenario (external calls at each iteration), the amount of gas wasted can be massive.
Consider storing the array’s length in a variable before the for-loop, and use this new variable instead.
Excluding the first loop costs, you also incur gas as follows:
storage arrays incur (100 gas)
memory arrays use MLOAD (3 gas)
calldata arrays use CALLDATALOAD (3 gas)_
Example wrong:

```java
143: for (uint256 index = 0; index < epochs.length; index++) {
157: for (uint256 index = 0; index < epochs.length; index++) {
180: for (uint256 tindex = 0; tindex < tokenids.length; tindex++) {
183: for (uint256 index = 0; index < epochs.length; index++) {
```

Code example:

```java
pragma solidity ^0.8.0;

contract EstimateGas {
    uint[] public fooArray;
    constructor (){
        for (uint256 i = 0; i < 500; ++i) {
            fooArray.push(i);
        }
    }
    function changeNoTemp() public{
        for (uint256 i = 0; i < fooArray.length; ++i) {
            fooArray[i] = fooArray[i] +1;
        }
    }
    function changeWithTemp() public{
        uint arrayLen = fooArray.length;
        for (uint256 i = 0; i < arrayLen; ++i) {
            fooArray[i] = fooArray[i] +1;
        }
    }
}
```

Gas execution costs:

```
changeNoTemp()
1) 2965426 gas
2) 2948326 gas
3) 2948326 gas

changeWithTemp()
1) 2911461 gas
2) 2894361 gas
3) 2894361 gas
```

<hr>
<br>

17. ## Use named variables in function returns

_When you dont use the named variables in your return statement, you waste deployment gas._

```java
return numGates;
```

<hr>
<br>

18. ## Don’t use SafeMath for >0.8.0

_Solidity version 0.8.0 introduces internal overflow checks, so using SafeMath is redundant and adds overhead._

<hr>
<br>

19. ## Don’t compare boolean expressions to boolean literals

```java
if (<x> == true)
if (<x>)
if (<x> == false)

// instead use
 if (!<x>)
```

### 19.1 Using bools for storage incurs overhead

_Use uint256(1) and uint256(2) for true/false to avoid a Gwarmaccess (100 gas) for the extra SLOAD, and to avoid Gsset (20000 gas) when changing from ‘false’ to ‘true’, after having been ‘true’ in the past_

Example:

```java
-  93 :-        if(vault.idExists(epochEnd) == false)
+  93 :+        if(vault.idExists(epochEnd) == 0)

- 211 :-        if(insrVault.idExists(epochEnd) == false || riskVault.idExists(epochEnd) == false)
+ 211 :+        if(insrVault.idExists(epochEnd) == 0 || riskVault.idExists(epochEnd) == 0)

- 54 :-    mapping(uint256 => bool) public idExists;
+ 54 :+    mapping(uint256 => uint256) public idExists;

- 80 :-        if(idExists[id] != true)
+ 80 :+        if(idExists[id] != 1)

- 314 :-        if(idExists[epochEnd] == true)
+ 314 :+        if(idExists[epochEnd] == 1)
```

<hr>
<br>

20. ## Use a more recent version of solidity

-   Use a solidity version of at least 0.8.0 to get overflow protection without SafeMath.
-   Use a solidity version of at least 0.8.2 to get compiler automatic inlining.
-   Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads.
-   Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than revert()/require() strings.
-   Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value.

<hr>
<br>

21. ## Shift Right instead of Dividing by 2

_A division by 2 can be calculated by shifting one to the right.
While the DIV opcode uses 5 gas, the SHR opcode only uses 3 gas. Furthermore, Solidity’s division operation also includes a division-by-0 prevention which is bypassed using shifting._

Replace `/2` with `>> 1`

<hr>
<br>

22. ## Use Modifiers Instead of Functions To Save Gas

Example of two contracts with modifiers and internal view function:

```java
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
contract Inlined {
    function isNotExpired(bool _true) internal view {
        require(_true == true, "Exchange: EXPIRED");
    }
function foo(bool _test) public returns(uint){
            isNotExpired(_test);
            return 1;
    }
}
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
contract Modifier {
modifier isNotExpired(bool _true) {
        require(_true == true, "Exchange: EXPIRED");
        _;
    }
function foo(bool _test) public isNotExpired(_test)returns(uint){
        return 1;
    }
}
```

Differences:

Deploy Modifier.sol: 108727

Deploy Inlined.sol: 110473

Modifier.foo: 21532

Inlined.foo: 21556

<br>
<hr>

23. ## Save gas with the use of the import statement
    _With the import statement, it saves gas to specifically import only the parts of the contracts, not the complete ones._

```java
6: import "@openzeppelin/contracts/token/ERC1155/IERC1155.sol";
```

Description:

Solidity code is also cleaner in another way that might not be noticeable: the struct Point. We were importing it previously with global import but not using it. The Point struct `polluted the source code` with an unnecessary object we were not using because we did not need it.

This was breaking the rule of modularity and modular programming: `only import what you need` Specific imports with curly braces allow us to apply this rule better.

Recommendation:
`import {contract1 , contract2} from "filename.sol";`

```java
import {ERC1155, ERC1155TokenReceiver} from “solmate/tokens/ERC1155.sol”;
```

<br>
<hr>

24. ## Use custom errors (Non-critical)
    _Use custom errors to save deployment and runtime costs in case of revert._
    _Instead of using strings for error messages (e.g., `require(msg.sender == owner, “unauthorized”)`), you can use custom errors to reduce both deployment and runtime gas costs. In addition, they are very convenient as you can easily pass dynamic information to them._

Before:

```java
function add(uint256 _amount) public {
    require(msg.sender == owner, "unauthorized");

    total += _amount;
}
```

After:

```java
error Unauthorized(address caller);

function add(uint256 _amount) public {
    if (msg.sender != owner)
        revert Unauthorized(msg.sender);

    total += _amount;
}
```

### 24.1 Use `revert` instead of `require`. (Use custom errors)

\_\_

<br>
<hr>

25. ## Use `indexed` events as they are less costly compared to non-indexed ones [[Ref](https://twitter.com/maurelian_/status/1488691543544320000)] - Event is missing `indexed` fields. (Non-critical)

    _Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it’s not necessarily best to index the maximum allowed per event (threefields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. `If there are fewer than three fields, all of the fields should be indexed`._

    _Using the `indexed` keyword for value types such as `uint`, `bool`, and `address` saves gas costs, as seen in the example below. However, this is only the case for value types, whereas indexing `bytes` and `strings` are more expensive than their unindexed version._

Before:

```java
event Withdraw(uint256, address);

function withdraw(uint256 amount) public {
    emit Withdraw(amount, msg.sender);
}
```

After:

```java
event Withdraw(uint256 indexed, address indexed);

function withdraw(uint256 amount) public {
    emit Withdraw(amount, msg.sender);
}
```

<br>
<hr>

26. # Using `private` rather than `public` for constants, saves gas.
    _If needed, the value can be read from the verified contract source code. Savings are due to the compiler not having to create non-payable getter functions for deployment calldata, and not adding another entry to the method ID table._

<br>
<hr>

27. # Unnecessary look up in `if` condition
    _If the `||` condition isn’t required, the second condition will have been looked up unnecessarily._

```java
95: if (notEnoughRetryInvocations || notEnoughTimeReachingMaxFailedAttempts) {...}
```

<br>
<hr>

28. # Don’t use `SafeMath` for `>0.8.0`

    _Solidity version 0.8.0 introduces internal overflow checks, so using SafeMath is redundant and adds overhead._

<br>
<hr>

29. # Using XOR (`^`) and OR (`|`) bitwise equivalents.

-   `(a != b || c != d || e != f)` costs 571
-   `((a ^ b) | (c ^ d) | (e ^ f)) != 0` costs 498 (saving 73 gas)

Consider rewriting as follows to save gas:

```java
  93:     if (
- 94:         execution.item.itemType != considerationItem.itemType ||
- 95:         execution.item.token != considerationItem.token ||
- 96:         execution.item.identifier != considerationItem.identifier
+ 94:         ((uint8(execution.item.itemType) ^ uint8(considerationItem.itemType)) |
+ 95:         (uint160(execution.item.token) ^ uint160(considerationItem.token)) |
+ 96:         (execution.item.identifier ^ considerationItem.identifier)) != 0
  97:       ) {
```

## Logic POC

-   Given 4 variables `a`, `b`, `c` and `d` represented as such:

```java
0 0 0 0 0 1 1 0 <- a
0 1 1 0 0 1 1 0 <- b
0 0 0 0 0 0 0 0 <- c
1 1 1 1 1 1 1 1 <- d
```

To have `a == b` means that every `0` and `1` match on both variables. Meaning that a XOR (operator `^`) would evaluate to 0 (`(a ^ b) == 0`), as it excludes by definition any equalities.
Now, if `a != b`, this means that there’s at least somewhere a `1` and a `0` not matching between `a` and `b`, making `(a ^ b) != 0`.

Both formulas are logically equivalent and using the XOR bitwise operator costs actually the same amount of gas:

```java
function xOrEquivalence(uint a, uint b) external returns (bool) {
        //return a != b; //370
        //return a ^ b != 0; //370
```

However, it is much cheaper to use the bitwise OR operator (`|`) than comparing the truthy or falsy values:

```java
function xOrOrEquivalence(uint a, uint b, uint c, uint d) external returns (bool) {
        //return (a != b || c != d); // 495
        //return (a ^ b | c ^ d) != 0; // 442
    }
```

These are logically equivalent too, as the OR bitwise operator (`|`) would result in a `1` somewhere if any value is not `0` between the XOR (`^`) statements, meaning if any XOR (`^`) statement verifies that its arguments are different.

Recommended mittigations steps:

-   Consider applying the suggested equivalence and add a comment mentioning what this is equivalent to, as this is less human-readable, but still understandable once it’s been taught.

<br>
<hr>

30. # Shift left by 5 instead of multiplying by 32. [Docs](https://docs.soliditylang.org/en/v0.5.3/types.html#shifts)
    _The equivalent of multiplying by 32 is shifting left by 5. On Remix, a simple POC shows some by replacing one with the other (Optimizer at 10k runs):_

```java
function shiftLeft5(uint256 a) public pure returns (uint256) {
        //unchecked { return a * 32; } //346
        //unchecked { return a << 5; } //344
    }
```

This is due to the fact that the MUL opcode costs 5 gas and the SHL opcode costs 3 gas. Therefore, saving those 2 units of gas is expected.

Places where this optimization can be applied are as such:

-   A simple multiplication by 32:

```java
- 220:    terminalMemoryOffset = (totalOrders + 1) * 32; //@audit-issue << 5
+ 220:    terminalMemoryOffset = (totalOrders + 1) << 5;
```

-   Multiplying by the constant `OneWord == 0x20`, as `0x20` in hex is actually `32` in decimals:

```java
-  386:   uint256 tailOffset = arrLength * OneWord;
+  386:   uint256 tailOffset = arrLength << 5;
```

<br>
<hr>

# 31. `10 ** 18` can be changed to `1e18` and save some gas.

_`10 ** 18` can be changed to `1e18` to aavoid unnecessary arithmetic operation and save gas_

# 32. Use assembly when getting a contract’s balance of ETH.

_You can use `selfbalance()` instead of `address(this).balance` when getting your contract’s balance of ETH to save gas. Additionally, you can use `balance(address)` instead of `address.balance()` when getting an external contract’s balance of ETH._

```java
contract Contract0 {
    function addressInternalBalance() public returns (uint256) {
        return address(this).balance;
    }
```

<br>
<hr>

# 33. Replacing `modifiers` with `private` functions. - Using `private` functions instead of `modifier` might be better.

_Can optimize their contracts bytecode size._

[See info this tweet](https://twitter.com/zaryab_eth/status/1652625870874451968)

-   𝐌𝐨𝐝𝐢𝐟𝐢𝐞𝐫𝐬 𝐚𝐫𝐞 𝐞𝐱𝐩𝐚𝐧𝐝𝐞𝐝 𝐢𝐧𝐥𝐢𝐧𝐞:

➡️ When you use a modifier, Solidity copies the entire code of the modifier into every function that calls it.

This means that the modifier code becomes a part of the function and is included in the bytecode generated for that function.

This technically means that the modifier code is replicated in every function where the modifier is used, which can increase the size of the contract's bytecode.

-   𝐏𝐫𝐢𝐯𝐚𝐭𝐞 𝐟𝐮𝐧𝐜𝐭𝐢𝐨𝐧𝐬 𝐚𝐫𝐞 𝐨𝐧𝐥𝐲 𝐰𝐫𝐢𝐭𝐭𝐞𝐧 𝐨𝐧𝐜𝐞:

➡️ On the other hand, a private function is only written once and can be called from multiple places in your contract without increasing its size.

➡️ This makes it a much more efficient way to perform checks without duplicating code.

Note: In fact, if your contract is fairly smaller and is quite below the maximum contract size threshold of 24.576 kb, modifiers shouldn’t even be a problem for you.
However, considering a bulky contract where a check/validation is supposed to be used multiple times which unnecessarily increases your contract’s size, then using `private` functions instead of `modifier` might be better.

<br>
<hr>

## 34. Revert as early as possible.

_You pay gas up intil the REVERT opcode is hit._

_You pay gas for everything before the `require()` in the revert case._

<br>
<hr>

## 35. Avoid comparing Boolean Expressions to Boolean Literals

```java
if (<x> == true) => if (<x>)

if (<x> == false) => if (!<x>)
```

<br>
<hr>

## 36. Inverting the condition of an `if-else-statement` wastes gas.

_Flipping the true and false blocks instead saves [3 gas](https://gist.github.com/IllIllI000/44da6fbe9d12b9ab21af82f14add56b9)._

<br>
<hr>

## 37. Functions guaranteed to revert when called by normal users can be marked `payable`.

_If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are `CALLVALUE(2)`, `DUP1(3)`, `ISZERO(3)`, `PUSH2(3)`, `JUMPI(10)`, `PUSH1(3)`, `DUP1(3)`, `REVERT(0)`, `JUMPDEST(1)`, `POP(2)`, which costs an average of about 21 gas per call to the function, in addition to the extra deployment cost._

<br>
<hr>

## 38. State variables only set in the constructor should be declared `immutable`.

_Avoids a Gsset (20000 gas) in the constructor, and replaces each Gwarmacces (100 gas) with a `PUSH32` (3 gas)._

_Avoids a Gsset (20000 gas) in the constructor, and replaces the first access in each transaction (Gcoldsload - 2100 gas) and each access thereafter (Gwarmacces - 100 gas) with a PUSH32 (3 gas)._

```java
contract PriceOracleImplementation is PriceOracle {
    address public cEtherAddress;

    constructor(address _cEtherAddress) public {
        cEtherAddress = _cEtherAddress;
    }
```

<br>
<hr>

## 39. Use assembly to check for `address(0)`.

```java
function ownerNotZero(address _addr) public pure {
    require(_addr != address(0), "zero address)");
}
```

Gas: 258

```java
function assemblyOwnerNotZero(address _addr) public pure {
    assembly {
        if iszero(_addr) {
            mstore(0x00, "zero address")
            revert(0x00, 0x20)
        }
    }
}
```

Gas: 252

_`isZero` and `mstore` we’ve already seen it above, so let’s check now what is the explanation for `revert`._

![](/Img/revert.webp)

<br>
<hr>

## 40. When possible, use assembly instead of `unchecked{++i}`.

_You can also use `unchecked{++i;}` for even more gas savings but this will not check to see if `i` overflows. For best gas savings, use inline assembly, however this limits the functionality you can achieve._

```java
//loop with unchecked{++i}
function uncheckedPlusPlusI() public pure {
    uint256 j = 0;
    for (uint256 i; i < 10; ) {
        j++;
        unchecked {
            ++i;
        }
    }
}
```

Gas: 1329

```java
//loop with inline assembly
function inlineAssemblyLoop() public pure {
    assembly {
        let j := 0
        for {
            let i := 0
        } lt(i, 10) {
            i := add(i, 0x01)
        } {
            j := add(j, 0x01)
        }
    }
}
```

Gas: 709

Now, the explanation of `lt` Yul instruction and similar and explanation:
![](/Img/it-Yul.webp)

## 40.1 For Loops & If Statements.

Example:

```java
function howManyEvens(uint256 startNum, uint256 endNum) external pure returns(uint256) {

    // the value we will return
    uint256 ans;

    assembly {

        // syntax for for loop
        for { let i := startNum } lt( i, add(endNum, 1)  ) { i := add(i,1) }
        {
            // if i == 0 skip this iteration
            if iszero(i) {
                continue
            }

            // checks if i % 2 == 0
            // we could of used iszero, but I wanted to show you eq()
            if  eq( mod( i, 2 ), 0 ) {
                ans := add(ans, 1)
            }

        }

    }


    return ans;

}
```

_The syntax for `if` statements is very similar to solidity, however, we do not need to wrap the condition in parentheses. For the `for` loop, notice we are using brackets when declaring `i` and incrementing `i`, but not when we evaluate the condition. Additionally, we used a `continue` to skip an iteration of the loop. We can also use `break` statements in Yul, as well._

## 41. State variables only set in the constructor should be declared `immutable`.

_Avoids a Gsset (20000 gas) in the constructor, and replaces the first access in each transaction (Gcoldsload - 2100 gas) and each access thereafter (Gwarmacces - 100 gas) with a `PUSH32` (3 gas)._

_In the example below, `twa.lookback` never changes, so it should be pulled out of the struct and converted to an immutable value. If you still want the struct to contain the lookback, create a private version of the twa struct, for storage purposes, that does not have the `lookback`, and use that to build the original twa structs on the fly, when requested, and pass the value as an argument to `TWA.getTwa()` when it’s called. Since this is in the hot path, you’ll save a lot of gas._

Example:

```java
File: maverick-v1/contracts/models/Pool.sol
/// @audit twa.lookback (constructor)
77:           twa.lookback = _lookback;
/// @audit twa (access)
102:          return twa;
```

<br>
<hr>
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
