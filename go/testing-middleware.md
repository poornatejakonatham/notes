# Testing a middleware within Golang

A middleware is basically the same as a route in Go. You can test it as if you are testing a route handler by using httptest as shown below.

## Middleware

```go
func Timer(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        timer := time.Now()
        next.ServeHTTP(w, r)

        if id, ok := r.Context().Value("some-key").(string); ok {
            // Do something with the id and timer
        }
    }
}
```

## Test

```go
func TestTimer(t *testing.T) {
    timerHandler := func(w http.ResponseWriter, r *http.Request) {}

    req := httptest.NewRequest(http.MethodGet, "http://www.your-domain.com", nil)
    req = req.WithContext(context.WithValue(req.Context(), "some-key", "123ABC"))

    res := httptest.NewRecorder()

    timerHandler(res, req)

    tim := Timer(timerHandler)
    tim.ServeHTTP(res, req)
}
```
