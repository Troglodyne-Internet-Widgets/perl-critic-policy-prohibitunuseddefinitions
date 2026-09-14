# NAME

Perl::Critic::Policy::ProhibitUnusedDefinitions - A sub nobody calls, or a global nobody reads, is code nobody needs.

# VERSION

version 0.002

# Perl::Critic::Policy::ProhibitUnusedDefinitions

A sub that nothing calls is still read, still reviewed, still kept working
through every refactor -- and still tells the next reader that something,
somewhere, needs it.  The same goes for an `our` variable nothing reads and a
constant nothing names.

Whether anything uses a definition is not a question one file can answer, so
this policy reads the whole distribution around the file being critiqued.  The
first time it is asked about a file, it finds the distribution's root, parses
everything under `bin/`, `lib/`, `t/` and `xt/` once, and notes every call
and every reference.  Every later file in the same distribution is checked
against that note rather than parsed again.

- Subs

    must be called at least once from `bin/` or `lib/`.  A sub only the tests
    call is a sub only the tests need.

- `our` variables and `use constant` constants

    must be used at least once anywhere in `bin/`, `lib/`, `t/` or `xt/`.
    A global the test suite sets to change the code's behaviour is doing its job.

Only definitions in files under `bin/` or `lib/` are reported.  A helper
defined in a test is the test's business.

## PROHIBITED

```perl
package My::Thing;
sub helper { ... }          # nothing in bin/ or lib/ calls it
our $DEBUG = 0;             # nothing anywhere reads it
use constant LIMIT => 10;   # nothing anywhere names it
```

## ALLOWED

```perl
package My::Thing;
sub helper { ... }
sub run    { helper() }     # ...and bin/thing calls My::Thing->run

our @EXPORT_OK = qw{ tool };
sub tool { ... }            # exported, so its callers are elsewhere

sub DESTROY { ... }         # perl calls it
```

## WHAT COUNTS AS A USE

- A call, bare or qualified: `helper()`, `My::Thing::helper()`.
- A method call, `$obj->helper`, wherever it is written -- a
subscript such as `$h{ $obj->helper }` included.  The class behind
`$obj` cannot be known statically, so this counts as a use of every sub named
`helper`.
- A reference: `\&helper`, `&helper`, `*helper`.
- Any of these inside an interpolating string or heredoc, as
`"@{[ $obj->helper ]}"` or `"${\ helper() }"`.  What is inside is
read as the code it is.
- For variables, any mention other than the declaration itself --
`$x`, `$x[0]` and `$#x` for `@x`, `$x{k}` for `%x`, qualified or not,
and inside an interpolating string or regex.

An unqualified name is resolved to the package it appears in.  A `bar()` in
package `Baz` is a use of `Baz::bar`, not of an unrelated `Foo::bar`.

A string that happens to spell a sub's name is **not** a use, so
`__PACKAGE__->can('helper')` and `{ list => 'do_list' }` do not
count, and nor do `"@{[ 'helper' ]}"` or `"${helper}"`.  Those are what
`allow_subs` and `## no critic` are for.

## EXEMPT

Anything listed in a package's `@EXPORT`, `@EXPORT_OK` or `%EXPORT_TAGS`.
Exporting it is the point, and its callers are in some other distribution.

The names perl or a framework calls for you, and the globals perl reads itself:

```
BEGIN END INIT CHECK UNITCHECK AUTOLOAD DESTROY import unimport
CLONE CLONE_SKIP BUILD BUILDARGS DEMOLISH FOREIGNBUILDARGS
and the tie interface: TIEHASH FETCH STORE and the rest

$VERSION @ISA @EXPORT @EXPORT_OK %EXPORT_TAGS $AUTOLOAD
```

## CONFIGURATION

- `allow_subs`

    Space separated subs and constants that are never reported, as a bare name or
    qualified with its package.  Adds to the built-in list rather than replacing
    it:

    ```perl
    [ProhibitUnusedDefinitions]
    allow_subs = new My::Plugin::register
    ```

- `allow_globals`

    The same for `our` variables, with their sigil:

    ```perl
    [ProhibitUnusedDefinitions]
    allow_globals = $DEBUG %My::Thing::REGISTRY
    ```

## CAVEATS

The distribution's root is the nearest directory above the file with a
`dist.ini`, `Makefile.PL`, `Build.PL`, `META.json`, `META.yml`,
`cpanfile` or `.git` in it.  Failing that, it is the directory holding the
`lib/` or `bin/` the file is in.  Source with no file name -- a string handed
to `critique` -- belongs to no distribution and is never reported.

The index is built once per distribution per process.  A file edited after it
was built is not seen again until the next run.

Anything reached only at runtime -- a symbolic call, a string `eval`, an
`AUTOLOAD`, a dispatch table of names, `use overload` with method names --
reads as unused, because the source does not say otherwise.

Every heredoc is read as though it interpolates, `<<'END'` included, so a
variable or an `@{[ ... ]}` spelled out in a literal one still counts as a
use.

Lexical scope is not tracked.  In a package that declares `our $x`, every
`$x` is read as the global, including the reads of a `my $x` that shadows
it.  Neither is `our`'s habit of reaching across a later `package` statement
in the same block, nor `${name}` written with braces outside a string.

## TEMPLATES

Templates are not read, so a sub called only from a template reads as unused.

For most templates that costs nothing.  [Text::Xslate](https://metacpan.org/pod/Text%3A%3AXslate),
[Template Toolkit](https://metacpan.org/pod/Template), [Mojo::Template](https://metacpan.org/pod/Mojo%3A%3ATemplate), [HTML::Template](https://metacpan.org/pod/HTML%3A%3ATemplate) and the rest
hand a template a hash of variables, and a key in a hash is not a sub:
`[% domain %]` or `[% vhost.name %]` on plain data reaches no perl code.

The exception is an object in that hash. Don't forget to search your templates
any time you are tempted to remove code flagged in classes by this policy.

## METHODS

### supported\_parameters

`allow_subs` and `allow_globals`, the names that are never reported, added
to the built-in lists.

### initialize\_if\_enabled

Folds the built-in exemptions back into whatever was configured, so a user's
list adds to the defaults instead of replacing them.

### default\_severity

SEVERITY\_LOW

### default\_themes

maintenance

### applies\_to

PPI::Statement::Sub, PPI::Statement::Variable and PPI::Statement::Include --
the three ways to define a sub, a global or a constant.

### violates

Standard [Perl::Critic::Policy](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy) interface.  Returns one violation for each
sub, global or constant the statement defines that nothing in the distribution
uses, builds the distribution's index the first time it is needed.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/Troglodyne-Internet-Widgets/perl-critic-policy-prohibitunuseddefinitions/issues](https://github.com/Troglodyne-Internet-Widgets/perl-critic-policy-prohibitunuseddefinitions/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
