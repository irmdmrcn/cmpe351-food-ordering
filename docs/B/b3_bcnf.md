# Part B3 - 3NF but not BCNF

**Scenario:** S3 - Food Ordering  
**Relation:** `zone_courier(district, branch_code, courier_id)`

## 1. Meaning and functional dependencies

The scenario states that a district can be served by several branches. For each district and branch, exactly one courier is responsible; each courier works for exactly one branch.

Let `D = district`, `B = branch_code`, and `C = courier_id`.

- `DB -> C`: a particular branch has exactly one responsible courier for a particular district
- `C -> B`: each courier belongs to exactly one branch

<!-- burada iki tane fd var biri district ve branchten courierı buluyor diğeri courierdan branchi -->

## 2. Candidate keys and attribute closures

- `(DB)+ = {D, B, C}`, using `DB -> C`; therefore `DB` is a candidate key
- `(DC)+ = {D, C, B}`, using `C -> B`; therefore `DC` is a candidate key
- `D+ = {D}`, `B+ = {B}`, and `C+ = {C, B}`; none of these is a key

Both candidate keys are minimal: removing either attribute from `DB` or `DC` leaves a set that cannot determine all attributes.

**Candidate keys:** `(district, branch_code)` and `(district, courier_id)`.

All three attributes are **prime attributes**, because each occurs in at least one candidate key.

<!-- iki aday anahtar var db ve dc tek başına hiçbiri tüm sütunları bulmuyor -->

## 3. Why the relation is in 3NF

The 3NF condition for every non-trivial functional dependency `X -> A` is: either `X` is a superkey or `A` is a prime attribute.

- `DB -> C`: `DB` is a candidate key, so this dependency satisfies 3NF
- `C -> B`: `C` is not a superkey, but `B` is prime (it is part of candidate key `DB`), so this dependency also satisfies 3NF

Therefore `zone_courier` is **in 3NF**.

<!-- c tek başına key değil ama b prime attribute olduğu için 3nf bozulmuyor -->

## 4. Why the relation is not in BCNF

BCNF requires the determinant of every non-trivial functional dependency to be a superkey.

The dependency `C -> B` violates BCNF because `courier_id` alone cannot determine `district` and thus is not a superkey.

Therefore `zone_courier` is **not in BCNF**.

<!-- couriera bakıp districti bulamıyoruz o yüzden bcnf olmuyor -->

## 5. BCNF decomposition

Decompose using the violating dependency `C -> B`:

1. `courier_branch(courier_id, branch_code)` with key `courier_id`
2. `district_courier(district, courier_id)` with key `(district, courier_id)`

Both resulting relations are in BCNF under their projected dependencies.

The common attribute is `courier_id`. It determines all attributes of `courier_branch`, so the binary decomposition is **lossless**.

<!-- iki tabloya ayırınca courier ortak alan oluyor ve join yapınca veri kaybolmuyor -->

## 6. Dependency preservation

The decomposition preserves `C -> B` in `courier_branch`.

However, `DB -> C` is **not preserved locally**: no single decomposed table contains all of `district`, `branch_code`, and `courier_id`.

For example, if `district_courier` contains `(Kadikoy, C1)` and `(Kadikoy, C2)`, and `courier_branch` contains `(C1, KDK)` and `(C2, KDK)`, joining them yields two different couriers for the same `(Kadikoy, KDK)`. This violates `DB -> C` even though both decomposed relations satisfy their local keys.

To enforce the lost dependency in a PostgreSQL implementation, use a controlled transaction/procedure or a trigger that checks the joined data for a duplicate `(district, branch_code)` assignment. An ordinary `UNIQUE` constraint on either separate table alone is not sufficient.

<!-- db courierı tek başına belirlemeli ama ayrılmış tablolarda bunu ayrıca kontrol etmek gerekiyor -->

## 7. Conclusion

The original relation is in **3NF but not BCNF**. The decomposition into `courier_branch` and `district_courier` is **lossless and BCNF-compliant**, but it does **not preserve** the functional dependency `(district, branch_code) -> courier_id` without an additional cross-table enforcement mechanism.
