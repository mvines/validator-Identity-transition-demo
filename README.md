# Validator Identity Transition Demo
This document demonstrates how to transition a staked validator from one machine
to another without a restart.  Using this technique, a validator software update
can be accomplished with virtually no downtime.

This method is less sophisticated than the [etcd failover
setup](https://docs.solana.com/running-validator/validator-failover) in that it
does not require you to setup an etcd cluster, and you must manage relocating
the tower file.

# Setup
The demo is run locally on your machine.

## Install Solana 1.9.7 or greater
See https://docs.solana.com/cli/install-solana-cli-tools

## Start a test validator to simulate the overall Solana cluster
A test validator instance is used to simulate the overall Solana cluster. In a
production environment you would be connecting to Mainnet Beta instead.

The `--slots-per-epoch 750` argument is also provided to decrease the epoch
durations from ~3 days to ~5 minutes for quicker testing:

```
$ solana-test-validator --reset --slots-per-epoch 750 --limit-ledger-size 500000000
```

Start it in a separate terminal and wait until the reported "Finalized Slot" is
100 before continuing

## Setup the validator identity, vote account and stake
Establish your validator identity and vote account for the test environment
```
$ solana-keygen new -s --no-bip39-passphrase -o staked-identity.json
$ solana-keygen new -s --no-bip39-passphrase -o authorized-withdrawer.json
$ solana-keygen new -s --no-bip39-passphrase -o vote-account.json
$ solana -ul --keypair staked-identity.json airdrop 42
$ solana -ul create-vote-account vote-account.json staked-identity.json authorized-withdrawer.json
```

then delegate stake to it.  Delegating 200,000 SOL will be ~16% of the stake in
the test validator setup, which is enough to be added to the test-validator
leader schedule:

```
$ solana-keygen new -s --no-bip39-passphrase -o stake-account.json
$ solana -ul create-stake-account stake-account.json 200000
$ solana -ul delegate-stake stake-account.json vote-account.json --force
```

## Setup the primary and secondary validator identities
Create two unstaked identity keypairs for the primary and secondary validator
machines:
```
$ solana-keygen new -s --no-bip39-passphrase -o primary-unstaked-identity.json
$ solana-keygen new -s --no-bip39-passphrase -o secondary-unstaked-identity.json
```

Then create symlinks for both so that the primary validator starts using the
staked identity and the secondary validator starts unstaked:
```
$ ln -sf staked-identity.json primary-identity.json
$ ln -sf secondary-unstaked-identity.json secondary-identity.json
```

## Start the primary and secondary validators
### Primary
```
$ solana-validator --entrypoint 127.0.0.1:1024 \
    --identity primary-identity.json \
    --vote-account vote-account.json \
    --authorized-voter staked-identity.json \
    --ledger primary-ledger --allow-private-addr --rpc-port 9999
```

Once started, from another terminal confirm that the validator is well by running the monitor:
```
$ solana-validator -l primary-ledger monitor
```

### Secondary
This validator instance represents the secondary validator. In a production
environment you'd run the secondary validator on a different physical machine:
```
$ solana-validator --entrypoint 127.0.0.1:1024 \
    --identity secondary-identity.json \
    --vote-account vote-account.json \
    --authorized-voter staked-identity.json \
    --ledger secondary-ledger --allow-private-addr --rpc-port 19999
```

Once started, from another terminal confirm that the validator is well by running the monitor:
```
$ solana-validator -l secondary-ledger monitor
```

## Check the list of validators
Run
```
$ solana -ul validators
```
and you should see two validators. The test-validator representing the overall
cluster and your primary validator.

The secondary validator is not shown in the output because it is not actively
voting.


# Transition Steps
The situation at the start of the transition is:
* The primary validator is using the `staked-identity.json` identity account and
  actively voting with `vote-account.json`
* The secondary validator is using the `secondary-unstaked-identity.json` identity
  account and not actively voting

The situation at the end of the transition will be:
* The primary validator will be using the `primary-unstaked-identity.json` identity
  account and not actively voting
* The secondary validator will be using the `staked-identity.json` identity account and
  actively voting with `vote-account.json`

In production you will likely want to automate all the following steps.

## Step 1. Wait for a restart window
It's undesirable to transition between machines when the validator is about to
produce blocks, as that will likely result in skipped blocks. Two minutes of
idle time should be plenty:
```
$ solana-validator -l primary-ledger wait-for-restart-window --min-idle-time 2 --skip-new-snapshot-check
```

## Step 2. Move your primary validator to the unstaked identity

Direct the primary validator to stop using the staked identity
```
$ solana-validator -l primary-ledger set-identity primary-unstaked-identity.json
```

Rewrite the identity symlink, so that if the primary validator restarts it
will not revert back to using the staked identity
```
$ ln -sf primary-unstaked-identity.jso primary-identity.json
```

Important! Your primary validator is now not voting and the following steps must
be performed as quickly as possible to avoid delinquency.

## Step 3. Transfer your tower file from primary to secondary validator
In this demo environment, a simple `mv` command is sufficient to transfer the
tower file:
```
$ mv primary-ledger/tower-$(solana-keygen pubkey staked-identity.json).bin secondary-ledger/
```

In a production environment, `scp` and `rm` can be used to transfer the tower file from
the primary to secondary machine.

Transferring the tower file is essential to ensure that when your secondary
validator resumes voting it will respect any voting lockouts that your primary
validator was under at the time. Skipping this step may result in being slashed
in the future.

## Step 4. Move your secondary validator to the staked identity

Direct the secondary validator to start using the staked identity. The `--require-tower` flag is used to ensure that the tower file is actually present
```
$ solana-validator -l secondary-ledger set-identity --require-tower staked-identity.json
```

Rewrite the identity symlink, so that if the secondary validator restarts it
will resume using the staked identity
```
$ ln -sf staked-identity.jso secondary-identity.json
```

## Done!
The secondary validator is now voting. Running `solana -ul validators` to
confirm and at this point halting the primary validator will not cause
delinquency.

Upgrade your primary validator, restart it using `--identity
primary-unstaked-identity.json`, and then perform the transition steps in
reverse to move voting and block production back to your primary machine
