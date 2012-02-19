== Clutch +++<a href="http://travis-ci.org/#!/clojure-clutch/clutch/builds">+++image:https://secure.travis-ci.org/clojure-clutch/clutch.png[]+++</a>+++

Clutch is a http://clojure.org:[Clojure] library for http://couchdb.apache.org/[Apache CouchDB].

# "Installation"

To include Clutch in your project, simply add the following to your `project.clj` dependencies:

```clojure
[com.ashafa/clutch "0.3.1-SNAPSHOT"]
```

and run...

```bash
lein deps
```

Or, if you're using Maven, add this dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.ashafa</groupId>
    <artifactId>clutch</artifactId>
    <version>0.3.1-SNAPSHOT</version>
</dependency>
```

Clutch is compatible with Clojure 1.2.0 and 1.3.0.  The most recent release was `0.3.0`; but, be a mensch and use the SNAPSHOT, and report issues to the http://groups.google.com/group/clojure-clutch[Clutch mailing list].

## Status

Although it's in an early stage of development (Clutch API subject to change), Clutch supports most of the Apache CouchDB API:

* Essentially all of the http://wiki.apache.org/couchdb/HTTP_Document_API[core document API]
* http://wiki.apache.org/couchdb/HTTP_Bulk_Document_API[Bulk document APIs]
* Most of the http://wiki.apache.org/couchdb/HTTP_database_API[Database API], including `_changes` (implemented using watches)
* http://wiki.apache.org/couchdb/HTTP_view_API[Views], including access, update, and a Clojure view server implementation

At the moment, you'll have to look at the source or introspect the docs once you've loaded Clutch up to get around the API.  Proper API documentation (via autodoc or marginalia) coming soon.

Clutch does not currently provide any direct support for the various couchapp-related APIs, including update handlers and validation, shows and lists, and so on.

That said, it is very easy to call whatever CouchDB API feature that Clutch doesn't support using the lower-level `com.ashafa.clutch.http-client/couchdb-request` function.

### Tests / CI

Clutch's test suite is run on every commit thanks to http://travis-ci.org/#!/clojure-clutch/clutch/builds[Travis].

## Usage

```clojure
=> (get-database "clutch_example")  ;; creates database if it's not available yet
#com.ashafa.clutch.utils.URL{:protocol "http", :username nil, :password nil, :host "localhost", :port -1,
:path "clutch_example", :query nil, :disk_format_version 5, :db_name "clutch_example", :doc_del_count 0,
:committed_update_seq 0, :disk_size 79, :update_seq 0, :purge_seq 0, :compact_running false,
:instance_start_time "1323701753566374", :doc_count 0}

=> (bulk-update "clutch_example" [{:test-grade 10 :_id "foo"}
                                  {:test-grade 20}
                                  {:test-grade 30}])
[{:id "foo", :rev "1-8a15da0db077cd05b45ec93b3a207d09"}
 {:id "0896fbf57128d7f1a1b238a52b0ec372", :rev "1-796ebf042b42fa3585332c3aa4a6f706"}
 {:id "0896fbf57128d7f1a1b238a52b0ecda8", :rev "1-01f063c5aeb1b63992c90c72c7a515ed"}]
=> (get-document "clutch_example" "foo")
{:_id "foo", :_rev "1-8a15da0db077cd05b45ec93b3a207d09", :test-grade 10}
```

All Clutch functions accept a first argument indicating the database API endpoint for that operation.
This argument can be a string URL, or, if the string is provided without a protocol, etc., it is assumed to be
the name of a database on localhost:5984 as above:

```clojure
=> (put-document "https://username:password`XXX.cloudant.com/databasename/" {:a 5 :b 6})
{:_id "36b807aacf227f921aa256b06ab094e5", :_rev "1-d4d04a5b59bcd73893a84de2d9595c4c", :a 5, :b 6}
```

Alternatively, Clutch defines its own record type for URLs:

```clojure
=> (com.ashafa.clutch.utils/url "databasename")
#com.ashafa.clutch.utils.URL{:protocol "http", :username nil, :password nil,
:host "localhost", :port -1, :path "databasename", :query nil}
```

You can `assoc` in whatever you like to a `URL` record, which is handy for keeping database URLs and
credentials separate:

```clojure
=> (def db (assoc (utils/url "https://XXX.cloudant.com/")
                  :username "username"
                  :password "password"
                  :path "databasename"))
#'test-clutch/db
=> (put-document db {:a 5 :b [0 6]})
{:_id "17e55bcc31e33dd30c3313cc2e6e5bb4", :_rev "1-a3517724e42612f9fbd350091a96593c", :a 5, :b [0 6]}
```

You can optionally provide configuration using dynamic scope via `with-db`:

```clojure
=> (with-db "clutch_example"
     (put-document {:_id "a" :a 5})
     (put-document {:_id "b" :b 6})
     (-> (get-document "a")
       (merge (get-document "b"))
       (dissoc-meta)))
{:b 6, :a 5}
```

## Experimental: a Clojure-idiomatic CouchDB type

Clutch provides a pretty comprehensive API, but 95% of database
interactions require using something other than the typical Clojure vocabulary of
`assoc`, `conj, `dissoc`, `get`, `seq`, `reduce`, etc, even though those semantics are entirely appropriate
(modulo the whole stateful database thing).

This is (the start of) an attempt to create a type to provide most of the
functionality of Clutch with a more pleasant, concise API (it uses the Clutch API
under the covers, and rare operations will generally remain accessible only
at that lower level).

Would like to eventually add:

* support for views (aside from `_all_docs` via `seq`)
* support for `_changes` (via a `seque`?), maybe a more natural place than the (free-for-all) pool of watches in Clutch's current API
* support for bulk update, maybe via `IReduce`?
* Other CouchDB types:
** to provide specialized query interfaces e.g. cloudant indexes
** to return custom map and vector types to support e.g.

```clojure
(assoc-in! db ["ID" :key :key array-index] x)
(update-in! db ["ID" :key :key array-index] assoc :key y)
```

Feedback wanted on the mailing list: http://groups.google.com/group/clojure-clutch

This part of the API is subject to change at any time, so no detailed examples.  For now, just a REPL interaction will do:

```clojure
=> (use 'com.ashafa.clutch)     ;; My apologies for the bare `use`!
nil
=> (def db (couch "test"))
#'user/db
=> (create! db)
#<CouchDB user.CouchDB@3f460a4a>
=> (:result (meta *1))
#com.ashafa.clutch.utils.URL{:protocol "http", :username nil, :password nil,
:host "localhost", :port -1, :path "test", :query nil, :disk_format_version 5,
:db_name "test", :doc_del_count 0, :committed_update_seq 0, :disk_size 79,
:update_seq 0, :purge_seq 0, :compact_running false, :instance_start_time
"1324037686108297", :doc_count 0}
=> (reduce conj! db (for [x (range 5000)]
                      {:_id (str x) :a [1 2 x]}))
#<CouchDB user.CouchDB@71d1be4e>
=> (count db)
5000
=> (get-in db ["68" :a 2])
68
=> (def copy (into {} db))
#'user/copy
=> (get-in copy ["68" :a 2])
68
=> (first db)
["0" {:_id "0", :_rev "1-79fe783154bff972172bc30732783a68", :a [1 2 0]}]
=> (dissoc! db "68")
#<CouchDB user.CouchDB@48f50903>
=> (get db "68")
nil
=> (assoc! db :foo {:a 6 :b 7})
#<CouchDB user.CouchDB@79d7999e>
=> (:result (meta *1))
{:_rev "1-ac3fe57a7604cfd6dcca06b25204b590", :_id ":foo", :a 6, :b 7}
```

## Configuring your CouchDB installation to use the Clutch view server

CouchDB needs to know how to exec Clutch's view server.  Getting this command string together can be tricky, especially given potential classpath complexity.  You can either (a) produce an uberjar of your project, in which case the exec string will be something like:

```bash
java -cp <path to your uberjar> clojure.main -m com.ashafa.clutch.view-server
```

or, (b) you can use the `com.ashafa.clutch.utils/view-server-exec-string` function to dump a likely-to-work exec string.  For example:

```clojure
user=> (use '[com.ashafa.clutch.view-server :only (view-server-exec-string)])
nil
user=> (println (view-server-exec-string))
java -cp "clutch/src:clutch/test:clutch/classes:clutch/resources:clutch/lib/clojure-1.3.0-beta1.jar:clutch/lib/clojure-contrib-1.2.0.jar:clutch/lib/data.json-0.1.1.jar:clutch/lib/tools.logging-0.1.2.jar" clojure.main -m com.ashafa.clutch.view-server
```

This function assumes that `java` is on CouchDB's PATH, and it's entirely possible that the classpath might not be quite right (esp. on Windows — the above only tested on OS X and Linux so far).  In any case, you can test whether the view server exec string is working properly by trying it yourself and attempting to get it to echo back a log message:

```bash
[catapult:~/dev/clutch] chas% java -cp "clutch/src:clutch/test:clutch/classes:clutch/resources:clutch/lib/clojure-1.3.0-beta1.jar:clutch/lib/clojure-contrib-1.2.0.jar:clutch/lib/data.json-0.1.1.jar:clutch/lib/tools.logging-0.1.2.jar" clojure.main -m com.ashafa.clutch.view-server
["log" "echo, please"]
["log",["echo, please"]]
```

Enter the first JSON array, and hit return; the view server should immediately reply with the second JSON array.  Anything else, and your exec string is flawed, or something else is wrong.

Once you have a working exec string, you can use Clojure for views and filters by adding a view server configuration to CouchDB.  This can be as easy as passing the exec string to the `com.ashafa.clutch/configure-view-server` function:

```clojure
(configure-view-server (view-server-exec-string))
```

Alternatively, use Futon to add the `clojure` query server language to your CouchDB instance's config.

In the end, both of these methods add the exec string you provide it to the `local.ini` file of your CouchDB installation, which you can modify directly if you like (this is likely what you'll need to do for non-local/production CouchDB instances):

```clojure
  [query_servers]
  clojure = java -cp …rest of your exec string…
```

### View server configuration & view API usage

```clojure
=> (configure-view-server "clutch_example" (com.ashafa.clutch.view-server/view-server-exec-string))
""
=> (save-view "clutch_example" "demo_views" (view-server-fns :clojure
                                              {:sum {:map (fn [doc] [[nil (:test-grade doc)]])
                                                     :reduce (fn [keys values _] (apply + values))}}))
{:_rev "1-ddc80a2c95e06b62dd2923663dc855aa", :views {:sum {:map "(fn [doc] [[nil (:test-grade doc)]])", :reduce "(fn [keys values _] (apply + values))"}}, :language :clojure, :_id "_design/demo_views"}
=> (-> (get-view "clutch_example" "demo_views" :sum) first :value)
60
=> (get-view "clutch_example" "demo_views" :sum {:reduce false})
({:id "0896fbf57128d7f1a1b238a52b0ec372", :key nil, :value 20}
 {:id "0896fbf57128d7f1a1b238a52b0ecda8", :key nil, :value 30}
 {:id "foo", :key nil, :value 10})
=> (map :value (get-view "clutch_example" "demo_views" :sum {:reduce false}))
(20 30 10)
```

Note that all view access functions (i.e. `get-view`, `all-documents`, etc) return a lazy seq of their results (corresponding to the `:rows` slot in the data that couchdb returns in its view data).  Other values (e.g. `total_rows`, `offset`, etc) are added to the returned lazy seq as metadata. 

```clojure
=> (meta (all-documents "databasename"))
{:total_rows 20000, :offset 0}
```

## (Partial) Changelog

### 0.3.0

Many breaking changes to refine/simplify the API, clean up the implementation, and add additional features:

Core API:

* Renamed `create-document` => `put-document`; `put-document` now supports both creation and update of a document depending upon whether  `:_id` and `:_rev` slots are present in the document you are saving.
* Renamed `update-attachment` => `put-attachment`; `filename` and `mime-type` arguments now kwargs, `InputStream` can now be provided as attachment data
* `update-document` semantics have been simplified for the case where an "update function" and arguments are supplied to work well with core Clojure functions like `update-in` and `assoc` (fixes issue #8) — e.g. can be used like `swap!` et al.
* Optional `:id` and `:attachment` arguments to `put-document` (was `create-document`) are now specified via keyword arguments
* Removed "update map" argument from `bulk-update` fn (replace with e.g. `(bulk-update db (map #(merge % update-map) documents)`)
* Renamed `get-all-documents-meta` => `all-documents`
* `com.ashafa.clutch.http-client/*response-code*` is no longer assumed to be an atom. Rather, it is `set!`-ed directly when it is thread-bound. (Fixes issue #29)

View-related API:

* All views (`get-view`, `all-documents`, etc) now return lazy seqs corresponding to the `:rows` slot in the view data returned by couch. Other values (e.g. `total_rows`, `offset`, etc) are added to the returned lazy seq as metadata.
* elimination of inconsistency between APIs between `save-view` and `save-filter`.  The names of individual views and filters are now part of the map provided to these functions, instead of sometimes being provided separately.
* `:language` has been eliminated as part of the dynamically-bound configuration map
* `with-clj-view-server` has been replaced by the more generic `view-server-fns` macro, which takes a `:language` keyword or map of options that includes a `:language` slot (e.g. `:clojure`, `:javascript`, etc), and a map of view/filter/validator names => functions.
* A `view-transformer` multimethod is now available, which opens up clutch to dynamically support additional view server languages. 
* Moved `view-server-exec-string` to `com.ashafa.clutch.view-server` namespace

## Contributors

Appreciations go out to:

* http://cemerick.com[Chas Emerick]
* http://github.com/pierrel[Pierre Larochelle]
* http://github.com/mattdw[Matt Wilson]
* http://github.com/WizardofWestmarch[Patrick Sullivan]
* http://tbatchelli.org[Toni Batchelli]
* http://github.com/hugoduncan[Hugo Duncan]
* http://github.com/senior[Ryan Senior]
