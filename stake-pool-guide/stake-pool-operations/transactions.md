# Create a simple transaction

You will need a new **payment address** \(payment2.addr\) so that you can send funds to it. First, get a new payment key pair for that:

```text
cardano-cli address key-gen \
--verification-key-file payment2.vkey \
--signing-key-file payment2.skey
```

This has created two new keys: **payment2.vkey** and **payment2.skey**

To generate **payment2.addr** we can use the same stake key pair that we already have:

```text
cardano-cli address build \
--payment-verification-key-file payment2.vkey \
--out-file payment2.addr \
--testnet-magic 1097911063
```

Let's send 100 ada (100,000,000 lovelace) from `payment.addr` to `payment2.addr`

Creating a transaction is a process that requires various steps:

* Get the protocol parameters
* Define the time-to-live \(TTL\) for the transaction
* Calculate the fee
* Build the transaction
* Sign the transaction
* Submit the transaction

## Get protocol parameters

Get the protocol parameters and save them to `protocol.json` with:

```text
cardano-cli query protocol-parameters \
   --mary-era \
   --testnet-magic 1097911063 \
   --out-file protocol.json
```

## Determine the TTL \(Time To Live\) for the transaction

We need the CURRENT TIP of the blockchain, this is, the height of the last block produced. We are looking for the value of `SlotNo`

```text
cardano-cli query tip --testnet-magic 1097911063

{
    "blockNo": 16829,
    "headerHash": "3e6f59b10d605e7f59ba8383cb0ddcd42480ddcc0a85d41bad1e4648eb5465ad",
    "slotNo": 369215
}
```

So at this moment the tip is on block: 369215
.

To build the transaction we need to specify the **TTL \(Time To Live\)**, this is the block height limit for our transaction to be included in a block, if it is not in a block by that slot the transaction will be cancelled.

From `protocol.json` we know that we have 1 slot per second. Lets say that it will take us 10 minutes to build the transaction, and that we want to give it another 10 minutes window to be included in a block. So we need 20 minutes or 1200 slots. So we add 1200 to the current tip: 369215 + 1200 = 370415. So our TTL is 370415

## Draft the transaction
We create a draft for the transaction and save it in tx.raw

We need the transaction hash and index of the **UTXO** we want to spend:

```text
cardano-cli query utxo \
--allegra-era \
--address $(cat payment.addr) \
--testnet-magic 1097911063

>                            TxHash                                 TxIx        Lovelace
> ----------------------------------------------------------------------------------------
> 4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99     1       1000000000
```

Note that for `--tx-in` we use the following syntax: `TxId#TxIx` where `TxId` is the transaction hash and `TxIx` is the index (you found these values in the above step) they will be added like this ```4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99#1```.

For `--tx-out` we use: `TxOut+Lovelace` where `TxOut` is the hex encoded address followed by the amount in `Lovelace`. --tx-out $(cat payment.addr)+0, --ttl, --fees, are all 0 for now. We will revisit these in a later step after we calculate fees and ttl.

```text
cardano-cli transaction build-raw \
--tx-in 4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99#1 \ 
--tx-out $(cat payment2.addr)+100000000 \
--tx-out $(cat payment.addr)+0 \
--ttl 0 \
--fee 0 \
--out-file tx.raw
```

## Calculate the fee

The transaction needs one \(1\) input: a valid UTXO from `payment.addr`, and two \(2\) outputs: The receiving address **payment2.addr** and an address to send the change back, in this case we use **payment.addr.** You also need to include the **Draft transaction file. Witnesses** are \_\*\*\_number of signing keys used to sign the transaction.

```text
   cardano-cli transaction calculate-min-fee \
   --tx-body-file tx.raw \
   --tx-in-count 1 \
   --tx-out-count 2 \
   --witness-count 1 \
   --byron-witness-count 0 \
   --testnet-magic 1097911063 \
   --protocol-params-file protocol.json


   > 174433 Lovelace
```

\(`--testnet-magic 1097911063` identifies the Shelley Testnet. Other testnets will use other numbers, and mainnet uses `--mainnet` instead.\)

So we need to pay **174433 lovelace** fee to create this transaction.

Assuming we want to send 100 ada to **payment2.addr** spending a **UTXO** containing 1,000,000 ada \(1,000,000,000,000 lovelace\), now we need to calculate how much is the **change** to send back to **payment.addr**

```text
expr 1000000000 - 100000000 - 174433

> 899825567
```

## Build the transaction

Now we revisit the `tx.raw` file from the ["Draft the transaction"](#draft-the-transaction) section above.

Note that all the arguments will be the same for the first three lines, but we will now add the values we discovered above for the next 3 lines.


```text
cardano-cli transaction build-raw \
--tx-in 4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99#1 \
--tx-out $(cat payment2.addr)+100000000 \
--tx-out $(cat payment.addr)+899825567 \
--ttl 370415 \
--fee 174433 \
--out-file tx.raw
```

## Sign the transaction

Sign the transaction with the signing key **payment.skey** and save the signed transaction in **tx.signed**

```text
cardano-cli transaction sign \
--tx-body-file tx.raw \
--signing-key-file payment.skey \
--testnet-magic 1097911063 \
--out-file tx.signed
```

## Submit the transaction

Make sure that your node is running and set CARDANO\_NODE\_SOCKET\_PATH variable to:

```text
export CARDANO_NODE_SOCKET_PATH=~/cardano-node/relay/db/node.socket
```

and submit the transaction:

```text
cardano-cli transaction submit \
        --tx-file tx.signed \
        --testnet-magic 1097911063
```

## Check the balances

We must give it some time to get incorporated into the blockchain, but eventually, we will see the effect:

```text
    cardano-cli query utxo \
        --allegra-era \
        --address $(cat payment.addr) \
        --testnet-magic 1097911063

    >                            TxHash                                 TxIx        Lovelace
    > ----------------------------------------------------------------------------------------
    > b64ae44e1195b04663ab863b62337e626c65b0c9855a9fbb9ef4458f81a6f5ee     1        899825567

    cardano-cli query utxo \
        --allegra-era \
        --address $(cat payment2.addr) \
        --testnet-magic 1097911063

    >                            TxHash                                 TxIx        Lovelace
    > ----------------------------------------------------------------------------------------
    > b64ae44e1195b04663ab863b62337e626c65b0c9855a9fbb9ef4458f81a6f5ee     0         100000000
```

{% hint style="info" %}
QUESTIONS AND FEEDBACK

If you have any questions and suggestions while taking the lessons please feel free to ask in the [forum](https://forum.cardano.org/c/english/operators-talk/119) and we will respond as soon as possible.
{% endhint %}

