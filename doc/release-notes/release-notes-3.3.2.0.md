# Linda 3.3.2.0

This is a new patch version release.

Please report bugs using the issue tracker at github: https://github.com/thelindaproject/linda/issues

## How to Upgrade
Shut down Linda, wait until it has completely shut down (which might take a few minutes for older versions), then just copy over the appropriate Lindad file.

If you are upgrading from 3.1.0.1 or older, the first time you run after the upgrade a re-indexing process will be started that will take anywhere from 30 minutes to several hours, depending on the speed of your machine.

## Downgrading warnings
The 'chainstate' for this release is not always compatible with previous releases, so if you run 3.3.x and then decide to switch back to a 3.2 release you might get a blockchain validation error when starting the old release (due to 'pruned outputs' being omitted from the index of unspent transaction outputs).

Running the old release with the -reindex option will rebuild the chainstate data structures and correct the problem.

Also, the first time you run a 3.2 release on a 3.3.x wallet it will rescan the blockchain for missing spent coins, which will take a long time (tens of minutes on a typical machine).

## Linda-cli
We are moving away from the Lindad executable functioning both as a server and as a RPC client. The RPC client functionality ("tell the running Lindacoin daemon to do THIS") was split into a separate executable, 'Linda-cli'. The RPC client code will eventually be removed from Lindad, but will be kept for backwards compatibility for a release or two.

## About this Release

### What's New

#### Staking
Fixed an issue that prevented attempting to create a stake every second. This will allow nodes to earn stake rewards more often.

#### RPC Commands
- Added pubkey filter to 'masternode status' command to find an active masternode with the given public key
- Fix `masternode status` not showing current masternode when masternodeaddr isn't set in Linda.conf