---
layout: single
classes: wide
title:  "Dealing with long strings in YAML Hiera data"
date:   2019-02-23 20:45:00 -0500
categories: puppet ruby yaml
---

We have a step in our CI/CD Pipeline that lints YAML data using [adrienverge/yamllint](https://github.com/adrienverge/yamllint). One of the rules limits the number of characters on a single
line. Normally not a huge deal... except when dealing with long URIs or other long strings.

The problem is, I have to change URI Hiera data so infrequently that I always forget exactly what the syntax is. This post solves that problem.

And while we're on the topic:

* [YAML Cookbook for Ruby](https://yaml.org/YAML_for_ruby.html) is an awesome resource for YAML syntax in general.
* Puppet parses YAML using the following function in Ruby:
  ```ruby
  YAML.safe_load(File.read(path), [Symbol], [], true)
  ```
  Combine with Interactive Ruby Shell or `ruby -e` for a quick and easy way to test any YAML file you create.

## Plain Style

### "Flow" Styles

No character escaping, or characters matching `/[#:]/`.

```yaml
---
key: This is a very
     long string.
```

```ruby
{ "key" => "This is a very long string." }
```

### Single-quoted style

No special characters, no escaping. Literal `'` must be doubled-up (`''`).

```yaml
---
key: 'This isn''t
     a very short string.'
```

```ruby
{ "key" => "This isn't a very short string." }
```

### Double-quoted style

Characters matching `/[\\"]/` must be escaped by a `\` character. Common escape sequences may be used. Line concatenation with a trailing `\`. 

*VERY* useful for long URIs.

```yaml
---
key: "http://this.is.my\
     .very.long.string"
```

```ruby
{ "key" => "http://this.is.my.very.long.string" }
```

## Block Notation

### Literal Style

```yaml
---
key: |
  This is a very
  long string.
```

```ruby
{ "key" => "This is a very\nlong string.\n" }
```

### Folded Style

```yaml
---
key: >
  This is a very
  long string.
```

```ruby
{ "key" => "This is a very long string.\n" }
```

### Block chomping indicator

You may notice that both strings have newlines attached to the end. Want those gone? Use a different chomping indicator:

* `|`, `>`: "clip". Keeps the newline.
* `|+`, `>+`: "keep". Keeps the newline, and also keeps tailing blank lines.
* `|-`, `>-`: "strip". Removes the newline.

```yaml
---
key0: >
  Do. Or do not.
  There is no try.

key1: >+
  I find your lack of
  faith disturbing.
  
key2: >-
  The Force will be
  with you. Always.
```

```ruby
{
  "key0" => "Do. Or do not. There is no try.\n",
  "key1" => "I find your lack of faith disturbing.\n\n",
  "key2" => "The Force will be with you. Always."
}
```

Source: [YAML v1.2 Specification](https://yaml.org/spec/1.2/spec.html)
