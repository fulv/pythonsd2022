# Installing and using pg_stat_statements in your local Postgres

## Installation
Think of Postgres extensions as plugins or add-ons.  
Extensions must be added to the static configuration before the Postgres runtime can load them.  
Once loaded, an extension must be “created”, i.e. all its objects, functions and views need to be constructed and linked.  
Some extensions come with the Postgres installation, albeit not installed, others can be downloaded and/or built from scratch.   
`pg_stat_statements` comes with the installation but is not installed by default, so we have to do it ourselves.
