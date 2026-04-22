# DDD Tactical Patterns in Go

> Adapted from Eric Evans (2003), Vaughn Vernon (2013), and Effective Aggregate Design (Vernon).

## Building Blocks Overview

```
Aggregate Root (Entity)
├── Child Entity     ← owned, no standalone repo
├── Child Entity     ← owned, no standalone repo
├── Value Object     ← immutable, no ID
└── Domain Event     ← record of change (future)

Repository Port ← one per aggregate root
```

---

## Entity

An object with **identity** that persists through time. Two entities are equal if they have the same ID.

### Go Pattern

```go
// internal/domain/user.go

type User struct {
    ID            UserID       // typed ID — prevents mix-ups with other IDs at compile time
    PrepUserID    PrepUserID   // int64 typedef; use .IsSet() instead of comparing to 0
    DisplayName   string
    Email         string
    AvatarURL     string
    PreferredLang string
    IsGuest       bool
    DeviceID      string

    // Owned relationships
    LearningPreferences *LearningPreferences
    TopicIDs            []TopicID
}

// Business method — keeps logic in the entity
func (user *User) CanAccessPremiumFeature() bool {
    return !user.IsGuest && user.PrepUserID.IsSet()
}

// Validation as domain method
func (user *User) ValidateProfile() error {
    if user.DisplayName == "" {
        return ErrDisplayNameRequired
    }
    if user.PreferredLang != "" && !isValidLang(user.PreferredLang) {
        return ErrInvalidLanguage
    }
    return nil
}
```

### Key Rules

- **Typed ID required** (`FooID`, not raw `uuid.UUID`). See Typed ID section below.
- Equality by ID, not by field values
- Contains behavior (methods that enforce business rules)
- Can hold references to value objects and child entities

---

## Typed ID (Aggregate Identity)

Every aggregate root has a typed ID so the compiler catches mix-ups between `UserID`,
`CollectionID`, `VocabularyID`, etc.

### UUID-backed (most aggregates)

```go
// internal/domain/<feature>.go
type FeatureID struct {
    uuid.UUID  // embed → free String/MarshalJSON/UnmarshalJSON/comparability
}

func NewFeatureID() (FeatureID, error) {
    u, err := uuid.NewV7()
    if err != nil {
        return FeatureID{}, err
    }
    return FeatureID{UUID: u}, nil
}

func ParseFeatureID(s string) (FeatureID, error) {
    u, err := uuid.Parse(s)
    if err != nil {
        return FeatureID{}, ErrInvalidFeatureID
    }
    return FeatureID{UUID: u}, nil
}
```

### Int-backed (reference data or external IDs)

```go
type ProficiencyLevelID int16  // DB surrogate key
type PrepUserID         int64  // external system ID

// Add value-object methods for hidden invariants:
func (id PrepUserID) IsSet() bool { return id != 0 }  // 0 = not linked to Prep
```

### Usage Rules

- `New*` = generate fresh ID when creating an aggregate
- `Parse*` = from string at edges (HTTP handler, message consumers)
- Unwrap to raw UUID **only** at the sqlc boundary: `id.UUID`
- Map raw UUID back when building domain: `FeatureID{UUID: row.ID}`
- Never compare to sentinel values at call sites — expose `IsSet()` / `IsValid()` on the type

---

## Value Object

Defined by its **attributes**, not identity. Immutable. Two VOs are equal if all fields match.

### Go Patterns

**Type alias (simple):**

```go
// Proficiency system name
type ProficiencySystem string

const (
    HSK30 ProficiencySystem = "HSK3.0"
    HSK20 ProficiencySystem = "HSK2.0"
    TOCFL ProficiencySystem = "TOCFL"
)

func (sys ProficiencySystem) IsValid() bool {
    switch sys {
    case HSK30, HSK20, TOCFL:
        return true
    }
    return false
}
```

**Struct (composite):**

```go
// Value object — no ID, immutable, equality by fields
type PinyinSyllable struct {
    Initial string
    Final   string
    Tone    int
}

func NewPinyinSyllable(initial, final string, tone int) (PinyinSyllable, error) {
    if tone < 0 || tone > 5 {
        return PinyinSyllable{}, ErrInvalidTone
    }
    return PinyinSyllable{Initial: initial, Final: final, Tone: tone}, nil
}

func (ps PinyinSyllable) String() string {
    return fmt.Sprintf("%s%s%d", ps.Initial, ps.Final, ps.Tone)
}

func (ps PinyinSyllable) Equals(other PinyinSyllable) bool {
    return ps == other // struct comparison works for simple VOs
}
```

### When to Use Value Objects

| Scenario | Type |
|----------|------|
| HSK level number (1-9) | `type HSKLevel int16` with validation |
| Language code (en, vi, zh) | `type LangCode string` with validation |
| Sort direction (asc, desc) | `type SortDir string` with constants |
| Cursor token | `type Cursor string` with encode/decode |
| Money amount | Struct with `Amount int64` + `Currency string` |

### Value Objects vs Primitives

```go
// BAD: primitive obsession
type Vocabulary struct {
    Pinyin string  // any string? what format?
}

// BETTER: value object with validation
type Pinyin string

func NewPinyin(raw string) (Pinyin, error) {
    normalized := strings.TrimSpace(strings.ToLower(raw))
    if normalized == "" {
        return "", ErrEmptyPinyin
    }
    return Pinyin(normalized), nil
}
```

**Pragmatic note:** lu-backend uses typed IDs for every aggregate root and simple typedefs/methods for hidden invariants. Full-blown value object structs stay incremental — add them when a concept needs validation at the type level (like `PinyinSyllable`).

---

## Aggregate

A cluster of entities and value objects treated as a **consistency boundary**.

### Current Aggregates in lu-backend

| Aggregate Root | Children | Referenced By ID | Behavior |
|---------------|----------|-----------------|----------|
| `Collection` | `CollectionItem` | `User`, `Vocabulary` | **Rich** — `NewCollection()`, `Rename()`, `IsOwnedBy()`, error sentinels |
| `Vocabulary` | `Definition`, `Example` | `Character`, `Topic`, `VocabularyRelation` | **Anemic** — data struct only |
| `User` | — | `Topic` (via `user_topics`), `ProficiencyLevel` | **Factory-only** — `NewUser()`, `NewGuestUser()` |
| `Character` | `CharacterRadical` (VO) | `Vocabulary` | **Anemic** — data struct only |
| `Topic` | — (reference data) | `User`, `Vocabulary` | **Has behavior** — `Name(lang)` i18n resolution |
| `ProficiencyLevel` | `MemoryStats` (VO) | `Vocabulary`, `User` | **Anemic** — data struct only |

### Aggregate Rules

1. **One aggregate root** — single entry point for modifications
2. **Reference by typed ID** — `UserID`, not `*User`, not raw `uuid.UUID`
3. **One aggregate per transaction** — no cross-aggregate TX
4. **Root controls children** — external code never modifies children directly
5. **Small aggregates** — prefer 1-5 child entities

### Go Pattern (Actual Code from lu-backend)

```go
// internal/domain/collection.go — the project's exemplary rich aggregate

const CollectionNameMaxLength = 100

var (
    ErrCollectionNameEmpty   = errors.New("collection name is empty")
    ErrCollectionNameTooLong = errors.New("collection name exceeds max length")
)

type Collection struct {
    ID        CollectionID
    UserID    UserID           // typed cross-aggregate reference
    Name      string
    Source    string           // "manual" | "ocr" | "hsk"
    ImageURL  string
    Items     []CollectionItem // owned children
    CreatedAt time.Time
    UpdatedAt time.Time
}

type CollectionItem struct {
    VocabularyID uuid.UUID // polymorphic FK (system vocab OR user_vocabulary) — stays raw
    VocabType    string    // "system" | "user"
    DisplayOrder int
    AddedAt      time.Time
}

// Factory — validates + generates typed ID
func NewCollection(userID UserID, name string) (*Collection, error) {
    name, err := validateCollectionName(name)
    if err != nil { return nil, err }
    id, err := NewCollectionID()
    if err != nil { return nil, err }
    return &Collection{ID: id, UserID: userID, Name: name, Source: "manual"}, nil
}

// Ownership check — typed UserID prevents passing the wrong ID
func (collection *Collection) IsOwnedBy(userID UserID) bool {
    return collection.UserID == userID
}

// Business method — validates + applies (not just a setter)
func (collection *Collection) Rename(name string) error {
    name, err := validateCollectionName(name)
    if err != nil { return err }
    collection.Name = name
    return nil
}

// Private reusable validation
func validateCollectionName(name string) (string, error) {
    name = strings.TrimSpace(name)
    if name == "" { return "", ErrCollectionNameEmpty }
    if len([]rune(name)) > CollectionNameMaxLength { return "", ErrCollectionNameTooLong }
    return name, nil
}
```

### Aggregate Sizing Heuristics

| Metric | Healthy | Warning | Split |
|--------|---------|---------|-------|
| Child entities | 1-5 | 6-10 | >10 |
| Aggregate root methods | <15 | 15-25 | >25 |
| Concurrent modification conflicts | Rare | Occasional | Frequent |

### Questions to Ask

- Can parts be eventually consistent? → Separate aggregates
- Do all parts change together? → Same aggregate
- Are there independent lifecycles? → Separate aggregates
- Is one part read-heavy, other write-heavy? → Separate aggregates

---

## Repository

Provides collection-like access to aggregates. Interface in `application/port/outbound.go`, implementation in `adapter/repository/`.

### Rules

1. **One repository per aggregate root** — not per table
2. **Interface in application layer** — implementation in adapter layer
3. **Returns domain types** — maps from DB model internally
4. **Raw errors only** — never returns `AppError` (use case wraps)

### Go Pattern

```go
// application/port/outbound.go
type CollectionRepositoryPort interface {
    Create(ctx context.Context, collection *domain.Collection) (*domain.Collection, error)
    FindByID(ctx context.Context, id domain.CollectionID) (*domain.Collection, error)
    Update(ctx context.Context, collection *domain.Collection) (*domain.Collection, error)
    SoftDelete(ctx context.Context, id domain.CollectionID) error
    List(ctx context.Context, filter *dto.CollectionFilter) ([]*domain.Collection, error)
    ExistsByName(ctx context.Context, userID domain.UserID, name string, excludeID *domain.CollectionID) (bool, error)
    CountVocabulary(ctx context.Context, collectionID domain.CollectionID) (int, error)
}
```

### Repository vs Read Model

For complex queries that don't fit the aggregate pattern, add query-specific methods to the repo port or create a separate read model:

```go
// Aggregate-focused (write)
type CollectionRepositoryPort interface {
    Create(ctx context.Context, collection *domain.Collection) (*domain.Collection, error)
    FindByID(ctx context.Context, id domain.CollectionID) (*domain.Collection, error)
}

// Query-focused (read) — can join across aggregates
type CollectionVocabularyRepositoryPort interface {
    List(ctx context.Context, filter *dto.CollectionVocabularyFilter) ([]dto.CollectionVocabularyRow, error)
}
```

---

## Domain Events (Future)

Not yet implemented in lu-backend, but the pattern for when needed:

```go
// internal/domain/events.go

type DomainEvent interface {
    EventType() string
    OccurredAt() time.Time
}

type CollectionCreated struct {
    CollectionID CollectionID
    UserID       UserID
    Name         string
    Timestamp    time.Time
}

func (event CollectionCreated) EventType() string     { return "collection.created" }
func (event CollectionCreated) OccurredAt() time.Time { return event.Timestamp }
```

### Naming Convention

- Past tense: `CollectionCreated`, `VocabularyAdded`, `PreferencesUpdated`
- NOT imperative: `CreateCollection`, `AddVocabulary`

---

## Domain Service

Stateless operations that span multiple aggregates or don't belong to a single entity.

```go
// internal/domain/learning_service.go

// CalculateStudyOrder determines vocabulary study sequence based on
// user level, topic preferences, and spaced repetition intervals.
func CalculateStudyOrder(
    vocab []Vocabulary,
    userLevel ProficiencyLevelID,
    preferredTopics []TopicID,
) []Vocabulary {
    // Pure business logic, no I/O
    // Sort by relevance to user's level and topics
    // ...
    return sorted
}
```

### When to Use Domain Service

- Logic spans multiple aggregate types
- Logic is stateless (no entity state needed)
- Doesn't naturally fit as a method on one entity
- Examples: pricing calculation, study scheduling, difficulty scoring

---

## Domain Error Sentinels

```go
// internal/domain/errors.go

var (
    ErrCollectionNameRequired = errors.New("collection name is required")
    ErrCollectionNameTooLong  = errors.New("collection name exceeds max length")
    ErrDefinitionIncomplete   = errors.New("definition requires lang and meaning")
    ErrDefinitionDuplicate    = errors.New("duplicate definition for language")
    ErrInvalidTone            = errors.New("tone must be 0-5")
)
```

Use cases map these to AppError:

```go
err := collection.Rename(req.Name)
if errors.Is(err, domain.ErrCollectionNameRequired) {
    return apperr.ValidationFailed("collection.name_required")
}
```
