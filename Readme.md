# Reference implementation for proposed SG14 `status_code` in C++ 11

(C) 2018 Niall Douglas http://www.nedproductions.biz/

Solves the problems with `<system_error>` et al as listed by https://wg21.link/P0824.

## Features:

- Portable to any C++ 11 compiler.
- Aims to cause zero code generated by the compiler most of the time.
- Never calls `malloc()`.
- Header-only library friendly.
- Type safe yet with type erasure in public interfaces.

## Problems with `<system_error>` listed by P0824 solved:

1. Does not include `<string>`, and thus the entire STL allocator and algorithm
machinery, thus preventing use in freestanding C++ as well as substantially
impacting compile times which can be a showstopper for very large C++ projects.
Only includes the following headers:
 - `<cerrno>` for the generic POSIX error codes (`errno`) which is required to define `errc`.
 - `<stdexcept>` for the generic POSIX exception types which is required to throw
 the appropriate C++ exception type for a POSIX error code.
 - `<type_traits>` as we need to do some very limited metaprogramming.

2. Unlike `std::error_code` which was designed before `constexpr`, this proposed
implementation has all-`constexpr` construction and destruction with as many operations
as possible being trivial or literal, with only those exact minimum operations which
require runtime code generation doing so (note: requires C++ 14 for a complete
implementation of this).

3. This in turn means that we solve a long standing problem with `std::error_category`
in that it is not possible to define a safe custom C++ 11 error category in a header
only library where semantic comparisons would randomly break depending on the direction of wind
blowing when the linker ran. This proposed design is 100% safe to use in header only libraries.

4. `std::error_code`'s boolean conversion operator i.e. `if(ec) ...` has become
unfortunately ambiguous in real world C++ out there. Its correct meaning is
"if `ec` has a non-zero value". Unfortunately, much code out in the wild uses
it as if "if `ec` is errored". This is incorrect, though safe most of the time
where `ec`'s category is well known i.e. non-zero values are always an error. For
unknown categories supplied by third party code however, it is dangerous and leads
to unpleasant, hard-to-debug, surprise.
 The `status_code` proposed here suffers from no such ambiguity. It can be one of
exactly three meanings: (i) success (ii) failure (iii) empty (uninitialised). There
is no boolean conversion operator, so users must write out exactly what they mean
e.g. `if(ec.success()) ...`, `if(ec.failure()) ...`, ``if(ec.empty()) ...`.

5. Relatedly, `status_code` can now represent successful (informational) codes as
well as failure codes. Unlike `std::error_code` where zero is given special meaning,
we impose no requirements at all on the choice of coding.

6. The relationship between `std::error_code` and `std::error_condition` is
confusing to many users reading code based on `<system_error>`, specifically when is
a comparison between codes *semantic* vs *literal*. `status_code` makes all
comparisons *semantic*, always. If you want a literal comparison, you can do one
by hand by comparing domains and values directly.

7. `std::error_code` enforced its value to always be an `int`. This is problematic
for coding systems which might use a `long` and implement coding namespaces within
the extended number of bits, or for end users wishing to combine a code with a `void *`
in order to transmit payload or additional context. As a result, `status_code` is
templated to its domain, and the domain sets its type. A type erased edition of
`status_code<D>` is available as `status_code<void>`, this is for obvious reasons
non-copyable, non-movable and non-destructible.
 A more useful type erased edition is `status_code<erased<unsigned long long>>` 
which is available if `D::value_type` is trivially copyable, and
`sizeof(unsigned long long) >= sizeof(D::value_type)`. This lets you use
`status_code<erased<unsigned long long>>` in all your public interfaces without
restrictions.


