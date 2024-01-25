# sia-ds

>[WARNING] **Work In progress**: Can lead to data loss!

> A datastore implementation using sharded directories and flat files to store data backed by Sia renterd for backup

`sia-ds` is used by `go-ipfs` to store raw block contents on disk. It supports several sharding functions (prefix, suffix, next-to-last/*).
It is based on the [go-ds-flatfs](https://github.com/ipfs/go-ds-flatfs) implementation with additional functionality to connect with Sia Renterd.
All the blocks are backed up to the renterd and are automatically restored during lookup operations if not found locally. 

It is _not_ a general-purpose datastore and has several important restrictions.
See the restrictions section for details.

## Lead Maintainer

[Mayank Pandey](https://github.com/LexLuthr)
[Alvin Reyes](https://github.com/alvin-reyes)

## Table of Contents

- [Install](#install)
- [Usage](#usage)
- [Contribute](#contribute)
- [License](#license)

## Install

`sia-ds` can be used like any Go module:


```
import "github.com/IPFSR/sia-ds"
```

## Design
* Each PUT operation within IPFS is consistently forwarded to the Renterd node. If the API call to Renterd fails, then operation is discarded in IPFS too. This done was to chose consistency over availability in case of partitioning.
* The DELETE operations are not synced with Renterd by default. This strategy allows the users to restore back any data they might have deleted from IPFS just by trying to access it.
* Since all blocks are unique and this the resulting files names are unique, we don't need to account for object already existing in the bucket. IPFS already does a GET check before doing a PUT operation.
* The default Renterd bucket name is `IPFS` but can be edited to allow for multiple IPFS node connecting to the same Renterd node. Users can choose a different bucket name for each IPFS node.
* All accidental deletes from IPFS or Filesystem should be restored by just accessing the block if the DELETE ops are not being synced. In case, they are being synced, only Filesystem level deletes can be restored.

## Usage

You can run the Sia backed IPFS node by either downloading the customised go-IPFS implementation from [here](https://github.com/IPFSR/kubo) or
you can use this plugin with a vanilla Kubo by replacing the `github.com/ipfs/go-ds-flatfs` with `github.com/IPFS/sia-ds` in file 
`plugin/plugins/flatfs/flatfs.go`

Please make sure to setup your renterd() first. Once it is running, export the following variables to a terminal and initiate a new IPFS node.

| Env Var                   | Default  | Description                                                              |
|---------------------------|----------|--------------------------------------------------------------------------|
| `IPFS_SIA_RENTERD_PASSWORD` |          | Renterd Password                                                         |
|`IPFS_SIA_RENTERD_WORKER_ADDRESS`|          | Renterd worker API address (ex: http://127.0.0.1:9980)                   |
|`IPFS_SIA_RENTERD_BUCKET`| IPFS     | A private bucket with this name will be created and used                 |
|`IPFS_SIA_SYNC_DELETE`| False(0) | If set to True(1), the DELETE operation will be synced to Renterd bucket |


### Instructions
* Set up a renterd node and note down the password and API address.
* Open a new terminal and clone the [IPFSR Kubo (go-IPFS)](https://github.com/IPFSR/kubo)
```shell
git clone https://github.com/IPFSR/kubo
```
* Checkout the `feat/sia-ds` branch and build the binary
```shell
git checkout feat/sia-ds
make build
```
* Export the `IPFS_SIA_RENTERD_PASSWORD` and `IPFS_SIA_RENTERD_WORKER_ADDRESS` environment variables.
* Initiate a new IPFS node
```shell
cmd/ipfs/ipfs init
```
* Verify that a new bucket named "IPFS" was created the Renterd
* Your IPFS node is not connected to the Renterd node.

### Restrictions

FlatFS keys are severely restricted. Only keys that match `/[0-9A-Z+-_=]\+` are
allowed. That is, keys may only contain upper-case alpha-numeric characters,
'-', '+', '_', and '='. This is because values are written directly to the
filesystem without encoding.

Importantly, this means namespaced keys (e.g., /FOO/BAR), are _not_ allowed.
Attempts to write to such keys will result in an error.

### DiskUsage and Accuracy

This datastore implements the [`PersistentDatastore`](https://godoc.org/github.com/ipfs/go-datastore#PersistentDatastore) interface. It offers a `DiskUsage()` method which strives to find a balance between accuracy and performance. This implies:

* The total disk usage of a datastore is calculated when opening the datastore
* The current disk usage is cached frequently in a file in the datastore root (`diskUsage.cache` by default). This file is also
written when the datastore is closed.
* If this file is not present when the datastore is opened:
  * The disk usage will be calculated by walking the datastore's directory tree and estimating the size of each folder.
  * This may be a very slow operation for huge datastores or datastores with slow disks
  * The operation is time-limited (5 minutes by default).
  * Upon timeout, the remaining folders will be assumed to have the average of the previously processed ones.
* After opening, the disk usage is updated in every write/delete operation.

This means that for certain datastores (huge ones, those with very slow disks or special content), the values reported by
`DiskUsage()` might be reduced accuracy and the first startup (without a `diskUsage.cache` file present), might be slow.

If you need increased accuracy or a fast start from the first time, you can manually create or update the
`diskUsage.cache` file.

The file `diskUsage.cache` is a JSON file with two fields `diskUsage` and `accuracy`.  For example the JSON file for a
small repo might be:

```
{"diskUsage":6357,"accuracy":"initial-exact"}
```

`diskUsage` is the calculated disk usage and `accuracy` is a note on the accuracy of the initial calculation.  If the
initial calculation was accurate the file will contain the value `initial-exact`.  If some of the directories have too
many entries and the disk usage for that directory was estimated based on the first 2000 entries, the file will contain
`initial-approximate`.  If the calculation took too long and timed out as indicated above, the file will contain
`initial-timed-out`.

If the initial calculation timed out the JSON file might be:
```
{"diskUsage":7589482442898,"accuracy":"initial-timed-out"}

```

To fix this with a more accurate value you could do (in the datastore root):

    $ du -sb .
    7536515831332    .
    $ echo -n '{"diskUsage":7536515831332,"accuracy":"initial-exact"}' > diskUsage.cache

## Contribute

PRs accepted.

Small note: If editing the README, please conform to the [standard-readme](https://github.com/RichardLitt/standard-readme) specification.

## License

MIT
