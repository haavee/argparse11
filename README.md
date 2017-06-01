# argparse11
C++11 (header only) library for Python argparse style command line parsing 


Python's argparse [https://docs.python.org/3/library/argparse.html] is,
after a steep learning curve, quite a powerful command line parser. C++11
has a lot of features that should allow for something somewhat close to
Python's argparse style of command line parsing.

Using the argparse11 library, Python argparse's first example
[https://docs.python.org/3/library/argparse.html#example] can be
implemented like this:


```c++
#include <argparse.h>
#include <list>
#include <iterator>
#include <algorithm>
#include <numeric>

// shorthands
namespace AP = argparse;

template <typename T>
using accumulation_fn = std::function<T(T, T)>;

int main(int argc, char*const*const argv) {
    auto           cmd = AP::ArgumentParser( AP::docstring("Process some integers.") );
    std::list<int> ints;    // the integers collected from the command line

    // We have '-h|--help', '--sum' and the arguments, integers
    // The argparse11 library does not automatically add "--help" - it's at
    // your discretion wether to add it and under which flag(s).
    cmd.add( AP::long_name("help"), AP::short_name('h'), AP::print_help() );

    // If '--sum' is provided, use that, otherwise find the max.
    cmd.add( AP::docstring("Sum the integers (default: find the max)"), AP::long_name("sum"),
             AP::store_const(accumulation_fn<int>(std::plus<int>())),
             AP::set_default(accumulation_fn<int>([](int a, int b) { return std::max(a, b); })) );

    // Let the command line parser collect the converted command line
    // options into our STL container. By not giving this 'option' a name, the
    // system takes that to mean "oh, these must be the command line arguments then"
    // Note: we require at least one integer
    cmd.add( AP::collect_into(ints), AP::at_least(1), AP::docstring("an integer for the accumulator") );

    // Parse the command line
    cmd.parse(argc, argv);

    // Extract the selected accumulator from the cmd line object
    // (note that it stores the actual std::function instance).
    // C++11 is strongly typed so we'll have to tell the compiler (and
    // callee) which type we actually want.
    auto accumulator = cmd.get<accumulation_fn<int>>("sum");

    // And do the accumulation. Python "sum([...])" and "max([...])" are
    // obviously smarter then C++11's std algorithms but still quite cool it
    // can be done.
    // We can safely use "++begin()" because we required "at_least(1)" ;-)
    std::cout
        << std::accumulate(++std::begin(ints), std::end(ints), *std::begin(ints), accumulator)
        << std::endl;
    return 0;
}
```
