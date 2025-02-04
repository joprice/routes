## Routes &nbsp; ![BuildStatus](https://github.com/anuragsoni/routes/workflows/RoutesTest/badge.svg) [![Coverage Status](https://coveralls.io/repos/github/anuragsoni/routes/badge.svg?branch=master)](https://coveralls.io/github/anuragsoni/routes?branch=master)

This library will help with adding typed routes to OCaml applications.
The goal is to have a easy to use portable library with
reasonable performance ([See benchmark folder](./bench/)).

Users can create a list of routes, and handler function to work
on the extracted entities using the combinators provided by
the library. To perform URL matching one would just need to forward
the URL's path to the router.

#### Demo

You can follow along with these examples in the OCaml toplevel (repl).
[down](https://github.com/dbuenzli/down) or [utop](https://github.com/ocaml-community/utop) are recommended to enhance
your REPL experience while working through these examples. They will add autocompletion support
which can be useful when navigating a new library.

We will start by setting up the toplevel by asking it to load routes.

```ocaml
# #require "routes";;
```

We will start by defining a few simple routes that don't need to extract any path parameter.

```ocaml
# (* A simple route that matches the empty path segments. *);;
# let root () = Routes.nil;;
val root : unit -> ('a, 'a) Routes.path = <fun>

# (* We can combine multiple segments using `/` *);;
# let users () = Routes.(s "users" / s "get" /? nil);;
val users : unit -> ('a, 'a) Routes.path = <fun>
```

We can use these route definitions to get a string "pattern" that can potentially be used
to show what kind of routes your application can match.
```ocaml
# Routes.string_of_path (root ());;
- : string = "/"

# Routes.string_of_path (users ());;
- : string = "/users/get"
```

Matching routes where we don't need to extract any parameter could be done with a simple string match.
The part where routers are useful is when there is a need to extract some parameters are extracted from the
path.

```ocaml
# let sum () = Routes.(s "sum" / int / int /? nil);;
val sum : unit -> (int -> int -> 'a, 'a) Routes.path = <fun>
```

Looking at the type for `sum` we can see that our route knows about our two integer path parameters.
A route can also extract parameters of different types.

```ocaml
# let get_user () = Routes.(s "user" / str / int64 /? nil);;
val get_user : unit -> (string -> int64 -> 'a, 'a) Routes.path = <fun>
```

We can still pretty print such routes to get a human readable "pattern" that can be used to inform
someone what kind of routes are defined in an application.


Once we start working with routes that extract path parameters, there is another operation that can sometimes
be useful. Often times there can be a need to generate a URL from a route. It could be for creating
hyperlinks in HTML pages, creating target URLs that can be forwarded to HTTP clients, etc.


Using routes we can create url targets from the same type definition that is used for performing a route match.
Using this approach for creating url targets has the benefit that whenever a route definition is updated,
the printed format for the url target will also reflect that change. If the types remain the same,
then the printing functions will automatically start generating url targets that
reflect the change in the route type, and if the types change the user will get a compile time error about mismatched
types. This can be useful in ensuring that we avoid using bad/outdated URLs in our application.

```ocaml
# Routes.string_of_path (sum ());;
- : string = "/sum/:int/:int"

# Routes.string_of_path (get_user ());;
- : string = "/user/:string/:int64"

# Routes.sprintf (sum ());;
- : int -> int -> string = <fun>

# Routes.sprintf (get_user ());;
- : string -> int64 -> string = <fun>

# Routes.sprintf (sum ()) 45 12;;
- : string = "/sum/45/12"

# Routes.sprintf (sum ()) 11 56;;
- : string = "/sum/11/56"

# Routes.sprintf (get_user ()) "JohnUser" 1L;;
- : string = "/user/JohnUser/1"

# Routes.sprintf (get_user ()) "foobar" 56121111L;;
- : string = "/user/foobar/56121111"
```

We've seen a few examples so far, but none of any actual routing. Before we can perform a route match,
we need to connect a route definition to a handler function that gets called when a successful match happens.

```ocaml
# let sum_route () = Routes.(sum () @--> fun a b -> Printf.sprintf "%d" (a + b));;
val sum_route : unit -> string Routes.route = <fun>

# let user_route () = Routes.(get_user () @--> fun name id -> Printf.sprintf "(%Ld) %s" id name);;
val user_route : unit -> string Routes.route = <fun>

# let root () = Routes.(root () @--> "Hello World");;
val root : unit -> string Routes.route = <fun>
```

Now that we have a collection of routes connected to handlers, we can create a router and perform route matching.
Something to keep in mind is that we can only combine routes that have the same final return type, i.e. handlers
attached to every route in a router should have the same type for the values they return.

```ocaml
# let routes = Routes.one_of [sum_route (); user_route (); root ()];;
val routes : string Routes.router = <abstr>

# Routes.match' routes ~target:"/";;
- : string Routes.match_result = Routes.FullMatch "Hello World"

# Routes.match' routes ~target:"/sum/25/11";;
- : string Routes.match_result = Routes.FullMatch "36"

# Routes.match' routes ~target:"/user/John/1251";;
- : string Routes.match_result = Routes.FullMatch "(1251) John"

# Routes.match' routes ~target:(Routes.sprintf (sum ()) 45 11);;
- : string Routes.match_result = Routes.FullMatch "56"

# (* This route is not an exact match because of the final trailing slash. *);;
# Routes.match' routes ~target:"/sum/1/2/";;
- : string Routes.match_result = Routes.MatchWithTrailingSlash "3"
```

#### Dealing with trailing slashes

Every route definition can control what behavior it expects when it encounters
a trailing slash. In the examples above all route definitions ended with
`/? nil`. This will result in an exact match if the route does not end in a trailing slash.
If the input target matches every paramter but has an additional trailing slash, the route will
still be considered a match, but it will inform the user that the matching route was found, 
effectively having disregarded the trailing slash.

```ocaml
# let no_trail () = Routes.(s "foo" / s "bar" / str /? nil @--> fun msg -> String.length msg);;
val no_trail : unit -> int Routes.route = <fun>

# Routes.(match' (one_of [ no_trail () ]) ~target:"/foo/bar/hello");;
- : int Routes.match_result = Routes.FullMatch 5

# Routes.(match' (one_of [ no_trail () ]) ~target:"/foo/bar/hello/");;
- : int Routes.match_result = Routes.MatchWithTrailingSlash 5
```

To create a route that returns an exact match if there is a trailing slash, the route still needs to
end with `nil`, but instead of `/?`, `//?` needs to be used. (note the extra slash).

```ocaml
# let trail () = Routes.(s "foo" / s "bar" / str /? nil @--> fun msg -> String.length msg);;
val trail : unit -> int Routes.route = <fun>

# Routes.(match' (one_of [ trail () ]) ~target:"/foo/bar/hello");;
- : int Routes.match_result = Routes.FullMatch 5

# Routes.(match' (one_of [ trail () ]) ~target:"/foo/bar/hello/");;
- : int Routes.match_result = Routes.MatchWithTrailingSlash 5
```

More example of library usage can be seen in the [examples](https://github.com/anuragsoni/routes/tree/main/example) folder,
and as part of the [test](https://github.com/anuragsoni/routes/blob/main/test/routing_test.ml) definition.

## Installation

###### To use the version published on opam:
```
opam install routes
```

###### For development version:
```
opam pin add routes git+https://github.com/anuragsoni/routes.git
```
