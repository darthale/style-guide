# SQL Style Guide and Best Practices

SQL has a very simple set of rules, and does not have advanced structure. While there are many flavors of SQL, this guide will in particular focus on generic SQL92, with the belief that the simple definitions herein will apply to all implementations.

Because SQL is not turing complete, and is straight-forward, it is often ignored in terms of good style and ways of working within an organization. This style guide is intended to provide best-practices (through lessons learned) as well as style that makes it easiest to communicate changes and achieve better maintainability of SQL.

## Table of Contents

* [Source Code Layout](#source-code-layout)
* [Syntax](#syntax)
* [Naming](#naming)
* [Comments](#comments)
* [Temporary Tables](#tmp-tables)
* [Safety](#safety)
* [Text Search](#text-search)
* [Misc](#misc)

## Source Code Layout

> Nearly everybody is convinced that every style but their own is
> ugly and unreadable. Leave out the "but their own" and they're
> probably right... <br/>
> -- Jerry Coffin (on indentation)

* Use `UTF-8` or `latin1` as the source file encoding (try to match it to the database).
* Use two **spaces** per indentation level.
* Seperate things onto multiple lines.
* Capitalize all keywords

    ```SQL
    -- Good
    SELECT
      a,
      b,
      c
    FROM
      my_table;

    -- Bad - four spaces
    SELECT
        a,
        b,
        c
    FROM
        my_table;

    -- Bad - all in one line, lowercase keywords
    select a,b,c from my_table;
    ```

* Use spaces around operators.

    ```SQL
    SELECT
      a * b AS result
    FROM
      my_table
    WHERE
      b > 3;
    ```

* Seperate CASE statements as you would in other languages
  * Here you can choose whether WHEN is at the same indentation level as CASE or one greater.

    ```SQL
    SELECT
      CASE
        WHEN a = 4 THEN "good"
        WHEN a = 3 THEN "OK"
        ELSE "bad"
      END
    FROM
      my_table;
    ```

* Keep ANDs on the same line, JOINS on new lines

  ```SQL
  SELECT
    my_table.a,
    second_table.d
  FROM
    my_table
    INNER JOIN second_table ON my_table.value = second_table.value AND my_table.value_2 = second_table.value_2;
  ```

* Keep parenthesis on the same line when doing nested JOINs

  ```SQL
  SELECT
    my_table.a,
    t.d
  FROM
    my_table
    INNER JOIN (
      SELECT
        d
      FROM
        another_table
        INNER JOIN third_table ON another_table.value = third_table.value
      WHERE
        another_table.some_date > '2014-01-01'
    ) t
  GROUP BY
    my_table.a,
    t.d;
  ```

* Limit lines to 120 characters where possible. This might mean splitting up `JOIN`s, windowing, etc.
  ```SQL
  -- Good
  SELECT
    my_table.a,
    second_table.b
  FROM
    my_table
    INNER JOIN second_table ON
      my_table.attr_1 = second_table.attr_1 AND
      my_table.attr_2 = second_table.attr_2 AND
      my_table.attr_3 = second_table.attr_3 AND
      my_table.attr_4 = second_table.attr_4
  GROUP BY
    my_table.a,
    second_table.b

  -- Good
  SELECT
    imps.user_id AS user,
    imps.campaign_uid AS campaign,
    page_visits.engagement_eid,
    page_visits.activity_name,
    imps.event_id AS impression_eid,
    page_visits.source_timestamp-imps.source_timestamp AS delta,
    RANK() OVER (
        PARTITION BY
            page_visits.engagement_eid
        ORDER BY
            page_visits.source_timestamp - imps.source_timestamp
    ) AS hops
  FROM
      rwh.rpt_impressions imps
      INNER JOIN (
        SELECT
          activity.event_id AS engagement_eid,
          homepage.activity_name,
          activity.created_dt,
          activity.source_timestamp,
          activity.user_id
        FROM
          rwh.rpt_activities activity
          INNER JOIN centralized_engagement homepage ON activity.pixel_id = homepage.pixel_id
      ) page_visits ON imps.user_id = page_visits.user_id AND
        imps.created_dt >= page_visits.created_dt - 30 AND
        imps.source_timestamp < page_visits.source_timestamp
  WHERE
      imps.created_dt BETWEEN '2014-11-03' AND '2014-12-31'
      AND imps.campaign_uid = 'ACampaignUID'
  ```

* Avoid trailing whitespace (Remember `whitespace`)

## Syntax

* Fully qualify all tables (do not rely on `search_path`)
* Use the order of operations with the least number of operations

  ```SQL
  -- Bad
  sum(rpt.microDollarsToDollars(raw_spend))

  -- Good
  rpt.microDollarsToDollars(sum(raw_spend))
  ```

* Prefer existing functions for clarity.

  ```SQL
  -- Bad
  round(sum(raw_spend) / 1000000, 2)

  -- Good
  round(rpt.microDollarsToDollars(sum(raw_spend)), 2)
  ```

* Define new functions only when the task is repeatable. The more repeatable the more a new function is warranted.
* In `GROUP BY` statements use the fully qualified column name in place of numbers where possible.
* In any query (especially within nested `JOIN`s) avoid reusing names or aliases

## Naming

* Use `snake_case` for table and column names, but `CamelCase` for function/methods
* Use `SCREAMING_SNAKE_CASE` for other constants.
* Don't use short (1 or 2 char) names
* Do not reuse names, especially across scopes, and make them descriptive

## Comments

* Write commments where appropriate, don't be shy, but avoid superfluous comments.
* Keep existing comments up-to-date. Favor explanatory commit messages instead.
* Never push commented out code to master.
* Remove ALL comments for DB code that is to be installed; putting comments in functions may BREAK the function

## Temporary Tables
* If you would use a temporarily defined table more than once within a query drop and define a temporary table explicitly

  ```SQL
  -- Bad
  with tmp_table AS (
    SELECT
      value,
      value2
    FROM
      expensive_table
    WHERE
      conditions = 'conditions'
  );

  -- Good
  DROP TABLE IF EXISTS tmp_table;
  CREATE TEMP TABLE tmp_table AS
    SELECT
      value,
      value2
    FROM
      expensive_table
    WHERE
      conditions = 'conditions';
  ```

## Safety

* The ```coalesce``` keyword is there to default NULL values, use it!
* Use the ELSE branch in your CASE statement
* Watch out for division by 0, use a CASE statement to avoid this!
* In general use fully qualified names, whether on tables, functions, or aliases:
  * Don't rely on `search_path` (as above), use fully qualified tables and functions
  * When using aliases always use fully qualified names


## Text Search

Dealing with string search can be tricky. Instead of talking about benchmarks I would instead point you to http://blog.2ndquadrant.com/text-search-strategies-in-postgresql/ which provides great insight into the contexts in which you should use different kinds of string search.

## Misc
