# A High performance HTTP request router with Context

* [Features](#features)
* [Examples](#examples)
  * [Basic Example](#basic-example)
  * [With Powerful Context](#with-powerful-context)
  * [Decode Request Before Business Layer](#decode-request-before-business-layer)
  * [Normal HTTP Handler](#normal-http-handler)
  * [Static Files](#static-files)
  * [Restful Api](#restful-api)
* [Full Example](#full-example)

# Features

* Context Append on Current Function
* Best Performance (no regexp match)
* Wildcards Router Support (PathPrefix)
* Decode request body before business layer (JSON, xml or other)
* Decode request url before business layer
* Zero Garbage

# Examples


## Basic Example

```go
package main

import (
	"github.com/leenanxi/ctxrouter"
	"net/http"
)

//context style
func (ctx *Context) Hello(name string) {
	//ctx.Request ...
	ctx.Writer.Write([]byte("hello " + name))
}
//normal style
func NormalHello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello " + ctxrouter.Params(r)[0]))
}
func main() {
	r := ctxrouter.New()
	r.Get("/basic/:name", (*Context).Hello)
    r.Get("/normal/:name", NormalHello)
	r.Get("/", (*Context).Index)
	//auto decode url with string or int
	r.Get("/basic/:name/json/:age", (*Context).Json)
	//match path prefixes /all/*:
	r.All("/basic/*path",(*Context).All)
	//a simple func without implement ctxrouter.Context
	r.Get("/basic/:name/simple",Simple)
	http.ListenAndServe(":8081", r)
}

type Context struct {
	ctxrouter.Context
}

func (c *Context) Index() {
	c.Text("index")
}

func (c *Context) All(path string) {
	c.Text("all router goes here " +  path)
}

func (c *Context) Json(name string, age int) {
	type Person struct {
		Name string
		Age   int
	}
	c.JSON(Person{Name:name,Age:age})
}

func Simple(ctx *ctxrouter.Context, name string) {
	ctx.Text("simple " + name)
}
```


## With Powerful Context

```go
//do something  Workflow with ctx router
package main

import (
	"net/http"
	"github.com/leenanxi/ctxrouter"
)

type Context struct {
	ctxrouter.Context
	Data  map[string]string
}

func (c *Context) Start() {
	c.Data = make(map[string]string)
	c.Data["context"] = "0"
	c.Step()
}

func (c *Context) Step() {
	c.Data["context1"] = "1"
	c.End()
}

func (c *Context) End() {
	c.Data["context2"] = "2"
	c.JSON(c.Data)
}

func main() {
	r := ctxrouter.New()
	r.Get("/context/",(*Context).Start)
	http.ListenAndServe(":8081", r)
}
```


## Decode Request Before Business Layer

```go
package main

import (
	"net/http"
	"github.com/leenanxi/ctxrouter"
)

//decode request sample
type User struct {
	Id      int             `json:"int"`
	Name    string          `json:"name"`
}

type UserContext struct {
	ctxrouter.Context
	Data  *User
}

//Auto Decode Json or other request
func (ctx *UserContext) DecodeRequest() error {
	ctx.Data = new(User)
	ctx.Context.Data = ctx.Data
	return ctx.Context.DecodeRequest()
}

func (ctx *UserContext) SayHello() {
	ctx.Text("Hello "+ ctx.Data.Name)
}

func main() {
	r := ctxrouter.New()
	r.Post("/users/hello",(*UserContext).SayHello)
	http.ListenAndServe(":8081", r)
}
```

```bash
curl -i -X POST \
   -H "Content-Type:application/json" \
   -d \
'{"name":"leenanxi"}' \
 'http://localhost:8081/users/hello'
```


## Normal HTTP Handler

Alert: This is Not recommended if you start a new project.

```go
package main

import (
	"github.com/leenanxi/ctxrouter"
	"net/http"
)

func NormalHelloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("HELLO"))
}

func NormalHandler(w http.ResponseWriter, r *http.Request) {
	params := ctxrouter.Params(r)
	w.Write([]byte("Name:" + params[0] + "\nAge:" + params[1] ))
}

func main() {
	r := ctxrouter.New()
	r.Get("/normal/hello",NormalHelloHandler)
	r.Get("/normal/v1/:name/:age",NormalHandler)
	//support any http.Handler interface
	r.Get("/404",http.NotFoundHandler())
	http.ListenAndServe(":8081", r)
}

```

## Static Files

```go
package main

import (
	"github.com/leenanxi/ctxrouter"
	"net/http"
)

func main() {
	var dir = "/your/static/dir/path"
	r := ctxrouter.New()
	r.All("/static/*path",http.StripPrefix("/static/", http.FileServer(http.Dir(dir))))
	http.ListenAndServe(":8081", r)
}
```


## Restful Api

```go
package main

import (
	"net/http"
	"github.com/leenanxi/ctxrouter"
)
func main() {
	r := ctxrouter.New()
	r.Get("/apps", (*AppContext).GetApps)
	r.Get("/apps/:id", (*AppContext).GetApp)
	r.Post("/apps", (*AppContext).PostApps)
	r.Patch("/apps/:id", (*AppContext).PatchApp)
	r.Put("/apps/:id", (*AppContext).PutApp)
	r.Delete("/apps/:id", (*AppContext).DeleteApp)
	http.ListenAndServe(":8081", r)
}
type AppContext struct {
	ctxrouter.Context
}
func (ctx *AppContext) GetApps() {
	ctx.Text("get apps")
}
func (ctx *AppContext) GetApp(id string) {
	ctx.Text("get app " + id)
}
func (ctx *AppContext) PostApps() {
	ctx.Text("post apps")
}
func (ctx *AppContext) DeleteApp(id string) {
	ctx.Text("delete app " + id)
}
func (ctx *AppContext) PutApp(id string) {
	ctx.Text("put app " + id)
}
func (ctx *AppContext) PatchApp(id string) {
	ctx.Text("patch app " + id)
}
```


## Full Example

```go
//full example with all features in one file, you can read sections above
package main

import (
	"net/http"
	"fmt"
	"github.com/leenanxi/ctxrouter"
)

func main() {
	r := ctxrouter.New()
	r.Get("/", (*Controller).Index)
	r.Get("/basic/:name", (*Controller).Hello)
	//match path prefixes /all/*:
	r.All("/basic/*path",(*Controller).All)
	//auto decode url with string or int
	r.Get("/basic/:name/json/:age", (*Controller).Json)
	//a simple func without implement ctxrouter.Context
	r.Get("/basic/:name/simple",Simple)

	r.Post("/users/hello",(*UserContext).PrintHello)

	//do something  Workflow with ctx router
	r.Get("/context/",(*Context).Start)


	r.Get("/normal/hello",NormalHelloHandler)
	r.Get("/normal/v1/:name/:age",NormalHandler)
	//support any http.Handler interface
	r.Get("/404",http.NotFoundHandler())

	//static files
	var dir = "/your/static/dir/path"
	r.All("/static/*path",http.StripPrefix("/static/", http.FileServer(http.Dir(dir))))
	http.ListenAndServe(":8081", r)
}

type Controller struct {
	ctxrouter.Context
}

func (c *Controller) Index() {
	c.Text("index")
}

func (c *Controller) Hello(name string) {
	fmt.Fprintln(c.Writer, "hello "+name)
}

func (c *Controller) All(path string) {
	c.Text("all router goes here " +  path)
}
//input json and output json
func (c *Controller) Json(name string, age int) {
	type Person struct {
		Name string
		Age   int
	}
	c.JSON(Person{Name:name,Age:age})
}

func Simple(ctx *ctxrouter.Context, name string) {
	ctx.Text("simple " + name)
}

//decode request sample
type User struct {
	Id      int             `json:"int"`
	Name    string          `json:"name"`
}

type UserContext struct {
	ctxrouter.Context
	Data  *User
}

//Auto Decode Json or other request
func (ctx *UserContext) DecodeRequest() error{
	ctx.Data = new(User)
	ctx.Context.Data = ctx.Data
	return ctx.Context.DecodeRequest()
}

func (ctx *UserContext) PrintHello() {
	ctx.Text("Hello "+ ctx.Data.Name)
}

type Context struct {
	ctxrouter.Context
	Data  map[string]string
}

func (c *Context) Start() {
	c.Data = make(map[string]string)
	c.Data["context"] = "0"
	c.Step()
}

func (c *Context) Step() {
	c.Data["context1"] = "1"
	c.End()
}

func (c *Context) End() {
	c.Data["context2"] = "2"
	c.JSON(c.Data)
}

func NormalHelloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("HELLO"))
}

func NormalHandler(w http.ResponseWriter, r *http.Request) {
	//get router Params from "X-Ctxrouter-Params" without any extra function
	params := ctxrouter.Params(r)
	w.Write([]byte("Name:" + params[0] + "\nAge:" + params[1] ))
}
```




# Thanks 

* tree.go & tree_test.go is edited from httprouter https://github.com/julienschmidt/httprouter