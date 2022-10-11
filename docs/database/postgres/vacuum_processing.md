# Vacuum Processing

Vacuum processing is a maintenance process that facilitates the persistent operation of PostgreSQL. Its two main tasks are:

- Removing dead tuples
- Freezing transaction ids

To remove dead tuples, vacuum processing provides two modes, i.e. **Concurrent VACUUM** and **Full VACUUM**

## Visibility Map

Vacuum processing is costly; thus, the VM was been introduced to reduce this cost.

The basic concept of the VM is simple. Each table has an individual visibility map that holds the visibility of each page in the table file. The visibility of pages determines whether each page has dead tuples. Vacuum processing can skip a page that does not have dead tuples.

Suppose that the table consists of three pages, and the 0th and 2nd pages contain dead tuples and the 1st page does not. The VM of this table holds information about which pages contain dead tuples. In this case, vacuum processing skips the 1st page by referring to the VM's information.

![](https://user-images.githubusercontent.com/17776979/195036087-a7dd817e-55d8-42c6-acaa-e5e1c0c0d952.png) 

## Concurrent VACUUM

Vacuum processing performs the following tasks for specified tables or all tables in the database.

1. Removing dead tuples
- Remove dead tuples and defragment live tuples for each page.
- Remove index tuples that point to dead tuples.

![](https://user-images.githubusercontent.com/17776979/195034188-bf03f533-234b-470d-b76a-4950cdc2a92c.png) 

Assume that the table contains three pages. We focus on the first page. This page has three tuples. `Tuple_2` is a dead tuple. In this case, PostgreSQL removes `Tuple_2` and reorders the remaining tuples to repair fragmentation, and then updates both the FSM and VM of this page. PostgreSQL continues this process until the last page.

Note that unnecessary line pointers are not removed and they will be reused in future. Because, if line pointers are removed, all index tuples of the associated indexes must be updated.

2. Freezing old txids
- Freeze old txids of tuples if necessary.
- Update frozen txid related system catalogs (pg_database and pg_class).
- Remove unnecessary parts of the clog if possible.

Freeze processing has two modes, and it is performed in either mode depending on certain conditions. For convenience, these modes are referred to as lazy mode and eager mode.

Freeze processing typically runs in lazy mode; however, eager mode is run when specific conditions are satisfied.

- In lazy mode, freeze processing scans only pages that contain dead tuples using the respective VM of the target tables.
- In contrast, the eager mode scans all pages regardless of whether each page contains dead tuples or not, and it also updates system catalogs related to freeze processing and removes unnecessary parts of the clog if possible.

3. Others
- Update the FSM and VM of processed tables.
- Update several statistics (pg_stat_all_tables, etc).

### Autovacuum Daemon

Vacuum processing has been automated with the autovacuum daemon; thus, the operation of PostgreSQL has become extremely easy.

The autovacuum daemon periodically invokes several autovacuum_worker processes. By default, it wakes every 1 min (defined by [autovacuum_naptime](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html#GUC-AUTOVACUUM-NAPTIME)), and invokes three workers (defined by [autovacuum_max_works](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html#GUC-AUTOVACUUM-MAX-WORKERS)).

## Full VACUUM

Although Concurrent VACUUM is essential for operation, it is not sufficient. For example, it cannot reduce table size even if many dead tuples are removed.

![](https://user-images.githubusercontent.com/17776979/195037249-ece71936-2646-4320-b010-67dfa708320f.png) 

In the image above, the VACUUM command is executed to remove dead tuples. The dead tuples are removed; however, the table size is not reduced. This is both a waste of disk space and has a negative impact on database performance.

To deal with this situation, PostgreSQL provides the Full VACUUM mode.

![](https://user-images.githubusercontent.com/17776979/195037757-dbde6db3-b160-4b42-8649-6934870de9a9.png) 

When you run VACUUM FULL on a table, that table is locked for the duration of the operation, so nothing else can work with the table. VACUUM FULL is much slower than a normal VACUUM, so the table may be unavailable for a while.

