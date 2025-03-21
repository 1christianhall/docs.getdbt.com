---
title: "Sources"
id: "using-sources"
---

## Related reference docs
* [Source properties](source-properties)
* [Source configurations](source-configs)
* [`{{ source() }}` jinja function](dbt-jinja-functions/source)
* [`source freshness` command](commands/source)

## Using sources
Sources make it possible to name and describe the data loaded into your warehouse by your Extract and Load tools. By declaring these tables as sources in dbt, you can then
- select from source tables in your models using the `{{ source() }}` function, helping define the lineage of your data
- test your assumptions about your source data
- calculate the freshness of your source data

### Declaring a source

Sources are defined in `.yml` files nested under a `sources:` key.

<File name='models/<filename>.yml'>

```yaml
version: 2

sources:
  - name: jaffle_shop
    tables:
      - name: orders
      - name: customers

  - name: stripe
    tables:
      - name: payments
```

</File>

If you're not already familiar with these files, be sure to check out [the documentation on schema.yml files](configs-and-properties) before proceeding.

### Selecting from a source

Once a source has been defined, it can be referenced from a model using the [`{{ source()}}` function](dbt-jinja-functions/source).


<File name='models/orders.sql'>

```sql
select
  ...

from {{ source('jaffle_shop', 'orders') }}

left join {{ source('jaffle_shop', 'customers') }} using (customer_id)

```

</File>

dbt will compile this to the full table name:

<File name='target/compiled/jaffle_shop/models/my_model.sql'>

```sql

select
  ...

from raw.jaffle_shop.orders

left join raw.jaffle_shop.customers using (customer_id)

```

</File>

Using the `{{ source () }}` function also creates a dependency between the model and the source table.

<Lightbox src="/img/docs/building-a-dbt-project/sources-dag.png" title="The source function tells dbt a model is dependent on a source "/>

### Testing and documenting sources
You can also:
- Add tests to sources
- Add descriptions to sources, that get rendered as part of your documentation site

These should be familiar concepts if you've already added tests and descriptions to your models (if not check out the guides on [testing](building-a-dbt-project/tests) and [documentation](documentation)).

<File name='models/<filename>.yml'>

```yaml
version: 2

sources:
  - name: jaffle_shop
    description: This is a replica of the Postgres database used by our app
    tables:
      - name: orders
        description: >
          One record per order. Includes cancelled and deleted orders.
        columns:
          - name: id
            description: Primary key of the orders table
            tests:
              - unique
              - not_null
          - name: status
            description: Note that the status can change over time

      - name: ...

  - name: ...
```

</File>

You can find more details on the available properties for sources in the [reference section](source-properties).

### FAQs
<FAQ src="source-has-bad-name" />
<FAQ src="source-in-different-database" />
<FAQ src="source-quotes" />
<FAQ src="testing-sources" />
<FAQ src="running-models-downstream-of-source" />

## Snapshotting source data freshness
With a couple of extra configs, dbt can optionally snapshot the "freshness" of the data in your source tables. This is useful for understanding if your data pipelines are in a healthy state, and is a critical component of defining SLAs for your warehouse.

### Declaring source freshness
To configure sources to snapshot freshness information, add a `freshness` block to your source and `loaded_at_field` to your table declaration:

<File name='models/<filename>.yml'>

```yaml
version: 2

sources:
  - name: jaffle_shop
    database: raw
    freshness: # default freshness
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _etl_loaded_at

    tables:
      - name: orders
        freshness: # make this a little more strict
          warn_after: {count: 6, period: hour}
          error_after: {count: 12, period: hour}

      - name: customers # this will use the freshness defined above


      - name: product_skus
        freshness: null # do not check freshness for this table
```

</File>

In the `freshness` block, one or both of `warn_after` and `error_after` can be provided. If neither is provided, then dbt will not calculate freshness snapshots for the tables in this source.

Additionally, the `loaded_at_field` is required to calculate freshness for a table. If a `loaded_at_field` is not provided, then dbt will not calculate freshness for the table.

These configs are applied hierarchically, so `freshness` and `loaded_at` field values specified for a `source` will flow through to all of the `tables` defined in that source. This is useful when all of the tables in a source have the same `loaded_at_field`, as the config can just be specified once in the top-level source definition.

### Checking source freshness
To snapshot freshness information for your sources, use the `dbt source freshness` command ([reference docs](commands/source)):

```
$ dbt source freshness
```

Behind the scenes, dbt uses the freshness properties to construct a `select` query, shown below. You can find this query in the logs.

```sql
select
  max(_etl_loaded_at) as max_loaded_at,
  convert_timezone('UTC', current_timestamp()) as snapshotted_at
from raw.jaffle_shop.orders

```

The results of this query are used to determine whether the source is fresh or not:

<Lightbox src="/img/docs/building-a-dbt-project/snapshot-freshness.png" title="Uh oh! Not everything is as fresh as we'd like!"/>


### FAQs
<FAQ src="exclude-table-from-freshness" />
<FAQ src="snapshotting-freshness-for-one-source" />
<FAQ src="snapshot-freshness-output" />
