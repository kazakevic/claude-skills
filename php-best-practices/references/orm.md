# Doctrine ORM (PHP Attributes Mapping)

Use Doctrine ORM 3.x with PHP Attribute mappings (no XML, no YAML, no docblock annotations).

## Setup

```json
{
  "require": {
    "doctrine/orm": "^3.0",
    "doctrine/dbal": "^4.0"
  }
}
```

---

## Entity Mapping with Attributes

```php
<?php
declare(strict_types=1);

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: ProductRepository::class)]
#[ORM\Table(name: 'products')]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer', options: ['unsigned' => true])]
    private int $id;

    #[ORM\Column(type: 'string', length: 200)]
    private string $name;

    #[ORM\Column(name: 'price_cents', type: 'integer', options: ['unsigned' => true])]
    private int $priceCents;

    #[ORM\Column(name: 'currency', type: 'string', length: 3, enumType: Currency::class)]
    private Currency $currency;

    #[ORM\Column(name: 'created_at', type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(name: 'updated_at', type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $updatedAt = null;

    public function __construct(string $name, int $priceCents, Currency $currency)
    {
        $this->name       = $name;
        $this->priceCents = $priceCents;
        $this->currency   = $currency;
        $this->createdAt  = new \DateTimeImmutable();
    }

    // Getters — Doctrine needs private properties to be accessible
    public function getId(): int                          { return $this->id; }
    public function getName(): string                     { return $this->name; }
    public function getPriceCents(): int                  { return $this->priceCents; }
    public function getCurrency(): Currency               { return $this->currency; }
    public function getCreatedAt(): \DateTimeImmutable    { return $this->createdAt; }
    public function getUpdatedAt(): ?\DateTimeImmutable   { return $this->updatedAt; }

    // Domain logic
    public function rename(string $name): void {
        if (strlen($name) < 1) throw new \InvalidArgumentException('Name empty.');
        $this->name      = trim($name);
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function price(): Money {
        return new Money($this->priceCents, $this->currency);
    }
}
```

> **Note on property hooks + Doctrine**: Doctrine 3.x does not yet support PHP 8.4 property hooks on mapped properties. Use private properties + public getter methods for mapped columns. You can still use hooks on computed/unmapped properties.

---

## Relationships

### One-to-Many / Many-to-One

```php
#[ORM\Entity]
#[ORM\Table(name: 'orders')]
class Order
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;

    #[ORM\Column(name: 'customer_id', type: 'integer')]
    private int $customerId;

    #[ORM\Column(name: 'status', type: 'string', enumType: OrderStatus::class)]
    private OrderStatus $status = OrderStatus::Draft;

    /** @var \Doctrine\Common\Collections\Collection<int, OrderItem> */
    #[ORM\OneToMany(
        targetEntity: OrderItem::class,
        mappedBy: 'order',
        cascade: ['persist', 'remove'],
        orphanRemoval: true,
    )]
    private \Doctrine\Common\Collections\Collection $items;

    public function __construct(int $customerId)
    {
        $this->customerId = $customerId;
        $this->items      = new \Doctrine\Common\Collections\ArrayCollection();
    }

    public function addItem(OrderItem $item): void
    {
        if ($this->status !== OrderStatus::Draft) {
            throw new \DomainException('Cannot modify confirmed order.');
        }
        $this->items->add($item);
    }

    /** @return list<OrderItem> */
    public function getItems(): array { return $this->items->toArray(); }
}

#[ORM\Entity]
#[ORM\Table(name: 'order_items')]
class OrderItem
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;

    #[ORM\ManyToOne(targetEntity: Order::class, inversedBy: 'items')]
    #[ORM\JoinColumn(nullable: false)]
    private Order $order;

    #[ORM\ManyToOne(targetEntity: Product::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Product $product;

    #[ORM\Column(type: 'integer')]
    private int $quantity;
}
```

### Many-to-Many

```php
/** @var \Doctrine\Common\Collections\Collection<int, Tag> */
#[ORM\ManyToMany(targetEntity: Tag::class, cascade: ['persist'])]
#[ORM\JoinTable(
    name: 'product_tags',
    joinColumns: [new ORM\JoinColumn(name: 'product_id', referencedColumnName: 'id')],
    inverseJoinColumns: [new ORM\JoinColumn(name: 'tag_id', referencedColumnName: 'id')],
)]
private \Doctrine\Common\Collections\Collection $tags;
```

---

## Repository

```php
<?php
declare(strict_types=1);

use Doctrine\ORM\EntityManagerInterface;
use Doctrine\ORM\EntityRepository;

/**
 * @extends EntityRepository<Product>
 */
final class ProductRepository extends EntityRepository implements ProductRepositoryInterface
{
    public function __construct(private EntityManagerInterface $em)
    {
        parent::__construct($em, $em->getClassMetadata(Product::class));
    }

    public function find(int $id): ?Product
    {
        return $this->em->find(Product::class, $id);
    }

    /** @return list<Product> */
    public function findAll(): array
    {
        return $this->createQueryBuilder('p')
            ->orderBy('p.createdAt', 'DESC')
            ->getQuery()
            ->getResult();
    }

    /** @return list<Product> */
    public function findByPriceRange(int $minCents, int $maxCents): array
    {
        return $this->createQueryBuilder('p')
            ->where('p.priceCents BETWEEN :min AND :max')
            ->setParameter('min', $minCents)
            ->setParameter('max', $maxCents)
            ->orderBy('p.priceCents', 'ASC')
            ->getQuery()
            ->getResult();
    }

    public function save(Product $product): void
    {
        $this->em->persist($product);
        $this->em->flush();
    }

    public function delete(Product $product): void
    {
        $this->em->remove($product);
        $this->em->flush();
    }
}
```

---

## EntityManager Configuration

```php
<?php
declare(strict_types=1);

use Doctrine\ORM\EntityManager;
use Doctrine\ORM\ORMSetup;
use Doctrine\DBAL\DriverManager;

// PHP 8 Attribute mapping
$config = ORMSetup::createAttributeMetadataConfiguration(
    paths: [__DIR__ . '/src/Domain'],
    isDevMode: true,
);

$connection = DriverManager::getConnection([
    'driver'   => 'pdo_pgsql',
    'host'     => 'localhost',
    'port'     => 5432,
    'dbname'   => 'myapp',
    'user'     => 'app',
    'password' => 'secret',
], $config);

$em = new EntityManager($connection, $config);
```

---

## Migrations (doctrine/migrations)

```bash
composer require doctrine/migrations

# Generate migration from entity changes
vendor/bin/doctrine-migrations diff

# Run pending migrations
vendor/bin/doctrine-migrations migrate

# Show status
vendor/bin/doctrine-migrations status
```

```php
// Example generated migration
use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20260101000000 extends AbstractMigration
{
    public function up(Schema $schema): void
    {
        $this->addSql('CREATE TABLE products (
            id INT UNSIGNED AUTO_INCREMENT NOT NULL,
            name VARCHAR(200) NOT NULL,
            price_cents INT UNSIGNED NOT NULL,
            currency CHAR(3) NOT NULL,
            created_at DATETIME NOT NULL COMMENT \'(DC2Type:datetime_immutable)\',
            PRIMARY KEY(id)
        ) DEFAULT CHARACTER SET utf8mb4');
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP TABLE products');
    }
}
```

---

## Testing with Doctrine (Integration)

```php
<?php
declare(strict_types=1);

use Doctrine\ORM\EntityManagerInterface;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

abstract class DoctrineTestCase extends TestCase
{
    protected EntityManagerInterface $em;

    protected function setUp(): void
    {
        // Boot a test EntityManager with an in-memory SQLite DB
        $config = ORMSetup::createAttributeMetadataConfiguration(
            [__DIR__ . '/../../src/Domain'],
            isDevMode: true,
        );
        $conn   = DriverManager::getConnection(['driver' => 'pdo_sqlite', 'memory' => true], $config);
        $this->em = new EntityManager($conn, $config);

        // Create schema
        $schemaTool = new \Doctrine\ORM\Tools\SchemaTool($this->em);
        $schemaTool->createSchema($this->em->getMetadataFactory()->getAllMetadata());
    }
}

final class ProductRepositoryTest extends DoctrineTestCase
{
    #[Test]
    public function it_persists_and_retrieves_product(): void
    {
        $repo    = new ProductRepository($this->em);
        $product = new Product('Widget', 999, Currency::EUR);

        $repo->save($product);
        $this->em->clear();

        $found = $repo->find($product->getId());
        $this->assertNotNull($found);
        $this->assertSame('Widget', $found->getName());
    }
}
```
