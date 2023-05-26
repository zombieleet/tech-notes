# Table of Contents

Personal notes from [High Perofmrnace Go Workshop](https://dave.cheney.net/high-performance-go-workshop/gophercon-2019.html)

#### BENCHMARKING

In benchmarking a go functions, the go benchmark framework uses `testing.B.N` to represent the number of iterations the function under benchmarked will be executed.

The `b.N` starts at 1, if our function under benchmark executes in 1 seconds or less, go increases
`b.N` and runs the benchmark again.

The go benchmark framework is smart enough to increase `b.N` to a much more higher value if our function runs fine with smaller values.

Run the above test with this command `~go test -bench=. fib_test.go`

```go
package main
import "testing"
    
func Fib3(n int) int {
    switch n {
    case 0:
        return 0
    case 1:
    	return 1
    default:
    	return Fib(n-1) + Fib(n-2)
    }
}
    
func BenchmarkFib20(b *testing.B) {
    for n := 0; n < b.N; n += 1 {
        Fib(20)
    }
}
    
func BenchmarkFib28(b *testing.B) {
    for n := 0; n < b.N; n += 1 {
        Fib(28)
    }
}

```

The output of go benchmark results is

```
benchmarkfuncname(GOMAXPROCS used for running the benchmark)  iterations of the benchmark   time taken per iteration
BenchmarkFib20-12                                                61278                      19972 ns/op
BenchmarkFib28-12                                                1347                       903803 ns/op

```

1.  From the output above `BenchmarkFib20` and `BenchmarkFib28` are both the names of the benchmark function that will call the function we are benchmarking
2.  The prefix `-12` stands for the number of GOMAXPROCS needed in running our benchmark.  FYI, the default value of GOMAXPROCS is the number of CPU we have in our computer.
    1.  This value can be altered by setting `GOMAXPROCSS` to any value.
    2.  The value can also be altered by using the `-cpu` flag when running `go test -bench`
        Specifying a list of values to `-cpu` tells the go benchmark framework to run our benchmark
        using the listed cpus.
        ```
        For example `-cpu=1,2,4` tells the go benchmark framework to run each benchmark using `1` cpu first, then using
        `2` cpus then using `4` cpus
        
        BenchmarkFib20             61753             20174 ns/op
        BenchmarkFib20-2           62487             19690 ns/op
        BenchmarkFib20-4           54854             20118 ns/op
        BenchmarkFib28              1318            919956 ns/op
        BenchmarkFib28-2            1318            921706 ns/op
        BenchmarkFib28-4            1285            937667 ns/op
        ```
        
3.  The iteration means, the amount of times the benchmark ran
4.  The time taken is the number of nanoseconds per iteration, i.e per `b.N` (when increased), the average time taken.
    All the iterations / `b.N` will give ns/op

To increase the number of iterations the benchmark function does, we can set the `-benchtime` variable to a `time.Duration` value

What the `-benchtime` option does is that for each iteration of the benchmark (i.e when the go benchmark framework increases `b.N`)
The execution of the benchmark for each `b.N` will take upto the value of  `-benchtime`, for example `-benchtime=10s` means, each `b.N` will take upto 10seconds. i.e `b.N` will be increased to accomodate the value of `-benctime`

So, for example, if you specify -benchtime=10s and the benchmarking function executes quickly, the testing framework will increase the value of b.N so that the function is executed more times within the 10-second duration. Conversely, if the function executes slowly, the testing framework will decrease the value of b.N so that the function is executed fewer times within the 10-second duration.

The goal is to ensure that the benchmark runs for the specified duration while still providing reliable and accurate results.

Why is the total time reporteded to be >10 seconds, not 10? 
If you have a benchmark which runs for millons or billions of iterations resulting in a time per operation in the micro or nano second range, you may find that your benchmark numbers are unstable because thermal scaling, memory locality, background processing, gc activity, etc.

To address the above problem, a benchmark should be run using `-count=<number>` flag. The default value for `-count` is `1`, this is why whenever you run a benchmark only one benchmark is carried out for each benchmark function.

`go test -bench=. -count=10`

```
BenchmarkFib20-3           62016             19612 ns/op
BenchmarkFib20-3           63309             19030 ns/op
BenchmarkFib20-3           59824             19570 ns/op
BenchmarkFib20-3           63070             20058 ns/op
BenchmarkFib20-3           60199             19222 ns/op
BenchmarkFib20-3           61807             19385 ns/op
BenchmarkFib20-3           60682             20332 ns/op
BenchmarkFib20-3           62098             19134 ns/op
BenchmarkFib20-3           61610             19442 ns/op
BenchmarkFib20-3           60838             19278 ns/op
BenchmarkFib28-3            1332            927677 ns/op
BenchmarkFib28-3            1299            915776 ns/op
BenchmarkFib28-3            1198            914667 ns/op
BenchmarkFib28-3            1272            928730 ns/op
BenchmarkFib28-3            1294            921601 ns/op
BenchmarkFib28-3            1296            926791 ns/op
BenchmarkFib28-3            1266            950244 ns/op
BenchmarkFib28-3            1296            913354 ns/op
BenchmarkFib28-3            1134            918758 ns/op
BenchmarkFib28-3            1286            946153 ns/op
```

When we mix `-count` with `-cpus` , for each list in the `-cpus` flag, the amount of count we have specified will be carried out per element in the list.

**Compairaing benchmark results across several benchmarks**

We can use `benchstat` to compaire the result from multiple runs of our benchmark to know how stable the results are.

Benchmark results can be skewed, it is advisable to run benchmarks multiple times, can compaire how stable the results are across multiple runs.
```
:$ go install golang.org/x/perf/cmd/benchstat
:$ go test -bench=Fib20 -count=10 | tee old.txt
:$ benchstat old.txt
```


The output of old.txt looks like this
```
         │   old.txt   │
         │   sec/op    │
 Fib20-3   29.64µ ± 2%
```


`18.966µ` represents how long it took for each iteration of the benchmark to execute. In this case it took `18.966` microseconds.

`± 1%` represents the variations in each run. The measurements across each run is stable with an error margin of `1%`. The lower the measurement, the lower the error rate.

\#+Improving Fib
Aside from saving the output of a benchmark as seen above, we can also save the binary that produced the benchmark results using the `-c` flag `go test -c`

Use `go test -c` to save the binarythat produced the benchmark results and use `mv fb.test fb.golden` to rename the binary, we will be using benchstat to compare the results from the former fib implementation with the new one.

```go
func Fib3(n int) int {
    switch n {
    case 0:
        return 0
    case 1:
        return 1
    case 2:
    	return 1
    default:
    	return Fib(n-1) + Fib(n-2)
	}
}
```

We will be using benchstat to compare the results from the previous fib and the current fib implementation
```
:$ go test -bench=Fib20 -count=10 | tee new.txt
:$ go test -c
:$ ./fb.golden -test.bench=. -test.count=10 > old.txt
:$ ./fb.test -test.bench=. -test.count=10 > new.txt
```

```
         │   old.txt    │               new.txt               │
         │    sec/op    │   sec/op     vs base                │
Fib20-3    29.38µ ± 1%   19.00µ ± 1%  -35.33% (p=0.000 n=10)
Fib28-3    1385.7µ ± 1%   895.3µ ± 1%  -35.39% (p=0.000 n=10)
geomean    201.8µ        130.4µ       -35.36%

```

The first column shows the name of the benchmarks, in this case `Fib20` and `Fib28`, the `3` prefix means the number of goroutines used in running the benchmarks

The second column (`old.txt`), shows the performance of the previous benchmark, in terms of how long each iteration of the benchmark took represented as `29.38µ` and `1385.7µ` in `Fib20` and `Fib28` respectively, `± 1%` is the percentage of variation in terms of benchmark results in each iteration

The same explanation of the second row applies to `new.txt` but with different values, the benchmark results for `new.txt` shows a performance improvement compared to the benchmark results of `old.txt`

The `vs base` column shows the percentage of improvement of the new benchmark compaired to the old benchmark. The results shows that there was a `35.33%` improvement in performance for `Fib20` in the new benchmark and a `35.39%` improvement in performance for `Fib28-3` in the new benchmark results

The `p-value` is a statistic term, that determins if the benchmark results (performance improvmenet) is significant or it just happened randomly. `p-value` ranges from `0` to `1`, when a `p-value` is lower, it means the benchmark results (performance improvement) did not happen randomly. The accepted `p-value` is `0.05` which means there is a `5%` chance that the benchmark result (performance improvement) did not happen by chance.

The value `n=10` is the number of benchmark runs

**benchmark startup costs**

If we have some expensive setup cost that might affect the benchmark results, its advisable to use, `b.ResetTimer` , `b.StartTimer` , and/or `b.StopTimer` to modify the behaviour of the benchmark

For example, expensive startup cost

    package main
    import "testing"
    
    func BenchmarkExpensiveStartupCost(b *testing.B) {
    	expensiveStartupFunction()
    	// the timer already started counting as soon as
    	// BenchmarkExpesniveStartupCost was called
    	// expensiveStartupFunction() is just some setup cost
    	// that has nothing to do with the actual code we want to benchmark
    	// b.ResetTimer should be used to reset the benchmark timer
    	b.ResetTimer()
    	for n := 0 ; n < b.N; n++ {
    
    	}
    }

Complicated operation inside of the `for loop`

    package main
    import "testing"
    
    func BenchmarkComplicatedSetup(b *testing.B) {
    	for n := 0; n < b.N; n++ {
    		// pause the timer, becuase the next operations
    		// is unlreated to the benchmark
    		b.StopTimer()
    		complicatedSetup()
    		// start the timer
    		b.StartTimer()
    	}
    }

The performance of a program is also dependent on allocation size and count.

We can tell the go benchmark framework to report th enumber of allocations per benchmark

    package main
    import "testing"
    
    func BenchmarkAllocs(b *testing.B) {
    	// we can also use -benchmem with the go test -bench command
    	// as a replacement for *testing.B.ReportAllocs()
      b.ReportAllocs()
    
    	for n := 0; n < b.N; n++ {
    		f();
    	}
    }

**Compiler Optimizations and it's effect on benchmarks**

    package main
    import "testing"
    
    const m1 = 0x5555555555555555
    const m2 = 0x3333333333333333
    const m4 = 0x0f0f0f0f0f0f0f0f
    const h01 = 0x010101010101010
    
    func popcnt(x uint64) uint64 {
    	x -= (x >> 1) & m1
    	x = (x & m2) + ((x >> 2) & m2)
      x = (x + (x >> 4 )) & m4
      return (x * h01) >> 56
    }
    
    func BenchmarkPopcnt(b * testing.B) {
    	for i := 0 ; i < b.N; i++ {
    	   popcnt(uint64(i))
      }
    }

Running the above code with `go test -bench=.` runs for about `0.2501 ns/op` depending on your machine. But this value is actually too low and this is as a result of an optimization the compiler is appliying.

`popcnt` is a leaf function , leaf functions are functions that are not calling any other function, and therefore is subject to certain compiler optimization tricks such as function inlining.

In this situation `popcnt` is inlined. The reason for this optimziation is because of the overhead in allocating memory for each function call.

When you run the above code with `go test -gcflags='-m -m'`
You will see the compiler applying the optimziation, the output looks like this

    ./popcnt_test.go:10:6: can inline popcnt with cost 33 as: func(uint64) uint64 { x -= x >> 1 & m1; x = x & m2 + x >> 2 & m2; x = (x + x >> 4) & m4; return x * h01 >> 56 }
    ./popcnt_test.go:17:6:...
    ./popcnt_test.go:19:9: inlining call to popcnt

To fix the benchmark , we need to assign the result from `popcnt`  to a variable.

    
    package main
    import "testing"
    
    var Result uint64
    
    const m1 = 0x5555555555555555
    const m2 = 0x3333333333333333
    const m4 = 0x0f0f0f0f0f0f0f0f
    const h01 = 0x010101010101010
    
    func popcnt(x uint64) uint64 {
    	x -= (x >> 1) & m1
    	x = (x & m2) + ((x >> 2) & m2)
      x = (x + (x >> 4 )) & m4
      return (x * h01) >> 56
    }
    
    
    func BenchmarkPopcnt(b *testing.B) {
    	var r uint64
    	for i := 0; i < b.N; i++ {
    		r = popcnt(uint64(i))
    	}
    	Result = r
    }

**Benchmarking mistakes**

The following benchmarks runs indefinetly

The below benchmark function will run indefinetly because, `Fib` will always be called with `b.N` (which gets really large) whenever the go benchmark framework calls `BenchmarkFibWrong`. Since `Fib` is a resursive function, it grows exponentially as the value of `b.N` increases.

    func BenchmarkFibWrong(b *testing.B) {
    	Fib(b.N)
    }

`n` is always incremented for every time the benchmark is executed with a larger value of `b.N` by the go benchmark framework. `Fib` will always be called with a larger value of `n` which leads to an infinite loop.

    func BenchmarkFibWrong2(b *testing.B) {
    	for n := 0; n < b.N; n++ {
    		Fib(n)
    	}
    }

#### PROFILING

Benchmarking is only suitable to get an insight into the performance of a function.

In other to investigate why an entire program is running slow, we need to *profile* the whole program.

In golang *pprof* is use in profiling programs. *pprof* originated from [Google PerfTools](https://github.com/gperftools/gperftools)

In golang *pprof* is use in two places.

1.  via code (`runtime/pprof` package)
2.  via the cli (`go tool pprof`)

**cpu profiling:**
  cpu profiling shows how much **cpu time** the program is taking
  in go, the go runtime is interruprted every 10ms and stack traces of the current gorutines are recordedly
  the more time a funciton appeares, the more time that code path is taking as a percentage of the total runtime.

**memory profiling:**
  When a heap allocation is made, memory profiling records the stack trace of the allocation.
  **stack allocations are not tracked** in the memory profile.

Memory profiling are sample based, by default memory profiling samples 1 in every 1000 allocations.
The reason for the above is to reduce the overhead inherent in recording heap allocation, which might drastically affect the performance of the profiler. For every 1000 memory allocation, the profiler will only capture  one of the allocations.

Increasing the memory profiling rate might have a negative effect on application performance.

Memory profiling only tracks memory allocated not memory currently used.

**block profiling:**
  A block profile records how long a gorouting is waiting for a shared resource.
  Waiting/Blocking a gorutine includes

1.  sending or receiving on an unbuffered channel
2.  sending to a full buffered channel , receving from an empty one
3.  Trying to acquire a mutex lock, that is currently held unto by another gorutine

**Mutex profiling:**
  This is similar to block profiling, but it is specifically targetted towards gatthering insight on the delays cause by a mutex contention.

It is advisableto run one profile at a time , reason been that profiling an application can have a negative impact on the performance of the program.

#### Collecting a profile

The below program counts the number of words in a file

    package main
    
    import (
    	"bufio"
    	"fmt"
    	"io"
    	"log"
    	"os"
    	"unicode"
    
    	"github.com/pkg/profile"
    )
    
    func readByte(r io.Reader) (rune, error) {
    	var buf [1]byte
    	_, err := r.Read(buf[:])
    	return rune(buf[0]), err
    }
    
    func main() {
    	defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()
    
    	f, err := os.Open(os.Args[1])
    
    	if err != nil {
    		log.Fatalf("could not open file %q: %v", os.Args[1], err)
    	}
    
    	words := 0
    
    	inword := false
    
    	b := bufio.NewReader(f)
    
    	for {
    		r, err := readByte(b)
    		if err == io.EOF {
    			break
    		}
    		if err != nil {
    			log.Fatalf("could not read file %q: %v", os.Args[1], err)
    		}
    		if unicode.IsSpace(r) && inword {
    			words++
    			inword = false
    		}
    		inword = unicode.IsLetter(r)
    	}
    	fmt.Printf("%q: %d words\n", os.Args[1], words)
    }

Download [Moby-Dick](https://www.gutenberg.org/files/2701/2701-0.txt)
`go build count_words.go && time ./count_words moby.txt`

    real	0m0.232s
    user	0m0.029s
    sys	0m0.010s

Compairing the difference between the `count_words` impelementation and `wc` `time wc -w moby.txt`, there is a big difference between their running time. We can use pprof to investigate the difference

    real	0m0.015s
    user	0m0.008s
    sys	0m0.00s5

**CPU PROFILING**

    package main
    
    import (
    	"bufio"
    	"fmt"
    	"io"
    	"log"
    	"os"
    	"unicode"
    
    	"github.com/pkg/profile"
    )
    
    func readByte(r io.Reader) (rune, error) {
    	var buf [1]byte
    	_, err := r.Read(buf[:])
    	return rune(buf[0]), err
    }
    
    func main() {
    	defer profile.Start().Stop()
    
    	f, err := os.Open(os.Args[1])
    
    	if err != nil {
    		log.Fatalf("coul dnot open file %q: %v", os.Args[1], err)
    	}
    
    	words := 0
    
    	inword := false
    
    	b := bufio.NewReader(f)
    
    	for {
    		r, err := readByte(b)
    		if err == io.EOF {
    			break
    		}
    		if err != nil {
    			log.Fatalf("could not read file %q: %v", os.Args[1], err)
    		}
    		if unicode.IsSpace(r) && inword {
    			words++
    			inword = false
    		}
    		inword = unicode.IsLetter(r)
    	}
    	fmt.Printf("%q: %d words\n", os.Args[1], words)
    }

running `go run count_words.go moby.txt` generates a `cpu.pprof` file.

Use `go tool pprof cpu.pprof` to analyze the profiling information

We can use the `top` command to see what function took the most cpu time.

Using the `top` command shows an that looks like this

    flat  flat%   sum%        cum   cum%
    10ms 50.00%   50.00%      10ms 50.00%  runtime.(*mspan).init (inline)
    10ms 50.00%   100%        10ms 50.00%  syscall.syscall
    0     0%      100%        10ms 50.00%  bufio.(*Reader).Read
    0     0%      100%        10ms 50.00%  internal/poll.(*FD).Read
    0     0%      100%        10ms 50.00%  internal/poll.ignoringEINTRIO (inline)
    0     0%      100%        20ms   100%  main.main
    0     0%      100%        20ms   100%  main.readByte (inline)
    0     0%      100%        10ms 50.00%  os.(*File).Read
    0     0%      100%        10ms 50.00%  os.(*File).read (inline)
    0     0%      100%        10ms 50.00%  runtime.(*mcache).nextFree

**top command columns and their meaning**
`flat` The amount of cpu time a function or method took excluding other functions or method called by it.
`flat%` The percentage of cpu time a function or method took excluding other functions or method called by it.
`sum%` The percentage of cpu time a function or method took including other functions or method called by it.
`cum` The cumulative cpu time a function or method took including other functions or method called by it.
`cum%` The cumulative percentage of cputime a function or method took including other functions or method called by it.

`runtime.(*mspan).init (inline)` and `syscall.syscall` individually took 10ms of cpu time (`flat`) and 50% of cpu time (`flat%`) (this excludes all other functions/method called by them). The total amount of cum cpu time spent on calling this methods and other functions/method called by it is 10ms it also took 50% of total cpu time.

`bufio.(*Reader).Read` , `internal/poll.(*FD).Read` and `internal/poll.ignoringEINTRIO (inline)` took 0ms of cpu time , this is because these method  s were only responsible for calling other methods/functions, excecuting them took 0ms fo cpu time. The methods called by the above methods all together took 50% of cpu time.

`main.main` and `main.readByte` took 0ms of cpu time on their own. But together they took 100% of cpu time.

`os.(*File).Read`, `os.(*File).read (inline)`, `runtime.(*mcache).nextFree` took 0ms of cpu time on their own, but took 50% of cpu time all together

The most expensive operation here is `syscall.syscall`

1.  Executing the method alone took 10ms and 10% of cpu time
2.  Executing the method and other method/functions call by it took 100% of cpu time
3.  SystemCalls are also expensive operations. Syscall operations are executed by the kernel, for this to happen they need to be a transition from user mode to kernel mode which involves context switching (a time consuming operation). When the kernel finishes executing the syscall operation, it has to return control back to the go runtime, which also involves a context switch. This will be problematic when we have toomany syscall operations to carry out.
4.  Each call to `readByte` calls `bufio.Read`, and for every call to `bufio.Read`  `syscall.syscall` is also called, therefore for every single byte in the moby-dick text we are `syscall.syscall` is been called.

The profiling file can also be visualize with this command
`go tool pprof -http=:port cpu.pprof`

From the graph, the bigger box(es) is/are where the cpu took much time on.

**MEMORY PROFILING**

    package main
    
    import (
    	"bufio"
    	"fmt"
    	"io"
    	"log"
    	"os"
    	"unicode"
    
    	"github.com/pkg/profile"
    )
    
    func readByte(r io.Reader) (rune, error) {
    	var buf [1]byte
    	_, err := r.Read(buf[:])
    	return rune(buf[0]), err
    }
    
    func main() {
    	defer profile.Start(profile.MemProfile, profile.ProfilePath(".")).Stop()
    
    	f, err := os.Open(os.Args[1])
    
    	if err != nil {
    		log.Fatalf("coul dnot open file %q: %v", os.Args[1], err)
    	}
    
    	words := 0
    
    	inword := false
    
    	b := bufio.NewReader(f)
    
    	for {
    		r, err := readByte(b)
    		if err == io.EOF {
    			break
    		}
    		if err != nil {
    			log.Fatalf("could not read file %q: %v", os.Args[1], err)
    		}
    		if unicode.IsSpace(r) && inword {
    			words++
    			inword = false
    		}
    		inword = unicode.IsLetter(r)
    	}
    	fmt.Printf("%q: %d words\n", os.Args[1], words)
    }

running `go tool pprof mem.pprof` gives the below output

        flat  flat%   sum%        cum   cum%
    488.95kB 97.44% 97.44%   488.95kB 97.44%  main.readByte (inline)
      4.59kB  0.91% 98.36%     4.59kB  0.91%  runtime.allocm
      4.21kB  0.84% 99.19%     4.21kB  0.84%  runtime.malg
      4.05kB  0.81%   100%      493kB 98.25%  runtime.main
           0     0%   100%   488.95kB 97.44%  main.main
           0     0%   100%     4.59kB  0.91%  runtime.mstart
           0     0%   100%     4.59kB  0.91%  runtime.mstart0
           0     0%   100%     4.59kB  0.91%  runtime.mstart1
           0     0%   100%     4.59kB  0.91%  runtime.newm
           0     0%   100%     4.21kB  0.84%  runtime.newproc.func1

`main.readByte` has most allocation which accounts for 488.95kb allocated memory.

`go tool pprof -http=:port mem.pprof` gives a visual representation of the memory profile.
The bigger boxes shows where most memory is been allocated in.

The reason `main.readByte` has the highest allocation is because for every word in the moby-dick text, a 1 byte array allocation is been made at `var buf [1]byte` on the heap

    func readByte(r io.Reader) (rune, error) {
    	var buf [1]byte
    	_, err := r.Read(buf[:])
    	return rune(buf[0]), err
    }

#### GO COMPILER OPTIMIZATIONS

The **FRONTEND** of the go compiler performs the following optimizations for example.

1.  Escape Analysis
2.  Inlining

While the code is still in its AST form; then the code is passed to the SSA compiler for further optimzations like:

1.  Dead code elimination
2.  Bounds check elimination
3.  Nil check elimination

**Escape Analysis**

Escale analysis has to do with the go compiler determining if a value should be in the heap and also moving a value to the heap.

A value is moved to the heap if it lives beyond the lifecycle of a function. The term used here is `escapes to heap`

For example `&Foo{A: 3, b: 1, c: 4, d: 7}` will exist outside of `NewFoo` hence it will escape to the heap.

    package main
    import "fmt"
    type Foo struct {
    	a, b, c, d int
    }
    
    func NewFoo() *Foo {
    	return &Foo{a: 3, b: 1, c: 4, d: 7}
    }
    func main() {
    	x := NewFoo()
    	fmt.Println(x)
    }

Build the above code with `go build -gcflags="-m" escape_analysis_one.go`

    ...
    ./escape_analysis.go:10:9: &Foo{...} escapes to heap
    ./escape_analysis.go:14:13: &Foo{...} escapes to heap

#### INLINING

Inlining is a compiler optimization trick to enhance the performance of a program. Instead of creating/executing a function as a separate entity on its own, the compiler copies the definition of that function into the caller. The purpose of this is reducing the amount of call made to the function been inlined. Inlining reduces the overhead inherent in setting up and tearing down of functions.

Inlining example

    package main
    func Max(a, b int) int {
    	if a > b {
    		return a
    	}
    	return b
    }
    
    func F() {
    	const a, b = 100, 20
    	if Max(a, b) == b {
    		panic(b)
    	}
    }
    
    func main() {
    	F()
    }

Building the above program with `go build -gcflags="-m"` shows the inlining trick done by the compiler

    ./inlining.go:3:6: can inline Max
    ./inlining.go:10:6: can inline F
    ./inlining.go:12:8: inlining call to Max
    ./inlining.go:17:6: can inline main
    ./inlining.go:18:3: inlining call to F
    ./inlining.go:18:3: inlining call to Max

To see what asembly instructions where generated for the inlining, run

`go build -gcflags=-S`, the `-S` outputs the asm instruction for a go program.

    main.F STEXT nosplit size=1 args=0x0 locals=0x0 funcid=0x0 align=0x0
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	TEXT	main.F(SB), NOSPLIT|ABIInternal, $0-0
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	FUNCDATA	$0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	FUNCDATA	$1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:15)	RET

Our concern in the output is the `RET` keyword, control is been immediately returned back to teh caller `F`, the complicated steps neeeded to setup the function `F` is no longer carried out.
Such an operation look is thus


> marshalling parameters into registers or onto the stack (depending on the ABI) and reversing the process on return. Invoking a function call involves jumping the program counter from one point in the instruction stream to another which can cause a pipeline stall. Once inside the function there is usually some preamble required to prepare a new stack frame for the function to execute and a similar epilogue needed to retire the frame before returning to the caller. - [David Cheney](https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go)

We can tell the `go build` to show the unlinined version of the code either using `//go:nonline` or using the `-N` flag with `gcflags`

Build the code again with `go build -gcflags=-S -N`

    main.F STEXT nosplit size=65 args=0x0 locals=0x20 funcid=0x0 align=0x0
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	TEXT	main.F(SB), NOSPLIT|ABIInternal, $32-0
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	SUBQ	$32, SP
    	0x0004 00004 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	MOVQ	BP, 24(SP)
    	0x0009 00009 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	LEAQ	24(SP), BP
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	FUNCDATA	$0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:10)	FUNCDATA	$1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	MOVQ	$100, main.a+16(SP)
    	0x0017 00023 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	MOVQ	$20, main.b+8(SP)
    	0x0020 00032 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	MOVQ	$0, main.~R0(SP)
    	0x0028 00040 (<unknown line number>)	NOP
    	0x0028 00040 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:4)	  JMP	42
    	0x002a 00042 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	MOVQ	$100, main.~R0(SP)
    	0x0032 00050 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	JMP	52
    	0x0034 00052 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:12)	JMP	54
    	0x0036 00054 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:15)	MOVQ	24(SP), BP
    	0x003b 00059 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:15)	ADDQ	$32, SP
    	0x003f 00063 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:15)	NOP
    	0x0040 00064 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/inlining.go:15)	RET

The first thing to notice here is the `main.F STEXT nosplit size=65 args=0x0 locals=0x20 funcid=0x0 align=0x0`, the unoptimized version have a code size of `65 bytes` whereas the optimized version has a code size of `1 byte`

We can also see some operations in the Stack Pointer (SP) where local variables and function argument are been stored, also points to the currention location on the stack.

Inlining levels can also be adjusted using `-gcflags='-l -l'`
Using one `-l` argument disables inlining, using multiple `--l` arguments enables inlining in a more aggressive manner, but it will produce bigger binaries.
`-gcflags-l=4` enables mid-stack inlining

#### Bounds Check Elimination

Go is a bound checked langauge, this means that, the go runtime checks if the location to be accssed in a slice or array is within the bounds of that array/slice or else it panics.
Bound check elimination can improve the performance of a go program.

Take for example this code, and run it with `go test -gcflags='-S' bound_check_test.go`


    package main
    
    var v = make([]int, 9)
    
    var A,B,C,D,E,F,G,H,I int
    
    func BenchmarkBoundsCheckInOrder(b *testing.B) {
    	var a, _b, c, d, e, f, g, h, i int
    	for n := 0; n < b.N; n++ {
    		a = v[0]
    		_b = v[1]
    		c = v[2]
    		d = v[3]
    		e = v[4]
    		f = v[5]
    		g = v[6]
    		h = v[7]
    		i = v[8]
    	}
    	A, B, C, D, E, F , G, H, I = a, _b, c, d, e, f, g, h, i
    }

Running the above benchmark with `go test -bench=.` shows that, for `549188632` iteration the benchmark took `2.123 ns/op`

The output you get from runing the above code with `-gclfags='-S'` generates the asm instructions for the code, but we are concerned with the output from `main.BenchmarkBoundsCheckInOrder`

Our focus in this output is on the `CALL` instruction

    command-line-arguments.BenchmarkBoundsCheckInOrder STEXT nosplit size=414 args=0x8 locals=0x18 funcid=0x0 align=0x0
    0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	TEXT	command-line-arguments.BenchmarkBoundsCheckInOrder(SB), NOSPLIT|ABIInternal, $24-8
    0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	SUBQ	$24, SP
    0x0004 00004 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	MOVQ	BP, 16(SP)
    0x0009 00009 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	LEAQ	16(SP), BP
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$0, gclocals·wgcWObbY2HYnK2SU/U22lA==(SB)
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB)
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$5, command-line-arguments.BenchmarkBoundsCheckInOrder.arginfo1(SB)
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$6, command-line-arguments.BenchmarkBoundsCheckInOrder.argliveinfo(SB)
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	PCDATA	$3, $1
    0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	CX, CX
    0x0010 00016 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	DX, DX
    0x0012 00018 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	BX, BX
    0x0014 00020 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	SI, SI
    0x0016 00022 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	DI, DI
    0x0018 00024 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R8, R8
    0x001b 00027 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R9, R9
    0x001e 00030 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R10, R10
    0x0021 00033 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R11, R11
    0x0024 00036 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R12, R12
    0x0027 00039 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	JMP	78
    0x0029 00041 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	INCQ	CX
    0x002c 00044 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	MOVQ	64(BX), DX
    0x0030 00048 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R13, BX
    0x0033 00051 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	SI, R15
    0x0036 00054 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R12, SI
    0x0039 00057 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R15, R12
    0x003c 00060 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	DI, R13
    0x003f 00063 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R11, DI
    0x0042 00066 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R13, R11
    0x0045 00069 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R8, R13
    0x0048 00072 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R10, R8
    0x004b 00075 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R13, R10
    0x004e 00078 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	CMPQ	416(AX), CX
    0x0055 00085 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	JLE	226
    0x005b 00091 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	command-line-arguments.v+8(SB), DX
    0x0062 00098 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	command-line-arguments.v(SB), BX
    0x0069 00105 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	TESTQ	DX, DX
    0x006c 00108 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	JLS	403
    0x0072 00114 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	(BX), SI
    0x0075 00117 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	CMPQ	DX, $1
    0x0079 00121 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	JLS	390
    0x007f 00127 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	MOVQ	8(BX), DI
    0x0083 00131 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	CMPQ	DX, $2
    0x0087 00135 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	JLS	377
    0x008d 00141 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	MOVQ	16(BX), R8
    0x0091 00145 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	CMPQ	DX, $3
    0x0095 00149 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	JLS	364
    0x009b 00155 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	MOVQ	24(BX), R9
    0x009f 00159 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	NOP
    0x00a0 00160 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	CMPQ	DX, $4
    0x00a4 00164 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	JLS	351
    0x00aa 00170 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	MOVQ	32(BX), R10
    0x00ae 00174 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	CMPQ	DX, $5
    0x00b2 00178 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	JLS	338
    0x00b8 00184 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	MOVQ	40(BX), R11
    0x00bc 00188 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	NOP
    0x00c0 00192 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	CMPQ	DX, $6
    0x00c4 00196 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	JLS	325
    0x00c6 00198 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	MOVQ	48(BX), R12
    0x00ca 00202 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	CMPQ	DX, $7
    0x00ce 00206 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	JLS	312
    0x00d0 00208 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	MOVQ	56(BX), R13
    0x00d4 00212 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	CMPQ	DX, $8
    0x00d8 00216 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	JHI	41
    0x00de 00222 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	NOP
    0x00e0 00224 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	JMP	299
    0x00e2 00226 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R12, command-line-arguments.A(SB)
    0x00e9 00233 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R11, command-line-arguments.B(SB)
    0x00f0 00240 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R10, command-line-arguments.C(SB)
    0x00f7 00247 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R9, command-line-arguments.D(SB)
    0x00fe 00254 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R8, command-line-arguments.E(SB)
    0x0105 00261 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	DI, command-line-arguments.F(SB)
    0x010c 00268 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	SI, command-line-arguments.G(SB)
    0x0113 00275 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	BX, command-line-arguments.H(SB)
    0x011a 00282 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	DX, command-line-arguments.I(SB)
    0x0121 00289 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	MOVQ	16(SP), BP
    0x0126 00294 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	ADDQ	$24, SP
    0x012a 00298 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	RET
    0x012b 00299 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	MOVL	$8, AX
    0x0130 00304 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	MOVQ	AX, CX
    0x0133 00307 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	PCDATA	$1, $1
    0x0133 00307 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	CALL	runtime.panicIndex(SB)
    0x0138 00312 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	MOVL	$7, AX
    0x013d 00317 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	MOVQ	AX, CX
    0x0140 00320 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	CALL	runtime.panicIndex(SB)
    0x0145 00325 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	MOVL	$6, AX
    0x014a 00330 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	MOVQ	AX, CX
    0x014d 00333 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	CALL	runtime.panicIndex(SB)
    0x0152 00338 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	MOVL	$5, AX
    0x0157 00343 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	MOVQ	AX, CX
    0x015a 00346 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	CALL	runtime.panicIndex(SB)
    0x015f 00351 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	MOVL	$4, AX
    0x0164 00356 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	MOVQ	AX, CX
    0x0167 00359 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	CALL	runtime.panicIndex(SB)
    0x016c 00364 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	MOVL	$3, AX
    0x0171 00369 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	MOVQ	AX, CX
    0x0174 00372 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	CALL	runtime.panicIndex(SB)
    0x0179 00377 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	MOVL	$2, AX
    0x017e 00382 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	MOVQ	AX, CX
    0x0181 00385 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	CALL	runtime.panicIndex(SB)
    0x0186 00390 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	MOVL	$1, AX
    0x018b 00395 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	MOVQ	AX, CX
    0x018e 00398 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	CALL	runtime.panicIndex(SB)
    0x0193 00403 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	XORL	AX, AX
    0x0195 00405 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	AX, CX
    0x0198 00408 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	CALL	runtime.panicIndex(SB)
    0x019d 00413 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	XCHGL	AX, AX

For each index in a slice that is been acceessed, go the runtime calls \`runtime.panicIndex\` on that index, if it is out of range, the go program panics.
From the above instruction, \`runtime.panicIndex\` is called multiple times, the reason for this is because , the go program is accessing the slice index in an ascending manner. There is no way for the go runtime to ascertain that subsequent accesses are within the bounds of the slice at runtime

The below code, accesses the array index from the highest index

    package main
    
    import (
    	"testing"
    )
    
    var v = make([]int, 9)
    
    var A, B, C, D, E, F, G, H, I int
    
    func BenchmarkBoundsCheckInOrder(b *testing.B) {
    	var a, _b, c, d, e, f, g, h, i int
    	for n := 0; n < b.N; n++ {
    		a = v[8]
    		_b = v[0]
    		c = v[1]
    		d = v[2]
    		e = v[3]
    		f = v[4]
    		g = v[5]
    		h = v[6]
    		i = v[7]
    	}
    	A, B, C, D, E, F, G, H, I = a, _b, c, d, e, f, g, h, i
    }

Running the above benchmark with `go test -bench=.` shows that the benchmark result outperforms the previous benchmark by `50%` for `638168262` iteration it was able to take `1.838 ns/op` to execute.

Run it with `go test -gcflags='-S' bound_check2_test.go`, the asm instruction will look like this

    command-line-arguments.BenchmarkBoundsCheckInOrder STEXT nosplit size=198 args=0x8 locals=0x18 funcid=0x0 align=0x0
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	TEXT	command-line-arguments.BenchmarkBoundsCheckInOrder(SB), NOSPLIT|ABIInternal, $24-8
    	0x0000 00000 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	SUBQ	$24, SP
    	0x0004 00004 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	MOVQ	BP, 16(SP)
    	0x0009 00009 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	LEAQ	16(SP), BP
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$0, gclocals·wgcWObbY2HYnK2SU/U22lA==(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$5, command-line-arguments.BenchmarkBoundsCheckInOrder.arginfo1(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	FUNCDATA	$6, command-line-arguments.BenchmarkBoundsCheckInOrder.argliveinfo(SB)
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	PCDATA	$3, $1
    	0x000e 00014 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	CX, CX
    	0x0010 00016 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	DX, DX
    	0x0012 00018 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	BX, BX
    	0x0014 00020 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	SI, SI
    	0x0016 00022 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	DI, DI
    	0x0018 00024 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R8, R8
    	0x001b 00027 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R9, R9
    	0x001e 00030 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R10, R10
    	0x0021 00033 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R11, R11
    	0x0024 00036 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:11)	XORL	R12, R12
    	0x0027 00039 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	JMP	79
    	0x0029 00041 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	INCQ	CX
    	0x002c 00044 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	64(DX), R12
    	0x0030 00048 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:15)	MOVQ	(DX), R11
    	0x0033 00051 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:16)	MOVQ	8(DX), R10
    	0x0037 00055 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:17)	MOVQ	16(DX), R9
    	0x003b 00059 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:18)	MOVQ	24(DX), R8
    	0x003f 00063 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:19)	MOVQ	32(DX), DI
    	0x0043 00067 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:20)	MOVQ	40(DX), SI
    	0x0047 00071 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:21)	MOVQ	48(DX), BX
    	0x004b 00075 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:22)	MOVQ	56(DX), DX
    	0x004f 00079 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	CMPQ	416(AX), CX
    	0x0056 00086 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:13)	JLE	110
    	0x0058 00088 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	command-line-arguments.v(SB), DX
    	0x005f 00095 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	command-line-arguments.v+8(SB), BX
    	0x0066 00102 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	CMPQ	BX, $8
    	0x006a 00106 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	JHI	41
    	0x006c 00108 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	JMP	183
    	0x006e 00110 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R12, command-line-arguments.A(SB)
    	0x0075 00117 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R11, command-line-arguments.B(SB)
    	0x007c 00124 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R10, command-line-arguments.C(SB)
    	0x0083 00131 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R9, command-line-arguments.D(SB)
    	0x008a 00138 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	R8, command-line-arguments.E(SB)
    	0x0091 00145 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	DI, command-line-arguments.F(SB)
    	0x0098 00152 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	SI, command-line-arguments.G(SB)
    	0x009f 00159 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	BX, command-line-arguments.H(SB)
    	0x00a6 00166 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:24)	MOVQ	DX, command-line-arguments.I(SB)
    	0x00ad 00173 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	MOVQ	16(SP), BP
    	0x00b2 00178 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	ADDQ	$24, SP
    	0x00b6 00182 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:25)	RET
    	0x00b7 00183 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVL	$8, AX
    	0x00bc 00188 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	MOVQ	BX, CX
    	0x00bf 00191 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	PCDATA	$1, $1
    	0x00bf 00191 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	NOP
    	0x00c0 00192 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	CALL	runtime.panicIndex(SB)
    	0x00c5 00197 (/Users/victoryosikwemhe/org-mood-jots/golang/david/high-perf-go/n_test.go:14)	XCHGL	AX, AX

From the above output we can see only one call to `runtime.panicIndex`, the reason for this is because the slice was accessed from the highest index (8), there was no need for subsequent index access to be bound checked, since the index are already within the bounds of the first check, this is where the go runtime eliminates bound checks.

Moving `make([]int, 9)` inside of the benchmark function significantly improves the execution time of the benchmark,

1.  the slice does not escape to the heap, and accessing the stack is way more faster than accessing heap memory.
2.  There is also no need for bound checks here because at compile time the compiler can determine the size of the array or slice. . The compiler can verify that the index being accessed is within the bounds of the array or slice and generate code that directly accesses the memory location without performing a bounds check at runtime. This is different from heap memory because the slice or array can resized dynamically at runtime.

