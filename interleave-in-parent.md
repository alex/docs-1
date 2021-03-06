---
title: INTERLEAVE IN PARENT
summary: Interleaving tables improves query performance by optimizing the key-value structure of closely related table's data.
toc: false
toc_not_nested: true
---

Interleaving tables improves query performance by optimizing the key-value structure of closely related tables, attempting to keep data on the same [key-value range](frequently-asked-questions.html#how-does-cockroachdb-scale) if it's likely to be read and written together.

{{site.data.alerts.callout_info}}Interleaving tables does not affect their behavior within SQL.{{site.data.alerts.end}}

<div id="toc"></div>

## How Interleaved Tables Work

When tables are interleaved, data written to one table (known as the **child**) is inserted directly into another (known as the **parent**) in the key-value store. This is accomplished by matching the child table's Primary Key to the parent's.

### Interleave Prefix

For interleaved tables to have Primary Keys that can be matched, the child table must use the parent table's entire Primary Key as a prefix of its own Primary Key––these matching columns are referred to as the **interleave prefix**. It's easiest to think of these columns as representing the same data, which is usually implemented with Foreign Keys.

{{site.data.alerts.callout_success}}To formally enforce the relationship between each table's interleave prefix columns, we recommend using <a href="foreign-key.html">Foreign Key constraints</a>.{{site.data.alerts.end}}

For example, if you want to interleave `orders` into `customers` and the Primary Key of customers is `id`, you need to create a column representing `customers.id` as the first column in the Primary Key of `orders`&mdash;e.g., with a column called `customer`. So the data representing `customers.id` is the interleave prefix, which exists in the `orders` table as the `customer` column.

### Key-Value Structure

When you write data into the child table, it is inserted into the key-value store immediately after the parent table's key matching the interleave prefix.

For example, if you interleave `orders` into `customers`, the `orders` data is written directly within the `customers` table in the key-value store. The following is a crude, illustrative example of what the keys would look like in this structure:

~~~
/customers/1
/customers/1/orders/1000
/customers/1/orders/1002
/customers/2
/customers/2/orders/1001
/customers/2/orders/1003
...
/customers/n/
/customers/n/orders/<order belonging to n>
~~~

By writing data in this way, related data is more likely to remain on the same key-value range, which can make it much faster to read from and write to. Using the above example, all of customer 1's data is going to be written to the same range, including its representation in both the `customers` and `orders` tables.

## When to Interleave Tables

You're most likely to benefit from interleaved tables when:

  - Your tables form a [hierarchy](#interleaved-hierarchy)
  - Queries maximize the [benefits of interleaving](#benefits)
  - Queries do not suffer too greatly from interleaving's [tradeoffs](#tradeoffs)

### Interleaved Hierarchy

Interleaved tables typically work best when the tables form a hierarchy. For example, you could interleave the table `orders` (as the child) into the table `customers` (as the parent, which represents the people who placed the orders). You can extend this example by also interleaving the tables `invoices` (as a child) and `packages` (as a child) into `orders` (as the parent).

The entire set of these relationships is referred to as the **interleaved hierarchy**, which contains all of the tables related through [interleave prefixes](#interleave-prefix).

### Benefits

In general, reads, writes, and joins of values related through the interleave prefix are *much* faster. However, you can also improve performance with any of the following:

- Filtering more columns in the interleave prefix (from left to right).<br/><br/>For example, if the interleave prefix of `packages` is `(customer, order)`, filtering on `customer` would be fast, but filtering on `customer` *and* `order` would be faster.
- Using only tables in the interleaved hierarchy.

### Tradeoffs

- In general, reads and deletes over ranges of table values (e.g., `WHERE column > value`) in interleaved tables are slower.<br/><br/>However, an exception to this is performing operations on ranges of table values in the greatest descendent in the interleaved hierarchy that filters on all columns of the interleave prefix with constant values.<br/><br/>For example, if the interleave prefix of `packages` is `(customer, order)`, filtering on the entire interleave prefix with constant values while calculating a range of table values on another column, like `WHERE customer = 1 AND order = 1001 AND delivery_date > DATE '2016-01-25'`, would still be fast.
- If the amount of interleaved data stored for any Primary Key value of the root table is larger than [a key-value range's maximum size](configure-replication-zones.html#replication-zone-format) (64MB by default), the interleaved optimizations will be diminished.<br/><br/>For example, if one customer has 200MB of order data, their data is likely to be spread across multiple key-value ranges and CockroachDB will not be able to access it as quickly, despite it being interleaved.

## Syntax

{% include sql/diagrams/interleave.html %}

## Parameters

| Parameter | Description |
|-----------|-------------|
| `CREATE TABLE ...` | For help with this section of the syntax, [`CREATE TABLE`](create-table.html).
| `parent_table` | The name of the parent table you want to interleave the new child table into. |
| `interleave_prefix` | A comma-separated list of columns from the child table's Primary Key that represent the parent table's Primary Key (i.e., the interleave prefix). |

## Requirements

- You can only interleave tables when creating the child table.
- Each child table's Primary Key must contain its parent table's Primary Key as a prefix (known as the **interleave prefix**).<br/><br/>For example, if the parent table's primary key is `(a INT, b STRING)`, the child table's primary key could be `(a INT, b STRING, c DECIMAL)`.
  {{site.data.alerts.callout_info}}This requirement is enforced only by ensuring that the columns use the same data types. However, we recommend ensuring the columns refer to the same values by using the  <a href="foreign-key.html">Foreign Key constraint</a>.{{site.data.alerts.end}}
- Interleaved tables cannot be the child of more than 1 parent table. However, each parent table can have many children tables. Children tables can also be parents of interleaved tables.

## Recommendations

- Use interleaved tables when your schema forms a hierarchy, and the Primary Key of the root table (for example, a "user ID" or "account ID") is a parameter to most of your queries.
- To enforce the relationship between the parent and children table's Primary Keys, use [Foreign Key constraints](foreign-key.html) on the child table.
- In cases where you're uncertain if interleaving tables will improve your queries' performance, test how tables perform under load when they're interleaved and when they aren't.

## Examples

### Interleaving Tables

This example creates an interleaved hierarchy between `customers`, `orders`, and `packages`, as well as the appropriate Foreign Key constraints. You can see that each child table uses its parent table's Primary Key as a prefix of its own Primary Key (the **interleave prefix**).

~~~ sql
> CREATE TABLE customers (
    id INT PRIMARY KEY,
    name STRING(50)
  );

> CREATE TABLE orders (
    customer INT, 
    id INT,
    total DECIMAL(20, 5), 
    PRIMARY KEY (customer, id), 
    CONSTRAINT fk_customer FOREIGN KEY (customer) REFERENCES customers
    ) INTERLEAVE IN PARENT customers (customer)
  ;

> CREATE TABLE packages (
    customer INT,
    "order" INT,
    id INT,
    address STRING(50),
    delivered BOOL,
    delivery_date DATE,
    PRIMARY KEY (customer, "order", id),
    CONSTRAINT fk_order FOREIGN KEY (customer, "order") REFERENCES orders
    ) INTERLEAVE IN PARENT orders (customer, "order")
  ;
~~~

### Key-Value Storage Example

It can be easier to understand what interleaving tables does by seeing what it looks like in the key-value store. For example, using the above example of interleaving `orders` in `customers`, we could insert the following values:

~~~ sql
> INSERT INTO customers 
  (id, name) VALUES 
  (1, 'Ha-Yun'),
  (2, 'Emanuela');

> INSERT INTO orders 
  (customer, id, total) VALUES 
  (1, 1000, 100.00),
  (2, 1001, 90.00),
  (1, 1002, 80.00),
  (2, 1003, 70.00);
~~~

Using an illustrative format of the key-value store (keys are represented in colors; values are represented by `-> value`), the data would be written like this:

<pre class="highlight">
<span style="color:#62B6CB">/customers/</span><span style="color:#47924a">&lt;customers.id = 1&gt;</span> -> 'Ha-Yun'
<span style="color:#62B6CB">/customers/</span><span style="color:#47924a">&lt;orders.customer = 1&gt;</span><span style="color:#FC9E4F">/orders/</span><span style="color:#ef2da8">&lt;orders.id = 1000&gt;</span> -> 100.00
<span style="color:#62B6CB">/customers/</span><span style="color:#47924a">&lt;orders.customer = 1&gt;</span><span style="color:#FC9E4F">/orders/</span><span style="color:#c4258a">&lt;orders.id = 1002&gt;</span> -> 80.00
<span style="color:#62B6CB">/customers/</span><span style="color:#2f6246">&lt;customers.id = 2&gt;</span> -> 'Emanuela'
<span style="color:#62B6CB">/customers/</span><span style="color:#2f6246">&lt;orders.customer = 2&gt;</span><span style="color:#FC9E4F">/orders/</span><span style="color:#EF2D56">&lt;orders.id = 1001&gt;</span> -> 90.00
<span style="color:#62B6CB">/customers/</span><span style="color:#2f6246">&lt;orders.customer = 2&gt;</span><span style="color:#FC9E4F">/orders/</span><span style="color:#c42547">&lt;orders.id = 1003&gt;</span> -> 70.00
</pre>

You'll notice that `customers.id` and `orders.customer` are written into the same position in the key-value store. This is how CockroachDB relates the two table's data for the interleaved structure. By storing data this way, accessing any of the `orders` data alongside the `customers` is much faster.

{{site.data.alerts.callout_info}}If we didn't set Foreign Key constraints between <code>customers.id</code> and <code>orders.customer</code> and inserted <code>orders.customer = 3</code>, the data would still get written into the key-value in the expected location next to the <code>customers</code> table identifier, but <code>SELECT * FROM customers WHERE id = 3</code> would not return any values.{{site.data.alerts.end}}

To better understand how CockroachDB writes key-value data, see our blog post [Mapping Table Data to Key-Value Storage](https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/).

## See Also

- [`CREATE TABLE`](create-table.html)
- [Foreign Keys](foreign-key.html)
- [Column Families](column-families.html)

