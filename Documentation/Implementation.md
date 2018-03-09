# Implementation notes

## Database structure

## Metadata

Internally, Catena stores metadata in the following tables that are *not* visible to (nor modifyable by) clients:

* __info_: holds information about the current block hash and index. When a block transaction is executed, this contains information on the *last* block processed (i.e. not the block the transaction is part of)
* __blocks_: holds an archive of all blocks in the chain.
* __users_: holds the transaction counter for each transaction invoker public key (SHA-256 hashed)
* _grants_: holds information about database privileges (see 'authentication' below).
* _databases_: holds information about the existing databases and the hashed public keys of their owners

## Authentication

Catena uses Ed25519 key pairs for authentication. A transaction contains an 'invoker' field which holds the public key of the
invoker of the query. The transactions signature needs to validate for the public key of the invoker.

### Keys

The following key types are distinguished:

* *Private key*, formatted using base58check (version=11)
* *Public key*, formatted using base58check (version=88)
* *Public hash*, which s a hash (SHA-256) of the public key, encoded using  Base64.

The public hash is used when rights are granted to a key - using a hash makes it possible to do this without
(yet) revealing the public key itself.

## Proof of Work

A block signature is the SHA-256 hash of the following:

* _block version_ (64-bit, unsigned, little endian)
* _block index_ (64-bit, unsigned, little endian)
* _block nonce_ (64-bit, unsigned, little endian)
* _previous block hash_ (32 bytes)
* _miner public key hash_ (SHA256 hash of public key, 32 bytes)
* _block timestamp_  (64-bit, unsigned, little endian, UNIX-timestamp; omitted for genesis blocks)
* _payload data for signing_ (see below)

The payload data for signing is constructed as follows:

* If the block is a genesis block, the payload data for signing is the seed string encoded as UTF-8 without zero termination.
* If the block is a regular block, the payload data for signing is the concatenation of the transaction signatures

A transaction signature is the Ed25519 signature of the following data, using the private key of the invoker:

* _invoker key_: The public key of the invoker
* _transaction counter_: the transaction counter (64-bit, unsigned, little endian)
* _transaction statement_: the SQL statement for the transaction, encoded as UTF-8.

## Limits

The following limits are currently enforced;

* _Block size_: a block's payload may not be larger than 1 MiB (measured against the data to be signed)
* _Transactions per block_: a block may not contain more than 100 transactions
* _Transaction size_: a transaction may not be larger than 10 KiB (measured against the data to be signed)

## Genesis block

The genesis block (block #0) in Catena is 'special' in the sense that instead of transactions, it contains a
special 'seed string'. The block is deterministically mined (starting from nonce=0) so that a given seed
string and difficulty will lead to a specific genesis block and signature.

## Transactions

### Canonical form

Transactions contain an invoker public key, the database name, a counter and an SQL statement and a
signature. The signature is based on the public key, counter and SQL statement, serialized in a particular
way. Transactions can be stored and transmitted in many different formats (usually JSON). In order to
validate a transaction's signature, the 'data to be signed' is reconstructed from the data, after which the
signature is verified against that data.

Every statement that is accepted as valid by the Catena parser maps to exactly one equivalent valid
statement (the 'canonical form'). Transactions are required to contain only these canonical SQL statements.
The exact formatting rules can be found in the `SQLStandardDialect` class. Among others, the canonical
form follows the following rules:

* SQL keywords are capitalized (`SELECT` instead of `select`)
* Extraneous whitespace is removed (e.g. `SELECT a,b` instead of `SELECT a,  b`)
* There is always whitespace in specific locations, such as between an operator and its operands (e.g. `1 + 1` instead of `1+1`)
* Field and table names are always quoted (`CREATE TABLE "foo" ("x" INT);` instead of `x INT`, `TABLE foo`)
* There are parenthesis around specific parts of an expression (sometimes more than would be required)

### Playback

When a node receives a valid block, it appends it to a queue. When the queue grows beyond a preset size, the oldest block
in the queue is persisted on disk. When the chain needs to be spliced and the splice happens inside the queue, the client can
perform this splice efficiently. If the splice happens in a block older than the oldest block in the queue, the client needs to
replay the full set of transactions from block #0 to the splice point.

All read queries (sent over the Postgres wire protocol) execute in a 'hypothetical' transaction - this is a transaction that is started
before the query and in which the queued block transactions have been executed already. The transaction is automatically
rolled back after the query is finished, such that the changes from the queued blocks are not persisted to disk.

