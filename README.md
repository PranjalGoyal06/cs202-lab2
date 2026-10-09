# cs202-lab2

## Question 2: Finding the memory corruption with GDB

`rr` does not work on macOS, so this investigation uses `gdb` (the GNU
Debugger) in a GitHub Codespaces Cloud VM.

### 1. Compile and run the program

Compile with debug symbols and disable optimization. The `-g` flag lets GDB
map machine instructions back to source lines, while `-O0` keeps the source
execution order easy to follow.

```console
@PranjalGoyal06 ➜ /workspaces/cs202-lab2 (main) $ gcc -g -O0 corruption.c -o corruption
```

Run the program and print its exit status:

```console
@PranjalGoyal06 ➜ /workspaces/cs202-lab2 (main) $ ./corruption; echo "exit=$?"
=== Initial state ===
Student 0: id=1001
Student 1: id=1002

=== After curving ===
Student 0: id=1001
Student 1: id=1007

ERROR: Student 1's ID was corrupted! Expected 1002, got 1007
exit=1
```

The output confirms that `students[1].id` changes from `1002` to `1007`.

### 2. Launch GDB and stop at `main`

```console
@PranjalGoyal06 ➜ /workspaces/cs202-lab2 (main) $ gdb ./corruption
```

At the GDB prompt, disable pagination, set a breakpoint at `main`, and start
the program:

```gdb
(gdb) set pagination off
(gdb) break main
Breakpoint 1 at 0x123f: file corruption.c, line 29.
(gdb) run
Starting program: /workspaces/cs202-lab2/corruption
```

If GDB asks whether to enable automatic debuginfo downloads, answer `y`.
Those downloads are optional and are only for viewing system-library source.
The program should stop here:

```text
Breakpoint 1, main () at corruption.c:29
29          init();
```

The breakpoint stops before initialization so the watchpoint can observe every
later change to the ID.

### 3. Watch the ID for writes

Set a hardware watchpoint on the exact variable and resume execution:

```gdb
(gdb) watch -l students[1].id
Hardware watchpoint 2: -location students[1].id
(gdb) continue
```

The `-l` option watches the variable's memory location rather than repeatedly
evaluating the expression. The first watchpoint hit is the expected
initialization:

```text
Hardware watchpoint 2: -location students[1].id

Old value = 0
New value = 1002
init () at corruption.c:17
17          students[1].scores[0] = 90;
```

Continue once more to find the unexpected write:

```gdb
(gdb) continue
```

GDB prints the program's initial output and then stops when the ID is
corrupted:

```text
=== Initial state ===
Student 0: id=1001
Student 1: id=1002

Hardware watchpoint 2: -location students[1].id

Old value = 1002
New value = 1007
curve_scores (student_idx=0, curve=5) at corruption.c:23
23          for (int i = 0; i < 4; i++) {
```

This watchpoint is the key result: the ID changes to `1007` inside
`curve_scores`, before the program prints the "After curving" output.

### 4. Inspect the loop state and call stack

Run the following commands at the watchpoint:

```gdb
(gdb) print i
$1 = 3
(gdb) print student_idx
$2 = 0
(gdb) print students[student_idx].scores[i]
$3 = 1007
(gdb) print students[1].id
$4 = 1007
(gdb) backtrace
#0  curve_scores (student_idx=0, curve=5) at corruption.c:23
#1  0x000055555555529f in main () at corruption.c:34
(gdb) list corruption.c:20,26
20      }
21
22      void curve_scores(int student_idx, int curve) {
23          for (int i = 0; i < 4; i++) {
24              students[student_idx].scores[i] += curve;
25          }
26      }
```

`i` is `3`, but `scores` was declared as `scores[3]`, so its valid indices are
only `0`, `1`, and `2`. The expression on line 24 is therefore accessing
`students[0].scores[3]`, which is outside the array.

The value printed for `students[student_idx].scores[i]` is already `1007`
because the watchpoint stops after the out-of-bounds addition has completed.

### 5. Prove which variable is being overwritten

Compare the address of the invalid array element with the address of the
student ID:

```gdb
(gdb) print &students[0].scores[3]
$5 = (int *) 0x555555558050 <students+16>
(gdb) print &students[1].id
$6 = (int *) 0x555555558050 <students+16>
```

The addresses are identical. This proves that
`students[0].scores[3]` aliases `students[1].id`.

The memory layout explains the corruption:

```text
students[0].id       -> 1001
students[0].scores[0] -> 85
students[0].scores[1] -> 92
students[0].scores[2] -> 78
students[0].scores[3] -> students[1].id (1002)
```

When the loop executes its fourth iteration, it adds the curve value:

```text
1002 + 5 = 1007
```

That is why Student 1's ID becomes `1007`.

### 6. Identify and fix the bug

The bug is the loop bound in `curve_scores`:

```c
for (int i = 0; i < 4; i++) {
```

There are only three scores, so the loop must stop before index `3`:

```c
for (int i = 0; i < 3; i++) {
```

After making the fix, recompile and run:

```console
gcc -g -O0 corruption.c -o corruption
./corruption; echo "exit=$?"
```

Output:
```text
=== Initial state ===
Student 0: id=1001
Student 1: id=1002

=== After curving ===
Student 0: id=1001
Student 1: id=1002
exit=0
```

The corruption is now resolved!
