# conman

![status](https://circleci.com/gh/luminus-framework/conman.svg?style=shield&circle-token=da462d56802b9a0ff94ecc7992d1d0caf8475d00)

Luminus database connection management and SQL query generation library

The library provides pooled connections using the [HikariCP](https://github.com/brettwooldridge/HikariCP) library.

The queries are generated using [HugSQL](https://github.com/layerware/hugsql) and wrapped with
connection aware functions.

## Usage

[![Clojars Project](http://clojars.org/conman/latest-version.svg)](http://clojars.org/conman)

Conman relies on a dynamic variable to manage the connection. The dynamic variable allows the connection to be
rebound contextually for transactions. When working with multiple databases, a separate var is required
to track each database connection.

### Defining Queries

SQL statements should be populated in files that are accessible on the resource path.
For example, we could create a file `resources/sql/queries.sql` with
the following content:

``` sql
-- :name create-user! :! :n
-- :doc creates a new user record
INSERT INTO users
(id, first_name, last_name, email, pass)
VALUES (:id, :first_name, :last_name, :email, :pass)

-- :name get-user :? :1
-- :doc retrieve a user given the id.
SELECT * FROM users
WHERE id = :id

-- :name get-all-users :? :*
-- :doc retrieve all users.
SELECT * FROM users
```
See the official [HugSQL docs](http://www.hugsql.org/) for further examples.

### Using `bind-connection`

The queries are bound to the connection using the `bind-connection` macro. This macro
accepts the connection var followed by one or more strings representing SQL query files.

The lifecycle of the connection is expected to be managed using a library such as [mount](https://github.com/tolitius/mount). The full list of options that can be passed to `pool-spec` can be found [here](https://github.com/tomekw/hikari-cp#configuration-options).


```clojure
(ns myapp.db
  (:require [mount.core :refer [defstate]]
            [conman.core :as conman]))

(def pool-spec
  {:jdbc-url "jdbc:postgresql://localhost/myapp?user=user&password=pass"})

(defstate ^:dynamic *db*
          :start (conman/connect! pool-spec)
          :stop (conman/disconnect! *db*))

(conman/bind-connection *db* "sql/queries.sql")
```

The `bind-connection` generates `create-user!` and `get-user` functions
in the current namespace. These functions can be called in four different ways:

```clojure
;; when called with no argument then the HugSQL generated function
;; will be called with an empty parameter map and the connection specified in the *db* var
(get-all-users)

;; when a parameter map is passed as the argument, then the map and the connection specified
;; in the *db* var will be passed to the HugSQL generated function
(create-user! {:id "foo" :first_name "Bob" :last_name "Bobberton" :email nil :pass nil})

;; an explicit connection and a parameter map can be
;; passed to the function
(get-user some-other-conn {:id "foo"})

;; finally, an explicit connection and a parameter map, options, and optional command options
;; can be passed to the function
(get-user some-other-conn {:id "foo"} opts)
(get-user some-other-conn {:id "foo"} opts cmd-opt1 cmd-opt2)
```

### Using `bind-connection-map`

Alternatively, you may wish to use `bind-connection-map` to define the queries. This function will return
a map containing `:snips` and `:fns` keys that point to maps of snippets and queries respectively.
The `snip` and `query` helper functions are provided for accessing the connection map returned by the `bind-connection-map`
function.

```clojure
;; create a connection map given the connection instance and one or more query files:
(def queries (bind-connection-map conn "queries.sql"))

;; a HugSQL options map can be passed as the second argument:
(def queries (bind-connection-map conn {:quoting :ansi} "queries.sql"))

;; run a query
(query
  queries
  :add-fruit!
  {:name       "apple"
   :appearance "red"
   :cost       1
   :grade      1})

;; run a query specifying the connection explicitly
(query
  conn
  queries
  :add-fruit!
  {:name       "apple"
   :appearance "red"
   :cost       1
   :grade      1})

;; run a query with a snippet
(query
  conn
  queries
  :get-fruit-by
  {:by-appearance
   (snip queries :by-appearance {:appearance "red"})})
```

If you use `:cljc` [mode of mount](https://github.com/tolitius/mount/blob/master/doc/clojurescript.md#clojure-and-clojurescript-mode),
where you would need to explicitly `deref` each state, you'll need to use `conman/bind-connection-deref` instead of
`conman/bind-connection`. Both `conman/bind-connection-deref` and `conman/with-transaction` macros still accept undereferenced state,
while in every other case it has to be dereferenced:

```clojure
(mount/in-cljc-mode)

(conman/bind-connection-deref *db* "sql/queries.sql")

(conman/with-transaction [*db*]
  (sql/db-set-rollback-only! @*db*)
  (add-memo! {:id 123 :text "Hello"})
  (sql/query @*db* ["SELECT * FROM memos;"]))

```

Next, the `connect!` function should be called to initialize the database connection.
The function accepts a map with the database specification.

```clojure
(def pool-spec
  {:jdbc-url   "jdbc:postgresql://localhost/myapp?user=user&password=pass"})

(connect! pool-spec)
```

For the complete list of configuration options refer to the official [hikari-cp](https://github.com/tomekw/hikari-cp) library documentation. Conman supports the following additional options:

* `:datasource`
* `:datasource-classname`

The connection can be terminated by running the `disconnect!` function:

```clojure
(disconnect! conn)
```

A connection can be reset using the `reconnect!` function:

```clojure
(reconnect! conn pool-spec)
```

When using a dynamic connection, it's possible to use the `with-transaction`
macro to rebind it to the transaction connection. The SQL query functions
generated by the `bind-connection` macro will automatically use the transaction
connection in that case:

```clojure
(with-transaction [conn]
  (jdbc/db-set-rollback-only! conn)
  (create-user!
    {:id         "foo"
     :first_name "Sam"
     :last_name  "Smith"
     :email      "sam.smith@example.com"})
  (get-user {:id "foo"}))
```

The isolation level and readonly status of the transaction may be specified using the `:isolation`
and `:read-only?` keys respectively:

```clojure
(with-transaction
    [conn {:isolation :serializable}]
    (= java.sql.Connection/TRANSACTION_SERIALIZABLE
       (.getTransactionIsolation (sql/db-connection conn))))
(with-transaction
  [conn {:isolation :read-uncommitted}]
  (= java.sql.Connection/TRANSACTION_READ_UNCOMMITTED
     (.getTransactionIsolation (sql/db-connection conn))))
```

## License

Copyright © 2015 Dmitri Sotnikov and Carousel Apps Ltd.

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
