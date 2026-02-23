# PHP 8.4 Features

## Property Hooks

Define `get`/`set` logic directly on a property. Replaces getter/setter methods.

### Basic Syntax

```php
class User {
    // set hook with validation
    public string $email {
        set {
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new \InvalidArgumentException("Invalid email: $value");
            }
            $this->email = strtolower($value);
        }
    }

    // get hook — transforms on read
    public string $username {
        get => ucfirst($this->username);
        set {
            if (strlen($value) < 3) throw new \LengthException('Min 3 chars');
            $this->username = trim($value);
        }
    }
}
```

### Virtual Properties (no backing storage)

```php
class Rectangle {
    public int $area {
        get => $this->width * $this->height;
    }

    public function __construct(
        public int $width,
        public int $height,
    ) {}
}

$r = new Rectangle(4, 5);
echo $r->area;    // 20 — computed, not stored
$r->area = 10;    // Error: no set hook defined
```

### Bidirectional Conversion

```php
class Temperature {
    public float $fahrenheit {
        get => ($this->celsius * 9 / 5) + 32;
        set => $this->celsius = ($value - 32) * 5 / 9;
    }
    public function __construct(public float $celsius) {}
}

$t = new Temperature(100);
echo $t->fahrenheit; // 212
$t->fahrenheit = 32;
echo $t->celsius;    // 0
```

### Hooks in Interfaces

```php
interface Nameable {
    public string $name { get; }        // must be publicly readable
}

interface Auditable {
    public \DateTimeImmutable $createdAt { get; }
    public ?\DateTimeImmutable $updatedAt { get; }
}

class Article implements Nameable, Auditable {
    public string $name { get; set; }
    public \DateTimeImmutable $createdAt { get; }
    public ?\DateTimeImmutable $updatedAt { get; set; }

    public function __construct(string $name) {
        $this->name = $name;
        $this->createdAt = new \DateTimeImmutable();
        $this->updatedAt = null;
    }
}
```

### Rules & Limitations

- **Incompatible with `readonly`** — readonly has no set semantic after construction
- **No static property hooks** — only instance properties
- Short form: `get => expr` is an arrow function body
- Set parameter type must be same or wider (contravariant) than property type
- Access `$this` inside hooks normally; accessing another hooked property calls its hooks too

### Best Practices

```php
// ✅ Keep hooks simple — complex logic belongs in methods
public string $slug {
    set => $this->slug = $this->generateSlug($value);
}

// ✅ Use for validation + normalization
public string $phone {
    set {
        $cleaned = preg_replace('/\D/', '', $value);
        if (strlen($cleaned) < 7) throw new \InvalidArgumentException('Invalid phone');
        $this->phone = $cleaned;
    }
}

// ❌ Don't add hooks just because you can — plain properties are fine for simple data
public string $name { get; set; } // redundant — just use: public string $name;
```

---

## Asymmetric Visibility

Different access levels for reading and writing.

```php
class Order {
    // Public read, private write — only this class mutates status
    public private(set) Status $status = Status::Draft;

    // Public read, protected write — subclasses can update
    public protected(set) \DateTimeImmutable $updatedAt;

    public function confirm(): void {
        $this->status = Status::Confirmed; // ✅ inside class
        $this->updatedAt = new \DateTimeImmutable();
    }
}

$order = new Order();
echo $order->status->value;         // ✅ public read
$order->status = Status::Confirmed; // ❌ Error: private(set)
```

Works with constructor promotion:
```php
class Product {
    public function __construct(
        public private(set) int $id,
        public private(set) string $sku,
        public string $name,           // fully public
    ) {}
}
```

---

## New Chaining Syntax

No parentheses needed around `new` to chain methods.

```php
// PHP 8.3 — required wrapping
$user = (new UserBuilder())->withName('Alice')->build();

// PHP 8.4 — clean
$user = new UserBuilder()->withName('Alice')->build();

$result = new QueryBuilder('products')
    ->select('id', 'name', 'price')
    ->where('active', true)
    ->orderBy('name')
    ->limit(20)
    ->get();
```

---

## `#[\Deprecated]` Attribute

Emits `E_USER_DEPRECATED` automatically when marked element is called. Machine-readable, IDE-navigable.

```php
class UserRepository {
    #[\Deprecated(message: 'Use findByEmail() instead.', since: '2.0')]
    public function findUser(string $email): ?User {
        return $this->findByEmail($email);
    }

    public function findByEmail(string $email): ?User { /* ... */ }
}
```

---

## New Array Functions (8.4)

```php
$users = [
    ['name' => 'Alice', 'age' => 30, 'admin' => true],
    ['name' => 'Bob',   'age' => 17, 'admin' => false],
];

// First element matching predicate (or null)
$admin    = array_find($users, fn($u) => $u['admin']);

// Key of first matching element (or null)
$adminKey = array_find_key($users, fn($u) => $u['admin']);

// True if any element matches
$hasMinor = array_any($users, fn($u) => $u['age'] < 18);  // true

// True if all elements match
$allAdults = array_all($users, fn($u) => $u['age'] >= 18); // false
```

---

## New DOM API

Standards-compliant HTML5 parsing under the `Dom\` namespace.

```php
$dom = \Dom\HTMLDocument::createFromString(
    '<main><article class="featured">Content</article></main>',
    LIBXML_NOERROR,
);

$node = $dom->querySelector('article.featured');
echo $node->textContent;                      // Content
$node->classList->contains('featured');       // true
$dom->querySelectorAll('article')->count();   // 1
```

Legacy `DOMDocument` is deprecated in 8.4. Use `Dom\HTMLDocument` / `Dom\XMLDocument` for new code.

---

## BcMath OO API (8.4)

```php
$a = new \BcMath\Number('10.5');
$b = new \BcMath\Number('2.3');

$result = $a->add($b);   // BcMath\Number('12.8')
$result = $a->mul($b);   // BcMath\Number('24.15')
echo $result;             // '24.15'
```
