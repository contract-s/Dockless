# Dockless

A prototype desk that tracks your phone with a camera and moves a wireless charger under it on a two-axis stepper gantry.

> **Status: prototype.** Phone tracking and the motor firmware each work on their own. The full camera-to-gantry loop is still being tuned.

## How it works

```
camera --> phone tracker (YOLOv8 + OpenCV) --> homography: pixels -> desk cm
       --> motion.py: cm -> motor steps --> USB serial --> ESP32 (charger_mover.ino) --> 2x NEMA17
```

- **Detection:** `phone_tracker_screen_off.py` is a hybrid tracker. It combines YOLOv8n ("cell phone") with an OpenCV dark-rectangle detector from `phone_tracker_cv.py`, so a face-down phone or one with the screen off is still found. `yolo_phone_tracker.py` is the YOLO-only version.
- **Calibration:** you click the 4 desk corners once. `cv2.findHomography` maps camera pixels to a 24 × 18 in (60.96 × 45.72 cm) desk surface, and the result is saved to `calibration.json`.
- **Motion:** `motion.py` converts desk cm into absolute step counts. The rate is 80 steps/mm: 200-step motors × 1/16 microstepping ÷ GT2 20-tooth pulleys (40 mm/rev). Travel is 434 × 242 mm. The result is sent to the ESP32.
- **Gantry:** crossed rods. The X motor (top track) moves the vertical rod, the Y motor (left track) moves the horizontal rod, and the charger sits where they cross.

## Hardware

- ESP32 Dev Module (ELEGOO ESP-WROOM-32)
- 2 × NEMA17 steppers with A4988 drivers at 1/16 microstepping
- GT2 belts and 20T pulleys
- Webcam mounted above the desk
- Wireless (MagSafe) charger

| Signal | ESP32 pin |
|---|---|
| X STEP / DIR | 26 / 27 |
| Y STEP / DIR | 25 / 33 |
| Driver ENABLE (shared, active LOW) | 13 |
| Limit switches (optional, off by default) | X 4, Y 16 |

## Firmware

Open `charger_mover/charger_mover.ino` in the **Arduino IDE**. Install **AccelStepper** (by Mike McCauley) from the Library Manager, select **ESP32 Dev Module** and upload.

Serial protocol (115200 baud, one command per line, replies `ok` or `error: ...`):

| Command | Action |
|---|---|
| `G <x_steps> <y_steps>` | Move to absolute step position |
| `Z` | Set the current position as 0,0 |
| `H` | Home with limit switches (if enabled) |
| `S` | Stop (decelerate) |
| `?` | Status: `pos <x> <y> moving <0/1>` |
| `E <0/1>` | Motors off / on |

## Run

```bash
python -m venv .venv && source .venv/bin/activate
pip install ultralytics opencv-python numpy pyserial

# test the gantry on its own
python motion.py --list-ports
python motion.py --zero              # charger is at home: call it 0,0
python motion.py --jog x 50          # move X by 50 mm
python motion.py --goto 30 20        # go to (30 cm, 20 cm) on the desk
python motion.py --square            # visit the 4 corners of the travel
python motion.py --dry-run --goto 30 20   # print steps without moving

# track the phone
python yolo_phone_tracker.py --calibrate  # click the 4 desk corners once
python phone_tracker_screen_off.py        # hybrid tracker
```

## Legacy files

These are from an earlier GRBL / FluidNC approach and are not used by the current code: `grbl_controller.py`, `charging_desk_fluidnc.yaml`, `hook_motion.py`, `patch_ptc.py`, `yolo_phone_tracker.before_motion.py`.

## Tech stack

Python, OpenCV, YOLOv8 (Ultralytics), C++ / Arduino IDE, ESP32, AccelStepper, NEMA17 + A4988

Built at MHacks 2026.
