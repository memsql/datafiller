# DIRECTIVES AND DATA GENERATORS

Directives drive the data sizes and the underlying data generators. They must appear in SQL comments after the object on which they apply, although possibly on the same line, introduced by '-- df: '.

      CREATE TABLE Stuff(         -- df: mult=2.0
	id SERIAL PRIMARY KEY,    -- df: step=19
	data TEXT UNIQUE NOT NULL -- df: prefix=st length=30 lenvar=3
      );

In the above example, with option --size=1000, 2000 tuples will be generated (2.0*1000) with id 1+(i*19)%2000 and unique text data of length about 30+-3 prefixed with st. The sequence for id will be restarted at 2001.

The default size is the number of tuples of the containing table. This implies many collisions for a uniform generator.

## DATA GENERATORS

There are 29 data generators which are selected by the attribute type or possibly directives. Most generators are subject to the null directive which drives the probability of a NULL value. There is also a special shared generator.

Generators are triggered by using directives of their names. If none is specified, a default is chosen based on the attribute type.

`alt` generator:
This generator aggregates other generators by choosing one. The list of sub-generators must be specified as a list of comma-separated weighted datafiller macros provided by directive alt, see below. These generator definition macros must contain an explicit directive to select the underlying generator.

`array` generator:
This generator generates an SQL array from another generator. The sub-generator is specified as a macro name with the array directive. It takes into account the length, lenvar, lenmin and lenmax directives.

`bit` generator:
This generator handles BIT and VARBIT data to store sequences of bits. It takes into account the length, lenvar, lenmin and lenmax directives.

`blob` generator:
This is for blob types, such as PostgreSQL's BYTEA. It uses an int generator internally to drive its extent. It takes into account the length, lenvar, lenmin and lenmax directives. This generator does not support UNIQUE, but uniqueness is very likely if the blob length is significant and the size is large.

`bool` generator:
This generator is used for the boolean type. It is subject to the rate directive.

`cat` generator:
This generator aggregates other generators by concatenating their textual output. The list of sub-generators must be specified as a list of comma-separated datafiller macros provided by a cat directive, see below.

`chars` generator:
This alternate generator for text types generates random string of characters. It is triggered by the chars directive. In addition to the underlying int generator which allows to select values, another int generator is used to build words from the provided list of characters, The cgen directives is the name of a macro which specifies the int generator parameters for the random character selection. It also takes into account the length, lenvar, lenmin and lenmax directives. This generator does not support UNIQUE.

`const` generator:
This generator provides a constant text value. It is driven by the const directive.

`count` generator:
This generator provides a simple counter. It takes into account directives start, step and format.

`date` generator:
This generator is used for the date type. It uses an int generator internally to drive its extent. Its internal working is subject to directives start, end and prec.

`ean` generator:
This is for International Article Number (EAN!) generation. It uses an int generator internally, so the number of distinct numbers can be adjusted with directive size. It takes into account the length and prefix directives. Default is to generate EAN-13 numbers. This generator does not support UNIQUE, but uniqueness is very likely.

`file` generator:
Inline file contents. The mandatory list of files is specified with directive files. See also directive mode.

`float` generator:
This generator is used for floating point types. The directive allows to specify the sub-generator to use, see the float directive below. Its configuration also relies on directives alpha and beta. It does not support UNIQUE, but uniqueness is very likely.

`inet` generator:
This is for internet ip types, such as PostgreSQL's INET and CIDR. It uses an int generator internally to drive its extent. It takes into account the network directive to specify the target network. Handling IPv6 networks requires module netaddr.

`int` generator:
This generator is used directly for integer types, and indirectly by other generators such as text, word and date. Its internal working is subject to directives: sub, size (or mult), offset, shift, step, xor and mangle.

`interval` generator:
This generator is used for the time interval type. It uses the int generator internally to drive its extent. See also the unit directive.

`isnull` generator:
Generates the special NULL value.

`luhn` generator:
This is for numbers which use a Luhn's algorithm checksum, such as bank card numbers. It uses an int generator internally, so the number of distinct numbers can be adjusted with directive size. It takes into account the length and prefix directives. Default length is 16, default prefix is empty. This generator does not support UNIQUE, but uniqueness is very likely.

`mac` generator:
This is for MAC addresses, such as PostgreSQL's MACADDR. It uses an int generator internally, so the number of generated addresses can be adjusted with directive size.

`pattern` generator:
This alternative generator for text types generates text based on a regular expression provided with the pattern directive. It uses internally the alt, cat, repeat, const and chars generators.

`reduce` generator:
This generator applies the reduction operation specified by directive op to generators specified with reduce as a comma-separated list of macros.

`repeat` generator:
This generator aggregates the repetition of another generator. The repeated generator is specified in a macro with a repeat directive, and the number of repetitions relies on the extent directive. It uses an int generator internally to drive the number of repetitions, so it can be skewed by specifying a subtype with the sub directive.

`string` generator:
This generator is used by default for text types. This is a good generator for filling stuff without much ado. It takes into account prefix, and the length can be specified with length, lenvar lenmin and lenmax directives. The generated text is of length length +- lenvar, or between lenmin and lenmax. For CHAR(n) and VARCHAR(n) text types, automatic defaults are set.

`text` generator:
This aggregate generator generates aggregates of words drawn from any other generator specified by a macro in directive text. It takes into account directives separator for separator (default ), prefix and suffix (default empty). It also takes into account the length, lenvar, lenmin and lenmax directives which handle the number of words to generate. This generator does not support UNIQUE, but uniqueness is very likely for a text with a significant length drawn from a dictionary.

`timestamp` generator:
This generator is used for the timestamp type. It is similar to the date generator but at a finer granularity. The tz directive allows to specify the target timezone.

`tuple` generator:
This aggregate generator generates composite types. The list of sub-generators must be specified with tuple as a list of comma-seperated macros.

`uuid` generator:
This generator is used for the UUID type. It is really a pattern generator with a predefined pattern.

`value` generator:
This generator uses per-tuple values from another generator specified as a macro name in the value directive. If the same value is specified more than once in a tuple, the exact same value is generated.

`word` generator:
This alternate generator for text types is triggered by the word directive. It uses int generator to select words from a list or a file. This generator handles UNIQUE if enough words are provided.

## GLOBAL DIRECTIVES

A directive macro can be defined and then used later by inserting its name between the introductory df and the :. The specified directives are stored in the macro and can be reused later. For instance, macros words, mangle cfr and cen can be defined as:

      --df words: word=/etc/dictionaries-common/words sub=power alpha=1.7
      --df mix: offset=10000 step=17 shift=3
      --df cfr: sub=scale alpha=6.7
      --df cen: sub=scale alpha=5.9

Then they can be used in any datafiller directive with use=...:

      --df: use=words use=mix
      --df: use=mix

Or possibly for chars generators with cgen=...:

      --df: cgen=cfr chars='esaitnru...'

There are four predefined macros: cfr and cen define skewed integer generators with the above parameters. french, english define chars generators which tries to mimic the character frequency of these languages.

The size, offset, null, seed and directives can be defined at the schema level to override from the SQL script the default size multiplier, primary key offset, null rate or seed. However, they are ignored if the corresponding options are set.

The type directive at the schema level allows to add custom types, similarly the --type option above.

## TABLE DIRECTIVES

`mult=float`:
Size multiplier for scaling, that is computing the number of tuples to generate. This directive is exclusive from size.

`nogen`:
Do not generate data for this table.

`null`:
Set defaut null rate for this table.

`size=int`:
Use this size, so there is no scaling with the --size option and mult directive. This directive is exclusive from mult.

`skip=float`:
Skip (that is generate but do not insert) some tuples with this probability. Useful to create some holes in data. Tables with a non-zero skip cannot be referenced.

## ATTRIBUTE DIRECTIVES

A specific generator can be specified by using its name in the directives, otherwise a default is provided based on the attribute SQL type. Possible generators are: alt array bit blob bool cat chars const count date ean file float inet int interval isnull luhn mac pattern reduce repeat string text timestamp tuple uuid value word. See also the sub directive to select a sub-generator to control skewness.

`alt=some:2,weighted,macros:2`:
List of macros, possibly weighted (default weight is 1) defining the generators to be used by an alt generator. These macros must include an explicit directive to select a generator.

`array=macro`:
Name of the macro defining an array built upon this generator for the array generator. The macro must include an explicit directive to select a generator.

`cat=list,of,macros`:
List of macros defining the generators to be used by a cat generator. These macros must include an explicit directive to select a generator.

`chars='0123456789A-Z\n' cgen=macro`:
The chars directive triggers the chars generator described above. Directive chars provides a list of characters which are used to build words, possibly including character intervals with '-'. A leading '-' in the list means the dash character as is. Characters can be escaped in octal (e.g. \041 for !) or in hexadecimal (e.g. \x3D for =). Unicode escapes are also supported: eg \u20ac for the Euro symbol and \U0001D11E for the G-clef. Also special escaped characters are: null \0 (ASCII 0), bell \a (7), backspace \b (8), formfeed \f (12), newline \n (10), carriage return \r (13), tab \t (9) and vertical tab \v (11). The macro name specified in directive cgen is used to setup the character selection random generator.

For exemple:

      ...
      -- df skewed: sub=power rate=0.3
      , stuff TEXT -- df: chars='a-f' sub=uniform size=23 cgen=skewed

The text is chosen uniformly in a list of 23 words, each word being built from characters 'abcdef' with the skewed generator described in the corresponding macro definition on the line above.

`const=str`:
Specify the constant text to generate for the const generator. The constant text can contains the escaped characters described with the chars directive above.

`extent=int or extent=int-int`
Specify the extent of the repetition for the repeat generator. Default is 1, that is not to repeat.

`files=str`:
Path-separated patterns for the list for files used by the file generator. For instance to specify image files in the ./img UN*X subdirectory:

      files='./img/*.png:./img/*.jpg:./img/*.gif'

`float=str`:
The random sub-generators for floats are those provided by Python's random:

 -  `beta`:
    Beta distribution, alpha and beta must be >0.

 -  `exp`:
    Exponential distribution with mean 1.0 / alpha

 -  `gamma`:
    Gamma distribution, alpha and beta must be >0.

 -  `gauss`:
    Gaussian distribution with mean alpha and stdev beta.

 -  `log`:
    Log normal distribution, see normal.

 -  `norm`:
    Normal distribution with mean alpha and stdev beta.

 -  `pareto`:
    Pareto distribution with shape alpha.

 -  `uniform`:
    Uniform distribution between alpha and beta. This is the default distribution.

 -  `vonmises`:
    Circular data distribution, with mean angle alpha in radians and concentration beta.

 -  `weibull`:
    Weibull distribution with scale alpha and shape beta.

`format=str`:
Format output for the count generator. Default is d. For instance, setting 08X displays the counter as 0-padded 8-digits uppercase hexadecimal.

`inet=str`:
Use to specify in which IPv4 or IPv6 network to generate addresses. For instance, inet=10.2.14.0/24 chooses ip addresses between 10.2.14.1 and 10.2.14.254, that is network and broadcast addresses are not generated. Similarily, inet=fe80::/112 chooses addresses between fe80::1 and fe80::ffff. The default subnet limit is 24 for IPv4 and 64 for IPv6. A leading , adds the network address, a leading . adds the broadcast address, and a leading ; adds both, thus inet=';10.2.14.0/24' chooses ip addresses between 10.2.14.0 and 10.2.14.255.

`length=int lenvar=int lenmin=int lenmax=int`:
Specify length either as length and variation or length bounds, for generated characters of string data, number of words of text data or blob.

`mangle`:
Whether to automatically choose random shift, step and xor for an int generator.

`mode=str`:
Mode for handling files for the file generator. The value is either blob for binaries or text for text file in the current encoding. Default is to use the binary format, as it is safer to do so.

`mult=float`:
Use this multiplier to compute the generator size. This directive is exclusive from size.

`nogen`:
Do not generate data for this attribute, so it will get its default value.

`null=float`:
Probability of generating a null value for this attribute. This applies to all generators.

`offset=int shift=int step=int`:
Various parameters for generated integers. The generated integer is offset+(shift+step*i)%size. step must not be a divider of size, it is ignored and replaced with 1 if so.

Defaults: offset is 1, shift is 0, step is 1.

`op=(+|*|min|max|cat)`:
Reduction operation for reduce generator.

`pattern=str`:
Provide the regular expression for the pattern generator.

They can involve character sequences like calvin, character escape sequences (octal, hexadecimal, unicode, special) as in directive chars above, character classes like [a-z] and [^a-z] (exclusion), character classes shortcuts like . \d \h \H \s and \w which stand for [ -~] [0-9] [0-9a-f] [0-9A-F] [ \f\n\r\t\v] and [0-9a-zA-Z_] respectively, as well as POSIX character classes within [:...:], for instance [:alnum:] for [0-9A-Za-z] or [:upper:] for [A-Z].

Alternations are specified with |, for instance (hello|world). Repetitions can be specified after an object with {3,8}, which stands for repeat between 3 and 8 times. The question mark ? is a shortcut for {0,1}, the star sign * for {0,8} and the plus sign + for {1,8}.

For instance:

      stuff TEXT NOT NULL -- df: pattern='[a-z]{3,5} ?(!!|\.{3}).*'

means 3 to 5 lower case letters, maybe followed by a space, followed by either 2 bangs or 3 dots, and ending with any non special character.

The special [:GEN ...:] syntax allow to embedded a generator within the generated pattern, possibly including specific directives. For instance the following would generate unique email addresses because of the embedded counter:

      email TEXT UNIQUE NOT NULL
	-- df: pattern='\w+\.[:count format=X:]@somewhere\.org'

`prefix=str`:
Prefix for string ean luhn and text generators.

`rate=float`:
For the bool generator, rate of generating True vs False. Must be in [0, 1]. Default is 0.5.

For the int generator, rate of generating value 0 for generators power and scale.

`repeat=macro`:
Macro which defines the generator to repeat for the repeat generator. See also the extent directive.

`seed=str`:
Set default global seed from the schema level. This can be overriden by option --seed. Default is to used the default random generator seed, usually relying on OS supplied randomness or the current time.

`separator=str`:
Word separator for text generator. Default is (space).

`share=macro`:
Specify the name of a macro which defines an int generator used for synchronizing other generators. If several generators share the same macro, their values within a tuple are correlated between tuples.

`size=int`:
Number of underlying values to generate or draw from, depending on the generator. For keys (primary, foreign, unique) , this is necessarily the corresponding number of tuples. This directive is exclusive from mult.

`start=date/time , end=date/time, prec=int`:
For the date and timestamp generators, issue from start up to end at precision prec. Precision is in days for dates and seconds for timestamp. Default is to set end to current date/time and prec to 1 day for dates et 60 seconds for timestamps. If both start and end are specified, the underlying size is adjusted.

For example, to draw from about 100 years of dates ending on January 19, 2038:

      -- df: end=2038-01-19 size=36525

`sub=SUGENERATOR`:
For integer generators, use this underlying sub-type generator.

The integer sub-types also applies to all generators which inherit from the int generator, namely blob date ean file inet interval luhn mac string text timestamp repeat and word.

The sub-generators for integers are:

 -  `serial`:
    This is really a counter which generates distinct integers, depending on offset, shift, step and xor.

 -  `uniform`:
    Generates uniform random number integers between offset and offset+size-1. This is the default.

 -  `serand`:
    Generate integers based on serial up to size, then use uniform. Useful to fill foreign keys.

 -  `power` with parameter `alpha` or `rate`:
    Use probability to this alpha power. When rate is specified, compute alpha so that value 0 is drawn at the specified rate. Uniform is similar to power with alpha=1.0, or rate=1.0/size The higher alpha, the more skewed towards 0.

    Example distribution with --test='!int sub=power rate=0.3 size=10':

	  value     0   1   2   3   4   5   6   7   8   9
	  percent  30  13  10   9   8   7   6   6   5   5

 -  `scale` with parameter `alpha` or `rate`:
    Another form of skewing. The probability of increasing values drawn is less steep at the beginning compared to power, thus the probability of values at the end is lower.

    Example distribution with --test='!int sub=scale rate=0.3 size=10':

	  value     0   1   2   3   4   5   6   7   8   9
	  percent  30  19  12   9   7   6   5   4   3   2

`suffix=str`:
Suffix for text generator. Default is empty.

`text=macro`:
Macro which defines the word provided generator for the text generator.

`type=str`:
At the schema level, add a custom type which will be recognized as such by the schema parser. At the attribute level, use the generator for this type.

`unit=str`:
The unit directive specifies the unit of the generated intervals. Possible values include s m h d mon y. Default is s, i.e. seconds.

`word=file or word=:list,of,words`:
The word directive triggers the word generator described above, or is used as a source for words by the text generator. Use provided word list or lines of file to generate data. The default size is the size of the word list.

If the file contents is ordered by word frequency, and the int generator is skewed (see sub), the first words can be made to occur more frequently.

`xor=int`:
The xor directive adds a non-linear xor stage for the int generator.
