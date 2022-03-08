# Constraining-Multiplexed-Data-Ports

# *Question: I have data input and output ports that I would like to constrain against multiple clock domains. What's the best way to do this?*

*Answer:*

The answer to this depends on whether the clocks themselves are multiplexed or exist simultaneously. Let's take a look at both scenarios. Note that although input port scenarios are shown below, the solutions presented apply equally well to data output ports.

# *Multiplexed Data Input Ports, Multiplexed Clocks*
In this scenario, we have both clock and data input ports which are multiplexed:

<img src= "https://github.com/abdelazeem201/Constraining-Multiplexed-Data-Ports/blob/main/PICs/Figure1.png">

Since only one clock can come through the input port at a time, we can simply define a physically exclusive relationship for the clocks:
```
  create_clock -name CLKA -period 10 [get_ports CLK]
  create_clock -name CLKB -period 20 [get_ports CLK] -add
  create_clock -name CLKC -period 40 [get_ports CLK] -add
  set_clock_groups -physically_exclusive -group CLKA -group CLKB -group CLKC
```
We can then simply define the input delays at the multiplexed data port:
```  
  set_input_delay -clock CLKA 1 DATA
  set_input_delay -clock CLKB 2 DATA -add
  set_input_delay -clock CLKC 3 DATA -add
```
The physically exclusive relationship disables any cross-clock paths between the input port and capturing flop, such that the flop will only capture data edges against the same clock which launched that edge at the input port.

# *Multiplexed Data Input Ports, Multiple Simultaneous Clocks*

In this scenario, multiple on-chip clocks exist simultaneously and we wish to constrain the paths such that the capture clock is the same as the launch clock:

<img src= "https://github.com/abdelazeem201/Constraining-Multiplexed-Data-Ports/blob/main/PICs/Figure2.png">

If we can guarantee that there are no other cross-clock paths which must be timed, we can simply create a clock relationship which suppresses paths between the clock domains, and apply the needed input delays.

For a set of asynchronous clocks, we would use an asynchronous clock relationship:

```
  create_clock -period 10 [get_ports CLKA]
  create_clock -period 16 [get_ports CLKB]
  create_clock -period 27 [get_ports CLKC]
  set_clock_groups -asynchronous -group CLKA -group CLKB -group CLKC

  set_input_delay -clock CLKA 1 DATA
  set_input_delay -clock CLKB 2 DATA -add
  set_input_delay -clock CLKC 3 DATA -add
```

For a set of synchronously-timed but otherwise independent clocks, we would use a logically exclusive relationship:
```
  create_clock -period 10 [get_ports CLKA]
  create_clock -period 20 [get_ports CLKB]
  create_clock -period 40 [get_ports CLKC]
  set_clock_groups -logically_exclusive -group CLKA -group CLKB -group CLKC

  set_input_delay -clock CLKA 1 DATA
  set_input_delay -clock CLKB 2 DATA -add
  set_input_delay -clock CLKC 3 DATA -add
```

However, the situation gets more complicated if the clocks are synchronous and there are valid synchronous cross-clock communication paths:

<img src= "https://github.com/abdelazeem201/Constraining-Multiplexed-Data-Ports/blob/main/PICs/Figure3.png">

In the example above, there is communication between CLKB and CLKC. If we were to apply a logically exclusive relationship between these two clocks, we would disable this valid timing path between FF1 and FF2!

In such a case, there are two potential methods to eliminate cross-clock paths only at the input ports.

*Using Virtual Clocks*

If we constrain the inputs against virtual clocks instead of real clocks, we can then suppress cross-clock paths between the real clocks and their virtual counterparts. For example:
```
  create_clock -period 10 [get_ports CLKA]
  create_clock -period 10 -name V_CLKA
  create_clock -period 20 [get_ports CLKB]
  create_clock -period 20 -name V_CLKB
  create_clock -period 40 [get_ports CLKC]
  create_clock -period 40 -name V_CLKC

  set_input_delay -clock V_CLKA 1 DATA
  set_input_delay -clock V_CLKB 2 DATA -add
  set_input_delay -clock V_CLKC 3 DATA -add

  set_false_path -from [get_clocks V_CLKA] -to [get_clocks {CLKB CLKC}]
  set_false_path -from [get_clocks V_CLKB] -to [get_clocks {CLKA CLKC}]
  set_false_path -from [get_clocks V_CLKC] -to [get_clocks {CLKA CLKB}]
```
*Using False Paths Through Ports*

Another option is to use a little-known form of false paths where we specify the startpoint -through, and clock objects for -from/-to:
 ```
  create_clock -period 10 [get_ports CLKA]
  create_clock -period 20 [get_ports CLKB]
  create_clock -period 40 [get_ports CLKC]

  set_input_delay -clock CLKA 1 DATA
  set_input_delay -clock CLKB 2 DATA -add
  set_input_delay -clock CLKC 3 DATA -add

  set_false_path \
    -from    [get_clocks CLKA] \
    -through [get_ports DATA] \
    -to      [get_clocks {CLKB CLKC}]
  set_false_path \
    -from    [get_clocks CLKB] \
    -through [get_ports DATA] \
    -to      [get_clocks {CLKA CLKC}]
  set_false_path \
    -from    [get_clocks CLKC] \
    -through [get_ports DATA] \
    -to      [get_clocks {CLKA CLKB}]
```    
In general, this method is preferable because it doesn't require the addition of extra clocks to the analysis.

Both of the techniques above work even when the capturing flops themselves aren't clock-multiplexed. Consider a case where separate flops in each clock domain are meant to capture clock-specific data:

<img src= "https://github.com/abdelazeem201/Constraining-Multiplexed-Data-Ports/blob/main/PICs/Figure4.png">

With either of the techniques above, each flop will only capture data using the edge with the input delay for that clock. Edges from other clocks will not be captured. Just as before, this property is also true in the output direction when data from multiple separate clock domain launching flops is multiplexed together into a single output data port.
