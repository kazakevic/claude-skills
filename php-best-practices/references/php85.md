# PHP 8.5 Features

Released November 20, 2025. Upgrade from 8.4 is straightforward — no major
breaking changes for well-maintained codebases.

---

## Pipe Operator `|>`

Chains callables left-to-right. The output of the left side is passed as
the single argument to the callable on the right. Compiled away at compile
time — zero runtime overhead.

```php
// Before — nested or step-by-step
$slug = strtolower(
    str_replace(' ', '-', trim($title))
);

// Before — intermediate variables
$result = trim($title);
$result = str_replace(' ', '-', $result);
$result = strtolower($result);

// PHP 8.5 — pipe operator
$slug = $title
    |> trim(...)
    |> fn(string $s) => str_replace(' ', '-', $s)
    |> strtolower(...);
```

**Rules:**
- Right side must be a callable accepting exactly one argument
- Built-in functions that take multiple args need wrapping in an arrow fn or closure
- Inline lambdas need parentheses: `|> (fn($x) => doSomething($x))`
- Short built-ins work directly: `|> trim(...)`, `|> strtolower(...)`

```php
// Real-world: sanitize and normalize input
$cleanEmail = $rawInput
    |> trim(...)
    |> strtolower(...)
    |> fn(string $s) => filter_var($s, FILTER_SANITIZE_EMAIL);

// Processing pipeline
$processedItems = $rawItems
    |> fn(array $items) => array_filter($items, fn($i) => $i['active'])
    |> fn(array $items) => array_map(fn($i) => new Item($i), $items)
    |> fn(array $items) => array_values($items);

// Can pass as callable (first-class callable syntax)
$pipeline = strtolower(...);
$result   = $input |> $pipeline;
```

---

## Clone With (v2)

Clone an object and update specific properties in one expression. Solves
the long-standing pain of cloning `readonly` classes.

```php
// PHP 8.4 — verbose wither pattern
readonly class Money {
    public function __construct(
        public int $amount,
        public Currency $currency,
    ) {}

    // Had to restate all constructor args
    public function withAmount(int $amount): self {
        return new self($amount, $this->currency);
    }
}

// PHP 8.5 — clone with
readonly class Money {
    public function __construct(
        public int $amount,
        public Currency $currency,
    ) {}

    public function withAmount(int $amount): self {
        return clone($this, ['amount' => $amount]);
    }

    public function add(Money $other): self {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Currency mismatch.');
        }
        return clone($this, ['amount' => $this->amount + $other->amount]);
    }
}

$price    = new Money(1000, Currency::EUR);
$doubled  = clone($price, ['amount' => 2000]);
$inUSD    = clone($price, ['amount' => 1100, 'currency' => Currency::USD]);
```

`clone` now acts as a language construct with an optional second argument —
an array of property names to values. It honours `__clone()` and property
hooks. Can also be used as a first-class callable:

```php
$clones = array_map(clone(...), $originalObjects);
```

---

## `#[\NoDiscard]` Attribute

Marks a function/method whose return value must not be silently ignored.
PHP emits `E_USER_WARNING` when the return value is discarded.

```php
#[\NoDiscard('Result contains validation errors — ignoring it is a bug.')]
function validate(array $data): ValidationResult { /* ... */ }

validate($data);           // ⚠ Warning: return value discarded
$result = validate($data); // ✅ OK
(void) validate($data);    // ✅ OK — explicit discard suppresses warning

// Great for fluent builders, event dispatchers, query results
#[\NoDiscard]
public function withTimeout(int $seconds): self {
    return clone($this, ['timeout' => $seconds]);
}

// Built-in functions like json_encode already emit warnings on failure;
// use #[\NoDiscard] on your own APIs that callers tend to forget to check
#[\NoDiscard('Check for false on encoding failure')]
function encodePayload(array $data): string|false {
    return json_encode($data, JSON_THROW_ON_ERROR);
}
```

---

## URI Extension

Standards-compliant URI/URL parsing built into PHP. Always available — no
extension to install. Supports RFC 3986 (`Uri\Rfc3986\Uri`) and WHATWG URL
(`Uri\WhatWg\Url`). Both are `readonly` classes.

```php
use Uri\Rfc3986\Uri;
use Uri\WhatWg\Url;

// RFC 3986 — strict standard, great for APIs and custom protocols
$uri = new Uri('https://api.example.com/v1/products?page=2#results');

$uri->getScheme();    // 'https'
$uri->getHost();      // 'api.example.com'
$uri->getPath();      // '/v1/products'
$uri->getQuery();     // 'page=2'
$uri->getFragment();  // 'results'

// Immutable — "with" methods return new instances
$next = $uri->withQuery('page=3');
$prod = $uri->withPath('/v1/orders');

// WHATWG URL — browser-compatible, handles unicode, punycode
$url = new Url('https://münchen.de/events');
$url->getHost();      // 'xn--mnchen-3ya.de' (punycode normalized)

// Parse factory method
$uri = Uri::parse('https://example.com');   // returns Uri|null
$url = Url::parse('https://example.com');   // returns Url|null
```

**When to use which:**
- `Uri\Rfc3986\Uri` — APIs, custom protocols, strict validation
- `Uri\WhatWg\Url` — browser-like URL handling, user-facing input, web scraping

**Migration from `parse_url()`:**
```php
// Before
$parts = parse_url($input); // can return false or wrong results on malformed URLs
$host  = $parts['host'] ?? null;

// After
$uri  = Uri::parse($input);
$host = $uri?->getHost();
```

---

## `array_first()` / `array_last()`

```php
$users = ['Alice', 'Bob', 'Charlie'];

array_first($users);  // 'Alice'
array_last($users);   // 'Charlie'

// Associative arrays — returns value, not key
$data = ['name' => 'John', 'age' => 30];
array_first($data);   // 'John'
array_last($data);    // 30

// Empty array — returns null (does not throw)
array_first([]);      // null
array_last([]);       // null
```

Does not modify the array's internal pointer (unlike `reset()`/`end()`).

---

## Expanded Asymmetric Visibility: Static Properties

PHP 8.4 added asymmetric visibility for instance properties. PHP 8.5 extends
it to **static properties**.

```php
class Registry {
    public private(set) static int $count = 0;

    public static function register(): void {
        self::$count++; // ✅ can set inside class
    }
}

echo Registry::$count;     // ✅ public read
Registry::$count = 5;      // ❌ Error: private(set)
```

---

## `#[\Override]` on Properties (8.5)

```php
class Base {
    public int $timeout = 30;
}

class Extended extends Base {
    #[\Override]
    public int $timeout = 60; // ✅ attribute confirms intentional override
}

class Broken extends Base {
    #[\Override]
    public int $typo = 60;    // ❌ Fatal: no matching parent property
}
```

---

## `#[\Deprecated]` on Traits (8.5)

```php
#[\Deprecated('Use the HasTimestamps trait instead.')]
trait LegacyTimestamps {
    // ...
}

class Order {
    use LegacyTimestamps; // ⚠ Deprecated: Trait LegacyTimestamps used by Order is deprecated
}
```

---

## OPcache Always Compiled In

OPcache is now statically compiled into PHP (like `ext/date`, `ext/hash`).
No more `extension=opcache` in php.ini — it's always available, still
controlled by `opcache.enable = 1`.

---

## Key Deprecations in PHP 8.5

| Deprecated | Replace with |
|-----------|-------------|
| Backtick operator `` `cmd` `` | `shell_exec('cmd')` |
| `(boolean)`, `(integer)`, `(double)`, `(binary)` casts | `(bool)`, `(int)`, `(float)`, `(string)` |
| `null` as array key / in `array_key_exists()` | Use `''` (empty string) |
| `case X;` (semicolon in switch) | `case X:` (colon) |
| `disable_classes` INI setting | Removed entirely |

---

## PHP 8.5 Version Timeline

| Version | Active support until | Security until |
|---------|---------------------|---------------|
| PHP 8.4 | Dec 31, 2026 | Dec 31, 2028 |
| PHP 8.5 | Dec 31, 2027 | Dec 31, 2029 |
