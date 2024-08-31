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
Obviously, the true power of an ORM comes from being able to define
*models*, and then use those models in your code. Defining models
in this package is relatively straight forward.

Models are defined as structs, with a special tag of `postgres.orm.model`.
In practice, you will probably bring the `postgres.orm` package in directly,
so you can say `orm.model` instead. Here is an example.

```onyx
use postgres {orm}

@orm.model.{"user"}
User :: struct {
}
```

Here, the string provided to `orm.model` is the table name for the model.
Currently, this is required, but in the future the table name could be
automatically inferred by the name of the structure. If you separate your
tables into multiple schema, you can specify the schema name after the
table name. By default the schema name is `"public"`.

Inside of the structure, you can simply declare members, and they will be
mapped to columns with *almost* the same name in the database's table. The
only different between the member names and the columns names is that column
names are always all lowercase. The ORM simply does `str.to_lowercase` on
all member names before using them. Note that members that are pointers or
slices of pointers are skipped, as those are used for relationships.

```onyx
use postgres {orm}

@orm.model.{"user"}
User :: struct {
    id: u32;
    username: str;
    password: str;

    is_cool: bool;
}
```

While the above structure would work as a model, it is probably a good idea
to add some metadata about the columns to help the database understand your
intensions. Below is a list of tags you can add to members.

### @orm.primary_key
- Specifies the primary key for the model.
- Currently, models can only have at most 1 primary key, and models without
    a primary key cannot be automatically saved or deleted.

### @orm.not_null
- Specifies the column cannot contain null.
- This is enforced on the database by adding a `NOT NULL` constraint to the
    column.

### @orm.nullable
- Specifies the column can contain null.
- This is the default behavior.

### @orm.unique
- Specifies the column's value should not match any other row's value for
    that column.
- This is enforced on the database by adding a `UNIQUE` constraint to the
    column.

### @orm.auto_increment
- Specifies that the integer should auto increment, if not specified.
- This can only be applied to `u16`, `u32`, and `u64` typed members.
- This is achieved by changing the type in the database to `smallserial`,
    `serial`, and `bigserial` respectively.

### @orm.constraint.{name, cond}
- Adds a constraint to the table, generally referring to the column
    following this constraint.

### @orm.default.{value}
- Specifies the default for the column.
- Generally this is only used when the value is not a static value.

### @orm.foreign_key.{foreign_model, referenced_key, on_delete, on_update}
- `foreign_model` is the ORM model structure.
- `referenced_key` is optional. Defaults to the primary key of `foreign_model`.
- `on_delete` and `on_update` define what happens when the foreign model is changed.
    There are 5 possible values:
    - `.No_Action`: default behavior
    - `.Restrict`: prevent the model being updated/deleted
    - `.Cascade`: also update/delete this key
    - `.Set_Null`: set this key to null
    - `.Set_Default`: set this key to the default value


A more complete version of the example above could be:
```onyx
use postgres {orm}

@orm.model.{"user"}
User :: struct {
    @orm.primary_key
    @orm.auto_increment
    id: u32;

    @orm.unique,orm.not_null
    username: str;

    @orm.not_null
    password: str;

    is_cool: bool = false;
}
```

Notice that you can also specify default values for column using
the normal default value syntax from Onyx.

When you have an instance of a model, you can save or delete it easily
using `ctx->save(instance)` or `ctx->delete(instance)` respectively.
This only works if the model has a primary key.

## Defining Relationships
In addition to the aforementioned tags you can place on members,
you can use tags to define relationships between models. When defined,
the ORM can automatically populate the members with other values.

There are 4 relationship types that the ORM supports:
- Belongs to
- Has one
- Has Many
- Many to many

With each relationship type, the foreign model is inferred from the
member type. For example if you have a member that has the type: `^User`,
the foreign model type is inferred to be `User`.

### Belongs to
This association creates a one-to-one link with another model. With a
`belongs to` relationship, the foreign key is stored on the *owned* object,
NOT the *owner*. `@orm.belongs_to` should only be applied on a member that is
a pointer to a model. `orm.belongs_to` has one mandatory member, `local_key`
which the column to use from the same structure that references the primary
key in the foreign model. If you do not want to use the primary key, you
can specify the `foreign_key` to the column you with to use.

```onyx
@orm.model.{"user"}
User :: struct {
    @orm.primary_key @orm.auto_increment
    id: u32;

    @orm.unique
    username: str;
    //...
}

@orm.model.{"user_settings"}
Settings :: struct {
    @orm.primary_key @orm.auto_increment
    id: u32;

    // orm.foreign_key is not required here, but is good practice to specify.
    @orm.foreign_key.{User}
    user_id: u32;

    // "user_id" refers to which column to use as the foreign key.
    @orm.belongs_to.{"user_id"}
    user: ^User;

    // This is equivalent, as "id" is the primary key of "user".
    // @orm.belongs_to.{local_key="user_id", foreign_key="id"}
    // user: ^User;
}
```

### Has one
This association also creates a one-to-one link with another model.
It is the opposite of `belongs_to`. To add on the example from before,
a `has_one` relationship could be append to the user struct.

```onyx
@orm.model.{"user"}
User :: struct {
    // same as before ...

    // "user_id" refers to the foreign column name which should match
    // the primary key of this table.
    @orm.has_one.{"user_id"}
    settings: ^Settings;

    // Just like belongs_to, you can change the key refered to locally.
    // Notice that argument names are "backwards" from before, as this
    // is the converse relationship.
    // @orm.has_one.{foreign_key="user_id", local_key="id"}
    // settings: ^Settings;
}
```

### Has many
This association creates a one-to-many link between models.
It behaves just like `has_one`, except the type of the member should
be a slice of pointers, instead of a single pointer.

```onyx
@orm.model.{"user"}
User :: struct {
    // same as in belongs_to ...

    // "user_id" refers to the foreign column name which should match
    // the primary key of this table.
    @orm.has_many.{"user_id"}
    settings: [] ^Settings;
}
```

### Many to many
This association is very different from the rest. It creates a many-to-many
link between two models, using an intermediate model (table). Currently,
you have to manually create this intermediate model, but that may change in
the future.

`orm.many_to_many` takes 3 mandatory arguments:
- The mapping model
- The column name on that model that should match the primary key of the local model.
- The column name on that model that should match the primary key of the foreign model.

This is best explained through an example:

```onyx
@orm.model.{"user"}
User :: struct {
    @orm.primary_key @orm.auto_increment
    id: u32;

    // ...

    // The following tag says:
    // To lookup the teams this user is a part of, query the TeamUser
    // model where "user_id" matches my primary_key. With those results
    // query Team where the team's primary_key is in the list of "team_id"s.
    @orm.many_to_many.{TeamUser, "user_id", "team_id"}
    teams: [] ^Team;
}

@orm.model.{"team"}
Team :: struct {
    @orm.primary_key @orm.auto_increment
    id: u32;

    // ...

    // Notice that this is backwards from above, because the ORM
    // must first query based on matching "team_id", then on matching
    // "user_id".
    @orm.many_to_many.{TeamUser, "team_id", "user_id"}
    users: [] ^User;
}

@orm.model.{"team_user"}
TeamUser :: struct {
    // Notice that this structure does not have a primary key.
    // This is not an issue in this case, but if this model had
    // more data and you wanted to use the `ctx->save(obj)` and
    // `ctx->delete(obj)` methods, you would not be able.

    // Good practice to delete the link when the either foreign
    // model is deleted.

    @orm.foreign_key.{User, on_delete=.Cascade}
    user_id: u32;

    @orm.foreign_key.{User, on_delete=.Cascade}
    team_id: u32;
}
```

To make working with many-to-many models easier, there are two
convienence methods provided on the ORM context, `link(a, b)` and `unlink(a, b)`.
Simply give `link` or `unlink` two objects that are in a many to many relation,
and the ORM will automatically create/destroy an instance of the linking model
between those two objects. `link` also returns the newly created instance. Provided
your intermediate model has a primary key, you can then modify the other properties
on the instance and save it.

