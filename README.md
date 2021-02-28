# SQLite Plus: all the missing SQLite functions

SQLite has very few functions compared to other DBMS. SQLite authors see this as a feature rather than a bug, because SQLite has extension mechanism in place.

There are a lot of SQLite extensions out there, but they are incomplete, inconsistent and scattered across the internet.

SQLite Plus brings them all togeher, neatly packaged by domain modules and built for Linux, Windows and macOS.

Here is what we got right now:

## `sqlite3-stats`: statistics

Common statistical functions for SQLite.
Adapted from [extension-functions.c](https://sqlite.org/contrib/) by Liam Healy.

Provides following functions:

-   `mode` - mode,
-   `median` - median (50th percentile),
-   `percentile_25` - 25th percentile,
-   `percentile_75` - 75th percentile,
-   `percentile_90` - 90th percentile,
-   `percentile_95` - 95th percentile,
-   `percentile_99` - 99th percentile,
-   `stddev` or `stddev_samp` - sample standard deviation,
-   `stddev_pop` - population standard deviation,
-   `variance` or `var_samp` - sample variance,
-   `var_pop` - population variance.

CLI usage:

```
sqlite> .load ./sqlite3-stats.so;
sqlite> select median(value) from generate_series(1, 100);
```

In-app usage:

```python
import sqlite3

connection = sqlite3.connect(":memory:")
connection.enable_load_extension(True)
connection.load_extension("./sqlite3-stats.so")
connection.execute("select median(value) from generate_series(1, 100)")
connection.close()
```

## `sqlite3-vsv`: CSV files as virtual tables

Provides virtual table for working directly with CSV files, without importing data into the database. Useful for very large datasets.

Adapted from [vsv.c](http://www.dessus.com/files/vsv.c) by Keith Medcalf.

Usage:

```sql
create virtual table temp.vsv using vsv(...);
select * from vsv;
```

The parameters to the vsv module (the vsv(...) part) are as follows:

```
filename=STRING     the filename, passed to the Operating System
data=STRING         alternative data
schema=STRING       Alternate Schema to use
columns=N           columns parsed from the VSV file
header=BOOL         whether or not a header row is present
skip=N              number of leading data rows to skip
rsep=STRING         record separator
fsep=STRING         field separator
validatetext=BOOL   validate UTF-8 encoding of text fields
affinity=AFFINITY    affinity to apply to each returned value
nulls=BOOL          empty fields are returned as NULL
```

Defaults:

```
filename / data     nothing.  You must provide one or the other
                    it is an error to provide both or neither

schema              nothing.  If not provided then one will be
                    generated for you from the header, or if no
                    header is available then autogenerated using
                    field names manufactured as cX where X is the
                    column number

columns             nothing.  If not specified then the number of
                    columns is determined by counting the fields
                    in the first record of the VSV file (which
                    will be the header row if header is specified),
                    the number of columns is not parsed from the
                    schema even if one is provided

header=no           no header row in the VSV file
skip=0              do not skip any data rows in the VSV file
fsep=','            default field separator is a comma
rsep='\n'           default record separator is a newline
validatetext=no     do not validate text field encoding
affinity=none       do not apply affinity to each returned value
nulls=off           empty fields returned as zero-length
```

Parameter types:

-   `STRING` means a quoted string
-   `N` means a whole number not containing a sign
-   `BOOL` means something that evaluates as true or false. Case insensitive: `yes`, `no`, `true`, `false`, `1`, `0`. Defaults to `true`
-   `AFFINITY` means an SQLite3 type specification. Case insensitive: `none`, `blob`, `text`, `integer`, `real`, `numeric`
-   STRING means a quoted string. The quote character may be either
    a single quote or a double quote. Two quote characters in a row
    will be replaced with a single quote character. STRINGS do not
    need to be quoted if it is obvious where they begin and end
    (that is, they do not contain a comma). Leading and trailing
    spaces will be trimmed from unquoted strings.

The `separator` string containing exactly one character, or a valid
escape sequence. Recognized escape sequences are:

```
\t horizontal tab, ascii character 9 (0x09)
\n linefeed, ascii character 10 (0x0a)
\v vertical tab, ascii character 11 (0x0b)
\f form feed, ascii character 12 (0x0c)
\xhh specific byte where hh is hexadecimal
```

The `validatetext` setting will cause the validity of the field
encoding (not its contents) to be verified. It effects how
fields that are supposed to contain text will be returned to
the SQLite3 library in order to prevent invalid utf8 data from
being stored or processed as if it were valid utf8 text.

The `nulls` option will cause fields that do not contain anything
to return NULL rather than an empty result. Two separators
side-by-each with no intervening characters at all will be
returned as NULL if nulls is true and if nulls is false or
the contents are explicity empty ("") then a 0 length blob
(if affinity=blob) or 0 length text string.

For the `affinity` setting, the following processing is applied to
each value returned by the VSV virtual table:

-   `none` no affinity is applied, all fields will be
    returned as text just like in the original
    csv module, embedded nulls will terminate
    the text. if validatetext is in effect then
    an error will be thrown if the field does
    not contain validly encoded text or contains
    embedded nulls
-   `blob` all fields will be returned as blobs
    validatetext has no effect
-   `text` all fields will be returned as text just
    like in the original csv module, embedded
    nulls will terminate the text.
    if validatetext is in effect then a blob
    will be returned if the field does not
    contain validly encoded text or the field
    contains embedded nulls
-   `integer` if the field data looks like an integer,
    (regex "^ _(\+|-)?\d+ _$"),
    then an integer will be returned as
    provided by the compiler and platform
    runtime strtoll function
    otherwise the field will be processed as
    text as defined above
-   `real` if the field data looks like a number,
    (regex "^ _(\+|-)?(\d+\.?\d_|\d*\.?\d+)([eE](+|-)?\d+)? *$")
    then a double will be returned as
    provided by the compiler and platform
    runtime strtold function otherwise the
    field will be processed as text as
    defined above
-   `numeric` if the field looks like an integer
    (see integer above) that integer will be
    returned; if the field looks like a number
    (see real above) then the number will
    returned as an integer if it has no
    fractional part; otherwise a double will be returned
