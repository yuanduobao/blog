

# PG系统函数实现原理-ydb

# `pg_proc`初始化文件`pg_proc.dat`



​        Postgresql的perl脚本文件`src/backend/utils/Gen_fmgrtab.pl`会读取`pg_proc`的初始化文件`pg_proc.dat`，自动生成c语言文件`src/backend/utils/fmgrtab.c`。文件`fmgrtab.c`是pg系统函数的函数管理表。



## 文件路径

```sh
src/include/catalog/pg_proc.dat
```

 

## 初始化文件的说明

```c
#----------------------------------------------------------------------
#
# pg_proc.dat
#    Initial contents of the pg_proc system catalog.
#
# Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
# src/include/catalog/pg_proc.dat
#
#----------------------------------------------------------------------

[

# Note: every entry in pg_proc.dat is expected to have a 'descr' comment,
# except for functions that implement pg_operator.dat operators and don't
# have a good reason to be called directly rather than via the operator.
# (If you do expect such a function to be used directly, you should
# duplicate the operator's comment.)  initdb will supply suitable default
# comments for functions referenced by pg_operator.

# Try to follow the style of existing functions' comments.
# Some recommended conventions:
#
# "I/O" for typinput, typoutput, typreceive, typsend functions
# "I/O typmod" for typmodin, typmodout functions
# "aggregate transition function" for aggtransfn functions, unless
# they are reasonably useful in their own right
# "aggregate final function" for aggfinalfn functions (likewise)
# "convert srctypename to desttypename" for cast functions
# "less-equal-greater" for B-tree comparison functions

# Note: pronargs is computed when this file is read, so it does not need
# to be specified in entries here.  See AddDefaultValues() in Catalog.pm.

# Once upon a time these entries were ordered by OID.  Lately it's often
# been the custom to insert new entries adjacent to related older entries.
# Try to do one or the other though, don't just insert entries at random.
```



## 文件内容

部分内容如下：

```sh
#----------------------------------------------------------------------
#
# pg_proc.dat
#    Initial contents of the pg_proc system catalog.
#
# Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
# src/include/catalog/pg_proc.dat
#
#----------------------------------------------------------------------

[

# Note: every entry in pg_proc.dat is expected to have a 'descr' comment,
# except for functions that implement pg_operator.dat operators and don't
# have a good reason to be called directly rather than via the operator.
# (If you do expect such a function to be used directly, you should
# duplicate the operator's comment.)  initdb will supply suitable default
# comments for functions referenced by pg_operator.

# Try to follow the style of existing functions' comments.
# Some recommended conventions:
#
# "I/O" for typinput, typoutput, typreceive, typsend functions
# "I/O typmod" for typmodin, typmodout functions
# "aggregate transition function" for aggtransfn functions, unless
# they are reasonably useful in their own right
# "aggregate final function" for aggfinalfn functions (likewise)
# "convert srctypename to desttypename" for cast functions
# "less-equal-greater" for B-tree comparison functions

# Note: pronargs is computed when this file is read, so it does not need
# to be specified in entries here.  See AddDefaultValues() in Catalog.pm.

# Once upon a time these entries were ordered by OID.  Lately it's often
# been the custom to insert new entries adjacent to related older entries.
# Try to do one or the other though, don't just insert entries at random.

# OIDS 1 - 99

{ oid => '1242', descr => 'I/O',
  proname => 'boolin', prorettype => 'bool', proargtypes => 'cstring',
  prosrc => 'boolin' },
{ oid => '1243', descr => 'I/O',
  proname => 'boolout', prorettype => 'cstring', proargtypes => 'bool',
  prosrc => 'boolout' },
{ oid => '1244', descr => 'I/O',
  proname => 'byteain', prorettype => 'bytea', proargtypes => 'cstring',
  prosrc => 'byteain' },
{ oid => '31', descr => 'I/O',
  proname => 'byteaout', prorettype => 'cstring', proargtypes => 'bytea',
  prosrc => 'byteaout' },
```



# Makefile会调用perl脚本`Gen_fmgrtab.pl`生成文件`fmgrtab.c`

文件`src/backend/utils/Makefile`会调用perl脚本`Gen_fmgrtab.pl`生成c语言源文件`fmgrtab.c`：

```sh
# fmgr-stamp records the last time we ran Gen_fmgrtab.pl.  We don't rely on
# the timestamps of the individual output files, because the Perl script
# won't update them if they didn't change (to avoid unnecessary recompiles).
fmgr-stamp: Gen_fmgrtab.pl $(catalogdir)/Catalog.pm $(top_srcdir)/src/include/catalog/pg_proc.dat $(top_srcdir)/src/include/access/transam.h
	$(PERL) -I $(catalogdir) $< --include-path=$(top_srcdir)/src/include/ $(top_srcdir)/src/include/catalog/pg_proc.dat
	touch $@
```



# Perl脚本`Gen_fmgrtab.pl`

## 文件名

```sh
src/backend/utils/Gen_fmgrtab.pl
```





## 脚本内容

该脚本会调用`Catalog.pm`来解析`pg_proc.dat`文件的内容并生成perl语言的数据结构。



```sh
#! /usr/bin/perl
#-------------------------------------------------------------------------
#
# Gen_fmgrtab.pl
#    Perl script that generates fmgroids.h, fmgrprotos.h, and fmgrtab.c
#    from pg_proc.dat
#
# Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
#
# IDENTIFICATION
#    src/backend/utils/Gen_fmgrtab.pl
#
#-------------------------------------------------------------------------

use Catalog;

use strict;
use warnings;
use Getopt::Long;

my $output_path = '';
my $include_path;

GetOptions(
	'output:s'       => \$output_path,
	'include-path:s' => \$include_path) || usage();

# Make sure output_path ends in a slash.
if ($output_path ne '' && substr($output_path, -1) ne '/')
{
	$output_path .= '/';
}

# Sanity check arguments.
die "No input files.\n"                   unless @ARGV;
die "--include-path must be specified.\n" unless $include_path;

# Read all the input files into internal data structures.
# Note: We pass data file names as arguments and then look for matching
# headers to parse the schema from. This is backwards from genbki.pl,
# but the Makefile dependencies look more sensible this way.
# We currently only need pg_proc, but retain the possibility of reading
# more than one data file.
my %catalogs;
my %catalog_data;
foreach my $datfile (@ARGV)
{
	$datfile =~ /(.+)\.dat$/
	  or die "Input files need to be data (.dat) files.\n";

	my $header = "$1.h";
	die "There in no header file corresponding to $datfile"
	  if !-e $header;

	my $catalog = Catalog::ParseHeader($header);
	my $catname = $catalog->{catname};
	my $schema  = $catalog->{columns};

	$catalogs{$catname} = $catalog;
	$catalog_data{$catname} = Catalog::ParseData($datfile, $schema, 0);
}

# Collect certain fields from pg_proc.dat.
my @fmgr = ();

foreach my $row (@{ $catalog_data{pg_proc} })
{
	my %bki_values = %$row;

	# Select out just the rows for internal-language procedures.
	next if $bki_values{prolang} ne 'internal';

	push @fmgr,
	  {
		oid    => $bki_values{oid},
		strict => $bki_values{proisstrict},
		retset => $bki_values{proretset},
		nargs  => $bki_values{pronargs},
		prosrc => $bki_values{prosrc},
	  };
}

# Emit headers for both files
my $tmpext     = ".tmp$$";
my $oidsfile   = $output_path . 'fmgroids.h';
my $protosfile = $output_path . 'fmgrprotos.h';
my $tabfile    = $output_path . 'fmgrtab.c';

open my $ofh, '>', $oidsfile . $tmpext
  or die "Could not open $oidsfile$tmpext: $!";
open my $pfh, '>', $protosfile . $tmpext
  or die "Could not open $protosfile$tmpext: $!";
open my $tfh, '>', $tabfile . $tmpext
  or die "Could not open $tabfile$tmpext: $!";

print $ofh <<OFH;
/*-------------------------------------------------------------------------
 *
 * fmgroids.h
 *    Macros that define the OIDs of built-in functions.
 *
 * These macros can be used to avoid a catalog lookup when a specific
 * fmgr-callable function needs to be referenced.
 *
 * Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * NOTES
 *	******************************
 *	*** DO NOT EDIT THIS FILE! ***
 *	******************************
 *
 *	It has been GENERATED by src/backend/utils/Gen_fmgrtab.pl
 *
 *-------------------------------------------------------------------------
 */
#ifndef FMGROIDS_H
#define FMGROIDS_H

/*
 *	Constant macros for the OIDs of entries in pg_proc.
 *
 *	NOTE: macros are named after the prosrc value, ie the actual C name
 *	of the implementing function, not the proname which may be overloaded.
 *	For example, we want to be able to assign different macro names to both
 *	char_text() and name_text() even though these both appear with proname
 *	'text'.  If the same C function appears in more than one pg_proc entry,
 *	its equivalent macro will be defined with the lowest OID among those
 *	entries.
 */
OFH

print $pfh <<PFH;
/*-------------------------------------------------------------------------
 *
 * fmgrprotos.h
 *    Prototypes for built-in functions.
 *
 * Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * NOTES
 *	******************************
 *	*** DO NOT EDIT THIS FILE! ***
 *	******************************
 *
 *	It has been GENERATED by src/backend/utils/Gen_fmgrtab.pl
 *
 *-------------------------------------------------------------------------
 */

#ifndef FMGRPROTOS_H
#define FMGRPROTOS_H

#include "fmgr.h"

PFH

print $tfh <<TFH;
/*-------------------------------------------------------------------------
 *
 * fmgrtab.c
 *    The function manager's table of internal functions.
 *
 * Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * NOTES
 *
 *	******************************
 *	*** DO NOT EDIT THIS FILE! ***
 *	******************************
 *
 *	It has been GENERATED by src/backend/utils/Gen_fmgrtab.pl
 *
 *-------------------------------------------------------------------------
 */

#include "postgres.h"

#include "access/transam.h"
#include "utils/fmgrtab.h"
#include "utils/fmgrprotos.h"

TFH

# Emit #define's and extern's -- only one per prosrc value
my %seenit;
foreach my $s (sort { $a->{oid} <=> $b->{oid} } @fmgr)
{
	next if $seenit{ $s->{prosrc} };
	$seenit{ $s->{prosrc} } = 1;
	print $ofh "#define F_" . uc $s->{prosrc} . " $s->{oid}\n";
	print $pfh "extern Datum $s->{prosrc}(PG_FUNCTION_ARGS);\n";
}

# Create the fmgr_builtins table, collect data for fmgr_builtin_oid_index
print $tfh "\nconst FmgrBuiltin fmgr_builtins[] = {\n";
my %bmap;
$bmap{'t'} = 'true';
$bmap{'f'} = 'false';
my @fmgr_builtin_oid_index;
my $last_builtin_oid = 0;
my $fmgr_count       = 0;
foreach my $s (sort { $a->{oid} <=> $b->{oid} } @fmgr)
{
	print $tfh
	  "  { $s->{oid}, $s->{nargs}, $bmap{$s->{strict}}, $bmap{$s->{retset}}, \"$s->{prosrc}\", $s->{prosrc} }";

	$fmgr_builtin_oid_index[ $s->{oid} ] = $fmgr_count++;
	$last_builtin_oid = $s->{oid};

	if ($fmgr_count <= $#fmgr)
	{
		print $tfh ",\n";
	}
	else
	{
		print $tfh "\n";
	}
}
print $tfh "};\n";

printf $tfh qq|
const int fmgr_nbuiltins = (sizeof(fmgr_builtins) / sizeof(FmgrBuiltin));

const Oid fmgr_last_builtin_oid = %u;
|, $last_builtin_oid;


# Create fmgr_builtin_oid_index table.
printf $tfh qq|
const uint16 fmgr_builtin_oid_index[%u] = {
|, $last_builtin_oid + 1;

for (my $i = 0; $i <= $last_builtin_oid; $i++)
{
	my $oid = $fmgr_builtin_oid_index[$i];

	# fmgr_builtin_oid_index is sparse, map nonexistent functions to
	# InvalidOidBuiltinMapping
	if (not defined $oid)
	{
		$oid = 'InvalidOidBuiltinMapping';
	}

	if ($i == $last_builtin_oid)
	{
		print $tfh "  $oid\n";
	}
	else
	{
		print $tfh "  $oid,\n";
	}
}
print $tfh "};\n";


# And add the file footers.
print $ofh "\n#endif\t\t\t\t\t\t\t/* FMGROIDS_H */\n";
print $pfh "\n#endif\t\t\t\t\t\t\t/* FMGRPROTOS_H */\n";

close($ofh);
close($pfh);
close($tfh);

# Finally, rename the completed files into place.
Catalog::RenameTempFile($oidsfile,   $tmpext);
Catalog::RenameTempFile($protosfile, $tmpext);
Catalog::RenameTempFile($tabfile,    $tmpext);

sub usage
{
	die <<EOM;
Usage: perl -I [directory of Catalog.pm] Gen_fmgrtab.pl [--include-path/-i <path>] [path to pg_proc.dat]

Options:
    --output         Output directory (default '.')
    --include-path   Include path in source tree

Gen_fmgrtab.pl generates fmgroids.h, fmgrprotos.h, and fmgrtab.c from
pg_proc.dat

Report bugs to <pgsql-bugs\@lists.postgresql.org>.
EOM
}

exit 0;

```





# Perl脚本`Catalog.pm`

该脚本从catalog文件中提取数据到Perl数据结构，方便其他文件继续解析文件中的数据。



## 文件路径

```sh
src/backend/catalog/Catalog.pm
```



## 脚本内容

```sh
#----------------------------------------------------------------------
#
# Catalog.pm
#    Perl module that extracts info from catalog files into Perl
#    data structures
#
# Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
# src/backend/catalog/Catalog.pm
#
#----------------------------------------------------------------------

package Catalog;

use strict;
use warnings;

use File::Compare;


# Parses a catalog header file into a data structure describing the schema
# of the catalog.
sub ParseHeader
{
	my $input_file = shift;

	# There are a few types which are given one name in the C source, but a
	# different name at the SQL level.  These are enumerated here.
	my %RENAME_ATTTYPE = (
		'int16'         => 'int2',
		'int32'         => 'int4',
		'int64'         => 'int8',
		'Oid'           => 'oid',
		'NameData'      => 'name',
		'TransactionId' => 'xid',
		'XLogRecPtr'    => 'pg_lsn');

	my %catalog;
	my $declaring_attributes = 0;
	my $is_varlen            = 0;
	my $is_client_code       = 0;

	$catalog{columns}     = [];
	$catalog{toasting}    = [];
	$catalog{indexing}    = [];
	$catalog{client_code} = [];

	open(my $ifh, '<', $input_file) || die "$input_file: $!";

	# Scan the input file.
	while (<$ifh>)
	{

		# Set appropriate flag when we're in certain code sections.
		if (/^#/)
		{
			$is_varlen = 1 if /^#ifdef\s+CATALOG_VARLEN/;
			if (/^#ifdef\s+EXPOSE_TO_CLIENT_CODE/)
			{
				$is_client_code = 1;
				next;
			}
			next if !$is_client_code;
		}

		if (!$is_client_code)
		{
			# Strip C-style comments.
			s;/\*(.|\n)*\*/;;g;
			if (m;/\*;)
			{

				# handle multi-line comments properly.
				my $next_line = <$ifh>;
				die "$input_file: ends within C-style comment\n"
				  if !defined $next_line;
				$_ .= $next_line;
				redo;
			}

			# Strip useless whitespace and trailing semicolons.
			chomp;
			s/^\s+//;
			s/;\s*$//;
			s/\s+/ /g;
		}

		# Push the data into the appropriate data structure.
		# Caution: when adding new recognized OID-defining macros,
		# also update src/include/catalog/renumber_oids.pl.
		if (/^DECLARE_TOAST\(\s*(\w+),\s*(\d+),\s*(\d+)\)/)
		{
			push @{ $catalog{toasting} },
			  { parent_table => $1, toast_oid => $2, toast_index_oid => $3 };
		}
		elsif (/^DECLARE_(UNIQUE_)?INDEX\(\s*(\w+),\s*(\d+),\s*(.+)\)/)
		{
			push @{ $catalog{indexing} },
			  {
				is_unique => $1 ? 1 : 0,
				index_name => $2,
				index_oid  => $3,
				index_decl => $4
			  };
		}
		elsif (/^CATALOG\((\w+),(\d+),(\w+)\)/)
		{
			$catalog{catname}            = $1;
			$catalog{relation_oid}       = $2;
			$catalog{relation_oid_macro} = $3;

			$catalog{bootstrap} = /BKI_BOOTSTRAP/ ? ' bootstrap' : '';
			$catalog{shared_relation} =
			  /BKI_SHARED_RELATION/ ? ' shared_relation' : '';
			if (/BKI_ROWTYPE_OID\((\d+),(\w+)\)/)
			{
				$catalog{rowtype_oid}        = $1;
				$catalog{rowtype_oid_clause} = " rowtype_oid $1";
				$catalog{rowtype_oid_macro}  = $2;
			}
			else
			{
				$catalog{rowtype_oid}        = '';
				$catalog{rowtype_oid_clause} = '';
				$catalog{rowtype_oid_macro}  = '';
			}
			$catalog{schema_macro} = /BKI_SCHEMA_MACRO/ ? 1 : 0;
			$declaring_attributes = 1;
		}
		elsif ($is_client_code)
		{
			if (/^#endif/)
			{
				$is_client_code = 0;
			}
			else
			{
				push @{ $catalog{client_code} }, $_;
			}
		}
		elsif ($declaring_attributes)
		{
			next if (/^{|^$/);
			if (/^}/)
			{
				$declaring_attributes = 0;
			}
			else
			{
				my %column;
				my @attopts = split /\s+/, $_;
				my $atttype = shift @attopts;
				my $attname = shift @attopts;
				die "parse error ($input_file)"
				  unless ($attname and $atttype);

				if (exists $RENAME_ATTTYPE{$atttype})
				{
					$atttype = $RENAME_ATTTYPE{$atttype};
				}

				# If the C name ends with '[]' or '[digits]', we have
				# an array type, so we discard that from the name and
				# prepend '_' to the type.
				if ($attname =~ /(\w+)\[\d*\]/)
				{
					$attname = $1;
					$atttype = '_' . $atttype;
				}

				$column{type}      = $atttype;
				$column{name}      = $attname;
				$column{is_varlen} = 1 if $is_varlen;

				foreach my $attopt (@attopts)
				{
					if ($attopt eq 'BKI_FORCE_NULL')
					{
						$column{forcenull} = 1;
					}
					elsif ($attopt eq 'BKI_FORCE_NOT_NULL')
					{
						$column{forcenotnull} = 1;
					}

					# We use quotes for values like \0 and \054, to
					# make sure all compilers and syntax highlighters
					# can recognize them properly.
					elsif ($attopt =~ /BKI_DEFAULT\(['"]?([^'"]+)['"]?\)/)
					{
						$column{default} = $1;
					}
					elsif (
						$attopt =~ /BKI_ARRAY_DEFAULT\(['"]?([^'"]+)['"]?\)/)
					{
						$column{array_default} = $1;
					}
					elsif ($attopt =~ /BKI_LOOKUP\((\w+)\)/)
					{
						$column{lookup} = $1;
					}
					else
					{
						die
						  "unknown or misformatted column option $attopt on column $attname";
					}

					if ($column{forcenull} and $column{forcenotnull})
					{
						die "$attname is forced both null and not null";
					}
				}
				push @{ $catalog{columns} }, \%column;
			}
		}
	}
	close $ifh;
	return \%catalog;
}

# Parses a file containing Perl data structure literals, returning live data.
#
# The parameter $preserve_formatting needs to be set for callers that want
# to work with non-data lines in the data files, such as comments and blank
# lines. If a caller just wants to consume the data, leave it unset.
sub ParseData
{
	my ($input_file, $schema, $preserve_formatting) = @_;

	open(my $ifd, '<', $input_file) || die "$input_file: $!";
	$input_file =~ /(\w+)\.dat$/
	  or die "Input file $input_file needs to be a .dat file.\n";
	my $catname = $1;
	my $data    = [];

	# Scan the input file.
	while (<$ifd>)
	{
		my $hash_ref;

		if (/{/)
		{
			# Capture the hash ref
			# NB: Assumes that the next hash ref can't start on the
			# same line where the present one ended.
			# Not foolproof, but we shouldn't need a full parser,
			# since we expect relatively well-behaved input.

			# Quick hack to detect when we have a full hash ref to
			# parse. We can't just use a regex because of values in
			# pg_aggregate and pg_proc like '{0,0}'.  This will need
			# work if we ever need to allow unbalanced braces within
			# a field value.
			my $lcnt = tr/{//;
			my $rcnt = tr/}//;

			if ($lcnt == $rcnt)
			{
				# We're treating the input line as a piece of Perl, so we
				# need to use string eval here. Tell perlcritic we know what
				# we're doing.
				eval '$hash_ref = ' . $_;   ## no critic (ProhibitStringyEval)
				if (!ref $hash_ref)
				{
					die "$input_file: error parsing line $.:\n$_\n";
				}

				# Annotate each hash with the source line number.
				$hash_ref->{line_number} = $.;

				# Expand tuples to their full representation.
				AddDefaultValues($hash_ref, $schema, $catname);
			}
			else
			{
				my $next_line = <$ifd>;
				die "$input_file: file ends within Perl hash\n"
				  if !defined $next_line;
				$_ .= $next_line;
				redo;
			}
		}

		# If we found a hash reference, keep it, unless it is marked as
		# autogenerated; in that case it'd duplicate an entry we'll
		# autogenerate below.  (This makes it safe for reformat_dat_file.pl
		# with --full-tuples to print autogenerated entries, which seems like
		# useful behavior for debugging.)
		#
		# Only keep non-data strings if we are told to preserve formatting.
		if (defined $hash_ref)
		{
			push @$data, $hash_ref if !$hash_ref->{autogenerated};
		}
		elsif ($preserve_formatting)
		{
			push @$data, $_;
		}
	}
	close $ifd;

	# If this is pg_type, auto-generate array types too.
	GenerateArrayTypes($schema, $data) if $catname eq 'pg_type';

	return $data;
}

# Fill in default values of a record using the given schema.
# It's the caller's responsibility to specify other values beforehand.
sub AddDefaultValues
{
	my ($row, $schema, $catname) = @_;
	my @missing_fields;

	# Compute special-case column values.
	# Note: If you add new cases here, you must also teach
	# strip_default_values() in include/catalog/reformat_dat_file.pl
	# to delete them.
	if ($catname eq 'pg_proc')
	{
		# pg_proc.pronargs can be derived from proargtypes.
		if (defined $row->{proargtypes})
		{
			my @proargtypes = split /\s+/, $row->{proargtypes};
			$row->{pronargs} = scalar(@proargtypes);
		}
	}

	# Now fill in defaults, and note any columns that remain undefined.
	foreach my $column (@$schema)
	{
		my $attname = $column->{name};

		# No work if field already has a value.
		next if defined $row->{$attname};

		# Ignore 'oid' columns, they're handled elsewhere.
		next if $attname eq 'oid';

		# If column has a default value, fill that in.
		if (defined $column->{default})
		{
			$row->{$attname} = $column->{default};
			next;
		}

		# Failed to find a value for this field.
		push @missing_fields, $attname;
	}

	# Failure to provide all columns is a hard error.
	if (@missing_fields)
	{
		die sprintf "missing values for field(s) %s in %s.dat line %s\n",
		  join(', ', @missing_fields), $catname, $row->{line_number};
	}
}

# If a pg_type entry has an array_type_oid metadata field,
# auto-generate an entry for its array type.
sub GenerateArrayTypes
{
	my $pgtype_schema = shift;
	my $types         = shift;
	my @array_types;

	foreach my $elem_type (@$types)
	{
		next if !(ref $elem_type eq 'HASH');
		next if !defined($elem_type->{array_type_oid});

		my %array_type;

		# Set up metadata fields for array type.
		$array_type{oid}           = $elem_type->{array_type_oid};
		$array_type{autogenerated} = 1;
		$array_type{line_number}   = $elem_type->{line_number};

		# Set up column values derived from the element type.
		$array_type{typname} = '_' . $elem_type->{typname};
		$array_type{typelem} = $elem_type->{typname};

		# Arrays require INT alignment, unless the element type requires
		# DOUBLE alignment.
		$array_type{typalign} = $elem_type->{typalign} eq 'd' ? 'd' : 'i';

		# Fill in the rest of the array entry's fields.
		foreach my $column (@$pgtype_schema)
		{
			my $attname = $column->{name};

			# Skip if we already set it above.
			next if defined $array_type{$attname};

			# Apply the BKI_ARRAY_DEFAULT setting if there is one,
			# otherwise copy the field from the element type.
			if (defined $column->{array_default})
			{
				$array_type{$attname} = $column->{array_default};
			}
			else
			{
				$array_type{$attname} = $elem_type->{$attname};
			}
		}

		# Lastly, cross-link the array to the element type.
		$elem_type->{typarray} = $array_type{typname};

		push @array_types, \%array_type;
	}

	push @$types, @array_types;

	return;
}

# Rename temporary files to final names.
# Call this function with the final file name and the .tmp extension.
#
# If the final file already exists and has identical contents, don't
# overwrite it; this behavior avoids unnecessary recompiles due to
# updating the mod date on unchanged header files.
#
# Note: recommended extension is ".tmp$$", so that parallel make steps
# can't use the same temp files.
sub RenameTempFile
{
	my $final_name = shift;
	my $extension  = shift;
	my $temp_name  = $final_name . $extension;

	if (-f $final_name
		&& compare($temp_name, $final_name) == 0)
	{
		unlink($temp_name) || die "unlink: $temp_name: $!";
	}
	else
	{
		rename($temp_name, $final_name) || die "rename: $temp_name: $!";
	}
	return;
}

# Find a symbol defined in a particular header file and extract the value.
# include_path should be the path to src/include/.
sub FindDefinedSymbol
{
	my ($catalog_header, $include_path, $symbol) = @_;
	my $value;

	# Make sure include path ends in a slash.
	if (substr($include_path, -1) ne '/')
	{
		$include_path .= '/';
	}
	my $file = $include_path . $catalog_header;
	open(my $find_defined_symbol, '<', $file) || die "$file: $!";
	while (<$find_defined_symbol>)
	{
		if (/^#define\s+\Q$symbol\E\s+(\S+)/)
		{
			$value = $1;
			last;
		}
	}
	close $find_defined_symbol;
	return $value if defined $value;
	die "$file: no definition found for $symbol\n";
}

# Similar to FindDefinedSymbol, but looks in the bootstrap metadata.
sub FindDefinedSymbolFromData
{
	my ($data, $symbol) = @_;
	foreach my $row (@{$data})
	{
		if ($row->{oid_symbol} eq $symbol)
		{
			return $row->{oid};
		}
	}
	die "no definition found for $symbol\n";
}

# Extract an array of all the OIDs assigned in the specified catalog headers
# and their associated data files (if any).
# Caution: genbki.pl contains equivalent logic; change it too if you need to
# touch this.
sub FindAllOidsFromHeaders
{
	my @input_files = @_;

	my @oids = ();

	foreach my $header (@input_files)
	{
		$header =~ /(.+)\.h$/
		  or die "Input files need to be header files.\n";
		my $datfile = "$1.dat";

		my $catalog = Catalog::ParseHeader($header);

		# We ignore the pg_class OID and rowtype OID of bootstrap catalogs,
		# as those are expected to appear in the initial data for pg_class
		# and pg_type.  For regular catalogs, include these OIDs.
		if (!$catalog->{bootstrap})
		{
			push @oids, $catalog->{relation_oid}
			  if ($catalog->{relation_oid});
			push @oids, $catalog->{rowtype_oid} if ($catalog->{rowtype_oid});
		}

		# Not all catalogs have a data file.
		if (-e $datfile)
		{
			my $catdata =
			  Catalog::ParseData($datfile, $catalog->{columns}, 0);

			foreach my $row (@$catdata)
			{
				push @oids, $row->{oid} if defined $row->{oid};
			}
		}

		foreach my $toast (@{ $catalog->{toasting} })
		{
			push @oids, $toast->{toast_oid}, $toast->{toast_index_oid};
		}
		foreach my $index (@{ $catalog->{indexing} })
		{
			push @oids, $index->{index_oid};
		}
	}

	return \@oids;
}

1;

```





# PG系统（内部）函数的函数管理表

## 文件路径

```sh
src/backend/utils/fmgrtab.c
```



## 文件功能

- 该文件是由Perl脚本`src/backend/utils/Gen_fmgrtab.pl`根据`pg_proc`的初始化文件`src/include/catalog/pg_proc.dat`自动生成的。



- 该文件定义了以下数据结构的一个数组`fmgr_builtins`，存放每一个PG内部函数的信息，比如：c语言的函数名，c语言的函数地址（指针），参数个数，返回的数据是否是集合等。

```c
/*
 * This table stores info about all the built-in functions (ie, functions
 * that are compiled into the Postgres executable).
 */

typedef struct
{
	Oid			foid;			/* OID of the function */
	short		nargs;			/* 0..FUNC_MAX_ARGS, or -1 if variable count */
	bool		strict;			/* T if function is "strict" */
	bool		retset;			/* T if function returns a set */
	const char *funcName;		/* C name of the function */   /* c语言函数名，pg源程序根据该名字查找系统函数对应的c语言函数的指针。 */
	PGFunction	func;			/* pointer to compiled function */   /* 函数指针 */
} FmgrBuiltin;
```



数组`fmgr_builtins`中的成员的Oid不是连续的，有部分Oid没有使用，为了方便根据Oid去数组`fmgr_builtins`查找相应函数的信息，又定义了以下变量记录当前已经使用的最大Oid的值，然后再定义了一个数组`const uint16 fmgr_builtin_oid_index[6122]`。数组`fmgr_builtin_oid_index`使用系统函数的Oid作为数组的下标，对应的数组成员的值如果不是`InvalidOidBuiltinMapping`，那就是对应系统函数在数组`fmgr_builtins`中的位置；否则，就说明该Oid没有使用，没有对应的系统函数。

```c
const Oid fmgr_last_builtin_oid = 6121;
```





## 查找系统函数的方法

文件`src/backend/fmgr/fmgr.c`中的函数`fmgr_lookupByName()`根据函数名查找系统函数信息：



```c
/*
 * Lookup a builtin by name.  Note there can be more than one entry in
 * the array with the same name, but they should all point to the same
 * routine.
 */
static const FmgrBuiltin *
fmgr_lookupByName(const char *name)
{
	int			i;

	for (i = 0; i < fmgr_nbuiltins; i++)
	{
		if (strcmp(name, fmgr_builtins[i].funcName) == 0)
			return fmgr_builtins + i;
	}
	return NULL;
}

```



## 判断sql函数是否是系统函数

文件`src/backend/fmgr/fmgr.c`中：

```c
/*
 * Lookup routines for builtin-function table.  We can search by either Oid
 * or name, but search by Oid is much faster.
 */

static const FmgrBuiltin *
fmgr_isbuiltin(Oid id)
{
	uint16		index;

	/* fast lookup only possible if original oid still assigned */
	if (id > fmgr_last_builtin_oid)
		return NULL;

	/*
	 * Lookup function data. If there's a miss in that range it's likely a
	 * nonexistent function, returning NULL here will trigger an ERROR later.
	 */
	index = fmgr_builtin_oid_index[id];
	if (index == InvalidOidBuiltinMapping)
		return NULL;

	return &fmgr_builtins[index];
}

```



# 数据字典表`pg_proc`

字典表`pg_proc`的第26列是字段`prosrc`，如果是postgresql使用c语言实现的系统函数（即字段prolang等于12），则该字段保存了c语言的函数名称。函数`fmgr_symbol`可以根据该字段通过函数`fmgr_lookupByName`查找数组`fmgr_builtins`获取到函数指针。

```sql
postgres=# \d pg_proc
                   Table "pg_catalog.pg_proc"
     Column      |     Type     | Collation | Nullable | Default 
-----------------+--------------+-----------+----------+---------
 oid             | oid          |           | not null | 
 proname         | name         |           | not null | 
 pronamespace    | oid          |           | not null | 
 proowner        | oid          |           | not null | 
 prolang         | oid          |           | not null | 
 procost         | real         |           | not null | 
 prorows         | real         |           | not null | 
 provariadic     | oid          |           | not null | 
 prosupport      | regproc      |           | not null | 
 prokind         | "char"       |           | not null | 
 prosecdef       | boolean      |           | not null | 
 proleakproof    | boolean      |           | not null | 
 proisstrict     | boolean      |           | not null | 
 proretset       | boolean      |           | not null | 
 provolatile     | "char"       |           | not null | 
 proparallel     | "char"       |           | not null | 
 pronargs        | smallint     |           | not null | 
 pronargdefaults | smallint     |           | not null | 
 prorettype      | oid          |           | not null | 
 proargtypes     | oidvector    |           | not null | 
 proallargtypes  | oid[]        |           |          | 
 proargmodes     | "char"[]     |           |          | 
 proargnames     | text[]       | C         |          | 
 proargdefaults  | pg_node_tree | C         |          | 
 protrftypes     | oid[]        |           |          | 
 prosrc          | text         | C         | not null | 
 probin          | text         | C         |          | 
 proconfig       | text[]       | C         |          | 
 proacl          | aclitem[]    |           |          | 
Indexes:
    "pg_proc_oid_index" UNIQUE, btree (oid)
    "pg_proc_proname_args_nsp_index" UNIQUE, btree (proname, proargtypes, pronamespace)

postgres=# 
```



# 数据结构

## 结构体`List`和`ListCell`

### 源文件

`src/include/nodes/pg_list.h`





### 定义

```c
typedef struct ListCell ListCell;

typedef struct List
{
	NodeTag		type;			/* T_List, T_IntList, or T_OidList */
	int			length;
	ListCell   *head;
	ListCell   *tail;
} List;

struct ListCell
{
	union
	{
		void	   *ptr_value;
		int			int_value;
		Oid			oid_value;
	}			data;
	ListCell   *next;
};

```



### 结构体`ListCell`的字段说明

如果结构体`ListCell`的字段`data`使用的是字段`ptr_value`，则存储的是数据结构`Value`的值，其结构见下面。



## 结构体`Value`

### 源文件

`src/include/nodes/value.h`



### 定义

```c
/*----------------------
 *		Value node
 *
 * The same Value struct is used for five node types: T_Integer,
 * T_Float, T_String, T_BitString, T_Null.
 *
 * Integral values are actually represented by a machine integer,
 * but both floats and strings are represented as strings.
 * Using T_Float as the node type simply indicates that
 * the contents of the string look like a valid numeric literal.
 *
 * (Before Postgres 7.0, we used a double to represent T_Float,
 * but that creates loss-of-precision problems when the value is
 * ultimately destined to be converted to NUMERIC.  Since Value nodes
 * are only used in the parsing process, not for runtime data, it's
 * better to use the more general representation.)
 *
 * Note that an integer-looking string will get lexed as T_Float if
 * the value is too large to fit in an 'int'.
 *
 * Nulls, of course, don't need the value part at all.
 *----------------------
 */
typedef struct Value
{
	NodeTag		type;			/* tag appropriately (eg. T_String) */
	union ValUnion
	{
		int			ival;		/* machine integer */
		char	   *str;		/* string */
	}			val;
} Value;
```



### 相关的宏

```c
#define intVal(v)		(((Value *)(v))->val.ival)
#define floatVal(v)		atof(((Value *)(v))->val.str)
#define strVal(v)		(((Value *)(v))->val.str)
```





# 函数`LookupFuncNameInternal`

## 功能描述

根据函数名查找函数的`oid`。





## 源文件

`src/backend/parser/parse_func.c`





## 函数代码

```c
/*
 * LookupFuncNameInternal
 *		Workhorse for LookupFuncName/LookupFuncWithArgs
 *
 * In an error situation, e.g. can't find the function, then we return
 * InvalidOid and set *lookupError to indicate what went wrong.
 *
 * Possible errors:
 *	FUNCLOOKUP_NOSUCHFUNC: we can't find a function of this name.
 *	FUNCLOOKUP_AMBIGUOUS: nargs == -1 and more than one function matches.
 */
static Oid
LookupFuncNameInternal(List *funcname, int nargs, const Oid *argtypes,
					   bool missing_ok, FuncLookupError *lookupError)
{
	FuncCandidateList clist;

	/* Passing NULL for argtypes is no longer allowed */
	Assert(argtypes);

	/* Always set *lookupError, to forestall uninitialized-variable warnings */
	*lookupError = FUNCLOOKUP_NOSUCHFUNC;

	clist = FuncnameGetCandidates(funcname, nargs, NIL, false, false,
								  missing_ok);

	/*
	 * If no arguments were specified, the name must yield a unique candidate.
	 */
	if (nargs < 0)
	{
		if (clist)
		{
			/* If there is a second match then it's ambiguous */
			if (clist->next)
			{
				*lookupError = FUNCLOOKUP_AMBIGUOUS;
				return InvalidOid;
			}
			/* Otherwise return the match */
			return clist->oid;
		}
		else
			return InvalidOid;
	}

	/*
	 * Otherwise, look for a match to the arg types.  FuncnameGetCandidates
	 * has ensured that there's at most one match in the returned list.
	 */
	while (clist)
	{
		if (memcmp(argtypes, clist->args, nargs * sizeof(Oid)) == 0)
			return clist->oid;
		clist = clist->next;
	}

	return InvalidOid;
}
```



# 函数`fmgr_info_cxt_security`

## 功能

  根据传入的函数的`oid`，调用函数`fmgr_isbuiltin`找到内部函数的地址信息`FmgrInfo`:

```c
/*
 * This struct holds the system-catalog information that must be looked up
 * before a function can be called through fmgr.  If the same function is
 * to be called multiple times, the lookup need be done only once and the
 * info struct saved for re-use.
 *
 * Note that fn_expr really is parse-time-determined information about the
 * arguments, rather than about the function itself.  But it's convenient to
 * store it here rather than in FunctionCallInfoBaseData, where it might more
 * logically belong.
 *
 * fn_extra is available for use by the called function; all other fields
 * should be treated as read-only after the struct is created.
 */
typedef struct FmgrInfo
{
	PGFunction	fn_addr;		/* pointer to function or handler to be called */
	Oid			fn_oid;			/* OID of function (NOT of handler, if any) */
	short		fn_nargs;		/* number of input args (0..FUNC_MAX_ARGS) */
	bool		fn_strict;		/* function is "strict" (NULL in => NULL out) */
	bool		fn_retset;		/* function returns a set */
	unsigned char fn_stats;		/* collect stats if track_functions > this */
	void	   *fn_extra;		/* extra space for use by handler */
	MemoryContext fn_mcxt;		/* memory context to store fn_extra in */
	fmNodePtr	fn_expr;		/* expression parse tree for call, or NULL */
} FmgrInfo;
```



## 源文件

`src/backend/utils/fmgr/fmgr.c`



##  函数调用栈

```gdb
#0  fmgr_info_cxt (functionId=3784, finfo=0x1e9f6a8, mcxt=0x1e9f0b0) at fmgr.c:137
#1  0x00000000006f558d in init_sexpr (foid=3784, input_collation=950, node=0x1ea4fc0, sexpr=0x1e9f688, parent=0x1e9f420, sexprCxt=0x1e9f0b0, allowSRF=true, 
    needDescForSRF=false) at execSRF.c:714
#2  0x00000000006f4641 in ExecInitTableFunctionResult (expr=0x1ea4fc0, econtext=0x1e9f538, parent=0x1e9f420) at execSRF.c:81
#3  0x000000000070a483 in ExecInitFunctionScan (node=0x1e9bff8,	estate=0x1e9f1c8, eflags=16) at nodeFunctionscan.c:352
#4  0x00000000006f1a50 in ExecInitNode (node=0x1e9bff8,	estate=0x1e9f1c8, eflags=16) at execProcnode.c:247
#5  0x00000000006e88cd in InitPlan (queryDesc=0x1e938b8, eflags=16) at execMain.c:1020
#6  0x00000000006e776a in standard_ExecutorStart (queryDesc=0x1e938b8, eflags=16) at execMain.c:266
#7  0x00000000006e7506 in ExecutorStart (queryDesc=0x1e938b8, eflags=0)	at execMain.c:148
#8  0x00000000008e3978 in PortalStart (portal=0x1e51988, params=0x0, eflags=0, snapshot=0x0) at pquery.c:518
#9  0x00000000008ddfea in exec_simple_query (query_string=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);") at postgres.c:1176
#10 0x00000000008e2256 in PostgresMain (argc=1,	argv=0x1de7f60,	dbname=0x1de7da0 "postgres", username=0x1db9db8 "admin") at postgres.c:4247
#11 0x0000000000839272 in BackendRun (port=0x1ddfa80) at postmaster.c:4448
#12 0x0000000000838a38 in BackendStartup (port=0x1ddfa80) at postmaster.c:4139
#13 0x0000000000834df1 in ServerLoop ()	at postmaster.c:1704
#14 0x00000000008346c0 in PostmasterMain (argc=3, argv=0x1db7d20) at postmaster.c:1377
#15 0x0000000000755bd8 in main (argc=3,	argv=0x1db7d20)	at main.c:228
(gdb) 
```



## 函数源码

```c
/*
 * Fill a FmgrInfo struct, specifying a memory context in which its
 * subsidiary data should go.
 */
void
fmgr_info_cxt(Oid functionId, FmgrInfo *finfo, MemoryContext mcxt)
{
	fmgr_info_cxt_security(functionId, finfo, mcxt, false);
}

/*
 * This one does the actual work.  ignore_security is ordinarily false
 * but is set to true when we need to avoid recursion.
 */
static void
fmgr_info_cxt_security(Oid functionId, FmgrInfo *finfo, MemoryContext mcxt,
					   bool ignore_security)
{
	const FmgrBuiltin *fbp;
	HeapTuple	procedureTuple;
	Form_pg_proc procedureStruct;
	Datum		prosrcdatum;
	bool		isnull;
	char	   *prosrc;

	/*
	 * fn_oid *must* be filled in last.  Some code assumes that if fn_oid is
	 * valid, the whole struct is valid.  Some FmgrInfo struct's do survive
	 * elogs.
	 */
	finfo->fn_oid = InvalidOid;
	finfo->fn_extra = NULL;
	finfo->fn_mcxt = mcxt;
	finfo->fn_expr = NULL;		/* caller may set this later */

	if ((fbp = fmgr_isbuiltin(functionId)) != NULL)  /* 如果是内部函数，在查找内部函数表fmgr_builtins并返回数组相应成员的信息. */
	{
		/*
		 * Fast path for builtin functions: don't bother consulting pg_proc
		 */
		finfo->fn_nargs = fbp->nargs;
		finfo->fn_strict = fbp->strict;
		finfo->fn_retset = fbp->retset;
		finfo->fn_stats = TRACK_FUNC_ALL;	/* ie, never track */
		finfo->fn_addr = fbp->func;  /* sql函数对应的c语言函数的指针. */
		finfo->fn_oid = functionId;    /* 函数的oid */
		return;
	}

	/* Otherwise we need the pg_proc entry */
	procedureTuple = SearchSysCache1(PROCOID, ObjectIdGetDatum(functionId));
	if (!HeapTupleIsValid(procedureTuple))
		elog(ERROR, "cache lookup failed for function %u", functionId);
	procedureStruct = (Form_pg_proc) GETSTRUCT(procedureTuple);

	finfo->fn_nargs = procedureStruct->pronargs;
	finfo->fn_strict = procedureStruct->proisstrict;
	finfo->fn_retset = procedureStruct->proretset;

	/*
	 * If it has prosecdef set, non-null proconfig, or if a plugin wants to
	 * hook function entry/exit, use fmgr_security_definer call handler ---
	 * unless we are being called again by fmgr_security_definer or
	 * fmgr_info_other_lang.
	 *
	 * When using fmgr_security_definer, function stats tracking is always
	 * disabled at the outer level, and instead we set the flag properly in
	 * fmgr_security_definer's private flinfo and implement the tracking
	 * inside fmgr_security_definer.  This loses the ability to charge the
	 * overhead of fmgr_security_definer to the function, but gains the
	 * ability to set the track_functions GUC as a local GUC parameter of an
	 * interesting function and have the right things happen.
	 */
	if (!ignore_security &&
		(procedureStruct->prosecdef ||
		 !heap_attisnull(procedureTuple, Anum_pg_proc_proconfig, NULL) ||
		 FmgrHookIsNeeded(functionId)))
	{
		finfo->fn_addr = fmgr_security_definer;
		finfo->fn_stats = TRACK_FUNC_ALL;	/* ie, never track */
		finfo->fn_oid = functionId;
		ReleaseSysCache(procedureTuple);
		return;
	}

	switch (procedureStruct->prolang)
	{
		case INTERNALlanguageId:

			/*
			 * For an ordinary builtin function, we should never get here
			 * because the isbuiltin() search above will have succeeded.
			 * However, if the user has done a CREATE FUNCTION to create an
			 * alias for a builtin function, we can end up here.  In that case
			 * we have to look up the function by name.  The name of the
			 * internal function is stored in prosrc (it doesn't have to be
			 * the same as the name of the alias!)
			 */
			prosrcdatum = SysCacheGetAttr(PROCOID, procedureTuple,
										  Anum_pg_proc_prosrc, &isnull);
			if (isnull)
				elog(ERROR, "null prosrc");
			prosrc = TextDatumGetCString(prosrcdatum);
			fbp = fmgr_lookupByName(prosrc);
			if (fbp == NULL)
				ereport(ERROR,
						(errcode(ERRCODE_UNDEFINED_FUNCTION),
						 errmsg("internal function \"%s\" is not in internal lookup table",
								prosrc)));
			pfree(prosrc);
			/* Should we check that nargs, strict, retset match the table? */
			finfo->fn_addr = fbp->func;
			/* note this policy is also assumed in fast path above */
			finfo->fn_stats = TRACK_FUNC_ALL;	/* ie, never track */
			break;

		case ClanguageId:
			fmgr_info_C_lang(functionId, finfo, procedureTuple);
			finfo->fn_stats = TRACK_FUNC_PL;	/* ie, track if ALL */
			break;

		case SQLlanguageId:
			finfo->fn_addr = fmgr_sql;
			finfo->fn_stats = TRACK_FUNC_PL;	/* ie, track if ALL */
			break;

		default:
			fmgr_info_other_lang(functionId, finfo, procedureTuple);
			finfo->fn_stats = TRACK_FUNC_OFF;	/* ie, track if not OFF */
			break;
	}

	finfo->fn_oid = functionId;
	ReleaseSysCache(procedureTuple);
}

```



# 函数`FuncnameGetCandidates`

## 函数功能

根据函数名和参数个数，获取可能的函数清单。

1. 如果函数参数个数`nargs`等于-1，则返回函数名匹配的所有函数而忽略函数的参数个数。
2. 





## 函数调用示例

```c
FuncCandidateList raw_candidates;	
/* Get list of possible candidates from namespace search */
raw_candidates = FuncnameGetCandidates(funcname, nargs, fargnames,
										   expand_variadic, expand_defaults,
										   false);
```



## 函数说明

```c
/*
 * FuncnameGetCandidates
 *		Given a possibly-qualified function name and argument count,
 *		retrieve a list of the possible matches.
 *
 * If nargs is -1, we return all functions matching the given name,
 * regardless of argument count.  (argnames must be NIL, and expand_variadic
 * and expand_defaults must be false, in this case.)
 *
 * If argnames isn't NIL, we are considering a named- or mixed-notation call,
 * and only functions having all the listed argument names will be returned.
 * (We assume that length(argnames) <= nargs and all the passed-in names are
 * distinct.)  The returned structs will include an argnumbers array showing
 * the actual argument index for each logical argument position.
 *
 * If expand_variadic is true, then variadic functions having the same number
 * or fewer arguments will be retrieved, with the variadic argument and any
 * additional argument positions filled with the variadic element type.
 * nvargs in the returned struct is set to the number of such arguments.
 * If expand_variadic is false, variadic arguments are not treated specially,
 * and the returned nvargs will always be zero.
 *
 * If expand_defaults is true, functions that could match after insertion of
 * default argument values will also be retrieved.  In this case the returned
 * structs could have nargs > passed-in nargs, and ndargs is set to the number
 * of additional args (which can be retrieved from the function's
 * proargdefaults entry).
 *
 * It is not possible for nvargs and ndargs to both be nonzero in the same
 * list entry, since default insertion allows matches to functions with more
 * than nargs arguments while the variadic transformation requires the same
 * number or less.
 *
 * When argnames isn't NIL, the returned args[] type arrays are not ordered
 * according to the functions' declarations, but rather according to the call:
 * first any positional arguments, then the named arguments, then defaulted
 * arguments (if needed and allowed by expand_defaults).  The argnumbers[]
 * array can be used to map this back to the catalog information.
 * argnumbers[k] is set to the proargtypes index of the k'th call argument.
 *
 * We search a single namespace if the function name is qualified, else
 * all namespaces in the search path.  In the multiple-namespace case,
 * we arrange for entries in earlier namespaces to mask identical entries in
 * later namespaces.
 *
 * When expanding variadics, we arrange for non-variadic functions to mask
 * variadic ones if the expanded argument list is the same.  It is still
 * possible for there to be conflicts between different variadic functions,
 * however.
 *
 * It is guaranteed that the return list will never contain multiple entries
 * with identical argument lists.  When expand_defaults is true, the entries
 * could have more than nargs positions, but we still guarantee that they are
 * distinct in the first nargs positions.  However, if argnames isn't NIL or
 * either expand_variadic or expand_defaults is true, there might be multiple
 * candidate functions that expand to identical argument lists.  Rather than
 * throw error here, we report such situations by returning a single entry
 * with oid = 0 that represents a set of such conflicting candidates.
 * The caller might end up discarding such an entry anyway, but if it selects
 * such an entry it should react as though the call were ambiguous.
 *
 * If missing_ok is true, an empty list (NULL) is returned if the name was
 * schema- qualified with a schema that does not exist.  Likewise if no
 * candidate is found for other reasons.
 */
```



## 源文件

```c
src/backend/catalog/namespace.c
```



## 函数处理流程

```sh
FuncnameGetCandidates -> DeconstructQualifiedName    -- 得到函数名 funcname 和 schename的名字
                      -> SearchSysCacheList1(PROCNAMEARGSNSP, CStringGetDatum(funcname));  -- 根据函数名查找字典表pg_proc.
                      																				-- 根据函数名查找内存字典，得到结构体 CatCList。
                      										
                      
```





# 函数`func_get_detail`

## 函数功能

```c
/* func_get_detail()
 *
 * Find the named function in the system catalogs.
 *
 * Attempt to find the named function in the system catalogs with
 * arguments exactly as specified, so that the normal case (exact match)
 * is as quick as possible.
 *
 * If an exact match isn't found:
 *	1) check for possible interpretation as a type coercion request
 *	2) apply the ambiguous-function resolution rules
 *
 * Return values *funcid through *true_typeids receive info about the function.
 * If argdefaults isn't NULL, *argdefaults receives a list of any default
 * argument expressions that need to be added to the given arguments.
 *
 * When processing a named- or mixed-notation call (ie, fargnames isn't NIL),
 * the returned true_typeids and argdefaults are ordered according to the
 * call's argument ordering: first any positional arguments, then the named
 * arguments, then defaulted arguments (if needed and allowed by
 * expand_defaults).  Some care is needed if this information is to be compared
 * to the function's pg_proc entry, but in practice the caller can usually
 * just work with the call's argument ordering.
 *
 * We rely primarily on fargnames/nargs/argtypes as the argument description.
 * The actual expression node list is passed in fargs so that we can check
 * for type coercion of a constant.  Some callers pass fargs == NIL indicating
 * they don't need that check made.  Note also that when fargnames isn't NIL,
 * the fargs list must be passed if the caller wants actual argument position
 * information to be returned into the NamedArgExpr nodes.
 */
```



## 源文件

```c
src/backend/parser/parse_func.c
```



## 函数调用栈

```gdb
(gdb) bt
#0  func_get_detail (funcname=0x1dbdd58, fargs=0x1dbe5d0, fargnames=0x0, nargs=3, argtypes=0x7ffd40b80940, expand_variadic=true, expand_defaults=true, 
    funcid=0x7ffd40b80ad0, rettype=0x7ffd40b80ad4, retset=0x7ffd40b8092f, nvargs=0x7ffd40b80928, vatype=0x7ffd40b80924, true_typeids=0x7ffd40b80938, 
    argdefaults=0x7ffd40b80930) at parse_func.c:1418
#1  0x000000000060fe0a in ParseFuncOrColumn (pstate=0x1dbe320, funcname=0x1dbdd58, fargs=0x1dbe5d0, last_srf=0x0, fn=0x1dbdf10, proc_call=false, location=14)
    at parse_func.c:267
#2  0x000000000060b7db in transformFuncCall (pstate=0x1dbe320, fn=0x1dbdf10) at parse_expr.c:1552
#3  0x0000000000608bf4 in transformExprRecurse (pstate=0x1dbe320, expr=0x1dbdf10) at parse_expr.c:265
#4  0x00000000006087ca in transformExpr (pstate=0x1dbe320, expr=0x1dbdf10, exprKind=EXPR_KIND_FROM_FUNCTION) at parse_expr.c:155
#5  0x00000000005fc105 in transformRangeFunction (pstate=0x1dbe320, r=0x1dbdf68) at parse_clause.c:595
#6  0x00000000005fd7ab in transformFromClauseItem (pstate=0x1dbe320, n=0x1dbdf68, top_rte=0x7ffd40b80fe8, top_rti=0x7ffd40b80fe4, namespace=0x7ffd40b80fd8)
    at parse_clause.c:1119
#7  0x00000000005fb5ec in transformFromClause (pstate=0x1dbe320, frmList=0x1dbe138) at parse_clause.c:138
#8  0x00000000005c4f2d in transformSelectStmt (pstate=0x1dbe320, stmt=0x1dbe170) at analyze.c:1229
#9  0x00000000005c34b6 in transformStmt (pstate=0x1dbe320, parseTree=0x1dbe170) at analyze.c:301
#10 0x00000000005c3391 in transformOptionalSelectInto (pstate=0x1dbe320, parseTree=0x1dbe170) at analyze.c:246
#11 0x00000000005c324f in transformTopLevelStmt (pstate=0x1dbe320, parseTree=0x1dbe288) at analyze.c:196
#12 0x00000000005c30a7 in parse_analyze (parseTree=0x1dbe288, sourceText=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);", 
    paramTypes=0x0, numParams=0, queryEnv=0x0) at analyze.c:116
#13 0x00000000008dd918 in pg_analyze_and_rewrite (parsetree=0x1dbe288, query_string=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);", 
    paramTypes=0x0, numParams=0, queryEnv=0x0) at postgres.c:695
#14 0x00000000008ddf4b in exec_simple_query (query_string=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);") at postgres.c:1140
#15 0x00000000008e2256 in PostgresMain (argc=1, argv=0x1de7f60, dbname=0x1de7da0 "postgres", username=0x1db9db8 "admin") at postgres.c:4247
#16 0x0000000000839272 in BackendRun (port=0x1ddfa80) at postmaster.c:4448
#17 0x0000000000838a38 in BackendStartup (port=0x1ddfa80) at postmaster.c:4139
#18 0x0000000000834df1 in ServerLoop () at postmaster.c:1704
#19 0x00000000008346c0 in PostmasterMain (argc=3, argv=0x1db7d20) at postmaster.c:1377
#20 0x0000000000755bd8 in main (argc=3, argv=0x1db7d20) at main.c:228
(gdb) 
```





# 函数`make_fn_arguments`

## 函数功能

```c
/*
 * make_fn_arguments()
 *
 * Given the actual argument expressions for a function, and the desired
 * input types for the function, add any necessary typecasting to the
 * expression tree.  Caller should already have verified that casting is
 * allowed.
 *
 * Caution: given argument list is modified in-place.
 *
 * As with coerce_type, pstate may be NULL if no special unknown-Param
 * processing is wanted.
 */
```



## 源文件

```c
src/backend/parser/parse_func.c
```



## 函数代码





## 函数调用栈

```c
(gdb) bt
#0  make_fn_arguments (pstate=0x1dbe320, fargs=0x1dbe5d0, actual_arg_types=0x7ffd40b80940, declared_arg_types=0x1dbe800) at parse_func.c:1835
#1  0x0000000000610f52 in ParseFuncOrColumn (pstate=0x1dbe320, funcname=0x1dbdd58, fargs=0x1dbe5d0, last_srf=0x0, fn=0x1dbdf10, proc_call=false, location=14)
    at parse_func.c:671
#2  0x000000000060b7db in transformFuncCall (pstate=0x1dbe320, fn=0x1dbdf10) at parse_expr.c:1552
#3  0x0000000000608bf4 in transformExprRecurse (pstate=0x1dbe320, expr=0x1dbdf10) at parse_expr.c:265
#4  0x00000000006087ca in transformExpr (pstate=0x1dbe320, expr=0x1dbdf10, exprKind=EXPR_KIND_FROM_FUNCTION) at parse_expr.c:155
#5  0x00000000005fc105 in transformRangeFunction (pstate=0x1dbe320, r=0x1dbdf68) at parse_clause.c:595
#6  0x00000000005fd7ab in transformFromClauseItem (pstate=0x1dbe320, n=0x1dbdf68, top_rte=0x7ffd40b80fe8, top_rti=0x7ffd40b80fe4, namespace=0x7ffd40b80fd8)
    at parse_clause.c:1119
#7  0x00000000005fb5ec in transformFromClause (pstate=0x1dbe320, frmList=0x1dbe138) at parse_clause.c:138
#8  0x00000000005c4f2d in transformSelectStmt (pstate=0x1dbe320, stmt=0x1dbe170) at analyze.c:1229
#9  0x00000000005c34b6 in transformStmt (pstate=0x1dbe320, parseTree=0x1dbe170) at analyze.c:301
#10 0x00000000005c3391 in transformOptionalSelectInto (pstate=0x1dbe320, parseTree=0x1dbe170) at analyze.c:246
#11 0x00000000005c324f in transformTopLevelStmt (pstate=0x1dbe320, parseTree=0x1dbe288) at analyze.c:196
#12 0x00000000005c30a7 in parse_analyze (parseTree=0x1dbe288, sourceText=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);", 
    paramTypes=0x0, numParams=0, queryEnv=0x0) at analyze.c:116
#13 0x00000000008dd918 in pg_analyze_and_rewrite (parsetree=0x1dbe288, query_string=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);", 
    paramTypes=0x0, numParams=0, queryEnv=0x0) at postgres.c:695
#14 0x00000000008ddf4b in exec_simple_query (query_string=0x1dbd108 "SELECT * FROM pg_logical_slot_peek_changes('w2m_slot', NULL, NULL);") at postgres.c:1140
#15 0x00000000008e2256 in PostgresMain (argc=1, argv=0x1de7f60, dbname=0x1de7da0 "postgres", username=0x1db9db8 "admin") at postgres.c:4247
#16 0x0000000000839272 in BackendRun (port=0x1ddfa80) at postmaster.c:4448
#17 0x0000000000838a38 in BackendStartup (port=0x1ddfa80) at postmaster.c:4139
#18 0x0000000000834df1 in ServerLoop () at postmaster.c:1704
#19 0x00000000008346c0 in PostmasterMain (argc=3, argv=0x1db7d20) at postmaster.c:1377
#20 0x0000000000755bd8 in main (argc=3, argv=0x1db7d20) at main.c:228
(gdb) 
```



# 函数`fmgr_symbol`

函数`fmgr_symbol`根据函数的`oid`获取到sql函数名：

```c
/*
 * Return module and C function name providing implementation of functionId.
 *
 * If *mod == NULL and *fn == NULL, no C symbol is known to implement
 * function.
 *
 * If *mod == NULL and *fn != NULL, the function is implemented by a symbol in
 * the main binary.
 *
 * If *mod != NULL and *fn !=NULL the function is implemented in an extension
 * shared object.
 *
 * The returned module and function names are pstrdup'ed into the current
 * memory context.
 */
void
fmgr_symbol(Oid functionId, char **mod, char **fn)
{
	HeapTuple	procedureTuple;
	Form_pg_proc procedureStruct;
	bool		isnull;
	Datum		prosrcattr;
	Datum		probinattr;

	/* Otherwise we need the pg_proc entry */
	procedureTuple = SearchSysCache1(PROCOID, ObjectIdGetDatum(functionId));
	if (!HeapTupleIsValid(procedureTuple))
		elog(ERROR, "cache lookup failed for function %u", functionId);
	procedureStruct = (Form_pg_proc) GETSTRUCT(procedureTuple);

	/*
	 */
	if (procedureStruct->prosecdef ||
		!heap_attisnull(procedureTuple, Anum_pg_proc_proconfig, NULL) ||
		FmgrHookIsNeeded(functionId))
	{
		*mod = NULL;			/* core binary */
		*fn = pstrdup("fmgr_security_definer");
		ReleaseSysCache(procedureTuple);
		return;
	}

	/* see fmgr_info_cxt_security for the individual cases */
	switch (procedureStruct->prolang)
	{
		case INTERNALlanguageId:
			prosrcattr = SysCacheGetAttr(PROCOID, procedureTuple,
										 Anum_pg_proc_prosrc, &isnull);
			if (isnull)
				elog(ERROR, "null prosrc");

			*mod = NULL;		/* core binary */
			*fn = TextDatumGetCString(prosrcattr);
			break;

		case ClanguageId:
			prosrcattr = SysCacheGetAttr(PROCOID, procedureTuple,
										 Anum_pg_proc_prosrc, &isnull);
			if (isnull)
				elog(ERROR, "null prosrc for C function %u", functionId);

			probinattr = SysCacheGetAttr(PROCOID, procedureTuple,
										 Anum_pg_proc_probin, &isnull);
			if (isnull)
				elog(ERROR, "null probin for C function %u", functionId);

			/*
			 * No need to check symbol presence / API version here, already
			 * checked in fmgr_info_cxt_security.
			 */
			*mod = TextDatumGetCString(probinattr);
			*fn = TextDatumGetCString(prosrcattr);
			break;

		case SQLlanguageId:
			*mod = NULL;		/* core binary */
			*fn = pstrdup("fmgr_sql");
			break;

		default:
			*mod = NULL;
			*fn = NULL;			/* unknown, pass pointer */
			break;
	}

	ReleaseSysCache(procedureTuple);
}

```

