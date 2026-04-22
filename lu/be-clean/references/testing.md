# Testing Strategy — DDD + Hexagonal in Go

> "Test behavior, not implementation."

## Testing Pyramid

```
        ╱╲
       ╱  ╲        E2E Tests (few, slow, expensive)
      ╱────╲       API contract tests with real DB
     ╱      ╲
    ╱────────╲     Integration Tests (moderate)
   ╱          ╲    Repository + DB, handler + router
  ╱────────────╲
 ╱              ╲  Unit Tests (many, fast, cheap)
╱────────────────╲ Domain entities, use cases with mocked ports
```

---

## Unit Tests: Domain Layer

**Goal:** Test business logic in complete isolation. Zero infrastructure.

Domain entities have no external dependencies, so tests are pure and fast.

```go
// internal/domain/collection_test.go

func TestCollection_Rename(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr error
    }{
        {
            name:    "valid name",
            input:   "My Vocabulary",
            wantErr: nil,
        },
        {
            name:    "empty name",
            input:   "",
            wantErr: ErrCollectionNameRequired,
        },
        {
            name:    "whitespace only",
            input:   "   ",
            wantErr: ErrCollectionNameRequired,
        },
        {
            name:    "exceeds max length",
            input:   strings.Repeat("a", 101),
            wantErr: ErrCollectionNameTooLong,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            collection := &Collection{
                ID:   uuid.New(),
                Name: "Original",
            }
            err := collection.Rename(tt.input)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("Rename(%q) error = %v, want %v", tt.input, err, tt.wantErr)
            }
        })
    }
}
```

### What to Test in Domain

- Entity creation with validation
- Business rule enforcement (state transitions, invariants)
- Value object creation and equality
- Domain service calculations
- Edge cases and error sentinels

### What NOT to Test in Domain

- Struct field access (getters/setters are trivial)
- Constructor with no validation logic
- Third-party library behavior

---

## Unit Tests: Use Case Layer

**Goal:** Test orchestration logic with mocked ports.

Use cases depend on port interfaces. Create simple mock implementations for testing.

```go
// internal/application/usecase/collection_test.go

// Mock repository — implements CollectionRepositoryPort
type mockCollectionRepo struct {
    collections map[uuid.UUID]*domain.Collection
    findByIDErr error
}

func newMockCollectionRepo() *mockCollectionRepo {
    return &mockCollectionRepo{
        collections: make(map[uuid.UUID]*domain.Collection),
    }
}

func (mock *mockCollectionRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.Collection, error) {
    if mock.findByIDErr != nil {
        return nil, mock.findByIDErr
    }
    collection, ok := mock.collections[id]
    if !ok {
        return nil, nil // not found convention
    }
    return collection, nil
}

func (mock *mockCollectionRepo) Create(ctx context.Context, collection *domain.Collection) (*domain.Collection, error) {
    mock.collections[collection.ID] = collection
    return collection, nil
}

// ... implement other methods

func TestCollectionUseCase_CreateCollection(t *testing.T) {
    tests := []struct {
        name       string
        userID     uuid.UUID
        collName   string
        setupMock  func(*mockCollectionRepo)
        wantErr    bool
        wantErrMsg string
    }{
        {
            name:     "success",
            userID:   uuid.New(),
            collName: "HSK 1 Words",
            wantErr:  false,
        },
        {
            name:     "duplicate name",
            userID:   uuid.New(),
            collName: "Existing",
            setupMock: func(mock *mockCollectionRepo) {
                // ExistsByName returns true
            },
            wantErr:    true,
            wantErrMsg: "collection.name_exists",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := newMockCollectionRepo()
            if tt.setupMock != nil {
                tt.setupMock(repo)
            }

            useCase := NewCollectionUseCase(repo, /* other deps */)
            result, err := useCase.CreateCollection(context.Background(), tt.userID, tt.collName)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error, got nil")
                }
                // Check AppError message
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if result.Name != tt.collName {
                t.Errorf("name = %q, want %q", result.Name, tt.collName)
            }
        })
    }
}
```

### What to Test in Use Cases

- Happy path orchestration
- Error handling: not found → `apperr.NotFound`
- Error handling: DB error → `apperr.InternalServerError`
- Business rule enforcement (delegated to domain)
- Port interaction order (when order matters)

### Mock Strategy

- **No mock framework** — lu-backend convention. Hand-written mocks implementing port interfaces.
- Keep mocks minimal — only implement methods used by the test.
- Use `t.Fatal` for unexpected errors, `t.Error` for assertion failures.

---

## Integration Tests: Repository Layer

**Goal:** Verify SQL queries and DB mapping work correctly with real PostgreSQL.

```go
// internal/adapter/repository/vocabulary_test.go

func TestVocabularyRepository_FindByID(t *testing.T) {
    // Requires running PostgreSQL (docker-compose up)
    pool := setupTestDB(t)
    defer pool.Close()

    queries := db.New(pool)
    repo := NewVocabularyRepository(queries)

    // Seed test data
    vocabID := seedVocabulary(t, pool)

    t.Run("existing vocabulary", func(t *testing.T) {
        result, err := repo.FindByID(context.Background(), vocabID, "en")
        if err != nil {
            t.Fatalf("FindByID error: %v", err)
        }
        if result == nil {
            t.Fatal("expected vocabulary, got nil")
        }
        if result.ID != vocabID {
            t.Errorf("ID = %v, want %v", result.ID, vocabID)
        }
    })

    t.Run("non-existent vocabulary", func(t *testing.T) {
        result, err := repo.FindByID(context.Background(), uuid.New(), "en")
        if err != nil {
            t.Fatalf("FindByID error: %v", err)
        }
        if result != nil {
            t.Errorf("expected nil, got %v", result)
        }
    })
}
```

### What to Test in Repositories

- Query correctness (filters, joins, pagination)
- DB model → domain entity mapping
- Not found returns `(nil, nil)`
- Error propagation (raw errors)
- Edge cases: empty results, null fields, large datasets

---

## Integration Tests: Handler Layer

**Goal:** Verify HTTP binding, status codes, and response envelope.

```go
// internal/adapter/handler/vocabulary_test.go

func TestVocabularyHandler_GetVocabulary(t *testing.T) {
    gin.SetMode(gin.TestMode)

    t.Run("valid ID", func(t *testing.T) {
        mockUseCase := &mockVocabularyUseCase{
            getResult: &dto.VocabularyDetailResponse{
                ID:         uuid.New(),
                Simplified: "你好",
            },
        }

        handler := NewVocabularyHandler(mockUseCase)
        router := gin.New()
        router.GET("/vocabulary/:vocabulary_id", handler.GetVocabulary)

        req := httptest.NewRequest("GET", "/vocabulary/"+uuid.New().String(), nil)
        rec := httptest.NewRecorder()
        router.ServeHTTP(rec, req)

        if rec.Code != http.StatusOK {
            t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
        }

        var body map[string]any
        json.Unmarshal(rec.Body.Bytes(), &body)
        if body["success"] != true {
            t.Error("expected success = true")
        }
    })

    t.Run("invalid UUID", func(t *testing.T) {
        handler := NewVocabularyHandler(&mockVocabularyUseCase{})
        router := gin.New()
        router.GET("/vocabulary/:vocabulary_id", handler.GetVocabulary)

        req := httptest.NewRequest("GET", "/vocabulary/not-a-uuid", nil)
        rec := httptest.NewRecorder()
        router.ServeHTTP(rec, req)

        if rec.Code != http.StatusBadRequest {
            t.Errorf("status = %d, want %d", rec.Code, http.StatusBadRequest)
        }
    })
}
```

---

## Architecture Tests

Verify dependency rules are not violated.

```go
// internal/architecture_test.go

func TestDomainHasNoExternalImports(t *testing.T) {
    // Scan domain/ package imports
    pkgs, err := packages.Load(&packages.Config{
        Mode: packages.NeedImports,
    }, "./internal/domain/...")
    if err != nil {
        t.Fatal(err)
    }

    forbidden := []string{
        "gorm.io",
        "github.com/gin-gonic",
        "github.com/jackc/pgx",
        "lu-vocab-service/internal/application",
        "lu-vocab-service/internal/adapter",
        "lu-vocab-service/internal/shared",
        "lu-vocab-service/internal/infra",
    }

    for _, pkg := range pkgs {
        for imp := range pkg.Imports {
            for _, f := range forbidden {
                if strings.Contains(imp, f) {
                    t.Errorf("domain package %s imports forbidden %s", pkg.PkgPath, imp)
                }
            }
        }
    }
}
```

---

## Test Organization

```
internal/
├── domain/
│   ├── collection.go
│   └── collection_test.go          # Unit: domain logic
├── application/usecase/
│   ├── collection.go
│   └── collection_test.go          # Unit: orchestration with mocks
├── adapter/
│   ├── handler/
│   │   ├── collection.go
│   │   └── collection_test.go      # Integration: HTTP binding
│   └── repository/
│       ├── collection.go
│       └── collection_test.go      # Integration: DB queries
└── architecture_test.go            # Architecture: dependency rules
```

### Conventions

- **Colocated** — `*_test.go` next to implementation
- **Table-driven** — `tests := []struct{...}` with `t.Run()`
- **No mock framework** — hand-written mocks implementing ports
- **Race detector** — always run with `-race` flag
- **Test naming** — `Test{Type}_{Method}` or `Test{Type}_{Method}_{Scenario}`
