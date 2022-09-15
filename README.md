# Postgres Object Relational Mapper

This document describes the abilities and common use cases
for using this package in an Onyx project.

## Basic Usage


## Issuing Queries
Queries are generated automatically using the query builder object.
To get an instance of the query builder object, simply call the
query() method on your ORMContext global variable, and provide it
with the table you are querying.

```onyx
use postgres {orm}

@orm.model.{"user"}
User :: struct {
    @orm.primary_key @orm.auto_increment
    id: u32;

    @orm.not_null
    username: str;
}

db := orm.create(connection);

// This creates a query builder for Users.
query := db->query(User);
```

Once you have a query builder, there are several methods you can
use to define your query:

### filter
Use `filter()` to limit your query results. There are two overloads
currently for filter.

- `filter(str)` - This string is directly added to the `WHERE` clause in
    the SQL statement.

- `filter(format, ..any)` - Allows for conveniently formatting a filter
    directly in place. It is roughly equivalent to `filter(tprintf(format, ....))`;

### order
Use `order()` to define the order of your query results. There is
only one overload for order at the moment.

- `order(field_name, .ASC or .DESC)` - Order by the field named `field_name`,
    either in ascending (`.ASC`) or descending (`.DESC`) order.

`order()` can be called multiple times to define a multi-layered radix sort.

### group
Use `group()` to define if multiple rows should be grouped into one.
Usually this is combined with `select()` to specify the aggregation functions
for each column. Group is has the same overloads as `filter()`.

### select
Use `select()` to change which columns are being selected. By default this is
`*` representing all columns. You would want to use this when you know there
is only a subset of things you need, and want to reduce the amount of traffic
needed to retrieve the object.

- `select(str)` - Specifies the select query columns.

### having
Use `having()` to specify the `HAVING` clause in a select statement. `HAVING`
is a weird quirk of SQL that behaves much like `WHERE` (`filter()`), but allows
for aggregation functions. To not break backwards compatibility with existing
implementation, `HAVING` was added. `having()` has the same overloads as `filter()`
and `group()`.

### limit
Limits the number of queries to be received, using the `LIMIT` SQL clause.

- `limit(number)`

### offset
Skips the first entries in the result, using the `OFFSET` SQL clause.

- `offset(number)` - Skips the `number` results.


The above methods allow you to filter and refine your query, but do not actually
issue an query. To do this, you have to end your query with one of the follow methods.

### all
`all()`: Returns all results as an array

### first
`first()`: Returns only the first result as an object.
*Effectively, `->limit(1)->all()[0]`*

### find
`find(primary_key)`: Returns the first result where the primary key is `primary_key`.
*Effectively, `->filter("pk={}", primary_key)->first()`*

### delete
`delete()`: Deletes ALL entries where the query matches.

### update
`update(column, value)`: Updates a single column on all matching rows to be `value`.


The query builder object was designed to compose very easily. Because of this,
you can chain query calls together into one expression. Here are some examples:

```onyx
db->query(User)->filter("last_logged_in < {}", '2022-01-01')->all();
// SELECT * FROM "user" where last_logged_in < '2022-01-01';

db->query(Player)->order("score", .DESC)->limit(5)->all();
// SELECT * FROM "player" order by score desc limit 5;
```


## Defining Models


## Defining Relationships


