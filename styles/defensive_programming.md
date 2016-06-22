# Defensive Programming

By M. Tim Jones, February 01, 2005

* * *
Defensive programming is a practice where developers anticipate failures in
their code, then add supporting code to detect, isolate, and in some cases,lis
recover from the anticipated failure. It can assist us by targeting defects in
the source where they most commonly occur. By targeting defects in this way,
we can find them sooner, leading to more stable and less exploitable software
in a shorter period.

Defensive programming operates on the premise that bugs exist in our code, so
we should identify ways to more easily identify or counteract these problems.
For example, watchdog timers are a common component of embedded systems that
are designed to restart software and/or hardware after identifying anomalous
behavior. Checksums are common elements of packets transferred between nodes
to identify errors induced during communication. These are two examples of
defensive programming that are now commonplace.

Defensive programming covers a large variety of techniques. In this article, I
look at a number of aspects of defensive programming, including coding
policies, code instrumentation, and tools.

### Coding Policies

The first set of defensive measures that I discuss are what can be called
"coding policies." These refer to the manner in which you develop software,
with the intent of avoiding common defects found in software. A side effect of
these policies is that they not only minimize the types of common errors that
occur, but they can also help to isolate them. Isolating defects helps in the
debugging process because otherwise, faults can manifest themselves elsewhere,
making their source difficult to identify.

One of the most common errors made in software is ignoring the return status
of functions. Function return status provides an indication of the success or
failure of a function and ignoring it assumes that the function completed
successfully. This may be true much of the time, but ignoring this status can
then be difficult to debug once the function does fail. These types of
failures commonly manifest themselves later in ways that may not associate
with the original function whose status was ignored. So, what's the solution?
Always capture and check the return status of functions.

Another common error that leads to interesting behavior is the absence of
variable initialization. When a variable is created that is not initialized,
its value is whatever value is contained in that memory location. This can
lead to anomalous operation, but may not lead to a crash. The solution is
simply to initialize all local and global variables upon declaration. Good C
compilers also provide warnings when uninitialized variables are used. A
critical policy that helps protect against buffer overflow (a common technique
that can give intruders the ability to execute arbitrary code) is mandated use
of **strn** functions rather than the old **str** functions (such as
**strncpy** instead of **strcpy**). Consider this code, which is unsafe
because of the **strcpy** function:

    
    
    int func( char *userdata ) {
       char myarray[ MAX_LENGTH ];
       strcpy( myarray, userdata );
       ...
    }
    
    

Not knowing what the user passed in, the assumption is being made that it
won't exceed **MAX_LENGTH**. If the length does exceed **MAX_LENGTH**, then
the stack is compromised, allowing the return address to be manipulated. By
inserting a new return address, control can be passed to malicious code,
giving intruders control over the computer. This type of problem exists
everywhere (just look at the Microsoft Security Advisories), but many cases
can be easily solved by using the **strn **versions. This code is safer
because of the **strncpy** function:

    
    
    int func( char *userdata ) {
       char myarray[ MAX_LENGTH+1 ];
       strncpy( myarray, userdata,
         MAX_LENGTH );
       myarray[MAX_LENGTH] = 0;
       ...
    }
    
    

By using **strncpy**, the memory space cannot be exploited by the buffer
overrun. If the number of characters copied equals the **MAX_LENGTH**, the
string is unterminated (no trailing **NULL** is inserted). Therefore, if the
array is defined as size "+1," then a slot is automatically reserved for the
terminating **NULL** within the string for this case. This policy also applies
to the other Standard C Library functions such as **strncat**, **strncmp**,
**strncasecmp**, **snprintf**, and **fgets**.

Another important policy to follow is ensuring that all conditional test cases
are covered. Consider this code, which demonstrates incomplete conditionals:

    
    
    float bias = 0;
     ...
    if ( errorSignal > 50.0 ) bias = 2.0;
    else if ( errorSignal > 10.0 )
      bias = 4.0;
    delta = signal / bias;
    
    

When **errorSignal** lies under 10.0, a failure is imminent with a divide by
zero. In this case, an **else bias = 1.0;** should be present. Even if the
anticipated final clause is never expected, asserting at this point could help
identify the potential error. This same issue applies to **switch** statements
where a **default** case should be provided to handle similar issues.

Not clearing a reference to freed memory is not only a common defect, but is
also difficult to debug because its behavior is often not reproducible.
Tracing **NULL** pointer accesses can be easier; therefore, one trick is to
convert unused pointers to **NULL** pointers:

    
    
    CONTEXT_TYPE *contextPtr;
     ...
    free( contextPtr );
    contextPtr = (CONTEXT_TYPE *)NULL;
    
    

In subsequent code, asserts detect that the **contextPtr** has been nulled-
out, or a hardware-based memory access violation identifies the error. Not
having cleared this pointer could have resulted in difficult-to-debug
behavior.

While we all code in different ways, we tend to repeat some of the same types
of errors. Therefore, we should know ourselves and our most common errors and
either deal with these problems up front (while we code), or instrument in
ways that help us to find these errors.

A final topic in coding policy is adequate source-based documentation.
Commenting code not only helps us to remember why we wrote our code the way
that we did, but also helps others to maintain it. Maintenance costs can be
directly linked to how well the maintainer understands the code. In many
cases, if a solid understanding does not exist, new defects can be introduced,
leading to even higher maintenance costs. A potential problem with commenting
is that too much documentation exists in the source, or that the comments are
inaccurate (source was changed, comments were not). The solution is to include
comments whenever something needs to be pointed out that cannot be "read" from
the code.

### Code Instrumentation

The second set of defensive measures focuses on ways to instrument source to
help identify and isolate bugs. In many cases, these measures are removed or
compiled away once the verification process is complete. The concept of Design
by Contract is a useful construct to stop defects at their source. This
feature lets you specify the preconditions that must be met for the function
to execute properly, and the postconditions that must be met for the function
to properly complete (a contract between the caller and the callee). Some
languages (such as Eiffel [1]) include direct language support for software
contracts.

You can simulate this type of the software contract in C by simply providing a
set of macroized preconditions and postconditions for a function. Consider
this code, which illustrates pre- and postconditions with **assert()**:

    
    
    #define PRECONDITION( x ) assert( (x) )
    #define POSTCONDITION( x ) assert( (x) )
    int getPacket( Packet_T *newPacket )
    {
       int packetSize;
       PRECONDITION( (int)newPacket );
       packetSize = retrieveLastPacket( newPacket );
       POSTCONDITION( packetSize );
       return packetSize;
    }
    
    

The **assert** function takes an expression as an argument, and if the
expression evaluates to **false**, then an error message is emitted. In this
example, you check the input to the function using the **PRECONDITION** macro,
then verify that our return value is correct using the **POSTCONDITION**
macro. To disable these checks from occurring in the code, you simply specify
a macro define to the compiler (**-DNDEBUG**).

Liberal use of **assert()** is also useful within functions to ensure that
they are internally consistent (also known as sanity checking). A common use
is to test memory allocation, such as this code (used when a memory allocation
failure constitutes a critical failure):

    
    
    memPtr = malloc( sizeof(myStruct) );
    assert( (int)memPtr );
    
    

For applications that must withstand memory allocation failures, building
error-inducing functions can help identify where memory failures are not
properly managed. Consider this error-inducing **malloc()** function:

    
    
    #include <stdlib.h>
    #define FAILURE_EVERY 3
    void *mmalloc( size_t size ) {
       static int failcount = FAILURE_EVERY;
       void *retp = (void *)0;
       if ( —failcount ) retp = malloc( size );
       else failcount = FAILURE_COUNT;
       return ( retp ) ;
    }
    
    

The function **mmalloc** returns a **NULL** pointer every third allocation (as
defined by **FAILURE_EVERY**).

A good temporary measure to ensure the quality of firmware is the inclusion of
regression tests that are executed at startup or in a special test mode. The
stack component provides the ability to push an integer onto the stack and pop
an integer off the stack. The return value of the stack function provides the
error status. The function argument represents the pushed or popped value.
[Listing 1](#listing-1) provides an example
in-code test of the stack component. In the first test, a pop operation is
attempted on an empty stack that should elicit an error response (Test 1). If
an error response is not detected, the control is aborted using the break
statement to exit the single loop construct. Otherwise, control continues to
the next test. In this test (Test 2), I push an item onto the stack, then pop
it back off. Both operations are tested, though in the second operation, I
also ensure that the value pushed onto the stack is what was popped off.

More tests would be added into this regression example to fully verify the
functionality, or in response to a new failure found in the component.

From an embedded-systems perspective, filling memory with a known pattern is a
well-known technique to identify uninitialized pointers as well as rogue
pointers. Additionally, initializing the stack(s) with known values permits
identification of stack overflow (in addition to worst-case sizing).

### Tools

The final defensive measure I examine is that of tools. Software tools can
help us to produce higher quality code, but only if we use them. Compilers
typically include a warning option to enable the emission of potential errors
that are encountered during the compilation process. The GNU compiler (GCC)
[2] provides the **-Wall** option to emit all recommended warnings:

    
    
    gcc -Wall -c file.c -o file.o
    
    

Other warnings are possible that are not included within the scope of
**-Wall** (such as **-Wshadow**, to warn when a local variable shadows another
or **-Wstrictprototyptes**, to warn when a function is declared without
argument types). These warning types can be more specialized, so the GCC or
your tool chain's documentation should be consulted for more details. Enabling
optimization (such as **-O2**) can also help identify problems during the
compilation process, and should, therefore, be enabled early.

A useful dynamic memory testing library is the electric fence library written
by Bruce Perens (available in standard Linux distributions). This library
replaces the **malloc**, **free**, and **delete** Standard Library functions
with versions that perform internal checks and can be used to identify
overruns and access to previously freed memory. Using the electric fence
library is as simple as linking with the library:

    
    
    gcc program.c -o program -lefence
    
    

A symbolic debugger such as GDB can be used to identify where the error
actually occurred.

In many cases, software that is complex is also more prone to errors.
Identifying the complexity of a given set of functions can therefore be
important to know which require simplification or refactoring. But how does
one identify the relative complexity of a set of functions? One answer is the
use of a complexity metric such as McCabe. McCabe's cyclomatic complexity
metric [3] provides a measure of the complexity of a function (roughly defined
as the number of paths through a function, taking into account loops and
conditionals). A McCabe metric of greater than 15 typically means that the
function should be reviewed and simplified, if possible. The number of returns
within a function can also be a predictor of complexity and errors and should
therefore be minimized where possible. A good source of free metrics
collection tools is provided in [4].

The Doxygen tool [5] can be used to automatically generate documentation for
C, C++, Java, and other languages. Doxygen scans source and header files and
extracts functions with tagged comments to produce HTML-formatted
documentation. Doxygen can also create dependency graphs, inheritance
diagrams, and even extract code structure from large, undocumented systems to
help simplify their understanding.

In testing software, some believe that an indirect measure of software quality
is the code coverage of those tests. Determining code coverage can be simple
using the GNU gcov tool. The gcov tool can identify the frequency of execution
of every line of an application, which also identifies which source lines were
not covered at all. To enable coverage testing in an application, you need
only compile the application with a special set of flags:

    
    
    gcc -ftest-coverage -fprofile-arcs test.c
    
    

These flags tell the compiler to generate a flow graph of the application and
include additional information to determine the execution profile. Once the
application is executed, a set of files is generated that include the
frequency of execution. The gcov utility can then be executed using the source
file to be analyzed:

    
    
    gcov test.c
    
    

A new file, named test.c.gcov, includes results that contain a listing of the
source with the frequency of execution for each line in the source. Source
lines that are not executed at all are preceded with '######' [6]. The
presence of these lines indicates further testing is required to provide
complete coverage.

The splint tool (Secure Programming lint) from the University of Virginia
Secure Programming Group is a useful tool that can be used to identify
security vulnerabilities and common programming mistakes [7]. Splint is
powerful, but can be used to perform simple checks:

    
    
    splint -weak -I/inc_dir *.c
    
    

where **-I** specifies where your header files are located and ***.c**
specifies all local C source files. Splint helps identify type
inconsistencies, ignored return values, memory leaks, and many other errors.
Omitting the **-weak** argument enables default checking, allowing much more
extensive tests.

### Conclusion

Someday, the construction of software will be just that—the joining of
existing raw materials in the form of software components to build a working
system. The components may carry special timing or memory-use characteristics,
or a certificate of their validity. Unfortunately, we're still in the dark
ages of software engineering, so we must continue to use our existing medieval
methods to do the best we can.

<a name="listing-1"></a>
#### Listing 1

    
    
    #ifdef AUTO_REGRESS
    int stack_regress( void )
    {
       int a, test, ret;
       extern int push( int a );
       extern int pop( int *a );
       a = 1;
       do {
          /* Test 1 -- Empty stack test */
          ret = pop( &test );
          if (test != ERROR) break;
          /* Test 2 -- One push/pop test */
          ret = push( a );
          if (test == ERROR) break;
          ret = pop( &test );
          if ((test == ERROR) || (a != test)) break;
          /* ... more tests, break on failure */ 
          return( SUCCESS );
       } while (0);
       printf("Stack Failure\n");
       return( ERROR );
    }
    #endif
    

### References

[1] Eiffel Language (<http://c2.com/cgi-bin/wiki?EiffelLanguage>).

[2] GCC Documentation (<http://www.gnu.org/software/gcc/onlinedocs/>).

[3] McCabe Cyclomatic Complexity Metric
(<http://www.mccabe.com/iq_research_nist.htm>).

[4] Chris Lott's Metric Tools (<http://www.chris-
lott.org/resources/cmetrics/>).

[5] Doxygen Documentation System (<http://www.doxygen.org/>).

[6] Code Coverage Analysis (<http://www.bullseye.com/webCoverage.html>).

[7] Secure Programming Lint (<http://www.splint.org/>).
