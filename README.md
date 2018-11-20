# jjq
A multi-subscriber queue with an infinite history.

## Front end API interface
- create_queue(name)
- get_queue(name)
- get_queue(subscriber)
- create_subscriber(queue, starting_point)
  - starting point: beginning, end/current, somewhere in between (see kafka docs for how to do this
    well)
  - returns a subscriber id
- pop(subscriber)
  - returns queue element
- peek(subscriber)
  - returns queue element
- push(queue, element) (no subscriber)
  - returns status
- sleep/wakeup(subscriber)
  - unload/load the subscribers data into the hot store
  - optional for now

## Hot Store
Maintains queues and subscribers, manages queue data in memory  and provides an interface to pull
data out of the queues.

### public interface
- create a queue
- list all queues
- create a subscriber
- push
- pop
- peek
- load all elements from a block into the internal database

### relational database structure (normalized)
- subscribers
  - id
  - name
  - queue id
  - current block name
  - current block offset
- queue
  - id
  - name

### fast database structure
- list of all blocks in cold storage
  - list of all blocks in hot storage
  - lists of subscribers in each block, automatically updated when a subscriber moves

- list of queue elements in memory and their associated blocks
  - when the last subscriber leaves a block, the whole block is dropped out of the store
- queue element
  - next element
  - data

### what happens on a pop
- look up the queue position for the subscriber i the relational database
- check if the record for the queue position is loaded
- synchronously load it from cold storage if necessary
- popthe queue element
- check the length of the queue in memory for the subscriber/topic
- if the length is too short, run
  - get the block id for the subscriber/topic out of relational database
  - query the cold side for the next block id
  - update the relational database with the new block id and a negative number for the number of
    things currently in the queue
  - request that the cold side loads the next block id's data into the in memory store
- A peek works the same as a pop except the asynchronous steps don't need to be run

### what happens on a push
- write a queue element to hot storage to the hot-hot block (not yet in cold storage)
  - attach to the end of the queue so that a peek on a subscriber at the end of the queue sees the
    element
  - note that pushes are atomic, only one can happen at a time
- if the hot-hot block reaches the minimum block size asynchronously save to cold storage
- change the hot-hot block into a regular hot block
- start new hot-hot block
- note that if s3 is used for long term storage, the hot-hot block should be written to a local disk
  as it is written in case the hot storage goes down.  Attempting to minimize loss data on a server
  crash
- note that the hot-hot block stays in memory whether or not it has subscribers

## startup procedure
- load the subscriber list/queue position into hot storage
- go to the block data location and read every metadata file into the hot store
  - now we have a list of all the blocks that exist
- for each subscriber, load the block it is in into the hot store and maybe the next one
  - probably in an asynchronous way
- allocate the new live/hot-hot block for new pushes

## sharding
- what happens when the block metadata + block data is too big to handle in memory on a single
  server
  - create shards that only have access to certain blocks
  - system for redirecting requests to the correct shard
    - shared way for learning which shard has access to a specific block?
  - system for rebalancing the system when the shards need to hold a different number of blocks
    - can you add nodes that handle a portion of the blocks, then remove blocks from other shards
      that are no longer serving them?

## Cold Store
Manages long term storage blocks.  It saves them and makes them available.

### public interface
- create a new queue
- create a new block
- write a block
- list all blocks

- read the block for a given block id, subscriber, and  queue name and write into in memory db
- next block id after the given block id
- previous block id before the given block id

### how are blocks organized
- metadata 
  - minimum and maximum queue element in the block
  - checksum on the data
  - path to the data file
  - next block
  - previous block
- data
  - queue element id/timestamp/etc
  - text contents of the element: string, jsonlines, etc
    - version 2 can store complex objects like images or references to other files
- external settings
  - block size is settable by the thing that writes it

### Creating a block
- public 
