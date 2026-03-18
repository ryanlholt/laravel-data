---
name: laravel-data-development
description: "Develops applications using the Spatie Laravel Data package. Activates when creating data objects, implementing validation, transforming data to/from arrays, working with Eloquent casts, generating TypeScript definitions, implementing lazy properties, or when the user mentions data objects, DTOs, form requests, API resources, data validation, transformers, or laravel-data."
license: MIT
metadata:
  author: spatie
---
# Laravel Data Development

## When to Apply

Activate this skill when:

- Creating or modifying data objects
- Implementing data validation or transformation
- Working with API resources or form requests
- Converting between data formats (arrays, JSON, models)
- Implementing lazy properties or nested data objects
- Generating TypeScript definitions
- Using Eloquent data casts
- Working with data collections

## Documentation

Use the markdown files in the `laravel-data/docs` directory for Laravel Data patterns and documentation.

## Basic Usage

### Creating Data Objects

<!-- Basic Data Object -->
```php
use Spatie\LaravelData\Data;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
    ) {}
}
```

### Creating from Various Sources

<!-- Creating Data Objects -->
```php
// From array
SongData::from(['title' => 'Song Title', 'artist' => 'Artist Name']);

// From model
SongData::from(Song::find($id));

// From request (auto-validates)
public function store(SongData $data)
{
    Song::create($data->toArray());
}

// Optional creation (returns null if null)
SongData::optional(null);
```

### Magical Creation Methods

<!-- Custom From Methods -->
```php
class SongData extends Data
{
    public static function fromModel(Song $song): self
    {
        return new self(
            title: "{$song->title} ({$song->year})",
            artist: $song->artist
        );
    }

    public static function fromString(string $string): self
    {
        [$title, $artist] = explode('|', $string);
        return new self($title, $artist);
    }
}

// Automatically uses the appropriate from* method
SongData::from(Song::first());
SongData::from('Title|Artist');
```

## Validation

### Auto-Generated Rules

<!-- Validation Rules -->
```php
class SongData extends Data
{
    public function __construct(
        public string $title,        // ['required', 'string']
        public int $plays,          // ['required', 'integer']
        public ?string $album,      // ['nullable', 'string']
    ) {}
}
```

### Validation Attributes

<!-- Validation Attributes -->
```php
use Spatie\LaravelData\Attributes\Validation\*;

class SongData extends Data
{
    public function __construct(
        #[Max(255)]
        public string $title,

        #[Uuid]
        public string $id,

        #[Date, After('now')]
        public ?CarbonImmutable $release_date,

        #[Unique('songs', ignore: new RouteParameterReference('song'))]
        public string $slug,
    ) {}
}
```

### Manual Validation Rules

<!-- Manual Rules -->
```php
class SongData extends Data
{
    public static function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'artist' => ['required', 'string'],
        ];
    }

    public static function authorize(): bool
    {
        return Auth::user()->can('create-song');
    }
}
```

## Transformers & Casts

### Casts (Input Transformation)

<!-- Using Casts -->
```php
use Spatie\LaravelData\Attributes\WithCast;
use Spatie\LaravelData\Casts\DateTimeInterfaceCast;

class SongData extends Data
{
    public function __construct(
        public string $title,
        #[WithCast(DateTimeInterfaceCast::class)]
        public DateTime $released_at,
    ) {}
}
```

### Transformers (Output Transformation)

<!-- Using Transformers -->
```php
use Spatie\LaravelData\Attributes\WithTransformer;
use Spatie\LaravelData\Transformers\DateTimeInterfaceTransformer;

class SongData extends Data
{
    public function __construct(
        #[WithTransformer(DateTimeInterfaceTransformer::class, format: 'Y-m-d')]
        public Carbon $released_at,
    ) {}
}

// Transform to array/JSON
$data = SongData::from($song);
$data->toArray();
$data->toJson();
```

## Lazy Properties

### Basic Lazy Properties

<!-- Lazy Properties -->
```php
use Spatie\LaravelData\Lazy;

class AlbumData extends Data
{
    public function __construct(
        public string $title,
        public Lazy|Collection $songs,
    ) {}

    public static function fromModel(Album $album): self
    {
        return new self(
            $album->title,
            Lazy::create(fn() => SongData::collect($album->songs))
        );
    }
}

// Usage
AlbumData::from($album)->toArray(); // Excludes songs
AlbumData::from($album)->include('songs')->toArray(); // Includes songs
```

### Conditional Lazy Properties

<!-- Conditional Lazy -->
```php
class UserData extends Data
{
    public function __construct(
        public string $name,
        public Lazy|string $email,
        public Lazy|array $permissions,
    ) {}

    public static function fromModel(User $user): self
    {
        return new self(
            $user->name,
            Lazy::when(fn() => auth()->user()?->isAdmin(), fn() => $user->email),
            Lazy::whenLoaded('permissions', $user, fn() => $user->permissions->pluck('name'))
        );
    }
}
```

### Including Lazy Properties

<!-- Including Properties -->
```php
// Single property
$data->include('songs');

// Multiple properties
$data->include('songs', 'artist');

// Nested properties
$data->include('songs.title', 'songs.artist');
$data->include('songs.*');

// Via query string (requires allowedRequestIncludes)
class AlbumData extends Data
{
    public static function allowedRequestIncludes(): ?array
    {
        return ['songs', 'artist'];
    }
}
```

## Collections

### Creating Collections

<!-- Data Collections -->
```php
// Returns same collection type as input
SongData::collect($songs);

// Specify collection type
SongData::collect($songs, DataCollection::class);
SongData::collect(Song::paginate(), PaginatedDataCollection::class);

// Transform collection methods
$collection->include('artist');
$collection->except('lyrics');
$collection->filter(fn($song) => $song->plays > 1000);
```

### Nested Collections

<!-- Nested Collections -->
```php
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        /** @var Collection<int, SongData> */
        public Collection $songs,
    ) {}
}

// Auto-converts nested arrays to collections
AlbumData::from([
    'title' => 'Album Name',
    'songs' => [
        ['title' => 'Song 1', 'artist' => 'Artist 1'],
        ['title' => 'Song 2', 'artist' => 'Artist 2'],
    ]
]);
```

## Eloquent Integration

### Creating from Models

<!-- From Models -->
```php
// Basic
SongData::from(Song::find($id));

// Auto-load relations
use Spatie\LaravelData\Attributes\LoadRelation;

class ArtistData extends Data
{
    public function __construct(
        public string $name,
        #[LoadRelation]
        public array $songs,
    ) {}
}
```

### Eloquent Casting

<!-- Eloquent Casts -->
```php
use Illuminate\Database\Eloquent\Model;

class Song extends Model
{
    protected $casts = [
        'artist' => ArtistData::class,
        'songs' => DataCollection::class . ':' . SongData::class,
    ];
}

// Usage
Song::create([
    'artist' => new ArtistData(name: 'Rick Astley'),
]);

$song = Song::find($id);
$song->artist; // ArtistData instance
```

## Property Types

### Optional Properties

<!-- Optional Properties -->
```php
use Spatie\LaravelData\Optional;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public string|Optional $artist, // May not be in payload
    ) {}
}

// When 'artist' not provided, it's omitted (not null)
SongData::from(['title' => 'Song'])->toArray();
// ['title' => 'Song']
```

### Computed Properties

<!-- Computed Properties -->
```php
use Spatie\LaravelData\Attributes\Computed;

class PersonData extends Data
{
    #[Computed]
    public string $full_name;

    public function __construct(
        public string $first_name,
        public string $last_name,
    ) {
        $this->full_name = "{$this->first_name} {$this->last_name}";
    }
}
```

## Property Name Mapping

### Input/Output Mapping

<!-- Name Mapping -->
```php
use Spatie\LaravelData\Attributes\MapInputName;
use Spatie\LaravelData\Attributes\MapOutputName;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

// Single property
class SongData extends Data
{
    public function __construct(
        #[MapInputName('song_title')]
        public string $title,
    ) {}
}

// Class-level mapping
#[MapInputName(SnakeCaseMapper::class)]
#[MapOutputName(SnakeCaseMapper::class)]
class SongData extends Data
{
    public function __construct(
        public string $recordCompany, // Expects 'record_company'
    ) {}
}
```

## TypeScript Generation

### Generating TypeScript

<!-- TypeScript -->
```php
/** @typescript */
class SongData extends Data
{
    public function __construct(
        public string $title,
        public int $plays,
        public ?string $album,
        public Lazy|string $lyrics,
        /** @var array<string> */
        public array $tags,
    ) {}
}
```

```bash
# Generate TypeScript definitions
php artisan typescript:transform
```

## Common Patterns

### Form Request Replacement

<!-- Form Request Pattern -->
```php
class SongData extends Data
{
    public function __construct(
        #[Max(255)]
        public string $title,
        public string $artist,
    ) {}
}

public function store(SongData $data)
{
    Song::create($data->toArray());
    return $data;
}
```

### API Resource Replacement

<!-- API Resource Pattern -->
```php
class SongResource extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
        public Lazy|string $lyrics,
    ) {}

    public static function fromModel(Song $song): self
    {
        return new self(
            $song->title,
            $song->artist,
            Lazy::create(fn() => $song->lyrics)
        );
    }
}

public function show(Song $song)
{
    return SongResource::from($song);
}
```

### Partial Updates

<!-- Partial Updates -->
```php
class UpdateSongData extends Data
{
    public function __construct(
        public Optional|string $title,
        public Optional|string $artist,
        public Optional|string $album,
    ) {}
}

public function update(UpdateSongData $data, Song $song)
{
    $song->update($data->toArray());
}
```

### Nested Data Objects

<!-- Nested Objects -->
```php
class PostData extends Data
{
    public function __construct(
        public string $title,
        public AuthorData $author,
        /** @var array<CommentData> */
        public array $comments,
    ) {}
}

// Auto-converts nested arrays
PostData::from([
    'title' => 'Post Title',
    'author' => ['name' => 'John'],
    'comments' => [
        ['text' => 'Comment 1'],
        ['text' => 'Comment 2'],
    ]
]);
```

## Advanced Features

### Factories

<!-- Factories -->
```php
SongData::factory()
    ->withoutMagicalCreation()
    ->withoutValidation()
    ->withoutOptionalValues()
    ->alwaysInclude('artist')
    ->from($data);
```

### Wrapping

<!-- Wrapping -->
```php
class SongData extends Data
{
    public function defaultWrap(): string
    {
        return 'data';
    }
}

// Manual wrapping
$data->wrap('data');
$data->withoutWrapping();
```

## Verification

1. Test auto-generated validation rules
2. Verify transformations match expected output
3. Test lazy property inclusion/exclusion
4. Verify Eloquent casts work correctly
5. Check TypeScript generation output
6. Test nested data object creation

## Common Pitfalls

- Avoid circular relations with `LoadRelation` (use Lazy instead)
- Cannot override nested data object validation rules
- Optional vs nullable: Optional omits property, nullable includes as null
- Computed properties cannot be set from payload
- Nested DataCollections are wrapped by default
- Remember to use `Lazy::whenLoaded()` for Eloquent relations
