JavaCard U2F Applet
=================

# Overview

This applet is a Java Card implementation of the [FIDO Alliance U2F standard](https://fidoalliance.org/). It is a fork of the JavaCard applet from [Ledger](https://github.com/LedgerHQ/ledger-u2f-javacard) with the following modifications

 - offcard key generation
 - some countermeasures against electromagnetic side-channel attacks
 - works on JavaCard 3.0.1 up to 3.0.5

# Creating Your Own U2F Token using provided CAP File

 1. First flash the .cap with GlobalPlatformPro and install parameters
 2. Install the attestation certificate

The following install parameters are expected :

 - 1 byte flag : provide 01 to pass the current Fido NFC interoperability tests, or 00
 - 2 bytes length (big endian encoded) : length of the attestation certificate to load, supposed to be using a private key on the P-256 curve
 - 32 bytes : private (EC) key of the attestation certificate
 - 32 bytes : master key (AES-256) <- replace this with your own secret key!!!

Example parameters with 
 - flag set to `01`, 
 - length of certificate is set to `0x0140` bytes 
 - EC key bytes `f3fccc0d00d8031954f90864d43c247f4bf5f0665c6b50cc17749a27d1cf7664`
 - AES master key `00112233445566778899AABBCCDDEEFF000102030405060708090A0B0C0D0E0F`

`java -jar gp.jar --reinstall ..\cap\u2f.cap -params 010140f3fccc0d00d8031954f90864d43c247f4bf5f0665c6b50cc17749a27d1cf766400112233445566778899AABBCCDDEEFF000102030405060708090A0B0C0D0E0F`

The command is included in `install.bat`

Next install the attestation certificate using one `SELECT`and multiple `UPLOAD` apdu's. Simply
 - install `python3` and `pyscard`, e.g. via `pip`
 - run `python3 tools/install_attestation.py`. Each APDU should return `0x90 0x00`

# Purpose of this Fork

1. What if I lose my token?

This is a frequently asked question. If you have 2-factor authentication activated for an account, and you lose your second factor, you are probably in trouble.

You could register a backup token 'just in case'. But since you rarely use that backup token, it might also be stored 'securely' at a forgotten location or it might be broken.

There is probably some account recovery process for the website in question. This could be something really difficult (e.g. tied to your phone number - but what if your phone number changes?) up to some very easy questions (What was the city you grew up in?) which an attacker could overcome by social engineering. Also, very often these questions are ambiguous. For example, what if you grew up in two different cities? Do you remeber what you typed in during registration? Or what if the spelling of that city is ambiguous - can you recall what kind of spelling you used during registration?

The approach taken here is to generate a secret `master key` off-card, and then flash it on the token. The master-key can be written down on a piece of paper and stored somewhere safely. One could also generate the master-key from the hash of a secret sentence or a bunch of key-words, as done e.g. by some bitcoin wallets.

As far as I know, most U2F tokens use on-card key generation. As said, the approach is is different. The master key is a 256 bit AES key which you must generate off-card, and supply it to the card during installation. This way you can generate infinitely many tokens with the same key. In case your token gets lost or stolen, you can generate a second token with the key, and, in case of a stolen token, transfer all your accounts safely to a token with a new master key.

2. Side Channel Attacks

There are different ways of implementing a U2F token. First recall how a U2F token works:

- you generate one elliptic curve (EC) key-pair for each website/login and register your public EC key together with a *key handle* with the website
- during authentication the website sends a challenge as well as an application id (essentially the URL of the website) and the *key handle*
- you sign the challenge with your private EC key. If the signature verifies with your public EC key, you're logged in

Of course there is only limited space available on a U2F token. Thus it is impossible to store infinitely many EC keys on the token. There are essentially two approaches to deal with that:

1. encrypt the private EC with a symmetric cipher using a master key into the *key handle*, and submit that to the site during registration (you let the website store your EC private key for you). During authentication you decrypt the key handle to the your private EC key and sign the challenge
2. you use some key derivation function (for example a keyed hash function) to *derive* the key from the application id, some secret master key and/or a unique nonce generated for each website. If a nonce is used, this can be stored (unencrypted) inside the key handle as well.

The second approach looks somehow 'cleaner', as sensitive data never leave the token. This is the approach that Yubico used to [use](https://www.yubico.com/blog/yubicos-u2f-key-wrapping). The technical problem is however with JavaCard. Remember that a EC private key is a scalar s, and the corresponding EC public key P is then P = s x G. During registration, one derives the scalar s, and then computes s x G. The problam is that JavaCards prior 3.0.5 do not have library call to do so. JavaCards with version 3.0.5 are not available on the free market as of today (Mid 2022). Implementing scalar multiplication manually in Java is too slow. Some JavaCard vendors have implemented proprietary extensions, but to get those linker libraries you have to pay money and sign an NDA - this is practically not working if you wnat a few cards for personal use.

For the first approach you do not only want to encrypt the private EC key, but also to authenticate the cipher as genuine, e.g. by authenticated encryption, like AES in Galois Counter Mode or AES-CCM. There later is what Yubico seems to use [currently](https://developers.yubico.com/U2F/Protocol_details/Key_generation.html). The problem is that such modes are also not available on older JavaCards (support varies from manufacturer to manufacturer).

Another way to do it would be to use encrypt-then-MAC, i.e. encrypt the EC private key, and compute a MAC over the cipher and the application id. A possible way to do it is to use AES in CBC (ECB also works: Note that the EC private key is always chosen at random), and then e.g. CBC-MAC for the message authentication.

The ledge applet does it in a different way. They used AES-CBC with zero IV over the EC private key and the application id. After decryption, the decrypted application id is compared byte-wise with the application id provided by the website. If one would implement this straight away, there would be a problem with privacy. As we use a zero IV, the same plaintext always result in the same ciphertext. If you have different logins on the same website (with same application id), your key-handle would reveal your token id, as some parts of the key handle (the encrypted application id) are the same. To avoid this, they interleave the (randome) EC private key nibble-wise with the application id.

On the one hand I perceive this as cryptographically ugly. On the other hand, only one AES decryption is required for authentication, which makes this small and very universal, as almost every card supports AES in CBC.

From a side channel perspective, the interleave/deinterleave operation is very problematic though. Touching the EC private key nibble by nibble just begs for [emplate attacks](https://eprint.iacr.org/2013/770).

A more general problem is that it is difficult to allow for an infinite AES key usage on a chipcard. If there is even a very small amount of leakage, an infinite AES key usage allows to practically record infinitely many execution traces which allows for attacks. Some vendors have proprietary extensions with additional countermeasures, but these extensions do not follow the JavaCard standard, and you have to sign an NDA to get the libraries.

I have therefore implemented two countermeasures:

- The interleave method is randomized in that access to the 32 bytes of each array occur in random order.
- A few amount of AES dummy rounds (random encryptions) is executed in each registration and authentication step (currently just three dummy rounds for each real encryption. More dummy rounds would be better, but also consume a lot of time.

Both will not be enough to stop a attacker with unlimited ressources, but it will make any side channel attack more difficult in practice. 

The current applet uses booleans for comparisons (e.g. comparing the decrypted application id with the one submitted by the website). This is not good practice; nevertheless fault attacks are difficult to mount in practice and I doubt this is a realistic attack scenario if your token gets stolen and the attacker is not a nation state.

3. The attestation key & certificate

The attestation key is a group key that a manufacturer installs on all or a batch of his sold tokens. When queried with a challenge, the token signs the challenge. A remote party can therefore be sure to talk to a "genuine" token with certain properties, e.g. the remote party knows it is talking to a secure hardware token, or just a software token etc.

If the attestation key is not a group key, the attestation key is however a privacy risk. I.e. if you install your own generated attestation key on your device which noone else uses, it essentially allows to track you over several websites. This is definitely a privacy risk, even though I am not aware of pages that actively exploit such group key issues. In any case, no remote party will be able to verify properties of your token unless you officially certify your token with FIDO and register your public key.

However instead of installing your own attestation key, I highly recommend to just use the test key. It was originally generated by [CCU2F](https://github.com/tsenger/CCU2F). With some luck, there are a bunch of enthusiasts out there who build their own U2F tokens and use that test key, so identification is not possible.

# Building 

- clone this repo
- download and install JavaCard Development Kit Classic libs, e.g. 3.0.3 [here](https://github.com/martinpaljak/oracle_javacard_sdks) (<- should work on 3.0.1 cards up to 3.0.5)
- install Eclipse, preferably an older version like 2020-06 (4.16.0)
- install JDK 1.8 (i.e. Adoptium JDK 8)
- Open Eclipse, File -> Import -> General -> Existing Project into Workspace
- add JavaCard Libs: Right Click on Project -> Properties -> Java Build Path -> Libraries -> add all .jar's from the `lib` folder of the JavaCard Development Kit (folder should contain amongst others `api_classic.jar`)
- adjust the path' in build.xml
- right click on build.xml and Run As -> 1 Ant Build 

The source code should now be compiled to a `.cap`

# Testing

Tested successfully on

- G+D Smartcafe 7.0
- NXP JCOP 3 J3H145, JavaCard 3.0.4
- NXP JCOP 2.1, 2.4.2R3, JavaCard 3.0.1

Cards that did not work 
- ACOSJ 40k (<- gave up debugging for now...)

I bought my cards from [SmartCardFocus](https://www.smartcardfocus.com).

# License

This application is licensed under [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)
