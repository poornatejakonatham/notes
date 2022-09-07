# Access log middleware in Golang

This example will help you to capture request and response related data then log it.

## Middleware

```go
package middleware

import (
	"bufio"
	"fmt"
	"net"
	"net/http"
	"time"

	"github.com/sirupsen/logrus"
)

// AccessLog helps log request and response related data.
func AccessLog(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t := time.Now()

		wr := newWriter(w, r, t)

		h.ServeHTTP(wr, r)

		logrus.WithFields(logrus.Fields{
			"http_req_client_ip":  wr.reqClientIP,
			"http_req_duration":   time.Since(t).Milliseconds(),
			"http_req_host":       wr.reqHost,
			"http_req_method":     wr.reqMethod,
			"http_req_path":       wr.reqPath,
			"http_req_protocol":   wr.reqProto,
			"http_req_size":       wr.reqSize,
			"http_req_time":       wr.reqTime,
			"http_req_user_agent": wr.reqUserAgent,
			"http_res_size":       wr.resSize,
			"http_res_status":     wr.resStatus,
		}).Info()
	})
}

// Acts as an adapter for http.ResponseWriter type to store request and
// response data.
type writer struct {
	http.ResponseWriter

	reqClientIP  string
	reqHost      string
	reqMethod    string
	reqPath      string
	reqProto     string
	reqSize      int64 // bytes
	reqTime      string
	reqUserAgent string

	resStatus int
	resSize   int // bytes
}

func newWriter(w http.ResponseWriter, r *http.Request, t time.Time) *writer {
	return &writer{
		ResponseWriter: w,

		reqClientIP:  r.Header.Get("X-Forwarded-For"),
		reqMethod:    r.Method,
		reqHost:      r.Host,
		reqPath:      r.RequestURI,
		reqProto:     r.Proto,
		reqSize:      r.ContentLength,
		reqTime:      t.Format(time.RFC3339),
		reqUserAgent: r.UserAgent(),
	}
}

// Overrides http.ResponseWriter type.
func (w *writer) WriteHeader(status int) {
	if w.resStatus == 0 {
		w.resStatus = status
		w.ResponseWriter.WriteHeader(status)
	}
}

// Overrides http.ResponseWriter type.
func (w *writer) Write(body []byte) (int, error) {
	if w.resStatus == 0 {
		w.WriteHeader(http.StatusOK)
	}

	var err error
	w.resSize, err = w.ResponseWriter.Write(body)

	return w.resSize, err
}

// Overrides http.Flusher type.
func (w *writer) Flush() {
	if fl, ok := w.ResponseWriter.(http.Flusher); ok {
		if w.resStatus == 0 {
			w.WriteHeader(http.StatusOK)
		}

		fl.Flush()
	}
}

// Overrides http.Hijacker type.
func (w *writer) Hijack() (net.Conn, *bufio.ReadWriter, error) {
	hj, ok := w.ResponseWriter.(http.Hijacker)
	if !ok {
		return nil, nil, fmt.Errorf("the hijacker interface is not supported")
	}

	return hj.Hijack()
}
```

## Examples

```json
{
    "http_req_client_ip": "1.18.258.61",
    "http_req_duration": 33,
    "http_req_host": "localhost:8080",
    "http_req_method": "POST",
    "http_req_path": "/api/v1/leagues",
    "http_req_protocol": "HTTP/1.1",
    "http_req_size": 133,
    "http_req_time": "2021-07-10T22:49:47+01:00",
    "http_req_user_agent": "PostmanRuntime/7.26.8",
    "http_res_size": 56,
    "http_res_status": 201,
    "level": "info",
    "message": "",
    "time": "2021-07-10T22:49:47+01:00"
}
```

```json
{
    "http_req_client_ip": "1a01:1c8:411:f815:1:2:961d:22e1",
    "http_req_duration": 1000,
    "http_req_host": "localhost:8080",
    "http_req_method": "GET",
    "http_req_path": "/api/v1/leagues/f47a0eaf-cfd3-4842-9a3e-19eda62404a8",
    "http_req_protocol": "HTTP/1.1",
    "http_req_size": 0,
    "http_req_time": "2021-07-10T22:59:50+01:00",
    "http_req_user_agent": "Mozilla/5.0 (Android 9; Mobile; rv:88.0) Gecko/88.0 Firefox/88.0",
    "http_res_size": 225,
    "http_res_status": 200,
    "level": "info",
    "message": "",
    "time": "2021-07-10T22:59:51+01:00"
}
```
