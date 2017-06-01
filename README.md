# argparse11
C++11 (header only) library for Python argparse style command line parsing 


Python's argparse [https://docs.python.org/3/library/argparse.html] is,
after a steep learning curve, quite a powerful command line parser. C++11
has a lot of features that should allow for something somewhat close to
Python's argparse style of command line parsing, including automatically
generated help text.

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

Options may have any number of docstring(...) entries which will be the
option's description in the help text.

More detailed information about all of these can be found in the header of
the [argparse_actions.h](argparse_actions.h) header file.
