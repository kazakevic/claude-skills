# Entities, Value Objects & PHP Attributes

## PHP Attributes (8.0)

Define custom metadata as first-class language constructs. Used for ORM mapping,
validation, routing, serialization — replaces docblock annotations.

### Defining Attributes

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
final class Table {
    public function __construct(public readonly string $name) {}
}

#[\Attribute(\Attribute::TARGET_PROPERTY)]
final class Column {
    public function __construct(
        public readonly string  $name,
        public readonly string  $type     = 'VARCHAR(255)',
        public readonly bool    $nullable = false,
    ) {}
}

#[\Attribute(\Attribute::TARGET_PROPERTY)]
final class Id {}

// TARGET constants: TARGET_CLASS, TARGET_FUNCTION, TARGET_METHOD,
//                  TARGET_PROPERTY, TARGET_CLASS_CONSTANT, TARGET_PARAMETER
// Combine: TARGET_CLASS | TARGET_METHOD
```

### Reading Attributes via Reflection

```php
function getTableName(string $class): string {
    $ref   = new \ReflectionClass($class);
    $attrs = $ref->getAttributes(Table::class);
    if (empty($attrs)) throw new \RuntimeException("No #[Table] on $class");
    return $attrs[0]->newInstance()->name;
}

function getColumnMap(string $class): array {
    $ref     = new \ReflectionClass($class);
    $columns = [];
    foreach ($ref->getProperties() as $prop) {
        $attrs = $prop->getAttributes(Column::class);
        if ($attrs) $columns[$prop->getName()] = $attrs[0]->newInstance();
    }
    return $columns;
}
```

---

## Entity Pattern

Entities have identity (usually a DB ID) and mutable state over time.
Use asymmetric visibility + property hooks + promoted constructor.

```php
<?php
declare(strict_types=1);

#[Table(name: 'orders')]
class Order
{
    /** @var list<OrderItem> */
    private array $items = [];

    public private(set) OrderStatus $status = OrderStatus::Draft;
    public private(set) \DateTimeImmutable $updatedAt;

    public function __construct(
        public private(set) int $id,
        public private(set) int $customerId,
        public private(set) \DateTimeImmutable $createdAt,
    ) {
        $this->updatedAt = $createdAt;
    }

    public function addItem(Product $product, int $quantity): void
    {
        if ($this->status !== OrderStatus::Draft) {
            throw new \DomainException('Cannot modify a confirmed order.');
        }
        $this->items[]   = new OrderItem($product, $quantity);
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function confirm(): void
    {
        if (empty($this->items)) {
            throw new \DomainException('Cannot confirm an empty order.');
        }
        $this->status    = OrderStatus::Confirmed;
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function total(): Money
    {
        return array_reduce(
            $this->items,
            fn(Money $carry, OrderItem $item) => $carry->add($item->subtotal()),
            new Money(0, Currency::EUR),
        );
    }

    /** @return list<OrderItem> */
    public function items(): array { return $this->items; }
}

enum OrderStatus: string {
    case Draft     = 'draft';
    case Confirmed = 'confirmed';
    case Shipped   = 'shipped';
    case Cancelled = 'cancelled';
}
```

---

## Value Object Pattern

No identity. Fully immutable. Equality by value, not reference.
Use `readonly class` (PHP 8.2+).

```php
<?php
declare(strict_types=1);

readonly class Money
{
    public function __construct(
        public int      $amount,   // in cents/minor unit
        public Currency $currency,
    ) {
        if ($amount < 0) throw new \InvalidArgumentException('Negative amount.');
    }

    public function add(Money $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Cannot add different currencies.');
        }
        // PHP 8.5: clone($this, ['amount' => ...]) instead of new self(...)
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(int $factor): self
    {
        return new self($this->amount * $factor, $this->currency);
    }

    public function isGreaterThan(Money $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount > $other->amount;
    }

    public function format(): string
    {
        return sprintf('%s %s',
            number_format($this->amount / 100, $this->currency->decimalPlaces()),
            $this->currency->symbol(),
        );
    }

    private function assertSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Currency mismatch.');
        }
    }
}

readonly class EmailAddress
{
    public string $value;
    public string $localPart;
    public string $domain;

    public function __construct(string $email)
    {
        $normalized = strtolower(trim($email));
        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: $email");
        }
        [$this->localPart, $this->domain] = explode('@', $normalized, 2);
        $this->value = $normalized;
    }

    public function equals(self $other): bool { return $this->value === $other->value; }
    public function __toString(): string      { return $this->value; }
}
```

---

## Domain Event Pattern

```php
<?php
declare(strict_types=1);

interface DomainEvent {
    public \DateTimeImmutable $occurredAt { get; }
}

readonly class OrderConfirmed implements DomainEvent {
    public \DateTimeImmutable $occurredAt { get; }

    public function __construct(
        public int   $orderId,
        public int   $customerId,
        public Money $total,
    ) {
        $this->occurredAt = new \DateTimeImmutable();
    }
}
```

---

## Repository Interface (Domain Contract)

The domain defines the interface; infrastructure implements it.

```php
<?php
declare(strict_types=1);

/**
 * @template T of object
 */
interface Repository {
    /** @return T|null */
    public function find(int $id): ?object;

    /** @return list<T> */
    public function findAll(): array;

    /** @param T $entity */
    public function save(object $entity): void;

    public function delete(int $id): void;
}

/**
 * @extends Repository<Order>
 */
interface OrderRepository extends Repository {
    /** @return list<Order> */
    public function findByCustomer(int $customerId): array;

    public function findByStatus(OrderStatus $status): array;
}
```

---

## Annotated Entity for ORM (Attributes + Hooks)

```php
<?php
declare(strict_types=1);

#[Table(name: 'products')]
class Product
{
    #[Id]
    #[Column(name: 'id', type: 'INT UNSIGNED')]
    public int $id {
        get;
        private(set);
    }

    #[Column(name: 'name')]
    public string $name {
        get;
        set {
            if (strlen($value) < 1) throw new \InvalidArgumentException('Name empty.');
            $this->name = trim($value);
        }
    }

    #[Column(name: 'price_cents', type: 'INT UNSIGNED')]
    public int $priceCents {
        get;
        set {
            if ($value < 0) throw new \DomainException('Price cannot be negative.');
            $this->priceCents = $value;
        }
    }

    #[Column(name: 'currency', type: 'CHAR(3)')]
    public Currency $currency { get; set; }

    #[Column(name: 'created_at', type: 'DATETIME', nullable: false)]
    public \DateTimeImmutable $createdAt { get; }

    public function __construct(string $name, int $priceCents, Currency $currency)
    {
        $this->name       = $name;
        $this->priceCents = $priceCents;
        $this->currency   = $currency;
        $this->createdAt  = new \DateTimeImmutable();
    }

    public function price(): Money
    {
        return new Money($this->priceCents, $this->currency);
    }
}
```
