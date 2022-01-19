# Interchain Accounts

### Warning 
> Beware of dragons!</br></br>
> The interchain accounts module is currently under development and has been moved to the `ibc-go` repo [here](https://github.com/cosmos/ibc-go/tree/main/modules/apps/27-interchain-accounts). Interchain Accounts is aiming to release in January 2022.</br></br>
> This repo aims to demonstrate demo modules that utilize interchain accounts and serve as a developer guide for teams aiming to use interchain accounts functionality.</br></br>
> The existing demo outlined below will be updated to coincide with the interchain accounts release.</br>

### Developer Documentation

> Coming soon! 

## Local Demo

### Setup

```bash
# Clone this repository and build
git clone https://github.com/cosmos/interchain-accounts.git
cd interchain-accounts
make install 

# Hermes Relayer
# [Hermes](https://hermes.informal.systems/) is a Rust implementation of a relayer for the [Inter-Blockchain Communication (IBC)](https://ibcprotocol.org/) protocol.
#
# Currently supported by Hermes v0.9.0
# 
# Alternatively set a custom binary path using $HERMES_BINARY in /network/hermes/variables.sh
cargo install --version 0.9.0 ibc-relayer-cli --bin hermes --locked

# Bootstrap two chains & create an IBC connection
make init

# Start the hermes relayer
make start-rly
```

### Demo

For the purposes of this demo the setup scripts have been provided with a set of hardcoded mnemonics. 
This generates deterministic wallet addresses that are used below.

#### Registering an Interchain Account via IBC

Register an Interchain Account using the `intertx register` command where the message signer is the account owner.

```bash
# Open a seperate terminal

# Store the following account addresses within the current shell env
export DEMOWALLET_1=$(icad keys show demowallet1 -a --keyring-backend test --home ./data/test-1) && echo $DEMOWALLET_1;
export DEMOWALLET_2=$(icad keys show demowallet2 -a --keyring-backend test --home ./data/test-2) && echo $DEMOWALLET_2;

# Register an interchain account on behalf of DEMOWALLET_1 where chain test-2 is the interchain accounts host
icad tx intertx register --from $DEMOWALLET_1 --connection-id connection-0 --chain-id test-1 --gas 150000 --home ./data/test-1 --node tcp://localhost:16657 --keyring-backend test -y

# Query the address of the interchain account
icad query intertx interchainaccounts $DEMOWALLET_1 --home ./data/test-1 --node tcp://localhost:16657

# Store the interchain account address by parsing the query result: cosmos1hd0f4u7zgptymmrn55h3hy20jv2u0ctdpq23cpe8m9pas8kzd87smtf8al
export ICA_ADDR=$(icad query intertx interchainaccounts $DEMOWALLET_1 --home ./data/test-1 --node tcp://localhost:16657 -o json | jq -r '.interchain_account_address') && echo $ICA_ADDR
```

#### Funding the Interchain Account wallet

Allocate funds to the new Interchain Account wallet by using the `bank send` command.

```
# Check the interchain account's balance on test-2 chain. It should be empty.
icad q bank balances $ICA_ADDR --chain-id test-2 --node tcp://localhost:26657

# Send some assets to $ICA_ADDR.
icad tx bank send $DEMOWALLET_2 $ICA_ADDR 10000stake --chain-id test-2 --home ./data/test-2 --node tcp://localhost:26657 --keyring-backend test -y

# Check that the balance has been updated
icad q bank balances $ICA_ADDR --chain-id test-2 --node tcp://localhost:26657

# Test sending assets from interchain account via ibc.
icad tx intertx send $ICA_ADDR $DEMOWALLET_2 5000stake --chain-id test-1 --gas 90000 --home ./data/test-1 --node tcp://localhost:16657 --from $DEMOWALLET_1 --keyring-backend test -y

# Wait until the relayer has relayed the packet

# Query the interchain account balance and observe the changes in funds
icad q bank balances $ICA_ADDR --chain-id test-2 --node tcp://localhost:26657
```

#### Sending Interchain Account transactions

Send transactions to be executed using the new Interchain Account.

1. Staking Delegation

```
# Output the host chain validator operator address: cosmosvaloper1qnk2n4nlkpw9xfqntladh74w6ujtulwnmxnh3k
cat ./data/test-2/config/genesis.json | jq -r '.app_state.genutil.gen_txs[0].body.messages[0].validator_address'

# Submit a staking delegation tx using the interchain account via ibc
icad tx intertx submit \
'{
    "@type":"/cosmos.staking.v1beta1.MsgDelegate",
    "delegator_address":"cosmos1hd0f4u7zgptymmrn55h3hy20jv2u0ctdpq23cpe8m9pas8kzd87smtf8al",
    "validator_address":"cosmosvaloper1qnk2n4nlkpw9xfqntladh74w6ujtulwnmxnh3k",
    "amount": {
        "denom": "stake",
        "amount": "1000"
    }
}' --from $DEMOWALLET_1 --chain-id test-1 --home ./data/test-1 --node tcp://localhost:16657 --keyring-backend test -y

# Wait until the relayer has relayed the packet

# Inspect the staking delegations on the host chain
icad q staking delegations-to cosmosvaloper1qnk2n4nlkpw9xfqntladh74w6ujtulwnmxnh3k --home ./data/test-2 --node tcp://localhost:26657
```

## Collaboration

Please use conventional commits  https://www.conventionalcommits.org/en/v1.0.0/

```
chore(bump): bumping version to 2.0
fix(bug): fixing issue with...
feat(featurex): adding feature...
```
