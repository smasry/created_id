# Created ID

[![Continuous Integration](https://github.com/bdurand/created_id/actions/workflows/continuous_integration.yml/badge.svg)](https://github.com/bdurand/created_id/actions/workflows/continuous_integration.yml)
[![Ruby Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://github.com/testdouble/standard)

The gem is designed to optimize queries for ActiveRecord models that filter by the `created_at` timestamp. It can make queries more efficient by pre-calculating the ranges of id's for specific dates.

The use case this code is designed to solve is when you have a large table with an auto-populated `created_at` column where you want to run queries that filter on that column. In most cases, simply adding an index on the `created_at` column will work just fine., However, once you start constructing more complex queries or adding joins and your table grows very large, the index can become less effective and not even be used at all.

For instance, suppose you have a `Task` model backed by these tables:

```ruby
create_table :tasks do |t|
  t.string :status, index: true
  t.bigint, :user_id, index: true
  t.datetime :created_at, index: true
  t.string :description
end

create_table :users do |t|
  t.string :name
  t.string :group_name, index: true
end

class Task < ApplicationRecord
  belongs_to :user
end

class User < ApplicationRecord
  has_many :tasks
end
```

And now suppose you want to count the tasks completed by users in the "public" group within the last day:

```ruby
Task.joins(:users)
  .where(status: "completed", users: { group_name: "public" })
  .where(created_at: [24.hours.ago...Time.current])
  .count
```

This will construct a SQL query like this:

```sql
SELECT COUNT(*)
FROM tasks
INNER JOIN users ON users.id = tasks.user_id
WHERE tasks.status = 'completed'
  AND users.group_name = 'public'
  AND tasks.created_at >= ?
  AND tasks.created_at < ?
```

The query optimizer will have it's choice of several indexes to use to figure out the best query plan. The most important choice will be the first step of the plan to reduce the number of rows that the query needs to look at. Depending on the shape of your data, the query optimizer may decide to simply filter by `status` or `user_id` and then perform a table scan on all the rows to filter by `created_at`, not using the index on that column at all.

This gem solves for this case by keeping track of the range ids created in each hour in a separate table. When you query on the `created_at` column, it will then look up the possible id range and add that to the query, so the SQL becomes:

```sql
SELECT COUNT(*)
FROM tasks
INNER JOIN users ON users.id = tasks.user_id
WHERE tasks.status = 'completed'
  AND users.group_name = 'public'
  AND tasks.created_at >= ?
  AND tasks.created_at < ?
  AND tasks.id >= ?
  AND tasks.id < ?
```

Because the `id` column is the primary key, it will always be indexed and the query optimizer will generally make better decisions about how to filter the query rows. You won't even need the index on `created_at` since the primay key would always be preferred.

Another good use case is if you have some periodic tasks to calculate daily stats for some large tables. You will be able to make these queries more efficient without having to add an index on the `created_at` column that's only used on one query per day.

## Usage

First, include the `CreatedId` module into your models. Note that any model you wish to include this module in must have a numeric primary key and a `created_at` columm.

```ruby
class Task < ApplicationRecord
  include CreatedId

  belongs_to :user
end
```

Now when you want to query by a range on the `created_at` column, you can use the `created_after`, `created_before`, or `created_between` scopes on the model.

```ruby
Task.where(status: "completed").created_after(24.hours.ago)

Task.where(user_id: 1000).created_before(7.days.ago)

Task.created_between(25.hour.ago, 24.hours.ago)
```

You'll then need to set up a periodic task to store the id ranges for your models. For each model that includes `CreatedId`, you need to run the `index_ids_for` once per hour. This task should be run shortly after the top of the hour.

```ruby
Task.index_ids_for(1.hour.ago)
```

Finally, you'll need to run a script to calculate the id ranges for all of your existing data.

```ruby
first_time = Task.first(created_at).utc
time = Time.utc(first_time.year, first_time.month, first_time.date, first_time.hour)
while time < Time.now
  Task.index_ids_for(time)
  time += 3600
end
```

Don't worry if the id range for a specific hour does not get recorded, the queries will still work and they can be re-calcuated at any time. Queries will just be a bit less efficient if the ranges don't exist because queries will be given a large span of ids to filter on.

There is an additional requirement for using this gem that you do not change the `created_at` value after a row is inserted since this can mess up the assumption about the correlation between ids and `created_at` timestamps. An error will be thrown if you try to change a record's timestamp after the id range has been created. The query logic can handle small variations between id order and timestamp order (i.e. if id 1000 has a timestamp a few seconds after id 1001).

## Installation

Add this line to your application's Gemfile:

```ruby
gem "created_id"
```

And then execute:
```bash
$ bundle
```

Or install it yourself as:
```bash
$ gem install created_id
```

## Contributing

Open a pull request on GitHub.

Please use the [standardrb](https://github.com/testdouble/standard) syntax and lint your code with `standardrb --fix` before submitting.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
