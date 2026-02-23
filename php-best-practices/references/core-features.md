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
        $this->name = $name;
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
    $age < 18 => 'minor',
    $age < 65 => 'adult',
    default   => 'senior',
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

---

## PHP 7.x Foundation Features

These are still everyday syntax — know them cold.

### Null Coalescing Operator `??` (7.0)

Returns the left side if it exists and is not `null`, otherwise the right side.
No notice on undefined array keys or variables.

```php
// Old
$name = isset($data['name']) ? $data['name'] : 'Anonymous';

// Modern
$name = $data['name'] ?? 'Anonymous';

// Chains
$city = $user['address']['city'] ?? $defaults['city'] ?? 'Unknown';

// Works on method calls too
$locale = $user->getPreferences()?->locale ?? 'en';
```

### Null Coalescing Assignment `??=` (7.4)

Assign only if the left side is null or undefined.

```php
// Old
$config['timeout'] = $config['timeout'] ?? 30;

// Modern
$config['timeout'] ??= 30;

// Common use: lazy-initialise cache
$this->cache[$key] ??= $this->computeExpensiveValue($key);
```

### Spaceship Operator `<=>` (7.0)

Returns -1, 0, or 1. Purpose-built for sorting callbacks.

```php
// Old
usort($users, function($a, $b) {
    if ($a->age === $b->age) return 0;
    return $a->age < $b->age ? -1 : 1;
});

// Modern
usort($users, fn($a, $b) => $a->age <=> $b->age);

// Multi-field sort
usort($products, fn($a, $b) =>
    [$a->category, $a->price] <=> [$b->category, $b->price]
);
```

### Scalar Type Declarations (7.0)

```php
function add(int $a, int $b): int {
    return $a + $b;
}

function slugify(string $text): string {
    return strtolower(str_replace(' ', '-', trim($text)));
}
```

Always pair with `declare(strict_types=1)` — without it, PHP silently coerces
`"3"` to `3` and you lose the benefit.

### Nullable Types `?Type` (7.1)

```php
function findUser(int $id): ?User {
    return $this->repository->find($id); // User or null
}

function setDeadline(?DateTimeImmutable $deadline): void {
    $this->deadline = $deadline;
}
```

Equivalent to `Type|null` — both forms are valid in 8.x.

### Void Return Type (7.1)

```php
public function save(Order $order): void {
    $this->em->persist($order);
    $this->em->flush();
    // no return value — returning null is still allowed
}
```

### Typed Properties (7.4)

Declare a type on class properties directly — no docblock needed.

```php
class User {
    public int $id;
    public string $name;
    public ?string $bio = null;
    private DateTimeImmutable $createdAt;
    protected array $roles = [];
}
```

Accessing an uninitialized typed property throws `Error`. Always initialise in
the constructor or with a default value.

### Arrow Functions `fn` (7.4)

Single-expression closures that auto-capture outer scope by value.

```php
// Old closure — must explicitly use()
$multiplier = 3;
$triple = array_map(function($n) use ($multiplier) {
    return $n * $multiplier;
}, $numbers);

// Arrow function — captures $multiplier automatically
$triple = array_map(fn($n) => $n * $multiplier, $numbers);

// Common patterns
$activeUsers  = array_filter($users, fn($u) => $u->isActive());
$names        = array_map(fn($u) => $u->name, $users);
$total        = array_reduce($items, fn($carry, $i) => $carry + $i->price, 0);
```

Arrow functions are read-only captures — they cannot mutate the outer variable.
Use a regular closure with `use (&$var)` when you need mutation.

### Spread Operator in Arrays (7.4) / String Keys (8.1)

```php
$defaults = ['timeout' => 30, 'retries' => 3];
$override = ['timeout' => 60];

// Merge — later keys win
$config = [...$defaults, ...$override];
// ['timeout' => 60, 'retries' => 3]

// Build arrays without array_merge
$base  = [1, 2, 3];
$extra = [4, 5];
$all   = [...$base, ...$extra, 6]; // [1, 2, 3, 4, 5, 6]

// String key spread (8.1+) replaces array_merge for named keys
$merged = [...$defaults, ...$override]; // works with string keys in 8.1+
```

### Anonymous Classes (7.0)

Useful for one-off interface implementations, especially in tests.

```php
// Quick fake without a separate file
$logger = new class implements LoggerInterface {
    public array $logs = [];
    public function log(mixed $level, string $message, array $context = []): void {
        $this->logs[] = [$level, $message];
    }
};

$service = new MyService($logger);
$service->doSomething();
assert(count($logger->logs) === 1);
```

### Named Arguments + Spread Together

```php
function createUser(string $name, string $email, int $age = 0): User { ... }

$data = ['email' => 'a@b.com', 'name' => 'Alice'];
$user = createUser(...$data); // spread associative array as named args
```

### `list()` / Short Destructuring (7.1)

```php
// Positional
[$first, $second, $third] = $array;
[, , $third] = $array; // skip elements

// Key-based destructuring
['name' => $name, 'email' => $email] = $userData;

// In foreach
foreach ($points as [$x, $y]) {
    echo "$x, $y";
}

foreach ($rows as ['id' => $id, 'label' => $label]) {
    echo "$id: $label";
}
```

### Class Constant Visibility (7.1)

```php
class HttpStatus {
    public const int OK = 200;
    public const int CREATED = 201;
    protected const int SERVER_ERROR = 500;
    private const string VERSION = '1.1';
}
```

### `array_key_first()` / `array_key_last()` (7.3)

```php
$map = ['a' => 1, 'b' => 2, 'c' => 3];

array_key_first($map); // 'a'
array_key_last($map);  // 'c'

// Safe — does not modify internal array pointer (unlike reset()/end())
```
