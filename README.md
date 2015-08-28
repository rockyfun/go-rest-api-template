# go-rest-api-template

*WORK IN PROGRESS*

Reusable template for building REST Web Services in Golang. Uses gorilla/mux as a router/dispatcher and Negroni as a middleware handler.

## Introduction

### Why?

After writing many REST APIs with Java Dropwizard, Node.js/Express and Go, I wanted to distil my lessons learned into a reusable template for writing REST APIs, in the Go language (my favourite).

It's mainly for myself. I don't want to keep reinventing the wheel and just want to get the foundation of my REST API 'ready to go' so I can focus on the business logic and integration with other systems and data stores.

Just to be clear: this is not a framework, library, package or anything like that. This tries to use a couple of very good Go packages and libraries that I like and cobbled them together.

The main ones are:

* [gorilla/mux](http://www.gorillatoolkit.org/pkg/mux) for routing
* [negroni](https://github.com/codegangsta/negroni) as a middleware handler
* [testify](https://github.com/stretchr/testify) for writing easier test assertions
* [godep](https://github.com/tools/godep) for dependency management
* [render](https://github.com/unrolled/render) for HTTP response rendering

Whilst working on this I've tried to write up as much as my thought process as possible. Everything from the design of the API and routes, some details of the Go code like JSON formatting in structs and my thoughts on testing. However, if you feel that there is something missing, send a PR, raise an issue or contact me on twitter [@leeprovoost](https://twitter.com/leeprovoost).

### Knowledge of Go

If you're new to programming in Go, I would highly recommend you to read the following two resources:

* [A Tour of Go](https://tour.golang.org/welcome/1)
* [Effective Go](https://golang.org/doc/effective_go.html)

You can work your way through those in two to three days. I wouldn't advise buying any books right now. I unfortunately did and I have yet to open them. If you really want to get some books, there is a [github](https://github.com/dariubs/GoBooks) repo that tries to list the main ones.

### Development tools

I've tried many different IDEs and it seems like it's a common Go newbie [frustration](https://groups.google.com/forum/#!topic/golang-nuts/6ZgrZsPzHr0). Coming from Java, I've tried "proper" IDEs like [LiteIDE](https://github.com/visualfc/liteide) and IntelliJ with the [golang plugin](https://github.com/go-lang-plugin-org/go-lang-idea-plugin) but never really fell in love with those.

I am a big fan of SublimeText that I've been using extensively for Node.js development so I set up my environment with:

* [Sublime Text 3](http://www.sublimetext.com/3) editor
* [GoSublime](https://github.com/DisposaBoy/GoSublime) plugin
* [GitGutter](https://github.com/jisaacks/GitGutter) to see git diffs in gutter

I have tried Atom with the [go-plus](https://github.com/joefitzgerald/go-plus) in the past, but was never good enough to push Sublime Text aside for me. I have to admit that I haven't tried it in a while and Atom is maturing rapidly so please feel free to give it a go.

### Manage dependencies

In order to manage package dependencies, we're using the [Godep](https://github.com/tools/godep) package tool.

Install Godep on your system:

```
go get github.com/tools/godep
```

When you add new packages, just use the standard `go get package` first, edit your code, then run the `godep save`. It will add an extra entry to `Godeps/Godeps.json`.

### Live Code Reloading

TO DO

### How to run

`go run main.go` works fine if you have a single file you're working on, but once you have multiple files you'll have to start using the proper go build tool and run the compiled executable.

```
go build && ./go-rest-api-template
```

## Code deep dive

### Code Structure

Main server file (bootstrapping of http server and router):

```
main.go
```

Route handlers:

```
handlers.go
```

Data model descriptions:

```
passport.go
user.go
```

Mock database with operations:

```
database.go
```

Tests:

```
database_test.go
```

Third-party packages:

```
/Godeps
```

TO DO talk about the layers of the applications

### main.go

TO DO

### Data model

We are going to use a travel Passport for our example. I've chosen Id as the unique key for the passport because (in the UK), passport book numbers these days have a unique 9 character field length (e.g. 012345678). A passport belongs to a user and a user can have one or more passports.

```
type User struct {
  Id              int    `json:"id"`
  FirstName       string `json:"firstName"`
  LastName        string `json:"lastName"`
  DateOfBirth     string `json:"dateOfBirth"`
  LocationOfBirth string `json:"locationOfBirth"`
}

type Passport struct {
  Id           string `json:"id"`
  DateOfIssue  string `json:"dateOfIssue"`
  DateOfExpiry string `json:"dateOfExpiry"`
  Authority    string `json:"authority"`
  UserId       int    `json:"userId"`
}
```

The first time you create a struct, you may not be aware that uppercasing and lowercasing your field names have a meaning in Go. It's similar to public and private members in Java. Uppercase = public, lowercase = private. There are some good discussions on Stackoverflow about [this](http://stackoverflow.com/questions/21825322/why-golang-cannot-generate-json-from-struct-with-front-lowercase-character). The gist is that field names that start with a lowercase letter will not be visible to json.Marshal.

You may not want to expose your data to the consumer of your web service in this format, so you can override the way your fields are marshalled by adding ``json:"firstName"`` to each field with the desired name. I admit that in the past I had the habit of using underscores for my json field names, e.g. `first_name`. However after reading [this](http://www.slideshare.net/stormpath/rest-jsonapis) excellent presentation on API design, I got reminded that the JS in JSON stands for JavaScript and in the JavaScript world, it's common to use camelCasing so the preffered way of writing the same fieldname would be: `firstName`.

### Operations on our (mock) data

I wanted to create a template REST API that didn't depend on a database, so started with a simple in-memory database that we can work with. The good thing is that this will be the start of a so-called data access layer that abstracts away the underlying data store. We can achieve that by starting with creating an interface (which is a good practice in Go anyway):

```
type DataStore interface {
  List() map[string]User
  Get(i int) (User, error)
  Add(u User) User
  Update(u User) (User, error)
  Delete(i int) (bool, error)
}
```

This allows us to define a set of operations on the data as a contract, without people having to worry about the actual implementation of how the data is stored and accessed. I've added the basic operations to list, retrieve, create, update and delete data, so the standard CRUD-style operations (accepting that CRUD has some subtle differences with REST).

Let's have a look at the type signature of the `Get` operation:

```
Get(i int) (User, error)
```

What this tells us is that it is expecting an integer as an argument (which will be the User id in our case), and returns a pair of values: a user object and an error object. Returning pairs of values is a nice Go feature and is often used to return information about errors.

An example of how this could be used is the following:

```
  user, err := db.Get(uid)
  if err == nil {
    Render.JSON(w, http.StatusOK, user)
  } else {
    Render.JSON(w, http.StatusNotFound, err)
  }
```

We check whether the error object is nil. If it is, then we return a HTTP 200 OK, if not then we return HTTP 404 NOT FOUND. Let's go into more detail when we talk about our API handlers.

Let's have a look at the actual mock in-memory database. We need to create a Database struct that will hold the data:

```
type Database struct {
  UserList  map[int]User
  MaxUserId int
}
```
The UserList will hold a list of User structs and the MaxUserId holds the latest used integer. MaxUserId mimicks the behaviour of an autogenerated ID in conventional databases.

We will now create a global database variable so that it's accessible across our whole API:

```
var db *Database
```

In order to make it a bit more useful, we will initialise it with some user objects.Luckily, we can make use of the `init` function that gets automatically called when you start the application. This init() function will be in our `main.go` file when you start up the server:

```
func init() {
  list := make(map[int]User)
  list[0] = User{0, "John", "Doe", "31-12-1985", "London"}
  list[1] = User{1, "Jane", "Doe", "01-01-1992", "Milton Keynes"}
  db = &Database{list, 1}
}
```

We now need to implement the various methods from our DataStore interface.

TO DO document implemented methods

### API routes and route handlers

Now that we have defined the data access layer, we need to translate that to a REST interface:

* Retrieve a list of all users: `GET /users` -> The `GET` just refers to the HTTP action you would use. If you want to test this in the command line, then you can use curl: `curl -X GET http://localhost:3009/users` or `curl -X POST http://localhost:3009/users`
* Retrieve the details of an individual user: `GET /users/{uid}` -> {uid} allows us to create a variable, named uid, that we can use in our code. An example of this url would be `GET /users/1`
* Create a new user: `POST /users`
* Update a user: `PUT /users/{uid}`
* Delete a user: `DELETE /users/{uid}`

We now need to do the same for handling passports. Don't forget that a passport belongs to a user, so to retrieve a list of all passports for a given user, we would use `GET /users/{uid}/passports`.

When we want to retrieve an specific passport, we don't need to prefix the route with `/users/{uid}` anymore because we know exactly which passport we want to retrieve. So, instead of `GET /users/{uid}/passports/{pid}`, we can just use `GET /passports/{pid}`.

Once you have the API design sorted, it's just a matter of creating the code that gets called when a specific route is hit. We implement those with Handlers.

```golang
  router.HandleFunc("/users", UsersHandler).Methods("GET")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("GET")
  router.HandleFunc("/users", UsersHandler).Methods("POST")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("PUT")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("DELETE")

  router.HandleFunc("/users/{uid}/passports", PassportsHandler).Methods("GET")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("GET")
  router.HandleFunc("/users/{uid}/passports", PassportsHandler).Methods("POST")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("PUT")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("DELETE")
```

Last but not least, we want to handle two special cases:

```golang
  router.HandleFunc("/", HomeHandler)
  router.HandleFunc("/healthcheck", HealthcheckHandler).Methods("GET")
```

When someone hits our API, without a specified route, then we can handle that with either a standard 404 (not found), or any other type of feedback.

We also want to set up a health check that monitoring tools like [Sensu](https://sensuapp.org/) can call: `GET /healthcheck`. The health check route can return a 204 OK when the serivce is up and running, including some extra stats. A 204 means "Hey, I got your request, all is fine and I have nothing else to say". It essentially tells your client that there is no body content.

```
func HealthcheckHandler(w http.ResponseWriter, req *http.Request) {
  Render.Text(w, http.StatusNoContent, "")
}
```

This health check is very simple. It just checks whether the service is up and running, which can be useful in a build and deployment pipelines where you can check whether your newly deployed API is running (as part of a smoke test). More advanced health checks will also check whether it can reach the database, message queue or anything else you'd like to check. Trust me, your DevOps colleagues will be very grateful for this. (Don't forget to change your HTTP status code to 200 if you want to report on the various components that your health check is checking.)

Let's have a look at interacting with our data. Returning a list of users is quite easy, it's just showing the UserList:

```
func ListUsersHandler(w http.ResponseWriter, req *http.Request) {
  Render.JSON(w, http.StatusOK, db.List())
}
```

BTW, notice the `Render.JSON`? That's part of `"github.com/unrolled/render"` and allows us to render JSON output when we send data back to the client.

So, this will return the following to the client:

```
{
    "users": [
        {
            "dateOfBirth": "01-01-1992",
            "firstName": "Jane",
            "id": 1,
            "lastName": "Doe",
            "locationOfBirth": "Milton Keynes"
        },
        {
            "dateOfBirth": "31-12-1985",
            "firstName": "John",
            "id": 0,
            "lastName": "Doe",
            "locationOfBirth": "London"
        }
    ]
}
```

It may surprise you that we are returning a JSON object that holds an array with multiple JSON objects, rather than an array with multiple JSON objects, as seen in the example below:

```
{
    [
        {
            "dateOfBirth": "01-01-1992",
            "firstName": "Jane",
            "id": 1,
            "lastName": "Doe",
            "locationOfBirth": "Milton Keynes"
        },
        {
            "dateOfBirth": "31-12-1985",
            "firstName": "John",
            "id": 0,
            "lastName": "Doe",
            "locationOfBirth": "London"
        }
    ]
}
```

They're both valid and there are lots of views and opinions (as always in developer / architecture communities!), but the reason why I prefer to wrap the array in a JSON object is because later on we can easily add more data without causing significant changes to the client. What if we want to add the concept of pagination to our API?

Example:

```
{
    "offset": 0,
    "limit":  25,
    "users":  [
        {
            "dateOfBirth": "01-01-1992",
            "firstName": "Jane",
            "id": 1,
            "lastName": "Doe",
            "locationOfBirth": "Milton Keynes"
        },
        {
            "dateOfBirth": "31-12-1985",
            "firstName": "John",
            "id": 0,
            "lastName": "Doe",
            "locationOfBirth": "London"
        }
    ]
}
```

Another example is the retrieval of a specific object:

```
func GetUserHandler(w http.ResponseWriter, req *http.Request) {
  vars := mux.Vars(req)
  uid, _ := strconv.Atoi(vars["uid"])
  user, err := db.Get(uid)
  if err == nil {
    Render.JSON(w, http.StatusOK, user)
  } else {
    Render.JSON(w, http.StatusNotFound, err)
  }
}
```

This reads the uid variable from the route (`/users/{uid}`), converts the string to an integer and then looks up the user in our UserList by ID. If the user does not exit, we return a 404 and an error object. If the user exists, we return a 200 and a JSON object with the user.

Example:

```
{
    "dateOfBirth": "01-01-1992",
    "firstName": "Jane",
    "id": 1,
    "lastName": "Doe",
    "locationOfBirth": "Milton Keynes"
}
```

We add the attributes of the user object to the root of the JSON response, rather than wrapping it up in an explicit JSON object. I can quite easily add extra data to the response, without breaking the existing data.

### Testing your routes with curl commands

Let's start with some simple curl tests. Open your terminal and try the following curl commands.

Retrieve a list of users:

```
curl -X GET http://localhost:3009/users | python -mjson.tool
```

That should result in the following result:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   216  100   216    0     0  14466      0 --:--:-- --:--:-- --:--:-- 15428
{
    "users": [
        {
            "dateOfBirth": "01-01-1992",
            "firstName": "Jane",
            "id": 1,
            "lastName": "Doe",
            "locationOfBirth": "Milton Keynes"
        },
        {
            "dateOfBirth": "31-12-1985",
            "firstName": "John",
            "id": 0,
            "lastName": "Doe",
            "locationOfBirth": "London"
        }
    ]
}
```

The `| python -mjson.tool` at the end is for pretty printing (formatting). It essentially tells to pipe the output of the curl command to the SJON formatting tool. If we only typed `curl -X GET http://localhost:3009/users` then we'd have something like this:

```
{"users":[{"id":0,"firstName":"John","lastName":"Doe","dateOfBirth":"31-12-1985","locationOfBirth":"London"},{"id":1,"firstName":"Jane","lastName":"Doe","dateOfBirth":"01-01-1992","locationOfBirth":"Milton Keynes"}]}
```

So not that easy to read as the earlier nicely formatted example.

Get a specific user:

```
curl -X GET http://localhost:3009/users/0 | python -mjson.tool
```

Results in:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   104  100   104    0     0   6625      0 --:--:-- --:--:-- --:--:--  6933
{
    "dateOfBirth": "01-01-1992",
    "firstName": "Jane",
    "id": 1,
    "lastName": "Doe",
    "locationOfBirth": "Milton Keynes"
}
```

Adding a user:

```
TO DO
```

Deleting a user:

```
TO DO
```

Updating an existing user:

```
TO DO
```

### Testing

There are lots of opinions on testing, how much you should be testing, which layers of your applications, etc. When I'm working with micro services, I tend to focus on two types of tests to start with: testing the data access layer and testing the actual HTTP service.

In this example, we want to test the List, Add, Get, Update and Delete operations on our in-memory document database. The data access code is stored in the `database.go` file, so following Go convention we will create a new file called `database_test.go`.

In the `database_test.go` file, we have two sections:

First, we're going to create a common initialiser, this code sets up our database, inserts a couple of test records and then fires off the tests:

```
func TestMain(m *testing.M) {
  list := make(map[int]User)
  list[0] = User{0, "John", "Doe", "31-12-1985", "London"}
  list[1] = User{1, "Jane", "Doe", "01-01-1992", "Milton Keynes"}
  db = &Database{list, 1}
  retCode := m.Run()
  os.Exit(retCode)
}
```
Once this is ready, we can start writing tests. Let's have a look at the easiest one where we list the elements in our database:

```
func TestList(t *testing.T) {
  list := db.List()
  count := len(list["users"])
  assert.Equal(t, 2, count, "There should be 2 items in the list.")
}
```

This first calls the `db.List()` function, which returns a list of users. We then count the number of elements and last but not least we then check whether that count equals 2.

In standard Go, you would actually write something like:

```
if 2 != count {
  t.Errorf("Expected 2 elements in the list, instead got %v", count)
}
```

However there is a neat Go package called [testify](https://github.com/stretchr/testify) that gives you assertions like Java and that's why we can write cleaner test code like:

```
assert.Equal(t, 2, count, "There should be 2 items in the list.")
```

The `TestList` is only testing for a positive result, but we really need to test for failures as well.

This is our test code for the Delete functionality:

```
func TestDeleteSuccess(t *testing.T) {
  ok, err := db.Delete(1)
  assert.Equal(t, true, ok, "they should be equal")
  assert.Nil(t, err)
}

func TestDeleteFail(t *testing.T) {
  ok, err := db.Delete(10)
  assert.Equal(t, false, ok, "they should be equal")
  assert.NotNil(t, err)
}
```

The first test function `TestDeleteSuccess` tries to delete a known existing user, with Id 1. We're expecting that the error object is Nil. The second test function `TestDeleteFail` tries to look up a non-existing user with Id 10, and as expected, this should return an actual Error object.

How do we run the tests?

Simple:

```
go test
```

If you want it more verbose, then:

```
go test -v
```

Which will give you:

```
=== RUN TestList
--- PASS: TestList (0.00s)
=== RUN TestGetSuccess
--- PASS: TestGetSuccess (0.00s)
=== RUN TestGetFail
--- PASS: TestGetFail (0.00s)
=== RUN TestAdd
--- PASS: TestAdd (0.00s)
=== RUN TestUpdateSuccess
--- PASS: TestUpdateSuccess (0.00s)
=== RUN TestUpdateFail
--- PASS: TestUpdateFail (0.00s)
=== RUN TestDeleteSuccess
--- PASS: TestDeleteSuccess (0.00s)
=== RUN TestDeleteFail
--- PASS: TestDeleteFail (0.00s)
PASS
ok    github.com/leeprovoost/go-rest-api-template 0.008s
```

Do you want to get some more info on your code coverage? No worries, Go has you covered (no pun intended):

```
go test -cover
```

This will give you:

```
PASS
coverage: 34.9% of statements
ok    github.com/leeprovoost/go-rest-api-template 0.009s
```

TO DO Testing the HTTP service

### Environment Variables

TO DO

## Useful references

* [Structs and JSON formatting](http://stackoverflow.com/questions/21825322/why-golang-cannot-generate-json-from-struct-with-front-lowercase-character)
* [Undertanding method receivers and pointers](http://nathanleclaire.com/blog/2014/08/09/dont-get-bitten-by-pointer-vs-non-pointer-method-receivers-in-golang/)
* [Read JSON POST body](http://stackoverflow.com/questions/15672556/handling-json-post-request-in-go)
* [Writing modular GO REST APIs](http://thenewstack.io/make-a-restful-json-api-go/)
* [Use render for generating JSON](https://github.com/unrolled/render/issues/7) for use of global variable
* [Testing techniques](https://talks.golang.org/2014/testing.slide#1)
* [Testing Go HTTP API](http://dennissuratna.com/testing-in-go/)
* [Great overview of HTTP response codes](http://stackoverflow.com/a/2342631)
* Design beautiful REST + JSON APIs
