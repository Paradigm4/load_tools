load_tools
==========

Tools for efficient and error-tolerant loading of character-delimited data into SciDB. This work is currently in prototype phase.

## <a name="use_different_versions"></a>Important: use with different SciDB versions
As of SciDB version 15.7, the load_tools package has been added to the source code as a prototype package. Users of 15.7 no longer need to install the package separately.

#### If you are running SciDB 15.7, simply execute the following AFL:
```
AFL% load_library('prototype_load_tools');
```
After that, all of the functions and operators below will be usable as documented below.

If you are using the earlier SciDB 14.12, follow the installation instructions at the bottom of this page.
The code in this repository remains faithful to 14.12. This code will not build under 15.7. Moreover, this code will be removed at a later date as more users move on to 15.7 and later releases.

## Intro

load_tools is a collection of load-related User-Defined Operators and Functions. The key component is the two-phase split-parse design. The input file (or fs object) is read by the split() operator and converted into simple large string chunks as fast as possible. Those chunks are simultaneously redistributed among all the instances in the cluster. The operator parse() can then be run in parallel to process the chunks into a set number of string attributes. The result of parse may be stored, or converted directly into the final form. The framework offers the following features:

 * performance superior to loadcsv.py (based on preliminary benchmark results)
 * ingesting data into the system with a tolerance for errors 
 * ability to map errors to the exact line number in the original file
 * ability to correct errors in-database
 * a few helpful functions for casting and string manipulation
 * split() and parse() are independent; either operator can be repalced with a different piece of code
 * in the future, split() can be taught to read from multiple files on different instances at the same time

The only drawback of the approach is that it does not respect escaped or quoted delimiters, as doing so will necessarily reduce performance. We like the TSV format as it expressly forbits escaped or quoted delimiters. It is possible to implement a more sophisticated, slower versions of split and parse for those cases.

## Example
We will motivate with this malformed TSV file:
```
$ cat /tmp/testfile 
col1	col2	col3
"alex"	1	3.5
"b"ob"	2	4.8
jake	4.0
random		3.1	"extra stuff"
bill	abc	9
alice	4	not_a_number
```

### Split
Split reads from an absolute path (instance 0 only at the moment) and returns an array of 
```
<value:string> [source_instance_id = 0:*,1,0, chunk_no = 0:*,1,0]
```
####  Split parameters (all optional)
Must be specified in string form; see examples below
 * `lines_per_chunk=n`   where n is a nonnegative integer value of delimited lines per array chunk. Default is 1,000,000.
 * `delimiter=c` where c is a single character or escaped tab (\t) newline (\n) or space. Default is newline.
 * `header=n`  where n is a nonnegative integer value of header lines to skip. Default is 0.

Note, value is a large string that stores an entire block of the file, 1 million lines by default - in a single chunk. The number of lines per chunk and the delimiter character can be passed as optional parameters:
```
$ iquery -aq "split('/tmp/testfile', 'lines_per_chunk=2')"
{source_instance_id,chunk_no} value
{0,0} 'col1	col2	col3
"alex"	1	3.5'
{0,1} '"b"ob"	2	4.8
jake	4.0'
{0,2} 'random		3.1	"extra stuff"
bill	abc	9'
{0,3} 'alice	4	not_a_number'

$ iquery -aq "split('/tmp/testfile', 'lines_per_chunk=3', 'delimiter=\t')"
{source_instance_id,chunk_no} value
{0,0} 'col1	col2	col3
"alex"'
{0,1} '1	3.5
"b"ob"	2'
{0,2} '4.8
jake	4.0
random	'
{0,3} '3.1	"extra stuff"
bill	abc'
{0,4} '9
alice	4	not_a_number
'
```

### Parse
After the file is split, it can be parsed into a desired number of attributes, a0...aN - all nullable strings. The result array is
```
<a0:string null, a1:string null... aN:string null, error:string null> 
[source_instance_id=0:*,1,0, chunk_no=0:*,1,0, line_no=0:*,CS,0]"
```
#### Parse parameters
Must be specified in string form; see examples below
 * `num_attributes=n`   required; n is a nonnegative integer value of the number of columns in the file
 * `chunk_size=CS`  optional; CS is a nonnegative integer chunk size; must match the lines_per_chunk value used in split. Default is 1,000,000.
 * `attribute_delimiter=d`  optional; d is a single character attribute that separates columns in the file, or escaped tab (\t), newline (\n) or space. Default is tab.
 * `line_delimiter=l`  optional; n is single character that separates lines in the file, or escapted tab (\t), newline (\n) or space. Default is newline.
 * `split_on_dimension=s` optional; s is either 0 or 1. If set to 1, the output schema will be single attribute and an extra dimension will be added along which columns are populated. Default is 0. See example below.

The attribute "error" is always provided and populated with null, unless a particular line does not have the matching number of attributes:
```
$ iquery -aq "
 store(
  parse(
   split(
    '/tmp/testfile', 
    'lines_per_chunk=2'
   ), 
   'num_attributes=3', 'chunk_size=2'
  ), 
 tmp
)"
{source_instance_id,chunk_no,line_no} a0,a1,a2,error
{0,0,0} 'col1','col2','col3',null
{0,0,1} '"alex"','1','3.5',null
{0,1,0} '"b"ob"','2','4.8',null
{0,1,1} 'jake','4.0',null,'short'
{0,2,0} 'random','','3.1','long	"extra stuff"'
{0,2,1} 'bill','abc','9',null
{0,3,0} 'alice','4','not_a_number',null
```
There are only two possible non-null error values: "long" and "short" as shown above. Note we've stored this result into a temporary array where we can perform some data correction. Also note, we ingested the header line of the file "col1,...". However, we can easily filter() it out.

### Split Columns along a Dimension
Instead of creating an array with multiple attributes, we can create another dimension along which the columns are populated. The result chema will then be:
```
<a: string null>
[source_instance_id=0:*,1,0, chunk_no=0:*,1,0, line_no=0:*,CS,0, attribute_no=0:{NA},NA+1,0]
```
where "NA" is the 'num_attributes' parameter as passed to the parse operator. Note, we use zero-based intexing and add one more value to contain the "error" status of each line. Indeed, this is just another way to represent the same data. It's very efficient for loading large matrix-like data where all, or most of the columns are the same type. For example, this form is very useful when loading large multi-sample VCF files. To split columns along a new dimension like this, supply the argument 'split_on_dimension=1' to parse like so:
```
$ iquery -aq "
 parse(
  split(
   '/tmp/testfile', 
   'lines_per_chunk=2'
  ), 
  'num_attributes=3', 'split_on_dimension=1'
 )"
{source_instance_id,chunk_no,line_no,attribute_no} a
{0,0,0,0} 'col1'
{0,0,0,1} 'col2'
{0,0,0,2} 'col3'
{0,0,0,3} null
{0,0,1,0} '"alex"'
{0,0,1,1} '1'
{0,0,1,2} '3.5'
{0,0,1,3} null
{0,1,0,0} '"b"ob"'
{0,1,0,1} '2'
{0,1,0,2} '4.8'
{0,1,0,3} null
{0,1,1,0} 'jake'
{0,1,1,1} '4.0'
{0,1,1,2} null
{0,1,1,3} 'short'
{0,2,0,0} 'random'
{0,2,0,1} ''
{0,2,0,2} '3.1'
{0,2,0,3} 'long	"extra stuff"'
{0,2,1,0} 'bill'
{0,2,1,1} 'abc'
{0,2,1,2} '9'
{0,2,1,3} null
{0,3,0,0} 'alice'
{0,3,0,1} '4'
{0,3,0,2} 'not_a_number'
{0,3,0,3} null
```

We can then use an operator like slice to pick out the first column. 
```
$ iquery -aq "
 slice(
  parse(
   split(
    '/tmp/testfile', 
    'lines_per_chunk=2'
   ), 
   'num_attributes=3', 
   'split_on_dimension=1'
  ), 
  attribute_no, 0
 )"
{source_instance_id,chunk_no,line_no} a
{0,0,0} 'col1'
{0,0,1} '"alex"'
{0,1,0} '"b"ob"'
{0,1,1} 'jake'
{0,2,0} 'random'
{0,2,1} 'bill'
{0,3,0} 'alice'
```
We can even join the first few slices together build a partial multi-attribute representation (for example to pick out the first few columns of a VCF file).

### Where are the errors?
We can easily find all the erroneous lines by filtering on the error attribute. We can also map these cells to the original line number in the file by using this simple formula. Remember: the number "2" is the chunk size we used during the load:
```
$ iquery -aq "apply(filter(tmp, error is not null), original_line_number, 1+line_no+chunk_no*2)"
{source_instance_id,chunk_no,line_no} a0,a1,a2,error,original_line_number
{0,1,1} 'jake','4.0',null,'short',4
{0,2,0} 'random','','3.1','long	"extra stuff"',5
```

### dcast: error-tolerant casting
The supplied UDF dcast can be used to cast a string to a double, float, bool, int{64,32,16,8} or uint{64,32,16,8} (at the moment). Note, the default casting in scidb will fail the query on first error. The second argument to dcast is the "default value" to use when the cast fails, usually a null or missing code. For example:
```
$ iquery -aq "
 project(
  apply(
   filter(
   tmp, not (line_no=0 and chunk_no=0)), 
  da2, dcast(a2, double(missing(1)))), 
 a2, da2)"
{source_instance_id,chunk_no,line_no} a2,da2
{0,0,1} '3.5',3.5
{0,1,0} '4.8',4.8
{0,1,1} null,null
{0,2,0} '3.1',3.1
{0,2,1} '9',9
{0,3,0} 'not_a_number',?1
```
Note, the value at {0,1,1} was not present in the file and is thus "null". The value {0,3,0} was not castable, and we decided to use ?1 to differentiate.

dcast ignores any whitespace preceding or following the numeric value. When converting to an unsigned type, a negative input is considered non-convertible and the supplied defaul is returned. An input out of range is also considered non-convertible for integers (overflow will not happen). When converting to float or double, all ranges of values are supported but inf or -inf may be returned if the input exceeds machine limits. When converting to bool, values of 0,N,NO,F,FALSE or 1,Y,YES,T,TRUE (ignore case) are supported. Internally strtod, strtoll, strtoull are used - see those routines for details on locale sensitivity and what "whitespace" means.

### Some string utilities
#### trim() removes specific characters from the beginning and end of a string:
```
$ iquery -aq "
 project(
  apply(
   filter(
    tmp, 
    not (line_no=0 and chunk_no=0)
   ), 
  ta0, trim(a0, '\"')), 
 a0, ta0
 )"
{source_instance_id,chunk_no,line_no} a0,ta0
{0,0,1} '"alex"','alex'
{0,1,0} '"b"ob"','b"ob'
{0,1,1} 'jake','jake'
{0,2,0} 'random','random'
{0,2,1} 'bill','bill'
{0,3,0} 'alice','alice'

$ iquery -aq "
 project(
  apply(
   filter(
    tmp, 
    not (line_no=0 and chunk_no=0)
   ), 
   ta0, trim(a0, '\"b')
  ), 
  a0, ta0
 )"
{source_instance_id,chunk_no,line_no} a0,ta0
{0,0,1} '"alex"','alex'
{0,1,0} '"b"ob"','o'
{0,1,1} 'jake','jake'
{0,2,0} 'random','random'
{0,2,1} 'bill','ill'
{0,3,0} 'alice','alice'
```

#### char_count() counts the occurrences of a particular character or list of characters:
```
$ iquery -aq "apply(build(<val:string>[i=0:0,1,0], 'abc, def, xyz'), cc, char_count(val, 'x'))"
{i} val,cc
{0} 'abc, def, xyz',1

$ iquery -aq "apply(build(<val:string>[i=0:0,1,0], 'abc, def, xyz'), cc, char_count(val, ','))"
{i} val,cc
{0} 'abc, def, xyz',2
```

#### nth_tdv() and nth_csv() extract a substring from a field that contains delimiters:
```
$ iquery -aq "apply(build(<val:string>[i=0:0,1,0], 'abc, def, xyz'), n, nth_csv(val, 0))"
{i} val,n
{0} 'abc, def, xyz','abc'

$ iquery -aq "apply(build(<val:string>[i=0:0,1,0], 'abc, def, xyz'), n, nth_csv(val, 1))"
{i} val,n
{0} 'abc, def, xyz',' def'
```
nth_tdv lets you specify a delimiter other than a comma:
```
$ iquery -aq "apply(build(<val:string>[i=0:0,1,0], 'abc, def, xyz'), n, nth_tdv(val, 1, ' '))"
{i} val,n
{0} 'abc, def, xyz','def,'
```
Note the benefit of using small cross_joins to decompose compound fields.
The iif is present to overcome a limitation in SciDB's logic that determines when an attribute can be nullable,
the iif can be skipped if you know exactly how many fields there are:
```
$ iquery -aq "
 apply(
  cross_join(
   build(
    <val:string>[i=0:0,1,0], 
    'abc, def, xyz'), 
   build(
    <x:int64>[j=0:3,4,0], j)
  ), 
  n, iif(nth_csv(val, j) is null, null, nth_csv(val,j))
 )"
{i,j} val,x,n
{0,0} 'abc, def, xyz',0,'abc'
{0,1} 'abc, def, xyz',1,' def'
{0,2} 'abc, def, xyz',2,' xyz'
{0,3} 'abc, def, xyz',3,null
```
Similarly, maxlen_csv() and maxlen_tdv() first split a string along a delimiter but then return the length of the longest field as an integer. 

####keyed_value() pulls values out of key-value lists
It expects an input in the form of "KEY1=VALUE1;KEY2=VALUE2;.." and returns a value for a given key name. The third argument is a default to return when the key is not found:
```
$ iquery -aq "
 apply(
  build(
   <val:string>[i=0:0,1,0], 'LEN=43;WID=35.3'), 
  l, double(keyed_value(val, 'LEN', null)), 
  w, double(keyed_value(val, 'WID', null))
 )"
{i} val,l,w
{0} 'LEN=43;WID=35.3',43,35.3
```

####throw() terminates a query with an error if a particular condition is not met:
```
$ iquery -aq "apply(apply(build(<val:string>[i=0:0,1,0], 'LEN=43;WID=35.3'), l, double(keyed_value(val, 'LEN', null)), w, double(keyed_value(val, 'WID', null))), input_check, iif(w < 50, true, throw('Invalid Width')))"
{i} val,l,w,input_check
{0} 'LEN=43;WID=35.3',43,35.3,true

$ iquery -aq "
 apply(
  apply(
   build(
    <val:string>[i=0:0,1,0], 'LEN=43;WID=35.3'
   ), 
   l, double(keyed_value(val, 'LEN', null)), 
   w, double(keyed_value(val, 'WID', null))
  ), 
  input_check, iif(w < 30, true, throw('Invalid Width'))
 )"
SystemException in file: Functions.cpp function: toss line: 392
Error id: scidb::SCIDB_SE_INTERNAL::SCIDB_LE_ILLEGAL_OPERATION
Error description: Internal SciDB error. Illegal operation: Invalid Width
```

####codify() converts a given string to its ascii representation:
```
$ iquery -aq "project(apply(tmp, ca0, codify(a0)), a0, ca0)"
{source_instance_id,chunk_no,line_no} a0,ca0
{0,0,0} 'col1','99|111|108|49|0|'
{0,0,1} '"alex"','34|97|108|101|120|34|0|'
{0,1,0} '"b"ob"','34|98|34|111|98|34|0|'
{0,1,1} 'jake','106|97|107|101|0|'
{0,2,0} 'random','114|97|110|100|111|109|0|'
{0,2,1} 'bill','98|105|108|108|0|'
{0,3,0} 'alice','97|108|105|99|101|0|'
```

For an example of using regular expressions, consult the regular expression substitution routine provided in https://github.com/paradigm4/superfunpack

===

## Installing the plug in

**NOTE**: As of SciDB version 15.7, you do not need to install load_tools separately (it is installed by default; see note [here](#use_different_versions))

If you are using an older version of SciDB, read on:
 
First, you'll need SciDB installed. 

After that, the easiest way to install **load_tools**, and load the plugin is by using https://github.com/paradigm4/dev_tools

Otherwise, you can build manually using the SciDB development header packages. The names vary depending on your operating system type, but they are the package that have "-dev" in the name. You *don't* need the SciDB source code to compile and install this.

Run `make` and copy  `*.so` to the `lib/scidb/plugins`
directory on each of your SciDB cluster nodes. Here is an example:

```
cd load_tools
make
cp *.so /opt/scidb/14.8/lib/scidb/plugins

iquery -aq "load_library('load_tools')"
```
Remember to copy the plugin to __all__ your SciDB cluster nodes.
