# SuperFastRails

Most of the time, optimizing a Rails application requires repeating the same techniques.
For example, at the database layer, it's about creating the proper indexes, preventing 1+N queries, etc.
Could we do that automatically?

---

<img align="left" height="24px" src="rorvswild_logo.jpg" alt="RorVsWild logo"/>Made by <a href="https://www.rorvswild.com">RorVsWild</a>, performances & exceptions monitoring for Ruby on Rails applications.

---

## Introducing SuperFastRails!

We are releasing a gem to help developers write ultra-optimized code.
The goal is to allow developers to write code as fast as possible without caring about performance.
Rails scales; it's just a matter of writing the correct code.

*SuperFastRails* automatically improves the requests in your Rails application.
Thus, we focus only on the business logic and don't have to think about indexes, 1+n queries, dangerous migrations, etc.

For the first version, *SuperFastRails* takes good care of the database layer.
We want to keep adding more automatic optimizations in the future.
Here is the list of the current automatic optimizations.


## Create automatically missing indexes

Let's see the following query:

```sql
SELECT * FROM projects WHERE projects.user_id = ?
```

Without an index on user_id, the query planner scans the entire table, which is slow because it must go through all rows.
By enabling the following option:

```ruby
SuperFastRails.create_missing_indexes = 500.in_milliseconds
```

If the query takes longer than 500ms, SuperFastRails analyzes the query plan with `EXPLAIN` to detect if an index is missing.
In that case, it creates it to match the condition in the `where` clause.
Thus, when creating a new table in a migration, we don't have to think about which indexes to make because it does so automatically when the application is running.


## Remove unused indexes

Indexes speed up reads, but they slow down writes.
So, unused indexes are not good and should be removed.
By enabling the following options, it will automatically get rid of unused indexes:

```ruby
SuperFastRails.remove_unused_indexes = true
```

The combination of the two options `create_missing_indexes` and `remove_unused_indexes` ensures that the database always has relevant indexes, even if the where clauses change.


## Optimise automatically SQL

Indexes are essential, but SQL also needs to be written correctly.
For example, the following query:

```sql
SELECT *
FROM users
WHERE (SELECT count(*) FROM projects WHERE projects.user_id = users.id) > 1
```

is slower than:

```sql
SELECT *
FROM users
WHERE EXISTS (SELECT 1 FROM projects WHERE projects.user_id = users.id)
```

Because it stops once it finds a row instead of keep counting.
There are a bunch of tricks to know about SQL.
Learning them requires time, and we need to remember them.
The option `SuperFastRails.auto_optimise_queries = true` parses the SQL, and rewrites it before sending it to the database.
Thanks to it, we don't have to care about all these tricks.

You can also use it with a block if you prefer to enable it on a smaller scope:

```ruby
SuperFastRails.optimise_queries do
  User.where("(SELECT count(*) FROM projects WHERE user_id = users.id) > 1")
  # SELECT *
  # FROM users
  # WHERE EXISTS (SELECT 1 FROM projects WHERE user_id = users.id)
end

```


## Get rid of 1+N queries

The 1+N queries problem happens when iterating through a collection where the same query is repeated.
For the following example:

```ruby
Project.all.each do |project|
  puts "#{project.name} by #{project.user.name}"
end
```

It triggers one extra query for each project:

```sql
SELECT * FROM projects
SELECT * FROM users WHERE users.id = ?
SELECT * FROM users WHERE users.id = ?
SELECT * FROM users WHERE users.id = ?
SELECT * FROM users WHERE users.id = ?
SELECT * FROM users WHERE users.id = ?
SELECT * FROM users WHERE users.id = ?
-- And so on
```

To solve this, SuperFastRails adds a method `each_without_1_plus_n_queries` to ActiveRecord relations.
It detects when two identical queries are triggered to load all the missing data in a single query.

```ruby
Project.all.each_without_1_plus_n_queries do |project|
  puts "#{project.name} by #{project.user.name}"
end
```

```sql
SELECT * FROM projects
SELECT * FROM users WHERE users.id = ?
-- Before the 2nd repetition, superFastRails detects the 1+N pattern.
-- So it loads all relevant users in a single query.
SELECT * FROM users WHERE users.id IN (?)
-- No more queries
```



## Protect against dangerous migrations

There are already many gems that help to detect dangerous migrations.
They are great, but we still have to do the work manually.
For example, renaming a column must be achieved in many steps:

1. Create a new column
2. Backfill values
3. Synchronizing both columns
4. Switching the code to the new column
5. Ignoring the old column
6. Removing the old column

All this is tiring and wasting time.
Thanks to SuperFastRails, it can be achieved in a single step with only one line:

```ruby
SuperFastRails.rename_column :table, :old_name, :new_name
```

It takes care of creating the new columns, backfilling, and switching them.


## Tune database settings

A database must be tuned to efficiently use the hardware.
PostgreSQL should be told how much RAM it can use for the cache, the wall size, and the working memory.
It's the same for SQLite, where the journal mode, page size, and so on can be changed.

Currently, there is no other way to do that manually.
That's why we added a method that tunes perfectly the settings:

```ruby
SuperFastRails.tune_database!
```

It modifies the settings according to many parameters such as the database hardware, the ratio of reads and writes, the number of concurrent connections, etc.


## Install

Install the gem right today and speed up your app automatically:

```ruby
gem "super_fast_rails"
```
