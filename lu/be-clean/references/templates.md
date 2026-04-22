# Go Templates — New Feature Scaffold

Copy-paste templates per layer. Replace `Feature` with your subdomain name.

## Domain Entity + Typed ID

```go
// internal/domain/<feature>.go
package domain

import (
    "errors"
    "strings"
    "time"

    "github.com/google/uuid"
)

var (
    ErrInvalidFeatureID   = errors.New("invalid feature id")
    ErrFeatureNameEmpty   = errors.New("feature name is empty")
    ErrFeatureNameTooLong = errors.New("feature name exceeds max length")
)

type FeatureID struct {
    uuid.UUID
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

type Feature struct {
    ID        FeatureID
    UserID    UserID          // reference other aggregate by typed ID
    Name      string
    Items     []FeatureItem
    CreatedAt time.Time
    UpdatedAt time.Time
}

type FeatureItem struct {
    ID      uuid.UUID // child entity — not a cross-aggregate reference
    Value   string
    AddedAt time.Time
}

func NewFeature(userID UserID, name string) (*Feature, error) {
    name = strings.TrimSpace(name)
    if name == "" {
        return nil, ErrFeatureNameEmpty
    }
    id, err := NewFeatureID()
    if err != nil {
        return nil, err
    }
    return &Feature{ID: id, UserID: userID, Name: name}, nil
}

func (feature *Feature) IsOwnedBy(userID UserID) bool {
    return feature.UserID == userID
}

func (feature *Feature) Rename(name string) error {
    name = strings.TrimSpace(name)
    if name == "" {
        return ErrFeatureNameEmpty
    }
    feature.Name = name
    return nil
}
```

## Ports

```go
// internal/application/port/inbound.go — add interface
type FeatureUseCasePort interface {
    Create(ctx context.Context, userID domain.UserID, req *dto.CreateFeatureRequest) (*dto.FeatureResponse, error)
    GetByID(ctx context.Context, userID domain.UserID, id domain.FeatureID) (*dto.FeatureResponse, error)
    List(ctx context.Context, userID domain.UserID, query *dto.ListFeatureQuery) ([]dto.FeatureListItem, int64, error)
}

// internal/application/port/outbound.go — add interface
type FeatureRepositoryPort interface {
    Create(ctx context.Context, feature *domain.Feature) (*domain.Feature, error)
    FindByID(ctx context.Context, id domain.FeatureID) (*domain.Feature, error)
    List(ctx context.Context, filter *dto.FeatureFilter) ([]*domain.Feature, int64, error)
    SoftDelete(ctx context.Context, id domain.FeatureID) error
}
```

## DTO

```go
// internal/application/dto/feature.go
type CreateFeatureRequest struct {
    Name string `json:"name" binding:"required,max=100"`
}

type FeatureResponse struct {
    ID        domain.FeatureID `json:"id"`          // typed ID serializes as UUID string
    Name      string           `json:"name"`
    CreatedAt time.Time        `json:"created_at"`
}

type ListFeatureQuery struct {
    Page     int    `form:"page,default=1"        binding:"omitempty,min=1"`
    PageSize int    `form:"page_size,default=10"  binding:"omitempty,min=1,max=100"`
    Sort     string `form:"sort,default=created_at" binding:"omitempty,oneof=created_at name"`
    Order    string `form:"order,default=desc"      binding:"omitempty,oneof=asc desc"`
}
```

## Use Case (command + query split)

For subdomains with both reads and writes, split across 3 files:
- `usecase/<feature>.go` — struct + constructor + shared helpers (e.g. `mapDomainError`)
- `usecase/<feature>_command.go` — mutations (Create/Update/Delete…)
- `usecase/<feature>_query.go` — reads (Get/List…) + filter builders + cursor types

```go
// internal/application/usecase/feature.go
type FeatureUseCase struct {
    featureRepo port.FeatureRepositoryPort
}

func NewFeatureUseCase(featureRepo port.FeatureRepositoryPort) port.FeatureUseCasePort {
    return &FeatureUseCase{featureRepo: featureRepo}
}

func mapFeatureDomainError(err error) *apperr.AppError {
    switch {
    case errors.Is(err, domain.ErrFeatureNameEmpty):
        return apperr.ValidationFailed("feature.name_required")
    case errors.Is(err, domain.ErrFeatureNameTooLong):
        return apperr.ValidationFailed("feature.name_too_long")
    default:
        return apperr.InternalServerError("common.internal_error", err)
    }
}
```

```go
// internal/application/usecase/feature_command.go
func (useCase *FeatureUseCase) Create(ctx context.Context, userID domain.UserID, req *dto.CreateFeatureRequest) (*dto.FeatureResponse, error) {
    feature, err := domain.NewFeature(userID, req.Name)
    if err != nil {
        return nil, mapFeatureDomainError(err)
    }

    created, err := useCase.featureRepo.Create(ctx, feature)
    if err != nil {
        return nil, apperr.InternalServerError("common.internal_error", err)
    }

    return mapFeatureToResponse(created), nil
}
```

```go
// internal/application/usecase/feature_query.go
func (useCase *FeatureUseCase) GetByID(ctx context.Context, userID domain.UserID, id domain.FeatureID) (*dto.FeatureResponse, error) {
    feature, err := useCase.featureRepo.FindByID(ctx, id)
    if err != nil {
        return nil, apperr.InternalServerError("common.internal_error", err)
    }
    if feature == nil || !feature.IsOwnedBy(userID) {
        return nil, apperr.NotFound("feature.not_found")
    }
    return mapFeatureToResponse(feature), nil
}
```

## Handler

```go
// internal/adapter/handler/feature.go
type FeatureHandler struct {
    featureUseCase port.FeatureUseCasePort
}

func NewFeatureHandler(featureUseCase port.FeatureUseCasePort) *FeatureHandler {
    return &FeatureHandler{featureUseCase: featureUseCase}
}

func (handler *FeatureHandler) Create(c *gin.Context) {
    userID := c.MustGet("user_id").(domain.UserID)

    var req dto.CreateFeatureRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.ValidationError(c, err)
        return
    }

    result, err := handler.featureUseCase.Create(c.Request.Context(), userID, &req)
    if err != nil {
        response.HandleError(c, err)
        return
    }

    response.Success(c, http.StatusCreated, result)
}

func (handler *FeatureHandler) GetByID(c *gin.Context) {
    userID := c.MustGet("user_id").(domain.UserID)

    id, ok := parseFeatureID(c)
    if !ok {
        return
    }

    result, err := handler.featureUseCase.GetByID(c.Request.Context(), userID, id)
    if err != nil {
        response.HandleError(c, err)
        return
    }

    response.Success(c, http.StatusOK, result)
}

// Domain factory owns the "what is a valid FeatureID" invariant;
// handler only extracts + translates error to HTTP.
func parseFeatureID(c *gin.Context) (domain.FeatureID, bool) {
    id, err := domain.ParseFeatureID(c.Param("feature_id"))
    if err != nil {
        response.NotFound(c, "feature.not_found")
        return domain.FeatureID{}, false
    }
    return id, true
}
```

`c.MustGet("user_id").(domain.UserID)` panics if the invariant is broken. Acceptable because
`AuthMiddleware` always sets the key before the handler runs, and `RecoveryMiddleware`
converts any panic to a 500 with sanitized response.

## Repository

Repos are I/O adapters — no business logic, no filtering, no defensive checks. Accept domain
types in / domain types out. Unwrap typed IDs to `uuid.UUID` at the sqlc boundary.

Transaction propagation uses a package-level `getQueries(ctx, repo.queries)` helper defined
in `internal/adapter/repository/tx_manager.go`. It returns `queries.WithTx(tx)` if a tx is
smuggled via context, otherwise the plain queries. Repos must always route through it.

```go
// internal/adapter/repository/feature.go
type FeatureRepository struct {
    queries *db.Queries
}

func NewFeatureRepository(queries *db.Queries) port.FeatureRepositoryPort {
    return &FeatureRepository{queries: queries}
}

func (repo *FeatureRepository) FindByID(ctx context.Context, id domain.FeatureID) (*domain.Feature, error) {
    row, err := getQueries(ctx, repo.queries).GetFeatureByID(ctx, id.UUID)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, nil
        }
        return nil, err
    }
    return toFeatureDomain(row), nil
}

func (repo *FeatureRepository) Create(ctx context.Context, feature *domain.Feature) (*domain.Feature, error) {
    row, err := getQueries(ctx, repo.queries).CreateFeature(ctx, db.CreateFeatureParams{
        ID:     feature.ID.UUID,
        UserID: feature.UserID.UUID,
        Name:   feature.Name,
    })
    if err != nil {
        return nil, err
    }
    return toFeatureDomain(row), nil
}

func toFeatureDomain(row db.Feature) *domain.Feature {
    return &domain.Feature{
        ID:     domain.FeatureID{UUID: row.ID},
        UserID: domain.UserID{UUID: row.UserID},
        Name:   row.Name,
    }
}
```

## Composition Root

```go
// internal/app.go — in NewApp()
featureRepo := repository.NewFeatureRepository(queries)
featureUseCase := usecase.NewFeatureUseCase(featureRepo)
featureHandler := handler.NewFeatureHandler(featureUseCase)

// in RegisterRoutes()
protected.POST("/features", app.featureHandler.Create)
protected.GET("/features/:feature_id", app.featureHandler.GetByID)
```

## Error Flow

```
Repository           Use Case              Handler            Client
   |                    |                     |                  |
   +- (nil, nil)  ---> apperr.NotFound() --> HandleError() --> 404
   +- (nil, err)  ---> apperr.ISE()      --> HandleError() --> 500
   +- (result, nil) -> return dto         --> Success()     --> 200
   |                    |                     |                  |
   |                    +- domain.ErrXxx  --> HandleError() --> 422
   |                    +- apperr.Conflict -> HandleError() --> 409
```
