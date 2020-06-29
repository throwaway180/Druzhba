 
# `Druzhba`

# Overview

Druzhba is a hardware switch simulator for testing compilers targeting programmable switches.
High level programs compiled to Druzhba's instruction set can be simulated, testing
the abilities for these compilers to map programs to switch hardware primitives. Druzhba enables 
simulation through 2 components: ```dgen``` and ```dsim```. ```dgen``` generates a Rust file corresponding to the
pipeline to be simulated and ```dsim``` uses this file to model the pipeline's feedforward behavior.

# Installation

1. Install ```rust```. For Unix-like OS run ```curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh```. If further help is needed, please see https://www.rust-lang.org/tools/install.

2. Enable ```rust nightly```:

    ```rustup default nightly```

3. Clone this repo. 

4. Ensure ```build_dgen.sh``` is executable:
    
    ```chmod +x build_dgen.sh```

5. You're good to go! 

# Usage

To easily execute Druzhba use druzhba_run.py (this will execute both ```dgen``` and ```dsim```):

    python3 druzhba_run.py  -h
    usage: druzhba_run.py [-h] [-c [CONSTANTS]] [-g GEN] [-t [TICKS]] [-O [OPTI]]
                          [-n]
                          program_name stateful_alu stateless_alu pipeline_depth
                          pipeline_width num_stateful_alus hole_configs

    Druzhba execution

    positional arguments:
      program_name          Program spec name
      stateful_alu          Path to stateful ALU file
      stateless_alu         Path to stateless ALU file
      pipeline_depth        Depth of pipeline
      pipeline_width        Width of pipeline
      num_stateful_alus     Number of stateful ALUs per stage (number of state
                            variables in spec)
      hole_configs          File path for the file containing the machine code
                            assignments

    optional arguments:
      -h, --help            show this help message and exit
      -c [CONSTANTS], --constants [CONSTANTS]
                            Constant vector for Chipmunk
      -g GEN, --gen GEN     Number of PHV containers to randomly initialize by
                            traffic generator. Rest of PHV containers initialized
                            with 0
      -t [TICKS], --ticks [TICKS]
                            Number of ticks
      -O [OPTI], --opti [OPTI]
                            Number corresponding to optimization level (0 for
                            unoptimized, 1 for sparse conditional constant
                            propagation, 2 for inlining)
      -n                    Set if attempting to simulate the previous
                            configuration to prevent recompiling dsim


Example:

    python3 druzhba_run.py simple example_alus/stateful_alus/raw.alu example_alus/stateless_alus/stateless_alu.alu 2 2 1  hole_configurations/simple_raw_stateless_alu_2_2_hole_cfgs.txt -c "0,1,2,3" -g 1 

More examples can be found at the bottom. Note: the -n recompile flag should be used if you have already compiled ```dsim``` previously and would like to rerun it without recompiling.
This is especially useful if machine code pairs are to be swapped for unoptimized Druzhba executions 
or if the program is to be run with a different number of ticks. 
If the recompile flag is turned
on for a new pipeline configuration (e.g. different stateful ALU or pipeline dimensions) this will not work; ```dgen``` will need to be run again
and the resulting file needs to be compiled with ```dsim```.

To execute ```dgen``` alone (note that this is within the ```dgen``` directory):

    dgen -h

    dgen 1.0
    Code generator for Druzhba

    USAGE:
            dgen [FLAGS] [OPTIONS] <spec_name> <stateful_alu> <stateless_alu> <pipeline_depth> <pipeline_width> <num_stateful_alus>

    FLAGS:
        -h, --help       Prints help information
        -V, --version    Prints version information

    OPTIONS:
        -c, --constants <constant_vec>    Constant vector for Chipmunk
        -i, --input <input_file>          Druzhba machine code (only required for optimzed code generation)
        -O, --opti <opti>                 Optimization level: 0, 1, or 2
        -o, --output <output_file>        Output generated file

    ARGS:
        spec_name            Name of input program
        stateful_alu         Path to stateful ALU file
        stateless_alu        Path to stateless ALU file
        pipeline_depth       Depth of pipeline to simulate
        pipeline_width       Width of pipeline to simulate
        num_stateful_alus    Number of stateful ALUs per stage

Example:

    cd dgen
    cargo run simple ../example_alus/stateful_alus/raw.alu ../example_alus/stateless_alus/stateless_alu.alu 2 2 1 -c 0,1,2,3 -i ../hole_configurations/simple_raw_stateless_alu_2_2_hole_cfgs.txt -O2 
    cd ..


The output will be a Rust file containing the pipeline description to be simulated. To compile with ```dsim```, rename the file to 
prog_to_run.rs and move it into the src directory.

To execute ```dsim```:

    dsim -h

    dsim 1.0
    Hardware switch simulator for compiler testing

    USAGE:
            dsim [FLAGS] [OPTIONS]

    FLAGS:
        -h, --help       Prints help information
        -V, --version    Prints version information

    OPTIONS:
        -i, --input <input_file>    Path to file containing machine code pairs.
        -g, --gen <num_phv_cons>    Number of PHV containers to be initialized by traffic generator
        -t, --ticks <ticks>         Number of ticks to execute for. A PHV enters the pipeline at every tick.


Example:

    cargo run -- -g 1 -t 20 2>/dev/null


Machine code file can be given to ```dsim``` instead of ```dgen``` if more flexibility is desired by not having to rerun ```dgen``` and recompile ```dsim``` but optimizations would not be able to be used.

The above example directs STDERR to null device due to compiler warnings from test generated ```dgen``` files. Many of these warnings 
are either unavoidable or helpful for readibilty. While some can be fixed we leave that to future work due to the tediousness 
of the task. 

# Test

Tests will ensure the druhzba pipeline is outputting
the correct packets relative to the input packets
given to the pipeline. Run ```dgen``` first to get a 
prog_to_run.rs file in src first. Otherwise the simulator
won't compile. 

To run tests:

    ./build_dgen.sh && cargo test

To run benchmarks:

    ./build_dgen.sh && cargo bench

Similarly, the ```dgen``` tests ensure that the alu grammar
is being parsed correctly and that the ast is being
generated properly. 

To run these tests:

    cd dgen
    cargo test


# More Examples


Simulating simple:

    python3 druzhba_run.py simple example_alus/stateful_alus/raw.alu example_alus/stateless_alus/stateless_alu.alu 2 2 1  hole_configurations/simple_raw_stateless_alu_2_2_hole_cfgs.txt -c "0,1,2,3" -g 1 -t 30 -s "{{0}}"

Simulating marple_tcp_nmo:

    python3 druzhba_run.py marple_tcp_nmo_equivalent_1_canonicalizer_equivalent_0 example_alus/stateful_alus/pred_raw.alu example_alus/stateless_alus/stateless_alu.alu 3 2 2  hole_configurations/marple_tcp_nmo_equivalent_1_canonicalizer_equivalent_0_pred_raw_stateless_alu_3_2_hole_cfgs.txt -g 1 -t 20 -O2

Simulating blue_increase:

    python3 druzhba_run.py blue_increase_equivalent_2_canonicalizer_equivalent_0 example_alus/stateful_alus/pred_raw.alu example_alus/stateless_alus/stateless_alu_arith.alu 4 2 2  hole_configurations/blue_increase_equivalent_2_canonicalizer_equivalent_0_pred_raw_stateless_alu_arith_4_2_hole_cfgs.txt -g 2 -t 10 -O2 -c "11,21,10,12,0,3,1,2,10,2,1"

Simulating snap_heavy_hitter:

    python3 druzhba_run.py snap_heavy_hitter example_alus/stateful_alus/pair.alu example_alus/stateless_alus/stateless_alu.alu 2 3 1 hole_configurations/snap_heavy_hitter_pair_stateless_alu_2_3_hole_cfgs.txt -g 1 -c "0,1,2,3,999,997,1002,1000,4"


