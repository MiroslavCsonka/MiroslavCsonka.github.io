---
classes: wide
---

The core of this technique can be distilled into this query.
```sql
SELECT *
FROM orders,
     LATERAL json_array_elements(orders.raw_payload -> 'line_items') json_line_item
```

If there's even a single word from this query you don't quite understand, you are in the right place.

## Schema and test data
Before we get started with queries, let me introduce a simple schema.
```sql
CREATE TABLE orders (
    id          int PRIMARY KEY,
    raw_payload json
)
```
`raw_payload` is just a content from [Shopify's order object](https://shopify.dev/docs/admin-api/rest/reference/orders/order). It's going to give us plenty to work for as typical order contains ~400 lines of JSON. 

For the purposes of this articles, imagine the payload only contains:
```json
{
  "line_items": [
    {
      "id": 1,
      "discount_allocations": [
        {
          "amount_set": {
            "shop_money": {
              "amount": "10.30",
              "currency": "GBP"
            }
          }
        }
      ]
    },
    {
      "id": 2,
      "discount_allocations": []
    }
  ]
}
```
With order like this, someone purchased 2 products, and one of them received £10.30 discount

Now that we understand the data, we can start digging into it.

## Lateral join with json
This example is build on 2 important and not so often used parts. First one is `json_array_elements` (or `jsonb_array_elements`) and second `LATERAL JOIN`.

### json_array_elements
It expands an array inside a JSON into an set of jsons as individual rows. These value will be available under new `value` column

If we look at example from [PostgreSQL's documentation](https://www.postgresql.org/docs/9.5/functions-json.html):
```sql
SELECT *
FROM json_array_elements('[1, true, [2, false]]')
```

Will get turned into a result with 3 rows:

![json_array_elements_result.png](/assets/postgresql-quering-json/json_array_elements_result.png)

### LATERAL join
Now let's tackle the `LATERAL` part. Think of it as a `each` from a typical programming language. It allows you join a different set of rows while giving you an access to an individual row from the previous association. 

In our example we are using 

```sql
SELECT orders.id, json_line_item
FROM orders,
     LATERAL json_array_elements(orders.raw_payload -> 'line_items') json_line_item
```

which would produce
![lateral_join.png](/assets/postgresql-quering-json/lateral_join.png)

Let's break down each part of the syntax:
1. `->` access json key and return it as json field. Same as `raw_payload.line_items` in Javascript, or `raw_payload['line_items']` in Ruby
2. `json_array_elements` expand array of jsons into rows
3. for each order, add the newly created rows from `json_array_elements`
4. `json_line_item` call the newly created column `json_line_item`


## Real world examples

Now that we know the basics of querying jsons, let's answer a few answers.

Q: How many line items does each order have?

```sql
SELECT orders.id, COUNT(json_line_item)
FROM orders,
     LATERAL JSONB_ARRAY_ELEMENTS(orders.raw_payload -> 'line_items') shopify_line_items(json_line_item)
GROUP BY 1
```
![num_of_line_items_per_order.png](/assets/postgresql-quering-json/num_of_line_items_per_order.png)

Q: Report discounts for all line items and orders
```sql
SELECT orders.id                                                        AS order_id,
       json_line_item -> 'id'                                           AS line_item_id,
       discount_allocation -> 'amount_set' -> 'shop_money' ->> 'amount' AS discount_pounds
FROM orders,
     LATERAL JSONB_ARRAY_ELEMENTS(orders.raw_payload -> 'line_items') json_line_item,
     LATERAL JSONB_ARRAY_ELEMENTS(json_line_item -> 'discount_allocations') discount_allocations(discount_allocation)
```
![orders_with_line_items_and_discounts.png](/assets/postgresql-quering-json/orders_with_line_items_and_discounts.png)

This technique was shown to me by [@pawelpacana](https://twitter.com/pawelpacana). You won't regret his content.
