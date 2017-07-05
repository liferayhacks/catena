# Catena - SQL on a blockchain

Catena is a distributed database based on a blockchain, accessible using SQL. Catena timestamps database transactions (SQL) in a decentralized way between nodes that do not or cannot trust each other, while enforcing modification permissions ('grants') that were agreed upon earlier.

A Catena blockchain contains SQL transactions that, when executed in order, lead to the agreed-upon state of the database.
The transactions are automatically replicated to, validated by, and replayed on participating clients. A Catena database 
can be connected to by client applications using the PostgreSQL wire protocol (pq). 

Only SQL statements that modify data or structure are included in the blockchain. This is very similar to replication logs
used by e.g. MySQL ('binlog').

## Building

### macOS

Catena builds on macOS. You need a recent version of XCode (>=8.3.2) on your system. Use the following commands to clone
the Catena repository and build in debug configuration:

````
git clone https://github.com/pixelspark/catena.git catena
cd catena
swift build
````

It is also possible to generate an XCode project and build Catena from it:

````
swift package generate-xcodeproj
````

### Linux

Building on Linux is experimental. Due to the fact that there is no cross-platform WebSocket client implementation in Swift
(the current implementation uses Starscream), outgoing peer connections are not supported for the Linux client. Incoming peer connections are possible and can be used by the client to 'talk back', so the client is functional regardless.

To try and compile, first ensure Swift 3.1 is installed. Then ensure clang and required libraries are present:

````
apt install clang build-essential libicu-dev libcurl4-openssl-dev openssl libssl-dev
git clone https://github.com/pixelspark/catena.git catena
cd catena
swift build
````

The above was tested on Ubuntu 16.04 with Swift 3.1.

### Building a Docker image

````
git clone https://github.com/pixelspark/catena ./catena
cd ./catena
docker build -t pixelspark/catena .
````

## Running

### Natively

The following command starts Catena and initializes a new chain:

````
./.build/debug/Catena -p 8338 -s 'my seed string' -i -m
````

The -i switch tells Catena to initialize a chain (this deletes any persisted data, which is stored by default in catena.sqlite in the current directory). The -s switch provides Catena with a string that tells it which genesis block to accept.

To start another peer locally, use the following:

````
./.build/debug/Catena -p 8340 -s 'my seed string' -j 127.0.0.1:8338 -d peer2.sqlite
````

### Running with Docker

```
docker pull pixelspark/catena
docker run -p 8338:8338 -p 8339:8339 pixelspark/catena /root/.build/debug/Catena --help
````

Note: the port number on which Catena listens inside the container must be equal to the port number used outside the
container (as Catena advertises it to peers).

## Using Catena

Catena provides an HTTP interface on port 8338 (default), which can be used for introspecting the blockchain. It also
provides a WebSocket service which is used for communication between peers. 

The (private)  SQL interface is available on port 8339 (by default). If you set a different HTTP port (using the '-p' command line switch), the SQL interface will assume that port+1. You can connect to the SQL interface using the PostgreSQL command line client:

````
psql -h localhost -p 8334 -U random
````

Your username should be the public key (generated by Catena) and your password the private key. Catena will print the public
and private key for the 'root' user when initializing a new chain (with the -i option) and will also print a psql command line for
you to use to connect as root.

By default, only the 'root' user created during initialization has any rights on the database. Add rows to the 'grants' table to
create more 'users' (identified by public key). For example (as root):

````
CREATE TABLE test(foo INT);
INSERT INTO grants(kind, user, table) VALUES ('insert', $invoker, 'test');
INSERT INTO test(foo) VALUES (1337);
````

For testing, you can supply the username 'random' with any password; Catena will generate a new keypair for each (mutating) query and report it back to you as INFO messages in the Postgres client.

To enable block mining, add the '-m' command line switch.

## FAQ

### Is Catena a drop-in replacement for a regular SQL database?

No. The goal of Catena is to make it as easy as possible for developers and administrators that are used to working with 
SQL to adopt blockchain technology. Catena supports the PostgreSQL (pq) wire protocol to submit queries, which allows
Catena to be used from many different languages (such as PHP, Go, C/C++). However, there are fundamental differences 
between Catena and 'regular' database systems:

* Catena currently does not support many SQL features.
* Catena's consistency model is very different from other databases. In particular, any changes you make are not immediately visible nor confirmed. Transactions may roll back at any time depending on which transactions are included in the 'winning' blockchain.
* Catena will (in the future) check user privileges when changing or adding data, but can never prevent users from seeing all data (all users that are connected to a Catena blockchain can 'see' all transactions). Of course it is possible to set up a private chain.

### Which SQL features are supported by Catena?

Catena supports a limited subset of SQL (Catena implements its own SQL parser to sanitize and canonicalize SQL queries).
Currently, the following types of statements are supported:

* CREATE TABLE foo (bar TEXT, baz INT);
* INSERT INTO table (x, y, z) VALUES ('text', 1337);
* SELECT DISTINCT *, x, y, 'value', 123 FROM table LEFT JOIN other ON x=y WHERE x=10;
* DELETE FROM foo WHERE x=10;
* UPDATE foo SET bar=baz WHERE bar=1;
* DROP TABLE foo;

SQL has the following limitations:

* Column and table names are case-insensitive, and must start with an alphabetic (a-Z) character, and may subsequently contain numbers and underscores. Column names may be placed between double quotes.
* SQL keywords (such as 'SELECT') are case-insensitive.
* Whitespace is allowed between different tokens in an SQL statement, but not inside (e.g. "123 45" will not parse).
* All statements must end with a semicolon.
* Values can be a string (between 'single quotes'), an integer, blobs (X'hex' syntax) or NULL.
* An expression can be a value, '*' a column name, or a supported operation
* Supported comparison operators are "=", "<>", "<", ">", ">=", "<="
* Supported mathematical operators are "+", "-", "/" and "*". The concatenation operator "||" is also supported.
* Other supported operators are the prefix "-" for negation, "NOT", and "x IS NULL" / "x IS NOT NULL"
* Currently only the types 'TEXT' , 'INT' and 'BLOB' are supported.
* The special `$invoker` variable can be used to refer to the SHA256-hash of the current public key of the transaction invoker

In the future, the Catena parser will be expanded to support more types of statements. Only deterministic queries will
be supported (e.g. no functions that return current date/time or random values).

### What kind of blockchain is implemented by Catena?

Catena uses a Blockchain based on SHA-256 hashes for proof of work, with configurable difficulty. Blocks contain 
transactions which contain SQL statements. Catena is written from scratch and is therefore completely different from
Bitcoin, Ethereum etc.

### How does a Catena node talk to other nodes?

Catena nodes expose an HTTP/WebSocket interface. A node connects to the WebSocket interface of all other nodes it knows
about (initially specified from the command line) to fetch block information and exchange peers. In order for two nodes
to be able to communicate, at least one must be able to accept incoming connections (i.e. not be behind NAT or firewall).

### What is the consistency model for Catena?

SQL statements are grouped in transactions, which become part of a block. Once a block as been accepted in the blockchain and
is succeeded by a sufficient number of newer blocks, the block has become an immutable part of the blockchain ledger.

As new blocks still run the risk of being 'replaced' by competing blocks that have been mined (which may or may not include
a recent transaction), the most recent transactions run the risk of being rolled back. 

### How are changes to a Catena blockchain authenticated?

Transactions are signed using a private key.

In the future, Catena will provide a grants system (like in regular databases) based on the public key.
 A transaction that modifies a certain table or row needs to be signed with a private key that has the required grants. Grants
 will be stored in a special 'grants' table (which, in turn, can be modified by those that have a grant to modify that table).

To prevent replaying signed transactions, Catena will record a transaction number for each public key, which is atomically 
incremented for every transaction that is executed. A transaction will not execute (again) if it has a lower transaction
number than the latest number recorded in the blockchain.

### Where does the name come from?

Catena is Italian for 'chain'.

### Can I run a private Catena chain?
Chains are identified by their genesis (first) block's hash. To create a private chain, use the '-s'  option to specify 
a different starting seed. 

## MIT license

````
Copyright (c) 2017 Pixelspark, Tommy van der Vorst

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
````

## Contributing

We welcome contributions of all kinds - from typo fixes to complete refactors and new features. Just be sure to contact us if you want to work on something big, to prevent double effort. You can help in the following ways:

* Open an issue with suggestions for improvements
* Submit a pull request (bug fix, new feature, improved documentation)

Note that before we can accept any new code to the repository, we need you to confirm in writing that your contribution is made available to us under the terms of the MIT license.
