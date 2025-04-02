
## How this works

This repository is divided into two parts. There is a `program/`
directory which contains the smart contract that is actually deployed
to the Solana blockchain and a `client/` program which handles
collecting funds, creating accounts, and invoking the deployed
program.

Details about how the deployed program works can be found
[here](https://github.com/solana-labs/example-helloworld#learn-about-the-on-chain-program). The
actually interesting part of this repository is the client which I'll
discuss below.

## The client program

In order to execute a program that has been deployed to the Solana
blockchain we need the following things:

1. A connection to a Solana cluster
2. An account with a large enough balance to pay for the program's
   execution.
3. An account that we will transfer to the program to store state for
   the program.

## The technical details

I'll now take some time to walk through the technical details of how
the client collects what is needed and then submits the transaction to
Solana.

### Connection establishment

The function `establish_connection` in `client/src/client.rs` creates
a RPC connection to Solana over which we'll do all of our
communication with the blockchain. The URL that we use for connecting
to the blockchain is read from the Solana config in
`~/.config/solana/cli/config.yml`. It can be changed by running
`solana config set --url <URL>`.

### The data that we will be storing

Our program that we will deploy tracks the number of times that a
given user has said hello to it. This requires a small amount of state
which we represent with the following Rust structure:

```rust
struct GreetingSchema {
    counter: u32,
}
```

In order to make sure that this data can be serialized and
unserialized independently to how Rust lays out a struct like that we
use a serialization protocol called [borsh](https://borsh.io/). We can
determine the size of this after serialization by serializing an
instance of this struct with borsh and then getting its length.

### Determining the balance requirement

To determine how much the program invocation will cost we use the
function `get_balance_requirement` located in
`client/src/client.rs`. The total cost of the invocation will be the
cost of submitting the invocation transaction and the cost of storing
the program state on the blockchain.

On Solana the cost of storing data on the blockchain is zero if that
data is inside an account with a balance greater than the cost of two
years of rent. You can read more about that
[here](https://docs.solana.com/implemented-proposals/rent).

It appears as if the standard is to load two years of rent into
accounts so that is what we do. Source: the "programs and accounts"
section of
[this](https://2501babe.github.io/posts/solana101.html#programs%20and%20accounts)
writeup. We can determine what this "two years of rent" amount is by
running `connection.get_minimum_balance_for_rent_exemption(data_size)`
where `data_size` is the amount of data that we will be storing in
bytes.

### Determining the payer and their balance

In order to determine who will be paying for this transaction we once
again consult the solana config in
`~/.config/solana/cli/config.yml`. In that file there is a
`keypair_path` field and we read the keypair from where that points.

To determine the payer's balance we use our connection and run
`connection.get_balance(player_pubkey)`.

### Increasing the balance if there are not enough funds

If there are not enough funds in the payer's account we will need to
airdrop funds there. This is done via the `request_airdrop` function
in `client/src/client.rs`. Airdrops are only available on test
networks so this will not work on the mainnet.

### Locating the program that we will execute

Both the payer and the program are accounts on Solana. The only
difference is that the program account is executable.

Our client program takes a single argument which is the path to the
keypair of the program that has been deployed to the blockchain. In
the `get_program` function located in `client/src/client.rs` the
keypair is loaded again and then it is verified that the specified
program is executable.

## Creating an account to store state

The function `create_greeting_account` in `client/src/client.rs`
handles the creation of an account to store program state in. This is
the most complicated part of our client program as the address of the
account must be derived from the payer's public key, the program's
public key, and a seed phrase.

The reason that we derive the address of the storage account like this
is so that it can be located later without storing any state across
invocations of the client. Solana supports this method of account
creation with the `create_account_with_seed` system instruction.

Arguments to this instruction are poorly documented and different
across the Typescript and Rust SDKs. Here are what they are and their
meanings in the Rust SDK:

- `from_pubkey` the public key of the creator of the new account. In
  our case this is the payer's public key.
- `to_pubkey` the public key that the generated account will have. In
  our case this is the public key that we generate.
- `base` the payer's public key as it is the "base" in the derivation
  of the generated account's public key. The other ingredients being
  the program's public key and the seed phrase.
- `seed` the seed phrase that was used in the generation of the
  generated account's public key.
- `lamports` the number of lamports to send to the generated
  account. In our case this is equal to the amount of lamports
  required to live on the chain rent free.
- `space` the size of the data that will be stored in the generated
  account.
- `owner` the owner of the generated account. In our case this is the
  program's public key.

You may ask yourself after reading this why we need to both provide
all of the ingredients needed to generate the new accounts public key
(also called its address) and the public key that we have generated
with those ingredients. The Solana source code seems to suggest that
this is some method for error checking but it seems slightly shitty to
me.

## Sending the "hello" transaction

Sending the hello transaction to the program is actually the easy
part. It is done in the `say_hello` function in
`client/src/client.rs`. This function just creates a new instruction
with the generated storage account as an argument and sends it to the
program that we deployed.

## Querying the account that stores state

We can query the state of our generated account and thus determine the
output of our program using the `get_account` method on our
connection. This is done in the `count_greetings` function in
`client/src/client.rs`.
