# APU_100KG

Controls the APU through the Moons M54S drive (Modbus RTU), 100:1 gearbox, with a live dial.

## Screen layout
One screen, no scrolling: the header (**STOP + BRAKE**, live X1 / X2 / brake / drive state) and the dial stay on screen
all the time; the right side switches between pages with the tabs: **Homing**, **Move & jog**, **Compass heading**,
**Auto sequence**, **Status**, **Connection**. `Esc` (or `Space` when no field has focus) also stops the APU.

## Files
- `apu_config.py`      all settings (COM port, speeds, sensors, limits)
- `apu_controller.py`  encoder tracking, homing, go-to-angle, jog, sensor/limit stop, brake
- `app.py`             web server        -> `python app.py`, open http://localhost:5000
- `templates/index.html` the dial and controls
- keep your `run_motor.py` next to these for diagnostics (`python run_motor.py inputs` shows X1/X2)

## Install / run
    pip install -r requirements.txt
    python app.py                # or: python app.py --port COM5 --baudrate 115200

## How it works
- Angle (home at one END): the end stoppers only allow about +-175 deg, so home is no longer in the centre.
  **X1 = home = +175 deg**, 0 deg = centre (no sensor), **X2 = limit = -175 deg**. + = clockwise, - = counter-clockwise.
  Usable range after homing: **-175 ... +175**.
  angle = (encoder - zero) / counts-per-rev / 100 * 360. One APU turn = 100 motor turns.
- Find home - only two cases:
  1. **X1 already ON** -> nothing moves. The screen shows "Homing done" and the current position is taken as +175 deg.
  2. **X1 not ON** -> the APU turns toward **+** at the homing speed (motor rpm, set on screen) until X1 switches ON.
     The encoder value read with that edge is +175 deg; the APU settles exactly on it and shows "Homing done".
  If X2 switches ON while it is searching (wrong rotation direction or a sensor fault) the APU stops with a
  "HOMING FAILED" message. Starting on X2 is fine: it simply turns + away from it.
  Slower homing speed = more accurate +175 deg. X1 has a width, so "already ON" can mean a position a little
  inside the sensor window; set `HOME_REFINE_IF_ON_SENSOR = True` if you prefer the APU to step off toward 0 and
  come back to catch the true X1 edge.
- **Manual homing** (Homing page, "Manual - set home here"): hold the **Hold to move -/+** buttons (the APU moves only while the
  button is held, at the speed set on the page) until the APU stands where home should be, then press **Set home here**.
  That encoder value becomes the home angle (`MANUAL_HOME_ANGLE_DEG`, default +175) and the dial is valid at once, exactly as
  after sensor homing. No sensor is needed. Do it at the real + end (X1 end) so the +-175 limits match the stoppers.
  Refused while a motion is running (press STOP first). The compass north reference is measured again afterwards.
- Rotations: travel = 350 deg = **97.22 motor rotations** (48.61 from the centre to either end). The screen shows
  motor rotations from 0 (centre, -48.61 ... +48.61) and from home (X1 end, 0 ... 97.22).
- Go to angle: target is converted to an absolute encoder count, so every move is corrected from the
  live encoder (no accumulated error). Moves are refused until homed and beyond -175 / +175.
- X2 (-175 deg end): turning ON stops the motor at once. If you sit on X2, only moves back toward + are allowed.
  X1 (the +175 end) works the same way: when the APU turns toward + and X1 switches ON, the motor stops at once
  (`HOME_SENSOR_STOPS_MOTION`), and from there only moves back toward - are allowed. A target of +175 (or -175) counts as
  reached when its end sensor stops the motor there (`LIMIT_ARRIVAL_TOL_DEG`). The software limits +-175 stay as a second guard.
- Auto sequence: a list of angles (each with an optional wait in seconds), run once or in a loop until STOP.
  Every step is an absolute move from the live encoder, with the same X2 limit stop, soft limit and brake
  handling as normal moves. A step at -175 counts as reached when X2 stops the motor there
  (`LIMIT_ARRIVAL_TOL_DEG`). The brake stays released between steps; long waits (`SEQ_BRAKE_DWELL_S`) engage it.
  The list is remembered by the browser.
- Fast response: the servo stays enabled, the brake (Y2) is the only thing switched between moves, accel/decel/
  speed/distance reach the drive in one write, and status comes from the background poll. Remaining delays
  are `BRAKE_RELEASE_DELAY` (0.05 s) and the Modbus speed - use 115200 baud for the biggest gain.
- X1 drift check: when X1 is crossed toward + (the same direction as when home was latched), its ON edge must be
  ~+175 deg; a larger deviation shows a warning ("Last X1 crossing" turns red) - home again if it appears.
- Dial: 0 deg at the top. The needle shows the live angle; the X1 (+175) and X2 (-175) markers sit on either side of
  the red stopper zone at the bottom and light up when the sensor is ON. The dial is greyed out until homed.
  "Last X1 crossing" shows the angle at which X1 switched during later moves - if it drifts away from +175,
  something slipped.
- Brake (digital output Y2): Y2 ON = brake engaged, Y2 OFF = brake released. Before every movement the servo is
  enabled and Y2 is switched OFF; when the movement ends (or STOP / limit sensor) Y2 is switched ON first, and
  then the servo is disabled. If the drive refuses the Y2 command the APU will not move (release) or the servo
  stays enabled (engage). Small Engage / Release buttons next to the Brake indicator work only while idle.
  In Luna, Y2 must be a plain programmable output. `python brake_test.py` tests Y2 on its own.

## Compass (LEC315) - true-north headings
Two separate ways to move, so each can be checked on its own:
- **Go to angle - APU based**: only the APU angle (-175 ... +175), the encoder and the sensors. Unchanged.
- **Go to heading - compass based**: a compass bearing (0 = north, clockwise). The APU turns until it points there.

How it works
- The APU can stand anywhere, pointing anywhere. After homing, the controller reads the compass (several readings,
  while the APU stands still) and works out which bearing the APU's own 0 deg points to: the **north offset**.
  Then  `bearing the APU points to = north offset + APU angle`  (shown on the dial and in the compass card).
- Target heading -> APU angle = wrap(heading - north offset). The rotation direction is chosen automatically: a bearing
  that is "beyond +175" is reached by turning the other way (-160 instead of +200). Because the travel is 350 deg,
  only a 10 deg **blind zone** behind the stoppers (centred 180 deg from the north offset) cannot be reached; then a
  pop-up explains it and shows the nearest reachable headings. An APU angle beyond +-175 in the APU-based section also
  gives a pop-up (with the equivalent angle, if there is one).
- After a heading move the compass is read again (rotating mount). If it differs by more than `COMPASS_TOL_DEG` the
  APU is trimmed (at most `COMPASS_MAX_TRIMS` times; a correction larger than `COMPASS_MAX_TRIM_DEG` is NOT made,
  because then the compass and the encoder disagree and something should be checked).
- The north offset is measured automatically after every homing (`COMPASS_AUTO_CAL_AFTER_HOME`), and the
  "Calibrate north" button measures it again (e.g. after the base was turned). "Reference check" compares the compass
  with the encoder at any time; it turns red if they disagree.
- Readings that scatter more than `COMPASS_MAX_SPREAD_DEG` (magnetic interference from motor, brake, steel) are refused.

Files: `lec315.py` (compass driver), `apu_compass.py` (connection + north maths), `compass_cli.py` / `compass_app.py`
(stand-alone compass tests, no APU needed).

Step by step (do them in this order)
1. Compass alone: `python compass_cli.py` -> 1) auto-detect, 3) live view. Or `python compass_app.py` and open
   http://localhost:5001. Turn the compass by hand: the heading must change smoothly, N = 0, clockwise.
2. Main page: `python app.py --compass-port COM10` (or Connect in "Compass connection"). Use the compass's own adapter.
3. Home the APU. The north reference is set automatically (compass card: "North reference set").
4. Jog + a little: the compass bearing must INCREASE. If it decreases set `COMPASS_SENSE = -1`.
   If the compass is on the fixed base (does not turn with the APU) set `COMPASS_MOUNT = "fixed"`.
5. Go to heading 0 / 90 / 180 / 270 and compare with a real reference (phone compass, map, known landmark).
   A constant error = `COMPASS_MOUNT_OFFSET_DEG` (compass body rotated) or `COMPASS_DECLINATION_DEG`
   (magnetic -> true north; leave 0 for magnetic north).
6. Try a heading inside the blind zone: the pop-up must appear and nothing moves.
Note: a magnetic compass is only as good as its surroundings - mount it away from the motor, brake and steel.

## First start-up checklist
1. `python run_motor.py inputs`: confirm X1 / X2 read as you expect, then check `SENSOR_ACTIVE_BIT_VALUE`.
2. Jog + for a moment: the dial must turn clockwise and the angle must increase. If not, set `INVERT_DIRECTION = True`.
3. Run "Find home" with a hand near STOP: from any position it must turn + and stop at X1 (+175). If X2 comes
   ON first the APU stops with "HOMING FAILED" - then the direction is wrong (`INVERT_DIRECTION`).
4. Go to +90, 0 and -90, then -175 and +175; check the real APU against the dial.

## Notes
- Software stops depend on Modbus speed. At 9600 baud a sensor is seen within ~50 ms; 115200 is ~10x better
  (set the drive BR to match). For a hard backup, also wire X2 as the drive's own limit input.
- If X2 switches before -175 deg, a -175 target still counts as reached (X2 stops the motor). To change the
  travel, edit `ANGLE_MAX_DEG`, `ANGLE_MIN_DEG` and `HOME_ANGLE_DEG` in `apu_config.py` (the dial and presets follow).
- `apu_state.json` from the old (centre-home) version is not valid for the new convention and is ignored.
- `RESTORE_ZERO_ON_START` is off: homing is required after every start. Only enable it if you have verified
  the drive keeps its absolute multi-turn position across power cycles.
- Drive alarms 24/25/26/28 (absolute encoder) clear the homed flag automatically.#   A P U _ C O N T R O L _ D Y N A  
 