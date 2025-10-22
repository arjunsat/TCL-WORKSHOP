TCL Master Notes — From First Principles to EDA-Grade Automation
0) What is Tcl (and why EDA loves it)

Tcl = Tool Command Language (pronounced “tickle”). Small, embeddable, interpreted.

Everything is a string (with structure). Commands take words, return strings.

Batteries included: lists, dicts, arrays, regexp, sockets, file I/O, events, coroutines, OO (TclOO), GUI (Tk).

Why EDA? Easy to embed/invoke, perfect for glueing tools, massaging reports, and automating flows.

1) Command model, words, substitution
1.1 Command structure
commandName  arg1  arg2  ... argN


Words are separated by whitespace.

The first word is the command; rest are arguments.

Commands return a result (a string). That result can be captured or substituted into other commands.

1.2 The three substitution rules

Variable substitution: $var → replaced by value of var.

Arrays: $A(key), namespaced vars: $::ns::var

Command substitution: [...] → replaced by the result of the command inside.

Backslash substitution: escapes: \n, \t, \\, \", \xNN, …

1.3 Quoting and grouping (critical!)

Braces { ... } → no substitution inside, used to group literally. Perfect for code bodies.

Double quotes " ... " → allow $ and [...] substitutions, keep spaces together.

Square brackets [...] → run the command and substitute its result.

Golden rule: Use {} for code bodies to prevent early substitution.

set x 3
expr {$x + 2}       ;# good (no double substitution)
expr "$x + 2"       ;# also works but slower and riskier in nested cases

2) Variables, scope, namespaces
2.1 Basics
set v 42            ;# create/assign
unset v             ;# delete
incr count          ;# count = count + 1 (default increment 1)
incr count 5
append s " more"    ;# efficient string concat

2.2 Scope in procs
proc demo {} {
  set x 10
  # access global:
  global gvar
  # access variable in caller:
  upvar 1 callerVar localAlias
}

2.3 Namespaces
namespace eval ::eda {
  variable root "/work"
  proc joinPath {p1 p2} { file join $p1 $p2 }
}

# Access:
set ::eda::root
::eda::joinPath "/tmp" "a.v"

3) Core data structures
3.1 Lists (homogeneous or not)

Tcl lists are strings with a structure. Use list to build safely.

set L [list a "b c" d]   ;# 3 elements: a, "b c", d
llength $L
lindex  $L 1
lrange  $L 0 1
lappend L newItem
lset    L 1 "B C"
linsert L 2 X
lreplace L 1 2 Y
lsort -unique -integer $L2

3.2 Dictionaries (hash maps) — Tcl 8.5+
set D [dict create design alu_top lib fast.lib]
dict set D outDir ./out
dict get D design
dict exists D lib
dict keys D
dict values D
# Nested:
dict set D cfg clocks period 10.0
dict get D cfg clocks period

3.3 Arrays (associative array variable)
array set A {key1 val1 key2 val2}
set A(key1)
array exists A
array names  A  ;# all keys
unset A(key1)


Use dict for pass-around immutable maps; use array as a mutable variable-scoped map.

4) Strings & regular expressions
4.1 String ops
string length "abc"
string tolower "ABC"
string toupper "abc"
string trim "  x  "
string map {" " ""} "Design Name"   ;# -> "DesignName"
string match "alu_*" "alu_top"      ;# glob match
string equal -nocase a A
string first "timing" $text
string range $s 2 5

4.2 Regexp & regsub
# Test and capture
if {[regexp {WNS:\s*([-\d.]+)} $report -> wns]} {
  puts "WNS=$wns"
}

# Replace (all)
set sanitized [regsub -all {[\\*"]} $netlist ""]
# Extended mode with comments:
regexp -expanded -line {
  ^Startpoint:\s+(\S+).*$
} $log -> start

5) Expressions (expr) and operators

expr evaluates arithmetic, logic, comparisons.

Always brace expr bodies: expr {$a + $b * 2}

Common operators: + - * / %, == != < <= > >=, && || !, ternary ?:, bit ops & | ^ ~ << >>.

String vs numeric compare: use eq/ne for strings, ==/!= for numbers.

expr {$a eq $b}      ;# string equal
expr {$x == $y}      ;# numeric equal

6) Control flow
if {$x > 0} { ... } elseif {$x == 0} { ... } else { ... }

switch -exact -- $mode {
  fast { set lib $fast }
  slow { set lib $slow }
  default { error "unknown mode $mode" }
}

while {$i < 10} { incr i }
for {set i 0} {$i < 10} {incr i} { ... }

# foreach can iterate multiple lists in lockstep
foreach {k v} {a 1 b 2 c 3} {
  puts "$k -> $v"
}

7) Procedures (procs), returns, errors
proc area {w h {unit "mm"}} {
  if {![string is double -strict $w]} { error "w must be number" }
  return [format "%.2f %s^2" [expr {$w*$h}] $unit]
}

# Variadic args
proc log {level args} {
  puts "[$level] [join $args]"
}

# Return codes:
return -code ok -level 0 "done"
error  "message"           ;# return -code error
return -code return        ;# like 'return' from nested levels

7.1 catch and try (8.6+)
if {[catch {dangerous op} err opts]} {
  puts "Failed: $err"
  dict with opts { puts "At $errorInfo" }
}

try {
   # body
} on error {err opts} {
   puts "Failed: $err"
} finally {
   puts "cleanup"
}

8) File system, paths, I/O, channels
8.1 Path utilities (file)
file exists $p
file isdir  $p
file dirname $p
file tail    $p
file join $root "constraints" "a.csv"
file normalize "~/proj"
file copy -force src dst
file rename -force old new
file delete -force $p
file mkdir  $dir

8.2 Streams (open, puts, gets, read)
set fh [open $path r]
set line [gets $fh]            ;# 1 line (no newline)
set all  [read $fh]            ;# whole file
close $fh

# Write
set out [open $sdc w]          ;# "w" write, "a" append, "r+" rw
puts $out "create_clock ..."
close $out

# Binary & buffering
fconfigure $fh -translation binary -encoding utf-8 -blocking 1 -buffering line

8.3 Processes (exec) and pipes
set yosysLog [exec yosys -p {read_verilog top.v; synth -top top; stat} 2>@1]
set p [open "|yosys -s flow.ys" r]
set log [read $p]
close $p

9) Time, events, and the event loop
after 1000 {puts "1s later"}   ;# schedule
after cancel $id
vwait ::done                   ;# wait until variable ::done is written
update                         ;# process events now
clock format [clock seconds] -format {%Y-%m-%d %H:%M:%S}
clock scan "2025-10-23 21:30"

10) Introspection & meta-programming
info commands
info vars
info procs
info body  area
info args  area
info exists var
uplevel 1 {set x 5}           ;# run in caller
upvar   1 x xref               ;# ref caller var as xref
apply {args body ?ns?}         ;# anonymous proc
set f [apply {{x} {expr {$x*$x}}}]
$f 7

11) Pattern you’ll use constantly in EDA automation
11.1 Parse 2-column CSV → dict
proc csv2dict {csvPath} {
  set fh [open $csvPath r]
  set D {}
  while {[gets $fh line] >= 0} {
    if {[string trim $line] eq ""} continue
    set parts [split $line ","]
    set key [string map {" " ""} [string trim [lindex $parts 0]]]
    set val [string trim [join [lrange $parts 1 end] ","]]
    dict set D $key $val
  }
  close $fh
  return $D
}
# Usage
set cfg [csv2dict ./constraints/design.csv]
set top [dict get $cfg DesignName]

11.2 Emit SDC from discovered columns
proc emit_sdc {csv outSdc} {
  set fh [open $csv r]
  set rows {}
  while {[gets $fh line] >= 0} {
    if {[string trim $line] eq ""} continue
    lappend rows [split $line ","]
  }
  close $fh
  # Find headers
  set headers [lindex $rows 0]
  set colIdx {}
  foreach {name} {port clock period src_latency trans in_delay out_delay} {
    set idx [lsearch -exact $headers $name]
    dict set colIdx $name $idx
  }
  # Write SDC
  set s [open $outSdc w]
  puts $s "# Auto-generated SDC"
  foreach row [lrange $rows 1 end] {
    set port [lindex $row [dict get $colIdx port]]
    set clk  [lindex $row [dict get $colIdx clock]]
    set per  [lindex $row [dict get $colIdx period]]
    set src  [lindex $row [dict get $colIdx src_latency]]
    set tr   [lindex $row [dict get $colIdx trans]]
    set id   [lindex $row [dict get $colIdx in_delay]]
    set od   [lindex $row [dict get $colIdx out_delay]]
    if {$per ne ""} { puts $s "create_clock -name $clk -period $per [get_ports $clk]" }
    if {$src ne ""} { puts $s "set_clock_latency -source $src [get_clocks $clk]" }
    if {$tr  ne ""} { puts $s "set_clock_transition $tr [get_clocks $clk]" }
    if {$id  ne ""} { puts $s "set_input_delay  $id  -clock $clk [get_ports $port]" }
    if {$od  ne ""} { puts $s "set_output_delay $od  -clock $clk [get_ports $port]" }
  }
  close $s
}

11.3 Run Yosys from Tcl and capture logs
proc run_yosys {top rtlDir lib outDir} {
  file mkdir $outDir
  set ys [file join $outDir "${top}.ys"]
  set fh [open $ys w]
  puts $fh "read_liberty -lib $lib"
  puts $fh "read_verilog [glob -nocomplain -directory $rtlDir *.v]"
  puts $fh "hierarchy -top $top"
  puts $fh "synth -top $top"
  puts $fh "stat"
  puts $fh "write_verilog $outDir/${top}_synth.v"
  close $fh
  return [exec yosys -s $ys 2>@1]
}

11.4 Extract QOR from timing log
proc extract_qor {timingLog} {
  set data [read [open $timingLog r]]
  set WNS ""; set TNS ""
  regexp {WNS:\s*([-\d.]+)} $data -> WNS
  regexp {TNS:\s*([-\d.]+)} $data -> TNS
  return [dict create WNS $WNS TNS $TNS]
}

12) Packages and modules
12.1 Version & package
package require Tcl 8.6
package provide mypkg 1.0

namespace eval ::mypkg {
  variable version 1.0
  proc hello {} { return "hi" }
}


Place in mypkg.tcl, and add a pkgIndex.tcl to enable package require mypkg.

# pkgIndex.tcl
package ifneeded mypkg 1.0 [list source [file join $dir mypkg.tcl]]


Add the directory to auto_path or set TCLLIBPATH.

13) Error handling & defensive programming

Always brace expressions: expr {...}

Validate types: string is integer -strict $n

Fail fast with error, wrap flows in try/catch.

Be explicit with encodings: fconfigure $fh -encoding utf-8 -translation lf.

Never build file paths by string concat → use file join.

Avoid double substitution by using {} for code bodies.

Sanitize external tool inputs/outputs (regex & string map helpers).

14) Events, async patterns, and long runs

Schedule background work with after, then vwait done.

For pipelines, prefer open "|cmd" and non-blocking fconfigure -blocking 0.

UI (Tk) flows call update wisely (not in tight loops).

For headless automation, keep event loop minimal and deterministic.

15) Coroutines (8.6+) — lightweight generators
proc gen {n} {
  for {set i 0} {$i < $n} {incr i} {
    yield $i
  }
}
coroutine it gen 5
it  ;# -> 0
it  ;# -> 1 ...


Useful for streaming large logs or incremental CSV processing without blocking.

16) TclOO (object-orientation)
oo::class create Report {
  variable data
  constructor {} { set data {} }
  method add {k v} { dict set data $k $v }
  method toCSV {} {
    set out ""
    foreach k [dict keys $data] {
      append out "$k,[dict get $data $k]\n"
    }
    return $out
  }
}
set r [Report new]
$r add WNS -0.03
$r add TNS -1.25
puts [$r toCSV]
$r destroy


OO is optional but neat for complex reporters/transformers.

17) Testing & debugging

tcltest:

package require tcltest
namespace import ::tcltest::*
test add-1 {adds numbers} -body {expr {2+3}} -result 5
cleanupTests


Debug tips:

puts breadcrumbs with unique prefixes.

info level, info frame, errorInfo, errorCode.

Log to file: set log [open ./reports/run.log a]; puts $log "..."; close $log

18) Mini cookbook (copy/paste)
18.1 Safe eval of arithmetic user input
proc safeAdd {a b} {
  if {![string is double -strict $a] || ![string is double -strict $b]} {
    error "inputs must be numeric"
  }
  expr {$a + $b}
}

18.2 Robust line walker over big files
proc eachLine {path callback} {
  set fh [open $path r]
  fconfigure $fh -encoding utf-8 -buffering line
  while {[gets $fh line] >= 0} {
    uplevel #0 [list $callback $line]
  }
  close $fh
}
# Usage:
proc countClocks {line} {
  if {[string match "*create_clock*" $line]} {incr ::nClocks}
}
set ::nClocks 0
eachLine constraints.sdc countClocks
puts "Found $::nClocks clocks"

18.3 Table-to-dict (headers + rows)
proc table2dicts {csvPath} {
  set fh [open $csvPath r]
  set headers [split [gets $fh] ","]
  set rows {}
  while {[gets $fh line] >= 0} {
    if {[string trim $line] eq ""} continue
    set cols [split $line ","]
    set d {}
    for {set i 0} {$i < [llength $headers]} {incr i} {
      dict set d [string map {" " ""} [lindex $headers $i]] [lindex $cols $i]
    }
    lappend rows $d
  }
  close $fh
  return $rows
}

19) Style guide (opinionated but battle-tested)

Brace bodies and exprs: if {...}, expr {...}.

Name things clearly: verbs for procs (emit_sdc, run_yosys), nouns for data (cfg, report, qor).

Prefer dict for passing structured data; arrays only for scoped vars.

One responsibility per proc; small proc sizes.

Fail fast; never continue with broken inputs.

No magic globals—use namespaces or pass data explicitly.

Write small tests for your parsing and generation logic.

20) Common gotchas (read these twice)

if "$x > 0" is wrong; you must use if {$x > 0}.

expr $x+1 is wrong; use expr {$x + 1}.

String vs numeric compare: eq/ne vs ==/!=.

Double substitution bugs: using "" for code bodies can evaluate variables too early—use {}.

file join over manual '/$a/$b'.

Windows paths: prefer file normalize to avoid backslash hell.

Large concatenations: use append/lappend (efficient), not repeated set s "$s...".

21) Putting it all together (EDA-grade skeleton)
# vsdsynth.tcl — CSV→variables→SDC→Yosys→OpenTimer→QOR
package require Tcl 8.6

proc die {msg} { puts stderr $msg; exit 1 }

# 1) Read config CSV
set cfgCsv [lindex $argv 0]
if {$cfgCsv eq "" || ![file exists $cfgCsv]} { die "config CSV missing" }
set CFG [csv2dict $cfgCsv]
foreach k {OutputDirectory NetlistDirectory EarlyLibraryPath LateLibraryPath ConstraintsFile} {
  if {[dict exists $CFG $k]} { dict set CFG $k [file normalize [dict get $CFG $k]] }
}

# 2) Emit SDC
set top   [dict get $CFG DesignName]
set out   [dict get $CFG OutputDirectory]
file mkdir $out
emit_sdc [dict get $CFG ConstraintsFile] [file join $out "${top}.sdc"]

# 3) Yosys
set log [run_yosys $top [dict get $CFG NetlistDirectory] [dict get $CFG EarlyLibraryPath] $out]
set ysRpt [file join $out "yosys.log"]; set fh [open $ysRpt w]; puts $fh $log; close $fh

# 4) Timing (your OpenTimer invocation here) → produce timing.log
 exec opentimer -f flow.cfg > [file join $out timing.log]
 set qor [extract_qor [file join $out timing.log]]
puts "QOR: $qor"
