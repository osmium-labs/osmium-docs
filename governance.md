# Governance: proposals, superblocks and voting

Osmium funds work on the network from the network itself. Part of every block reward goes into a
budget, and once a month that budget pays the **proposals** that masternode and supernode owners vote
for. The payment block is called a **superblock**.

This guide explains how it works, how to submit a proposal, and how to vote. It applies to Osmium
Core **v1.2.0**.

## How it works

### The budget

* **5 % of every block reward** goes to the budget.
* A **superblock** comes every **28,800 blocks**: about 30 days, at a block every 1.5 minutes.
  Superblocks fall on block heights that divide exactly by 28,800 (block 864,000 is one).
* The budget for one superblock is 5 % of the block reward, times 28,800 blocks.
* The block reward drops by one-seventh every 172,800 blocks (about every 6 months), so the budget
  drops with it:

  | Superblock at block | Budget        |
  | ------------------- | ------------- |
  | 864,000             | 777.28 OSMI   |
  | 892,800 onwards     | 666.24 OSMI   |

* **Budget that is not used is never created.** It does not roll over to the next month.

To see the live numbers, run this in the wallet's console (**Window → Console**):

```
getgovernanceinfo
```

| Field              | What it is                                                  |
| ------------------ | ----------------------------------------------------------- |
| `nextsuperblock`   | block height of the next superblock                         |
| `governancebudget` | budget of the next superblock, in OSMI                      |
| `fundingthreshold` | net yes votes a proposal needs to be paid                   |
| `proposalfee`      | cost of submitting a proposal, in OSMI                      |
| `superblockcycle`  | blocks between superblocks (28,800)                         |

`getsuperblockbudget HEIGHT` shows the budget of the superblock at any given height.

### Proposals

* **Anyone can submit a proposal.** It costs a **1 OSMI fee**, which is burned: it is not refunded,
  whatever the outcome.
* A proposal asks for an **amount per superblock**, to a **payment address**, between a **start date**
  and an **end date**. It is paid at **every** superblock inside those dates, for as long as it has
  enough votes. A proposal for three months of work therefore asks for one month's amount and spans
  three superblocks.
* A proposal links to a web page (the **URL**) that describes the work in full. The proposal itself
  only holds the short details below.
* After its end date, a proposal expires.

### Votes

* Each **masternode has 1 vote** and each **supernode has 4**. Votes are cast by the node's owner,
  with the node's **voting address**.
* A vote on funding is **yes**, **no** or **abstain**.
* **Net yes votes** are the yes votes minus the no votes.
* A proposal is **passing** when its net yes votes reach the **funding threshold**: one tenth of all
  votes held by working masternodes and supernodes, and never fewer than 10. A supernode's vote counts
  4 times on both sides.
* Owners can change their vote later, at most once an hour per node. The latest vote counts.
* If two thirds of all votes are **delete** votes, the proposal is removed. This is for spam.

### Who gets paid

About **3 days before each superblock** (the last 2,880 blocks), the masternodes fix the list of
payments:

1. They take every passing proposal whose dates include the superblock.
2. They rank them by net yes votes, highest first.
3. They go down the list, funding each proposal that still fits in what is left of the budget. A
   proposal too big for what is left is skipped, and smaller ones below it can still be funded.

Votes cast after the list is fixed do not count for that superblock, so a proposal needs its votes
**before** those last 3 days.

| When                          | What happens                                          |
| ----------------------------- | ----------------------------------------------------- |
| Any time                      | Submit a proposal (1 OSMI fee)                         |
| Until ~3 days before          | Owners vote; the proposal collects net yes votes       |
| ~3 days before the superblock | The list of payments is fixed                          |
| The superblock                | Funded proposals are paid in that block                |

***

## See the proposals

**In the wallet.** Go to **Settings → Options → Wallet**, tick **Show Governance Tab**, and restart
Osmium Core. The **Governance** tab lists every proposal with its title, dates, amount and status,
which reads **Passing +N** (N votes above the threshold) or **Needs additional N votes**.

**In the console:**

| Command                         | Shows                                                    |
| ------------------------------- | -------------------------------------------------------- |
| `gobject list valid proposals`  | every valid proposal, with its votes                     |
| `gobject get PROPOSAL_HASH`     | one proposal in full                                     |
| `gobject getcurrentvotes PROPOSAL_HASH` | each vote cast on it                             |
| `getgovernanceinfo`             | the next superblock, its budget and the threshold        |

***

## Submit a proposal

You need Osmium Core with at least **1 OSMI plus a little for fees**, and a web page describing your
proposal. All the steps happen in the wallet's console (**Window → Console**).

### 1. Plan the amount and the dates

* Decide the **amount per superblock**. All proposals funded in a month share one budget (see
  `getgovernanceinfo`), so an amount close to the whole budget is unlikely to be paid.
* Decide **which superblocks** should pay you. Find the next one with `getgovernanceinfo`; the
  following ones come every 28,800 blocks, about 30 days apart.
* Pick a **start date** before the first of those superblocks and an **end date** a few days after
  the last one. Nodes only estimate when a superblock will arrive, and the estimate can be off by a
  couple of days, so leave room at the end.

Dates are written as **Unix time**: seconds since 1 January 1970. On Linux or macOS,
`date -d '2026-11-20' +%s` (Linux) or `date -j -f '%Y-%m-%d' '2026-11-20' +%s` (macOS) prints it;
any "epoch converter" website does the same.

### 2. Write the proposal details

The details are a short piece of JSON, written on one line:

```json
{"type":1,"name":"talon-wallet-beta-3","url":"https://example.com/beta-3","payment_address":"SPAYMENTADDRESS","payment_amount":250,"start_epoch":1795000000,"end_epoch":1803000000}
```

| Field             | Rule                                                                     |
| ----------------- | ------------------------------------------------------------------------ |
| `type`            | always `1` (a proposal)                                                  |
| `name`            | up to 40 characters: letters, digits, `-` and `_`, no spaces              |
| `url`             | the page describing your proposal, without spaces                        |
| `payment_address` | a normal Osmium address from `getnewaddress`; script addresses are refused |
| `payment_amount`  | OSMI per superblock, more than 0                                          |
| `start_epoch`     | start date, in Unix time                                                 |
| `end_epoch`       | end date, in Unix time; later than the start and not in the past          |

The whole JSON must be at most 512 characters.

### 3. Turn it into hex

The wallet takes the details as hexadecimal text.

On Linux or macOS:

```bash
echo -n '{"type":1,"name":"talon-wallet-beta-3", ... }' | xxd -p | tr -d '\n'
```

On Windows, in PowerShell:

```powershell
-join ([Text.Encoding]::UTF8.GetBytes('{"type":1,"name":"talon-wallet-beta-3", ... }') | ForEach-Object { $_.ToString('x2') })
```

Copy the result: this is your **PROPOSAL_HEX**.

### 4. Check it

```
gobject check PROPOSAL_HEX
```

It must print `"Object status": "OK"`. Otherwise it names the field to fix.

### 5. Pay the fee

Note the current Unix time; this is your **TIME**. Then run:

```
gobject prepare 0 1 TIME PROPOSAL_HEX
```

This sends the 1 OSMI fee and prints the **fee transaction id**. Keep the exact TIME and PROPOSAL_HEX
you used: the next step must repeat them unchanged.

{% hint style="warning" %}
If you lose track of them, `gobject list-prepared` shows the proposals you have prepared, with their
time, data and fee transaction.
{% endhint %}

### 6. Wait for 6 confirmations

The fee needs **6 confirmations**, about 9 minutes. Check with `gettransaction FEE_TXID`.

### 7. Submit

```
gobject submit 0 1 TIME PROPOSAL_HEX FEE_TXID
```

It prints the **proposal hash**. Share it, with your URL, so owners can find and vote on your
proposal.

***

## Vote on proposals

Votes are signed with a node's **voting address**, so you vote from the wallet that holds it. If you
followed the [node guide](masternode-setup.md), that is the wallet you registered the node from.

**Vote with all your nodes at once.** This uses every node whose voting address is in this wallet:

```
gobject vote-many PROPOSAL_HASH funding yes
```

**Vote with one node:**

```
gobject vote-alias PROPOSAL_HASH funding yes PROTXHASH
```

Use `no` or `abstain` instead of `yes` as you choose. A supernode's vote counts 4 times
automatically.

**Check your votes:**

```
gobject getcurrentvotes PROPOSAL_HASH
```

**Remove spam.** Vote to delete a proposal:

```
gobject vote-many PROPOSAL_HASH delete yes
```

### Let someone else vote for your node

The owner can hand a node's votes to someone else by changing its voting address to one of theirs:

```
protx update_registrar PROTXHASH "" THEIR_VOTING_ADDRESS "" FEE_ADDRESS
```

The two empty values keep the node's current masternode key and payout address. Run the same command
with your own voting address to take the votes back.

***

## Commands at a glance

| Command                                         | Does                                              |
| ----------------------------------------------- | ------------------------------------------------- |
| `getgovernanceinfo`                             | next superblock, budget, threshold, fee           |
| `getsuperblockbudget HEIGHT`                    | budget of the superblock at a height              |
| `gobject list valid proposals`                  | lists proposals                                   |
| `gobject get PROPOSAL_HASH`                     | shows one proposal                                |
| `gobject check PROPOSAL_HEX`                    | checks proposal details before paying the fee     |
| `gobject prepare 0 1 TIME PROPOSAL_HEX`         | pays the 1 OSMI fee                               |
| `gobject list-prepared`                         | shows proposals you have prepared                 |
| `gobject submit 0 1 TIME PROPOSAL_HEX FEE_TXID` | submits the proposal                              |
| `gobject vote-many PROPOSAL_HASH funding yes`   | votes with all your nodes                         |
| `gobject vote-alias PROPOSAL_HASH funding yes PROTXHASH` | votes with one node                      |
| `gobject getcurrentvotes PROPOSAL_HASH`         | shows the votes cast                              |
