# My Ethernaut Notes: Levels 1-12

This repository contains my personal notes and solutions for the first 12 levels of the [Ethernaut](https://ethernaut.openzeppelin.com/) wargame. Each level highlights a common Ethereum smart contract vulnerability or attack vector.

---

## Level 1: Fallback

* **Vulnerability:** Misunderstanding `fallback()` and `receive()` functions.
* **Key Concept:** In Solidity `< 0.8.0`, if a contract receives Ether with no `msg.data`, the `receive()` function is executed. If `receive()` does not exist but a `payable` `fallback()` function does, the `fallback()` will be executed instead.
* **Solution:**
    1.  Call the `contribute()` function to become a contributor.
    2.  Send a transaction with a value greater than 0 (e.g., `1 wei`) *directly* to the contract's address (with no `msg.data`).
    3.  This triggers the `payable` `fallback()` function, which assigns us as the `owner`.
    4.  Call the `withdraw()` function to take all the funds.

---

## Level 2: Fallout

* **Vulnerability:** A typo in the constructor function's name.
* **Key Concept:** Before Solidity `0.4.22`, constructors were functions that had the *exact same name* as the contract. A typo (e.g., `Fal1out` instead of `Fallout`) makes it a regular, public function that anyone can call.
* **Solution:** Simply call the misnamed `Fal1out()` function. This "constructor" will then run and assign us as the `owner`.

---

## Level 3: Coin Flip

* **Vulnerability:** Insecure on-chain randomness.
* **Key Concept:** Using public, predictable blockchain variables like `block.timestamp` or `blockhash` as a source of randomness is insecure. An attacker can read the same variable in their own contract and predict the "random" outcome within the same transaction.
* **Solution:**
    1.  Create an attacker contract (`CoinFlipAttacker`).
    2.  In the attacker contract, create a function that reads the same block variables (`blockhash`, `block.timestamp`) as the `CoinFlip` contract.
    3.  Perform the *exact same calculation* as the `CoinFlip` contract to predict the outcome.
    4.  Call the `flip()` function on the `CoinFlip` contract with the correct, predicted guess.
    5.  Repeat this 10 times.

---

## Level 4: Telephone

* **Vulnerability:** Misunderstanding `msg.sender` vs. `tx.origin`.
* **Key Concept:**
    * `tx.origin`: This is *always* the original Externally Owned Account (EOA) that signed and started the transaction chain.
    * `msg.sender`: This is the *immediate* caller. It can be an EOA or another contract.
* **Solution:** The `Telephone` contract requires `msg.sender != tx.origin` to change the owner. To do this, we create an attacker contract (`TelephoneAttacker`) and call a function on it. This attacker contract then calls the `changeOwner()` function on the `Telephone` contract.
    * In this scenario: `tx.origin` is *you* (your EOA), but `msg.sender` is the `TelephoneAttacker` contract. The check passes.

---

## Level 5: Token

* **Vulnerability:** Integer Underflow.
* **Key Concept:** In Solidity versions `< 0.8.0`, arithmetic operations do *not* automatically check for underflow or overflow. Subtracting 1 from a `uint` with a value of 0 will cause it to "underflow" and wrap around to the maximum possible value (`2**256 - 1`).
* **Solution:** We are given 20 tokens. We call `transfer()` to send 21 tokens (or any amount > 20) to another address. The line `balances[msg.sender] -= _value` will underflow (e.g., `20 - 21`), giving our account a massive token balance.

---

## Level 6: Delegation

* **Vulnerability:** Improper use of `delegatecall`.
* **Key Concept:** `delegatecall` is a powerful but dangerous opcode. It executes the code of a target contract (the "library") in the *context* of the calling contract. This means the target's code modifies the *calling contract's storage*. The attack works if the storage layout of the two contracts is not identical.
* **Solution:**
    1.  The `Delegation` contract (caller) has its `owner` variable in storage **slot 0**.
    2.  The `Delegate` contract (target) has a public function `pwn()` which writes to *its* `owner` variable, which is also in **slot 0**.
    3.  We send a transaction to the `Delegation` contract, using the function selector for `pwn()` as the `msg.data`.
    4.  `Delegation`'s `fallback()` function executes `delegatecall(msg.data)`.
    5.  This runs the `pwn()` function *in the storage context of `Delegation`*, modifying `Delegation`'s **slot 0** and making us the owner.

---

## Level 7: Force

* **Vulnerability:** A contract cannot refuse Ether sent via `selfdestruct()`.
* **Key Concept:** Even if a contract has no `payable` functions, no `receive()`, and no `payable` `fallback()`, it can be forcibly sent Ether. The `selfdestruct(address)` function will send all of the contract's balance to the target address, no matter what.
* **Solution:**
    1.  Create an attacker contract (`ForceAttacker`).
    2.  Fund this new contract with some Ether (e.g., `1 wei`).
    3.  Add a public function to `ForceAttacker` that calls `selfdestruct(payable(address_of_Force_contract))`.
    4.  Call this function. The `ForceAttacker` contract is destroyed, and its balance is forcibly transferred to the `Force` level instance, completing the level.

---

## Level 8: Vault

* **Vulnerability:** `private` data is not private on the blockchain.
* **Key Concept:** All data stored on-chain (including state variables marked `private`) is publicly visible. The `private` keyword only prevents other *contracts* from reading the variable directly. It does not hide it from external observers.
* **Solution:**
    1.  Use `web3.eth.getStorageAt(contractAddress, slotNumber)` to read the contents of the contract's storage slots.
    2.  The `locked` variable is in **slot 0**.
    3.  The `password` variable is in **slot 1**.
    4.  Read the data from slot 1: `const password = await web3.eth.getStorageAt(contract.address, 1)`.
    5.  Call `await contract.unlock(password)` with the retrieved value to unlock the vault.

---

## Level 9: King

* **Vulnerability:** A `payable` function that does not check for re-entrancy or failure.
* **Key Concept:** When a contract sends Ether (e.g., via `_king.transfer(msg.value)`), it transfers execution to the recipient's `receive()` or `fallback()` function. If that function `revert()`s, the entire transaction reverts, potentially locking the calling contract.
* **Solution:**
    1.  Create an attacker contract (`KingAttacker`) with a `payable` `fallback()` (or `receive()`) function that **always reverts**.
    2.  Call `becomeKing()` from this contract, sending an amount of Ether *greater* than the current prize.
    3.  Our `KingAttacker` contract is now the new king.
    4.  When the *next* player tries to become king, the `King` contract will try to send the prize to our `KingAttacker` contract.
    5.  Our `fallback()`/`receive()` will execute and `revert()`, causing the *entire transaction* from the new player to fail.
    6.  No one else can ever become king. This is a "Denial of Service" (DoS) attack.

---

## Level 10: Re-entrancy

* **Vulnerability:** The "Checks-Effects-Interactions" pattern is violated.
* **Key Concept:** A function should: (1) **Check** conditions, (2) **Effect** state changes (like updating balances), and *then* (3) **Interact** with external contracts (like sending Ether). This contract interacts *before* effecting its state change.
* **Solution:**
    1.  Create an attacker contract (`ReentranceAttacker`).
    2.  **Step 1 (Donate):** Call `donate()` on the `Reentrance` contract, giving our attacker contract a small balance (e.g., 0.001 ETH).
    3.  **Step 2 (Withdraw):** Call `withdraw(0.001)` from our attacker contract.
    4.  **Step 3 (Re-enter):** The `Reentrance` contract sends 0.001 ETH back to us. This triggers our `receive()` function.
    5.  **Inside `receive()`:** The `Reentrance` contract *has not yet updated our balance to 0*. Its code is paused. Our `receive()` function calls `withdraw(0.001)` *again*.
    6.  The `Reentrance` contract checks our balance (which is *still* 0.001) and sends us *another* 0.001 ETH.
    7.  This repeats until the `Reentrance` contract is empty.

---

## Level 11: Elevator

* **Vulnerability:** Trusting an external contract to be stateless.
* **Key Concept:** An external contract's function (`isLastFloor`) can be stateful and is not guaranteed to return the same value twice, even within the same transaction.
* **Solution:**
    1.  Create an attacker contract (`ElevatorAttacker`) that implements the `Building` interface.
    2.  Implement the `isLastFloor()` function to be stateful: it should return `false` the *first time* it's called and `true` the *second time* it's called.
    3.  Call the `Elevator.goTo(1)` function from our attacker contract.
    4.  The `Elevator` calls our `isLastFloor()` **(Call 1)**. We return `false`, passing the `if` check.
    5.  The `Elevator` sets `floor = 1`.
    6.  The `Elevator` calls our `isLastFloor()` again **(Call 2)**. We return `true`.
    7.  The `Elevator` sets `top = true`.

---

## Level 12: Privacy

* **Vulnerability:** `private` data in complex storage layouts is still public.
* **Key Concept:** Data is packed into 32-byte storage slots. `bool`, `uint8`, `bytes16`, etc., are often packed together. We can still read the entire 32-byte slot to extract the "private" data.
* **Solution:**
    1.  The `locked`, `ID`, `flattening`, `denomination`, and `awkwardness` variables are all packed into **slot 0**.
    2.  The `data` array starts at **slot 1**.
    3.  The contract requires `data[2]`, which is stored in `slot 1 + 2 = 3`.
    4.  Read the raw data from **slot 3**: `const slot3Data = await web3.eth.getStorageAt(contract.address, 3)`.
    5.  This 32-byte value is our `key`. The `unlock` function expects a `bytes16`, so we must truncate the 32-byte value.
    6.  Cast the 32-byte hex string to a 16-byte hex string (e.g., `key.slice(0, 34)` in JS).
    7.  Call `await contract.unlock(key16)` to pass the level.