* Construct a base handler for all HTTP servers to glog.Fatal the
  process with a deferred recover func to prevent HTTP from swallowing
  panics which might otherwise be holding locks, etc.

* Transactions

  - Modified ops:

    - Get: logic to support clock skew uncertainty and concurrency.

      Simple, common case: if most recent timestamp for key is less
      than Timestamp and committed, read value.

      If most recent timestamp for key is greater than MaxTimestamp
      (it can be either committed or an intent), and there are no
      versions of key between Timestamp and MaxTimestamp, read value
      for key at Timestamp.

      If there are version(s) of the key (can include most recent
      intent) with timestamp between Timestamp and MaxTimestamp,
      return WriteWithinUncertaintyInterval error. The ResponseHeader
      will contain the latest version's timestamp (actually, just the
      latest version with timestamp <= MaxTimestamp); on transaction
      retry, Timestamp will be min(key's timestamp + 1, MaxTimestamp),
      and MaxTimestamp will be max(key's timestamp + 1, MaxTimestamp).

      In the event an intent is encountered with timestamp <=
      Timestamp, try to push the transaction to Timestamp + 1. If
      already committed, resolve and retry the read. If push succeeds,
      read value for key at Timestamp. If push fails, backoff and
      retry the transaction.

      After reading any value, update the read-timestamp-cache with
      the txn's Timestamp.

    - Put: additions to support intent marker.

      If entry exists but is intent:

        - If intent owned by another txn, try to push transaction
          (with "Abort"=true). If result of push is already-committed
          or aborted, resolve the existing intent and retry put. If
          push succeeds (meaning the other txn is aborted), delete
          existing intent and write new intent. If push fails, backoff
          and retry the transaction.

* StoreFinder using Gossip protocol to filter


* Range split

  - Split criteria: is range larger than max_range_bytes?

  - Transactionally rewrite range addressing indexes.

    - Need a range-wide "split" intent which blocks all mutating
      ops to range.

* Rebalance range replica. Only fully-replicated ranges may be
  rebalanced.

  - Keep a rebalance queue in memory. Range replicas are added to the
    queue from a store during initial range scan and also during
    operation as a response to certain conditions. Listed here:

    - A range is split. Each replica in the split range is marked as
      needing rebalancing.

    - Replica not matching zone config. When zone config changes happen,
      all ranges are scanned by each store and any mismatched replicas
      are added to the queue.

  - Rebalance away from stores finding themselves in top N space
    utilized, taking care to account for fewer than N stores in
    cluster. Only stores finding themselves in the top N space
    utilized set may have rebalances in effect.

  - Rebalance target is selected from available stores in bottom N
    space utilized. Adjacent stores are exempted.

  - Add rebalance target to replica set and rewrite range addressing
    indexes.

  - Rebalance targets are added to replica set always exactly one at a
    time. Targets are marked as REBALANCING. Obsolete sources are
    marked as PENDING_DELETION. Any time a range becomes fully
    replicated, the range leader replica will move REBALANCING
    replicas into state OK and will remove PENDING_DELETION replicas
    from the RangeDescriptor. The store which owns a removed replica
    is responsible for clearing the relevant portion of the key space
    as well as any other housekeeping details.

* Implement all ops that operate on a range for the case in which
  the given key range overlaps multiple logical ranges (Scan, DeleteRange etc).
  Requires transactions. Split the operation addressed to a key range into
  subranges that each hit a single range only, and run all of those as a
  distributed transaction.