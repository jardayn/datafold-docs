---
title: How to Implement a New Database Driver
---

First, read through the [CONTRIBUTING.md](https://github.com/datafold/data-diff/blob/master/CONTRIBUTING.md) document.

Make sure data-diff is set up for development, and that all the tests pass (try to at least set it up for mysql and postgresql).

Look at the other database drivers for examples and inspiration.


## 1. Add dependencies to ``pyproject.toml``

Most new drivers will require a 3rd party library in order to connect to the database.

These dependencies should be specified in the `pyproject.toml` file, in `[tool.poetry.extras]`. Example:

```
    [tool.poetry.extras]
    postgresql = ["psycopg2"]
```

Then, users can install the dependencies needed for your database driver, with `pip install 'data-diff[postgresql]'`.

This way, data-diff can support a wide variety of drivers, without requiring our users to install libraries that they won't use.

## 2. Implement database module

New database modules belong in the `data_diff/databases` directory.

### Import on demand

Database drivers should not import any 3rd party library at the module level.

Instead, they should be imported and initialized within a function. Example:

```
    from .base import import_helper

    @import_helper("postgresql")
    def import_postgresql():
        import psycopg2
        import psycopg2.extras

        psycopg2.extensions.set_wait_callback(psycopg2.extras.wait_select)
        return psycopg2
```

We use the `import_helper()` decorator to provide a uniform and informative error. The string argument should be the name of the package, as written in `pyproject.toml`.

### Choosing a base class, based on threading Model

You can choose to inherit from either `base.Database` or `base.ThreadedDatabase`.

Usually, databases with cursor-based connections, like MySQL or Postgresql, only allow one thread per connection. In order to support multithreading, we implement them by inheriting from `ThreadedDatabase`, which holds a pool of worker threads, and creates a new connection per thread.

Usually, cloud databases, such as Snowflake and BigQuery, open a new connection per request, and support simultaneous queries from any number of threads. In other words, they already support multithreading, so we can implement them by inheriting directly from `Database`.


### `_query()`

All queries to the database pass through `_query()`. It takes SQL code, and returns a list of rows. Here is its signature:

```
    def _query(self, sql_code: str) -> list: ...
```

For standard cursor connections, it's sufficient to implement it with a call to `base._query_conn()`, like:

```
        return _query_conn(self._conn, sql_code)
```

### `select_table_schema()` / `query_table_schema()`

If your database does not have a `information_schema.columns` table, or if its structure is unusual, you may have to implement your own `select_table_schema()` function, which returns the query needed to return column information in the form of a list of tuples, where each tuple is _column_name, data_type, datetime_precision, numeric_precision, numeric_scale_.

If such a query isn't possible, you may have to implement `query_table_schema()` yourself, which extracts this information from the database, and returns it in the proper form.

If the information returned from `query_table_schema()` requires slow or error-prone post-processing, you may delay that post-processing by overriding `_process_table_schema()` and implementing it there. The method `_process_table_schema()` only gets called for the columns that will be diffed.

Documentation:

- `data_diff.databases.database_types.AbstractDatabase.select_table_schema()`

- `data_diff.databases.database_types.AbstractDatabase.query_table_schema()`

### `TYPE_CLASSES`

Each database class must have a `TYPE_CLASSES` dictionary, which maps between the string data-type, as returned by querying the table schema, into the appropriate data-diff type class, i.e. a subclass of `database_types.ColType`.

### `ROUNDS_ON_PREC_LOSS`

When providing a datetime or a timestamp to a database, the database may lower its precision to correspond with the target column type.

Some databases will lower precision of timestamp/datetime values by truncating them, and some by rounding them.

`ROUNDS_ON_PREC_LOSS` should be True if this database rounds, or False if it truncates.

### `__init__()`, `create_connection()`

The options for the database connection will be given to the `__init__()` method as keywords.

If you inherit from `Database`, your `__init__()` method may create the database connection.

If you inherit from `ThreadedDatabase`, you should instead create the connection in the `create_connection()` method.

### `close()`

If you inherit from `Database`, you will need to implement this method to close the connection yourself.

If you inherit from `ThreadedDatabase`, you don't have to implement this method.

Docs:

- `data_diff.databases.database_types.AbstractDatabase.close()`

### `quote()`, `to_string()`, `normalize_number()`, `normalize_timestamp()`, `md5_to_int()`

These methods are used when creating queries.

They accept an SQL code fragment, and returns a new code fragment representing the appropriate computation.

For more information, read their docs:

- `data_diff.databases.database_types.AbstractDatabase.quote()`

- `data_diff.databases.database_types.AbstractDatabase.to_string()`

## 3. Add tests

Add your new database to the `DATABASE_TYPES` dict in `tests/test_database_types.py`

The key is the class itself, and the value is a dict of {category: [type1, type2, ...]}

Categories supported are: `int`, `datetime`, `float`, and `uuid`.

Example:

```
    DATABASE_TYPES = {
        ...
        db.PostgreSQL: {
            "int": [ "int",  "bigint" ],
            "datetime": [
                "timestamp(6) without time zone",
                "timestamp(3) without time zone",
                "timestamp(0) without time zone",
                "timestamp with time zone",
            ],
            ...
        },
```

Then run the tests and make sure your database driver is being tested.

You can run the tests with `unittest`.

To save time, we recommend running them with `unittest-parallel`.

When debugging, we recommend using the `-f` flag, to stop on error. Also, use the `-k` flag to run only the individual test that you're trying to fix.

## 4. Create Pull-Request


Open a pull request [on github](https://github.com/datafold/data-diff), and we'll take it from there!

