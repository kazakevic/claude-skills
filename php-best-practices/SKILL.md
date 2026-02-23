---
name: php-best-practices
description: >
  Modern PHP 8.4 best practices, patterns, and code generation. Use this skill
  whenever the user asks to write, review, or refactor PHP code — including
  writing entities, value objects, services, tests, ORM mappings, or setting
  up a project. Trigger on: PHP classes, interfaces, enums, property hooks,
  constructor promotion, attributes, PHPUnit tests, Doctrine entities, PHPStan
  config, Rector setup, or any PHP architecture question. Also trigger when the
  user asks to "add tests", "write a PHP class", "create an entity", "map to
  database", or "set up static analysis" in a PHP context. Covers PHP 8.0–8.4.
---

# PHP Best Practices (8.0–8.4)

Always start with `declare(strict_types=1);`. All code targets PHP 8.4 unless
specified otherwise. Never use framework-specific patterns (no Laravel facades,
no Symfony DI annotations) — use pure PHP and PSR standards.

## Reference Files — Load Only What You Need

Read the relevant file(s) before writing any code. Choose based on the task:

| Task | File to read |
|------|-------------|
| Language features, type system, enums, readonly, named args, match | `references/core-features.md` |
| PHP 8.4 specifics: property hooks, asymmetric visibility, array helpers | `references/php84.md` |
| Entities, Value Objects, Domain Events, PHP Attributes metadata | `references/entities.md` |
| PHPUnit 11 tests, mocks, data providers, test structure | `references/testing.md` |
| PHPStan, Rector, project layout, composer.json, toolchain | `references/tooling.md` |
| Doctrine ORM mappings, repositories, migrations | `references/orm.md` |

**Multiple tasks?** Read multiple files. Example: writing a tested entity with
Doctrine mapping → read `entities.md` + `testing.md` + `orm.md`.

## Quick Principles (always apply, no file read needed)

- `declare(strict_types=1)` on every file
- Constructor property promotion for all injected/simple props
- Enums over string/int constants for any finite set of values  
- `readonly` / `readonly class` for immutable data
- Return types on every method (including `void` and `never`)
- `final` on classes by default; only remove when extension is needed
- Named arguments for clarity at call sites with 3+ params
- PSR-4 autoloading, PSR-3 logging, PSR-7 HTTP, PSR-11 container
- No `array` types without PHPDoc generics (`@return list<User>`)
- No `mixed` without explicit justification
