# General purpose struct to reduce dependency injection

Assume that you have a service class that depends on two database repository classes. This will require you to store both of them in the service struct. This is ok for small amount of dependencies but if you need more, you would have a big struct and messy code to work with. You might have other services do the same thing but for different repository classes. Finally you have a lot of duplicates all over the place. The solution to this problem is simple. We are going to create a main repository struct which will store all the other/sub repository interfaces in it. We than just use the main repository where we need. This will allow us to access all the embedded repositories form it. A bit like a strategy pattern.

## Structure

```bash
├── cmd
│   └── client
│       └── main.go
└── internal
    ├── dao
    │   ├── article.go
    │   └── label.go
    ├── repository
    │   ├── article.go
    │   ├── label.go
    │   └── repository.go
    └── service
        ├── article.go
        └── label.go
```

## Files

**dao/article.go**

```go
package dao

import "time"

type ArticleRepository interface {
	Insert(article *Article) error
	Find(ID string) (*Article, error)
}

type Article struct {
	ID          string
	Title       string
	PublishedAt time.Time
}
```

**dao/label.go**

```go
package dao

type LabelRepository interface {
	Insert(label *Label) error
	List(limit uint8) ([]*Label, error)
}

type Label struct {
	ID   string
	Name string
}
```

**repository/repository.go**

```go
package repository

import "github.com/you/client/internal/dao"

type Repository struct {
	Article dao.ArticleRepository
	Label   dao.LabelRepository
}

func New() Repository {
	return Repository{
		Article: article{},
		Label:   label{},
	}
}
```

**repository/acticle.go**

```go
package repository

import (
	"log"

	"github.com/you/client/internal/dao"
)

type article struct{}

func (a article) Insert(article *dao.Article) error {
	log.Println("inserting article")
	return nil
}

func (a article) Find(ID string) (*dao.Article, error) {
	log.Println("finding article")
	return nil, nil
}
```

**repository/label.go**

```go
package repository

import (
	"log"

	"github.com/you/client/internal/dao"
)

type label struct{}

func (l label) Insert(label *dao.Label) error {
	log.Println("inserting label")
	return nil
}

func (l label) List(limit uint8) ([]*dao.Label, error) {
	log.Println("listing labels")
	return nil, nil
}
```

**service/article.go**

```go
package service

import (
	"github.com/you/client/internal/dao"
	"github.com/you/client/internal/repository"
)

type Article struct {
	repo repository.Repository
}

func NewArticle(repo repository.Repository) Article {
	return Article{repo: repo}
}

func (a Article) DoSomething() {
	_ = a.repo.Article.Insert(&dao.Article{})
	_, _ = a.repo.Article.Find("id")
}
```

**service/label.go**

```go
package service

import (
	"github.com/you/client/internal/dao"
	"github.com/you/client/internal/repository"
)

type Label struct {
	repo repository.Repository
}

func NewLabel(repo repository.Repository) Label {
	return Label{repo: repo}
}

func (l Label) DoSomething() {
	_ = l.repo.Label.Insert(&dao.Label{})
	_, _ = l.repo.Label.List(1)
}
```

**main.go**

```go
package main

import (
	"github.com/you/client/internal/repository"
	"github.com/you/client/internal/service"
)

func main() {
	repo := repository.New()

	articleService := service.NewArticle(repo)
	articleService.DoSomething()

	labelService := service.NewLabel(repo)
	labelService.DoSomething()
}
```

## Test

```bash
$ go run -race cmd/client/main.go
2020/12/22 23:26:14 inserting article
2020/12/22 23:26:14 finding article
2020/12/22 23:26:14 inserting label
2020/12/22 23:26:14 listing labels
```
