## Task 2: Extend the Program (CPI to Registration Program)

### What was changed

The `withdraw` instruction in `programs/pre-req-vault/src/instructions/withdraw.rs`
was extended to perform a Cross-Program Invocation (CPI) into the registration
program's `initialize` instruction immediately after transferring SOL out of
the vault. This records the caller's GitHub username on-chain as part of the
same withdrawal transaction.

```rust
let cpi_accounts = Initialize {
    user: self.user.to_account_info(),
    account: self.application_account.to_account_info(),
    system_program: self.system_program.to_account_info(),
};
let cpi_ctx = CpiContext::new(self.application_program.key(), cpi_accounts);

initialize(cpi_ctx, "rohan911438".to_string())?;
```

### Deployment (own program, not the shared one)

This program was deployed under its own freshly generated program ID —
**not** the original repo's ID — as required, so the CPI verification comes
from an independently deployed program rather than a call routed through the
maintainers' existing deployment.

| | |
|---|---|
| **Cluster** | Devnet |
| **Program ID** | `3kbcxQyHtEiU4du5v7gqi9trvT1LFEq6YnemQCQeJ3ca` |
| **Deployer wallet** | `APUGzpmU1cQvn5ti7MZDnbwWR2kwW4SakMfhkLNVham7` |
| **Registration program (external, called via CPI)** | `TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM` |
| **GitHub username recorded** | `rohan911438` |

### How to reproduce

```sh
anchor build
anchor keys sync
anchor build
anchor deploy --provider.cluster devnet
anchor test --skip-local-validator
```

### Success criteria verification

`anchor test --skip-local-validator` runs the full instruction sequence
(`initialize` → `deposit` → `withdraw` → `close`) against the deployed devnet
program. The `withdraw` test triggers the CPI described above; a successful
run confirms the registration program's `ApplicationAccount` PDA (seeds
`["prereqs", user]`, owned by the registration program) was created and
populated with the GitHub username during that same transaction.

Note: registration is one-per-wallet on the registration program's side —
re-running `withdraw` against an already-registered wallet will fail on the
CPI step, which is expected behavior, not a bug.

### Test run

All 4 tests passed:

```
✔ Initialize the vault (2077ms)
✔ Deposit 1 Sol in to the vault (1338ms)
✔ Withdraw 0.5 Sol from the vault (1099ms)
✔ Close the vault and withdraw all the funds (1486ms)
4 passing (6s)
```

### On-chain proof

Withdraw transaction on devnet (verified directly against the devnet RPC,
not just the explorer UI):

```
5ZaBdAMmpN9Xw78pe4bgmXgjas1HeiSWpqcCNBnS7mYzMa1aBc7pQc6hYedK2ezyE4Q66SCraLXiTCKS7VrKLEa9
```

https://explorer.solana.com/tx/5ZaBdAMmpN9Xw78pe4bgmXgjas1HeiSWpqcCNBnS7mYzMa1aBc7pQc6hYedK2ezyE4Q66SCraLXiTCKS7VrKLEa9?cluster=devnet

The transaction's instruction logs show the CPI succeeding end to end:

```
Program 3kbcxQyHtEiU4du5v7gqi9trvT1LFEq6YnemQCQeJ3ca invoke [1]
Program log: Instruction: Withdraw
Program TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM invoke [2]
Program log: Instruction: Initialize
Program TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM success
Program 3kbcxQyHtEiU4du5v7gqi9trvT1LFEq6YnemQCQeJ3ca success
```

The resulting `ApplicationAccount` PDA (`HEBmtdS3bwCug617XYidHtYGwWMtg69eLVz4MhX3sKA4`,
owned by the registration program) was fetched from devnet and its raw account
data decodes to the GitHub username `rohan911438`, confirming the registration
was recorded on-chain by this transaction.
