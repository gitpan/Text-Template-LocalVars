# NAME

Text::Template::LocalVars - Text::Template with localized variables

# SYNOPSIS

    use Text::Template::LocalVars 'fill_in_string';

    # store values in 'MyPkg' package
    fill_in_string( $str1, hash => \%vars1, package => 'MyPkg' );

    # use values from MyPkg package, but don't store any new
    # ones there.
    fill_in_string( $str2, hash => \%vars2, package => 'MyPkg',
                    localize => 1 );

    # use the variable package in the last call to a template fill
    # routine in the call stack which led to this code being executed.
    fill_in_string( $str, trackvarpkg => 1 );

# DESCRIPTION

**Text::Template::LocalVars** is a subclass of [Text::Template](https://metacpan.org/pod/Text::Template), with
additional options to manage how and where template variables are stored.
These come in particularly handy when template fragments themselves
perform template fills, either inline or by calling other functions
which do so.

([Text::Template](https://metacpan.org/pod/Text::Template) stores template variables either in a package
specified by the caller or in the caller's package.  Regardless of
where it comes from, for conciseness let's call that package the
_variable package_.  Likewise, invoking a template fill function or
method, such as **fill\_in\_string**, **fill\_in\_file**, **fill\_this\_in**,
or **fill\_in** is called _filling_, or a _fill_. )

**Text::Template::LocalVars** provides the following features:

- localized variable packages

    The variable package may be _cloned_ instead of being used directly
    (see ["Localized Variable Packages"](#localized-variable-packages)), providing fills with a
    sandboxed environment.

- tracked parent variable packages

    If a fill routine is called without a package name, the package in
    which the fill routine is invoked is used as the variable
    package. This works well if the fill routine is invoked in a template
    fragment, but doesn't if the it is invoked in code compiled in another
    package (such as a support subroutine). **Text::Template::LocalVars**
    keeps track of the appropriate package to use, and can pass that
    package to the fill routine automatically (see ["Tracking Variable
    Packages"](#tracking-variable-packages)).

## Localized Variable Packages

Localized variable packages come in handy if your template fragments
perform template expansions of their own, and while they should have
access to the existing values in the package, you'd prefer they not
alter it.

Here's an example:

    use Text::Template::LocalVars 'fill_in_string';
    Text::Template::LocalVars->always_prepend
          ( q[use Text::Template::LocalVars 'fill_in_string';] );

    my $tpl = q[{
           fill_in_string(
               q[boo + foo = { $boo + $foo }],
               hash    => { boo => 2 },
           );}
    foo = { $foo; }
    boo = { $boo; }
    ];

    fill_in_string( $tpl, hash => { foo => 3 },
                          package => 'Foo',
                          output => \*STDOUT );

We're explicitly specifying a template variable package to
ensure that we don't contaminate our environment.  This outputs

    boo + foo = 5
    foo = 3
    boo = 2

Note that the inner fill sees `$foo` from the top level fill, and
adds `$boo` to its own environment _as well as that of the upper
level fill_.

What if you don't want to pollute the upper fill's environment?  You
might try giving the inner fill it's own package,

    my $tpl = q[{
           fill_in_string(
               q[boo + foo = { $boo + $foo }],
               hash    => { boo => 2 },
               package => 'Boo',
           );}
    foo = { $foo; }
    boo = { $boo; }
    ];

But then it has no access to the `Foo` package, so you get this:

    boo + foo = 2
    foo = 3
    boo =

**Text::Template::LocalVars** gives you the best of both worlds. If you
pass the `localize` option, the fill routine gets a copy of the
parent fill's environment (or of the specified package), so it
can't muck things up:

    my $tpl = q[{
           fill_in_string(
               q[boo + foo = { $boo + $foo }],
               hash    => { boo => 2 },
               localize => 1,
           );}
    foo = { $foo; }
    boo = { $boo; }
    ];

results in

    boo + foo = 5
    foo = 3
    boo =

Unlike Perl's `local` command, which retains the identity of a variable,
**Text::Template::LocalVars** creates a new package and copies the contents
of the original variable package into it (with some caveats, see below).

For example, without localization, the package retains its name.  The
following code

    fill_in_string( qq[Package is { __PACKAGE__ }\n],
                    package => 'Foo',
                    output => \*STDOUT,
                  );

outputs

    Package is Foo

while

    fill_in_string( qq[Package is { __PACKAGE__ }\n],
                    package => 'Foo',
                    localize => 1,
                    output => \*STDOUT,
                  );
  outputs

    Package is Text::Template::LocalVars::Package::Q0

Don't make assumptions about the name.

Certain constructs in packages are not easily cloned, so the
cloned package isn't identical to the original.  The `HASH`, and
`ARRAY` values in the package are cloned using [Storable::dclone](https://metacpan.org/pod/Storable::dclone); the
`SCALAR` values are copied if they are not references, and the
`CODE` values are copied.  All other entries are ignored.  This
is not a perfect sandbox.

## Tracking Variable Packages

If your processing becomes complicated enough that you begin nesting
template fills and abstracting some into subroutines, keeping track of
variable packages may get complicated.  For instance

    sub name {
        my ( $reverse ) = @_;

        my $tpl
          = $reverse
          ? q[ { $last },  { $first } ]
          : q[ { $first }, { $last }  ];

        fill_in_string( $tpl );
    }

    my $tpl = q[
          name = { name( $reverse ) }
      ];

    fill_in_string(
        $tpl,
        hash => {
            first   => 'A',
            last    => 'Uther',
            reverse => 1,
        },
        package => 'Foo'
    );

Here, we're implementing some complicated template logic in a
subroutine, generating a new string with a template fill, and then
returning that to an upper level template fragment for inclusion.
All of the data required are provided to the top level template fill
via the package `Foo`, but how does that percolate down to the `name()`
subroutine?  There are several ways to do this:

- Explicitly pass the _data_ to `name()`:

        my $tpl = q[
              name = { name( $reverse, $first, $last ) }
          ];

- Explicitly pass the _template package_ to `name()`:

        my $tpl = q[
              name = { name( $reverse, __PACKAGE__ ) }
          ];

- Turn on template package tracking in `name()`:

        fill_in_string( $tpl, trackvarpkg => 1 );

    `Text::Template::LocalVars` keeps track of which variable packages are
    used in _nested calls_ to fill routines; setting `trackvarpkg` tells
    `fill_in_string` to use the package used by the last fill routine in
    the call stack which led to this one.  In this case, it'll be the one
    setting `package` to `Foo`.  If there is none, it falls back to
    the standard means of determining which package to use.

# METHODS

## new

See ["new" in Text::Template](https://metacpan.org/pod/Text::Template#new).

## compile

See ["compile" in Text::Template](https://metacpan.org/pod/Text::Template#compile).

## fill\_in

The API is the same as in ["fill\_in" in Text::Template](https://metacpan.org/pod/Text::Template#fill_in), with the
addition of the following options:

- localize

    If true, a clone of the template variable package is used.

- trackvarpkg

    If true, and no template variable package is specified, use the one used
    in the last **Text::Template::LocalVars** fill routine which led to invoking
    this one.

# FUNCTIONS

## fill\_this\_in

## fill\_in\_string

## fill\_in\_file

The API is the same as See ["fill\_this\_in" in Text::Template](https://metacpan.org/pod/Text::Template#fill_this_in), with the
addition of the `localize` and `trackvarpkg` options (see ["fill\_in"](#fill_in)).

# EXPORT

The following are available for export (the same as [Text::Template](https://metacpan.org/pod/Text::Template)):

- fill\_in\_file
- fill\_in\_string
- TTerror

# BUGS

Please report any bugs or feature requests to `bug-text-template-local at rt.cpan.org`, or through
the web interface at [http://rt.cpan.org/NoAuth/ReportBug.html?Queue=Text-Template-Local](http://rt.cpan.org/NoAuth/ReportBug.html?Queue=Text-Template-Local).  I will be notified, and then you'll
automatically be notified of progress on your bug as I make changes.

# SUPPORT

You can find documentation for this module with the perldoc command.

    perldoc Text::Template::Local

You can also look for information at:

- RT: CPAN's request tracker (report bugs here)

    [http://rt.cpan.org/NoAuth/Bugs.html?Dist=Text-Template-Local](http://rt.cpan.org/NoAuth/Bugs.html?Dist=Text-Template-Local)

- Search CPAN

    [http://search.cpan.org/dist/Text-Template-Local/](http://search.cpan.org/dist/Text-Template-Local/)

# ACKNOWLEDGEMENTS

Mark Jason Dominus for [Text::Template](https://metacpan.org/pod/Text::Template)

# AUTHOR

Diab Jerius, `<djerius at cpan.org>`

# LICENSE AND COPYRIGHT

Copyright (C) 2013 Mark Jason Dominus

Copyright (C) 2014 Smithsonian Astrophysical Observatory

Copyright (C) 2014 Diab Jerius

Text::Template::LocalVars is free software: you can redistribute it
and/or modify it under the terms of the GNU General Public License as
published by the Free Software Foundation, either version 3 of the
License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
