# Should my methods return structs or interfaces in Go?

Like most things in software design, it depends. And I cannot stress this enough, it really depends on the situation and what you wish to achieve.

Now, I hear some of you already screaming at the top of your lungs a very useful rule of thumb (or even proverb, if you will): *"Accept interfaces, return structs"*. That's very good advice, really, but there always are exceptions. No pattern is so flexible that it fits every situation.

With that said, I'd wager this rule does make sense more often than not, so let's have a look at the reasoning behind returning structs.

But first, some basics about how interfaces work in Go. If you're already familiar with them, feel free to skip ahead to the next section.

## Interfaces in Go

It's important to keep in mind that Go interfaces are not strictly bound to types. What I mean by this is that you never explicitly say that a type implements a particular interface. All that you need to do in order to implement an interface in Go is to implement the methods that satisfy the interface itself.

Think of an interface as a set of behaviors that are expected from a type. You don't usually see ducks with a label written "Duck", but you know it is one because it looks like a duck, it quacks like a duck, it has feathers like a duck, etc. You infer what it is from its behavior, and that is exactly what Go does in compile time.

If you're wondering, Go uses ["structural typing", not "duck typing"](https://en.wikipedia.org/wiki/Duck_typing#Comparison_with_other_type_systems), meaning this all happens in compile time, not run time. Remember that Go is statically typed.

*Example, please*. Take a look at the following `Person` interface:

```go
type Person interface {
    Name() string
    Age() int
}
```

A possible implementation of that interface is:

```go
type Gamer struct {
    name  string
    age   int
}

func NewGamer(name string, age int) *Gamer {
    return &Gamer{
        name: name,
        age:  age,
    }
}

func (g *Gamer) Name() string {
    return g.name
}

func (g *Gamer) Age() int {
    return g.age
}
```

Does Go understand that `Gamer` is a `Person`?

```go
func main() {
    var person Person = NewGamer("Jane Doe", 42)
    fmt.Println(person.Name())
}
```

The output from this program:

```sh
$ go run example.go
Jane Doe
```

Yes, Go understands that the `Gamer` type is a `Person` (or that it implements the `Person` interface)!

## When to return structs

Alright, now that we've refreshed our understanding of interfaces in Go, let's imagine that we wish to extend the `Gamer` behavior. We'll also say it has a slice of games:

```go
type Person interface {
    Name() string
    Age() int
}

type Game string

type Gamer struct {
    name  string
    age   int
    games []Game
}

func NewGamer(name string, age int) *Gamer {
    return &Gamer{
        name: name,
        age:  age,
    }
}

func (g *Gamer) Name() string {
    return g.name
}

func (g *Gamer) Age() int {
    return g.age
}

func (g *Gamer) Games() []Game {
    return g.games
}

func (g *Gamer) AddGame(game Game) {
    g.games = append(g.games, game)
}
```

Note that the program still works at this point as `Gamer` still implements `Person`, we just added more behavior and state to it.

Now, let's try simply modifying the `main` function we had before to make use of this new stuff we added:

```go
func main() {
    var person Person = NewGamer("Jane Doe", 42)
    fmt.Println(person.Name())
    person.AddGame("Metal Gear Solid 3")
}
```

We'll run into:

```sh
$ go run example.go
# command-line-arguments
./example.go.go:46:8: person.AddGame undefined (type Person has no field or method AddGame)
```

This is because the `person` variable is of type `Person`, which does not have the `AddGame()` method. If we create a variable `gamer` of type `*Gamer` we can access those new cool and shiny methods:

```go
func main() {
    var person Person = NewGamer("Jane Doe", 42)
    fmt.Println(person.Name())

    var gamer *Gamer = NewGamer("John Doe", 29)
    gamer.AddGame("Metal Gear Solid 3")
    fmt.Println(gamer.Games())
}
```

It runs fine:

```sh
$ go run example.go
Jane Doe
[Metal Gear Solid 3]
```

Now, what if the `NewGamer()` function actually returned the `Person` interface?

```go
func NewGamer(name string, age int) Person {
    return &Gamer{
        name: name,
        age:  age,
    }
}
```

In this scenario we'd never be able to make the following assignment:

```go
var gamer *Gamer = NewGamer("John Doe", 29)
```

Because `NewGamer` returns `Person`, not `*Gamer`. And even if we change `gamer`'s type to `Person` we'd be stuck with the behavior from the `Person` interface only. We'd lose access to the nice methods only the `Gamer` type has.

This is the main motivation behind returning structs instead of interfaces: it leaves it up to the caller if they want to use it as the interface or if they want to use it as the struct, and in the end that makes a lot of sense.

## When to return interfaces?

Let's expand a bit on our previous `Person` implementations. Now we'll also have a `Student`:

```go
type Student struct {
    name      string
    age       int
    knowledge int
}

func NewStudent(name string, age int) *Student {
    return &Student{
        name: name,
        age: age,
    }
}

func (s *Student) Name() string {
    return s.name
}

func (s *Student) Age() int {
    return s.age
}

func (s *Student) Study(hours int) {
    s.knowledge += hours
}
```

Becoming a person filled with knowledge isn't as simple as depicted here, but this model meets our requirements. Remember to [KISS](https://en.wikipedia.org/wiki/KISS_principle).

What if we needed to print the names of every people we know? In this case, an auxiliary function returning a slice of `Person` would make a lot of sense, because `Name()` is part of the `Person` interface:

```go
func PrintAllNames() {
    for _, person := range getAllPeople() {
        fmt.Printf("Name: %s\n", person.Name())
    }
}

func getAllPeople() []Person {
    var people []Person
    // logic implementation
    return people
}
```

This is simple, but one of the cases on which it's actually ok for a method or function to return an interface instead of the struct itself.

Without a function like this, you would need to create a separate function to get all instances of each implementation of `Person`, another for printing the names of these instances for each implementation of `Person` and a final one that orchestrates all of these.

```go
func PrintAllNames() {
    printAllGamerNames()
    printAllStudentNames()
}

func printAllGamerNames() {
    for _, gamer := range getAllGamers() {
        fmt.Printf("Name: %s\n", gamer.Name())
    }
}

func getAllGamers() []Gamer {
    var gamers []Gamer
    // logic implementation
    return gamers
}

func printAllStudentNames() {
    for _, student := range getAllStudents() {
        fmt.Printf("Name: %s\n", student.Name())
    }
}

func getAllStudents() []Student {
    var students []Student
    // logic implementation
    return students
}
```

See that nasty repetition of code? And worse: get ready to do this every time you add another implementation of `Person`. Not looking forward to it, I assume. And even worse: this sort of manual addition is error-prone and is likely to lead to bugs.

You could say "well, even if you returned a slice of `Person` you would still need to write the necessary logic to get all the instances of a new implementation of `Person`" and you would be correct. But notice how you do not need to write specific logic for printing the names, you only do that once in the previous version of the `PrintAllNames()` function:

```go
func PrintAllNames() {
    for _, person := range getAllPeople() {
        fmt.Printf("Name: %s\n", person.Name())
    }
}
```

**Summarizing**: returning structures is good most of the time because you leave it up to the caller to decide how to make the assignment and how to use it, but there are cases on which using an interface just makes more sense and can save you a lot of unnecessary writing.
