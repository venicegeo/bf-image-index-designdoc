Design for `bf-ia-broker` Self-hosted Image Index
===

This is a draft of design suggestions for the upcoming self-hosted image index
within `bf-ia-broker`.

Contents
---

-   [Postgres Database Connection](#postgres-database-connection)
    -   [Connection handling](#connection-handling)
    -   [Schema management](#schema-management)
    -   [PostGIS support](#postgis-support)
    -   [Database best practices](#database-practices)
-   [Runtime Modes](#runtime-modes)
    -   [`serve` and `ingest_landsat`](#serve-ingest)
    -   [`migrate`](#migrate)
-   [REST API](#rest-api)
    -   [Endpoints](#endpoints)

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

#### PostGIS support

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


<a id="database-practices"/>

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
| `migrate`        | <code>\[ up     &#124; down MIGRATION_NUMBER ]</code>`                                                                        | apply the proper upgrade/downgrade migration scripts to the database defined in VCAP |

Delegating to the right piece of code would be done with the `urfave/cli`
library, which is already in use to route between `version` and `serve`. Most
notably, this means that all IA Broker code would continue to be packaged as a
_single_ executable, which simplifies the `cf push` process.


<a id="serve-ingest"/>

#### Long-running Processes: `serve` and `ingest_landsat`

These two processes need to start at `cf start` and keep running. Keeping them
as separate processes means we can scale one while keeping the other constant,
or monitor them separately (both in logs and in the PCF UI). Since they use the
same binary image ("droplet" in CF parlance), we can accomplish this by using
multiple CF process types in our `Procfile`. This process is documented
[here](https://www.cloudfoundry.org/blog/build-cf-push-learn-procfiles/). As
such, our new Procfile would look like:

```
web: bf-ia-broker serve
ingest: bf-ia-broker ingest_landsat
```

Assuming Pivotal CloudFoundry implements the CF specs appropriately, this
should start both processes in separate containers, and monitor each one
appropriately &mdash; the web one via HTTP, and the non-web one via Unix
process.

<a id="migrate"/>

#### One-off Process: `migrate`

`migrate` poses a challenge. As a one-off script, it should be run using
`cf run-task` and the appropriate droplet. However, it must:

*   start _after_ the old `serve` and `ingest_landsat` processes shut down,
    so there is no chance of a race condition during migration
*   complete _before_ the main `serve` and `ingest_landsat` processes start,
    so they get their expected database schemas
*   supply output regarding the version to revert to in case of failure, so
    that...
*   in the event of failure of `cf start`, it can run again _before_ the old
    version of the broker processes gets relaunched

This means a thorough overhaul of the logic in our JenkinsFile, which
introduces a short downtime while the migration runs. The new logic of the
"Deploy" step looks something looks like this:

```groovy
current_app = sh 'cf apps | grep "bf-ia-broker"'
new_app = "bf-ia-broker-" + version

sh 'cf push ${new_app} --no-start /*other args*/'
sh 'cf set-env ${new_app} SPACE ${PCF_SPACE}'
sh 'cf set-env ${new_app} DOMAIN ${PCF_DOMAIN}'
// More `cf set-env`

sh 'cf stop ${current_app}'

try {
  db_rollback_version = sh 'cf run-task ${new_app} "bf-ia-broker migrate up"'
} catch { // Problem migrating, start old version of app and fail
  sh 'cf start ${current_app}'
  exit 'migration failed!'
}

try {
  sh 'cf start ${new_app}'
} catch { // Problem starting new app, downgrade DB, restart old version and fail
  try {
    sh 'cf run-task ${new_app} "bf-ia-broker migrate down ${db_rollback_version}"'
    sh 'cf delete ${new_app}'
    sh 'cf start ${current_app}'
    exit 'starting new version failed!'
  } catch { // Rollback failed, everything is terrible :(
    exit 'new version start and rollback failed!'
  }
}
sh 'cf delete ${current_app}''

// Handle route updates to use new_app
```

This change would be neessary both in the regular `JenkinsFile` and in
`JenkinsFile.Promote`. It could be somewhat avoided/simplified if we wanted to
either assume the risk of race conditions at launch time.

> **Note:** `bf-api` currently performs Liquibase migrations at server startup,
> and never rolls back on a failed start, which can lead to a corrupted database
> on a failed deploy. It should receive a similar enhancement.

REST API
---

Like the original Planet-based broker functionality, the new image index will
be accessed via a REST API.

#### Endpoints

The Planet layer of the broker currently provides the following endpoints:

| Endpoint | Output | Purpose |
| -------- | ------ | ------- |
| `/planet/discover/<sourceType>`  | `geojson.FeatureCollection`  | Searching imagery of multiple different sources  |
| `/planet/activate/<sourceType>/<id>`   | `geojson.Feature`  | Triggering availability "activation" of a Rapideye/Planetscope image  |
| `/planet/<sourceType>/<id>`  | `geojson.Feature`  | Metadata retrieval for a single image  |

The new Landsat image index can mirror the same GeoJSON-based search and metadata
system that the Planet endpoints are using. Mirroring the activation step is
unnecessary, as all Landsat images are already "active" in an S3 bucket.

As such, the new endpoints would be:

| Endpoint | Output | Purpose |
| -------- | ------ | ------- |
| `/local_index/discover/<sourceType>` | `geojson.FeatureCollection`  |  Searching imagery in the local index (currently `landsat` only for source type) |
| `/local_index/<sourceType>/<id>` | `geojson.FeatureCollection`  | Metadata retrieval from the local index (currently `landsat` only for source type) |

#### Response data (and future-proof consistency/compatibility)

The endpoint return data is more complicated. The results from searching planet
are not mere GeoJSON Features/FeatureCollections; they have several specific
important metadata set in the GeoJSON `properties` that make them actually
useful. This response schema is not currently standardized in any way, and the
way the GeoJSON is built is a little
[opaque and arcane](https://github.com/venicegeo/bf-ia-broker/blob/58a0582/planet/planet.go#L386-L475).

For these reasons, a refactor of the response structure is due, improving
both old Planet code and providing a reasonable common interface for future
other data sources.

A common interface for building common broker responses can be accomplished
by using Go type renaming on top of `geojson.FeatureCollection` and
`geojson.Feature` to wrap GeoJSON interactions in friendlier functions. For
example, we can define:

```go
type BrokerMultiResult geojson.FeatureCollection

func NewBrokerMultiResult() *BrokerMultiResult {
  return geojson.NewFeatureCollection(nil)
}
func (mr *BrokerMultiResult) Append(results ...[]BrokerResult) {
  mr.Features = append(mr.Features, results...)
}

type BrokerResult geojson.Feature

func NewBrokerResult() *BrokerResult {
  return geojson.NewFeature(nil, nil, nil)
}

func (br *BrokerResult) SetGeometry(geometry interface{}) {
  br.Geometry = geometry
  br.Bbox = br.ForceBbox()
}

func (br *BrokerResult) SetCloudCover(cloudCover float64) {
  br.Properties["cloud_cover"] = cloudCover
}

// ... etc
```

This `BrokerResult` type can then be further renamed to add more specific
"subclass" features, such as being able to inject Planet asset metadata:

```go
type PlanetBrokerResult BrokerResult

func (br *PlanetBrokerResult) InjectAssetMetadata(asset planet.Asset) {
  if asset.ExpiresAt != "" {
		br.Properties["expires_at"] = asset.ExpiresAt
	}
	if asset.Location != "" {
		br.Properties["location"] = asset.Location
	}
	if len(asset.Permissions) > 0 {
		br.Properties["permissions"] = asset.Permissions
	}
	if asset.Status != "" {
		br.Properties["status"] = asset.Status
	}
	if asset.Type != "" {
		br.Properties["type"] = asset.Type
	}
}
```

> [Old not-typesafe function operating directly on GeoJSON](https://github.com/venicegeo/bf-ia-broker/blob/58a0582/planet/planet.go#L458-L475)

Or implement source-specific behavior to add custom properties:

```go
type LandsatS3BrokerResult BrokerResult

func (br LandstS3BrokerResult) InferS3Bands() error {
  id := br.ID.(string)
  if !landsat.IsValidLandSatID(id) {
		return errors.New("Not a valid LandSat ID: " + landSatID)
	}

	awsFolder, prefix, err := landsat.GetSceneFolderURL(landSatID, dataType)
	if err != nil {
		return err
	}

	bands := make(map[string]string)
	for band, suffix := range landSatBandsSuffixes {
		bands[band] = awsFolder + prefix + suffix
	}
	br.Properties["bands"] = bands

	return nil
}
```

> [Old not-typesafe function not encapsulated into any data structure](https://github.com/venicegeo/bf-ia-broker/blob/58a0582/planet/planet.go#L458-L475)

This approach takes advantage of Go's strong typing to rigidly define what our
results are, what they can do, and how various pieces of them are constructed.
Contrasted with the current approach of various data conversion/building
functions scattered across multiple files and packages.

A potential "heirarchy" of these renames, and the provided functionality:

<img src="data:image/png;base64,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" style="cursor:pointer;max-width:100%;" onclick="(function(img){if(img.wnd!=null&&!img.wnd.closed){img.wnd.focus();}else{var r=function(evt){if(evt.data=='ready'&&evt.source==img.wnd){img.wnd.postMessage(decodeURIComponent(img.getAttribute('src')),'*');window.removeEventListener('message',r);}};window.addEventListener('message',r);img.wnd=window.open('https://www.draw.io/?client=1&lightbox=1&edit=_blank');}})(this);"/>