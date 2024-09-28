# Proposal: Rich Extensibility for Ecto
There's an increasingly significant market share of Postgres extensions that are radically changing the way people interact with Postgres. These extensions can provide lucrative and key-enabling features without requiring applications to host and synchronize external data stores. At least two extensions in this category include:

* [ParadeDB](https://www.paradedb.com/) which adds BM25 full text search and hybrid search when used in concert with pgvector.
* [Apache AGE](https://age.apache.org/) which provides graph database functionality.

These extensions embed their calling conventions within SQL and even compose quite well within its relational model. It's desirable to extend Ecto to incorporate these calling conventions so developers can use these extensions with the amazing composition, caching, mapping, and other ammenities Ecto currently provides.

The following SQL query is a deliberately limit-testing example from ParadeDB:
```sql
SELECT "p0".*
FROM "posts" AS "p0"
WHERE
  "p0".id @@@ paradedb.boolean(
    must => ARRAY[
      paradedb.disjunction_max(
        disjuncts => ARRAY[
          paradedb.parse($1),
          paradedb.phrase(field => 'body', phrases => ARRAY['is', 'awesome'], slop => $2),
          paradedb.phrase(field => 'body', phrases => ARRAY['are', 'awesome'], slop => $2)
        ]
      ),
      paradedb.range(field => 'likes', range => int4range(9000, NULL, '[)')
      )
    ]
  )
LIMIT 20;
```

> I'm leaning on ParadeBD as it's what I know -- If anyone would like to propose further examples for Apache AGE or other extensions, I highly encourage you to share them here!

A user may want to compose the above SQL query in Ecto like so:
```elixir
import Ecto.Query

i_think_therefore_i_param = "body:elixir OR body:erlang"
slop = 1

query =
  from(
    p in Post,
    search: disjunction_max([
      parse(p, ^i_think_therefore_i_param),
      phrase(p.body, ["is", "awesome"], ^slop),
      phrase(p.body, ["are", "awesome"], ^slop)
    ]),
    limit: 20
  )

some_condition = true

query =
  if some_condition do
    search(query, [p], range(p.rating, 9000, nil, "[)"))
  else
    query
  end

Repo.all(query)
```

Ecto's existing fragment API can cover simple variations of these queries ad-hoc, but it doesn't provide the desired level of extensibility demonstrated above. Additionally, users may wish to augment Ecto's other modules like `Ecto.Schema` to provide additional information.

One example includes mapping a ParadeDB search index for a given schema/table, like so:
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
    search_index [
      :title, :body, {:likes, :int4}, {:author_id, :int8}, #...
    ]
  end
end
```

# For the morning:
* Now that the problem domain's somewhat well defined, I can start proposing a solution here.
