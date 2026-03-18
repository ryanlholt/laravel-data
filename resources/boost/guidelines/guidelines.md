# Laravel Data Development Guidelines

## Core Principles

**Type Safety First:** Leverage PHP's type system to auto-generate validation rules.

**Single Source of Truth:** Define data structure once, use everywhere (validation, API resources, TypeScript).

**Explicit Over Implicit:** Use attributes and docblocks for clarity.

## Data Object Design

- Keep data objects focused on a single concern
- Use nested data objects for complex structures
- Prefer immutability
- Suffix classes with `Data`: `UserData`, `CreatePostData`, `PostResource`

```php
class SongData extends Data
{
    public function __construct(
        public string $title,
        public int $plays,
        public ?string $album,
    ) {}
}
```

## Validation

### Auto-Generated Rules
Types automatically infer validation rules:
- `string` → `['required', 'string']`
- `?string` → `['nullable', 'string']`
- `UserRole` (enum) → `['required', 'enum:admin,user']`

### Validation Attributes
```php
use Spatie\LaravelData\Attributes\Validation\*;

class UserData extends Data
{
    public function __construct(
        #[Max(255)]
        public string $name,

        #[Email, Unique('users', 'email')]
        public string $email,

        #[Min(18), Max(120)]
        public int $age,
    ) {}
}
```

### Route Parameter References
```php
class UpdateUserData extends Data
{
    public function __construct(
        #[Unique('users', 'email', ignore: new RouteParameterReference('user'))]
        public string $email,
    ) {}
}
```

### Authorization
```php
class CreatePostData extends Data
{
    public static function authorize(): bool
    {
        return auth()->user()?->can('create-posts') ?? false;
    }
}
```

## Transformation

### Casts (Input)
```php
#[WithCast(DateTimeInterfaceCast::class)]
public Carbon $start_date;
```

### Transformers (Output)
```php
#[WithTransformer(DateTimeInterfaceTransformer::class, format: 'Y-m-d H:i')]
public Carbon $start_date;
```

### Global Configuration
Configure in `config/data.php` for common types (DateTime, Enums).

## Lazy Properties

Use for expensive computations, optional relations, or large data:

```php
class PostResource extends Data
{
    public function __construct(
        public string $title,
        public Lazy|string $content,
        public Lazy|Collection $comments,
    ) {}
}
```

### Patterns
```php
// Eloquent relations
Lazy::whenLoaded('comments', $post, fn() => CommentData::collect($post->comments))

// Conditional
Lazy::when(fn() => auth()->user()?->isAdmin(), fn() => $user->email)

// Define allowed includes
public static function allowedRequestIncludes(): ?array
{
    return ['comments', 'author', 'tags'];
}
```

## Collections

Always type collections:
```php
/** @var Collection<int, SongData> */
public Collection $songs;
```

Preserve collection types:
```php
return PostResource::collect(
    Post::with('author')->paginate(),
    PaginatedDataCollection::class
);
```

## Eloquent Integration

### Model Casting
```php
protected $casts = [
    'settings' => UserSettingsData::class,
    'addresses' => DataCollection::class . ':' . AddressData::class,
];
```

### Auto-loading Relations
```php
#[LoadRelation]
public AuthorData $author;
```

**Warning:** Avoid circular `LoadRelation`. Use `Lazy` properties instead.

## Property Types

### Optional vs Nullable
```php
public ?string $middle_name;                 // Can be null
public string|Optional $middle_name;         // Can be absent from output
public string|Optional|null $middle_name;    // Both
```

### Computed Properties
```php
#[Computed]
public string $full_name;

public function __construct(
    public string $first_name,
    public string $last_name,
) {
    $this->full_name = "{$this->first_name} {$this->last_name}";
}
```

## Property Mapping

```php
// Class-level
#[MapName(SnakeCaseMapper::class)]
class UserData extends Data { }

// Property-level
#[MapInputName('external_id')]
public string $id;
```

## TypeScript Generation

```bash
composer require spatie/laravel-typescript-transformer
```

```php
/** @typescript */
class UserData extends Data
{
    public function __construct(
        public string $name,
        /** @var array<string> */
        public array $roles,
    ) {}
}
```

```bash
php artisan typescript:transform
```

## Common Patterns

### Form Request Replacement
```php
public function store(CreatePostData $data)
{
    $post = Post::create($data->toArray());
    return PostResource::from($post);
}
```

### API Resource
```php
class PostResource extends Data
{
    public static function fromModel(Post $post): self
    {
        return new self(
            $post->title,
            Lazy::whenLoaded('author', $post, fn() => AuthorData::from($post->author)),
        );
    }
}
```

### Partial Updates
```php
class UpdatePostData extends Data
{
    public function __construct(
        public Optional|string $title,
        public Optional|string $content,
    ) {}
}
```

## Performance

### Enable Caching
```php
// config/data.php
'structure_caching' => [
    'enabled' => true,
    'directories' => [app_path('Data')],
],
```

### Avoid N+1
```php
$posts = Post::with('author', 'comments')->get();
return PostResource::collect($posts);
```

## Common Pitfalls

**Circular Relations:** Use `Lazy` properties, not `LoadRelation`.

**Nested Validation Override:** Use `#[WithoutValidation]` attribute.

**Computed Properties in Payload:** Enable `ignore_exception_when_trying_to_set_computed_property_value` in config.

**Optional/Nullable Confusion:** `?string` includes null in output, `Optional` omits property.

## Code Organization

```
app/Data/
├── User/
│   ├── UserData.php
│   ├── CreateUserData.php
│   └── UserResource.php
├── Post/
│   └── PostResource.php
└── Shared/
    └── AddressData.php
```

Extract common structures into shared data objects.

## Testing

```php
test('creates and validates data', function () {
    $data = SongData::from(['title' => 'Test', 'plays' => 100]);
    expect($data->title)->toBe('Test');
});

test('lazy properties excluded by default', function () {
    $data = AlbumData::from($album);
    expect($data->toArray())->not->toHaveKey('songs');
});
```

## Contributing

Run tests: `composer test`
Format code: `composer format`
Follow TDD, update docs, ensure backward compatibility.

## Resources

- [Official Documentation](https://spatie.be/docs/laravel-data)
- [GitHub Repository](https://github.com/spatie/laravel-data)
- Laravel Data v4.x: Laravel 10.x/11.x, PHP 8.2+
