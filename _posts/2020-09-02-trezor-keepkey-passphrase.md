---
layout: post
title: "A ransom attack on Trezor's and KeepKey's passphrase handling"
---

*tl;dr:*

*The Trezor One before v1.9.3 and the Trezor Model T before v2.3.3 were vulnerable to a remote
ransom attack when entering the passphrase on the computer/phone. The issue affected all
cryptocurrencies on the device. Users are advised to upgrade their devices before receiving any more
coins.*

[Shift Crypto](https://shiftcrypto.ch/) responsibly disclosed the issue described below to
[SatoshiLabs](https://satoshilabs.com/) (Trezor) on April 15th, 2020, and to
[ShapeShift](https://classic.shapeshift.com/) (KeepKey) on May 7th, 2020 to.

A bounty was paid out by SatoshiLabs.

Trezor released a fix in Trezor One v1.9.3 and in Model T v2.3.3 on September 2nd, 2020.

I have been in regular contact with a KeepKey representative. They didn't slate a fix for
development yet, stating that they are working on higher priority items first.

I performed the attack successfully by interactively modifying Electrum, running on Bitcoin Testnet.

# Preface

I am one of the main developers of the [BitBox02](https://shiftcrypto.ch/bitbox02/) hardware
wallet. The BitBox02 supports the use of [optional
passhrases](https://guides.shiftcrypto.ch/bitbox02/advanced/passphrase/) (aka mnemonic passphrase,
aka 25th word [^1]) for expert users. The passphrase is entered on the BitBox02 itself during the
unlock process.

Passphrases can be used to create hidden wallets. They are not stored anywhere on the device and are
not part of the regular backup or recovery seed. Every passphrase is valid and derives a unique
wallet. Losing, forgetting or misremembering a passphrase leads to losing access to the wallet
permanently.

Since entering a long passphrase on the BitBox02 keyboard can be a bit time-consuming, I was
brainstorming about how to allow entering the passphrase from the host wallet instead
(e.g. [BitBoxApp](https://shiftcrypto.ch/app/) or Electrum), in a secure manner. I was thinking that
the passphrase should be sent to the device encrypted after the secure pairing is established, and
that the passphrase would have to be visually confirmed by the user on the device.

As a long time Trezor-owner, I remembered using this feature often with the Trezor, and realized
that I never had to confirm the passphrase on the device itself after entering it in the computer.

The KeepKey was originally derived from Trezor, and behaves the same way in this regard.

# Vulnerability

As a hardware wallet user, you should assume your computer and mobile phone are compromised,
including any wallet software installed. That is the reason to use a hardware wallet in the first
place.

Hence, it is important that the hardware wallet validates any input it receives from the
computer. In this case, the passphrase should be confirmed with the user on the device before using
it to derive the seed. The Trezor and KeepKey did not do this in the case of the passphrase entered
on the computer.

As a consequence, a malicious wallet or a
[man-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) modifying data
transferred via USB could send an arbitrary fake passphrase to the Trezor / KeepKey, and hold any
coins received in this wallet hostage. The passphrase entered by the user could simply be ignored,
and the actual passphrase used would be only known to the attacker.

If that happens, the Trezor and the computer wallet load normally, and the user has no way of
noticing that an attack is ongoing, even if they use the hardware wallet flawlessly, verifying all
receive addresses according to best practices. Receive addresses in the computer wallet match the
address shown on the device as usual, but the addresses do not belong to the user: the attacker can
lock access to them by withholding the passphrase needed to spend from them.

With some sophistication, the user wouldn't notice that anything is off at all until the attacker
blocks access to the coins, demanding a ransom for releasing the coins back to the victim.

Practically, the attacker could run a server from which the malicious wallet would fetch a fake
passphrase every time the user unlocks the wallet, and stop serving the passphrase once there are
enough coins in the wallet to be held to ransom. Without the passphrase, the user has no way of
regaining control of the coins without the attacker's cooperation.

# Impact and severity

Since the attack can be performed remotely, it counts as a critical issue according to [Shift's
security assessment
guideline](https://medium.com/shiftcrypto/how-we-do-security-assessments-at-shift-c85a0c648283).

The attack scales very well: an attacker who has compromised lots of computers can run the attack in
parallel on many victims, and wait to strike until enough coins can collectively be locked and held
hostage.

Since the private keys for all cryptocurrencies (Bitcoin, Litecoin, etc.) are derived from the same
seed and passphrase, coins from all cryptocurrencies can be locked up at once.

The attack is particularly dangerous for newly created wallets, but it is possible to attack
existing wallets as well. For example, the malicious wallet could hold on to the account xpub from
the previous session and use it to show the account as the user would expect it, and some additional
tricks.

An obstacle to performing the attack at scale is that the user needs to have the optional passphrase
feature enabled on the device to be vulnerable.

[^1]: In my opinion, the passphrase should never be called “25th word”, as it is misleading in two
    ways:
    - It can be an arbitrary passphrase with numbers and special characters, it does not have to be
      one word.
    - It could lead users to use a word from the official [mnemonic word
      list](https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt).

    In either case, brute-forcing the passphrase would become trivial.
