# Troubleshooting Drives

## Encountered Symptoms and Solutions

### Symptom: No activity from drive upon power up. Excessive current draw.

Cause: shorted capacitors on the 12 volt rail.

In both a ST-251 w/ 20938 (sn 30507423) and a ST-251 w/ 20949 (sn 01583284, this symptom was concurrent with one of the problem capacitors smoking. This did not result in board damage in either case.

The problem capacitors were desoldered and power was applied. In both cases the drives responded with activity. The drive w/ 20938 spun the spindle motor and began the startup head movement. The drive with the 20949 began to move the spindle but shutdown.

On the drive with the 20949, the failed capacitor was the 4.7uF 20v (marking "475 20K") near the 44pin IC 10206-501.

Solution: The problem capacitors were replaced.

### Symptom: No activity from drive upon power up. Normal current draw.

Cause: MCU is locked or it's oscillator has not started.

This was observed on an ST-251 w/ 20938 (sn 30507423).

Probing the pins of the 4MHz crystal will show that it is not operating. This may trigger it to start, however, resulting in the expected start-up behavior.

Solution: Reflow solder to the pins of the 4MHz crystal and relevant pins of the MCU R6518AJ. Check traces in the region for breaks.

### Symptom: Spindle motor starts briefly then spins down.

Cause: unknown

Hypothesis: Capacitors relating to the spindle motor drive have failed.

Repeatedly power cycling the drive while the spindle was still spinning would allow enough time for the normal start-up head movement to begin. Removing power at this point would result in the proper head-parking behavior. This indicated that those subsystems appeared to be operational.

Inspecting the 22uF (marking "226 20K"), 10uF (marking "10u 25V"), and 1uF (marking "1u0" capacitors near the spindle motor driver IC HA13406W revealed that several of the 22uF capacitors near the spindle motor driver had failed.

The power supply filtering capacitors near the molex connector were also inspected but were found to be within spec.

Solution: not found. on-going. Capacitor replacement in-progress.



