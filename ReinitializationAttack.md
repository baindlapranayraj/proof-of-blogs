<img
 width="1000px"
 height="480px"
 src="./images/bhudha.png"
/>

# Reinitialization Attack

Program Example(Btw I use Neovim 🙂):- [github link](https://github.com/baindlapranayraj/reinitialization-attack-demo)
To understand how reinitialization attacks work, we must first understand the inner workings of the `init` and `init_if_needed` constraints in Anchor ⚓️.

## **`init`:-**

In Simple words init ensures the account is created and initialize **only once** for a given address (determined by seeds and bump), `init` internally performs the following two sets of instructions:

### **System Level:-**

- It uses `create_account` instruction in a System Program.
- It allocates space for on-chain for the account
- It funds the rent-exempt lamports using `Rent::minimum_balace` .
- It assigns an owner program (e.g., your custom program).

### initailization **of an Account :-**

- Your program defines the account data structure using `#[account]` struct.
- writes the **8-byte discriminator and attached to your data(which is used as type checker)**
- Stores the initial data like show in the example below PDA account struct.

```rust
#[account]
#[derive(InitSpace)]
pub struct User {
  pub user_pubkey: Pubkey,
  #[max_len(30)]
  pub user_name: Option<String>,
  pub balance: u64,
  pub user_vault_bump: u8,
  pub user_bump: u8,
}
```

```rust
#[account(
    init,
    payer = USER,
    space = 8 + User::INIT_SPACE,
    seeds = [
              USER,
              user.key().to_bytes().as_ref(),
              chau_config.key().to_bytes().as_ref()
            ],
    bump
)]
pub user_profile: Account<'info, User>,
```

### Security of `init` Constraint 🛡️ : -

The security of the `init` constraint is rigid, unlike `init_if_needed`. You cannot create a fake PDA and pass it to the program, nor can you initialize the same account twice, as doing so will throw an error.

**Anchor checks whether the account is initialized by evaluating the following conditions:**

1. If the account has some Lamports and it is owned by the System Program, Anchor considers it an uninitialized account and it throws an error.
2. If an Account has zero Lamports then anchor init can create and intialize the account (because it is a new account).

**How Anchor checks the PDAs 🔍:**

1. When you serialize your account, Anchor adds an extra discriminator (or tag) that stores the name of your account (e.g., `User` for the PDA above) in 8 bytes using the first 8 bytes of the SHA256 hash of the account’s name.
2. During deserialization, this discriminator acts as a type checker. If a malicious account is passed, Anchor compares the discriminator in the account’s data to the expected discriminator for the account type. If they don’t match, Anchor will reject the account, preventing the malicious account from being used.
3. In Solana every data is stored as raw-binary data, so anchor could not figure that is this account is `User` or not, This discriminator acts like lable/tag so that anchor can deserialize it back to Bettor struct with given Struct.

### **Attack Scernario 🔪:-**

- Let’s say there is an instruction that requires a `User` PDA, as shown in the first example code, and an attacker tries to send a malicious PDA to the `User` address.
- Here, Anchor first checks the discriminator of the `Fake` account and compares it with the discriminator of `User`. If the discriminator doesn’t match, Anchor rejects the account and throws an error.

You might (and you should) start appreciating Anchor. It takes care of a lot of important security checks, as well as the serialization and deserialization of accounts (and some other footguns), under the hood.

Note:- Hear Account created by anchor is a wrapper around the AccountInfo(shown below),which helps anchor to verifies program ownership.

```rust
pub struct AccountInfo<'a> {
     /// Public key of the account
     pub key: &'a Pubkey,
     /// The lamports in the account.  Modifiable by programs.
     pub lamports: Rc<RefCell<&'a mut u64>>,
     /// The data held in this account.  Modifiable by programs.
     pub data: Rc<RefCell<&'a mut [u8]>>,
     /// Program that owns this account
     pub owner: &'a Pubkey,
     /// The epoch at which this account will next owe rent
     pub rent_epoch: u64,
     /// Was the transaction signed by this account's public key?
     pub is_signer: bool,
     /// Is the account writable?
     pub is_writable: bool,
     /// This account's data contains a loaded program (and is now read-only)
     pub executable: bool,
}
```

## `init_if_needed`:-

Unlike `init`, `init_if_needed` is not rigid and it is flexible to use, `init_if_needed` is as same as `init` but it is not as rigid as `init` when you use `init_if_needed` instead of `init` anchor follows these checks:

1. Does the account Exist and initialized, if not create and initialize and move forward.
2. Does the account Exist but it is not intialized then initialized and reset the data and move forward.
3. Account and Exist and initialized then it skips initialization and lets you modify the existing account with out permission(This is dangerous with any proper validations).

### **Attack Scenario(**Reinitialization Attack**) 🔪:-**

> **Note:**

Anchor considers an account "**uninitialized**" when:

- The 8-byte discriminator is missing or invalid (doesn't match the expected hash).
- The account ownership doesn't match the expected program.
- The account has zero lamports (effectively making it non-existent on-chain)

### Attacker Flow:

### **Normal Flow of Code:**

In a normal scenario, `init_if_needed` is used for accounts that might need lazy initialization. For example:

```rust
#[account(
  init_if_needed,
  payer = user,
  space = 8 + User::INIT_SPACE,
  seeds = [b"user", user.key().as_ref()],
  bump
)]
pub user_account: Account<'info, User>,
```

Here, the account is safely initialized only if it doesn't exist or is uninitialized.

### **How a Malicious User Exploits This:**

An attacker can exploit `init_if_needed` by:

1. **Uninitializing an account** by draining its lamports (setting balance to zero) or changing its ownership.
2. **Forcing reinitialization** by passing the same account to an instruction that uses `init_if_needed`, tricking the program into resetting the account's state.

### **Methods of Uninitializing an Account:**

1. **Draining Lamports:**
   - An attacker can withdraw all lamports from an account, making it "uninitialized" in Anchor's eyes (since zero-lamport accounts are treated as nonexistent).
2. **Changing Ownership:**
   - If an attacker changes the account's owner to another program, Anchor will consider it uninitialized for the original program.
3. **Corrupting the Discriminator:**
   - If an attacker modifies the first 8 bytes (discriminator) of the account data, Anchor will treat it as uninitialized.

### **What Happens After the Attack?**

- The account's data is **reset to default values** (e.g., balance = 0, user_name = None).
- An attacker can abuse this to:
  - Reset their debt in a lending protocol.
  - Regain access to a locked account.
  - Exploit logic that depends on the account's state.

### **Security Practices for Using `init_if_needed` 🛡️:**

1. **Avoid `init_if_needed` for Critical State Accounts:**
   - If an account stores important data (e.g., user balances, protocol settings), use `init` instead to prevent reinitialization.
2. **Add Explicit Checks:**
   - Manually verify whether an account is already initialized before using `init_if_needed`.

```rust
#[account]
#[derive(InitSpace)]
   pub struct User {
   pub user_pubkey: Pubkey,
   #[max_len(30)]
   pub user_name: Option<String>,
   pub balance: u64,
   pub is_intialized: bool,
   pub user_vault_bump: u8,
   pub user_bump: u8,
}
```

```rust
require!(user_account.discriminator.is_empty(), AlreadyInitialized);
```

### **Conclusion:**

While `init_if_needed` provides flexibility, it introduces risks if misused. **Always prefer `init` for security-critical accounts** and only use `init_if_needed` when absolutely necessary, with proper safeguards. Anchor's discriminator check helps prevent some attacks, but developers must implement additional checks to fully secure their programs against reinitialization exploits.

## References:-

https://solana.com/developers/courses/program-security/reinitialization-attacks

https://www.rareskills.io/post/init-if-needed-anchor

https://medium.com/@calc1f4r/init-vs-init-if-needed-a-deep-dive-d33fe59e4de5

https://syedashar1.medium.com/program-security-in-anchor-framework-solana-smart-contract-security-b619e1e4d939

https://chatgpt.com/. 🙂

## Source Code:-

https://github.com/baindlapranayraj/reinitialization-attack-demo
