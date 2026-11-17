# Software Requirements Specification (SRS): ExpressMe E-Commerce Platform

| Document metadata | Value |
|---|---|
| Author | Zeinab Suliman Mohamed Osman |
| Document version | **0.12** |
| Status | Reviewed draft |
| Project type | Portfolio project |
| Date | 16 November 2026 |

## 1. Introduction

This SRS defines the requirements for **ExpressMe**, an e-commerce website that sells products owned and supplied by ExpressMe. ExpressMe operates as a single retailer and manages its own product catalog, inventory, orders, customer payments, and fulfillment.

### 1.1 Scope

**In scope (v1):**

- Customer accounts, Admin accounts, and role-based permissions.
- Company-owned product catalog (categories, products, variants), pricing, and inventory management.
- Customer storefront: browsing, search, and style-based personalization.
- Cart and checkout with one standard payment method per order, optionally combined with one gift card.
- Discounts, vouchers, and optional gift cards.
- Orders and company-managed fulfillment.
- Cash on delivery (COD) collected through a third-party shipping provider, and recording of COD collection and settlement.
- Sandbox card payment.
- A simulated customer wallet with a double-entry ledger.
- Returns and refunds.
- English and Arabic interfaces.
- Multiple currencies and basic country-based tax calculation.
- Basic SEO and a documented REST API.
- Performance engineering: Redis caching, RabbitMQ-based order processing, and load testing.

**Out of scope (v1):**

- Native mobile applications.
- Third-party Seller accounts and Seller storefronts.
- Processing real card payments or real wallet top-ups. In this project, card payments and wallet top-ups use a payment-gateway sandbox and simulated funds.
- Direct integration with banks or shipping-provider systems. For v1, an authorized Admin records collection, remittance, and refund confirmations using references or evidence supplied by the provider or bank. Actual bank transfers take place outside ExpressMe.
- A full general ledger or statutory accounting system. The wallet ledger and COD settlement records support their respective workflows but do not replace a complete accounting system.
- Multi-currency conversion between wallet balances.
- Automated tax filing or automated tax-authority payments.
- Partial shipments: all lines of an order are dispatched together.
- Failed delivery of prepaid orders. In v1, failed delivery is handled only for COD orders (see FR-054).
- Per-product tax rates or tax exemptions, and shipping charges calculated by weight or by a carrier service. Shipping is a flat charge per destination country.

### 1.2 User roles

| Role | Responsibilities |
| --- | --- |
| Guest | Browse and search products and manage a guest cart. Must log in or register to check out. |
| Customer | Shop, manage a profile and saved addresses, view orders, use a wallet, and request returns. |
| Admin | Manage customers, the company product catalog, inventory, orders, fulfillment, supported currencies, exchange rates, taxes, shipping charges, promotions, platform settings, and COD settlement records, within the permissions granted to the Admin (see FR-007). |

### 1.3 Conventions

Requirement IDs begin with **FR** for functional requirements and **NFR** for non-functional requirements. The word **must** indicates a mandatory requirement, **should** indicates a desirable requirement that may be deferred, and **could** indicates an optional capability. "Authorized Admin" means an Admin who holds the permission group required for the function (see FR-007). Default values given in a requirement are initial settings that an Admin can change unless stated otherwise.

### 1.4 Definitions

| Term | Definition |
| --- | --- |
| On-hand stock | Quantity of a product variant physically held by ExpressMe and not yet dispatched. It includes units reserved for orders. |
| Reserved stock | The part of on-hand stock allocated to orders that have not been dispatched. |
| Available stock | On-hand stock minus reserved stock. |
| Standard payment method | `CARD`, `WALLET`, or `COD`. An order uses exactly one standard payment method, except when a gift card covers the whole amount due. |
| Payment component | One part of an order's payment, made by one method (`WALLET`, `CARD`, `COD`, or `GIFT_CARD`) for a stated amount. An order has one standard component and, optionally, one gift card component. Vouchers and seasonal discounts are price reductions, not payment components. |
| Prepaid component | A payment component made by `WALLET`, `CARD`, or `GIFT_CARD`, collected before fulfillment. |
| Voucher | A code that gives a price reduction on an order's merchandise before tax is calculated. It is a discount, not a payment method. |
| COD | Cash on delivery: the Customer pays the shipping provider by cash when the order is delivered. |
| COD charge | A fee charged to the Customer when an order includes a COD component. It is retained by ExpressMe. |
| Remittance | The transfer of collected COD money minus the retained shipping charge, from the shipping provider to ExpressMe. |
| Minor unit | The smallest unit of a currency as defined by ISO 4217 (for example cents). |
| Merchandise amount | The value of the items after discounts and before tax, shipping, and COD charge. |
| Idempotency key | A unique value supplied with a request so that repeating the request has no additional effect. |
| Fulfillment task | A work item created for each confirmed order for ExpressMe's warehouse operations. |
| Active exchange rate | The most recent exchange rate whose effective date-time is current. |

## 2. Overall Description

ExpressMe must be implemented as a **modular monolith** for v1. The application must expose a REST API and contain separate business modules with clearly defined responsibilities. The modules must run as one application and use one PostgreSQL database with two schemas: a `finance` schema for money data and an `app` schema for all other data (see NFR-018).

ExpressMe is the only retailer represented in the system. Products, inventory, customer orders, and sales belong to ExpressMe. The system must not support third-party Seller accounts, Seller-specific catalogs, Seller fulfillment groups, Seller commissions, or Seller payouts.

The wallet and its ledger must be implemented as a protected module within the application. The wallet module must own its wallet and ledger data, and no other module must write to it except through the Wallet module's published interface.

### 2.1 Architecture and module boundaries

| Module | Responsibility |
| --- | --- |
| User Management | Registration, authentication, roles and permissions, customer profiles, and addresses |
| Product | Company-owned products, variants, categories, prices, images, product content, and search |
| Inventory | Company stock quantities, stock movements, and reservations |
| Cart | Cart operations and checkout coordination |
| Currency and Tax | Supported currencies, exchange rates, destination countries, tax rates, shipping charges, and COD charge |
| Orders | Orders, order lines, order status, and fulfillment tasks |
| Promotions | Promotion rules, discounts, vouchers, and gift cards |
| Payments | Payment components, payment records, and payment-gateway integration |
| Returns | Return requests, decisions, and refund coordination |
| Wallet | Simulated customer wallet accounts, wallet transactions, ledger entries, balances, and reconciliation |
| Settlement | COD collections, remittances received by ExpressMe, and tax and sales allocation |
| Shipping | Shipping providers, provider assignment to orders, and shipping-charge deductions |
| Notifications | Notification templates and delivery status |
| Audit | Audit log of Admin actions |

### 2.2 Technology and deployment environment

- **Technology:** Java 21, Spring Boot 4, Spring Security, Spring Data JPA, PostgreSQL 16, Redis 7, RabbitMQ, Flyway database migration tool and Docker Compose.
- **API:** The application must expose a REST API documented with OpenAPI/Swagger (see FR-100).
- **Deployment:** `docker compose up` must start the application and its required local dependencies, including PostgreSQL, Redis, and RabbitMQ, and initialize the database schema (see NFR-016).
- **Money representation:** Monetary amounts, including COD collections and remittances, must be stored as integers (BIGINT) in the minor units defined for the relevant currency and must include an ISO 4217 currency code.

### 2.3 Assumptions

- A payment-gateway sandbox is available.
- Supported currencies and exchange rates are managed by an Admin or loaded from an available exchange-rate source.
- A third-party shipping provider offers COD collection and remittance services under an agreement with ExpressMe.
- Product text is supplied by Admins in English and Arabic.
- The storefront user interface technology is chosen during design; the requirements in this SRS apply regardless of that choice.

## 3. Functional Requirements

### 3.1 Accounts, roles, and audit

- **FR-001:** Users must be able to register with a unique email address and a password of at least 8 characters. Registration with an email address that is already registered must be rejected with an error message.
- **FR-002:** Registered users must be able to log in using their email address and password.
- **FR-003:** The system must authenticate users using JWT access tokens (default lifetime 15 minutes) and refresh tokens, and must enforce role-based access for Customers and Admins. Requests for Admin functions from unauthenticated users must be rejected as unauthenticated, and from Customers as forbidden. Public storefront functions must be available to Guests without authentication.
- **FR-004:** The system should allow a user to reset a forgotten password through an emailed single-use link that is valid for 60 minutes.
- **FR-005:** Customers must be able to view and edit their profile (name, phone number, preferred language, and preferred currency) and to add, edit, and delete saved shipping addresses and mark one as the default.
- **FR-006:** Admin accounts must be created only by an initial seeded Admin or by an existing Admin who holds the platform configuration permission. Public registration must not create Admin accounts.
- **FR-007:** The system must support Admin permission groups of at least: catalog and inventory, orders and fulfillment, promotions, finance (COD settlement and wallet ledger), and platform configuration (currencies, exchange rates, taxes, shipping charges, and Admin accounts). An action outside an Admin's permission groups must be rejected.
- **FR-008:** Admins must be able to search and view Customer accounts and deactivate or reactivate them. A deactivated Customer must not be able to log in.
- **FR-009:** The system must record an audit entry (Admin, timestamp, action, entity, and old and new values) for every Admin create, change, delete, and status-change action on products, prices, stock, orders, promotions, currencies, exchange rates, taxes, shipping charges, Customer accounts, COD collections, remittances, and settlement allocations. Audit entries must not be editable or deletable through the application.

### 3.2 Company product catalog, pricing, and inventory

- **FR-010:** Admins must be able to create, edit, and archive product categories. Archived categories must not appear in the storefront.
- **FR-011:** Admins must be able to create and edit a product containing a title and description in English and Arabic, a category, style tags, images, and one or more variants. Each variant must have a unique SKU and attributes such as size and color.
- **FR-012:** Product images must be JPEG, PNG, or WebP files of at most 5 MB each. A product must have at least one image before it can be published.
- **FR-013:** A product must have the status `DRAFT`, `PUBLISHED`, or `ARCHIVED`. Only `PUBLISHED` products must be visible in the storefront. Removing a product must archive it. An archived product must remain referenced by existing orders and returns and must not be addable to a cart.
- **FR-014:** Admins must be able to set a price for each variant in the base currency, as a positive amount in minor units, and may set explicit prices in other supported currencies.
- **FR-015:** Admins must be able to record stock received for a variant. Registering received stock must increase both the on-hand quantity and the available quantity by the received quantity.
- **FR-016:** Admins must be able to make authorized stock adjustments (increase or decrease) to a variant's on-hand quantity with a mandatory reason. An adjustment that would make the on-hand quantity less than the reserved quantity must be rejected. Every stock receipt and adjustment must be saved in a stock movement history showing variant, quantity change, reason, Admin, and timestamp.
- **FR-017:** The system must track on-hand and reserved quantities for each product variant and must calculate available stock as on-hand quantity minus reserved quantity.
- **FR-018:** The reserved quantity must never exceed the on-hand quantity and the available stock must never be negative. An operation that would violate this must be rejected and must leave all quantities unchanged.
- **FR-019:** At checkout, the system must reserve the required quantity for every order line atomically in one database transaction. Reserving stock must increase the reserved quantity and reduce the available quantity without changing the on-hand quantity. If any line cannot be fully reserved, no line must be reserved and the Customer must be told which items are unavailable.
- **FR-020:** The system must not allow concurrent checkouts to reserve more units than are available. A request that cannot be reserved must be rejected with an out-of-stock message.
- **FR-021:** A stock reservation must remain in place when all required prepaid payment components succeed, or when an order with no prepaid component is accepted for fulfillment.
- **FR-022:** The system must release a stock reservation exactly once when a required prepaid payment component fails or expires, or when an order is cancelled before it is packed. Releasing a reservation must reduce the reserved quantity without changing the on-hand quantity.
- **FR-023:** When an order transitions to `SHIPPED`, the system must reduce both the on-hand quantity and the reserved quantity of each order line by the line quantity. This operation must occur exactly once per order. The on-hand quantity must not be reduced merely because an order is paid, confirmed, or packed.
- **FR-024:** Inventory reservations, releases, dispatch deductions, stock receipts, adjustments, approved-return restocking, and failed-delivery restocking must be applied atomically and idempotently in the database. Repeating the same operation for the same order and operation type must not change quantities a second time.
- **FR-025:** Inventory availability and reservation must not depend on RabbitMQ or any other asynchronous message.

### 3.3 Storefront and personalization

- **FR-026:** Customers and Guests must be able to browse `PUBLISHED` products by category and by style tag and to view product detail pages showing title, description, images, variants, price in the selected currency, and availability. Product lists must be paginated (default 24 products per page) and sortable by newest, price ascending, and price descending.
- **FR-027:** A variant whose available quantity is zero must be shown as "Out of stock" and must not be addable to a cart.
- **FR-028:** Customers and Guests must be able to search by keyword (case-insensitive match on product title, description, and variant SKU, in English or Arabic) and filter results by category, price range, and size. Search results must be paginated, and a search without results must show a "no results" message.
- **FR-029:** Customers must be able to save style preferences, including favorite styles, colors, and sizes. The storefront must display a personalized list of `PUBLISHED` products that match at least one saved preference (style tag, color, or size), newest first. A Customer without saved preferences must be prompted to set them.

### 3.4 Cart and checkout

- **FR-030:** Guests and Customers must be able to add, update, and remove cart items. The quantity of a cart line must be a whole number from 1 to 99.
- **FR-031:** A Guest cart must be kept for at least 30 days on the same browser. When a Guest logs in or registers, the Guest cart must be merged into the Customer's cart; quantities of the same variant must be added together up to the maximum of 99. A Customer's cart must persist across sessions.
- **FR-032:** The cart must show line prices, discounts, and subtotal in the selected currency and must update when its contents or applicable prices and discounts change.
- **FR-033:** A Guest who starts checkout must be required to log in or register, and the cart must be preserved.
- **FR-034:** Checkout must (a) collect a shipping address in a supported destination country and the payment method (and optional gift card), (b) revalidate current prices, discounts, and stock, (c) calculate tax, shipping charge, and COD charge, (d) reserve the required stock, and (e) create the order. If prices or discounts changed since the Customer last viewed the cart, the Customer must be shown the updated totals and must confirm them before the order is created.
- **FR-035:** Each checkout submission must carry a client-generated idempotency key. Repeating a submission with the same key (through a retry or double-click) must return the original result and must create no more than one order and no more than one stock reservation.
- **FR-036:** Admins must be able to configure, for each supported destination country, a flat shipping charge for each supported currency. Checkout to a country without a configured shipping charge must be rejected.
- **FR-037:** Admins must be able to configure a COD charge (a fixed amount for each supported currency, which may be zero) that applies when an order includes a COD component.

### 3.5 Marketing: discounts, vouchers, and gift cards

- **FR-038:** Admins must be able to create percentage discounts (1 to 100 percent) and fixed-amount discounts with start and end date-times, applicable to selected products, selected categories, or the whole order. A discount must apply only between its start and end date-times. A discount may belong to a named seasonal campaign (for example "Black Friday"); while the campaign is active, the storefront must show the campaign name and the reduced price next to the original price for the affected products.
- **FR-039:** For each order line, at most one product or category discount must apply: the one giving the largest reduction. At most one voucher and at most one gift card may be used on an order. An attempt to apply a second voucher or a second gift card must be rejected with an error message and must leave the order unchanged. A voucher applies after automatic discounts. Seasonal discounts, one voucher, one gift card, and the single standard payment method may all be used on the same order. They must be applied in this order: line discounts, then the voucher, then tax calculation, and finally the gift card, which is a payment and not a price reduction. A discount must never reduce a line below zero, and the order total must never be negative.
- **FR-040:** The system must support voucher codes (unique, case-insensitive) with an expiry date, a total usage limit, a per-Customer usage limit, and a minimum spend. An invalid voucher must be rejected with the reason (not found, expired, usage limit reached, or minimum spend not met). A voucher use must be counted when an order is created and must be restored if that order is cancelled.
- **FR-041:** The system must support a cart rule that sets the shipping charge to zero when the merchandise amount after discounts reaches a threshold configured by an Admin for the order currency.
- **FR-042:** The system should support gift cards. An Admin must be able to issue a gift card with a unique code, value, currency, and expiry date. A Customer must be able to redeem it at checkout as a `GIFT_CARD` payment component in the gift card's currency for an amount up to the remaining balance (see FR-061). The balance must decrease by the amount applied and must be restored by any refund of that component.
- **FR-043:** Order-level discounts must be allocated to order lines in proportion to each line's value after line-level discounts. The rounding remainder must be assigned to the last line. The line amount after allocated discounts must be used for tax calculation, returns, and settlement.

### 3.6 Orders and fulfillment

- **FR-044:** An order must contain a unique order number, the Customer, the order date, the order lines (variant, SKU, name, quantity, unit price, discount, and tax), the shipping address, currency, item subtotal, discount total, tax, shipping charge, COD charge (zero if no COD component), final total, the payment components, and an order status.
- **FR-045:** The order must retain the prices, discounts, tax rate and tax amount for each line, shipping charge, COD charge, currency, and, where a price was converted, the exchange rate and converted amount used at checkout.
- **FR-046:** Changes to product prices, discounts, tax rates, shipping charges, or exchange rates after checkout must not change the financial amounts recorded for an existing order.
- **FR-047:** An order with at least one prepaid component must be created as `PENDING_PAYMENT` and must become `CONFIRMED` when all its prepaid components are `PAID`. It then follows the status sequence `CONFIRMED → PACKED → SHIPPED → DELIVERED`.
- **FR-048:** An order with no prepaid component must be created as `CONFIRMED` (accepted for fulfillment) while its COD component remains `PENDING_COLLECTION`. `CONFIRMED` does not mean that a COD order has been paid.
- **FR-049:** The only allowed status transitions must be: `PENDING_PAYMENT → CONFIRMED`, `PENDING_PAYMENT → CANCELLED`, `CONFIRMED → PACKED`, `CONFIRMED → CANCELLED`, `PACKED → SHIPPED`, `SHIPPED → DELIVERED`, and `SHIPPED → DELIVERY_FAILED` (COD orders only). Any other transition must be rejected with an error and leave the order unchanged.
- **FR-050:** A prepaid order whose required payment fails or expires must be cancelled and its stock reservation released (see FR-022 and FR-068).
- **FR-051:** Customers must be able to view their order history, newest first and paginated, and the details of each order, including items, amounts, status, and payment components.
- **FR-052:** A Customer must be able to cancel their own order while it is `PENDING_PAYMENT` or `CONFIRMED`. The cancellation must take effect immediately without Admin approval and must be rejected once the order is `PACKED` or later.
- **FR-053:** A COD order cancelled before it is packed must release the reserved stock and must create no refund, collection, or settlement record.
- **FR-054:** An authorized Admin must be able to mark a `SHIPPED` COD order as `DELIVERY_FAILED` when the shipping provider reports that the order was refused or could not be delivered, and must record a reason (refused by the Customer, Customer unreachable, address not found, or other). An order for which a collection confirmation has been recorded cannot be marked `DELIVERY_FAILED`.
- **FR-055:** When an order becomes `DELIVERY_FAILED`, the system must: (a) restore stock exactly once by increasing the on-hand and available quantities of each order line by the line quantity, reversing the dispatch deduction; (b) set the COD payment component to `CANCELLED`; (c) create no collection, remittance, tax allocation, or refund record for the order; and (d) notify the Customer.
- **FR-056:** The system must count the `DELIVERY_FAILED` orders of each Customer. When the count reaches 3 (configurable threshold), the system must flag the Customer account for review and alert authorized Admins, and an authorized Admin may then deactivate the account (see FR-008).
- **FR-057:** When an order becomes `CONFIRMED`, the system must publish an order-processing event through the outbox pattern (see NFR-019), and a RabbitMQ consumer must create exactly one fulfillment task for the order within 30 seconds in the reference test environment.
- **FR-058:** A message that still fails after the maximum number of processing attempts (default 5, with increasing delays between attempts) must be moved to a dead-letter queue. An Admin with the platform configuration permission must be able to replay dead-lettered messages, and a replay must not create duplicate fulfillment tasks.
- **FR-059:** Admins must be able to list orders, filter them by status, date range, and payment status, and advance the order status along the allowed transitions. Each status change must record the Admin and the timestamp.

### 3.7 Payments

- **FR-060:** Each order must be paid with exactly one standard payment method chosen by the Customer: `CARD` (through a hosted payment-gateway sandbox), `WALLET`, or `COD`. The system must not allow two standard payment methods, or more than one card, to be combined on one order, and a checkout submission that does so must be rejected with an error. A `WALLET` payment must cover the whole amount due and must be rejected if it exceeds the Customer's available wallet balance in the order currency; the Customer must then top up the wallet or choose another method.
- **FR-061:** If FR-042 is implemented, one gift card may be combined with the chosen standard payment method. At most one gift card may be used per order (see FR-039). The gift card amount must not exceed the gift card's balance or the amount due, and, when the standard method is `COD`, must not reduce the COD component below the shipping charge plus the COD charge. If the gift card covers the whole amount due, no standard payment method is required. The gift card component must be `PAID` when the order is created.
- **FR-062:** The system must allocate the order's amounts to its payment components. If the order has a COD component, the full shipping charge and the COD charge must be allocated to it. The remaining merchandise amount, tax, and (when there is no COD component) shipping charge must be allocated between the standard component and the gift card component in proportion to each component's amount remaining after those deductions. A rounding remainder of at most one minor unit must be assigned to the merchandise amount.
- **FR-063:** Card payments must be processed through a hosted payment gateway. ExpressMe must store only the gateway reference and the payment details required to track the payment; it must not store card numbers or security codes.
- **FR-064:** The system must record each payment component with its amount, currency, method, refunded amount, gateway reference (where applicable), and status. The statuses must be `PENDING`, `PENDING_COLLECTION`, `PAID`, `FAILED`, `EXPIRED`, `CANCELLED`, `PARTIALLY_REFUNDED`, and `REFUNDED`.
- **FR-065:** A pending prepaid payment component must expire after a configurable payment window (default 30 minutes) and its status must become `EXPIRED`.
- **FR-066:** The system must verify the signature of every gateway callback. A callback with an invalid signature must be rejected, logged, and must change no state. A valid callback must be processed at most once per gateway event identifier.
- **FR-067:** An order must be considered fully paid only when every one of its payment components is `PAID`.
- **FR-068:** When an order is cancelled, every payment component that is not `PAID` must be set to `CANCELLED` (unless it is already `FAILED` or `EXPIRED`), and every prepaid component that is `PAID` must be refunded in full (see FR-087).

### 3.8 COD collection and settlement

- **FR-069:** When an order with a COD component is moved to `SHIPPED`, the Admin must select the designated shipping provider from a list of providers managed by Admins with the platform configuration permission. The system must record the provider against the order.
- **FR-070:** An authorized Admin must be able to record the shipping provider's collection confirmation for the COD component of an order that is `SHIPPED` or `DELIVERED`, including the order, provider, collection reference, gross amount collected, shipping charge retained, currency, and collection date. The collection reference must be unique for the provider. The gross amount collected must equal the COD component amount and the shipping charge retained must equal the shipping charge allocated to that component; otherwise the system must reject the confirmation and show the difference.
- **FR-071:** Once a collection confirmation is recorded, the COD payment component must be marked `PAID`. This confirms Customer payment only; it does not confirm receipt of funds by ExpressMe.
- **FR-072:** An authorized Admin must be able to record a shipping-provider remittance to ExpressMe, including the provider, bank deposit reference, deposit date, amount, currency, and the COD collections covered by the remittance. A remittance may cover one or more COD collections of the same provider and currency.
- **FR-073:** The expected net remittance for a COD collection must equal the gross amount collected minus the shipping charge retained by the provider. The amount expected for a remittance must equal the sum of the expected net remittances of the collections it covers.
- **FR-074:** A remittance must be moved to a confirmed state only by an explicit action of an authorized Admin that requires the bank deposit reference and deposit date, after the Admin has verified the deposit in ExpressMe's bank account.
- **FR-075:** The system must track, for each COD collection and each remittance, the amount expected, the amount received, and the outstanding or unmatched amount.
- **FR-076:** The system must prevent the same provider remittance reference or the same bank deposit reference from being recorded more than once.
- **FR-077:** The total amount allocated to a COD collection must not exceed the expected net remittance for that collection.
- **FR-078:** Remittance statuses must be `PENDING` (recorded and not yet confirmed), `RECEIVED` (confirmed and every covered collection fully reconciled), and `DISCREPANCY` (confirmed but the received amount differs from the expected amount). A remittance must be marked `RECEIVED` only when its allocated collections have been fully reconciled, meaning the amount allocated to each collection equals its expected net remittance.
- **FR-079:** A remittance in `DISCREPANCY` must remain open until an additional allocation or an adjustment recorded by an authorized Admin with a reason reconciles it, after which its status must become `RECEIVED`.
- **FR-080:** After the funds for a COD collection have been received and reconciled, the system must allocate the net remittance for that collection into three separately recorded amounts: the merchandise amount after allocated discounts, recorded as ExpressMe sales proceeds; the tax amount, recorded as tax collected by ExpressMe; and the COD charge, recorded as COD charge income. The three amounts must add up to the net remittance for the collection.
- **FR-081:** The system must maintain traceable links between each COD order, provider collection confirmation, shipping-charge deduction, provider remittance, and tax allocation. It must separately record the gross amount collected, the shipping charge retained by the provider, the net amount remitted to ExpressMe, the merchandise amount, the COD charge, and the tax amount. The system must also retain the allocation of each remitted amount to its COD collections and orders.
- **FR-082:** For each COD collection, the amount collected from the Customer must equal the shipping charge retained by the provider plus the net amount due to ExpressMe, and the net amount received by ExpressMe must equal the merchandise amount after discounts plus the COD charge plus the tax amount. Any difference must be flagged as an outstanding amount or discrepancy and must appear in a COD reconciliation report available to authorized Admins.

### 3.9 Returns and refunds

- **FR-083:** Customers must be able to request a return for a delivered order line within a configurable return window (default 14 days from the order's `DELIVERED` date), selecting a reason from a configurable list and optionally adding a comment. The requested quantity must not exceed the delivered quantity minus the quantity already returned. A request outside the return window must be rejected.
- **FR-084:** A return request must have the status `REQUESTED`, `APPROVED`, `REJECTED`, `RECEIVED`, or `COMPLETED`.
- **FR-085:** Admins must be able to approve or reject a return request. A rejection must require a reason, and the Customer must be notified of the decision.
- **FR-086:** The refund amount for an approved return must equal the returned quantity multiplied by the unit price after allocated discounts, plus the tax on those units. The shipping charge and COD charge must not be refunded in v1.
- **FR-087:** On approval, the system must initiate the refund and allocate it across the order's `PAID` payment components in proportion to their amounts, without exceeding the amount collected through a component minus what was already refunded. A `WALLET` amount must be credited to the Customer's wallet in the same currency through a reversing ledger transaction (see FR-113); a `CARD` amount must be refunded through the payment-gateway sandbox; a `GIFT_CARD` amount must be restored to the gift card balance. A refund of a collected COD amount must be handled outside ExpressMe and recorded by an authorized Admin when confirmed, with the confirmation reference and date.
- **FR-088:** The system must retain the original payment and settlement records and record each refund as a separate transaction linked to them. A payment component must be set to `PARTIALLY_REFUNDED` or `REFUNDED` according to the refunded amount.
- **FR-089:** A returned item must be added back to on-hand stock only after it has been received, inspected, and approved as suitable for resale. An item not approved must not be restocked.

### 3.10 Languages, currencies, and tax

- **FR-090:** The user interface and system messages must support English and Arabic. The Customer must be able to select the language and the selection must be remembered. English must be the default, and Arabic pages must use a right-to-left layout.
- **FR-091:** Where a product has no Arabic title or description, the storefront must show the English text.
- **FR-092:** Admins must be able to manage supported currencies (ISO 4217), including exactly one base currency, and exchange rates from the base currency to each other supported currency. An exchange rate must have an effective date-time and a positive value of up to six decimal places.
- **FR-093:** Customers must be able to select one of the supported currencies, with the base currency as default. Prices must be displayed in the selected currency, and the order currency must be the currency selected at checkout.
- **FR-094:** The system must use an explicitly configured product price in the selected currency where available and otherwise convert the base price using the active exchange rate. The exchange rate and converted amount must be retained with the order (see FR-045).
- **FR-095:** Converted amounts and calculated tax amounts must be rounded half up to the minor unit of the currency as defined by ISO 4217.
- **FR-096:** Admins must be able to define a tax rate (0 to 100 percent, up to two decimal places) for each supported destination country. Checkout to a country with no configured tax rate must be rejected.
- **FR-097:** The system must calculate tax at checkout for each order line as the line amount after discounts multiplied by the tax rate of the shipping destination country, rounded according to FR-095. The order tax must be the sum of the line taxes. Shipping charge and COD charge must not be taxed in v1. Discounts must be applied before tax is calculated.
- **FR-098:** The system must record tax amounts separately from merchandise sales, shipping charges, and COD charges. Tax must not be included in ExpressMe's merchandise sales amount.

### 3.11 SEO, API, and notifications

- **FR-099:** Product and category pages must have clean URLs containing a readable name (for example `/products/{slug}`), a unique page title, a unique meta description, and a canonical URL. The system must publish an XML sitemap listing all published product and category pages and updated when a product or category is published or archived. Page content must be available to search-engine crawlers without client-side scripting (server-side or pre-rendered pages).
- **FR-100:** Every function available in the user interface must also be available through versioned REST endpoints (under `/api/v1`) that are documented in an OpenAPI 3 specification. Errors must be returned in a consistent format with an HTTP status code and a message, and list endpoints must support pagination.
- **FR-101:** The system must send an email notification to the Customer, in the Customer's selected language, when: the account is registered, an order is confirmed, shipped, delivered, or cancelled, a payment fails or expires, a refund is issued, a COD delivery fails, and a return is approved or rejected.
- **FR-102:** A notification delivery failure must not prevent the related business operation from completing, and the failed delivery must be retried up to 5 times with increasing delays.

### 3.12 Digital wallet and double-entry wallet ledger

The wallet ledger records simulated wallet movements only. It is not a full general ledger. Wallet amounts must be represented in the relevant currency's minor units. The ledger supports Customer wallet accounts and ExpressMe platform accounts, classified as follows:

| Account | Type | Debit means | Credit means |
| --- | --- | --- | --- |
| Customer wallet | Liability | Wallet balance decreases | Wallet balance increases |
| Funding/clearing (simulated) | Asset | Simulated funds received | Simulated funds returned |
| Merchandise sales | Revenue | Sales reversed | Sales recognized |
| Tax clearing | Liability | Tax liability reduced | Tax liability increased |
| Shipping clearing | Liability | Shipping amount reduced | Shipping amount increased |

COD settlement records are kept by the Settlement module and are not posted to the wallet ledger.

- **FR-103:** A Customer must have a wallet account in each supported currency, created when the Customer first uses that currency for a wallet transaction.
- **FR-104:** The platform must have the corresponding internal accounts for simulated funding/clearing, merchandise sales, tax clearing, and shipping clearing (see the account table above).
- **FR-105:** Every posted wallet movement must be recorded as a ledger transaction containing at least two entries. Total debits must equal total credits for each transaction and currency. The system must reject an unbalanced transaction and must persist none of its entries.
- **FR-106:** Posted ledger transactions and entries must be append-only and must not be edited or deleted. This must be enforced in the database, for example by giving the application's database user only insert and select privileges on the ledger tables, so that an attempted update or delete fails.
- **FR-107:** Corrections must be recorded through reversing or correcting transactions that reference the original transaction.
- **FR-108:** Customers must be able to top up their wallets by card through the payment-gateway sandbox, within a minimum and maximum top-up amount configured by an Admin. Out-of-range amounts must be rejected.
- **FR-109:** A top-up must be credited to the wallet only after the sandbox gateway confirms successful payment, and a repeated confirmation for the same top-up must not credit the wallet twice.
- **FR-110:** A confirmed top-up must post one balanced ledger transaction that debits the simulated funding/clearing account and credits the Customer's wallet for the top-up amount.
- **FR-111:** A Customer must be able to pay an order with the wallet, as the order's standard payment method, using a wallet account in the order currency. The wallet debit must be posted when the order is created. The system must not convert wallet balances between currencies.
- **FR-112:** For the `WALLET` component of an order, the system must post one balanced ledger transaction that debits the Customer's wallet and credits the merchandise sales, tax clearing, and shipping clearing accounts for the amounts allocated to that component (see FR-062). The total credits must equal the Customer's wallet debit.
- **FR-113:** A refund of a wallet-funded amount must post a reversing ledger transaction that credits the Customer's wallet and references the original purchase transaction.
- **FR-114:** A wallet balance must be derived from ledger entries as total credits minus total debits for the wallet account. Any stored balance must be updated in the same database transaction as the ledger entries.
- **FR-115:** Each wallet operation must carry an idempotency key that uniquely identifies the requested operation. Repeating a request with the same key and the same content must return the original result without posting the operation again. Reusing a key with different content must be rejected.
- **FR-116:** Wallet operations must be safe under concurrent requests. A wallet payment that would make a Customer's wallet balance negative must be rejected with an insufficient-funds error.
- **FR-117:** A scheduled reconciliation job must run at least once every 24 hours, verify that each ledger transaction balances and that each stored wallet balance matches the sum of its ledger entries, and raise an alert (an ERROR log entry and an alert record visible to authorized Admins) when a mismatch is found.
- **FR-118:** Reconciliation must not change posted ledger entries or balances.
- **FR-119:** Customers must be able to view their wallet balance for each currency and their wallet transaction history (type, amount, currency, date, and related order or top-up reference), paginated and newest first.
- **FR-120:** Authorized Admins must be able to view the wallet ledger, filtered by account, date range, transaction type, and order reference. The view must be read-only.

## 4. Non-Functional Requirements

### 4.1 Performance, caching, and load testing

- **NFR-001:** Performance: Catalog read pages (category lists, product detail, and search) must respond in under 500 ms at the 95th percentile at a load of 1,000,000 concurrent users. This target must be verified by a distributed load test against a scaled-out deployment (several application instances behind a load balancer, with Redis caching) whose configuration, number of instances, and test environment are recorded with the results.
- **NFR-002:** Performance on the reference environment: on the single-host Docker Compose reference environment, catalog read pages must respond in under 500 ms at the 95th percentile with at least 1,000 concurrent virtual users and under 1% failed requests. The maximum concurrency reached while still meeting the 500 ms threshold must be recorded in the load-test report.
- **NFR-003:** Reliability: Each database operation that creates or updates an order, payment record, stock reservation, wallet transaction, COD collection, provider shipping-charge deduction, remittance, or tax allocation must be atomic within the database. A failure must not leave partially written records or inconsistent stock or settlement records. Operations involving external providers must use safe retry and duplicate-prevention behavior where required.
- **NFR-004:** Scalability: The application must be stateless (no session or cart state held in application memory) so that multiple application instances can run behind a load balancer. Two instances behind a load balancer must serve requests of the same user session correctly.
- **NFR-005:** Caching: The system must use Redis with the cache-aside pattern for catalog, category, promotion, exchange-rate, and tax data. The cache hit ratio must be at least 90% for catalog reads in load-test scenario (a) of NFR-008.
- **NFR-006:** Cache correctness: Cached entries must have a time-to-live (TTL) and must be invalidated within 5 seconds when their source data changes. The system should use TTL jitter and protection against cache stampedes for frequently requested keys. Wallet balances and ledger data must never be served from cache. Available stock, calculated from on-hand stock minus reserved stock, and the prices, discounts, tax rates, shipping charges, and exchange rates used to create an order must be read from the database during checkout, not from the cache.
- **NFR-007:** Cache failure: If Redis is unavailable, the system must fall back to the database for data that can be read from the database, without failing otherwise valid requests. Calls to Redis must time out after at most 200 ms.
- **NFR-008:** Load testing: The system must be load-tested using k6 or Gatling with three scenarios: (a) browse-heavy traffic at the load defined in NFR-002, run for at least 10 minutes at steady load; (b) a flash sale with 1,000 buyers competing for 10 units; and (c) 100 parallel wallet payments against wallets funded for exactly 10 of them. Scenarios (a) and (b) must be measured with cache enabled and disabled. Scenario (b) must result in exactly 10 reserved units and no negative stock; scenario (c) must result in exactly 10 successful payments and balanced ledger transactions. Reports must record p50, p95, and p99 latency, throughput, and error rate.
- **NFR-009:** Database tuning: During load testing, every query whose 95th-percentile execution time exceeds 100 ms must be analyzed (for example with `EXPLAIN ANALYZE`) and, where a fix is possible, improved through indexing, removal of N+1 queries, or connection-pool tuning. Before-and-after results must be documented in the README.

### 4.2 Security and privacy

- **NFR-010:** Security: Passwords must be stored using bcrypt (cost factor at least 10) or Argon2id. Traffic must use HTTPS (TLS 1.2 or later). All inputs must be validated on the server, and users must only access data and perform actions permitted by their roles. The application must mitigate the OWASP Top 10 risks, and the build must fail on dependencies with known critical vulnerabilities.
- **NFR-011:** Brute-force protection: After 5 consecutive failed login attempts for an account within 15 minutes, further login attempts for that account must be rejected for 15 minutes.
- **NFR-012:** Privacy: A Customer should be able to request deletion of their account. Personal data must then be anonymized, while orders, payments, ledger entries, and settlement records are retained without personal identifiers.
- **NFR-013:** Retention: Orders, payment records, ledger entries, and COD settlement records must be retained for at least 7 years and must not be physically deleted by any application function.

### 4.3 Usability and quality

- **NFR-014:** Usability: The storefront must be responsive for screen widths from 360 px to 1920 px and must work in the current and previous major versions of Chrome, Firefox, Safari, and Edge. An authorized Admin must be able to add a product with one variant in no more than five form steps or screens.
- **NFR-015:** Quality: Core business logic must have at least 70% line coverage from unit tests, measured by a coverage tool. Critical workflows, including checkout, inventory, payments, wallet operations, and COD settlement, must also have integration tests using Testcontainers with PostgreSQL, Redis, and RabbitMQ where applicable.
- **NFR-016:** Deployability: One `docker compose up` command must start the application, PostgreSQL, Redis, and RabbitMQ and initialize the required database schema through versioned migrations.
- **NFR-017:** Backup and recovery: A documented procedure must exist to back up and restore the PostgreSQL database, and the restore must be tested once before the first release.

### 4.4 Architecture and integration

- **NFR-018:** Module boundaries and data ownership: The v1 application must use one PostgreSQL database with two schemas. The `finance` schema must hold money data: the Wallet module's accounts, ledger entries, and idempotency records, the Payments module's payment records, and the Settlement module's COD collections, remittances, and allocations. The `app` schema must hold all other data. Each module must own its tables within its schema, and other modules must access its data through the owning module's published application interfaces rather than accessing its tables or implementation directly. The application's database user must have only insert and select privileges on the ledger tables (see FR-106). Automated tests, such as Spring Modulith or ArchUnit, must enforce these boundaries.
- **NFR-019:** Messaging: RabbitMQ must be used for asynchronous order processing. Events must be published using the outbox pattern. Consumers must be idempotent and must support retries and a dead-letter queue (see FR-058).
- **NFR-020:** External integration resilience: Calls to the payment gateway and the email provider must use timeouts (default 10 seconds). Retries must be used only where they cannot create duplicate payments or notifications; idempotency keys or provider references must be used to prevent duplicate processing. For v1, shipping-provider collection confirmations and bank deposits or transfers must be recorded by an authorized Admin; direct integrations with shipping providers or banks are not required.
- **NFR-021:** Observability: Every application request and every asynchronous order-processing message must carry a correlation ID (the `X-Correlation-ID` header or message property). The system must produce structured logs including the correlation ID. It must expose health checks and metrics for request rate, latency, and error rate, database connection-pool usage, cache hit ratio, RabbitMQ queue depth, and dead-letter queue size.

## 5. Acceptance Criteria

Each criterion lists the requirements it verifies.

1. An Admin can publish a product with two variants and prices in two currencies, and a Customer can find the product through search. (Verifies FR-011, FR-013, FR-014, FR-028)
2. A Customer can add a product to the cart, apply a valid voucher, pay by card through the sandbox, and see the order in the `CONFIRMED` state with its payment recorded as `PAID`. (Verifies FR-030, FR-040, FR-034, FR-047, FR-063)
3. When two Customers attempt to buy the last available unit at the same time, only one can reserve it and the other is told that the item is out of stock. (Verifies FR-019, FR-020)
4. Retrying the same checkout request with the same idempotency key creates no more than one order and does not reserve stock more than once. (Verifies FR-035)
5. A Customer can request a return for a delivered order line. After approval, a refund is initiated through the original payment method for collected sandbox card or wallet amounts. A refund for a collected COD amount is recorded after it is handled outside ExpressMe. The item is restocked only if it is assessed as suitable for resale. (Verifies FR-083, FR-085, FR-087, FR-089)
6. Two concurrent wallet payments of 40 against a Customer wallet balance of 50 result in one successful payment and one rejected payment. The wallet balance never becomes negative. (Verifies FR-116, FR-111)
7. After wallet top-ups, purchases, refunds, and reversals, every posted ledger transaction balances within its currency, and every stored wallet balance matches the ledger entries. (Verifies FR-105, FR-114, FR-117)
8. A posted ledger transaction cannot be edited or deleted, including by an attempted direct database update or delete with the application's database user. A correction creates a separate reversing or correcting transaction linked to the original. (Verifies FR-106, FR-107)
9. With Redis stopped, catalog pages continue to load using the database, and valid requests do not fail solely because Redis is unavailable. (Verifies NFR-007)
10. In the flash-sale test with 1,000 competing buyers for 10 units, exactly 10 units are reserved, 990 buyers receive an out-of-stock response, and stock never becomes negative. (Verifies FR-020, NFR-008)
11. If an order is paid with a gift card and a card, and the card payment fails, the order is cancelled and is not considered fully paid. The gift card amount is restored to the gift card balance, and the stock reservation is released. (Verifies FR-050, FR-068, FR-022, FR-067, FR-061)
12. If the email provider is unavailable, order processing still completes. Failed notifications are retried and are delivered after the provider becomes available. (Verifies FR-102)
13. If an order-processing event is delivered more than once, the consumer processes its business effect only once. (Verifies FR-057, NFR-019)
14. A correlation ID can be used to trace a checkout request through the application logs and the related order-processing message. (Verifies NFR-021)
15. If the order-processing consumer is stopped, committed messages remain available in RabbitMQ and are processed when the consumer restarts. A message replay does not create duplicate fulfillment tasks. (Verifies FR-057, FR-058)
16. A COD-only order is created as `CONFIRMED` and processed for fulfillment while its payment remains `PENDING_COLLECTION`. When an authorized Admin records the shipping provider's confirmation that cash was collected from the Customer, the payment status changes to `PAID`, while the provider remittance to ExpressMe remains separately outstanding. (Verifies FR-048, FR-070, FR-071)
17. If an exchange rate, tax rate, or shipping charge changes after an order is placed, the currency, rate, tax, shipping, and total stored for that order remain unchanged. (Verifies FR-045, FR-046)
18. A checkout that selects two standard payment methods (for example wallet and card), two cards, or two gift cards is rejected. An order paid with a card and one gift card records two payment components whose total equals the order total, and any refund does not exceed the amount collected through the corresponding component. (Verifies FR-060, FR-061, FR-064, FR-087)
19. The load-test report includes measured results for the defined scenarios, including cache-on versus cache-off results for browse-heavy and flash-sale traffic. (Verifies NFR-008, NFR-002, NFR-001)
20. An authorized Admin can record a shipping-provider remittance covering one or more COD collections. For each collection, the system uses the shipping charge retained by the provider and calculates the expected net remittance as the COD amount collected less that charge. The system records the bank deposit reference and amount, allocates the received amount to the covered collections, and does not mark the remittance `RECEIVED` until the expected net amounts are fully reconciled. (Verifies FR-072, FR-073, FR-074, FR-078)
21. After COD funds are received and reconciled, ExpressMe records the merchandise amount after allocated discounts as sales proceeds, the tax amount separately as tax collected, and the COD charge as COD charge income. The shipping charge is retained by the provider before remittance and is not included in the amount received by ExpressMe. (Verifies FR-080, FR-098)
22. COD settlement records link the order, shipping-provider collection, shipping-charge deduction, remittance received by ExpressMe, and tax allocation. The amount collected from the Customer equals the shipping charge retained by the provider plus the net amount due to ExpressMe. The net amount received by ExpressMe reconciles to the COD-funded merchandise amount after discounts plus the COD charge and the tax amount. Any difference is flagged for reconciliation. (Verifies FR-081, FR-082)
23. For a COD-only order with $100 in merchandise after discounts, $5 tax, $5 shipping, and a $2 COD charge, the provider collects $112, retains $5 shipping, and remits $107 to ExpressMe. ExpressMe records $100 as merchandise sales proceeds, $5 as tax collected, and $2 as COD charge income. These allocations total the $107 received. (Verifies FR-073, FR-080)
24. For an order paid with COD and a gift card, the full shipping charge and the COD charge are allocated to the COD component. The gift card amount is limited so that the COD component is never less than the shipping charge plus the COD charge. The expected provider remittance equals the COD amount collected less the shipping charge retained. (Verifies FR-061, FR-062, FR-073)
25. If the system has 100 units on hand and none reserved, and a Customer checks out with 15 units, the system records 100 units on hand, 15 reserved, and 85 available. When the order is shipped, the system records 85 units on hand, zero reserved, and 85 available. (Verifies FR-019, FR-023, FR-017)
26. If an order with 15 reserved units is cancelled before packing, the system releases the 15-unit reservation, leaves on-hand stock unchanged, and increases available stock by 15. (Verifies FR-022, FR-049)
27. A Customer who cancels an order while it is `CONFIRMED` sees it become `CANCELLED` immediately, while an attempt to cancel an order that is `PACKED` is rejected, and every paid prepaid component of the cancelled order is refunded. (Verifies FR-052, FR-049, FR-068)
28. A Guest adds items to a cart, logs in, and finds the items merged into the Customer's cart. Starting checkout as a Guest requires logging in or registering. (Verifies FR-031, FR-033)
29. A voucher that is expired, below its minimum spend, or past its usage limit is rejected with the matching reason. After one voucher is applied to an order, an attempt to apply a second voucher is rejected, and an attempt to use a second gift card is also rejected. (Verifies FR-040, FR-039)
30. An Admin without the finance permission cannot record a COD collection or view the wallet ledger, and every Admin change to a price or stock quantity appears in the audit log with the Admin and timestamp. (Verifies FR-007, FR-009)
31. While a seasonal campaign discount is active, an order can use that discount, one valid voucher, one gift card, and one card payment together. The discount and voucher reduce the price before tax is calculated, and the gift card reduces only the amount to be paid. After the campaign end date-time, the discount no longer applies. (Verifies FR-038, FR-039, FR-061, FR-097)
32. When the shipping provider reports that a COD order could not be delivered, an authorized Admin marks it `DELIVERY_FAILED` with a reason. The on-hand and available quantities of its items are restored once, the COD component is set to `CANCELLED`, no collection or settlement record exists, and the Customer is notified. When the third `DELIVERY_FAILED` order of the same Customer is recorded, the account is flagged for review and an Admin can deactivate it. (Verifies FR-054, FR-055, FR-056, FR-049, FR-101)

## 6. Future Extensions (not in v1)

- A production deployment topology (load balancer, multiple application instances, database read replicas, and a CDN) to demonstrate the scale target in NFR-001, and observability dashboards.
- Customer analytics, reviews and ratings, social login, and third-party webhooks.
- Automated shipping-provider and bank integrations.
- Multi-currency wallet conversion.
- A full general ledger and broader accounting functionality, if required.
- Automated tax filing or tax-authority payment integrations, if required.
- Partial shipments, per-product tax rates, and carrier-calculated shipping rates.
- Extracting selected modules into separate services or databases if scale, operational needs, or isolation requirements justify it.

## 7. Confirmed Assumptions and Decisions

The following assumptions and decisions were confirmed by the author.

1. **COD charge:** The COD charge is a fee kept by ExpressMe, not taxed, and included in the remittance (see FR-037, FR-080).
2. **Payment methods:** Each order uses one standard payment method (card, wallet, or COD), optionally combined with one gift card. The wallet is treated as a standard method and cannot be combined with card or COD (see FR-060 and FR-061).
3. **Order creation:** An order with no prepaid component is created as `CONFIRMED` immediately (see FR-048).
4. **COD failed delivery:** A shipped COD order that is refused or cannot be delivered is marked `DELIVERY_FAILED`, its items are restored to stock, and the Customer is notified. When the same Customer reaches 3 failed COD deliveries, the account is flagged and may be suspended by an Admin (see FR-054, FR-055, and FR-056).
5. **Refund policy:** Refunds are initiated when a return is approved, and shipping and COD charges are not refunded (see FR-086).
6. **Discount stacking:** A seasonal discount, one voucher, one gift card, and one standard payment method can be used together; at most one voucher and one gift card per order, and one product or category discount per line (the largest) (see FR-039).
7. **Scale target:** 1,000,000 concurrent users is kept as the design target in NFR-001, but it cannot be demonstrated on a single-host Docker Compose setup. NFR-002 defines a target that can be measured on the reference environment.
8. **Defaults:** Return window of 14 days, payment window of 30 minutes, 7-year record retention, 5 processing attempts, and the other default values in the requirements are accepted as initial settings.
9. **Voucher treatment:** A voucher is a price discount applied before tax, not a payment method. Only the standard payment methods and gift cards are payment components (see FR-039 and FR-040).
10. **Database schemas:** One PostgreSQL database with two schemas: `finance` for money data (wallet, ledger, payments, COD settlement) and `app` for all other data (see NFR-018).
