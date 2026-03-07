# Laravel Data Package Development Guidelines

## Overview

The Laravel Data package enables the creation of rich data objects that unify form requests, API resources, and TypeScript definitions into a single, typed PHP object. This guide provides best practices for developing with and contributing to the laravel-data package.

## Core Principles

### 1. Type Safety First

Always leverage PHP's type system to automatically generate validation rules and ensure data consistency:

```php
// Good: Types infer validation rules
class SongData extends Data
{
    public function __construct(
        public string $title,      // Automatically: ['required', 'string']
        public int $plays,         // Automatically: ['required', 'integer']
        public ?string $album,     // Automatically: ['nullable', 'string']
    ) {}
}

// Avoid: Manual rules when types suffice
class SongData extends Data
{
    public function __construct(
        public $title, // No type = no automatic validation
    ) {}
}
```

### 2. Single Source of Truth

Define your data structure once and use it everywhere:

- Form validation
- API transformation
- TypeScript definitions
- Eloquent casts

Avoid duplicating property definitions across resources, requests, and DTOs.

### 3. Explicit Over Implicit

Use attributes and docblocks to make intentions clear:

```php
// Good: Clear intent
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        /** @var Collection<int, SongData> */
        public Collection $songs,
    ) {}
}

// Avoid: Ambiguous types
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        public $songs, // What type? What structure?
    ) {}
}
```

## Data Object Design

### Structure Guidelines

**Keep data objects focused:** Each data object should represent a single, cohesive concept.

```php
// Good: Focused on song data
class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
        public int $duration,
    ) {}
}

// Avoid: Mixed responsibilities
class SongAndAlbumAndArtistData extends Data
{
    // Too many concerns in one object
}
```

**Use nested data objects for complex structures:**

```php
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        public ArtistData $artist,
        /** @var array<SongData> */
        public array $songs,
    ) {}
}
```

**Prefer immutability:** Data objects should represent immutable data snapshots.

### Naming Conventions

- Suffix data classes with `Data`: `UserData`, `PostData`, `CommentData`
- Use descriptive names that indicate purpose: `CreatePostData`, `UpdatePostData`, `PostResource`
- For API resources, consider `*Resource` suffix: `PostResource`, `UserResource`

## Validation Best Practices

### Leverage Auto-Generated Rules

Let the type system do the heavy lifting:

```php
class UserData extends Data
{
    public function __construct(
        public string $name,           // ['required', 'string']
        public string $email,          // ['required', 'string']
        public int $age,              // ['required', 'integer']
        public ?string $phone,        // ['nullable', 'string']
        public UserRole $role,        // ['required', 'enum:admin,user']
    ) {}
}
```

### Add Validation Attributes for Complex Rules

Use attributes for rules that can't be inferred from types:

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

### Handle Route Parameter References

For update operations, use `RouteParameterReference` to ignore the current record:

```php
class UpdateUserData extends Data
{
    public function __construct(
        #[Unique('users', 'email', ignore: new RouteParameterReference('user'))]
        public string $email,
    ) {}
}
```

### Authorization in Data Objects

Add authorization checks when data creation requires permissions:

```php
class CreatePostData extends Data
{
    public static function authorize(): bool
    {
        return auth()->user()?->can('create-posts') ?? false;
    }
}
```

## Transformation Patterns

### Casts for Input

Use casts to transform incoming data into complex types:

```php
use Spatie\LaravelData\Attributes\WithCast;
use Spatie\LaravelData\Casts\DateTimeInterfaceCast;

class EventData extends Data
{
    public function __construct(
        public string $name,
        #[WithCast(DateTimeInterfaceCast::class)]
        public Carbon $start_date,
    ) {}
}
```

### Transformers for Output

Use transformers to control output format:

```php
use Spatie\LaravelData\Attributes\WithTransformer;
use Spatie\LaravelData\Transformers\DateTimeInterfaceTransformer;

class EventData extends Data
{
    public function __construct(
        #[WithTransformer(DateTimeInterfaceTransformer::class, format: 'Y-m-d H:i')]
        public Carbon $start_date,
    ) {}
}
```

### Global vs Local Configuration

Configure common casts/transformers globally in `config/data.php`:

```php
'casts' => [
    DateTimeInterface::class => DateTimeInterfaceCast::class,
    BackedEnum::class => EnumCast::class,
],
'transformers' => [
    DateTimeInterface::class => DateTimeInterfaceTransformer::class,
    BackedEnum::class => EnumTransformer::class,
],
```

## Lazy Properties

### When to Use Lazy Properties

Use lazy properties for:
- Expensive computations
- Relations that aren't always needed
- Large data that should be opt-in
- Different access levels (public vs admin)

```php
class PostResource extends Data
{
    public function __construct(
        public string $title,
        public string $excerpt,
        public Lazy|string $full_content,        // Large text
        public Lazy|Collection $comments,        // Expensive relation
        public Lazy|array $admin_metadata,       // Sensitive data
    ) {}
}
```

### Lazy Property Patterns

**Use `Lazy::whenLoaded()` for Eloquent relations:**

```php
public static function fromModel(Post $post): self
{
    return new self(
        $post->title,
        $post->excerpt,
        Lazy::whenLoaded('comments', $post, fn() => CommentData::collect($post->comments)),
    );
}
```

**Use `Lazy::when()` for conditional data:**

```php
public static function fromModel(User $user): self
{
    return new self(
        $user->name,
        Lazy::when(fn() => auth()->user()?->isAdmin(), fn() => $user->email),
    );
}
```

**Define allowed includes for API endpoints:**

```php
class PostResource extends Data
{
    public static function allowedRequestIncludes(): ?array
    {
        return ['comments', 'author', 'tags'];
    }
}
```

## Collections

### Creating Collections

Always specify collection types for nested arrays:

```php
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        /** @var Collection<int, SongData> */
        public Collection $songs,
    ) {}
}
```

### Collection Transformation

Preserve collection types when transforming:

```php
// Returns paginated collection
public function index()
{
    return PostResource::collect(
        Post::with('author')->paginate(),
        PaginatedDataCollection::class
    );
}
```

## Eloquent Integration

### Model Casting

Use data objects for complex model attributes:

```php
class User extends Model
{
    protected $casts = [
        'settings' => UserSettingsData::class,
        'addresses' => DataCollection::class . ':' . AddressData::class,
    ];
}
```

### Loading Relations Efficiently

Use `LoadRelation` attribute to automatically eager-load relations:

```php
class PostResource extends Data
{
    public function __construct(
        public string $title,
        #[LoadRelation]
        public AuthorData $author,
    ) {}
}
```

**Warning:** Avoid circular references with `LoadRelation`. Use lazy properties instead:

```php
// Good
class AuthorData extends Data
{
    public function __construct(
        public string $name,
        public Lazy|array $posts, // Prevent circular loading
    ) {}
}

// Avoid
class AuthorData extends Data
{
    public function __construct(
        public string $name,
        #[LoadRelation]
        public array $posts, // May cause circular loading
    ) {}
}
```

## Property Types

### Optional vs Nullable

Understand the difference:

```php
// Nullable: Can be null in payload
public ?string $middle_name; // ['nullable', 'string']
UserData::from(['middle_name' => null]); // Valid

// Optional: May not be in payload
public string|Optional $middle_name;
UserData::from([]); // Valid - middle_name omitted from output

// Combined
public string|Optional|null $middle_name; // Can be absent OR null
```

### Computed Properties

Use computed properties for derived values:

```php
use Spatie\LaravelData\Attributes\Computed;

class PersonData extends Data
{
    #[Computed]
    public string $full_name;

    #[Computed]
    public int $age;

    public function __construct(
        public string $first_name,
        public string $last_name,
        public Carbon $birth_date,
    ) {
        $this->full_name = "{$this->first_name} {$this->last_name}";
        $this->age = $this->birth_date->age;
    }
}
```

**Note:** Computed properties are NOT reactive. They're calculated once during construction.

## Property Name Mapping

### Input/Output Mapping

Map between different naming conventions:

```php
use Spatie\LaravelData\Attributes\MapName;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

// Class-level mapping
#[MapName(SnakeCaseMapper::class)]
class UserData extends Data
{
    public function __construct(
        public string $firstName,      // Maps to/from 'first_name'
        public string $lastName,       // Maps to/from 'last_name'
    ) {}
}

// Property-level mapping
use Spatie\LaravelData\Attributes\MapInputName;

class ApiData extends Data
{
    public function __construct(
        #[MapInputName('external_id')]
        public string $id,
    ) {}
}
```

### Global Mapping

Configure global mapping strategy in `config/data.php`:

```php
'name_mapping_strategy' => [
    'input' => SnakeCaseMapper::class,
    'output' => SnakeCaseMapper::class,
],
```

## TypeScript Generation

### Enabling TypeScript

Install and configure the TypeScript transformer:

```bash
composer require spatie/laravel-typescript-transformer
php artisan vendor:publish --tag=typescript-transformer-config
```

Add to `config/typescript-transformer.php`:

```php
'transformers' => [
    Spatie\LaravelData\Support\Transformation\DataTypeScriptTransformer::class,
],
```

### Marking Data for TypeScript

Use `@typescript` docblock or `#[TypeScript]` attribute:

```php
/** @typescript */
class UserData extends Data
{
    public function __construct(
        public string $name,
        public string $email,
        /** @var array<string> */
        public array $roles,
    ) {}
}
```

Generate definitions:

```bash
php artisan typescript:transform
```

## Common Patterns

### Form Request Replacement

Replace form requests with validated data objects:

```php
// Controller
public function store(CreatePostData $data)
{
    $post = Post::create($data->toArray());
    return PostResource::from($post);
}

// Data object
class CreatePostData extends Data
{
    public function __construct(
        #[Max(255)]
        public string $title,

        public string $content,

        #[Exists('categories', 'id')]
        public int $category_id,
    ) {}
}
```

### API Resource Pattern

Use data objects as API resources:

```php
class PostResource extends Data
{
    public function __construct(
        public int $id,
        public string $title,
        public string $excerpt,
        public Lazy|string $content,
        public Lazy|AuthorData $author,
        public Carbon $created_at,
    ) {}

    public static function fromModel(Post $post): self
    {
        return new self(
            $post->id,
            $post->title,
            $post->excerpt,
            Lazy::create(fn() => $post->content),
            Lazy::whenLoaded('author', $post, fn() => AuthorData::from($post->author)),
            $post->created_at,
        );
    }
}

// Controller
public function show(Post $post)
{
    return PostResource::from($post->load('author'));
}
```

### Partial Update Pattern

Use Optional for PATCH endpoints:

```php
class UpdatePostData extends Data
{
    public function __construct(
        public Optional|string $title,
        public Optional|string $content,
        public Optional|int $category_id,
    ) {}
}

// Controller
public function update(UpdatePostData $data, Post $post)
{
    $post->update($data->toArray()); // Only updates provided fields
    return PostResource::from($post);
}
```

## Testing

### Testing Data Object Creation

```php
test('creates song data from array', function () {
    $data = SongData::from([
        'title' => 'Song Title',
        'artist' => 'Artist Name',
    ]);

    expect($data->title)->toBe('Song Title');
    expect($data->artist)->toBe('Artist Name');
});
```

### Testing Validation

```php
test('validates song data', function () {
    expect(fn() => SongData::from([
        'title' => '', // Empty title should fail
        'artist' => 'Artist',
    ]))->toThrow(ValidationException::class);
});

test('validation rules include required fields', function () {
    $rules = SongData::getValidationRules([]);

    expect($rules)->toHaveKey('title');
    expect($rules['title'])->toContain('required');
});
```

### Testing Transformations

```php
test('transforms song data to array', function () {
    $song = Song::factory()->create();
    $data = SongData::from($song);

    $array = $data->toArray();

    expect($array)->toHaveKeys(['title', 'artist']);
    expect($array['title'])->toBe($song->title);
});
```

### Testing Lazy Properties

```php
test('excludes lazy properties by default', function () {
    $album = Album::factory()->create();
    $data = AlbumData::from($album);

    $array = $data->toArray();

    expect($array)->not->toHaveKey('songs');
});

test('includes lazy properties when requested', function () {
    $album = Album::factory()->create();
    $data = AlbumData::from($album)->include('songs');

    $array = $data->toArray();

    expect($array)->toHaveKey('songs');
});
```

## Performance Optimization

### Enable Structure Caching

In `config/data.php`, enable caching for production:

```php
'structure_caching' => [
    'enabled' => true,
    'directories' => [
        app_path('Data'),
    ],
    'cache' => [
        'store' => env('DATA_STRUCTURE_CACHE_STORE', 'file'),
        'duration' => null, // Forever
    ],
],
```

Clear cache after deployment:

```bash
php artisan cache:clear
```

### Avoid N+1 Queries

Use `Lazy::whenLoaded()` and eager loading:

```php
// Good
public function index()
{
    $posts = Post::with('author', 'comments')->get();
    return PostResource::collect($posts);
}

// Avoid
public function index()
{
    $posts = Post::all(); // N+1 when accessing author/comments
    return PostResource::collect($posts);
}
```

### Limit Transformation Depth

Set maximum depth to prevent infinite recursion:

```php
// config/data.php
'max_transformation_depth' => 20,
```

## Common Pitfalls

### Circular Relations

**Problem:** Circular `LoadRelation` attributes cause infinite loops.

```php
// Avoid
class AuthorData extends Data {
    #[LoadRelation]
    public array $posts; // Each post loads author → infinite loop
}
class PostData extends Data {
    #[LoadRelation]
    public AuthorData $author; // Loads posts → infinite loop
}
```

**Solution:** Use lazy properties:

```php
class AuthorData extends Data {
    public Lazy|array $posts; // Only loads when explicitly included
}
```

### Validation with Nested Objects

**Problem:** Cannot override nested data object validation rules.

```php
// Won't work
class AlbumData extends Data {
    public function __construct(
        public SongData $song,
    ) {}

    public static function rules(): array {
        return [
            'song.title' => ['nullable'], // Ignored! SongData rules take precedence
        ];
    }
}
```

**Solution:** Use `WithoutValidation` attribute:

```php
use Spatie\LaravelData\Attributes\WithoutValidation;

class AlbumData extends Data {
    public function __construct(
        #[WithoutValidation]
        public SongData $song,
    ) {}
}
```

### Computed Properties in Payload

**Problem:** Trying to set computed properties throws exception.

```php
#[Computed]
public string $full_name;

// Throws CannotSetComputedValue
PersonData::from(['full_name' => 'Test']);
```

**Solution:** Enable silent ignore in `config/data.php`:

```php
'features' => [
    'ignore_exception_when_trying_to_set_computed_property_value' => true,
],
```

### Optional vs Nullable Confusion

**Problem:** Using wrong type for the use case.

```php
// Wrong: Wants to omit property but uses nullable
public ?string $middle_name; // Output: {'middle_name': null}

// Correct: Use Optional to omit
public string|Optional $middle_name; // Output: {}
```

## Code Organization

### Directory Structure

Organize data objects by feature or domain:

```
app/
├── Data/
│   ├── User/
│   │   ├── UserData.php
│   │   ├── CreateUserData.php
│   │   ├── UpdateUserData.php
│   │   └── UserResource.php
│   ├── Post/
│   │   ├── PostData.php
│   │   ├── PostResource.php
│   │   └── CommentData.php
│   └── Shared/
│       ├── AddressData.php
│       └── PaginationData.php
```

### Reusable Data Objects

Extract common structures:

```php
// Shared address data
class AddressData extends Data
{
    public function __construct(
        public string $street,
        public string $city,
        public string $zip,
    ) {}
}

// Use in multiple contexts
class UserData extends Data
{
    public function __construct(
        public string $name,
        public AddressData $address,
    ) {}
}

class CompanyData extends Data
{
    public function __construct(
        public string $name,
        public AddressData $headquarters,
    ) {}
}
```

## Contributing to Laravel Data

### Running Tests

```bash
composer test
```

### Code Style

Follow PSR-12 and Laravel conventions:

```bash
composer format
```

### Adding Features

1. Add tests first (TDD approach)
2. Implement feature
3. Update documentation
4. Ensure backward compatibility
5. Update CHANGELOG.md

### Documentation

- Update relevant markdown files in `docs/`
- Add code examples
- Explain use cases and edge cases
- Update README if adding major features

## Resources

- [Official Documentation](https://spatie.be/docs/laravel-data)
- [GitHub Repository](https://github.com/spatie/laravel-data)
- [Video Introduction](https://www.youtube.com/watch?v=CrO_7Df1cBc)
- [Package Source Code](https://github.com/spatie/laravel-data/tree/main/src)

## Version Compatibility

- Laravel Data v4.x: Laravel 10.x and 11.x
- PHP 8.2+ required
- Check `composer.json` for specific version requirements

When upgrading, review UPGRADE.md for breaking changes and migration paths.
