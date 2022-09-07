# Developing a Web Application in Go using the Layered Architecture

Writing a web-server using Go is very simple. But the challenge comes when the code has to be testable, structured, clean and maintainable.

Below, we are writing a simple server that stores and retrieves data from MySQL.

```go
package main
import (
   "bytes"
   "net/http"
   "net/http/httptest"
   "os"
   "reflect"
   "testing"
)
func Test_Handler(t *testing.T) {
   // initializing db
   conf := mysqlConfig{
      host:     os.Getenv("SQL_HOST"),
      user:     os.Getenv("SQL_USER"),
      password: os.Getenv("SQL_PASSWORD"),
      port:     os.Getenv("SQL_PORT"),
      db:       os.Getenv("SQL_DB"),
   }
   var err error
   db, err = connectToMySQL(conf)
   if err != nil {
      t.Errorf("could not connect to sql, err:%v", err)
   }


   testcases := []struct {
      // input
      method string
      body   []byte
      // output
      expectedStatusCode int
      expectedResponse   []byte
   }{
      {"GET", nil, http.StatusOK, []byte(`[{"Name":"Hippo","Age":10}]`)},
      {"POST", []byte(`{"Name":"Dog","Age":12}`), http.StatusOK, []byte(`success`)},
      {"DELETE", nil, http.StatusMethodNotAllowed, nil},
   }


   for _, v := range testcases {
      req := httptest.NewRequest(v.method, "/animal", bytes.NewReader(v.body))
      w := httptest.NewRecorder()


      h := http.HandlerFunc(handler)
      h.ServeHTTP(w, req)


      if w.Code != v.expectedStatusCode {
         t.Errorf("Expected %v\tGot %v", v.expectedStatusCode, w.Code)
      }


      expected := bytes.NewBuffer(v.expectedResponse)
      if !reflect.DeepEqual(w.Body, expected) {
         t.Errorf("Expected %v\tGot %v", expected.String(), w.Body.String())
      }
   }
}
```

The code above needs **refinement**. Why?

- One function is doing more than one thing. When this **application grows**, it becomes **hard to manage and test**.
- There is **no way to test the handlers without the database**. The datastore is not injected and it is a global variable, thus making it **hard to mock the database in the handler**.

In an ideal scenario, the handler should not depend on the underlying datastore. A good approach to tackle this problem is to follow a layered architecture. Each layer will do only one thing.

## The Layered Architecture

The three independent layers are **delivery**, **use-case** and **datastore**.

### Delivery Layer:

The delivery layer will receive the request and parse anything that is required from the request. It calls the use case layer, ensures that the response is the required format and writes it to the response writer.

### Use-Case Layer:

The use case layer does the business logic that is required for the application. This layer will communicate with the datastore layer. It takes whatever it needs from the delivery layer and then calls the datastore layer. Before and after calling the datastore layer, it applies the business logic that is required.

### Datastore Layer:

The datastore stores the data. It can be any data storage. The use case layer is the only layer that communicates with datastore. This way each layer can be tested independently without depending on the other.

Since each layer is independent of each other, if the application grows to have gRPC support, only the delivery layer will change. Datastore and use case layer will remain the same. Even when there is a change in datastore, the entire application need not change. Only the datastore layer will change. This way, it is easy to isolate any bugs, maintain the code and grow the application.

**Note**: If you don’t have any business logic you can skip the use-case layer and have only delivery and datastore layer.

**Each layer will communicate with each other through interfaces.**

Refer to the following schema.

**datastore/animal.sql**

```sql
DROP DATABASE IF EXISTS animal;
CREATE DATABASE animal;
USE animal;

CREATE TABLE animals(
    id int NOT NULL AUTO_INCREMENT,
    name varchar(50),
    age int,

    PRIMARY KEY(id)
);

INSERT INTO animals VALUES (1, 'Hippo', 10);
INSERT INTO animals VALUES (2, 'Ele', 20);
```

For better structure, including a package **entities** and **driver**. **entities** will maintain all the structs that represent each entity in the application.
**driver** will have the functions to connect to datastores.

**entities/animal.go**

```go
package entities

type Animal struct {
     ID   int
     Name string
     Age  int
}
```

**driver/mysql.go**

```go
package driver

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

type MySQLConfig struct {
	Host     string
	User     string
	Password string
	Port     string
	Db       string
}

// ConnectToMySQL takes mysql config, forms the connection string and connects to mysql.
func ConnectToMySQL(conf MySQLConfig) (*sql.DB, error) {
	connectionString := fmt.Sprintf("%v:%v@tcp(%v:%v)/%v", conf.User, conf.Password, conf.Host, conf.Port, conf.Db)

	db, err := sql.Open("mysql", connectionString)
	if err != nil {
		return nil, err
	}

	return db, nil
}
```

The directory structure:

```sh
├── datastore
│   ├── animal
│   │   ├── mysql.go
│   │   ├── mysql_test.go
│   ├── interface.go
│   │
├── delivery
│   ├── animal
│   │   └── http.go
│   │   └── http_test.go
│   │
├── driver
│   ├── mysql.go
│   │
├── entities
│   ├── animal.go
│   │
├── animal.sql
│   │
├── main.go
```

## Datastore Layer

This layer will use MySQL to store and retrieve the data related to `Animal`.

**datastore/interface.go**

```go
package datastore

import "web-app/entities"

type Animal interface {
    Get(id int) ([]entities.Animal, error)
    Create(entities.Animal) (entities.Animal, error)
}
```

**datastore/animal/mysql.go**

```go
package animal

import (
	"database/sql"
	"web-app/entities"
)

type AnimalStorer struct {
	db *sql.DB
}

func New(db *sql.DB) AnimalStorer {
	return AnimalStorer{db: db}
}

func (a AnimalStorer) Get(id int) ([]entities.Animal, error) {
	var (
		rows *sql.Rows
		err  error
	)

	if id != 0 {
		rows, err = a.db.Query("SELECT * FROM animals where id = ?", id)
	} else {
		rows, err = a.db.Query("SELECT * FROM animals")
	}

	if err != nil {
		return nil, err
	}

	defer rows.Close()

	var animals []entities.Animal

	for rows.Next() {
		var a entities.Animal
		_ = rows.Scan(&a.ID, &a.Name, &a.Age)
		animals = append(animals, a)
	}
	return animals, nil
}

func (a AnimalStorer) Create(animal entities.Animal) (entities.Animal, error) {
	res, err := a.db.Exec("INSERT INTO animals (name,age) VALUES(?,?)", animal.Name, animal.Age)
	if err != nil {
		return entities.Animal{}, err
	}

	id, _ := res.LastInsertId()
	animal.ID = int(id)

	return animal, nil
}
```

**datastore/animal/mysql_test.go**

```go
package animal

import (
	"database/sql"
	"os"
	"reflect"
	"testing"
	"web-app/driver"
	"web-app/entities"
)

func initializeMySQL(t *testing.T) *sql.DB {
	conf := driver.MySQLConfig{
		Host:     os.Getenv("SQL_HOST"),
		User:     os.Getenv("SQL_USER"),
		Password: os.Getenv("SQL_PASSWORD"),
		Port:     os.Getenv("SQL_PORT"),
		Db:       os.Getenv("SQL_DB"),
	}

	var err error
	db, err := driver.ConnectToMySQL(conf)
	if err != nil {
		t.Errorf("could not connect to sql, err:%v", err)
	}

	return db
}

func TestDatastore(t *testing.T) {
	db := initializeMySQL(t)
	a := New(db)
	testAnimalStorer_Get(t, a)
	testAnimalStorer_Create(t, a)

}

func testAnimalStorer_Create(t *testing.T, db AnimalStorer) {
	testcases := []struct {
		req      entities.Animal
		response entities.Animal
	}{
		{entities.Animal{Name: "Hen", Age: 1}, entities.Animal{3, "Hen", 1}},
		{entities.Animal{Name: "Pig", Age: 2}, entities.Animal{4, "Pig", 2}},
	}
	for i, v := range testcases {
		resp, _ := db.Create(v.req)

		if !reflect.DeepEqual(resp, v.response) {
			t.Errorf("[TEST%d]Failed. Got %v\tExpected %v\n", i+1, resp, v.response)
		}
	}
}

func testAnimalStorer_Get(t *testing.T, db AnimalStorer) {
	testcases := []struct {
		id   int
		resp []entities.Animal
	}{
		{0, []entities.Animal{{1, "Hippo", 10}, {2, "Ele", 20}}},
		{1, []entities.Animal{{1, "Hippo", 10}}},
	}
	for i, v := range testcases {
		resp, _ := db.Get(v.id)

		if !reflect.DeepEqual(resp, v.resp) {
			t.Errorf("[TEST%d]Failed. Got %v\tExpected %v\n", i+1, resp, v.resp)
		}
	}
}
```

## Delivery Layer

This will receive the HTTP request and validate the filter for GET request and validate the request body in POST request.

This layers needs the store layer to be injected since it communicates with datastore layer to store and retrieve data.

**delivery/animal/http.go**

```go
package animal

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"strconv"
	"web-app/datastore"
	"web-app/entities"
)

type AnimalHandler struct {
	datastore datastore.Animal
}

func New(animal datastore.Animal) AnimalHandler {
	return AnimalHandler{datastore: animal}
}

func (a AnimalHandler) Handler(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		a.get(w, r)
	case http.MethodPost:
		a.create(w, r)
	default:
		w.WriteHeader(http.StatusMethodNotAllowed)
	}
}

func (a AnimalHandler) get(w http.ResponseWriter, r *http.Request) {
	id := r.URL.Query().Get("id")

	i, err := strconv.Atoi(id)
	if err != nil {
		_, _ = w.Write([]byte("invalid parameter id"))
		w.WriteHeader(http.StatusBadRequest)

		return
	}

	resp, err := a.datastore.Get(i)
	if err != nil {
		_, _ = w.Write([]byte("could not retrieve animal"))
		w.WriteHeader(http.StatusInternalServerError)

		return
	}

	body, _ := json.Marshal(resp)
	_, _ = w.Write(body)
}

func (a AnimalHandler) create(w http.ResponseWriter, r *http.Request) {
	var animal entities.Animal

	body, _ := ioutil.ReadAll(r.Body)

	err := json.Unmarshal(body, &animal)
	if err != nil {
		fmt.Println(err)
		_, _ = w.Write([]byte("invalid body"))
		w.WriteHeader(http.StatusBadRequest)

		return
	}

	resp, err := a.datastore.Create(animal)
	if err != nil {
		_, _ = w.Write([]byte("could not create animal"))
		w.WriteHeader(http.StatusInternalServerError)

		return
	}

	body, _ = json.Marshal(resp)
	_, _ = w.Write(body)
}
```

This handler can be tested without depending on the datastore. You can mock the responses from the datastore and write the unit tests, testing only the handlers.

**delivery/animal/http_test.go**

```go
package animal

import (
	"bytes"
	"errors"
	"net/http"
	"net/http/httptest"
	"reflect"
	"testing"
	"web-app/entities"
)

func TestAnimalHandler_Handler(t *testing.T) {
	testcases := []struct {
		method             string
		expectedStatusCode int
	}{
		{"GET", http.StatusOK},
		{"POST", http.StatusOK},
		{"DELETE", http.StatusMethodNotAllowed},
	}

	for _, v := range testcases {
		req := httptest.NewRequest(v.method, "/animal", nil)
		w := httptest.NewRecorder()

		a := New(mockDatastore{})
		a.Handler(w, req)

		if w.Code != v.expectedStatusCode {
			t.Errorf("Expected %v\tGot %v", v.expectedStatusCode, w.Code)
		}
	}
}

func TestAnimalGet(t *testing.T) {
	testcases := []struct {
		id       string
		response []byte
	}{
		{"1", []byte("could not retrieve animal")},
		{"1a", []byte("invalid parameter id")},
		{"2", []byte(`[{"ID":2,"Name":"Dog","Age":8}]`)},
		{"0", []byte(`[{"ID":1,"Name":"Ken","Age":23},{"ID":2,"Name":"Dog","Age":8}]`)},
	}

	for i, v := range testcases {
		req := httptest.NewRequest("GET", "/animal?id="+v.id, nil)
		w := httptest.NewRecorder()

		a := New(mockDatastore{})

		a.get(w, req)

		if !reflect.DeepEqual(w.Body, bytes.NewBuffer(v.response)) {
			t.Errorf("[TEST%d]Failed. Got %v\tExpected %v\n", i+1, w.Body.String(), string(v.response))
		}
	}
}

func TestAnimalPost(t *testing.T) {
	testcases := []struct {
		reqBody  []byte
		respBody []byte
	}{
		{[]byte(`{"Name":"Hen","Age":12}`), []byte(`could not create animal`)},
		{[]byte(`{"Name":"Maggie","Age":10}`), []byte(`{"ID":12,"Name":"Maggie","Age":10}`)},
		{[]byte(`{"Name":"Maggie","Age":"10"}`), []byte(`invalid body`)},
	}
	for i, v := range testcases {
		req := httptest.NewRequest("GET", "/animal", bytes.NewReader(v.reqBody))
		w := httptest.NewRecorder()

		a := New(mockDatastore{})

		a.create(w, req)

		if !reflect.DeepEqual(w.Body, bytes.NewBuffer(v.respBody)) {
			t.Errorf("[TEST%d]Failed. Got %v\tExpected %v\n", i+1, w.Body.String(), string(v.respBody))
		}
	}
}

type mockDatastore struct{}

func (m mockDatastore) Get(id int) ([]entities.Animal, error) {
	if id == 1 {
		return nil, errors.New("db error")
	} else if id == 2 {
		return []entities.Animal{{2, "Dog", 8}}, nil
	}

	return []entities.Animal{{1, "Ken", 23}, {2, "Dog", 8}}, nil
}

func (m mockDatastore) Create(animal entities.Animal) (entities.Animal, error) {
	if animal.Age == 12 {
		return entities.Animal{}, errors.New("db error")
	}

	return entities.Animal{12, "Maggie", 10}, nil
}
```

## main.go

This will create the server. It connects to db by fetching the config from environment and then injects this db to the delivery layer.

**main.go**

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"web-app/datastore/animal"
	handlerAnimal "web-app/delivery/animal"
	"web-app/driver"
)

func main() {
	// get the mysql configs from env:
	conf := driver.MySQLConfig{
		Host:     os.Getenv("SQL_HOST"),
		User:     os.Getenv("SQL_USER"),
		Password: os.Getenv("SQL_PASSWORD"),
		Port:     os.Getenv("SQL_PORT"),
		Db:       os.Getenv("SQL_DB"),
	}
	var err error

	db, err := driver.ConnectToMySQL(conf)
	if err != nil {
		log.Println("could not connect to sql, err:", err)
		return
	}

	datastore := animal.New(db)
	handler := handlerAnimal.New(datastore)

	http.HandleFunc("/animal", handler.Handler)
	fmt.Println(http.ListenAndServe(":9000", nil))
}
```

As you can see, with this layered architecture it becomes easy to maintain the code.

- When there is a bug, it becomes easier to isolate and fix it.
- When the application grows and you decide to have another datastore, let us say you want to include caching then only the datastore layer will change without touching the delivery or use-case layer.
- Due to the independent layers, writing unit tests is simpler.

Happy Going.