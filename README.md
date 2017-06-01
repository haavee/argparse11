# argparse11
C++11 (header only) library for Python argparse style command line parsing 


Python's argparse [https://docs.python.org/3/library/argparse.html] is,
after a steep learning curve, quite a powerful command line parser. C++11
has a lot of features that should allow for something somewhat close to
Python's argparse style of command line parsing, including automatically
generated help text.

## 1 minute intro
Using the argparse11 library, Python argparse's first example
[https://docs.python.org/3/library/argparse.html#example] can be
implemented like in [example0.cc](example0.cc).

Stripped down to the pure argparse11 basics, [example0.cc](example0.cc) does
the following. Even without knowing too much, it should be fairly readable
(that is, by C++ standards ...).

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

# Basic usage
Command line options are added through the `.add(...)` method, passing in
any number of properties (see below) you'd want this command line option to
have. Some compile-time checks are enforced (e.g. that the type of a given
default matches the type of the argument stored, things like that).

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


The library supports mutually-exclusive (exclusive or, XOR) command line options. They need to
be added slightly differently: `.addXOR( option(...), option(...), ...)`
where the parameters passed in each `option(...)` are what you'd use to add a non-mutually-exlusive option through `.add(...)`


# Actions, constraints, requirements, conversion

The library has many more actions and features:
- `store_true()/store_false()/store_const(<value>)/store_value<T>()`
- `store_into(<variable>)`
- `collect<T>(), collect_into(<variable>)`
- `count(), count_into(<variable>)`

The library allows for placing constraints on the (converted) value(s) from
the command line option arguments, which will be automatically enforced:
- `minimum_value(v)/maximum_value(v)`
- `is_member_of({<set of values>})`
- `minimum_size(s)/maximum_size(s)/exact_size(s)`
- `constrain(Callable&&, "description")` - program your own constraint
      (e.g. put in a lambda)
- `match("<regex expression>")` - the argument's string version should
      match the regex

It is possible to put requirements on how often the option/argument may be
found on the command line:
- `at_least(n), at_most(n), exactly(n)`

The library has a built-in converter which knows about the standard POD
types enriched with "std::string". By adding:
- `convert(Callable&&)`
to an option's definition, it is possible to use your own string-to-user-data-type conversion routine, implemented by `Callable`.

Options may have any number of `docstring(...)` entries which will be the
option's description in the help text.

More detailed information about all of these can be found in the header of
the [argparse_actions.h](argparse_actions.h) header file.
