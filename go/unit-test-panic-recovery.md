
# Unit testing panic recovery middleware in Golang

Assume that you have a panic recovery middleware in your Golang application and you want to make sure it works as expected. You can use example test below.

## Middleware

```go
package middleware

import (
	"log"
	"net/http"

	"internal/pkg/response"
)

func PanicRecovery(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				// Log the error or something else...

				if err := response.Write(w, response.Response{
					Status:  http.StatusInternalServerError,
					Code:    response.CodeInternalError,
					Message: response.MessageInternalError,
				}); err != nil {
					log.Println(err)
				}
			}
		}()

		handler.ServeHTTP(w, r)
	})
}
```

## Unit Test

```go
package middleware

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestPanicRecovery(t *testing.T) {
	homeHandler := func(http.ResponseWriter, *http.Request) { panic("some error") }
	panicHandler := PanicRecovery(http.HandlerFunc(homeHandler))

	req := httptest.NewRequest("GET", "/", nil)
	res := httptest.NewRecorder()

	panicHandler.ServeHTTP(res, req)

	assert.Equal(t, 500, res.Code)
	assert.Equal(t, `{"code":"internal_error","message":"Internal error"}`, res.Body.String())
}
```
