# helpers.sh

Collection of helper functions to be used with flexilooks.

## Overview

Output generic message for a path $1 that does not exist
This function does NOT check to see if the path exists.

## Index

* [path_does_not_exist_msg](#pathdoesnotexistmsg)
* [has_binary](#hasbinary)
* [netcdf_var_to_paths](#netcdfvartopaths)

### path_does_not_exist_msg

Output generic message for a path $1 that does not exist
This function does NOT check to see if the path exists.

#### Arguments

* **$1** (Path): to file that does not exist.

#### Exit codes

* **1**: Always exits with exitcode 1.

### has_binary

Check if binary $1 is available, exit if not

#### Arguments

* **$1** (string): Name of binary to search for.

#### Exit codes

* **1**: If binary not found.

### netcdf_var_to_paths

Return hash map where each key is a variable (or hyphen separated 
variables) mapping to a space separated string of netcdf paths containing that 
(those) variable(s). If you want to update only certain optional arguments,
you may have to pass the default argument of the preceding OPTIONAL argument
to maintain correctness.

#### Example

```bash
# Assuming in project root
declare -A var_to_paths
input_dir="./test/assets/ncheaders/vortex" 
vars_file="./test/assets/vars_file.txt"
vars_regex=$(cat $var_file | tr '\n' '|' | sed 's/|*$//')
exp_name="ICON_NWP_R2B7_VORTEX_srbc"
exclude_substrings="ua 2d"
include_substrings="phy_tend 3d"
start_year=1979
end_year=1979
start_month=1
end_month=1
debug=1
. src/helpers
netcdf_var_to_paths\
   var_to_paths\
   $input_dir\
   $vars_regex\
   $exp_name\
   "$exclude_substrings"\
   "$include_substrings"\
   $start_year\
   $end_year\
   $start_month\
   $end_month\
   $debug 
 
```

#### Arguments

* **$1** (hashmap): (INOUT) Hash map of variable name(s) to string with space
* **$2** (string): Input directory containing netcdf files.
* **$3** (string): Variable names regular expression to be used to select which variables
* **$4** (string): Name of experiment of interest (e.g., ICON_NWP_R2B7_VORTEX_srbc).
* **$5** (string): (OPTIONAL) Space separated string where each substring excludes
* **$6** (string): (OPTIONAL) Space separated string where each substring includes
* **$7** (string): (OPTIONAL) Start year of interest (e.g., 1979). (default: "").
* **$8** (string): (OPTIONAL) End year (inclusive) of interest (e.g., 1979).
* **$9** (string): (OPTIONAL) Start month of interest (e.g., 1). (default: 1).
* **$10** (string): (OPTIONAL) End month (inclusive) of interest (e.g., 12).
* **$11** (string): (OPTIONAL) 1 to print key,value pairs in map, 0 otherwise.

## Source Code

```bash
path_does_not_exist_msg () 
{ 
    local mypath=$1;
    echo "ERROR: Value is a path that does not exist! Double check $mypath";
    echo "NOTE: '~' won't be recognized. If '~' in path, provide absolute path.";
    exit 1
}
```

```bash
has_binary () 
{ 
    local binary_name="$1";
    if ! which "$binary_name" > /dev/null 2>&1; then
        echo "ERROR: Binary not found! Got '$binary_name'.";
        if [[ "$binary_name" == "ncdump" ]]; then
            echo "Load with 'module load netcdf-c' or install it.";
        else
            echo "Load with 'module load $binary_name' or install it.";
        fi;
        exit 1;
    else
        echo "helpers::has_binary::SUCCESS: Found '$binary_name'";
    fi
}
```

```bash
netcdf_var_to_paths () 
{ 
    has_binary "ncdump";
    local -n return_var_to_paths=$1;
    local input_dir=$2;
    local vars_regex=$3;
    local exp_name=$4;
    local exclude_substrings="${5:-""}";
    local include_substrings="${6:-""}";
    local start_year="${7:-""}";
    local end_year="${8:-""}";
    local start_month="${9:-01}";
    local end_month="${10:-12}";
    local debug=${11:-0};
    return_var_to_paths_type=$(declare -p $1);
    if [[ ! "$return_var_to_paths_type" =~ "declare -A" ]]; then
        echo "ERROR: Type of return_var_to_paths (argument \$1)" "must be 'declare -A' type!";
        echo "Got '$return_var_to_paths_type' instead.";
        exit 1;
    fi;
    if [[ ! -d $input_dir ]]; then
        echo "ERROR: Value of 'input_dir' must be a directory that exists!";
        echo "Got $input_dir";
        exit 1;
    fi;
    if [[ -z "${vars_regex// }" ]]; then
        echo "ERROR: Value of 'vars_regex' must be non-empty!";
        exit 1;
    fi;
    if [ ! -z $start_year ] && [ -z $end_year ]; then
        echo "ERROR: Value of 'end_year' must be provided if 'start_year'!";
        echo "Got start_year=$start_year and end_year=$end_year";
        exit 1;
    fi;
    if [ ! -z $end_year ] && [ ! -z $start_year ]; then
        if [ $end_year -lt $start_year ]; then
            echo "ERROR: Value of 'end_year' must be >= 'start_year'!";
            echo "Got end_year < start_year ==> $end_year < $start_year";
            exit 1;
        fi;
    fi;
    if [[ $start_month -lt 1 || $start_month -gt 12 || $end_month -lt 1 || $end_month -gt 12 ]]; then
        echo "ERROR: Value of 'start_month' and 'end_month' must be in [1..12]!";
        echo "Got $start_month and $end_month";
        exit 1;
    fi;
    if [ $end_month -lt $start_month ]; then
        echo "ERROR: Value of 'end_month' must be >= 'start_month'!";
        echo "Got end_month < start_month ==> $end_month < $start_month";
        exit 1;
    fi;
    if [[ $debug != 0 && $debug != 1 ]]; then
        echo "ERROR: Value of 'debug' must be 0 or 1!";
        echo "Got $debug";
        exit 1;
    fi;
    if [ -z $start_year ]; then
        fname_regex="^${exp_name}.*\.nc$";
    else
        years_regex=$(seq -s '|' $start_year $end_year);
        months_regex=$(printf "%02d|" $(seq $start_month $end_month) | sed 's/|$//');
        fname_regex="^${exp_name}.*_($years_regex)($months_regex).*\.nc$";
    fi;
    declare -A vars_set;
    for f in $(ls $input_dir);
    do
        fullpath="${input_dir}/${f}";
        if [[ $f =~ $fname_regex ]]; then
            valid_exclude=1;
            for exclude_substring in $exclude_substrings;
            do
                if [[ $f == *$exclude_substring* ]]; then
                    valid_exclude=0;
                    break;
                fi;
            done;
            if [[ ! -z $include_substrings ]]; then
                valid_include=0;
            else
                valid_include=1;
            fi;
            if [ $valid_exclude == 1 ] && [ $valid_include == 0 ]; then
                for include_substring in $include_substrings;
                do
                    if [[ $f == *$include_substring* ]]; then
                        valid_include=1;
                        break;
                    fi;
                done;
            fi;
            if [ $valid_exclude == 1 ] && [ $valid_include == 1 ]; then
                ncheader=$(ncdump -h $fullpath);
                tmp_vars_in_nc=$(echo $ncheader | grep -o -w -E "$vars_regex" | awk '!seen[$0]++' | tr '\n' ' ');
                vars_in_nc=$tmp_vars_in_nc;
                for var in $tmp_vars_in_nc;
                do
                    if [[ ! -v vars_set[$var] ]]; then
                        vars_set[$var]=1;
                    else
                        remove_exp_name_sedexp="s:${exp_name}_::g";
                        data_spec_with_timestamp=$(echo $f | sed "$remove_exp_name_sedexp");
                        data_spec="${data_spec_with_timestamp%_*}";
                        append_data_spec_to_var_sedexp="s:${var}:${var}_${data_spec}:g";
                        vars_in_nc=$(echo $vars_in_nc | sed -e "$append_data_spec_to_var_sedexp");
                    fi;
                done;
                vars_in_nc=$(echo $vars_in_nc | tr ' ' ',');
                if [[ -n $vars_in_nc ]]; then
                    vars_in_nc=${vars_in_nc%%,};
                    if [[ ! -v return_var_to_paths[$vars_in_nc] ]]; then
                        return_var_to_paths[$vars_in_nc]="$fullpath";
                    else
                        return_var_to_paths[$vars_in_nc]+=" $fullpath";
                    fi;
                fi;
            fi;
        fi;
    done;
    if [[ $debug == 1 ]]; then
        echo "key";
        echo "value";
        echo "----------------------------------------------------------------";
        for key in ${!return_var_to_paths[@]};
        do
            echo $key;
            echo ${return_var_to_paths[$key]};
            echo "----------------------------------------------------------------";
        done;
    fi
}
```

#  ops.sh

Collection of cdo operator functions (should be exported if parallel)

## Overview

Perform cdo monmean operation on selected variables in infile and 
write result.

## Index

* [cdo_monmean_vars_in_file](#cdomonmeanvarsinfile)
* [cdo_ymonmean_var_in_files](#cdoymonmeanvarinfiles)
* [cdo_remap](#cdoremap)

### cdo_monmean_vars_in_file

Perform cdo monmean operation on selected variables in infile and 
write result.

#### Arguments

* **$1** (string): String where commas separate variables to be selected by `cdo -selvar`.
* **$2** (string): Path to file on which `cdo monmean` operation is performed.
* **$3** (string): Directory to write outputs of `cdo monmean` operation TODO with format
* **$4** (string): (OPTIONAL) Job id.

### cdo_ymonmean_var_in_files

Perform cdo ymonmean on selected variable and concatenation on 
desired infiles.

#### Arguments

* **$1** (string): Variable to perform cdo operation on.
* **$2** (string): String with spaces separating infile paths to concatenate OR file paths
* **$3** (string): Directory result of operation will be written TODO with format?
* **$4** (string): String for name of experiment of interest. Used for naming.
* **$5** (string): (OPTIONAL) Job id.
* **$6** (string): (OPTIONAL) String that represents the separator between file paths

#### See also

* [cdo tutorial, cdo -cat](https://code.mpimet.mpg.de/projects/cdo/embedded/cdo.pdf)

### cdo_remap

Perform cdo remap operation using grid file on desired infile.

#### Arguments

* **$1** (string): The path to the grid file used for remapping.
* **$2** (string): The path to the netcdf file to remap.
* **$3** (string): Directory to write remapped outfile to TODO format.
* **$4** (string): (OPTIONAL) Job id.

## Source Code

```bash
cdo_monmean_vars_in_file () 
{ 
    local vars="$1";
    local infile="$2";
    local outdir="$3";
    local jid=${4:-0};
    if [[ -z "${vars// }" ]]; then
        echo "ERROR: Value of 'vars' cannot be an empty string!";
        exit 1;
    fi;
    if [[ ! -f $infile ]]; then
        echo "ERROR: Value of 'infile' is a path that does not exist!";
        echo "Got $infile";
        exit 1;
    fi;
    if [[ ! -d $outdir ]]; then
        echo "ERROR: Value of 'outdir' is a directory that does not exist!";
        echo "Got $outdir";
        exit 1;
    fi;
    local file_name=$(basename "$infile" .nc);
    local outfile="${file_name}_monmean.nc";
    local outpath="${outdir}/${outfile}";
    if [[ ! -e $outpath ]]; then
        [[ $jid != 0 ]] && echo job : $jid, infile: $infile;
        cdo monmean -selvar,$vars "$infile" "$outpath";
    fi
}
```

```bash
cdo_ymonmean_var_in_files () 
{ 
    local var=$1;
    local infiles=$2;
    local outdir=$3;
    local exp_name=$4;
    local jid=${5:-0};
    local sep=${6:-""};
    local outpath="${outdir}/${exp_name}_${var}_concatenated.nc";
    if [[ $jid != 0 && -z "${sep// }" ]]; then
        echo "ERROR: Value of 'sep' cannot be an empty string for a parallel job!";
        exit 1;
    fi;
    if [[ $jid != 0 ]]; then
        local sedexp="s:${sep}: :g";
        infiles=$(echo $infiles | sed -e "$sedexp");
    fi;
    for infile in $infiles;
    do
        if [[ ! -f $infile ]]; then
            echo "ERROR: Value of 'infile' is a path in 'infiles' (argument \$2)" "that does not exist!";
            echo "Got $infile";
            exit 1;
        fi;
    done;
    if [[ ! -d $outdir ]]; then
        echo "ERROR: Value of 'outdir' directory does not exist!";
        echo "Got $outdir";
        exit 1;
    fi;
    if [[ -z "${exp_name// }" ]]; then
        echo "ERROR: Value of 'exp_name' cannot be an empty string!";
        exit 1;
    fi;
    if [[ ! -e $outpath ]]; then
        [ $jid != 0 ] && echo job : $jid, var : $var, outpath : $outpath;
        cdo ymonmean -selvar,$var -cat $infiles $outpath;
    fi
}
```

```bash
cdo_remap () 
{ 
    local grid_file=$1;
    local infile=$2;
    local outdir=$3;
    local jid=${4:-0};
    local infile_fname=$(basename $infile .nc);
    local outpath="${outdir}/${infile_fname}_RM.nc";
    if [[ ! -f ${grid_file%%:*} ]]; then
        echo "ERROR: grid_file does not exist!";
        echo "Got $grid_file, checked path ${grid_file%%:*}";
        exit 1;
    fi;
    if [[ ! -f $infile ]]; then
        echo "ERROR: infile does not exist!";
        echo "Got $infile";
        exit 1;
    fi;
    if [[ ! -d $outdir ]]; then
        echo "ERROR: outdir does not exist!";
        echo "Got $outdir";
        exit 1;
    fi;
    if [[ ! -e $outpath ]]; then
        [[ $jid != 0 ]] && echo job : $jid, var : $var, outpath : $outpath;
        cdo remapdis,n45 -setgrid,$grid_file $infile $outpath;
    fi
}
```

