# clitest-rs

clitest-rs aims to be the following:

* An expressive / literate way to define tests of CLI applications.
* A standalone executable that can evaluate CLI tests. Just point it at an
  executable you want to test and some test files and it does the rest.
* A testing library that can be embedded into Rust code bases. Rust projects
  can execute integration tests via `cargo test`. Behavior of the test runner
  can be customized using the crate's Rust API to tailor to the specific use
  cases of individual projects.

**This project is still in its design phase. We're working on the test
file format definition and execution semantics before writing a lot of
code because as we learned from deficiencies in prior art mistakes here
are difficult to impossible to correct. So we want to *get it right* from
the beginning.**

## Example Test File

Here's a high-level example showing the proposed syntax and some high-level features.

~~~
This is a test file. This line is ignored by the test runner.

The HTML processing instruction says this file and its tests should only
run on Windows.
<?clitest require=windows ?>

The code fence below defines commands to execute.

```
$ myapp help
MyApp Version 1.0

Usage: myapp <action>
```

Demonstrate that we can redirect output to a file and use common
coreutils programs `cut` and `hexdump` to effectively create a snapshot
test of content.

```
$ myapp hello > hello
$ cut -z -b 1-32 hello | hexdump -C
00000000  23 20 63 6c 69 74 65 73  74 2d 72 73 0a 0a 63 6c  |# clitest-rs..cl|
00000010  69 74 65 73 74 2d 72 73  20 61 69 6d 73 20 74 6f  |itest-rs aims to|
00000020  00                                                |.|
00000021

```

```
$ myapp process
< input to stderr of spawned process
>1 expected stdout output
>2 expected stderr output
```

```ignore
This code fence is ignored by the test runner. It does not define any commands.
```

~~~

## `.clitest` Syntax

A `.clitest` is an expressive / literate file format to define tests that
run executables and verify their behavior.

The syntax of `.clitest` file leverages Markdown syntax conventions and
many `.clitest` files can also be parsed / rendered as Markdown.

The most important part of the file format are *test cases*. These describe
commands to execute and their expected behavior.

Test cases live between [code fences](https://spec.commonmark.org/0.30/#code-fence),
which are pairs of 3 backticks or tildes. e.g.

~~~
```
Code fence
```
~~~

```
~~~
Another code fence
~~~
```

Unlike the CommonMark specification, we do not (yet) support optionally
prefixing the code fence with up to 3 spaces.

All lines outside of *code fences* are ignored with the exception of lines
starting with a [processing instruction](https://spec.commonmark.org/0.30/#processing-instruction)
with the `clitest` tag. e.g.

~~~
```
<?clitest this metadata is recognized ?>
```

```
<?xml this is ignored since the tag is xml not clitest ?>
```

```
  <?clitest ignored since it isn't at the start of a line ?>
```
~~~

### Test Case Code Fences

Each code fence is parsed into a single *test case*. Each case can
define 0 to N command invocations to run. The execution environment
for each test case is shared among its invocations, allowing invoked
commands to interact with each other.

The opening line of the *code fence* can contain an optional *info block*
after the backticks or tildes, per the CommonMark specification. This
info block content is made available to the test runner so it can influence
behavior.

Within each code fence we define a custom syntax for declaring commands
to invoke and their expected output.

The initial content of each line can denote special meaning:

* `$` denotes a command string to run.
* `<` define stdin to feed to the spawned command.
* `[N]` defines the expected exit code of the command. MUST be the final
  line for a command definition.
* `>1` and `>2` define expected output for stdout and stderr, respectively.
  If not used, stdout and stderr are merged. This syntax allows verification
  that output is written to a specific stdio file descriptor.
* All other lines are treated as expected output from the command.

### Test Case Info Block Directives

Optional *code fence* *info blocks* may pass directives to the test runner
to influence execution of that code fence. Directives are intended to be
single words. But the library API allows custom parsing to be performed.

The following single word directives are recognized by default:

* `ignore` says to ignore this *code fence*. It will not be parsed as a
  test case.

### Processing Instructions

HTML processing instructions (`<?clitest ... ?>`) are used to convey
instructions to the test runner outside the context of a single test
case.

The default test runner exposes various functionality via processing
instructions. But behavior can be customized. Library customers may
implement their own processing instructions.

## Test Case Execution Semantics

By default each *test case* is executed in a temporary directory in order
to try to ensure a reproducible test environment. The temporary directory
is populated with the executable under test along with copies of additional
files that have been registered.

Environment variables exposed to spawned processes are normalized by default
to try to ensure a reproducible test environment. The following environment
variables are set by default:

* `PWD` Current directory the test case is executing in.
* `USER` The value `clitest`, which may not exist on the current system.
* ...

## Principles

## Simplicity and intuitiveness

The test format should be simple, easy to read, and easy to understand.

We base the test format on Markdown, which is well understood among the
target developer community. We leverage features of markdown - code fences
and HTML processing instructions - which are generally known or familiar
to many developers.

## Embrace Coreutils

We embed a copy of the [uutils](https://github.com/uutils/coreutils) Rust
implementation of GNU coreutils. This allows users to harness as little or
much power from these common utilities (like `grep`, `sed`, and `awk`)
for additional text processing / analysis as they want.

By leaning on these common utilities we defer the onus of having to reinvent
this complexity in our test format and test harness. Our test harness can focus
on executing processes and not complex text stream processing.

By using the Rust coreutils implementation, test execution uses a
deterministic version of these tools, not whatever version or variant you
have on the system. On Windows you don't need to install Cygwin, msys, or
run in WSL to get access to these utilities.

## Commitment to Backwards Compatibility

We strive to not require end-users to rewrite test files when upgrading. If you
need to spend hours updating tests when upgrading the version of this project
you are using, we've failed.

The test file format should evolve to be backwards and forwards compatible.

The test execution semantics should also be highly backwards compatible.

When you upgrade the version of this project you are using hopefully the
worst thing that happens is you need to run a command to migrate to slightly
different file syntax or accept machine proposed changes to expectations
defined in the test files.

## Embrace the Concept of the Shell Without an Actual Shell

While it could be useful to execute tests in a shell (like bash, zsh,
or even nushell), we reject the existence of an explicit shell during test
execution (unless end-users opt into it).

Practical experience with Mercurial and cram demonstrated there are too
many sharp edges with actual shells. At the least you need a shell binary
on every machine under test. And the shell version/features needs to behave
the same on all systems. This can be difficult, especially on Windows.

A fully embeddable shell (like Nushell) could be a viable alternative. But
now we're externalizing knowledge of a specific shell onto our end-users.

End-users are opinionated about shells. Bash. Zsh. Powershell. We don't want
to pick sides in this debate.

But shells are useful. Variables. Pipes. Redirection. Control flow and loops.
Builtin commands (like `cd` and `echo`). Background process execution.

We want to expose some of the features of shells without using an actual
shell. Make the user think they have a shell for common functionality to
test command invocations but don't let them juggle chainsaws.

## TODO

Here are some ideas for the test format and execution semantics that we'd
like to flush out more. Many are potential future features we could add to
the test runner. We should focus on defining an extensible test format
so future features don't require new syntax.

* Define syntax for regex matching of single lines.
* Define syntax for eliding multiple lines of expected output.
* Define escaping / alternative directive encoding mechanism so expected
  process output colliding with our special syntax can be worked around and
  allow expression of all outputs in the test format.
* Define syntax for matching common / repeated patterns. Logically allow
  expansion of tokens in expected output. We'll want to define some tokens
  by default, such as the current executable name and working directory.
* Define mechanism to verifying binary output. Maybe a directive / info block
  to escape output as hex or base64? Consider humans wanting to add arbitrary
  line breaks so 1 byte insertions/removals don't invalidate all following
  lines.
* Define additional minimal environment variables. (e.g. `TZ`, `PATH`.) Stuff
  that is in most environments.
* Syntax for setting additional environment variables.
* Consider `%` and possibly other line-leading tokens inside code fences to
  influence operations. e.g. `% cd foo` to change the CWD for future command
  invocations.
* Consider shell-like file redirection syntax for sending output of a command to
  a file, including to `/dev/null` (or its equivalent).
* Consider some magical commands or syntax (like `$ cat <path>`) to allow
  verification of produced file content.
* Mechanism for conditional execution. e.g. only execute on Windows or POSIX.
  Only execute if allowed to access a network. Maybe allow tags to be specified
  to the test runner so tests can be filtered. This can allow expression and
  skipping of *slow* tests. See also `requires` and keywords mechanisms in
  Mercurial's test harness.
* Command / test timeout support. How do you express that?
* Support for delivering a signal to a process. e.g. simulate a ^C to test
  error handling.
* Consider semantics for automatic drift overwrites when the test runner
  executes. We want to expose a turnkey mechanism where new/different/drifted
  test output can be recorded in the original test file without humans having
  to edit the file. The more custom syntax we add to the file format the harder
  this becomes. e.g. if you support regular expression matching of lines, how
  do you preserve those expressions or tokens when overwriting expected output?
  This is a known wort in Mercurial's and cram's more expressive output
  processing world.
* Define semantics around stdio file descriptor buffers. Could be nice to
  test differences when stdout is buffered/unbuffered.
* Consider ability to implement an interactive execution mode where you can
  step through executions. This allows developers to attach debuggers or
  inspect the filesystem sandbox when they want. It could also allow selectively
  accepting/rejecting/modifying output drift.
* Consider mechanism to define stdin/stdout in separate files. Useful for
  large content, like verifying behavior over input data sets. Can also be
  useful for sharing inputs across files and test cases.
* Consider mechanism for reusing a series of common commands. e.g. common
  test fixture setup code. How can we enable developers to minimize the
  overhead of authoring tests by minimizing DRY violations.
  * Idea: directories containing seeds for filesystem sandboxes. Test files or
    cases can specify directories to use to populate content of the filesystem
    sandbox.
  * Idea: Directive so test files or cases can specify another `.clitest` file
    whose single test case can be prepended to cases.
  * Idea: code fence info block to name a test case so it can be included in
    another one.
  * Idea: code fence info block to denote the prepending of content to all
    test cases.
* Consider allowing test cases to execute within a shell. Maybe a shell
  implemented in pure Rust for portability.

## Project History

The goal of this project is to facilitate [literate testing](https://arrenbrecht.ch/testing/)
of command line applications. Command line applications are complex
and are often designed with user interaction in mind. Their behavior is
important to test and understand. Changes/differences in behavior are
important to detect and capture to ensure consistent user experiences.

We believe that literate testing of CLI applications results in higher
quality CLIs by increasing the surface area under test and forcing developers
to confront their CLI UX. This belief is grounded in the experience of
project contributors, namely around the use of the practice in the Mercurial
version control tool. We want to lower the barrier to literate CLI
testing in the Rust ecosystem and beyond to encourage its broader use.

At the time of this project's inception (mid 2023), literate CLI testing in
the Rust ecosystem was not very mature. Popular CLI testing crates relied on
spawning processes and inspecting their behavior from Rust code. See
[assert_cmd](https://crates.io/crates/assert_cmd),
[insta-cmd](https://crates.io/crates/insta-cmd), and
[rexpect](https://crates.io/crates/rexpect). The [trycmd](https://github.com/assert-rs/trycmd)
seemed to be the lone attempt at higher-order literate CLI testing in Rust.
But it had various limitations (see below).

This project is inspired by [Mercurial's](https://www.mercurial-scm.org/) custom
test harness. Mercurial inspired [cram](https://bitheap.org/cram/). And cram
inspired [trycmd](https://github.com/assert-rs/trycmd).

This project came about due to various limitations with all of the above tools.
We wanted to invent a modern, powerful literate command testing tool that
exceeded capabilities of tools before while hopefully avoiding many pitfalls
of earlier tools.

Here are some limitations with Mercurial and cram:

* Format was completely custom. Developers had to learn a new DSL for writing
  tests. It was relatively intuitive. But IDEs lacked parsing / highlighting.
* Indent of 2 spaces was kinda annoying to manually paste output differences.
  You were constantly doing multi-line indents in your editor.
* Line output processing features (like `(re)`, `(glob)`, `(esc)` `(no-eol)`)
  often got swallowed when outputs changed. The interaction of these directives
  often had unexpected consequences. You often needed to define these on
  several lines, which was annoying.
* Test files were effectively normalized to an auto-generated shell script which
  was executed on the current system. Commands were literal shell expressions.
  You were constantly fighting portability issues. e.g. on Windows you needed
  to install cygwin or msys to provide a shell and other common commands. You
  could have all the common problems with shell programming, including esoteric
  escaping and variable referencing issues. State leaking between commands.
  Poor debugging story. The full power of the shell could be useful but it was
  difficult to wield.
* Implemented in Python. Test harness was relatively slow. Required a Python
  interpreter to execute. Standalone binaries much easier for end-users.

And limitations with trycmd:

* Public library API does not facilitate advanced use. You can't construct test
  cases via the Rust API. See [issue 217](https://github.com/assert-rs/trycmd/issues/217).
* Extensions to file format not possible. Related to above. But the low-level
  test format parser doesn't seem designed with an extensibility mechanism in
  place. See [issue 218](https://github.com/assert-rs/trycmd/issues/218).
* Currently lacking a lot of features that Mercurial and cram have. See their
  [enhancements list](https://github.com/assert-rs/trycmd/issues?q=label%3Aenhancement+)
  for them all. Examples include: lack of piping, lack of output redirection,
  lack of dynamic capture/matching, turnkey support for coreutils commands,
  filesystem sandboxing for `.trycmd` tests, handling escaping when there is
  a conflict between command output and trycmd test file syntax.
* No standalone test runner/binary. Scope seems limited to embedding a test runner
  in Rust crates.

Trycmd feels like the most modern command testing tool in the Rust space right
now since it supports a literate test format and is relatively easy to use (e.g.
no Python dependency). But it is still a far cry from the features of Mercurial
and cram. The crate maintainer doesn't seem interested in opening up the crate's
public API to support additional customization. This means we'll need to build
something ourselves. We should hopefully be able to leverage some of trycmd's
prior art, such as the [snapbox](https://crates.io/crates/snapbox) crate.
