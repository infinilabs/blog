---
title: "Build a Vector Extension for Postgres - Introduction"
meta_title: "Build a Vector Extension for Postgres - Introduction"
description: ""
date: 2024-12-18T16:00:00+08:00
image: "/images/posts/2024/build_a_vector_extension_for_postgres_introduction/bg.png"
categories: ["vector database", "Postgres"]
author: "SteveLauC"
tags: ["vector database", "Postgres"]
lang: "en"
category: "Blog"
subcategory: "Technology"
draft: false
---

## Why and What

Vector databases are really hot topics nowadays. I have always been curious about what they are and how they work under the hood, so let's build one ourselves. Building a whole new database from scratch is not practical, we need some building blocks, or, just a real database system. Postgres has a long-standing reputation for its extensibility, which makes it a perfect fit for our needs, and projects like [pgvector][pgvector] have already demonstrated it is viable to add vector support to Postgres as an extension.

We are going to implement vector support for Postgres, but, what are the detailed features to implement? This is not a hard question, the definition of [Vector database][vector_db_wikipedia] from Wikipedia shows us the right direction:

> A vector database, vector store or vector search engine is a database that can store vectors (fixed-length lists of numbers) along with other data items. Vector databases typically implement one or more Approximate Nearest Neighbor algorithms so that one can search the database with a query vector to retrieve the closest matching database records

Alright, so we need to enable Postgres to store vectors, and be able to perform Top-K queries, i.e., for a given input vector, Postgres should return the K vectors that are most similar (or closest) to it. If we express them in SQL, it would look like this:

```sql
-- Create a table, which has a column of type `vector(3)`, 3 is the dimension of the vector
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));

-- Insert vectors, Postgres should store them!
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');

-- Now, Postgres should return the Top-5 vectors that are most similar to
-- [3, 1, 2]
SELECT * FROM items ORDER BY embedding <=> '[3,1,2]' LIMIT 5;
```

Example SQL makes things clear, to summarize, we need to:

1. Implement a `vector` type for Postgres, it should accept a dimension parameter
2. Implement that `<=>` binary operator, which should calculate the similarity of the given 2 vectors and return it

## Set up the environment

I will use the Rust language and a library called [`pgrx`][pgrx], to install Rust you simply need to follow the instructions [here][install_rust], then you run this command to set up `cargo-pgrx`, a cargo sub-command to manage everything related to `pgrx`:

```sh
$ cargo install --locked cargo-pgrx
$ cargo pgrx --version # to verify that it gets installed
```

Now we need a Postgres server to run and test our project, I would just let `pgrx` install a brand new Postgres for me to make things easier. At the time of writing, [Postgres 17 is the latest version][pg17_release], so I will use it.

`pgrx` builds Postgres from source, so you need to ensure these [requirements][build_pg_requirements] are satisfied. `pgrx` also has a page about the [system requirements][pgrx_system_requiremens], but Postgres is really well-documented, it deserves a read. Once you have everything set up, run:

```sh
$ cargo pgrx init --pg17 download
```

## The initial commit

Now let's write some code, `cargo pgrx`, just like `cargo`, provides a `new` sub-command to create new projects, say we call our project `pg_vector_ext`, run:

```sh
$ cargo pgrx new pg_vector_ext
```

```sh
$ cd pg_vector_ext
$ tree .
pg_vector_ext/
├── Cargo.toml
├── pg_vector_ext.control
├── sql
└── src
    ├── bin
    │   └── pgrx_embed.rs
    └── lib.rs

4 directories, 4 files
```

From this, we can see, `pgrx` creates some template files for us. For now, we only care about the `src/lib.rs` file.

```sh
$ bat src/lib.rs
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: src/lib.rs
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ use pgrx::prelude::*;
   2   │
   3   │ ::pgrx::pg_module_magic!();
   4   │
   5   │ #[pg_extern]
   6   │ fn hello_pg_vector_ext() -> &'static str {
   7   │     "Hello, pg_vector_ext"
   8   │ }
   9   │
  10   │ #[cfg(any(test, feature = "pg_test"))]
  11   │ #[pg_schema]
  12   │ mod tests {
  13   │     use pgrx::prelude::*;
  14   │
  15   │     #[pg_test]
  16   │     fn test_hello_pg_vector_ext() {
  17   │         assert_eq!("Hello, pg_vector_ext", crate::hello_pg_vector_ext());
  18   │     }
  19   │
  20   │ }
  21   │
  22   │ /// This module is required by `cargo pgrx test` invocations.
  23   │ /// It must be visible at the root of your extension crate.
  24   │ #[cfg(test)]
  25   │ pub mod pg_test {
  26   │     pub fn setup(_options: Vec<&str>) {
  27   │         // perform one-off initialization when the pg_test framework starts
  28   │     }
  29   │
  30   │     #[must_use]
  31   │     pub fn postgresql_conf_options() -> Vec<&'static str> {
  32   │         // return any postgresql.conf settings that are required for your tests
  33   │         vec![]
  34   │     }
  35   │ }
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Ignore the `tests` module (as it is for testing), we can see that `pgrx` creates a function `hello_pg_vector_ext()`, this is something callable in SQL, if we run the project via:

> Before running it, you need to edit the `Cargo.toml` file, within the `features` section, change the default feature to `pg17`, and optionally, you can remove the `pg*` features other than `pg17` as they won't be used:
>
> ```toml
> [features]
> default = ["pg17"]
> pg17 = ["pgrx/pg17", "pgrx-tests/pg17" ]
> pg_test = []
> ```

```sh
$ cargo pgrx run
```

It will start the Postgres 17 instance and connect to it via `psql`, we can install our extension and run the function:

```sql
pg_vector_ext=# CREATE EXTENSION pg_vector_ext;
CREATE EXTENSION
pg_vector_ext=# SELECT hello_pg_vector_ext();
 hello_pg_vector_ext
----------------------
 Hello, pg_vector_ext
(1 row)
```

This is our first attempt at `pgrx` and also our first commit to the project. In the next post, I will implement the `vector` type so that Postgres can store vectors.

[pgvector]: https://github.com/pgvector/pgvector
[vector_db_wikipedia]: https://en.wikipedia.org/wiki/Vector_database
[pgrx]: https://github.com/pgcentralfoundation/pgrx
[install_rust]: https://www.rust-lang.org/tools/install
[pg17_release]: https://www.postgresql.org/about/news/postgresql-17-released-2936/
[build_pg_requirements]: https://www.postgresql.org/docs/current/install-requirements.html
[pgrx_system_requiremens]: https://github.com/pgcentralfoundation/pgrx/?tab=readme-ov-file#system-requirements
