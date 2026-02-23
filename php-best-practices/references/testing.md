# PHPUnit 11 Testing

PHPUnit 11 uses **Attributes** as the primary API. Docblock annotations are
deprecated and will be removed. Always use Attribute-based configuration.

## Setup

```json
{ "require-dev": { "phpunit/phpunit": "^11.0" } }
```

```xml
<!-- phpunit.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="vendor/autoload.php"
         colors="true"
         failOnWarning="true"
         failOnRisky="true">
    <testsuites>
        <testsuite name="unit">        <directory>tests/Unit</directory>        </testsuite>
        <testsuite name="integration"> <directory>tests/Integration</directory> </testsuite>
    </testsuites>
    <source>
        <include><directory>src</directory></include>
    </source>
    <coverage>
        <report>
            <html outputDirectory="coverage/html"/>
            <clover outputFile="coverage/clover.xml"/>
        </report>
    </coverage>
</phpunit>
```

---

## Test Anatomy

```php
<?php
declare(strict_types=1);

namespace Tests\Unit\Domain;

use App\Domain\Money;
use App\Domain\Currency;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\TestWith;
use PHPUnit\Framework\TestCase;

#[CoversClass(Money::class)]
final class MoneyTest extends TestCase
{
    // setUp/tearDown still work as normal methods
    private Money $fiveEuro;

    protected function setUp(): void
    {
        $this->fiveEuro = new Money(500, Currency::EUR);
    }

    #[Test]
    public function it_stores_amount_and_currency(): void
    {
        $this->assertSame(500, $this->fiveEuro->amount);
        $this->assertSame(Currency::EUR, $this->fiveEuro->currency);
    }

    #[Test]
    public function it_rejects_negative_amount(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('Negative amount');

        new Money(-1, Currency::EUR);
    }

    #[Test]
    public function it_adds_same_currency(): void
    {
        $result = $this->fiveEuro->add(new Money(300, Currency::EUR));

        $this->assertSame(800, $result->amount);
    }

    #[Test]
    public function it_throws_on_currency_mismatch(): void
    {
        $this->expectException(\DomainException::class);

        $this->fiveEuro->add(new Money(100, Currency::USD));
    }

    // ── Data Providers ──────────────────────────────────────────────────────

    /**
     * @return array<string, array{int, Currency, string}>
     */
    public static function formatProvider(): array
    {
        return [
            'zero EUR'  => [0,    Currency::EUR, '0.00 €'],
            'small EUR' => [99,   Currency::EUR, '0.99 €'],
            'USD'       => [2500, Currency::USD, '25.00 $'],
        ];
    }

    #[Test]
    #[DataProvider('formatProvider')]
    public function it_formats_correctly(int $cents, Currency $currency, string $expected): void
    {
        $money = new Money($cents, $currency);
        $this->assertSame($expected, $money->format());
    }

    // Inline data for simple cases
    #[Test]
    #[TestWith([0])]
    #[TestWith([1])]
    #[TestWith([9999])]
    public function it_accepts_non_negative_amount(int $amount): void
    {
        $money = new Money($amount, Currency::EUR);
        $this->assertSame($amount, $money->amount);
    }
}
```

---

## Testing Entities (with setup)

```php
<?php
declare(strict_types=1);

namespace Tests\Unit\Domain;

use App\Domain\Order;
use App\Domain\OrderStatus;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

#[CoversClass(Order::class)]
final class OrderTest extends TestCase
{
    private Order $order;

    protected function setUp(): void
    {
        $this->order = new Order(
            id: 1,
            customerId: 42,
            createdAt: new \DateTimeImmutable('2026-01-01'),
        );
    }

    #[Test]
    public function it_starts_as_draft(): void
    {
        $this->assertSame(OrderStatus::Draft, $this->order->status);
    }

    #[Test]
    public function it_cannot_confirm_empty_order(): void
    {
        $this->expectException(\DomainException::class);
        $this->expectExceptionMessage('empty order');

        $this->order->confirm();
    }

    #[Test]
    public function it_transitions_to_confirmed(): void
    {
        $product = $this->makeProduct(priceCents: 999);
        $this->order->addItem($product, 1);
        $this->order->confirm();

        $this->assertSame(OrderStatus::Confirmed, $this->order->status);
    }

    #[Test]
    public function it_prevents_modification_after_confirmation(): void
    {
        $product = $this->makeProduct();
        $this->order->addItem($product, 1);
        $this->order->confirm();

        $this->expectException(\DomainException::class);
        $this->order->addItem($product, 1);
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    private function makeProduct(int $priceCents = 100): Product
    {
        return new Product('Widget', $priceCents, Currency::EUR);
    }
}
```

---

## Mocks & Fakes

```php
<?php
declare(strict_types=1);

namespace Tests\Unit\Application;

use App\Application\CreateOrderService;
use App\Domain\OrderRepository;
use Psr\Clock\ClockInterface;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

#[CoversClass(CreateOrderService::class)]
final class CreateOrderServiceTest extends TestCase
{
    #[Test]
    public function it_persists_new_order(): void
    {
        $clock     = $this->createMock(ClockInterface::class);
        $repo      = $this->createMock(OrderRepository::class);
        $frozenNow = new \DateTimeImmutable('2026-03-15');

        $clock->method('now')->willReturn($frozenNow);
        $repo->expects($this->once())->method('save');

        $service = new CreateOrderService($repo, $clock);
        $order   = $service->create(customerId: 1, currency: Currency::EUR);

        $this->assertSame($frozenNow, $order->createdAt);
    }
}
```

**Prefer hand-written fakes over mocks for repositories:**
```php
final class InMemoryOrderRepository implements OrderRepository
{
    /** @var array<int, Order> */
    private array $store = [];
    private int $nextId  = 1;

    public function find(int $id): ?Order     { return $this->store[$id] ?? null; }
    public function findAll(): array          { return array_values($this->store); }
    public function save(object $entity): void { $this->store[$entity->id] = $entity; }
    public function delete(int $id): void     { unset($this->store[$id]); }
    public function findByCustomer(int $customerId): array {
        return array_values(array_filter(
            $this->store,
            fn(Order $o) => $o->customerId === $customerId,
        ));
    }
}
```

---

## PHPUnit 11 Attribute Reference

| Attribute | Purpose |
|-----------|---------|
| `#[Test]` | Marks a method as a test |
| `#[DataProvider('method')]` | External data provider |
| `#[TestWith([...])]` | Inline data (simple cases) |
| `#[CoversClass(Foo::class)]` | Coverage tracking |
| `#[CoversMethod(Foo::class, 'bar')]` | Covers specific method |
| `#[Depends('testFoo')]` | Execution dependency |
| `#[Group('slow')]` | Filter group |
| `#[RequiresPhp('>= 8.4')]` | Skip if PHP version unmet |
| `#[Before]` | Run before each test |
| `#[BeforeClass]` | Run once before all tests |
| `#[WithoutErrorHandler]` | Disable PHPUnit error handler |

---

## Running Tests

```bash
vendor/bin/phpunit                         # all suites
vendor/bin/phpunit --testsuite unit        # unit only
vendor/bin/phpunit --group slow            # by group
vendor/bin/phpunit --coverage-html coverage/html  # with coverage
vendor/bin/phpunit tests/Unit/Domain/MoneyTest.php  # single file
```
