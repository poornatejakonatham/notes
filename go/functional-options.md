# Golang functional options with errors

There will be cases where an option you want to use might return an error. Based on the return, you would either continue or terminate the process. This simple example will help you to handle errors coming from functional options in Golang.

**car.go**

```go
package car

import "fmt"

type option func() (func(*Car), error)

func success(opt func(*Car)) option {
	return func() (func(*Car), error) {
		return opt, nil
	}
}

func failure(err error) option {
	return func() (func(*Car), error) {
		return nil, err
	}
}

type Car struct {
	make  string
	model string
}

func New(options ...option) (*Car, error) {
	car := &Car{}

	for _, option := range options {
		opt, err := option()
		if err != nil {
			return nil, err
		}

		opt(car)
	}

	return car, nil
}

func WithMake(val string) option {
	if val == "ford" {
		return failure(fmt.Errorf("make: we do not produce %s", val))
	}

	return success(func(c *Car) {
		c.make = val
	})
}

func WithModel(val string) option {
	return success(func(c *Car) {
		c.model = val
	})
}
```

## Test

```go
package main

import (
	"log"

	"practise/car"
)

func main() {
	// Without any options.
	car1, err := car.New()
	if err != nil {
		log.Fatalln(err)
	}
	log.Printf("%+v\n", car1)

	// With valid options.
	car2, err := car.New(car.WithMake("bmw"), car.WithModel("m3"))
	if err != nil {
		log.Fatalln(err)
	}
	log.Printf("%+v\n", car2)

	// With valid options.
	car3, err := car.New(car.WithMake("ford"), car.WithModel("mustang"))
	if err != nil {
		log.Fatalln(err)
	}
	log.Printf("%+v\n", car3)
}
```

```bash
$ go run .
2020/06/09 00:00:08 &{make: model:}
2020/06/09 00:00:08 &{make:bmw model:m3}
2020/06/09 00:00:08 make: we do not produce ford
exit status 1
```
