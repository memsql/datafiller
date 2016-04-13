# TUTORIAL

This tutorial introduces how to use DataFiller to fill a PostgreSQL database for testing functionalities and performances.

## DIRECTIVES IN COMMENTS

The starting point of the script to generate test data is the SQL schema of the database taken from a file. It includes important information that will be used to generate data: attribute types, uniqueness, not-null-ness, foreign keys... The idea is to augment the schema with directives in comments so as to provide additional hints about data generation.

A datafiller directive is a special SQL comment recognized by the script, with a df marker at the beginning. A directive must appear after the object about which it is applied, either directly after the object declaration, in which case the object is implicit, or much later, in which case the object must be explicitely referenced:

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


## A SIMPLE LIBRARY EXAMPLE

Let us start with a simple example involving a library where readers borrow books. Our schema is defined in file library.sql as follows:

  
      CREATE TABLE Book( 
	bid SERIAL PRIMARY KEY,
	title TEXT NOT NULL, 
	isbn ISBN13 NOT NULL 
      );
      CREATE TABLE Reader( 
	rid SERIAL PRIMARY KEY,
	firstname TEXT NOT NULL, 
	lastname TEXT NOT NULL, 
	born DATE NOT NULL, 
	gender BOOLEAN NOT NULL, 
	phone TEXT 
      );
      CREATE TABLE Borrow( 
	borrowed TIMESTAMP NOT NULL, 
	rid INTEGER NOT NULL REFERENCES Reader,
	bid INTEGER NOT NULL REFERENCES Book, 
	PRIMARY KEY(bid) -- a book is borrowed once at a time!
      );

The first and only information you really need to provide is the relative or absolute size of relations. For scaling, the best way is to specify a relative size multiplier with the mult directive on each table, which will be multiplied by the size option to compute the actual size of data to generate in each table. Let us say we want 100 books in stock per reader, with 1.5 borrowed books per reader on average:

      CREATE TABLE Book( -- df: mult=100.0
      ...
      CREATE TABLE Borrow( --df: mult=1.5

The default multiplier is 1.0, it does not need to be set on Reader. Then you can generate a data set with:

      sh> datafiller --size=1000 library.sql > library_test_data.sql

Note that explicit constraints are enforced on the generated data, so that foreign keys in *Borrow* reference existing *Books* and *Readers*. However the script cannot guess implicit constraints, thus if an attribute is not declared NOT NULL, then some NULL values will be generated. If an attribute is not unique, then the generated values will probably not be unique.

## IMPROVING GENERATED VALUES

In the above generated data, some attributes may not reflect the reality one would expect from a library. Changing the default with per-attribute directives will help improve this first result.

First, book titles are all quite short, looking like title_number, with some collisions. Indeed the default is to generate strings with a common prefix based on the attribute name and a number drawn uniformly from the expected number of tuples. We can change to texts composed of between 1 and 7 English words taken from a dictionary:

      title TEXT NOT NULL
      -- df English: word=/etc/dictionaries-common/words
      -- df: text=English length=4 lenvar=3

Also, we may have undesirable collisions on the ISBN attribute, because the default size of the set is the number of tuples in the table. We can extend the size at the attribute level so as to avoid this issue:

      isbn ISBN13 NOT NULL -- df: size=1000000000

If we now look at readers, the result can also be improved. First, we can decide to keep the prefix and number form, but make the statistics more in line with what you can find. Let us draw from 1000 firstnames, most frequent 3%, and 10000 lastnames, most frequent 1%:

      firstname TEXT NOT NULL,
	 -- df: sub=power prefix=fn size=1000 rate=0.03
      lastname TEXT NOT NULL,
	 -- df: sub=power prefix=ln size=10000 rate=0.01

The default generated dates are a few days around now, which does not make much sense for our readers' birth dates. Let us set a range of birth dates.

      birth DATE NOT NULL, -- df: start=1923-01-01 end=2010-01-01

Most readers from our library are female: we can adjust the rate so that 25% of readers are male, instead of the default 50%.

      gender BOOLEAN NOT NULL, -- df: rate=0.25

Phone numbers also have a prefix_number structure, which does not really look like a phone number. Let us draw a string of 10 digits, and adjust the nullable rate so that 1% of phone numbers are not known. We also set the size manually to avoid too many collisions, but we could have chosen to keep them as is, as some readers do share phone numbers.

      phone TEXT
	-- these directives could be on a single line
	-- df: chars='0-9' length=10 lenvar=0
	-- df: null=0.01 size=1000000

The last table is about currently borrowed books. The timestamps are around now, we are going to spread them on a period of 50 days, that is 24 * 60 * 50 = 72000 minutes (precision is 60 seconds).

      borrowed TIMESTAMP NOT NULL -- df: size=72000 prec=60

Because of the primary key constraint, the borrowed books are the first ones. Let us mangle the result so that referenced book numbers are scattered.

      bid INTEGER REFERENCES Book -- df: mangle

Now we can generate improved data for our one thousand readers library, and fill it directly to our library database:

      sh> datafiller --size=1000 --filter library.sql | psql library

Our test database is ready. If we want more users and books, we only need to adjust the size option. Let us query our test data:

      -- show firstname distribution
      SELECT firtname, COUNT(*) AS cnt FROM Reader
      GROUP BY firstname ORDER BY cnt DESC LIMIT 3;
	-- fn_1_... | 33
	-- fn_2_... | 15
	-- fn_3_... | 12
      -- compute gender rate
      SELECT AVG(gender::INT) FROM Reader;
	-- 0.246

## DISCUSSION

We could go on improving the generated data so that it is more realistic. For instance, we could skew the borrowed timestamp so that there are less old borrowings, or skew the book number so that old books (lower numbers) are less often borrowed, or choose firtnames and lastnames from actual lists.

When to stop improving is not obvious: On the one hand, real data may show particular distributions which impact the application behavior and performance, thus it may be important to reflect that in the test data. On the other hand, if nothing is really done about readers, then maybe the only relevant information is the average length of firstnames and lastnames because of the storage implications, and that's it.

## ADVANCED FEATURES

Special generators allow to combine or synchronize other generators.

Let us consider an new email attribute for our Readers, for which we want to generate gmail or yahoo addresses. We can use a pattern generator for this purpose:

      email TEXT NOT NULL CHECK(email LIKE '%@%')
	-- df: pattern='[a-z]{3,8}\.[a-z]{3,8}@(gmail|yahoo)\.com'

The pattern sets 3 to 8 lower case characters, followed by a dot, followed by 3 to 8 characters again, followed by either gmail or yahoo domains. With this approach, everything is chosen uniformly: letters in first and last names appear 1/26, their six possible sizes between 3 to 8 are equiprobable, and each domain is drawn on average by half generated email addresses.

In order to get more control about these distributions, we could rely on the chars or alt generators which can be skewed or weighted, as illustrated in the next example.

Let us now consider an ip address for the library network. We want that 20% comes from librarian subnet '10.1.0.0/16' and 80% from reader subnet '10.2.0.0/16'. This can be achieved with directive alt which tells to make a weighted choice between macro-defined generators:

      -- define two macros
      -- df librarian: inet='10.1.0.0/16'
      -- df reader: inet='10.2.0.0/16'
      ip INET NOT NULL
      -- df: alt=reader:8,librarian:2
      -- This would do as well: --df: alt=reader:4,librarian

Let us finally consider a log table for data coming from a proxy, which stores correlated ethernet and ip addresses, that is ethernet address always get the same ip addess, and we have about 1000 distinct hosts:

	 -- df distinct: int size=1000
      ,  mac MACADDR NOT NULL -- df share=distinct
      ,  ip INET NOT NULL -- df share=distinct inet='10.0.0.0/8'

Because of the share directive, the same mac and ip will be generated together in a pool of 1000 pairs of addresses.

## TUTORIAL CONCLUSION

There are many more directives to drive data generation, from simple type oriented generators to advanced combinators. See the documentation and examples below.

For very application-specific constraints that would not fit any generator, it is also possible to apply updates to modify the generated data afterwards.
