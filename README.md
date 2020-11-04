# kok

The toolkit of [Go kit][1].


## Zen

- Just write code for your business logic, generate everything else.
- Implement the service once, consume it in various ways (in-process function call or RPC).


## Features

1. Code Generation Tool

    - [x] Endpoint
    - [x] Transport
        - [x] HTTP
            + [x] HTTP Server
            + [x] HTTP Test
            + [x] HTTP Client
            + [x] [OAS-v2][2] Documentation
        - [ ] gRPC

2. Useful Packages

    - [appx](pkg/appx): Application framework for HTTP and CRON applications (a wrapper of [appx][3]).
    - [prometheus](pkg/prometheus): Prometheus metrics utilities.
    - [trace](pkg/trace): A thin wrapper of [x/net/trace][4] for Go kit.
    - [werror](pkg/werror): Classified business errors.


## Installation

```bash
$ go get -u github.com/RussellLuo/kok/cmd/kokgen
```

<details>
  <summary> Usage </summary>

```bash
$ kokgen -h
kokgen [flags] source-file interface-name
  -fmt
    	whether to make code formatted (default true)
  -out string
    	output directory (default ".")
  -pkg string
    	package name (default will infer)
  -test string
    	the YAML file that provides test-cases for HTTP (default "./http.test.yaml")
  -trace
    	whether to enable tracing
```

</details>


## Quick Start

**NOTE**: The following code is located in [helloworld](examples/helloworld).

1. Define the interface

    ```go
    type Service interface {
        SayHello(ctx context.Context, name string) (message string, err error)
    }
    ```

2. Implement the service

    ```go
    type Greeter struct{}

    func (g *Greeter) SayHello(ctx context.Context, name string) (string, error) {
        return "Hello " + name, nil
    }
    ```

3. Add HTTP annotations

    ```go
    type Service interface {
        // @kok(op): POST /messages
        SayHello(ctx context.Context, name string) (message string, err error)
    }
    ```

4. Generate the HTTP code

    ```bash
    $ cd examples/helloworld
    $ kokgen ./service.go Service
    ```

5. Consume the service

    Run the HTTP server:

    ```bash
    $ go run cmd/main.go
    2020/09/15 18:06:22 transport=HTTP addr=:8080
    ```

    Consume by [HTTPie](https://github.com/jakubroztocil/httpie):

    ```bash
    $ http POST :8080/messages name=Tracey
    HTTP/1.1 200 OK
    Content-Length: 27
    Content-Type: application/json; charset=utf-8
    Date: Tue, 15 Sep 2020 10:06:34 GMT

    {
        "message": "Hello Tracey"
    }
    ```

6. See the OAS documentation

    <details>
      <summary> (Click to show details) </summary>

    ```bash
    $ http GET :8080/api
    HTTP/1.1 200 OK
    Content-Length: 848
    Content-Type: text/plain; charset=utf-8
    Date: Tue, 15 Sep 2020 10:08:24 GMT

    swagger: "2.0"
    info:
      version: "1.0.0"
      title: "Swagger Example"
      description: ""
      license:
        name: "MIT"
    host: "example.com"
    basePath: "/api"
    schemes:
      - "https"
    consumes:
      - "application/json"
    produces:
      - "application/json"

    paths:
      /messages:
        post:
          description: ""
          operationId: "SayHello"
          parameters:
            - name: body
              in: body
              schema:
                $ref: "#/definitions/SayHelloRequestBody"

          produces:
            - application/json; charset=utf-8
          responses:
            200:
              description: ""
              schema:
                $ref: "#/definitions/SayHelloResponse"


    definitions:
      SayHelloRequestBody:
        type: object
        properties:
          name:
            type: string
      SayHelloResponse:
        type: object
        properties:
          message:
            type: string
    ```

    </details>

See more examples [here](examples).


## HTTP

### Annotations

<details>
  <summary> Define the HTTP request operation </summary>

- Key: `@kok(op)`
- Value: `<method> <pattern>`
    + **method**: The request method
    + **pattern**: The request URL
- Example:

    ```go
    type Service interface {
        // @kok(op): POST /users
        CreateUser(ctx context.Context) (err error)
    }
    ```

</details>

<details>
  <summary> Define the HTTP request parameters </summary>

- Key: `@kok(param)`
- Value: `<argName> < in:<in>,name:<name>,type:<type>,required:<required>`
    + **argName**: The name of the method argument.
        - *Argument aggregation*: By specifying the same **argName**, multiple request parameters (each one is of basic type) can be aggregated into one method argument (of any type).
            + You do not need to repeat the **argName**, only the first one is required.
    + **in**:
        - **path**: The method argument is sourced from a [path parameter](https://swagger.io/docs/specification/describing-parameters/#path-parameters).
        - **query**: The method argument is sourced from a [query parameter](https://swagger.io/docs/specification/describing-parameters/#query-parameters).
        - **header**: The method argument is sourced from a [header parameter](https://swagger.io/docs/specification/describing-parameters/#header-parameters).
        - **cookie**: The method argument is sourced from a [cookie parameter](https://swagger.io/docs/specification/describing-parameters/#cookie-parameters).
            + Not supported yet.
        - **body**: The method argument is sourced from the [request body](https://swagger.io/docs/specification/describing-request-body/).
            + Optional: All method arguments, unless otherwise specified, are in **body**.
        - **request**: The method argument is sourced from a property of Go's [http.Request](https://golang.org/pkg/net/http/#Request).
            + This is a special case, and only one property `RemoteAddr` is available now.
            + Note that parameters located in **request** have no relationship with OAS.
    + **name**: The name of the corresponding request parameter.
        - Optional: Defaults to **argName** if not specified.
    + **type**: The type of the corresponding request parameter.
        - Optional: Defaults to the type of the method argument, if not specified.
        - **Required** for *Argument aggregation* for generating correct OAS documentation.
    + **required**: Determines whether this parameter is mandatory.
        - Optional: Defaults to `false`, if not specified.
        - If the parameter location is **path**, this property will be set to `true` internally, whether it's specified or not.
- Example:
    + Simple argument:

        ```go
        type Service interface {
            // @kok(op): DELETE /users/{id}
            // @kok(param): id < in:path
            DeleteUser(ctx context.Context, id int) (err error)
        }

        // HTTP request: DELETE /users/101
        ```
    + Argument aggregation:

        ```go
        type User struct {
            Name string `kok:"query.name"`
            Age  int    `kok:"query.age"`
        }

        type Service interface {
            // @kok(op): POST /users
            // @kok(param): user < in:query,name:name,type:string
            // @kok(param): user < in:query,name:age,type:int
            CreateUser(ctx context.Context, user User) (err error)
        }

        // The equivalent annotations.
        type Service interface {
            // @kok(op): POST /users
            // @kok(param): user < in:query,name:name,type:string
            // @kok(param):      < in:query,name:age,type:int
            CreateUser(ctx context.Context, user User) (err error)
        }

        // HTTP request: POST /users?name=tracey&age=1
        ```

</details>

<details>
  <summary> Define the success HTTP response </summary>


- Key: `@kok(success)`
- Value: `statusCode:<statusCode>`
    + **statusCode**: The status code of the success HTTP response.
        - Optional: Defaults to 200 if not specified.
- Example:

    ```go
    type Service interface {
        // @kok(op): POST /users
        // @kok(success): statusCode:201
        CreateUser(ctx context.Context) (err error)
    }
    ```

</details>

### Encoding and decoding

See the [HTTP Codec](https://github.com/RussellLuo/kok/blob/master/pkg/codec/httpv2/codec.go#L8-L22) interface.

### OAS Schema

See the [OAS Schema](https://github.com/RussellLuo/kok/blob/master/pkg/oasv2/schema.go#L18-L21) interface.


## Documentation

Checkout the [Godoc][5].


## License

[MIT](LICENSE)


[1]: https://github.com/go-kit/kit
[2]: https://swagger.io/specification/v2/
[3]: https://github.com/RussellLuo/appx
[4]: https://pkg.go.dev/golang.org/x/net/trace
[5]: https://pkg.go.dev/github.com/RussellLuo/kok
