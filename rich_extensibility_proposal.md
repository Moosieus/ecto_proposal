There's an increasingly significant market share of Postgres extensions that are radically changing the way people interact with Postgres. These extensions often desirable as they provide lucrative and key-enabling features without requiring applications to synchronize external data stores. At least two extensions in this category include:

* [ParadeDB](https://www.paradedb.com/) which adds BM25 full text search and hybrid search when used in concert with pgvector.
* [Apache AGE](https://age.apache.org/) which provides graph database functionality.

A primary barrier to adoption for these extensions is that many (most?) developers do not write SQL directly, but instead use database access layers (DBALs) to compose queries. This is overwhelmingly the case for Ecto, which serves as the DBAL for almost all Elixir applications.

While Ecto provides an escape hatch for inserting SQL fragments 

Today it's possible to extend Ecto in two ways:
* The fragment API, which provides users an escape hatch to inject SQL.
* With Postgrex extensions, which provides users a means to encode and decode custom data types.
