---
layout: default
description: >-
  You can configure collections to generate document attributes when documents
  are created or modified, using an AQL expression
---
# Computed Values

{{ page.description }}
{:class="lead"}

<small>Introduced in: v3.10.0</small>

If you want to add default values to new documents, maintain auxiliary
attributes for search queries, or similar, you can set these attributes manually
in every operation that inserts or modifies documents. However, it is more
convenient and reliable to let the database system take care of it.

The computed values feature lets you define expressions on the collection level
that run on inserts, modifications, or both. You can access the data of the
current document to compute new values using a subset of AQL. The result of each
expression becomes the value of a top-level attribute. These attributes are
stored like any other attribute, which means that you may use them in indexes,
but also that the schema validation is applied if you use one.

Possible use cases are to add timestamps of the creation or last modification to
every document, or to automatically combine multiple attributes into one,
possibly with filtering and a conversion to lowercase characters, to then index
the attribute and use it to perform case-insensitive searches.

## JavaScript API

Computed values are defined per collection, either when creating the collection
or by modifying the collection later on. The collection property is called
`computedFields` and accepts an array of objects.

`db._create(<collection-name>, { computedFields: [ { … }, … ] })`

`db.<collection-name>.properties({ computedFields: [ { … }, … ] })`

Each object represents a computed value and can have the following attributes:

- `name` (string, _required_):
  The name of the target attribute. Can only be a top-level attribute, but you
  may return a nested object. Cannot be `_key`, `_id`, `_rev`, `_from`, `_to`,
  or a shard key attribute.

- `expression` (string, _required_):
  An AQL `RETURN` operation with an expression that computes the desired value.
  See [Computed Value Expressions](#computed-value-expressions) for details.

- `overwrite` (boolean, _required_):
  Whether the computed value shall take precedence over a user-provided or
  existing attribute.

- `computeOn` (array, _optional_):
  An array of strings to define on which write operations the value shall be
  computed. The possible values are `"insert"`, `"update"`, and `"replace"`.
  The default is `["insert", "update", "replace"]`.

- `keepNull` (boolean, _optional_):
  Whether the result of the expression shall be stored if it evaluates to `null`.
  This can be used to skip the value computation if any pre-conditions are not met.

If you add, remove, or modify computed value definitions at a later point, they
are only applied to subsequent write operations and older documents remain in
their state.

## Computed Value Expressions

You can use a subset of AQL for computed values, namely a single
[`RETURN` operation](aql/operations-return.html) with an expression that
computes the value. 

You can access the document data via the `@doc` bind variable. It contains the
data as it will be stored, including the `_key`, `_id`, and `_rev`
system attributes. On inserts, you get the user-provided values (plus the
system attributes), and on modifications, you get the updated or replaced
document to work with.

Computed value expressions have the following requirements:

- The expression must start with a `RETURN` statement.

- No `FOR` loops, `LET` statements, and subqueries are allowed in the expression.
  `FOR` loops can be substituted using the [array expansion operator `[*]`](aql/advanced-array-operators.html#inline-expressions),
  for example, with an inline expressions like the following:

  `RETURN @doc.values[* FILTER CURRENT > 42 RETURN CURRENT * 2]`

- You cannot access any stored data other than the current document via the
  `@doc` bind parameter. AQL functions that read from the database system cannot
  be used in the expression (e.g. `DOCUMENT()`, `PREGEL_RESULT()`,
  `COLLECTION_COUNT()`).

- You cannot base a computed value on another. If you reference an attribute
  that results from another computed value, the value is implicitly `null`,
  like any access of non-existent attributes.

- You can use AQL functions in the expression but only those that can be
  executed on DB-Servers, regardless of your deployment type. The following
  functions cannot be used in the expression:
  - `CALL()`
  - `APPLY()`
  - `DOCUMENT()`
  - `V8()`
  - `SCHEMA_GET()`
  - `SCHEMA_VALIDATE()`
  - `VERSION()`
  - `COLLECTIONS()`
  - `CURRENT_USER()`
  - `CURRENT_DATABASE()`
  - `COLLECTION_COUNT()`
  - `NEAR()`
  - `WITHIN()`
  - `WITHIN_RECTANGLE()`
  - `FULLTEXT()`
  - [User-defined functions (UDFs)](aql/extending.html)

<!-- TODO
When using arangodump to restore data into a collection with computed attributes defined, the restore does not recalculate the computed values, but will restore the documents as they are contained inside the dump.

Values of computed attributes are properly replicated to followers. Followers will receive the documents as they are stored on the leader, including any computed attributes. Followers will not rerun the computed attributes computations themselves.

Non-deterministic computation expressions, such as RETURN RAND() will still work correctly, so that leaders and followers will have the same value for the computed attribute. This is achieved by leaders replicating the document data including the computed attributes values, and the followers not carrying out the computation again.
-->
