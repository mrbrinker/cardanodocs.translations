---
layout: default
title: HD wallets
permalink: /technical/hd-wallets/
group: technical
visible: true
---
[//]: # (Reviewed at <no commit>)

# HD wallets

<!-- @martoon TODO: perhaps this part better fits to protocols section -->

HD wallets is feature which allows user to derive keys in deterministic way from common seed.
Basically, you generate initial secret key `SK₀` out of random seed. Then you can derive children `SK₀-₀`, `SK₀-₁`
out of `SK₀`. Then `SK₀-₀-₀`, `SK₀-₀-₁`, `SK₀-₁-₀` and so on (derivations for a tree of arbitrary depth).

<!-- Advertisement: for subscripts symbols: https://en.wikipedia.org/wiki/Unicode_subscripts_and_superscripts -->

We distinguish two types of keys:

* **Hardened**
* **Non-hardened**

Only distinction here is that **hardened** keys allow only secret key out of secret key generation,
while **non-hardened** allow one to derive child public key out of parent public key (not having secret key available).
Thus, for hardened keys to compute child key, you have to own private key.

Each child is assigned a 4-byte index `i`

* `i > 2³¹ - 1` for **non-hardened**
* `i <= 2³¹ - 1` for **hardened**

## Requirements

Let `A(K)` denote address, which holds information about keypair `K`.
Let `child(K, i)` denote `i`-th child keypair of `K`.
Let `tree(K)` denote tree of addresses for keypairs, derived from `K` (all which have positive balance) held in **utxo**.

`a -> b` denotes `b` is derivable form `a`.
`a -x b` denotes `b` is not derrivable from `a` (under no circumstances)

     priv(K) -> pub(K)
     pub(K) -> A(K)
     pub(K) -x priv(K)


For **Hardened** keys:

    A(K) -x pub(K)
    A(K) -x A(child(K, i))
    pub(K) -x pub(child(K, i))
    priv(K) -> utxo -> tree(K)
    priv(K) -> priv(child(K, i))


For **Non-hardened** keys

    A(K) -x pub(K)
    pub(K) -> utxo -> tree(K)
    pub(K) -> pub(child(K, i))
    priv(K) -> priv(child(K, i))

## Properties:

1. Tree structure is held in root address. User needs to copy his public key and pass it to everyone he wants to be able to restore tree.

## Address format

We use `PubKey` address (already present in system) and add attributes field.
In attribute indexed by `0` (**HD wallets attribute**) we store tree data.

Tree is stored as list of **derivation paths**.
Each **derivaion path** is specified as list of **derivation indices**.
Each **derivation index** is 4-byte unsigned int.
<!-- TODO: refer to binary spec section? -->

Resulting object is serialized and encrypted with symmetric scheme (*ChaChaPoly1305* algorithm) with passphrase computed as SHA-512 hash of root public key. This won’t allow adversary to map all addresses on chain to their root as long as we don’t actually store any funds on root key (which isn't forced by consensus rules, rather by UI).

**Crucial point in design:** root public key isn't used to actually store money.

## Usecases

#### Financial audit

One should provide auditor hash of root public key to let auditor find all keys in hierarchy.

#### Payment server
(applicable for **non-hardened** only)

For server to be able to derive subsequent addresses to receive payment to them, one needs to upload there either:

* Root public key

* Payload of:

  * Public key `PK` of level `i`

  * Hash of root public key

  * Tree path for `PK`

#### Wallet

For wallet to operate over some subtree one needs to provide either:

* Root secret key

* Payload of:

  * Secret key `SK` of level `i`

  * Hash of root public key

  * Tree path for `SK`

## Derivation crypto interface

#### Notation:

* `kp` denotes private key with index `p`.

  Just a **Ed25519** private key

* `Kp`  denotes public key with index `p`.

  Just a Ed25519 public key

* `cp`  denotes chain code with index `p`.

###### Entropy

In BTC they use 512-bit hash, but `kp` is only 256 bit. For this reason we need to comply key to 512 bit for not to reduce hashing space.

* Extended private key is a pair denoted as `(ki, ci)`

* Extended public key is a pair denoted as `(Ki, ci)`.

<!-- @martoon TODO: looks like we actually don't use extended keys :/ -->

From application perspective HD wallets (as for BIP-32) introduce following crypto primitives:

* `CKDpriv :: ((kpar, cpar), i) → (ki, ci)`

  Computes a child extended private key from the parent extended private key.

* `CKDpub :: ((Kpar, cpar), i) → (Ki, ci)`

  Сomputes a child extended public key from the parent extended public key.


# Daedalus HD wallets

This section describes the way HD wallets feature is actually used, it's splitted up
to two parts:

1. Extension of wallet backend API to support HD wallet structure locally (as it is done in Bitcoin)

2. Extension to blockchain handling to utilize new address attribute to keep HD structure on several client instances in sync.

## Local storage

### Old storage

Old wallet storage stored list of addresses. Each address was associated a name and was derived from separate secret key (backed up by mnemonics and encrypted with spending password).

### New storage

Wallet storage is extended to store list of **wallet sets**.
Each wallet set corresponds to single root secret key (backed up by mnemonics and encrypted with spending password).

Each wallet set contains a number of **wallets**.

Each wallet contains a number of **addresses** (i.e. address is key of 2nd level in HD tree).

This maps to HD tree:

* Wallet set corresponds to key of 0-th level (*root*)

* Wallet corresponds to key of 1-th level (children of root)

* Address corresponds to key of 2-th level (grandchildren of root)

Money are kept only on addresses.

When money are being spent from one or several addresses, new one is to generated for money remainder, if any.

### Usability

User is able to:

* import/export arbitrary amount of **wallet sets**

* generate arbitrary amount of **wallets**

* assign name to **wallet sets** and **wallets**

* generate arbitrary amount of addresses

* change **wallet set** spending password. Since key derivation depends on spending password, this action changes all account addresses (transfering money from old accounts to newly created ones in process).

## Read HD wallet data from blockchain

There are two ways of importing/exporting wallet set:

* Via **mnemonics**

  **Mnemonics** is generated on frontend side and allows to deterministically generate secret key. Names won't be restored.

* Via export file

  This file allows to restore whole **wallet set** structure.

#### Import

In both cases we have secret root key. Following procedure should be applied for import:

* Root key is checked to be absent in local storage

* **utxo** is traversed to find all addresses with positive balance corresponding to this root key and add them to storage along with their parents (wallets)

* In case of file import, structure resulted from step 2 is labeled with names. Also wallets/addresses which existed in file and were not spent are created.

#### New transaction handling

When new transaction gets available (appears either in block or in mempool), inputs are analyzed. If input corresponds to public key address with **HD wallet attribute**, it is checked whether this address corresponds to one of our *wallet set* s and if yes, address is imported to structure (to show balance in user interface).