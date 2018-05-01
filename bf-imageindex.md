Design for `bf-ia-broker` Self-hosted Image Index
===

This is a draft of design suggestions for the upcoming self-hosted image index
within `bf-ia-broker`.


Postgres Database Connection
---

Go has a standard interface for connecting with any SQL database in the form
of its builtin `database/sql` ([docs](https://godoc.org/database/sql)) package. However, to actually connect to each
individual type of database, a driver is necessary. The most popular and stable
Go driver for PostgreSQL is [`github.com/lib/pq`](https://github.com/lib/pq)
([docs](https://godoc.org/github.com/lib/pq)). Once we inject the Postgres
server connection URL (using PCF VCAP), establishing the actual connection
is trivial.

#### Connection handling

`database/sql` handles connection pooling and recycling itself. We can
instantiate a global `sql.DB` object at the start of runtime, and it will
handle connection pooling/reuse by itself. The DB object has several functions
for configuring the number of simultaneous connections involved, connection
life length, etc.

For each specific "session" (typically, each request), we should open a new
transaction (`sql.Tx`) using `db.Begin()`. This ensures sanity of the
potentially multiple SQL queries to be done, and provides rollback/commit
features.

To check that there is at least one database connection alive (and create one,
if necessary), we can use `db.Ping()`. This can help if done on a timer to make
sure we always have the database available and catch any outages ASAP.

The DB object, and any database-related interactions, should be implemented in
a "common" package, which can then be appropriately used by the various runtime
modes of `bf-ia-broker` (see the [Runtime Modes](#runtime-modes) section). This would ensure the
code is DRY and easily maintainable.

#### Schema management

Same as `bf-api` uses Liquibase to maintain its table schemas, `bf-ia-broker`
must have a solution for keeping track of its schemas and perform migrations.
A popular and well-maintained library to do this using native Go code is
[`github.com/pressly/goose`](https://github.com/pressly/goose)
([docs](http://godoc.org/github.com/pressly/goose)).

The best way for us to use it is to invoke it directly from Go code, not from
the command line. By passing the migration library a database transaction
produced from our common connection code, we can ensure consistency in the way
queries are done (similar to what Java Spring accomplishes with a global
`DataSource`).

The migration process should be invoked as one-off tasks during the Jenkins
pipeline's "Deploy" step. They should be called using `cf run-task` after the
`cf push` and `cf set-env` commands, but before `cf start` is called.  If
`cf start` fails, then `cf run-task` can be called again with a migration
rollback.

More details on the migration runtime mode can be found under [Runtime Modes](#runtime-modes)
below.

#### PostGIS support (!)

By default, the `github.com/lib/pq` driver handles unknown types like so:

> All other types are returned directly from the backend as []byte values in text format.

This is suboptimal, since our tables must depend on types such as the PostGIS
`GEOMETRY` type. Having some sort of way to use these types without needing to
handle the conversions ourselves is important.

There is a library that claims to do what we need: [`github.com/cridenour/go-postgis`](https://github.com/cridenour/go-postgis).
However, it has not seen contributions since 2 years ago, and development seems
to be otherwise incomplete and stalled.

_If we find no suitable solutions, we may need to roll our own._ If we do, the
best way would be to create our own implementation of
[`sql.Scanner`](https://godoc.org/database/sql#Scanner) for each PostGIS
datatype we need.

#### Go database best practices (plus testing!)

To achieve separation of concerns &mdash; and all its associated benefits &mdash;
database queries and models should be implemented in their own, separate
package. That package should then contain a variety of code like this
(if this were hypothetically a blog):

```go
func CreateBlogPost(tx *sql.Tx, bp BlogPost) error {
  _, err := tx.Exec(
    "INSERT INTO blog_posts (title, timestamp, author, text) VALUES ($1, $2, $3, $4)",
    bp.Title, bp.Timestamp, bp.Author, bp.Text,
  )
  return err
}

func GetBlogPostsByAuthor(tx *sql.Tx, author string) ([]BlogPost, error) {
  rows, err := tx.Query(`
    SELECT p.title, p.timestamp, p.author, p.text
      FROM blog_posts p
     WHERE p.author = $1
     ORDER BY p.timestamp DESC
  `, author)
  if err != nil {
    return nil, err
  }

  results := []BlogPost{}
  for rows.Next() {
    bp := BlogPost{}
    err = rows.Scan(&bp.Title, &bp.Timestamp, &bp.Author, &bp.Text)
    if err != nil {
      return nil, err
    }
    results = append(results, bp)
  }
  return results, err
}
```

> **Note:** this is approximate code and might not actually be functional

The code using this can then mock this by storing the function as a variable
that can be replaced at test time:

```go
// title_list.go
var getBlogPostsByAuthor = brokerdb.GetBlogPostsByAuthor

func GetBlogTitleList(tx *sql.Tx, author string) ([]string, error) {
  posts, err := getBlogPostsByAuthor()
  if err != nil {
    return nil, err
  }
  titles := []string{}
  for _, post := range posts {
    titles = append(titles, post.Title)
  }
  return titles, nil
}

// title_list_test.go

func TestGetBlogTitleList_Success(t *testing.T) {
  // Mock
  getBlogPostsByAuthor = func(*sql.Tx, string) ([]BlogPost, error){
    return []BlogPost{
      BlogPost{Title: "Hello World", Timestamp: time.Now(), Author: "filip", Body: "blah blah"},
      BlogPost{Title: "Foo Bar", Timestamp: time.Now(), Author: "filip", Body: "blah blah"},
    }, nil
  }

  // Tested code
  titles, err := GetBlogTitleList(nil, "filip")

  // Asserts
  assert.Nil(t, err)
  assert.Equal(t, []string{"Hello World", "Foo Bar"}, titles)
}

func TestGetBlogTitleList_Error(t *testing.T) {
  // Mock
  getBlogPostsByAuthor = func(*sql.Tx, string) ([]BlogPost, error){
    nil, errors.New("mock database error")
  }

  // Tested code
  _, err := GetBlogTitleList(nil, "filip")

  // Asserts
  assert.NotNil(t, err)
  assert.Contains(t, err.Error(), "mock database error")
}
```

Runtime Modes
---

Beachfront 1.0 and 2.0 implementations of `bf-ia-broker` featured it as a
single executable binary that ran as a HTTP microservice. This was launched by
running `bf-ia-broker serve`, and configured itself using the environment.
It had another (virtually unused) command that just printed its version as well:
`bf-ia-broker version`. These "subcommands" &mdash; serve and version &mdash;
can be considered to be different "runtime modes" that the code runs under; one
launches a web server, while the other just prints some stuff and exits.

We should extend this subcommand structure to support some new runtime modes,
with specific purposes that are wise to segregate from the main web process.
The full list of modes would then be:

| Subcommand       | Arguments | Purpose                                                                                         |                                                                                      |
|------------------|-----------|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `version`        |           | print the version and exit                                                                      |                                                                                      |
| `serve`          |           | run the ia-broker web service                                                                   |                                                                                      |
| `ingest_landsat` |           | on an interval, parse the scenes CSV found at `LANDSAT_HOST` and index the images it references |                                                                                      |
| `migrate`        | `[ up     | down MIGRATION_NUMBER ]`                                                                        | apply the proper upgrade/downgrade migration scripts to the database defined in VCAP |

