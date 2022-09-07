# Test Fixtures in Golang

## TL;DR

1. For fixtures that need to be accessed from a single package, stick them in a `testdata` folder within the package
2. For fixtures that must be kept outside the package or accessed by multiple packages, create a test-helpers package and anchor their paths with `runtime.Caller(0)`
3. For fixtures that are complex and have a lot of variations, use the [functional builder pattern](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)

## Overview

A test fixture is any kind of data needed for our tests to run. These can be things like files, binary data, mock database responses, or test structs.

In our test suite, we need a clear, consistent means of accessing our various test fixtures. While we could jam these into our tests as strings or inline structs, this is cumbersome and leads to potential mistakes. For example, let’s say we needed to test this (trivial) unmarshalling code:

```go
// /petstore/petstore.go

type Pet struct {
    ID   int64  `json:"id"`
    Name string `json:"name"`
    Tag  string `json:"tag,omitempty"`
}

func ReadPet(b []byte) (*Pet, error) {
    f := &Pet{}

    if err := json.Unmarshal(b, f); err != nil {
      return nil, err
    }

    return f, nil
}
```

We could embed the test JSON object inline like this:

```go
// /petstore/petstore_test.go

func TestReadPet_Inline(t *testing.T) {
    j := `
    {
      "id": 42,
      "name": "Foo",
      "tag": "dog"
    }
    `

    result, err := petstore.ReadPet([]byte(j))
    if err != nil {
        t.Errorf("error while unmashalling: %v", err)
    }

    if result.ID != 42 && result.Name != "foo" {
        t.Errorf("%v: output doesn't match expected result", result)
    }
}
```

While this may work, it looks clunky when put inline in our test. Moreover, our IDE doesn’t recognize this as a JSON object, which makes it harder to edit or catch formatting errors.

Binary data exacerbates this problem. While possible to keep it as strings via an encoding like base64, it obscures what the test data represents.

## Keeping Test Data Under `testdata`

The simplest solution is to keep all the test data in the same directory as the code itself under a special folder called `testdata`. This is a special folder ignored by the compiler, but still accessible for reading during tests. For example, we could keep the above sample JSON in its own file:

```json
/petstore/testdata/sample.json

{
    "id": 42,
    "name": "Foo",
    "tag": "dog"
}
```

Then read it during our test:

```go
// /petstore/petstore_test.go

func TestReadPet_TestData(t *testing.T) {
    b, err := ioutil.ReadFile("testdata/sample.json")
    if err != nil {
        t.Fatal(err)
    }

    result, err := petstore.ReadPet(b)
    ...
```

For more information, see Dave Cheney’s blog post about [test fixtures in go](https://dave.cheney.net/2016/05/10/test-fixtures-in-go).

## Accessing Test Data from a Separate Directory

Sometimes the files we need must be kept in a separate directory from our code. This may be because multiple packages' tests need to access them, or because things outside our code depend on them. For example, let’s say we have a JSON schema spec living in `/api/pet-schema.json`:

```json
/api/pet-schema.json

{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["id", "name"],
    "properties": {
        "id": {
            "type": "integer",
            "format": "int64"
        },
        "name": {
            "type": "string"
        },
        "tag": {
            "type": "string"
        }
    }
}
```

We could reference this file in our tests via a relative path such as `../api/pet-schema.json`. However, this couples our test to a specific project layout. If we changed the file path of either `pet-schema.json` or any of our modules, this could break our tests.

Another strategy is using a separate test helpers module that anchors the relative directory of our test data using `runtime.Caller(0)`. This returns information about the current call stack, including the filename running the `runtime.Caller` command. We could create a file called `/test/test_helpers.go`, then anchor the location of our schema relative to it:

```go
// /test/test_helpers.go

func GetPetSchema() (*jsonschema.Schema, error) {
    _, filename, _, ok := runtime.Caller(0)
    if !ok {
        return nil, ErrPathDiscovery
    }

    path := path.Join(path.Dir(filename), "..", "api", "pet-schema.json")
    ...
}
```

We can then use our test helpers module like any other module:

```go
// /petstore/schema_test.go

func TestSchemaValidation(t *testing.T) {
    schema, err := test.GetPetSchema()
    if err != nil {
        t.Fatal(err)
    }

    pet := petstore.Pet{
        ID:   42,
        Name: "foo",
    }

    errs := schema.Validate(context.TODO(), pet).Errs
    if len(*errs) > 0 {
        t.Errorf("got the following errors while validating data: %v", errs)
    }
}
```

Moreover, this works no matter where the calling code lives:

```go
// /petstore/subpackage/subpackage_test.go

func TestSubpackage(t *testing.T) {
    _, err := test.GetPetSchema()
    if err != nil {
        t.Fatal(err)
    }
}
```

## Creating Complex Fixtures using Functional Builders

Sometimes it’s not enough to have a single, static variation of a test object. For example, consider the following validation code:

```go
// /petstore/petstore.go

func (p *Pet) Validate() error {
    if p.ID < 0 {
        return fmt.Errorf("%w: 'id' must be greater than 0", ErrValidationFail)
    }

    if p.Name == "" {
        return fmt.Errorf("%w: 'name' must not be empty", ErrValidationFail)
    }

    if len(p.Name) > 100 {
        return fmt.Errorf("%w: 'name' must be less than 100 characters", ErrValidationFail)
    }

    if p.Tag != "" {
        validTags := []string{
            "cat",
            "dog",
        }
        hasValidTag := false

        for _, t := range validTags {
        if p.Tag == t {
            hasValidTag = true
        }
        }

        if !hasValidTag {
            return fmt.Errorf("%w: 'tag' must be one of the following: [%s]",
                ErrValidationFail, strings.Join(validTags, ","))
        }
    }

    return nil
}
```

Using a table-driven test, we could test many variations of the `petstore.Pet` object like so:

```go
// /petstore/petstore_test.go

func TestValidate_NoBuilder(t *testing.T) {
	tests := []struct {
		name    string
		pet     *petstore.Pet
		wantErr bool
	}{
		{
			name: "Validates a valid pet",
			pet: &petstore.Pet{
				Name: "Foo",
				ID:   64,
				Tag:  "dog",
			},
			wantErr: false,
		},
		{
			name: "Validates if tag is empty",
			pet: &petstore.Pet{
				Name: "Foo",
				ID:   64,
				Tag:  "",
			},
			wantErr: false,
		},
		{
			name: "Errors if ID less than 0",
			pet: &petstore.Pet{
				Name: "Foo",
				ID:   -200,
				Tag:  "dog",
			},
			wantErr: true,
		},
		{
			name: "Errors if name is empty",
			pet: &petstore.Pet{
				Name: "",
				ID:   64,
				Tag:  "dog",
			},
			wantErr: true,
		},
		{
			name: "Errors if name is greater than 100 characters",
			pet: &petstore.Pet{
				Name: strings.Repeat("A", 101),
				ID:   64,
				Tag:  "dog",
			},
			wantErr: true,
		},
		{
			name: "Errors if tag is invalid",
			pet: &petstore.Pet{
				Name: "Foo",
				ID:   64,
				Tag:  "turkey",
			},
			wantErr: true,
		},
	}

	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			if err := tt.pet.Validate(); (err != nil) != tt.wantErr {
				t.Errorf("error = %v, wantErr = %v", err, tt.wantErr)
			}
		})
	}
}
```

While this is workable for testing all variations of the `Pet` object, it’s difficult to determine what’s changing between each test. Most of the fields in each `pet` only contain default data for the sake of the test. In the "Errors if ID less than 0" test, we don’t particularly care what `Name` and `Tag` are: all we care about is that `ID` is set to a negative number.

To solve this, we can instead use Dave Cheney’s functional builder pattern. This allows us to use a sane default for all tests entries, while allowing each one to customize the resulting fixture as it sees fit. We can implement this pattern like so:

```go
// /petstore/petstore_test.go

func getTestPet(options ...petOption) *petstore.Pet {
	p := &petstore.Pet{
		Name: "Foo",
		ID:   42,
		Tag:  "dog",
	}

	for _, option := range options {
		option(p)
	}

	return p
}

type petOption func(*petstore.Pet)

func withTag(tag string) petOption {
	return func(p *petstore.Pet) {
		p.Tag = tag
	}
}

func withName(name string) petOption {
	return func(p *petstore.Pet) {
		p.Name = name
	}
}

func withID(id int64) petOption {
	return func(p *petstore.Pet) {
		p.ID = id
	}
}
```

Then our tests become simplified to this:

```go
// /petstore/petstore_test.go

func TestValidate_Builder(t *testing.T) {
	tests := []struct {
		name    string
		pet     *petstore.Pet
		wantErr bool
	}{
		{
			name:    "Validates a valid pet",
			pet:     getTestPet(),
			wantErr: false,
		},
		{
			name:    "Validates if tag is empty",
			pet:     getTestPet(withTag("")),
			wantErr: false,
		},
		{
			name:    "Errors if ID less than 0",
			pet:     getTestPet(withID(-200)),
			wantErr: true,
		},
		{
			name:    "Errors if name is empty",
			pet:     getTestPet(withName("")),
			wantErr: true,
		},
		{
			name:    "Errors if name is greater than 100 characters",
			pet:     getTestPet(withName(strings.Repeat("A", 101))),
			wantErr: true,
		},
		{
			name:    "Errors if tag is invalid",
			pet:     getTestPet(withTag("turkey")),
			wantErr: true,
		},
	}

	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			if err := tt.pet.Validate(); (err != nil) != tt.wantErr {
				t.Errorf("error = %v, wantErr = %v", err, tt.wantErr)
			}
		})
	}
}
```

Now it becomes immediately obvious what’s changing in each test! In the "Errors if ID less than 0" test, the only thing we see is that our default pet’s ID is set to -200 via `withID(-200)`. This helps improve test readability and maintainability.

### References

All example code can be found here: https://gitlab.com/hackandsla.sh/blog-examples/-/tree/master/2020-11-23-golang-test-fixtures

Source: https://hackandsla.sh/posts/2020-11-23-golang-test-fixtures/