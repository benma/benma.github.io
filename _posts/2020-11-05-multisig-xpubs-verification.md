---
layout: post
title: "How nearly all personal hardware wallet multisig setups are insecure"
---

The thesis of this post is this:

*If you use hardware wallets in a Bitcoin multisig setup, using a single computer to handle them, you are
likely to be exposed to remote theft or ransom attacks.*

Multisig using multiple hardware wallets is often used as a security upgrade for personal funds
previously held in a single-signature wallet. In reality, it often achieves the opposite
when it comes to remote attacks.

Combining hardware wallets from multiple vendors is a popular strategy to minimize the risk of a
single point of failure, e.g. a serious vulnerability in one of involved wallets. Unfortunately, one
weak hardware wallet can compromise the whole setup.

I work on the [BitBox02](https://shiftcrypto.ch/bitbox02/) hardware wallet. Back when we first
evaluated how to properly add multisig support, we stumbled across numerous pitfalls and security
issues related to how people use hardware wallets to secure their coins in a multisig setup. Our
[blog
post](https://shiftcrypto.ch/blog/the-pitfalls-of-multisig-when-using-hardware-wallets/)
summarizing them is one of our most popular posts.

This post is an in-depth look at this small, but hugely important point of the original article:

> Your hardware wallet should verify, or let you verify, the following information provided by the untrusted computer:
> [...]
> * The xpubs of the cosigners in order to prevent an attacker from swapping them

The scope of this post is limited to setups where you use multiple hardware wallets and handle them
using a single computer (or mobile phone).

My standard assumption is that your computer is already compromised. Hardware wallets' main value
proposition is to protect against this. If your computer was safe to use, there would be little
reason to use hardware wallets in the first place.

# The problem

There are plenty of tutorials about how to set up multisig with hardware wallets. Due to the
complicated nature of it, all of them either skip over **xpub verification**, cannot perform it due
to a lack of hardware wallet support, or simply do it incorrectly. Any of those leave the door open
for an attacker to do harm.

A quick recap: an xpub, short for extended public key, is used to derive all the addresses in an
account. In the case of multisig, a threshold and one xpub per cosigner are combined to generate
multisig addresses.

Let's first discuss what can go wrong if the xpubs are not verified properly.

After that, I will show that despite your best efforts, you likely will not have verified the xpubs
properly if you used only one computer.

**Theft attack**

Hardware wallet users are well educated to ultimately trust the device screen. If the device screen
shows something, it must be safe to use.

This does not hold true for multisig receive addresses.

A multisig address is derived from the multisig threshold and all cosigner xpubs. One hardware
wallet can only silently verify its own xpub. The other xpubs are provided by the computer wallet,
e.g. Electrum.

A compromised computer wallet can provide fake cosigner xpubs to the hardware wallet. For example,
in a 2-of-3 multisig, the compromised wallet will provide two xpubs controlled by the
attacker. After verifying the compromised receive adddress, any funds sent to it belong to the
attacker.

A common security measure people do is to test the multisig setup first: receive a small amount,
spend it and then, if it all worked, send in the big funds. **Unfortunately, this cannot prevent this
theft attack**. A compromised computer can simulate the whole procedure of receiving and spending
the test amount, including verification on all hardware wallets. More details on this in a future
blog post.

Possible mitigations are:
- Comparing that the same address is shown on *each* cosigner wallet. This is often not feasible,
  e.g. in a 2-of-3 setup, the third device could be in a remote location.
- Proper xpub verification. I will get into why this is nearly impossible in a bit.

**Ransom attack**

In a ransom attack, the attacker does not outright steal the coins, but controls your access to
them. By locking you out of your coins, the attacker can negotiate a payout to release the funds
back to you.

One simple example works exactly like the theft attack described above, but with a 2-of-2 multisig
instead of a 2-of-3 multisig. If one xpub belongs to you, and one xpub belongs to the attacker, it
requires the cooperation of both parties to release any coins sent to this account.

A more subtle attack exploits your **backups**.

> Did you know that in a Bitcoin 2-of-3 multisig setup, you need all three pubkeys to be able to recover your funds, not just two of them?

[https://twitter.com/_benma_/status/1229414020391800833](https://twitter.com/_benma_/status/1229414020391800833)

In order to be able to restore a 2-of-3 multisig wallet, the threshold and **all cosigner xpubs**
need to be known. It is best practice to store all of this information along with each cosigner
backup. Then you can restore a wallet even if one cosigner seed is permanently lost or inaccessible
by using the private keys of the other two cosigners and all three xpubs.

In all multisig tutorials and all multisig setups I have witnessed or reviewed, at best, the xpub
backup was created by copying the xpubs as provided by the computer wallet (e.g. by copying them into
a text file to be printed).

A compromised computer can at this point display fake xpubs. Even if only one xpub is incorrect,
your backup is compromised.

If you ever find yourself in a situation where you need to restore your 2-of-3 multisig setup by
using two out of three cosigner backups, you are in trouble. In theory, the two cosigner seeds and
three xpubs backed up along with them should be sufficient to regain access to your funds. If
however the xpubs in the backup are incorrect, you will not be able to recover. The wallet restored
from this backup would be an empty, wrong wallet. **The attacker is the only one that can help you
recover.**

In order to create valid backups, one must be sure that the xpubs printed on paper match the xpubs
as shown on the hardware wallet screens. Depending on the hardware wallets involved, this is very
hard or impossible to achieve, as we'll see in the next section.

# Obstacles to xpub verification

Table of common obstacles. Each of the entries is explained in more detail below.

|                                               | BitBox02 | Ledger | Trezor | Coldcard |
|-----------------------------------------------|----------|--------|--------|----------|
| Display own xpub on demand                    | ✅       | ❌     | ❌     | ✅       |
| Display cosigner xpubs                        | ✅       | ❌     | ✅     | ✅       |
| Show Electrum `Zpub...` and `Ypub...` formats | ✅       | ❌     | ❌     | ❌       |
| Register xpubs inside the device              | ✅       | ❌     | ❌     | ✅       |

**Display own xpub on demand**

To ensure your backup of the xpubs is correct, you should verify the xpubs in your backup on the
hardware wallet displays (one xpub per device). For example, when using a BitBox02 with Electrum,
you can click the eye icon to verify the xpub on the BitBox02 display:

![BitBox02 xpub verification in Electrum](/assets/img/bitbox02_electrum_xpubs_eye.png)

It is important to verify the xpub displayed on the hardware wallet against the physical printout,
not against what is shown in the computer wallet.

As far as I know, only the BitBox02 and Coldcard provide this function. In the Coldcard, it possible
like this: Go to `Advanced -> View Identity`. Make a note of the master key fingerprint. Then in
`Settings -> Multisig Wallets -> <wallet> -> View Details`, you can find the Coldcard's xpub below
the device fingerprint.

**Display cosigner xpubs**

When verifying a receive address on one device, it can only be trusted if the xpubs that make up the
address also are verified on the device. BitBox02:

![BitBox02 cosigner verification](/assets/img/bitbox02_show_xpub.png)

The Ledger does not currently have the ability to display its own or any cosigner xpub.

Last year I responsibly disclosed the issue of possible theft/ransom attacks due to missing xpub
verification to SatoshiLabs. Following this, the Trezor gained the ability to verify the xpubs by
clicking on `QR` and then on `XPUBs` when verifying a receive address:

![Trezor xpubs verification](/assets/img/trezor_xpubs.gif)

However, it does not indicate which of the xpubs belongs to the Trezor itself. More problematic is
that the xpubs are hidden behind the QR-code, which in my opinion is not discoverable. I fear that
due to this, next to no Trezor user makes use of this important feature.

The Coldcard shows them during the setup of a multisig account.

The big issue in general is that it is not clear what to compare the xpubs against. Let's say you
have a Trezor, a Ledger and a Coldcard in a multisig setup. All three xpubs are shown by the
Trezor. How to verify them?

- You cannot verify them against what the computer wallet shows, as that can be fake (see the
  previous section "Display own xpub on demand").
- You would need to verify them against a trusted physical copy of the xpubs every time you verify a
  receive address. Making such a trusted copy is extremely difficult or impossible, as the xpubs
  cannot be shown at the right time on all hardware wallets.
- Ledger does not show its own xpub nor any cosigner xpub, so the Ledger xpub cannot be verified,
  period.

**Show Electrum `Zpub...` and `Ypub...` formats**

This issue appears when using the popular [Electrum wallet](https://electrum.org) or any compatible
wallet. Electrum encodes xpubs differently depending on the account type. For example, native segwit
multisig account xpubs are formatted as `Zpub...`, while wrapped segwit multisig xpubs are formatted
as `Ypub...`.  See the [full table
here](https://github.com/spesmilo/electrum-docs/blob/8adc2c7df1af23ed240c861a545de2ce9a6e8d68/xpub_version_bytes.rst#specification).

While all the different formats encode the same extended public key, they look completely different. For example:

`Zpub72qW3YoLjossD9Z3CCsoiSYQNAsGHPEFiFBR3V2MPxFJtNgKGpEnYqtXyJ8Jz1ypUrwZQXx9av4w3Wi9cX95ZHmBELsM73g53MN4PUm1MD8`

encodes the same xpub as:

`xpub6CGtJyj4sVEY5z1RapqaUC1bJSY7BnZfZkVJZRy3GAf99zUx16XPDbiL8BFfRmT5S7dy2zkunNyL9msFsuA6pLHJfCN6Xpa6cAxu2VvQuRZ`.

You cannot manually verify that they are the same, it is not only a matter of swapping the four
letter prefix.

If a hardware wallet does not support these formats, verifying xpubs in Electrum is impossible, even
for wallets like Trezor and Coldcard which are able to show xpubs on their screens otherwise. The
Coldcard even exports the xpubs in the right formats (`Ypub...` and `Zpub...`) to the microSD card,
but only shows `xpub...` when verifying them.

One could use an [xpub converter](https://jlopp.github.io/xpub-converter/) to convert `Zpub...` into
an `xpub...` before verification. This is not failsafe of course, as a compromised computer can
modify the result.

To my knowledge, only the BitBox02 is able to display the xpubs in the format required by the
computer wallet:

![BitBox02 Zpub verification](/assets/img/bitbox02_show_xpub.png)

Electrum is probably the most widely used multisig wallet today, so it is important to properly
support these xpubs formats. Some are of the opinion they will be obsolete as
[descriptor-based](https://github.com/bitcoin/bitcoin/blob/42b66a6b814bca130a9ccf0a3f747cf33d628232/doc/descriptors.md)
wallets become adopted. This however can take a very long time, and we should not wait for it.

**Register xpubs inside the device**

Registration of the multisig configuration means that you only have to verify the xpubs once. The
device remembers the verification. After that, you can safely verify receive addresses without
worrying about the cosigner xpubs.

The BitBox02 lets you verify and register the xpubs and give the account a name. When verifying a
receive address, the name is displayed, and you can be sure that the address indeed belongs to the
account you created and verified at the start.

![BitBox02 multisig receive](/assets/img/bitbox02_multisig_receive.jpeg)

The Coldcard works the same way, minus the account name.

The Ledger does not verify nor register any multisig data.

While the Trezor can show the cosigner xpubs when verifying a receive address, it does not register
them. This means that you have to diligently verify all cosigner xpubs against a trusted physical
copy every time you generate a receive address. The previous section goes into the difficulties of
doing that. In particular, verifying the xpubs against what the computer wallet displays is no good,
as those can be fake.

# Conclusion and recommendations

Multisig is hard. Almost one year after the [pitfalls of multisig
article](https://shiftcrypto.ch/blog/the-pitfalls-of-multisig-when-using-hardware-wallets/)
was released, my conclusion is still the same: verification, backup and UX pitfalls might very well
mean that multisig is less secure in practice than single-signature. Still too many footguns exist
in the setup, use and backup/recovery of multisig wallets.

One of these footguns is the lack of cosigner xpub verification. The most pressing issue is the
inability to verify the xpub of a device on demand, which can lead to remote theft or ransom
attacks. I urge all hardware wallet vendors to implement this functionality and to expose it in
multisig wallets like Electrum, in order to reduce the difficulty of properly verifying the xpubs.

If you still want to use multisig with hardware wallets, I recommend using multiple
[BitBox02s](https://shiftcrypto.ch/bitbox02/) and to follow [this
tutorial](https://medium.com/shiftcrypto/bitbox02-multisignature-wallet-5c6023dd10eb) carefully.

If you insist on combining hardware wallets of multiple vendors, I recommend to verify each receive
address on *all* hardware wallets, every time. This rules out Ledger, as the Ledger does not seem to
be able to show a multisig receive address for verification. Furthermore, I recommend to test
backup and recovery on a **separate, independent** computer.
