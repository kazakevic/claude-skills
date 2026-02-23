# PHP Toolchain & Project Setup

## Project Structure (PSR-4)

```
project/
├── src/
│   ├── Domain/              # Entities, VOs, Enums, Domain Events, Interfaces
│   │   ├── Order.php
│   │   ├── OrderStatus.php
│   │   ├── Money.php
│   │   ├── Currency.php
│   │   └── Contracts/
│   │       ├── OrderRepository.php
│   │       └── EventDispatcher.php
│   ├── Application/         # Use cases / Services (orchestrate domain)
│   │   ├── CreateOrderService.php
│   │   └── ConfirmOrderService.php
│   └── Infrastructure/      # DB, HTTP, Queue implementations
│       ├── Persistence/
│       │   └── DoctrineOrderRepository.php
│       └── Http/
│           └── OrderController.php
├── tests/
│   ├── Unit/                # Pure domain tests — no I/O
│   │   └── Domain/
│   │       ├── MoneyTest.php
│   │       └── OrderTest.php
│   └── Integration/         # Tests that touch DB/HTTP
│       └── Persistence/
│           └── DoctrineOrderRepositoryTest.php
├── composer.json
├── phpunit.xml
├── phpstan.neon
└── rector.php
```

---

## composer.json

```json
{
    "require": {
        "php": "^8.4",
        "psr/clock": "^1.0",
        "psr/log": "^3.0",
        "psr/http-message": "^2.0",
        "psr/container": "^2.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^2.0",
        "phpstan/phpstan-strict-rules": "^1.6",
        "phpstan/phpstan-deprecation-rules": "^1.2",
        "rector/rector": "^2.0",
        "friendsofphp/php-cs-fixer": "^3.0"
    },
    "autoload": {
        "psr-4": { "App\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "Tests\\": "tests/" }
    },
    "scripts": {
        "test":    "phpunit",
        "analyse": "phpstan analyse",
        "fix":     "php-cs-fixer fix",
        "rector":  "rector process --dry-run",
        "check":   ["@test", "@analyse"]
    },
    "config": {
        "optimize-autoloader": true,
        "sort-packages": true,
        "platform": { "php": "8.4.0" }
    }
}
```

---

## PHPStan

```bash
composer require --dev phpstan/phpstan phpstan/phpstan-strict-rules
```

```neon
# phpstan.neon
parameters:
    level: 8            # Start at 6, aim for 8-9 on greenfield projects
    paths:
        - src
        - tests
    checkMissingIterableValueType: true
    checkGenericClassInNonGenericObjectType: true
    treatPhpDocTypesAsCertain: false
    reportUnmatchedIgnoredErrors: true

includes:
    - vendor/phpstan/phpstan-strict-rules/rules.neon
    - vendor/phpstan/phpstan-deprecation-rules/rules.neon
```

```bash
vendor/bin/phpstan analyse --memory-limit=512M
```

**PHPStan generics in PHPDoc** (no runtime cost, full static analysis):

```php
/**
 * @template T of object
 * @param list<T> $items
 * @param callable(T): bool $predicate
 * @return list<T>
 */
function filter(array $items, callable $predicate): array {
    return array_values(array_filter($items, $predicate));
}

/**
 * @implements Repository<Order>
 */
class InMemoryOrderRepository implements Repository { ... }
```

---

## Rector

Automated PHP upgrades and refactoring.

```bash
composer require --dev rector/rector
```

```php
<?php
// rector.php
declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;
use Rector\PHPUnit\Set\PHPUnitSetList;

return RectorConfig::configure()
    ->withPaths([__DIR__ . '/src', __DIR__ . '/tests'])
    ->withSets([
        LevelSetList::UP_TO_PHP_84,        // Apply all PHP 8.x features
        SetList::CODE_QUALITY,              // Simplify code patterns
        SetList::DEAD_CODE,                 // Remove unused code
        SetList::TYPE_DECLARATION,          // Add missing type declarations
        PHPUnitSetList::PHPUNIT_110,        // Migrate to PHPUnit 11 attributes
    ]);
```

```bash
vendor/bin/rector process --dry-run  # preview
vendor/bin/rector process            # apply
```

Common transformations Rector handles automatically:
- Add `declare(strict_types=1)`
- Convert to constructor property promotion
- Replace `@deprecated` docblocks with `#[\Deprecated]`
- Migrate PHPUnit docblock annotations to attributes
- Convert `switch` to `match` where safe

---

## PHP-CS-Fixer

```php
<?php
// .php-cs-fixer.php
$finder = PhpCsFixer\Finder::create()
    ->in(__DIR__ . '/src')
    ->in(__DIR__ . '/tests');

return (new PhpCsFixer\Config())
    ->setRules([
        '@PER-CS2.0'          => true,
        '@PHP84Migration'     => true,
        'declare_strict_types'=> true,
        'trailing_comma_in_multiline' => ['elements' => ['arguments', 'arrays', 'match', 'parameters']],
        'ordered_imports'     => ['sort_algorithm' => 'alpha'],
        'no_unused_imports'   => true,
    ])
    ->setFinder($finder);
```

```bash
vendor/bin/php-cs-fixer fix          # fix
vendor/bin/php-cs-fixer fix --diff   # preview diff
```

---

## CI Pipeline (GitHub Actions example)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          coverage: xdebug
      - run: composer install --no-progress
      - run: vendor/bin/phpunit --coverage-clover coverage.xml
      - run: vendor/bin/phpstan analyse --no-progress
      - run: vendor/bin/php-cs-fixer fix --diff --dry-run
```

---

## Deprecations to Fix Before PHP 9

**PHP 8.4 deprecations:**

| Issue | Fix |
|-------|-----|
| `function foo(string $x = null)` | `function foo(?string $x = null)` |
| `$obj->undeclaredProp = 'x'` | Declare the property explicitly |
| Old DOM (`DOMDocument`) | Use `Dom\HTMLDocument` / `Dom\XMLDocument` |
| `FILTER_SANITIZE_STRING` | Use `htmlspecialchars()` or `strip_tags()` |
| `$str++` on string var | Use `str_increment($str)` |
| `@deprecated` docblocks | Use `#[\Deprecated(message: '...')]` |
| PHPUnit docblock annotations | Run `vendor/bin/rector process` with PHPUnitSetList |

**PHP 8.5 deprecations:**

| Issue | Fix |
|-------|-----|
| Backtick operator `` `cmd` `` | `shell_exec('cmd')` |
| `(boolean)`, `(integer)`, `(double)`, `(binary)` casts | `(bool)`, `(int)`, `(float)`, `(string)` |
| `null` as array key / in `array_key_exists()` | Use `''` (empty string) |
| `case X;` (semicolon in switch) | `case X:` (colon) |

Run `vendor/bin/phpstan analyse` with `phpstan-deprecation-rules` to catch all of these automatically.
