# Heap Only Tuple and Index-Only Scans

## Heap Only Tuple (HOT)

The HOT was implemented in version 8.3 to effectively use the pages of both index and table when the updated row is stored in the same table page that stores the old row. The HOT also reduces the necessity of VACUUM processing.

**Update a Row Without HOT**

![](https://user-images.githubusercontent.com/17776979/197468288-2409bc9b-e983-4cac-b0e1-acf3de1f8da4.png)

In this case, PostgreSQL inserts not only the new table tuple but also the new index tuple in the index page

**Update a row with HOT**

![](https://user-images.githubusercontent.com/17776979/197468464-ab1b89ed-72f0-482f-baa0-1584faca66f2.png)

For example, in this case, `Tuple_1` and `Tuple_2` are set to the `HEAP_HOT_UPDATED` bit and the `HEAP_ONLY_TUPLE` bit, respectively. These bits are used regardless of the `pruning` and the `defragmentation` processes.

**Pruning of the line pointers**

![](https://user-images.githubusercontent.com/17776979/197469324-c068161d-b716-4c04-a3ba-1b103677f1ba.png)

**Defragmentation of the dead tuples**

![](https://user-images.githubusercontent.com/17776979/197469339-c4c82a68-723e-4d7f-b323-6e9504027f34.png)

Note that the cost of defragmentation is less than the cost of normal VACUUM processing because defragmentation does not involve removing the index tuples.

Thus, using HOT reduces the consumption of both indexes and tables of pages; this also reduces the number of tuples that the VACUUM processing has to process. Therefore, HOT has a good influence on performance because it eventually reduces the number of insertions of the index tuples by updating and the necessity of VACUUM processing.

**The Cases in which HOT is not available**

When the updated tuple is stored in the other page, which does not store the old tuple, the index tuple that points to the tuple is also inserted in the index page.

## Index-Only Scans

To reduce the I/O (input/output) cost, index-only scans (often called index-only access) directly use the index key without accessing the corresponding table pages when all of the target entries of the SELECT statement are included in the index key.

In the following, using a specific example, a description of how index-only scans in PostgreSQL perform is given.

We have a table ‘tbl’ of which the definition is shown below:

```bash
Table: "public.tbl"

 Column |  Type   | Modifiers
--------+---------+-----------
 id     | integer |
 name   | text    |
 data   | text    |

Indexes: "tbl_idx" btree (id, name)
```

Let us explore how PostgreSQL reads tuples when the following SELECT command is executed.

```
SELECT id, name FROM tbl WHERE id BETWEEN 18 and 19;
```

This query gets data from two columns of the table: ‘id’ and ‘name’, and the index ‘tbl_idx’ is composed of these columns. Thus, when using index scan, it seems at first glance that accessing the table pages is not required because the index tuples contain the necessary data.

However, in fact, PostgreSQL has to check the **visibility** of the tuples in principle, and the index tuples do not have any information about transactions such as the `t_xmin` and `t_xmax` of the heap tuples.

PostgreSQL uses the `visibility map` of the target table. If all tuples stored in a page are visible, PostgreSQL uses the key of the index tuple and does not access the table page that is pointed at from the index tuple to check its visibility; otherwise, PostgreSQL reads the table tuple that is pointed at from the index tuple and checks the visibility of the tuple, which is the ordinary process.

![](https://user-images.githubusercontent.com/17776979/197470728-dee9a28b-ff8a-4e9d-97d7-944ac5339b56.png)

In above example, `Tuple_18` need not be accessed because the `0th` page that stores `Tuple_18` is visible, that is, all tuples including `Tuple_18` in the `0th` page are visible. In contrast, `Tuple_19` needs to be accessed to treat the concurrency control because the visibility of the `1st` page is not visible
