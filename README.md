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

### Probe a function:

    probe PROBEPOINT [, PROBEPOINT] { [STMT ...] }

    probe kernel.function("sys_mkdir").call { log ("enter") }
    probe kernel.function("sys_mkdir").return { log ("exit") }

### User space probing:

    probe process(path).function(function_name)
    {
    }

### Filter by target (PID):

    my_pid = pid()
    if (my_pid == target()) {
    }

    $ sudo stap -x pid ./my.stp

### Define a function:

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

### Calling a function:

    probe begin {
        for (i = 0; i < 50; i++)
            if (isprime (i)) printf("%d\n", i)
        exit()
    }

Note: function can be called recursively.

### Variable:

Target variables (or "context variables") are variables defined in the source
code at that location. They are presented to the script as variables whose names
are prefixed with a dollar sign ($).

#### Target variable:

    $my_var

#### SystemTap plain local variable:

    a = 1

Note: all SystemTap variables have "long" type

#### Variable types:

Scalar variables are implicitly typed as either string or integer. Associative
arrays also have a string or integer value, and a tuple of strings or integers
serves as a key. Arrays must be declared as global. Local arrays are not allowed.

### Type casting:

#### C definitions:

    typedef struct {
        int y;
    } inner_struct;

    typedef struct {
        int x;
        inner_struct *next;
        inner_struct  one;
    } outer_struct;

#### Type casting in SystemTap:

    b = @cast(a, "inner_struct")
    b = @cast(a, "inner_struct")->x
    b = @cast($my_var, "outer_struct")->next
    b = &@cast($my_var, "outer_struct")->one

### Print

    printf("%d %p\n", num, pointer)

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

Chapter 4+ of https://sourceware.org/systemtap/langref.pdf
