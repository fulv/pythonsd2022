# Installing and using pg_stat_statements in your local Postgres

## Installation
Think of Postgres extensions as plugins or add-ons.  
Extensions must be added to the static configuration before the Postgres runtime can load them.  
Once loaded, an extension must be “created”, i.e. all its objects, functions and views need to be constructed and linked.  
Some extensions come with the Postgres installation, albeit not installed, others can be downloaded and/or built from scratch.   
`pg_stat_statements` comes with the installation but is not installed by default, so we have to do it ourselves.

## Add the extension to the Postgres configuration
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
## Connect to Postgres as a Superuser

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
