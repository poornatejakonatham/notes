# Using the `httptest` package in Golang

## Testing at the edge

Testing your code is a great practice, and can give confidence to the developers shipping it to production. Unit and integration tests are great for testing things like application logic or independent pieces of functionality, but there are other areas of code at the “edges” of the application that are a bit harder to test because they deal with incoming or outgoing requests from third parties. Luckily Go has baked into its standard library the `httptest` package, a small set of structures and functions to help create end-to-end tests for these edges of your application.

## Using the `ResponseRecorder` and `NewRequest`

A common "edge" in a Go application is where a server exposes `http.Handler` functions to respond to web traffic. Normally, to test these, it would require standing up your server somewhere, but the `httptest` package gives us `NewRequest` and `NewRecorder` to help simplify these sorts of test cases.

### Testing an HTTP handler

This test calls an HTTP handler function and checks it a few behaviors, namely: a 200 response code is sent back and a header with the API version is returned.

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    // Tell the client that the API version is 1.3
    w.Header().Add("API-VERSION", "1.3")
    w.Write([]byte("ok"))
}

func TestHandler(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "http://example.com", nil)
    w := httptest.NewRecorder()
    Handler(w, req)

    // We should get a good status code
    if want, got := http.StatusOK, w.Result().StatusCode; want != got {
        t.Fatalf("expected a %d, instead got: %d", want, got)
    }

    // Make sure that the version was 1.3
    if want, got := "1.3", w.Result().Header.Get("API-VERSION"); want != got {
        t.Fatalf("expected API-VERSION to be %s, instead got: %s", want, got)
    }
}
```

`httptest.NewRequest` provides a convenience wrapper around `http.NewRequest` so you don't have to check the error making a `Request` object. Below that `httptest.NewRecorder` makes a recorder that the HTTP handler writes to as its `http.ResponseWriter`, and it captures all of the changes that would have been returned to a client caller. Using this, there’s no need to start your server: just hand the recorder directly to the function and it invokes it the same way it would if the request came in over HTTP. After the handler call, the recorder’s `Result` call provides the values written to it for checking any behaviors you may need to to assert in the rest of your test.

## Using the Test Server

While servers often intake requests, there’s another “edge” to be tested on the other side where a server makes outbound requests. Testing these behaviors can be difficult since it requires that you either mock the code calling out or call out to the real thing (maybe even a test instance). Thankfully `httptest` gives us `Server`, a way to start a local server to respond to real HTTP requests inside of a test.

### Test Setup

```go
func TestTrueSundayResponseReturnsTrue(t *testing.T) {
    // Create a server that returns a static JSON response
    s := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"isItSunday": true}`))
    }))
    // Be sure to clean up the server or you might run out of file descriptors!
    defer s.Close()

    // The pretend client under test
    c := client.New()

    // Ask the client to reach out to the server and see if it's Sunday
    if !c.Sunday(s.URL) {
        t.Fatalf("expected client to return true!")
    }
}
```

`httptest.NewServer` accepts an `http.Handler`, so for this test, we gave it a function that responds in JSON that it’s Sunday. When you make a new test server, it binds to a random free port on the local machine, but we can access the URL it exposed by passing the `URL` field on the server struct to the client code. From there, the client can make an actual request to a server, not a mock, and parse the response like it would in production. Note that you should clean up the server by calling `Close` when the test finishes to free up resources or you may find yourself out of available ports for running further tests.

## Leveraging the http.Handler interface

Because `NewServer` accepts an instance of an `http.Handler`, the test server can do a lot more than just return static responses, like providing multiple route handlers. In the next example, the test server will provide two endpoints:

1. `/token` which returns some secret created during the test.

2. `/resource` returns what day it is, but only from requests that have the secret bearer token in their header.

The goal of this test is to ensure that the client code calls both endpoints and takes information from one endpoint and properly passes it to the other.

```go
func TestPseudOAuth(t *testing.T) {
    // Make a secret for the instance of this test
    secret := fmt.Sprintf("secret_%d", time.Now().Unix())

    // Implements the http.Handler interface to be passed to httptest.NewServer
    mux := http.NewServeMux()

    // The /resource handler checks the headers for a non-expired token.
    // It returns a 401 if it is, otherwise returns the treasure inside.
    mux.HandleFunc("/resource", func(w http.ResponseWriter, r *http.Request) {
        auth := r.Header.Get("Auth")
        if auth != secret {
            http.Error(w, "Auth header was incorrect", http.StatusUnauthorized)
            return
        }

        // Header was good, tell 'em what day it is
        w.Write([]byte(`{ "day": "Sunday" }`))
    })

    // The /token handler mints a new token that's good for 5 minutes to
    // access the /resource endpoint
    mux.HandleFunc("/token", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(secret))
    })

    s := httptest.NewServer(mux)
    defer s.Close()

    // The pretend client under test
    c := client.New()

    // Make the call and make sure it's Sunday
    got, err := c.GetDay(s.URL)
    if err != nil {
        t.Fatalf("unexpected error: %s", err)
    }
    if want := "Sunday"; want != got {
        t.Fatalf("I thought it was %s! Instead it's: %s", want, got)
    }
}
```

This test looks a lot like the previous one, except we’re passing a different implementation of an `http.Handler` to the test server. Although the code uses `http.NewServeMux`, you can use anything, like `gorilla/mux`, so long as it implements the interface. Just by changing the `http.Handler` to be a more elaborate route handler, the tests can make more detailed assertions about outgoing HTTP requests and flows.

## Avoiding fragile End-To-End tests

End-To-End tests by nature call every part of your application required to serve a request, and so they can rely on quite a few components within your code to function. While they’re great for adding test coverage and spreading that coverage to the very edges of your application, they can also be flakier than their unit/integration test counterparts. To avoid writing flaky tests, be sure to only test for observable behaviors and avoid testing for the internals as those are more likely to change than the output of the feature under test.

## Conclusion

Go’s `httptest` package provides a small, but incredibly useful set of tools for testing the edge portions of HTTP handling code, both for servers and their clients. It provides some neat tools to add test coverage to the “edges” of your applications using real servers and requests. Best of all, it’s included in Go’s standard library, so the Continuous Integration pipeline for your Go code already has support for it without further hassle. If you have any interest in the additional utilities the `httptest` package provides, you can read the documentation itself.
