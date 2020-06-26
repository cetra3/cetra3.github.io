+++

title = "Implementing a Job queue with SQLx and Postgres"
description = "An example of async postgres in rust"
date = 2020-06-26

[taxonomies]
tags = ["rust", "tmq", "sqlx"]
+++

**SQLx** is a new [async SQL Toolkit](https://github.com/launchbadge/sqlx) for rust that is closer to standard SQL than a more opinionated ORM like [Diesel](http://diesel.rs/). I wanted to give it a bit of a test run and see how easy it would be to convert usage from [tokio-postgres](https://github.com/sfackler/rust-postgres).  So as the next saga in the jobq series ([part 1](../implementing-a-jobq/), [part 2](../implementing-a-jobq-with-tokio/)), we will be converting [jobq](https://github.com/cetra3/jobq) crate to use `sqlx`.  You can find the *SQLx* branch here: [https://github.com/cetra3/jobq/tree/sqlx](https://github.com/cetra3/jobq/tree/sqlx)

As a little spoiler: I found it quite easy to adjust and a pleasure to use but found the documentation a little lacking.

## Object Relational Mappers (ORMs)

**ORMs** definitely serve a purpose. *ORMs* provide an opinionated way to manage the database schema and craft queries in an easy way.  Normally you construct queries using builders, or other language constructs, and get given the results back straight in your language types (structs in the case of rust).  Migration is a lot easier as most offer a managed way to run migrations for you.  You don't need to know SQL either, and can simply run functions in your language to build up queries to the database.

Writing raw SQL in some ORMs is a second class citizen, which is where I have a problem.  I personally find the cognitive load of having to wrangle the builder pattern to pull out SQL is not worth it.  I am quite comfortable writing SQL and have done for many years, and so I have steered away from using highly opinionated ORMs in the past.  In the world of Java I much prefer [MyBatis](https://mybatis.org/mybatis-3/) or [Jdbi](http://jdbi.org/) over [Hibernate](https://hibernate.org/).  In Rust, I am using standard [tokio-postgres](https://github.com/sfackler/rust-postgres) over [diesel](http://diesel.rs/) for the same reasons (that: and diesel async support is not there yet and [doesn't look like it will be](https://github.com/diesel-rs/diesel/issues/399) anytime soon...).

Now if you are not familiar with SQL this is definitely a different story, and may find you have a different opinion on how productive you are.  That's fine too, as this is one reason ORMs exist: to make it easier to code.  I would still recommend spending time grokking SQL, as it is quite a great skill to have in your toolbox.

### Diesel ORM

Picking on [diesel](http://diesel.rs/) for a second: here is their *Complex Queries* example:

```rust
let versions = Version::belonging_to(krate)
  .select(id)
  .order(num.desc())
  .limit(5);
let downloads = version_downloads
  .filter(date.gt(now - 90.days()))
  .filter(version_id.eq(any(versions)))
  .order(date)
  .load::<Download>(&conn)?;
```

Here is the same query in SQL:

```sql
SELECT version_downloads.*
  WHERE date > (NOW() - '90 days')
    AND version_id = ANY(
      SELECT id FROM versions
        WHERE crate_id = 1
        ORDER BY num DESC
        LIMIT 5
    )
  ORDER BY date
```

Which one do you find more readable? I find the SQL easier to read.

### SQLx ORM

I still consider `SQLx` an ORM, but with SQL as a first-class citizen.

Here's how you'd do the same with **SQLx**:

```rust
let query = "SELECT version_downloads.*
  WHERE date > (NOW() - '90 days')
    AND version_id = ANY(
      SELECT id FROM versions
        WHERE crate_id = $1
        ORDER BY num DESC
        LIMIT 5
    )
  ORDER BY date";


sqlx::query_as(query)
    .bind(&num)
    .fetch_all(&*self.pool);
```

If your `Version` struct derived `sqlx::FromRow`, then this would all work nicely.

## Converting to SQLx

Converting to SQLx was a mostly painless experience, and only [very minimal changes](https://github.com/cetra3/jobq/compare/051b2c2a89fbed0a67bc989c4d3bf29a745ab3ce...sqlx) were needed for the jobq code base.  In the end, only the structs & db files were needed to be adjusted, but I could reuse all of the query strings and sql building logic.

As I am not starting with SQLx, some of the touted features I didn't use (like [compile time verification](https://github.com/launchbadge/sqlx#compile-time-verification)).  There were also a few gotchas I ran into, but did not find it hard to work through them after a bit of reading.

### Migrations and Raw SQL

I haven't implemented migrations properly in jobq: it's just a [simple sql script](https://github.com/cetra3/jobq/blob/sqlx/src/setup.sql) that is run at startup.  SQLx does have [utilities for this](https://github.com/launchbadge/sqlx/tree/master/sqlx-cli), and is also quite strong on [compile time verification](https://github.com/launchbadge/sqlx#compile-time-verification) to make this more type safe.

**However:** I just want to run the script at startup.  It's simple & I can do it from standard postgres cli, so I should be able to it simply from rust as well.  The `sqlx::query` method will convert the SQL to a prepared statement, which does not play nice with such a query:

```rust
sqlx::query(include_str!("setup.sql")).execute(&pool).await?;
```

Here's the error you may get:

```
cannot insert multiple commands into a prepared statement
```

Instead, you can use the [`Executor` trait](https://docs.rs/sqlx/0.3.5/sqlx/trait.Executor.html) which is implemented for `&PgPool`.  This does make it a little awkward to write having to have the extra brackets:

```rust
(&pool).execute(include_str!("setup.sql")).await?;
```

### Mapping a postgresql query to rust structs

sqlx has a `query_as` method which allows you to convert returned result rows into rust structs, a bit like serde is used to serialize/deserialize data.

This works by deriving [`sqlx::FromRow`](https://docs.rs/sqlx/0.3.5/sqlx/derive.FromRow.html) on your structs:

```rust
#[derive(Serialize, Deserialize, Debug, Clone, sqlx::FromRow)]
pub struct Job {
    pub id: i64,
    pub username: String,
    pub name: String,
    pub uuid: Uuid,
    pub params: Value,
    pub priority: Priority,
    pub status: Status,
}
```

As I am using [custom enums](https://github.com/cetra3/jobq/blob/f4441203fd4b4d2d73d5f7a91b228723c7330aa6/src/setup.sql#L2-L9), I also need to derive [`sqlx::Type`](https://docs.rs/sqlx/0.3.5/sqlx/derive.Type.html) on them:

```rust
#[derive(Serialize, Deserialize, Debug, Clone, sqlx::Type)]
pub enum Status {
    Queued,
    Processing,
    Completed,
    Failed,
}
```
(*Side note: custom enums may not be a [hot idea for maintainability](https://softwareengineering.stackexchange.com/questions/305148/why-would-you-store-an-enum-in-db)*)

With that I can do away with the old [`get_jobs`](https://github.com/cetra3/jobq/blob/051b2c2a89fbed0a67bc989c4d3bf29a745ab3ce/src/db.rs#L53-L79) method, as this is handled for me now by SQLx.

I can then use the [`fetch_all`](https://docs.rs/sqlx/0.3.5/sqlx/postgres/trait.PgQueryAs.html#tymethod.fetch_all) method to return is as a `Vec`:

```rust
pub(crate) async fn get_processing_jobs(&self) -> Result<Vec<Job>, Error> {
    let query = "select id, name, username, uuid, params, priority, status from jobq
                 where status = 'Processing' order by priority asc, time asc";

    Ok(sqlx::query_as(query).fetch_all(&*self.pool).await?)
}
```

Worthy of note: the [`fetch`](https://docs.rs/sqlx/0.3.5/sqlx/postgres/trait.PgQueryAs.html#tymethod.fetch) method returns a futures `Stream` to iterate through, making a much tighter integration with async

### Binding Parameters

I didn't need to change SQL statements at all, but did need to adjust some of the binding parameters using the `bind` builder pattern:

```rust
pub(crate) async fn complete_job(&self, id: i64) -> Result<(), Error> {
    let query = "update jobq set status = 'Completed', duration = extract(epoch from now() - \"time\") where id = $1";

    sqlx::query(query).bind(&id).execute(&*self.pool).await?;

    Ok(())
}
```

This is different to `tokio-postgres` accepting an array of parameters.  I have ran into issues before when trying to pass dynamic arguments and trying to wrangle life times.  Here is how I have worked around this in the past (not exactly pretty):

```rust
for row in self.pool.get()?.query(
    &*query,
    params
        .iter()
        .map(|val| &**val as &(dyn ToSql + Sync))
        .collect::<Vec<_>>()
        .as_slice(),
)?
```

## Conclusions

As you can see from the [diff of the commits](https://github.com/cetra3/jobq/compare/051b2c2a89fbed0a67bc989c4d3bf29a745ab3ce...sqlx), the conversion to use SQLx was an easy excercise.  I have found SQLx quite a bit more lightweight and more to my liking than some of the heavier ORMs I've dealt with in the past.  Being async first means that this will be a great fit for the ecosystem: I can see it becoming the defacto async story for databases in the future.

One area which may need some improvement is the documentation & examples.  As it is a comparitively young library with the [first commit a year ago](https://github.com/launchbadge/sqlx/commit/d29b24abf0f491d912333f34d7b5187e4c30bb08), I assume that this will only get better with time.  The [README](https://github.com/launchbadge/sqlx)  and [api docs](https://docs.rs/sqlx) were enough for me to get into trouble.

If you are looking to talk to a database with async rust, make sure you consider SQLx!