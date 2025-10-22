# VSDSYNTH — TCL-Driven Synthesis & STA Automation (Yosys + OpenTimer)

I built a script-first flow that turns a **CSV/SDC design spec** into clean **SDC**, synthesizes RTL with **Yosys**, prepares inputs for **OpenTimer**, and finally extracts and formats **QOR**. The entrypoint is a small wrapper `vsdsynth` that validates inputs and hands off to my TCL driver `vsdsynth.tcl`.

---

## What this repository is about

- A practical TCL toolbox that **parses design/config CSVs**, **generates constraints**, and **orchestrates tools**.
- End-to-end automation: **CSV → SDC → Yosys (netlist) → OpenTimer (timing) → QOR**.
- Reusable **procs** for constraint translation, path setup, multi-threading, and report extraction.

---

## Highlights 
- ✅ `vsdsynth` wrapper with robust **argument checks** and a clear **`-help` usage**.
- ✅ CSV → **matrix/array** parsing; **auto-created variables** (`DesignName`, `OutputDirectory`, `NetlistDirectory`, `EarlyLibraryPath`, `LateLibraryPath`, `ConstraintsFile`).
- ✅ **Constraint extraction** from CSV: clocks, inputs, outputs; **bus/bit** handling; SDC emitted in **write** mode.
- ✅ **Yosys** integration: hierarchy checks, script generation, synthesis, logs.
- ✅ **OpenTimer** prep: netlist cleanup, SDC→OT formatting, `.spef/.conf` prep where needed.
- ✅ **QOR** extraction: WNS/TNS/runtime and formatted result summaries.

---

---

## Quick start


# make entrypoint executable
chmod +x scripts/vsdsynth

# run with your design CSV
./scripts/vsdsynth constraints/examples/<your_design>_constraints.csv


CSV

<img width="976" height="530" alt="image" src="https://github.com/user-attachments/assets/e7511c7b-28ce-4683-a7fe-7a05dc9de90f" />


1) Wrapper & usage scenarios

Wrote vsdsynth to:

Print USAGE when no file is provided or -help is used.

Error out with a helpful message if the CSV is missing or unreadable.

Delegate to TCL: tclsh scripts/vsdsynth.tcl <csv>.

Verified three scenarios:

no file given,

user asks for help,

CSV path provided but doesn’t exist (clean exit with message).

2) Variable creation from CSV

Created vsdsynth.tcl and integrated CSV → matrix → array parsing.

Auto-created variables from header/value pairs; removed spaces in keys
(e.g., "Design Name" → DesignName).

Normalized all paths (file normalize) so the flow works from any location.

Added checks to fail fast if critical paths are wrong.

3) Constraint processing (CSV → SDC)

Converted the constraints CSV into a matrix, counted rows/columns, and located start indices for sections (ports, clocks, inputs, outputs).

Found column indices for parameters (e.g., early_rise_delay, late_fall_delay).

Emitted SDC in write mode:

create_clock (period/duty),

set_clock_latency (source/propagated),

set_clock_transition (slew),

set_input_delay / set_output_delay with proper clock association.

Implemented bus/bit detection and “bit-blasting” where needed (* suffix or pattern matching).

4) Yosys flow

Generated a Yosys script programmatically (read liberty, read RTL, set top, synth, write netlist).

Ran hierarchy checks; logged failures; added debug prints for quick fixes.

Collected synthesis logs and stats in reports/yosys/.

5) Procs (reusable building blocks)

Wrote TCL procs to:

set library paths / netlist directories,

adjust SDC for OpenTimer expectations,

configure standard output & thread count,

run small validation tasks without code repetition.

6) Netlist & SDC preparation for OpenTimer

Cleaned synthesized verilog (removed *, \, stray quotes) before STA ingest.

Converted/filtered SDC into OpenTimer-friendly form, and generated related config/artifact files as needed.

7) STA & QOR

Invoked OpenTimer with prepared inputs; captured logs/results.

Parsed results via TCL to extract WNS/TNS/runtime and formatted the final output for quick review.




THE TCL FILE


#! /bin/env tclsh
#-----------------------------------------------------------#
#----- Checks whether vsdsynth usage is correct or not -----#
#-----------------------------------------------------------#
set enable_prelayout_timing 1
set working_dir [exec pwd]
set vsd_array_length [llength [split [lindex $argv 0] .]]
set input [lindex [split [lindex $argv 0] .] $vsd_array_length-1]

if {![regexp {^csv} $input] || $argc!=1 } {
	puts "Error in usage"
	puts "Usage: ./vsdsynth <.csv>"
	puts "where <.csv> file has below inputs"
	exit
} else {
#-----------------------------------------------------------------------------------------------------------------------------------------------------#
#------ converts .csv to matrix and creates initial variables "DesignName OutputDirectory NetlistDirectory EarlyLibraryPath LateLibraryPath"----------#
#----------- If you are modifying this script, please use above variables as starting point. Use "puts" command to report above variables-------------#
#-----------------------------------------------------------------------------------------------------------------------------------------------------#
	set filename [lindex $argv 0]
	package require csv
	package require struct::matrix
	struct::matrix m
	set f [open $filename]
	csv::read2matrix $f m , auto
	close $f
	set columns [m columns]
	#m add columns $columns
	m link my_arr
	set num_of_rows [m rows]
	set i 0
	while {$i < $num_of_rows} {
		 puts "\nInfo: Setting $my_arr(0,$i) as '$my_arr(1,$i)'"
		 if {$i == 0} {
			 set [string map {" " ""} $my_arr(0,$i)] $my_arr(1,$i)
		 } else {
			 set [string map {" " ""} $my_arr(0,$i)] [file normalize $my_arr(1,$i)]
		 }
		  set i [expr {$i+1}]
	}
} 

puts "\nInfo: Below are the list of initial variables and their values. User can use these variables for further debug. Use 'puts <variable name>' command to query value of below variables"
puts "DesignName = $DesignName"
puts "OutputDirectory = $OutputDirectory"
puts "NetlistDirectory = $NetlistDirectory"
puts "EarlyLibraryPath = $EarlyLibraryPath"
puts "LateLibraryPath = $LateLibraryPath"
puts "ConstraintsFile = $ConstraintsFile"
#-------------------------------------------------------------------------------------------#
#-----Below script checks if directories and files mentioned in csv file, exists or not-----#
#-------------------------------------------------------------------------------------------#


if {! [file exists $EarlyLibraryPath] } {
	puts "\nError: Cannot find early cell library in path $EarlyLibraryPath. Exiting... "
	exit
} else {
	puts "\nInfo: Early cell library found in path $EarlyLibraryPath"
}


if {! [file exists $LateLibraryPath]} {
        puts "\nError: Cannot find late cell library in path $LateLibraryPath. Exiting... "
        exit
} else {
	puts "\nInfo: Late cell library found in path $LateLibraryPath"
}

if {![file isdirectory $OutputDirectory]} {
	puts "\nInfo: Cannot find output directory $OutputDirectory. Creating $OutputDirectory"
	file mkdir $OutputDirectory
} else {
	puts "\nInfo: Output directory found in path $OutputDirectory"
}

if {! [file isdirectory $NetlistDirectory]} {
	puts "\nError: Cannot find RTL netlist directory in path $NetlistDirectory. Exiting..."
	exit	
} else {
	puts "\nInfo: RTL netlist directory found in path $NetlistDirectory"
}

if {! [file exists $ConstraintsFile] } {
        puts "\nError: Cannot find constraints file in path $ConstraintsFile. Exiting... "
        exit
} else {
        puts "\nInfo: Constraints file found in path $ConstraintsFile"
}
#----------------------------------------------------------------------------#
#----------------------  Constraints FILE creations--------------------------#
#----------------------------- SDC Format -----------------------------------#
#----------------------------------------------------------------------------#
puts "\nInfo: Dumping SDC constraints for $DesignName"
::struct::matrix constraints
set chan [open $ConstraintsFile]
csv::read2matrix $chan constraints  , auto
close $chan
set number_of_rows [constraints rows]
set number_of_columns [constraints columns]

#-----check row number for "clocks" and column number for "IO delays and slew section" in constraints.csv---##
set clock_start [lindex [lindex [constraints search all CLOCKS] 0 ] 1]
set clock_start_column [lindex [lindex [constraints search all CLOCKS] 0 ] 0]

#-----check row number for "inputs" section in constraints.csv---##
set input_ports_start [lindex [lindex [constraints search all INPUTS] 0 ] 1]

#-----check row number for "outputs" section in constraints.csv---##
set output_ports_start [lindex [lindex [constraints search all OUTPUTS] 0 ] 1]

#-------------------clock constraints--------------------##
#-------------------clock latency constraints------------#

set clock_early_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  early_rise_delay] 0 ] 0]

set clock_early_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  early_fall_delay] 0 ] 0]

set clock_late_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  late_rise_delay] 0 ] 0]

set clock_late_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  late_fall_delay] 0 ] 0]

#-------------------clock transition constraints------------#

set clock_early_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  early_rise_slew] 0 ] 0]

set clock_early_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  early_fall_slew] 0 ] 0]

set clock_late_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  late_rise_slew] 0 ] 0]

set clock_late_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$number_of_columns-1}] [expr {$input_ports_start-1}]  late_fall_slew] 0 ] 0]

set sdc_file [open $OutputDirectory/$DesignName.sdc "w"]
set i [expr {$clock_start+1}]
set end_of_ports [expr {$input_ports_start-1}]
puts "\nInfo-SDC: Working on clock constraints....."
while { $i < $end_of_ports } {
        puts -nonewline $sdc_file "\ncreate_clock -name [constraints get cell 0 $i] -period [constraints get cell 1 $i] -waveform \{0 [expr {[constraints get cell 1 $i]*[constraints get cell 2 $i]/100}]\} \[get_ports [constraints get cell 0 $i]\]"
	puts -nonewline $sdc_file "\nset_clock_transition -rise -min [constraints get cell $clock_early_rise_slew_start $i] \[get_clocks [constraints get cell 0 $i]\]"
	puts -nonewline $sdc_file "\nset_clock_transition -fall -min [constraints get cell $clock_early_fall_slew_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_transition -rise -max [constraints get cell $clock_late_rise_slew_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_transition -fall -max [constraints get cell $clock_late_fall_slew_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_latency -source -early -rise [constraints get cell $clock_early_rise_delay_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_latency -source -early -fall [constraints get cell $clock_early_fall_delay_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_latency -source -late -rise [constraints get cell $clock_late_rise_delay_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        puts -nonewline $sdc_file "\nset_clock_latency -source -late -fall [constraints get cell $clock_late_fall_delay_start $i] \[get_clocks [constraints get cell 0 $i]\]"
        set i [expr {$i+1}]
}
#------------------------------------------------------------------------------##
#-------------------create input delay and slew constraints--------------------##
#------------------------------------------------------------------------------##

set input_early_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  early_rise_delay] 0 ] 0]
set input_early_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  early_fall_delay] 0 ] 0]
set input_late_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  late_rise_delay] 0 ] 0]
set input_late_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  late_fall_delay] 0 ] 0]

set input_early_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  early_rise_slew] 0 ] 0]
set input_early_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  early_fall_slew] 0 ] 0]
set input_late_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  late_rise_slew] 0 ] 0]
set input_late_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  late_fall_slew] 0 ] 0]


set related_clock [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$number_of_columns-1}] [expr {$output_ports_start-1}]  clocks] 0 ] 0]
set i [expr {$input_ports_start+1}]
set end_of_ports [expr {$output_ports_start-1}]
puts "\nInfo-SDC: Working on IO constraints....."
puts "\nInfo-SDC: Categorizing input ports as bits and bussed"

while { $i < $end_of_ports } {
#--------------------------optional script----differentiating input ports as bussed and bits------#
set netlist [glob -dir $NetlistDirectory *.v]
set tmp_file [open /tmp/1 w]
foreach f $netlist {
        set fd [open $f]
	puts "reading file $f"
        while {[gets $fd line] != -1} {
		set pattern1 " [constraints get cell 0 $i];"
                if {[regexp -all -- $pattern1 $line]} {
			puts "pattern1 \"$pattern1\" found and matching line in verilog file \"$f\" is \"$line\""
			set pattern2 [lindex [split $line ";"] 0]
			puts "creating pattern2 by splitting pattern1 using semi-colon as delimiter => \"$pattern2\""
			if {[regexp -all {input} [lindex [split $pattern2 "\S+"] 0]]} {
			puts "out of all patterns, \"$pattern2\" has matching string \"input\". So preserving this line and ignoring others"	
			set s1 "[lindex [split $pattern2 "\S+"] 0] [lindex [split $pattern2 "\S+"] 1] [lindex [split $pattern2 "\S+"] 2]"
			puts "printing first 3 elements of pattern2 as \"$s1\" using space as demiliter"
			puts -nonewline $tmp_file "\n[regsub -all {\s+} $s1 " "]"
			puts "replace multiple spaces in s1 by single space and reformat as \"[regsub -all {\s+} $s1 " "]\""
			}
                }
        }
close $fd
}
close $tmp_file
set tmp_file [open /tmp/1 r]
#puts "reading [read $tmp_file]"
#puts "reading /tmp/1 file as [split [read $tmp_file] \n]"
#puts "sorting /tmp/1 contents as [lsort -unique [split [read $tmp_file] \n ]]"
#puts "joining /tmp/1 as [join [lsort -unique [split [read $tmp_file] \n ]] \n]"
set tmp2_file [open /tmp/2 w]
puts -nonewline $tmp2_file "[join [lsort -unique [split [read $tmp_file] \n]] \n]"
close $tmp_file
close $tmp2_file
set tmp2_file [open /tmp/2 r]
#puts "count is  [llength [read $tmp2_file]] "
set count [llength [read $tmp2_file]]
#puts "splitting content of tmp_2 using space and counting number of elements as $count"
if {$count > 2} {
	set inp_ports [concat [constraints get cell 0 $i]*]
	puts "bussed"
} else {
	set inp_ports [constraints get cell 0 $i]
	puts "not bussed"
}
	puts "input port name is $inp_ports since count is $count\n"
        puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -rise -source_latency_included [constraints get cell $input_early_rise_delay_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -fall -source_latency_included [constraints get cell $input_early_fall_delay_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -rise -source_latency_included [constraints get cell $input_late_rise_delay_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -fall -source_latency_included [constraints get cell $input_late_fall_delay_start $i] \[get_ports $inp_ports\]"

        puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -rise -source_latency_included [constraints get cell $input_early_rise_slew_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -fall -source_latency_included [constraints get cell $input_early_fall_slew_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -rise -source_latency_included [constraints get cell $input_late_rise_slew_start $i] \[get_ports $inp_ports\]"
        puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -fall -source_latency_included [constraints get cell $input_late_fall_slew_start $i] \[get_ports $inp_ports\]"


        set i [expr {$i+1}]
}
close $tmp2_file
#------------------------------------------------------------------------------##
#-------------------create output delay and load constraints--------------------##
#------------------------------------------------------------------------------##

set output_early_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  early_rise_delay] 0 ] 0]
set output_early_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  early_fall_delay] 0 ] 0]
set output_late_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  late_rise_delay] 0 ] 0]
set output_late_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  late_fall_delay] 0 ] 0]
set output_load_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  load] 0 ] 0]
set related_clock [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$number_of_columns-1}] [expr {$number_of_rows-1}]  clocks] 0 ] 0]
set i [expr {$output_ports_start+1}]
set end_of_ports [expr {$number_of_rows}]
puts "\nInfo-SDC: Working on IO constraints....."
puts "\nInfo-SDC: Categorizing output ports as bits and bussed"

while { $i < $end_of_ports } {
#--------------------------optional script----differentiating input ports as bussed and bits------#
set netlist [glob -dir $NetlistDirectory *.v]
set tmp_file [open /tmp/1 w]
foreach f $netlist {
        set fd [open $f]
        while {[gets $fd line] != -1} {
                set pattern1 " [constraints get cell 0 $i];"
                if {[regexp -all -- $pattern1 $line]} {
                        set pattern2 [lindex [split $line ";"] 0]
                        if {[regexp -all {output} [lindex [split $pattern2 "\S+"] 0]]} {
                        set s1 "[lindex [split $pattern2 "\S+"] 0] [lindex [split $pattern2 "\S+"] 1] [lindex [split $pattern2 "\S+"] 2]"
                        puts -nonewline $tmp_file "\n[regsub -all {\s+} $s1 " "]"
                        }
                }
        }
close $fd
}
close $tmp_file
set tmp_file [open /tmp/1 r]
set tmp2_file [open /tmp/2 w]
puts -nonewline $tmp2_file "[join [lsort -unique [split [read $tmp_file] \n]] \n]"
close $tmp_file
close $tmp2_file
set tmp2_file [open /tmp/2 r]
set count [split [llength [read $tmp2_file]] " "]
if {$count > 2} {
        set op_ports [concat [constraints get cell 0 $i]*]
} else {
        set op_ports [constraints get cell 0 $i]
}
        puts -nonewline $sdc_file "\nset_output_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -rise -source_latency_included [constraints get cell $output_early_rise_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -fall -source_latency_included [constraints get cell $output_early_fall_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -rise -source_latency_included [constraints get cell $output_late_rise_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -fall -source_latency_included [constraints get cell $output_late_fall_delay_start $i] \[get_ports $op_ports\]"
	puts -nonewline $sdc_file "\nset_load [constraints get cell $output_load_start $i] \[get_ports $op_ports\]"
	set i [expr {$i+1}]
}
close $tmp2_file
close $sdc_file

puts "\nInfo: SDC created. Please use constraints in path  $OutputDirectory/$DesignName.sdc"
return

#----------------------------------------------------------------------------#
#--------------------------- Hierarchy check --------------------------------#
#----------------------------------------------------------------------------#

puts "\nInfo: Creating hierarchy check script to be used by Yosys"
set data "read_liberty -lib -ignore_miss_dir -setattr blackbox ${LateLibraryPath}"
set filename "$DesignName.hier.ys"
set fileId [open $OutputDirectory/$filename "w"]
puts -nonewline $fileId $data

set netlist [glob -dir $NetlistDirectory *.v]
foreach f $netlist {
	set data $f
	puts -nonewline $fileId "\nread_verilog $f"
}
puts -nonewline $fileId "\nhierarchy -check"
close $fileId

puts "\nChecking hierarchy....."

if { [catch { exec yosys -s $OutputDirectory/$DesignName.hier.ys >& $OutputDirectory/$DesignName.hierarchy_check.log} msg]} {
	set filename "$OutputDirectory/$DesignName.hierarchy_check.log"
	set pattern {referenced in module}
	set count 0
	set fid [open $filename r]
	while {[gets $fid line] != -1} {
		incr count [regexp -all -- $pattern $line]
		if {[regexp -all -- $pattern $line]} {
			puts "\nError: module [lindex $line 2] is not part of design $DesignName. Please correct RTL in the path '$NetlistDirectory'" 
			puts "\nInfo: Hierarchy check FAIL"
		}
	}
	close $fid
} else {
	puts "\nInfo: Hierarchy check PASS"
}
puts "\nInfo: Please find hierarchy check details in [file normalize $OutputDirectory/$DesignName.hierarchy_check.log] for more info"
cd $working_dir

#----------------------------------------------------------------------------#
#------------------------- Main synthesis script-----------------------------#
#----------------------------------------------------------------------------#
puts "\nInfo: Creating main synthesis script to be used by Yosys"
set data "read_liberty -lib -ignore_miss_dir -setattr blackbox ${LateLibraryPath}"
set filename "$DesignName.ys"
set fileId [open $OutputDirectory/$filename "w"]
puts -nonewline $fileId $data

set netlist [glob -dir $NetlistDirectory *.v]
foreach f $netlist {
        set data $f
        puts -nonewline $fileId "\nread_verilog $f"
}
puts -nonewline $fileId "\nhierarchy -top $DesignName"
puts -nonewline $fileId "\nsynth -top $DesignName"
puts -nonewline $fileId "\nsplitnets -ports -format ___\nproc; memory; opt; fsm; opt\ntechmap; opt\ndfflibmap -liberty ${LateLibraryPath}"
puts -nonewline $fileId "\nabc -liberty ${LateLibraryPath} "
puts -nonewline $fileId "\nflatten"
puts -nonewline $fileId "\nclean -purge\niopadmap -outpad BUFX2 A:Y -bits\nopt\nclean"
puts -nonewline $fileId "\nwrite_verilog $OutputDirectory/$DesignName.synth.v"
close $fileId
puts "\nInfo: Synthesis script created and can be accessed from path $OutputDirectory/$DesignName.ys"

puts "\nInfo: Running synthesis........"


#----------------------------------------------------------------------------#
#-------------------- Run synthesis script using yosys ----------------------#
#----------------------------------------------------------------------------#


if {[catch { exec yosys -s $OutputDirectory/$DesignName.ys >& $OutputDirectory/$DesignName.synthesis.log} msg]} {
	puts "\nError: Synthesis failed due to errors. Please refer to log $OutputDirectory/$DesignName.synthesis.log for errors"
	exit
} else {
	puts "\nInfo: Synthesis finished successfully"
}
puts "\nInfo: Please refer to log $OutputDirectory/$DesignName.synthesis.log"

#---------------------------------------------------------------------------------------------#
#------------------------- Edit synth.v to be usable by Opentimer-----------------------------#
#---------------------------------------------------------------------------------------------#

set fileId [open /tmp/1 "w"]
puts -nonewline $fileId [exec grep -v -w "*" $OutputDirectory/$DesignName.synth.v]
close $fileId

set output [open $OutputDirectory/$DesignName.final.synth.v "w"]

set filename "/tmp/1"
set fid [open $filename r] 
        while {[gets $fid line] != -1} {
	puts -nonewline $output [string map {"\\" ""} $line]
	puts -nonewline $output "\n"
}
close $fid
close $output

puts "\nInfo: Please find the synthesized netlist for $DesignName at below path. You can use this netlist for STA or PNR"
puts "\n$OutputDirectory/$DesignName.final.synth.v"

#--------------------------------------------------------------------------------------------------#
#-------------------------Static timing analysis using Opentimer-----------------------------------#
#--------------------------------------------------------------------------------------------------#
puts "\nInfo: Timing Analysis Started...."
puts "\nInfo: Initializing number of threads, libraries, sdc, verilog netlist path..."
source /home/kunalg/Desktop/vsdflow/procs/reopenStdout.proc
source /home/kunalg/Desktop/vsdflow/procs/set_num_threads.proc
reopenStdout $OutputDirectory/$DesignName.conf
set_multi_cpu_usage -localCpu 4

source /home/kunalg/Desktop/vsdflow/procs/read_lib.proc
read_lib -early /home/kunalg/Desktop/work/openmsp430/openmsp430/osu018_stdcells.lib 

read_lib -late /home/kunalg/Desktop/work/openmsp430/openmsp430/osu018_stdcells.lib

source /home/kunalg/Desktop/vsdflow/procs/read_verilog.proc
read_verilog $OutputDirectory/$DesignName.final.synth.v

source /home/kunalg/Desktop/vsdflow/procs/read_sdc.proc
read_sdc $OutputDirectory/$DesignName.sdc
reopenStdout /dev/tty

if {$enable_prelayout_timing == 1} {
	puts "\nInfo: enable_prelayout_timing is $enable_prelayout_timing. Enabling zero-wire load parasitics"
	set spef_file [open $OutputDirectory/$DesignName.spef w]
puts $spef_file "*SPEF \"IEEE 1481-1998\" " 
puts $spef_file "*DESIGN \"$DesignName\" " 
puts $spef_file "*DATE \"Tue Sep 25 11:51:50 2012\" " 
puts $spef_file "*VENDOR \"TAU 2015 Contest\" " 
puts $spef_file "*PROGRAM \"Benchmark Parasitic Generator\" " 
puts $spef_file "*VERSION \"0.0\" " 
puts $spef_file "*DESIGN_FLOW \"NETLIST_TYPE_VERILOG\" " 
puts $spef_file "*DIVIDER / " 
puts $spef_file "*DELIMITER : " 
puts $spef_file "*BUS_DELIMITER [ ] " 
puts $spef_file "*T_UNIT 1 PS " 
puts $spef_file "*C_UNIT 1 FF " 
puts $spef_file "*R_UNIT 1 KOHM " 
puts $spef_file "*L_UNIT 1 UH " 
}
close $spef_file

set conf_file [open $OutputDirectory/$DesignName.conf a]
puts $conf_file "set_spef_fpath $OutputDirectory/$DesignName.spef"
puts $conf_file "init_timer "
puts $conf_file "report_timer "
puts $conf_file "report_wns "
puts $conf_file "report_worst_paths -numPaths 10000 "
close $conf_file

set time_elapsed_in_us [time {exec /home/kunalg/Desktop/tools/opentimer/OpenTimer-1.0.5/bin/OpenTimer < $OutputDirectory/$DesignName.conf >& $OutputDirectory/$DesignName.results}]
set time_elapsed_in_sec "[expr {[lindex $time_elapsed_in_us 0]/100000}]sec"
puts "\nInfo: STA finished in $time_elapsed_in_sec seconds"
puts "\nInfo: Refer to $OutputDirectory/$DesignName.results for warnings and errors"
set tcl_precision 3

#-----find worst output violation------#
set worst_RAT_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r]
set pattern {RAT}
while {[gets $report_file line] != -1} {
        if {[regexp $pattern $line]} {
        set worst_RAT_slack  "[expr {[lindex [join $line " "] 3]/1000}]ns"
        break
        } else {
        continue
        }
}
close $report_file

#-----find number of output violation------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
        incr count [regexp -all -- $pattern $line]
}
set Number_output_violations $count
close $report_file

#-----find worst setup violation------#
set worst_negative_setup_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r]
set pattern {Setup}
while {[gets $report_file line] != -1} {
        if {[regexp $pattern $line]} {
        set worst_negative_setup_slack "[expr {[lindex [join $line " "] 3]/1000}]ns"
        break
        } else {
        continue
        }
}
close $report_file


#-----find number of setup violation------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
        incr count [regexp -all -- $pattern $line]
}       
set Number_of_setup_violations $count
close $report_file

#-----find worst hold violation------#
set worst_negative_hold_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r]
set pattern {Hold}
while {[gets $report_file line] != -1} {
        if {[regexp $pattern $line]} {
        set worst_negative_hold_slack "[expr {[lindex [join $line " "] 3]/1000}]ns"
        break
        } else {
        continue
        }
}
close $report_file


#-----find number of hold violation------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
        incr count [regexp -all -- $pattern $line]
}
set Number_of_hold_violations $count
close $report_file

#-----find number of instance------#
set pattern {Num of gates}
set report_file [open $OutputDirectory/$DesignName.results r]
while {[gets $report_file line] != -1} {
        if {[regexp -all -- $pattern $line]} {
        set Instance_count [lindex [join $line " "] 4 ]
        break
        } else {
        continue
        }
}
close $report_file

puts "\n"
puts "                                         ****PRELAYOUT TIMING RESULTS****                                                  "
set formatStr {%15s%15s%15s%15s%15s%15s%15s%15s%15s}

puts [format $formatStr "-----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]
puts [format $formatStr "Design Name" "Runtime" "Instance Count" "WNS setup" "FEP Setup" "WNS Hold" "FEP Hold" "WNS RAT" "FEP RAT"]
puts [format $formatStr "-----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]

foreach design_name $DesignName runtime $time_elapsed_in_sec instance_count $Instance_count wns_setup $worst_negative_setup_slack fep_setup $Number_of_setup_violations wns_hold $worst_negative_hold_slack fep_hold $Number_of_hold_violations wns_rat $worst_RAT_slack fep_rat $Number_output_violations {
        puts [format $formatStr $design_name $runtime $instance_count $wns_setup $fep_setup $wns_hold $fep_hold $wns_rat $fep_rat]
}

puts [format $formatStr "-----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]
puts "\n"




## THE VDSYNTH Shell  script

#!/bin/tcsh -f
echo
echo
echo "***                  ***    ***********     **********         ***********    ****       ****  *******        **** ************** ****     ****"
echo " ***                ***   ***************   ***********      ***************   ****     ****   ********       **** ************** ****     ****"
echo "  ***              ***    ****     	    ***      ****    **** 	        *****  ****    **** ****      ****      ****      ****     ****"
echo "   ***            ***     *****		    ***       ****   *****	         *********     ****  ****     ****      ****      ****     ****"
echo "    ***          ***      *************	    ***        ****  *************        *******      ****   ****    ****      ****      *************"
echo "     ***        ***       ***************   ***       ****   ***************       ******      ****    ****   ****      ****      *************"
echo "      ***      ***                 ******   ***       ****            ******  	   ******      ****     ****  ****      ****      ****     ****"
echo "       ***    ***                   *****   ***      ****              *****       ******      ****      **** ****      ****      ****     ****"
echo "        ***  ***          ***************   ***********      ***************       ******      ****       ********      ****      ****     ****"
echo "         ******             ***********     **********         ***********         ******      ****        *******      ****      ****     ****"
echo ""
echo "			An unique User Interface (UI) that will take RTL netlist & SDC constraints as an input, and will generate "
echo "			sythnesized netlist & pre-layout timing report as an output. It uses Yosys open-source tool for synthesis"
echo "						and Opentimer to generate pre-layout timing reports."
echo " "
echo "					Developed and Maintained by VLSI System Design Corporation Pvt. Ltd."
echo "				       For any queries and bugs, please drop a mail to kunalpghosh@gmail.com"
echo ""
echo "						 ********* A vlsisystemdesign.com initiative *********	"
echo 

set my_work_dir = `pwd`

#---------------------------------------------------------------------#
#------------------------ Tool initialization ------------------------#
#---------------------------------------------------------------------#

if ($#argv != 1) then 
	echo "Info: Please provide the csv file"
	exit 1
endif

if (! -f $argv[1] || $argv[1] == "-help") then
        if ($argv[1] != "-help") then
                echo "Error:  Cannot find csv file $argv[1]. Exiting..."
                exit 1
        else
                echo USAGE: ./vsdsynth \<csv file\>
                echo
                echo         where \<csv file\> consists of 2 columns, below keyword being in 1st column and is Case Sensitive. Please request VSD team for sample csv file
                echo
                echo         \<Design Name\> is the name of top level module
                echo
                echo         \<Output Directory\> is the name of output directory where you want to dump synthesis script, synthesized netlist and timing reports
                echo
                echo         \<Netlist Directory\> is the name of directory where all RTL netlist are present
                echo
                echo         \<Early Library Path\> is the file path of the early cell library to be used for STA
                echo
                echo         \<Late Library Path\> is file path of the late cell library to be used for STA
                echo
                echo         \<Constraints file\> is csv file path of constraints to be used for STA
                echo
                exit 1
        endif
else
		tclsh vsdsynth.tcl $argv[1]
endif


## The tasks

<img width="940" height="505" alt="image" src="https://github.com/user-attachments/assets/495fe56f-5f36-4bd4-8461-057a65386986" />

## vsdsynth 
<img width="852" height="269" alt="image" src="https://github.com/user-attachments/assets/86dc8bda-e1f0-4b72-bd5c-39f2586dbcc1" />

<img width="940" height="844" alt="image" src="https://github.com/user-attachments/assets/89440299-2e75-40e1-ad7b-d50c90364f39" />


<img width="940" height="238" alt="image" src="https://github.com/user-attachments/assets/185db1ea-a96e-4497-ba5c-0a0d30fe2b1b" />

<img width="940" height="206" alt="image" src="https://github.com/user-attachments/assets/0098dd03-a48a-4331-903b-11de38fb58d7" />

Acknowledgements

Thanks to VLSI System Design (VSD) for the guidance and the learning environment that pushed me to build a clean, automated TCL flow from scratch.

Bibliography (short list)

Tcl — Tool Command Language (language guide & best practices)

Yosys — Open-source Verilog synthesis

OpenTimer — Open-source static timing analysis

