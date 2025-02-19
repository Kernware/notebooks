# testing mysql with sqlite layer

The production application connects to a mysql database from a customer, for testing
the idea came up instead of running mysql in a docker container, just use sqlite
and let it create a local file for testing.

The application is a python application, the idea was to just redirect each
query to the sqlite db, cause both is sql so it should match with few small
adoptions.

To connect to the db `pymysql` is used, with a classic:
```Python
return pymysql.connect(
    host=os.environ["MYSQL_HOST"],
    port=int(os.environ["MYSQL_PORT"]),
    user=os.environ["MYSQL_USER"],
    password=os.environ["MYSQL_PASSWORD"],
    db=os.environ["MYSQL_DB"],
    cursorclass=pymysql.cursors.DictCursor,
)
```

The sqlite db should be directly created in the tmp folder and be deleted
```Python
self.tmp_db = tempfile.NamedTemporaryFile(
    delete=True, prefix="unittest_sqlite_"
)
self.conn = sqlite3.connect(self.tmp_db.name)
self.conn.row_factory = sqlite3.Row
cursor = self.conn.cursor()
```

Now the goal was to not modify the application code, if the environment variable
`UNIT_TEST` exists and is equal to one, the connection function return the sqlite
connection otherwise the mysql connection.

```Python
def get_mysql_connection():
    if "UNIT_TEST" in os.environ and os.environ["UNIT_TEST"] == "1":
        from .sqlite_wrapper import SqliteConnection
        return SqliteConnection(Path(__file__).parent / "initdb" / "ddl_sqlite.sql")

    return pymysql.connect(
        host=os.environ["MYSQL_HOST"],
        port=int(os.environ["MYSQL_PORT"]),
        user=os.environ["MYSQL_USER"],
        password=os.environ["MYSQL_PASSWORD"],
        db=os.environ["MYSQL_DB"],
        cursorclass=pymysql.cursors.DictCursor,
    )
```

The sqlite connection is not fully equal to a mysql connection so a wrapper around
it was build as a PoC. Writing the wrapper showed that there are more differences
than initially thought. The wrapper looked as follow:

```Python
import sqlite3
import tempfile
from pathlib import Path

class SqliteCursor:
    def __init__(self, sqlite_cursor):
        self.sqlite_cursor = sqlite_cursor

    def __enter__(self):
        return self

    def __exit__(self, type, value, tb):
        pass

    def execute(self, query: str, args = ()):
        query = query.replace("<scheme used>", "") # sqlite has no scheme's
        query = query.replace("%s", "?") # variables have different placeholder
        query = query.replace(
            "CONCAT(column1, '', column2),",
            "column1 || '' || column2,",
        )
        query = query.replace(
            "LEFT(column1, 2)",
            "SUBSTR(column2, 1, 2)",
        )
        # ...
        return self.sqlite_cursor.execute(query, args)

    def fetchone(self):
        return [dict(r) for r in self.sqlite_cursor.fetchone()]

    def fetchall(self):
        return [dict(r) for r in self.sqlite_cursor.fetchall()]


class SqliteConnection:
    def __init__(self, init_db_file: Path | None):
        self.tmp_db = tempfile.NamedTemporaryFile(
            delete=True, prefix="test_sqlite_"
        )
        self.conn = sqlite3.connect(self.tmp_db.name)
        self.conn.row_factory = sqlite3.Row
        cursor = self.conn.cursor()

        if init_db_file is not None and init_db_file.exists():
            cursor.executescript(init_db_file.read_text())
            self.conn.commit()

    def __enter__(self):
        return self

    def __exit__(self, type, value, tb):
        pass

    def cursor(self):
        return SqliteCursor(self.conn.cursor())
```

For mysql a dictionary cursor was used, that one does not exist on sqlite,
therefore the detour using `row_factory` has been used, this still requires to
convert every row returned into a dictionary, see `fetchone`, `fetchall`.

After these changes the application run fine, until some queries got executed.
They aren't even that special but the sqlite syntax is just slightly different.
For simplicity replace's have been added (not ideal but good enough for a PoC).
Even though after these replace's the application run successfully and testing
was easily done locally, it felt too much like a hack.

Given that the application anyhow had another docker container a mysql
compose entry and a test profile has been created for testing. Even though the
changes above are kind a nice, spinning up an extra local docker container for
testing requires zero code changes and local testing is still possible.


# Moral of the story
Being able to run an application locally is enormously important for developers,
to enable fast turn around times during bug fixing. Local testing should never
depend on external resources.

In the scenario above, having a single mysql connection, it would make sense to
use a sqlite file for local testing, starting with an empty db gradually adding
information and running the tests.

Given that the whole application has already a docker compose file, with a stage
and production profile it is easier to create a test profile which creates an
empty mysql instance, as this also will run locally, just docker is needed, but
every good developer has that installed anyway.

----
written 20.2.2025
