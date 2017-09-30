---
layout: post
title: How to paginate Rails find_by_sql result
date: 30-09-2017
comments: true
categories: [rails, paginate, find_by_sql]
tags: [rails, paginate, find_by_sql]
excerpt_separator: <!-- more -->
---

For Rails developers, we are all familiar with pagination. I am using and happy with Kaminari to do pagination. However, there is a problem, quite a serious problem with Kaminari is that I could not use Kaminari to perform pagination on the famous ``find_by_sql`` function as ``find_by_sql`` simply returns a Ruby array while Kaminari requires an ActiveRecord::Relation in order to append the pagination query portion. In this article, let's go through how I still can do pagination on ``find_by_sql`` without using Kaminari.

<!-- more -->

## Why find_by_sql?

I love ``ActiveRecord``, ``ActiveRecord::Relation`` and its chainability. I would definitely prefer to use that instead of the very dry ``find_by_sql``. The reason is that with ``ActiveRecord::Relation``, everything can become magic. I could chain ``where`` condition, add Kaminari pagination just by simply do:

```ruby
# get the 1st page of 10 users who are created 7 days ago
User
  .where('created_at >= ?', 7.days_ago)
  .page(1)
  .per(10)
```

However, there are certain times that we could not avoid using ``find_by_sql``, simply because not all SQL qsyntax are supported by Rails ``ActviveRecord`` like ``UNION`` in the below example:

```sql
-- query all name of quizzes and surveys

SELECT name AS name, 'quizzes' AS entity_type
FROM quizzes
UNION
SELECT title AS name, 'survey' AS  entity_type
FROM surveys
```

For this particular case, to get out the data from Rails, we would need to use ``find_by_sql`` or ``ActiveRecord::Base.connection.execute``. Let's say we use ``find_by_sql``, we could normalize all the data into Quiz model like below:

```ruby
sql = %(
  SELECT name AS name, 'quizzes' AS entity_type
  FROM quizzes
  UNION
  SELECT title AS name, 'survey' AS  entity_type
  FROM surveys
)

quizzes = Quiz.find_by_sql(sql)
```

The result ``quizzes`` will return __a Ruby Array__ of ``Quiz`` objects in which each object has only 2 attributes ``name`` and ``entity_type``. So now we have a background of why would need to use ``find_by_sql``. Let's move on to the possible solutions of how to paginate this __Ruby Array__ result.

## Paginate find_by_sql result

### __1. Naive solution__ (time and memory inefficient)

An obvious solution that we can think of immediately is paginating the __Ruby Array__. Kamiari __does__ provide a function to peform this like below:

```ruby
# ...
Kaminari.paginate_array(quizzes).page(1).per(10)
```

However, I would strongly recommend our developers to stay away from this function if the data is from ``find_by_sql``. The simple reason is that what would happen if the result of ``find_by_sql`` is an array of __1000__ or more records, i.e. 1000 records have been loaded into memory. ``Kaminari.paginate_array`` would simply cut the 10 records based on the suppy page number and get rid of the rest of 9990 records. It is quite a wasted resources of loading 1000 records into memory and yet we would only consume 10 of them.

This inheritly would consume more time by querying out 1000 records and load these records into memory. We would need to think a way that it should only query the 10 records that we need to consume only based on the current page.

### __2. Real solution__ (both time and memory efficient)

To have time and memory efficient, we would have to think of a way to query from the database only the records of the page that we are looking for only. In order to do that, we would need to inject pagination function into the original SQL query. To do that, we would need to wrap our original SQL to something like below:

```sql
-- query all name of quizzes and surveys

SELECT * FROM (
  SELECT name AS name, 'quizzes' AS entity_type
  FROM quizzes
  UNION
  SELECT title AS name, 'survey' AS  entity_type
  FROM surveys
) AS paginatable
```

Then to query only the records of the current page, we would need to use 2 SQL functions ``LIMIT`` and ``OFFSET``. The SQL ``LIMIT`` function would limit the number of records returned, which would be equivalent to our ``per_page`` value. Whreas the SQL ``OFFSET`` function would tell database the location of the first record that we would get. Note that, OFFSET is like index, it starts at 0. We could write a function to return the OFFSET based on the ``page` and ``per_page`` value like below:

```ruby
def offset(page, per_page)
  (page - 1) * per_page
end
```

After that we can inject in ``LIMIT`` and ``OFFSET`` like below:

```ruby
def query_report(page: 1, per_page: 10)
  sql = %(
    SELECT * FROM (
      SELECT name AS name, 'quizzes' AS entity_type
      FROM quizzes
      UNION
      SELECT title AS name, 'survey' AS  entity_type
      FROM surveys
    ) AS paginatable
    LIMIT :limit OFFSET :offset
  )

  Quiz.find_by_sql(
    [
      sql,
      {
        limit: per_page,
        offset: offset(page, per_page)
      }
    ]
  )
end
```

With this, we have managed to query only the records in the current page. However, this is not yet the end of the story. In order to have a proper pagination view, we would need to know the number of pages and the total number of records (if we want to display the text: Viewing 10 out of 200 records). To do that, we would need to do an SQL count for the total records returning from the original query. Below is a possible implementation:

```ruby
# original SQL
def query_report_sql
  @query_report_sql ||=
    %(
      SELECT name AS name, 'quizzes' AS entity_type
      FROM quizzes
      UNION
      SELECT title AS name, 'survey' AS  entity_type
      FROM surveys
    )
end

# paginatable SQL
def query_report_paginate_sql
  @query_report_paginate_sql ||=
    %(
      SELECT *
      FROM (#{query_report_sql}) AS paginatable
      LIMIT :limit OFFSET :offset
    )
end

# count all records SQL
def query_report_total_count_sql
  @query_report_total_count_sql ||=
    %(
      SELECT COUNT(*) AS count
      FROM (#{query_report_sql}) AS paginatable
    )
end

def query_report(page: 1, per_page: 10)
  records =
    Quiz.find_by_sql(
      [
        query_report_paginate_sql,
        {
          limit: per_page,
          offset: offset(page, per_page)
        }
      ]
    )

  records
    .instance_variable_set(:@per_page, per_page)
  records
    .instance_variable_set(:@query_report_total_count_sql,
                           query_report_total_count_sql)

  add_pagination_methods(records)

  records
end

def add_pagination_methods(records)
  records.instance_eval do
    def total_count
      @total_count ||=
        Quiz
        .find_by_sql(@query_report_total_count_sql)
        .first
        .count
    end

    def total_pages
      @total_pages ||= (total_count * 1.0 / @per_page).ceil.to_i
    end
  end
end

private

def offset(page, per_page)
  (page - 1) * per_page
end
```

In the first 3 methods, we would just outline the 3 queries that we would need. In the method ``query_report``, we would first find the records for the current page like what we did before. However, this time, we would also add ``per_page`` and ``query_report_total_count_sql`` as the attribute of the returned ``records`` sot that these can be used in the ``total_count`` and ``total_pages`` functions. Hence, only when we call ``total_pages`` or ``total_count``, it will trigger the query to get the total pages and total acount.

With this, we have managed to find all the essential information to perform a proper pagination rendering.
