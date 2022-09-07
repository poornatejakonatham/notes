# Implementing the Functional Options or Options Pattern in Golang

Assume that you have a function that returns an object with/without a default values attached to its fields. It is like a constructor. What about if you want to override the default values when you first create the object instance. This is where the **Functional Options** come handy. Your function takes extra arguments in a certain way and structure. These arguments then override fields that are only relevant to them, not all the others! See example below.

```go
package main

import (
	"fmt"
)

type Car struct {
	make   string
	model  string
	colour string
}

func NewCar() *Car {
	return &Car{
		make:   "BMW",
		model:  "3.18",
		colour: "Silver",
	}
}

func main() {
	car := NewCar()

	fmt.Printf("%+v\n", car)
}
```

This is what we get, which is useless for now.

```bash
&{make:BMW model:3.18 colour:Silver}
```

## Functional option

Let's apply the **Options Pattern** to make the `Car` struct open for modification at initialisation.

```go
package main

import (
	"fmt"
)

type Car struct {
	make   string
	model  string
	colour string
}

type Option func(*Car)

func Make(v string) Option {
	return func(c *Car) {
		c.make = v
	}
}
func Model(v string) Option {
	return func(c *Car) {
		c.model = v
	}
}
func Colour(v string) Option {
	return func(c *Car) {
		c.colour = v
	}
}

func NewCar(opts... Option) *Car {
	c := &Car{
		make:   "BMW",
		model:  "3.18",
		colour: "Silver",
	}

	for _, opt := range opts {
		opt(c)
	}

	return c
}

func main() {
	car1 := NewCar()
	fmt.Printf("%+v\n", car1)

	car2 := NewCar(Make("Mercedes"), Model("SEL 500"))
	fmt.Printf("%+v\n", car2)
}
```

This is what we get now which is highly customisable. We left the colour intact.

```bash
&{make:BMW model:3.18 colour:Silver}
&{make:Mercedes model:SEL 500 colour:Silver}
```
