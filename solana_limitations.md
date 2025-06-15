<img
 width="1000px"
 height="440px"
 src="./images/shinchan .jpg"
/>

# 🦀 Solana Limitatons (Still writing)

GM GM everyone 😁,

In this blog, we will go through the various types of Solana resource limitations you might encounter while developing smart contracts. You may run into errors containing words like "limit" or "exceed." These errors represent boundaries predefined by Solana programs to maintain fairness and performance on the blockchain. If you’d like to get more information on these topics, you’re in the right place!

Hear what u can expect from this blog:-

- CU Limitations ✅
- transaction size limitations ✅
- Stack size
- PDA Accounts limitations

# Limitations of Solana:

Solana is a high-performance public blockchain that stands apart from traditional blockchains like Bitcoin and Ethereum due its unique approach to transaction processing separates logic and state into different accounts, which allows Solana to process many transactions simultaneously.

However, programs running on Solana are subject to several types of resource limitations. These limitations ensure that programs use system resources fairly while maintaining high performance.

Knowing these boundaries is very helpful for developers, as it guides them in building efficient and reliable program.

## 1. Compute Unite Limitations:

### What is Compute Units ?

CU (Compute Unit) the name itself suggests that CU is the fundamental measurement of the computational work (CPU cycles and memory usage) performed by a transaction or instruction on Solana. It's similar to "Gas" fees in Ethereum but is much more predictable and low-latency.

Every instruction your smart contract executes on-chain, such as reading or writing to accounts, performing cryptographic operations (like zk-ElGamal), or verifying signatures and serailization and deserialization consumes a certain number of compute units (CUs). This is roughly proportional to the amount of work done by the nodes.

If you perform simple transactions, then nodes can process those smart contracts efficiently, resulting in lower CU consumption. However, if you perform complicated mathematical operations or heavy loops, nodes consume a large amount of memory and CPU, and it takes more time to run the program (smart contract), resulting in higher CU consumption

_For example, for an simple transaction like sending a SOL form wallet A to wallet B it takes around 3000 Compute Units_

```rust

let transfer_amount_accounts = Transfer {
      from: ctx.accounts.signer.to_account_info(),
      to: ctx.accounts.recipient.to_account_info(),
    };

let ctx = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        transfer_amount_accounts,
      );

transfer(ctx, amount * LAMPORTS_PER_SOL)?;

 // Takes around 3000 CU
```

The code in the above was written in Anchor ⚓️, It performs an simple transfer of SOL, and for that it taking 3000 CUs.

### Compute Unit Budget

- As we know, heavy mathematical operations or loops consume a large amount of compute units. However, there is a default budget of 200,000 CUs for every transaction or instruction.

- If a transaction or instruction exhausts the 200,000 CU limit, the transaction or instruction is simply reverted, all state changes are undone, and the fees are not refunded to the signer who invoked the transaction. (This mechanism prevents attackers from running never-ending or computationally intensive programs on nodes, which could slow down or halt the chain.)

```rust
Error: exceeded maximum number of instructions allowed (200000) compute units
```

- Personally, I encountered this error while building my Chaubet project. When a bettor buys shares,[this function](https://github.com/baindlapranayraj/Chaubet/blob/86b5c91de727dd173a0593a7378a0acb3eb25b2a/programs/chaubet/src/utils/helper.rs#L89C5-L132C6) performs some heavy mathematical computations and checks. Due to this, the instruction exceeded the 200,000 CU limit, and the entire instruction was reverted.

- Well we can increase our computation limit using `SetComputeUnitLimit` we can request a specific calculation unit limit by adding an instruction to our transaction. But 🍑 we can only increase CUs Budget only upto 1.4 Million Units.

```rust
const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
  units: 500000,  // Increased from 200k to 500k CUs.
});
```

- Regardless of the compute units consumed, the transaction charged 5000 lamports or 0.000005 SOL.

### Why we have limited Compute Unit Budget ?

In short and simple terms, Solana has CU limitations to ensure fair resource allocation. But what does that mean ?

Solana validators are individual computers (or nodes) that process blockchain transactions and maintain the network’s state (these are the basic fundamentals you must know 😒). Each validator has limited CPU power and memory, just like any regular computer. When programs run inside these nodes, they allocate some memory to process all the instructions.

Now, if a malicious user sends a transaction that contains infinite loops, it can use a huge amount of memory and may slow or even crash the system. Since the blockchain is a network of shared computers/nodes, if one user performs a huge number of CPU tasks and uses a large amount of CPU/memory, it hogs the system, starving other users.

Due to this limitation in CU Budget it helps solana network to prevent denial-of-service (DoS) attacks and resource exhaustion on validators.

## 2. Transaction Size Limit.

Before getting into Transaction Size Limit, lets peak a little into what are transactions what do they contain.
A transaction on Solana is a request sent by users to interact with the network, typically to mutate data such as transferring lamports (the native token unit) between accounts.

Each transaction contains one or more instructions that specify the operations to perform on the blockchain. This execution logic is stored in raw bytes in program state.

<img
 width="1000px"
 height="550px"
 src="./images/trx_arch.png"
/>

### The Transaction Contains:

- **Array of signatures**:- A sequence of signatures included in the transaction.
- **Message**:- This is the core part of the transaction. It contains everything needed to describe what the transaction intends to do.

  As you are already seeing the above Image, The Message is the core part of the transaction which contains the :-

- Header:- This one indicates the number of signers and read-only accounts **(3 bytes)**
- Array of Accounts:- It contains all the accounts required to perfrom the transaction, provided from the client (Each Pubkey 32 bytes)
- Recent Blockhash:- A recent blockhash was attached to this transaction (32 bytes)
- Array of Instruction:- Contains all instructions invoked by the signer(The size is depended on complexity of Instruction code).

### Transacion Size Limitaions:-

<img
 width="1000px"
 height="550px"
 src="./images/trx_size.png"
/>

Every Solana transaction has a size limit of **1,232 bytes**. The total transaction size is the sum of the bytes for all signatures, accounts involved, the blockhash, and the instructions (including their accounts and data), and must be less than 1,232 bytes. Because of this limit, developers need to fit all signers, account addresses, and required data within the 1,232-byte constraint

You would have encounter the `transaction size limit exceeded` error when you try to send too many accounts from client side or too many instructions in single transaction.

**you might wonder why only limited to 1,232 bytes ?**

The transaction size limit of 1,232 bytes in Solana is closely tied to how data is transmitted over the internet using IPv6 (Internet Protocol version 6). Here's why:

1. In networking, there's a concept called Maximum Transmission Unit (MTU) - the largest packet size that can be transmitted over a network.
2. The standard MTU for IPv6 is 1,280 bytes.
3. Solana's transaction size limit of 1,232 bytes ensures that transactions can fit within a single IPv6 packet, accounting for some overhead bytes needed for network headers.

This limitation helps maintain network efficiency and reduces the chances of packet fragmentation, which could slow down transaction processing.

## 3. Stack Size Limitations

In Solana programs (smart contracts), there's a limitation on the stack size - the amount of memory allocated for local variables and function calls. The current stack frame size limit is **4KB per frame**.

### What is Stack Memory?

Stack memory is used for:

- Local variables in functions
- Function parameters
- Return addresses
- Temporary data

### Stack Size Constraints

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    // This large array on stack might cause issues
    let big_array = [0u8; 4096]; // ❌ This would fail

    // Better to use smaller arrays or heap allocation
    let small_array = [0u8; 1000]; // ✅ This is fine
    Ok(())
}
```

To work around stack limitations:

1. Use heap allocation when necessary
2. Break large functions into smaller ones
3. Avoid large stack-allocated arrays
4. Use references instead of moving large data structures

## 4. PDA (Program Derived Address) Account Limitations

PDAs in Solana have several important limitations that developers need to be aware of:

### Seed Length Limitations

1. Maximum Combined Seed Length:
   - The total length of all seeds combined cannot exceed 32 bytes
   - This includes the length of each seed plus any separator bytes

```rust
// Example of seed length limitation
let seeds = &[
    "marketplace".as_bytes(),    // 11 bytes
    user.key().as_ref(),        // 32 bytes
    &[bump]                     // 1 byte
];
// ❌ This would exceed the limit as total > 32 bytes
```

### Number of Seeds Limitation

1. Maximum Number of Seeds:
   - A PDA can have up to 16 seeds
   - Each seed can be of variable length

```rust
// Good practice
let pda = Pubkey::find_program_address(
    &[
        "marketplace".as_bytes(),
        user.key().as_ref(),
        &[1u8]
    ],
    program_id
);

// ❌ Bad practice - too many seeds
let pda = Pubkey::find_program_address(
    &[
        "type".as_bytes(),
        "marketplace".as_bytes(),
        "version".as_bytes(),
        "id".as_bytes(),
        user.key().as_ref(),
        timestamp.to_le_bytes().as_ref(),
        // ... more seeds
    ],
    program_id
);
```

### PDA Derivation Constraints

1. Program ID Limitation:

   - A PDA must be derived from exactly one program ID
   - Cannot be derived from multiple programs

2. Bump Seed:
   - Must be valid (0-255)
   - Typically use bump 255 and work backwards

### Best Practices for PDA Usage

1. Keep Seeds Minimal:

   ```rust
   // ✅ Good: Minimal, meaningful seeds
   let seeds = &[
       "user_profile".as_bytes(),
       user.key().as_ref(),
   ];
   ```

2. Use Consistent Seed Structure:

   ```rust
   // ✅ Good: Consistent structure across your program
   let seeds = &[
       prefix.as_bytes(),
       entity_type.as_bytes(),
       identifier.as_ref(),
   ];
   ```

3. Document Seed Structure:
   ```rust
   /// PDA Seeds Structure:
   /// 1. Prefix: "user_profile"
   /// 2. User's Public Key
   /// 3. Bump Seed
   pub fn initialize_user_profile(ctx: Context<InitUserProfile>) -> Result<()> {
       // Implementation
   }
   ```

[solana_optimization_github](https://github.com/solana-developers/cu_optimizations)
[rare_skill_blog_post](https://www.rareskills.io/post/solana-compute-unit-price)
