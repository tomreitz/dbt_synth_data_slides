[comment]: # (controls: true)
[comment]: # (keyboard: true)
[comment]: # (markdown: { smartypants: true })
[comment]: # (hash: false)
[comment]: # (respondToHashChanges: false)



<code style="color:#0eb9a3;">dbt_synth_data</code><br />
<small>a `dbt` package for creating synthetic data</small>

<hr style="border-width:1px 0 0 0;" />

<!-- <strong>Tom Reitz</strong> &nbsp; &nbsp;  -->
<div style="font-size:26px; line-height:56px;">Data Engineering @ <img src="media/ea_logo.png" style="height:48px; width:auto; margin:0; padding:0 10px; vertical-align:middle;" /></div>



[comment]: # (!!! data-auto-animate)

Two approaches for creating fake data

[comment]: # (||| data-auto-animate)

![disguise](https://media.giphy.com/media/1ziDTlTl9z9iwVK5QA/giphy.gif)

de-identify real data &rarr; possibly "fuzz" some values

<em>can be suscpetible to re-identification</em>  <!-- .element: class="fragment" data-fragment-index="2" -->

[comment]: # (||| data-auto-animate)

![sculpting](https://media.giphy.com/media/1BeG0fUaEQtVns1uKb/giphy.gif)

start with nothing &rarr; synthesize data by describing it

<em>safer, but more work</em>  <!-- .element: class="fragment" data-fragment-index="2" -->

<div>&uarr; <code style="color:#0eb9a3;">dbt_synth_data</code> does this</div>  <!-- .element: class="fragment" data-fragment-index="3" -->



[comment]: # (!!! data-auto-animate)

### motivation

Other tools ([Faker](https://faker.readthedocs.io/en/master/) for Python, [Mimesis](https://mimesis.name/en/master/) for Postgres) exist for creating synthetic data... why build another?

* **cross-platform:** write code once, run on<br />Snowflake, Postgres, SQLite...
* **performance:** creating 100B rows with Faker takes months, Snowflake can do it in ~1 hour


[comment]: # (!!! data-auto-animate)

### how it works

[comment]: # (||| data-auto-animate)

most SQL flavors can `generate()` rows:

```sql
-- Snowflake:
select
	row_number() over (order by 1) as rownum
from table(generator( rowcount => 100 ));

-- Postgres:
select
	s.rownum as rownum
from generate_series( 1, 100 ) as s(rownum);
```

[comment]: # (||| data-auto-animate)

produces...

| rownum |
|---|
| 1 |
| 2 |
| 3 |
| ⋮ |
| 100 |

[comment]: # (||| data-auto-animate)

most SQL flavors can generate `random()` numbers:

```sql
-- Snowflake:
select random();
--> int between -BIGINT_MIN : +BIGINT_MAX
select uniform(0::float, 1::float, random());
--> float between 0.0 : 1.0

-- Postgres
select random();
--> float between 0.0 : 1.0
```

[comment]: # (||| data-auto-animate)

put these together to generate rows of randomness:

```sql
-- Snowflake:
select
	row_number() over (order by 1) as rownum,
  uniform(0::float, 1::float, random()) as randval
from table(generator( rowcount => 100 ));

-- Postgres:
select
	s.idx as rownum,
  random() as randval
from generate_series( 1, {{rows}} ) as s(idx);
```

[comment]: # (||| data-auto-animate)

produces...

| rownum | randval |
|---|---|
| 1 | 0.01736233267 |
| 2 | 0.8667546837 |
| 3 | 0.4433875307 |
| ⋮ | ⋮ |

[comment]: # (||| data-auto-animate)

other (non-uniform) distributions are possible too

[comment]: # (||| data-auto-animate)

normal distribution:

``` sql
-- Snowflake:
select
  row_number() over (order by 1) as rownum,
  normal(1::float, 1::float, random()) as normal_randval
  --     ^ mean    ^ stddev  ^ rand
from table(generator( rowcount => 100 ));

-- Postgres:
select
  s.idx as rownum,
  ( ( 1.0 * sqrt(-2*log(random()))*sin(2*pi()*random()) ) + 1.0 ) as normal_randval
  --  ^ stddev          ^ rand 1              ^ rand 2      ^ mean
from generate_series( 1, {{rows}} ) as s(idx);
```

[comment]: # (||| data-auto-animate)

exponential distribution:

``` sql
-- Snowflake:
select
	row_number() over (order by 1) as rownum,
  -ln( abs ( uniform(0::float, 1::float, random()) ) ) * (1/1.0) as exp_randval
  --         ^ uniform rand                                 ^ lambda
from table(generator( rowcount => 100 ));
```


[comment]: # (!!! data-auto-animate)

![come on](https://media.giphy.com/media/4ZrFRwHGl4HTELW801/giphy.gif)

precise syntax varies by SQL flavor

[comment]: # (||| data-auto-animate)

![nodding](https://media.giphy.com/media/OF4PIvoHuO2ze/giphy.gif)

dbt to the rescue

[comment]: # (||| data-auto-animate)

<b>dbt</b> (data build tool)
* used extensively by EA's data engineering team,<br />and by data engineers generally
* compiles and runs SQL
* allows for abstraction, writing macros (functions) that produce SQL
* abstract across different SQL flavors/engines

[comment]: # (||| data-auto-animate)

example:
```sql
{{ synth_distribution_continuous_normal(mean=0, stddev=1) }}
```
compiles to
```
normal(0.0::float, 1.0::float, random())
-- when run on Snowflake

(
  (
    1.0::float
    * sqrt( -2 * log( random() ) )
    * sin( 2 * pi() * random() ) 
  ) + 0.0::float
) -- when run on Postgres
```

[comment]: # (||| data-auto-animate)

![normal distribution](media/continuous_normal.png)

<small>(histogram with 1M values)</small>



[comment]: # (!!! data-auto-animate)

<code style="color:#0eb9a3;">dbt_synth_data</code> uses CTEs<br />(common table expressions)<br />extensively to deal with two problems:

[comment]: # (||| data-auto-animate)

![really?](https://media.giphy.com/media/Pn1gZzAY38kbm/giphy.gif)

**problem 1:** SQL engines "optimize away"<br />randomness inside subqueries

[comment]: # (||| data-auto-animate)

```sql
-- select a random district
select
  ceil(uniform(125::float, 175::float, random())) as rand_lea,
  (
    select k_lea
    from db.schema.int_leas
    where lea_id=ceil(uniform(125::float, 175::float, random()))
    limit 1
  ) as k_lea
from table(generator( rowcount => 100 ));
```

[comment]: # (||| data-auto-animate)

produces

| rand_lea | k_lea |
|---|---|
| 165 | f8de717cfb7ed59631852d1c7f63bc70 |
| 156 | f8de717cfb7ed59631852d1c7f63bc70 |
| 139 | f8de717cfb7ed59631852d1c7f63bc70 |
| ⋮ | ⋮ |

... oops!

[comment]: # (||| data-auto-animate)

```sql [5-8]
-- select a random district
select
  ceil(uniform(125::float, 175::float, random())) as rand_lea,
  (
    select k_lea
    from db.schema.int_leas
    where lea_id=ceil(uniform(125::float, 175::float, random()))
    limit 1
  ) as k_lea
from table(generator( rowcount => 100 ));
```
inner subquery is run once,<br />result reused for every row of outer query!

[comment]: # (||| data-auto-animate)

CTEs fix this:
```sql
-- select a random district
with leas as (
  select k_lea, lea_id
  from db.schema.int_leas
),
base as (
  select
    ceil(uniform(125::float, 175::float, random())) as rand_lea
  from table(generator( rowcount => 100 ))
)
select base.*, leas.k_lea
from base
  join leas on base.rand_lea=leas.lea_id;
```

[comment]: # (||| data-auto-animate)

produces

| rand_lea | k_lea |
|---|---|
| 174 | 23bd8cc78d10d8a3f6a9ec35caa38d33 |
| 171 | 80ed1896bc0fed93d078ac5126181f3d |
| 156 | 7b96ed1ecf4d0f6ff41ee65f698e7d69 |
| ⋮ | ⋮ |

... much better

[comment]: # (||| data-auto-animate)

**problem 2**: some SQL engines<br />don't suppoprt column reuse

[comment]: # (||| data-auto-animate)

```sql
select
  ceil(uniform(0::float, 10::float, random())) as randint,
  5 * randint as multint
from table(generator( rowcount => 100 ));
-- works in Snowflake, but not Postgres :(
```

[comment]: # (||| data-auto-animate)

CTEs fix this:
```sql
with step1 as (
  select
    ceil(uniform(0::float, 10::float, random())) as randint
  from table(generator( rowcount => 100 ));
),
step2 as (
  select
    randint,
    5 * randint as multint
  from step1
)
select * from step2
```

[comment]: # (!!! data-auto-animate)

<code style="color:#0eb9a3;">dbt_synth_data</code> puts all of this together, implementing cross-platform macros for many types of columns, and providing seed data for building synthetic data.

[comment]: # (||| data-auto-animate)

example:

```sql
with
{{ synth_column_primary_key(name='k_customer') }}
{{ synth_column_firstname(name='first_name') }}
{{ synth_column_lastname(name='last_name') }}
{{ synth_column_expression(name='full_name', expression="first_name || ' ' || last_name") }}
{{ synth_column_expression(name='sort_name', expression="last_name || ', ' || first_name") }}
{{ synth_column_date(name="birth_date", min='1938-01-01', max='1994-12-31') }}
{{ synth_column_address(name='shipping_address', countries=['United States'], parts=['street_address', 'city', 'geo_region_abbr', 'postal_code']) }}
{{ synth_column_phone_number(name='phone_number') }}
{{ synth_table(rows=100) }}

select * from synth_table
```

[comment]: # (!!! data-auto-animate)

