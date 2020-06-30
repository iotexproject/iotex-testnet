# BlockDAO
BlockDAO manages data access to the blockchain, for example reading a raw data block, querying a transaction by hash, and committing a block to the blockchain

The blockchain data consist of multiple block files (chain.db, chain-00000001.db, etc), the state trie db file (trie.db), and the index db (index.db)

We'll be focusing on block file in the following

## Legacy file (before db split activation at Ithaca height)
Each db file is a key-value db storing raw data in multiple buckets, using *block hash as the key*
1. block header in bucket named "bhr"
2. block body in bucket named "bbd"
3. block footer in bucket named "bfr"
4. block receipt in bucket named "rpt"

In particular, the very first db file (`chain.db`) stores the additional information:
1. the height and hash of the tip block is stored in bucket named "blk"
2. the height/hash mapping of all blocks in bucket named "h2h"
3. transaction log of all blocks in bucket named "syl"

When committing a block, 2 things happen sequentially:
1. block data are written into corresponding db file (like chain-00000001.db)
2. height/hash of the new block, height/hash mapping, and transaction log are written to `chain.db`

## New file (after db split activation at Ithaca height)
The quest of a new file comes from 2 considerations:

First, storing the height/hash mapping of all blocks in the first file (chain.db) has led to dependency of the BlockDAO service on this special file, which is a dependency we want to remove

Second, using hash as primary key and breaking a block into 3 parts has 1) cost 3 DB read to fetch a block, and 2) consumed more storage space. Earlier testing result shows that using block height as (single incrementing) key and storing the whole block can reduce file size by 30% (10GB chain.db reduced to 7GB using new file format)

The new file is still a key-value db, but *using block height as the key*
1. the height/hash mapping, and transaction log are written in the file itself (to remove dependency on `chain.db`)
2. in addition to height/hash of tip block, we also store the height of the starting block in this file. Doing so allows us to quickly know if a block is stored within a particular db file (starting height <= height <= tip height)

With these adjustments, each block db file is autonomous, meaning it can serve block/transaction query in itself, without having to rely on any other special db or index db

Here's the comparison

File | Primary key | Height/hash mapping | Transaction log | Start height
--- | --- | :---: | :---: | :---:
Legacy | block hash | No (except `chain.db') | No (except `chain.db') | No
New | block height | Yes | Yes | Yes

## Starting the chain soley from state DB
Given the chain db size steadily increasing (today about 30GB), it could become curbersome that a node has to carry on all the history files for it to work.
Now with the help of autonomous db file restructure, we want to achieve a more flexible and lightweight configuration:

1. chain db are broken into multiple data files with user-defined segment size (default 4GB)
2. build the intelligence into BlockDAO, such that a node is able to start up and running from any block db file, or even without a block db file (in which case BlockDAO will create a new one and start from there)

Here are 3 examples:

1. node starts from `chain-00000002.db` which is the latest db file, node continues to add blocks, creating new file `chain-00000003.db` once the current file size grows to defined segment size
2. node starts without any chain db file. BlockDAO will create new file `chain.db` and starts adding blocks into it. Once the file size grows to defined segment size, a new file `chain-00000001.db` will be created
3. node starts from `chain-00000002.db` which has height 500. The next block comes in with height = 1000, BlockDAO will create a new file `chain-00000003.db` and starts adding blocks into it. Once the file size grows to defined segment size, a new file `chain-00000004.db` will be created

In example 3 above, blocks 501~1000 are missing, we might consider improving blocksync module's functionality to reconcile this kind of data missing/inconsistency. This could be another topic for enhancement

## Managing legacy and new files in BlockDAO
In order to handle each individual db file we abstract a FileDAO interface, which contains API to read/write block/action in current BlockDAO interface
```go
FileDAO interface {
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Height() (uint64, error)
    GetBlockHash(uint64) (hash.Hash256, error)
    GetBlockHeight(hash.Hash256) (uint64, error)
    GetBlock(hash.Hash256) (*block.Block, error)
    GetBlockByHeight(uint64) (*block.Block, error)
    GetReceipts(uint64) ([]*action.Receipt, error)
    ContainsTransactionLog() bool
    TransactionLogs(uint64) (*iotextypes.TransactionLogs, error)
    PutBlock(context.Context, *block.Block) error
    DeleteTipBlock() error
}
```

and a FileDAONew interface for the new db file 

```go
FileDAONew interface {
    FileDAO
    Bottom() (uint64, error)
    ContainsHeight(uint64) bool
}
```

the fileDAO object to implement the interface

```go
// fileDAO implements FileDAO
fileDAO struct {
    currFd   FileDAO
    legacyFd FileDAO    
    newFd    map[uint64]FileDAONew
}
```

1. legacyFd is the interface that manages **all** legacy files
2. newFd is a map of FileDAONew interface, each of which points to **one** new file
3. currFd points to the newest db file (that commit block should write to)

![BlockDAO](https://github.com/iotexproject/iotex-bootstrap/blob/master/blockdao/blockdao.jpg "BlockDAO example")

