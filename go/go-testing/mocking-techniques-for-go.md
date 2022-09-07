# 5 Mocking Techniques for Go

Unit testing. It's one of those situations that every engineer faces.

If you're dealing with a downstream dependency, you'll want to keep your unit test fast, self-contained, and reliable.

It can be daunting when you're new to a language or framework.

We talked with Kyle Yost, a Software Engineer at CB Insights, who's been working with Go throughout his career. His team emphasizes testing so they can have confidence in the code they ship. Along the way, Kyle found that Go provides all the tools they need to achieve mocking and accomplish their unit tests.

While third-party solutions were an option, he discovered that Go's mocking techniques were the best route.

Third-party tools can make things easier, but Kyle didn't want to sacrifice his knowledge in the process. He found it worthwhile to understand exactly how the team achieves the unit tests. Staying close to what's happening, rather than farming it out to a third-party, ensured they didn't sacrifice their knowledge in the process.

To make things easier, he created a resource that categorizes different mocking techniques (including the situations that would lead you to use them).

The techniques below frame the problem in terms of what's trying to be achieved or the situation that's faced. Examples are included with each technique.

The 5 Mocking Techniques:

1. Higher-Order Functions
2. Monkey Patching
3. Interface Substitution
4. Embedding Interfaces
5. Mocking out Downstream HTTP Calls

## 1. Higher-Order Functions

**Use when you need to mock some package level function.**

Consider this source code that you want to test. It opens a DB connection to mysql.

```go
func OpenDB(user, password, addr, db string) (*sql.DB, error) {
    conn := fmt.Sprintf("%s:%s@%s/%s", user, password, addr, db)

    return sql.Open("mysql", conn)
}
```

We want to mock out the call to `sql.Open`. We can make the following change to the source code to pass in a function to open the connection.

```go
type (
	sqlOpener func(string, string) (*sql.DB, error)
)

func OpenDB(user, password, addr, db string, open sqlOpener) (*sql.DB, error) {
	conn := fmt.Sprintf("%s:%s@%s/%s", user, password, addr, db)

	return open("mysql", conn)
}
```

When calling this function in our source code, we can supply the `sql.Open` function to it:

```go
OpenDB("myUser", "myPass", "localhost", "foo", sql.Open)
```

When we are testing the function, we can supply our own definition of the function in each table test. Here is a complete example with one happy path test and one mock error test:

```go
func TestOpenDB(t *testing.T) {
    mockError := errors.New("uh oh")
    subtests := []struct {
        name        string
        u, p, a, db string
        sqlOpener   func(string, string) (*sql.DB, error)
        expectedErr error
    }{
        {
            name: "happy path",
            u:    "u",
            p:    "p",
            a:    "a",
            db:   "db",
            sqlOpener: func(s string, s2 string) (db *sql.DB, err error) {
                if s != "u:p@a/db" {
                    return nil, errors.New("wrong connection string")
                }
                return nil, nil
            },
        },
        {
            name: "error from sqlOpener",
            sqlOpener: func(s string, s2 string) (db *sql.DB, err error) {
                return nil, mockError
            },
            expectedErr: mockError,
        },
    }
    for _, subtest := range subtests {
        t.Run(subtest.name, func(t *testing.T) {
            _, err := OpenDB(subtest.u, subtest.p, subtest.a, subtest.db, subtest.sqlOpener)
            if !errors.Is(err, subtest.expectedErr) {
                t.Errorf("expected error (%v), got error (%v)", subtest.expectedErr, err)
            }
        })
    }
}
```

Exercise caution when putting this technique to use. HOFs may be difficult to reason about since you are passing in logic that is not proximal to the function body. You may also expand function parameter lists beyond what is reasonable to read. Also consider that you can end up expanding the list of dependencies for packages that call your function. In the example above, callers of `OpenDB(...)` now need to import the `sql` package.

## 2. Monkey Patching

**Use when you need to mock some package level function.**

This technique is very similar to higher-order functions. We make a package level variable in our source code that points to the real call that we need to mock. Instead of passing in a function to `OpenDB()`, we just use the variable for the actual call.

```go
var (
    SQLOpen = sql.Open
)

func OpenDB(user, password, addr, db string) (*sql.DB, error) {
    conn := fmt.Sprintf("%s:%s@%s/%s", user, password, addr, db)

    return SQLOpen("mysql", conn)
}
```

In your test file, you simply reassign the `SQLOpen` variable in the source code with your mock implementation right before you call the function under test.

```go
// Our table tests can remain exactly the same, but we
// inject our subtest definition of the SQLOpen variable
// right before we call the function under test
for _, subtest := range subtests {
    t.Run(subtest.name, func(t *testing.T) {
        SQLOpen = subtest.sqlOpener
        _, err := OpenDB(subtest.u, subtest.p, subtest.a, subtest.db)

        if !errors.Is(err, subtest.expectedErr) {
            t.Errorf("expected error (%v), got error (%v)", subtest.expectedErr, err)
        }
    })
}
```

Sometimes package level variables may not be the best way to write testable code. You may not be able to run tests in parallel without synchronization when many tests are manipulating a single variable. Similarly, if you are writing tests from a test package ( ex: `package mypkg_test` ), you will need to make this variable public so that your test package can change it. This would also allow other callers of your package to do the same, which is usually not an intended consequence.

Use caution with this technique and beware of side effects!

## 3. Interface Substitution

**Use when you need to mock a method on a concrete type.**

In Go, interfaces are implicitly and statically satisfied by implementing types. That means you do not need to explicitly mention that your type will **implement** an interface. If it can do the behaviors of the interface, it is allowed to be treated that way. The static satisfaction means you find out at compile time whether or not your concrete type can be substituted as an interface type. This is one distinguishing mark from true **duck typing** that you see in dynamic languages like python. Because of this, interfaces are incredibly powerful for mocking in tests. The following technique follows the "D" from SOLID design pattern considerations - API boundaries should depend on abstractions rather than concrete implementations.

Sometimes we need to mock a method defined on a type. The simplest way to do this is to define an interface which describes the behaviors that you need rather than dealing with the concrete type. One example is reading from a file. Maybe we do not want to actually read from a file in our unit test. Consider the code below that opens a file in the main function and then calls another method on the os.File type to read a specified number of bytes and close the file.

```go
func main() {
    f, err := os.Open("foo.txt")
    if err != nil {
        fmt.Printf("error opening file %v \n", err)
    }

    data, err := ReadContents(f, 50)
    if err != nil {
        fmt.Printf("error from ReadContents %v \n", err)
    }

    fmt.Printf("data from file: %s", string(data))
}

func ReadContents(f *os.File, numBytes int) ([]byte, error) {
    defer f.Close()
    data := make([]byte, numBytes)

    _, err := f.Read(data)
    if err != nil {
        return nil, err
    }

    return data, nil
}
```

We need to mock out the functionality from the file that is used during `ReadContents(...)`. Specifically, we read from the file with `f.Read(data)` and we eventually close the file with `defer f.Close()`.

We allow for a mock by accepting interfaces rather than an `os.File` struct. In the `io` package in the standard library, there are useful interfaces that we can use:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

type ReadCloser interface {
    Reader
    Closer
}
```

`ReadCloser` **embeds** `Reader` and `Closer`, meaning that it is satisfied when `Reader` and `Closer` are satisfied. More on embedding interfaces in the next technique.

We will use `io.ReadCloser` in our function signature. Note that `os.File` is still what our source code will supply to the function, and this works even though the call to `rc.Read(data)` is only using the method that is intended to satisfy `io.Reader`.

```go
func ReadContents(rc io.ReadCloser, numBytes int) ([]byte, error) {
    defer rc.Close()
    data := make([]byte, numBytes)

    _, err := rc.Read(data)
    if err != nil {
        return nil, err
    }

    return data, nil
}
```

This follows the pattern to "**accept interfaces, return structs**" in Go, which allows you to consistently abstract what you need as a caller, rather than a supplier of functionality. An `os.File` struct is returned from the call to `os.Open()`, and we may use any of the methods defined on that type. For the specific methods that we need to mock (`Read()` and `Close()`) we accept an interface instead of the concrete type in `ReadContents(...)`. In most cases, you may need to create these interfaces yourself, but here we were able to reuse those defined in the `io` package.

Now our test for this function can easily be mocked.

```go
type (
    mockReadCloser struct {
        expectedData []byte
        expectedErr  error
    }
)

// allow subtests to fill out expectedData and expectedErr field
func (mrc *mockReadCloser) Read(p []byte) (n int, err error) {
    copy(p, mrc.expectedData)
    return 0, mrc.expectedErr
}

// do nothing on close
func (mrc *mockReadCloser) Close() error { return nil }

func TestReadContents(t *testing.T) {
    subtests := []struct {
        name         string
        rc           io.ReadCloser
        numBytes     int
        expectedData []byte
        expectedErr  error
    }{
        {
            name: "happy path",
            rc: &mockReadCloser{
                expectedData: []byte(`hello`),
                expectedErr:  nil,
            },
            numBytes:     5,
            expectedData: []byte(`hello`),
            expectedErr:  nil,
        },
    }
    for _, subtest := range subtests {
        t.Run(subtest.name, func(t *testing.T) {
            data, err := ReadContents(subtest.rc, subtest.numBytes)
            if !reflect.DeepEqual(data, subtest.expectedData) {
                t.Errorf("expected (%b), got (%b)", subtest.expectedData, data)
            }
            if !errors.Is(err, subtest.expectedErr) {
                t.Errorf("expected error (%v), got error (%v)", subtest.expectedErr, err)
            }
        })
    }
}
```

Notice that the `mockReadCloser` struct has fields that dictate what the mock will return. This way, each table test can instantiate the struct with its desired return values.

## 4. Embedding Interfaces

**Use when you need to mock out a small set of methods defined in a large interface.**

A great example of this situation comes from the DynamoDB documentation.

When working with the aws-sdk, they provide interfaces for all of their major services that are quite large since they contain all of the calls that can be made for each particular client. Take a look at the `dynamodbiface.DynamoDBAPI` interface from the link. Rather than pass around the concrete client type, you should pass around this interface to other functions. But then, when testing some of your code that calls one particular function of the interface, how do you mock out that call only without mocking every other function in an attempt to satisfy the interface? Here is the example from the link:

Source Code:

```go
// myFunc uses an SDK service client to make a request to
// Amazon DynamoDB.
func myFunc(svc dynamodbiface.DynamoDBAPI) bool {
    // Make svc.BatchGetItem request
}

func main() {
    sess := session.New()
    svc := dynamoDB.New(sess)

    myFunc(svc)
}
```

This is an incomplete example for simplicity, but notice that `myFunc` is signed with the `dynamodbiface.DynamoDBAPI` interface which contains the entire API for DynamoDB. It will use it only for a call to `BatchGetItem`, so that is what we need to mock.

Test:

```go
// Define a mock struct to be used in your unit tests of myFunc.
type mockDynamoDBClient struct {
    dynamodbiface.DynamoDBAPI
}

func (m *mockDynamoDBClient) BatchGetITem(input *dynamodb.BatchGetItemInput) (
    *dynamodb.BatchGetItemOutput,
    error,
) {
    // mock response/functionality
}

func TestMyFunc(t *testing.T) {
    // Setup Test
    mockSvc := &mockDynamoDBClient{}

    myFunc(mockSvc)

    // Verify myFunc's functionality
}
```

So instead of having to create our own type that satisfies the entire interface, we can simply embed the `dynamodbiface.DynamoDBAPI` inside our mock struct (to implicitly satisfy the interface contract) and then redefine the function(s) that we care about.

## 5. Mocking out Downstream HTTP Calls

**Use when your code under test makes an HTTP call to a downstream service.**

It is generally understood that unit tests should not connect to external services in order to remain reliable and self-contained. Any one of the previous mocking techniques would suffice for this situation (depending on the construction of your code), but the standard library provides a better way to achieve this.

The `net/http/httptest` package provides a Server type that will listen on your system's local loopback interface. This is a server completely self-contained within your system's network, so no external network calls are made, but you can still get the benefit of exercising code that is very similar to the actual calls that your source code will make. To swap out the actual server for a test server during your test, simply parameterize the URL that you will be connecting to, and then call your function under test with the URL of the test server.

Consider this function, which makes an HTTP call and returns a struct containing the data from the body of the response:

```go
type Response struct {
    ID          int    `json:"id"`
    Name        string `json:"name"`
    Description string `json:"description"`
}

func MakeHTTPCall(url string) (*Response, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    r := &Response{}
    if err := json.Unmarshal(body, r); err != nil {
        return nil, err
    }

    return r, nil
}
```

Test:

```go
func TestMakeHTTPCall(t *testing.T) {
    testTable := []struct {
        name             string
        server           *httptest.Server
        expectedResponse *Response
        expectedErr      error
    }{
        {
            name: "happy-server-response",
            server: httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                w.WriteHeader(http.StatusOK)
                w.Write([]byte(`{"id": 1, "name": "kyle", "description": "novice gopher"}`))
            })),
            expectedResponse: &Response{
                ID:          1,
                Name:        "kyle",
                Description: "novice gopher",
            },
            expectedErr: nil,
        },
    }
    for _, tc := range testTable {
        t.Run(tc.name, func(t *testing.T) {
            defer tc.server.Close()
            resp, err := MakeHTTPCall(tc.server.URL)
            if !reflect.DeepEqual(resp, tc.expectedResponse) {
                t.Errorf("expected (%v), got (%v)", tc.expectedResponse, resp)
            }
            if !errors.Is(err, tc.expectedErr) {
                t.Errorf("expected (%v), got (%v)", tc.expectedErr, err)
            }
        })
    }
}
```

Here we let every test case in our test table to create and close a test server with a mocked response. Since the call to `httptest.NewServer()` takes in an `http.Handler`, you may decide to just create one test server for all of your test cases, but with different routes, logic, or custom responses.

## Conclusion

Without lecturing on the importance of keeping unit tests reliable and self-contained, this article can hopefully serve as a reference for the many situations you may find yourself in while writing tests in Go.

Tests only add value if you have complete confidence in your approach. The main goal of automated testing should be to give you confidence in the code that you are shipping. Any mystery introduced by a third party package works counter to that goal. If other packages are perfectly understood and they make your life easier, then go for it!

Don't accept that any code is "untestable", and keep a tight grip on your tests!
