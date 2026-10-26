# Masternode and supernode setup

This guide sets up an Osmium **masternode** or **supernode** on an Ubuntu 24.04 server, for Osmium Core
**v1.2.0**. Masternodes and supernodes keep the network running, vote on superblock proposals, and earn
a share of every block.

|                  | Masternode           | Supernode                  |
| ---------------- | -------------------- | -------------------------- |
| Collateral       | 500 OSMI             | 2,000 OSMI                 |
| Paid             | 1 block per turn     | 4 blocks in a row per turn |
| Governance votes | 1                    | 4                          |
| Register with    | `protx register`     | `protx register_super`     |

The return per OSMI is the same for both. The steps are the same too, except for the collateral amount
and the commands in [step 4](#step-4-register-the-node) and [step 6](#step-6-keep-it-running).

**Two roles.** The **owner** holds the collateral in a wallet and registers the node. The **operator**
runs the server. Most people are both; this guide assumes you are. If someone else runs your server, or
you run servers for others, also read [Hosting a node for someone else](#hosting-a-node-for-someone-else).

## What you need

* **Osmium Core v1.2.0** on your computer, fully synced, holding the collateral (500 or 2,000 OSMI) plus
  about 1 OSMI for transaction fees. Download it from the
  [v1.2.0 release](https://github.com/osmium-labs/osmium/releases/tag/v1.2.0).
* **A server** running **Ubuntu 24.04** with at least:
  * 1 CPU core
  * 1 GB of RAM
  * 5 GB of disk for the blockchain
  * a **static public IPv4 address** (IPv6 is not accepted)
  * **port 9969** reachable from the internet (mainnet nodes must use this port)

  Each node needs its own IP address.

The order matters. Set up and sync the server **before** you register the node: a registered node
that does not answer gets penalised by the network and can be banned.

***

## Step 1: Set up the server

Log in to the server with SSH as a normal user who can use `sudo` (not `root`).

### Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Download and verify Osmium Core

```bash
cd ~
wget https://github.com/osmium-labs/osmium/releases/download/v1.2.0/osmiumcore-1.2.0-x86_64-linux-gnu.tar.gz
wget https://github.com/osmium-labs/osmium/releases/download/v1.2.0/checksums.txt
sha512sum --ignore-missing -c checksums.txt
```

The last command must print `osmiumcore-1.2.0-x86_64-linux-gnu.tar.gz: OK`. If it does not, delete the
file and download it again.

{% hint style="info" %}
On an ARM server, use `osmiumcore-1.2.0-aarch64-linux-gnu.tar.gz` instead, here and in the next step.
{% endhint %}

### Install it

The archive unpacks its programs straight into the current folder, so unpack it into a folder of its
own:

```bash
mkdir -p osmium-1.2.0
tar -xzf osmiumcore-1.2.0-x86_64-linux-gnu.tar.gz -C osmium-1.2.0
sudo install -m 0755 osmium-1.2.0/osmiumd osmium-1.2.0/osmium-cli /usr/local/bin/
```

### Open the firewall

Allow SSH first, so you do not lock yourself out, then the Osmium port:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 9969/tcp
sudo ufw enable
```

If your hosting provider has its own firewall, open TCP port 9969 there too.

### Create the configuration

```bash
mkdir -p ~/.osmiumcore
nano ~/.osmiumcore/osmium.conf
```

Paste this, replacing `203.0.113.10` with your server's public IPv4 address:

```ini
server=1
listen=1
externalip=203.0.113.10
```

Save with **Ctrl+O**, **Enter**, then exit with **Ctrl+X**. You will add the masternode key in step 3.

{% hint style="info" %}
You do not need `rpcuser` or `rpcpassword`: `osmium-cli` on the same server logs in automatically.
Do not add `daemon=1`, because the service below runs the node in the foreground. `masternode=1` is
no longer used; the masternode key alone switches masternode mode on.
{% endhint %}

### Run the node as a service

This makes the node start on boot and restart if it stops:

```bash
sudo tee /etc/systemd/system/osmiumd.service > /dev/null <<EOF
[Unit]
Description=Osmium node
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/osmiumd -datadir=$HOME/.osmiumcore
ExecStop=/usr/local/bin/osmium-cli -datadir=$HOME/.osmiumcore stop
Restart=on-failure
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now osmiumd
```

### Let it sync

The node now downloads the blockchain. Check its progress with:

```bash
osmium-cli getblockcount
```

Compare the number with the latest block on the [explorer](https://explorer.osmium.space). Carry on
with step 2 in the meantime.

***

## Step 2: Prepare everything in your wallet

Open Osmium Core on your computer and go to **Window → Console**. Run the commands below in the
console, and keep a note of every result.

### Send the collateral

Create an address for the collateral:

```
getnewaddress
```

Click **Send** in the wallet and send **exactly 500 OSMI** (masternode) or **exactly 2,000 OSMI**
(supernode) to that address, in a single transaction. Make sure **Subtract fee from amount** is
**not** ticked; the collateral must be the exact amount.

### Send the fee money

Create a second address for paying the registration fee:

```
getnewaddress
```

Send **1 OSMI** to it. This keeps the fee away from the collateral.

### Find the collateral

Wait until both transactions are confirmed (a few blocks), then run:

```
masternode outputs
```

Example output:

```
[
  "fdab9dff1ff9caf5d291905ad43b9f7d69775189d4d22cb085d7fedd94ea1c6a-0"
]
```

The part before the dash is the **collateral transaction id**, and the number after it is the
**collateral index**. Here they are `fdab9dff…1c6a` and `0`.

### Create the owner, voting and payout addresses

Run `getnewaddress` three times, once for each:

| Address | What it is for                                                       | Rule                                     |
| ------- | -------------------------------------------------------------------- | ---------------------------------------- |
| Owner   | proves you own the node; used to change it later                     | new and unused; not the collateral address |
| Voting  | votes on superblock proposals                                        | may be the same as the owner address; not the collateral address |
| Payout  | receives the node's rewards                                          | must differ from the owner and voting addresses |

### Create the masternode key

```
bls generate
```

Example output:

```
{
  "secret": "64f6f0e27be5d171b23f91803e9ad8fa1a92cb6a1f857eb664d5ab1ac35e514b",
  "public": "8b961a0679d231f2837515b4e7952792fae047143089fc7bc160ec5946496a61485993e358fad5342be32e3ce239fe8f",
  "scheme": "basic"
}
```

* The **secret** goes on your server (step 3). Keep it private: anyone who has it can operate your node.
* The **public** key goes into the registration (step 4).

***

## Step 3: Turn on masternode mode on the server

Add the secret key to the configuration:

```bash
nano ~/.osmiumcore/osmium.conf
```

Add this line, with your own **secret**:

```ini
masternodeblsprivkey=64f6f0e27be5d171b23f91803e9ad8fa1a92cb6a1f857eb664d5ab1ac35e514b
```

Save, exit, and restart the node:

```bash
sudo systemctl restart osmiumd
```

Before you register, wait until the node has fully synced. This command must show `"IsSynced": true`:

```bash
osmium-cli mnsync status
```

***

## Step 4: Register the node

Back in the wallet's console, fill in this command with your own values. All on one line, in this
order:

**Masternode**

```
protx register COLLATERAL_TXID COLLATERAL_INDEX SERVER_IP:9969 OWNER_ADDRESS BLS_PUBLIC VOTING_ADDRESS 0 PAYOUT_ADDRESS FEE_ADDRESS
```

**Supernode**

```
protx register_super COLLATERAL_TXID COLLATERAL_INDEX SERVER_IP:9969 OWNER_ADDRESS BLS_PUBLIC VOTING_ADDRESS 0 PAYOUT_ADDRESS FEE_ADDRESS
```

| Value              | Where it comes from                                          |
| ------------------ | ------------------------------------------------------------ |
| `COLLATERAL_TXID`  | `masternode outputs`, the part before the dash               |
| `COLLATERAL_INDEX` | `masternode outputs`, the number after the dash              |
| `SERVER_IP:9969`   | your server's public IPv4 address, then `:9969`              |
| `OWNER_ADDRESS`    | the owner address                                            |
| `BLS_PUBLIC`       | `public` from `bls generate`                                 |
| `VOTING_ADDRESS`   | the voting address                                           |
| `0`                | the operator's share of the rewards in percent; `0` when you run the node yourself |
| `PAYOUT_ADDRESS`   | the payout address                                           |
| `FEE_ADDRESS`      | the address you sent 1 OSMI to                               |

If it works, the command prints a long hex string: the **registration transaction id**, also called
the **proTxHash**. Keep it; you need it to check and manage the node.

***

## Step 5: Check that it works

On the **server**:

```bash
osmium-cli masternode status
```

`"state": "READY"` and `"status": "Ready"` mean the node is running as a masternode.

In the **wallet** console, with your proTxHash:

```
protx info PROTXHASH
```

Look at the `state` section:

* `"service"` is your server's address and port.
* `"PoSePenalty": 0` means the network is happy with the node.
* `"PoSeBanHeight": -1` means it is not banned.

The node joins the payment queue once its registration has **15 confirmations**: 15 blocks of about
1.5 minutes each. A
supernode is then paid 4 blocks in a row each time its turn comes.

***

## Step 6: Keep it running

### Update Osmium Core

When a new version is released, repeat the download, verify and install steps from
[step 1](#step-1-set-up-the-server) with the new file names, stopping the node first:

```bash
sudo systemctl stop osmiumd
# download, verify and install the new version
sudo systemctl start osmiumd
```

Your configuration and the blockchain stay where they are.

### If the node gets banned

A node that stays offline or out of date collects penalties and is eventually banned (`PoSeBanHeight`
in `protx info` is then a block number). Fix the server, make sure it has synced, then revive the node
from the wallet console:

**Masternode**

```
protx update_service PROTXHASH SERVER_IP:9969 BLS_SECRET "" FEE_ADDRESS
```

**Supernode**

```
protx update_service_super PROTXHASH SERVER_IP:9969 BLS_SECRET "" FEE_ADDRESS
```

`BLS_SECRET` is the **secret** from `bls generate`. Use the same command to move the node to a new IP
address.

### Stop the node for good

Spending the collateral removes the node from the network. Stop the service on the server with
`sudo systemctl disable --now osmiumd`.

***

## Hosting a node for someone else

When the owner and the operator are different people, the work is split:

* The **owner** keeps the collateral, the owner, voting and payout addresses, and registers the node.
* The **operator** runs the server and keeps the masternode key. The operator never needs the
  collateral or the owner's addresses.

### The operator

1. Set up the server as in [step 1](#step-1-set-up-the-server).
2. Create the masternode key on the server:

   ```bash
   osmium-cli bls generate
   ```

3. Turn on masternode mode with the **secret**, as in [step 3](#step-3-turn-on-masternode-mode-on-the-server).
4. Send the owner the **public** key and the server's `IP:9969`. Never send the secret.

### The owner

1. Do [step 2](#step-2-prepare-everything-in-your-wallet), but skip `bls generate`: use the operator's
   public key instead.
2. Register as in [step 4](#step-4-register-the-node). Put the operator's agreed **share of the rewards
   in percent** where the guide uses `0`; for example `5` gives the operator 5 % of every payment, and
   `7.5` gives 7.5 %.
3. Send the operator the **proTxHash**.

### Getting the operator's share paid

If the operator takes a share, the operator chooses where it is paid by running this once, from a
wallet that holds a little OSMI for the fee:

```
protx update_service PROTXHASH SERVER_IP:9969 BLS_SECRET OPERATOR_PAYOUT_ADDRESS FEE_ADDRESS
```

For a supernode, use `protx update_service_super`. Until this is done, the operator's share stays with
the owner.

### Changing the operator

The owner moves the node to a new operator by setting the new operator's public key:

```
protx update_registrar PROTXHASH NEW_BLS_PUBLIC "" "" FEE_ADDRESS
```

The two empty values keep the current voting and payout addresses. The new operator then needs to
revive the node with `protx update_service` (or `update_service_super`).

### When the collateral is in another wallet

If the collateral sits in a wallet that cannot run `protx register` (for example a hardware wallet), the
owner can register in three parts from Osmium Core:

1. `protx register_prepare` takes the same values as `protx register` and returns `tx`,
   `collateralAddress` and `signMessage`. Use `register_prepare_super` for a supernode.
2. In the wallet that holds the collateral, sign the `signMessage` text with the `collateralAddress`.
3. `protx register_submit TX SIGNATURE` sends the registration.

***

## Common errors

| Error                          | What it means                                                       |
| ------------------------------ | ------------------------------------------------------------------- |
| `bad-protx-ipaddr-port`        | The port is not 9969. Mainnet nodes must use 9969.                  |
| `bad-protx-dup-addr`           | Another node already uses this IP address and port.                 |
| `bad-protx-dup-key`            | The owner address or the masternode key is already used by another node. |
| `bad-protx-collateral`         | The collateral is missing, spent, or not the exact amount: 500 OSMI for `protx register`, 2,000 OSMI for `protx register_super`. |
| `bad-protx-collateral-reuse`   | The owner or voting address is the collateral address. Use new addresses. |
| `bad-protx-collateral-dest`, `bad-protx-collateral-pkh` | The collateral is not on a normal wallet address. Send it to an address from `getnewaddress`. |
| `bad-protx-operator-pubkey`    | The masternode public key is in the old (legacy) format. Create a new one with `bls generate`. |
| `bad-protx-payee-reuse`        | The payout address is the same as the owner or voting address.      |
| `operatorReward must be between 0 and 10000` | The operator's share must be from 0 to 100 percent.   |
| `Invalid masternodeblsprivkey` | The node cannot read the key in `osmium.conf`. Copy the **secret** again. |
