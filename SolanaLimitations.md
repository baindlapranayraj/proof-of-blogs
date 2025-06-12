<img
 width="1000px"
 height="480px"
 src="./images/shinchan .jpg"
/>

# 🦀 Solana Limitatons (Still writing)

GM GM everyone 😁,

In this blog, we will go through the various types of Solana resource limitations you might encounter while developing smart contracts. You may run into errors containing words like "limit" or "exceed." These errors represent boundaries predefined by Solana programs to maintain fairness and performance on the blockchain. If you’d like to get more information on these topics, you’re in the right place!

Hear what u can expect from this blog:-

1. Types of limitations

- CU Limitations
- Storage Limitations
- transaction size limitations
- Call depth limitations
- Stack size
- PDA Accounts limitations

# Limitations of Solana:

Solana is a high-performance public blockchain that stands apart from traditional blockchains like Bitcoin and Ethereum due its unique approach to transaction processing separates logic and state into different accounts, which allows Solana to process many transactions simultaneously.

However, programs running on Solana are subject to several types of resource limitations. These limitations ensure that programs use system resources fairly while maintaining high performance.

Knowing these boundaries is very helpful for developers, as it guides them in building efficient and reliable program.

## 1. Compute Unite Limitations:

### What is Compute Units ?

CU (Compute Unit) the name itself suggests that CU is the fundamental measurement of the computational work (CPU cycles and memory usage) performed by a transaction or instruction on Solana. It's similar to "Gas" fees in Ethereum but is much more predictable and low-latency.

Every instruction your smart contract executes on-chain, such as reading or writing to accounts, performing cryptographic operations (like zk-ElGamal), or verifying signatures, consumes a certain number of compute units (CUs). This is roughly proportional to the amount of work done by the nodes.

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

### Why we have limited Compute Unit Budget ?

In short and simple terms, Solana has CU limitations to ensure fair resource allocation. But what does that mean ?

Solana validators are individual computers (or nodes) that process blockchain transactions and maintain the network’s state (these are the basic fundamentals you must know 😒). Each validator has limited CPU power and memory, just like any regular computer. When programs run inside these nodes, they allocate some memory to process all the instructions.

Now, if a malicious user sends a transaction that contains infinite loops, it can use a huge amount of memory and may slow or even crash the system. Since the blockchain is a network of shared computers/nodes, if one user performs a huge number of CPU tasks and uses a large amount of CPU/memory, it hogs the system, starving other users.

Due to this limitation in CU Budget it helps solana network to prevent denial-of-service (DoS) attacks and resource exhaustion on validators.
