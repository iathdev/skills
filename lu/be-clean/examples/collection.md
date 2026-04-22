# Example: Collection Aggregate — Full Layer Walkthrough (Rich Domain)

Real-world example from lu-backend showing the Collection aggregate — the project's **exemplary DDD implementation** — flowing through all layers.

---

## 1. Domain Layer — `internal/domain/collection.go`

```go
package domain

import (
    "errors"
    "strings"
    "time"
    "github.com/google/uuid"
)

const (
    CollectionNameMaxLength = 100
    CollectionSourceManual  = "manual"
    CollectionSourceOCR     = "ocr"
    CollectionSourceHSK     = "hsk"
    VocabTypeSystem         = "system"
    VocabTypeUser           = "user"
)

// Domain error sentinels — mapped to AppError in use case
var (
    ErrCollectionNameEmpty   = errors.New("collection name is empty")
    ErrCollectionNameTooLong = errors.New("collection name exceeds max length")
)

// Collection is the aggregate root for a user's word folder.
type Collection struct {
    ID        uuid.UUID
    UserID    uuid.UUID
    Name      string
    Source    string
    ImageURL  string
    Items     []CollectionItem  // owned children
    CreatedAt time.Time
    UpdatedAt time.Time
}

// CollectionItem is a child entity — one vocabulary entry inside the collection.
type CollectionItem struct {
    VocabularyID uuid.UUID
    VocabType    string // "system" | "user"
    DisplayOrder int
    AddedAt      time.Time
}

// NewCollection — factory with validation.
func NewCollection(userID uuid.UUID, name string) (*Collection, error) {
    name, err := validateCollectionName(name)
    if err != nil {
        return nil, err
    }

    id, err := uuid.NewV7()
    if err != nil {
        return nil, err
    }

    return &Collection{
        ID:     id,
        UserID: userID,
        Name:   name,
        Source: CollectionSourceManual,
    }, nil
}

// IsOwnedBy — ownership check (used by use case to prevent unauthorized access).
func (collection *Collection) IsOwnedBy(userID uuid.UUID) bool {
    return collection.UserID == userID
}

// Rename — validates and applies a new name.
func (collection *Collection) Rename(name string) error {
    name, err := validateCollectionName(name)
    if err != nil {
        return err
    }
    collection.Name = name
    return nil
}

func validateCollectionName(name string) (string, error) {
    name = strings.TrimSpace(name)
    if name == "" {
        return "", ErrCollectionNameEmpty
    }
    if len([]rune(name)) > CollectionNameMaxLength {
        return "", ErrCollectionNameTooLong
    }
    return name, nil
}
```

### What Makes This Rich (vs Anemic)

| Pattern | Present | Implementation |
|---------|---------|---------------|
| Factory constructor | `NewCollection()` | Validates name, generates UUID v7, sets default source |
| Domain error sentinels | `ErrCollectionNameEmpty`, `ErrCollectionNameTooLong` | `errors.New()` in domain, mapped in use case |
| Business method | `Rename()` | Validates + applies (not just setter) |
| Authorization check | `IsOwnedBy()` | Domain knows ownership rule |
| Constants | `CollectionSourceManual`, `VocabTypeSystem` | Business enums in domain |
| Private validation | `validateCollectionName()` | Reused by factory + Rename |

---

## 2. Ports — `internal/application/port/`

### Inbound (what app offers)

```go
// port/inbound.go
type CollectionUseCasePort interface {
    ListCollections(ctx context.Context, userID uuid.UUID, query *dto.ListCollectionsQuery) ([]dto.CollectionListItem, string, bool, error)
    CreateCollection(ctx context.Context, userID uuid.UUID, name string) (*dto.CollectionCreatedResponse, error)
    GetCollection(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID) (*dto.CollectionDetailResponse, error)
    UpdateCollection(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID, name *string) (*dto.CollectionUpdatedResponse, error)
    DeleteCollection(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID) error
    AddVocabularies(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID, req *dto.CollectionVocabularyIdsRequest) (*dto.CollectionVocabulariesAddedResponse, error)
    RemoveVocabularies(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID, req *dto.CollectionVocabularyIdsRequest) error
    ListCollectionVocabulary(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID, query *dto.ListCollectionVocabularyQuery) ([]dto.CollectionVocabularyItem, string, bool, error)
}
```

### Outbound (what app needs)

```go
// port/outbound.go
type CollectionRepositoryPort interface {
    Create(ctx context.Context, collection *domain.Collection) (*domain.Collection, error)
    FindByID(ctx context.Context, id uuid.UUID) (*domain.Collection, error)
    Update(ctx context.Context, collection *domain.Collection) (*domain.Collection, error)
    SoftDelete(ctx context.Context, id uuid.UUID) error
    List(ctx context.Context, filter *dto.CollectionFilter) ([]*domain.Collection, error)
    ExistsByName(ctx context.Context, userID uuid.UUID, name string, excludeID *uuid.UUID) (bool, error)
    BatchCountVocabulary(ctx context.Context, collectionIDs []uuid.UUID) (map[uuid.UUID]int, error)
}

type CollectionVocabularyRepositoryPort interface {
    BatchAdd(ctx context.Context, collectionID uuid.UUID, items []domain.CollectionItem) (added int, err error)
    BatchRemove(ctx context.Context, collectionID uuid.UUID, vocabIDs []uuid.UUID) error
    List(ctx context.Context, filter *dto.CollectionVocabularyFilter) ([]dto.CollectionVocabularyRow, error)
}
```

### Why Two Outbound Ports?

- `CollectionRepositoryPort` — for the Collection aggregate root
- `CollectionVocabularyRepositoryPort` — for the M2M join table (collection_vocabularies)

This is a pragmatic choice: the join table has its own query patterns (batch add, batch remove, paginated list with vocab detail) that don't fit naturally into the aggregate root repo.

---

## 3. Use Case — `internal/application/usecase/collection.go`

### Key Patterns Demonstrated

**1. Factory → Domain validation → Transaction → Repo:**

```go
func (useCase *CollectionUseCase) CreateCollection(ctx context.Context, userID uuid.UUID, name string) (*dto.CollectionCreatedResponse, error) {
    // Domain factory validates name
    collection, err := domain.NewCollection(userID, name)
    if err != nil {
        return nil, mapDomainError(err)  // domain sentinel → AppError
    }

    // Transaction: uniqueness check + create must be atomic
    var created *domain.Collection
    if err := useCase.txManager.Execute(ctx, func(ctx context.Context) error {
        exists, err := useCase.collectionRepo.ExistsByName(ctx, userID, collection.Name, nil)
        if err != nil {
            return apperr.InternalServerError("common.internal_error", err)
        }
        if exists {
            return apperr.Conflict("collection.name_exists")
        }

        created, err = useCase.collectionRepo.Create(ctx, collection)
        if err != nil {
            return apperr.InternalServerError("common.internal_error", err)
        }
        return nil
    }); err != nil {
        return nil, err
    }

    return &dto.CollectionCreatedResponse{...}, nil
}
```

**2. Domain method → Rename with transaction:**

```go
func (useCase *CollectionUseCase) UpdateCollection(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID, name *string) (*dto.CollectionUpdatedResponse, error) {
    collection, err := useCase.loadOwnedCollection(ctx, userID, collectionID)
    if err != nil {
        return nil, err
    }

    if name == nil {
        return &dto.CollectionUpdatedResponse{...}, nil // nothing to update
    }

    // Domain validates name
    if err := collection.Rename(*name); err != nil {
        return nil, mapDomainError(err)
    }

    // Transaction: uniqueness check + update must be atomic
    var updated *domain.Collection
    if err := useCase.txManager.Execute(ctx, func(ctx context.Context) error {
        exists, err := useCase.collectionRepo.ExistsByName(ctx, userID, collection.Name, &collectionID)
        if err != nil {
            return apperr.InternalServerError("common.internal_error", err)
        }
        if exists {
            return apperr.Conflict("collection.name_exists")
        }

        updated, err = useCase.collectionRepo.Update(ctx, collection)
        if err != nil {
            return apperr.InternalServerError("common.internal_error", err)
        }
        if updated == nil {
            return apperr.NotFound("collection.not_found")
        }
        return nil
    }); err != nil {
        return nil, err
    }

    return &dto.CollectionUpdatedResponse{...}, nil
}
```

**3. Ownership check via domain method:**

```go
// loadOwnedCollection — reusable helper. Returns 404 for both not-found and not-owned.
func (useCase *CollectionUseCase) loadOwnedCollection(ctx context.Context, userID uuid.UUID, collectionID uuid.UUID) (*domain.Collection, error) {
    collection, err := useCase.collectionRepo.FindByID(ctx, collectionID)
    if err != nil {
        return nil, apperr.InternalServerError("common.internal_error", err)
    }
    if collection == nil || !collection.IsOwnedBy(userID) {
        return nil, apperr.NotFound("collection.not_found")
    }
    return collection, nil
}
```

**4. Domain error → AppError mapping:**

```go
func mapDomainError(err error) *apperr.AppError {
    switch {
    case errors.Is(err, domain.ErrCollectionNameEmpty):
        return apperr.ValidationFailed("collection.name_required")
    case errors.Is(err, domain.ErrCollectionNameTooLong):
        return apperr.ValidationFailed("collection.name_too_long")
    default:
        return apperr.InternalServerError("common.internal_error", err)
    }
}
```

---

## 4. Handler — `internal/adapter/handler/collection.go`

Thin adapter: bind → call → respond.

```go
func (handler *CollectionHandler) CreateCollection(c *gin.Context) {
    userID := c.MustGet("user_id").(domain.UserID)

    name := strings.TrimSpace(c.PostForm("name"))
    if name == "" {
        response.ValidationError(c, nil)
        return
    }

    res, err := handler.useCase.CreateCollection(c.Request.Context(), userID, name)
    if err != nil {
        response.HandleError(c, err)
        return
    }

    response.Success(c, http.StatusCreated, res)
}

func (handler *CollectionHandler) UpdateCollection(c *gin.Context) {
    userID := c.MustGet("user_id").(domain.UserID)

    collectionID, ok := parseCollectionID(c)
    if !ok { return }

    var name *string
    if nameVal := c.PostForm("name"); nameVal != "" {
        name = &nameVal
    }

    res, err := handler.useCase.UpdateCollection(c.Request.Context(), userID, collectionID, name)
    if err != nil {
        response.HandleError(c, err)
        return
    }

    response.Success(c, http.StatusOK, res)
}

func parseCollectionID(c *gin.Context) (domain.CollectionID, bool) {
    id, err := domain.ParseCollectionID(c.Param("collection_id"))
    if err != nil {
        response.NotFound(c, "collection.not_found")
        return domain.CollectionID{}, false
    }
    return id, true
}
```

### Handler Pattern Summary

Every handler method follows the same 4 steps:
1. `c.MustGet("user_id").(domain.UserID)` / `parseCollectionID(c)` — extract context + path values
2. `c.ShouldBindJSON(&req)` or `c.PostForm()` — bind input
3. `handler.useCase.Xxx(ctx, ...)` — single use case call
4. `response.Success()` or `response.HandleError()` — respond

---

## 5. Full Request Flow

```
POST /api/v1/collections  (multipart/form-data: name="HSK 1 Words")

 1. Gin Router → CollectionHandler.CreateCollection
 2. Handler extracts userID from auth middleware context
 3. Handler reads name from form data
 4. Handler calls useCase.CreateCollection(ctx, userID, "HSK 1 Words")
 5. Use case calls domain.NewCollection(userID, "HSK 1 Words")
    → domain validates name (trim, empty check, length check)
    → domain generates UUID v7
    → domain sets Source = "manual"
 6. Use case opens transaction via txManager.Execute
 7. Use case calls collectionRepo.ExistsByName(ctx, userID, "HSK 1 Words", nil)
    → checks uniqueness per user
 8. Use case calls collectionRepo.Create(ctx, collection)
    → repo maps domain.Collection → sqlc params → INSERT
 9. Transaction commits
10. Use case builds dto.CollectionCreatedResponse
 9. Handler calls response.Success(c, 201, result)
10. Client receives: { success: true, data: { id, name, ... }, meta: { request_id, timestamp } }
```

---

## Contrast: Vocabulary Aggregate (Anemic)

For comparison, `internal/domain/vocabulary.go` is currently **anemic** — data-only, no methods:

```go
type Vocabulary struct {
    ID          uuid.UUID
    Simplified  string
    Traditional string
    Pinyin      string
    LevelIDs    []int16
    Definitions []Definition  // owned child
    Examples    []Example     // owned child
}
```

All orchestration lives in `usecase/vocabulary.go`. When business rules grow (e.g. `AddDefinition` with duplicate detection, `ValidatePinyin`), follow Collection's pattern:

1. Add domain error sentinels to `domain/vocabulary.go`
2. Add methods on `Vocabulary` struct
3. Add `mapDomainError()` in the use case
4. Keep use case as orchestrator, not business logic owner
