# container

A type-keyed DI container for Echo. You bind factories on a `Builder` at the composition root and resolve by type on the sealed `Container`. Types in the middle take constructors and never mention either.

- `get<Logger>()` returns a `Logger`. The type is the key.
- Lifetimes are labels: `once:`, `each:`, `scope:`, `put:`.
- `param<Hostname>('smtp.example.com')` stores a config fact. The type is the name; there is no string parameter id. Constructor args of `T` are gathered at the call.
- `bind` returns an `Entry` so you can hang typed metadata on a row (`RouteMeta`, and so on) and discover those rows with `with<M>()`.
- Interface keys work. The container erases the binding, not `T`, so `Logger` does not have to be a class.
- There is no reflection and no string service id.

```echo
use container::{Builder, Container};

interface Logger
{
    function write(const string& $line) : void;
}

class FileLogger : Logger
{
    public string $path;

    public constructor(string $path)
    {
        $this->path = $path;
    }

    public function write(const string& $line) : void
    {
        echo $line;
    }
}

class UserService
{
    public Logger $log;

    public constructor(Logger $log)
    {
        $this->log = $log;
    }
}

Builder $b = Builder();
$b->bind<Logger>(once: function() : Logger {
    return FileLogger('app.log');
});
$b->bind<UserService>(once: function(Container $c) : UserService {
    return UserService($c->get<Logger>());
});
Container $app = $b->seal(warm: true);

UserService $users = $app->get<UserService>();
```

Factories take a `Container`. Packs take a `Builder`. A factory cannot `bind`.

## Install

```bash
epm add --path /path/to/container
```

Or from git once this repo is published:

```bash
epm add echolibs/container --git https://github.com/echolibs/container --range ^0.1
```

## Lifetimes

`once:` runs the factory the first time someone asks, then hands back the same value. The cache lives on the binding, shared by every child.

`each:` runs the factory on every `get`.

`scope:` runs the factory the first time this container asks, then caches on that container. A child has its own cache. Bind `scope:` for anything that reads a child `put` (a `Request`, a session).

`put:` stores a value you already built, or constructs one from the arguments at the call (`bind<Hostname>(put: 'smtp.example.com')`). On the builder it is a catalog binding. On a sealed container it is an overlay for this ask.

A factory can take no arguments, or it can take the container being resolved against. Use the second shape when the factory looks up collaborators, especially from a child:

```echo
$b->bind<Handler>(scope: function(Container $c) : Handler {
    return Handler($c->get<Logger>(), $c->get<Request>());
});
```

A `once` service that pulls a request-scoped collaborator keeps the first request forever. Bind that service `scope:` or `each:`.

## Parameters

A hostname in the catalog is a `Hostname`. The type is the key: `Mailer($c->get<Hostname>(), $c->get<Port>())` type-checks the two values apart.

```echo
public struct Hostname
{
    public const string $value;
}

public struct Port
{
    public const int32 $value;
}

Builder $b = Builder();
$b->param<Hostname>('smtp.example.com');
$b->param<Port>(587);
$b->bind<Mailer>(once: function(Container $c) : Mailer {
    return Mailer($c->get<Hostname>(), $c->get<Port>());
});
```

`param` is `bind(put:)` under a name that means config. Same catalog, same duplicate/replace/overlay rules. Set params, then install packs that read them. `Builder.get<T>()` returns an already-built value and does not run factories, so a pack can read `Hostname` at install time:

```echo
$b->param<Hostname>('smtp.example.com');
$b->install(function(Builder& $c) : void {
    Hostname $host = $c->get<Hostname>();
    $c->bind<Mailer>(once: function() : Mailer {
        return Mailer($host);
    });
});
```

A pack that owns several settings groups them on one fact instead of one type per field:

```echo
public struct Options
{
    public const string $host;
    public const int32 $port;
}

$b->param<mailer::Options>('smtp.example.com', 587);
```

Shared values (a hostname two packs both need) stay their own type. `param('localhost')` is legal and is the only `string` in the graph. Wrap the scalar when you have more than one.

After `seal`, the word is `get`. A child may `put<Hostname>` to shadow the fact for one ask.

## Children

`$app->child()` is a nested container. Lookups that miss there walk parent overlays, then the sealed catalog. The child can `put` a value that exists only for this ask:

```echo
Container $req = $app->child();
$req->put<Request>($request);
Handler $h = $req->get<Handler>();
$req->drop();
```

`put` on a child shadows `get` and `fetch` for that type. It does not add a catalog row and does not hide a parent row in `with<M>()`.

## Metadata

`bind` returns the row. You can ignore it. You can also record a typed value on it.

```echo
$b->bind<GetUserAction>(once: function(Container $c) : GetUserAction {
    return GetUserAction($c->get<UserRepo>());
})
    ->meta<RouteMeta>($method: .get, $path: '/v1/users/{id}')
    ->view<Handler>();
```

`view<I>()` is how many different actions surface as one `Handler` in the catalog. `alias<I, C>()` is how one concrete type answers `get<I>()`.

| | `view<I>()` | `alias<I, C>()` |
|---|---|---|
| Registers a `get<I>()` key | no | yes |
| `with<M>()` then `fetch<I>()` | yes | if `I` is the bound type |
| How many implementations | N rows | one |

(`as` is the recast operator, so the method cannot take that name.) `with<RouteMeta>()` lists every catalog row that carries that metadata, in bind order.

```echo
foreach ($app->with<RouteMeta>() as &$row) {
    RouteMeta $route = $row->meta<RouteMeta>();
    Handler $action = $row->fetch<Handler>();
}
```

`RouteMeta` and `Handler` are your types. The container only stores them.

## Packs, seal, warm, drop

Libraries export a function. The app calls `install`. A second pack that `bind`s the same type dies. `overlay` runs a pack so `bind` of an existing type replaces. `replace` after install is the app override; it dies if the type was never bound.

```echo
$b->install(function(Builder& $c) : void {
    $c->bind<Logger>(once: function() : Logger {
        return FileLogger('app.log');
    });
});
$b->overlay(VerboseLogPack());
$b->replace<Logger>(put: $testLog);
Container $app = $b->seal();
$app->warm();
```

`seal()` freezes the catalog and returns the graph. `seal(warm: true)` also constructs every `once` binding. `warm()` on a child dies. Seal and warm on one thread; workers each take a child. Do not `get` concurrently on the same `Container`. `#[atomic]` makes the refcount safe so a worker can hold a copy of the handle and resolve alone.

`drop()` releases this container's overlays and scoped instances in reverse construction order. On the root it also releases every `once` cache. Drop the request child at the end of the request. Drop the root at process end. A `get` of a factory after `drop` reconstructs. Catalog `put` and `param` facts stay. Services that need a close implement `destructor`; reverse release is what lets dependents go first.

A missing binding, a duplicate bind, a replace of a missing type, a cycle, a write after seal, a warm on a child, a `get` of a factory on the builder, missing metadata, or a missing view stops the program in every build mode. The messages name the rule (`container: missing binding`). They cannot name the type: `type_id` is not printable.

`replace` on the builder is the intentional override. Tests swap a `Clock`. A second `bind` of the same type is a wiring bug and dies.

## This is not a blackboard

A per-cycle type-keyed bus for pipeline facts is architecture pattern A2. This library builds long-lived services. Put the `Request` on a child if handlers resolve through the container. Leave import summaries and exchange rates on a blackboard.

## Develop

```bash
echoc test
echoc run -m . examples/App.eco
```
