# Hydra Console

The framework's batteries-included console commands — the generic verbs every
Hydra app needs, lifted out of the app skeleton so two projects don't carry two
copies. This package ships the *commands*; the app owns its `bin/console`
entrypoint, wires them to its composition root, and adds its own app-specific
generators.

There is no `ServiceProvider` and no seam to register. A command is a plain
`Symfony\Component\Console\Command\Command`; the app constructs each one with the
dependencies it needs and hands it to a `Symfony\Component\Console\Application`.
The package exists to be `new`-ed, not bound.

## What ships here

| Command | Class | Needs |
| --- | --- | --- |
| `key:generate` | `KeyGenerateCommand` | a path to `.env` |
| `make:migration` | `MakeMigrationCommand` | a migrations directory |
| `route:cache` | `RouteCacheCommand` | `Hydra\Http\RouteCache` + a controller list |
| `route:cache:clear` | `RouteCacheClearCommand` | `Hydra\Http\RouteCache` |
| `migrate:run` | `MigrateRunCommand` | `Hydra\Database\MigrationRunner` |
| `migrate:fresh` | `MigrateFreshCommand` | a `MigrationRunner` + the debug flag |
| `migrate:status` | `MigrateStatusCommand` | a `MigrationRunner` |

Every dependency is a constructor argument — a path, a list, an already-built
collaborator. Nothing is resolved from a container in here, so the same command
is trivially testable against a temp file or in-memory sqlite.

## The `MakeClassCommand` seam

`MakeClassCommand` is the abstract base the class-emitting generators share. It
owns the mechanics — turn a loose name into a PascalCase class (appending a fixed
suffix if missing), refuse to clobber an existing file without `--force`, write
the rendered stub. Subclasses supply only the policy: the target directory, the
suffix, and the file body.

The app's own generators live in the app, because their stubs are
app-namespaced (a `make:controller` emits into `App\Controllers`). They extend
this base:

```php
use Hydra\Console\Commands\MakeClassCommand;

final class MakeControllerCommand extends MakeClassCommand
{
    protected function nameHint(): string { return 'The controller name'; }
    protected function suffix(): string   { return 'Controller'; }
    protected function stub(string $class): string { /* app-namespaced body */ }
}
```

## Wiring it up

The app's `bin/console` builds the composition root and constructs commands from
it. DB-free commands are added eagerly; the `migrate:*` commands are loaded
lazily through a `FactoryCommandLoader` so the database is only touched when one
of them actually runs:

```php
$application->addCommands([
    new KeyGenerateCommand(__DIR__ . '/../.env'),
    new RouteCacheCommand($container->get(RouteCache::class), AppServiceProvider::CONTROLLERS),
    new RouteCacheClearCommand($container->get(RouteCache::class)),
    new MakeMigrationCommand(__DIR__ . '/../database/migrations'),
    // ...app-specific generators (make:controller, make:ability, make:user)
]);

$application->setCommandLoader(new FactoryCommandLoader([
    'migrate:run'    => static fn () => new MigrateRunCommand($container->get(MigrationRunner::class)),
    'migrate:fresh'  => static fn () => new MigrateFreshCommand($container->get(MigrationRunner::class), $debug),
    'migrate:status' => static fn () => new MigrateStatusCommand($container->get(MigrationRunner::class)),
]));
```
