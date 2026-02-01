# Unit Testing
```go
package domain

type Counter struct {
	Id    int `gorm:"primaryKey;autoIncrement"`
	Count int
}

```

```go
package repositories

import (
	"context"
	"demo/domain"

	"gorm.io/gorm"
)

type RepoContracts interface {
	AddCounter(ctx context.Context) (err error)
	GetCounter(ctx context.Context, id int) (domain.Counter, error)
	ListCounter(ctx context.Context) ([]domain.Counter, error)
}

type repos struct {
	db *gorm.DB
}

func NewRepository(db *gorm.DB) *repos {
	return &repos{
		db: db,
	}
}

func (r *repos) AddCounter(ctx context.Context) (err error) {
	data := domain.Counter{Count: 1}
	err = r.db.WithContext(ctx).Create(&data).Error
	return
}

func (r *repos) GetCounter(ctx context.Context, id int) (domain.Counter, error) {
	count := domain.Counter{}
	err := r.db.WithContext(ctx).First(&count, id).Error
	if err != nil {
		return count, err
	}
	return count, nil
}
func (r *repos) ListCounter(ctx context.Context) ([]domain.Counter, error) {
	counts := []domain.Counter{}
	err := r.db.WithContext(ctx).Find(&counts).Error
	if err != nil {
		return nil, err
	}
	return counts, nil
}

```


```go
package service

import (
	"context"
	"demo/domain"
	"demo/repositories"
)

type ServiceContracts interface {
	AddCounter(ctx context.Context) (err error)
	GetCounter(ctx context.Context, id int) (domain.Counter, error)
	ListCounter(ctx context.Context) ([]domain.Counter, error)
}

type service struct {
	db repositories.RepoContracts
}

func NewService(db repositories.RepoContracts) *service {
	return &service{
		db: db,
	}
}

func (s *service) AddCounter(ctx context.Context) (err error) {
	err = s.db.AddCounter(ctx)
	if err != nil {
		return err
	}
	return nil
}

func (s *service) GetCounter(ctx context.Context, id int) (domain.Counter, error) {
	res, err := s.db.GetCounter(ctx, id)
	if err != nil {
		return domain.Counter{}, err
	}
	return res, nil
}

func (s *service) ListCounter(ctx context.Context) ([]domain.Counter, error) {
	res, err := s.db.ListCounter(ctx)
	if err != nil {
		return []domain.Counter{}, err
	}
	return res, nil
}

```


```go
package service

import (
	"context"
	"demo/domain"
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

// 1. Define the Mock Repository using testify/mock
type MockRepo struct {
	mock.Mock
}

func (m *MockRepo) AddCounter(ctx context.Context) error {
	args := m.Called(ctx)
	return args.Error(0)
}

func (m *MockRepo) GetCounter(ctx context.Context, id int) (domain.Counter, error) {
	args := m.Called(ctx, id)
	return args.Get(0).(domain.Counter), args.Error(1)
}

func (m *MockRepo) ListCounter(ctx context.Context) ([]domain.Counter, error) {
	args := m.Called(ctx)
	return args.Get(0).([]domain.Counter), args.Error(1)
}

// 2. Unit Tests
func TestService_GetCounter_Testify(t *testing.T) {
	ctx := context.TODO()
	mockRepo := new(MockRepo)
	svc := NewService(mockRepo)

	t.Run("Success - Found Counter", func(t *testing.T) {
		expected := domain.Counter{Id: 101}

		// Setup expectations
		mockRepo.On("GetCounter", ctx, 101).Return(expected, nil).Once()

		// Execute
		res, err := svc.GetCounter(ctx, 101)

		// Assertions
		assert.NoError(t, err)
		assert.Equal(t, expected, res)
		mockRepo.AssertExpectations(t) // Verifies GetCounter was actually called
	})

	t.Run("Failure - Not Found", func(t *testing.T) {
		mockRepo.On("GetCounter", ctx, 99).Return(domain.Counter{}, errors.New("not found")).Once()

		res, err := svc.GetCounter(ctx, 99)

		assert.Error(t, err)
		assert.Empty(t, res)
		assert.Equal(t, "not found", err.Error())
	})
}

func TestService_ListCounter_Testify(t *testing.T) {
	ctx := context.TODO()
	mockRepo := new(MockRepo)
	svc := NewService(mockRepo)

	t.Run("Success - List Items", func(t *testing.T) {
		list := []domain.Counter{{Id: 1}, {Id: 2}}
		mockRepo.On("ListCounter", ctx).Return(list, nil).Once()

		res, err := svc.ListCounter(ctx)

		assert.NoError(t, err)
		assert.Len(t, res, 2)
		assert.Equal(t, 1, res[0].Id)
	})
}

func TestService_AddCounter_Testify(t *testing.T) {
	ctx := context.TODO()
	mockRepo := new(MockRepo)
	svc := NewService(mockRepo)

	t.Run("DB Error on Add", func(t *testing.T) {
		mockRepo.On("AddCounter", ctx).Return(errors.New("disk full")).Once()

		err := svc.AddCounter(ctx)

		assert.Error(t, err)
		assert.Contains(t, err.Error(), "disk full")
	})
}

```
