# Hardware Managing System

**Category:** Hardware / Reverse Engineering
**Flag:** `HTB{BCKD000RD_HRDWR3}`

A "Critical System Management System" claims to shut down its managed system when the temperature reaches 100. The vendor was reported for industrial sabotage; suspect a deliberate hardware backdoor. Find an input to the simulation that causes the emergency-off feature to fail.

## Files

- [`WRITEUP.md`](WRITEUP.md) — full walkthrough (markdown)
- [`WRITEUP.docx`](WRITEUP.docx) — same writeup in Word format
- [`challenge_files/notes.txt`](challenge_files/notes.txt) — original challenge notes
- [`challenge_files/CtfTask.v`](challenge_files/CtfTask.v) — Yosys-synthesized Verilog netlist (~4,700 lines)
- [`challenge_files/VCtfTask`](challenge_files/VCtfTask) — Verilator simulation binary (Linux x86-64)

## Quick verification

```
$ printf "HTB{BCKD000RD_HRDWR3}" | ./challenge_files/VCtfTask
...
[8] EMERGENCY SHUTDOWN MISSED!
TEST FAILED, EMERGENCY SHUTDOWN WAS NOT ACTIVED!
INPUT 'HTB{BCKD000RD_HRDWR3}' NEEDS FURTHER ANALYSIS
```
