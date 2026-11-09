# Pump Duty/Standby Changeover (Siemens TIA Portal V19, SCL)

A function block (`Pump_DutyStandby`) written in SCL that controls two pumps in a duty/standby arrangement. It balances run hours between the pumps and switches automatically to the healthy pump when the running one fails.

Tested in S7-PLCSIM V19 on a CPU 1511-1 PN (6ES7 511-1AL03-0AB0, firmware V3.1).

## Features

- **Run-hours balancing:** on every new start demand, the pump with fewer accumulated run hours becomes the duty pump.
- **Automatic failover:** if the running pump trips on overload, or does not give run feedback within the timeout (default 5 s), the other pump starts.
- **Latched faults:** a fault stays active until the operator resets it, and a pump with an active overload cannot be reset.
- **Total failure handling:** if both pumps are faulted, both commands are switched off and `All_Failed` is raised.
- **No mid-run swapping:** the duty pump is selected only on the rising edge of `Demand`.

## Interface

| Section | Name | Type | Description |
|---|---|---|---|
| Input | `Demand` | Bool | Start request (e.g. from a tank level controller) |
| Input | `Reset` | Bool | Fault reset (operator) |
| Input | `Fb_Run_1`, `Fb_Run_2` | Bool | Run feedback from each pump |
| Input | `Ovl_1`, `Ovl_2` | Bool | Overload trip (TRUE = fault) |
| Input | `Fb_Timeout` | Time | Allowed time without run feedback (default `T#5s`) |
| Output | `Cmd_1`, `Cmd_2` | Bool | Start command to each pump |
| Output | `Fail_1`, `Fail_2` | Bool | Latched fault per pump |
| Output | `All_Failed` | Bool | Both pumps faulted |
| Output | `Duty` | Int | Current duty pump (1 or 2), default 1 |
| Static | `Hours_Sec_1`, `Hours_Sec_2` | DInt | Accumulated run time in seconds |
| Static | `Tick`, `Fb_Tmr_1`, `Fb_Tmr_2` | TON_TIME | One-second pulse and feedback timers |
| Static | `Demand_Prev` | Bool | Edge detection memory |

## SCL code

```scl
// 1) One-second pulse (used to count run time)
#Tick(IN := NOT #Tick.Q, PT := T#1s);

// 2) Feedback timers: command given but no run feedback
#Fb_Tmr_1(IN := #Cmd_1 AND NOT #Fb_Run_1, PT := #Fb_Timeout);
#Fb_Tmr_2(IN := #Cmd_2 AND NOT #Fb_Run_2, PT := #Fb_Timeout);

// 3) Latch faults
IF #Ovl_1 OR #Fb_Tmr_1.Q THEN #Fail_1 := TRUE; END_IF;
IF #Ovl_2 OR #Fb_Tmr_2.Q THEN #Fail_2 := TRUE; END_IF;

IF #Reset THEN
  IF NOT #Ovl_1 THEN #Fail_1 := FALSE; END_IF;
  IF NOT #Ovl_2 THEN #Fail_2 := FALSE; END_IF;
END_IF;

#All_Failed := #Fail_1 AND #Fail_2;

// 4) Select duty pump on a new demand (rising edge)
IF #Demand AND NOT #Demand_Prev THEN
  IF #Fail_1 THEN
    #Duty := 2;
  ELSIF #Fail_2 THEN
    #Duty := 1;
  ELSIF #Hours_Sec_1 <= #Hours_Sec_2 THEN
    #Duty := 1;
  ELSE
    #Duty := 2;
  END_IF;
END_IF;
#Demand_Prev := #Demand;

// 5) Automatic changeover while running
IF #Duty = 1 AND #Fail_1 AND NOT #Fail_2 THEN #Duty := 2; END_IF;
IF #Duty = 2 AND #Fail_2 AND NOT #Fail_1 THEN #Duty := 1; END_IF;

// 6) Pump commands
#Cmd_1 := #Demand AND (#Duty = 1) AND NOT #Fail_1;
#Cmd_2 := #Demand AND (#Duty = 2) AND NOT #Fail_2;

// 7) Run-hours counter (based on actual run feedback)
IF #Tick.Q THEN
  IF #Fb_Run_1 THEN #Hours_Sec_1 += 1; END_IF;
  IF #Fb_Run_2 THEN #Hours_Sec_2 += 1; END_IF;
END_IF;
```

## Test scenarios (PLCSIM)

| # | Action | Expected result |
|---|---|---|
| 1 | `Demand = 1`, equal run hours | Pump 1 starts (`Cmd_1 = 1`, `Duty = 1`) |
| 2 | `Fb_Run_1 = 1` within 5 s | Pump 1 keeps running, `Hours_Sec_1` counts up |
| 3 | `Fb_Run_1` stays 0 for more than 5 s | `Fail_1 = 1`, pump 2 starts (`Duty = 2`) |
| 4 | Clear the cause, pulse `Reset` | `Fail_1` returns to 0 |
| 5 | `Demand` off then on after pump 1 has more hours | Pump 2 is selected (fewer hours) |
| 6 | `Ovl_1 = 1` while pump 1 runs | Immediate changeover to pump 2 |
| 7 | `Ovl_1 = 1` and `Ovl_2 = 1` | `All_Failed = 1`, both commands off |

All scenarios passed in simulation.

## How to use

1. Create an FB named `Pump_DutyStandby` (language: SCL) and fill the interface as in the table above.
2. Paste the SCL code into the block body and compile.
3. Call the block from OB1 (or another cyclic OB) with an instance DB, and connect the I/O.

## Possible improvements

- Use a retentive memory area for the run hours so they survive a CPU restart.
- Add a manual selection mode (force pump 1 or pump 2).
- Convert run seconds to hours for display on an HMI.
- Log run hours and faults from the PLC and analyse them with Python.

## Tools

Siemens TIA Portal V19, S7-PLCSIM V19, SCL.
