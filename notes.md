# Ethernaut Solutions & Learnings

## Level 1: Fallback

**Vulnerability:** Unprotected receive() function that assigns ownership

**Exploit:**
1. Make small contribution (satisfy require in receive)
2. Send ETH directly to contract (triggers receive())
3. Owner changed without proper checks
4. Withdraw funds

**Key Learning:** 
- receive() and fallback() functions need same security as regular functions
- Direct ETH transfers can trigger unexpected code
- Always validate state changes in fallback functions

**Prevention:**
- Add proper access control to receive()/fallback()
- Avoid state changes in these functions when possible
- Use explicit functions for critical operations

## Level 2: Fallout

**Vulnerability:** Typo in constructor name (Fal1out vs Fallout)

**Exploit:** 
Simply call the public function that was meant to be a constructor.

**Key Learning:**
- Pre-Solidity 0.4.22: constructors were functions with same name as contract
- Easy to make typos (especially with similar characters like 1/l, 0/O)
- Modern Solidity uses `constructor` keyword (safer)

**Real-world impact:**
Rubixi contract (2016) had exactly this bug. Constructor was renamed 
but old initialization function remained public. Attacker claimed ownership 
and drained contract.

**Prevention:**
- Use modern Solidity (0.4.22+) with `constructor` keyword
- Thorough code review for typos in critical functions
- Automated tools should flag public functions matching contract name

## Level 3: Coin Flip

**Vulnerability:** Predictable randomness using blockhash

**Exploit:**
Create attack contract that:
1. Calculates same blockhash-based value
2. Determines winning guess
3. Calls target contract with "prediction"
4. Repeat 10 times (consecutive wins)

**Key Learning:**
- blockhash is PUBLIC information (anyone can read it)
- All on-chain data is deterministic and visible
- "Random" numbers from blockhash can be predicted
- Attacker in same block has same blockhash

**Prevention:**
- Use Chainlink VRF (Verifiable Random Function)
- Commit-reveal schemes
- Off-chain randomness with cryptographic proofs
- Never use blockhash/timestamp for critical randomness

## Level 4: Telephone

**Vulnerability:** Using tx.origin for authentication

**Exploit:**
Call target contract through intermediate contract so:
- tx.origin = your wallet
- msg.sender = intermediate contract
- Check passes, ownership transferred

**Key Learning:**
- tx.origin = original EOA in transaction chain
- msg.sender = immediate caller
- tx.origin should NEVER be used for authentication
- Always use msg.sender for access control

**Prevention:**
- Always use msg.sender for authentication
- Linters should flag tx.origin usage
- Code reviews should catch this pattern

## Level 5: Token

**Vulnerability:** Integer underflow (pre-Solidity 0.8)

**Exploit:**
Transfer more tokens than you have. Subtraction underflows, wrapping 
around to maximum uint256 value.

**Key Learning:**
- Pre-Solidity 0.8: no automatic overflow/underflow protection
- uint256 - 1 when uint256 = 0 → wraps to 2^256-1
- require() check happens AFTER calculation (too late!)
- SafeMath library was used to prevent this

**Real-world impact:**
- BEC Token (2018): underflow bug allowed attacker to generate unlimited tokens

**Prevention:**
- Use Solidity 0.8+ (built-in protection)
- Or use OpenZeppelin SafeMath library
- Check BEFORE arithmetic operations

**Modern Solidity:**
```solidity
// Solidity 0.8+ would revert automatically
function transfer(address _to, uint _value) public returns (bool) {
    balances[msg.sender] -= _value; // Reverts if underflow
    balances[_to] += _value;
    return true;
}
```