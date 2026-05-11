# jo: full option reference

Authoritative source:
the `jo(1)` man page in https://github.com/jpmens/jo
(`jo.md` upstream).
This file is a quick lookup table —
when behaviour is ambiguous,
trust the man page.

The synopsis is:

```
jo [-p] [-a] [-B] [-D] [-e] [-n] [-o outfile] [-v] [-V]
   [-d keydelim] [-f file] [--] [ [-s|-n|-b] word ...]
```

## Global flags (before any word)

### `-p` — pretty-print

Adds indentation and newlines to the output.
Useful for human inspection;
drop it for production payloads
(the wire format should stay terse).

```sh
$ jo -p name=jo n=17 parser=false
{
   "name": "jo",
   "n": 17,
   "parser": false
}
```

### `-a` — emit an array

Treat the word list as array elements
instead of object key/value pairs.

```sh
$ jo -a a b c
["a","b","c"]
```

### `-B` — disable true/false/null literal detection

By default `jo` recognises the literal strings
`true`, `false`, and `null`
as their JSON equivalents.
`-B` turns this off,
so those strings stay as strings.

```sh
$ jo  switch=true off=null
{"switch":true,"off":null}

$ jo -B switch=true off=null
{"switch":"true","off":"null"}
```

### `-D` — deduplicate object keys

By default,
duplicate keys are appended verbatim
(producing invalid-ish but RFC-permitted JSON).
With `-D`,
the last value for each key wins.

```sh
$ jo    a=1 b=2 a=3
{"a":1,"b":2,"a":3}

$ jo -D a=1 b=2 a=3
{"a":3,"b":2}
```

### `-n` (global) — omit empty values

With `-n` placed **before** the word list,
keys whose value is empty are skipped entirely
instead of becoming `null`.

```sh
$ jo    a=1 b= c=3
{"a":1,"b":null,"c":3}

$ jo -n a=1 b= c=3
{"a":1,"c":3}
```

**Name clash:**
`-n` is also the *per-word* coercion to number
when it appears **after** `--`.
Position decides meaning.

### `-d sep` — object-path separator

Sets the character that splits a key into a nested object path.
Most idiomatic value is `.`,
which lets you write `geo.lat=10` instead of `'geo[lat]=10'`.

```sh
$ jo -d. geo.lat=10 geo.lon=20
{"geo":{"lat":10,"lon":20}}
```

`-d` paths share the same one-level depth limit as bracket forms;
for deeper nesting use nested `jo` substitution.

### `-f file` — load a base JSON document

Reads `file` as the starting JSON object or array,
then applies the rest of the word list on top
(adding or replacing keys).
`-` reads the base from stdin.

```sh
$ echo '{"x":1,"y":2}' | jo -f - z=3
{"x":1,"y":2,"z":3}
```

Combine with a pipe to amend an upstream response:

```sh
curl -sSL https://api.example.com/v1/config \
    | jo -f - updated_at="$(date -Iseconds)"
```

### `-e` — ignore empty stdin

Without `-e`,
`jo` blocks reading stdin if it's a terminal,
or errors on an empty pipe.
With `-e`,
an empty stdin is silently accepted.
Useful when an outer pipeline may or may not feed `jo`.

### `-o file` — write to file instead of stdout

Equivalent to `jo ... > file`,
but the redirection happens inside `jo`
(handy when stdout is otherwise busy).

### `-v` / `-V` — version

`-v` prints the version string.
`-V` prints the version as a JSON object
(useful for embedding in CI metadata).

## Per-word coercion (after `--`)

Place `--` before the word list to end global-option parsing,
then prefix individual words with `-s`/`-n`/`-b`
to override the auto-detected type.

| Prefix | Coerces to | Empty value | Non-matching value |
|--------|------------|-------------|--------------------|
| `-s`   | string     | `""`        | unchanged          |
| `-n`   | number     | `0`         | string *length*    |
| `-b`   | boolean    | `false`     | `true`             |

```sh
$ jo -- name=alice -s zip=02118 -n digits="four"
{"name":"alice","zip":"02118","digits":4}
```

Works inside `-a` arrays too:

```sh
$ jo -a -- -s 123 hello -n "five"
["123","hello",4]
```

`--` is documented as mandatory before the word list;
in practice `jo` is lenient if the first word does not start with `-`,
but pass `--` always for clarity and forward compatibility.

## End-of-options marker

### `--`

Ends global-option parsing.
Required before per-word `-s`/`-n`/`-b` coercion
to disambiguate them from the global `-n`.
After `--`,
every dash-prefixed token is interpreted as a per-word coercion,
not a global flag.
