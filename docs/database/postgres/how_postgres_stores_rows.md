# How Postgres Stores Rows

## Logical Structure of Database Cluster

A database cluster is a collection of databases managed by a PostgreSQL server. The term `database cluster` in PostgreSQL does not mean a group of database servers. A PostgreSQL server runs on a single host and manages a single database cluster.

![](https://user-images.githubusercontent.com/17776979/194694319-47f53feb-71eb-4fd5-bb53-7d76a8f29ded.png)

A database is a collection of database objects such as tables, indexes, views... All the database objects in PostgreSQL are internally managed by respective **object identifiers (OIDs)** , which are unsigned 4-byte integers

The relations between database objects and the respective OIDs are stored in appropriate system [catalogs](https://www.postgresql.org/docs/current/catalogs.html), depending on the type of objects.

For example, `OIDs` of `databases` and `tables` are stored in `pg_database` and `pg_class` respectively

## Physical Structure of Database Cluster

A database cluster basically is one directory referred to as **base directory**, and it contains some subdirectories and lots of files, the path of the base directory is usually set to the environment variable **PGDATA**.

![](https://user-images.githubusercontent.com/17776979/194694740-1dcb154e-0792-4eb9-b39b-c0e6b3aad819.png)

The database file layout can be found in [here](https://www.postgresql.org/docs/current/storage-file-layout.html)

A database is a subdirectory under the base subdirectory; and the database directory names are identical to the respective `OIDs`. For example, when the `OID` of the database `sampledb` is `16384`, its subdirectory name is `16384`.

```bash
$ cd $PGDATA
$ ls -ld base/16384
drwx------  213 postgres postgres  7242  8 26 16:33 16384
```

Each table or index whose size is less than 1GB is a single file stored under the database directory it belongs to. Tables and indexes as database objects are internally managed by individual `OIDs`, while those data files are managed by the variable, `relfilenode`.

For example `19427` is the `relfilenode` of the table `sampletb` then the data will be store under file `19427`

Each file size should not exceed 1GB, When the file size of tables and indexes exceeds 1GB, PostgreSQL creates a new file named like `relfilenode.1` and uses it. If the new file has been filled up, next new file named like `relfilenode.2` will be created, and so on. This arrangement avoids problems on platforms that have file size limitations.

```bash
$ cd $PGDATA
$ ls -la -h base/16384/19427*
-rw------- 1 postgres postgres 1.0G  Apr  21 11:16 data/base/16384/19427
-rw------- 1 postgres postgres  45M  Apr  21 11:20 data/base/16384/19427.1
...
```

## Internal Layout of a Heap Table File

Inside the data file (heap table and index, as well as the free space map and visibility map), it is divided into **pages** (or **blocks**) of fixed length, the default is 8192 byte (8 KB). Those pages within each file are numbered sequentially from 0, and such numbers are called as **block numbers**. If the file has been filled up, PostgreSQL adds a new empty page to the end of the file to increase the file size.

![](https://user-images.githubusercontent.com/17776979/194695442-dc4619f9-94f9-4d53-bb26-e5017e4521c3.png)

- `page header` (24 bytes) carries page meta information
- `lp(1..N)` is line pointer array. It points to a logical offset within that page. Since these are arrays, elements are fixed sized but the number of elements can vary.
- `row(1..N)` represents actual SQL rows. They are added in reverse order within a page
- `special space` is typically used when storing indexes in these page, usually sibling nodes in a B-Tree for example. For table data this is not used.
- `lower` and 1upper1 pointer mark the locations that have already been used in a page. Thus, (upper - lower) = free page space

PostgreSQL implements **row storage**. All columns of a row `sampletb` are held in contiguous memory (in the same heap file page).

![](https://user-images.githubusercontent.com/17776979/194696419-57e02696-6c38-458e-9cc5-d6a62fbe99e6.png)

Loading one heap file page reads **all columns** of all contained rows, regardless of whether the query uses `select *` or `select id` to access the row

### Writing Heap Tuples

Suppose a table composed of one page which contains just one heap tuple. When the second tuple is inserted, it is placed after the first one. The second line pointer is pushed onto the first one, and it points to the second tuple. The pd_lower changes to point to the second line pointer, and the pd_upper to the second heap tuple.

![](https://user-images.githubusercontent.com/17776979/194695830-c9937cb6-4908-4973-9575-df9a45c2733f.png)

### Reading Heap Tuples

Two typical access methods, sequential scan and B-tree index scan, are outlined here:

- **Sequential scan** – All tuples in all pages are sequentially read by scanning all line pointers in each page.
- **B-tree index scan** – An index file contains index tuples, each of which is composed of an index key and a TID pointing to the target heap tuple. If the index tuple with the key that you are looking for has been found, PostgreSQL reads the desired heap tuple using the obtained TID value. For example, TID value of the obtained index tuple is `(block = 7, Offset = 2)`. It means that the target heap tuple is `2nd` tuple in the `7th` page within the table, so PostgreSQL can read the desired heap tuple without unnecessary scanning in the pages.

![](https://user-images.githubusercontent.com/17776979/194695890-f43be3b4-89a7-440c-bcad-d08e67ffeefc.png)

## Peek into pages

Internally PostgreSQL maintains a unique row id `(ctid)` for our data which is usually opaque to users. We can query it explicitly to see its value.

```
select ctid, id from sampletb

+-------+----+
| ctid  | id |
+-------+----+
| (0,8) | 7  |
+-------+----+
```

In **ctid**, first digit stand for the **page number** and the second digit stands for the **tuple number**. PostgreSQL moves around these tuples when VACUUM is run to defragment the page.
