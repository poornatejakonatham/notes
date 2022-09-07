# Variables

## What is a variable?

Variable is the **name given to a memory location** to store a value of a specific type. There are various syntaxes to declare variables in Go. Let's look at them one by one.

## Declaring a single variable

`var name type` is the syntax to declare a single variable.

```go
package main

import "fmt"

func main() {
    var age int // variable declaration
    fmt.Println("My age is", age)
}
```
