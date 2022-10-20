---
layout: post
title: "CoinStats users beware - coins sent to your watch-only wallet are not where you think they are"
---

This is a quick write-up about how I helped a [BitBox02](https://shiftcrypto.ch/) hardware wallet
user recover bitcoins they sent to their [CoinStats](https://coinstats.app/) portfolio tracker.

The combination of **three** bugs in CoinStats made a manual recovery of the coins not only
necessary, but also a lot harder than expected.

This post serves as a warning and as a guide. If you imported an xpub/ypub/zpub into CoinStats and
deposited coins to it, you might not be able to spend them without manual intervention. You might have
not become aware of this until you tried spending the coins. This post contains a guide on how to
recover such coins at the bottom.

## What happened

A BitBox02 user imported the **zpub** extended public key of their Bitcoin account into CoinStats, a
portfolio tracker app. The zpub represents all Native Segwit addresses of the form `bc1q...` of the
account and allows watch-only wallets such as CoinStats to display the transactions and balance and
generate new deposit addresses.

The user generated a deposit address and sent some coins to it. When the user connected their
BitBox02 again some time later, the transaction didn't show up in the
[BitBoxApp](https://shiftcrypto.ch/app/) or in [Electrum](https://electrum.org/).

The affected user reached out to CoinStats support. They exchanged close to 100 emails, in which the
user was sent into various wrong directions, such as:

- telling the user to merge the watch-only wallet with the CoinStats hot wallet and then exporting
  the private key - this obviously can't work, unless the watch-only wallet was not derived from the
  imported zpub at all and was fact a hot wallet, which luckily it turned out not to be
- telling the user to reach out to https://cryptoapis.io/, which CoinStats apparently uses to create
  the deposit addresses

Ultimately the support team was not able to tell the user how the deposit addresses were derived
from the zpub and could not resolve the issue. I got involved after the user posted about this
problem in the BitBox community telegram chat. My own inquiry via
[Twitter](https://twitter.com/_benma_/status/1576869542781161480) was left unanswered for multiple
weeks.

## Solution

The coins sent to the watch-only account, which was created from the user's account zpub, ended up
in a legacy Pay-to-PubkeyHash address at the keypath `m/1/465`. If you are familiar with how common
Bitcoin wallets work, you might recognize **three** bugs from this information alone.

## Bug #1

The user reported that they sent the coins to a deposit address that looked like `1Bzut...`. This is
very strange, as that is a legacy Pay-to-PubkeyHash (`p2pkh`) address. A **zpub**, which the user
imported, represents Pay-to-Witness-Pubkeyhash (`p2wpkh`) addresses of the form `bc1q...`.

While `xpub...`  represents a generic extended public key according to
[BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), the `ypub...` and
`zpub...` formats are a [standard defined by
Electrum](https://electrum.readthedocs.io/en/latest/xpub_version_bytes.html). Wallets supporting
thse formats should follow the standard and use the appropriate scripts (e.g. `p2wpkh` when using
`zpub`).

CoinStats should have created `p2wpkh` deposit addresses, but created `p2pkh` addresses instead. As
a result, coins sent to these addresses are not visible in the BitBoxApp, nor in Electrum in the
default configuration, nor in any other compatible wallet.

The BitBox02 follows [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) and
standard keypaths. The standard keypath for the `p2pkh` account is `m/44'/0'/0'`, while the standard
keypath for `p2wpkh` is `m/84'/0'/0'`. The user should be able to find the misplaced coins by
entering their backup words into Electrum, choosing legacy `p2pkh` and then entering `m/84'/0'/0` as
the keypath instead of using the default `m/44'/0'/0'`.

![](/assets/img/coinstats/electrum-p2pkh.jpeg)

The coins were not there however.

## Bug #2

A common [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) wallet creates
deposit addresses using keypaths `m/0/<index>`, e.g `m/0/0` for the first deposit address, `m/0/1`
for the second deposit address, etc.

Such wallets also scan deposit addresses one by one until there are 20 consecutive unused
addresses. 20 is the [standard gap
limit](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki#address-gap-limit) for receive
addresses. Funds received beyond this gap will not be visible by default in most wallets. In
particular, Electrum uses this gap limit, so wallets importing a `zpub` (an Electrum standard)
should use the same gap limit. To stay within the limit, wallets should not create addresses after a
gap of 20 unused addresses.

I noticed that CoinStats generates a new address every time you go to the deposit screen, even if
the previous addresses didn't receive any coins. This was a hint that maybe they were not obeying
the gap limit. I manually forced the gap limit for deposit addresses in the account restored in
Electrum to be 1000 instead of 20, i.e. I let Electrum scan a lot more addresses than normal. The
missing coins however were still not discovered.

Almost ready to give up, I changed the gap limit also for *change addresses*, where normally a lower
gap limit of 6 is used. When increasing the limit to 1000 there as well, the coins were finally
found.

## Bug #3

Common BIP32 wallets and Electrum in particular use the keypaths `m/0/<index>` to generate deposit
addresses, and `m/1/<index>` to generate [internal change
addresses](https://en.bitcoin.it/wiki/Change). See also [BIP44 -
Change](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki#change). Change addresses
should not be provided as deposit addresses.

CoinStats made a deposit address at the keypath `m/1/465`, which is the keypath to a change address,
not a deposit address. As a result, looking for the coins in `m/0/<index>` by increasing the gap
limit for deposit addresses didn't lead to success. CoinStats should instead have created deposit
addresses at keypaths `m/0/<index>` instead.

## Recovering the coins

*Warning: entering backup words in Electrum is dangerous, as it exposes the private keys. If you
can, try connecting the hardware wallet directly instead. If that fails, you are advised to use a
clean environment, e.g. by booting [Tails](https://tails.boum.org/), which comes with Electrum
pre-installed.*

In the end, the user could recover their coins by entering their BIP39 backup words in Electrum,
choosing legacy `p2pkh`, entering `m/84'/0'/0` as the keypath, and then change the gap limit **for
change addresses** to anything higher than 466 to make sure the address at `m/1/465` was covered.

If you imported a `ypub` instead of a `zpub`, the keypath to enter is `m/49'/0'/0'`. For an `xpub`,
you can leave the default keypath, which is `m/44'/0'/0'`. The keypath should match the keypath shown
in the original wallet, e.g. in the BitBoxApp.

To increase the gap limits in Electrum, go to View -> Show Console, and enter for example:

- `wallet.change_gap_limit(1000)` to increase the deposit address gap limit to 1000
- `wallet.gap_limit_for_change = 1000` to increase the change address gap limit to 1000

It is unclear how CoinStats chooses the deposit address keypath when creating an address, and what
the highest index is they would use. If 1000 does not work, maybe a higher number will.

## Conclusion

Getting your funds stuck can be a stressful experience. For a long time, it was not clear if the
coins could ever be recovered.

For wallet application developers, it is important to thoroughly test features before releasing
them. Even basic tests of the CoinStats watch-only deposit feature, such as importing an xpub from a
different wallet, receiving coins and trying to spend them again, would have uncovered the issues
detailed in this post.

This incident highlights the importance of following industry standards and to document any
deviations.
