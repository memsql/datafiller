#! /usr/bin/env python
# -*- coding: utf-8 -*-
#
# $Id: datafiller.py 792 2014-03-23 09:23:40Z fabien $
#
# TODO:
#
# - other types: geometry? json/xml? others? extensibility?
# - disable/enable constraints? within a transaction?
#   not possible with PostgreSQL, must DROP/CREATE? defer?
# - improve "parsing" regexpr...
# - also CREATE UNIQUE INDEX would be useful to parse?
# - real parser? PLY? see http://nedbatchelder.com/text/python-parsers.html
# - check for skip on referenced fk?
#   implement skip with a DELETE CASCADE?
# - option/directive to change: +* bounds?
# - int: add rot(ation)? bit permutation?
# - more examples in documentation?
# - non int FK? no??
# - --test="!inet size=10" # bof? feature? use share?
# - update online blog for new syntax & generators
# - add something --test=<...> in (advanced) tutorial
# - alt pattern: possibly add weigths "{...#3}"?
# - add special size=max?
# - when sub generators are used, size is +- ignored
#   size is not set consistently when subgens?
# - allow nested [:...:]? hmmm... can always use a macro.
# - the value/macro constraint is somehow awkward at times, requiring
#   cascading macros. not sure how it could be simplified.
# - embed latin1 test? depends on locally available locales?
#
# NOT TO DO:
# - pg_dump -s ... | datafiller.py ... ## nope, how to set directives?!
# - remove 'sub', use int=XXX instead?
#   *NO*, sub is used for all generators which inherit from int.

# see other python2/3 compatibility hacks below...
from __future__ import print_function

VERSION = '2.0.0'
Id='$Id: datafiller.py 792 2014-03-23 09:23:40Z fabien $'

# extract revision informations from svn revision identifier
import re
revision, revdate, revyear = \
  re.search(r' (\d+) ((\d{4})-\d\d-\d\d) ', Id).group(1, 2, 3)

version = "{0} (r{1} on {2})".format(VERSION, revision, revdate)

# python 2/3 in-compatibility hacks -- this is tiring:-(
# - D.has_key(K) => K in D # better anyway
# - "..." % ... => "...".format(...) # advised, not necessary?
# - print ... => print(...) # ok
# - print => print('')      # '' is necessary for 2.7
# - xrange => range         # grrr...
# - a lot of previously implicit casts must be set with python3
#   s[:l] => s[:int(l)], sn * fp => sn * int(fp), ...
# - StringIO: changed place   # should be transparent
# - argparse is *in* python 2.7 & 3.2, outside before
# - 3.2/3.3: argparse.ArgumentParser(version="") is KO
# - 3.3 joke: u'unicode' is "accepted again" for str
#   python features may go at one version but come back at another...
# - another funny one: int.bit_length requires 2.7 or 3.1, but not in 3.0...
#   maybe it is because 3.0 was released one year before 2.7
# - module ipaddress: available from 3.3-
# - 1/2 == 0 (v2) vs 0.5 (v3)
# - in 3.0: the 'long' type has been renamed to 'int', and the 'int' type,
#   which was really a 'C long', has been dropped from user space (PEP237).
#   the '123L' syntax is now invalid.
# - also, str seems to have be replaced with unicode renamed as str,
#   and str has been renamed bytes.
# - sys.version_info.major: v >= 2.7
# - datetime.timedelta.total_seconds(): v >= 2.7
# - {:d} -> {0:d} format syntax for python 2.6
# - 3.2/3.3: random.vonmisesvariate slight change in output,
#   possibly a rounding issue or _acos change.
# - 2.6/2.7: random.expovariante change of output
# - expr if ... else expr not supported by 2.4
# - print is not an actual function in 2.5, cannot be redefined, and no end=
# - cannot use super() consistently between 2 & 3?
#   at least the compatible version is extra heavy.
# - stackoverflow help about python is randomly about python2 or python3
# - unicode handling/forcing is a pain, even with python3
# - python3.4 requires a flush on sys.stdout where 3.2/3.3 did not.
# - performance regressions 'internal' on a desktop: python3 is a joke...
#   2.6: 12.94, 2.7: 11.56, 3.2: 14.68, 3.3: 15.47, 3.4b1: 15.93
# - reduce as been moved in python3

import sys
if sys.version_info < (3,):
    # python 2
    from StringIO import StringIO # StringIO.StringIO...
    range = xrange # ah!
    numeric_types = (int, long, float, complex)
    u = lambda s: unicode(s, opts.encoding) # for outside strings
    u8 = lambda s: unicode(s, 'utf-8') # for strings in this file
    bytes = bytearray # ah!
    str = unicode # ah! should it be u? u8? depends?
    # see also print_function import above
    # see also print overriding below
else:
    # python 3
    from io import StringIO
    numeric_types = (int, float, complex)
    u = lambda s: s # hmmm... too late, managed by fileinput...
    u8 = lambda s: s # the file is in utf-8
    unichr = chr # ah!
    long = int # ah!
    from functools import reduce
    # see print overriding below

# plain old embedded documentation... Yes, the perl thing;-)
# could use pandoc/markdown, but it seems that pandoc cannot display
# a manual page interactively as pod2usage, and as a haskell script
# it is not installed by default. However, this POD thing breaks pydoc
# without a clear message about the issue...
POD = """

=pod

=head1 NAME

B<{name}> - generate random data from database schema extended with directives

=head1 SYNOPSIS

B<{name}> [--help --man ...] [schema.sql ...] > data.sql

=head1 DESCRIPTION

This script generates random data from a database schema enriched with
simple directives in SQL comments to drive {NGENS} data generators which
cover typical data types and their combination.
Reasonable defaults are provided, especially based on key and type constraints,
so that few directives should be necessary. The minimum setup is to specify the
relative size of tables with directive B<mult> so that data generation can
be scaled.

See the L</TUTORIAL> section below. Also, run with C<--validate=comics> or
C<--validate=library> and look at the output for didactic examples.

=head1 OPTIONS

=over 4

=item C<--debug> or C<-D>

Set debug mode.
Repeat for more.

Default is no debug.

=item C<--drop>

Drop tables before recreating them.
This implies option C<--filter>, otherwise there would be no table to fill.

Default is not to.

=item C<--encoding=enc> or C<-e enc>

Set this encoding for input and output files.

Default is no explicit encoding.

=item C<--filter> or C<-f>, reverse with C<--no-filter>

Work as a filter, i.e. send the schema input script to stdout and then
the generated data.
This is convenient to pipe the result of the script directly for
execution to the database command.

Default is to only ouput generated data.

=item C<--help> or C<-h>

Show basic help.

=item C<--man> or C<-m>

Show full man page based on POD. Yes, the perl thing:-)

=item C<--null RATE> or C<-n RATE>

Probability to generate a null value for nullable attributes.

Default is 0.01, which can be overriden by the B<null> directive at
the schema level, or per-attributes provided B<null> rate.

=item C<--offset OFFSET> or C<-O OFFSET>

Set default offset for integer generators on I<primary keys>.
This is useful to extend the already existing content of a database
for larger tests.

Default is 1, which can be overriden by the B<offset> directive at
the schema level, or per-attribute provided B<offset>.

=item C<--pod COMMAND>

Override pod conversion command used by option C<--man>.

Default is 'pod2usage -verbose 3'.

=item C<--quiet> or C<-q>

Generate less verbose SQL output.

Default is to generate one echo when starting to fill a table.

=item C<--seed SEED> or C<-S SEED>

Seed overall random generation with provided string.

Default uses OS supplied randomness or current time.

=item C<--size SIZE>

Set overall scaling. The size is combined with the B<mult> directive value
on a table to compute the actual number of tuples to generate in each table.

Default is 100, which can be overriden with the B<size> directive at the
schema level.

=item C<--target (postgresql|mysql)> or C<-t ...>

Target database engine. MySQL support is really experimental.

Default is to target PostgreSQL.

=item C<--test='directives...'>

Run tests for any generator with some directives.
If the directives start with B<!>, show an histogram.
If the directives start with B<->, show results on one-line.
This is convenient for unit testing, or to check what would
be the output of a set of directives.

Examples:
C<--size=100 --test='int size=10 mangle'> would show 100 integers
between 1 and 10 drawn uniformly.
C<--test='!bool rate=0.3'> may show I<False: 69.32%, True: 30.68%>,
stating the rate at which I<True> and I<False> were seen during the test.

Directive B<type> can be used within C<--test=...> to provide the target
SQL type when testing.

=item C<--transaction> or C<-T>

Use a global transaction.

Default is not to.

=item C<--tries=NUM>

How hard to try to satisfy a compound unique constraint before giving up
on a given tuple.

Default is 10.

=item C<--truncate>

Delete table contents before filling.

Default is not to.

=item C<--type=CUSTOM>

Add a custom type. The default generator for this type will rely
on a macro of the same name, if it exists. This option can be repeated
to add more custom types. See also the B<type> directive at the schema level.

=item C<--validate=(unit|internal|comics|library|pgbench)>

Output validation test cases. All but the C<unit> tests can
be processed with C<psql>.

  sh> {name} --validate=internal | psql

This option sets C<--filter> automatically, although it can
be forced back with C<--no-filter>.

Default is to process argument files or standard input.

=item C<--version>, C<-v> or C<-V>

Show script version.

=back

=head1 ARGUMENTS

Files containing SQL schema definitions. Standard input is processed if empty.

=head1 TUTORIAL

This tutorial introduces how to use DataFiller to fill a PostgreSQL database
for testing functionalities and performances.

=head2 DIRECTIVES IN COMMENTS

The starting point of the script to generate test data is the SQL schema
of the database taken from a file. It includes important information that
will be used to generate data: attribute types, uniqueness, not-null-ness,
foreign keys... The idea is to augment the schema with B<directives> in
comments so as to provide additional hints about data generation.

A datafiller directive is a special SQL comment recognized by the script, with
a C<df> marker at the beginning. A directive must appear I<after> the object
about which it is applied, either directly after the object declaration,
in which case the object is implicit, or much later, in which case the
object must be explicitely referenced:

  -- this directive sets the default overall size
    -- df: size=10
  -- this directive defines a macro named "fn"
    -- df fn: word=/path/to/file-containing-words
  -- this directive applies to table "Foo"
  CREATE TABLE Foo( -- df: mult=10.0
    -- this directive applies to attribute "fid"
    fid SERIAL -- df: offset=1000
    -- use defined macro, choose "stuff" from the list of words
  , stuff TEXT NOT NULL -- df: use=fn
  );
  -- ... much later
  -- this directive applies to attribute "fid" in table "Foo"
  -- df T=Foo A=fid: null=0.8

=head2 A SIMPLE LIBRARY EXAMPLE

Let us start with a simple example involving a library where
I<readers> I<borrow> I<books>.
Our schema is defined in file C<library.sql> as follows:

{library}

The first and only information you really need to provide is the relative
or absolute size of relations. For scaling, the best way is to specify a
relative size multiplier with the C<mult> directive on each table, which will
be multiplied by the C<size> option to compute the actual size of data to
generate in each table. Let us say we want 100 books in stock per reader,
with 1.5 borrowed books per reader on average:

  CREATE TABLE Book( -- df: mult=100.0
  ...
  CREATE TABLE Borrow( --df: mult=1.5

The default multiplier is 1.0, it does not need to be set on C<Reader>.
Then you can generate a data set with:

  sh> {name} --size=1000 library.sql > library_test_data.sql

Note that I<explicit> constraints are enforced on the generated data,
so that foreign keys in *Borrow* reference existing *Books* and *Readers*.
However the script cannot guess I<implicit> constraints, thus if an attribute
is not declared C<NOT NULL>, then some C<NULL> values I<will> be generated.
If an attribute is not unique, then the generated values will probably
not be unique.

=head2 IMPROVING GENERATED VALUES

In the above generated data, some attributes may not reflect the reality
one would expect from a library. Changing the default with per-attribute
directives will help improve this first result.

First, book titles are all quite short, looking like I<title_number>,
with some collisions. Indeed the default is to generate strings with a
common prefix based on the attribute name and a number drawn uniformly
from the expected number of tuples.
We can change to texts composed of between 1 and 7 English words taken
from a dictionary:

  title TEXT NOT NULL
  -- df English: word=/etc/dictionaries-common/words
  -- df: text=English length=4 lenvar=3

Also, we may have undesirable collisions on the ISBN attribute, because
the default size of the set is the number of tuples in the table. We
can extend the size at the attribute level so as to avoid this issue:

  isbn ISBN13 NOT NULL -- df: size=1000000000

If we now look at readers, the result can also be improved.
First, we can decide to keep the prefix and number form, but make the
statistics more in line with what you can find. Let us draw from 1000
firstnames, most frequent 3%, and 10000 lastnames, most frequent 1%:

  firstname TEXT NOT NULL,
     -- df: sub=power prefix=fn size=1000 rate=0.03
  lastname TEXT NOT NULL,
     -- df: sub=power prefix=ln size=10000 rate=0.01

The default generated dates are a few days around now, which does not make
much sense for our readers' birth dates. Let us set a range of birth dates.

  birth DATE NOT NULL, -- df: start=1923-01-01 end=2010-01-01

Most readers from our library are female: we can adjust the rate
so that 25% of readers are male, instead of the default 50%.

  gender BOOLEAN NOT NULL, -- df: rate=0.25

Phone numbers also have a I<prefix_number> structure, which does not really
look like a phone number. Let us draw a string of 10 digits, and adjust the
nullable rate so that 1% of phone numbers are not known.
We also set the size manually to avoid too many collisions, but we could
have chosen to keep them as is, as some readers do share phone numbers.

  phone TEXT
    -- these directives could be on a single line
    -- df: chars='0-9' length=10 lenvar=0
    -- df: null=0.01 size=1000000

The last table is about currently borrowed books.
The timestamps are around now, we are going to spread them on a period of
50 days, that is 24 * 60 * 50 = 72000 minutes (precision is 60 seconds).

  borrowed TIMESTAMP NOT NULL -- df: size=72000 prec=60

Because of the primary key constraint, the borrowed books are the first ones.
Let us mangle the result so that referenced book numbers are scattered.

  bid INTEGER REFERENCES Book -- df: mangle

Now we can generate improved data for our one thousand readers library,
and fill it directly to our B<library> database:

  sh> {name} --size=1000 --filter library.sql | psql library

Our test database is ready.
If we want more users and books, we only need to adjust the C<size> option.
Let us query our test data:

  -- show firstname distribution
  SELECT firtname, COUNT(*) AS cnt FROM Reader
  GROUP BY firstname ORDER BY cnt DESC LIMIT 3;
    -- fn_1_... | 33
    -- fn_2_... | 15
    -- fn_3_... | 12
  -- compute gender rate
  SELECT AVG(gender::INT) FROM Reader;
    -- 0.246

=head2 DISCUSSION

We could go on improving the generated data so that it is more realistic.
For instance, we could skew the I<borrowed> timestamp so that there are less
old borrowings, or skew the book number so that old books (lower numbers) are
less often borrowed, or choose firtnames and lastnames from actual lists.

When to stop improving is not obvious:
On the one hand, real data may show particular distributions which impact
the application behavior and performance, thus it may be important to reflect
that in the test data.
On the other hand, if nothing is really done about readers, then maybe the
only relevant information is the average length of firstnames and lastnames
because of the storage implications, and that's it.

=head2 ADVANCED FEATURES

Special generators allow to combine or synchronize other generators.

Let us consider an new C<email> attribute for our I<Readers>, for which we
want to generate gmail or yahoo addresses. We can use a B<pattern> generator
for this purpose:

  email TEXT NOT NULL CHECK(email LIKE '%@%')
    -- df: pattern='[a-z]{{3,8}}\.[a-z]{{3,8}}@(gmail|yahoo)\.com'

The pattern sets 3 to 8 lower case characters, followed by a dot, followed
by 3 to 8 characters again, followed by either gmail or yahoo domains.
With this approach, everything is chosen uniformly: letters in first and last
names appear 1/26, their six possible sizes between 3 to 8 are equiprobable,
and each domain is drawn on average by half generated email addresses.

In order to get more control about these distributions, we could rely on
the B<chars> or B<alt> generators which can be skewed or weighted,
as illustrated in the next example.

Let us now consider an ip address for the library network.  We want that
20% comes from librarian subnet '10.1.0.0/16' and 80% from reader subnet
'10.2.0.0/16'. This can be achieved with directive B<alt> which tells to
make a weighted choice between macro-defined generators:

  -- define two macros
  -- df librarian: inet='10.1.0.0/16'
  -- df reader: inet='10.2.0.0/16'
  ip INET NOT NULL
  -- df: alt=reader:8,librarian:2
  -- This would do as well: --df: alt=reader:4,librarian

Let us finally consider a log table for data coming from a proxy, which
stores correlated ethernet and ip addresses, that is ethernet address always
get the same ip addess, and we have about 1000 distinct hosts:

     -- df distinct: int size=1000
  ,  mac MACADDR NOT NULL -- df share=distinct
  ,  ip INET NOT NULL -- df share=distinct inet='10.0.0.0/8'

Because of the B<share> directive, the same C<mac> and C<ip> will be generated
together in a pool of 1000 pairs of addresses.

=head2 TUTORIAL CONCLUSION

There are many more directives to drive data generation, from simple type
oriented generators to advanced combinators. See the documentation and
examples below.

For very application-specific constraints that would not fit any generator,
it is also possible to apply updates to modify the generated data afterwards.

=head1 DIRECTIVES AND DATA GENERATORS

Directives drive the data sizes and the underlying data generators.
They must appear in SQL comments I<after> the object on which they apply,
although possibly on the same line, introduced by S<'-- df: '>.

  CREATE TABLE Stuff(         -- df: mult=2.0
    id SERIAL PRIMARY KEY,    -- df: step=19
    data TEXT UNIQUE NOT NULL -- df: prefix=st length=30 lenvar=3
  );

In the above example, with option C<--size=1000>, 2000 tuples
will be generated I<(2.0*1000)> with B<id> I<1+(i*19)%2000> and
unique text B<data> of length about 30+-3 prefixed with C<st>.
The sequence for B<id> will be restarted at I<2001>.

The default size is the number of tuples of the containing table.
This implies many collisions for a I<uniform> generator.

=head2 DATA GENERATORS

There are {NGENS} data generators which are selected by the attribute type
or possibly directives. Most generators are subject to the B<null>
directive which drives the probability of a C<NULL> value.
There is also a special shared generator.

Generators are triggered by using directives of their names.
If none is specified, a default is chosen based on the attribute type.

=over 4

=item B<alt generator>

This generator aggregates other generators by choosing one.
The list of sub-generators must be specified as a list of comma-separated
weighted datafiller macros provided by directive B<alt>, see below.
These generator definition macros must contain an explicit directive
to select the underlying generator.

=item B<array generator>

This generator generates an SQL array from another generator.
The sub-generator is specified as a macro name with the B<array> directive.
It takes into account the B<length>, B<lenvar>, B<lenmin> and
B<lenmax> directives.

=item B<bit generator>

This generator handles BIT and VARBIT data to store sequences of bits.
It takes into account the B<length>, B<lenvar>, B<lenmin> and
B<lenmax> directives.

=item B<blob generator>

This is for blob types, such as PostgreSQL's C<BYTEA>.
It uses an B<int generator> internally to drive its extent.
It takes into account the B<length>, B<lenvar>, B<lenmin> and
B<lenmax> directives.
This generator does not support C<UNIQUE>, but uniqueness is very likely
if the blob B<length> is significant and the B<size> is large.

=item B<bool generator>

This generator is used for the boolean type.
It is subject to the B<rate> directive.

=item B<cat generator>

This generator aggregates other generators by I<concatenating> their textual
output.
The list of sub-generators must be specified as a list of comma-separated
datafiller macros provided by a B<cat> directive, see below.

=item B<chars generator>

This alternate generator for text types generates random string of characters.
It is triggered by the B<chars> directive.
In addition to the underlying B<int generator> which allows to select values,
another B<int generator> is used to build words from the provided list
of characters,
The B<cgen> directives is the name of a macro which specifies the
B<int generator> parameters for the random character selection.
It also takes into account the B<length>, B<lenvar>, B<lenmin> and
B<lenmax> directives.
This generator does not support C<UNIQUE>.

=item B<const generator>

This generator provides a constant text value.
It is driven by the B<const> directive.

=item B<count generator>

This generator provides a simple counter.
It takes into account directives B<start>, B<step> and B<format>.

=item B<date generator>

This generator is used for the date type.
It uses an B<int generator> internally to drive its extent.
Its internal working is subject to directives B<start>, B<end> and B<prec>.

=item B<ean generator>

This is for International Article Number (EAN!) generation.
It uses an B<int generator> internally, so the number of distinct numbers
can be adjusted with directive B<size>.
It takes into account the B<length> and B<prefix> directives.
Default is to generate EAN-13 numbers.
This generator does not support C<UNIQUE>, but uniqueness is very likely.

=item B<file generator>

Inline file contents.
The mandatory list of files is specified with directive B<files>.
See also directive B<mode>.

=item B<float generator>

This generator is used for floating point types.
The directive allows to specify the sub-generator to use,
see the B<float> directive below.
Its configuration also relies on directives B<alpha> and B<beta>.
It does not support C<UNIQUE>, but uniqueness is very likely.

=item B<inet generator>

This is for internet ip types, such as PostgreSQL's C<INET> and C<CIDR>.
It uses an B<int generator> internally to drive its extent.
It takes into account the B<network> directive to specify the target network.
Handling IPv6 networks requires module B<netaddr>.

=item B<int generator>

This generator is used directly for integer types, and indirectly
by other generators such as B<text>, B<word> and B<date>.
Its internal working is subject to directives: B<sub>, B<size> (or B<mult>),
B<offset>, B<shift>, B<step>, B<xor> and B<mangle>.

=item B<interval generator>

This generator is used for the time interval type.
It uses the B<int generator> internally to drive its extent.
See also the B<unit> directive.

=item B<isnull generator>

Generates the special NULL value.

=item B<luhn generator>

This is for numbers which use a Luhn's algorithm checksum, such
as bank card numbers.
It uses an B<int generator> internally, so the number of distinct numbers
can be adjusted with directive B<size>.
It takes into account the B<length> and B<prefix> directives.
Default B<length> is 16, default B<prefix> is empty.
This generator does not support C<UNIQUE>, but uniqueness is very likely.

=item B<mac generator>

This is for MAC addresses, such as PostgreSQL's C<MACADDR>.
It uses an B<int generator> internally, so the number of generated addresses
can be adjusted with directive B<size>.

=item B<pattern generator>

This alternative generator for text types generates text based on a regular
expression provided with the B<pattern> directive. It uses internally the
B<alt>, B<cat>, B<repeat>, B<const> and B<chars> generators.

=item B<reduce generator>

This generator applies the reduction operation specified by directive B<op>
to generators specified with B<reduce> as a comma-separated list of macros.

=item B<repeat generator>

This generator aggregates the repetition of another generator.
The repeated generator is specified in a macro with a B<repeat> directive,
and the number of repetitions relies on the B<extent> directive.
It uses an B<int generator> internally to drive the number of repetitions,
so it can be skewed by specifying a subtype with the B<sub> directive.

=item B<string generator>

This generator is used by default for text types.
This is a good generator for filling stuff without much ado.
It takes into account B<prefix>, and the length can be specified with
B<length>, B<lenvar> B<lenmin> and B<lenmax> directives.
The generated text is of length B<length> +- B<lenvar>,
or between B<lenmin> and B<lenmax>.
For C<CHAR(n)> and C<VARCHAR(n)> text types, automatic defaults are set.

=item B<text generator>

This aggregate generator generates aggregates of words drawn from
any other generator specified by a macro in directive B<text>.
It takes into account directives B<separator> for separator (default C< >),
B<prefix> and B<suffix> (default empty).
It also takes into account the B<length>, B<lenvar>, B<lenmin> and
B<lenmax> directives which handle the number of words to generate.
This generator does not support C<UNIQUE>, but uniqueness is very likely
for a text with a significant length drawn from a dictionary.

=item B<timestamp generator>

This generator is used for the timestamp type.
It is similar to the date generator but at a finer granularity.
The B<tz> directive allows to specify the target timezone.

=item B<tuple generator>

This aggregate generator generates composite types.
The list of sub-generators must be specified with B<tuple>
as a list of comma-seperated macros.

=item B<uuid generator>

This generator is used for the UUID type.
It is really a B<pattern> generator with a predefined pattern.

=item B<value generator>

This generator uses per-tuple values from another generator specified as a
macro name in the B<value> directive. If the same B<value> is specified more
than once in a tuple, the exact same value is generated.

=item B<word generator>

This alternate generator for text types is triggered by the B<word> directive.
It uses B<int generator> to select words from a list or a file.
This generator handles C<UNIQUE> if enough words are provided.

=back

=head2 GLOBAL DIRECTIVES

A directive macro can be defined and then used later by inserting its name
between the introductory C<df> and the C<:>. The specified directives are
stored in the macro and can be reused later.
For instance, macros B<words>, B<mangle> B<cfr> and B<cen> can be defined as:

  --df words: word=/etc/dictionaries-common/words sub=power alpha=1.7
  --df mix: offset=10000 step=17 shift=3
  --df cfr: sub=scale alpha=6.7
  --df cen: sub=scale alpha=5.9

Then they can be used in any datafiller directive with B<use=...>:

  --df: use=words use=mix
  --df: use=mix

Or possibly for chars generators with B<cgen=...>:

  --df: cgen=cfr chars='esaitnru...'

There are four predefined macros:
B<cfr> and B<cen> define skewed integer generators with the above parameters.
B<french>, B<english> define chars generators which tries to mimic the
character frequency of these languages.

The B<size>, B<offset>, B<null>, B<seed> and directives
can be defined at the schema level to override from the SQL script
the default size multiplier, primary key offset, null rate or seed.
However, they are ignored if the corresponding options are set.

The B<type> directive at the schema level allows to add custom types,
similarly the C<--type> option above.

=head2 TABLE DIRECTIVES

=over 4

=item B<mult=float>

Size multiplier for scaling, that is computing the number of tuples to
generate.
This directive is exclusive from B<size>.

=item B<nogen>

Do not generate data for this table.

=item B<null>

Set defaut B<null> rate for this table.

=item B<size=int>

Use this size, so there is no scaling with the C<--size> option
and B<mult> directive.
This directive is exclusive from B<mult>.

=item B<skip=float>

Skip (that is generate but do not insert) some tuples with this probability.
Useful to create some holes in data. Tables with a non-zero B<skip> cannot be
referenced.

=back

=head2 ATTRIBUTE DIRECTIVES

A specific generator can be specified by using its name in the directives,
otherwise a default is provided based on the attribute SQL type.
Possible generators are: {GLIST}.
See also the B<sub> directive to select a sub-generator to control skewness.

=over 4

=item B<alt=some:2,weighted,macros:2>

List of macros, possibly weighted (default weight is 1) defining the generators
to be used by an B<alt> generator.
These macros must include an explicit directive to select a generator.

=item B<array=macro>

Name of the macro defining an array built upon this generator for
the B<array> generator.
The macro must include an explicit directive to select a generator.

=item B<cat=list,of,macros>

List of macros defining the generators to be used by a B<cat> generator.
These macros must include an explicit directive to select a generator.

=item B<chars='0123456789A-Z\\n' cgen=macro>

The B<chars> directive triggers the B<chars generator> described above.
Directive B<chars> provides a list of characters which are used to build words,
possibly including character intervals with '-'. A leading '-' in the list
means the dash character as is.
Characters can be escaped in octal (e.g. C<\\041> for C<!>) or
in hexadecimal (e.g. C<\\x3D> for C<=>).
Unicode escapes are also supported: eg C<\\u20ac> for the Euro symbol
and C<\\U0001D11E> for the G-clef.
Also special escaped characters are: null C<\\0> (ASCII 0), bell C<\\a> (7),
backspace C<\\b> (8), formfeed C<\\f> (12), newline C<\\n> (10),
carriage return C<\\r> (13), tab C<\\t> (9) and vertical tab C<\\v> (11).
The macro name specified in directive B<cgen> is used to setup the character
selection random generator.

For exemple:

  ...
  -- df skewed: sub=power rate=0.3
  , stuff TEXT -- df: chars='a-f' sub=uniform size=23 cgen=skewed

The text is chosen uniformly in a list of 23 words, each word being
built from characters 'abcdef' with the I<skewed> generator described
in the corresponding macro definition on the line above.

=item B<const=str>

Specify the constant text to generate for the B<const> generator.
The constant text can contains the escaped characters described
with the B<chars> directive above.

=item B<extent=int> or B<extent=int-int>

Specify the extent of the repetition for the I<repeat> generator.
Default is 1, that is not to repeat.

=item B<files=str>

Path-separated patterns for the list for files used by the B<file> generator.
For instance to specify image files in the C<./img> UN*X subdirectory:

  files='./img/*.png:./img/*.jpg:./img/*.gif'

=item B<float=str>

The random sub-generators for floats are those provided by Python's C<random>:

=over 4

=item B<beta>

Beta distribution, B<alpha> and B<beta> must be >0.

=item B<exp>

Exponential distribution with mean 1.0 / B<alpha>

=item B<gamma>

Gamma distribution, B<alpha> and B<beta> must be >0.

=item B<gauss>

Gaussian distribution with mean B<alpha> and stdev B<beta>.

=item B<log>

Log normal distribution, see B<normal>.

=item B<norm>

Normal distribution with mean B<alpha> and stdev B<beta>.

=item B<pareto>

Pareto distribution with shape B<alpha>.

=item B<uniform>

Uniform distribution between B<alpha> and B<beta>.
This is the default distribution.

=item B<vonmises>

Circular data distribution, with mean angle B<alpha> in radians
and concentration B<beta>.

=item B<weibull>

Weibull distribution with scale B<alpha> and shape B<beta>.

=back

=item B<format=str>

Format output for the B<count> generator.
Default is C<d>.
For instance, setting C<08X> displays the counter as 0-padded 8-digits
uppercase hexadecimal.

=item B<inet=str>

Use to specify in which IPv4 or IPv6 network to generate addresses.
For instance, B<inet=10.2.14.0/24> chooses ip addresses between
C<10.2.14.1> and C<10.2.14.254>, that is network and broadcast addresses
are not generated.
Similarily, B<inet=fe80::/112> chooses addresses between
C<fe80::1> and C<fe80::ffff>.
The default subnet limit is B<24> for IPv4 and B<64> for IPv6.
A leading C<,> adds the network address,
a leading C<.> adds the broadcast address,
and a leading C<;> adds both, thus B<inet=';10.2.14.0/24'> chooses
ip addresses between C<10.2.14.0> and C<10.2.14.255>.

=item B<length=int lenvar=int lenmin=int lenmax=int>

Specify length either as length and variation or length bounds, for
generated characters of string data, number of words of text data
or blob.

=item B<mangle>

Whether to automatically choose random B<shift>, B<step> and B<xor> for
an B<int> generator.

=item B<mode=str>

Mode for handling files for the B<file> generator. The value is either
C<blob> for binaries or C<text> for text file in the current encoding.
Default is to use the binary format, as it is safer to do so.

=item B<mult=float>

Use this multiplier to compute the generator B<size>.
This directive is exclusive from B<size>.

=item B<nogen>

Do not generate data for this attribute, so it will get its default value.

=item B<null=float>

Probability of generating a null value for this attribute.
This applies to all generators.

=item B<offset=int shift=int step=int>

Various parameters for generated integers.
The generated integer is B<offset+(shift+step*i)%size>.
B<step> must not be a divider of B<size>, it is ignored and replaced
with 1 if so.

Defaults: offset is 1, shift is 0, step is 1.

=item B<op=(+|*|min|max|cat)>

Reduction operation for B<reduce> generator.

=item B<pattern=str>

Provide the regular expression for the B<pattern> generator.

They can involve character sequences like C<calvin>,
character escape sequences (octal, hexadecimal, unicode, special) as
in directive B<chars> above,
character classes like C<[a-z]> and C<[^a-z]> (exclusion),
character classes shortcuts like C<.> C<\\d> C<\\h> C<\\H> C<\\s> and C<\\w>
which stand for C<[{DOT}]> C<[0-9]> C<[0-9a-f]> C<[0-9A-F]>
C<[ \\f\\n\\r\\t\\v]> and C<[0-9a-zA-Z_]> respectively,
as well as POSIX character classes within C<[:...:]>,
for instance C<[:alnum:]> for C<[0-9A-Za-z]> or C<[:upper:]> for C<[A-Z]>.

Alternations are specified with C<|>, for instance C<(hello|world)>.
Repetitions can be specified after an object with C<{{3,8}}>,
which stands for repeat between 3 and 8 times.
The question mark C<?> is a shortcut for C<{{0,1}}>,
the star sign C<*> for C<{{0,8}}> and the plus sign C<+> for C<{{1,8}}>.

For instance:

  stuff TEXT NOT NULL -- df: pattern='[a-z]{{3,5}} ?(!!|\\.{{3}}).*'

means 3 to 5 lower case letters, maybe followed by a space, followed by
either 2 bangs or 3 dots, and ending with any non special character.

The special C<[:GEN ...:]> syntax allow to embedded a generator within
the generated pattern, possibly including specific directives. For
instance the following would generate unique email addresses because
of the embedded counter:

  email TEXT UNIQUE NOT NULL
    -- df: pattern='\w+\.[:count format=X:]@somewhere\.org'

=item B<prefix=str>

Prefix for B<string> B<ean> B<luhn> and B<text> generators.

=item B<rate=float>

For the bool generator, rate of generating I<True> vs I<False>.
Must be in [0, 1]. Default is I<0.5>.

For the int generator, rate of generating value 0 for generators
B<power> and B<scale>.

=item B<repeat=macro>

Macro which defines the generator to repeat for the repeat generator.
See also the B<extent> directive.

=item B<seed=str>

Set default global seed from the schema level.
This can be overriden by option C<--seed>.
Default is to used the default random generator seed, usually
relying on OS supplied randomness or the current time.

=item B<separator=str>

Word separator for B<text> generator. Default is C< > (space).

=item B<share=macro>

Specify the name of a macro which defines an int generator used for
synchronizing other generators. If several generators B<share> the
same macro, their values within a tuple are correlated between tuples.

=item B<size=int>

Number of underlying values to generate or draw from, depending on the
generator. For keys (primary, foreign, unique) , this is necessarily the
corresponding number of tuples.
This directive is exclusive from B<mult>.

=item B<start=date/time> , B<end=date/time>, B<prec=int>

For the B<date> and B<timestamp> generators,
issue from B<start> up to B<end> at precision B<prec>.
Precision is in days for dates and seconds for timestamp.
Default is to set B<end> to current date/time and B<prec> to
1 day for dates et 60 seconds for timestamps.
If both B<start> and B<end> are specified, the underlying size is
adjusted.

For example, to draw from about 100 years of dates ending on
January 19, 2038:

  -- df: end=2038-01-19 size=36525

=item B<sub=SUGENERATOR>

For integer generators, use this underlying sub-type generator.

The integer sub-types also applies to all generators which inherit from the
B<int> generator, namely B<blob> B<date> B<ean> B<file> B<inet> B<interval>
B<luhn> B<mac> B<string> B<text> B<timestamp> B<repeat> and B<word>.

The sub-generators for integers are:

=over 4

=item B<serial>

This is really a counter which generates distinct integers,
depending on B<offset>, B<shift>, B<step> and B<xor>.

=item B<uniform>

Generates uniform random number integers between B<offset> and B<offset+size-1>.
This is the default.

=item B<serand>

Generate integers based on B<serial> up to B<size>, then use B<uniform>.
Useful to fill foreign keys.

=item B<power> with parameter B<alpha> or B<rate>

Use probability to this B<alpha> power.
When B<rate> is specified, compute alpha so that value 0 is drawn
at the specified rate.
Uniform is similar to B<power> with B<alpha=1.0>, or I<B<rate>=1.0/size>
The higher B<alpha>, the more skewed towards I<0>.

Example distribution with C<--test='!int sub=power rate=0.3 size=10'>:

  value     0   1   2   3   4   5   6   7   8   9
  percent  30  13  10   9   8   7   6   6   5   5

=item B<scale> with parameter B<alpha> or B<rate>

Another form of skewing. The probability of increasing values drawn
is less steep at the beginning compared to B<power>, thus the probability
of values at the end is lower.

Example distribution with C<--test='!int sub=scale rate=0.3 size=10'>:

  value     0   1   2   3   4   5   6   7   8   9
  percent  30  19  12   9   7   6   5   4   3   2

=back

=item B<suffix=str>

Suffix for B<text> generator. Default is empty.

=item B<text=macro>

Macro which defines the word provided generator for the B<text> generator.

=item B<type=str>

At the schema level, add a custom type which will be recognized as such
by the schema parser.
At the attribute level, use the generator for this type.

=item B<unit=str>

The B<unit> directive specifies the unit of the generated intervals.
Possible values include B<s m h d mon y>. Default is B<s>, i.e. seconds.

=item B<word=file> or B<word=:list,of,words>

The B<word> directive triggers the B<word generator> described above,
or is used as a source for words by the B<text generator>.
Use provided word list or lines of file to generate data.
The default B<size> is the size of the word list.

If the file contents is ordered by word frequency, and the int generator is
skewed (see B<sub>), the first words can be made to occur more frequently.

=item B<xor=int>

The B<xor> directive adds a non-linear xor stage for the B<int> generator.

=back

=head1 EXAMPLES

The first example is taken from B<pgbench>.
The second example is a didactic schema to illustrate directives.
See also the B<library> example in the tutorial above.
As these schemas are embedded into this script, they can be invoked
directly with the C<--test> option:

  sh> {name} --test=pgbench -T --size=10 | psql pgbench
  sh> {name} --test=comics -T --size=10 | psql comics
  sh> {name} --test=library -T --size=1000 | psql library

=head2 PGBENCH SCHEMA

This schema is taken from the TCP-B benchmark.
Each B<Branch> has B<Tellers> and B<Accounts> attached to it.
The B<History> records operations performed when the benchmark is run.

{pgbench}

The integer I<*balance> figures are generated with a skewed generator
defined in macro B<regress>. The negative B<offset> setting on I<abalance>
will help generate negative values, and the I<regress> skewed generator
will make small values more likely.

If this is put in a C<tpc-b.sql> file, then working test data can be
generated with:

  sh> {name} -f -T --size=10 tpc-b.sql | psql bench

=head2 COMICS SCHEMA

This schema models B<Comics> books written in a B<Language> and
published by a B<Publisher>. Each book can have several B<Author>s
through B<Written>. The B<Inventory> tells on which shelf books are
stored. Some of the books may be missing and a few may be available
twice or more.

{comics}

=head1 WARNINGS, BUGS, FEATURES AND GRUMBLING

All software has bug, this is a software, hence it has bugs.

If you find one, please sent a report, or even better a patch that fixes it!

BEWARE, you may loose your hairs or your friends by using this script.
Do not use this script on a production database and without dumping
your data somewhere safe beforehand.

There is no SQL parser, table and attributes are analysed with
optimistic regular expressions.

Foreign keys cannot reference compound keys.

Handling of quoted identifiers is partial and may not work at all.

Beware that unique constraint checks for big data generation may require
a lot of memory.

The script works more or less with Python versions 2.6, 2.7, 3.2, 3.3 and 3.4.
However Python compatibility between major or even minor versions is not the
language's I<forte>. The script tries to remain agnostic with respect to
this issue at the price of some hacks. Sometimes it fails. Complain to
Python's developers who do not care about script stability, and in some
way do not respect other people work. See discussion in PEP387, which
in my opinion is not always implemented.

Python is quite paradoxical, as it can achieve I<forward> compatibility
(C<from __future__ import ...>) but often fails miserably at I<backward>
compatibility.  Guido, this is the other way around!  The useful feature
is to make newer versions handle older scripts.

=head1 LICENSE

=for html
<img src="https://www.gnu.org/graphics/gplv3-127x51.png"
alt="GNU GPLv3" align="right" />

Copyright 2013-{year} Fabien Coelho <fabien at coelho dot net>.

This is free software, both inexpensive and with sources.

The GNU General Public License v3 applies, see
L<http://www.gnu.org/copyleft/gpl.html> for details.

The summary is: you get as much as you paid for, and I am not responsible
for anything.

If you are happy with this software, feel free to send me a postcard saying so!
See my web page for current address L<http://www.coelho.net/>.

If you have ideas or patches to improve this script, send them to me!

=head1 DOWNLOAD

This is F<{name}> version {version}.

Latest version and online documentation should be available from
L<http://www.coelho.net/datafiller.html>.

Download script: L<https://www.cri.ensmp.fr/people/coelho/datafiller>.

=head1 VERSIONS

History of versions:

=over 4

=item B<version {version}>

=over 4

=item New generators

Add regular expression B<pattern> generator.
Add B<luhn> generator for data which use Luhn's algorithm checksum,
such as bank card numbers.
Add B<ean> generator for supporting B<EAN13>, B<ISBN13>, B<ISSN13>, B<ISMN13>,
B<UPC>, B<ISBN>, B<ISSN> and B<ISMN> types.
Add B<file> generator to inline file contents.
Add B<uuid> generator for Universally Unique IDentifiers.
Add B<bit> generator for BIT and VARBIT types.
Add aggregate generators B<alt>, B<array>, B<cat>, B<reduce>, B<repeat>
and B<tuple>.
Add simple B<isnull>, B<const> and B<count> generators.
Add special B<share> generator synchronizer, which allow to generate
correlated values within a tuple. Add special B<value> generator which
allow to generate the exact same value within a tuple.

=item Changes

Simplify and homogenize per-attribute generator selection, and possibly
its subtype for B<int> and B<float>.
Remove B<nomangle> directive.
Remove B<mangle> directive from table and schema levels.
Remove B<--mangle> option.
Improve B<chars> directive to support character intervals with '-'
and various escape characters (octal, hexadecimal, unicode...).
Use B<--test=...> for unit testing and B<--validate=...> for the
validation test cases.
These changes are incompatible with prior versions and may require modifying
some directives in existing schemas or scripts.

=item Enhancements

Add a non-linear B<xor> stage to the B<int> generator.
Integer mangling now relies on more and larger primes.
Check that directives B<size> and B<mult> are exclusive.
Add the B<type> directive at the schema level.
Improve B<inet> generator to support IPv6 and not to generate
by default network and broadcast addresses in a network;
adding leading characters C<,.;> to the network allows to change this behavior.
Add B<lenmin> and B<lenmax> directives to specify a length.
Be more consistent about seeding to have deterministic results for some tests.

=item Options

Add C<--quiet>, C<--encoding> and C<--type> options.
Make C<--test=...> work for all actual data generators.
Make C<--validate=...> handle all validation cases.
Add internal self-test capabilities.

=item Bug fixes

Check directives consistency.
Do ignore commented out directives, as they should be.
Generate escaped strings where appropriate for PostgreSQL.
Handle size better in generators derived from B<int> generator.
Make UTF-8 and other encodings work with both Python 2 and 3.
Make it work with Python 2.6 and 3.2.

=item Documentation

Improved documentation and examples, including a new L</ADVANCED FEATURES>
Section in the L</TUTORIAL>.

=back

=item B<version 1.1.5 (r360 on 2013-12-03)>

Improved user and code documentations.
Improved validation.
Add C<--no-filter> option for some tests.
Add some support for INET, CIDR and MACADDR types.

=item B<version 1.1.4 (r326 on 2013-11-30)>

Improved documentation, in particular add a L</TUTORIAL> section and
reorder some other sections.
Add some C<pydoc> in the code.
Memoize enum regular expression.
Raise an error if no generator can be created for an attribute,
instead of silently ignoring it.
Fix a bug which could generate primary key violations when handling
some unique constraints.

=item B<version 1.1.3 (r288 on 2013-11-29)>

Improved documentation, including a new L</KEYWORDS> section.
Check consistency of parameters in B<float generator>.
Add some partial support for C<ALTER TABLE>.
Multiple C<--debug> increases verbosity, especially in the parser stage.
Support settings directives out of declarations.
Better support of PostgreSQL integer types.
Add separate L</VERSIONS> section.
One fix for python 3.3...

=item B<version 1.1.2 (r268 on 2013-06-30)>

Improved and simplified code, better comments and validation.
Various hacks for python 2 & 3 compatibility.
Make validations stop on errors.
Check that B<lenvar> is less than B<length>.
Fixes for B<length> and B<lenvar> overriding in B<string generator>.

=item B<version 1.1.1 (r250 on 2013-06-29)>

Minor fix to the documentation.

=item B<version 1.1.0 (r248 on 2013-06-29)>

Improved documentation, code and comments.
Add C<--test> option for demonstration and checks,
including an embedded validation.
Add C<--validate> option as a shortcut for script validation.
Add B<seed>, B<skip>, B<mangle> and B<nomangle> directives.
Add B<null> directive on tables.
Add B<nogen> and B<type> directives on attributes.
Accept size 0.
Change B<alpha> behavior under B<float sub=scale> so that the higher B<alpha>,
the more skewed towards 0.
Add alternative simpler B<rate> specification for B<scale> and B<power>
integer generators.
Deduce B<size> when both B<start> and B<end> are specified for the
date and timestamp generators.
Add B<tz> directive for the timestamp generator.
Add float, interval and blob generators.
Add some support for user-defined enum types.
Add C<--tries> option to control the effort to satisfy C<UNIQUE> constraints.
Add support for non integer foreign keys.
Remove B<square> and B<cube> integer generators.
Change macro definition syntax so as to be more intuitive.
Add C<-V> option for short version.
Some bug fixes.

=item B<version 1.0.0 (r128 on 2013-06-16)>

Initial distribution.

=back

=head1 KEYWORDS

Automatically populate, fill PostgreSQL database with test data.
Generate test data with a data generation tool (data generator) for PostgreSQL.
Import test data into PostgreSQL.

=head1 SEE ALSO

Relational data generation tools are often GUI or even Web applications,
possibly commercial. I did not find a simple filter-oriented tool driven
by directives, and I wanted to do something useful to play with python.

=over 4

=item L<http://en.wikipedia.org/wiki/Test_data_generation>

=item L<http://generatedata.com/> PHP/MySQL

=item L<https://github.com/francolaiuppa/datafiller> unrelated PHP/* project.

=item L<http://www.databasetestdata.com/> generate one table

=item L<http://www.mobilefish.com/services/random_test_data_generator/random_test_data_generator.php>

=item L<http://www.sqledit.com/dg/> Win GUI/MS SQL Server, Oracle, DB2

=item L<http://sourceforge.net/projects/spawner/> Pascal, GUI for MySQL

=item L<http://sourceforge.net/projects/dbmonster/> 2005 - Java from XML
for PostgreSQL, MySQL, Oracle

=item L<http://sourceforge.net/projects/datagenerator/> 2006, alpha in Pascal, Win GUI

=item L<http://sourceforge.net/projects/dgmaster/> 2009, Java, GUI

=item L<http://sourceforge.net/projects/freshtrash/> 2007, Java

=item L<http://www.gsapps.com/products/datagenerator/> GUI/...

=item L<http://rubyforge.org/projects/datagen>

=item L<http://msdn.microsoft.com/en-us/library/dd193262%28v=vs.100%29.asp>

=item L<http://stackoverflow.com/questions/3371503/sql-populate-table-with-random-data>

=item L<http://databene.org/databene-benerator>

=item L<http://www.daansystems.com/datagen/> GUI over ODBC

=item Perl (Data::Faker Data::Random), Ruby (Faker ffaker), Python random

=back

=cut

"""

PGBENCH = """
  -- TPC-B example adapted from pgbench

  -- df regress: int sub=power alpha=1.5
  -- df: size=1

  CREATE TABLE pgbench_branches( -- df: mult=1.0
    bid SERIAL PRIMARY KEY,
    bbalance INTEGER NOT NULL,   -- df: size=100000000 use=regress
    filler CHAR(88) NOT NULL
  );

  CREATE TABLE pgbench_tellers(  -- df: mult=10.0
    tid SERIAL PRIMARY KEY,
    bid INTEGER NOT NULL REFERENCES pgbench_branches,
    tbalance INTEGER NOT NULL,   -- df: size=100000 use=regress
    filler CHAR(84) NOT NULL
  );

  CREATE TABLE pgbench_accounts( -- df: mult=100000.0
    aid BIGSERIAL PRIMARY KEY,
    bid INTEGER NOT NULL REFERENCES pgbench_branches,
    abalance INTEGER NOT NULL,   -- df: offset=-1000 size=100000 use=regress
    filler CHAR(84) NOT NULL
  );

  CREATE TABLE pgbench_history(  -- df: nogen
    tid INTEGER NOT NULL REFERENCES pgbench_tellers,
    bid INTEGER NOT NULL REFERENCES pgbench_branches,
    aid BIGINT NOT NULL REFERENCES pgbench_accounts,
    delta INTEGER NOT NULL,
    mtime TIMESTAMP NOT NULL,
    filler CHAR(22)
    -- UNIQUE (tid, bid, aid, mtime)
  );
"""

# embedded PostgreSQL validation
INTERNAL = """

-- df: size=2000 null=0.0

-- define some macros:
-- df name: chars='a-z' length=6 lenvar=2
-- df dot: word=':.'
-- df domain: word=':@somewhere.org'
-- df 09: int offset=0 size=10
-- df AZ: chars='A-Z' length=2 lenvar=0
-- df ELEVEN: int size=11

CREATE SCHEMA df;

CREATE TYPE df.color AS ENUM ('red','blue','green');

-- an address type and its generator
CREATE TYPE df.addr AS (nb INTEGER, road TEXT, code VARCHAR(6), city TEXT);

-- df addr_nb: int size=1000 offset=1 sub=scale rate=0.01 null=0.0
-- df addr_rd: pattern='[A-Z][a-z]{2,8} ([A-Z][a-z]{3,7} )?(st|av|rd)' null=0.0
-- df CITY: int size=100 sub=scale rate=0.5
-- df addr_po: pattern='[0-9]{5}' null=0.0 share=CITY
-- df addr_ct: pattern='[A-Z][a-z]+' null=0.0 share=CITY
-- df addr_tuple: tuple=addr_nb,addr_rd,addr_po,addr_ct

CREATE TABLE df.Stuff( -- df: mult=1.0
  id SERIAL PRIMARY KEY
  -- *** INTEGER ***
, i0 INTEGER CHECK(i0 IS NULL) -- df: null=1.0 size=1
, i1 INTEGER CHECK(i1 IS NOT NULL AND i1=1) -- df: null=0.0 size=1
, i2 INTEGER NOT NULL CHECK(i2 BETWEEN 1 AND 6) --df: size=5
, i3 INTEGER UNIQUE -- df: offset=1000000
, i4 INTEGER CHECK(i2 BETWEEN 1 AND 6) -- df: sub=power rate=0.7 size=5
, i5 INTEGER CHECK(i2 BETWEEN 1 AND 6) -- df: sub=scale rate=0.7 size=5
, i6 INT8  -- df: size=1800000000000000000 offset=-900000000000000000
, i7 INT4  -- df: size=4000000000 offset=-2000000000
, i8 INT2  -- df: size=65000 offset=-32500
  -- *** BOOLEAN ***
, b0 BOOLEAN NOT NULL
, b1 BOOLEAN -- df: null=0.5
, b2 BOOLEAN NOT NULL -- df: rate=0.7
  -- *** FLOAT ***
, f0 REAL NOT NULL CHECK (f0 >= 0.0 AND f0 < 1.0)
, f1 DOUBLE PRECISION -- df: float=gauss alpha=5.0 beta=2.0
, f2 DOUBLE PRECISION CHECK(f2 >= -10.0 AND f2 < 10.0)
    -- df: float=uniform alpha=-10.0 beta=10.0
, f3 DOUBLE PRECISION -- df: float=beta alpha=1.0 beta=2.0
-- 'exp' output changes between 2.6 & 2.7
-- , f4 DOUBLE PRECISION -- df: float=exp alpha=0.1
, f5 DOUBLE PRECISION -- df: float=gamma alpha=1.0 beta=2.0
, f6 DOUBLE PRECISION -- df: float=log alpha=1.0 beta=2.0
, f7 DOUBLE PRECISION -- df: float=norm alpha=20.0 beta=0.5
, f8 DOUBLE PRECISION -- df: float=pareto alpha=1.0
-- 'vonmises' output changes between 3.2 and 3.3
-- , f9 DOUBLE PRECISION -- df: float=vonmises alpha=1.0 beta=2.0
, fa DOUBLE PRECISION -- df: float=weibull alpha=1.0 beta=2.0
, fb NUMERIC(2,1) CHECK(fb BETWEEN 0.0 AND 9.9)
    -- df: float=uniform alpha=0.0 beta=9.9
, fc DECIMAL(5,2) CHECK(fc BETWEEN 100.00 AND 999.99)
    -- df: float=uniform alpha=100.0 beta=999.99
  -- *** DATE ***
, d0 DATE NOT NULL CHECK(d0 BETWEEN '2038-01-19' AND '2038-01-20')
     -- df: size=2 start='2038-01-19'
, d1 DATE NOT NULL CHECK(d1 = DATE '2038-01-19')
     -- df: start=2038-01-19 end=2038-01-19
, d2 DATE NOT NULL
       CHECK(d2 = DATE '2038-01-19' OR d2 = DATE '2038-01-20')
       -- df: start=2038-01-19 size=2
, d3 DATE NOT NULL
       CHECK(d3 = DATE '2038-01-18' OR d3 = DATE '2038-01-19')
       -- df: end=2038-01-19 size=2
, d4 DATE NOT NULL
       CHECK(d4 = DATE '2013-06-01' OR d4 = DATE '2013-06-08')
       -- df: start=2013-06-01 end=2013-06-08 prec=7
, d5 DATE NOT NULL
       CHECK(d5 = DATE '2013-06-01' OR d5 = DATE '2013-06-08')
       -- df: start=2013-06-01 end=2013-06-14 prec=7
  -- *** TIMESTAMP ***
, t0 TIMESTAMP NOT NULL
          CHECK(t0 = TIMESTAMP '2013-06-01 00:00:05' OR
                t0 = TIMESTAMP '2013-06-01 00:01:05')
          -- df: start='2013-06-01 00:00:05' end='2013-06-01 00:01:05'
, t1 TIMESTAMP NOT NULL
          CHECK(t1 = TIMESTAMP '2013-06-01 00:02:00' OR
                t1 = TIMESTAMP '2013-06-01 00:02:05')
          -- df: start='2013-06-01 00:02:00' end='2013-06-01 00:02:05' prec=5
, t2 TIMESTAMP NOT NULL
          CHECK(t2 >= TIMESTAMP '2013-06-01 01:00:00' AND
                t2 <= TIMESTAMP '2013-06-01 02:00:00')
          -- df: start='2013-06-01 01:00:00' size=30 prec=120
, t3 TIMESTAMP WITH TIME ZONE NOT NULL
          CHECK(t3 = TIMESTAMP '2013-06-22 09:17:54 CEST')
          -- df: start='2013-06-22 07:17:54' size=1 tz='UTC'
  -- *** INTERVAL ***
, v0 INTERVAL NOT NULL CHECK(v0 BETWEEN '1 s' AND '1 m')
     -- df: size=59 offset=1 unit='s'
, v1 INTERVAL NOT NULL CHECK(v1 BETWEEN '1 m' AND '1 h')
     -- df: size=59 offset=1 unit='m'
, v2 INTERVAL NOT NULL CHECK(v2 BETWEEN '1 h' AND '1 d')
     -- df: size=23 offset=1 unit='h'
, v3 INTERVAL NOT NULL CHECK(v3 BETWEEN '1 d' AND '1 mon')
     -- df: size=29 offset=1 unit='d'
, v4 INTERVAL NOT NULL CHECK(v4 BETWEEN '1 mon' AND '1 y')
     -- df: size=11 offset=1 unit='mon'
, v5 INTERVAL NOT NULL -- df: size=100 offset=0 unit='y'
, v6 INTERVAL NOT NULL -- df: size=1000000 offset=0 unit='s'
  -- *** TEXT ***
, s0 CHAR(12) UNIQUE NOT NULL
, s1 VARCHAR(15) UNIQUE NOT NULL
, s2 TEXT NOT NULL -- df: length=23 lenvar=1 size=20 seed=s2
, s3 TEXT NOT NULL CHECK(s3 LIKE 'stuff%') -- df: prefix='stuff'
, s4 TEXT NOT NULL CHECK(s4 ~ '^[a-f]{9,11}$')
    -- df: chars='abcdef' size=20 length=10 lenvar=1
, s5 TEXT NOT NULL CHECK(s5 ~ '^[ab]{30}$')
    -- df skewed: int sub=scale rate=0.7
    -- df: chars='ab' size=50 length=30 lenvar=0 cgen=skewed
, s6 TEXT NOT NULL -- df: word=:calvin,hobbes,susie
, s7 TEXT NOT NULL -- df: word=:one,two,three,four,five,six,seven size=3 mangle
, s8 TEXT NOT NULL CHECK(s8 ~ '^((un|deux) ){3}(un|deux)$')
    -- df undeux: word=:un,deux
    -- df: text=undeux length=4 lenvar=0
, s9 VARCHAR(10) NOT NULL CHECK(LENGTH(s9) BETWEEN 8 AND 10)
  -- df: length=9 lenvar=1
, sa VARCHAR(8) NOT NULL CHECK(LENGTH(sa) BETWEEN 6 AND 8) -- df: lenvar=1
, sb TEXT NOT NULL CHECK(sb ~ '^10\.\d+\.\d+\.\d+$')
  -- df: inet='10.0.0.0/8'
, sc TEXT NOT NULL CHECK(sc ~ '^([0-9A-F][0-9A-F]:){5}[0-9A-F][0-9A-F]$')
  -- df: mac
, sd TEXT NOT NULL CHECK(sd ~ '^[ \\007\\n\\f\\v\\t\\r\\\\]{10}')
  -- df: chars=' \\a\\n\\t\\f\\r\\v\\\\' length=10 lenvar=0
  -- user defined *** ENUM ***
, e0 df.color NOT NULL
  -- *** EAN *** and other numbers
, e1 EAN13 NOT NULL
, e2 ISBN NOT NULL
, e3 ISMN NOT NULL
, e4 ISSN NOT NULL
, e5 UPC NOT NULL
, e6 ISBN13 NOT NULL
, e7 ISMN13 NOT NULL
, e8 ISSN13 NOT NULL
, e9 CHAR(16) NOT NULL CHECK(e9 ~ '^[0-9]{16}$') -- df: luhn
, ea CHAR(12) NOT NULL CHECK(ea ~ '^1234[0-9]{8}$')
    -- df: luhn prefix=1234 length=12
  -- *** BLOB ***
, l0 BYTEA NOT NULL
, l1 BYTEA NOT NULL CHECK(LENGTH(l1) = 3) -- df: length=3 lenvar=0
, l2 BYTEA NOT NULL CHECK(LENGTH(l2) BETWEEN 0 AND 6) -- df: length=3 lenvar=3
  -- *** INET ***
, n0 INET NOT NULL CHECK(n0 << INET '10.2.14.0/24') -- df: inet=10.2.14.0/24
, n1 CIDR NOT NULL CHECK(n1 << INET '192.168.0.0/16')
    -- df: inet=192.168.0.0/16
, n2 MACADDR NOT NULL
, n3 MACADDR NOT NULL -- df: size=17
, n4 INET NOT NULL CHECK(n4 = INET '1.2.3.5' OR n4 = INET '1.2.3.6')
    -- df: inet='1.2.3.4/30'
, n5 INET NOT NULL CHECK(n5 = INET '1.2.3.0' OR n5 = INET '1.2.3.1')
    -- df: inet=';1.2.3.0/31'
, n6 INET NOT NULL CHECK(n6::TEXT ~ '^fe80::[1-9a-f][0-9a-f]{0,3}/128$')
    -- df: inet='fe80::/112'
  -- *** AGGREGATE GENERATORS ***
, z0 TEXT NOT NULL CHECK(z0 ~ '^\w+\.\w+\d@somewhere\.org$')
  -- df: cat=name,dot,name,09,domain
, z1 TEXT NOT NULL CHECK(z1 ~ '^([0-9]|[A-Z][A-Z])$')
  -- df: alt=09,AZ:9
, z2 TEXT NOT NULL CHECK(z2 ~ '^[A-Z]{6,10}$')
  -- df: repeat=AZ extent=3-5
  -- *** SHARED GENERATOR ***
, h0 TEXT NOT NULL CHECK(h0 LIKE 'X%')
  -- df: share=ELEVEN size=1000000 prefix='X'
, h1 TEXT NOT NULL CHECK(h1 LIKE 'Y%')
  -- df: share=ELEVEN size=2000000 prefix='Y'
, h2 TEXT NOT NULL CHECK(h2 LIKE 'Z%')
  -- df: share=ELEVEN size=3000000 prefix='Z'
, h3 DOUBLE PRECISION NOT NULL
  -- df: share=ELEVEN float=uniform alpha=1.0 beta=2.0
, h4 DOUBLE PRECISION NOT NULL
  -- df: share=ELEVEN float=gamma alpha=2.0 beta=3.0
, h5 INTEGER NOT NULL CHECK(h5 BETWEEN 1 AND 1000000)
  -- df: share=ELEVEN offset=1 size=1000000
  -- df one2five: word=:one,two,three,four,five
, h6 TEXT NOT NULL -- df: share=ELEVEN text=one2five
  -- *** MISC GENERATORS ***
, u0 UUID NOT NULL
, u1 CHAR(36) NOT NULL
    CHECK(u1 ~ '^[0-9a-fA-F]{4}([0-9a-fA-F]{4}-){4}[0-9a-fA-F]{12}$')
    -- df: uuid
, u2 BIT(3) NOT NULL
, u3 VARBIT(7) NOT NULL
, u4 TEXT NOT NULL CHECK(u4 ~ '^[01]{8}') -- df: bit length=8
);

ALTER TABLE df.Stuff
  ADD COLUMN a0 INTEGER
, ADD COLUMN a1 INTEGER CHECK(a1 BETWEEN 3 AND 5)
, ADD COLUMN a2 INTEGER NOT NULL
, ADD COLUMN a3 INTEGER
, ADD COLUMN a4 INTEGER;

ALTER TABLE df.Stuff
  ALTER COLUMN a1 SET NOT NULL,
  ADD CONSTRAINT a2_unique UNIQUE(a2),
  ADD CONSTRAINT a3_a4_unique UNIQUE(a3, a4);

CREATE TABLE df.ForeignKeys( -- df: mult=2.0
  id SERIAL PRIMARY KEY
, fk1 INTEGER NOT NULL REFERENCES df.stuff
, fk2 INTEGER REFERENCES df.Stuff -- df: null=0.5
, fk3 INTEGER NOT NULL REFERENCES df.Stuff -- df: sub=serial
, fk4 INTEGER NOT NULL REFERENCES df.Stuff -- df: sub=serial mangle
, fk5 INTEGER NOT NULL REFERENCES df.Stuff -- df: sub=serand
, fk6 INTEGER NOT NULL REFERENCES df.Stuff -- df: sub=serand mangle
, fk7 INTEGER NOT NULL REFERENCES df.Stuff(id) -- df: sub=serand mangle
, fk8 INTEGER NOT NULL REFERENCES df.stuff(i3) -- df: sub=serand mangle
, fk9 INTEGER NOT NULL REFERENCES df.stuff(i3) -- df: sub=uniform
, fka INTEGER NOT NULL REFERENCES df.stuff(i3) -- df: sub=scale rate=0.2
, fkb CHAR(12) NOT NULL REFERENCES df.stuff(s0)
);

CREATE TABLE df.NotFilled( -- df: nogen
  id SERIAL PRIMARY KEY CHECK(id=1)
);
INSERT INTO df.NotFilled(id) VALUES(1);

CREATE TABLE df.Ten( -- df: size=10 null=1.0
  id SERIAL PRIMARY KEY CHECK(id BETWEEN 18 AND 27) -- df: offset=18
, nogen INTEGER CHECK(nogen = 123) DEFAULT 123 -- df: nogen
, n TEXT
  -- forced generators
, x0 TEXT NOT NULL CHECK(x0 ~ '^[0-9]+$') -- df: int size=1000
, x1 TEXT NOT NULL CHECK(x1 = 'TRUE' OR x1 = 'FALSE') -- df: bool
, x2 TEXT NOT NULL CHECK(x2::DOUBLE PRECISION >=0 AND
                         x2::DOUBLE PRECISION <= 100.0)
                         -- df: float alpha=0.0 beta=100.0
, x3 TEXT NOT NULL CHECK(x3 ~ '^\d{4}-\d\d-\d\d$')
   -- df: date end=2013-12-16
, x4 TEXT NOT NULL CHECK(x4 ~ '^\d{4}-\d\d-\d\d \d\d:\d\d:\d\d$')
                        -- df: timestamp end='2038-01-19 03:14:07'
, x5 TEXT NOT NULL CHECK(x5 LIKE 'boo%') -- df: string prefix=boo
, x6 TEXT NOT NULL CHECK(x6 ~ '^\d+\s\w+$') -- df: interval unit='day'
  -- more forced generators
, y0 INTEGER NOT NULL CHECK(y0 BETWEEN 2 AND 29)
     -- df: word=:2,3,5,7,11,13,17,19,23,29
, y1 BOOLEAN NOT NULL -- df: word=:TRUE,FALSE
, y2 DOUBLE PRECISION NOT NULL CHECK(y2=0.0 OR y2=1.0) -- df: word=:0.0,1.0
, y3 FLOAT NOT NULL CHECK(y3=0.0 OR y3=1.0) -- df: word=:0.0,1.0
, y4 DATE NOT NULL CHECK(y4 = DATE '2013-06-23' OR y4 = DATE '2038-01-19')
     -- df: word=:2013-06-23,2038-01-19
, y5 TIMESTAMP NOT NULL CHECK(y5 = TIMESTAMP '2013-06-23 19:54:55')
     -- df: word=':2013-06-23 19:54:55'
, y6 INTEGER NOT NULL CHECK(y6::TEXT ~ '^[4-8]{1,9}$')
     -- df: chars='45678' length=5 lenvar=4 size=1000000
, y7 INTERVAL NOT NULL
     -- df: word=:1y,1mon,1day,1h,1m,1s
  -- *** COUNTER ***
, c0 INTEGER NOT NULL CHECK(c0 BETWEEN 3 AND 21)
  -- df: count start=3 step=2
, c1 TEXT NOT NULL CHECK(c1 ~ '^0{7}[1-9A]$') -- df: count format=08X
  -- *** FILE GENERATOR ***
, f0 BYTEA NOT NULL -- df: file=*.blob mode=blob
, f1 TEXT NOT NULL -- df: file=*.txt mode=text
  -- *** GEOMETRY ***
  -- no predefined generators... make do with what is available
, g0 POINT NOT NULL CHECK(g0 <@ BOX '((1,2)(2,3))')
     -- df: pattern='[:float alpha=1.0 beta=2.0:], [:float alpha=2.0 beta=3.0:]'
, g1 BOX NOT NULL CHECK(g1 <@ BOX '((0.0,0.0)(100.0,100.0))')
     -- df pt: float=uniform alpha=1.0 beta=99.0
     -- df ptx: use=pt
     -- df pty: use=pt
     -- df around: float=gauss alpha=0.0 beta=0.01
     -- df ptx_val: value=ptx
     -- df pty_val: value=pty
     -- df: pattern='([:value=ptx:],[:value=pty:])([:reduce=ptx_val,around:],[:reduce=pty_val,around:])'
  -- *** ARRAYS ***
  -- df smalls: int size=9
  -- df directions: word=:N,NE,E,SE,S,SW,W,NW
, a0 INT[] NOT NULL
, a1 INT ARRAY NOT NULL
, a2 TEXT ARRAY[2] NOT NULL
, a3 INT[3] NOT NULL CHECK(array_length(a3, 1) = 3) -- df: array=smalls length=3
, a4 TEXT[2] NOT NULL CHECK(array_length(a4, 1) = 2)
   -- df: array=directions length=2
   -- *** TUPLE ***
, t0 df.addr NOT NULL -- df: use=addr_tuple
);

CREATE TABLE df.Skip( -- df: skip=0.9 size=1000
  id SERIAL PRIMARY KEY
, data CHAR(3) NOT NULL CHECK(data = 'C&H') -- df: const='C&H'
);

-- one may set a directive after the table definition
-- df T=df.Stuff A=a1: size=3 offset=3

-- user-defined types
CREATE DOMAIN df.stuffit AS TEXT;
-- df: type=df.stuffit
-- df df.stuffit: pattern=Calvin|Hobbes|Susie|Moe|Rosalyn
CREATE DOMAIN df.bluff AS TEXT;
-- df: type=df.bluff

CREATE TABLE df.Pattern(
  id SERIAL PRIMARY KEY
, p0 TEXT NOT NULL CHECK(p0 = '') -- df: pattern=''
  --- df: this must be ignored
  ---- df: this must also be ignored
  -- -- df: this must be ignored as well
  -- ignore -- df: again, to be ignored
, p1 TEXT NOT NULL CHECK(p1 = 'hi') -- df: pattern='hi'
, p2 TEXT NOT NULL CHECK(p2 = 'hh') -- df: pattern='h{2}'
, p3 TEXT NOT NULL CHECK(p3 ~ '^[a-z]$') -- df: pattern='[a-z]'
, p4 TEXT NOT NULL CHECK(p4 ~ '^[0-9]{5}$') -- df: pattern='[0-9]{5}'
, p5 TEXT NOT NULL CHECK(p5 ~ '^[A-Z]{3,7}$') -- df: pattern='[A-Z]{3,7}'
, p6 TEXT NOT NULL CHECK(p6 = 'hello') -- df: pattern='(hello)'
, p7 TEXT NOT NULL CHECK(p7 ~ '^(he|ll|o!)$') -- df: pattern='(he|ll|o!)'
, p8 TEXT NOT NULL CHECK(p8 ~ '^(|x|yy){3}$') -- df: pattern='(|x|yy){3}'
, p9 TEXT NOT NULL CHECK(p9 = 'hello') -- df: pattern='hel{2}o'
, pa TEXT NOT NULL CHECK(pa ~ '^[ab][cd]$') -- df: pattern='[ab][cd]'
, pb TEXT NOT NULL CHECK(pb ~ '^[ab][cd]$') -- df: pattern='(a|b)(c|d)'
, pc TEXT NOT NULL CHECK(pc ~ '^a[bB][cC]D$') -- df: pattern='a[bB](c|C)D'
, pd TEXT NOT NULL CHECK(pd ~ '^a?[bB]?[cC]?D?$')
   -- df: pattern='a?[bB]?(c|C)?D?'
, pe TEXT NOT NULL CHECK(pe ~ '^(ac|ad|bc|bd){1,2}$')
   -- df: pattern='((a|b)(c|d)){1,2}'
, pf TEXT NOT NULL CHECK(pf = '') -- df: pattern='(){5,10}'
, pg TEXT NOT NULL CHECK(pg ~ '^(0[a-z]0|1[A-Z]1)$')
   -- df: pattern='0[a-z]0|1[A-Z]1'
, ph TEXT NOT NULL CHECK(ph = '[(|?{') -- df: pattern='\\[\\(\\|\\?\\{'
, pi TEXT NOT NULL CHECK(pi ~ '^0{0,8}1{1,8}2{0,1}$') -- df: pattern='0*1+2?'
, pj TEXT NOT NULL CHECK(pj ~ '^.{2,9}$') -- df: pattern='..+'
, pk TEXT NOT NULL CHECK(pk ~ '^A[0-9]{1,8}[0-9a-f][0-9A-F]!$')
   -- df: pattern='\101\d+\h\H\041'
, pl CHAR(9) NOT NULL CHECK(pl ~ '^[^0-9a-z]{9}$') -- df: pattern='[^0-9a-z]{9}'
-- pm, see below
, pn TEXT NOT NULL CHECK(pn ~ '^[0-9A-Fa-f][0-9][a-z][A-Z]$')
  -- df: pattern='[:xdigit:][:digit:][:lower:][:upper:]'
, po TEXT NOT NULL CHECK(po ~ '^[€œ௲ℝ가𝄞]{1,8}$')
  -- df: pattern='[\\u20ac\\u0153\\U00000bf2\\u211d\\uAC00\\U0001D11E]+'
, pq df.stuffit NOT NULL CHECK(pq ~ '^(Calvin|Hobbes|Susie|Moe|Rosalyn)$')
, pr df.bluff NOT NULL CHECK(pr ~ '^bluff_[_\d]+$') -- df: string prefix=bluff
);

ALTER TABLE df.Pattern
  ADD COLUMN pm TEXT NOT NULL
  CHECK(pm ~ '^[œéÊµ±Þøå€çýð«©ß漢字Яαあア]{1,8}$');
  -- df: pattern='[œéÊµ±Þøå€çýð«©ß漢字Яαあア]+'
"""

# some checks about the internal schema
# other checks are implicitely performed for contraints.
# some of these test occasionnaly fail with a low probability, eg s5
INTERNAL_CHECK = """
-- useful for additional checks
CREATE OR REPLACE FUNCTION df.assert(what TEXT, ok BOOLEAN) RETURNS BOOLEAN
IMMUTABLE CALLED ON NULL INPUT AS $$
BEGIN
  IF ok IS NULL OR NOT ok THEN
    RAISE EXCEPTION 'assert failed: %', what;
  END IF;
  RETURN ok;
END
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION df.assert_eq(what TEXT, v BIGINT, r BIGINT)
RETURNS BOOLEAN
IMMUTABLE CALLED ON NULL INPUT AS $$
BEGIN
  IF v IS NULL OR r IS NULL OR v <> r THEN
    RAISE EXCEPTION 'assert failed: % (% vs %)', what, v, r;
  END IF;
  RETURN TRUE;
END
$$ LANGUAGE plpgsql;

-- one if true else 0
CREATE FUNCTION df.oitez(ok BOOLEAN) RETURNS INTEGER
IMMUTABLE STRICT AS $$
  SELECT CASE WHEN ok THEN 1 ELSE 0 END;
$$ LANGUAGE SQL;

-- check value to a precision
CREATE FUNCTION df.value(d DOUBLE PRECISION,
                         val DOUBLE PRECISION, epsilon DOUBLE PRECISION)
RETURNS BOOLEAN
IMMUTABLE STRICT AS $$
  SELECT d BETWEEN val-epsilon AND val+epsilon;
$$ LANGUAGE SQL;

\\echo '# generator checks'
SELECT
  -- check ints
  df.assert('skewed power',
      df.value(AVG(df.oitez(i4=1)), 0.7, 0.1)) AS "i4"
, df.assert('skewed scale',
      df.value(AVG(df.oitez(i5=1)), 0.7, 0.1)) AS "i5"

  -- check bools
, df.assert('b0 rate',
      df.value(AVG(df.oitez(b0)), 0.5, 0.1) AND
      df.value(AVG(df.oitez(b0)), 0.5, 0.1)) AS "b0"
, df.assert('b1 rates',
      df.value(AVG(df.oitez(b1 IS NULL)), 0.5, 0.1) AND
      df.value(AVG(df.oitez(b1)), 0.5, 0.1)) AS "b1"
, df.assert('b2 0.7 rate',
      df.value(AVG(df.oitez(b2)), 0.7, 0.1)) AS "b2"

  -- check floats
, df.assert('uniform', -- 0.5 +- 0.29
      df.value(AVG(f0), 0.5, 0.05) AND df.value(STDDEV(f0), 0.289, 0.05))
      AS "f0"
, df.assert('gaussian',
      df.value(AVG(f1), 5.0, 0.5) AND df.value(STDDEV(f1), 2.0, 0.5))
      AS "f1"
, df.assert('uniform 2',
      df.value(AVG(f2), 0.0, 1.0) AND df.value(STDDEV(f2), 5.77, 0.5))
      AS "f2"
-- , df.assert('exponential', df.value(AVG(f4), 1.0/0.1, 1.0)) AS "f4"
, df.assert('normal',
      df.value(AVG(f7), 20.0, 0.1) AND df.value(STDDEV(f7), 0.5, 0.1)) AS "f7"

  -- check dates
, df.assert_eq('d4 days', COUNT(DISTINCT d4), 2) AS "d4"
, df.assert_eq('d5 days', COUNT(DISTINCT d5), 2) AS "d5"

  -- check timestamps
, df.assert_eq('t0 stamps', COUNT(DISTINCT t0), 2) AS "t0"
, df.assert_eq('t1 stamps', COUNT(DISTINCT t1), 2) AS "t1"
, df.assert_eq('t2 stamps', COUNT(DISTINCT t2), 30) AS "t2"

  -- check text
, df.assert_eq('s2 text 1', COUNT(DISTINCT s2), 20) AS "s2-1"
, df.assert_eq('s2 text 2', MAX(LENGTH(s2))-MIN(LENGTH(s2)), 2) AS "s2-2"
, df.assert_eq('s4 text', COUNT(DISTINCT s4), 20) AS "s4"
, df.assert_eq('s5 text 1', COUNT(DISTINCT s5), 50) AS "s5-1"
, df.assert('s5 text 2',
            df.value(AVG(LENGTH(REPLACE(s5,'b',''))), 30*0.7, 1.0)) AS "s5-2"
, df.assert_eq('s6 text', COUNT(DISTINCT s6), 3) AS "s6"
, df.assert_eq('s7 text', COUNT(DISTINCT s7), 3) AS "s7"
, df.assert_eq('s8 text', COUNT(DISTINCT s8), 16) AS "s8"
, df.assert_eq('n3 mac', COUNT(DISTINCT n3), 17) AS "n3"
  -- expected z1 length is 0.9 * 2 + 0.1 * 1 = 1.9
, df.assert('z1 list', AVG(LENGTH(z1)) BETWEEN 1.85 AND 1.95) AS "z1"

  -- check ELEVEN share
, df.assert_eq('h0', COUNT(DISTINCT h0), 11) AS "h0"
, df.assert_eq('h1', COUNT(DISTINCT h1), 11) AS "h1"
, df.assert_eq('h2', COUNT(DISTINCT h2), 11) AS "h2"
, df.assert_eq('h3', COUNT(DISTINCT h3), 11) AS "h3"
, df.assert_eq('h4', COUNT(DISTINCT h4), 11) AS "h4"
, df.assert_eq('h5', COUNT(DISTINCT h5), 11) AS "h5"
, df.assert_eq('h6', COUNT(DISTINCT h6), 11) AS "h6"
, df.assert_eq('h0-6 share',
            COUNT(DISTINCT (h0, h1, h2, h3, h4, h5, h6)), 11) AS "h0-6"
FROM df.stuff;

\\echo '# foreign key checks'
SELECT
  df.assert('fk2', df.value(AVG(df.oitez(fk2 IS NULL)), 0.5, 0.1)) AS "fk2"
, df.assert('fka', df.value(AVG(df.oitez(fka = 1000000)), 0.2, 0.05)) AS "fka"
FROM df.ForeignKeys;

\\echo '# miscellaneous checks'
SELECT
  df.assert_eq('ten', COUNT(*), 10) AS "ten"
, df.assert_eq('123', SUM(df.oitez(nogen=123)), 10) AS "123"
, df.assert_eq('null', SUM(df.oitez(n IS NULL)), 10) AS "null"
, df.assert_eq('c0', COUNT(DISTINCT c0), 10) AS "c0"
, df.assert_eq('c1', COUNT(DISTINCT c1), 10) AS "c1"
FROM df.Ten;

\\echo '# skip check'
SELECT
  df.assert('skip', COUNT(*) BETWEEN 50 AND 150) AS "skip"
FROM df.Skip;

DROP SCHEMA df CASCADE;
"""

COMICS = """
  -- Comics didactic example.

  -- Set default scale to 10 tuples for one unit (1.0).
  -- This can be overwritten with the size option.
  -- df: size=10

  -- This relation is not re-generated.
  -- However the size will be used when generating foreign keys to this table.
  CREATE TABLE Language( --df: nogen size=2
    lid SERIAL PRIMARY KEY,
    lang TEXT UNIQUE NOT NULL
  );

  INSERT INTO Language(lid, lang) VALUES
    (1, 'French'),
    (2, 'English')
  ;

  -- Define a char generator for names:
  -- df chfr: int sub=scale rate=0.17
  -- df name: chars='esaitnrulodcpmvqfbghjxyzwk' cgen=chfr length=8 lenvar=3

  CREATE TABLE Author( --df: mult=1.0
    aid SERIAL PRIMARY KEY,
    -- There are 200 firstnames do draw from, most frequent 2%.
    -- In the 19th century John & Mary were given to >5% of the US population,
    -- and rates reached 20% in England in 1800. Currently firstnames are much
    -- more diverse, most frequent at about 1%.
    firstname TEXT NOT NULL,
      -- df: use=name size=200 sub=scale rate=0.02
    -- There are 10000 lastnames to draw from, most frequent 8/1000 (eg Smith)
    lastname TEXT NOT NULL,
      -- df: use=name size=10000 sub=scale rate=0.008
    -- Choose dates in the 20th century
    birth DATE NOT NULL,     -- df: start=1901-01-01 end=2000-12-31
    -- We assume that no two authors of the same name are born on the same day
    UNIQUE(firstname, lastname, birth)
  );

  -- On average, about 10 authors per publisher (1.0/0.1)
  CREATE TABLE Publisher( -- df: mult=0.1
    pid SERIAL PRIMARY KEY,
    pname TEXT UNIQUE NOT NULL -- df: prefix=pub length=12 lenvar=4
  );

  -- On average, about 15.1 books per author (15.1/1.0)
  CREATE TABLE Comics( -- df: mult=15.1
    cid SERIAL PRIMARY KEY,
    title TEXT NOT NULL,     -- df: use=name length=20 lenvar=12
    published DATE NOT NULL, -- df: start=1945-01-01 end=2013-06-25
    -- The biased scale generator is set for 95% 1 and 5% for others.
    -- There are 2 language values because of size=2 on table Language.
    lid INTEGER NOT NULL REFERENCES Language, -- df: int sub=scale rate=0.95
    pid INTEGER NOT NULL REFERENCES Publisher,
    -- A publisher does not publish a title twice in a day
    UNIQUE(title, published, pid)
  );

  -- Most books are stored once in the inventory.
  -- About 1% of the books are not in the inventory (skip).
  -- Some may be there twice or more (15.2 > 15.1 on Comics).
  CREATE TABLE Inventory( -- df: mult=15.2 skip=0.01
    iid SERIAL PRIMARY KEY,
    cid INTEGER NOT NULL REFERENCES Comics, -- df: sub=serand
    shelf INTEGER NOT NULL -- df: size=20
  );

  -- on average, about 2.2 authors per comics (33.2/15.1)
  CREATE TABLE Written( -- df: mult=33.2
    -- serand => at least one per author and one per comics, then random
    cid INTEGER NOT NULL REFERENCES Comics, -- df: sub=serand mangle
    aid INTEGER NOT NULL REFERENCES Author, -- df: sub=serand mangle
    PRIMARY KEY(cid, aid)
  );
"""

LIBRARY = """
  -- df English: word=/etc/dictionaries-common/words
  CREATE TABLE Book( -- df: mult=100.0
    bid SERIAL PRIMARY KEY,
    title TEXT NOT NULL, -- df: text=English length=4 lenvar=3
    isbn ISBN13 NOT NULL -- df: size=1000000000
  );

  CREATE TABLE Reader( -- df: mult=1.0
    rid SERIAL PRIMARY KEY,
    firstname TEXT NOT NULL, -- df: prefix=fn size=1000 sub=power rate=0.03
    lastname TEXT NOT NULL, -- df: prefix=ln size=10000 sub=power rate=0.01
    born DATE NOT NULL, -- df: start=1923-01-01 end=2010-01-01
    gender BOOLEAN NOT NULL, -- df: rate=0.25
    phone TEXT -- df:  chars='0-9' null=0.01 size=1000000 length=10 lenvar=0
  );

  CREATE TABLE Borrow( -- df: mult=1.5
    borrowed TIMESTAMP NOT NULL, -- df: size=72000
    rid INTEGER NOT NULL REFERENCES Reader,
    bid INTEGER NOT NULL REFERENCES Book, -- df: mangle
    PRIMARY KEY(bid) -- a book is borrowed once at a time!
  );
"""

# LIBRARY example without directives
LIBRARY_NODIR = re.sub(r'-- df.*', '', LIBRARY)

# test name, start with '!' if expected to fail
# test option, '!' if histogram, '-' if short
UNITS = [
    #
    # working tests
    #
    ("count",
     ["-count",
      "-count format=02X",
      "-count start=17 format=2o",
      "-count start=9 step=-1"]),
    ("const",
     ["-const", "-const=", "-const=hi", "-const='ho' null=0.5"]),
    ("bool",
     ["-bool", "-type=BOOL",
      "-bool rate=1.0", "-bool rate=0.0", "-bool rate=0.7", "!bool rate=0.85"]),
    ("int",
     ["-int", "-type=INTEGER",
      "-int sub=scale rate=0.4", "-int sub=power rate=0.4",
      "-int sub=serial", "-int sub=serial size=5", "-int sub=serand size=5",
      "-int sub=serial xor=17 step=7 size=20",
      "!int sub=power rate=0.8 size=3", "!int sub=scale rate=0.7 size=4",
      "-int sub=uniform size=8", "-type=SERIAL", "-type=tinyint",
      "-int null=0", "-int null=1", "-int null=0.5"]),
    ("float", # avoid 'exp' (2.[67]) and 'vonmises' (3.[23])
     ["-float", "-type='DOUBLE PRECISION'",
      "-float=gauss alpha=0.8 beta=0.01",
      "-float=beta alpha=2.0", "-float=log alpha=5.0 beta=2.0"]),
    ("chars",
     ["-chars=abcde length=5 size=1", "-chars=abçde length=5 size=2",
      "-chars=0-9 length=5", "-chars=\dABC length=4 lenvar=3", "-chars=\w",
      r'-chars=\u211d\u20ac length=3 lenvar=2',
      "80pc:int sub=scale rate=0.8", "-chars='1.' length=10 cgen=80pc"]),
    ("string",
     ["-string", "-type=varchar(9)", "-type=CHAR(5) prefix=s",
      "-string size=5 prefix=i length=5 lenvar=1",
      "-string offset=10 size=10 lenmin=6 lenmax=7" ]),
    ("word",
     ["-word=:", "-word=:un,2,3,4,5 sub=serial", "-word=:a,b,c,d,e,f size=2",
      "-word=/etc/dictionaries-common/words",]),
    ("text",
     ["w1:string prefix=s size=10 length=5",
      "w2:word=:Gondor,Mordor,Moria,Rohan,Shire",
      "w3:int size=10 offset=0",
      "-text=w1 lenmin=1 lenmax=3 prefix=|",
      "-text=w2 length=2 lenvar=1 suffix=.",
      "-text=w3 lenmax=3 suffix=,", # again...
      "-text=w3 lenmax=3 suffix=, separator=+",
      # hmmm... shoud I create a share just with 'size=4'?
      # does a subgen imply a share?
      "four: int size=4",
      "-text=w2 lenmax=5 share=four suffix=."]),
    ("pattern",
     ["-pattern", "-pattern=",
      "-pattern=[a-z]+", "-pattern=\w*", "-pattern=abc", "-pattern=(hi|you)",
      "-pattern=\d{2}[:lower:]{2}", "-pattern=(0|1)+[a-f]?",
      "-pattern=N=[:count:]", "-pattern='N=[:count start=9 step=-1:]'",
      "-pattern='N=[:count:]?'", "-pattern='\([:float:]\)'" ]),
    ("blob",
     ["-blob", "-type=BLOB lenmin=3 lenmax=6",
      "-blob length=5 lenvar=2", "-blob length=6 lenvar=0",
      "-blob length=0 lenvar=0", "-blob length=3 size=3" ]),
    ("date",
     ["-date end=1970-03-20 size=3650",
      "-date start=1970-03-20 size=36500",
      "-date start=2038-01-19 end=2038-01-19",
      "-date start=2038-01-19 end=2038-01-21",
      "-date start=1970-01-18 end=1970-03-20",
      "-date start=2038-01-19 prec=3 size=3" ]),
    ("timestamp",
     ["-timestamp start='2013-12-26 17:31:05' size=7 prec=10",
      "-timestamp end='2013-12-26 17:32:01' prec=3600 size=3" ]),
    ("interval",
     ["-interval", "-interval unit=d size=365", "-interval unit=y size=42"]),
    ("inet",
     ["-inet", "-inet=", "-type=INET",
      "-inet=10.2.14.0", "-inet=10.2.14.16/30", "-inet=10.2.14.0/22",
      "-inet=;10.2.14.0/31", "-inet=.10.2.15.0/31", "-inet=,10.2.16.0/31",
      "-inet=;10.2.17.12/32", "-inet=fe80::/120",
      "-inet=fe80::/64", "-inet=fe80::/16", "-inet=fe80::/24"]),
    ("mac",
     ["-mac", "-type=MACADDR", "-mac size=10"]),
    ("luhn",
     ["-luhn", "-luhn length=12", "-luhn length=8", "-luhn prefix=4800"]),
    ("ean",
     ["-ean", "-type=EAN13", "-type=ISBN13", "-type=ISSN13", "-type=ISMN13",
      "-type=UPC", "-type=ISBN", "-type=issn", "-type=ISMN size=10",
      "-type=ISSN prefix=1234567","-type=ISMN prefix=M123",
      "-type=UPC prefix=1000000"]),
    ("uuid",
     ["-uuid", "-type=UUID", "-type=uuid"]),
    ("bit",
     ["-bit", "-bit type=BIT(3)", "-bit type=varbit(7)",
      "-bit length=7", "-bit lenmin=3 lenmax=5"]),
    ("array",
     ["ints: int size=9", "words: word=:one,two,three,four,five",
      "strings: string prefix=s size=9 length=3", "arrays: array=ints length=2",
      "-array=ints", "-array=words length=3", "-array=strings length=2",
      "-array=arrays length=3",
      "-type='INT[2]'", "-type='TEXT ARRAY'", "-type='INT ARRAY[2][3]'",
      "-type='INET[]'", "-type='FLOAT ARRAY [] [3]'"]),
    ("cat",
     ["f1: word=:un,deux,trois", "f2: int size=3",
      "-cat=f2,f1,f2"]),
    ("repeat",
     ["zero:chars='0' length=1", "-repeat=zero", "-repeat=zero extent=1-5",
      "-repeat=zero extent=1-5 sub=scale rate=0.85"]),
    ("isnull",
     ["-isnull", "-isnull null=1.0"]),
    ("tuple",
     ["digit: int size=10 offset=0", "none: isnull", "cst: const='hi' null=0.3",
      "stuff: pattern='[a-z]{2}'",
      "-tuple", "-tuple=", "-tuple=''", "-tuple null=0.3", # empty tuples
      "-tuple=digit", "-tuple=digit null=0.8", "-tuple=digit,cst,none,stuff",
      "-tuple=digit,isnull,digit,cst null=0.3",
      "THREE: int size=3", "nb: int size=1000 share=THREE",
      r"city: pattern='[A-Z][a-z]+(\?|\!)' share=THREE",
      "-tuple=nb,city" ]),
    ("reduce",
     ["f1: float=gauss beta=0.05", "f2: float", "i1: int size=2 offset=0",
      "-reduce=f1,f2", "-reduce=f1,i1", "-reduce=f1,i1 op=*",
      "-reduce=f1,f2 op=*", "-reduce=f1,f2 op=cat", "-reduce=f1,i1 op=max",
      "-reduce=f1,i1 op=min" ]),
    ("value",
     ["i1: int size=10 offset=0", "f1: float", "f1v: value=f1",
      "f2: float=gauss alpha=0.0 beta=0.01",
      "-pattern='[:value=i1:]=[:value=i1:]'",
      "-pattern='[:use=f1v:],[:reduce=f1v,f2:]'"]),
    #
    # error tests
    #
    ("! count 1", ["count step=0"]),
    ("! bool 1", ["bool rate=2.0"]),
    ("! int 0", ["-int size=0"]),
    ("! int 1", ["int sub=scale rate=0.1 alpha=2.0"]),
    ("! int 2", ["int sub=foo"]),
    ("! int 3", ["int alpha=1.0"]),
    ("! int 4", ["int prefix=12 length=7 suffix=0"]),
    ("! int 5", ["int="]),
    ("! int 6", ["int=2"]),
    ("! int 7", ["int size=17.3"]),
    ("! int 8", ["int size=three"]),
    ("! int 9", ["int mult=2.0 size=20"]),
    ("! int A", ["int null=2.0"]),
    ("! float 1", ["float=exp beta=2.0"]),
    ("! chars 1", ["chars"]), # empty set
    ("! chars 2", ["chars="]), # idem
    ("! chars 3", ["chars=a-"]),
    ("! chars 4", ["chars=a-z\\"]),
    ("! string 1", ["string lenmin=-3 lenmax=1"]),
    ("! string 2", ["string lenmin=3 lenmax=2"]),
    ("! string 3", ["string length=10 lenmin=3"]),
    ("! word 1", ["-word="]),
    #("! word 2", ["-word=:one,two,three size=5"]),
    ("! text 1", ["text"]),
    ("! text 2", ["text="]),
    ("! text 3", ["text=undefined"]),
    ("! text 4", ["useless: size=3", "text=useless"]), # should it be int?
    ("! pattern 1", ["pattern=[:unknown:]"]),
    ("! date 1", ["date="]),
    ("! date 2", ["date=True"]),
    ("! date 3", ["date start=2038-01-19 end=2038-01-18" ]), # empty set
    ("! inet 1", ["inet=10"]),
    ("! inet 2", ["inet=10.0.0.0/33"]),
    ("! inet 3", ["inet=10.2.14.0/31"]), # no available address
    ("! inet 4", ["inet=10.2.14.1/32"]), # idem
    ("! luhn 1", ["luhn length=1"]),
    ("! luhn 2", ["luhn length=4 prefix=4800"]),
    ("! luhn 3", ["luhn prefix=000B"]),
    ("! ean 1", ["type=ISSN prefix=12345678"]),
    ("! isnull 1", ["isnull null=0.5"]),
    ("! reduce 1", ["f1: float", "f2: float", "reduce=f1,f2 op=sum"]),
    ("! * 1", [ "int float" ]),
    ("! * 2", [ "type=flt" ]),
    ("! * 3", [ "int: chars=0-9"]),
]

class StdoutExitError(BaseException):
    """Exception class used for unit tests."""
    def __init__(self, msg=''):
        print('## ' + msg)
        if not opts.debug:
            sys.exit(1)

def run_unit_tests(seed=None):
    fail = 0
    op = []
    if opts.self_test_hack:
        op.append("--self-test-hack")
    for v, lt in UNITS:
        ok = v[0] != '!'
        print('')
        print("** {0} **".format(v))
        sys.stdout.flush()
        status = self_run(validate=lt, seed=seed, op=op).wait()
        if ok != (status == 0):
            print("** FAILED **")
            fail += 1
    if fail:
        sys.stderr.write("unit tests failed: {0}\n".format(fail))
    sys.exit(fail)

# escaped list of caracters for . in regular expressions
RE_DOT = r' -~' # from ASCII 0x20 to 0x7E
RE_POSIX_CC = { \
    'alpha':'A-Za-z', 'alnum':'A-Za-z0-9', 'ascii':' -~', 'blank':r' \t',
    'cntrl':'\000-\037\177', 'digit':'0-9', 'graph':'!-~', 'lower':'a-z',
    'print':' -~', 'punct':' -/:-@[-^`{-~', 'space':r'\s', 'upper':'A-Z',
    'word':r'\w', 'xdigit':'0-9a-fA-F'
}

# re helpers: alas this is not a parser.

# identifier
RE_IDENT=r'"[^"]+"|`[^`]+`|[a-z0-9_]+'
# possibly schema-qualified
# ??? this won't work with quoted identifiers?
# 1=schema, 2=table
RE_IDENT2=r'({0})\.({0})|{0}'.format(RE_IDENT)

# SQL commands
RE_CMD=r'CREATE|ALTER|DROP|SELECT|INSERT|UPDATE|DELETE|SET|GRANT|REVOKE|SHOW'

# types
RE_SER=r'(SMALL|BIG)?SERIAL|SERIAL[248]'
RE_BLO=r'BYTEA|BLOB'
RE_INT=r'{0}|(TINY|SMALL|MEDIUM)INT|INT[248]|INTEGER|INT\b'.format(RE_SER)
RE_FLT=r'REAL|FLOAT|DOUBLE\s+PRECISION|NUMERIC|DECIMAL'
RE_TXT=r'TEXT|CHAR\(\d+\)|VARCHAR\(\d+\)'
RE_BIT=r'BIT\(\d+\)|VARBIT\(\d+\)'
RE_TSTZ=r'TIMESTAMP(\s+WITH\s+TIME\s+ZONE)?'
RE_INTV=r'INTERVAL'
RE_TIM=r'DATE|{0}|{1}'.format(RE_TSTZ, RE_INTV)
RE_BOO=r'BOOL(EAN)?'
RE_IPN=r'INET|CIDR'
RE_MAC=r'MACADDR'
RE_EAN=r'EAN13|IS[BMS]N(13)?|UPC'
RE_GEO=r'POINT|LINE|LSEG|BOX|PATH|POLYGON|CIRCLE'
RE_OTH=r'UUID'

# all predefined PostgreSQL types
RE_TYPE='|'.join([RE_INT, RE_FLT, RE_TXT, RE_BIT, RE_TIM, RE_BOO, RE_BLO,
                  RE_IPN, RE_MAC, RE_EAN, RE_GEO, RE_OTH])

RE_ARRAY=r'((\s+ARRAY)?(\s*\[[][0-9 ]*\])?)?'

# and arrays thereof
RE_ALLT = r'({0}){1}'.format(RE_TYPE, RE_ARRAY)

# SQL syntax
new_object = re.compile(r"^\s*({0})\s".format(RE_CMD), re.I)
# 1=table name, [2, 3]
create_table = \
    re.compile(r'^\s*CREATE\s+TABLE\s*({0})\s*\('.format(RE_IDENT2), re.I)
# 1=type name, [2, 3]
create_enum = \
    re.compile(r'^\s*CREATE\s+TYPE\s+({0})\s+AS\s+ENUM'.format(RE_IDENT2), re.I)
# 1=type
create_type = \
    re.compile(r'\s*CREATE\s+TYPE\s+({0})\s+AS'.format(RE_IDENT2), re.I)

# 1=?, 2=column, 3=type
r_column = r'^\s*,?\s*(ADD\s+COLUMN\s+)?({0})\s+({1})'.format(RE_IDENT, RE_ALLT)
column = re.compile(r_column, re.I)
is_int = re.compile(r'^({0})$'.format(RE_INT), re.I)
is_ser = re.compile(r'^({0})$'.format(RE_SER), re.I)
# enums are added when definitions are encountered
column_enum = None

s_reference = \
    r'.*\sREFERENCES\s+({0})\s*(\(({1})\))?'.format(RE_IDENT2, RE_IDENT)
# 1=table, [2, 3], 4=?, 5=dest columns (?)
reference = re.compile(s_reference, re.I)
primary_key = re.compile('.*\sPRIMARY\s+KEY', re.I)
unique = re.compile(r'.*\sUNIQUE', re.I)
not_null = re.compile(r'.*\sNOT\s+NULL', re.I)
s_unicity = r'.*(UNIQUE|PRIMARY\s+KEY)\s*\(([^\)]+)\)'
# 1=unicity type, 2=columns
unicity = re.compile(s_unicity, re.I)
# 1=?, 2=table name, [3, 4]
alter_table = \
    re.compile(r'\s*ALTER\s+TABLE\s+(ONLY\s+)?({0})'.format(RE_IDENT2), re.I)
add_constraint = r',?\s*ADD\s+CONSTRAINT\s+({0})\s+'.format(RE_IDENT)
# 1=constraint name, 2=unicity type, 3=columns
add_unique = re.compile(add_constraint + s_unicity, re.I)
# 1=constraint_name, 2=source columns, 3=table, [4, 5], 6=, 7=dest columns
add_fk = \
    re.compile(add_constraint + r'FOREIGN\s+KEY\s*\(([^\)]+)\)' + s_reference,
               re.I)
# 1=column
alter_column = re.compile(r',?\s*ALTER\s+COLUMN\s+({0})'.format(RE_IDENT), re.I)

# DETECT DATAFILLER DIRECTIVES
# commented-out directives, say '--- df...' or '---- df...' or '-- -- df...'
df_junk = re.compile('.*?--.*-\s*df.*')
# simple directive: 1=contents
df_dir = re.compile(r'.*--\s*df[^:]*:\s*(.*)')
# macro definition: 1=name, 2=contents
df_mac = re.compile(r'.*--\s*df\s+([\w\.]+)\s*:\s*(.*)')
# explicit table: 2=name
df_tab = \
    re.compile(r'.*--\s*df[^:]*\s+(t|table)=({0})(\s|:)'. \
               format(RE_IDENT2), re.I)
# explicit attribute: 2=name
df_att = \
    re.compile(r'.*--\s*df[^:]*\s+(a|att|attribute)=({0})(\s|:)'. \
               format(RE_IDENT), re.I)
# string/float/int directives: 1=name 2=value 3=reminder
df_txt = re.compile(r'(\w+)=\'([^\']*)\'\s+(.*)')
df_flt = re.compile(r'(\w+)=(-?\d+\.\d*)\s+(.*)')
df_int = re.compile(r'(\w+)=(-?\d+)\s+(.*)')
df_str = re.compile(r'(\w+)=(\S*)\s+(.*)')
# simple directive: 1=name 2=reminder
df_bol = re.compile(r'(\w+)\s+(.*)')

# remove SQL comments & \xxx commands
comments = re.compile(r'(.*?)\s*--.*')
backslash = re.compile(r'\s*\\')

import random

# some ASCII control characters
UCHARS = { '0':'\0', 'a':'\a', 'b':'\b', 'f':'\f', \
           'n':'\n', 'r':'\r', 't':'\t', 'v':'\v' }
# re special escaped characters, which may appear within [] or out of them
RECHARS = { 'd':'0-9', 's':' \t\r\n\v\f', 'w':'a-zA-Z0-9_', \
            'h':'0-9a-f', 'H':'0-9A-F' }
def unescape(s, regexp=False):
    """Return an unescaped string."""
    r, i = u8(''), 0
    # hmmm... not very efficient, and probably buggy
    while i < len(s):
        c = s[i]
        if c == '\\':
            assert i+1 < len(s), \
                "escaped string '{0}' must not end with '\\'".format(s)
            if re.match(r'[012][0-7]{2}', s[i+1:]): # octal
                r += chr(int(s[i+1:i+4], 8))
                i += 4
            elif re.match(r'[xX][0-9a-fA-F]{2}', s[i+1:]): # hexadecimal
                r += chr(int(s[i+2:i+4], 16))
                i += 4
            elif re.match(r'u[0-9a-fA-F]{4}', s[i+1:]): # 4 digit unicode
                r += unichr(int(s[i+2:i+6], 16))
                i += 6
            elif re.match(r'U[0-9a-fA-F]{8}', s[i+1:]): # 8 digit unicode
                r += unichr(int(s[i+2:i+10], 16))
                i += 10
            else: # other escaped characters
                c = s[i+1]
                r += u8(RECHARS[c]) if regexp and c in RECHARS else \
                     u8(UCHARS[c]) if c in UCHARS else \
                     c
                i += 2
        else:
            r += c
            i += 1
    return r

#
# DATA GENERATORS, with some inheritance
#

# global tuple counter
tuple_count = 0
generator_count = 0

class Generator(object):
    """Generator is the base class for generating valid data,
    typically for a given attribute.

    Generators are driven by parameters (directives).
    They may depend on a random generator.
    The framework also allows to generate NULL values.

    - {} params: parameters
    - Attribute att: attribute (may be None in some cases)
    - str type: type to consider, possibly from att
    - float nullp: NULL value rate
    - int gens: call count
    - int size: number of underlying values (may be None)
    """
    # expected/allowed directives
    DIRS = {'null':float, 'type':str }
    def __init__(self, att, params=None):
        global generator_count
        generator_count += 1
        self.att = att
        self.params = {}
        self.params.update(params if params != None else \
                           att.params if att != None else \
                           {})
        # show generators
        if opts.debug:
            debug(1, "{0}: {1} {2}".format(att, generator_count,
                                           self.__class__))
            debug(2, "params: {0}".format(strDict(self.params)))
        self.type = self.params['type'].lower() if 'type' in self.params else \
                    att.type if att else \
                    None
        # null probability
        self.nullp = 0.0
        if self.att == None: # for tests
            self.nullp = \
                self.params.get('null', opts.null if opts.null else 0.0)
        elif self.att.isNullable():
            self.nullp = self.params['null'] if 'null' in self.params else \
                self.att.table.params['null'] \
                    if 'null' in self.att.table.params else \
                opts.null
        assert 0.0 <= self.nullp and self.nullp <= 1.0, \
            "{0}: 'null' not in [0,1]".format(self)
        # underlying call counter and overall size, common to all generators
        self.gens, self.size = 0, None
        self.cleanParams(Generator.DIRS)
    def cleanParams(self, dirs):
        """Remove parameters once processed."""
        for d in dirs:
            if d in self.params:
                del self.params[d]
    def mapGen(self, func):
        """Apply func on all generators."""
        func(self)
    def shareSeed(self):
        """Reseed when under 'share'"""
        pass
    def notNull(self):
        self.nullp = 0.0
    def setShare(self, shared):
        self.shared = shared
    def __str__(self):
        if self.att:
            return "{0} {1}".format(self.__class__.__name__, self.att)
        else:
            return self.__class__.__name__
    def genData(self):
        """Generate actual data."""
        return None

class NULLGenerator(Generator):
    """Generate a NULL value."""
    DIRS = {}
    def __init__(self, att=None, params=None):
        # check 'null'
        self.att = att
        p = params if params else att.params if att else {}
        if 'null' in p:
            assert p['null'] == 1.0, \
                "{0}: 'null' is {1} instead of 1.0".format(self, p['null'])
        Generator.__init__(self, att, params)
        self.cleanParams(NULLGenerator.DIRS)
    def genData(self):
        return None
    def getData(self):
        return None

class WithLength(Generator):
    """Set {min,max}len attributes."""
    # this is needed by String, Text & Blob...
    DIRS = { 'lenmin':int, 'lenmax':int, 'length':int, 'lenvar':int }
    def __init__(self, lenmin=None, lenmax=None):
        self.lenmin, self.lenmax = lenmin, lenmax
        mm = 'lenmin' in self.params or 'lenmax' in self.params
        lv = 'length' in self.params or 'lenvar' in self.params
        assert not (mm and lv), \
            "{0}: not both 'length'/'lenvar' & 'lenmin'/'lenmax'".format(self)
        if self.type != None and not mm and not 'length' in self.params:
            # try length/var based on type
            clen = re.match(r'(var)?(char|bit)\((\d+)\)', self.type)
            self.lenmin, self.lenmax = None, None
            if clen:
                self.lenmax = int(clen.group(3))
                if re.match('var(char|bit)', self.type):
                    if 'lenvar' in self.params:
                        self.lenmin = self.lenmax - 2 * self.params['lenvar']
                        del self.params['lenvar']
                    else:
                        self.lenmin = int(self.lenmax * 3 / 4)
                else: # char(X)
                    assert self.params.get('lenvar', 0) == 0, \
                        "{0}: non zero 'lenvar' on CHARS(*)".format(self)
                    self.lenmin = self.lenmax
        else:
            self.lenmin, self.lenmax = None, None
        if 'lenmax' in self.params:
            self.lenmax = self.params['lenmax']
        if 'lenmin' in self.params:
            self.lenmin = self.params['lenmin']
        if 'length' in self.params or 'lenvar' in self.params:
            length = self.params.get('length', int((lenmin+lenmax)/2))
            lenvar = self.params.get('lenvar', 0)
            self.lenmin, self.lenmax = length-lenvar, length+lenvar
        if self.lenmin != None and self.lenmax == None:
            self.lenmax = int(self.lenmin * 4 / 3)
        elif self.lenmax != None and self.lenmin == None:
            self.lenmin = int(self.lenmax * 3 / 4)
        elif self.lenmin == None and self.lenmax == None:
            self.lenmin, self.lenmax = lenmin, lenmax
        assert 0 <= self.lenmin and self.lenmin <= self.lenmax, \
            "{0}: inconsistent length [{1},{2}]". \
            format(self, self.lenmin, self.lenmax)
        self.cleanParams(WithLength.DIRS)

class RandomGenerator(Generator):
    """Generate something based on random, possibly seeded.

    - Random _random: internal random number generator, possibly seeded
    """
    DIRS = { 'share':str, 'seed':str }
    def __init__(self, att, params=None):
        Generator.__init__(self, att, params)
        self._rand = random.Random()
        # set generator seed, depending on the determinism requirements
        # self.seed = <gc>_(<share>_)?<seed/opts.seed/random>_
        self.seed = str(generator_count) + '_'
        if 'share' in self.params:
            self.shared = \
                SharedGenerator.getGenerator(self, self.params['share'])
            # what about self.shared.seed?
            self.seed += self.params['share'] + '_'
        else:
            self.shared = None
        # complete seeding
        if 'seed' in self.params:
            self.seed += self.params['seed'] + '_'
        elif opts.seed:
            self.seed += opts.seed + '_'
        #elif self.att:
        #    self.seed = self.att.table.name + '_' + self.att.name + '_'
        else:
            # is it necessary/useful? depending on python version?
            self.seed += str(random.random()) + '_'
        if not self.shared:
            self._rand.seed(self.seed)
        # else: it will be reseeded for each tuple in shareSeed, when called
        self.cleanParams(RandomGenerator.DIRS)
    def shareSeed(self):
        if self.shared:
            self._rand.seed(self.seed + str(self.shared.genData()))
        super(RandomGenerator, self).shareSeed() # pass
    def setShare(self, shared):
        self.shared = shared
        super(RandomGenerator, self).setShare(shared) # pass
    def getData(self):
        """Generate a NULL or some data with genData()."""
        # only call the random generated if really needed
        return None if self.nullp == 1.0 else \
            self.genData() if self.nullp == 0.0 else \
            None if self._rand.random() < self.nullp else \
            self.genData()

# yep, RandomGenerator because of 'null'
class ConstGenerator(RandomGenerator):
    """Generate a constant possibly escaped string.

    - str cst: constant string to return.
    """
    DIRS = { 'const':str }
    def __init__(self, att=None, params=None, cst=None, escape=True):
        RandomGenerator.__init__(self, att, params)
        if cst == None:
            cst = self.params['const'] if 'const' in self.params else ''
        if isinstance(cst, bool): # fix if empty directive
            cst = ''
        self.cst = unescape(cst) if escape and cst.find('\\') != -1 else cst
        self.cleanParams(ConstGenerator.DIRS)
    def genData(self):
        return self.cst

class CountGenerator(RandomGenerator):
    """Generate a count."""
    DIRS = { 'start':str, 'step':int, 'format':str }
    def __init__(self, att, params=None):
        RandomGenerator.__init__(self, att, params)
        self.start = int(self.params.get('start', 1))
        self.step = int(self.params.get('step', 1))
        self.format = u8("{{0:{0}}}").format(self.params.get('format', 'd'))
        assert self.step != 0, "{0}: 'step' must not be zero".format(self)
        self.cleanParams(CountGenerator.DIRS)
    def genData(self):
        self.gens += 1
        return self.format.format((self.gens - 1) * self.step + self.start)

class BoolGenerator(RandomGenerator):
    """Generate True/False at a given 'rate'.

    - float rate: truth's rate, defaults to 0.5
    """
    DIRS = { 'rate':float }
    def __init__(self, att, params=None):
        RandomGenerator.__init__(self, att, params)
        self.rate = self.params.get('rate', 0.5)
        assert 0.0 <= self.rate and self.rate <= 1.0, \
            "{0}: rate {1} not in [0,1]".format(self, self.rate)
        self.cleanParams(BoolGenerator.DIRS)
    def genData(self):
        return False if self.rate == 0.0 else \
               True  if self.rate == 1.0 else \
               self._rand.random() < self.rate

class FloatGenerator(RandomGenerator):
    """Generate floats with various distributions.

    The generation is driven by 'type' and parameters 'alpha' and 'beta'.
    - str sub: subtype of random generator
    - float alpha, beta: parameters
    """
    DIRS = { 'float':str, 'alpha':float, 'beta':float }
    def __init__(self, att, params=None):
        RandomGenerator.__init__(self, att, params)
        self.sub = self.params.get('float', 'uniform')
        if self.sub == True or self.sub == '':
            self.sub = 'uniform'
        self.alpha = self.params.get('alpha', 0.0)
        self.beta = self.params.get('beta', 1.0)
        r, a, b, s = self._rand, self.alpha, self.beta, self.sub
        # check consistency
        if self.sub in [ 'exp', 'pareto' ]:
            assert not 'beta' in self.params, \
                "{0}: unexpected 'beta' for float generator '{1}'".\
                format(self, s)
        # genData() is overwritten depending on the generator subtype
        self.genData = \
            (lambda: r.gauss(a, b))           if s == 'gauss'    else \
            (lambda: r.betavariate(a, b))     if s == 'beta'     else \
            (lambda: r.expovariate(a))        if s == 'exp'      else \
            (lambda: r.gammavariate(a, b))    if s == 'gamma'    else \
            (lambda: r.lognormvariate(a, b))  if s == 'log'      else \
            (lambda: r.normalvariate(a, b))   if s == 'norm'     else \
            (lambda: r.paretovariate(a))      if s == 'pareto'   else \
            (lambda: r.uniform(a, b))         if s == 'uniform'  else \
            (lambda: r.vonmisesvariate(a, b)) if s == 'vonmises' else \
            (lambda: r.weibullvariate(a, b))  if s == 'weibull'  else \
            None
        assert self.genData, \
            "{0}: unexpected float generator '{1}'".format(self, s)
        self.cleanParams(FloatGenerator.DIRS)

from fractions import gcd
import math

class IntGenerator(RandomGenerator):
    """Generate integers, possibly mangled and offset.

    - int offset: generate between offset and offset+size-1
    - int shift, step, xor: mangling parameters
      return offset + (shift + step * (i "^" xor)) % size
    - str sub: serial, serand, uniform, power & scale
      . serial: a counter
      . serand: serial up to size, then random
    - uniform, power and scale are random generators
    - power & scale use either parameter 'alpha' or 'rate' to define
      their skewness.
    """
    # 60 handy primes for step mangling, about every 10,000,000
    PRIMES = [ 1234567129, 1244567413, 1254567911, 1264567631, 1274567381,
               1284567247, 1294567787, 1304567897, 1314568139, 1324568251,
               1334568007, 1344567943, 1354567987, 1364568089, 1374568339,
               1384568699, 1394567981, 1404568153, 1414568359, 1424568473,
               1434567973, 1444568269, 1454567999, 1464568463, 1474568531,
               1484568011, 1494568219, 1504568887, 1514568533, 1524567899,
               1534568531, 1544568271, 1554568441, 1564568519, 1574568419,
               1584567949, 1594568149, 1604568283, 1614568231, 1624568417,
               1634568427, 1644568397, 1654568557, 1664568677, 1674568109,
               1684568321, 1694568241, 1704567959, 1714568899, 1724568239,
               1734567899, 1744567901, 1754567891, 1764567913, 1774567901,
               1784567899, 1794567911, 1804567907, 1814567891, 1824567893 ]
    DIRS = { 'sub':str, 'mangle':bool,
             'size':int, 'offset':int, 'step':int, 'shift':int, 'xor':int,
             'alpha':float, 'rate':float }
    def __init__(self, att, params=None):
        RandomGenerator.__init__(self, att, params)
        # set generator subtype depending on attribute
        self.sub = self.params.get('sub')
        if not self.sub:
            self.sub = 'serial' if att != None and att.isUnique() else 'uniform'
        # {'x','y'} does not work with 2.6
        assert self.sub in ['serial', 'serand', 'uniform', 'power', 'scale'], \
            "{0}: invalid int generator '{1}'".format(self, self.sub)
        # set offset from different sources
        # first check for explicit directives
        if 'offset' in self.params:
            self.offset = self.params['offset']
        # then PK or FK information
        elif att != None and att.isPK and opts.offset:
            self.offset = opts.offset
        elif att != None and att.FK:
            fk = att.FK.getPK()
            self.offset = \
                fk.params.get('offset', opts.offset if opts.offset else 1)
        else:
            self.offset = 1
        # scale & power
        if 'alpha' in self.params or 'rate' in self.params:
            assert self.sub in ['power', 'scale'], \
                "{0}: unexpected 'alpha'/'beta' for int generator '{1}'". \
                format(self, self.sub)
        assert not ('alpha' in self.params and 'rate' in self.params), \
            "{0}: not both 'alpha' and 'rate' for '{1}'".format(self, self.sub)
        if 'alpha' in self.params:
            self.alpha, self.rate = float(self.params['alpha']), None
        elif 'rate' in self.params:
            self.alpha, self.rate = None, float(self.params['rate'])
        else:
            self.alpha, self.rate = None, None
        # set step, shift & xor...
        self.shift, self.xor, self.step = 0, 0, 1
        self.mangle = self.params.get('mangle', False)
        self.step = self.params['step'] if 'step' in self.params else \
            IntGenerator.PRIMES[random.randrange(0, \
                                len(IntGenerator.PRIMES))] if self.mangle else \
            1
        assert self.step != 0, "{0}: 'step' must not be zero".format(self)
        self.shift = self.params['shift'] if 'shift' in self.params else None
        self.xor = self.params['xor'] if 'xor' in self.params else None
        self.mask = 0
        # set size if explicit, other will have to be set later.
        if 'size' in self.params:
            self.setSize(self.params['size'])
        elif att != None and att.size != None:
            self.setSize(att.size) # possibly computed from table & mult
        else: # later? when??
            self.size = None
        self.cleanParams(IntGenerator.DIRS)
    def setSize(self, size):
        assert (isinstance(size, int) or isinstance(size, long)) and size > 0, \
            "{0}: 'size' {1} must be > 0".format(self, size)
        self.size = size
        # shortcut if nothing to generate...
        if size <= 1:
            self.shift = 0
            return
        # adjust step, xor, shift depending on size
        if self.step != 1 and gcd(size, self.step) != 1:
            # very unlikely for big primes steps
            sys.stderr.write("{0}: step {1} ignored for size {2}\n".
                             format(self.att, self.step, size))
            self.step = 1
        if self.xor == None:
            self.xor = random.randrange(1, 1000*size) if self.mangle else 0
        if self.shift == None:
            self.shift = random.randrange(0, size) if self.mangle else 0
        if self.xor != 0:
            # note: int.bit_length available from 2.7 & 3.1
            m = 1
            while m <= self.size:
                m *= 2
            self.mask = int(m/2) # ???
        # get generator parameters, which may depend on size
        if self.sub == 'power' or self.sub == 'scale':
            if self.rate != None:
                assert 0.0 < self.rate and self.rate < 1.0, \
                    "{0}: rate {1} not in (0,1)".format(self, self.rate)
                if self.sub == 'power':
                    self.alpha = - math.log(size) / math.log(self.rate)
                else: # self.sub == 'scale':
                    self.alpha = self.rate * (size - 1.0) / (1.0 - self.rate)
            elif self.alpha == None:
                self.alpha = 1.0
            assert self.alpha > 0, \
                "{0}: 'alpha' {1:f} not > 0".format(self, self.alpha)
        else: # should not get there
            assert self.alpha == None, \
                "{0}: useless 'alpha' {1} set".format(self, self.alpha)
    def genData(self):
        assert self.size != None and self.size > 0, \
            "{0}: cannot draw from empty set".format(self)
        # update counter
        self.gens += 1
        # set base in 0..size-1 depending on generator type
        if self.size == 1:
            return self.offset
        assert self.shift != None, "{0}: shift is set".format(self)
        if self.sub == 'serial' or \
           self.sub == 'serand' and self.gens - 1 < self.size:
            base = (self.gens - 1) % self.size
        elif self.sub == 'uniform' or self.sub == 'serand':
            base = int(self._rand.randrange(0, self.size))
        elif self.sub == 'power':
            base = int(self.size * self._rand.random() ** self.alpha)
        else: # self.sub == 'scale':
            v = self._rand.random()
            base = int(self.size * (v / ((1 - self.alpha ) * v + self.alpha)))
        assert 0 <= base and base < self.size, \
            "{0}: base {1:d} not in [0,{2:d})".format(self, base, self.size)
        # return possibly mangled result
        if self.xor != 0:
            # non linear step: apply xor to the largest possible power of 2
            m = self.mask
            while m > 0:
                if m & self.size != 0 and m & base == 0:
                    base = ((base ^ self.xor)  & (m - 1)) | (base & ~ (m - 1))
                    break
                m = int(m/2)
        # then linear step:
        return self.offset + (self.shift + self.step * base) % self.size

# ??? This could also be based on FloatGenerator? '4.2 days' is okay for pg.
class IntervalGenerator(IntGenerator):
    """Generate intervals.

    - str unit: time unit for the interval, default is 's' (seconds)
    """
    DIRS = { 'unit':str }
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        self.unit = self.params.get('unit', 's')
        self.cleanParams(IntervalGenerator.DIRS)
    def genData(self):
        # ??? maybe it should not depend on db?
        return db.intervalValue(super(IntervalGenerator, self).genData(),
                                self.unit)

from datetime import date, timedelta

class DateGenerator(IntGenerator):
    """Generate dates between 'start' and 'end' at precision 'prec'.

    - Date ref: reference date
    - int dir: direction from reference date
    - int prec: precision in days
    """
    DIRS = { 'start':str, 'end':str, 'prec':int }
    @staticmethod
    def parse(s):
        return datetime.date(datetime.strptime(s, "%Y-%m-%d"))
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        self.offset = 0
        start, end = 'start' in self.params, 'end' in self.params
        ref = self.params['start'] if start else \
              self.params['end'] if end else \
              None
        if ref != None:
            self.ref = DateGenerator.parse(ref)
            self.dir = 2 * ('start' in self.params) - 1
        else:
            self.ref = date.today()
            self.dir = -1
        # precision, defaults to 1 day
        self.prec = self.params.get('prec', 1)
        assert self.prec > 0, \
            "{0}: 'prec' {1} not > 0".format(self, self.prec)
        # adjust size of both start & end are specified
        if start and end:
            dend = DateGenerator.parse(self.params['end'])
            assert self.ref <= dend, \
                "{0}: 'end' must be after 'start'".format(self)
            delta = (dend - self.ref) / self.prec
            self.setSize(delta.days + 1)
        self.cleanParams(DateGenerator.DIRS)
    def genData(self):
        d = self.ref + self.dir * \
            timedelta(days=self.prec * IntGenerator.genData(self))
        return db.dateValue(d)

from datetime import datetime

class TimestampGenerator(IntGenerator):
    """Generate timestamps between 'start' and 'end'.

    - Timestamp ref: reference time
    - int dir: direction from reference time, -1 or 1
    - int prec: precision in seconds, default 60 seconds
    - str tz: set in this time zone, may be None
    """
    DIRS = { 'tz':str }
    DIRS.update(DateGenerator.DIRS)
    @staticmethod
    def parse(s):
        return datetime.strptime(s, "%Y-%m-%d %H:%M:%S")
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        self.offset = 0
        self.tz = self.params.get('tz')
        start, end = 'start' in self.params, 'end' in self.params
        ref = self.params['start'] if start else \
              self.params['end'] if end else \
              None
        if ref != None:
            self.ref = TimestampGenerator.parse(ref)
            self.dir = 2 * ('start' in self.params) - 1
        else:
            self.ref = datetime.today()
            self.dir = -1
        # precision, defaults to 60 seconds
        self.prec = self.params.get('prec', 60)
        # set size
        if start and end:
            dend = TimestampGenerator.parse(self.params['end'])
            assert self.ref <= dend, \
                "{0}: 'end' must be after 'start'".format(self)
            d = (dend - self.ref) / self.prec
            # t = d.total_seconds() # requires 2.7
            t = (d.microseconds + (d.seconds + d.days * 86400) * 10**6) / 10**6
            self.setSize(int(t + 1))
        self.cleanParams(TimestampGenerator.DIRS)
    def genData(self):
        t = self.ref + self.dir * \
            timedelta(seconds=self.prec * \
                      super(TimestampGenerator, self).genData())
        # ??? should not depend on db
        return db.timestampValue(t, self.tz)

class WithSubgen(Generator):
    """Set a sub-generator from a macro or parameters."""
    DIRS = {}
    def __init__(self, name=None, att=None, p={}, g=None, backup=False):
        if g:
            self.subgen = g
        else:
            assert (p and name in p) or backup, \
                "{0}: mandatory '{1}' directive".format(self, name)
            if p and name in p:
                self.subgen = macroGenerator(p[name], att)
                del p[name]
            else:
                self.subgen = IntGenerator(att)
        if self.shared and self.subgen:
            self.subgen.shared = self.shared
        self.cleanParams(WithSubgen.DIRS)
    def mapGen(self, func):
        super(WithSubgen, self).mapGen(func)
        if self.subgen: # may be None for empty arrays
            self.subgen.mapGen(func)

class StringGenerator(IntGenerator, WithLength):
    """Generate a basic string, like "stuff_12_12_12"

    - str prefix: use attribute name by default
    - int lenmin, lenmax: expected length
    """
    DIRS = { 'prefix':str } # | WithLength.DIRS
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        WithLength.__init__(self, 8, 16)
        # set defaults from attributes
        self.prefix = self.params.get('prefix', att.name if att else 'str')
        if self.size == None:
            self.setSize(opts.size if opts.size else 1000) # ???
        self.cleanParams(StringGenerator.DIRS)
    def lenData(self, length, n):
        """Generate a data for int n of length"""
        sn = '_' + str(n)
        s = self.prefix + sn * int(2 + (length - len(self.prefix)) / len(sn))
        return s[:int(length)]
    def baseData(self, n):
        # data dependent length so as to be deterministic
        s = self.lenData(self.lenmax, n)
        # ! hash(s) is not deterministic from python 3.3
        hs = sum(ord(s[i])*(997*i+1) for i in range(len(s)))
        length = self.lenmin + hs % (self.lenmax - self.lenmin + 1)
        return s[:int(length)]
    def genData(self):
        return self.baseData(super(StringGenerator, self).genData())

# two generators are needed, one for the chars & one for the words
# the parameterized inherited generator is used for the words
# BUG: unique is not checked nor structurally inforced
class CharsGenerator(WithSubgen, StringGenerator):
    """Generate a string based on a character list.

    The generated string is deterministic in length +- lenvar.

    - str chars: list of characters to draw from
    - IntGenerator cgen: generator for choosing characters"""
    DIRS = { 'chars':str, 'cgen':str }
    @staticmethod
    def parseCharSequence(obj, s):
        """Generate list of characters to choose from."""
        if isinstance(s, bool):
            s = ''
        c = unescape(s, regexp=True)
        # ((<stuff>)?X-Y|-)*<stuff>
        chars = ''
        d = c.find('-')
        while d != -1:
            if d == 0:
                chars += '-' # leading dash means itself
                c = c[1:]
            else:
                assert d < len(c)-1, \
                    "{0}: '{1}' cannot end with dash".format(obj, s)
                if d > 1: # extract leading <stuff>
                    chars += c[0:d-1]
                chars += \
                    ''.join(chr(x) for x in range(ord(c[d-1]), ord(c[d+1])+1))
                c = c[d+2:]
            d = c.find('-')
        # append remaining stuff
        chars += c
        return chars
    def __init__(self, att=None, params=None, chars=None):
        StringGenerator.__init__(self, att, params)
        if att != None and att.isUnique():
            raise Exception("chars generator does not support UNIQUE")
        self.chars = \
            CharsGenerator.parseCharSequence(self, self.params['chars']) \
                if chars == None else chars
        assert len(self.chars) > 0, \
            "{0}: no characters to draw from".format(self)
        WithSubgen.__init__(self, 'cgen', None, self.params, backup=True)
        self.subgen.setSize(len(self.chars)) # number of chars
        self.subgen.offset = 0
        self.cleanParams(CharsGenerator.DIRS)
    def lenData(self, length, n):
        # ??? be deterministic in n and depend on seed option
        self.subgen._rand.seed(self.seed + str(n))
        s = ''.join(self.chars[self.subgen.genData()] for i in range(length))
        return s

class SeedGenerator(IntGenerator):
    DIRS = {}
    def __init__(self, att, params):
        IntGenerator.__init__(self, att, params)
        self._rand2 = random.Random()
        self.cleanParams(SeedGenerator.DIRS)
    def reseed(self):
        self._rand2.seed(self.seed + str(super(SeedGenerator, self).genData()))
    def genData(self):
        raise Exception("{0}: cannot call genData() on this class")

class LuhnGenerator(SeedGenerator):
    """Generate stuff with a Luhn checksum.

    - int length: (default 16 for bank card numbers)
    - str prefix: optional prefix, defaults to empty
    - ckSum: function to compute check digit
    """
    DIRS = { 'length':int, 'prefix':str }
    def __init__(self, att, params=None):
        SeedGenerator.__init__(self, att, params)
        self.length = self.params.get('length', 16)
        assert self.length >= 2, \
            "{0}: 'length' {1} must be > 1".format(self, self.length)
        if self.size == None:
            self.setSize(opts.size if opts.size else 100)
        self.prefix = str(self.params.get('prefix', ''))
        assert len(self.prefix) < self.length, \
            "{0}: 'prefix' \"{1}\" length not smaller than 'length' {2}".\
            format(self, self.prefix, self.length)
        if self.__class__ == LuhnGenerator:
            assert re.match(r'\d*$', self.prefix), \
                "{0}: 'prefix' {1} not decimal".format(self, self.prefix)
        self.ckSum = self.luhnDigit
        self.cleanParams(LuhnGenerator.DIRS)
    def luhnDigit(self, s):
        """Luhn's algorithm, see http://en.wikipedia.org/wiki/Luhn_algorithm."""
        assert len(s) == self.length - 1
        t = s[-2::-2] + ''.join(str(2 * int(c)) for c in s[-1::-2])
        return str(9 * sum(int(i) for i in t) % 10)
    def genData(self):
        self.reseed()
        code = self.prefix + \
               ''.join(str(self._rand2.randint(0, 9)) \
                       for i in range(self.length - len(self.prefix) - 1))
        return code + self.ckSum(code)

# there are some isbn/ean management modules, but they must be installed
# they would be only useful for computing the checksum digit, no big deal
class EANGenerator(LuhnGenerator):
    """Generate random EAN identifiers.

    EAN = International Article Number (!)
    """
    DIRS = {}
    def __init__(self, att=None, params=None):
        LuhnGenerator.__init__(self, att, params)
        if not self.type:
            self.type = 'ean13'
        # override length, prefix, cksum
        self.length = db.eanType(self.type)
        # set prefix for IS*N embedded in EAN13. ISBN13 could also use 979.
        if not self.prefix:
            self.prefix = '977' if self.type == 'issn13' else \
                          '978' if self.type == 'isbn13' else \
                          '9790' if self.type == 'ismn13' else \
                          'M' if self.type == 'ismn' else \
                          ''
        assert re.match(r'M?\d*', self.prefix), \
            "{0}: invalid 'prefix' {0}".format(self, self.prefix)
        assert len(self.prefix) < self.length, \
            "{0}: 'prefix' \"{1}\" length must be smaller than 'length' {2}".\
                format(self, self.prefix, self.length)
        self.ckSum = self.ckSumISN \
                         if self.type == 'isbn' or self.type == 'issn' else \
                     self.ckSumEAN # *13 & UPC & ISMN
        self.cleanParams(EANGenerator.DIRS)
    # checksum methods
    def ckSumEAN(self, s):
        assert len(s) == self.length - 1
        t, w = 0, 3
        for i in range(1, self.length):
            # special 'M' hack for ISMN numbers
            t += w * int(3 if s[-i] == 'M' else s[-i])
            w = 4 - w # 3 -> 1 -> 3 ...
        return str(-t % 10)
    def ckSumISN(self, s):
        assert len(s) == self.length - 1
        t = sum(int(s[-i]) * (i + 1) for i in range(1, self.length)) % 11
        return '0' if t == 0 else 'X' if t == 1 else str(11 - t)

class WordGenerator(IntGenerator):
    """Generate words, from a list of file

    - str[] words: list of words to draw from
    """
    # list from http://en.wikipedia.org/wiki/List_of_Hobbits
    HOBBITS = u8('Adaldrida Adamanta Adalgrim Adelard Amaranth Andwise ' +
                 'Angelica Asphodel Balbo Bandobras Belba Bell Belladonna ' +
                 'Berylla Bilbo Bill Bingo Bodo Bowman Bucca Bungo Camellia ' +
                 'Carl Celandine Chica Daddy Daisy Déagol Diamond Dinodas ' +
                 'Doderic Dodinas Donnamira Dora Drogo Dudo Eglantine Elanor ' +
                 'Elfstan Esmeralda Estella Everard Falco Faramir Farmer ' +
                 'Fastolph Fastred Ferdibrand Ferdinand Ferumbras').split(' ')
    DIRS = { 'word':str }
    def __init__(self, att, params=None, words=None):
        # keep explicit size for later
        self.__size = params['size'] if params and 'size' in params else \
            att.params['size'] if att and 'size' in att.params else \
            None
        # NOT att.size: this is computed and does not supersede len(words)
        self.words = None # temporary
        IntGenerator.__init__(self, att, params)
        assert not (words and 'word' in self.params), \
            "internal constructor issue, two word list specification!"
        if words:
            self.words = words
        else:
            spec = self.params['word']
            assert len(spec) > 0, "{0}: empty word specification".format(self)
            if spec[0] == ':':
                self.words = spec[1:].split(',')
            elif opts.self_test_hack:
                # self tests cannot depend from an external file
                #print("-- use Hobbit list for testing...")
                self.words = WordGenerator.HOBBITS
            else:
                # load word list from file
                f = open(spec)
                assert f, "{0}: cannnot open '{1}'".format(self, spec)
                self.words = [u(l.rstrip()) for l in f]
                f.close()
        # TODO: should check that UNIQUE is ok?
        # overwrite default size from IntGenerator
        self.setSize(self.__size if self.__size else len(self.words))
        assert self.size <= len(self.words), \
            "{0}: 'size' {1} >  number of words {2}". \
                format(self, self.size, len(self.words))
        self.cleanParams(WordGenerator.DIRS)
    def setSize(self, size):
        if self.words == None:
            #  hack: too early!
            return
        IntGenerator.setSize(self, size)
        # ??? fix anyway...
        if self.size > len(self.words):
            self.size = len(self.words)
        # do not access the list out of bounds
        if self.offset + self.size > len(self.words):
            self.offset = self.size - len(self.words)
    def genData(self):
        return self.words[super(WordGenerator, self).genData()]

class ArrayGenerator(WithSubgen, WithLength, IntGenerator):
    """Generate list from another generator."""
    DIRS = { 'array':str }
    def __init__(self, att=None, params=None, dir='array',
                 lenmin=5, lenmax=25, gen=None):
        IntGenerator.__init__(self, att, params)
        if gen:
            assert not dir in self.params # constructor consistency
            self.subgen = gen
        else:
            if dir == 'array' and not dir in self.params:
                # empty array!
                self.subgen = None
            else:
                WithSubgen.__init__(self, dir, att, self.params)
        WithLength.__init__(self, lenmin, lenmax)
        # set repetition
        if self.subgen:
            self.setSize(self.lenmax - self.lenmin + 1)
            self.offset = self.lenmin
        else: # empty array
            self.offset, self.size = 0, 1
        self.cleanParams(ArrayGenerator.DIRS)
    def genData(self):
        return [self.subgen.genData()
                for i in range(super(ArrayGenerator ,self).genData())]

# similar to RepeatGenerator, but with a different syntax
class TextGenerator(ArrayGenerator):
    """Generate text from another generator."""
    DIRS = { 'text':str, 'separator':str, 'prefix':str, 'suffix':str }
    def __init__(self, att, params=None):
        ArrayGenerator.__init__(self, att, params, 'text')
        self.sep = self.params.get('separator', ' ')
        self.prefix = self.params.get('prefix', '')
        self.suffix = self.params.get('suffix', '')
        self.cleanParams(TextGenerator.DIRS)
    def genData(self):
        l = super(TextGenerator, self).genData()
        return self.prefix + self.sep.join(str(i) for i in l) + self.suffix

class BlobGenerator(WithLength, SeedGenerator):
    """Generate binary large object."""
    DIRS = {}
    def __init__(self, att, params=None):
        SeedGenerator.__init__(self, att, params)
        WithLength.__init__(self, 8, 16)
        self.cleanParams(BlobGenerator.DIRS)
    def genData(self):
        self.reseed()
        len = self._rand2.randint(self.lenmin, self.lenmax)
        return bytes([self._rand2.randrange(256) for o in range(len)])

#
# generate some files for file generator tests
#
some_tmp_files_to_unlink = []

def generate_some_tmp_files():
    # under self-test, generates a bunch of small temporary files
    # also do that for validation
    global some_tmp_files_to_unlink
    import tempfile as tf
    for w in WordGenerator.HOBBITS:
        f = tf.NamedTemporaryFile(delete=False)
        f.write(w.encode('utf-8') * int(3))
        f.write('\n'.encode('utf-8'))
        some_tmp_files_to_unlink.append(f.name)
        f.close()

def cleanup_some_tmp_files():
    import os
    global some_tmp_files_to_unlink
    for f in some_tmp_files_to_unlink:
        os.unlink(f)
    some_tmp_files_to_unlink = []

class FileGenerator(IntGenerator):
    """Generate contents from files.

    - str[] files: list of files
    """
    DIRS = { 'file':str, 'mode':str }
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        # get list of files
        import os, glob
        self.files = []
        assert 'file' in self.params, "{0}: no directive 'file'".format(self)
        if some_tmp_files_to_unlink:
            # ignore files
            self.files = some_tmp_files_to_unlink
        else:
            # normal operation, process directive files
            for f in self.params['file'].split(os.pathsep):
                self.files += glob.glob(f)
        # handle conversion
        mode = self.params.get('mode', 'blob')
        assert mode == 'blob' or mode == 'text', \
            "{0}: mode must be 'blob' or 'text'".format(self)
        self.conv = bytes if mode == 'blob' else \
                    lambda s: s.decode(opts.encoding)
        # set size & offset
        assert len(self.files) > 0, "{0}: non empty set of files".format(self)
        if self.size == None:
            self.setSize(len(self.files))
        if self.size > len(self.files):
            self.size = len(self.files)
        self.offset = 0
        self.cleanParams(FileGenerator.DIRS)
    def genData(self):
        f = open(self.files[super(FileGenerator, self).genData()], "rb")
        s = self.conv(f.read())
        f.close()
        return s

class InetGenerator(IntGenerator):
    """Generate Internet addresses.

    - str network: network in which to choose ips
    - int net, mask: ip and mask from 'network'
    - tonet: int to address conversion function
    """
    DIRS = { 'inet':str }
    def __init__(self, att, params=None):
        IntGenerator.__init__(self, att, params)
        # get network target and mask
        self.network = self.params.get('inet', ';0.0.0.0/0')
        if len(self.network):
            beg, end = self.network[0] in ',;', self.network[0] in '.;'
        else:
            beg, end, self.network = True, True, ';0.0.0.0/0'
        if beg or end:
            self.network = self.network[1:]
        if self.network.find(':') != -1:
            # IPv6: rely on netaddr, will fail if not available
            from netaddr import IPNetwork, IPAddress
            self.tonet = IPAddress
            if self.network.find('/') == -1:
                self.network += '/64'
            try:
                ip = IPNetwork(self.network)
            except:
                assert False, "{0}: invalid ipv6 {1}".format(self, self.network)
            self.net = ip.first
            self.mask = ip.hostmask.value
        else:
            # IPv4: local implementation, ipaddress is not always available
            self.tonet = lambda i: \
                '.'.join(str(int((i & (255 << n*8)) / (1 << n*8))) \
                         for n in range(3,-1,-1))
            if self.network.find('/') == -1:
                self.network += '/24'
            ipv4, m = re.match(r'(.*)/(.*)', self.network).group(1, 2)
            assert re.match(r'(\d+\.){3}\d+$', ipv4), \
                "{0}: invalid ipv4 {1}".format(self, self.network)
            try:
                n, m = 0, int(m)
                assert 0 <= m and m <= 32, \
                    "{0}: ipv4 mask not in 0..32".format(self, m)
                # extract network number
                for i in map(int, re.split('\.', ipv4)):
                    assert 0 <= i and i <= 255, \
                        "{0}: ipv4 byte {1} not in 0..255".format(self, i)
                    n = n * 256 + i
            except:
                assert False, "{0}: invalid ipv4 {1}".format(self, self.network)
            assert 0 <= n and n <= 0xffffffff, \
                "{0}: ipv4 address {1} not on 4 bytes".format(self, n)
            self.mask = (1 << (32-m)) - 1
            self.net = n & (0xffffffff ^ self.mask)
        # override size & offset if was not set explicitely
        size = self.mask - 1 + int(beg) + int(end)
        if self.size == None:
            assert size > 0, \
                "{0}: empty address range {1}".format(self, self.network)
            self.setSize(size)
        else:
            # ??? fix default size in some cases...
            if self.size > size:
                self.setSize(size)
        if not 'offset' in self.params:
            self.offset = int(not beg)
        self.cleanParams(InetGenerator.DIRS)
    def genData(self):
        return self.tonet(super(InetGenerator, self).genData() + self.net)

# maybe it should rely on IntGenerator?
class MACGenerator(SeedGenerator):
    """Generate MAC Addresses."""
    DIRS = {}
    def __init__(self, att, params=None):
        SeedGenerator.__init__(self, att, params)
        self.cleanParams(MACGenerator.DIRS)
    def genData(self):
        self.reseed()
        return ':'.join(format(self._rand2.randrange(256), '02X') \
                        for i in range(6))

#
# special aggregation generators...
#

class ListGenerator(RandomGenerator):
    """Generate from a list of generators.

    - Generator[] gens
    """
    DIRS = {}
    def __init__(self, att=None, params=None, name=None, gens=None):
        RandomGenerator.__init__(self, att, params)
        if gens:
            assert not params
            assert not name in self.params
            self.gens = gens
        else:
            assert name in self.params, \
                "{0}: mandatory '{1}' directive".format(self, name)
            l = self.params[name]
            if l:
                self.gens = [macroGenerator(n) if n in df_macro else \
                             createGenerator(None, n, {}) \
                             for n in l.split(',')]
            else:
                self.gens = []
            #self.gens = list(map(macroGenerator, self.params[name].split(',')))
        self.cleanParams(ListGenerator.DIRS)
    def mapGen(self, func):
        super(ListGenerator, self).mapGen(func)
        list(map(lambda g: g.mapGen(func), self.gens))
    def genData(self):
        return [c.getData() for c in self.gens]

class CatGenerator(ListGenerator):
    """Generate by concatenating texts from other generators. """
    DIRS = { 'cat':str }
    def __init__(self, att=None, params=None, gens=None):
        ListGenerator.__init__(self, att, params, 'cat', gens)
        self.cleanParams(CatGenerator.DIRS)
    def genData(self):
        return ''.join(str(c) for c in super(CatGenerator, self).genData())

class TupleGenerator(ListGenerator):
    """Generate a tuple"""
    DIRS = { 'tuple':str }
    def __init__(self, att=None, params=None):
        ListGenerator.__init__(self, att, params, 'tuple')
        self.cleanParams(TupleGenerator.DIRS)
    def genData(self):
        return tuple(super(TupleGenerator, self).genData())

class ReduceGenerator(ListGenerator):
    """Reduce a list of generators"""
    DIRS= { 'reduce':str, 'op':str }
    OPS = { '+':(lambda a, b: float(a) + float(b)),
            '*':(lambda a, b: float(a) * float(b)),
            'max':(lambda a, b: a if a > b else b),
            'min':(lambda a, b: a if a < b else b),
            # this is really akin to CatGenerator
            'cat':(lambda a, b: str(a) + str(b)) }
    def __init__(self, att=None, params=None):
        ListGenerator.__init__(self, att, params, 'reduce')
        op = self.params.get('op', '+')
        assert op in ReduceGenerator.OPS, \
            "{0}: unexpected operation '{1}', expecting {2}". \
            format(self, op, sorted(ReduceGenerator.OPS))
        self.op = ReduceGenerator.OPS[op]
        self.cleanParams(ReduceGenerator.DIRS)
    def genData(self):
        return reduce(self.op, super(ReduceGenerator, self).genData())

class AltGenerator(RandomGenerator):
    """Generate from one in a weighted set.

    - Generator[] alts: generators to choose from
    - int[] weight: their respective weights
    - int total_weight: sum of weights
    """
    DIRS = { 'alt':str }
    def __init__(self, att=None, params=None, gens=None):
        RandomGenerator.__init__(self, att, params)
        if gens != None:
            assert not params
            self.alts = gens
            self.total_weight = len(gens)
            self.weight = [1 for i in range(self.total_weight) ]
        else:
            assert 'alt' in self.params, \
                "{0}: mandatory 'alt' directive".format(self)
            self.alts = []
            self.weight = []
            self.total_weight = 0
            # parse weighted macro alt: 'macro1:3,macro2,macro3:3'
            # it could be nice to order them by decreasing weight
            for mw in self.params['alt'].split(','):
                if ':' in mw:
                    m, w = mw.split(':')
                    w = int(w)
                else:
                    m, w = mw, 1
                assert w > 0, "{0}: weight {1} must be > 0".format(self, w)
                gen = macroGenerator(m)
                self.alts.append(gen)
                self.weight.append(w)
                self.total_weight += w
                if isinstance(gen, IntGenerator) and gen.size == None: # ???
                    gen.setSize(opts.size)
        self.cleanParams(AltGenerator.DIRS)
    def mapGen(self, func):
        super(AltGenerator, self).mapGen(func)
        list(map(lambda g: g.mapGen(func), self.alts))
    def genData(self):
        # weighted choice
        index, weight = 0, self._rand.randrange(self.total_weight)
        while weight >= self.weight[index]:
            weight -= self.weight[index]
            index += 1
        return self.alts[index].genData()

class RepeatGenerator(ArrayGenerator):
    """Generate repeated stuff."""
    DIRS = { 'repeat':str, 'extent':str }
    def __init__(self, att, params=None, gen=None):
        p = params if params else att.params
        self.att = None
        if 'extent' in p:
            assert \
                not list(filter(lambda d: d in p, WithLength.DIRS.keys())), \
                "{0}: both 'extent' and 'len*' directives".format(self)
            extent = p['extent']
            assert re.match(r'\d+(-\d+)?$', extent), \
                "{0}: bad 'extent' {1}".format(self, extent)
            if extent.find('-') != -1:
                min, max = extent.split('-')
            else:
                min, max = extent, extent
            min, max = int(min), int(max)
            assert 0 <= min and  min <= max, \
                "{0}: extent must be 0 <= {1} <= {2}".format(self, min, max)
        else:
            min, max = 1, 1
        ArrayGenerator.__init__(self, att, params, 'repeat', min, max, gen)
        # ??? hmmmm... overwrite extent from type (CHAR(36))
        # WithLength logic shoud distinguish defauts and sets...
        self.setSize(max - min + 1)
        self.offset = min
        self.cleanParams(RepeatGenerator.DIRS)
    def genData(self):
        return ''.join(super(RepeatGenerator, self).genData())

#
# PATTERN PARSING UTILS
#
# Hmmm... This is pretty awful code. Not sure how to improve it.
class Pattern:
    """Pattern parsing utils."""
    @staticmethod
    def skipRepeat(pattern, i=0):
        """Return next character after repetition specification."""
        if i < len(pattern):
            if pattern[i] == '{':
                i = pattern.find('}', i + 1) + 1
                assert i != 0, "{0}: closing '}' not found".format(pattern)
                # assert re.match('\{\d+(,\d+)?}', ...)
            elif pattern[i] in '?+*':
                i += 1
        return i
    @staticmethod
    def catSplit(pattern):
        """Split pattern for cat.

        return [ [ pattern, extent ], ... ]
        """
        if len(pattern) == 0:
            return []
        if pattern[0] == '(':
            # find matching nested ')'
            opened, i = 1, 1
            while opened > 0:
                assert i < len(pattern), "{0}: no matching ')'".format(pattern)
                if pattern[i] == ')':
                    opened -= 1
                elif pattern[i] == '(':
                    opened += 1
                elif pattern[i] == '\\': # escaped character
                    i += 1
                elif pattern[i] == '[': # skip embedded [...]
                    i = pattern.index(']', i + (3 if pattern[i+1]=='^' else 2))
                i += 1
            # find whether it is repeated
            end = Pattern.skipRepeat(pattern, i)
            return [ [ pattern[0:i], pattern[i:end] ] ] + \
                Pattern.catSplit(pattern[end:])
        elif pattern[0] == '[':
            # find matching ']', skip 1/2 whatever, may be ']' or '^]'
            i = pattern.index(']', 3 if pattern[1] == '^' else 2) + 1
            end = Pattern.skipRepeat(pattern, i)
            return [ [ pattern[0:i], pattern[i:end] ] ] + \
                Pattern.catSplit(pattern[end:])
        elif pattern[0] == '.':
            end = Pattern.skipRepeat(pattern, 1)
            return [ [ '.', pattern[1:end] ] ] + \
                Pattern.catSplit(pattern[end:])
        else:
            # eat possibly escaped characters, handling \[dswhH] on the fly
            i = 0
            while i < len(pattern):
                if pattern[i] == '\\' and pattern[i+1] in RECHARS:
                    end = Pattern.skipRepeat(pattern, i + 2)
                    if i == 0: # \?{10}
                        return [ [ '[' + RECHARS[pattern[1]] + ']' ,
                                   pattern[2:end] ] ] + \
                                 Pattern.catSplit(pattern[end:])
                    else: # STUF\?{10}
                        return [ [ pattern[0:i], '' ] , \
                                 [ '[' + RECHARS[pattern[i+1]] + ']' ,
                                   pattern[i+2:end] ] ] + \
                            Pattern.catSplit(pattern[end:])
                elif pattern[i] == '\\':
                    i += 1
                    assert i < len(pattern), \
                        "{0}: cannot end on escape".format(pattern)
                elif pattern[i] in '{?+*':
                    assert i != 0, \
                        "{0}: cannot start with repeat".format(pattern)
                    if i == 1:
                        # F{3}
                        end = Pattern.skipRepeat(pattern, i)
                        return [ [ pattern[0:i], pattern[i:end] ] ] + \
                            Pattern.catSplit(pattern[end:])
                    else:
                        # STUFF{3} -> STUF & F{3}
                        end = Pattern.skipRepeat(pattern, i)
                        return [ [ pattern[0:i-1], '' ] , \
                                 [ pattern[i-1:i], pattern[i:end] ] ] + \
                            Pattern.catSplit(pattern[end:])
                elif pattern[i] in '([':
                    return [ [ pattern[0:i], '' ] ] + \
                        Pattern.catSplit(pattern[i:])
                i += 1
            return [ [ pattern, '' ] ]
    @staticmethod
    def altSplit(pattern):
        """Split pattern for Alt

        returns [patterns]
        """
        assert pattern[0] == '(' and pattern[-1] == ')', \
            "{0} is not an alternation".format(pattern)
        pattern = pattern[1:-1] # remove parentheses
        l, i, opened = [], 0, 0
        while i < len(pattern):
            if pattern[i] == '\\': # skip escaped character
                i += 1
            elif pattern[i] == '[': # skip embedded []
                i = pattern.index(']', i + (3 if pattern[i+1] == '^' else 2))
            elif pattern[i] == '(':
                opened += 1
            elif pattern[i] == ')':
                opened -= 1
            elif pattern[i] == '|' and opened == 0:
                l.append(pattern[0:i])
                pattern = pattern[i+1:]
                i = -1
            i += 1
        # remaining stuff
        l.append(pattern)
        return l
    @staticmethod
    def repeat(att, g, extent):
        if extent == '' or extent == '{1}'or extent == '{1,1}':
            return g
        extent = '0,1' if extent == '?' else \
                 '0,8' if extent == '*' else \
                 '1,8' if extent == '+' else \
                 extent[1:-1]
        if ',' in extent:
            min, max = extent.split(',')
        else:
            min, max = extent, extent
        return RepeatGenerator(att, params={'extent': min + '-' + max}, gen=g)
    @staticmethod
    def genAlt(att, pattern, extent=''):
        """Parses a single pattern like (...) or [...] or ."""
        if len(pattern) == 0:
            return ConstGenerator(att)
        # handle '.' shortcut and other special (character) classes
        if pattern == '.':
            pattern = '[' + RE_DOT + ']'
        elif pattern[:2] == '[:' and pattern[-2:] == ':]':
            desc = pattern[2:-2]
            if desc in RE_POSIX_CC:
                pattern = '[' + RE_POSIX_CC[desc] + ']'
            else:
                # allow something like [:count start=10 format=08X step=2:]
                g = createGenerator(att, None, getParams(desc), msg=pattern)
                return Pattern.repeat(att, g, extent)
        if pattern[0] == '(':
            # alternative, AltGenerator
            l = Pattern.altSplit(pattern)
            assert len(l) > 0
            if len(l) == 1:
                g = Pattern.genCat(att, l[0])
            else:
                g = AltGenerator(att, gens=[Pattern.genCat(att, p) for p in l])
            return Pattern.repeat(att, g, extent)
        elif pattern[0] == '[':
            # character class, CharGenerator...
            if pattern[1] == '^':
                assert len(pattern)>3 and pattern[-1] == ']'
                notch = CharsGenerator.parseCharSequence(att, pattern[2:-1])
                allch = CharsGenerator.parseCharSequence(att, RE_DOT)
                difch = ''.join(sorted(list(set(allch) - set(notch))))
                g = CharsGenerator(att, chars=difch)
            else:
                assert len(pattern)>2 and pattern[-1] == ']'
                g = CharsGenerator(att, params={'chars':pattern[1:-1]})
            g.size, g.lenmin, g.lenmax = 0xffffffff, 1, 1
            return Pattern.repeat(att, g, extent)
        else:
            # possibly repeated constant text
            return Pattern.repeat(att, ConstGenerator(att, cst=pattern), extent)
    @staticmethod
    def genCat(att, pattern, extent=''):
        """Parses a pattern sequence like (foo|bla){1,3}stuff[abc]{5}'"""
        if len(pattern) == 0:
            # assert extent == '' # what is the point of: (){3}
            return ConstGenerator(att)
        pats = Pattern.catSplit(pattern)
        if len(pats) == 0:
            assert extent == '', \
                "{0}: extent {1} on nothing".format(pattern, extent)
            return ConstGenerator(att)
        elif len(pats) == 1:
            g = Pattern.genAlt(att, pats[0][0], pats[0][1])
        else:
            g = CatGenerator(att, gens=[Pattern.genAlt(att, p[0], p[1]) \
                                        for p in pats])
        return Pattern.repeat(att, g, extent)

class PatternGenerator(RandomGenerator):
    """Generate pattern-based text.

    - Generator root
    """
    DIRS = { 'pattern':str }
    def __init__(self, att, params=None):
        RandomGenerator.__init__(self, att, params)
        assert 'pattern' in self.params, \
            "{0}: mandatory 'pattern' directive".format(self)
        self.pattern = '' if isinstance(self.params['pattern'], bool) else \
                       self.params['pattern']
        self.root = Pattern.genAlt(att, '(' + self.pattern + ')')
        # synchronize full generator tree
        self.root.mapGen(lambda s: s.notNull())
        if self.shared:
            self.root.mapGen(lambda s: s.setShare(self.shared))
        self.cleanParams(PatternGenerator.DIRS)
    def mapGen(self, func):
        super(PatternGenerator, self).mapGen(func)
        self.root.mapGen(func)
    def genData(self):
        return self.root.genData()

class WithPersistent(Generator):
    """Persistent value per tuple."""
    def __init__(self):
        self.last_tuple_count = 0
        self.last_value = None
    def super(self):
        return super(WithPersistent, self)
    def genData(self):
        if self.last_tuple_count != tuple_count:
            self.last_tuple_count = tuple_count
            self.last_value = self.super().genData()
        return self.last_value

class PersistentGenerator(WithPersistent):
    """Encapsulate a generator to make it persistent."""
    def __init__(self, gen):
        WithPersistent.__init__(self)
        self.gen = gen
    def super(self): # hmmm...
        return self.gen
    def mapGen(self, func):
        super(PersistentGenerator, self).mapGen(func)
        self.gen.mapGen(func)

class ValueGenerator(Generator):
    """Generate a constant value per tuple"""
    values = {}
    DIRS = { 'value':str }
    def __init__(self, att, params):
        Generator.__init__(self, att, params)
        assert 'value' in self.params, \
            "{0}: mandatory 'value' directive".format(self)
        value = self.params['value']
        if not value in ValueGenerator.values:
            ValueGenerator.values[value] = \
                PersistentGenerator(macroGenerator(value))
        self.value = ValueGenerator.values[value]
        self.cleanParams(ValueGenerator.DIRS)
    def mapGen(self, func):
        super(ValueGenerator, self).mapGen(func)
        self.value.mapGen(func)
    def genData(self):
        return self.value.genData()
    def getData(self):
        return self.genData()

class SharedGenerator(WithPersistent, IntGenerator):
    """Generate persistent integers."""
    _generators = {}
    DIRS = {}
    @staticmethod
    def getGenerator(obj, name):
        if not name in SharedGenerator._generators:
            assert name in df_macro, \
                "{0}: 'share' {1} directive must be a macro".format(obj, name)
            SharedGenerator._generators[name] = \
                SharedGenerator(name, df_macro[name])
        return SharedGenerator._generators[name]
    def __init__(self, name, params):
        assert name != None and params != None, "mandatory parameters"
        size = params['size'] if 'size' in params else None
        mult = params['mult'] if 'mult' in params else None
        IntGenerator.__init__(self, None, params)
        self.nullp = 0.0
        # set size, possibly overriding super constructor
        if size:
            self.setSize(size)
        elif mult and opts.size:
            self.setSize(mult * opts.size)
        elif opts.size:
            self.setSize(opts.size)
        assert self.size != None, \
            "{0}: size not set in macro {1}".format(self, name)
        # persistent
        WithPersistent.__init__(self)
        self.cleanParams(SharedGenerator.DIRS)

class BitGenerator(WithLength, RandomGenerator):
    DIRS = {}
    def __init__(self, att=None, params=None):
        RandomGenerator.__init__(self, att, params)
        WithLength.__init__(self, 8, 32)
        self.cleanParams(BitGenerator.DIRS)
    def genData(self):
        len = self._rand.randint(self.lenmin, self.lenmax)
        return "{0:0{1}b}".format(self._rand.getrandbits(len), len)

def UUIDGenerator(att, params=None):
    if not params:
        params = att.params
    params['pattern'] = r'\h{4}(\h{4}-){4}\h{12}'
    return PatternGenerator(att, params)

# all user-visible generators
GENERATORS = {
    'cat':CatGenerator, 'alt':AltGenerator, 'timestamp':TimestampGenerator,
    'repeat':RepeatGenerator, 'float':FloatGenerator, 'date':DateGenerator,
    'pattern':PatternGenerator, 'int':IntGenerator, 'bool':BoolGenerator,
    'interval':IntervalGenerator, 'const':ConstGenerator, 'word':WordGenerator,
    'string':StringGenerator, 'chars':CharsGenerator, 'text':TextGenerator,
    'blob':BlobGenerator, 'inet':InetGenerator, 'mac':MACGenerator,
    'ean':EANGenerator, 'luhn':LuhnGenerator, 'count':CountGenerator,
    'file':FileGenerator, 'uuid':UUIDGenerator, 'bit':BitGenerator,
    'array':ArrayGenerator, 'tuple':TupleGenerator, 'isnull':NULLGenerator,
    'reduce':ReduceGenerator, 'value':ValueGenerator
}

def strDict(d):
    return '{' + \
        ', '.join("{0!r}: {1!r}".format(k, d[k]) for k in sorted(d)) + '}'

def createGenerator(a, t, params=None, msg=None):
    """Create a generator from specified type.

    - Attribute a: target attribute, may be None
    - str t: name of generator, may be None
    - {} p: additional parameters, may be None
    """
    if params == None: params = a.params
    if not t:
        t = findGenerator(msg, params)
    if not t and 'type' in params:
        t = findGeneratorType(params['type'])
    assert t, "no generator in {0}".format(msg if msg else a)
    assert t in GENERATORS, \
        "unexpected generator '{0}' in {1}".format(t, msg if msg else a)
    gen = GENERATORS[t](a, params)
    # check that all directives have been used
    gen.params.pop(t, None)
    gen.params.pop('type', None)
    assert not gen.params, \
        "unexpected '{0}' directives: {1}".format(t, strDict(gen.params))
    return gen

def macroGenerator(name, att=None):
    assert name in df_macro, \
        "{0}: '{1}' must be a macro".format(att if att else 'Macro', name)
    gen = createGenerator(att, None, params=df_macro[name], msg=name)
    if isinstance(gen, IntGenerator) and gen.size == None: # ???
        gen.setSize(opts.size if opts.size else 1000) # ???
    return gen

#
# UTILS
#

def getParams(dfline):
    """return a dictionnary from a string like "x=123 y='str' ...". """
    if dfline == '':
        return {}
    params = {}
    dfline += ' '
    # eat up leading blanks, just in case
    while dfline[0] == ' ':
        dfline = dfline[1:]
    # do parse
    while len(dfline)>0:
        # python does not like combining assign & test, so use continue
        d = df_txt.match(dfline)
        if d:
            params[d.group(1)] = str(d.group(2))
            dfline = d.group(3)
            continue
        d = df_flt.match(dfline)
        if d:
            params[d.group(1)] = float(d.group(2))
            dfline = d.group(3)
            continue
        d = df_int.match(dfline)
        if d:
            params[d.group(1)] = int(d.group(2))
            dfline = d.group(3)
            continue
        d = df_str.match(dfline)
        if d:
            # handle use of a macro directly
            if d.group(1) == 'use':
                assert d.group(2) in df_macro, \
                    "macro {0} is defined".format(d.group(2))
                params.update(df_macro[d.group(2)])
            else:
                params[d.group(1)] = d.group(2)
            dfline = d.group(3)
            continue
        d = df_bol.match(dfline)
        if d:
            params[d.group(1)] = True
            dfline = d.group(2)
            continue
        raise Exception("cannot parse: '{0}'".format(dfline))
    return params

#
# Relation model
#
class Model(object):
    """Modelize something"""
    # global parameters and their types
    PARAMS = { 'size':int, 'offset':int, 'null':float, 'seed':str, 'type':str }
    @staticmethod
    def checkPARAMS(obj, params, PARAMS):
        """Check parameters against their description."""
        for k, v in params.items():
            assert k in PARAMS, "unexpected parameter '{0}'".format(k)
            if PARAMS[k] == float:
                assert type(v) is float or type(v) is int, \
                    "type {0} instead of float parameter '{1}'". \
                    format(v.__class__.__name__, k)
            elif PARAMS[k] == int:
                assert type(v) is int, \
                    "type {0} instead of int for parameter '{1}'". \
                    format(v.__class__.__name__, k)
            elif PARAMS[k] == bool:
                assert type(v) is bool, \
                    "type {0} instead bool for parameter '{1}'". \
                    format(v.__class__.__name__, k)
            # fix value type in some cases
            if type(v) == bool and PARAMS[k] == str:
                params[k] = ''
            elif type(v) != str and PARAMS[k] == str:
                params[k] = str(v)
            elif type(v) == int and PARAMS[k] == float:
                params[k] = float(v)
        assert not ('mult' in params and 'size' in params), \
            "{0}: must not have both 'mult' and 'size'".format(obj)
        # else everything is fine
    def __init__(self, name):
        # unquote attributes names (PostgreSQL or MySQL)
        self.quoted = name[0] == '"' or name[0] == '`'
        self.name = name[1:-1] if self.quoted else name.lower()
        self.size = None
        self.params = {}
    def __str__(self):
        return self.getName()
    def checkParams(self):
        Model.checkPARAMS(self, self.params, self.__class__.PARAMS)
        # handle type additions on the fly
        if 'type' in self.params:
            addType(self.params['type'])
            del self.params['type']
    def setParams(self, dfline):
        self.params.update(getParams(dfline))
        self.checkParams()
    def getName(self):
        return db.quoteIdent(self.name) if self.quoted else self.name

def findGenerator(obj, params):
    """Get generator from parameters."""
    found = None
    for k in sorted(params):
        if k in sorted(GENERATORS):
            assert not found, \
                "{0}: multiple generators '{1}' and '{2}'".format(obj, found, k)
            found = k
    return found

def findGeneratorType(type):
    """Get generator based on SQL type"""
    return \
        'array'     if db.arrayType(type)     else \
        'int'       if db.intType(type)       else \
        'string'    if db.textType(type)      else \
        'bool'      if db.boolType(type)      else \
        'date'      if db.dateType(type)      else \
        'timestamp' if db.timestampType(type) else \
        'interval'  if db.intervalType(type)  else \
        'float'     if db.floatType(type)     else \
        'blob'      if db.blobType(type)      else \
        'inet'      if db.inetType(type)      else \
        'mac'       if db.macAddrType(type)   else \
        'ean'       if db.eanType(type)       else \
        'uuid'      if db.uuidType(type)      else \
        'bit'       if db.bitType(type)       else \
        None

class Attribute(Model):
    """Represent an attribute in a table."""
    # all attribute PARAMETERS and their types
    PARAMS = { 'nogen':bool, 'mult':float }
    # directives from hidden generators & options
    PARAMS.update(Generator.DIRS)
    PARAMS.update(RandomGenerator.DIRS)
    PARAMS.update(SharedGenerator.DIRS)
    PARAMS.update(WithLength.DIRS)
    # directives from public generators
    import inspect
    for k in GENERATORS:
        if inspect.isclass(GENERATORS[k]):
            PARAMS.update(GENERATORS[k].DIRS)
    # bool directives from generator names
    for k in GENERATORS:
        if not k in PARAMS:
            PARAMS[k] = bool
    # methods
    def __init__(self, name, number, type):
        Model.__init__(self, name)
        self.number = number
        self.type = type.lower()
        self.FK = None
        self.isPK = False
        self.unique = False
        self.not_null = False
        self.gen = None
    def __repr__(self):
        return "{0}: {1} type={2} PK={3} U={4} NN={5} FK=[{6}]". \
               format(self, self.number, self.type,
                      self.isPK, self.unique, self.not_null, self.FK)
    def __str__(self):
        return "Attribute {0}.{1}". \
            format(self.table.getName(), self.getName())
    def getGenerator(self):
        return findGenerator(self, self.params)
    def getData(self):
        assert self.gen, "{0}: generator not set".format(self)
        return self.gen.getData()
    def checkParams(self):
        Model.checkParams(self)
        assert not ('mult' in self.params and 'size' in self.params), \
            "{0}: not both 'mult' and 'size'".format(self)
        # other consistency checks would be possible
    def isUnique(self):
        return self.isPK or self.unique
    def isNullable(self):
        return not self.not_null and not self.isPK
    def isSerial(self):
        return db.serialType(self.type)

class Table(Model):
    """Represent a relational table."""
    # table-level parameters
    PARAMS = { 'mult':float, 'size':int, 'nogen':bool,
               'skip':float, 'null':float }
    def __init__(self, name):
        Model.__init__(self, name)
        # attributes
        self.atts = {}
        self.att_list = [] # list of attributes in occurrence order
        self.unique = []
        self.ustuff = {} # uniques are registered in this dictionnary
        self.constraints = []
    def __str__(self):
        return "Table {0} [{1:d}] ({2})". \
            format(self.getName(), self.size, strDict(self.params))
    def __repr__(self):
        s = str(self) + "\n"
        for att in self.att_list:
            s += "  " + repr(att) + "\n"
        for u in self.unique:
            s += "  UNIQUE: " + str(u) + "\n"
        s += "  CONSTRAINTS: " + str(self.constraints) + "\n"
        return s
    def checkParams(self):
        Model.checkParams(self)
    def addAttribute(self, att):
        self.att_list.append(att)
        self.atts[att.name] = att
        # point back
        att.table = self
        # pg-specific generated constraint names
        if att.isPK:
            self.constraints.append(self.name + '_pkey')
            att.not_null = True
        elif att.isUnique():
            self.constraints.append(self.name + '_' + att.name + '_key')
        elif att.FK:
            self.constraints.append(self.name + '_' + att.name + '_fkey')
    def addUnique(self, atts, type=None):
        if opts.debug: debug(3, "addUnique " + str(atts) + " type=" + type)
        assert len(atts) >= 1
        if len(atts) == 1:
            att = self.getAttribute(atts[0])
            if type != None and type.lower() == 'unique':
                att.unique = True
            else:
                att.isPK = True
                att.not_null = True
        else: # multiple attributes
            self.unique.append([self.getAttribute(a).number for a in atts])
            self.constraints.append(self.name + '_' + \
                                    '_'.join(a.lower() for a in atts) +
                                ('_key' if type.lower()=='unique' else '_pkey'))
    def getAttribute(self, name):
        assert name.lower() in self.atts, "{0} Attribute {1}".format(self, name)
        return self.atts[name.lower()]
    def getPK(self):
        for a in self.att_list:
            if a.isPK:
                return a
        assert False, "{0}: no PK found".format(self)
    def mapGen(self, func):
        # apply func on all attribute's generators
        list(map(lambda g: g.mapGen(func), \
            map(lambda a: a.gen, filter(lambda x: x.gen, self.att_list))))
    def getData(self):
        # generate initial stuff
        lg = list(filter(lambda x: x.gen, self.att_list))
        # possibly reseed shared
        global tuple_count
        tuple_count += 1
        self.mapGen(lambda s: s.shareSeed())
        l0 = [g.getData() for g in lg]
        l = l0
        tries = opts.tries
        while tries:
            tries -= 1
            nu, collision = 0, False
            sul= []
            for u in self.unique:
                # one case is not implemented:
                # unique contains indexes in att_list, but we do not know
                # to which value it corresponds in the generated list because
                # some attributes may have been skipped in the generation.
                # should generate a mapping to get the index among generated
                assert len(l) == len(self.att_list), \
                    "{0}: no unique subset".format(self)
                nu += 1
                su = str(nu) + ':' + str([l[i-1] for i in u])
                if su in self.ustuff:
                    collision = True
                    break # non unique tuple
                else:
                    sul.append(su)
            if collision:
                # update l, but reuse l0 if serial/pk, otherwise it cycles!
                i = 0
                l = []
                # reseed again! otherwise the new value would be generated
                # unother option would be to keep tle previous value when
                # the generator is synchronized.
                tuple_count += 1
                self.mapGen(lambda s: s.shareSeed())
                for g in lg:
                    l.append(l0[i] if g.isUnique() or g.isSerial() \
                             else g.getData())
                    i += 1
                continue # restart while with updated values
            else: # record unique value
                for su in sul:
                    self.ustuff[su] = 1
                return l
        assert False, \
            "{0}: cannot build tuple after {1} tries".format(self, opts.tries)

#
# Databases
#

class Database(object):
    """Abstract a database target."""
    def echo(self, s):
        raise Exception('not implemented in abstract class')
    # transactions
    def begin(self):
        raise Exception('not implemented in abstract class')
    def commit(self):
        raise Exception('not implemented in abstract class')
    # operations
    def insertBegin(self, table):
        raise Exception('not implemented in abstract class')
    def insertValue(self, table, value, isLast):
        raise Exception('not implemented in abstract class')
    def insertEnd(self):
        raise Exception('not implemented in abstract class')
    def setSequence(self, att, number):
        raise Exception('not implemented in abstract class')
    def dropTable(self, table):
        return "DROP TABLE {0};".format(self.quote_ident(table.name))
    def truncateTable(self, table):
        return "DELETE FROM {0};".format(self.quote_ident(table.name))
    # types
    def arrayType(self, type):
        t = type.upper()
        return '[' in t or ' ARRAY' in t
    def serialType(self, type):
        return False
    def intType(self, type):
        t = type.lower()
        return t == 'smallint' or t == 'int' or t == 'integer' or t == 'bigint'
    def textType(self, type):
        t = type.lower()
        return t == 'text' or re.match(r'(var)?char\(', t)
    def boolType(self, type):
        t = type.lower()
        return t == 'bool' or t == 'boolean'
    def dateType(self, type):
        return type.lower() == 'date'
    def intervalType(self, type):
        return type.lower() == 'interval'
    def timestampType(self, type):
        return re.match(RE_TSTZ + '$', type, re.I)
    def floatType(self, type):
        return re.match(RE_FLT + '$', type, re.I)
    def blobType(self, type):
        return re.match(RE_BLO + '$', type, re.I)
    def inetType(self, type):
        return re.match(RE_IPN + '$', type, re.I)
    def macAddrType(self, type):
        return re.match(RE_MAC + '$', type, re.I)
    def eanType(self, s):
        return 0
    def uuidType(self, s):
        return s.upper() == 'UUID'
    def bitType(self, type):
        return re.match(RE_BIT + '$', type, re.I)
    # quoting
    def quoteIdent(self, ident):
        raise Exception('not implemented in abstract class')
    def quoteLiteral(self, literal):
        return '\'' + literal + '\''
    # values
    def null(self):
        raise Exception('not implemented in abstract class')
    def boolValue(self, b):
        return 'TRUE' if b else 'FALSE'
    def dateValue(self, d):
        return d.strftime('%Y-%m-%d')
    def timestampValue(self, t, tz=None):
        ts = t.strftime('%Y-%m-%d %H:%M:%S')
        if tz:
            ts += ' ' + tz
        return ts
    def intervalValue(self, val, unit):
        return str(val) + ' ' + unit
    def blobValue(self, lo):
        raise Exception('not implemented in abstract class')
    def showValue(self, s):
        return s
    analyse = None

class PostgreSQL(Database):
    """PostgreSQL database."""
    def echo(self, s):
        return '\\echo ' + s
    def begin(self):
        return 'BEGIN;'
    def commit(self):
        return 'COMMIT;'
    def insertBegin(self, table):
        return "COPY {0} ({1}) FROM STDIN{2};".format(
            table.getName(),
            ','.join(a.getName() \
                     for a in filter(lambda x: x.gen, table.att_list)),
            " (ENCODING '{0}')".format(opts.encoding) if opts.encoding else '')
    def quoteEsc(self, s, q, esc, n):
        if s == None:
            return n
        elif isinstance(s, list): # ',' is for all types but Box...
            return '{' + ','.join(self.dQuoteEsc(i) for i in s) + '}'
        elif isinstance(s, bool):
            return self.boolValue(s)
        elif isinstance(s, numeric_types):
            return str(s)
        elif isinstance(s, bytes):
            return db.blobValue(s)
        elif isinstance(s, tuple):
            return '(' + ','.join(self.dQuoteEsc(e, '') for e in s) + ')'
        else:
            return q + ''.join(esc[c] if c in esc else c for c in str(s)) + q
    # double quotes
    DQESC = { '"':r'\"', '\\':r'\\'}
    def dQuoteEsc(self, s, null='NULL'):
        return self.quoteEsc(s, '"', PostgreSQL.DQESC, null)
    # simple quotes
    SQESC = { "'":r"''"}
    def sQuoteEsc(self, s, null='NULL'):
        return self.quoteEsc(s, "'", PostgreSQL.SQESC, null)
    # how to escape some characters for PostgreSQL's COPY:
    CPESC = {'\n':r'\n', '\t':r'\t', '\b':r'\b', '\r':r'\r', '\f':r'\f',
            '\v':r'\v', '\a':r'\007', '\0':r'\000', '\\':r'\\'}
    def copyEsc(self, s):
        return self.quoteEsc(s, '', PostgreSQL.CPESC, self.null())
    # overwrite value display function
    showValue = copyEsc
    def insertValue(self, table, value, isLast):
        # generate tab-separated possibly escaped values
        return '\t'.join(self.copyEsc(i) for i in value)
    def insertEnd(self):
        return '\\.'
    def setSequence(self, tab, att, number):
        name = "{0}_{1}_seq".format(tab.name, att.name)
        if tab.quoted or att.quoted:
            name = db.quoteIdent(name)
        return "ALTER SEQUENCE {0} RESTART WITH {1};".format(name, number)
    def dropTable(self, table):
        return "DROP TABLE IF EXISTS {0};".format(table.getName())
    def truncateTable(self, table):
        return "TRUNCATE TABLE {0} CASCADE;".format(table.getName())
    def quoteIdent(self, ident):
        # no escaping?
        return '"{0}"'.format(ident)
    def null(self):
        return r'\N'
    def serialType(self, type):
        return is_ser.match(type)
    def intType(self, type):
        t = type.lower()
        return Database.intType(self, t) or \
            self.serialType(type) or is_int.match(t)
    def eanType(self, type):
        """retun the expected size."""
        t = type.upper()
        return 0 if not re.match('(' + RE_EAN + ')', t) else \
            12 if t == 'UPC' else \
            13 if t[-2:] == '13' else \
            8 if t == 'ISSN' else \
            10
    def blobValue(self, lo):
        return r'\\x' + ''.join("{0:02x}".format(o) for o in lo)
    def analyse(self, name):
        return "ANALYZE {0};".format(name)

class MySQL(Database):
    """MySQL database (experimental)."""
    def begin(self):
        return 'START TRANSACTION;'
    def commit(self):
        return 'COMMIT;'
    def insertBegin(self, table):
        return "INSERT INTO {0} ({1}) VALUES".format(table.getName(),
                        ','.join(a.getName() for a in table.att_list))
    def insertValue(self, table, value, isLast):
        qv = []
        for v in value:
            qv.append(self.boolValue(v) if type(v) is bool else
                      self.quoteLiteral(v) if type(v) is str else str(v))
        s = '  (' + ','.join(str(v) for v in qv) + ')'
        if not isLast:
            s += ','
        return s
    def insertEnd(self):
        return ';'
    def null(self):
        return 'NULL'
    def intType(self, type):
        t = type.lower()
        return Database.intType(self, t) or t == 'tinyint' or t == 'mediumint'

# class SQLite(Database):
# class CSV(Database):

def debug(level, message):
    """Print a debug message, maybe."""
    if opts.debug >= level:
        sys.stderr.write("*" * level + " " + message + "\n")

# option management
# --size=1000
# --target=postgresql|mysql
# --help is automatic
import sys
import argparse
#version="version {0}".format(version),
opts = argparse.ArgumentParser(
    description='Fill database tables with random data.')
opts.add_argument('-s', '--size', type=int, default=None,
                  help='scale to size')
opts.add_argument('-t', '--target', default='postgresql',
                  help='generate for this engine')
opts.add_argument('-e', '--encoding', type=str, default=None,
                  help='set input & output encoding')
opts.add_argument('-f', '--filter', action='store_true', default=False,
                  help='also include input in output')
opts.add_argument('--no-filter', action='store_true', default=None,
                  help='do turn off filtering, whatever!')
opts.add_argument('-T', '--transaction', action='store_true',
                  help='wrap output in a transaction')
opts.add_argument('-S', '--seed', default=None,
                  help='random generator seed')
opts.add_argument('-O', '--offset', type=int, default=None,
                  help='set global offset for integer primary keys')
opts.add_argument('--truncate', action='store_true', default=False,
                  help='truncate table contents before loading')
opts.add_argument('--drop', action='store_true', default=False,
                  help='drop tables before reloading')
opts.add_argument('-D', '--debug', action='count',
                  help='set debug mode')
opts.add_argument('-m', '--man', action='store_const', const=2,
                  help='show man page')
opts.add_argument('-n', '--null', type=float, default=None,
                  help='probability of generating a NULL value')
opts.add_argument('--pod', type=str, default='pod2usage -verbose 3',
                  help='override pod2usage command')
opts.add_argument('-q', '--quiet', action='store_true', default=False,
                  help='less verbose output')
opts.add_argument('--self-test', action='store_true', default=False,
                  help='run automatic self test')
opts.add_argument('--self-test-hack', action='store_true', default=False,
                  help='override system newline for self-test')
opts.add_argument('--self-test-python', type=str, default=None,
                  help="self-test must run with this python")
opts.add_argument('-X', '--test', action='append',
                  help='show generator output for directives')
opts.add_argument('--tries', type=int, default=10,
                  help='how hard to try to satisfy unique constraints')
opts.add_argument('--type', action='append', default=[],
                  help='add custom type')
opts.add_argument('--validate', type=str, default=None,
                  help='shortcut for script validation')
opts.add_argument('-V', action='store_true', default=False,
                  help='show short version on stdout and exit')
opts.add_argument('-v', '--version', action='version',
                  version="version {0}".format(version),
                  help='show version information')
opts.add_argument('file', nargs='*',
                  help='process files, or stdin if empty')
opts = opts.parse_args()

if opts.V:
    print(VERSION)
    sys.exit(0)

# set database
db = None
if opts.target == 'postgresql':
    db = PostgreSQL()
elif opts.target == 'mysql':
    db = MySQL()
else:
    raise Exception("unexpected target database {0}".format(opts.target))

# fix some options
if opts.validate:
    opts.transaction = True
    opts.filter = True

if opts.self_test_hack:
    # ensure some settings under self-test for determinism
    if opts.seed:
        random.seed(opts.seed)
    opts.quiet = True
    # consistent \n and unicode, see print overriding below
    assert not opts.encoding, "no --encoding under self-test"
    opts.encoding = 'utf-8'
    import os
    os.linesep = '\n'

# file generator test
if opts.validate == 'internal':
    generate_some_tmp_files()

if not opts.encoding:
    # note: this is ignored by python3 on input(?)
    opts.encoding =  sys.getfilesystemencoding() if opts.file else \
                     sys.stdin.encoding

if opts.encoding:
    # sys.stdout.encoding = opts.encoding
    import os
    if sys.version_info[0] == 3:
        def __encoded_print(s, end=os.linesep):
            # tell me there is something better than this...
            sys.stdout.buffer.write(s.encode(opts.encoding))
            sys.stdout.buffer.write(end.encode(opts.encoding))
            sys.stdout.flush() # for python 3.4?
        print = __encoded_print
    else: # python 2
        # simpler, although not sure why it works
        print = lambda s, end=os.linesep: \
                sys.stdout.write(s.encode(opts.encoding) + end)

# macros: { 'name':{...} }
df_macro = {}
# some example predefined macros
df_macro['cfr'] = getParams("gen=int:scale rate=0.17")
df_macro['french'] = getParams("chars='esaitnrulodcpmvqfbghjxyzwk' cgen=cfr")
df_macro['cen'] = getParams("gen=int:scale rate=0.15")
df_macro['english'] = getParams("chars='etaonrishdlfcmugypwbvkjxqz' cgen=cen")

# simple generator unit tests
if opts.test:
    AssertionError = StdoutExitError
    ntest = 0
    # process decoded tests
    for t in map(u, opts.test):
        ntest += 1
        print(u8("-- test {0}: {1}").format(ntest, t))
        # macro definition
        m = re.match(r'\s*([\w\.]+)\s*:\s*(.*)', t)
        if m:
            name = m.group(1)
            assert not name in GENERATORS, \
                "do not use generator name '{0}' as a macro name!".format(name)
            df_macro[name] = getParams(m.group(2))
            continue
        # else some directives
        d = re.match(r'\s*([!-]?)\s*(.*)', t)
        h, params = d.group(1), getParams(d.group(2))
        g = findGenerator('test', params)
        if not g and 'type' in params:
            g = findGeneratorType(params['type'])
            assert g, "unknown type {0} ({1})".format(params['type'], t)
        assert g, "must specify a generator: {0}".format(t)
        Model.checkPARAMS('--test', params, Attribute.PARAMS)
        gen = createGenerator(None, g, params)
        if isinstance(gen, IntGenerator) and gen.size == None:
            gen.setSize(opts.size if opts.size else 10)
        assert not gen.params, "unused parameter: {0}".format(gen.params)
        if h == '!': # show histogram
            n = opts.size if opts.size else 10000
            vals = {}
            for i in range(n):
                tuple_count += 1
                gen.mapGen(lambda s: s.shareSeed())
                v = gen.getData()
                vals[v] = 0 if not v in vals else vals[v] + 1
            print("histogram on {0} draws".format(n))
            for v in sorted(vals):
                print(u8("{0}: {1:6.3f} %").format(v, 100.0*vals[v]/n))
        else: # show values
            for i in range(opts.size if opts.size else 10):
                tuple_count += 1
                gen.mapGen(lambda s: s.shareSeed())
                d = db.showValue(gen.getData())
                if h == '-': # short
                    print(str(d), end=' ') # '\t' ?
                else:
                    print(u8("{0}: {1}").format(i, d))
            if h == '-':
                print('')
    # just in case
    cleanup_some_tmp_files()
    sys.exit(0)

# option consistency
if opts.drop or opts.test:
    opts.filter = True

if opts.no_filter: # may be forced back for some tests
    opts.filter = False

assert not (opts.filter and opts.truncate), \
    "option truncate does not make sense with option filter"

if opts.man:
    # Let us use Perl's POD from Python:-)
    import os, tempfile
    pod = tempfile.NamedTemporaryFile(prefix='datafiller_',mode='w')
    name = sys.argv[0].split('/')[-1]
    pod.write(POD.format(comics=COMICS, pgbench=PGBENCH, library=LIBRARY_NODIR,
                         name=name, DOT=RE_DOT, NGENS=len(GENERATORS),
            GLIST=' '.join("B<{0}>".format(g) for g in sorted(GENERATORS)),
              version=version, year=revyear))
    pod.flush()
    os.system(opts.pod + ' ' + pod.name)
    pod.close()
    sys.exit(0)

# auto run a test with some options
def self_run(validate=None, seed=None, pipe=False, op=[]):
    import subprocess
    # command to run
    cmd = [ sys.argv[0] ] + op
    if isinstance(validate, list):
        cmd += map(lambda t: '--test=' + t, validate)
    else: # must be str
        cmd += [ '--validate='+validate ]
    if opts.self_test_python:
        cmd += [ '--self-test-python=' + opts.self_test_python ]
    if seed:
        cmd.append('--seed='+seed)
    if opts.self_test_python:
        cmd.insert(0, opts.self_test_python)
    if opts.debug: debug(1, "self-test cmd: {0}".format(cmd))
    return subprocess.Popen(cmd, stdout=subprocess.PIPE if pipe else None)

# the self-test allows to test the script on hosts without PostgreSQL
def self_test(validate=None, seed='Calvin', D=None):
    import subprocess, hashlib, time
    start = time.time()
    h = hashlib.sha256()
    p = self_run(validate, seed=seed, pipe=True, op=['--self-test-hack'])
    for line in p.stdout:
        h.update(line)
    okay = p.wait() == 0
    d = h.hexdigest()[0:16]
    end = time.time()
    print("self-test {0} seed={1} hash={2} seconds={3:.2f}: {4}".
          format(validate, seed, d, end-start,
                 'PASS' if okay and d == D else 'FAIL'))
    return okay and d == D

if opts.self_test:
    # self test results for python 2 & 3
    TESTS = [
        # [test, seed, [ py2h, py3h ]]
        ['unit', 'Wormwood!', ['73d9b211839c90d6', '05183e0be6ac649e']],
        ['internal', 'Moe!!', ['8b56d03d6220dca3', 'aa7ccdc56a147cc4']],
        ['library', 'Calvin', ['d778fe6adc57eea7', 'adc029f5d9a3a0fb']],
        ['comics', 'Hobbes!', ['fff09e9ac33a4e2a', '49edd6998e04494e']],
        ['pgbench', 'Susie!', ['891cd4a00d89d501', '55d66893247d3461']]]
    fail = 0
    for test, seed, hash in TESTS:
        if not opts.validate or opts.validate == test:
            fail += not self_test(test, seed, hash[sys.version_info[0]-2])
    sys.exit(fail)

# reset arguments for fileinput
sys.argv[1:] = opts.file

#
# global variables while parsing
#

# list of tables in occurrence order, for regeneration
tables = []
all_tables = {}

# parser status
current_table = None
current_attribute = None
current_enum = None
dfstuff = None
att_number = 0

# enums
all_enums = {}
re_enums = ''

# custom types management
re_types = ''
column_type = None
def addType(t):
    global re_types, column_type
    if re_types:
        re_types += '|' + t
    else:
        re_types = t
    column_type = re.compile(r'^\s*,?\s*(ADD\s+COLUMN\s+)?({0})\s+({1}{2})'. \
                             format(RE_IDENT, re_types, RE_ARRAY), re.I)

for t in opts.type:
    addType(t)

# schema stores global parameters
schema = Model('df')

#
# INPUT SCHEMA
#
if opts.validate:
    if opts.validate == 'unit':
        run_unit_tests(seed=opts.seed)
    elif opts.validate == 'internal':
        lines = StringIO(INTERNAL).readlines()
    elif opts.validate == 'library':
        lines = StringIO(LIBRARY).readlines()
        if opts.self_test_hack:
            lines.append("--df T=Borrow A=borrowed:end='2038-01-19 03:14:07'\n")
    elif opts.validate == 'comics':
        lines = StringIO(COMICS).readlines()
    elif opts.validate == 'pgbench':
        lines = StringIO(PGBENCH).readlines()
    else:
        raise Exception("unexpected validation {0}".format(opts.validate))
    lines = [u8(l) for l in lines]
else:
    import fileinput # despite the name this is really a filter...
    fi = fileinput.input(openhook=fileinput.hook_encoded(opts.encoding))
    lines = [l for l in fi]

#
# SCHEMA PARSER
#

re_quoted = re.compile(r"[^']*'(([^']|'')*)'(.*)")
def sql_string_list(line):
    """Return an unquoted list of sql strings."""
    sl = []
    quoted = re_quoted.match(line)
    while quoted:
        sl.append(quoted.group(1))
        line = quoted.group(3)
        quoted = re_quoted.match(line)
    return sl

for line in lines:
    #if opts.debug: sys.stderr.write(u8("line={0}").format(line))
    # skip \commands
    if backslash.match(line):
        continue
    # skip commented out directives
    if df_junk.match(line):
        continue
    # get datafiller stuff
    # directives with explicit table & attribute
    d = df_tab.match(line)
    if d:
        if opts.debug: debug(2, "explicit TA directive")
        current_table = all_tables[d.group(2).lower()]
        current_attribute = None
    d = df_att.match(line)
    if d:
        if opts.debug: debug(2, "set attribute")
        assert current_table!=None
        current_attribute = current_table.getAttribute(d.group(2))
    # extract directive
    d = df_dir.match(line)
    if d:
        if opts.debug: debug(2, "is a directive")
        dfstuff = d.group(1)
    # get datafiller macro definition
    d = df_mac.match(line)
    if d:
        if opts.debug: debug(2, "is a macro")
        mname = d.group(1)
        if mname in df_macro:
            sys.stderr.write("warning: macro {0} is redefined\n".
                             format(mname))
        assert mname not in GENERATORS, \
            "do not use generator name '{0}' as a macro name!".format(mname)
        df_macro[mname] = getParams(d.group(2))
        # reset current params so that it is not stored in an object
        dfstuff = None
    # cleanup comments
    c = comments.match(line)
    if c:
        if opts.debug: debug(2, "cleanup comment")
        line = c.group(1)
    # reset current object
    # argh, beware of ALTER TABLE ... ALTER COLUMN!
    if new_object.match(line) and not alter_column.match(line):
        current_table = None
        current_attribute = None
        current_enum = None
        att_number = 0
    #
    # CREATE TYPE ... AS ENUM
    #
    is_ce = create_enum.match(line)
    if is_ce:
        current_enum = is_ce.group(1) # lower()?
        if opts.debug:
            debug(2, "create enum " + current_enum)
        if re_enums:
            re_enums += '|'
        # ??? should escape special characters such as "."
        re_enums += current_enum
        all_enums[current_enum] = sql_string_list(line)
        column_enum = re.compile(r'\s*,?\s*(ADD\s+COLUMN\s+)?({0})\s+({1})'. \
                                 format(RE_IDENT, re_enums), re.I)
        continue
    # follow up...
    if current_enum:
        all_enums[current_enum].extend(sql_string_list(line))
        continue
    #
    # CREATE TYPE ... AS
    #
    is_ty = create_type.match(line)
    if is_ty:
        addType(is_ty.group(1))
        continue
    #
    # ALTER/CREATE TABLE
    #
    is_at = alter_table.match(line)
    if is_at:
        name = is_at.group(2)
        if opts.debug: debug(2, "alter table " + name)
        current_table = all_tables[name.lower()]
        att_number = len(current_table.atts)
    is_ct = create_table.match(line)
    if is_ct:
        name = is_ct.group(1)
        if opts.debug: debug(2, "create table " + name)
        current_table = Table(name)
        tables.append(current_table)
        all_tables[name.lower()] = current_table
    elif current_table!=None:
        #
        # COLUMN
        #
        is_enum = False
        # try standard types
        c = column.match(line)
        # try enums
        if not c and column_enum:
            c = column_enum.match(line)
            is_enum = bool(c)
        # try other types
        if not c and column_type:
            c = column_type.match(line)
        if c:
            if opts.debug: debug(2, "column " + c.group(2))
            att_number += 1
            current_attribute = Attribute(c.group(2), att_number, c.group(3))
            current_attribute.is_enum = is_enum
            if primary_key.match(line):
                if opts.debug: debug(2, "primary key")
                current_attribute.isPK = True
            if unique.match(line):
                if opts.debug: debug(2, "unique")
                current_attribute.unique = True
            if not_null.match(line):
                if opts.debug: debug(2, "not null")
                current_attribute.not_null = True
            current_table.addAttribute(current_attribute)
            r = reference.match(line)
            if r:
                if opts.debug: debug(2, "reference")
                target = r.group(1)
                current_attribute.FK = all_tables[target.lower()]
                current_attribute.FKatt = r.group(5) if r.group(4) else None
        # ADD UNIQUE
        q = add_unique.match(line)
        if q:
            if opts.debug: debug(2, "add unique")
            current_table.addUnique(re.split(r'[\s,]+', q.group(3)), q.group(2))
        else:
            # UNIQUE()
            q = unicity.match(line)
            if q:
                if opts.debug: debug(2, "unicity")
                current_table.addUnique(re.split(r'[\s,]+', q.group(2)), \
                                        q.group(1))
        r = add_fk.match(line)
        if r:
            if opts.debug: debug(2, "add foreign key")
            src, target = r.group(2, 3)
            current_attribute = current_table.getAttribute(src)
            current_attribute.FK = all_tables[target.lower()]
            current_attribute.FKatt = r.group(7) if r.group(6) else None
        c = alter_column.match(line)
        if c:
            if opts.debug: debug(2, "alter column " + c.group(1))
            current_attribute = current_table.getAttribute(c.group(1))
            # ... SET NOT NULL
            if not_null.match(line):
                if opts.debug: debug(2, "set not null")
                current_attribute.not_null = True
    # attribute df stuff to current object: schema, table or attribute
    # this come last if the dfstuff is on the same line as its object
    if dfstuff!=None:
        if current_attribute!=None:
            current_attribute.setParams(dfstuff)
        elif current_table!=None:
            current_table.setParams(dfstuff)
        else:
            schema.setParams(dfstuff)
        dfstuff = None

#
# SET DEFAULT VALUES for some options, possibly from directives
#
if opts.size == None:
    opts.size = schema.params.get('size', 100)

if not opts.offset:
    opts.offset = schema.params.get('offset')

if not opts.null:
    opts.null = schema.params.get('null', 0.01)

if not opts.seed:
    opts.seed = schema.params.get('seed')

# set seed, default uses os random or time
random.seed(opts.seed)

#
# START OUTPUT
#
if not opts.self_test_hack:
    print('')
    print("-- data generated by {0} version {1} for {2}".
          format(sys.argv[0], version, opts.target))

if opts.test and opts.filter:
    print('')
    print("\\set ON_ERROR_STOP")

if opts.quiet and opts.target == 'postgresql':
    print('')
    print("SET client_min_messages = 'warning';")

if opts.transaction:
    print('')
    print(db.begin())

#
# DROP
#
if opts.drop:
    print('')
    print('-- drop tables')
    for t in reversed(tables):
        print(db.dropTable(t))

#
# SHOW INPUT
#
if opts.filter:
    print('')
    print('-- INPUT FILE BEGIN')
    for line in lines:
        print(line, end='')
    print('-- INPUT FILE END')

#
# TRUNCATE
#
if opts.truncate:
    print('')
    print('-- truncate tables')
    for t in filter(lambda t: not 'nogen' in t.params, reversed(tables)):
        print(db.truncateTable(t))

#
# SET TABLE AND ATTRIBUTE SIZES
#
# first table sizes
for t in tables:
    t.skip = t.params.pop('skip') if 'skip' in t.params else 0.0
    assert t.skip >= 0.0 and t.skip <= 1.0
    if t.size == None:
        t.size = t.params.pop('size') if 'size' in t.params else \
                 int(t.params.pop('mult')*opts.size) if 'mult' in t.params else\
                 opts.size

# *then* set att sizes and possible offset
for t in tables:
    for a in t.att_list:
        if a.FK != None:
            a.size = a.FK.size
            if a.FK.skip:
                raise Exception("reference on table {0} with skipped tuples".
                                format(a.FK.name))
            key = a.FK.atts[a.FKatt] if a.FKatt else a.FK.getPK()
            assert key.isUnique(), \
                "foreign key {0}.{1} target {2} must be unique". \
                format(a.table.name, a.name, key.name)
            # override default prefix
            # ??? only for text types!?
            if db.textType(a.type):
                assert not 'prefix' in a.params, \
                    "no prefix on FK {0}.{1}".format(a.table.name, a.name)
                a.params['prefix'] = key.params.get('prefix', key.name)
            # transfer all other directives
            for d, v in key.params.items():
                if not d in a.params:
                    a.params[d] = v
        elif 'size' in a.params:
            # the directive is not removed now, it should be done later?
            a.size = a.params['size']
        elif a.size == None:
            a.size = int(t.size * a.params.pop('mult', 1.0))

#
# CREATE DATA GENERATORS per attribute
#
for t in filter(lambda t: not 'nogen' in t.params, tables):
    for a in t.att_list:
        # do not generate
        if 'nogen' in a.params:
            del a.params['nogen']
            a.gen = None
            assert not a.params, \
                "unused '{0}' parameters: {1}".format(a.name, a.params)
            continue
        # generators triggered by their directives
        gname = a.getGenerator()
        if gname:
            a.gen = GENERATORS[gname](a)
            a.gen.params.pop(gname, None)
        # type-based default generators
        elif a.is_enum:
            a.gen = WordGenerator(a, None, words=all_enums[a.type])
        else:
            gname = findGeneratorType(a.type)
            if gname:
                a.gen = GENERATORS[gname](a)
            elif a.type in df_macro:
                # try a macro homonymous to the type name
                a.gen = macroGenerator(a.type, a)
            else:
                a.gen = None
        # checks!
        assert a.gen, \
            "generator for {0}.{1} type {2}". \
                         format(t.name, a.name, a.type)
        assert not a.gen.params, \
            "unused {0}.{1} directives: {2}". \
                         format(t.name, a.name, a.gen.params)

# print tables
if opts.debug:
    sys.stderr.write(str(tables) + "\n")

#
# CALL GENERATORS on each table
#
for t in tables:
    print('')
    if 'nogen' in t.params or t.size == 0:
        t.params.pop('nogen', None)
        assert not t.params, \
            "unused {0} parameters: {1}".format(t.name, t.params)
        print("-- skip table {0}".format(t.name))
    else:
        size = "{0:d}*{1:g}".format(t.size, 1.0-t.skip) if t.skip else \
               str(t.size)
        print("-- fill table {0} ({1})".format(t.name, size))
        if not opts.quiet:
            print(db.echo("# filling table {0} ({1})".format(t.name, size)))
        print(db.insertBegin(t))
        for i in range(t.size):
            tup = t.getData()
            # although the tuple is generated, it may yet not be inserted
            if not t.skip or not random.random() < t.skip or i==t.size-1:
                print(db.insertValue(t, tup, i==t.size-1))
        print(db.insertEnd())

#
# CLEANUP
#
cleanup_some_tmp_files()

#
# RESTART SEQUENCES
#
print('')
print('-- restart sequences')
for t in filter(lambda t: not 'nogen' in t.params, tables):
    for a in filter(lambda a: a.isSerial() and a.gen, t.att_list):
        print(db.setSequence(t, a, a.gen.offset + a.gen.size))

#
# DONE
#

if opts.transaction:
    print('')
    print(db.commit())

#
# ANALYZE, if needed
#

if db.analyse:
    print('')
    print('-- analyze modified tables')
    for t in filter(lambda t: not 'nogen' in t.params, tables):
        print(db.analyse(t.getName()))

#
# validation
#
if opts.validate == 'internal':
    print(INTERNAL_CHECK);
