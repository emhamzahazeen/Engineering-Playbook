# Go

## Overview

Go aka Golang is a programming language designed by folks at Google.

This guide provides some general resources for working in Go.
[Web Application in Go](../../web/server/go.md)
provides some specifics to web development.

## Learning Resources

### References

* [The Go Programming Language Official Website](https://golang.org/)
* [Effective Go](https://golang.org/doc/effective_go.html) (how to do things “the Go way”)
* [Go Pointer Primer](https://github.com/trussworks/go-pointer-primer)
* [pkg.go.dev](https://pkg.go.dev/) (where you can read the docs for nearly any Go package)
* [Go wiki](https://github.com/golang/go/wiki/Learn)
* _Book_: [The Go Programming Language](http://www.gopl.io/)
* Advanced Testing with Go
  [Video](https://www.youtube.com/watch?v=yszygk1cpEc)
  and [Article](https://about.sourcegraph.com/go/advanced-testing-in-go) (great overview of useful techniques, useful for all Go programmers)
* [Go Proverbs](https://go-proverbs.github.io/)
* [Line of Sight Go](https://medium.com/@matryer/line-of-sight-in-code-186dd7cdea88)
* [Go for Industrial Programming](https://peter.bourgon.org/go-for-industrial-programming/)

### Tours/Lessons

* [A Tour of Go](https://tour.golang.org) (in-browser interactive language tutorial)
* [How to Write Go Code](https://golang.org/doc/code.html) (info about the Go environment, testing, etc.)
* [Go by Example](https://gobyexample.com)
* [Go Track on Exercism](https://exercism.io/tracks/go)
* _Article_: [Copying data from S3 to EBS 30x faster using Golang](https://medium.com/@venks.sa/copying-data-from-s3-to-ebs-30x-faster-using-go-e2cdb1093284)

## Testing

### General

* Use [table-driven tests](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) where appropriate.
* Make judicious use of helper functions so that the intent of a test is not lost in a sea of error checking and boilerplate.
* Comments delineating the [3](https://medium.com/@pjbgf/title-testing-code-ocd-and-the-aaa-pattern-df453975ab80) or [4](https://thoughtbot.com/blog/four-phase-test) phases of your tests can help with comprehension.
* Use [`t.Helper()`](https://golang.org/pkg/testing/#T.Helper) in your test helper functions to keep stack traces clean.
* Use [`t.Parallel()`](https://rakyll.org/parallelize-test-tables/) to speed up tests.
* Trend away from using [`testify/suite`](https://pkg.go.dev/github.com/stretchr/testify/suite). (It used to address some shortcomings in the standard library `testing` tools that have since been addressed.)
* Lightweight assertion packages can help with expressiveness. Consider using [`testify/assert`](https://pkg.go.dev/github.com/stretchr/testify@v1.7.0/assert) or [`is`](https://pkg.go.dev/github.com/matryer/is)

### Coverage

* Always test exported functions.
  Exported functions should be treated as an API layer for other packages.
  Cover the expected behavior and error scenarios as a user of that API.
  * Consider using the [`_test` package suffix](https://golang.org/cmd/go/#hdr-Test_packages)
    to simulate calling the package under test from an external package
* Try not to test unexported functions.
  Unexported functions are implementation details of exported ones
  and should not change the intended usage.
  If you find that an unexported function is complex and needs testing,
  it might mean it needs to be refactored as it's exported function elsewhere.

## Packages

### Dependency Management

* [Go Modules](https://blog.golang.org/v2-go-modules) has become the standard way to manage your dependencies
* Before Go Modules were solidified, `dep` was frequently used for dependency management.
  * This might be helpful if you are dealing with an older project: [Daily Dep documentation](https://golang.github.io/dep/docs/daily-dep.html) (common tasks you’ll encounter with the dependency manager)

### Prefer Standard Libraries

In general,
when selecting new packages,
highly consider standard libraries over third party dependencies.
One of the strengths of Go
is its core packages,
such as
[http](https://golang.org/pkg/net/http/),
[json](https://golang.org/pkg/encoding/json/),
and [sql](https://golang.org/pkg/database/sql/).
These libraries also use vocabulary and patterns
easily accessible via popular public Go resources,
which are often translatable to modern programming approaches
in their respective areas.
This creates an easier bridge
for Engineers new to Go
or domain areas (such as relational databases)
to adjust and onboard.

Since the third party ecosystem is still new,
packages may have little community support,
follow opinionated patterns inconsistent with Go idioms,
or lack long term support.
When choosing a third party package,
carefully weigh the cost adoption and support contributions.
Document the decision
and any shortcomings of a comparable standard library
along with a rollback plan.

### Third Party Packages

Some examples of third party packages we've found to be helpful and stable are:

* [sqlx](https://github.com/jmoiron/sqlx)
  for SQL querying and struct marshalling.
* [cobra/pflag/viper](https://github.com/spf13/cobra)
  for writing command line utilities.

If you're exploring a new package,
[Awesome Go](https://awesome-go.com/)
is a good place to start.

## Time

### Clock Dependency

`time.Now()` can cause a lot of side effects in a codebase.
One example is
that you can't test the "current" time
that happened in a function you called in the past

For example, let's say we have the following:

```go
package mypackage

import "time"

func MyTimeFunc() time.Time {
    return time.Now()
}

func TestMyTimeFunc(t *testing.T) {
    if MyTimeFunc() != time.Now() {
        // This will error!
        // The time in the function and the test happen at different times
        t.Errorf("time was not now")
  }
}
```

How do we test the contents of the return here?
If we want to assert the time
we need a way to know what `time.Now()` was when the function was called.

Instead of directly using the `time` package,
we can pass a clock as a dependency and call `.Now()` on that.
Then in our tests, we can assert against that clock!
The clock can be anything as long as it adheres to the `clock.Clock` interface
as defined in the
[facebookgo clock package](https://godoc.org/github.com/facebookgo/clock#Clock).
We could, for example,
make the clock always return the year 0,
or the 2019 New Year,
or maybe your birthday!
In this clock package,
there are two clocks.

* The real clock where `clock.Now()` will call `time.Now()`.
* A mock clock where `clock.Now()` always returns epoch time.
  We'll show later how to change that!

Let's look at the example above with the `clock` package.

```go
package mypackage

import "fmt"
import "time"

import "github.com/facebookgo/clock"

func MyTimeFunc(clock clock.Clock) time.Time {
    return clock.Now()
}

// Then our caller
func main() {
    // clock.New() creates a clock that uses the time package
    // it will output current time when .Now() is called
    fmt.Print(MyTimeFunc(clock.New()))
}
```

Then in our tests we can use a mock clock that freezes `.Now()` at epoch time:

```go
func TestMyTimeFunc(t *testing.T) {
    testClock := clock.NewMock()
    if MyTimeFunc(testClock) != testClock.Now() {
        // both should equal epoch time, we won't hit this error
        t.Errorf("time was not now")
  }
}
```

Cool, but what if I want to use a different date?
Say my test relies on our `TestYear` constant.
The [clock.Mock clock](https://godoc.org/github.com/facebookgo/clock#Mock)
allows us to add durations to the clock and set the current time.
Note that the `clock.Clock` interface does not allow this,
it needs to happen before passing the mock clock through the interface parameter.

#### Setting the Mock Clock

Here's an example using the test above and setting the time to September 30 of TestYear:

```go
func TestMyTimeFunc(t *testing.T) {
    testClock := clock.NewMock()
    dateToTest := time.Date(TestYear, time.September, 30, 0, 0, 0, 0, time.UTC)
    timeDiff := dateToTest.Sub(c.Now())
    testClock.Add(timeDiff)
    if MyTimeFunc(testClock) != testClock.Now() {
        // both will now be September 30 of TestYear
        // we'll pass the test again
        t.Errorf("time was not now")
  }
}
```
