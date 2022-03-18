# GO Json scanner

**goj** is a small low-level JSON scanning library.  It is representation-free, providing
no in-memory representation (that's your job).

**goj** may be useful to you if the following are true:

**note** this is a soft-fork of [lloyd/goj](https://github.com/lloyd/goj) to get some PRs I was interested in.

1. you need fast json parsing
2. you do not need a streaming parser (the distinct JSON documents you are parsing
   are delimited in some fasion)
3. you either want to extract a subset of JSON documents, or have your own data
   representation in memory, or wish to transform JSON into a different format.

## Usage

Installation:
```
go get github.com/flowchartsman/goj
```

A program to extract and print the top level `.name` property from a json file passed in on stdin:

```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"

	"github.com/flowchartsman/goj"
)

func main() {
	buf, _ := ioutil.ReadAll(os.Stdin)

	p := goj.NewParser()

	depth := 0
	err := p.Parse(buf, func(t goj.Type, key []byte, value []byte) bool {
		switch t {
		case goj.String:
			if depth == 1 && string(key) == "name" {
				fmt.Printf("%s\n", string(value))
				return false
			}
		case goj.Array, goj.Object:
			depth++
		case goj.ArrayEnd, goj.ObjectEnd:
			depth--
		}
		return true
	})

	if err != nil && err != goj.ClientCancelledParse {
		fmt.Printf("error: %s\n", err)
	}
}
```

## Performance

All numbers below are on:
```
go version go1.11.1 linux/amd64
Intel(R) Xeon(R) CPU E5-2643 v4 @ 3.40GHz
```

Using the same JSON sample data as `encoding/json`, **goj** scans about 3x
faster than go's reflection based json parsing:

```
$ go test -bench . -run 'XXX'
goos: darwin
goarch: amd64
pkg: github.com/flowchartsman/goj
cpu: Intel(R) Core(TM) i9-9980HK CPU @ 2.40GHz
BenchmarkGojScanning-16                      325           3332903 ns/op         582.22 MB/s
BenchmarkGojOffsetScanning-16                397           3156122 ns/op         614.83 MB/s
BenchmarkStdJSONScanning-16                   98          10903834 ns/op         177.96 MB/s
PASS
ok      github.com/flowchartsman/goj    5.180s
```

See `test/bench_test.go` for the source.

Comparing against [jq](http://stedolan.github.io/jq/) (a tiny and awesome tool
written in C that extracts nested values from json data), **goj**
 is more than 4x faster.

```
$ go build example/main.go && time ./main < ~/4.9gb_sample.json > /dev/null
real 0m20.476s
user 0m18.838s
sys  0m1.734s

$ time jq -r .name < ~/4.9gb_sample.json > /dev/null
real   1m26.964s
user   1m25.515s
sys    0m1.372s
```
Compared against [yajl](https://github.com/lloyd/yajl) (a fast streaming json parser
written in C) in a fair fight, **goj** is about the same.

```
$ json_verify -s < ~/4.9gb_sample.json
...
real 0m14.504s
user 0m13.754s
sys  0m0.736s

$ go build cmd/prof/main.go && time ./main < ~/4.9gb_sample.json
...
real    0m14.171s
user    0m13.386s
sys     0m0.793s
```

## License

BSD 2 Clause, see `LICENSE`.
