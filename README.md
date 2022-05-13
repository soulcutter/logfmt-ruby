# Logfmt

Write and parse structured log lines in the [logfmt style][logfmt-blog].

## Installation

Add this line to your Gemfile:

```ruby
gem "logfmt"
```

And then install:

```bash
$ bundle
```

Or install it yourself:

```bash
$ gem install logfmt
```

### Versioning

This project adheres to [Semantic Versioning][semver].

## Usage

`Logfmt` is composed to two sides of these coin: writing structured log lines in the `logfmt` style, and parsing `logfmt`-style log lines.

While writing and parsing `logfmt` are related, we've found that it's common to only need to do one or there other in a single application.
To support that usage, `Logfmt` leverages Ruby's `autoload` to lazily load the `Logfmt::Parser` or `Logfmt::Logger` (and associated code) into memory.
In the general case that looks something like:

```ruby
require "logfmt"

Logfmt # This constant was already loaded, but neither Logfmt::Parser
       # nor Logfmt::Logger constants are loaded. Yet.

Logfmt.parse("…")
  # OR
Logfmt::Parser.parse("…")

# Either of the above will load the Logfmt::Parser constant.
# Similarly you can autoload the Logfmt::Logger via

Logfmt::Logger.new
```

If you want to eagerly load the logger or parser, you can do that by requiring them directly

### Parsing log lines

```ruby
require "logfmt/parser"

Logfmt::Parser.parse('foo=bar a=14 baz="hello kitty" cool%story=bro f %^asdf')
  #=> {"foo"=>"bar", "a"=>14, "baz"=>"hello kitty", "cool%story"=>"bro", "f"=>true, "%^asdf"=>true}
```

### Writing log lines

The `Logfmt::Logger` is built on the stdlib `::Logger` and adheres to its API.
The primary difference is that `Logfmt::Logger` defaults to a `logfmt`-style formatter.
Specifically, a `Logfmt::Logger::KeyValueFormatter`, which results in log lines something like this:

```ruby
require "logfmt/logger"

logger = Logfmt::Logger.new($stdout)

logger.info(foo: "bar", a: 14, "baz" => "hello kitty", "cool%story" => "bro", f: true, "%^asdf" => true)
  #=> time=2022-04-20T23:30:54.647403Z severity=INFO  foo=bar a=14 baz="hello kitty" cool%story=bro f %^asdf

logger.debug("MADE IT HERE!")
  #=> time=2022-04-20T23:33:44.912595Z severity=DEBUG msg="MADE IT HERE!"
```

#### Expected key/value transformations

When writing a log line with the `Logfmt::Logger::KeyValueFormatter` the keys and/or values will be transformed thusly:

* "Bare messages" (those with no key given when invoking the logger) will be wrapped in the `msg` key.

    ```ruby
    logger.info("here")
      #=> time=2022-04-20T23:33:49.912997Z severity=INFO  msg=here
    ```

* Values, including bare messages, containing white space or control characters (spaces, tabs, newlines, emoji, etc…) will be wrapped in double quotes (`""`) and fully escaped.

    ```ruby
    logger.info("👻 Boo!")
      #=> time=2022-04-20T23:33:35.912595Z severity=INFO  msg="\u{1F47B} Boo!"

    logger.info(number: 42, with_quotes: %{These "are" 'quotes', OK?})
      #=> time=2022-04-20T23:33:36.412183Z severity=INFO  number=42 with_quotes="These \"are\" 'quotes', OK?"
    ```

* Floating point values are truncated to three digits.

* Time values are formatted as ISO8601 strings, with six digits sub-second precision.

* A value that is an Array is wrapped in square brackets, and then the above rules applied to each Array value.
  This works well for arrays of simple values - like numbers, symbols, or simple strings.
  But complex data structures will result in human mind-breaking escape sequences.
  So don't do that.
  Keep values simple.

  ```ruby
  logger.info(an_array: [1, "two", :three])
    #=> time=2022-04-20T23:33:36.412183Z severity=INFO  an_array="[1, two, three]"
  ```

**NOTE**: it is **not** expected that log lines generated by `Logfmt` can be round-tripped by parsing the log line with `Logfmt`.
Specifically, this applies to Unicode and some control characters, as well as bare messages which will be wrapped in the `msg` key when writing.
Additionally, symbol keys will be parsed back into string keys.

[logfmt-blog]: https://brandur.org/logfmt "Structured log lines with key/value pairs"
[semver]: https://semver.org/spec/v2.0.0.html "Semantic Versioning 2.0.0"
