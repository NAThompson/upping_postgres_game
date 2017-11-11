slidenumbers: true

## Upping your Postgres Game

  Nick Thompson, bandgap.io


---

> A developer’s job encompasses more than just writing code. Our job is to produce results, and for that we have many tools at our disposal. SQL is one of them

 -Dimitri Fontaine, “Mastering PostgreSQL in Application Development.”

---

## Postgres

is the "premier" open-source relational database.

That's what the Postgres core devs say, anyway.


---

## Postgres vs. Sqlite

Sqlite can be deployed onto a user device-this is infeasible for postgres.


A postgres db can accessed by multiple servers, a sqlite db can't.

---

## Postgres vs. MySQL

Postgres supports more of the ANSI SQL standard

MySql supports multimaster replication, Postgres only supports single read/write master, multiple read-only slave horizontal scaling.

---

## I'm gonna build the next facebook

Can I use postgres without multimaster replication?


Yes, reddit uses [postgres](https://github.com/reddit/reddit/wiki/Architecture-Overview), so presumably this isn't a huge restriction.

---

## Postgres 10

We'll use postgres 10 in this presentation, release in October 2017.


---

## The build

```bash
$ sudo apt install -y gcc bison flex libssl-dev libreadline6 libreadline6-dev make
$ wget https://ftp.postgresql.org/pub/source/v10.0/postgresql-10.0.tar.gz
$ tar -xvf postgresql-10.0.tar.gz; cd postgresql-10.0.tar.gz
$ ./configure --prefix=/usr
$ make
$ sudo make install
```

---

## Starting off

```bash
$ sudo adduser postgres
$ sudo mkdir /pgsql/data
$ sudo chown postgres /pgsql/data
$ sudo su - postgres
$ initdb -D /pgsql/data
$ pg_ctl -D /pgsql/data/ -l logfile start
```

---

## Creating a new database

```bash
$ createdb testdb
createdb: could not connect to database template1: FATAL:  role "ubuntu" does not exist
```

---

## Fighting the Postgres Permission System

New users invariably struggle with the Postgres permission system, so let's fight it now.

---

## Postgres Roles $$\sim$$ Unix users

The `postgres` role is a superuser, so this will work:

```bash
$ createdb -U postgres testdb
```

but it is wise to create other roles with fewer permissions.

---

## Create a new Postgresql Role

Start a postgres session:

```bash
$ psql -U postgres
```

```sql
CREATE ROLE ubuntu LOGIN PASSWORD 'changeme';
```

---

## Who can do what?

```bash
$ psql
testdb=> \du
                                     List of roles
  Role name   |                         Attributes                         | Member of
--------------+------------------------------------------------------------+-----------
 ubuntu       |                                                            | {}
 postgres     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

---

## Editing permissions for Roles

```sql
ALTER ROLE ubuntu CREATEDB;
ALTER USER ubuntu WITH NOCREATEDB;
```

---

## Create new database with login role


```bash
$ createdb -U ubuntu testdb
$ psql testdb
psql (10.0)
Type "help" for help.

testdb=> \dt
```

---

## People who want to pwn you are real, use authentication!

> NoSQL, or rather NoAuthentication, has been a huge gift to the hacker
community. Just when I was worried that they'd finally patched all of the authentication bypass bugs in MySQL, new databases came into
style that lack authentication by design.

- [who knows?](http://pastebin.com/raw/0SNSvyjJ)


---

## Learn about Postgres Security

A lovely talk is [here](http://thebuild.com/presentations/securing-postgresql-scale15x.pdf)

---

## Password Login in Postgres

To force passwords to be used on login, use Postgres 10's new support for [SCRAM-SHA-256](https://en.wikipedia.org/wiki/Salted_Challenge_Response_Authentication_Mechanism)

Using SCRAM-SHA, the password never even goes over the network, only the hash.

---

## Enable SCRAM-SHA-256

The server doesn't know your password, so it stores the hash.

It either stores the MD5, or the SCRAM-SHA-256.

*If Postgres stores MD5, but accepts scram-sha-256, password authentication will fail, with no error message*.

The default password storage is MD5.

---

## Enable SCRAM-SHA-256

Edit `postgresql.conf`:

```c
password_encryption = scram-sha-256
```

Edit `pg_hba.conf`:


```bash
# TYPE  DATABASE        USER               METHOD
local   all             all                scram-sha-256
```

---

## Verify that passwords are stored using scram-sha-256

```sql
SELECT * FROM pg_shadow;
   username    | passwd
   postgres   | SCRAM-SHA-256$4096:2Wft2xkLOI6 . . .
```

---

## Changing password hashers

If you change the password hash from md5 to scram-sha-256, you have to reset all your users passwords.

```bash
$ psql
=> \password
Enter new password:
Enter it again:
```

---

## Changing `pg_hba.conf` requires a reload

```bash
$ pg_ctl reload -D /postgres/data/directory
```

Find the postgres data directory:

```sql
SHOW data_directory;
```

---

## Creating tables


```sql
create table users (email character[256], active bool, date_joined date);
```

---

## Examine table:

```
testdb=# \d users
                     Table "public.users"
   Column    |      Type      | Collation | Nullable | Default
-------------+----------------+-----------+----------+---------
 email       | character(1)[] |           |          |
 active      | boolean        |           |          |
 date_joined | date           |           |          |
```

---

## Get rid of foolish tables

```sql
drop table bad_table;
```

---

## List tables

```bash
$ psql -U ubuntu testdb
testdb=# \dt
         List of relations
 Schema | Name  | Type  |   Owner
--------+-------+-------+-----------
 public | users | table | ubuntu
(1 row)
```

---

## Adding rows to table

```sql
CREATE TABLE numbers (thing1 int);
INSERT INTO numbers VALUES (12);
SELECT * FROM numbers;
thing1
--------
    12
(1 row)
```

---

## Building schemas

A schema is a namespace for data tables.


```sql
$ psql -U ubuntu mydb;
mydb=> CREATE SCHEMA first_things;
mydb=> CREATE SCHEMA second_things;
mydb=> \dn # List schemas
List of schemas
 Name         |  Owner
--------------+----------
public        | ubuntu
first_things  | ubuntu
second_things | ubuntu
```


---

## Backups

```bash
$ pg_dump -h localhost -p 5432 -U ubuntu -F c -b -v -f compressed.backup mydb;
$ pg_restore --dbname=mydb compressed.backup;
```

---

## Getting help

The Postgres IRC has a bunch of really helpful people.

Before getting eaten alive on stackoverflow, go to #postgresql on irc.freenode.net.

---

## References

[PostgreSQL Up and Running](https://www.amazon.com/PostgreSQL-Running-Practical-Advanced-Database/dp/1491963417)

[Mastering PostgreSQL in Application Development](https://masteringpostgresql.com/)
