# Metrix 3.4.0.0

This is a minor version release, bringing both new features and bug fixes.
#### This is a mandatory update. Starting block 990 000 nodes will begin mining version 8 blocks which will be ignored by older nodes. Make sure you update before block 990 000 to avoid any service interruptions.

Please report bugs using the issue tracker at github: https://github.com/thelindaproject/metrix/issues

## How to Upgrade
Shut down Metrix, wait until it has completely shut down (which might take a few minutes
for older versions), then just copy over the appropriate metrixd file.

If you are upgrading from 3.1.0.1 or older, the first time you run after the upgrade
a re-indexing process will be started that will take anywhere from 30 minutes to
several hours, depending on the speed of your machine.

## Downgrading warnings
Release 3.4.0.0 makes use of headers-first synchronization and parallel
block download (see further), the block files and databases are not
backwards-compatible with older versions of Metrix Core:

* Blocks will be stored on disk out of order (in the order they are
received, really), which makes it incompatible with some tools or
other programs. Reindexing using earlier versions will also not work
anymore as a result of this.

* The block index database will now hold headers for which no block is
stored on disk, which earlier versions won't support.
This does not affect wallet forward or backward compatibility.

If you want to be able to downgrade smoothly, make a backup of your entire data
directory. Without this your node will need to start syncing (or importing from
bootstrap.dat) anew afterwards. It is possible that the data from a completely
synchronised 3.4 node may be usable in older versions as-is, but this is not
supported and may break as soon as the older version attempts to reindex.

## metrix-cli
We are moving away from the metrixd executable functioning both as a server and
as a RPC client. The RPC client functionality ("tell the running metrix daemon
to do THIS") was split into a separate executable, 'metrix-cli'.

## About this Release

### What's New

#### BIP65 soft fork to enforce OP_CHECKLOCKTIMEVERIFY opcode
This release includes several changes related to the BIP65 soft fork which redefines the existing OP_NOP2 opcode as OP_CHECKLOCKTIMEVERIFY (CLTV) so that a transaction output can be made unspendable until a specified point in the future.

This release will only relay and mine transactions spending a CLTV output if they comply with the BIP65 rules as provided in code.

This release will produce version 8 blocks by default.

Once 5,701 out of a sequence of 6,001 blocks on the local node's best block chain contain version 8 (or higher) blocks, this release will only accept version 8 blocks if they comply with the BIP65 rules for CLTV.

For more information about the soft-forking change, please see https://github.com/bitcoin/bitcoin/pull/6351

#### Faster synchronization
Metrix Core now uses 'headers-first synchronization'. This means that we first
ask peers for block headers (a total of ? megabytes, as of RELEASE DATE HERE) and
validate those. In a second stage, when the headers have been discovered, we
download the blocks. However, as we already know about the whole chain in
advance, the blocks can be downloaded in parallel from all available peers.

In practice, this means a much faster and more robust synchronization. On
recent hardware with a decent network link, it can be as little as 1 hour
for an initial full synchronization. You may notice a slower progress in the
very first few minutes, when headers are still being fetched and verified, but
it should gain speed afterwards.

A few RPCs were added/updated as a result of this:
- `getblockchaininfo` now returns the number of validated headers in addition to
the number of validated blocks.
- `getpeerinfo` lists both the number of blocks and headers we know we have in
common with each peer. While synchronizing, the heights of the blocks that we
have requested from peers (but haven't received yet) are also listed as
'inflight'.
- A new RPC `getchaintips` lists all known branches of the block chain,
including those we only have headers for.

#### Transaction fee changes
This release automatically estimates how high a transaction fee (or how
high a priority) transactions require to be confirmed quickly. The default
settings will create transactions that confirm quickly; see the new
'txconfirmtarget' setting to control the tradeoff between fees and
confirmation times.
Prior releases used hard-coded fees (and priorities), and would
sometimes create transactions that took a very long time to confirm.
Statistics used to estimate fees and priorities are saved in the
data directory in the `fee_estimates.dat` file just before
program shutdown, and are read in at startup.

#### BIP 66: Strict DER encoding for signatures
Metrix Core 3.40 implements BIP 66, which enforces the already 
adhered to consensus rule, which prohibits non-DER signatures.

This change breaks the dependency on OpenSSL's signature parsing, 
and is required if implementations would want to remove all of 
OpenSSL from the consensus code.

#### New command line options for transaction fee changes:
- `-txconfirmtarget=n` : create transactions that have enough fees (or priority)
so they are likely to begin confirmation within n blocks (default: 1). This setting
is over-ridden by the -paytxfee option.

##### New RPC commands for fee estimation
- `estimatefee nblocks` : Returns approximate fee-per-1,000-bytes needed for
a transaction to begin confirmation within nblocks. Returns -1 if not enough
transactions have been observed to compute a good estimate.
- `estimatepriority nblocks` : Returns approximate priority needed for
a zero-fee transaction to begin confirmation within nblocks. Returns -1 if not
enough free transactions have been observed to compute a good
estimate.

#### New RPC commands for split stake threshold:
- `-stakesplitthreshold=n` : adjust the stake split threshold to not split 
coins when creating a coin stake unless they are above the n threshold. This
setting overwrites the default value of 1 000 000 Metrix.

#### RPC access control changes
Subnet matching for the purpose of access control is now done
by matching the binary network address, instead of with string wildcard matching.
For the user this means that `-rpcallowip` takes a subnet specification, which can be
- a single IP address (e.g. `1.2.3.4` or `fe80::0012:3456:789a:bcde`)
- a network/CIDR (e.g. `1.2.3.0/24` or `fe80::0000/64`)
- a network/netmask (e.g. `1.2.3.4/255.255.255.0` or `fe80::0012:3456:789a:bcde/ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff`)
An arbitrary number of `-rpcallow` arguments can be given. An incoming connection will be accepted if its origin address
matches one of them.
For example:
| 0.9.x and before                           | 0.10.x                                |
|--------------------------------------------|---------------------------------------|
| `-rpcallowip=192.168.1.1`                  | `-rpcallowip=192.168.1.1` (unchanged) |
| `-rpcallowip=192.168.1.*`                  | `-rpcallowip=192.168.1.0/24`          |
| `-rpcallowip=192.168.*`                    | `-rpcallowip=192.168.0.0/16`          |
| `-rpcallowip=*` (dangerous!)               | `-rpcallowip=::/0` (still dangerous!) |
Using wildcards will result in the rule being rejected with the following error in debug.log:
    Error: Invalid -rpcallowip subnet specification: *. Valid are a single IP (e.g. 1.2.3.4),
    a network/netmask (e.g. 1.2.3.4/255.255.255.0) or a network/CIDR (e.g. 1.2.3.4/24).

#### REST interface
A new HTTP API is exposed when running with the `-rest` flag, which allows
unauthenticated access to public node data.
It is served on the same port as RPC, but does not need a password, and uses
plain HTTP instead of JSON-RPC.
Assuming a local RPC server running on port 8332, it is possible to request:
- Blocks: http://localhost:33821/rest/block/*HASH*.*EXT*
- Blocks without transactions: http://localhost:33821/rest/block/notxdetails/*HASH*.*EXT*
- Transactions (requires `-txindex`): http://localhost:33821/rest/tx/*HASH*.*EXT*

In every case, *EXT* can be `bin` (for raw binary data), `hex` (for hex-encoded
binary) or `json`.

For more details, see the `doc/REST-interface.md` document in the repository.

#### RPC Server "Warm-Up" Mode
The RPC server is started earlier now, before most of the expensive
intialisations like loading the block index.  It is available now almost
immediately after starting the process.  However, until all initialisations
are done, it always returns an immediate error with code -28 to all calls.
This new behaviour can be useful for clients to know that a server is already
started and will be available soon (for instance, so that they do not
have to start it themselves).

#### Watch-only support
The wallet can now track transactions to and from wallets for which you know
all addresses (or scripts), even without the private keys.

This can be used to track payments without needing the private keys online on a
possibly vulnerable system. In addition, it can help for (manual) construction
of multisig transactions where you are only one of the signers.

One new RPC, `importaddress`, is added which functions similarly to
`importprivkey`, but instead takes an address or script (in hexadecimal) as
argument.  After using it, outputs credited to this address or script are
considered to be received, and transactions consuming these outputs will be
considered to be sent.

The following RPCs have optional support for watch-only:
`getbalance`, `listreceivedbyaddress`, `listreceivedbyaccount`,
`listtransactions`, `listaccounts`, `listsinceblock`, `gettransaction`. See the
RPC documentation for those methods for more information.

Compared to using `getrawtransaction`, this mechanism does not require
`-txindex`, scales better, integrates better with the wallet, and is compatible
with future block chain pruning functionality. It does mean that all relevant
addresses need to added to the wallet before the payment, though.

#### metrix-tx
It has been observed that many of the RPC functions offered by metrixd are
"pure functions", and operate independently of the metrixd wallet. This
included many of the RPC "raw transaction" API functions, such as
createrawtransaction.
metrix-tx is a newly introduced command line utility designed to enable easy
manipulation of metrix transactions. A summary of its operation may be
obtained via "metrix-tx --help" Transactions may be created or signed in a
manner similar to the RPC raw tx API. Transactions may be updated, deleting
inputs or outputs, or appending new inputs and outputs. Custom scripts may be
easily composed using a simple text notation, borrowed from the metrix test
suite.
This tool may be used for experimenting with new transaction types, signing
multi-party transactions, and many other uses. Long term, the goal is to
deprecate and remove "pure function" RPC API calls, as those do not require a
server round-trip to execute.
Other utilities "metrix-key" and "metrix-script" have been proposed, making
key and script operations easily accessible via command line.

#### Improved signing security
For 3.4 the security of signing against unusual attacks has been
improved by making the signatures constant time and deterministic.
This change is a result of switching signing to use libsecp256k1
instead of OpenSSL. Libsecp256k1 is a cryptographic library
optimized for the curve Metrix uses which was created by Bitcoin
Core developer Pieter Wuille.
There exist attacks[1] against most ECC implementations where an
attacker on shared virtual machine hardware could extract a private
key if they could cause a target to sign using the same key hundreds
of times. While using shared hosts and reusing keys are inadvisable
for other reasons, it's a better practice to avoid the exposure.
OpenSSL has code in their source repository for derandomization
and reduction in timing leaks that we've eagerly wanted to use for a
long time, but this functionality has still not made its
way into a released version of OpenSSL. Libsecp256k1 achieves
significantly stronger protection: As far as we're aware this is
the only deployed implementation of constant time signing for
the curve Metrix uses and we have reason to believe that
libsecp256k1 is better tested and more thoroughly reviewed
than the implementation in OpenSSL.
[1] https://eprint.iacr.org/2014/161.pdf

#### Minimum Masternode online time
Starting 3.4 Masternodes will not be eligible for rewards until they
have been online for at least 24 hours.

#### Stake Modifier V2 soft fork
This release includes an update that defines a new 256-bit modifier for 
the proof of stake protocol, CBlockIndex::nStakeModifierV2. It is computed 
at every block, by taking the hash of the modifier of previous block along 
with the coinstake input. To meet the protocol, the PoS kernel must comprise 
the modifier of the previous block.

Enforcement will be take place once 5,701 out of a sequence of 6,001 blocks 
on the local node's best block chain contain version 8 (or higher) blocks, 
this release will only accept version 8 blocks if they comply with the 
Stake Modifier V2 rules.

#### Multi-level Masternodes
This release includes an update that allows Masternodes to run with collaterals 
of 2m,5m,25m,50m and 100m Metrix .

Enforcement will be take place once 5,701 out of a sequence of 6,001 blocks 
on the local node's best block chain contain version 8 (or higher) blocks.

#### Protocol and network:
- Bump protocol version to 70005
