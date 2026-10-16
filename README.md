# Timestamp Format Parser Differential

Paste timestamp strings and see, row by row, whether RFC 3339 (as updated by RFC 9557), the ECMAScript Date Time String Format, your browser's own Date.parse (run live), Python 3.14 datetime.fromisoformat and Go time.Parse(time.RFC3339) accept each one, and which instant each produces.

**Live demo:** https://0xelitesystem.github.io/timestamp-format-parser-differential/

## Use

1. Paste timestamps into the box, one per line (up to 500 lines). Empty lines are skipped. Nothing is trimmed, because a stray space or byte order mark is part of the string every parser sees.
2. Read the table. Each row has five columns:
   - **RFC 3339 (as updated by RFC 9557).** A validator for the section 5.6 ABNF and the 5.7 restrictions. Lower case `t` and `z` are valid (5.6 NOTE). A space separator is shown as "outside the ABNF; permitted by the NOTE as an application choice", never as plain valid or plain invalid. Second 60 is grammatical and flagged, because only an announced leap second makes it real. `-00:00` is grammatical, and the cell says RFC 9557 section 2.2 now uses `Z` for an unknown local offset.
   - **ECMA-262 Date Time String Format.** A validator for 21.4.1.32 and the instant rule of 21.4.3.2: with no offset, a date-only form is UTC and a date-time form is local time.
   - **Your browser, live.** `Date.parse` in your own engine, shown as an ISO string or NaN. Where the string conforms to the ECMA format, the cell flags a result that differs from the format's rule. Outside the format the result is implementation-defined, so it describes your engine only.
   - **Python 3.14 rules** and **Go time.Parse(time.RFC3339) rules.** Emulations of documented and source-read behaviour. They are not executions.
3. Pick a zone for the ECMA column's local-time reading. `Date.parse` itself always uses the host zone, and a page cannot change that zone for it.
4. Tick **Show parsed fields** to see the fields each reader extracted.

When two or more accepting columns produce different instants, the row says so and lists them.

### Use cases

- **An API emits timestamps that a Go service rejects.** Paste a sample of the payload. Go's RFC3339 layout needs an upper case `T`, an upper case `Z` or a colon offset, so `2024-01-15t10:00:00z` passes RFC 3339 and fails in Go.
- **A date-only string shifts by a day in the browser.** `2024-01-15` is UTC midnight under ECMA-262, while Python returns a naive datetime with no instant at all. The row shows both.
- **Choosing what to emit.** Load the examples and look for the rows every column accepts with the same instant: full date, upper case `T`, seconds, an upper case `Z` or a `+hh:mm` offset.
- **Reviewing a parser change.** Paste the strings from a bug report and see which reader each one depends on.

## Why this exists

Each runtime follows its own published rule. ECMA-262 says date-only forms are UTC and date-time forms without an offset are local; Python returns a naive datetime for both; Go's RFC3339 layout rejects the lower case `t` and `z` that RFC 3339 allows, and accepts an offset of `+24:00`, which RFC 3339 does not. None of these is a bug, and a payload that crosses between them changes meaning. This page puts the five readings side by side so the difference is visible before it ships.

For which strings belong to RFC 3339, ISO 8601 or HTML as formats, see https://ijmacd.github.io/rfc3339-iso8601/ (an independent project that compares format membership).

Single file, no build step, no network call, MIT licensed. Nothing to install, and nothing watching what you paste.

## Privacy

Everything runs in your browser. There is no network call of any kind: no request for code, fonts, analytics or telemetry, and nothing you paste is transmitted anywhere. A Content-Security-Policy in the page blocks every connection. The only thing stored is your light or dark theme choice, in your own browser's localStorage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/timestamp-format-parser-differential.git
cd timestamp-format-parser-differential
```

Open `index.html` in a browser. If you would rather serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and inline JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## How the emulations were checked

The Python column follows the documented exceptions in the `datetime.fromisoformat` documentation and the behaviour of CPython's C implementation. The Go column follows `parseRFC3339` in `src/time/format_rfc3339.go` (the fast path) and the general `parse` in `src/time/format.go` (the fallback). Both were run against the real runtimes on 2026-10-16: Python 3.14.8 locally and Go go1.27.2 on the Go playground, over the page's worked examples plus generated mutations of them, and the emulations matched every verdict and instant. The page's self test asserts the worked examples.

Python behaviour drifts across releases. Measured on 2026-10-16: Python 3.11.9 rejects `2024-01-15T24:00:00Z` ("hour must be in 0..23") and 3.14.8 accepts it as `2024-01-16T00:00:00+00:00`; the CPython changelog entry is gh-102450. Python 3.11.9 returns `10:00:00.500000` for `2024-01-15T10.5`, which 3.14.8 rejects. Rows with those shapes carry a version note.

To reproduce, save and run these two loops. Each prints one line per string.

`py.py` (run with `python3 py.py`):

```python
import sys
from datetime import datetime

CORPUS = [
    "2024-01-15", "2024-01-15T10:00:00Z", "2024-01-15T10:00:00",
    "2024-01-15 10:00:00Z", "2024-01-15t10:00:00Z", "2024-01-15T10:00:00z",
    "2024-01-15T10:00:00,5Z", "2024-01-15T10:00:00.123456789Z",
    "2024-01-15T10:00:00-05", "2024-01-15T10:00:00-0500", "2024-01-15T10:00:00-05:00",
    "2024-W03-1", "2024-015", "20240115T100000Z", "2024-01-15T24:00:00Z",
    "2024-01-15T10:00Z", "2024-01-15T10:00:00-00:00", "2024-01-15T10:00:00+24:00",
    "2024-01-15T10:00:00+05:60", "2024-01-15T1:00:00Z", "2016-12-31T23:59:60Z",
    "2024-02-30T10:00:00Z", "2024-01", "+002024-01-15T10:00:00Z",
    "2024-01-15T10:00:00.5+05:30", "2024-01-15T10.5",
]
print(sys.version)
for s in CORPUS:
    try:
        print(repr(s), "->", datetime.fromisoformat(s).isoformat())
    except ValueError as e:
        print(repr(s), "-> ERR", e)
```

`main.go` (run with `go run main.go`, or paste it into the Go playground):

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

var corpus = []string{
	"2024-01-15", "2024-01-15T10:00:00Z", "2024-01-15T10:00:00",
	"2024-01-15 10:00:00Z", "2024-01-15t10:00:00Z", "2024-01-15T10:00:00z",
	"2024-01-15T10:00:00,5Z", "2024-01-15T10:00:00.123456789Z",
	"2024-01-15T10:00:00-05", "2024-01-15T10:00:00-0500", "2024-01-15T10:00:00-05:00",
	"2024-W03-1", "2024-015", "20240115T100000Z", "2024-01-15T24:00:00Z",
	"2024-01-15T10:00Z", "2024-01-15T10:00:00-00:00", "2024-01-15T10:00:00+24:00",
	"2024-01-15T10:00:00+05:60", "2024-01-15T1:00:00Z", "2016-12-31T23:59:60Z",
	"2024-02-30T10:00:00Z", "2024-01", "+002024-01-15T10:00:00Z",
	"2024-01-15T10:00:00.5+05:30",
}

func main() {
	fmt.Println(runtime.Version())
	for _, s := range corpus {
		t, err := time.Parse(time.RFC3339, s)
		if err != nil {
			fmt.Printf("%q -> ERR %v\n", s, err)
		} else {
			fmt.Printf("%q -> %s\n", s, t.UTC().Format(time.RFC3339Nano))
		}
	}
}
```

## Sources

Re-read on 2026-10-16.

- RFC 3339, Date and Time on the Internet: Timestamps (July 2002), sections 5.6 and 5.7: https://www.rfc-editor.org/rfc/rfc3339.txt
- RFC 3339 metadata, `updated_by: ['RFC9557']`: https://www.rfc-editor.org/rfc/rfc3339.json
- RFC 9557, section 2.2, Update to RFC 3339: https://www.rfc-editor.org/rfc/rfc9557.txt
- ECMA-262, 21.4.1.32 Date Time String Format and 21.4.3.2 Date.parse (the page read was dated "Draft ECMA-262 / October 15, 2026"): https://tc39.es/ecma262/#sec-date-time-string-format and https://tc39.es/ecma262/#sec-date.parse
- MDN, Date.parse: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse
- Python, `datetime.fromisoformat`: https://docs.python.org/3/library/datetime.html, and the changelog entry gh-102450: https://docs.python.org/3/whatsnew/changelog.html
- Go, `src/time/format.go` and `src/time/format_rfc3339.go`: https://raw.githubusercontent.com/golang/go/master/src/time/format.go and https://raw.githubusercontent.com/golang/go/master/src/time/format_rfc3339.go

## Independence

This is an independent tool. It is not affiliated with or endorsed by the IETF, Ecma International, the Python Software Foundation or the Go project. Their documents are cited only as the rules each column follows.

## Third-party notices

Python and Go columns are emulations of documented and source-read behaviour, checked against Python 3.14.8 and Go 1.27.2; no code is reproduced. Behaviour was measured empirically and from the published sources.

- RFC 3339 is quoted on the page as the rule behind each verdict. It carries: "Copyright (C) The Internet Society (2002).  All Rights Reserved."
- RFC 9557 is quoted on the page. It carries: "Copyright (c) 2024 IETF Trust and the persons identified as the document authors.  All rights reserved."
- ECMA-262 is paraphrased, not quoted. The specification is © 2026 Ecma International.
- The Python documentation is paraphrased, not quoted.
- Two comment sentences from the Go source are quoted on the page. Go is BSD-3-Clause licensed, "Copyright 2009 The Go Authors." (https://raw.githubusercontent.com/golang/go/master/LICENSE).

## Related

- https://github.com/0xelitesystem/date-format-pattern-translator
- https://github.com/0xelitesystem/dst-ambiguous-time-converter
- https://github.com/0xelitesystem/unix-timestamp-converter

## License

MIT. Copyright (c) 2026 0xelitesystem.
