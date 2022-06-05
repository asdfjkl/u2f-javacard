JavaCard U2F Applet
=================

# Overview

This applet is a Java Card implementation of the [FIDO Alliance U2F standard](https://fidoalliance.org/). It is a fork of the JavaCard applet from [Ledger](https://github.com/LedgerHQ/ledger-u2f-javacard) with the following modifications

 - offcard key generation
 - some countermeasures against electromagnetic side-channel attacks
 - works on JavaCard 3.0.1 up to 3.0.5

# Creating Your Own U2F Token using provided CAP File

 1. First flash the .cap with GlobalPlatformPro and installk parameters
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
2. Side Channel Attacks
3. The attestation key & certificate

# Building 

# Testing

# License

This application is licensed under [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)
