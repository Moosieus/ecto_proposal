There's an increasingly significant market share of Postgres extensions that are radically changing the way people interact with Postgres. These extensions often desirable as they provide lucrative and key-enabling features without requiring applications to synchronize external data stores. At least two extensions in this category include:

* [ParadeDB](https://www.paradedb.com/) which adds BM25 full text search and hybrid search when used in concert with pgvector.
* [Apache AGE](https://age.apache.org/) which provides graph database functionality.

Extensions often introduce their own syntax and calling conventions that embed within SQL queries.

An example for ParadeDB may look like:
```sql
SELECT paradedb.score("p0") AS "p0_score", "p0".*
FROM "posts" AS "p0"
WHERE
  "p0".id @@@ paradedb.boolean(
    must => paradedb.parse("body:(elixir erlang)"),
    should => [
      paradedb.phrase(field => "body", phrase => ARRAY['is', 'awesome'], 1),
      paradedb.phrase(field => "body", phrase => ARRAY['are', 'awesome'], 1),
    ]
  )
ORDER BY "p0_score"
LIMIT 20;
```

An example for AGE may look like:
```sql
SELECT *
FROM cypher('graph_name', $$
  MATCH (a), (e) WHERE a.name = 'Alice' AND e.name = 'Eskil'
  RETURN a.age, e.age, abs(a.age - e.age)
$$) as (alice_age agtype, eskil_age agtype, difference agtype);
```
(source: [AGE docs](https://age.apache.org/age-manual/master/functions/numeric_functions.html#abs))

Alternatively:
```sql
SELECT id, 
    graph_query.name = t.name as names_match,
    graph_query.age = t.age as ages_match
FROM schema_name.sql_person AS t
JOIN cypher('graph_name', $$
        MATCH (n:Person)
        RETURN n.name, n.age, id(n)
$$) as graph_query(name agtype, age agtype, id agtype)
ON t.person_id = graph_query.id
```
(source: [AGE docs](https://age.apache.org/age-manual/master/advanced/advanced.html#using-cypher-in-a-join-expression))

A primary barrier to adoption for these extensions is that many (most?) developers do not write SQL directly, but instead use database access layers (DBALs) to compose queries. This is almost universally true for Elixir and Ecto, which provides an amazing interface for writing, composing, and mapping SQL queries.

Today it's possible to extend Ecto in two ways:
* The fragment API, which provides users an escape hatch to inject fragments of SQL.
* With Postgrex extensions, which provides users a means to encode and decode custom data types.

Two notable uses of these include the [`:timescale` package](https://github.com/davydog187/timescale) which uses fragments to implement Timescale's hyperfunctions, and the [`:pgvector` package](https://github.com/pgvector/pgvector-elixir) which is implemented in a Postgrex extension.

**Fluent example of a ParadeDB Ecto query:**
```elixir
import Ecto.Query

query =
  from(
    p in Post,
    search: boolean(p, [
      must: parse(p, "body:(elixir erlang)"),
      should: [
        phrase(p.body, ["is", "awesome"]),
        phrase(p.body, ["are", "awesome"])
      ]
    ]),
    select: {p_score in score(p), p},
    order_by: {:desc, p_score}
  )

query = search(query, [p], range(p.rating, 9000, nil, "[)"))

Repo.all(query)
```

**Example of a ParadeDB Ecto schema:**
```elixir
defmodule App.Post do
  use Ecto.Schema

  schema "posts" do
    field :title, :string
    field :body, :string
    field :likes, :integer

    belongs_to :author, App.User
    has_many :comments, App.Comment

    # Extend schema DSL to encode additional information
    # about the schema
    search_index [
      :title, :body, {:likes, :int4}
    ]
  end
end
```
