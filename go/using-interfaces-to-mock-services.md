# Using interfaces to mock services in Golang tests

If you wish to mock services while writing unit tests in Golang, you can simply use interfaces. Mocking often is required if you don't want your tests to invoke the "real" code. Our example authenticates with Twitter and posts messages to it.


## Structure

```bash
└── internal
    ├── app
    │   ├── app.go
    │   └── app_test.go
    └── twitter
        └── twitter.go
```

## Files

**app.go**

```go
package app

import (
	"github.com/inanzzz/internal/twitter"
)

type App struct {
	client twitter.Twitter
}

func New(c twitter.Twitter) App {
	return App{client: c}
}

func (a App) SendMessage(msg string) string {
	return a.client.Tweet(msg)
}
```

**twitter.go**

```go
package twitter

import (
	"fmt"
)

type Twitter struct {
	BaseURL      string
	ClientID     string
	ClientSecret string
}

func New() Twitter {
	return Twitter{
		BaseURL:      "https://www.real-url.com",
		ClientID:     "real-id",
		ClientSecret: "real-secret",
	}
}

func (t Twitter) Tweet(msg string) string {
	return fmt.Sprintf(
		"Sending '%s' to '%s' using '%s:%s' credentials",
		msg,
		t.BaseURL,
		t.ClientID,
		t.ClientSecret,
	)
}
```

**app_test.go**

```go
package app

import (
	"log"
	"testing"

	"github.com/inanzzz/internal/twitter"
)

func TestApp_SendMessage(t *testing.T) {
	twt := twitter.New()

	app := New(twt)

	res := app.SendMessage("hello")

	log.Println(res)
}
```

## Result

As you can see below, when we run the test, it actually called the real Twitter API with real configuration parameters. This is something you definitely want to avoid. Let's solve the issue in next stage.

```bash
=== RUN   TestApp_SendMessage
2020/05/12 17:50:39 Sending 'hello' to 'https://www.real-url.com' using 'real-id:real-secret' credentials
--- PASS: TestApp_SendMessage (0.00s)
PASS
```

## Structure

```bash
└── internal
    ├── app
    │   ├── app.go
    │   └── app_test.go
    └── twitter
        ├── mock
        │   └── twitter.go // This is new
        └── twitter.go
```

## Files

**app.go**

We have replaced concrete `twitter.Twitter` type with `twitter.SocialMedia` interface type.

```go
package app

import (
	"github.com/inanzzz/internal/twitter"
)

type App struct {
	client twitter.SocialMedia
}

func New(c twitter.SocialMedia) App {
	return App{client: c}
}

func (a App) SendMessage(msg string) string {
	return a.client.Tweet(msg)
}
```

**twitter.go**

We introduced `SocialMedia` interface type.

```go
package twitter

import (
	"fmt"
)

type SocialMedia interface {
	Tweet(msg string) string
}

type Twitter struct {
	BaseURL      string
	ClientID     string
	ClientSecret string
}

func New() Twitter {
	return Twitter{
		BaseURL:      "https://www.real-url.com",
		ClientID:     "real-id",
		ClientSecret: "real-secret",
	}
}

func (t Twitter) Tweet(msg string) string {
	return fmt.Sprintf(
		"Sending '%s' to '%s' using '%s:%s' credentials",
		msg,
		t.BaseURL,
		t.ClientID,
		t.ClientSecret,
	)
}
```

**mock/twitter.go**

We created this mock version which satisfies the signature of `twitter.SocialMedia` interface type.

```go
package mock

import (
	"fmt"
)

type TwitterMock struct {
	BaseURL      string
	ClientID     string
	ClientSecret string
}

func (t TwitterMock) Tweet(msg string) string {
	return fmt.Sprintf(
		"Sending '%s' to '%s' using '%s:%s' credentials",
		msg,
		t.BaseURL,
		t.ClientID,
		t.ClientSecret,
	)
}
```

**app_test.go**

We are injecting the `mock.TwitterMock` as dependency.

```go
package app

import (
	"log"
	"testing"

	"github.com/inanzzz/internal/twitter/mock"
)

func TestApp_SendMessage(t *testing.T) {
	twt := mock.TwitterMock{
		BaseURL:      "https://www.mock-url.com",
		ClientID:     "mock-id",
		ClientSecret: "mock-secret",
	}

	app := New(twt)

	res := app.SendMessage("hello")

	log.Println(res)
}
```

## Result

As you can see below, our test interacted with the mock service. When it makes sense, you can even hardcode the values in the mock files.

```bash
=== RUN   TestApp_SendMessage
2020/05/12 18:01:51 Sending 'hello' to 'https://www.mock-url.com' using 'mock-id:mock-secret' credentials
--- PASS: TestApp_SendMessage (0.00s)
PASS
```
