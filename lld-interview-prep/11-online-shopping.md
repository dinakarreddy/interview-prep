# Problem 11 — Online Shopping (Amazon-lite)

The "kitchen sink" LLD problem. Tests your ability to combine many patterns into a coherent system: Strategy (pricing, payment, shipping), State (order lifecycle), Composite (cart with bundles), Repository (every entity), and Saga (distributed transactions across services).

This file's design is a **moderate scope** version of an e-commerce system — enough to exercise the patterns, not so much that it becomes architecture rather than design.

## Interview opener

> **Interviewer:** Design an e-commerce system like Amazon. Focus on the order lifecycle from cart to delivery.

---

## Clarification phase

> **You:** What's in scope? Cart, checkout, order, payment, fulfillment? Or also catalog browsing, search, recommendations?
>
> **Interviewer:** Cart through delivery is the focus. Browsing is out of scope.

> **You:** Single-warehouse or multi-warehouse fulfillment?
>
> **Interviewer:** Multi-warehouse — but you don't need the warehouse-selection algorithm. Just model that a shipment may come from one of N locations.

> **You:** Inventory — should I worry about overselling?
>
> **Interviewer:** Yes — that's a key concurrency concern.

> **You:** Pricing — flat per item, or also discounts, taxes, shipping fees?
>
> **Interviewer:** All of those. Pluggable.

> **You:** Payment methods — card, wallet, COD?
>
> **Interviewer:** Yes. Pluggable.

> **You:** Order modifications — can users edit / cancel after placement?
>
> **Interviewer:** Cancel until shipped; partial refund.

> **You:** Returns / refunds after delivery?
>
> **Interviewer:** Mention but skip implementation.

> **You:** Single-currency, single-region?
>
> **Interviewer:** Yes for the core.

> **You:** Microservices architecture or monolith?
>
> **Interviewer:** Design as logical services (Order, Inventory, Payment, Shipping). Whether they're physically separated is your call — discuss the trade-off.

> **You:** Persistence?
>
> **Interviewer:** At the end.

OK — design needs: cart, polymorphic line items (bundles), order with state machine, pricing pipeline, payment + inventory + shipping coordination (Saga), refunds.

---

## Refined requirements

### Functional
1. Cart: add/remove items; bundles allowed; persistent across sessions.
2. Checkout: select shipping address + payment method.
3. Pricing pipeline: subtotal → discounts → tax → shipping → total.
4. Place order: atomically reserve inventory, charge payment, schedule shipment.
5. Order lifecycle: PLACED → CONFIRMED → SHIPPED → DELIVERED → CANCELLED / REFUNDED.
6. Cancel before shipment: release inventory, refund.

### Non-functional
- **No overselling**: inventory must be correct under concurrency.
- **No double-charge**: idempotent payment.
- **Fault tolerance**: payment failure rolls back inventory; inventory failure rolls back payment.

### Out of scope (mentioned)
- Catalog browsing, search, recommendations.
- Reviews / ratings.
- Returns and reverse logistics.
- Multi-currency, multi-region.
- Pricing calculations across promotions are simplified.

### Core operations
1. `Cart.add(productId, qty)` / `Cart.remove(...)` / `Cart.lineItems()`
2. `Cart.priceQuote()` → `PriceBreakdown`
3. `Cart.checkout(addressId, paymentMethodId)` → `Order`
4. `Order.cancel()` → `Refund`
5. `Order.markShipped()` (admin / fulfillment)
6. `Order.markDelivered()` (fulfillment)

---

## Domain model

### Entities (have ID)
- **`User`**.
- **`Product`** — catalog item.
- **`InventoryItem`** — per-product, per-warehouse stock count. (Aggregate root for inventory writes.)
- **`Cart`** — per user; transient until checkout.
- **`Order`** — aggregate root for the order lifecycle.
- **`Address`** — shipping address.
- **`PaymentMethod`** — card, wallet, COD.

### Value objects (immutable)
- **`Money(amount, currency)`**.
- **`LineItem(productId, qty, unitPrice)`**.
- **`PriceBreakdown(subtotal, discount, tax, shipping, total)`**.

### Enums
- `OrderStatus` — `PLACED`, `CONFIRMED`, `SHIPPED`, `DELIVERED`, `CANCELLED`, `REFUNDED`.
- `PaymentStatus` — `PENDING`, `AUTHORIZED`, `CAPTURED`, `REFUNDED`, `FAILED`.

### Aggregate boundaries
- `Order` is its own aggregate; references inventory, payment, shipping by ID.
- `InventoryItem` is its own aggregate (per product per warehouse).
- `Cart` is its own aggregate.
- Cross-aggregate updates are *eventually consistent* via events. The `OrderService` orchestrates with a Saga.

### Patterns
- **Strategy** — pricing rules, payment methods, shipping carriers.
- **State** — `Order` lifecycle.
- **Composite** — cart line items can be bundles (a bundle is a line item that contains other line items).
- **Repository** — per aggregate.
- **Chain of Responsibility** — pricing pipeline (each rule transforms the breakdown).
- **Saga** — checkout as a distributed transaction across Inventory + Payment + Order services.
- **Observer** — order events drive notifications, fulfillment, analytics.

---

## Class diagram

```
┌────────────────────────────┐
│           Cart             │
├────────────────────────────┤
│ -id; -userId               │
│ -lineItems : List<LineItem>│
├────────────────────────────┤
│ +add(productId, qty)       │
│ +remove(productId)         │
│ +priceQuote() : PriceBreak │
│ +checkout(...) : Order     │
└────────────────────────────┘
       │ ◇
       ▼
┌────────────────────────────────┐
│   <<abstract>> CartLineItem    │
├────────────────────────────────┤
│ -productId; -qty               │
├────────────────────────────────┤
│ +unitPrice() : Money           │
│ +totalPrice() : Money          │
└────────────────────────────────┘
        △
   ┌────┴─────┐
┌───────────────┐ ┌────────────────┐
│ ProductLineItem│ │ BundleLineItem │  ◆──► children: List<CartLineItem>
└───────────────┘ └────────────────┘

┌────────────────────────────────┐
│            Order               │
├────────────────────────────────┤
│ -id; -userId                   │
│ -lineItems; -shippingAddress   │
│ -paymentMethodId               │
│ -status : OrderStatus          │
│ -priceBreakdown : PriceBreakd. │
│ -version : long                │
├────────────────────────────────┤
│ +confirm() / ship() / etc.     │
└────────────────────────────────┘

┌────────────────────────────┐    ┌──────────────────────────────┐
│   InventoryItem            │    │   <<interface>>              │
├────────────────────────────┤    │   PricingRule                │
│ -productId; -warehouseId   │    ├──────────────────────────────┤
│ -availableQty; -reserved   │    │ +apply(b: PriceBreakdown)    │
│ -version : long            │    │   : PriceBreakdown           │
└────────────────────────────┘    └──────────────────────────────┘
                                          △
                                   ┌──────┼─────────────┬───────────┐
                              ┌──────┐  ┌──────┐    ┌──────┐    ┌──────┐
                              │Subtot│  │Discnt│    │ Tax  │    │Shipp.│
                              └──────┘  └──────┘    └──────┘    └──────┘

┌────────────────────────────┐    ┌────────────────────────────┐
│ <<interface>>              │    │ <<interface>>              │
│ PaymentGateway             │    │ ShippingCarrier            │
├────────────────────────────┤    ├────────────────────────────┤
│ +authorize / capture / ref │    │ +schedule / track          │
└────────────────────────────┘    └────────────────────────────┘

   record Money(BigDecimal amount, Currency currency)
   record PriceBreakdown(subtotal, discount, tax, shipping, total)
   enum OrderStatus { PLACED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED, REFUNDED }
```

---

## Key abstractions

```java
public interface PricingRule {
    PriceBreakdown apply(Cart cart, PriceBreakdown current);
}

public interface PaymentGateway {
    PaymentResult authorize(String userId, Money amount, String idempotencyKey);
    PaymentResult capture(String authRef, Money amount, String idempotencyKey);
    void refund(String captureRef, Money amount, String idempotencyKey);
}

public interface ShippingCarrier {
    ShipmentTracking schedule(Order order, Address address);
    ShipmentStatus track(String trackingNumber);
}

public interface InventoryService {
    /** Atomically reserve qty of product in any warehouse with stock. Returns reservation. */
    Reservation reserve(String productId, int qty, String idempotencyKey);
    /** Release a previously-made reservation. */
    void release(String reservationId);
    /** Convert a reservation to a fulfillment commitment. */
    void commit(String reservationId);
}
```

---

## Core code

### Value objects + enums

```java
public enum OrderStatus    { PLACED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED, REFUNDED }
public enum PaymentStatus  { PENDING, AUTHORIZED, CAPTURED, REFUNDED, FAILED }

public record Money(BigDecimal amount, Currency currency) {
    public Money plus(Money o)  { return new Money(amount.add(o.amount), currency); }
    public Money minus(Money o) { return new Money(amount.subtract(o.amount), currency); }
    public static Money zero(Currency c) { return new Money(BigDecimal.ZERO, c); }
}

public record PriceBreakdown(
    Money subtotal,
    Money discount,
    Money tax,
    Money shipping,
    Money total
) {
    public static PriceBreakdown empty(Currency c) {
        Money zero = Money.zero(c);
        return new PriceBreakdown(zero, zero, zero, zero, zero);
    }
}
```

### Cart with bundles (Composite)

```java
public abstract class CartLineItem {
    protected final String productId;
    protected final int qty;

    protected CartLineItem(String productId, int qty) {
        this.productId = productId; this.qty = qty;
    }

    public String productId()   { return productId; }
    public int qty()            { return qty; }

    public abstract Money totalPrice(ProductCatalog catalog);
    public abstract List<ResolvedItem> resolve(ProductCatalog catalog);

    public record ResolvedItem(String productId, int qty, Money unitPrice) { }
}

public final class ProductLineItem extends CartLineItem {
    public ProductLineItem(String productId, int qty) { super(productId, qty); }

    @Override
    public Money totalPrice(ProductCatalog catalog) {
        Money price = catalog.lookup(productId).price();
        return new Money(price.amount().multiply(BigDecimal.valueOf(qty)), price.currency());
    }

    @Override
    public List<ResolvedItem> resolve(ProductCatalog catalog) {
        Money unit = catalog.lookup(productId).price();
        return List.of(new ResolvedItem(productId, qty, unit));
    }
}

public final class BundleLineItem extends CartLineItem {
    private final List<CartLineItem> children;
    private final BigDecimal bundleDiscountPct;   // e.g., 0.10 for 10% off

    public BundleLineItem(String bundleId, int qty, List<CartLineItem> children, BigDecimal pct) {
        super(bundleId, qty);
        this.children = List.copyOf(children);
        this.bundleDiscountPct = pct;
    }

    @Override
    public Money totalPrice(ProductCatalog catalog) {
        Money sum = children.stream()
            .map(c -> c.totalPrice(catalog))
            .reduce(Money.zero(Currency.getInstance("USD")), Money::plus);   // assume same currency
        Money perBundleAfterDiscount = new Money(
            sum.amount().multiply(BigDecimal.ONE.subtract(bundleDiscountPct)),
            sum.currency()
        );
        return new Money(perBundleAfterDiscount.amount().multiply(BigDecimal.valueOf(qty)), sum.currency());
    }

    @Override
    public List<ResolvedItem> resolve(ProductCatalog catalog) {
        // Expand bundle into its components, multiplying qty
        var resolved = new ArrayList<ResolvedItem>();
        for (CartLineItem child : children) {
            for (ResolvedItem r : child.resolve(catalog)) {
                resolved.add(new ResolvedItem(r.productId(), r.qty() * qty, r.unitPrice()));
            }
        }
        return resolved;
    }
}
```

This is **Composite**: `BundleLineItem` contains other `CartLineItem`s, which themselves can be products or nested bundles. The total price recurses; the resolution flattens for inventory/shipping.

### Cart

```java
public final class Cart {
    private final String id;
    private final String userId;
    private final List<CartLineItem> lineItems = new ArrayList<>();
    private final ProductCatalog catalog;
    private final Currency currency;

    public Cart(String id, String userId, ProductCatalog catalog, Currency currency) {
        this.id = id; this.userId = userId;
        this.catalog = catalog; this.currency = currency;
    }

    public synchronized void add(CartLineItem item)   { lineItems.add(item); }
    public synchronized void remove(String productId) { lineItems.removeIf(i -> i.productId().equals(productId)); }
    public synchronized List<CartLineItem> items()    { return List.copyOf(lineItems); }

    public PriceBreakdown priceQuote(List<PricingRule> rules, Address shippingAddress) {
        Money subtotal = items().stream()
            .map(i -> i.totalPrice(catalog))
            .reduce(Money.zero(currency), Money::plus);
        PriceBreakdown breakdown = new PriceBreakdown(subtotal, Money.zero(currency),
                                                      Money.zero(currency), Money.zero(currency), subtotal);
        for (PricingRule rule : rules) breakdown = rule.apply(this, breakdown);
        return breakdown;
    }

    public List<CartLineItem.ResolvedItem> resolved() {
        var all = new ArrayList<CartLineItem.ResolvedItem>();
        for (CartLineItem i : items()) all.addAll(i.resolve(catalog));
        return all;
    }
}
```

### Pricing pipeline (Chain of Responsibility)

```java
public final class DiscountRule implements PricingRule {
    private final BigDecimal discountPct;
    public DiscountRule(BigDecimal pct) { this.discountPct = pct; }

    @Override
    public PriceBreakdown apply(Cart cart, PriceBreakdown b) {
        Money discount = new Money(b.subtotal().amount().multiply(discountPct), b.subtotal().currency());
        Money newTotal = b.subtotal().minus(discount).plus(b.tax()).plus(b.shipping());
        return new PriceBreakdown(b.subtotal(), b.discount().plus(discount), b.tax(), b.shipping(), newTotal);
    }
}

public final class TaxRule implements PricingRule {
    private final BigDecimal taxRate;
    public TaxRule(BigDecimal rate) { this.taxRate = rate; }

    @Override
    public PriceBreakdown apply(Cart cart, PriceBreakdown b) {
        Money taxable = b.subtotal().minus(b.discount());
        Money tax = new Money(taxable.amount().multiply(taxRate), taxable.currency());
        Money newTotal = taxable.plus(tax).plus(b.shipping());
        return new PriceBreakdown(b.subtotal(), b.discount(), tax, b.shipping(), newTotal);
    }
}

public final class ShippingRule implements PricingRule {
    private final Money baseShippingFee;
    public ShippingRule(Money fee) { this.baseShippingFee = fee; }

    @Override
    public PriceBreakdown apply(Cart cart, PriceBreakdown b) {
        Money newTotal = b.total().plus(baseShippingFee);
        return new PriceBreakdown(b.subtotal(), b.discount(), b.tax(), b.shipping().plus(baseShippingFee), newTotal);
    }
}
```

The pipeline is composed at wire-up time:
```java
List<PricingRule> rules = List.of(
    new DiscountRule(new BigDecimal("0.10")),  // 10% off
    new TaxRule(new BigDecimal("0.08")),       // 8% tax
    new ShippingRule(new Money(new BigDecimal("4.99"), USD))
);
```
Order matters: discount → tax → shipping (tax applies to the discounted subtotal; shipping is post-tax).

### Order

```java
public final class Order {
    private final String id;
    private final String userId;
    private final List<CartLineItem.ResolvedItem> items;
    private final Address shippingAddress;
    private final String paymentMethodId;
    private final PriceBreakdown priceBreakdown;
    private final Instant createdAt;
    private OrderStatus status;
    private long version;

    private final List<String> reservationIds = new ArrayList<>();
    private String paymentRef;
    private String shipmentTrackingId;

    public Order(String id, String userId, List<CartLineItem.ResolvedItem> items,
                 Address address, String paymentMethodId, PriceBreakdown breakdown,
                 Instant createdAt) {
        this.id = id; this.userId = userId; this.items = List.copyOf(items);
        this.shippingAddress = address; this.paymentMethodId = paymentMethodId;
        this.priceBreakdown = breakdown; this.createdAt = createdAt;
        this.status = OrderStatus.PLACED;
    }

    // State transitions
    public synchronized void confirm()      { transition(OrderStatus.PLACED, OrderStatus.CONFIRMED); }
    public synchronized void ship()         { transition(OrderStatus.CONFIRMED, OrderStatus.SHIPPED); }
    public synchronized void deliver()      { transition(OrderStatus.SHIPPED, OrderStatus.DELIVERED); }
    public synchronized void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED || status == OrderStatus.REFUNDED)
            throw new IllegalStateException("Cannot cancel: " + status);
        status = OrderStatus.CANCELLED;
        version++;
    }

    private void transition(OrderStatus from, OrderStatus to) {
        if (status != from) throw new IllegalStateException("Cannot " + to + " from " + status);
        status = to;
        version++;
    }

    public String id() { return id; }
    public String userId() { return userId; }
    public List<CartLineItem.ResolvedItem> items() { return items; }
    public OrderStatus status() { return status; }
    public PriceBreakdown priceBreakdown() { return priceBreakdown; }
    public Money total() { return priceBreakdown.total(); }
    public List<String> reservationIds() { return reservationIds; }
    public void addReservationId(String id) { reservationIds.add(id); }
    public void setPaymentRef(String ref) { this.paymentRef = ref; }
    public void setShipmentTrackingId(String id) { this.shipmentTrackingId = id; }
    public String paymentRef() { return paymentRef; }
}
```

### `OrderService` — Saga orchestrator

```java
public final class OrderService {
    private final InventoryService inventory;
    private final PaymentGateway payment;
    private final ShippingCarrier shipping;
    private final OrderRepository orderRepo;
    private final EventBus eventBus;
    private final Clock clock;
    private final IdGenerator ids;

    public OrderService(InventoryService inv, PaymentGateway pay, ShippingCarrier ship,
                        OrderRepository or, EventBus bus, Clock clock, IdGenerator ids) {
        this.inventory = inv; this.payment = pay; this.shipping = ship;
        this.orderRepo = or; this.eventBus = bus; this.clock = clock; this.ids = ids;
    }

    public Order checkout(Cart cart, Address address, String paymentMethodId, List<PricingRule> rules) {
        PriceBreakdown breakdown = cart.priceQuote(rules, address);
        var resolved = cart.resolved();

        Order order = new Order(ids.next(), cart.userId(), resolved, address, paymentMethodId,
                                breakdown, clock.instant());
        orderRepo.save(order);

        // Saga: reserve inventory → authorize payment → confirm order → schedule shipment
        try {
            // Step 1: reserve inventory
            for (var item : resolved) {
                Reservation r = inventory.reserve(item.productId(), item.qty(),
                                                  "order:" + order.id() + ":" + item.productId());
                order.addReservationId(r.id());
            }

            // Step 2: authorize payment
            PaymentResult auth = payment.authorize(cart.userId(), order.total(), "auth:" + order.id());
            if (!auth.success()) throw new PaymentFailedException();
            order.setPaymentRef(auth.ref());

            // Step 3: confirm order (state transition)
            order.confirm();
            orderRepo.save(order);

            // Step 4: capture payment + commit reservations + schedule shipment
            payment.capture(auth.ref(), order.total(), "capture:" + order.id());
            for (String resId : order.reservationIds()) inventory.commit(resId);
            ShipmentTracking tracking = shipping.schedule(order, address);
            order.setShipmentTrackingId(tracking.trackingNumber());
            orderRepo.save(order);

            eventBus.publish(new OrderPlacedEvent(order.id()));
            return order;

        } catch (Exception e) {
            // Compensating actions
            for (String resId : order.reservationIds()) {
                try { inventory.release(resId); } catch (Exception ignored) {}
            }
            if (order.paymentRef() != null) {
                try { payment.refund(order.paymentRef(), order.total(), "refund:" + order.id()); }
                catch (Exception ignored) {}
            }
            order.cancel();
            orderRepo.save(order);
            throw new CheckoutFailedException(e);
        }
    }

    public void cancel(String orderId, String userId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        if (!order.userId().equals(userId)) throw new ForbiddenException();
        if (order.status() == OrderStatus.SHIPPED || order.status() == OrderStatus.DELIVERED)
            throw new CancelTooLateException();

        // Refund
        payment.refund(order.paymentRef(), order.total(), "refund:" + orderId);

        // Release inventory (if not already committed) or restock
        for (String resId : order.reservationIds()) inventory.release(resId);

        order.cancel();
        orderRepo.save(order);
        eventBus.publish(new OrderCancelledEvent(orderId));
    }
}
```

### `InventoryService` — atomic reservation

```java
public final class InventoryServiceImpl implements InventoryService {
    private final InventoryRepository repo;

    public InventoryServiceImpl(InventoryRepository repo) { this.repo = repo; }

    @Override
    public Reservation reserve(String productId, int qty, String idempotencyKey) {
        // Idempotency check: if reservation already exists for this key, return it.
        Optional<Reservation> existing = repo.findReservationByKey(idempotencyKey);
        if (existing.isPresent()) return existing.get();

        // Atomic decrement-if-available across warehouses.
        // In SQL this is one transaction with row-locked SELECT FOR UPDATE.
        for (InventoryItem item : repo.findItemsForProduct(productId)) {
            if (item.availableQty() >= qty) {
                int newAvailable = item.availableQty() - qty;
                if (repo.updateConditional(item.id(), newAvailable, item.version())) {
                    Reservation r = new Reservation(idempotencyKey, productId,
                                                     item.warehouseId(), qty, idempotencyKey);
                    repo.saveReservation(r);
                    return r;
                }
                // Lost the version race; try next warehouse or retry
            }
        }
        throw new InsufficientStockException(productId, qty);
    }

    @Override
    public void release(String reservationId) {
        Reservation r = repo.findReservationById(reservationId).orElse(null);
        if (r == null || r.released()) return;   // idempotent
        repo.markReleased(r.id());
        InventoryItem item = repo.findItem(r.productId(), r.warehouseId()).orElseThrow();
        repo.updateConditional(item.id(), item.availableQty() + r.qty(), item.version());
    }

    @Override
    public void commit(String reservationId) {
        Reservation r = repo.findReservationById(reservationId).orElseThrow();
        if (r.committed()) return;
        repo.markCommitted(r.id());
    }
}
```

### Putting it together

```java
Cart cart = new Cart("cart-1", "alice", catalog, USD);
cart.add(new ProductLineItem("p1", 2));
cart.add(new BundleLineItem("bundle-coffee", 1, List.of(
    new ProductLineItem("p2", 1),
    new ProductLineItem("p3", 1)
), new BigDecimal("0.15")));

List<PricingRule> rules = List.of(
    new DiscountRule(new BigDecimal("0.10")),
    new TaxRule(new BigDecimal("0.08")),
    new ShippingRule(new Money(new BigDecimal("4.99"), USD))
);

PriceBreakdown quote = cart.priceQuote(rules, address);
Order order = orderService.checkout(cart, address, "pm-1", rules);
```

---

## Concurrency strategy

### What's contested
- **Inventory** — every order reduces stock; double-spending = overselling.
- **Payment** — retry without double-charging.
- **Order state** — admin and user actions can race.

### Strategy
- **Inventory**: optimistic locking with `version` per `InventoryItem`. `updateConditional` succeeds only if version matches. Retry on miss.
- **Payment idempotency**: every `authorize`, `capture`, `refund` carries an `idempotencyKey`. The gateway de-duplicates across retries.
- **Order state**: per-order `synchronized` (or version-locked DB updates) so transitions are serial.

### Saga compensations
- Each forward step has an inverse:
  - `reserve` ↔ `release`
  - `authorize` ↔ (auto-expires) or `void`
  - `capture` ↔ `refund`
  - `schedule shipment` ↔ `cancel shipment`
- On any failure, run compensations in reverse order. Idempotent compensations are safe to retry.

### What if `inventory.reserve` succeeds but the network drops before `payment.authorize`?
- The retry will see the existing reservation (idempotency key match) — no double reservation.
- The flow continues; order moves forward.

### What if everything succeeds but the response to the user gets lost?
- The user retries `checkout` with the same idempotency key (cart ID + state hash).
- The order service detects the existing order and returns it.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Insufficient stock for one item in a multi-item order** | Reserve fails → cascade compensation → cancel. |
| **Payment authorized but capture fails** | Refund the auth (or let it expire). Reservation released. |
| **User cancels mid-flow** | Compensations run; order cancelled. |
| **Bundle priced differently than sum of components** | Bundle's `totalPrice` applies its own discount. |
| **Same product appears multiple times in cart** | Resolve flattens; inventory reserves the total qty. |
| **Tax on discounted vs full price** | Specify in pricing rule order — usually tax on (subtotal - discount). |
| **Shipping address invalid** | Pre-validate; refuse before saga. |
| **Cancel after shipping** | Refuse; the order is committed to fulfillment. |
| **Two admin users mark an order shipped concurrently** | Order's `synchronized` (or version check) ensures one wins. |
| **Cart item price changes during checkout** | Lock the price at `checkout` time (price quote becomes part of the order). |
| **Out-of-stock during reservation** | Surface clearly; don't silently drop items from the order. |
| **Multi-warehouse: cheapest, fastest, or default?** | A `WarehouseSelectionStrategy` picks; out of LLD scope but the seam is there. |

---

## Extensions / interview follow-ups

### "Add returns / RMA workflow."
- New `Return` aggregate; references `Order`.
- State machine: REQUESTED → APPROVED → IN_TRANSIT → RECEIVED → REFUNDED.
- Restocks inventory on receive.

### "Add a wishlist."
- Separate aggregate from cart. No checkout flow.
- Move-to-cart operation transfers items.

### "Add coupons / promotions."
- New `PricingRule` implementations: `CouponRule`, `BuyNGetMRule`.
- Cart accepts coupon codes; rules validate and apply.

### "Multi-currency."
- `Money` already carries currency.
- Cart converts at checkout time using a snapshot FX rate.
- Inventory and product catalog stay in seller's currency; conversion happens at the price-quote boundary.

### "Why a Saga and not a 2-phase commit?"
- 2PC is hard across HTTP services and databases (distributed transaction managers).
- Sagas are local transactions per service + compensating actions on failure.
- Eventual consistency is the trade-off, but with idempotent operations + retries, it works for retail.

### "What if the payment service is down?"
- The saga fails at authorize; reservations are released; order is cancelled.
- A retry queue can handle transient failures: enqueue the checkout intent; retry until payment is up.

### "How do you prevent overselling?"
- Optimistic locking on `InventoryItem` ensures only one reservation can succeed per stock decrement.
- Reservations are durable (DB row), not in-memory only.
- Idempotency keys ensure a retried reservation doesn't double-decrement.

### "How do you scale this?"
- Each service (Inventory, Payment, Order, Shipping) deployed independently.
- Communication via API (HTTP) for the synchronous saga; Kafka for downstream events (notifications, analytics).
- Each service has its own DB.

### "Should this be microservices or monolith?"
- Start as a monolith with logical service boundaries (the interfaces above).
- Split when team boundaries warrant — Inventory team owns `InventoryService`, etc.
- The saga structure already gives you the seams.

---

## Scaling

The system above is designed for moderate scale (thousands of orders/day). For Amazon-scale:

### Service decomposition
- Each "service" becomes a real microservice with its own DB.
- Communication via gRPC or REST + Kafka.
- The saga becomes literal — each step is a remote call.

### Inventory at scale
- Per-product, per-warehouse rows are hot.
- Sharding by product; warehouses by region.
- Pre-allocate stock to "fast lanes" (Redis CAS) for popular items; fall through to SQL for cold items.

### Order ingestion spikes (Black Friday)
- Gateway with rate limiting and queues.
- Async order acceptance: receive order, queue, process. User sees "order placed, processing."
- Idempotency from the API gateway through to the saga.

### Read scaling
- Read replicas of order DB; cache hot user views.
- Search via Elasticsearch (separate from OLTP).

### Eventual consistency
- Order placed → inventory reserved → cache invalidation event → product page reflects new stock.
- Most queries (product detail, recommendations) tolerate seconds of staleness.

---

## Persistence tradeoffs

E-commerce is the domain where persistence patterns are most diverse — different data has different characteristics, and different stores fit each.

### What persistent data exists
- **Catalog**: products, prices, descriptions. Read-mostly.
- **Inventory**: per-product, per-warehouse. Hot writes during checkout.
- **Cart**: per-user. Frequent writes, small.
- **Order**: append-mostly with state transitions.
- **Payment**: PCI-sensitive; minimal storage on app side.
- **Shipping**: tracking numbers, status updates.

### Option 1 — All in PostgreSQL
- Single relational DB; one schema per logical service or one shared.
- ACID across operations within a single service.
- **Pros**: simple. Consistency by default.
- **Cons**: scaling write throughput becomes hard at large scale; cross-table joins become bottlenecks.
- **When to pick**: starting small, single team, < 10k orders/day.

### Option 2 — Polyglot persistence (per-service DB choice)
- **Catalog**: PostgreSQL (read-heavy, occasional writes) + Elasticsearch (search).
- **Inventory**: PostgreSQL with strong row-level optimistic locking. Or DynamoDB with conditional writes for cloud-native.
- **Cart**: Redis (TTL on abandoned carts) or DynamoDB (scale, durable).
- **Order**: PostgreSQL or document store (orders are mostly read by ID, occasionally listed).
- **Payment**: tokenized; the actual card data is at the payment gateway. Local DB stores payment refs only.
- **Shipping**: PostgreSQL; separate table for tracking events.
- **When to pick**: real-world recommendation; each service can pick what fits.

### Option 3 — Event sourcing for orders
- Every order state transition is an event: `OrderPlaced`, `PaymentCaptured`, `OrderShipped`, etc.
- Order state is the fold over events.
- **Pros**: complete audit trail; replay debugs; easy integration (other services subscribe to event stream).
- **Cons**: complexity; query latency without snapshots.
- **When to pick**: when orders need detailed audit (returns, fraud investigation, compliance).

### Option 4 — CQRS (Command Query Responsibility Segregation)
- Writes go to a write model (event-sourced or normalized).
- Reads go to denormalized read models (per query type).
- Async sync via events.
- **When to pick**: very different read and write patterns; high read throughput.

### Cart-specific considerations
- Cart can be ephemeral (Redis with 7-day TTL) or persistent (DB).
- Logged-in user: persistent cart across devices.
- Anonymous: cookie + Redis with short TTL.
- Production sites usually do both: Redis hot, sync to DB periodically.

### Inventory consistency under sharding
- Per-warehouse partitioning natural (each warehouse is a shard).
- Reservation crosses one shard; a single-warehouse fulfillment is one transaction.
- Multi-warehouse fulfillment (split shipments) becomes a saga across shards.

### Idempotency store
- Every cross-service call carries an idempotency key.
- Server-side de-dup table: `(idempotency_key, response)`.
- TTL-bounded (24h is typical).

### Concurrency under persistence
| Concern | In-memory | SQL | Redis | DynamoDB |
|---|---|---|---|---|
| Inventory reservation | `synchronized` + version | `UPDATE ... WHERE version = ?` | Lua script atomic | `UpdateItem` with condition |
| Order state transition | `synchronized` | `UPDATE ... WHERE version = ?` | `WATCH/MULTI/EXEC` | conditional write |
| Cart updates | `synchronized` | last-write-wins or version | last-write-wins | last-write-wins |

### Recommendation
- **For a real product**: polyglot. Each service picks what fits.
  - Inventory: PostgreSQL with row versioning.
  - Cart: Redis + periodic sync to PostgreSQL.
  - Order: PostgreSQL; consider event sourcing for the lifecycle.
  - Catalog: PostgreSQL + Elasticsearch.
- **For an interview**: PostgreSQL across the board with the right schemas; mention "we'd specialize per service if scaling demanded it."

---

## Talking points for the interview

- "Checkout is a Saga: reserve inventory → authorize payment → confirm order → capture payment → schedule shipment. Each step has a compensating action. No 2PC required."
- "Idempotency keys on every cross-service call so retries are safe. The keys are deterministic from the order ID."
- "Pricing is a pipeline of `PricingRule`s; order matters — discount before tax, shipping at the end. Each rule is independently testable."
- "Cart items are a Composite — bundles contain other line items. Pricing recurses; inventory resolution flattens."
- "Inventory uses optimistic version locking — concurrent checkouts can't oversell because only one wins the version check."
- "Order has a state machine; each transition is a synchronized method (or DB version update). The state machine prevents 'cancel after shipped' or 'ship before paid.'"
- "I'd start as a logical-service monolith with the seams in place; split when team boundaries warrant it. The saga structure already gives you the boundaries."

---

## Summary

This is the densest LLD problem because it touches every pattern you've learned. The skeleton:

- **Cart**: Composite for bundles, mutable until checkout.
- **Pricing**: Chain of Responsibility / Strategy.
- **Order**: State pattern lifecycle.
- **Checkout**: Saga across Inventory + Payment + Shipping with compensations.
- **Inventory**: Optimistic locking; idempotent reservations.
- **Payment**: Idempotent operations via keys.

The senior signal is the **failure-mode reasoning**: at every step in the saga, what happens on failure? Can the operation be retried safely? Can compensations be retried safely? The answer should be "yes" for both — that's what idempotency keys + state machines buy you.
