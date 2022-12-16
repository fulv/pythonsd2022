# Installing and using pg_stat_statements in your local Postgres

## Installation
Think of Postgres extensions as plugins or add-ons.  
Extensions must be added to the static configuration before the Postgres runtime can load them.  
Once loaded, an extension must be “created”, i.e. all its objects, functions and views need to be constructed and linked.  
Some extensions come with the Postgres installation, albeit not installed, others can be downloaded and/or built from scratch.   
`pg_stat_statements` comes with the installation but is not installed by default, so we have to do it ourselves.

### Add the extension to the Postgres configuration
Look for the main Postgres configuration file `postgresql.conf` in your local development environment.
In this file, look for the line containing the string shared_preload_libraries.  In my conf file this is in line 706.  Change the line to 

```
shared_preload_libraries = 'pg_stat_statements' # (change requires restart)
```

Make sure you have removed the hashtag from the beginning of the line.

#### Restart Postgres

```
> ./pg stop
> ./pg start
waiting for server to start.... done
server started
```
### Connect to Postgres as a Superuser

> The next step must be done by a superuser.  Connecting to Postgres as a superuser comes with the same caveats as running a shell as the `root` user - remember to switch back to a non-privileged user once you are done with the task that required elevated privileges.

#### Find a superuser
First of all, connect to your local Postgres instance with the `psql` tool using the following command.  
`psql` can be invoked with certain defaults, but this command leaves nothing implicit in the connection string and should work provided you know 
the `<username>`, the `<password>` and the `<dbname>`:

```
> psql postgresql://<username>:<password>@localhost:5432/<dbname>
```

Next, use the `\du` command to list all user accounts:

```
dbname=> \du
                                    List of roles
 Role name  |                         Attributes                         | Member of
------------+------------------------------------------------------------+-----------
 <username> | Create DB                                                  | {}
 <superuser>| Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 ```
 
This shows that the default `<username>` user does not have the Superuser role.  On your machine, there should be a user in Postgres with the same name as your local Unix user account (on mine that name is `<superuser>`), which should have the Superuser role.  The default password configured by the local dev setup is `<password>`, just like it is for the `<username>` user.

#### Use the Superuser

All this is to say that you should be able to connect as a Superuser with:

```
> psql postgresql://yourusername:<password>@localhost:5432/<dbname>
psql (13.8)
Type "help" for help.

dbname=#
```

Notice the `#` prompt indicating that you are a Superuser, analogous to if you were `root` in the shell.

### Create Extension

If you are curious, the command `select * from pg_available_extensions order by name;` shows all available and installed extensions.  
If there is a version number in the `installed_version` column, the corresponding extension is installed.  
If you run this command before and after the next step you will see the difference.

This command effectively installs the extension:

```
dbname=# create extension pg_stat_statements;
CREATE EXTENSION
```

Try the following commands:

```
dbname=# \dx+ pg_stat_statements
     Objects in extension "pg_stat_statements"
                Object description
---------------------------------------------------
 function pg_stat_statements(boolean)
 function pg_stat_statements_reset(oid,oid,bigint)
 view pg_stat_statements
(3 rows)
```

```
select pg_stat_statements_reset();
 pg_stat_statements_reset
--------------------------

(1 row)
```

```
dbname=# select query, calls from pg_stat_statements order by calls desc limit 20;
.....
```

If you don’t get any errors, congratulations!  You are ready to use `pg_stat_statements`.

### Uninstalling

The extension is not installed by default in Postgres because it puts some additional load on it.  
I have not found this to be a problem on my local, but to get back to a clean slate do this:

```
dbname=# drop extension pg_stat_statements;
```

Then remove or comment out this line from `fareharbor.com/postgres/postgresql.conf`:

```
shared_preload_libraries = 'pg_stat_statements' # (change requires restart)
```

And finally restart postgres.

## Usage

Reference: [F.32. pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

`pg_stat_statements` has a bunch of configuration parameters, but the defaults are fine for our purposes.  
It also provides dozens of columns with lots of details for each query.  
Our Systems Reliability Engineering Team uses many of these for sophisticated statement-level metrics.  
For our purposes, the only columns we care about are `calls` and `query`.  
Suffice to say that each row in the `pg_stat_statements` view represents a distinct SQL query, 
and the `calls` column tells us how many times that `query` was executed, 
either since Postgres was started or since the last call to `pg_stat_statements_reset()`.

However, note that literal constants appearing in each query are replaced by a parameter symbol, 
such as `$1`, in the value displayed for the `query` column.  That way, these two queries:

```
SELECT * FROM blog_entry WHERE headline LIKE '%Lennon%';
SELECT * FROM blog_entry WHERE headline LIKE '%Harrison%';
```

are considered identical and are represented in `query` as:

```
SELECT * FROM blog_entry WHERE headline LIKE $1;
```

## Alternatives

To be sure, there are other ways to achieve similar goals:

* The Django debug toolbar can show SQL queries, but it can't be used for RESTApi requests or webhooks or management tasks.
* You can run `./manage.py shell_plus --print-sql`, but it's indiscriminate and dumps **all** the queries, 
so it can become difficult to wade through all the output.
Also, when dealing with a code path that is very far down a specific stack, it can be tricky to set up all the objects involved just the way you need them to reproduce a desired scenario.
* It should be possible to start `DEBUG=True ./manage.py runserver`, but it would be indiscriminate and difficult to wade through all the output.
* In `shell_plus` or in the debugger, you can `print(somequeryset.query)`, but `query` is defined on the `QuerySet` base class and does not work for the various accessors, attributes and descriptors we use, which could also trigger a query.
* At FareHarbor, we automatically keep track of query counts in each RESTApi endpoint test case, e.g.:

```
 server.charter.tests.external_api.test_external_api.CompaniesTest: 
  endpoint_query_count: 
    default: 15 
    replica_1: 0 
```

So we can have tests fail when there is a regression in the number of query counts, which is great.  And this is scoped to the test case, which is also great.  But the count is the total for **all** the queries called in the test, it's not granular enough to tell which exact queries actually happened.
