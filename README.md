# SQLite 2.8.1 for learning / debugging

Modified version of SQLite 2.8.1 that allows visualizing the BTree structure
easily. Original source code downloaded from here:
https://www.sqlite.org/src/info/590f963b6599e4e2

## Why SQLite 2.8.1?

I wanted to visualize how the BTree pages evolve on disk as records are added to
database tables. Initially, I cloned the
[current version of SQLite](https://github.com/sqlite/sqlite) (3.x.x), but I
quickly found out that SQLite is not a such a "small and simple" codebase these
days.

After that, I tried
[SQLite 2.5.0](https://github.com/davideuler/SQLite-2.5.0-for-code-reading) but
run into some seg faults. So I looked up a version that fixes important issues
and compiles easily with modern GCC requiring minor changes to the source code.
See here:

- [Release History](https://www.sqlite.org/changes.html)
- [Check-ins (commits) from 2003](https://www.sqlite.org/src/timeline?c=590f963b65&y=ci&b=2003-04-24+01:45:04)

## Compilation

Create a `build` directory, use the [`./configure`](./configure) script to
generate the Makefile and then call `make`:

```bash
mkdir build
cd build
../configure
make
```

## Visualizing the BTree

SQLite 2.x.x has a function called `sqliteBtreePageDump` which
prints an entire BTree to `STDOUT`, including page numbers, children pointers
and payload (key + data). I added some options to the SQLite shell to avoid
uncommenting and recompiling the code all over again when you need to print the
BTree:

```bash
sqlite> .help

# Default SQLite options here...

CUSTOM OPTIONS
.path ON|OFF           Prints the page numbers acquired when executing SQL
.keyhash ON|OFF        Prints the hash generated for the given key in an SQL statement
.btree PAGE FILE       Prints the Btree rooted on PAGE to FILE (or STDOUT if ommited)
```

Here's an example of how you can use these options. First, create a database
file and open the SQLite shell:

```bash
# Move back to the root if you're still inside the build directory
cd ..

# Create the DB file
touch db.sqlite

# Open the shell
./build/sqlite ./db.sqlite
```

Now create a simple table like this one:

```sql
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(255));
```

Once that's done, you need to populate the table with some records. Open a new
terminal and use the [`inserts.sh`](./inserts.sh) script to generate a `.sql`
file that you can execute at once:

```bash
./inserts.sh > inserts.sql
```

Go back to the SQLite shell that you opened previously and execute the SQL file:

```bash
.read inserts.sql
```

Now you need to find the page numbers of the primary key index root node and the
table data root node (they are 2 different B-Trees stored in the same file). For
that you can use the `sqlite_master` table:

```sql
SELECT type, name, rootpage FROM sqlite_master;
```

You'll get something like this as the output:

```text
table|users|4
index|(users autoindex 1)|3
```

In this case, the root page of the index is 3 while the root page of the data
is 4. Dump both B-Trees in their own file:

```
.btree 3 users.index
.btree 4 users.data
```

You'll see something like this if you open the files:

`users.index`
```bash
PAGE 3:
cell  0: i=8..31      chld=621  nk=12   nd=0    payload: key=b0G3Wt1 data=6480..........
cell  1: i=32..55     chld=622  nk=12   nd=0    payload: key=b0G3XcH data=3456..........
cell  2: i=56..79     chld=1173 nk=12   nd=0    payload: key=b0G3Y1H data=1728..........
                right_chld=1724
freeblock  0: i=80..511    size=432  total=432

# Rest of pages
```

`users.data`

```bash
PAGE 4:
cell  0: i=8..47      chld=383  nk=4    nd=23   payload: key=2197 data=...7804.User Name
cell  1: i=48..87     chld=384  nk=4    nd=23   payload: key=4394 data=...5607.User Name
cell  2: i=88..127    chld=767  nk=4    nd=23   payload: key=6591 data=...3410.User Name
cell  3: i=128..167   chld=1149 nk=4    nd=23   payload: key=8619 data=...1382.User Name
                right_chld=1531

# Rest of pages
```

Now turn on all the custom options:

```
.path ON
.keyhash ON
```

Execute a simple statement like this one:

```sql
SELECT rowid, * FROM users WHERE id = 25;
```

You'll get an output similar to this:

```text
Reading page 1
Reading page 4
Reading page 1
Reading page 3
Generate hash for 25.000000 -> 0G1v
Reading page 3
Reading page 621
Reading page 53
Reading page 8
Reading page 4
Reading page 1531
Reading page 1740
Reading page 1735
25|User Name 25
```

Copy the generated hash for key 25 (`0G1v` in this example) and
<kbd>CTRL</kbd> + <kbd>F</kbd> it in the `users.index` file. You'll find a line
like this one:

```text
payload: key=b0G1v   data=9976
```

The data is the `ROWID` and is located on page `8`. You can see in the path
above that the index traversal stops at page `8` and then the data traversal
starts at page `4`. The index B-Tree is used to obtain the `ROWID` of the
record, and then the data B-Tree is used to get the actual tuple (columns). You
can <kbd>CTRL</kbd> + <kbd>F</kbd> the `ROWID` in the data file to check that
the B-Tree traversal is correct (in this example, starts at page `4` and stops
at `1735`).

You can experiment with different page sizes by changing the value of
`SQLITE_PAGE_SIZE` at [`./src/pager.h`](./src/pager.h#27).

## Debugging

I added [`.vscode/launch.json`](./.vscode/launch.json) to easily step through
the source. I suggest setting up a break point on line 1000 at
[`./src/shell.c`](./src/shell.c#L1081) and then clicking on the Run/Debug icon.
You can see where the code goes from there by writing commands or SQL in the
`sqlite` shell that opens up.
