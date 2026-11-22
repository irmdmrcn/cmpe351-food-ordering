# Requirements Document

## Project Information

Course: CMPE 351 - Database Management Systems

Scenario: S3 - Food Ordering 

Project Name: Lezzet Durağı

Database Schema: food  <!--postgresql'de tablolarımızın bulunacağı şema-->

## 1. Project Overview

Lezzet Durağı is a restaurant chain operating four
branches in Türkiye:

- Istanbul Kadıköy
- Istanbul Beşiktaş
- Ankara Çankaya
- Izmir Alsancak

The purpose of this project is to design and implement
a PostgreSQL database for managing the restaurant
chain's daily operations.

The system will manage customers, menu items,
ingredients, recipes, inventory, suppliers,
purchase orders, customer orders, payments,
couriers, deliveries, and employees.


## 2. System Actors

The system has the following actors:

### 2.1 Order Clerk <!--sipariş alan görevli-->
- Creates customer orders.
- Adds menu items to orders.
- Places orders after stock availability checks.
- Tracks order status.

### 2.2 Kitchen Clerk
- Manages food preparation.
- Records ingredient consumption.
- Manages preparation batches.
- Updates kitchen-related order statuses.

### 2.3 Dispatcher
- Assigns couriers to delivery orders.
- Checks courier availability and active shifts.
- Records dispatch and delivery information.
- Ensures that couriers do not exceed delivery limits.

### 2.4 Purchasing Officer
- Creates ingredient purchase orders.
- Selects suppliers.
- Records partial deliveries from suppliers.
- Monitors ingredient purchasing needs.

### 2.5 Customer Support Agent
- Handles customer inquiries and complaints.
- Reviews customer contact information.
- Assists with order problems and refunds.

### 2.6 Administrator
- Manages database users and permissions.
- Maintains system configuration.
- Monitors database operations.
- Has access to administrative functions.

### 2.7 Auditor
- Reviews audit logs and transaction records.
- Monitors changes to important data.
- Has read-only access to relevant information.


## 3. Functional Requirements
### FR1 - Customer Management
The system shall store customer information and allow each customer to maintain multiple saved delivery addresses.

### FR2 - Branch Management <!--restoran şubelerinin bilgilerini yönetiyoruz -->
The system shall maintain information about restaurant branches, including their locations and delivery fees.

### FR3 - Menu Management <!-- yemekler, kategoriler, fiyatlar ve tarifler -->
The system shall manage menu items, categories, prices, and ingredient-based recipes.

### FR4 - Ingredient Management <!--her yemeğin hangi malzemelerden ne kadar kullandığını takip ediyoruz-->
The system shall store ingredients, measurement units, and quantities required per portion.

### FR5 - Inventory Management <!--her şubenin stokları ayrı tutulacak. Tüm stok değişiklikleri kaydedilecek-->
The system shall track ingredient stock separately for each branch and record every stock movement.

### FR6 - Supplier Management <!--tdarikçiler, fiyat teklifleri ve teslim süreleri-->
The system shall manage suppliers, ingredient offers, prices, lead times, and preferred suppliers.

### FR7 - Purchasing Management <!--malzeme satın alma işlemleri ve parçalı teslimatlar -->
The system shall manage purchase orders, purchase order lines, and partial deliveries.

### FR8 - Order Management <!--müşteri siparişleri, sipariş satırları ve durum değişiklikleri -->
The system shall create and track customer orders, their order lines, statuses, and totals.

### FR9 - Payment Management <!--ödemeler, iadeler ve müşteri cüzdanı-->
The system shall record payments, refunds, wallet top-ups, and wallet transactions.

### FR10 - Delivery Managemen <!--kurye atama, vardiya ve teslimat işlemleri-->
The system shall manage courier assignments, active shifts, dispatch times, and delivery times.

### FR11 - Employee Management <!--çalışanların şubeleri, yöneticileri ve maaşları-->
The system shall maintain employee records, branch assignments, reporting relationships, and role-specific information.

### FR12 - Reporting and Security <!--raporlar, değişiklik kayıtları ve hassas müşteri bilgilerinin korunması-->
The system shall provide reporting views, record audited changes, and restrict access to confidential customer information.


## 4. Business Rules

<!--
hocanın verdiği 12 tane kural var burda
bunları veritabanında kontrol etmemiz gerekiyor
şimdilik hangi yöntemlerle yapacağımızı yazıyoruz
kodlarını sonraki aşamalarda yazıcaz
-->

### BR1 - Non-Negative Ingredient Stock

The stock on hand of an ingredient in a branch must never be negative.

Enforcement: CHECK constraint, trigger, and transactional stock validation.

<!--stok eksiye düşemez-->

### BR2 - Stock Availability Before Order Placement

An order can be placed only if the total quantity of every required ingredient is available in the order's branch.

Placing the order must reserve the required quantities.

Enforcement: Stored procedure with stock validation, row locking, and reservation updates.

<!--sipariş verilmeden önce malzemeler yeterli mi bakıcaz aynı malzemeyi kullanan yemeklerin miktarlarını toplamak lazım stok varsa siparişi onaylayıp malzemeleri rezerve edicez -->

### BR3 - Valid Order Status Transitions

Order statuses must follow this sequence:

CART -> PLACED -> PREPARING -> ON_THE_WAY -> DELIVERED

Cancellation is allowed only from CART or PLACED and must release any existing reservations.

Enforcement: Trigger and stored procedures.

<!-- sipariş kafasına göre durum değiştiremez mesela direkt hazırlanıyordan teslim edildiye geçmesin iptal de sadece cart veya placed durumundayken olabiliyor iptal edilince ayrılan stok geri açılmalı-->

### BR4 - Historical Price Preservation

Each order line must store the unit price charged when the order was created.

Each order must store its charged delivery fee.

Changes to current menu prices or delivery fees must not modify historical orders.

Enforcement: Stored price snapshots and order-line trigger.

<!--müşteri sipariş verdiği andaki fiyatı saklıyoruz sonradan lahmacuna zam gelse bile eski sipariş değişmesin teslimat ücreti de aynı mantık -->

### BR5 - Courier Assignment Limit

A courier may carry at most three active dispatched orders simultaneously.

The courier must have an open shift at the order's branch.

Dispatches that violate these conditions must be rejected.

Enforcement: Dispatch procedure with shift validation and concurrency-safe capacity checks.

<!--bir kurye aynı anda en fazla 3 sipariş taşıyabilir ayrıca o şubede vardiyası açık olmalı-->

### BR6 - Stock Movement Consistency

Every ingredient stock change must be recorded in the stock movement ledger.

The current stock quantity must always equal the sum of its stock movements.

Enforcement: Stock movement trigger and controlled inventory procedures.

<!-- stok değiştiyse bunun kaydı da olmalı stok tablosuyla hareket kayıtları aynı sonucu vermeli-->

### BR7 - Atomic Preparation Batches

Completing N preparation batches must consume the required quantities of their source ingredients.

For each preparation recipe line, the consumed quantity equals quantity_per_batch multiplied by N.

The prepared ingredient stock must increase by batch_yield multiplied by N.

All changes must succeed together or fail together.

Enforcement: Transactional preparation procedure with stock validation.

<!--mesela köfte harcı hazırlıyoruz kaç parti yapılıyorsa ona göre malzeme eksilecek hazırlanan köfte harcı da stoğa eklenecekbi işlem hata verirse hepsi geri alınmalı-->

### BR8 - Partial Purchase Order Receipts

Purchase order lines may be received in multiple partial deliveries.

The total received quantity must never exceed the originally ordered quantity.

Enforcement: Receipt procedure with quantity validation and transaction locking.

<!--toplam 20 kg geçemez-->

### BR9 - Order Totals and Payment Limits

An order's total must be calculated as:

(sum of order lines - campaign discount + delivery fee) * 1.20

The result must be rounded to the nearest kuruş.

Total payments must not exceed the order total.

Total refunds must not exceed the amount already paid.

Enforcement: Total-calculation function, triggers, and payment procedures.

<!--sipariş toplamını hesaplarken indirim ve kargo da var sonra yüzde 20 kdv ekliyoruz müşteri sipariş tutarından fazla ödeme yapmamalı iade de ödediğinden fazla olamaz -->

### BR10 - Restricted Deletion

A customer, supplier, menu item, or ingredient referenced by business documents cannot be deleted.

Deleting an order in CART status must also delete its order lines.

Enforcement: Foreign keys with appropriate ON DELETE actions.

<!--siparişte kullanılan müşteri veya ürün bilgisi silinmesin yoksa eski kayıtların bağlantısı bozulabilir ama cart durumundaki sipariş silinirse içindeki ürünler de silinsin-->

### BR11 - Audit Logging

Changes to menu item prices and employee wages must be recorded in an audit log.

Each audit record must contain the old value, new value, user, and timestamp.

Enforcement: Audit triggers.

<!--bir yemeğin fiyatı veya çalışanın maaşı değişince kim değiştirmiş ne zaman değiştirmiş kaydedicez eski ve yeni değerler de tutulmalı -->

### BR12 - Confidential Customer Information

Only customer support agents and administrators may read customers' phone numbers and email addresses.

Other roles must not have direct access to these confidential fields.

Enforcement: PostgreSQL roles, column-level privileges, and restricted views.

<!--müşterilerin telefon ve mail bilgilerini herkes göremesinsadece support agent ve admin görebilir diğerleri için bilgileri gizleyen view kullanıcaz -->


## 5. Design Assumptions
### A1 - Customer Addresses

Each customer may have multiple saved delivery addresses.
Each customer order uses exactly one saved address
belonging to the customer who placed the order.

<!-- müşteri kendi kayıtlı adreslerinden birini seçiyor -->

### A2 - Branch-Based Inventory

Each restaurant branch maintains its own ingredient
stock independently.

<!-- her şubenin stoğu ayrı tutuluyor -->

### A3 - Order Ownership

Each customer order belongs to exactly one customer
and one restaurant branch.

<!-- sipariş tek müşteri ve şubeye ait -->

### A4 - Menu Item Recipes

Each menu item has a recipe consisting of one or more
ingredients and their required quantities per portion.

<!-- yemeklerin tariflerini ve malzeme miktarlarını tutuyoruz -->

### A5 - Historical Prices

The unit price and delivery fee charged at ordering
time are stored in the order records.

<!-- sonradan zam gelse de eski siparişin fiyatı değişmiyor -->

### A6 - Courier Employment

Each courier is an employee assigned to exactly one
restaurant branch.

<!-- kuryeler de çalışan ve bir şubeye bağlılar -->

### A7 - Ingredient Units

Every ingredient has a defined measurement unit.
Recipe quantities, stock quantities, and purchase
quantities use compatible units.

<!-- kg litre adet gibi birimleri karıştırmamak lazım -->

### A8 - Delivery Locations

Orders may be delivered only to eligible addresses
in the same city as the restaurant branch.

<!-- şubeler sadece kendi şehirlerine teslimat yapıyor -->

### A9 - Wallet Transactions

Every wallet top-up, payment, and refund is recorded
as an individual wallet transaction when it affects
the customer's wallet.

<!-- cüzdandaki para hareketleri tek tek kaydediliyor -->

### A10 - Order Payments

An order may have multiple payments using supported
payment methods.

<!-- bir sipariş için birden fazla ödeme yapılabilir -->