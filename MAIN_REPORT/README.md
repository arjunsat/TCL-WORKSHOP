vsdsynth.tcl Script Explanation
## 1. Script Setup and Input Validation

#!/bin/env tclsh

set enable_prelayout_timing 1
set working_dir [exec pwd]
set vsd_array_length [llength [split [lindex $argv 0] .]]
set input [lindex [split [lindex $argv 0] .] [expr {$vsd_array_length-1}]]

if {![regexp {^csv} $input] || $argc != 1} {
    puts "Error in usage"
    puts "Usage: ./vsdsynth <.csv>"
    puts "where <.csv> file has below inputs"
    exit
} else {





This section initializes the Tcl interpreter and validates how the script is executed. It ensures that the user provides a .csv file as input and checks that the usage format is correct. If the file is missing or incorrectly specified, it prints an error message and exits. This prevents the rest of the flow from running with invalid inputs.

## 2. CSV Parsing and Variable Initialization


set columns [m columns]
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
puts "\nInfo: Below are the list of initial variables and their values."
puts "DesignName = $DesignName"
puts "OutputDirectory = $OutputDirectory"
puts "NetlistDirectory = $NetlistDirectory"
puts "EarlyLibraryPath = $EarlyLibraryPath"
puts "LateLibraryPath = $LateLibraryPath"
puts "ConstraintsFile = $ConstraintsFile"


The script reads the provided CSV file and converts its contents into a Tcl matrix. Each line in the file defines a design parameter, such as the design name, output directory, or library path. These are stored as Tcl variables, allowing the rest of the script to use them easily. It prints each variable as it’s set, helping users confirm that the input has been read correctly.

## 3. File and Directory Existence Checks

Before synthesis begins, the script checks that all required files and directories exist. This includes verifying the early and late timing libraries, ensuring the output directory exists (or creating it if not), and checking the presence of the netlist and constraint files. If any of these are missing, the script exits safely with a clear message.

## 4. Constraints File Reading

This part loads the design constraints CSV into memory. The constraints file usually includes information on clocks, input/output delays, and other timing requirements. The script determines where the “CLOCKS,” “INPUTS,” and “OUTPUTS” sections are located so it can later generate an SDC file that reflects the same structure.

## 5. Clock Section Setup

The script searches the constraints matrix for rows and columns that define clock timing parameters. These include early and late rise/fall delays and slews. It records the positions of these entries for use in subsequent SDC generation steps.

## 6. Clock Delay and Slew Setup Continuation

Here, the script finalizes the mapping of clock parameters. It ensures that every combination of delay and slew (early, late, rise, fall) is captured and stored in variables. This mapping is crucial because these references will later be used to generate precise timing constraints in the output file.

## 7. Generating Clock Constraints in SDC

This section writes the first part of the SDC (Synopsys Design Constraints) file. It loops through each clock entry in the constraints matrix and creates commands such as create_clock, set_clock_transition, and set_clock_latency. This automates the creation of detailed clock definitions required for synthesis and STA tools.

## 8. Hierarchy Check Script Creation

The next phase ensures that all RTL modules are correctly connected. The script generates a temporary Yosys script that loads the library, reads all Verilog files, and performs a hierarchy -check operation. This check ensures that no module instantiations are missing or mismatched before synthesis.

## 9. Hierarchy Check Validation

After running the Yosys hierarchy script, the log file is analyzed. The script searches for any error patterns indicating undefined modules or missing references. If issues are found, they are printed to the terminal. Otherwise, it confirms that the hierarchy check passed successfully.

## 10. Main Synthesis Script Creation

This section builds the main Yosys synthesis script. It reads all Verilog files, sets the top module, performs synthesis, technology mapping, and writes out the gate-level netlist. After generating the script, it runs Yosys and logs all messages, producing a synthesized version of the design.

## 11. Static Timing Analysis Setup

Once synthesis is done, the script moves to the timing analysis phase. It loads necessary Tcl procedures for OpenTimer, sets up multi-threading, and initializes the OpenTimer configuration file. Then it reads the early and late libraries, the synthesized Verilog netlist, and the SDC constraints.

## 12. Dummy SPEF File Generation

Because physical layout is not yet available, this section creates a dummy SPEF file with zero-wire parasitics. The file includes standard IEEE headers defining units for time, resistance, capacitance, and inductance. This allows OpenTimer to run pre-layout analysis without physical interconnect data.

## 13. Running OpenTimer

Here the script appends timing commands such as init_timer, report_timer, and report_wns to the configuration file. It executes OpenTimer using the generated configuration, measures total runtime, and reports completion. The output results file contains all the timing data for the analyzed design.

## 14. Output Violation Extraction

This part parses the OpenTimer results to find output-related (RAT) timing violations. It scans for the keyword “RAT,” extracts the worst slack value, and counts how many violations exist. These metrics indicate whether the design meets its output timing requirements.

## 15. Setup Timing Violation Extraction

The script continues by searching for “Setup” violations in the timing results. It records the worst setup slack and the total number of violations found. This data shows if there are any paths that fail to meet the setup time requirements.

## 16. Hold Timing Violation Extraction

This section performs a similar analysis for hold violations. It searches for “Hold” entries in the results, identifies the worst hold slack, and counts how many paths violate hold constraints. These values are crucial for verifying that the design maintains signal stability after the clock edge.

## 17. Instance Count Extraction

The script then finds the number of gates or instances synthesized in the design. It looks for a line containing the pattern “Num of gates” and extracts the count. This provides a measure of design complexity and will appear in the final report summary.

## 18. Preparing Pre-Layout Timing Report

With all timing data collected, the script prepares a formatted summary table. It defines column headers such as runtime, instance count, worst setup and hold slacks, and violation counts. This creates a structured, easy-to-read summary for the user.

## 19. Printing the Results Summary

The script populates the report table with actual data from the previous analysis. It prints one formatted line summarizing runtime, instance count, and timing results for the design. This summary acts as a concise overview of synthesis and STA outcomes.

## 20. Closing the Report

The final section prints a closing separator line to complete the report format. It leaves a blank line for readability and signifies that the entire synthesis and timing analysis process has completed successfully.
