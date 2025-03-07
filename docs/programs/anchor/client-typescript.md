---
title: JS/TS Client
description:
  Learn how to use Anchor's TypeScript client library to interact with Solana
  program
sidebarLabel: JS/TS Client
sidebarSortOrder: 3
---

Anchor provides a Typescript client library
([@coral-xyz/anchor](https://github.com/coral-xyz/anchor/tree/v0.30.1/ts/packages/anchor))
that simplifies the process of interacting with Solana programs from the client
in JavaScript or TypeScript.

## Client Program

To use the client library, first create an instance of a
[`Program`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/index.ts#L58)
using the [IDL file](/docs/programs/anchor/idl) generated by Anchor.

Creating an instance of the `Program` requires the program's IDL and an
[`AnchorProvider`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/provider.ts#L55).
An `AnchorProvider` is an abstraction that combines two things:

- `Connection` - the connection to a [Solana cluster](/docs/core/clusters.md)
  (i.e. localhost, devnet, mainnet)
- `Wallet` - (optional) a default wallet used to pay and sign transactions

<!-- prettier-ignore -->
<Tabs items={['Frontend/Node', 'Test File']}>
<Tab value="Frontend/Node">

When integrating with a frontend using the
[wallet adapter](https://solana.com/developers/guides/wallets/add-solana-wallet-adapter-to-nextjs),
you'll need to set up the `AnchorProvider` and `Program`.

```ts {9-10, 12-14}
import { Program, AnchorProvider, setProvider } from "@coral-xyz/anchor";
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react";
import type { HelloAnchor } from "./idlType";
import idl from "./idl.json";

const { connection } = useConnection();
const wallet = useAnchorWallet();

const provider = new AnchorProvider(connection, wallet, {});
setProvider(provider);

export const program = new Program(idl as HelloAnchor, {
  connection,
});
```

In the code snippet above:

- `idl.json` is the IDL file generated by Anchor, found at
  `/target/idl/<program-name>.json` in an Anchor project.
- `idlType.ts` is the IDL type (for use with TS), found at
  `/target/types/<program-name>.ts` in an Anchor project.

Alternatively, you can create an instance of the `Program` using only the IDL
and the `Connection` to a Solana cluster. This means there is no default
`Wallet`, but allows you to use the `Program` to fetch accounts or build
instructions without a connected wallet.

```ts {8-10}
import { clusterApiUrl, Connection, PublicKey } from "@solana/web3.js";
import { Program } from "@coral-xyz/anchor";
import type { HelloAnchor } from "./idlType";
import idl from "./idl.json";

const connection = new Connection(clusterApiUrl("devnet"), "confirmed");

export const program = new Program(idl as HelloAnchor, {
  connection,
});
```

</Tab>
<Tab value="Test File">

Anchor automatically sets up a `Program` instance in the default test file of
new projects. However, this setup differs from how you'd initialize a `Program`
outside the Anchor workspace, such as in React or Node.js applications.

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { HelloAnchor } from "../target/types/hello_anchor";

describe("hello_anchor", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.HelloAnchor as Program<HelloAnchor>;

  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods.initialize().rpc();
    console.log("Your transaction signature", tx);
  });
});
```

</Tab>
</Tabs>

## Invoke Instructions

Once the `Program` is set up using a program IDL, you can use the Anchor
[`MethodsBuilder`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/methods.ts#L155)
to:

- Build individual instructions
- Build transactions
- Build and send transactions

The basic format looks like the following:

<!-- prettier-ignore -->
<Tabs items={['methods', 'instruction', 'accounts', `signers`]}>
<Tab value="methods">

`program.methods` - This is the builder API for creating instruction calls from
the program's IDL

```ts /methods/ {1}
await program.methods
  .instructionName(instructionData)
  .accounts({})
  .signers([])
  .rpc();
```

</Tab>
<Tab value="instruction">

Following `.methods`, specify the name of an instruction from the program IDL,
passing in any required arguments as comma-separated values.

```ts /instructionName/ /instructionData1/ /instructionData2/ {2}
await program.methods
  .instructionName(instructionData1, instructionData2)
  .accounts({})
  .signers([])
  .rpc();
```

</Tab>
<Tab value="accounts">

`.accounts` - Pass in the address of the accounts required by the instruction as
specified in the IDL

```ts /accounts/ {3}
await program.methods
  .instructionName(instructionData)
  .accounts({})
  .signers([])
  .rpc();
```

Note that certain account addresses don't need to be explicitly provided, as the
Anchor client can automatically resolve them. These typically include:

- Common accounts (ex. the System Program)
- Accounts where the address is a PDA (Program Derived Address)

</Tab>
<Tab value="signers">

`.signers` - Optionally pass in an array of keypairs required as additional
signers by the instruction. This is commonly used when creating new accounts
where the account address is the public key of a newly generated keypair.

```ts /signers/ {4}
await program.methods
  .instructionName(instructionData)
  .accounts({})
  .signers([])
  .rpc();
```

Note that `.signers` should only be used when also using `.rpc()`. When using
`.transaction()` or `.instruction()`, signers should be added to the transaction
before sending.

</Tab>
</Tabs>

Anchor provides multiple methods for building program instructions:

<!-- prettier-ignore -->
<Tabs items={['.rpc', '.transaction', '.instruction']}>
<Tab value=".rpc">

The
[`rpc()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/methods.ts#L283)
method
[sends a signed transaction](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/rpc.ts#L29)
with the specified instruction and returns a `TransactionSignature`.

When using `.rpc`, the `Wallet` from the `Provider` is automatically included as
a signer.

```ts {13}
// Generate keypair for the new account
const newAccountKp = new Keypair();

const data = new BN(42);
const transactionSignature = await program.methods
  .initialize(data)
  .accounts({
    newAccount: newAccountKp.publicKey,
    signer: wallet.publicKey,
    systemProgram: SystemProgram.programId,
  })
  .signers([newAccountKp])
  .rpc();
```

</Tab>
<Tab value=".transaction">

The
[`transaction()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/methods.ts#L382)
method
[builds a `Transaction`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/transaction.ts#L18-L26)
with the specified instruction without sending the transaction.

```ts {12} /transaction()/1,2,4
// Generate keypair for the new account
const newAccountKp = new Keypair();

const data = new BN(42);
const transaction = await program.methods
  .initialize(data)
  .accounts({
    newAccount: newAccountKp.publicKey,
    signer: wallet.publicKey,
    systemProgram: SystemProgram.programId,
  })
  .transaction();

const transactionSignature = await connection.sendTransaction(transaction, [
  wallet.payer,
  newAccountKp,
]);
```

</Tab>
<Tab value=".instruction">

The
[`instruction()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/methods.ts#L348)
method
[builds a `TransactionInstruction`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/instruction.ts#L57-L61)
using the specified instruction. This is useful if you want to manually add the
instruction to a transaction and combine it with other instructions.

```ts {12} /instruction()/
// Generate keypair for the new account
const newAccountKp = new Keypair();

const data = new BN(42);
const instruction = await program.methods
  .initialize(data)
  .accounts({
    newAccount: newAccountKp.publicKey,
    signer: wallet.publicKey,
    systemProgram: SystemProgram.programId,
  })
  .instruction();

const transaction = new Transaction().add(instruction);

const transactionSignature = await connection.sendTransaction(transaction, [
  wallet.payer,
  newAccountKp,
]);
```

</Tab> 
</Tabs>

## Fetch Accounts

The `Program` client simplifies the process of fetching and deserializing
accounts created by your Anchor program.

Use `program.account` followed by the name of the account type defined in the
IDL. Anchor provides multiple methods for fetching accounts.

<!-- prettier-ignore -->
<Tabs items={['all', 'memcmp', 'fetch', 'fetchMultiple']}>
<Tab value="all">

Use
[`all()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/account.ts#L251)
to fetch all existing accounts for a specific account type.

```ts /all/
const accounts = await program.account.newAccount.all();
```

</Tab>
<Tab value="memcmp">

Use `memcmp` (memory compare) to filter for account data that matches a specific
value at a specific offset. Using `memcmp` requires you to understand the byte
layout of the data field for the account type you are fetching.

When calculating the offset, remember that the first 8 bytes in accounts created
by an Anchor program are reserved for the account discriminator.

```ts /memcmp/
const accounts = await program.account.newAccount.all([
  {
    memcmp: {
      offset: 8,
      bytes: "",
    },
  },
]);
```

</Tab>
<Tab value="fetch">

Use
[`fetch()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/account.ts#L165)
to fetch the account data for a single account

```ts /fetch/
const account = await program.account.newAccount.fetch(ACCOUNT_ADDRESS);
```

</Tab>
<Tab value="fetchMultiple">

Use
[`fetchMultiple()`](https://github.com/coral-xyz/anchor/blob/v0.30.1/ts/packages/anchor/src/program/namespace/account.ts#L200)
to fetch the account data for multiple accounts by passing in an array of
account addresses

```ts /fetchMultiple/
const accounts = await program.account.newAccount.fetchMultiple([
  ACCOUNT_ADDRESS_ONE,
  ACCOUNT_ADDRESS_TWO,
]);
```

</Tab>
</Tabs>
