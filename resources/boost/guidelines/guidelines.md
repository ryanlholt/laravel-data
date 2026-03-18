# Laravel Data Development Guidelines

## Core Principles
- **Type Safety:** Types auto-generate validation rules
- **Single Source of Truth:** One data object for validation, API resources, and TypeScript
- **Explicit:** Use attributes and docblocks for clarity

## Data Object Design
- Focused on single concern, suffix with `Data`
- Use nested data objects for complex structures
- Prefer immutability

## Validation
Types infer rules: `string` → `['required', 'string']`, `?string` → `['nullable', 'string']`

Add attributes for complex rules:
```php
#[Max(255), Email, Unique('users', 'email')]
public string $email;

#[Unique('users', 'email', ignore: new RouteParameterReference('user'))]  // Updates
```

Add authorization when needed:
```php
public static function authorize(): bool { return auth()->user()?->can('create-posts') ?? false; }
```

## Transformation
```php
#[WithCast(DateTimeInterfaceCast::class)]           // Input
#[WithTransformer(DateTimeInterfaceTransformer::class, format: 'Y-m-d H:i')]  // Output
```

Configure globally in `config/data.php` for common types.

## Lazy Properties
Use for expensive computations, optional relations, large data:
```php
public Lazy|string $content;
public Lazy|Collection $comments;

Lazy::whenLoaded('comments', $post, fn() => CommentData::collect($post->comments))
Lazy::when(fn() => auth()->user()?->isAdmin(), fn() => $user->email)

public static function allowedRequestIncludes(): ?array { return ['comments', 'author']; }
```

## Collections
Type collections: `/** @var Collection<int, SongData> */ public Collection $songs;`

Preserve types: `PostResource::collect(Post::paginate(), PaginatedDataCollection::class)`

## Eloquent Integration
```php
protected $casts = [
    'settings' => UserSettingsData::class,
    'addresses' => DataCollection::class . ':' . AddressData::class,
];

#[LoadRelation] public AuthorData $author;  // Warning: Avoid circular relations
```

## Property Types
```php
public ?string $name;                    // Nullable
public string|Optional $name;            // Absent from output if not provided
public string|Optional|null $name;       // Both

#[Computed] public string $full_name;    // Derived values (calculated once)
```

## Property Mapping
```php
#[MapName(SnakeCaseMapper::class)]       // Class-level
#[MapInputName('external_id')]           // Property-level
```

## TypeScript Generation
```php
/** @typescript */
class UserData extends Data { /** @var array<string> */ public array $roles; }
```
Run: `php artisan typescript:transform`

## Common Patterns
```php
// Form requests
public function store(CreatePostData $data) { return Post::create($data->toArray()); }

// API resources
public static function fromModel(Post $post): self {
    return new self($post->title, Lazy::whenLoaded('author', $post, fn() => AuthorData::from($post->author)));
}

// Partial updates
public Optional|string $title;  // PATCH endpoints
```

## Performance
Enable caching in `config/data.php`: `'structure_caching' => ['enabled' => true]`

Avoid N+1: `Post::with('author', 'comments')->get()`

## Common Pitfalls
- **Circular relations:** Use `Lazy`, not `LoadRelation`
- **Nested validation:** Use `#[WithoutValidation]` to override
- **Optional vs Nullable:** `?string` outputs null, `Optional` omits property

## Code Organization
Organize by feature: `app/Data/User/`, `app/Data/Post/`, `app/Data/Shared/`
