# argparse11
C++11 (header only) library for Python argparse style command line parsing 

Python's argparse [https://docs.python.org/3/library/argparse.html] is,
after a steep learning curve, quite a powerful command line parser. C++11
has a lot of features that should allow for something somewhat close to
Python's argparse style of command line parsing, including automatically
generated help text.

[![Build Status](https://travis-ci.org/haavee/argparse11.svg?branch=master)](https://travis-ci.org/haavee/argparse11)

## 1 minute intro
Using the argparse11 library, Python argparse's first example
[https://docs.python.org/3/library/argparse.html#example] can be
implemented like in [example0.cc](example0.cc).

Stripped down to the pure argparse11 basics, [example0.cc](example0.cc) does
the following, even without knowing too much, it should be fairly readable
(that is, by C++ standards ...):

```c++
#include <argparse.h>

using AP = argparse;

int main(int argc, char*const*const argv) {
    auto           cmd = AP::ArgumentParser( AP::docstring("Process some integers.") );
    std::list<int> ints;    // the integers collected from the command line

    cmd.add( AP::long_name("help"), AP::short_name('h'), AP::print_help() );

    cmd.add( AP::docstring("Sum the integers (default: find the max)"), AP::long_name("sum"),
             AP::store_const(accumulation_fn<int>(std::plus<int>())),
             AP::set_default(accumulation_fn<int>([](int a, int b) { return std::max(a, b); })) );

    cmd.add( AP::collect_into(ints), AP::at_least(1), AP::docstring("an integer for the accumulator") );

    cmd.parse(argc, argv);

    auto accumulator = cmd.get<accumulation_fn<int>>("sum");

    std::accumulate(++std::begin(ints), std::end(ints), *std::begin(ints), accumulator);
}
```

# The automatically generated documentation

From the command line options added as shown above, in the [example0.cc](example0.cc) file, the library generates the following output if the program is called with `--help`:

```
 [mymac]> Darwin-x86_64-native-opt/example0 --help
 Usage:  example0 [-h --help] [--sum] <int>...
 
 Process some integers.
 
 positional arguments:
 <int>...
     an integer for the accumulator
     Requirements:
       postcondition:argument count greater than or equal 1
 
 [-h --help]
 
 [--sum]
     Sum the integers (default: find the max)
     Default:  std::__1::function<int (int, int)>
```

The options are formatted according to these rules:

- `[` ... `]` if the option is optional, i.e. 0 or more times
- `...` (ellipsis) are added if the option may or must occur 1 or more times
  (the distinction between *may* or *must* can be inferred from the presence
(or lack of) `[` ... `]` brackets surrounding the option

If the option takes an argument then the (inferred, internal) type of the
argument is added:

- `-f <int>`, `--input-file <string>`, `--threshold <float>` etc

The library contains a facility macro `HUMANREADABLE(Type, text)` in
order to demangle user-defined-date-type `Type` into, well, human readable
text `text` in stead of its typically unreadable, though demangled, C++ type name.


# Supported command line syntax

The library supports the following flavours of command line options:

- `-x` `--long-name` for flags or options that do not take an argument
- `-xyz` is interpreted as `-x -y -z` and is only valid if none of `x, y` and `z` take an argument (otherwise a parse error occurs)
- `-f ARG` `--input-file ARG` `--input-file=ARG` for options taking an argument
- `ARG` for the unnamed command line option (if present, that is), also dubbed "the program's
  arguments"

Note that numerical options are not supported. The main reason for that was to be able to support negative numbers as arguments. But maybe this could be leveraged by a somewhat hairier parser.

At the moment there is no `--` sentinel to enable verbatim, unparsed &
unverified, collection of the remaining command line arguments. If desired
it shouldn't be too big a deal to add.

# Basic usage

Command line options are added through the `.add(...)` method, passing in
any number of properties (see below) you'd want this command line option to
have. Some compile-time checks are enforced (e.g. that the type of a given
default matches the type of the argument stored, if a default is set that it
doesn't violate any of the constraints placed on the value, things like that).

The `.parse(int, char const*const*const)` can be used to parse the command
line of the program.

There is no need to (try to) catch exceptions; if /anything/ goes wrong
during parsing the program is terminated - contraints have been violated and
thus there's no point going on.

After a succesful parse, the converted values can be found in the variables
that were `stored_into` or `collected_into` or can be extracted using:

```c++
    ...
    cmd.add(short_name('m'), store_value<int>());
    ...
    auto  count = cmd.get<int>("m");
```

## Mutually-exclusive command line options

The library supports mutually-exclusive (exclusive or, XOR) command line options. They need to
be added slightly differently, specifically through the  `.addXOR( option(...), option(...), ...)` member function.

The `option(...)` are the members of this mutually-exclusive group of
options. The arguments passed to the `option(...)` function are  are what you'd use to add a non-mutually-exlusive option through `.add(...)`:

```c++
    cmd.addXOR( option(long_name("print"),  store_true(), at_most(3)),
                option(long_name("no-print"), store_false(), at_most(2)) /*, ...*/ );
```

Obviously, the `argparse::` namespace qualifier has been omitted for readability.


# Actions, constraints, requirements, conversion

The library has many actions and other, potentially useful, attributes that
can be added to a command line option, think of constraints or setting
limit(s) on how often said option may or must be present.


## Actions

These define what happens if the option is found on the command line. Each
option *must* have one.

- `store_true()/store_false()/store_const(<value>)` don't take an argument, do the action if the option is present (a `store_const_into(<variable&>)` is planned, if only for the symmetry)
- `store_into(<variable&>)/store_value<T>()` convert argument to the type of `<variable>` or `T` and then store
- `collect<T>(), collect_into(<variable&>)` convert argument to type `T` or the `::value_type` of the type of `<variable>` (`<variable&>` must refer to an instance of a container); collect all converted values in the container of your choice (`std::list` by default or whatever `<variable&>` referred to)
- `count(), count_into(<variable&>)` (self explanatory?) count how often the option is present
- `print_help()`, `print_usage()`, `print_version()` - there are *not* added
  by default: it is your call to add them and under which flag/option.


## Constraints

Your code may benefit if the command line parser helps in only accepting
values that are within certain limits, so that you don't have to test them
manually (the library is, after all, already converting the value so why not
take a look at it whilst it's there).

`argparse11` allows for placing constraints on the (converted) value(s) from
the command line option arguments, which will be enforced if the option was
parsed from the command line.

- `minimum_value(v)/maximum_value(v)`
- `is_member_of({<set of values>})`
- `minimum_size(s)/maximum_size(s)/exact_size(s)`
- `constrain(Callable&&, "description")` - program your own constraint
      (e.g. put in a lambda to constrain on just about any condition)
- `match("<regex expression>")` - the argument's string version should
      match the regex

## Requirements

Quite often it is nice to limit (upper bound or lower bound) the amount of
times a command line option may or must be present.

`argparse11` has the following requirement primitives; combinations are
allowed, although it is not checked that the combination makes sense.

- `at_least(n), at_most(n), exactly(n)`

## Conversion

The library has a built-in converter which knows about the standard POD
types enriched with "std::string". So no work from you is required to read
any of these.

For specific conversion or conversion to user-defined-data-type, one can
add:

- `convert(Callable&&)`

to an option's definition: `Callable&&` should then implement one's own string-to-user-data-type conversion routine.

## Documentation

Options may have any number of `docstring(...)` entries which will be the
option's description in the help text.

The `ArgumentParser(...)` constructor also supports any number of
`docstring(...)` entries which will make up the program's description.

The `version(...)` primitive, when added to the `ArgumentParser(...)`
constructor will, as one would expect, store the program's version. 

# Exact details

More detailed information about all of these can be found in the header of
the [argparse_actions.h](argparse_actions.h) header file, specifically what
type of argument(s) they expect.

Typically, they are very flexible - if possible focussing on `duck typing`
(e.g. the `version(...)` accepts anything that supports
`operator<<(std::ostream&, ...)`)
