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

You could register a backup token 'just in case'. But you rarely use that one, so it's likely also hidden somewhere...

There is some account recovery process for the website in question. This could be something really difficult (e.g. tied to your phone number, but what if your phone number changes) up to some very easy questions (What was the city you grew up in?) which an attacker could overcome by social engineering. Also, very often these questions are ambigious (what if you grew up in two different cities), what was the spelling you used for these cities when registering the account, etc.

Another approach is to generate a secret `master key` off-card, and flash it on the token. You can then write that master-key down on a piece of paper and store it somewhere safely, or use some other form of electronic cold-storage. You could also generate the master-key from the hash of secret sentence or a bunch of key-words, as done by some bit-coin wallets.

As far as I know, most U2F tokens however use on-card key generation. This applet is different, here you must generate a 256 bit AES key off-card, and supply it to the card during installation. This way you can generate infinitely many tokens. In case your token gets lost or stolen, you can generate a second token with the key, and, in case of a stolen token, transfer all your accounts safely to a token with a new master key.

2. Side Channel Attacks

There are different ways of implementing a U2F token. Let's call them the good, the bad, and the ugly. First recall how a U2F token works:




3. The attestation key & certificate

# Building 

# Testing

# License

This application is licensed under [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)
