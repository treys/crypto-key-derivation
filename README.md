# crypto-key-derivation
Document all the mess around bip32+bip39+bip44 and similar key generation schemes

# Overview
Our overall goal is to generate bitcoin/litecon/ethereum/etc
private+public keys from a list of english words.

BIP39: Given a list of words, generate a seed (512 bit),
https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki

BIP32: Given a seed (any 128-512 bit will do), generate a master key
which can be used to generate further subkeys.

BIP44: Just a bunch of recommendations on how to use BIP39+BIP32
together.  This is followed by some wallets (e.g. Trezor), while not
followed by others (e.g. Electrum).

Finding out derivation paths and standards:
  - https://docs.trezor.io/trezor-firmware/misc/coins-bip44-paths.html
  - https://github.com/satoshilabs/slips/blob/master/slip-0044.md
  - https://github.com/trezor/trezor-firmware/tree/master/python/src/trezorlib
  - https://github.com/satoshilabs/slips/blob/master/slip-0010/testvectors.py

# Setup

    rm -rf venv
    virtualenv -p python3.9 venv
    . venv/bin/activate
    echo $PWD >venv/lib/python3.9/site-packages/crypto-key-derivation.pth
    pip install -r requirements.txt
    touch venv/lib/python3.9/site-packages/electrum/py.typed

# BIP39
Online tool: https://iancoleman.github.io/bip39/#english

The standard BIP39 seed derivation can be used like this:

    $ ./bip39.py <<EOF
        scissors invite lock maple supreme raw rapid void congress muscle digital elegant little brisk hair mango congress clump
        TREZOR
        EOF
    7b4a10be9d98e6cba265566db7f136718e1398c71cb581e1b2f464cac1ceedf4f3e274dc270003c670ad8d02c4558b2f8e39edea2775c9e232c7cb798b069e88

The string "TREZOR" here is the so called passphrase, which in BIP39
makes it possible for the user to add an additional phrase not in an
english dictionary.  You do not have to use it if you don't want to:

    $ ./bip39.py <<EOF
      scissors invite lock maple supreme raw rapid void congress muscle digital elegant little brisk hair mango congress clump
      EOF
    a555426999448df9022c3afc2ed0e4aebff3a0ac37d8a395f81412e14994efc960ed168d39a80478e0467a3d5cfc134fef2767c1d3a27f18e3afeb11bfc8e6ad

# BIP39, Electrum variant
If you create a new wallet with Electrum, they use a modified version
of BIP39, because they have some reasons how theirs is better...

    $ ./electrum_mnemonic.py <<EOF
        scissors invite lock maple supreme raw rapid void congress muscle digital elegant little brisk hair mango congress clump
        TREZOR
        EOF
    7d8b4005aa5e21e438057535a8a37944f5b110f3df91743bca22ffdcd2690fd1d83611d7740719199fa3e6093a756b2bf6a4e1975da733a114325733b056d86a
    $ ./electrum_mnemonic.py <<EOF
      scissors invite lock maple supreme raw rapid void congress muscle digital elegant little brisk hair mango congress clump
      EOF
    d27376868abe32360d112c0afd31298b0d5a91d895a3edebbccabb96a58161b67a2e7f35902d958bb3fedac0b818c87ce7e97b5309af612ee0a539ed029a107b

# BIP32
Once we have a seed, we can use BIP32 for the derivation of a
hierarchy of keys.

The tree starts with one root, and every node in the tree can have
2^31 non-hardened and 2^31 hardened child.

The generation unfortunately generates invalid children with
probability less than 1/2^127.  We haven't made any effort in this
toolchain to handle or detect these issues...

Every level (including the root and the leafs) of the hierarchy has a
so called extended private key, from which everything below the tree
can be generated.  This is a string starting with "xprv".  From this
we can generate an extended public key (starting with "xpub"), which
can be used to generate the 2^31 non-hardened sub extended public
keys.  Subnodes are indexed from 0 to 2^31-1.

This means that as far as you are using non-hardened derivation paths,
you can generate all the valid public keys from the root without
knowing any of the secrets.  This can be used to gather balances
without having the ability to spend them.

The standard notation of a path in the tree is like this:
"m/44'/0'/0'/1/3".  Here we generating the root from a seed and then
taking the path of: harden 44 -> harden 0 -> harden 0 -> non-harden 1
-> non-harden 3.  Note the apostrophes, when they are present, that's
a hardened derivation in the path.  According to BIP44, this is the
4th change address of the first account in a wallet.

We provide the following utilities to walk the tree.

`seed2xprv.py`: stdin is a hex encoded seed from the BIP39 chapter,
stdout is an xprv, containing the root of the tree.  This path is
usually called "m" or "m/".

`xprv2xpub.py`: stdin is an xprv, output is an xpub.

`xprv2xprv.py`: stdin is an xprv, first arg is a number, output is the derived xprv

`xprv2xprv-hardened.py`: stdin is an xprv, first arg is a number, output is the derived xprv

`xpub2xpub.py`: stdin is an xpub, first arg is a number, output is the derived xpub

`xpub2xpub-hardened.py`: intentionally doesn't exist, that's the point of hardening

`xprv2btc.py`: stdin is an xprv, output is a BTC secret key.

`xpub2btc.py`: stdin is an xpub, output is a BTC address, note that
this address must match the private key generated with
`xprv2btc.py`.  You can use https://www.bitaddress.org/ to check this.

`xprv2eth.py`: stdin is an xprv, output is an ETH secret key.

`xpub2eth.py`: stdin is an xpub, output is an ETH address.  Easiest
way to check if this is working correctly is to login into the secret
key with myetherwallet and check the public key on the screen.

`xprv2xrphex.py`: stdin is an xprv, output is an XRP secret key, usable with http://ripplerm.github.io/ripple-wallet/

`xpub2xrp.py`: stdin is an xpub, output is an XRP address.

`xprv2xlm.py`: stdin is an xprv, give stellar private key AND address. (see later)

Examples:

    $ ./seed2xprv.py <<< 000102030405060708090a0b0c0d0e0f
    xprv9s21ZrQH143K3QTDL4LXw2F7HEK3wJUD2nW2nRk4stbPy6cq3jPPqjiChkVvvNKmPGJxWUtg6LnF5kejMRNNU3TGtRBeJgk33yuGBxrMPHi
    $ ./xprv2xpub.py <<< xprv9s21ZrQH143K3QTDL4LXw2F7HEK3wJUD2nW2nRk4stbPy6cq3jPPqjiChkVvvNKmPGJxWUtg6LnF5kejMRNNU3TGtRBeJgk33yuGBxrMPHi
    xpub661MyMwAqRbcFtXgS5sYJABqqG9YLmC4Q1Rdap9gSE8NqtwybGhePY2gZ29ESFjqJoCu1Rupje8YtGqsefD265TMg7usUDFdp6W1EGMcet8
    $ ./xprv2xprv.py 0 <<< xprv9s21ZrQH143K31xYSDQpPDxsXRTUcvj2iNHm5NUtrGiGG5e2DtALGdso3pGz6ssrdK4PFmM8NSpSBHNqPqm55Qn3LqFtT2emdEXVYsCzC2U
    xprv9vHkqa6EV4sPZHYqZznhT2NPtPCjKuDKGY38FBWLvgaDx45zo9WQRUT3dKYnjwih2yJD9mkrocEZXo1ex8G81dwSM1fwqWpWkeS3v86pgKt
    $ echo "xprv9s21ZrQH143K31xYSDQpPDxsXRTUcvj2iNHm5NUtrGiGG5e2DtALGdso3pGz6ssrdK4PFmM8NSpSBHNqPqm55Qn3LqFtT2emdEXVYsCzC2U" | ./xprv2xprv.py 0 | ./xprv2xpub.py
    xpub69H7F5d8KSRgmmdJg2KhpAK8SR3DjMwAdkxj3ZuxV27CprR9LgpeyGmXUbC6wb7ERfvrnKZjXoUmmDznezpbZb7ap6r1D3tgFxHmwMkQTPH
    $ ./xprv2xprv-hardened.py 2147483647 <<< xprv9vHkqa6EV4sPZHYqZznhT2NPtPCjKuDKGY38FBWLvgaDx45zo9WQRUT3dKYnjwih2yJD9mkrocEZXo1ex8G81dwSM1fwqWpWkeS3v86pgKt
    xprv9wSp6B7kry3Vj9m1zSnLvN3xH8RdsPP1Mh7fAaR7aRLcQMKTR2vidYEeEg2mUCTAwCd6vnxVrcjfy2kRgVsFawNzmjuHc2YmYRmagcEPdU9
    $ ./xpub2xpub.py 1 <<< xpub6ASAVgeehLbnwdqV6UKMHVzgqAG8Gr6riv3Fxxpj8ksbH9ebxaEyBLZ85ySDhKiLDBrQSARLq1uNRts8RuJiHjaDMBU4Zn9h8LZNnBC5y4a
    xpub6DF8uhdarytz3FWdA8TvFSvvAh8dP3283MY7p2V4SeE2wyWmG5mg5EwVvmdMVCQcoNJxGoWaU9DCWh89LojfZ537wTfunKau47EL2dhHKon
    $ ./xprv2btc.py <<< xprv9xAtkRR4Ru4DcgwbM22eoVpBEnJysuy7mxxur8Lrqad6qTnnQHhtR64MvvhXUZYhzUc7FDNwT9xC3ym47vvWB3XEst63pjkPWDRf79a6DTP
    p2pkh:L15hj1TsQC1eCnMwd4CkZTGh8iusyHjDtz2oNL968oPz9dXGEeP4
    $ echo "xprv9xAtkRR4Ru4DcgwbM22eoVpBEnJysuy7mxxur8Lrqad6qTnnQHhtR64MvvhXUZYhzUc7FDNwT9xC3ym47vvWB3XEst63pjkPWDRf79a6DTP" | ./xprv2xpub.py | ./xpub2btc.py
    19C8rUkmD1QG13qrpqypo3pEGuVMfEd8q5

You can also get segwit addresses or P2SH compatible segwit addresses:

    $ echo "xprv9xAtkRR4Ru4DcgwbM22eoVpBEnJysuy7mxxur8Lrqad6qTnnQHhtR64MvvhXUZYhzUc7FDNwT9xC3ym47vvWB3XEst63pjkPWDRf79a6DTP" | ./xprv2xpub.py | ./xpub2btc.py p2wpkh
    bc1qt8wzxfq4p2ufpumd6w02p7kdr5c7uaqeekmeje
    $ echo "xprv9xAtkRR4Ru4DcgwbM22eoVpBEnJysuy7mxxur8Lrqad6qTnnQHhtR64MvvhXUZYhzUc7FDNwT9xC3ym47vvWB3XEst63pjkPWDRf79a6DTP" | ./xpub2btc.py p2wpkh-p2sh
    37GxRVzwaiQ7vdZbVJpmfG64gy8tWWomFz

Note, that for P2SH and legacy the casing of the address is important,
for the new segwit address type everything is lowercase.

# A full example for BTC+Electrum

Given the initial seed of "aerobic melody aerobic join crunch quiz
ring icon brisk speak someone marine", we generate the electrum
specific binary seed, from which we get an xprv and convert it to
xpub.  Once we have the xpub, the only thing we need to know is that
Electrum stores public addresses at "m/0/i" and change addresses at
"m/1/i".

Therefore:

    $ ./electrum_mnemonic.py <<< "aerobic melody aerobic join crunch quiz ring icon brisk speak someone marine" | ./seed2xprv.py | ./xprv2xprv.py 0 | ./xprv2xpub.py | { read xpub ; for i in `seq 0 19` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py ; done ; }
    19C8rUkmD1QG13qrpqypo3pEGuVMfEd8q5
    17p8unm85w7uDVpxhp16y6DKbJJT8S3ZgY
    1JkEikqjLMVpJLBUPwK3HqbTkMYN1tw6jg
    13Vzzos4GGiFuojJfphZnp56Z1wTgf84RB
    1PTp9KgEHM3oqN6jJuFjVgG3Dq4ZiedxNU
    12YA627tRfApBriCtwhA5gDqp4FhNjVTEX
    15xN2PGGhHdf8d186HmT81o87Vuof66Ycj
    12Uhpf3GDSeENinsVwdYZEH11YRSJ1ovcC
    18kBZBYU3rRSgFJSDmRqBR5Qha7eid88BN
    1JvxirnjZrvCeQtZivYisZUDKixuUvdA1L
    1KgGzrtDLyrgQXpymVWoMXovq8kV1A64Ya
    1P8L8eG1LTDUQCAsRR1LWqj2gZKkK1z8WD
    113SW2JqU5xDGVYckTcvWM9pWnfnUp8ehv
    1LQyZWScfG5Y81DwxUgPdJJbpAk3ZWWr1f
    17x4Lmw6yYa7zieziWmjsgLLYTsGwzgyrf
    1DmsZGdvwpBN4HZqATjSxcX9rnpXgVXb49
    1NMFAKSy3zKMA7tb7RicXG8Y1ESN5j67xJ
    126TCaqairxXAY3M1PCD3UokubBh6rN8aN
    1HJpJnHX2q1GZQHPKg1vLG2dkL6CNMr3E9
    15Ve2ATPKfSQrwvzn72yUPunQj3Eao7EyC

And for the change addresses:

    $ ./electrum_mnemonic.py <<< "aerobic melody aerobic join crunch quiz ring icon brisk speak someone marine" | ./seed2xprv.py | ./xprv2xprv.py 1 | ./xprv2xpub.py | { read xpub ; for i in `seq 0 9` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py ; done ; }
    1MTWgVQQLEj8YMckR2p6CuXyc3AviAHkY5
    17TBRJy4NG6sqmfMYnKJ2bb7w7prb2mDAx
    1BTY3J6dqst3zYjQtpty3HGxomUAQ4D2Vv
    1GfzFS1hyeMfch3nFc3ghosYK2MJrcnuJ5
    1JfLH3gU9iTeb19w2P6MbdxPyXePozepNz

Compatiblity with Electrum 2.9.3 has been verified on 2017-10-19.

Compatiblity with Electrum 4.0.9 has been verified on 2021-02-08.

# A full example for BTC+Trezor/LedgerNanoS

Given the following trezor seed: nation grab van ride cloth wash
endless gorilla speed core dry shop raise later wedding sweet minimum
rifle market inside have ill true analyst

You can generate the first 2 addresses of the first legacy, segwit and
native segwit account like this:

    $ echo "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py ; done ; }
    1GhzX8gLBfG96Qg1mk9S5ckhVPsxeeBiSC
    18sRn1xohBhxHH9XZHgFBmZmuH5cUWpWwz
    $ echo "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 49 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py p2wpkh-p2sh ; done ; }
    39Fh4GYAW5QWuT118vYMhfatQtz5UcTaa7
    38xnPRH891kEGkqQKi7yF3uPACGksjp56B
    $ echo "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 84 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py p2wpkh ; done ; }
    bc1q4qfcaq4fzcn3q5tvksds4e00hs45fcu6pznsy8
    bc1qgwtfdq3jzv2324hy7l9575x73sjmgky0ky08fk

Or if you have a secret wallet with the passphrase of "do not
show my wife", then you can get the addresses like this:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py ; done ; }
    1Hvv8XyMtT8QGAXLTRE1bgEYbpfj7bZmN9
    1JiWHCAHmzsfJcag78339H1UNbkgb7cyTz
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 49 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py p2wpkh-p2sh ; done ; }
    3FdXjsy32tZdrma6WvApUvFwFv8oNhPnp1
    3AuVfCnp19R8Eoa5CZXeGJjU1TQDy6fW58
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 84 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xpub2xpub.py $i | ./xpub2btc.py p2wpkh ; done ; }
    bc1q3jl5dkp57fgs3uxcyz56dpdgv7d3pg00jvlxc3
    bc1q7u0hytrgmd6xllkrz0npflcpzukjdktg6zuu6v

Get the private keys for the last 2 output:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 84 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2btc.py p2wpkh ; done ; }
    p2wpkh:Kwo5JqiLtw3ktrzQ3aqqJ3hoVfNcgCJZZ7TCbvrXjhz8tqQNPrQu
    p2wpkh:L1cc4Q1UTmzgcE8p7pgNX4pnekyejbDfXbH1rh3WHTpTdgVbmJmq

4 modes have been checked for compatility with a real Trezor on 2017-10-19:

  - legacy + no passphrase,
  - legacy + passphrase,
  - segwit + no passphrase,
  - segwit + passphrase.

6 modes have been checked for compatility with a real Trezor on 2021-02-08:

  - legacy + no passphrase,
  - legacy + passphrase,
  - segwit + no passphrase,
  - segwit + passphrase,
  - segwit native + no passphrase,
  - segwit native + passphrase,
  - private key matches public key was checked for "segwit native + passphrase".

4 modes have been checked with Ledger Nano S on 2021-02-08:

  - segwit + no passphrase,
  - segwit + passphrase,
  - segwit native + no passphrase,
  - segwit native + passphrase.

# A full example of ETH/ETC+Trezor/LedgerNanoS

First 2 private keys for Ethereum and for Ethereum Classic without passphrase:

    $ ./bip39.py <<< "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 60 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2eth.py ; done ; }
    bb36972e4db24cffd1dba3342c4c801c3344fe429500bdba192e2f49673f9139
    6a72d0345f06270cabe20f8f218f25073108586bc81c985eaa02a27533949a35
    $ ./bip39.py <<< "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 61 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2eth.py ; done ; }
    fc265563c97398fc26f6e47eda38827f865dead4f853f2f571654f4eaf6667f0
    31a6315abcc2fdcb47c14eea6d7c1b357cfdfe17674f34f43af7722d87d2350d

Addresses for these:

    $ ./bip39.py <<< "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 60 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2xpub.py | ./xpub2eth.py ; done ; }
    0x154D15BB73A7c01a208D3b7fEB1d77cd65756f86
    0xf8dFcB912129C22Db155Fb861D6F7A9DdDAEe0Be
    $ ./bip39.py <<< "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 61 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2xpub.py | ./xpub2eth.py ; done ; }
    0x919903583020fb1dc543284bcD75d4737baD415A
    0x3e605cD53A01Cbe4604f4C750855874CC90D6116

First 2 private keys for Ethereum and for Ethereum Classic with passphrase:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 60 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2eth.py ; done ; }
    be080f5d5bd51985b05b001159a1bb5676309006577e0c2711658b96face6c57
    115bf63c5495145d99434db3408cb323664c427c45a0a46b5b5e5a6dadc70ab1
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 61 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2eth.py ; done ; }
    8c7e65a7d43fbfca98ad6127511fd41a4d3ebc6adc976d8239bdc8f18e2b3e7d
    21f8616e3f200fdc7054565a7be929acf91ebd884b7244ea8727a7d9393d4820

Addresses for these:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 60 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2xpub.py | ./xpub2eth.py ; done ; }
    0xc629c12cea4a2Cf61286F46649282f482A872bbd
    0x961b12F54a3132b4EaA6108349FeBC90531B2bbb
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 61 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | { read xpub ; for i in `seq 0 1` ; do echo $xpub | ./xprv2xprv.py $i | ./xprv2xpub.py | ./xpub2eth.py ; done ; }
    0xACAA32F9dD8c6a028A78140bFCe50dCEcAA74554
    0xd01629D284C7d7d7D261d5F847328B23645Fb7E0

Compatibility for all these use cases (ETH/ETC with or without passphrase) has been checked:
  - Trezor 2021-02-08,
  - Ledger Nano S 2021-02-08,
  - myetherwallet.com 2021-02-08.

# XLM (Stellar) + Trezor/LedgerNanoS

Unfortunately XLM (Stellar) uses the BIP32 derivation a bit
differently (because it uses the curve ed25519 instead of secp256k1).

I don't understand the details and for now just hacked around the
secp256k1 lib to somehow provide the result we need for XLM.  So
beware of this part of the project, as it should be cleaned up and
redone properly.  But it works! :)

First keypair without passphrase:

    $ ./bip39.py <<< "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./seed2xprv-ed25519.py | ./xprv2xprv-hardened-ed25519.py 44 | ./xprv2xprv-hardened-ed25519.py 148 | ./xprv2xprv-hardened-ed25519.py 0 | ./xprv2xlm.py
    SCGVFOJNHSOR55IAQQT2R6PFHEHCD3HVTB7PGTC3DNVL74LZQBYUBHAT
    GD23O4PMK22FKSQECOOBOE3WUEPHTB2QKMALHZIADYREY3WGZFUBHNFX

First keypair with passphrase:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv-ed25519.py | ./xprv2xprv-hardened-ed25519.py 44 | ./xprv2xprv-hardened-ed25519.py 148 | ./xprv2xprv-hardened-ed25519.py 0 | ./xprv2xlm.py
    SB5UTCQCF3DO54PRN2TXDV2UGJR6UEBMNGMVFRWXMRM2U6W5WC7FJRNZ
    GB7MHY2KDXUW6MY3PPACEOJOFSZ7EFDSY4L2C3Z4AF5VT7WPJ6R3KO3V

Things to note:
  - these hacky tools only work with xprvs and hardened paths, so no luck for the webshop usecases yet, :(
  - as we only handle xprvs, the final xprv2xlm tool also prints the public key in the second line.

Compatibility has been checked:
  - Trezor One on 2021-02-08,
  - Ledger Nano S on 2021-02-08.

# XRP (Ripple)

Info about XRP key generation in the original XRP wallet: https://xrpl.org/cryptographic-keys.html

Info about address encoding: https://xrpl.org/accounts.html#address-encoding

Trezor One doesn't support XRP.  The Ledger Nano S wallet supports XRP
with derivation path of m/44'/144'/0'/0/0.  But the resulting keypair
at this derivation path is used DIRECTLY, and the picture described in
the XRP documentation
(https://xrpl.org/cryptographic-keys.html#secp256k1-key-derivation) is
not used at all.

This means that XRP is a very weird crypto regarding BIP32: I can
generate you the public key that will be the same as Ledger Nano's
key, but I can't give you a private seed that XRP wallets will accept,
because the XRP wallets want to have a seed and then run the picture
on the seed to get the private key.  But we don't have a seed, we have
the private key itself.  So because of this, there is no `xprv2xrp.py`
support and there never will be.

To interact with the XRP chain without a wallet, somebody would have
to implement a desktop/javascript client that accepts a private key
(in some encoding, e.g. hex), instead of a seed.  And there is an
implementation like that, here: http://ripplerm.github.io/ripple-wallet/

So for XRP you have to use `xprv2xrphex.py` that is compatible with
this tool.

Example without passphrase:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 144 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | ./xprv2xprv.py 0 | ./xprv2xrphex.py
    58fe510ffea22709defc4a2c55d1054d2d37ef46bb4a728f38d6b3fbe112850b
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 144 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | ./xpub2xpub.py 0 | ./xpub2xrp.py
    rH4b7nXtPgfjmtdfVLcY7qvMUFcMwLE5HX

Example with passphrase:

    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 144 | ./xprv2xprv-hardened.py 0 | ./xprv2xprv.py 0 | ./xprv2xprv.py 0 | ./xprv2xrphex.py
    dffefc577f9b97008f60592482d29d8da5a19b202b22f887d1b234df7c2edad2
    $ echo -e "nation grab van ride cloth wash endless gorilla speed core dry shop raise later wedding sweet minimum rifle market inside have ill true analyst\ndo not show my wife" | ./bip39.py | ./seed2xprv.py | ./xprv2xprv-hardened.py 44 | ./xprv2xprv-hardened.py 144 | ./xprv2xprv-hardened.py 0 | ./xprv2xpub.py | ./xpub2xpub.py 0 | ./xpub2xpub.py 0 | ./xpub2xrp.py
    rhbZf6hesPYXX1VgBqCXjK6jvAAbiwL4T6

Compatibility has been checked:
  - Trezor One HAS NO SUPPORT for XRP on 2021-02-08,
  - Ledger Nano S on 2021-02-08,
  - http://ripplerm.github.io/ripple-wallet/ on 2021-02-08.

# Generating the last word in the passphrase.

In bip39 the passphrase checksum should be valid. After choosing the first n-1
words randomly the last can be chosen only from a subset of words. Generate
the list like this:

    $ ./bip39-last-word/last_word.py <<< "scissors invite lock maple supreme raw rapid void congress muscle digital elegant little brisk hair mango congress"
    Good last words:
    alcohol
    another
    ball
    brother
    cherry
    clump
    ...
    A random choice would be:
    photo

For reading words interactively in shuffled order (against keylogger):

    $ ./bip39-last-word/interactive_input.py 23 | ./bip39-last-word/last_word.py
