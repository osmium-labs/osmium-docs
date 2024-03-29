# Osmium masternode setup

# Install a masternode for your coin on Ubuntu Server with the following tutorial.

## Open your Core wallet.

Go to Window -> Console.

## Type the following RPC command, to create an address for the masternode fee:
```
getnewaddress

Example output

SVf2wYD5JeCtURA76bKEL8Tx91zLTJTwfk
```

# Go back to your wallet overview.

Press on the toolbar button "Send".

Enter the address from the RPC command “getnewaddress” behind the text "Pay To:". (Example: SVf2wYD5JeCtURA76bKEL8Tx91zLTJTwfk)

Enter the following amount of coins behind the text "Amount:": 1

Press on the button "Send".

Wait at least one block until the transaction is confirmed.

Go back to the console of your wallet.

# Type the following RPC command, to create an address for the masternode collateral:
```
getnewaddress

Example output

SQKqF8aT2c5s5m9xgfqWgjJLmok7Gh27Vg
```

# Go back to your wallet overview.

Press on the toolbar button "Send".

Enter the address from the RPC command “getnewaddress” behind the text "Pay To:". (Example: SQKqF8aT2c5s5m9xgfqWgjJLmok7Gh27Vg)

Enter the following amount of coins behind the text "Amount:": 500

Make sure the "Subtract fee from amount" checkbox is not active.

Press on the button "Send".

Wait until the transaction is confirmed by 15 blocks.

Go back to the console of your wallet.

# Identify the transaction with the following RPC command:
```
masternode outputs

Example output

{
 "fdab9dff1ff9caf5d291905ad43b9f7d69775189d4d22cb085d7fedd94ea1c6a": "0"
}
```
# Generate a BLS key pair with the the following RPC command:
```
bls generate

Example output

{
 "secret": "64f6f0e27be5d171b23f91803e9ad8fa1a92cb6a1f857eb664d5ab1ac35e514b",
 "public": "0b961a0679d231f2837515b4e7952792fae047143089fc7bc160ec5946496a61485993e358fad5342be32e3ce239fe8f"
 "scheme": "legacy"
}
```
# Type the following RPC command, to create an address for the owner of the masternode:
```
getnewaddress

Example output

Sh5A68rWopdgQGAFbCssMVS5E1fUXUd31p
```
# Type the following RPC command, to create an address for used for proposal voting:
```
getnewaddress

Example output

SWu96a92Kv2w5PbcF793hofuQcPRDssaga
```

# Type the following RPC command, to create an address to receive the masternode reward:
```
getnewaddress

Example output

SY1kWMPDRvAysSwFLpDa2DP3ofnGYSH2v1

Prepare the ProRegTx transaction by modifying the following line.

protx register fdab9dff1ff9caf5d291905ad43b9f7d69775189d4d22cb085d7fedd94ea1c6a 0 1.2.3.4:9969 Sh5A68rWopdgQGAFbCssMVS5E1fUXUd31p 0b961a0679d231f2837515b4e7952792fae047143089fc7bc160ec5946496a61485993e358fad5342be32e3ce239fe8f SWu96a92Kv2w5PbcF793hofuQcPRDssaga 0 SWu96a92Kv2w5PbcF793hofuQcPRDssaga SQKqF8aT2c5s5m9xgfqWgjJLmok7Gh27Vg

fdab9dff1ff9caf5d291905ad43b9f7d69775189d4d22cb085d7fedd94ea1c6a - Transaction id from the RPC command “masternode outputs”.

0 - Single digit from the RPC command “masternode outputs”.

1.2.3.4:9969 - External IPv4 address of your VPS.

Sh5A68rWopdgQGAFbCssMVS5E1fUXUd31p - Address of the owner of the masternode.

064bb1741f4707cfe3629176857c41e0d23cbe751061fe5d0d67b506db10c8f3f6f2b684c3cec8e4a128193a001d12e9 - “public” value from the RPC command “bls generate”.

SWu96a92Kv2w5PbcF793hofuQcPRDssaga - Address used for proposal voting.

SWu96a92Kv2w5PbcF793hofuQcPRDssaga - Address to receive the masternode reward.

SQKqF8aT2c5s5m9xgfqWgjJLmok7Gh27Vg - Address to where you send the masternode amount fee.

Paste the modified line into your console.

Example output

7da2e1187202a1a497beca05e0e53a6e4df0dc06046f72fbf8b61c942db2982a
```

# Update your Ubuntu server with the following command:
```
sudo apt-get update && sudo apt-get upgrade -y
```
# Type the following command to go back to your home directory:
```
cd $HOME
```
# Extract the tar file with the following command:
```
tar -xzvf osmium-daemon-linux.tar.gz
```
# Type the following command to install the daemon and tools for your wallet:
```
sudo mv osmiumd osmium-cli osmium-tx /usr/local/bin/
```
# Create the data directory for your coin with the following command:
```
mkdir $HOME/.osmiumcore
```
# Open nano.
```
nano $HOME/.osmiumcore/osmium.conf -t
```
# Paste the following into nano.
```
rpcuser=rpc_osmium
rpcpassword=some_password
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
listen=1
server=1
daemon=1
maxconnections=125
masternode=1
masternodeblsprivkey=64f6f0e27be5d171b23f91803e9ad8fa1a92cb6a1f857eb664d5ab1ac35e514b
externalip=1.2.3.4
```
1.2.3.4 - External IPv4 address of your VPS.

64f6f0e27be5d171b23f91803e9ad8fa1a92cb6a1f857eb664d5ab1ac35e514b - “secret” value from the RPC command “bls generate”.

Save the file with the keyboard shortcut ctrl + x.

Type the following command to start your masternode:
```
osmiumd
```
