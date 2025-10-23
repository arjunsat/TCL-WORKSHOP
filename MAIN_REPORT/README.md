vsdsynth.tcl Script Explanation
## EXPLANATION OF THE PROJECT SOURE CODE USED BY THE INSTRUCTOR FOR THE WORKSHOP

## 1. Script Setup and Input Validation


<img width="908" height="491" alt="image" src="https://github.com/user-attachments/assets/4fec04f7-8fb9-4960-a39c-6b3dda0b52f4" />



This section initializes the Tcl interpreter and validates how the script is executed. It ensures that the user provides a .csv file as input and checks that the usage format is correct. If the file is missing or incorrectly specified, it prints an error message and exits. This prevents the rest of the flow from running with invalid inputs.

## 2. CSV Parsing and Variable Initialization

<img width="933" height="718" alt="image" src="https://github.com/user-attachments/assets/1ffcd2e0-54fc-4da4-80a3-74a5702f1869" />




The script reads the provided CSV file and converts its contents into a Tcl matrix. Each line in the file defines a design parameter, such as the design name, output directory, or library path. These are stored as Tcl variables, allowing the rest of the script to use them easily. It prints each variable as it’s set, helping users confirm that the input has been read correctly.

## 3. File and Directory Existence Checks

<img width="1307" height="771" alt="image" src="https://github.com/user-attachments/assets/1202171b-3828-4bbe-8e3a-361f689db8c4" />


Before synthesis begins, the script checks that all required files and directories exist. This includes verifying the early and late timing libraries, ensuring the output directory exists (or creating it if not), and checking the presence of the netlist and constraint files. If any of these are missing, the script exits safely with a clear message.

## 4. Constraints File Reading

<img width="1035" height="427" alt="image" src="https://github.com/user-attachments/assets/a72951bf-8779-4245-bc7c-b7bf12a6ecb4" />




This part loads the design constraints CSV into memory. The constraints file usually includes information on clocks, input/output delays, and other timing requirements. The script determines where the “CLOCKS,” “INPUTS,” and “OUTPUTS” sections are located so it can later generate an SDC file that reflects the same structure.

## 5. Clock Section Setup
<img width="1278" height="417" alt="image" src="https://github.com/user-attachments/assets/71f8ce60-b0ca-4c47-9bd3-96bc4dc24115" />


The script searches the constraints matrix for rows and columns that define clock timing parameters. These include early and late rise/fall delays and slews. It records the positions of these entries for use in subsequent SDC generation steps.

## 6. Clock Delay and Slew Setup Continuation

<img width="1229" height="364" alt="image" src="https://github.com/user-attachments/assets/d8cd7511-0a08-4d6b-974e-fe69c3cdf94b" />


Here, the script finalizes the mapping of clock parameters. It ensures that every combination of delay and slew (early, late, rise, fall) is captured and stored in variables. This mapping is crucial because these references will later be used to generate precise timing constraints in the output file.

## 7. Generating Clock Constraints in SDC

<img width="1396" height="469" alt="image" src="https://github.com/user-attachments/assets/22739714-44dd-4119-8518-244c33a9fd03" />


This section writes the first part of the SDC (Synopsys Design Constraints) file. It loops through each clock entry in the constraints matrix and creates commands such as create_clock, set_clock_transition, and set_clock_latency. This automates the creation of detailed clock definitions required for synthesis and STA tools.

## 8. Hierarchy Check Script Creation

<img width="930" height="393" alt="image" src="https://github.com/user-attachments/assets/e65b6bc5-7803-4cd3-9e1f-9cc1330085b7" />




The next phase ensures that all RTL modules are correctly connected. The script generates a temporary Yosys script that loads the library, reads all Verilog files, and performs a hierarchy -check operation. This check ensures that no module instantiations are missing or mismatched before synthesis.

## 9. Hierarchy Check Validation
<img width="921" height="553" alt="image" src="https://github.com/user-attachments/assets/fbdce749-cde9-43e2-837c-43df5f75ea03" />





After running the Yosys hierarchy script, the log file is analyzed. The script searches for any error patterns indicating undefined modules or missing references. If issues are found, they are printed to the terminal. Otherwise, it confirms that the hierarchy check passed successfully.

## 10. Main Synthesis Script Creation

<img width="893" height="502" alt="image" src="https://github.com/user-attachments/assets/f507f479-9d2d-43bf-a3b2-33d0ae0add82" />


This section builds the main Yosys synthesis script. It reads all Verilog files, sets the top module, performs synthesis, technology mapping, and writes out the gate-level netlist. After generating the script, it runs Yosys and logs all messages, producing a synthesized version of the design.

## 11. Static Timing Analysis Setup


<img width="938" height="321" alt="image" src="https://github.com/user-attachments/assets/e9faf8b5-0b82-4911-b653-317fe6e61183" />

Once synthesis is done, the script moves to the timing analysis phase. It loads necessary Tcl procedures for OpenTimer, sets up multi-threading, and initializes the OpenTimer configuration file. Then it reads the early and late libraries, the synthesized Verilog netlist, and the SDC constraints.

## 12. Dummy SPEF File Generation

<img width="901" height="386" alt="image" src="https://github.com/user-attachments/assets/07f6bc4f-e707-414c-9b14-7d2aa6d7ff10" />



Because physical layout is not yet available, this section creates a dummy SPEF file with zero-wire parasitics. The file includes standard IEEE headers defining units for time, resistance, capacitance, and inductance. This allows OpenTimer to run pre-layout analysis without physical interconnect data.

## 13. Running OpenTimer

<img width="943" height="455" alt="image" src="https://github.com/user-attachments/assets/70f3812c-aeaf-41fe-a262-4ceeddaa160e" />


Here the script appends timing commands such as init_timer, report_timer, and report_wns to the configuration file. It executes OpenTimer using the generated configuration, measures total runtime, and reports completion. The output results file contains all the timing data for the analyzed design.

## 14. Output Violation Extraction

<img width="940" height="390" alt="image" src="https://github.com/user-attachments/assets/3e9bc904-1271-4cc1-a7d9-a41681ad6cbb" />


This part parses the OpenTimer results to find output-related (RAT) timing violations. It scans for the keyword “RAT,” extracts the worst slack value, and counts how many violations exist. These metrics indicate whether the design meets its output timing requirements.

## 15. Setup Timing Violation Extraction
<img width="935" height="374" alt="image" src="https://github.com/user-attachments/assets/d87c09ed-8361-47a5-8427-c17387f96a0e" />

The script continues by searching for “Setup” violations in the timing results. It records the worst setup slack and the total number of violations found. This data shows if there are any paths that fail to meet the setup time requirements.

## 16. Hold Timing Violation Extraction
<img width="934" height="466" alt="image" src="https://github.com/user-attachments/assets/1b8643b9-e421-4c15-aa0c-fd11417acd0d" />



This section performs a similar analysis for hold violations. It searches for “Hold” entries in the results, identifies the worst hold slack, and counts how many paths violate hold constraints. These values are crucial for verifying that the design maintains signal stability after the clock edge.

## 17. Instance Count Extraction

<img width="951" height="356" alt="image" src="https://github.com/user-attachments/assets/b6f67f25-0b15-460a-9cc8-eb8ad91dac61" />



The script then finds the number of gates or instances synthesized in the design. It looks for a line containing the pattern “Num of gates” and extracts the count. This provides a measure of design complexity and will appear in the final report summary.

## 18. Preparing Pre-Layout Timing Report
<img width="940" height="258" alt="image" src="https://github.com/user-attachments/assets/e92c50f9-d79b-4942-afa0-84b8ca12fe5e" />


With all timing data collected, the script prepares a formatted summary table. It defines column headers such as runtime, instance count, worst setup and hold slacks, and violation counts. This creates a structured, easy-to-read summary for the user.

## 19. Printing the Results Summary

<img width="955" height="381" alt="image" src="https://github.com/user-attachments/assets/8af7b586-7ccd-4abd-a08b-a36bd159cadc" />


The script populates the report table with actual data from the previous analysis. It prints one formatted line summarizing runtime, instance count, and timing results for the design. This summary acts as a concise overview of synthesis and STA outcomes.

## 20. Closing the Report

<img width="941" height="182" alt="image" src="https://github.com/user-attachments/assets/5dd4783c-2f54-4d03-be15-606dea47e06d" />


The final section prints a closing separator line to complete the report format. It leaves a blank line for readability and signifies that the entire synthesis and timing analysis process has completed successfully.
