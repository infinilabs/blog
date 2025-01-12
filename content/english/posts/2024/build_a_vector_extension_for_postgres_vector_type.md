---
title: "Build a Vector Extension for Postgres - Vector Type"
meta_title: "Build a Vector Extension for Postgres - Vector Type"
description: ""
date: 2024-12-25T20:00:00+08:00
image: "/images/posts/2024/build_a_vector_extension_for_postgres_vector_type/bg.png"
categories: ["vector database", "Postgres"]
author: "SteveLauC"
tags: ["vector database", "Postgres"]
lang: "en"
category: "Technology"
subcategory: "Vector Type"
draft: false
---

In this post, we are going to implement the `vector` type for Postgres:

```sql
CREATE TABLE items (v vector(3));
```

## Postgres Extension Structure and pgrx Wrappers

Before we implement it, let's take a look at the typical extension structure, and how pgrx simplifies it for us.

A typical Postgres extension can be roughly split into 2 layers:

1. The implementation, which is usually done in a low-level language like C.
2. Higher level SQL statements that glue the implementation to Postgres.
3. A control file that specifies a few basic properties of the extension.

If you take a look at the source code of pgvector, this 3-layer structure is quite obvious, the [`src`](https://github.com/pgvector/pgvector/tree/master/src) directory is for C code, the [`sql`](https://github.com/pgvector/pgvector/tree/master/sql) directory contains higher SQL glues, and there is also a [control file](https://github.com/pgvector/pgvector/blob/master/vector.control). So how does pgrx make extension building easier?

1. It wraps the Postgres C APIs in Rust

   As we said, even though we build extensions in Rust, Postgres's APIs are still C, pgrx tries to wrap them in Rust so we don't need to bother with C.

2. SQL glues are generated using Rust macros, if possible

   Later we will see that pgrx could generate the SQL glues for us automatically.

3. pgrx generates the control file for us

## `CREATE TYPE vector`

Let's define our `Vector` type, using `std::vec::Vec` looks pretty straightforward, and since `vector` needs to store floats, we use `f64` here:

```rust
struct Vector {
    value: Vec<f64>
}
```

Then what?

The SQL statement used to create new types is `CREATE TYPE ...`, from its [documentation](https://www.postgresql.org/docs/current/sql-createtype.html), we would know that the `vector` type we are implementing is a _base type_, to create a base type, support functions `input_function` and `output_function` are required. And since it needs to take a dimension argument (`vector(DIMENSION)`), which is implemented using _modifer_, functions `type_modifier_input_function` and `type_modifier_output_function` are also needed. So we need to implement these 4 functions for our `Vector` type.

### `input_function`

To quote the documentation,

> The `input_function` converts the type's external textual representation to the internal representation used by the operators and functions defined for the type.
>
> The input function can be declared as taking one argument of type `cstring`, or as taking three arguments of types `cstring`, `oid`, `integer`. The first argument is the input text as a C string, the second argument is the type's own OID (except for array types, which instead receive their element type's OID), and the third is the typmod of the destination column, if known (-1 will be passed if not). The input function must return a value of the data type itself.

Ok, from the documentation, this `input_function` is used for deserialization, `serde` is the most popular de/serialization library in Rust, so let's use it. For the arguments, since the `vector` needs a type modifier, we need it to take 3 arguments. Here is what our `input_function` looks like:

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_input(
    input: &CStr,
    _oid: pg_sys::Oid,
    type_modifier: i32,
) -> Vector {
    let value = match serde_json::from_str::<Vec<f64>>(
        input.to_str().expect("expect input to be UTF-8 encoded"),
    ) {
        Ok(v) => v,
        Err(e) => {
            pgrx::error!("failed to deserialize the input string due to error {}", e)
        }
    };
    let dimension = match u16::try_from(value.len()) {
        Ok(d) => d,
        Err(_) => {
            pgrx::error!("this vector's dimension [{}] is too large", value.len());
        }
    };

    // cast should be safe as dimension should be a positive
    let expected_dimension = match u16::try_from(type_modifier) {
        Ok(d) => d,
        Err(_) => {
            panic!("failed to cast stored dimension [{}] to u16", type_modifier);
        }
    };

    // check the dimension
    if dimension != expected_dimension {
        pgrx::error!(
            "mismatched dimension, expected {}, found {}",
            expected_dimension,
            dimension
        );
    }

    Vector { value }
}
```

That's a bunch of stuff, let's go through it piece by piece.

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
```

If you mark a function with `pg_extern`, then `pgrx` will automatically generate SQL like `CREATE FUNCTION <your function>` for you, `immutable, strict, parallel_safe` are the attributes that you think your function has, they correspond to the attributes listed in the [`CREATE FUNCTION` doc](https://www.postgresql.org/docs/current/sql-createfunction.html). Because this Rust macro is used to generate SQL, and SQLs could depend on each other, this `requires = [ "shell_type" ]` is used to clarify this dependency relationship.

`shell_type` is the name of another SQL snippet that defines the shell type, what is shell type? It behaves like a placeholder so that we can have a `vector` type to use before we fully implement it. The SQL generated by this `#[pg_extern]` macro would be:

```sql
CREATE FUNCTION "vector_input"(
	"input" cstring,
	"_oid" oid,
	"type_modifier" INT
) RETURNS vector
```

As you can see, this function `RETURNS vector`, but how can we have a `vector` type before we implement these 4 required functions?

![circular_dependency](/images/posts/2024/build_a_vector_extension_for_postgres_vector_type/circular_dependency.png)

Shell type is exactly for this! We can define a shell type (A dummy type, no function needs to be provided), and let our functions depend on it:

![vector_type](/images/posts/2024/build_a_vector_extension_for_postgres_vector_type/vector_type.png)

`pgrx` won't define this shell type for us, we need to manually do this in SQL, here is what it looks like:

```rust
extension_sql!(
    r#"CREATE TYPE vector; -- shell type"#,
    name = "shell_type"
);
```

The [`extension_sql!()`](https://docs.rs/pgrx/latest/pgrx/macro.extension_sql.html) macro allows us to write SQL in Rust code, then `pgrx` will include it in the generated SQL script. The `name = "shell_type"` specifies this SQL snippet's identifier and can be used to refer to it. Our `vector_input()` function depends on this shell type, so it `requires = [ "shell_type" ]`.

```rust
fn vector_input(
    input: &CStr,
    _oid: pg_sys::Oid,
    type_modifier: i32,
) -> Vector {
```

The `input` argument is a string representing our vector input text, `_oid` is prefixed with `_` because we do not need it. The `type_modifier` argument is of type `i32`, this is how type modifier gets stored in Postgres. We will see it again when we implement the type modifier input/output functions.

```rust
let value = match serde_json::from_str::<Vec<f64>>(
    input.to_str().expect("expect input to be UTF-8 encoded"),
) {
    Ok(v) => v,
    Err(e) => {
        pgrx::error!("failed to deserialize the input string due to error {}", e)
    }
};
```

Then we convert `input` to a UTF-8 encoded `&str` and pass it to `serde_json::from_str()`. The input text should be UTF-8 encoded, so we should be safe. If any error happens during deserialization, just error out with [`pgrx::error!()`](https://docs.rs/pgrx/latest/pgrx/macro.error.html), which will log at the `error` level and terminate the current transaction.

```rust
let dimension = match u16::try_from(value.len()) {
    Ok(d) => d,
    Err(_) => {
        pgrx::error!("this vector's dimension [{}] is too large", value.len());
    }
};

// cast should be safe as dimension should be a positive
let expected_dimension = match u16::try_from(type_modifier) {
    Ok(d) => d,
    Err(_) => {
        panic!("failed to cast stored dimension [{}] to u16", type_modifier);
    }
};
```

The max dimension supported by us is `u16::MAX`, we do this simply because this is what pgvector does.

```rust
// check the dimension
if dimension != expected_dimension {
    pgrx::error!(
        "mismatched dimension, expected {}, found {}",
        expected_dimension,
        dimension
    );
}

Vector { value }
```

Lastly, we check if the input vector has the expected dimension, and error out if not. Otherwise, we return the parsed vector.

### `output_function`

The `output_function` performs the reverse action, it serializes the given vector to a string. Here is our implementation:

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_output(value: Vector) -> CString {
    let value_serialized_string = serde_json::to_string(&value).unwrap();
    CString::new(value_serialized_string).expect("there should be no NUL in the middle")
}
```

We just serialize the `Vec<f64>` and return it in a `CString`, quite simple.

### `type_modifier_input_function`

The `type_modifier_input_function` should parse the input modifier(s), check the parsed modifier(s), and if valid, encode it(them) into an integer, which is how Postgres stores type modifier.

A type can accept multiple type modifiers separated by `,`, this is the reason why we see an array here.

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_modifier_input(list: pgrx::datum::Array<&CStr>) -> i32 {
    if list.len() != 1 {
        pgrx::error!("too many modifiers, expect 1")
    }

    let modifier = list
        .get(0)
        .expect("should be Some as len = 1")
        .expect("type modifier cannot be NULL");
    let Ok(dimension) = modifier
        .to_str()
        .expect("expect type modifiers to be UTF-8 encoded")
        .parse::<u16>()
    else {
        pgrx::error!("invalid dimension value, expect a number in range [1, 65535]")
    };

    dimension as i32
}
```

The implementation is straightforward, the `vector` type takes only 1 modifier, so we verify if the array length is 1. And if yes, try to parse it to a `u16`, if no error happens, return it as an `i32` so that Postgres can store it.

### `type_modifier_output_function`

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_modifier_output(type_modifier: i32) -> CString {
    CString::new(format!("({})", type_modifier)).expect("no NUL in the middle")
}
```

`type_modifier_output_function` implementation is also simple, just return a string in format `(type_modifier)`.

All the required functions are implemented, now we can finally `CREATE` it!

```rust
// create the actual type, specifying the input and output functions
extension_sql!(
    r#"
CREATE TYPE vector (
    INPUT = vector_input,
    OUTPUT = vector_output,
    TYPMOD_IN = vector_modifier_input,
    TYPMOD_OUT = vector_modifier_output,
    STORAGE = external
);
"#,
    name = "concrete_type",
    creates = [Type(Vector)],
    requires = [
        "shell_type",
        vector_input,
        vector_output,
        vector_modifier_input,
        vector_modifier_output
    ]
);
```

You may be unfamiliar with the `STORAGE = external` argument. Let me give a brief introduction here. The default storage engine used by Postgres is heap, with this engine, a table's the on-disk file is split into pages, whose size defaults to 8192 bytes, Postgres hopes that at least 4 rows can be stored in a page, if some columns are too big so that 4 rows cannot be accommodated, Postgres moves them to an external table, which is call a TOAST table. This `STORAGE` argument controls Postgres's behavior when moving columns into TOAST tables. Setting it to `external` means this column can be moved to the TOAST table if it is too huge.

## Our first launch!

Looks like everything is ready, let's do our first launch!

```sh
$ cargo pgrx run
error[E0277]: the trait bound `for<'a> fn(&'a CStr, Oid, i32) -> Vector: FunctionMetadata<_>` is not satisfied
  --> pg_extension/src/vector_type.rs:87:1
   |
86 | #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
   | --------------------------------------------------------------------------- in this procedural macro expansion
87 | fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector {
   | ^^ the trait `FunctionMetadata<_>` is not implemented for `for<'a> fn(&'a CStr, Oid, i32) -> Vector`
   |
   = help: the following other types implement trait `FunctionMetadata<A>`:
             `unsafe fn() -> R` implements `FunctionMetadata<()>`
             `unsafe fn(T0) -> R` implements `FunctionMetadata<(T0,)>`
             `unsafe fn(T0, T1) -> R` implements `FunctionMetadata<(T0, T1)>`
             `unsafe fn(T0, T1, T2) -> R` implements `FunctionMetadata<(T0, T1, T2)>`
             `unsafe fn(T0, T1, T2, T3) -> R` implements `FunctionMetadata<(T0, T1, T2, T3)>`
             `unsafe fn(T0, T1, T2, T3, T4) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4)>`
             `unsafe fn(T0, T1, T2, T3, T4, T5) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4, T5)>`
             `unsafe fn(T0, T1, T2, T3, T4, T5, T6) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4, T5, T6)>`
           and 25 others
   = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0599]: the function or associated item `entity` exists for struct `Vector`, but its trait bounds were not satisfied
  --> pg_extension/src/vector_type.rs:86:1
   |
22 | pub(crate) struct Vector {
   | ------------------------ function or associated item `entity` not found for this struct because it doesn't satisfy `Vector: SqlTranslatable`
...
86 | #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ function or associated item cannot be called on `Vector` due to unsatisfied trait bounds
   |
   = note: the following trait bounds were not satisfied:
           `Vector: SqlTranslatable`
           which is required by `&Vector: SqlTranslatable`
note: the trait `SqlTranslatable` must be implemented
  --> $HOME/.cargo/registry/src/index.crates.io-6f17d22bba15001f/pgrx-sql-entity-graph-0.12.9/src/metadata/sql_translatable.rs:73:1
   |
73 | pub unsafe trait SqlTranslatable {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   = help: items from traits can only be used if the trait is implemented and in scope
   = note: the following traits define an item `entity`, perhaps you need to implement one of them:
           candidate #1: `FunctionMetadata`
           candidate #2: `SqlTranslatable`
   = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `Vector: RetAbi` is not satisfied
  --> pg_extension/src/vector_type.rs:87:73
   |
87 | fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector {
   |                                                                         ^^^^^^ the trait `BoxRet` is not implemented for `Vector`, which is required by `Vector: RetAbi`
   |
   = help: the following other types implement trait `BoxRet`:
             &'a CStr
             &'a [u8]
             &'a str
             ()
             AnyArray
             AnyElement
             AnyNumeric
             BOX
           and 33 others
   = note: required for `Vector` to implement `RetAbi`

error[E0277]: the trait bound `Vector: RetAbi` is not satisfied
   --> pg_extension/src/vector_type.rs:87:80
    |
86  |   #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
    |   --------------------------------------------------------------------------- in this procedural macro expansion
87  |   fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector {
    |  ________________________________________________________________________________^
88  | |     let value = match serde_json::from_str::<Vec<f64>>(
89  | |         input.to_str().expect("expect input to be UTF-8 encoded"),
90  | |     ) {
...   |
110 | |     Vector { value }
111 | | }
    | |_^ the trait `BoxRet` is not implemented for `Vector`, which is required by `Vector: RetAbi`
    |
    = help: the following other types implement trait `BoxRet`:
              &'a CStr
              &'a [u8]
              &'a str
              ()
              AnyArray
              AnyElement
              AnyNumeric
              BOX
            and 33 others
    = note: required for `Vector` to implement `RetAbi`
    = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `fn(Vector) -> CString: FunctionMetadata<_>` is not satisfied
   --> pg_extension/src/vector_type.rs:114:1
    |
113 | #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
    | --------------------------------------------------------------------------- in this procedural macro expansion
114 | fn vector_output(value: Vector) -> CString {
    | ^^ the trait `FunctionMetadata<_>` is not implemented for `fn(Vector) -> CString`
    |
    = help: the following other types implement trait `FunctionMetadata<A>`:
              `unsafe fn() -> R` implements `FunctionMetadata<()>`
              `unsafe fn(T0) -> R` implements `FunctionMetadata<(T0,)>`
              `unsafe fn(T0, T1) -> R` implements `FunctionMetadata<(T0, T1)>`
              `unsafe fn(T0, T1, T2) -> R` implements `FunctionMetadata<(T0, T1, T2)>`
              `unsafe fn(T0, T1, T2, T3) -> R` implements `FunctionMetadata<(T0, T1, T2, T3)>`
              `unsafe fn(T0, T1, T2, T3, T4) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4)>`
              `unsafe fn(T0, T1, T2, T3, T4, T5) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4, T5)>`
              `unsafe fn(T0, T1, T2, T3, T4, T5, T6) -> R` implements `FunctionMetadata<(T0, T1, T2, T3, T4, T5, T6)>`
            and 25 others
    = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0599]: the function or associated item `entity` exists for struct `Vector`, but its trait bounds were not satisfied
   --> pg_extension/src/vector_type.rs:113:1
    |
22  | pub(crate) struct Vector {
    | ------------------------ function or associated item `entity` not found for this struct because it doesn't satisfy `Vector: SqlTranslatable`
...
113 | #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ function or associated item cannot be called on `Vector` due to unsatisfied trait bounds
    |
    = note: the following trait bounds were not satisfied:
            `Vector: SqlTranslatable`
            which is required by `&Vector: SqlTranslatable`
note: the trait `SqlTranslatable` must be implemented
   --> $HOME/.cargo/registry/src/index.crates.io-6f17d22bba15001f/pgrx-sql-entity-graph-0.12.9/src/metadata/sql_translatable.rs:73:1
    |
73  | pub unsafe trait SqlTranslatable {
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    = help: items from traits can only be used if the trait is implemented and in scope
    = note: the following traits define an item `entity`, perhaps you need to implement one of them:
            candidate #1: `FunctionMetadata`
            candidate #2: `SqlTranslatable`
    = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `Vector: ArgAbi<'_>` is not satisfied
   --> pg_extension/src/vector_type.rs:114:18
    |
113 | #[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
    | --------------------------------------------------------------------------- in this procedural macro expansion
114 | fn vector_output(value: Vector) -> CString {
    |                  ^^^^^ the trait `ArgAbi<'_>` is not implemented for `Vector`
    |
    = help: the following other types implement trait `ArgAbi<'fcx>`:
              &'fcx T
              &'fcx [u8]
              &'fcx str
              *mut FunctionCallInfoBaseData
              AnyArray
              AnyElement
              AnyNumeric
              BOX
            and 36 others
note: required by a bound in `pgrx::callconv::Args::<'a, 'fcx>::next_arg_unchecked`
   --> $HOME/.cargo/registry/src/index.crates.io-6f17d22bba15001f/pgrx-0.12.9/src/callconv.rs:931:41
    |
931 |     pub unsafe fn next_arg_unchecked<T: ArgAbi<'fcx>>(&mut self) -> Option<T> {
    |                                         ^^^^^^^^^^^^ required by this bound in `Args::<'a, 'fcx>::next_arg_unchecked`
    = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)
```

Wow, that is a lot of compile errors at the first glance, but if we take a closer look, there are only 3 errors, let's fix them one by one:)

## Fix the errors!

### the trait bound `for<'a> fn(&'a CStr, Oid, i32) -> Vector: FunctionMetadata<_>` is not satisfied

Well, the message of this error is quite messy, but there is indeed one useful note:

```sh
note: the trait `SqlTranslatable` must be implemented
  --> $HOME/.cargo/registry/src/index.crates.io-6f17d22bba15001f/pgrx-sql-entity-graph-0.12.9/src/metadata/sql_translatable.rs:73:1
   |
73 | pub unsafe trait SqlTranslatable {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   = help: items from traits can only be used if the trait is implemented and in scope
   = note: the following traits define an item `entity`, perhaps you need to implement one of them:
           candidate #1: `FunctionMetadata`
           candidate #2: `SqlTranslatable`
   = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)
```

This note says that we should implement `SqlTranslatable` for our `Vector` type, what is trait for? Its doc says that this trait represents "A value which can be represented in SQL", which is kinda confusing. Let's me explain a bit. Since pgrx generates SQL for us, and it would use our `Vector` type, it requires us to tell it what our type should be called in SQL, a.k.a., translate it to SQL.

In SQL, our `Vector` type is simply called `vector`, so here is our implementation:

```rust
unsafe impl SqlTranslatable for Vector {
    fn argument_sql() -> Result<SqlMapping, ArgumentError> {
        Ok(SqlMapping::As("vector".into()))
    }

    fn return_sql() -> Result<Returns, ReturnsError> {
        Ok(Returns::One(SqlMapping::As("vector".into())))
    }
}
```

### the trait bound `Vector: RetAbi` and `Vector: ArgAbi` are not satisfied

Our `Vector` type has its memory representation (how it gets laid out in memory), and Postgres has its own way (or rule) to store things in memory. The `vector_input()` function returns a `Vector` to Postgres, and the `vector_output()` function takes a `Vector` argument, which will be supplied by Postgres:

```rust
fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector
fn vector_output(value: Vector) -> CString
```

Using `Vector` as function return value and argument requires it to be representable following Postgres's rule. Postgres uses [`Datum`](https://stackoverflow.com/q/53543909/14092446), the binary representation for all the SQL types. So we need to provide a round trip between our `Vector` type and `Datum`, through these 2 traits.

Since our `Vector` type is basically a `std::vec::Vec<f64>`, and these 2 traits are implemented for `Vec`, we can simply use these implementations:

```rust
impl FromDatum for Vector {
    unsafe fn from_polymorphic_datum(
        datum: pg_sys::Datum,
        is_null: bool,
        typoid: pg_sys::Oid,
    ) -> Option<Self>
    where
        Self: Sized,
    {
        let value = <Vec<f64> as FromDatum>::from_polymorphic_datum(datum, is_null, typoid)?;
        Some(Self { value })
    }
}

impl IntoDatum for Vector {
    fn into_datum(self) -> Option<pg_sys::Datum> {
        self.value.into_datum()
    }

    fn type_oid() -> pg_sys::Oid {
        rust_regtypein::<Self>()
    }
}

unsafe impl<'fcx> ArgAbi<'fcx> for Vector
where
    Self: 'fcx,
{
    unsafe fn unbox_arg_unchecked(arg: ::pgrx::callconv::Arg<'_, 'fcx>) -> Self {
        unsafe { arg.unbox_arg_using_from_datum().expect("expect it to be non-NULL") }
    }
}

unsafe impl BoxRet for Vector {
    unsafe fn box_into<'fcx>(
        self,
        fcinfo: &mut pgrx::callconv::FcInfo<'fcx>,
    ) -> pgrx::datum::Datum<'fcx> {
        match self.into_datum() {
            Some(datum) => unsafe {fcinfo.return_raw_datum(datum)}
            None => fcinfo.return_null()
        }
    }
}
```

`FromDatum` and `IntoDatum` are also implemented because they will be used in the `RetAbi` and `ArgAbi` implementations.

Let's see if there are any errors:

```sh
$ cargo check
$ echo $?
0
```

Great, no errors, let's launch it again!

## Our second launch!

```sh
$ cargo pgrx run
    ...
psql (17.2)
Type "help" for help.

pg_vector_ext=#
```

Yes, `psql` starts without any issue! Let's enable our extension and create a table with a `vector` column:

```sql
pg_vector_ext=# CREATE EXTENSION pg_vector_ext;
CREATE EXTENSION
pg_vector_ext=# CREATE TABLE items (v vector(3));
CREATE TABLE
```

So far so good, now let's insert some data:

```sql
pg_vector_ext=# INSERT INTO items values ('[1, 2, 3]');
ERROR:  failed to cast stored dimension [-1] to u16
LINE 1: INSERT INTO items values ('[1, 2, 3]');
                                  ^
DETAIL:
   0: std::backtrace_rs::backtrace::libunwind::trace
             at /rustc/f6e511eec7342f59a25f7c0534f1dbea00d01b14/library/std/src/../../backtrace/src/backtrace/libunwind.rs:116:5
             ...
  46: <unknown>
             at main.c:197:3
```

Well, Postgres panicked, because the `type_modifier` argument is `-1`, which can not be casted into a `u16`. But how can the `type_modifier` be `-1`, wouldn't it be `-1` if and only if the type modifier is unknown? The type modifier `3` is clearly stored in the metadata, so it is definitely known.

I was stuck here for a while, and guess what, [I am not the only one](https://www.postgresql.org/message-id/56EA3507.6090701@anastigmatix.net). IMHO, this is a bug, but not everyone thinks so. Let's accept the truth that it just can be `-1`, the workaround for this is to create a cast function that casts `Vector` to `Vector`, where you can access the stored type modifier:

```rust
/// Cast a `vector` to a `vector`, the conversion is meaningless, but we do need
/// to do the dimension check here if we cannot get the `typmod` value in vector
/// type's input function.
#[pgrx::pg_extern(immutable, strict, parallel_safe, requires = ["concrete_type"])]
fn cast_vector_to_vector(vector: Vector, type_modifier: i32, _explicit: bool) -> Vector {
    let expected_dimension = u16::try_from(type_modifier).expect("invalid type_modifier") as usize;
    let dimension = vector.value.len();
    if vector.value.len() != expected_dimension {
        pgrx::error!(
            "mismatched dimension, expected {}, found {}",
            type_modifier,
            dimension
        );
    }

    vector
}

extension_sql!(
    r#"
    CREATE CAST (vector AS vector)
    WITH FUNCTION cast_vector_to_vector(vector, integer, boolean);
    "#,
    name = "cast_vector_to_vector",
    requires = ["concrete_type", cast_vector_to_vector]
);
```

In our `vector_input()` function, no dimension check will be performed if the type modifier is unknown:

```rust
#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector {
    let value = match serde_json::from_str::<Vec<f64>>(
        input.to_str().expect("expect input to be UTF-8 encoded"),
    ) {
        Ok(v) => v,
        Err(e) => {
            pgrx::error!("failed to deserialize the input string due to error {}", e)
        }
    };
    let dimension = match u16::try_from(value.len()) {
        Ok(d) => d,
        Err(_) => {
            pgrx::error!("this vector's dimension [{}] is too large", value.len());
        }
    };

    // check the dimension in INPUT function if we know the expected dimension.
    if type_modifier != -1 {
        let expected_dimension = match u16::try_from(type_modifier) {
            Ok(d) => d,
            Err(_) => {
                panic!("failed to cast stored dimension [{}] to u16", type_modifier);
            }
        };

        if dimension != expected_dimension {
            pgrx::error!(
                "mismatched dimension, expected {}, found {}",
                expected_dimension,
                dimension
            );
        }
    }
    // If we don't know the type modifier, do not check the dimension.

    Vector { value }
}
```

Now everything should really work:

```sql
pg_vector_ext=# INSERT INTO items values ('[1, 2, 3]');
INSERT 0 1
```

`INSERT` works, let's try `SELECT`:

```sql
pg_vector_ext=# SELECT * FROM items;
       v
---------------
 [1.0,2.0,3.0]
(1 row)
```

Congratulation! You just added the `vector` type support for Postgres! Here is the full code of our implementation:

> src/vector_type.rs

```rust
//! This file defines a `Vector` type.

use pgrx::callconv::ArgAbi;
use pgrx::callconv::BoxRet;
use pgrx::datum::FromDatum;
use pgrx::datum::IntoDatum;
use pgrx::extension_sql;
use pgrx::pg_extern;
use pgrx::pg_sys;
use pgrx::pgrx_sql_entity_graph::metadata::ArgumentError;
use pgrx::pgrx_sql_entity_graph::metadata::Returns;
use pgrx::pgrx_sql_entity_graph::metadata::ReturnsError;
use pgrx::pgrx_sql_entity_graph::metadata::SqlMapping;
use pgrx::pgrx_sql_entity_graph::metadata::SqlTranslatable;
use pgrx::wrappers::rust_regtypein;
use std::ffi::CStr;
use std::ffi::CString;

/// The `vector` type
#[derive(Debug, serde::Deserialize, serde::Serialize)]
#[serde(transparent)]
pub(crate) struct Vector {
    pub(crate) value: Vec<f64>,
}

unsafe impl SqlTranslatable for Vector {
    fn argument_sql() -> Result<SqlMapping, ArgumentError> {
        Ok(SqlMapping::As("vector".into()))
    }

    fn return_sql() -> Result<Returns, ReturnsError> {
        Ok(Returns::One(SqlMapping::As("vector".into())))
    }
}

impl FromDatum for Vector {
    unsafe fn from_polymorphic_datum(
        datum: pg_sys::Datum,
        is_null: bool,
        typoid: pg_sys::Oid,
    ) -> Option<Self>
    where
        Self: Sized,
    {
        let value = <Vec<f64> as FromDatum>::from_polymorphic_datum(datum, is_null, typoid)?;
        Some(Self { value })
    }
}

impl IntoDatum for Vector {
    fn into_datum(self) -> Option<pg_sys::Datum> {
        self.value.into_datum()
    }

    fn type_oid() -> pg_sys::Oid {
        rust_regtypein::<Self>()
    }
}

unsafe impl<'fcx> ArgAbi<'fcx> for Vector
where
    Self: 'fcx,
{
    unsafe fn unbox_arg_unchecked(arg: ::pgrx::callconv::Arg<'_, 'fcx>) -> Self {
        unsafe {
            arg.unbox_arg_using_from_datum()
                .expect("expect it to be non-NULL")
        }
    }
}

unsafe impl BoxRet for Vector {
    unsafe fn box_into<'fcx>(
        self,
        fcinfo: &mut pgrx::callconv::FcInfo<'fcx>,
    ) -> pgrx::datum::Datum<'fcx> {
        match self.into_datum() {
            Some(datum) => unsafe { fcinfo.return_raw_datum(datum) },
            None => fcinfo.return_null(),
        }
    }
}

extension_sql!(
    r#"
CREATE TYPE vector; -- shell type
"#,
    name = "shell_type",
    bootstrap // declare this extension_sql block as the "bootstrap" block so that it happens first in sql generation
);

#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_input(input: &CStr, _oid: pg_sys::Oid, type_modifier: i32) -> Vector {
    let value = match serde_json::from_str::<Vec<f64>>(
        input.to_str().expect("expect input to be UTF-8 encoded"),
    ) {
        Ok(v) => v,
        Err(e) => {
            pgrx::error!("failed to deserialize the input string due to error {}", e)
        }
    };
    let dimension = match u16::try_from(value.len()) {
        Ok(d) => d,
        Err(_) => {
            pgrx::error!("this vector's dimension [{}] is too large", value.len());
        }
    };

    // check the dimension in INPUT function if we know the expected dimension.
    if type_modifier != -1 {
        let expected_dimension = match u16::try_from(type_modifier) {
            Ok(d) => d,
            Err(_) => {
                panic!("failed to cast stored dimension [{}] to u16", type_modifier);
            }
        };

        if dimension != expected_dimension {
            pgrx::error!(
                "mismatched dimension, expected {}, found {}",
                expected_dimension,
                dimension
            );
        }
    }
    // If we don't know the type modifier, do not check the dimension.

    Vector { value }
}

#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_output(value: Vector) -> CString {
    let value_serialized_string = serde_json::to_string(&value).unwrap();
    CString::new(value_serialized_string).expect("there should be no NUL in the middle")
}

#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_modifier_input(list: pgrx::datum::Array<&CStr>) -> i32 {
    if list.len() != 1 {
        pgrx::error!("too many modifiers, expect 1")
    }

    let modifier = list
        .get(0)
        .expect("should be Some as len = 1")
        .expect("type modifier cannot be NULL");
    let Ok(dimension) = modifier
        .to_str()
        .expect("expect type modifiers to be UTF-8 encoded")
        .parse::<u16>()
    else {
        pgrx::error!("invalid dimension value, expect a number in range [1, 65535]")
    };

    dimension as i32
}

#[pg_extern(immutable, strict, parallel_safe, requires = [ "shell_type" ])]
fn vector_modifier_output(type_modifier: i32) -> CString {
    CString::new(format!("({})", type_modifier)).expect("no NUL in the middle")
}

// create the actual type, specifying the input and output functions
extension_sql!(
    r#"
CREATE TYPE vector (
    INPUT = vector_input,
    OUTPUT = vector_output,
    TYPMOD_IN = vector_modifier_input,
    TYPMOD_OUT = vector_modifier_output,
    STORAGE = external
);
"#,
    name = "concrete_type",
    creates = [Type(Vector)],
    requires = [
        "shell_type",
        vector_input,
        vector_output,
        vector_modifier_input,
        vector_modifier_output
    ]
);

/// Cast a `vector` to a `vector`, the conversion is meaningless, but we do need
/// to do the dimension check here if we cannot get the `typmod` value in vector
/// type's input function.
#[pgrx::pg_extern(immutable, strict, parallel_safe, requires = ["concrete_type"])]
fn cast_vector_to_vector(vector: Vector, type_modifier: i32, _explicit: bool) -> Vector {
    let expected_dimension = u16::try_from(type_modifier).expect("invalid type_modifier") as usize;
    let dimension = vector.value.len();
    if vector.value.len() != expected_dimension {
        pgrx::error!(
            "mismatched dimension, expected {}, found {}",
            type_modifier,
            dimension
        );
    }

    vector
}

extension_sql!(
    r#"
    CREATE CAST (vector AS vector)
    WITH FUNCTION cast_vector_to_vector(vector, integer, boolean);
    "#,
    name = "cast_vector_to_vector",
    requires = ["concrete_type", cast_vector_to_vector]
);
```
