# SystemTap Cheat Sheet

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

    function isprime (x) {
        if (x < 2) return 0
        for (i = 2; i < x; i++) {
            if (x % i == 0) return 0
            if (i * i > x) break
        }
        return 1
    }

### Calling a function:

    probe begin {
        for (i = 0; i < 50; i++)
            if (isprime (i)) printf("%d\n", i)
        exit()
    }

### Variable:

#### function local variable:

    $my_var

#### SystemTap variable:

    a = 1

Note: all SystemTap variables have "long" type

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
