## [pyodbc](https://github.com/mkleehammer/pyodbc) is great for connecting to SQL Server databases

#### In this post

- Python
- SQL (T-SQL)

____

I am often asked to enable transfer of data from one database to another that doesn't involve the manual export of data, and manual re-insert.

The human-process is error prone, slow, and boring.

One way to get around this is simply to use pyodbc in Python to read and write the data.

- See the tips at the bottom on how I make this process more efficient


### Notes

- The source and the destination tables have identical schemas

____

## Part 1 - Getting the data

### Step 1 - Connect to the source of the data

```python
import pyodbc

# This connects to a network/local instance of SQL Server, using Active Directory (Trusted_Connection)
connection_string = ''' DRIVER={ODBC Driver 17 for SQL Server};
                        SERVER=test;
                        DATABASE=test;
                        Trusted_Connection=yes
                    '''

# Creates the connection
conn = pyodbc.connect(connection_string)

```

[Source 1](https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-SQL-Server-from-Windows)  
[Source 2](https://stackoverflow.com/questions/16515420/connecting-to-ms-sql-server-with-windows-authentication-using-python)

### Step 2 - Read the data

Create a variable to store the data you want to transfer, using where clauses and parameters where necessary.

```python
# For conditions, don't hardcode if it can be avoided - use a ? to indicate placeholder
sql_to_execute = 'SELECT * FROM MyDB.dbo.TestTable1 WHERE LastUpdatedDate = ?'

# The date in ISO format, to avoid regional date issues
iso_date = 20210807

# Tuple of parameters (to replace each ? - if one parameter, end with comma)
parameters = (iso_date,)

# Execute the query and immediatley return the results
results = conn.execute(sql_to_execute, parameters).fetchall()

# Evaluate if anything was returned
if not results:
    print('Nothing was returned from SQL query!')

    # If this was called from a function, return False, otherwise sys.exit(0)
    return False

```

## Part 2 - Inserting the data

There are two ways to insert the data. Step 3.1 is one possible method, faster than standard inserts.

Step 3.2 shows a faster method.

**For both steps**, repeat Step 1 again, but this time for the target server. We'll call this `conn_target`

### Step 3.1 - 

```python
# With the target connection, get the cursor
conn_target_cursor = conn_target.cursor()

# Enable fast_executemany, to enable faster loading
conn_target_cursor.fast_executemany = True

# Create an insert statement, with ? parameters for each field
# Safer to specify specifically which columns you want to insert into, incase columns shift
insert_statement = 'INSERT INTO MyDB.dbo.TestTable1 (field1, ..., fieldn) VALUES (?, ... ,?)

# Now insert all of the data in one go
conn_target_cursor.executemany(insert_statement, results)


```
The problem with this method is that it can take longer than you're expecting due to the way pyodbc works

This is because pyodbc automatically enables transactions, and with more rows to insert, the time to insert new records grows quite exponentially as the transaction log grows with each insert.

#### Time to insert: 250,000 rows: **92 minutes**

### Step 3.2

The alternative to Step 3.1 is to insert the data in chunks, committing the data with each chunk.

```python
def chunks(data, chunk_size):

    # Loop through the data in specified chunk size
    for i in range(0, len(data), chunk_size):
    
        # Returns a generator (iterators you can only iterate through once)
        yield data[i:i + chunk_size]

# Specified chunk size
chunk_size = 5000

# With each loop, sends back a generator, stored in chunk
for chunk in chunks(result, chunk_size):

    # With executemany active, it quickly inserts the chunk we extracted
    conn_target_cursor.executemany(insert_statement, chunk)
    
    # Commit after each load
    conn_target_cursor.commit()
    
```

For every 5000 rows it inserts (as specified in the `chunk_size`), it commits to the database, preventing the transaction log from getting bigger, therefore making the overall insert faster.

#### Time to insert: 250,000 rows: **8 minutes** - 91% quicker.

#### Cons of 3.2

It does mean that your data is ready available in your table for others to start accessing before the entire load is complete. This may cause issues if other systems are waiting for records to be populated to start running other processes, and may start running before the entire load is completed (although this can be rectified quite easily).

If an error does occur while loading the data, previously commited data cannot be rolled back (including any deletions prior to starting the load).
