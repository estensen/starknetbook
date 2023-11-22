# Security Considerations

When working with any blockchain programming language, it's crucial to be aware of potential vulnerabilities in smart contracts in order to protect your projects from threats that could compromise the trust users place in your systems. Cairo is no exception.

This section will explain some common security issues and vulnerabilities specific to Starknet and Cairo, and provide recommendations on how to prevent them from affecting your contracts.

Contributions to this chapter are welcome. If you have any suggestions, please submit a pull request to the [Book repo](https://github.com/starknet-edu/starknetbook).

> Please note that some of the code examples provided in this chapter are written in pseudo-code for the sake of simplicity and clarity when explaining the concepts. They are not meant to be used in production.

## 1. Access Control

Access control vulnerabilities arise when a smart contract's functions are inadequately secured, allowing unauthorized users to perform actions that should be restricted. This can lead to unintended smart contract behavior and manipulation of sensitive data.

For example, consider a smart contract that handles token minting without proper access control:

```rust
#[starknet::contract]
mod Token {
    #[storage]
    struct Storage {
        total_supply: u256,
    }

    #[external(v0)]
    impl ITokenImpl of IToken {
        fn mint_tokens(ref self: ContractState, amount: u256) {
            self.total_supply.write(self.total_supply.read() + amount);
        }
    }
}
```

In this example, any user can call the `mint_tokens` function and mint new tokens, potentially leading to an exploit or manipulation of the token supply.

### Recommendation

To mitigate access control vulnerabilities, implement proper authorization mechanisms such as role-based access control (RBAC) or ownership checks. You can create your own custom solution or use existing templates like those provided by <a href="https://docs.openzeppelin.com/contracts-cairo/access" target="_blank">OpenZeppelin</a>.

In the example above, we can add an owner variable, set the owner value in the constructor, and add an assert condition in the `mint_tokens` function to ensure that only the owner can mint new tokens.

```rust
#[starknet::contract]
mod Token {

    #[storage]
    struct Storage {
        owner: ContractAddress,
        total_supply: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState,) {
        let sender = get_caller_address();
        self.owner.write(sender);
    }

    #[external(v0)]
    impl ITokenImpl of IToken {
        fn mint_tokens(ref self: ContractState, amount: u256) {
            // Now, only owner can mint
            let sender = get_caller_address();
            assert(sender == self.owner.read());

            self.total_supply.write(self.total_supply.read() + amount);
        }
    }
}
```

By implementing proper access control, you can ensure that your smart contract functions are only executed by authorized parties, reducing the risk of unauthorized manipulation.

## 2. Reentrancy

Reentrancy vulnerabilities occur when a smart contract's function calls an external contract before updating its internal state, allowing the external contract to recursively call the initial function before it has completed execution.

For example, consider a game contract where whitelisted addresses can mint an NFT sword and execute an `on_receive_sword()` function to use it before returning it to the contract. However, the NFT contract is vulnerable to reentrancy attacks, allowing an attacker to mint multiple NFT swords.

```rust
#[storage]
struct Storage {
    available_swords: u256,
    sword: LegacyMap::<ContractAddress, u256>,
    whitelisted: LegacyMap::<ContractAddress, u256>,
    ...
    ...
}

#[constructor]
fn constructor(ref self: ContractState,) {
    self.available_swords.write(100);
}

#[external(v0)]
impl IGameImpl of IGame {
    fn mint_one_sword(ref self: ContractState) {
        let sender = get_caller_address();
        if self.whitelisted.read(sender) == true {
            // Update sword count
            let sword_count = self.available_swords.read();
            self.available_swords.write(sword_count - 1);
            // Mint one sword to caller
            self.sword.write(sender, 1);
            // Callback to sender
            let callback = ICallerDispatcher { contract_address: sender }.on_receive_sword();
            // Remove sender from whitelist
            self.whitelisted.write(sender, false);
        }
}
```

An attacker's contract can implement the `on_receive_sword` function to exploit the reentrancy vulnerability and mint multiple swords calling `mint_one_sword` again previous to remove the sender from `whitelisted`:

```rust
fn on_receive_sword(ref self: ContractState) {
    let nft_sword_contract = get_caller_address();
    let call_number: felt252 = self.total_calls.:read();
    self.total_calls.write(call_number + 1);
    if call_number < 10 {
        let call = ISwordDispatcher { contract_address: nft_sword_contract }.mint_one_sword();
    }
}
```

Reentrancy controls may need to be implemented in callback functions in many ERC standards with `safeTransfer` functions (ERC721, ERC777, ERC1155, ERC223, etc.) or in flash loans where lender contracts callback the borrower contract to use and return funds.

### Recommendation:

To mitigate reentrancy vulnerabilities, follow the check-effects-interactions pattern, ensuring that you update the relevant internal state before calling external contracts. In the example above, remove the sender from the whitelist before calling the external function.

```rust
if self.whitelisted.read(sender) == true {
    // Update sword count
    let sword_count = self.available_swords.read();
    self.available_swords.write(sword_count - 1);
    // Mint one sword to caller
    self.sword.write(sender, 1);
    // Remove sender from whitelist (before calling external function)
    self.whitelisted.write(sender, false);
    // Callback to sender (after setting all effects)
    let callback = ICallerDispatcher { contract_address: sender }.on_receive_sword();
}
```

By following the check-effects-interactions pattern, you can reduce the risk of reentrancy attacks, ensuring the integrity of your smart contract's internal state.

## 3. Tx.Origin Authentication

In Solidity, `tx.origin` is a global variable that stores the address of the transaction initiator, while `msg.sender` stores the address of the transaction caller. In Cairo, we have the `account_contract_address` global variable and `get_caller_address` function, which serve the same purpose.

Using `account_contract_address` (the equivalent of `tx.origin`) for authentication in your smart contract functions can lead to phishing attacks. Attackers can create custom smart contracts and trick users into placing them as intermediaries in a transaction call, effectively impersonating the contract owner.

For example, consider a Cairo smart contract that allows transferring funds to the owner and uses `account_contract_address` for authentication:

```rust
use starknet::get_caller_address;
use box::BoxTrait;

struct Storage {
    owner: ContractAddress,
}

#[constructor]
fn constructor(){
    // Set contract deployer as the owner
    let contract_deployer = get_caller_address();
    self.owner.write(contract_deployer)
}

#[external(v0)]
impl ITokenImpl of IToken {
    fn transferTo(ref self: ContractState, to: ContractAddress, amount: u256) {
        let tx_info = starknet::get_tx_info().unbox();
        let authorizer: ContractAddress = tx_info.account_contract_address;
        assert(authorizer == self.owner.read());
        self.balance.write(to + amount);
    }
}
```

An attacker can trick the owner into using a malicious contract, allowing the attacker to call the `transferTo` function and impersonate the contract owner:

```rust
#[starknet::contract]
mod MaliciousContract {
...
...
#[external(v0)]
impl IMaliciousContractImpl of IMaliciousContract {
    fn transferTo(ref self: ContractState, to: ContractAddress, amount: u256) {
        let callback = ICallerDispatcher { contract_address: sender }.transferTo(ATTACKER_ACCOUNT, amount);
    }
}
```

### Recommendation:

Replace `account_contract_address` (origin) authentication with `get_caller_address` (sender) in the `transferTo` function to prevent phishing attacks:

```rust
use starknet::get_caller_address;

struct Storage {
    owner: ContractAddress,
}

#[constructor]
fn constructor(){
    // Set contract deployer as the owner
    let contract_deployer = get_caller_address();
    self.owner.write(contract_deployer)
}

#[external(v0)]
impl ITokenImpl of IToken {
    fn transferTo(ref self: ContractState, to: ContractAddress, amount: u256) {
        let authorizer = get_caller_address();
        assert(authorizer == self.owner.read());
        self.balance.write(to + amount);
    }
}
```

By using the correct authentication method, you can prevent phishing attacks and ensure that only authorized users can execute specific smart contract functions.

## 4. Handling Overflow and Underflow in Smart Contracts

Overflow and underflow vulnerabilities occur when assigning a value that is too large (overflow) or too small (underflow) for a given data type. In this section, we'll explore how to mitigate these issues in Starknet smart contracts.

When using the `felt252` data type, adding or subtracting a value outside the valid range can lead to incorrect results:

```rust
    fn overflow_felt252() -> felt252 {
        // Assign max felt252 value = 2^251 + 17 * 2^192
        let max: felt252 = 3618502788666131106986593281521497120414687020801267626233049500247285301248 + 17 * 6277101735386680763835789423207666416102355444464034512896;
        max + 3
    }

    fn underflow_felt252() -> felt252 {
        let min: felt252 = 0;
        // Assign max felt252 value = 2^251 + 17 * 2^192
        let substract = (3618502788666131106986593281521497120414687020801267626233049500247285301248 + 17 * 6277101735386680763835789423207666416102355444464034512896);
        min - substract
    }
```

We will get wrong values:

<img alt="felt252" src="img/ch02-13-sec_over_felt.png" class="center" style="width: 75%;" />

### Recommendation:

To avoid incorrect results, _use protected data types_: Utilize data types like `u128` or `u256` that are designed to handle overflows and underflows.

Here's an example of how to use the `u256` data type to handle overflow and underflow:

```rust
    fn overflow_u256() -> u256 {
        let max_u128: u128 = 0xffffffffffffffffffffffffffffffff_u128;
        let max: u256 = u256 { low: max_u128, high: max_u128 }; // Assign max u256 value
        let three: u256 = u256 { low: 3_u128, high: 0_u128 }; // Assign 3 value
        max + three
    }

    fn underflow_u256() -> u256 {
        let min: u256 = u256 { low: 0_u128, high: 0_u128 }; // Assign 0 value
        let three: u256 = u256 { low: 3_u128, high: 0_u128 }; // Assign 3 value
        min - three
    }
```

Executing these functions will revert the transaction if an overflow is detected:

<img alt="u256" src="img/ch02-13-sec_over_u256.png" class="center" style="width: 75%;" />
<img alt="u256" src="img/ch02-13-sec_over_u256.png" class="center" style="width: 75%;" />

- _Failure reasons for u256_:
  - `0x753235365f616464204f766572666c6f77=u256_add Overflow`
  - `0x753235365f737562204f766572666c6f77=u256_sub Overflow`

Similarly, the `u128` data type can be used to handle overflow and underflow:

```rust
    fn overflow_u128() -> u128 {
        let max: u128 = 0xffffffffffffffffffffffffffffffff_u128; // Assign max u128 value
        (max + 3_u128
    }

    fn underflow_u128() -> u128 {
        let min: u128 = 0_u128;
        min - 3_u128
    }
```

If an overflow or underflow occurs, the transaction will be reverted with a corresponding failure reason:

<img alt="u128" src="img/ch02-13-sec_over_u128.png" class="center" style="width: 75%;" />
<img alt="u128" src="img/ch02-13-sec_under_u128.png" class="center" style="width: 75%;" />

- _Failure reasons for u128_:
  - `0x753132385f616464204f766572666c6f77=u128_add Overflow`
  - `0x753132385f737562204f766572666c6f77=u128_sub Overflow`

## 5. Private Data On-Chain.

In some cases, a smart contracts may needs to store secret values that can't be revelead to users, however this is not possible
if you store data on chain because all stored data is public and can be retrieved even if you don't publish your code. In the next
example, our smart contract will use a contructor parameter to set a password (12345678) and store it on chain:

```rust
#[starknet::contract]
mod StoreSecretPassword {
    struct Storage {
        password: felt252,
    }

    #[constructor]
    fn constructor(_password: felt252) {
        self.password.write(_password);
    }
}
```

<img alt="deploy" src="img/ch02-13-sec_priv01.png" class="center" style="width: 75%;" />

Then, understanding how <a href="https://book.cairo-lang.org/ch99-01-03-01-contract-storage.html?highlight=kecc#storage-addresses" target="_blank">storage layout</a> works in Cairo, let's build a script to read stored smart contract variables:

```javascript
import { Provider, hash } from "starknet";

const provider = new Provider({
  sequencer: {
    network: "goerli-alpha", // or 'goerli-alpha'
  },
});

var passHash = hash.starknetKeccak("password");
console.log(
  "getStor=",
  await provider.getStorageAt(
    "0x032d0392eae7440063ea0f3f50a75dbe664aaa1df76b4662223430851a113369",
    passHash,
    812512,
  ),
);
```

And we will get the stored value (hex value of 12345678):

<img alt="get_storage" src="img/ch02-13-sec_priv02.png" class="center" style="width: 75%;" />

Also, in a block explorer we can go to the deploy transaction and watch deployed parameters values:

<img alt="block explorer" src="img/ch02-13-sec_priv03.png" class="center" style="width: 75%;" />

### Recommendation:

If your smart contract needs to store private data on chain, then you must use off chain encryption before to send data to the blockchain
or may explore some alternatives like using hashes, merkle trees or commit-reveal patterns.

## Call for Contributions: Additional Vulnerabilities

We've covered a few common vulnerabilities in Cairo smart contracts, but there are several more security considerations that developers should be aware of. We are currently seeking contributions from the community to expand this chapter and cover more vulnerabilities, as listed in our To-Do section:

- Storage Collision
- Flash Loan Attacks
- Oracle Manipulation
- Bad Randomness
- Denial of Service
- Untrusted Delegate Calls
- Public Burn

If you have expertise in any of these areas, we encourage you to contribute to this chapter by adding explanations and examples of the respective vulnerabilities. Your contributions will help educate and inform the Starknet and Cairo developer community, promoting the creation of more secure and robust smart contracts.

Thank you for your support in making the Starknet ecosystem safer and more secure for all developers and users.

The Book is a community-driven effort created for the community.

- If you’ve learned something, or not, please take a moment to provide
  feedback through [this 3-question
  survey](https://a.sprig.com/WTRtdlh2VUlja09lfnNpZDo4MTQyYTlmMy03NzdkLTQ0NDEtOTBiZC01ZjAyNDU0ZDgxMzU=).

- If you discover any errors or have additional suggestions, don’t
  hesitate to open an [issue on our GitHub
  repository](https://github.com/starknet-edu/starknetbook/issues).