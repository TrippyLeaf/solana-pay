---
title: Wallet Integration
---

This section describes how a wallet provider can support payment links in their wallet. It shows how to parse the payment URL and create a transaction from it.

This guide walks through an **example** implementation for wallet providers. The purpose of this is to make it easy for wallets to implement the protocol correctly.

---

## 1. Set up Solana Pay

Install the packages and import them in your code.

**npm**

```shell=
npm install @solana/pay @solana/web3.js --save
```

**yarn**

```shell=
yarn add @solana/pay @solana/web3.js
```

## 2. Parse payment request link

As a wallet provider, you will have to parse the received URL to extract the parameters.

<details>
    <summary>Parse the URL to retrieve all possible fields:</summary>

<br/>

```ts
import { parseURL } from '@solana/pay';

/**
 * For example only
 *
 * The URL that triggers the wallet interaction; follows the Solana Pay URL scheme
 * The parameters needed to create the correct transaction is encoded within the URL
 */
const url =
    'solana:mvines9iiHiQTysrwkJjGf2gb9Ex9jXJX8ns3qwf2kN?amount=0.01&reference=82ZJ7nbGpixjeDCmEhUcmwXYfvurzAgGdtSMuHnUgyny&label=Michael&message=Thanks%20for%20all%20the%20fish&memo=OrderId5678';
const { recipient, amount, splToken, reference, label, message, memo } = parseURL(url);
```

See [full code snippet][9]

</details>

<br/>

The `recipient` is a native SOL address and the payee.

The parsed `amount` is always interpreted to be a decimal number of "user" units. For SOL, that's SOL and not lamports. If the provided decimal fractions exceed nine for SOL or the token specific mint decimal, the URL must be considered a malformed URL and rejected. If there is no `amount`, you **must** prompt the user to enter an amount.

Potential use cases where the amount could be empty:

- accepting donations
- accepting tips

The `label` and `message` are only for display and are not encoded into the on-chain transaction.

The `memo` can be used to record a message on-chain with the transaction.

The `reference` allow for the transaction to be located on-chain. For this, you should use a random, unique public key. You can think of this as a unique ID for the payment request that the Solana Pay protocol uses to locate the transaction.

The `spl-token` parameter is optional. If empty, it symbolizes this transfer is for native SOL. Otherwise, it's the SPL token mint address. The provided decimal fractions in the `amount` must not exceed the decimal count for this mint. Otherwise, the URL must be considered malformed.

## 3. Create transaction

Use the `createTransaction` function to create a transaction with the parameters from the `parseURL` function with an additional `payer`.

The `payer` **should** be the public key of the current users' wallet.

<details>
    <summary>Create transaction reference implementation</summary>

<br/>

```typescript
import { parseURL, createTransaction } from '@solana/pay';

const url =
    'solana:mvines9iiHiQTysrwkJjGf2gb9Ex9jXJX8ns3qwf2kN?amount=0.01&reference=82ZJ7nbGpixjeDCmEhUcmwXYfvurzAgGdtSMuHnUgyny&label=Michael&message=Thanks%20for%20all%20the%20fish&memo=OrderId5678';
const { recipient, amount, splToken, reference, label, message, memo } = parseURL(url);

/**
 * Create the transaction with the parameters decoded from the URL
 */
const payer = CUSTOMER_WALLET.publicKey;
const tx = await createTransaction(connection, payer, recipient, amount as BigNumber, {
    reference,
    memo,
});
```

See [full code snippet][10]

</details>

<br/>

This transaction **should** represent the original intent of the payment request link. The example implementation walks through the steps on how to construct the transaction:

### Native SOL transfer

1. Check that the payer and recipient accounts exist
2. Check the payer and recipient are valid native accounts
3. Check the payer has enough lamports for the transfer
4. Create an instruction to transfer native SOL
5. If references were included, add them to the instruction
6. If a memo was included, create an instruction for the memo program

### SPL token transfer

1. Check that the payer and recipient accounts exist
2. Check the token provided is an initialized mint
3. Check the payer and recipient's Associated Token Account (ATA) exists
4. Check the payer has enough lamports for the transfer
5. Create an instruction to transfer SPL tokens
6. If references were included, add them to the instruction
7. If a memo was included, create an instruction for the memo program

## 4. Complete transaction

With the transaction formed. The user must be prompted to approve the transaction.

The `label` and `message` **should** be shown to the user, as it gives added context to the user on the transaction.

<details>
    <summary>
        Finally, use <code>sendAndConfirmTransaction</code> to complete the transaction.
    </summary>

```typescript
const { recipient, message, memo, amount, reference, label } = parseURL(url);
console.log('label: ', label);
console.log('message: ', message);

/**
* Create the transaction with the parameters decoded from the URL
*/
const tx = await createTransaction(connection, CUSTOMER_WALLET.publicKey, recipient, amount as BigNumber, {
reference,
memo,
});

/**
* Send the transaction to the network
*/
sendAndConfirmTransaction(connection, tx, [CUSTOMER_WALLET]);
```

See [full code snippet][11]

</details>



<!-- References -->

[1]: https://github.com/kozakdenys/qr-code-styling
[2]: https://spl.solana.com/memo
[3]: https://github.com/solana-labs/solana/issues/19535
[4]: https://github.com/solana-labs/solana-pay/tree/master/point-of-sale
[5]: https://github.com/solana-labs/solana-pay/tree/master/core/example/payment-flow-merchant
[6]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/simulateCheckout.ts
[7]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/main.ts#L61
[8]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/main.ts#L105
[9]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/simulateWalletInteraction.ts#L13
[10]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/simulateWalletInteraction.ts#L27
[11]: https://github.com/solana-labs/solana-pay/blob/master/core/example/payment-flow-merchant/simulateWalletInteraction.ts#L35
