---
in_progress: true
---

Oil Language Idioms
===================

This is an informal, lightly-organized list of recommended idioms for the [Oil
language]($xref:oil-language).  Use these when you don't care about
compatibility with other shells.

TODO: Go through more of the [Pure Bash
Bible](https://github.com/dylanaraps/pure-bash-bible).  Oil provides
alternatives for such quirky syntax.

[QSN]: qsn.html

<!-- cmark.py expands this -->
<div id="toc">
</div>

## Use [Simple Word Evaluation](simple-word-eval.html) to Avoid "Quoting Hell"

### Substitute Variables

No:

    local x='my song.mp3'
    ls "$x"  # quotes required to avoid mangling

Yes:

    var x = 'my song.mp3'
    ls $x  # no quotes needed

### Splice Arrays

No:

    local myflags=( --all --long )
    ls "${myflags[@]}" "$@"

Yes:

    var myflags = %( --all --long )
    ls @myflags @ARGV

### Explicit Elision of Empty Elements

No:

    local maybe_empty=''
    cp $maybe_empty other_file $dest  # omitted if empty

Yes:

    var e = ''
    cp @maybe(e) other_file $dest

### Explicit Splitting

No:

    local packages='python-dev gawk'
    apt install $packages

Yes:

    var packages = 'python-dev gawk'
    apt install @split(packages)

Even better:

    var packages = %(python-dev gawk)  # array literal
    apt install @packages              # splice array

### Iterate a Number of Times (Split Command Sub)

No:

    local n=3
    for x in $(seq $n); do  # No implicit splitting of unquoted words in Oil
      echo $x
    done

Yes:

    var n = 3
    for x in @(seq $n) {   # Explicit splitting
      echo $x
    }

Note that `{1..3}` works in bash and Oil, but the numbers must be constant.


### Explicit Dynamic Globbing

No:

    local pat='*.py'
    echo $pat


Yes:

    var pat = '*.py'
    echo @glob(pat)


## Avoid Ad Hoc Parsing and Splitting

In other words, avoid *groveling through backslashes and spaces* in shell, or
limit it to "the edges" of your programs.

Take advantage of the the invariants that the [IO builtins](io-builtins.html)
respect.

TODO: Implement and test these.

    write --qsn  # also -q
    read --qsn   # also -q

    read --line --qsn :myline     # read a single line
    read --lines --qsn :myarray   # read many liens
    read --all :mystr  # slurp whole thing, not compatibl with qsn

### Wrap and Adapt External Tools

Parse their output and emit [QSN][] and QTSV.

These can be one-off, "bespoke" wrappers in your program, or maintained
programs.  Use the `proc` construct and `flagspec`!

Example: [uxy](https://github.com/sustrik/uxy) wrappers.

TODO: Examples written in Oil and in other languages.

### Patch Existing Tools

Enhance GNU grep, etc. to emit [QSN][] and QTSV.

<!--

  ls -q and -Q already exist, but --qsn is probably fine
  or --qtsv
-->

### Or Write Your Own Structured Versions

For example, you can write a structured subset of `ls` in Python with little
effort.

## The `write` Builtin Is Simpler Than `printf` and `echo`

### Write an Arbitrary Line

No:

    printf '%s\n' "$mystr"

Yes:

    write -- $mystr

The `write` builtin accepts `--` so it doesn't confuse flags and args.

### Write Without a Newline

No:

    echo -n "$mystr"  # breaks if mystr is -e

Yes:

    write --end '' -- $mystr
    write -n -- $mystr  # -n is an alias for --end ''

### Write an Array of Lines

    var myarray = %(one two three)
    write -- @myarray

## Use Long Flags on the `read` builtin

### Read A Line

TODO: implement this.

No:

    read line     # Bad because it mangles your backslashes!
    read -r line  # Better, but easy to forget

Yes:

    read --line  # ???

### Read a Whole File

TODO: figure this out.

No:

    mapfile -d ''
    read -d ''

Yes:

    read --all
    slurp ?


## Use Blocks To Save and Restore Context

### Do Something In Another Directory

No:

    ( cd /tmp; echo $PWD )  # subshell is unnecessary (and limited)

No:

    pushd /tmp
    echo $PWD
    popd

Yes:

    cd /tmp {
      echo $PWD
    }

### Temporarily Set Shell Options

TODO: Implement this.

No:

    set +o errexit
    myfunc  # without error checking
    set -o errexit

Yes:

    shopt --unset errexit {
      myfunc
    }

### Use the `forkwait` builtin for Subshells, not `()`

TODO: Implement this.

No:

    ( not_mutated=foo )
    echo $not_mutated

Yes:

    forkwait {
      setvar not_mutated = 'foo'
    }
    echo $not_mutated

### Use the `fork` builtin for async, not `&`

TODO: Implement this.

No:

    myproc &

    { sleep 1; echo one; sleep 2; } &

Yes:

    fork myproc

    fork { sleep 1; echo one; sleep 2 }

## Use Procs (Better Shell Functions)

### Named Parameters

No:

    f() {
      local src=$1
      local dest=${2:-/tmp}

      cp "$src" "$dest"
    }

Yes:

    proc f(src, dest='/tmp') {   # Python-like default values
      cp $src $dest
    }

### Variable Number of Arguments

TODO: Test this out.

No:

    f() {
      local first=$1
      shift

      echo $first
      echo "$@"
    }

Yes:

    proc f(first, @rest) {  # @ means "the rest of the arguments"
      write -- $first
      write -- @rest        # @ means "splice this array"
    }

### "Out" Params as Return Values

TODO: Test this out.

No:

    f() {
      local in=$1
      local -n out=$2

      out=PREFIX-$in
    }

    myvar='zzz'
    f zzz myvar  # assigns myvar to 'PREFIX-zzz'


Yes:

    proc f(in, :out) {  # : means accept a string "reference"
      setref out = "PREFIX-$in"
    }

    var myvar = 'zzz'
    f zzz :myvar        # : means pass a string "reference" (optional)

## Curly Braces Fix Semantic Problems

### Procs Don't Have Dynamic Scope

Shell functions can access variables in their caller:

    f() { echo $var_in_g; }
    g() { var_in_g=foo; }
    g

In Oil, you have to pass params explicitly:

    proc f { echo $var_in_g }  # error

### If and errexit

TODO: Implement this.


Bug in POSIX shell:

    if myfunc; then
      echo hi
    fi

Workaround suggested for POSIX shell:

    if $0 myfunc; then
      echo hi
    fi

    "$@"


Oil extension, without an extra process:

    if invoke myfunc; then
      echo hi
    fi

Even better:

    if myfunc {  # implicit 'invoke', equivalent to the above
      echo hi
    }


Note: `&&` and `||` and `!` still require explicit invoke.

No:

    myfunc || fail
    myfunc && echo 'success'
    ! myfunc

Yes:

    invoke myfunc || fail
    invoke myfunc && echo 'success'
    ! invoke myfunc


This explicit syntax avoids breaking POSIX shell.  You have to opt in to the
better behavior..

## Use Oil Expressions and Assignments

### Initialize and Assign Strings and Integers

No:

    local mystr=foo
    mystr='new value'

    local myint=42  # still a string in shell

Yes:

    var mystr = 'foo'
    setvar mystr = 'new value'

    var myint = 42  # a real integer

### Expressions on Integers

No:

    x=$(( 1 + 2*3 ))
    (( x = 1 + 2*3 ))

Yes:

    setvar x = 1 + 2*3

### Mutate Integers

No:

    (( i ++ ))  # interacts poorly with errexit
    i=$(( i+1 ))

Yes:

    setvar i += 1  # like Python, with a keyword

### Initialize and Assign Arrays

No:

    local -a myarray=(one two three)
    myarray[3]='THREE'

    local -A myassoc=(['key']=value ['k2']=v2)
    myassoc['key']=V

Yes:

    var myarray = %(one two three)
    setvar myarray[3] = 'THREE'

    var myassoc = %{key: 'value', k2: 'v2'}  # keys don't need to be quoted
    setvar myassoc['key'] = 'V'

Container literals start with the `%` sigil.  (TODO: Implement this.  It's `@`
right now.)

### Expressions on Arrays

No:

    local x=${a[i-1]}
    x=${a[i]}

    local y=${A['key']}

Yes:

    var x = a[i-1]
    setvar x = a[i]

    var y = A['key']

### Conditions and Comparisons

No:

    if (( x > 0 )); then
      echo positive
    fi

Yes:

    if (x > 0) {
      echo 'positive'
    }

### Substituting Expressions in Words

No:

    echo result=$((1 + a[i] * 3))  # C-like arithmetic

Yes:

    echo result=$[1 + a[i] * 3]    # Arbitrary Oil expressions

    # Possible, but a local var might be more readable
    echo flag=$['1' if x else '0']


## Use [Egg Expressions](eggex.html) instead of Regexes

### Test for a Match

No:

    local pat='[[:digit:]]+'
    if [[ $x =~ $pat ]]; then
      echo 'number'
    fi

Yes:

    if (x ~ /digit+/) {
      echo 'number'
    }

Or extract the pattern:

    var pat = / digit+ /
    if (x ~ pat) {
      echo 'number'
    }

### Extract Submatches

TODO: `BASH_REMATCH` alternative.

## Glob Matching

No:

    if [[ $x == *.py ]]; then
      echo Python
    fi

TODO: Implement the `~~` operator.

Yes:

    if (x ~~ '*.py') {
      echo 'Python'
    }


No:

    case $x in
      *.py)
        echo Python
        ;;
      *.sh)
        echo Shell
        ;;
    esac

Yes (purely a style preference):

    case $x {
      (*.py)
        echo 'Python'
        ;;
      (*.sh)
        echo 'Shell'
        ;;
    }

## TODO

### Consider Using `--long-flags` for builtins

TODO: implement this.

Easier to write:

    test -d /tmp
    test -d / -a -f /vmlinuz

    shopt -u extglob

Easier to read:

    test --dir /tmp
    test --dir / && test --file /vmlinuz

    shopt --unset extglob

Style note: Prefer `test` to `[`, because idiomatic Oil code doesn't use
"puns".


### Source Files and Namespaces

TODO

<!--

TODO: The `use` builtin (or keyword?) should enable this.  And there should be
a static variant for bundling.

Hypothetical example:

    use lib/html.sh  # 'html' is in the 'proc' namespace

    html header
    html footer

-->

## Related Documents

- [Shell Language Deprecations](deprecations.html)
