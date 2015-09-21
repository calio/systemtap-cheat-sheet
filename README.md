# SystemTap Cheat Sheet

### Synopsis

    global odds, evens

    probe begin {
        # "no" and "ne" are local integers
        for (i = 0; i < 10; i++) {
            if (i % 2) odds [no++] = i
            else evens [ne++] = i
        }

        delete odds[2]
        delete evens[3]
        exit()
    }

    probe end {
        foreach (x+ in odds)
            printf ("odds[%d] = %d", x, odds[x])
        foreach (x in evens-)
            printf ("evens[%d] = %d", x, evens[x])
    }

Note: all variable types are inferred, and that all locals and globals are
initialized. Integers are set to 0 and strings are set to the empty string.

### Probe a function

    probe PROBEPOINT [, PROBEPOINT] { [STMT ...] }

    probe kernel.function("sys_mkdir").call { log ("enter") }
    probe kernel.function("sys_mkdir").return { log ("exit") }

### Filter by target (PID)

    my_pid = pid()
    if (my_pid == target()) {
    }

    $ sudo stap -x pid ./my.stp

### Define a function

Syntax:

    function <name>[:<type>] ( <arg1>[:<type>], ... ) { <stmts> }


    function isprime (x) {
        if (x < 2) return 0
        for (i = 2; i < x; i++) {
            if (x % i == 0) return 0
            if (i * i > x) break
        }
        return 1
    }

C code can be embedded as embedded C functions. See "Embedded C functions" in
*SystemTap Language Reference* for more details.

### Calling a function

    probe begin {
        for (i = 0; i < 50; i++)
            if (isprime (i)) printf("%d\n", i)
        exit()
    }

Note: function can be called recursively.

### Variable

Target variables (or "context variables") are variables defined in the source
code at that location. They are presented to the script as variables whose names
are prefixed with a dollar sign ($).

#### Target variable

    $my_var

#### SystemTap plain local variable

    a = 1

#### Syntax

    /* in-scope variable var */
    $var
    /* alternative syntax for $varname */
    @var("varname")
    /* the global (either file local or external) variable varname defined when the file src/file.c was compiled */
    @var("varname@src/file.c")
    /* traverses a structure’s field */
    $var->field
    @var("var@file.c")->field
    /* indexes into an array */
    $var[N]
    @var("var@file.c")[N]
    /* get the address of a variable as a long */
    &$var
    &@var("var@file.c")
    /* provide the address of a particular field or an element in an array */
    &var->field
    &@var("var@file.c")[N]
    /* a string that only includes the values of all basic type values of fields of the variable structure type but not any nested complex type values */
    $var$
    /* a string that also includes all values of nested data types */
    @var("var")$$

    $$vars expands to a character string that is equivalent to sprintf("parm1=%x ... parmN=%x var1=%x ... varN=%x", $parm1, ..., $parmN, $var1, ..., $varN)
    $$locals expands to a character string that is equivalent to sprintf("var1=%x ... varN=%x", $var1, ..., $varN)
    $$parms expands to a character string that is equivalent to sprintf("parm1=%x ... parmN=%x", $parm1, ..., $parmN)

#### Variable types

Scalar variables are implicitly typed as either string or integer. Associative
arrays also have a string or integer value, and a tuple of strings or integers
serves as a key. Arrays must be declared as global. Local arrays are not allowed.

### Type casting

#### C definitions

    typedef struct {
        int y;
    } inner_struct;

    typedef struct {
        int x;
        inner_struct *next;
        inner_struct  one;
    } outer_struct;

#### Type casting in SystemTap

    b = @cast(a, "inner_struct")
    b = @cast(a, "inner_struct")->x
    b = @cast($my_var, "outer_struct")->next
    b = &@cast($my_var, "outer_struct")->one

### Print

    printf("%d %p\n", num, pointer)

### Probes

#### Examples probe points

    kernel.function("foo")
    kernel.function("foo").return
    module{"ext3"}.function("ext3_*")
    kernel.function("no_such_function") ?  /* Optional probe points */
    syscall.*
    end
    timer.ms(5000)

#### Built-in probe point types (DWARF probes)

    kernel.function(PATTERN)
    kernel.function(PATTERN).call
    kernel.function(PATTERN).return
    kernel.function(PATTERN).return.maxactive(VALUE)
    kernel.function(PATTERN).inline  /* include only instances of inlined functions */
    kernel.function(PATTERN).label(LPATTERN)
    module(MPATTERN).function(PATTERN)
    module(MPATTERN).function(PATTERN).call /* select opposite result of inline */
    module(MPATTERN).function(PATTERN).exported /* include only exported functions */
    module(MPATTERN).function(PATTERN).return.maxactive(VALUE)
    module(MPATTERN).function(PATTERN).inline
    kernel.statement(PATTERN)
    kernel.statement(ADDRESS).absolute
    module(MPATTERN).statement(PATTERN)

Note:

* Inline functions do not have an identifiable return point, so .return is not supported on .inline probes.
* MPATTERN: string literal that identifies the loaded kernel module of interest.
* LPATTERN: source program label.
* PATTERN: string literal that identifies a point in the program.

        <function> [@ <abs_path>|<rel_path>] [:<abs_line_num> | +<rel_line_num> | :* | :<x>-<y> ]
        | address
* Examples:

        process("/usr/local/sbin/nginx").statement("*@ngx_http_gunzip_filter_module.c:215")
        process("/usr/local/sbin/nginx").statement("ngx_http_gunzip_force_header_filter@ngx_http_gunzip_force_filter_module.c+7")

### User space probing

Examples:

    probe process(path).function(function_name) { }
    probe process("/lib64/libc-2.8.so").function("....") { ... }
    /* qualify a probe point to a location in a library required by a particular process */
    probe process("...").library("...").function("....") { ... }
    /* probe the functions in the program linkage table of a particular process */
    probe process("...").plt { ... }
    /* also add the program linkage tables of libraries required by that process */
    probe process("...").plt process("...").library("...").plt { ... }

Syntax:

    process.begin
    process("PATH").begin
    process(PID).begin
    process.thread.begin
    process("PATH").thread.begin
    process(PID).thread.begin
    process.end
    process("PATH").end
    process(PID).end
    process.thread.end
    process("PATH").thread.end
    process(PID).thread.end

    /* $syscall: system call number, $arg[1-6]: first six arguments of the system, $return: The return value of the system call */
    process.syscall
    process("PATH").syscall
    process(PID).syscall
    process.syscall.return
    process("PATH").syscall.return
    process(PID).syscall.return

    /* called for every single-stepped instruction, block-stepped instruction */
    process("PATH").insn process(PID).insn
    process("PATH").insn.block process(PID).insn.block

    /* syscall probes */
    /* argstr: A pretty-printed form of the entire argument list, without parentheses */
    /* name: The name of the system call */
    /* retstr: For return probes, a pretty-printed form of the system call result */
    syscall.NAME
    syscall.NAME.return

    /* runs every N jiffies */
    timer.jiffies(N)
    /* a linearly distributed random value in the range [-M ... +M] is added to N every time the handler executes */
    timer.jiffies(N).randomize(M)

    /* N and M are specified in milliseconds */
    timer.ms(N)
    timer.ms(N).randomize(M)
    /* full options for units are seconds (s or sec), milliseconds (ms or msec), microseconds (us or usec), nanoseconds (ns or nsec), and hertz (hz) */
    timer.ns(N)
    /* Randomization is not supported for hertz timers */
    timer.hz(99)

    /* all CPUs at each system tick */
    timer.profile

    /* special probe points */
    begin
    end
    error  /* probe handler runs when the session ends if an error occurred */
    never  /* grammar check, never runs */

#### Wildcarded file names, function names

    kernel.syscall.*
    kernel.function("sys_*)

### Probe aliases

Prologue-style aliases

    probe <alias> = <probepoint> { <prologue_stmts> }

Epilogue-style aliases

    probe <alias> += <probepoint> { <epilogue_stmts> }

Execution order

    prologue_stmts  ->  user statement block code  ->  epilogue_stmts

### Limits

Limits are set by -D option:

    sudo stap -DMAXACTION=20000 -v -x 32255 ./keepalive1.stp

Available limits are:

* MAXNESTING – The maximum number of recursive function call levels. The default is 10.
* MAXSTRINGLEN – The maximum length of strings. The default is 256 bytes for 32 bit machines and 512 bytes for all other machines.
* MAXTRYLOCK – The maximum number of iterations to wait for locks on global variables before declaring possible deadlock and skipping the probe. The default is 1000.
* MAXACTION – The maximum number of statements to execute during any single probe hit. The default is 1000.
* MAXMAPENTRIES – The maximum number of rows in an array if the array size is not specified explicitly when declared. The default is 2048.
* MAXERRORS – The maximum number of soft errors before an exit is triggered. The default is 0.
* MAXSKIPPED – The maximum number of skipped reentrant probes before an exit is triggered. The default is 100.
* MINSTACKSPACE – The minimum number of free kernel stack bytes required in order to run a probe handler. This number should be large enough for the probe handler’s own needs, plus a safety margin. The default is 1024.


### TODO

Chapter 5+ of https://sourceware.org/systemtap/langref.pdf
