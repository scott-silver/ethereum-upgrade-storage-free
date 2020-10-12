# Ethereum proxy beacon upgrade without storage usage

> :warning: This is a proof of concept. It has not been reviewed. **Do not** use in production.

Proof of concept for a delegatecall proxy upgrade pattern for Ethereum smart contracts, that does not require storage usage. Inspired by the [Dharma beacon pattern by @0age](https://github.com/dharma-eng/dharma-smart-wallet/tree/1.5.0/contracts/upgradeability), [CREATE2-based upgrades by @carver](https://medium.com/@jason.carver/defend-against-wild-magic-in-the-next-ethereum-upgrade-b008247839d2), [storage hacks by @Agusx1211](https://medium.com/@agusx1211/evm-istambul-storage-pricing-5befaac32403), and [blue-green deployments by Martin Fowler](https://martinfowler.com/bliki/BlueGreenDeployment.html).

## Motivation

When using proxies for managing upgradeability, it's in our users best interst to introduce the minimum amount of gas overhead per call. Most delegatecall-based proxies rely on keeping in storage their implementation address, or call to a static beacon that in turn keeps the implementation address in its own storage. Other proxy implementations, such as the transparent proxies, even requires two storage loads instead of one to guard against storage collisions.

This proof of concept is an attempt to have an upgrade pattern that uses no SLOADS at all, since it is one of the most expensive operations, and has already been [repriced once from 200 to 800 gas](https://eips.ethereum.org/EIPS/eip-1884), and [may be repriced again to 2100](https://eips.ethereum.org/EIPS/eip-2929).

## How it works

The pattern is similar to Dharma's beacon proxy. Each proxy has an `immutable` address for a beacon it follows. However, instead of `CALL`ing into the beacon to `SLOAD` the implementation address, it uses `EXTCODECOPY` to extract the address from the code, where it's stored as immutable as well. Every call to the proxy then requires just one `EXTCODECOPY`, plus the `DELEGATECALL` to the implementation, with no accesses to storage.

Upgrades are the interesting bit. Since the beacon has the implementation address as part of its runtime code, we need to destroy the beacon and recreate it with the new implementation address to perform an upgrade. This has the benefit of upgrading multiple proxies in a single tx, like Dharma's beacons. We use CREATE2 with a static init bytecode to ensure the beacon is always deployed to the same address, already hardcoded into the proxy. In this init bytecode, the beacon queries the controller contract, which keeps a dynamic mapping from contract name to implementation.

This has a problem though. Due to limitations of the EVM, destroying and recreating a contract cannot be done on the same tx. This means that there's a period during which any calls to the proxy would fail, since it cannot retrieve the implementation address from the destroyed beacon. 

To fix this, we take a page from blue-green deployments. The proxy _relies on two beacons instead of one_, using one as the main and another as backup. If the main beacon is down, it queries the backup, ensuring it will always be able to retrieve an implementation address.

Upgrades are then a two-tx process. The first tx updates the implementation address in the Controller's registry, deploys a backup beacon pointing to it, and destroys the main beacon. By this point, proxies are already upgraded. However, to prevent proxies from having to run two `EXTCODECOPY`s (since the first one fails) per call, we run a second tx that recreates the main beacon, and destroys the backup one in preparation for the next upgrade.

Note that, like Dharma's beacons, this process can update N proxies in a single operation, instead of requiring one transaction per proxy as other patterns.

## Gas cost per call

The scripts in `scripts/compare` run the same `Counter#increase()` example call through [OpenZeppelin Transparent proxies](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/proxy), [Dharma Beacon proxies](https://github.com/dharma-eng/dharma-smart-wallet/tree/master/contracts/upgradeability), and [EIP-1882 UUPS proxies](https://eips.ethereum.org/EIPS/eip-1822), and report the gas overhead of the call.

- OpenZeppelin's proxies require 2 `SLOAD`s per call (one for comparing the sender to the admin in order to prevent selector collisions, and one to load the implementation) as explained [here](https://blog.openzeppelin.com/the-transparent-proxy-pattern/).
- Dharma's proxies take 1 `CALL` (to call from the proxy to the beacon) and 1 `SLOAD` (to load the implementation at the beacon).
- EIP-1882's proxies only require a single `SLOAD` (to load the implementation from storage).
- Proxies in this proof-of-concept require a single `EXTCODECOPY` (to copy the implementation from the beacon's code).

The cost for calling `increase` on a non-zero `Counter` contract directly is `27045`. Following is the gas cost and gas overhead introduced per call for each standard (less is better).

| Proxy | Gas cost | Gas overhead  |
|-|-|-|
| OpenZeppelin Transparent | 29815 | 2770 |
| Dharma Beacon  | 29752  | 2707 |
| EIP-1882 UUPS  | 28679 | 1634 |
| Storageless Beacon  | 28629 | 1584 |

## Usage

Don't. Seriously, don't use this. This is a proof of concept, I thought it was clear from the beginning!