# Part B2 — Normalization of the Legacy Orders Export

**Scenario:** S3 — Food Ordering (Lezzet Durağı)  
**Schema:** `food`

<!-- eski sipariş tablosunu düzgün tablolara ayırıyoruz -->

## 1. Original relation and functional dependencies

The old system exports the following relation:

```text
legacy_orders(
    order_no, order_time, branch_code, branch_name, customer_no,
    customer_name, customer_phones, delivery_district, delivery_fee,
    courier_id, courier_name, item_code, item_name, menu_price,
    unit_price, quantity
)
```

The scenario specifies these functional dependencies (FDs):

```text
order_no                         -> order_time, branch_code, customer_no,
                                    delivery_district, courier_id
customer_no                      -> customer_name, customer_phones
branch_code                      -> branch_name
courier_id                       -> courier_name
(branch_code, delivery_district) -> delivery_fee
item_code                        -> item_name, menu_price
(order_no, item_code)            -> unit_price, quantity
```

**Important:** `customer_phones` contains lists such as `0216 555 10 10, 0532 555 20 20`, so it is not an atomic attribute. Strictly speaking, a list-valued field is a multivalued attribute rather than an ordinary scalar FD target. We use the given FDs as the assignment specifies and move phone numbers into their own relation when converting to 1NF.

## 2. Why the export is not in 1NF

First Normal Form (1NF) requires atomic values in every cell. `customer_phones` contains multiple phone numbers in a single cell, violating 1NF.

To fix this, put one phone number per row in a separate relation:

```text
CUSTOMER_PHONE(customer_no, phone)
PK = (customer_no, phone)
```

For the remaining order/item export, remove `customer_phones` and use one row per `(order_no, item_code)`. Each field then holds one value. This also avoids multiplying every order line by the number of the customer's phones.

<!-- telefonları virgülle tek hücrede tutmuyoruz -->

## 3. Candidate keys using attribute closure

Let `O = order_no`, `I = item_code`. Consider the atomic export relation after removing `customer_phones`.

- `O+` includes `order_time`, `branch_code`, `customer_no`, `delivery_district`, `courier_id`, then `branch_name`, `customer_name`, `courier_name`, and `delivery_fee` by transitivity. It does **not** include `item_code`, `item_name`, `menu_price`, `unit_price`, or `quantity`.
- `I+ = {item_code, item_name, menu_price}`; it does not include order information.
- `(O,I)+` includes **all attributes**: `O` determines order-level fields; `I` determines item-level fields; and `(O,I)` determines `unit_price` and `quantity`.

No FD has `order_no` or `item_code` on its right-hand side. Therefore, every key must contain both. Since together they determine all attributes, **the only candidate key is `(order_no, item_code)`** for the atomic order/item export.

<!-- sipariş no ve ürün kodu beraber satırı belirliyor -->

## 4. Decomposition to 2NF

2NF requires 1NF and no non-prime attribute depending on a proper subset of a candidate key. The key is `(order_no, item_code)`.

Violations:

- `order_no -> order_time, branch_code, customer_no, delivery_district, courier_id` (depends on `order_no` alone)
- `item_code -> item_name, menu_price` (depends on `item_code` alone)

Decompose into:

```text
ORDERS_2NF(order_no, order_time, branch_code, branch_name, customer_no,
           customer_name, delivery_district, delivery_fee, courier_id,
           courier_name)
ITEMS_2NF(item_code, item_name, menu_price)
ORDER_LINES_2NF(order_no, item_code, unit_price, quantity)
CUSTOMER_PHONE(customer_no, phone)
```

Keys: `ORDERS_2NF(order_no)`, `ITEMS_2NF(item_code)`, `ORDER_LINES_2NF(order_no, item_code)`, `CUSTOMER_PHONE(customer_no, phone)`.

`ORDERS_2NF` still has transitive dependencies, so 2NF alone is insufficient.

<!-- sipariş bilgilerini ve ürün bilgilerini ayırdık -->

## 5. Decomposition to 3NF

3NF disallows an FD `X -> A` with a non-superkey determinant `X` and a non-prime dependent attribute `A`. The remaining problematic dependencies inside `ORDERS_2NF` are:

```text
customer_no                      -> customer_name
branch_code                      -> branch_name
courier_id                       -> courier_name
(branch_code, delivery_district) -> delivery_fee
```

They cause transitive dependencies from `order_no`. Decompose into:

| Relation | Attributes | Primary key |
|---|---|---|
| `ORDERS` | `order_no, order_time, branch_code, customer_no, delivery_district, courier_id` | `order_no` |
| `CUSTOMERS` | `customer_no, customer_name` | `customer_no` |
| `CUSTOMER_PHONE` | `customer_no, phone` | `(customer_no, phone)` |
| `BRANCHES` | `branch_code, branch_name` | `branch_code` |
| `COURIERS` | `courier_id, courier_name` | `courier_id` |
| `DELIVERY_FEES` | `branch_code, delivery_district, delivery_fee` | `(branch_code, delivery_district)` |
| `ITEMS` | `item_code, item_name, menu_price` | `item_code` |
| `ORDER_LINES` | `order_no, item_code, unit_price, quantity` | `(order_no, item_code)` |

Foreign keys: `ORDERS.branch_code -> BRANCHES`, `ORDERS.customer_no -> CUSTOMERS`, `ORDERS.courier_id -> COURIERS`, `ORDERS.(branch_code,delivery_district) -> DELIVERY_FEES`, `CUSTOMER_PHONE.customer_no -> CUSTOMERS`, `ORDER_LINES.order_no -> ORDERS`, `ORDER_LINES.item_code -> ITEMS`.

<!-- şube müşteri kurye ve ücretleri kendi tablolarına aldık -->

## 6. BCNF check

BCNF requires the left-hand side of every nontrivial FD to be a superkey of its relation.

- `ORDERS`: `order_no` is the key.
- `CUSTOMERS`: `customer_no` is the key.
- `CUSTOMER_PHONE`: `(customer_no, phone)` is the key.
- `BRANCHES`: `branch_code` is the key.
- `COURIERS`: `courier_id` is the key.
- `DELIVERY_FEES`: `(branch_code, delivery_district)` is the key.
- `ITEMS`: `item_code` is the key.
- `ORDER_LINES`: `(order_no, item_code)` is the key.

Thus, **under the supplied FDs and without additional hidden dependencies, the 3NF decomposition is already in BCNF**. No additional split is necessary.

## 7. Lossless-join argument

The decomposition can be reconstructed without spurious order/item rows because each lookup table joins to its parent on a declared key. In particular:

1. `ORDERS` and `ORDER_LINES` intersect in `order_no`, a key of `ORDERS`.
2. `ORDER_LINES` and `ITEMS` intersect in `item_code`, a key of `ITEMS`.
3. `ORDERS` joins to `CUSTOMERS`, `BRANCHES`, and `COURIERS` on their respective primary keys.
4. `ORDERS` joins to `DELIVERY_FEES` on `(branch_code, delivery_district)`, the key of `DELIVERY_FEES`.

Each binary key-based join is lossless under the supplied dependencies, provided required referenced records exist. `CUSTOMER_PHONE` is deliberately not joined into the order/item view: including it would repeat order lines once per phone number. It is a separate 1NF projection of the original multivalued data.

### Bidirectional `EXCEPT ALL` sample-data check

The following SQL is a **template** for a check once the eight relations contain sample rows. The first comparison reconstructs the atomic **order/item** export without the list-valued `customer_phones`. It compares matching column names and order, and both `EXCEPT ALL` directions must return zero rows:

```sql
WITH reconstructed AS (
    SELECT o.order_no, o.order_time, o.branch_code, b.branch_name,
           o.customer_no, c.customer_name, o.delivery_district,
           f.delivery_fee, o.courier_id, cr.courier_name,
           l.item_code, i.item_name, i.menu_price,
           l.unit_price, l.quantity
    FROM orders o
    JOIN order_lines l ON l.order_no = o.order_no
    JOIN branches b ON b.branch_code = o.branch_code
    JOIN customers c ON c.customer_no = o.customer_no
    JOIN couriers cr ON cr.courier_id = o.courier_id
    JOIN delivery_fees f
      ON f.branch_code = o.branch_code
     AND f.delivery_district = o.delivery_district
    JOIN items i ON i.item_code = l.item_code
), original_atomic AS (
    SELECT order_no, order_time, branch_code, branch_name,
           customer_no, customer_name, delivery_district,
           delivery_fee, courier_id, courier_name,
           item_code, item_name, menu_price, unit_price, quantity
    FROM legacy_orders_atomic
)
SELECT 'original_minus_reconstructed' AS direction, count(*) AS differing_rows
FROM (SELECT * FROM original_atomic EXCEPT ALL SELECT * FROM reconstructed) x
UNION ALL
SELECT 'reconstructed_minus_original', count(*)
FROM (SELECT * FROM reconstructed EXCEPT ALL SELECT * FROM original_atomic) y;
```

`legacy_orders_atomic` means the legacy sample rows with `customer_phones` omitted. It must be created or loaded before running this check; this document does **not** claim to have executed it. Phone-number preservation can be checked separately by comparing a correctly split/deduplicated legacy `(customer_no, phone)` dataset with `CUSTOMER_PHONE`, also in both `EXCEPT ALL` directions.

## 8. Dependency preservation

Every supplied *scalar* FD is enforceable inside one final relation:

| Original dependency | Relation preserving it |
|---|---|
| `order_no -> order_time, branch_code, customer_no, delivery_district, courier_id` | `ORDERS` |
| `customer_no -> customer_name` | `CUSTOMERS` |
| `branch_code -> branch_name` | `BRANCHES` |
| `courier_id -> courier_name` | `COURIERS` |
| `(branch_code, delivery_district) -> delivery_fee` | `DELIVERY_FEES` |
| `item_code -> item_name, menu_price` | `ITEMS` |
| `(order_no, item_code) -> unit_price, quantity` | `ORDER_LINES` |

The original `customer_no -> customer_phones` describes a set/list of phones. Its intended association is preserved by `CUSTOMER_PHONE(customer_no, phone)`, not by keeping a list-valued scalar column. The decomposition is dependency-preserving for the supplied scalar FDs.

## 9. Scenario-specific extra question (BR4)

**Why are `menu_price` and `unit_price` different?**

- `menu_price` is the **current catalogue price** of a menu item, depending on `item_code`.
- `unit_price` is the **price actually charged** on a particular order line, depending on `(order_no, item_code)`.

The charged price must not change if the menu price is updated after an order is placed.

**Why store a delivery fee on the new order instead of always looking it up?**

The legacy FD `(branch_code, delivery_district) -> delivery_fee` models the old export's fixed tariff. In the new operational system, tariffs may change over time, so an old order must preserve the **delivery fee charged at the time of ordering**. Therefore `CUSTOMER_ORDER.delivery_fee` is a historical snapshot, while a separate tariff table can hold current branch/district fees. BR4 requires preserving historical prices and delivery fees.

<!-- fiyat değişse bile eski siparişin ücreti sabit kalmalı -->

## 10. Mapping to the new S3 design

The names above illustrate normalization of the **legacy export**; they do not replace the application's required table names. In the final `food` schema, the corresponding concepts map to `food.customer_order`, `food.order_line`, `food.branch`, `food.customer`, `food.menu_item`, plus courier/employee and delivery-fee data as designed in the ER model.

**Status:** Analytical decomposition prepared. The SQL check is not yet executed against populated PostgreSQL tables. The final report should include actual `EXCEPT ALL` output after loading sample data.
