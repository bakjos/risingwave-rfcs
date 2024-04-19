---
feature: Stream Executor with EMIT ON WINDOW CLOSE Semantics
authors:
  - "st1page"
start_date: "2023/02/14"
---

# Stream Executor with EMIT ON WINDOW CLOSE Semantics

## Summary

Introduce a new component Sort Buffer used in the streaming executor and give a general method to implement EMIT ON WINDOW CLOSE stream executor. Sort Buffer can support materialize and persistent the changes stream and drain the "stable" records with the input watermark. Because the input watermarks are always monotonically increasing, the output records are ordered by the watermark column and append-only.

Note: In this RFC, we will focus on the EMIT ON WINDOW CLOSE query. We abbreviate EMIT ON WINDOW CLOSE as **EOWC** in the rest of the RFC, and call the normal cases non-EOWC.

## Background and Definitions

Let's look back on the current designs and RFCs.

[RFC #30: The Semantics of EMIT ON WINDOW CLOSE](https://github.com/risingwavelabs/rfcs/pull/30) has given a strict definition of the semantics of EOWC. In short, the streaming job with the EOWC modifier emits the result when the window is closed and the result is complete, and no sooner, no later. 

According to the same RFC, The **User-defined Watermark** on the source make the user a way to declare which records from the source are outdated and can be discarded. Watermark gives a bound of the data as the data inputs, which provides the guarantee that some results are complete and triggers the emitting of the EMIT ON WINDOW CLOSE query.

And [RFC #2: The WatermarkFilter and StreamSort operator](https://github.com/risingwavelabs/rfcs/pull/2) introduces the WatermarkFilter stream operator. With the "user-defined watermark" and continuous ingested data, WatermarkFilter can generate consistent **Internal Watermark Stream Message** (hereinafter referred to as **"Watermark"** or **"Watermark message"**). A Watermark message with column index and value means there will be no data that have a larger watermark column value than the value in the watermark.

Also, the RFC proposes the **Watermark Derivation** of the stream operator. This design makes sure that all possible trigger messages generated by **User-defined Watermark** can be expressed as the **Watermark message**. The only thing that can transfer between
the stream operator is the Watermark message so the stream operator does not need to be aware of the "time window" or other concepts. The executor just needs to handle the input watermark no matter whether the watermark column is a "window_close" or a temporal filter's output column.

And then we can introduce **EOWC Flag** on any stream operator now. The stream operator with the flag works can make sure some result is complete with the input watermark and then emit them. If most downstream stream operator has EOWC Flag, the streaming query can satisfy the EOWC semantics.

In this RFC, we will focus on how to implement a stream executor with EOWC flag and reuse the logic of the normal stream executor.

## Sort Buffer Design

Compared with the normal stream executor, the EOWC executor needs to buffer the data until the result is complete. A general component `SortBuffer` is introduced here for executors.

SortBuffer consists of two parts:
- SortBufferTable
  - A normal stateTable to persist the buffered data.
  - The primary key's first column is a watermark column. We call it **sort key**.
- SortBufferCache (optional)
  - A row cache on the SortBufferTable.

SortBuffer should support the following operations:
- can accept watermark to know some rows have been complete.
- can drain the complete row in the memory ordered by the sort key.

```rust
impl SortBuffer {
  /// consume the next row ordered by the first column which is complete.
  /// return None if there is no remaining complete row.
  /// fill cache from the stateTable when cache miss.
  /// will consume the row in memory.
  fn next(&mut self, table: &StateTable) -> Option<Row>;

  /// get the minimum sort key in rows which is complete.
  /// return None if there is no remaining complete row.
  /// fill cache from the stateTable when cache miss.
  /// will not consume the row in memory.
  fn peek_sort_key(&self, table: &StateTable) -> Option<ScalarImpl>;

  /// use the first column's watermark to refresh the SortBuffer
  /// can trigger more rows complete which can be returned by `self.next`
  fn accept_watermark(&mut self, watermark: Watermark);
  
  /// apply changes operations
  fn insert(&mut self, row: Row);
  fn delete(&mut self, row: Row);
  fn apply_chunk(&mut self, c: Chunk);

  /// recovery from the stateTable, just gets the row belonging to stateTable's vnode.
  fn recovery(&mut self, table: &StateTable);
}
```

The component does not take over many things. There are still lots of things that should be done by the executor, such as

- all the operations should be applied on the table and cache, and the executor should write in duplicate.
- after getting the complete data, the executor must delete the useless data by itself (delete one by one/range delete with the last key). Notice that it is different with state clean and the data must be deleted consistently.

In the following, I will explain why and how it works in different situations.

## Executors

### Sort

SortExecutor can transform any stream into a append only stream ordered by a watermark column. It just wraps a SortBuffer.

- when stream chunk arrives
  - write the chunk into stateTable
  - write the chunk into SortBufferCache
- when watermark arrives
  - pass watermark into SortBufferCache
- when barrier arrives
  - drain complete rows from SortBufferCache, for each rows
    - emit the row downstream
    - delete the row in stateTable

Sort has no difference in EOWC and non-EOWC mode.

### New executors made availble by Sort

As mentioned in [RFC #2: The WatermarkFilter and StreamSort operator](https://github.com/risingwavelabs/rfcs/pull/2), some operators can only accept ordered input. They are made avaible after introducing Sort.

We only briefly introduce them here. Detailed designs are still needed.

#### SortAgg (with batch only agg calls)

Under EOWC semantics, if there is watermark in group key, we can use SortExecutor for the input stream, and then we get a sorted stream which can be processed by BatchSortAgg. Then, we can support all the aggregators which can run in batch mode.

The SortAgg supports more kinds of aggregators under EOWC, but should materialized all the input data, which could be worse than the non-EOWC GroupAgg. So we have another EOWC GroupAgg implementation as described below. I think the SortAgg is just to solve some aggregation functions which does not have a streaming version (hard to implement or with high cost).

#### OverAgg

See [RFC #8: Over Window on watermark](https://github.com/risingwavelabs/rfcs/pull/8).

We still need more detailed design, but it is clear that the row's completion and row's emission can not always happens at the same time.

### EOWC version of existing executors

The naive approach: By adding a SortBuffer after any executor, we immediately get its EOWC version.

For some executors, we can do better by reusing its result state table as the SortBufferTable, i.e., an invasive SortBuffer implementation. This is also why the `StateTable` is accepted as input for the `SortBuffer` API, instead of being owned by it.

#### GroupAgg

If there is watermark in group key, the EOWC GroupAgg calculates the aggregation with the same logic as non-EOWC GroupAgg, but it only emits the complete part of the agg result.

If we add a SortExecutor after a non-EOWC GroupAgg, we can find that the GroupAgg's result table and SortExecutor's stateTable exactly have the same schema and meaning. So the EOWC GroupAgg only needs to add a SortBufferCache on the agg's result table.

- when stream chunk arrives
  - do the same as non-EOWC GroupAgg
- when watermark arrives
  - pass watermark into SortBufferCache
- when barrier arrives
  - do the same as non-EOWC GroupAgg but not emit to down stream, just get the changes
    - write the changes into SortBufferCache
    - write the changes into result table
    - (the LRU cache should has been changed by the agg's logic)
  - drain complete rows from SortBufferCache, for each rows
    - emit the row downstream
    - delete the row in result table
  - emit watermark with the last emitted row's watermark

Note: This actually means we will have two caches with different cache policies for one state table (one for aggregate computation and one for emitting). It's arguable whether it's a good idea. Since it's easy to add a switch to enable/disable the SortBufferCache, we delay the decision to future work.

#### EqJoin

It is like window join in Flink.

When there are watermarks in the first join key of the both two sides. We can use two SortBuffers (table and cache) to implement a SortMergeJoin. SortBuffer’s `peek` method is used here.

#### IntervalJoin

We have discussed about the band join in [RFC #32: Band Join](https://github.com/risingwavelabs/rfcs/pull/32). And we will support the EOWC interval join with these formal definition which is same with FlinkSQL.

There should be at least one equal join key to shuffle and parallel process.

The interval condition should be on the two watermark columns. Considering they are `l_t` and `r_t` . The condition should be always able to transform to the form `l_t BETWEEN interval(r_t, r_t + constant)` .

There are two tables for each side and four tables in total. Assuming that `l_t` , `r_t` are the watermark columns, `l_pk` , `r_pk` are the stream keys and `l_k` , `r_k` as the equal join key, the tables' primary keys are:

- left state table: `[l_k, l_t, l_pk]`
- left sort table: `[l_t, l_k, l_pk]`
- right state table: `[r_k, r_t, r_pk]`
- right sort table: `[r_t, t_k, r_pk]`

The sort table is optimized to get the complete rows and the state table is optimized to match by equal key. The executor’s computing logic is that

1. get the lowest complete row **R** from one side’s sort buffer
2. Using the peek value from the other side’s sort buffer, check if the other side rows is complete match the **R** with the interval condition. If yes, consume the R from the sort buffer and go on
3. Get the rows in the other side’s state table with the join key and get all the rows which satisfy the equal condition and interval condition to do join calculation and emit the result.
4. Delete the R in the state table

And to accelerate the processing, we need to add the SortBuffer cache on the sort table and add a LRU cache on the state table. And because of the interval join's property, the read pattern is to find in a lowest time's interval with a group join key. So the cache strategy is the same as the cache of the max/min aggregator.