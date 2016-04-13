# DataFiller - generate random data from database schema

_This is a mirror of <https://www.cri.ensmp.fr/people/coelho/datafiller.html>_.
See that site for updated documentation.

## DESCRIPTION

This script generates random data from a database schema enriched with simple
directives in SQL comments to drive 29 data generators which cover typical data
types and their combination. Reasonable defaults are provided, especially based
on key and type constraints, so that few directives should be necessary. The
minimum setup is to specify the relative size of tables with directive mult so
that data generation can be scaled.

See the [TUTORIAL](TUTORIAL.md) section Also, run with --validate=comics or
--validate=library and look at the output for didactic examples.

## OPTIONS

--debug or -D

 -  Set debug mode. Repeat for more. Default is no debug.

--drop

 -  Drop tables before recreating them. This implies option --filter, otherwise
    there would be no table to fill.

    Default is not to.

--encoding=enc or -e enc

 -  Set this encoding for input and output files.

    Default is no explicit encoding.

--filter or -f, reverse with --no-filter

 -  Work as a filter, i.e. send the schema input script to stdout and then the
    generated data. This is convenient to pipe the result of the script directly
    for execution to the database command.

    Default is to only ouput generated data.

--help or -h

 -  Show basic help.

--man or -m

 -  Show full man page based on POD. Yes, the perl thing:-)

--null RATE or -n RATE

 -  Probability to generate a null value for nullable attributes.

    Default is 0.01, which can be overriden by the null directive at the schema
    level, or per-attributes provided null rate.

--offset OFFSET or -O OFFSET

 -  Set default offset for integer generators on primary keys. This is useful to
    extend the already existing content of a database for larger tests.

    Default is 1, which can be overriden by the offset directive at the schema
    level, or per-attribute provided offset.

--pod COMMAND

 -  Override pod conversion command used by option --man.

    Default is 'pod2usage -verbose 3'.

--quiet or -q

 -  Generate less verbose SQL output.

    Default is to generate one echo when starting to fill a table.

--seed SEED or -S SEED

 -  Seed overall random generation with provided string.

    Default uses OS supplied randomness or current time.

--size SIZE

 -  Set overall scaling. The size is combined with the mult directive value on a
    table to compute the actual number of tuples to generate in each table.

    Default is 100, which can be overriden with the size directive at the schema
    level.

--target (postgresql|mysql) or -t ...

 -  Target database engine. MySQL support is really experimental.

    Default is to target PostgreSQL.

--test='directives...'

 -  Run tests for any generator with some directives. If the directives start with
    !, show an histogram. If the directives start with -, show results on one-line.
    This is convenient for unit testing, or to check what would be the output of a
    set of directives.

    Examples: --size=100 --test='int size=10 mangle' would show 100 integers
    between 1 and 10 drawn uniformly. --test='!bool rate=0.3' may show False:
    69.32%, True: 30.68%, stating the rate at which True and False were seen during
    the test.

    Directive type can be used within --test=... to provide the target SQL type
    when testing.

--transaction or -T

 -  Use a global transaction.

    Default is not to.

--tries=NUM

 -  How hard to try to satisfy a compound unique constraint before giving up on a
    given tuple.

    Default is 10.

--truncate

 -  Delete table contents before filling.

    Default is not to.

--type=CUSTOM

 -  Add a custom type. The default generator for this type will rely on a macro of
    the same name, if it exists. This option can be repeated to add more custom
    types. See also the type directive at the schema level.

--validate=(unit|internal|comics|library|pgbench)

 -  Output validation test cases. All but the unit tests can be processed with
    psql.

        sh> datafiller --validate=internal | psql
  
    This option sets --filter automatically, although it can be forced back with
    --no-filter.

    Default is to process argument files or standard input.

--version, -v or -V

 -  Show script version.
