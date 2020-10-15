---
layout: post
title: "Coldcard isolation bypass"
---

[Shift Crypto](https://shiftcrypto.ch/) responsibly disclosed the remote exploit to
[Coinkite](https://coinkite.com/) (Coldcard) on August 4th, 2020, and mutually agreed to a 90-day
disclosure embargo period. Coinkite [created a fix on September
30th](https://github.com/Coldcard/firmware/pull/49/files) but, as of this date, have not yet
released a firmware update to mitigate against potential exploits in the wild. The fix on September
30th explicitly described the issue and attack scenarios in the changelog, and the 90-day embargo
period, including requested extensions, has lapsed. Therefore, we now disclose the issue, and we
encourage Coldcard users to take appropriate precautions until an update is available.

# The original issue

On August 4th 2020, [@mo_nokh](https://twitter.com/mo_nokh/) disclosed a
[vulnerability](https://monokh.com/posts/ledger-app-isolation-bypass) in the Ledger hardware wallets
in which a user could unknowingly confirm a Bitcoin transaction that was masquerading as an altcoin
or [testnet](https://en.bitcoin.it/wiki/Testnet) transaction.

> An attacker can exploit this method to transfer Bitcoin while the user is under the impression
> that a transaction of another, less valuable altcoin (e.g. Litecoin, Testnet Bitcoins, Bitcoin
> Cash, etc.) is being executed

A quick high-level summary of how this is possible:

When you create a transaction proposal on your computer, the transaction is sent to the hardware
wallet to be confirmed by the user and formally signed. The computer also tells the hardware wallet
what kind of coin we are dealing with, e.g. “this is a Litecoin transaction, please show the amounts
as Litecoin and display the addresses in Litecoin’s address formats”. The hardware wallet will just
take that info, and show, for example “Sending 1 LTC to ltc1...”.

Litecoin, Bitcoin, their testnets and some other coins all have the exact same transaction
representation under the hood. For example, there is nothing in a Bitcoin transaction that says it
is a Bitcoin transaction and not a Litecoin transaction. From a hardware wallet’s point of view, a
valid Litecoin transaction proposal is also a valid Bitcoin transaction proposal.

As a consequence, a compromised wallet on the computer can simply create a transaction spending
bitcoins to the attacker, and send it to the hardware wallet as a Litecoin transaction. The user
will only see Litecoin information on the device (addresses in Litecoin’s address format, amounts
denominated in Litecoin, etc) while not suspecting that they are in fact sending bitcoins.

The problem is mitigated by allocating separate private keys to separate coins, as described by
[BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki#coin-type). If all your
bitcoins use one set of private keys, and all your litecoins use a different set of private keys,
then signing a Litecoin transaction with Litecoin keys can never spend actual bitcoins. The
[BitBox02](https://shiftcrypto.ch/bitbox02/) [has always
enforced](https://twitter.com/ShiftCryptoHQ/status/1291012692606701576) this key separation
strictly.

# Coldcard

When the isolation bypass vulnerability was disclosed, Coldcard responded quickly that they [were not
affected](https://www.reddit.com/r/Bitcoin/comments/i3j5xo/ledger_app_isolation_bypass/g0bqtm8/){:target="_blank"}:

![](/assets/img/cc-reddit.png)

This, unfortunately, was not true. While the Coldcard does not support “shitcoins”, it does support
testnet. A quick test confirmed that the Coldcard was in fact vulnerable in the exact same way as
Ledger. A user confirming a testnet transaction on the device could be spending mainnet (i.e. the
real thing) funds without their knowledge.

For example, this real mainnet transaction:

![](/assets/img/cc-isolation-bypass-1.png)

…is confirmed and signed like this when sending the same real mainnet transaction to the Coldcard
while it is in testnet mode:

![](/assets/img/cc-isolation-bypass-2.png)

# Impact and severity

As a hardware wallet user, you should assume your computer is compromised. That is the reason to use
a hardware wallet in the first place. Starting from there, exploiting this attack is not very far
fetched. The attacker merely has to convince the user to e.g. “try a testnet transaction”, or to buy
an ICO with testnet coins (I’ve heard there was a ICO like this recently) or any number of social
engineering attacks to make the user perform a testnet transaction. After the user confirms a
testnet transaction, the attacker receives mainnet bitcoin in the same amount.

Since the attack can be performed remotely, it counts as a critical issue according to [Shift's
security assessment
guideline](https://shiftcrypto.ch/blog/how-we-do-security-assessments/). Severity points are
deducted as the attack does not scale very well:

- the device has to be unlocked and user interaction is required
- the attacker has to convince the user to make a testnet transaction to the attacker's testnet
  address

# Credit

I did not discover the isolation bypass attack, all credit goes to
[@mo_nokh](https://twitter.com/mo_nokh/), who published the original attack on the Ledger. After
reading about it, I remembered that the Coldcard has testnet support. I quickly built a
proof-of-concept to see if this attack is feasible. Seeing that it indeed is, I immediately
responsibly disclosed the issue to the Coldcard team.

# Disclosure timeline:

- The issue was responsibly disclosed to Coinkite on Aug. 4th, 2020.
- Coinkite acknowledged the issue on Aug. 5th and asked for 90 days to fix the issue.
- Coinkite created a fix on Sept. 30th, publicly disclosing the issue and attack scenario in the
  changelog: https://github.com/Coldcard/firmware/pull/49/files
- Noticing the fix, on Oct. 14th we asked when to expect a firmware release. Coinkite informed that
  additional items are being added in advance of the next firmware release.
- On Nov 12th, one week beyond the embargo period, we asked again when to expect a firmware release
  and gave notice that we would release our disclosure soon. Coinkite informed that the release
  would be further delayed, but imminent, as additional items were being included.
- On Nov 21st, we informed Coinkite that we would publish our disclosure on Nov 24th.
