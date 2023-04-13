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

5. ## Using `calldata` instead of memory for read-only arguments in `external` functions saves gas

_If you choose to make suggested above `public` functions as `external`, to continue gas optimizaton we can use `calldata` function arguments instead of `memory`.
When a function with a `memory` array is called externally, the `abi.decode()` step has to use a for-loop to copy each index of the calldata to the memory index. Each iteration of this for-loop costs at least 60 gas (i.e. `60 * <mem_array>.length`). Using `calldata` directly, obliviates the need for such a loop in the contract code and runtime execution. Structs have the same overhead as an array of length one._

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

<hr>
<br>

7. ## Splitting `require()` statements that use `&&` saves gas

_Instead of using the `&&` operator in a single require statement to check multiple conditions, I suggest using multiple require statements with 1 condition per require statement (saving 3 gas per `&`).
See [this issue](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) which describes the fact that there is a larger deployment gas cost, but with enough runtime calls, the change ends up being cheaper._

Example:

```java
239: require(attachments[_tokenId] == 0 && !voted[_tokenId], 'attached');
```

<hr>
<br>

8. ## `abi.encode()` is less efficient than `abi.encodePacked()`
    \_`abi.encode` will apply [ABI encoding rules](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html). Therefore all elementary types are padded to 32 bytes and dynamic arrays include their length. Therefore it is possible to also decode this data again (with `abi.decode`) when the type are known.

`abi.encodePacked` will only use the minimal required memory to encode the data. E.g. an address will only use 20 bytes and for dynamic arrays only the elements will be stored without length. For more info see the [Solidity docs for packed mode](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html?highlight=encodepacked#non-standard-packed-mode).\_

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

11. ## Using `> 0` costs more gas than `!= 0` when used on a `uint` in a `require()` and `assert` statement

_`> 0` is less efficient than `!= 0` for unsigned integers._
_`!= 0` costs less gas compared to `> 0` for unsigned integers in `require` and `assert` statements with the optimizer enabled (6 gas)._

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

_`!= 0` costs less gas compared to `> 0` for unsigned integers in require statements with the optimizer enabled. But `> 0` is cheaper than `!=`, with the optimizer enabled and outside a require statement. [See twwet](https://twitter.com/GalloDaSballo/status/1485430318706405378)_

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

_When you don;t use the named variables in your return statement, you waste deployment gas._

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

<hr>
<br>

19. ## Use a more recent version of solidity

-   Use a solidity version of at least 0.8.0 to get overflow protection without SafeMath.
-   Use a solidity version of at least 0.8.2 to get compiler automatic inlining.
-   Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads.
-   Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than revert()/require() strings.
-   Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value.

<hr>
<br>

20. ## Shift Right instead of Dividing by 2

_A division by 2 can be calculated by shifting one to the right.
While the DIV opcode uses 5 gas, the SHR opcode only uses 3 gas. Furthermore, Solidity’s division operation also includes a division-by-0 prevention which is bypassed using shifting._

Replace `/2` with `>> 1`

<hr>
<br>

21. ## Use Modifiers Instead of Functions To Save Gas

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
<br>

based on real reports [Code4arena](https://code4rena.com/reports)