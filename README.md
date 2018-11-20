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

### public interface
- create a queue
- list all queues
- create a subscriber
- push
- pop
- peek

### database structure
- subscriber_id -> queue position
- list of all blocks in cold storage
  - list of all blocks in hot storage
  - lists of subscribers in each block, automatically updated when a subscriber moves
- list of queue elements in memory and their associated blocks
  - when the last subscriber leaves a block, the whole block is dropped out of the store

### what happens on a pop
- look up the queue position for the subscriber
- check if the block for the queue position is loaded
- optionally load it
- return the queue element
- asynchronously check if the subscriber has left the block
  - if they were the last one in the block and there is no subscriber that is reaching the end of
    the immediately previous block, drop the block out of memory
  - check if one of these processes is already running
- asynchronously check if the subscriber is reaching the end of their current block
  - load the next block
  - check if one of these processes is already running
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

### public interface
- create a new block
- write a block
- list all blocks
- read a block

### how are blocks organized
- metadata 
  - minimum and maximum queue element in the block
  - checksum on the data
  - path to the data file
- data
  - queue element id/timestamp/etc
  - text contents of the element: string, jsonlines, etc
    - version 2 can store complex objects like images or references to other files
- external settings
  - block size is settable by the thing that writes it
