# CARTO ❤️ macOS (An installation guide)

There are a lot of reasons why you'd want to go bare metal with your CARTO installation. Among them:

+ Higher speeds
+ Full integration with your workflow (proper syntax highlight, copy and paste, etc)
+ Achieve a broader, more complete, understanding of how CARTO works.

## Toolbelt

You will need a couple of tools:

+ `git` (which you can get with XCode)
+ `brew` (which you can install get from [brew.sh](http://brew.sh))

NOTE: Make sure you have installed Xcode (via Mac App Store), the xcode command line tools (`xcode-select --install`) and agreed to the license (`sudo xcodebuild -license`), since it's a requirement for a lot of brew packages.

You can use other tools to install the necessary packages (`macports` for example) but I advise you stick with these ones and don't mix package managers. Even resorting to installing a package from source is better than mixing sym links and all that hell.

## Installation

All we're really doing, is following the [official install guide for CARTO](http://cartodb.readthedocs.io/en/latest/install.html) with some workarounds where non OSX tools are used.

As you may know, CARTO requires 5 main pieces (and its dependencies) in order to run:

+ PostgreSQL (and the CARTO PostgreSQL extension)
+ SQL API
+ Maps API
+ CARTO

We will now cover the installation of all of these. BTW, I'm assuming you're an organized human being and have a workspace. Mine is `~/Documents/workspace/carto/`. Also, NEVER run commands with `sudo` unless it's strictly necessary. Preventive `sudo` is a bad habit that spreads and messes up you're permissions. Don't do it! Always try without it first, and only `sudo` as a last resort.

### PostgreSQL

We will use `brew` to install PostgreSQL 9.5. There is one important bit though, we must specify `--with-python` to make sure we get that `plypythonu` extension we will later need.

If you have a previous version of PostgreSQL, make sure to migrate it. See `brew info postgresql` for details. If you decide to uninstall it completely, run `brew remove postgresql`. When you have sorted that out:
```
brew install homebrew/versions/postgresql95 --with-python
```

NOTE: Make sure you read the message left after the installation. It will give you instructions to add the postgres executables in your path. Follow those instructions before continuing or else you will get `command not found` errors for all `pg_...` commands.

Make PostgreSQL 9.5 server run:

```
pg_ctl -D /usr/local/var/postgresql@9.5 -l /usr/local/var/postgresql@9.5/server.log start
```

There is one point here where I'm sure there's a better solution, but it turns out that the db is created with a default superuser role with the name of you session user (in my case 'guido'). The (quite embarrassing) way of changing this for me is:

```
# Create a temp superadmin role
psql postgres
postgres=# CREATE USER banana WITH createrole superuser;
postgres=# \q

# Rename you default user to postgres using that temp role
psql -U banana postgres
postgres=> ALTER USER guido RENAME TO postgres;
postgres=> \q

# Drop the temp role
psql -U postgres
postgres=# DROP USER banana;
postgres=# \q
```

Ok! So we have a PostgreSQL database we can use. You can run `brew services start postgresql95` to start the PostgreSQL db (and restart automatically when you restart your computer). Now we have to configure it. By convention, `brew` will install everything under `/usr/local/var/`. This is important, we will have to do some symlinking later. For now, let's modify the `pg_hba.conf` file according to instructions by doing

```
vim /usr/local/var/postgres/pg_hba.conf
```

Inside this file, we will have to scroll to the bottom and add a single line

```
local all postgres trust
```

Now, our postgres user has access to the db. Now, let's add some roles that the app will use:

```
createuser publicuser --no-createrole --no-createdb --no-superuser -U postgres
createuser tileuser --no-createrole --no-createdb --no-superuser -U postgres
```

You're db is now configured! We will now install the cartodb-postgresql extension and its dependencies.

### PostgreSQL CARTO extension

To install the extension, all you need is to clone the extension's git (in your workspace, organized human being), `cd` into it and make it (no sudo!!!). This will build some files and put them in the postgres extension path we saw before. Note that you will have to visit the repo and find out what the latest [release tag](https://github.com/CartoDB/cartodb-postgresql/releases) is. At the time of writing this guide it's `v0.18.5`

```
cd ~/Documents/workspace/carto
git clone https://github.com/CartoDB/cartodb-postgresql.git
cd cartodb-postgresql
git checkout <latest-release-tag>
make all install
```

Of course, now we have to install the extension's dependencies.

#### Dependencies

One said dependency is `plypythonu`, but luckily that was taken care of when we specified `--with-python` in the PostgreSQL installation. Let's talk instead of `postgis`.

Installing `postgis` is very simple. We'll install from source to be sure sure that it will use PostgreSQL 9.5. You also need to create some postgis templates in the db for CARTO to work with:

```
brew install automake libtool
cd ~/Documents/workspace
git clone https://github.com/postgis/postgis.git
cd postgis
git checkout 2.2.1
./autogen.sh
./configure
make install
sudo createdb -T template0 -O postgres -U postgres -E UTF8 template_postgis
sudo createlang plpgsql -U postgres -d template_postgis
psql -U postgres template_postgis -c 'CREATE EXTENSION postgis;CREATE EXTENSION postgis_topology;'
```

Execute the above command now. This will take some time and if you're attentive enough you'll see the log say `installing gdal...`. Great right? Not so great!

We use a very specific version of `ogr2ogr`. At the time of this guide, it's `2.1.0`. The issue is that the `gdal` dependency that has been installed with `postgis` contains `ogr2ogr v1.1.11`, which is not good. To make things worse, there is no `brew` installer for this version and the automatic formula for installing "latest" (2.2) is broken atm (and "latest" builds are not maintained by brew, understandably).

Fear not, local installer, we will install from source! In your workspace (maybe not in `~/Documents/workspace/carto` since it's not really a part of CARTO), let's clone the GDAL repo:

```
cd ~/Documents/workspace
git clone https://github.com/OSGeo/gdal.git
cd gdal/gdal
git checkout tags/2.1.0
./configure
make install
```

NOTE: If you see an error mentioning jpeg2000 while installing GDAL 2.1, try replacing `JAS_CAST(uchar *, buf)` with `JAS_CAST(unsigned char*, buf)` in frmts/jpeg2000/jpeg2000_vsil_io.cpp, line 212. Via http://osgeo-org.1560.x6.nabble.com/gdal-dev-jpeg2000-jasper-error-compiling-gdal-2-1-from-git-release-branch-td5299100.html

NOTE: If you're having trouble with `bash-completion`, do: `brew remove bash-completion && brew install bash-completion@2` and try again.

That should be working. Now we need to focus on `schema_tiggers`. For that, we'll build from source as well. We will clone the `pg_schema_triggers` in our workspace and make it:

```
cd ~/Documents/workspace
git clone https://github.com/CartoDB/pg_schema_triggers.git
cd pg_schema_triggers
make
make install
```

That's it, all dependencies needed for building the `cartodb` extensions are here. Now we can:

```
psql -U postgres
postgres=# CREATE EXTENSION plpythonu;
CREATE EXTENSION
postgres=# CREATE EXTENSION schema_triggers;
CREATE EXTENSION
postgres=# CREATE EXTENSION postgis;
CREATE EXTENSION
postgres=# CREATE EXTENSION cartodb;
CREATE EXTENSION
postgres=# \q
```

This should finish successfully. If not, stop and ask for help.

We'll also install the `odbc_fdw` extension to make the latest connectors work. We'll clone a CARTO maintained version of the extension for PostgreSQL 9.5+ and compile+install.

```
cd ~/Documents/workspace
git clone https://github.com/CartoDB/odbc_fdw.git
cd odbc_fdw
make install
```

NOTE: If you find issues with a missing `sql.h` header file, do: `brew install psqlodbc` and try again.

NOTE: In case you've tried these instructions with PostgreSQL 9.6 and encountered an error with odbc_fdw, try editing `odbc_fdw.c` with the changes found here: https://github.com/CartoDB/odbc_fdw/pull/46/files

If the installation is successful we're done here. The user creation process will create the extension for all new users. If you want to upgrade an existent user, make sure to connect to the user database (you can find the database name in `user.database_name` field) and run:

```
whatever_user_database=# CREATE EXTENSION odbc_fdw;
CREATE EXTENSION
```

Your DB is ready to go!

### SQL API

Let's try to get the SQL API up and running.

#### Dependencies

Let's tackle first the dependencies. We need:

+ Redis
+ Node

To install redis, we can simply:

```
brew install redis
```

For node, you need to know we use an outdated version, so we need a good tool to manage node versions. This tool is called `nvm`. Let's install it:

```
brew install nvm
```

NOTE: Pay attention to the instructions at the end of this execution: it tells you how to set `nvm` to work in your machine (create dirs and export paths). If you don't do this, you might get a 'nvm command not found' error. If you closed the message (fast fella huh?) you can bring it back doing `brew info nvm` ;).

Now let's tell `nvm` to install the version of `node` we want and to set it as global:

```
nvm install v0.10.26
nvm use global v0.10.26
```

Let's download the actual SQL API code:

```
cd ~/Documents/workspace/carto
git clone git://github.com/CartoDB/CartoDB-SQL-API.git
cd CartoDB-SQL-API
git checkout master
```

Now we can install all the packages (timely):

```
npm install
```

By now, you might realise that the SQL API is a Node app. We need to configure it. Let's start by copying the sample config as our main config:

```
cp config/environments/development.js.example config/environments/development.js
```

One last step: let's point this app to listen in `0.0.0.0`, which is needed since CARTO will run locally and need all it's parts to listen to internal requests (you might be thinking `127.0.0.1`, but that would listen to local on a network but not same-machine comms).

Open `config/environments/development.js` in your favorite editor, find the line that reads `module.exports.node_host = '127.0.0.1';` and make sure you change it to `module.exports.node_host = '0.0.0.0';`. Save and let's try to start it up:


```
node app.js development
```

Congratulations! The SQL API is running in macOS!

### Maps API

The dependencies for the Maps API are the same as for the SQL API, if you skipped that section go install those dependencies and come back.

Let's clone the repo for the Maps API:

```
cd ~/Documents/workspace/carto
git clone https://github.com/CartoDB/Windshaft-cartodb.git
cd Windshaft-cartodb
git checkout master
```

I know I said all dependencies were installed, but that's not completely true. We need some stuff for `cairo` to work, so let's intall `pango` via brew:

```
brew install pango
```

Now we can install all the packages (timely):

```
npm install
```

By now, you might realise that the Maps API is a Node app. We need to configure it. Let's start by copying the sample config as our main config:

```
cp config/environments/development.js.example config/environments/development.js
```

One last step: let's point this app to listen in `0.0.0.0`, which is needed since CARTO will run locally and need all it's parts to listen to internal requests (you might be thinking `127.0.0.1`, but that would listen to local on a network but not same-machine comms).

Open `config/environments/development.js` in your favorite editor, find the line that reads `host: '127.0.0.1'` and make sure you change it to `host: '0.0.0.0'`. Save and let's try to start it up:

NOTE: If you're having segmentation fault errors, try to change your node version to 0.10.47.
```
node app.js development
```

Congrats! You have the Maps API (a full tiler!) running in macOS.

### CARTO Builder

Now let's go for the piece that consumes from them all, the Builder! This is a Rails app, so we need:

+ Ruby (v2.2.3)

#### Ruby

Like with the SQL API and Maps API where we needed `nvm` to keep track of the `node` version we were using, we'll use `rvm` to keep install the right version of Ruby for our need (v2.2.3). Let's first install `rvm` (no `brew` port, sorry!)

```
\curl -sSL https://get.rvm.io | bash -s stable --ruby
source ~/.rvm/scripts/rvm
```

And now we can do (these two will take a while):

```
rvm install ruby-2.2.3
rvm --default use 2.2.3
```

### Rails

Now that we have Ruby, we can tackle Rails and all other Ruby gems in one go. We just need `bundler` and the source code for the CARTO Builder.

```
gem install bundler
```

What bundler does is grab all the gems defined in the `Gemfile` file and install them with their specified version. Convenient! Just do (this is timely as well):

```
cd ~/Documents/workspace/carto
git clone https://github.com/CartoDB/cartodb.git
cd CartoDB
bundle install
sudo easy_install pip
```

In case you see an eventmachine/openssl issue with `bundle install`, either in this step or perhaps later, try `bundle config build.eventmachine --with-cppflags=-I$(brew --prefix openssl)/include` via http://stackoverflow.com/questions/30818391/gem-eventmachine-fatal-error-openssl-ssl-h-file-not-found/31516586#31516586

Edit python_requirements.txt and change GDAL version to 2.1.0: `gdal==2.1.0`.

```
sudo pip install -r python_requirements.txt
```

If previous command fails because of six, try adding `--ignore-installed six`.

Now we just have to configure our Rails server. We will have to copy two different sample config files and make them our real config files:

```
cp config/database.yml.sample config/database.yml
cp config/app_config.yml.sample config/app_config.yml
```

PRO TIP: You may also symlink these. That way it's easier to keep up with future changes done to the `.sample` file, usually meaning new features.

Let's take a moment to update submodules in the CartoDB repo, and make sure we have the latest version of the PostgreSQL extension.

```
git submodule init
git submodule update
cd lib/sql
make install
cd ../..
```

NOTE: You'll have to do this every time the PostgreSQL extension is updated!

Ok, we're almost done. Remember we built from source `gdal` v2.2.1 and installed it? Well, that brings `ogr2ogr` v2.2.1 with it and that's what we need, but sadly the config file is thought for on-premise installations were the binary is renamed with every version. So you need to open `config/app_config.yml` and change `binary:           'which ogr2ogr2.1` to `binary: 'which ogr2ogr'`.

While we're on the subject of `ogr2ogr`, used by the importer to grab a CSV and put it in a db table, let's install one other thing the importer will need: the `unp` unpacker.

```
brew install unp
```

Now, let's migrate our db to work with CARTO (in addition to postgresql, you must run redis):

```
redis-server &
bundle exec rake db:create
bundle exec rake db:migrate
```

PRO TIP: you might want to `alias bex='bundle exec'`, it's used often ;)

One last step is to install all the `node` modules (CARTO Builder also uses them for front stuff):

```
npm install
```

Let's start it up! (Make sure your PostgreSQL database is running, as well as your Redis server).

```
bundle exec rails server
```

Boom, you're running! Don't forget to start the resque script for queued jobs to run:

```
bundle exec script/resque
```

## After installing

Well, you've done away with the vagrant and you're free. Now let me suggest you create aliases for starting all the different pieces. I suggest these:

```
alias start_builder='cd ~/Documents/workspace/carto/cartoDB; bundle exec rails s -p 3000'
alias start_resque='cd ~/Documents/workspace/carto/cartoDB; bundle exec script/resque'
alias start_tiler='cd ~/Documents/workspace/carto/Windshaft-cartodb; node app.js development'
alias start_sql_api='cd ~/Documents/workspace/carto/CartoDB-SQL-API; node app.js development'
alias start_redis='redis-server &'
alias start_postgres='pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start'
alias stop_postgres='pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log stop'
```

Now you can open up a fullscreen terminal with 4 sections:

```
+---------------+-----------------+
|               |  start_postgres |
| start_sql_api |  start_redis    |
|               |  start_builder  |
+---------------+-----------------+
|               |                 |
|  start_tiler  |  start_resque   |
|               |                 |
+---------------+-----------------+
```

And there you have it, CARTO is running on the Mac.

### Creating a user

You need to create a user to be able to use CARTO. To do so (the `mkdir log` is just in case you don't have it):

```
cd ~/Documents/workspace/carto
mkdir log
script/create_dev_user
```

Follow the instructions. I'll assume you chose the users username (or domain, as the script calls it) `username`. Now all that is left is that you insert a rule in `/etc/hosts` to redirect `username.localhost.lan:3000` to Rails. Open up (you can't avoid sudo here) `/etc/hosts` in your editor and add:

```
127.0.0.1 username.localhost.lan
```

Save and you're done. Fire up all the pieces and visit `username.localhost.lan:3000` in you favorite browser.
