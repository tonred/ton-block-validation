> [!IMPORTANT]  
> This solution changes the `tests_dir`, so please either pass it explicitly as an argument or modify [contest/grader/grader.cpp](contest/grader/grader.cpp)

We focused mainly on single-threaded performance optimization and achieve 25-30% performance gain


# Changes

## `CellStorageStat::add_used_storage` Improvement

Replacing `std::map` with `absl::flat_hash_map` in `CellStorageStat::add_used_storage` significantly improved performance.
This is an obvious, but highly effective optimization

> [!WARNING]  
> Replacing `std::map` with `std::unordered_map` can be exploited by crafting cache with colliding keys,
> while `absl::flat_hash_map` mitigates this risk through built-in randomization

Additionally, I created a specialized method `CellStorageStat::add_used_storage_fast`, which:

- Always adds the cell to the seen set (no `kill_dup` param)
- Always counts if there are `bits` and `cells`  (no `skip_count_root` param)
- Optimizes the cell and bits overflow checks

After exploring various approaches for several days, current implementation of this method was the most efficient


## `CellStorageStat::add_used_storage_fast` Caching

I implemented caching for cells that have been seen in the current block - 
even if they appear in another transaction (or even account) within the same block

- Caching is activated for account with **more than 1 transaction**. In accounts with only
one transaction cache building takes more time than profit from it
- Only cells with **no more than 8 children** are cached.
Caching cells with more children may take more time and space, but almost never repeats in the same block
- Cache is cleared at the start of each block processing


## `CellHash` Hash Optimization

I optimized the abseil hash (`AbslHashValue`) of `CellHash` by eliminating intermediate steps
and performing a direct conversion of [8–16] bytes into `td::uint64`

**Default implementation**  `std::hash<vm::CellHash>()(cell_hash)` performs the following steps:
1. Slice hash calculation: `cell_hash_slice_hash(s.as_slice())`
2. Substring extraction: `hash.substr(8, 8)`
3. Conversion: `td::as<size_t>`

**Optimization:**  
All these steps are now consolidated into a single zero-copy call, improving efficiency


## Calculation of Cells and Merkle Depth in `DataCell` Creation

To further improve the efficiency of `CellStorageStat::add_used_storage_fast`,
I calculate the number of cells and the `merkle_depth` during `DataCell::create`.
This values can be reused later. This change can enable (but not yet implemented):

- Elimination of `merkle_depth`** from `CacheInfo`
- Conversion `seen` from a `flat_hash_map` to a `flat_hash_set` 
- Determine whether a cell should be cached in the beginning cell processing



# Tried but Not Implemented

## Threads

Parallelize `ContestValidateQuery::check_transactions` by processing each account in a new thread.
It takes a lot of time... However, this approach yielded less performance improvement and lead to
exceptions in random tests due to different non-thread-safe places. So we decided to focus on
single-threaded performance optimization


## Caching More Cells in `CellStorageStat::add_used_storage_fast`

It was observed that most cells appear only once per block, making it ineffective to cache every cell


## SIMD / AVX2 / AVX512

Platform specific


_PS I forgot to delete additional logs, `timer1` and `timer2` from code.
It will take some additional time during testing, so results can be even better_
