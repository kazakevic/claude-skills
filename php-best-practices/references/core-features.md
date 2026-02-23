# Core PHP 8.x Language Features

## Strict Types + Declare

```php
<?php
declare(strict_types=1);
```

Every file. Without it, PHP silently coerces types.

---

## Constructor Property Promotion (8.0)

Declare, document, and assign in one shot.

```php
// ❌ Before
class User {
    private string $name;
    private string $email;
    public function __construct(string $name, string $email) {
        $this->name  = $name;
        $this->email = $email;
    }
}

// ✅ After
class User {
    public function __construct(
        private string $name,
        private readonly string $email, // combine with readonly
    ) {}
}
```

Mix promoted and non-promoted freely. Always use trailing comma.

---

## Named Arguments (8.0)

```php
// Skip optional args, self-document call sites
$result = array_slice(array: $items, offset: 0, length: 5, preserve_keys: true);

$date = new \DateTime(datetime: '2026-01-01', timezone: new \DateTimeZone('UTC'));
```

---

## Match Expressions (8.0)

Strict (`===`), exhaustive, returns a value, throws `UnhandledMatchError` on no match.

```php
$label = match($status) {
    Status::Active   => 'Active',
    Status::Inactive => 'Inactive',
    Status::Banned   => 'Suspended',
    // No default needed if enum is exhaustive
};

// Multiple arms
$group = match(true) {
    $age < 18  => 'minor',
    $age < 65  => 'adult',
    default    => 'senior',
};
```

Never use `switch` for new code.

---

## Nullsafe Operator (8.0)

```php
// Old
$city = $user ? ($user->getAddress() ? $user->getAddress()->getCity() : null) : null;

// Modern
$city = $user?->getAddress()?->getCity();
```

---

## Union Types (8.0) & Intersection Types (8.1)

```php
function parseId(string|int $id): int {
    return is_string($id) ? (int) $id : $id;
}

// Intersection: must implement ALL interfaces
function process(Countable&Traversable $col): void { ... }

// DNF types (8.2): union of intersections
function handle((Countable&Traversable)|null $col): int {
    return $col?->count() ?? 0;
}
```

---

## Enums (8.1)

Replace all string/int constants. Backed enums for persistence.

```php
// Pure enum — no value, type-safe
enum Direction { case North; case South; case East; case West; }

// Backed enum — for DB/serialization
enum Currency: string {
    case EUR = 'EUR';
    case USD = 'USD';
    case GBP = 'GBP';

    public function symbol(): string {
        return match($this) {
            Currency::EUR => '€',
            Currency::USD => '$',
            Currency::GBP => '£',
        };
    }
}

// From serialized value
$c = Currency::from('EUR');        // throws ValueError if invalid
$c = Currency::tryFrom('INVALID'); // returns null
```

Enums can implement interfaces and use traits. They cannot have mutable properties.

---

## Readonly Properties & Classes (8.1 / 8.2)

```php
// PHP 8.1 — single property
class Order {
    public readonly int $id;
    public function __construct(int $id) { $this->id = $id; }
}

// PHP 8.2 — readonly class (all properties are readonly)
readonly class Money {
    public function __construct(
        public int $amount,
        public Currency $currency,
    ) {
        if ($amount < 0) throw new \InvalidArgumentException('Negative amount');
    }

    public function add(Money $other): self {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Currency mismatch');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }
}
```

---

## First-Class Callables (8.1)

```php
$upper = strtoupper(...);       // Closure from builtin
$fn    = $this->format(...);    // Closure from method

$result = array_map(strtoupper(...), ['hello', 'world']);
usort($items, $this->compare(...));
```

---

## Fibers (8.1)

Low-level coroutines. Typically used via Revolt/Amphp, not directly.

```php
$fiber = new \Fiber(function(): string {
    $value = \Fiber::suspend('first');
    return "got: $value";
});

$first  = $fiber->start();          // 'first'
$result = $fiber->resume('hello');  // terminates
echo $fiber->getReturn();           // 'got: hello'
```

---

## Type System — Standalone Types (8.2)

```php
function alwaysFails(): never { throw new \Exception('Always throws'); }
function isEnabled(): true|false { return true; }
function maybeNull(): null { return null; }

// #[SensitiveParameter] — redacts from stack traces (8.2)
function login(string $user, #[\SensitiveParameter] string $password): void { ... }
```

---

## Typed Class Constants (8.3)

```php
interface HasVersion {
    const string VERSION = '1.0';
}

class Config {
    const int    MAX_RETRIES = 3;
    const string DEFAULT_LOCALE = 'en';
}
```

---

## PSR Clock for Testable Time

```php
use Psr\Clock\ClockInterface;

class SubscriptionService {
    public function __construct(private ClockInterface $clock) {}

    public function isExpired(Subscription $sub): bool {
        return $sub->expiresAt < $this->clock->now();
    }
}

// In tests: inject a frozen FakeClock
class FakeClock implements ClockInterface {
    public function __construct(private \DateTimeImmutable $time) {}
    public function now(): \DateTimeImmutable { return $this->time; }
}
```

Never call `new \DateTime()` or `time()` directly in business logic.
