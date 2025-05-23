# Scale or Die at Accelerate 2025: Writing Optimized Solana Programs (Dean Little | Blueshift)

  ![](thumbnail.jpg)

```
This is a transcription of a YouTube video. Among the relevant resources for learning more about Solana programs optimizations, I found this video particularly interesting.

This content presents the opinions and perspectives of industry experts or other individuals. The opinions expressed in this content do not necessarily reflect my opinion.

Readers are encouraged to verify the information on their own and seek professional advice before making any decisions based on this content.
```

[Scale or Die at Accelerate 2025: Writing Optimized Solana Programs (Dean Little | Blueshift) - YouTube](https://www.youtube.com/watch?v=Fk_UtbEny0c)

## Introduction
- **[`00:08`](https://www.youtube.com/watch?v=Fk_UtbEny0c)** **[Dean Little](https://x.com/deanmlittle)** introduces himself as a representative of **[Blueshift](https://blueshift.gg/)**
- **[`00:12`](https://www.youtube.com/watch?v=Fk_UtbEny0c?t=12)** Announces that his presentation will focus on writing optimized Solana programs
- **[`00:25`](https://www.youtube.com/watch?v=Fk_UtbEny0c?t=25)** Begins by explaining the general philosophy of program optimization

## Optimization Philosophy

- **[`00:29`](https://youtu.be/Fk_UtbEny0c?t=29)** **Compute optimization**:
  - Lower CU (compute units) consumption gives transactions higher priority
  - For equal fees, an optimized program will be processed faster
  - Users save on priority fees, making the protocol more competitive

- **[`00:50`](https://www.youtube.com/watch?v=Fk_UtbEny0c?t=50)** **Storage optimization**:
  - Smaller on-chain footprint reduces operational costs
  - Even though rent is recoverable, there's an opportunity cost
  - Locked-up money (SOL) could be used to generate yield or for staking
  - The program size itself is a non-recoverable cost

- **[`01:25`](https://youtu.be/Fk_UtbEny0c?t=85)** **Data optimization**:
  - Smaller instructions, fewer required accounts
  - Makes the program easier to compose with others (CPI-friendly)
  - Example cited: a Jupiter swap is difficult to combine with other actions due to being too compute-intensive

- **[`01:50`](https://youtu.be/Fk_UtbEny0c?t=110)** **Necessary balance**:
  - Optimization isn't always profitable
  - Can make code difficult to maintain and hire qualified developers
  - Can increase audit costs and delays (less standard frameworks)
  - Can delay time-to-market

## CU Optimization Tips
- **[`03:02`](https://youtu.be/Fk_UtbEny0c?t=182)** **Avoid Solana Program framework**:

  ```rust
  use pinocchio::{
      account_info::AccountInfo, entrypoint, program_error::ProgramError,
      pubkey::Pubkey, ProgramResult,
  };

  entrypoint!(process_instruction);

  fn process_instruction(
      _program_id: &Pubkey,
      accounts: &[AccountInfo],
      instruction_data: &[u8],
  ) -> ProgramResult {
      let (discriminator, data) = instruction_data
          .split_first()
          .ok_or(ProgramError::InvalidInstructionData)?;
      match VaultInstructions::try_from(discriminator) {
          // instructions
      }
  }
  ```

  - Very inefficient with many heap allocations
  - Recommended alternatives: community libraries or Pinocchio
  - **[Pinocchio](https://github.com/anza-xyz/pinocchio)** is a good compromise between ease of use and optimization

- **[`03:44`](https://youtu.be/Fk_UtbEny0c?t=224)** **Use zero-copy**:

  ```rust
  #[repr(C)]
  #[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
  pub struct Token {
      pub owner:   [u8; 32],
      pub mint:    [u8; 32],
      pub balance: u64,
  }

  let token: &Token = bytemuck::from_bytes(data);
  ```
  - Compatible with Pinocchio, Steel, Anchor
  - Start with **[bytemuck](https://crates.io/crates/bytemuck)** as a simple solution
  - Not recommended for validator software

- **[`04:10`](https://youtu.be/Fk_UtbEny0c?t=250)** **Fixed-size data first**:

  ```rust
  pub struct Record {
      name:  String,
      owner: Pubkey,
  }
  ```
  **>>**

  ```rust
  pub struct Record {
      owner: Pubkey,
      name:  String,
  }
  ```

  - Reason 1: More **efficient deserialization** - no need to traverse variable data first
  - Reason 2: **Simplified RPC filtering** - known offsets for fields
  - Example: putting pubkey before string allows **easy filtering** on public keys

- **[`05:00`](https://youtu.be/Fk_UtbEny0c?t=300)** **Avoid dynamically sized data**:
  - For rarely accessed accounts, it's better to pay a bit more rent
  - Computational overhead of variable data persists with each access

- **[`05:43`](https://youtu.be/Fk_UtbEny0c?t=343)** **Don't include accounts solely for their public key**:

  ```rust
  #[derive(Accounts)]
  pub struct CheckOwner<'info> {
      pub signer: Signer<'info>,
      /// CHECK: Only used for PDA derivation
      pub owner: UncheckedAccount<'info>,
      #[account(
          seeds = [owner.key().as_ref()],
          bump,
      )]
      pub vault: Account<'info, VaultAccount>,
      pub system_program: Program<'info, System>,
  }
  ```
  **>>**

  ```rust
  #[derive(Accounts)]
  #[instruction(owner: Pubkey)]
  pub struct CheckOwner<'info> {
      pub signer: Signer<'info>,
      #[account(
          seeds = [owner.key().as_ref()],
          bump,
      )]
      pub vault: Account<'info, VaultAccount>,
  }
  ```

  - If an account isn't used for signing, don't include it in the accounts array
  - Pass the public key in instruction data instead

- **[`06:04`](https://youtu.be/Fk_UtbEny0c?t=364)** **PDAs and optimization**:

  ```rust
  let (pda, bump) = Pubkey::try_find_program_address(
      &[signer.key.as_ref()],
      &crate::ID
  )
  .ok_or(ProgramError::InvalidSeeds)?;
  assert!(pda.eq(vault.key));

  ```

  - Don't recalculate PDA addresses used as signers
  - The system already verifies the address during signing

- **[`06:22`](https://youtu.be/Fk_UtbEny0c?t=382)** **SHA-256 vs create_program_address**:

  ```rust
  use solana_nostd_sha256::hashv;

  const PDA_MARKER: &[u8; 21] = b"ProgramDeriveAddress";

  let bump = data[8];
  let pda  = hashv(&[
      signer.key().as_ref(),
      &[bump],
      ID.as_ref(),
      PDA_MARKER,
  ])

  assert_eq!(&pda, vault.key().as_ref());
  ```

  - For already created PDAs, use direct SHA-256 instead of `create_program_address`
  - `create_program_address` does SHA-256 + off-curve verification (unnecessary if already verified)
  - 100 CUs for standard SHA-256 vs 120 CUs with Solana program

- **[`07:41`](https://youtu.be/Fk_UtbEny0c?t=461)** **Avoid superfluous checks**:

  ```rust
  require_gte!(
      self.sender_token_account_amount,
      transfer_amount,
  );
  ```

  - Example: unnecessary to check if a token account has enough tokens before transferring
  - **The token program will already fail if the balance is insufficient**

## Practical Optimization Example

- **[`08:00`](https://youtu.be/Fk_UtbEny0c?t=480)** **Basic Anchor program**:

  ```rust
  use anchor_lang::prelude::*;

  declare_id!("222222222222222222222222222222222222222222");

  #[program]
  pub mod memo_anchor {
      use super::*;

      #[instruction(discriminator = [0])]
      pub fn log(_ctx: Context<Log>, string: Sring_ -> Result<()> {
          msg!(&string);
          Ok(())
      }
  }

  #[derive(Accounts)]
  pub struct Log {}
  ```

  - Simple program that takes a string and displays it
  - Uses `#[instruction(discriminator = [0])]` to save **1 CU**
  - Initial consumption: **649 CU**

- **[`8:28`](https://youtu.be/Fk_UtbEny0c?t=508)** **First optimization: disable logs**:


  ```toml
  [features]
  default = ["no-log-ix-name"]
  cpi = ["no-entrypoint"]
  no-entrypoint = []
  no-idl = []
  no-log_ix_name = []
  idl-build = ["anchor-lang/idl-build"]
  ```
  - Activating the no_log feature (`no-log-ix-name`)
  - New consumption: **544 CU** (105 CU savings)

- **[`08:45`](https://youtu.be/Fk_UtbEny0c?t=525)** **Anchor optimization**:

  ```rust
  pub use context::*;

  #[cfg(feature = "idl-build")]
  pub mod context {
      use anchor_lang::prelude::*;
      #[derive(Accounts)]
      pub struct Raw {}
  }

  #[cfg(not(feature = "idl-build"))]
  pub mod context {
      pub struct Raw {
          pub data: *const u8,
      }

      impl anchor_lang::Accounts<'_, RawBumps> for Raw {
          #[inline(never)]
          fn try_accounts(
              __program_id: &Pubkey,
              __accounts: &mut &[anchor_lang::solana_program::account_info::AccountInfo<'_>],
              __ix_data: &[u8],
              __bumps: &mut RawBumps,
              __reallocs: &mut std::collections::BTreeSet<Pubkey>,
          ) -> anchor_lang::Result<Self> {
              Ok(Raw {
                  data: __ix_data.as_ptr(),
              })
          }
      }

    // ... some more impls
    }

  ```

  - Custom account context implementation
  - Using raw pointers to instruction data

  ```rust
  #[feature(str_from_raw_parts)]
  use anchor_lang::prelude::*;
  mod raw;
  use raw::*;

  pub const ID: Pubkey = pubkey!("222222222222222222222222222222222222222222");

  #[program]
  pub mod anchor_memo {
      use super::*;

      #[cfg(feature = "idl-build")]
      pub fn log(ctx: Context<Raw>) -> Result<()> {
          Ok(())
      }

      #[cfg(not(feature = "idl-build"))]
      #[instruction(discriminator = [])]
      pub fn log(ctx: Context<Raw>) -> Result<()> {
          unsafe {
              anchor_lang::solana_program::log::sol_log(
                  core::str::from_raw_parts(
                      ctx.accounts.data,
                      *ctx.accounts.data.sub(8) as usize,
                  ),
              );
          }
          Ok(())
      }
  }
  ```
  - Result: **281 CU**

- **[`09:20`](https://youtu.be/Fk_UtbEny0c?t=560)** **Migration to Pinocchio**:

  ```rust
  use pinocchio::{
      account_info::AccountInfo, log, program_entrypoint, pubkey::Pubkey, ProgramResult
  };

  program_entrypoint!(process_instruction);

  fn process_instruction(
      _program_id: &Pubkey,
      _accounts: &[AccountInfo],
      instruction_data: &[u8],
  ) -> ProgramResult {
      unsafe {
          log::sol_log(core::str::from_utf8_unchecked(instruction_data));
      }
      Ok(())
  }
  ```

  - Almost identical code but more efficient framework
  - Immediate result: **109 CU**

- **[`09:41`](https://youtu.be/Fk_UtbEny0c?t=581)** **Pinocchio optimization**:

  ```rust
  #![no_std]

  use pinocchio::{
      entrypoint::InstructionContext, lazy_program_entrypoint, log, no_allocator,
      nostd_panic_handler, ProgramResult,
  };

  lazy_program_entrypoint!(process_instruction);
  no_allocator!();
  nostd_panic_handler!();

  fn process_instruction(context: InstructionContext) -> ProgramResult {
      unsafe {
          log::sol_log(core::str::from_utf8_unchecked(context.instruction_data())?);
      }
      Ok(())
  }
  ```

  - Using `lazy_program_entrypoint`
  - Result: **108 CU**

## Assembly Programming (sBPF assembly)

  ```rust
  .global entrypoint
  entrypoint:
    lddw r1, message
    lddw r2, 14
    call sol_log_
    exit
  .rodata
    message: .ascii "Hello, Solana!"
  ```

- **[`09:59`](https://youtu.be/Fk_UtbEny0c?t=599)** **Assembly project creation**:
  - Using the [**sBPF tool**](https://sbpf.xyz/) developed by Dean
  - Automatic generation of data offsets
    ```rust
    .equ NUM_ACCOUNTS, 0x0000
    .equ INSTRUCTION_DATA_LEN, 0X0008
    .equ INSTRUCTION_DATA, 0x0010

    .global entrypoint
    entrypoint:
      lddw r1, message
      lddw r2, 14
      call sol_log_
      exit
    .rodata
      message: .ascii "Hello, Solana!"
    ```

- **[`10:49`](https://youtu.be/Fk_UtbEny0c?t=649)** **Assembly optimization techniques**:
  - Using `r0` register to verify account count (saves a jump)
  - `r1` contains serialized data
  - `r2` used as length register for `sol_log_` system call
    ```rust
    .equ NUM_ACCOUNTS, 0x0000
    .equ INSTRUCTION_DATA_LEN, 0X0008
    .equ INSTRUCTION_DATA, 0x0010

    .global entrypoint
    entrypoint:
      lddw r0, [r1+NUM_ACCOUNTS]
      lddw r2, [r1+INSTRUCTION_DATA_LEN]
      lddw r1, INSTRUCTION_DATA
      call sol_log_
      exit
    ```

- **[`13:16`](https://youtu.be/Fk_UtbEny0c?t=796)** **Assembly performance**:
  - Assembly version: **105 CU**
  - After removing a check: **104 CU** (better than Pinocchio)
    ![](2025-05-22-14-14-17.png)
    ```rust
    .equ INSTRUCTION_DATA_LEN, 0X0008
    .equ INSTRUCTION_DATA, 0x0010

    .global entrypoint
    entrypoint:
      lddw r2, [r1+INSTRUCTION_DATA_LEN]
      lddw r1, INSTRUCTION_DATA
      call sol_log_
      exit
    ```

- **[`14:12`](https://youtu.be/Fk_UtbEny0c?t=852)** **Program size optimization**:
  - Creating a custom linker file
  - Reducing program size from 76 KB to under 1 KB
  - Enables deployment in a single transaction

- **[`14:50`](https://youtu.be/Fk_UtbEny0c?t=890)** **Final comparison**:
  - Anchor: 281 CU, 76 KB
  - Pinocchio: 108 CU, intermediate size
  - Assembly: 104 CU, under 1 KB

## Conclusions
- **[`14:58`](https://youtu.be/Fk_UtbEny0c?t=898)** Importance of choosing the right level of optimization based on needs
- **[`15:25`](https://youtu.be/Fk_UtbEny0c?t=925)** Announcement that Blueshift will publish open source content on advanced optimization
- **[`15:35`](https://youtu.be/Fk_UtbEny0c?t=935)** Mention that Blueshift just reached 1000 followers, triggering their public launch

## Key Takeaways
1. Optimization isn't an absolute goal but a balance between efficiency, maintainability, and time-to-market
2. Even small optimizations can make a big difference in CU (105 CU saved just by disabling logs)
3. Framework choice is crucial - Pinocchio offers 84% savings compared to Anchor with almost identical code
4. Advanced techniques (assembly) offer marginal but significant gains for critical applications
5. Program size has a direct impact on deployment and operational costs


----
# Links

- [Blueshift - Make the Shift. Build on Solana.](https://blueshift.gg/)
- [Create Next App](https://sbpf.xyz/)
- [Dean 利迪恩 (⚛️,🐱) | sbpf/acc](https://x.com/deanmlittle)
- [Anchor framework](https://www.anchor-lang.com)
- [GitHub - regolith-labs/steel: Solana smart contract framework.](https://github.com/regolith-labs/steel)
- [GitHub - anza-xyz/pinocchio: Create Solana programs with no dependencies attached](https://github.com/anza-xyz/pinocchio)
- [bytemuck - crates.io: Rust Package Registry](https://crates.io/crates/bytemuck)
- [bytemuck - Rust](https://docs.rs/bytemuck/latest/bytemuck/)

----

# Transcription

- [00:00:00](https://www.youtube.com/watch?v=n-ym1utpzhk?t=0) ➜ [Applause] Thank you very much. Uh, hi everyone. I'm Dean. Uh, you might have, uh, seen that I'm from, uh, something called Blue
- [00:00:08](https://www.youtube.com/watch?v=n-ym1utpzhk?t=8) ➜ Shift. I'll talk about that a bit later. Um, and today I'm going to talk about writing optimized Salana programs. Uh, something that's very dear to my heart.
- [00:00:16](https://www.youtube.com/watch?v=n-ym1utpzhk?t=16) ➜ Um, so I'll start by like the philosophy of like what is program optimization, right? And I think it comes down to three main points. We've got compute,
- [00:00:26](https://www.youtube.com/watch?v=n-ym1utpzhk?t=26) ➜ storage, and data. So when we think about optimization, most people only really think about compute I would say and really what that comes down to is
- [00:00:35](https://www.youtube.com/watch?v=n-ym1utpzhk?t=35) ➜ the lower the CU consumption of your program is the higher priority your transactions will get in the scheduling for the same fee. In other words, you
- [00:00:44](https://www.youtube.com/watch?v=n-ym1utpzhk?t=44) ➜ can save your users some money on priority fees, making your protocol or your program more competitive. The second thing is storage. the lower your
- [00:00:53](https://www.youtube.com/watch?v=n-ym1utpzhk?t=53) ➜ onchain footprint is in memory, the lower the operational cost of running your protocol and thus the the actual like cost that we see here is you even
- [00:01:03](https://www.youtube.com/watch?v=n-ym1utpzhk?t=63) ➜ though you can get your rent back, right? Technically, what we see here is actually opportunity cost, right? What could I be spending my soul on instead,
- [00:01:10](https://www.youtube.com/watch?v=n-ym1utpzhk?t=70) ➜ right? I could be earning yield, I could be securing the network by staking with a validator, right? Um, so and you also have to think about like the size of
- [00:01:19](https://www.youtube.com/watch?v=n-ym1utpzhk?t=79) ➜ your program. uh you cannot really recover that uh rent you know if you're going to keep that program on chain um so and then the third thing is data um
- [00:01:28](https://www.youtube.com/watch?v=n-ym1utpzhk?t=88) ➜ the smaller the instruction data in your program the less accounts etc the more CPI it is or the more consumable and composable it is and so the more people
- [00:01:37](https://www.youtube.com/watch?v=n-ym1utpzhk?t=97) ➜ who are going to build stuff with your program try to think about like the most common one of the most common transactions is a Jupyter swap it's
- [00:01:44](https://www.youtube.com/watch?v=n-ym1utpzhk?t=104) ➜ pretty hard to pair with too much else outside of the swap because it's so computationally intensive Right? Um, and this all really comes
- [00:01:52](https://www.youtube.com/watch?v=n-ym1utpzhk?t=112) ➜ down to one really overarching idea, which is capital efficiency, right? How do we save money or how do we not waste money? But then there's the flip side.
- [00:02:02](https://www.youtube.com/watch?v=n-ym1utpzhk?t=122) ➜ Optimization is actually not always capital efficient. And that's because people might have different priorities, right? For example, you may think that
- [00:02:09](https://www.youtube.com/watch?v=n-ym1utpzhk?t=129) ➜ you really want the most optimal thing, right? You want to optimize right down to like the best possible bite code that you can have. But then it might be
- [00:02:16](https://www.youtube.com/watch?v=n-ym1utpzhk?t=136) ➜ really hard for you to hire someone who's able to actually operate or maintain that software. Or perhaps you're also worried about like a tight
- [00:02:24](https://www.youtube.com/watch?v=n-ym1utpzhk?t=144) ➜ deadline on auditing. If you're using common frameworks and libraries that auditors are more familiar with, then typically it's going to cost you less
- [00:02:32](https://www.youtube.com/watch?v=n-ym1utpzhk?t=152) ➜ and get done quicker. And that feeds into shipping. Sometimes it's not about doing the best thing right out the gate. Sometimes it's about getting started and
- [00:02:40](https://www.youtube.com/watch?v=n-ym1utpzhk?t=160) ➜ putting something out to market before you go and fix it later. So I think uh there's the two sides of the argument here and that's why I'll never tell you
- [00:02:49](https://www.youtube.com/watch?v=n-ym1utpzhk?t=169) ➜ that uh optimization is about you must do everything some certain way right it's really about picking the right tool for the
- [00:02:58](https://www.youtube.com/watch?v=n-ym1utpzhk?t=178) ➜ job now uh with that in mind uh I'm going to go into some CU hacks that I I see uh that many programs out in the wild could benefit from so uh the
- [00:03:10](https://www.youtube.com/watch?v=n-ym1utpzhk?t=190) ➜ obvious one uh is try to avoid Salana program uh it's very inefficient. It has a lot of things like uh heap allocations and stuff like that. Um there's a lot of
- [00:03:21](https://www.youtube.com/watch?v=n-ym1utpzhk?t=201) ➜ uh really cool libraries coming out that are mostly actually written by community contributors like myself or Kave uh or you know uh even Kevin Heave's got a
- [00:03:30](https://www.youtube.com/watch?v=n-ym1utpzhk?t=210) ➜ few. Um you know so there's quite a few of them out there. Uh and there's also Pinocchio. It's a very good framework. Really recommend it. It's a good middle
- [00:03:37](https://www.youtube.com/watch?v=n-ym1utpzhk?t=217) ➜ ground between you know no optimization and like perfect optimization. and it really is like the the middle ground. Um the second thing I'd say is try to learn
- [00:03:47](https://www.youtube.com/watch?v=n-ym1utpzhk?t=227) ➜ how to use zero copy. Now this is something that works with you know Pinocchio, Steel, Ankor, whatever framework you're using. Uh you can start
- [00:03:54](https://www.youtube.com/watch?v=n-ym1utpzhk?t=234) ➜ off with bitemuck. I don't really use bitemuck. I usually do my own um you know zero copy implementations but you can try it. Uh bitemuck is actually not
- [00:04:01](https://www.youtube.com/watch?v=n-ym1utpzhk?t=241) ➜ that hard. There's plenty of examples out there and uh it's a good way to sort of get your foot in the door. Unless you're writing validator software then
- [00:04:08](https://www.youtube.com/watch?v=n-ym1utpzhk?t=248) ➜ maybe be a little careful about that. Um, now this is a really big anti-attern that I often see. Always try to put fixedsized data first. There's two
- [00:04:18](https://www.youtube.com/watch?v=n-ym1utpzhk?t=258) ➜ reasons. The first reason is if I'm trying to deserialize this this record here and I've got string, which is a variable sized uh um piece of data,
- [00:04:28](https://www.youtube.com/watch?v=n-ym1utpzhk?t=268) ➜ right? then I don't actually have a way to get the offset of pub key to des serialize that unless I've first des serialized the string stored a bunch of
- [00:04:37](https://www.youtube.com/watch?v=n-ym1utpzhk?t=277) ➜ data and figured out you know how far I got to go and the second thing is if I'm trying to actually uh filter on um RPC uh if I do it this way because the
- [00:04:47](https://www.youtube.com/watch?v=n-ym1utpzhk?t=287) ➜ string is variable length I don't know at what offset my pub key starts so it's hard to filter by pub key but if we just flip the order of these fields then all
- [00:04:55](https://www.youtube.com/watch?v=n-ym1utpzhk?t=295) ➜ of a sudden pub key we know it's 32 bytes if I look at byte zero I can filter by pub key. If I look at b by 32 I can filter by the
- [00:05:03](https://www.youtube.com/watch?v=n-ym1utpzhk?t=303) ➜ name. And you know one step beyond this it's even better is if you can try to avoid uh dynamically sized data at all. Now, a lot of people think like, "Oh, I
- [00:05:12](https://www.youtube.com/watch?v=n-ym1utpzhk?t=312) ➜ must optimize to save rent, right?" But if you think about it, right? Let's say you had like an NFT collection, right? And maybe like you want to like optimize
- [00:05:20](https://www.youtube.com/watch?v=n-ym1utpzhk?t=320) ➜ like the individual records that like, you know, there's a lot of them and they move around a lot, but maybe there's like a collection and it's like almost
- [00:05:26](https://www.youtube.com/watch?v=n-ym1utpzhk?t=326) ➜ never used, right? If accounts are almost never used, then it's almost always more worth it to pay like a couple more lamp ports and lock up a
- [00:05:33](https://www.youtube.com/watch?v=n-ym1utpzhk?t=333) ➜ couple more like a little bit more soul uh in you know in those bites to avoid the computational overhead because that's something that you know can play
- [00:05:42](https://www.youtube.com/watch?v=n-ym1utpzhk?t=342) ➜ into your CU costs forever, right? Um don't include accounts in your accounts array just to get its public key. If you're using it for signing and other
- [00:05:53](https://www.youtube.com/watch?v=n-ym1utpzhk?t=353) ➜ things, great. That's an optimization. If you're not, please don't include it. Please just like feed it in some other way in instruction data or something
- [00:06:00](https://www.youtube.com/watch?v=n-ym1utpzhk?t=360) ➜ like that. We don't need to be deserializing that account in your frameworks and whatever, right? Um, don't recmp compute PDA signers. So, uh,
- [00:06:09](https://www.youtube.com/watch?v=n-ym1utpzhk?t=369) ➜ if you're actually using a PDA as a signer, you don't need to do like the create program address thing, right? Because when you're doing the signing
- [00:06:17](https://www.youtube.com/watch?v=n-ym1utpzhk?t=377) ➜ with the PDA and you're feeding the seeds in, it's actually doing the exact same check for you. So, you can just skip this check. Um, but if you are
- [00:06:24](https://www.youtube.com/watch?v=n-ym1utpzhk?t=384) ➜ using a PDA for the purpose of just like uh let's say a sort of like a hashmap kind of uh you know way of storing your uh your data and stuff and you're not
- [00:06:34](https://www.youtube.com/watch?v=n-ym1utpzhk?t=394) ➜ using it as a ser consider once you've already made the PDA to never use create program address again. Instead try to use SHA 256 because when you're doing
- [00:06:45](https://www.youtube.com/watch?v=n-ym1utpzhk?t=405) ➜ create program address what's happening is it's first shout 256 hashing your seeds. It's got like in this case we've got assigner a bump the program ID and
- [00:06:53](https://www.youtube.com/watch?v=n-ym1utpzhk?t=413) ➜ then this PDA marker and the PDA marker is basically what causes it to be you know in a different space to everything else. It's like a salt and that says
- [00:07:02](https://www.youtube.com/watch?v=n-ym1utpzhk?t=422) ➜ program arrived derived address there. So if you actually so what it does it will like it will it will sh 56 hash and then it will actually do a check to see
- [00:07:10](https://www.youtube.com/watch?v=n-ym1utpzhk?t=430) ➜ if this address is offcurve right but if you've already made this address you already know it's offcurve then when you're using it in the future which is
- [00:07:18](https://www.youtube.com/watch?v=n-ym1utpzhk?t=438) ➜ what like this hashmap sort of pattern is with PDAs uh yeah you can skip the off curve check because there's no way the account could be there if you've
- [00:07:26](https://www.youtube.com/watch?v=n-ym1utpzhk?t=446) ➜ already handled your logic at the time of creating it. So this is like 100 CUS if you use Salana no standard SHA 256. Uh that was mine. Um you can also do it
- [00:07:36](https://www.youtube.com/watch?v=n-ym1utpzhk?t=456) ➜ with Salana program but it's 120 CUS. So this is a little bit cheaper. Give it a try. Uh skip superfluous checks. So like some people I see in their contracts
- [00:07:46](https://www.youtube.com/watch?v=n-ym1utpzhk?t=466) ➜ they'll do things like uh check if the token account has enough tokens to send tokens to some other token account. It's like token accounts already going to
- [00:07:54](https://www.youtube.com/watch?v=n-ym1utpzhk?t=474) ➜ fail. Skip that. Um, so that's like my like speedrun through a bunch of hacks. Now I'm going to show you like a real world example. And you might think, oh,
- [00:08:03](https://www.youtube.com/watch?v=n-ym1utpzhk?t=483) ➜ an anchor MIMO program. Very exciting. Um, but we can take this pretty far. So this is a very simple uh anchor MIMO program. We start with anchor 0.31
- [00:08:13](https://www.youtube.com/watch?v=n-ym1utpzhk?t=493) ➜ uh feature here. Instruction discriminator equals zero. So that's going to save us one CU just by setting the smaller discriminator. Uh, and then
- [00:08:20](https://www.youtube.com/watch?v=n-ym1utpzhk?t=500) ➜ it takes in a string and it logs it out. And you know, not very efficient. 649 CUS. Eh, you know, we can do better. But if we just activate this no
- [00:08:31](https://www.youtube.com/watch?v=n-ym1utpzhk?t=511) ➜ log exam, if you see here, it says instruction log, right? If we just deact if we activate this feature here, all of a sudden 544. We saved 105 CUS. Not bad,
- [00:08:43](https://www.youtube.com/watch?v=n-ym1utpzhk?t=523) ➜ right? Now, like let's think like how could we really really optimize anchor, right? I know it's crazy, but well, what if we just like created our own quote
- [00:08:51](https://www.youtube.com/watch?v=n-ym1utpzhk?t=531) ➜ unquote account context, right? like where it uh basically we hand rolled the uh derive accounts macro to implement try accounts and it gives us back an
- [00:09:01](https://www.youtube.com/watch?v=n-ym1utpzhk?t=541) ➜ account which is actually a raw pointer to the instruction data being passed in. Um well if we do that and we consume it using uh basically just like core string
- [00:09:10](https://www.youtube.com/watch?v=n-ym1utpzhk?t=550) ➜ from raw parts or from UTF8 unchecked or whatever uh we can get down to 281 cus. So you know who said anchor can't be fast but this is obviously absurd right?
- [00:09:21](https://www.youtube.com/watch?v=n-ym1utpzhk?t=561) ➜ A much better choice would be what if we use Pinocchio? Well, it's pretty simple, right? This is uh almost the same code. Uh optimization doesn't have to lead to
- [00:09:30](https://www.youtube.com/watch?v=n-ym1utpzhk?t=570) ➜ bad UX. Libraries are getting much better. It's getting much easier to do this sort of thing. And if you look just straight away in Pinocchio, we get 109
- [00:09:38](https://www.youtube.com/watch?v=n-ym1utpzhk?t=578) ➜ CUS just by doing the most naive implementation. And if we want to optimize it further, we can just use the lazy program entry point and that's
- [00:09:46](https://www.youtube.com/watch?v=n-ym1utpzhk?t=586) ➜ going to take us down to 108 CUS. Pretty good. So, where can we really go from here? Well, some of you might have a have a guess what we're about to do, but
- [00:09:59](https://www.youtube.com/watch?v=n-ym1utpzhk?t=599) ➜ um yeah, we're going to write we're going to try and live code some assembly. I I think it's a I think that's a first for a Salana conference.
- [00:10:07](https://www.youtube.com/watch?v=n-ym1utpzhk?t=607) ➜ So, so I'm going to do I'm going to use a tool I made called SPF. Um just SPF in. And that's going to help me quickly like scaffold this uh nice little
- [00:10:18](https://www.youtube.com/watch?v=n-ym1utpzhk?t=618) ➜ project here. While that's running, I'm gonna run SPF test to make sure that the uh Oh, I got a SPF build. One sec. I'm gonna run the test just to Oh, I'm not
- [00:10:28](https://www.youtube.com/watch?v=n-ym1utpzhk?t=628) ➜ in the directory. That would be why. Cool. So, I've run SPF test. I'm just going to get that running so that the Rust crates install um for my testing
- [00:10:37](https://www.youtube.com/watch?v=n-ym1utpzhk?t=637) ➜ purposes. Okay. So, here we got uh a very simple log program, but this is not a MIMO program, right? A MIMO needs to take in some instruction data. Well, if
- [00:10:46](https://www.youtube.com/watch?v=n-ym1utpzhk?t=646) ➜ we go over here to this app I made, which is spf.xyz, uh we have the ability to basically uh say how many accounts we have, right?
- [00:10:57](https://www.youtube.com/watch?v=n-ym1utpzhk?t=657) ➜ And it will just automatically generate all the offsets of where each of these pieces of data would be uh based on the number of accounts that you have. Uh and
- [00:11:05](https://www.youtube.com/watch?v=n-ym1utpzhk?t=665) ➜ you can do use it for rust and C. So if you're doing C, you know, or or Rust, which is like unsafe Rust, which is basically C. Um then, uh yeah, you can
- [00:11:13](https://www.youtube.com/watch?v=n-ym1utpzhk?t=673) ➜ use uh you can use the offsets there, too. In this case, we're going to have no accounts. Um so we're going to just copy these three offsets. Um and then
- [00:11:21](https://www.youtube.com/watch?v=n-ym1utpzhk?t=681) ➜ we're going to go back to our VS Code and just throw these at the top here, right? And uh this this will take you forever if you don't use this tool. Like
- [00:11:29](https://www.youtube.com/watch?v=n-ym1utpzhk?t=689) ➜ alignment and like figuring out where everything lives is is kind of hard, but um solved it for you. So the first thing that uh I'm going to do is I'm going to
- [00:11:39](https://www.youtube.com/watch?v=n-ym1utpzhk?t=699) ➜ uh I'm going to load the number of accounts into R0. Does anyone have any clue why I want to do
- [00:11:48](https://www.youtube.com/watch?v=n-ym1utpzhk?t=708) ➜ that? So R0 is the return register and what happens in the VM is if you return the number zero that is seen as success. If you return any number other than
- [00:11:59](https://www.youtube.com/watch?v=n-ym1utpzhk?t=719) ➜ zero, it's seen as an error code. And so this is a way for us to prove basically that the user put in zero accounts, right? It's a very nice hack that
- [00:12:08](https://www.youtube.com/watch?v=n-ym1utpzhk?t=728) ➜ doesn't cost us, you know, anything. We're basically uh one CU for this. And uh yeah, it it's just like a nice quick check to prove that there's zero
- [00:12:16](https://www.youtube.com/watch?v=n-ym1utpzhk?t=736) ➜ accounts in this uh instruction. We don't have to do any like jump not equal or anything like that. Pretty cool. This is only possible in assembly by the way.
- [00:12:23](https://www.youtube.com/watch?v=n-ym1utpzhk?t=743) ➜ Rust cannot really do this. Um so that's like step one, right? Uh step two, we're also going to load a double word into R2. And in this case, we're actually
- [00:12:33](https://www.youtube.com/watch?v=n-ym1utpzhk?t=753) ➜ going to load in the uh instruction. I'll just copy it. Uh we're going to load in the instruction data length into R2 because R2 register 2 is used as the
- [00:12:43](https://www.youtube.com/watch?v=n-ym1utpzhk?t=763) ➜ length register in the soul log sys call. Right? And then uh finally, what we're going to do is uh we're going to do an add
- [00:12:52](https://www.youtube.com/watch?v=n-ym1utpzhk?t=772) ➜ 64. Uh sorry, we're using R1, by the way. R1 is where all of your data gets serialized to in the VM. Um, so that's you need to use R1 to get the correct
- [00:13:01](https://www.youtube.com/watch?v=n-ym1utpzhk?t=781) ➜ data at these offsets. And then finally, what we do is we go to this R1 here and we add the offset of instruction data. Right? Um, now I'm going to clear out
- [00:13:13](https://www.youtube.com/watch?v=n-ym1utpzhk?t=793) ➜ this uh readonly data that says hello Salana here. Cool. And uh this should should maybe maybe I screwed up. Let's find out. This should be a functional
- [00:13:22](https://www.youtube.com/watch?v=n-ym1utpzhk?t=802) ➜ piece of code. And I'm going to try and just uh put some instruction data here. So let's try hello scale or die. Cool. And like let's try and run
- [00:13:33](https://www.youtube.com/watch?v=n-ym1utpzhk?t=813) ➜ SPF build. And that built successfully. Now run SPF test. And cool. We are at 105 CUS. So we managed to beat Pinocchio by three CUS.
- [00:13:47](https://www.youtube.com/watch?v=n-ym1utpzhk?t=827) ➜ Now, uh, you know, if we really want to go all in, uh, let's say that it's a scenario where we control how this is being invoked. So, we know there's going
- [00:13:56](https://www.youtube.com/watch?v=n-ym1utpzhk?t=836) ➜ to be no accounts. And guess what? We can skip a check, right? Cool. Let's try again. And we're down to uh 104. So, we beat Pinocchio by four CUS, right? Uh,
- [00:14:09](https://www.youtube.com/watch?v=n-ym1utpzhk?t=849) ➜ this is what we call extreme optimization. And I'm going to throw in one more because I think I can do this in about 30 seconds. We can actually
- [00:14:16](https://www.youtube.com/watch?v=n-ym1utpzhk?t=856) ➜ create a custom linker file to uh decrease the the size of our uh binary because if we go here it should be what deploy slash
- [00:14:28](https://www.youtube.com/watch?v=n-ym1utpzhk?t=868) ➜ mimo s is it cool. So we see here it's40 bytes. So if we knock off some of these uh unnecessary headers here and we uh do
- [00:14:39](https://www.youtube.com/watch?v=n-ym1utpzhk?t=879) ➜ another build with a custom linker uh we can see we can get it down to 928 bytes. uh pretty great. So, you know, free lamports, you don't want to pay that
- [00:14:48](https://www.youtube.com/watch?v=n-ym1utpzhk?t=888) ➜ rent. So, you might be looking at this and thinking this is all kind of crazy. Uh it is. Um but that's the reason why like you have to choose like what level
- [00:14:58](https://www.youtube.com/watch?v=n-ym1utpzhk?t=898) ➜ of optimization you're going for. It's really about what's the best tool for the job. And if we have a look at this quick side by side, I'm sorry I am
- [00:15:05](https://www.youtube.com/watch?v=n-ym1utpzhk?t=905) ➜ running out of time. anchor we got it down to 281 Pinocchio 108 assembly 104 and if we look at the file size from 76 bytes all the way down down to 76
- [00:15:15](https://www.youtube.com/watch?v=n-ym1utpzhk?t=915) ➜ kilobytes down to under 1 kilobyte you could actually uh deploy that in a single transaction just uh maybe just about so where do I go to learn more
- [00:15:25](https://www.youtube.com/watch?v=n-ym1utpzhk?t=925) ➜ about this sort of thing um well yeah blue shift as I said uh we're open sourcing a whole bunch of content for advanced program optimization uh you can
- [00:15:35](https://www.youtube.com/watch?v=n-ym1utpzhk?t=935) ➜ Check us out on Twitter. We're going public at 1,000 followers, uh, which we actually just hit. And, uh, yeah, thank you all for listening.
