# Golang Integration Testing

## TL;DR

1. Separate integration tests into separate `*_integration_test.go` files
2. Add `// +build integration` to the top of these files to not run them be default when using go test
3. Use a **.env** file to store sensitive info required by your integration tests
4. Use `go test -tags=integration` to run your tests

## Overview

Golang makes testing extremely easy. In most cases, all that's needed is to create a file called appended with `_test.go`, then write a test function. For example, given a function like the following:


```go
// add.go

func Sum(i, j int) int {
  return i + j
}
```

We can write a test function like this:

```go
// add_test.go

func TestSum(t *testing.T) {
    sum := Sum(2, 2)

    if sum != 4 {
        t.Errorf("Expected %v, got %v", 4, sum)
    }
}
```

However, let's imagine that we discovered a brand new web API that will take care of all our mathematical needs. Let's imagine this server lives at `math.example.com`, and the summation API can be called like so:

```
GET http://math.example.com/add?a=2&b=3&authtoken=abcdef123

{
    "result": 5
}
```

To migrate to this API, we need to create a function like this:

```go
// add.go

type MathClient struct {
    Token string
    Host  string
}

type AddResult struct {
    Result int `json:"result"`
}

func (c *MathClient) APISum(i, j int) (int, error) {
    query := fmt.Sprintf("http://%s/add?a=%v&b=%v&authtoken=%v", c.Host, i, j, c.Token)

    response, err := http.Get(query)
    if err != nil {
        return 0, err
    }
    defer response.Body.Close()

    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        return 0, err
    }

    a := AddResult{}

    err = json.Unmarshal(data, &a)
    if err != nil {
        return 0, err
    }

    return a.Result, nil
}
```

We also need to create a new test function:

```go
// api_test.go

func TestAPISum(t *testing.T) {
    client := MathClient{
        Token: "abcdef123",
        Host:  "math.example.com",
    }

    sum, err := client.APISum(2, 2)
    if err != nil {
        t.Errorf("No error expected, got %v", err)
    }

    if sum != 4 {
        t.Errorf("Expected %v, got %v", 4, sum)
    }
}
```

Since this tests how we integrate with an external API rather than being fully self-isolated, we can consider it an integration test rather than a basic unit test.

## The Problem

With these modifications, we can now rely on this external service while still testing that our package works as expected. However, this immediately introduces a couple of problems into our testing suite:

1. Our test suite is now dependent on another service. Our test suite will start failing if the API goes down, we have a network connection issue, or we hit an API rate-limit.
2. Since network connections take significantly longer than Golang functions, these API calls could start slowing down our test suite significantly.
3. We have to start managing potentially sensitive configurations for our test suite. Right now we have a hardcoded authentication token sitting in our test suite, which is lousy security practice.

While there are ways we could potentially stub out hardcoded API responses and remove our dependency on the external service, there's a lot of benefit to perform integration tests against the real API. However, we need to separate these fragile, expensive tests so they can be run less frequently. We also need to separate out any sensitive information from the test itself.

## Recommendations

### 1. Separate out the integration tests into their own files

This can be whatever naming convention you'd prefer, but my preference is `*_integration_test.go`. In this case, we would move the entire contents of function `TestAPISum` into a new file `add_integration_test.go`.

### 2. Fence off integration tests with build tags

Golang build tags are a means of separating which code gets included in our builds. We can use these to fence off our integration tests by adding the following to the first line of the integration test file:

```go
// add_integration_test.go

// +build integration
...
```

Now we have different options when running `go test`:

```bash
go test # Run all unit tests in our package
go test -tags=integration # Run all unit and integration tests in our package
go test -tags=integration -run ^TestAPISum$ # Run a specific integration test
```

This means we can run our unit tests as often as we like, while only running the integration test suite as-needed.

### 3. Use a .env file to hold our sensitive configuration info

`.env` files are a great way to store sensitive info without exposing it in our git repository. First, make sure you have the file excluded in your `.gitignore`:

```txt
// .gitignore

.env
```

Then add your sensitive info as key-value pairs:

```txt
// .env

AUTH_TOKEN=abcdef123
```

To make this available for our tests, we first need to load this .env file in the shell environment, then read the appropriate environment variables. One way of doing this is via the [godotenv](github.com/joho/godotenv) module. We can alter our integration test like so:

```go
// add_integration_test.go

func TestAPISum(t *testing.T) {
    err := godotenv.Load()
    if err != nil {
        t.Fatalf("could not load .env file: %v", err)
    }

    token := os.Getenv("AUTH_TOKEN")

    client := MathClient{
        Token: token,
        Host:  "math.example.com",
    }

    ...
```
