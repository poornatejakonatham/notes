
# Role-Based Access Control HTTP middleware in Golang

In this example we are going to rely on a **Role-Based Access Control (RBAC)** HTTP middleware in order to authorise authenticated users. Currently it has two built-in options. `And` requires all required permissions to be met by the user's request. `Or` requires at least one of the required permissions to be met by the user's request. If you wish, you can add more options with custom business logic.

## Requirements

- `GET /docs` - requires no permissions.
- `POST /accounts` - requires at least one of `super_admin`, `admin` permissions.
- `GET /accounts` - requires at least one of `admin`, `accounts_get` permissions.
- `GET /customers/:id` - requires all of `user`, `customers_get` permissions.
- `DELETE /customers/:id` - requires all of `super_admin`, `customers_delete` permissions.

## Structure

```bash
├── cmd
│   └── client
│       └── main.go
└── internal
    ├── account
    │   └── controller.go
    ├── customer
    │   └── controller.go
    ├── doc
    │   └── controller.go
    └── pkg
        └── authorisation
            ├── authorisation.go
            └── authorisation_test.go
```

## Files

I am not adding controllers because they are just empty HTTP handlers. You can create one for each.

**main.go**

```go
package main

import (
	"log"
	"net/http"

	"github.com/you/client/internal/account"
	"github.com/you/client/internal/customer"
	"github.com/you/client/internal/doc"
	"github.com/you/client/internal/pkg/authorisation"

	"github.com/julienschmidt/httprouter"
)

func main() {
	acc := account.Controller{}
	cus := customer.Controller{}

	rtr := httprouter.New()

	// Documentation (Public)
	rtr.HandlerFunc(http.MethodPost, "/docs", doc.Controller{}.Info)

	// Accounts (Private)
	rtr.HandlerFunc(http.MethodPost, "/accounts", authorisation.Check(acc.Create,
		authorisation.Or{Permissions: []string{"super_admin", "admin"}},
	))
	rtr.HandlerFunc(http.MethodGet, "/accounts/:id", authorisation.Check(acc.Balance,
		authorisation.Or{Permissions: []string{"admin", "accounts_get"}},
	))

	// Customers (Private)
	rtr.HandlerFunc(http.MethodGet, "/customers/:id", authorisation.Check(cus.Info,
		authorisation.And{Permissions: []string{"user", "customers_get"}},
	))
	rtr.HandlerFunc(http.MethodDelete, "/customers/:id", authorisation.Check(cus.Delete,
		authorisation.And{Permissions: []string{"super_admin", "customers_delete"}},
	))

	log.Fatalln(http.ListenAndServe(":3000", rtr))
}
```

**authorisation.go**

```go
// authorisation provides Role-Based Access Control (RBAC) like functionality
// in order to restrict resource access to authorised client. It currently has
// two built-in conditional permission checker types, however it accepts custom
// ones from outside.
package authorisation

import (
	"fmt"
	"net/http"
	"strings"
)

// HeaderXPermissions represents the HTTP request header which contains space
// separated permissions as its value.
//
// It is critical that the value is tamper-proofed and up to the develop how
// it is managed. For instance, if the client is asked to send it, the value
// could also be signed along with the access token (e.g. JWT) and verified
// as part of initial authentication process, or if the client is not asked
// to send it, an additional HTTP middleware would extract it from the token
// before injecting into request headers.
const HeaderXPermissions = "X-Permissions"

// checker enforces all built-in and custom permission checker types to obey
// its methods. It allows developers to implement their own permission checker
// types to run custom business logic.
type checker interface {
	IsSatisfied(perms string) bool
}

// And requires all permission to be match.
type And struct {
	Permissions []string
}

// isSatisfied checks if all the required permissions have been present in the
// HTTP request header.
func (a And) IsSatisfied(xPerms string) bool {
	if xPerms == "" || len(a.Permissions) == 0 {
		return false
	}

	perms := strings.Split(xPerms, " ")
	if len(perms) == 0 {
		return false
	}

	// Build map out of provided permissions for easy lookup.
	list := make(map[string]struct{}, len(perms))
	for _, perm := range perms {
		list[perm] = struct{}{}
	}

	// As soon as discovering a missing permission, early indicate a failure.
	for _, perm := range a.Permissions {
		if _, ok := list[perm]; !ok {
			return false
		}
	}

	return true
}

// Or requires at least one permission match.
type Or struct {
	Permissions []string
}

// isSatisfied checks if at least one of the required permissions has been
// present in the HTTP request header.
func (o Or) IsSatisfied(xPerms string) bool {
	if xPerms == "" || len(o.Permissions) == 0 {
		return false
	}

	perms := strings.Split(xPerms, " ")
	if len(perms) == 0 {
		return false
	}

	// Build map out of provided permissions for easy lookup.
	list := make(map[string]struct{}, len(perms))
	for _, perm := range perms {
		list[perm] = struct{}{}
	}

	// As soon as a permission match, early indicate a success.
	for _, perm := range o.Permissions {
		if _, ok := list[perm]; ok {
			return true
		}
	}

	return false
}

// Check accepts a built-in or a custom checker type and instructs it to
// check if the required permissions were satisfied or not. Based on the
// result, it either returns a 403 response or continues with the request.
func Check(h http.HandlerFunc, c checker) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if ok := c.IsSatisfied(r.Header.Get(HeaderXPermissions)); !ok {
			w.WriteHeader(http.StatusForbidden)
			return
		}

		h.ServeHTTP(w, r)
	}
}

// CreateHeaderValue accepts list of permissions and construct a standard
// space separated string value to go with the "X-Permissions" header.
func CreateHeaderValue(perms []string) (string, error) {
	if len(perms) == 0 {
		return "", fmt.Errorf("empty permissions")
	}

	return strings.Join(perms, " "), nil
}
```

**autrorisation_test.go**

```go
package authorisation

import "testing"

//gotest -v -bench=. -benchmem ./internal/pkg/authorisation/
//goos: darwin
//goarch: amd64
//cpu: Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz
//Benchmark_And_IsSatisfied
//Benchmark_And_IsSatisfied-4   	 4460810	       249.2 ns/op	      48 B/op	       1 allocs/op
func Benchmark_And_IsSatisfied(b *testing.B) {
	checker := And{[]string{"super_admin", "admin", "user"}}
	xPermissions := "admin user super_admin"

	for i := 0; i < b.N; i++ {
		_ = checker.IsSatisfied(xPermissions)
	}
}

//gotest -v -bench=. -benchmem ./internal/pkg/authorisation/
//goos: darwin
//goarch: amd64
//cpu: Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz
//Benchmark_Or_IsSatisfied
//Benchmark_Or_IsSatisfied-4   	 5280307	       232.4 ns/op	      48 B/op	       1 allocs/op
func Benchmark_Or_IsSatisfied(b *testing.B) {
	checker := Or{[]string{"super_admin", "admin", "user"}}
	xPermissions := "admin user super_admin"

	for i := 0; i < b.N; i++ {
		_ = checker.IsSatisfied(xPermissions)
	}
}

func Test_And_IsSatisfied(t *testing.T) {
	tests := []struct {
		name             string
		havePermissions  []string
		haveXPermissions string
		wantIsSatisfied  bool
	}{
		{
			"not satisfied when no permissions were required",
			nil,
			"admin user guest",
			false,
		},
		{
			"not satisfied when no permissions were found in header",
			[]string{"admin", "user", "guest"},
			"",
			false,
		},
		{
			"not satisfied when at least one permission was not found in header",
			[]string{"admin", "user", "guest"},
			"admin guest",
			false,
		},
		{
			"satisfied when all permissions were found in header",
			[]string{"admin", "user", "guest"},
			"user admin guest",
			true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			checker := And{Permissions: tt.havePermissions}

			if ok := checker.IsSatisfied(tt.haveXPermissions); ok != tt.wantIsSatisfied {
				t.Errorf("expected %v got %v", tt.wantIsSatisfied, ok)
			}
		})
	}
}

func Test_Or_IsSatisfied(t *testing.T) {
	tests := []struct {
		name             string
		havePermissions  []string
		haveXPermissions string
		wantIsSatisfied  bool
	}{
		{
			"not satisfied when no permissions were required",
			nil,
			"read write execute",
			false,
		},
		{
			"not satisfied when no permissions were found in header",
			[]string{"read", "write", "execute"},
			"",
			false,
		},
		{
			"satisfied when at least one permission was found in header",
			[]string{"read", "write", "execute"},
			"user admin read guest accounts",
			true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			checker := Or{Permissions: tt.havePermissions}

			if ok := checker.IsSatisfied(tt.haveXPermissions); ok != tt.wantIsSatisfied {
				t.Errorf("expected %v got %v", tt.wantIsSatisfied, ok)
			}
		})
	}
}
```

## Test

```bash
$ go test -v ./...

=== RUN   Test_And_IsSatisfied
=== RUN   Test_And_IsSatisfied/not_satisfied_when_no_permissions_were_required
=== RUN   Test_And_IsSatisfied/not_satisfied_when_no_permissions_were_found_in_header
=== RUN   Test_And_IsSatisfied/not_satisfied_when_at_least_one_permission_was_not_found_in_header
=== RUN   Test_And_IsSatisfied/satisfied_when_all_permissions_were_found_in_header
--- PASS: Test_And_IsSatisfied (0.00s)
    --- PASS: Test_And_IsSatisfied/not_satisfied_when_no_permissions_were_required (0.00s)
    --- PASS: Test_And_IsSatisfied/not_satisfied_when_no_permissions_were_found_in_header (0.00s)
    --- PASS: Test_And_IsSatisfied/not_satisfied_when_at_least_one_permission_was_not_found_in_header (0.00s)
    --- PASS: Test_And_IsSatisfied/satisfied_when_all_permissions_were_found_in_header (0.00s)
=== RUN   Test_Or_IsSatisfied
=== RUN   Test_Or_IsSatisfied/not_satisfied_when_no_permissions_were_required
=== RUN   Test_Or_IsSatisfied/not_satisfied_when_no_permissions_were_found_in_header
=== RUN   Test_Or_IsSatisfied/satisfied_when_at_least_one_permission_was_found_in_header
--- PASS: Test_Or_IsSatisfied (0.00s)
    --- PASS: Test_Or_IsSatisfied/not_satisfied_when_no_permissions_were_required (0.00s)
    --- PASS: Test_Or_IsSatisfied/not_satisfied_when_no_permissions_were_found_in_header (0.00s)
    --- PASS: Test_Or_IsSatisfied/satisfied_when_at_least_one_permission_was_found_in_header (0.00s)
PASS
ok  	github.com/you/client/internal/pkg/authorisation	0.204s
```
