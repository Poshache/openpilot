# 2023 Subaru Forester Angle-Steering Porting Guide

This runbook documents how to make the 2023 Subaru Forester drivable on a future
openpilot branch while preserving that branch's steering-controller and safety
logic. It is based on the port tested successfully on the car on 2026-08-05.

The known-good repositories and commits are:

- openpilot: `Poshache/openpilot`, branch `forester-jul-port`
- openpilot tested tip: `39112975985df32ee889c3f018d489be489099ca`
- opendbc: `Poshache/opendbc`, branch `forester-jul-port`
- opendbc Forester commit: `ed11119a1d516ddfb7909d3ae38ca743fafa1767`
- opendbc base used by that commit: `8acbeccdab45a47aa7af170d2d17b61f0eb5e4f0`
- historical working Forester opendbc commit used for research:
  `f97babe5a7b69c41dc06b078e267c969ea2bdf53`

The tested installer URL is:

```text
installer.comma.ai/Poshache/forester-jul-port
```

## Safety and scope

This is safety-critical vehicle-control software. Never replace a new base with
the old Forester branch and never blindly cherry-pick the historical Forester
opendbc commit. Re-evaluate the current code every time because bus routing,
cruise engagement, steering limits, and safety APIs can change.

The Forester is a Gen1 `LKAS_ANGLE` Subaru. In the tested port:

- angle steering commands transmit on the main bus;
- wheel speed and brake status are read on the main bus;
- cruise engagement is read from `ES_Brake` on the camera bus at 50 Hz;
- `ES_Brake->Cruise_Activated` is bit 7 of byte 4 in panda safety;
- Gen2 angle cars continue using `ES_Status` on the alternate bus at 20 Hz;
- Gen2 angle commands continue transmitting on the alternate bus.

Do not weaken or replace the new base's steering safety limits. The tested July
base intentionally preserved all of the following:

- `VehicleModel(get_safety_CP())`;
- `AngleSteeringLimitsVM`;
- `apply_steer_angle_limits_vm`;
- the low-speed angle deadzone filter;
- `steer_angle_cmd_checks_vm`;
- Crosstrek 2025 Gen2 angle support;
- lateral-acceleration and lateral-jerk safety tests.

## Repository model

There are two Git repositories and therefore two commits to create:

1. `opendbc_repo` contains the Subaru car and panda safety changes.
2. The top-level openpilot repository contains `launch_env.sh`, `.gitmodules`,
   and the Git submodule pointer to the new opendbc commit.

Commit and push opendbc first. Only then commit the resulting submodule pointer
in openpilot. A top-level openpilot commit that points to an unpublished opendbc
commit cannot be installed on another device.

## 1. Choose and prepare the new base

Start from the exact future branch whose controller behavior you want. This may
be from comma.ai or Jacob Waller. The examples below use placeholders; replace
them with the actual remote and branch.

```bash
cd /path/to/openpilot
git fetch --all --prune
git switch --create forester-NEW-BASE SOURCE_REMOTE/SOURCE_BRANCH
git status --short --branch
git submodule update --init --recursive
git -C opendbc_repo rev-parse HEAD
git -C opendbc_repo status --short --branch
```

Record the top-level base commit and opendbc commit before changing anything:

```bash
git rev-parse HEAD
git -C opendbc_repo rev-parse HEAD
```

Do not update opendbc to `latest`. Use the exact commit selected by the new
openpilot branch as the new port's base.

If the working tree is not clean, stop and identify every pre-existing change.
Do not overwrite unrelated work.

## 2. Inspect before editing

Confirm that the current base still defines the Forester and angle steering:

```bash
rg -n "SUBARU_FORESTER_2022|SUBARU_CROSSTREK_2025|LKAS_ANGLE" \
  opendbc_repo/opendbc/car/subaru

rg -n "apply_steer_angle_limits_vm|AngleSteeringLimitsVM|steer_angle_cmd_checks_vm" \
  opendbc_repo/opendbc/car/subaru \
  opendbc_repo/opendbc/safety/modes/subaru.h \
  opendbc_repo/opendbc/safety/tests/test_subaru.py
```

Inspect the historical solution, but treat it only as evidence:

```bash
git -C opendbc_repo show --stat \
  f97babe5a7b69c41dc06b078e267c969ea2bdf53

git -C opendbc_repo show \
  f97babe5a7b69c41dc06b078e267c969ea2bdf53 -- \
  opendbc/car/subaru/interface.py \
  opendbc/safety/modes/subaru.h \
  opendbc/safety/tests/test_subaru.py
```

Compare the known-good port with its July base when useful:

```bash
git -C opendbc_repo diff \
  8acbeccdab45a47aa7af170d2d17b61f0eb5e4f0..ed11119a1d516ddfb7909d3ae38ca743fafa1767 -- \
  opendbc/car/subaru/carstate.py \
  opendbc/car/subaru/interface.py \
  opendbc/safety/modes/subaru.h \
  opendbc/safety/tests/test_subaru.py
```

Do not apply that diff automatically to a newer base. Read the current files
and reproduce the behavior using the new APIs and structure.

## 3. Top-level forced fingerprint

In `launch_env.sh`, force the Forester fingerprint:

```diff
-export FINGERPRINT="SUBARU_CROSSTREK_2025"
+export FINGERPRINT="SUBARU_FORESTER_2022"
```

If the new base no longer forces a fingerprint, decide whether forcing is still
needed for the intended test build. Do not add it automatically to a normal
multi-car release branch. For this dedicated Forester test branch, forcing the
fingerprint ensures the expected platform is selected.

Do not copy unrelated settings from the historical Forester branch. In
particular, the old branch also changed `AGNOS_VERSION`; that was not required
for Forester support and was intentionally excluded from the tested port.

## 4. Enable the Forester in the Subaru interface

Open:

```text
opendbc_repo/opendbc/car/subaru/interface.py
```

The new base must:

- retain `LKAS_ANGLE` in the Forester platform definition;
- set the panda safety parameter for `SubaruSafetyFlags.LKAS_ANGLE`;
- keep hybrids and preglobal cars restricted as required by the new base;
- allow the validated Forester to run outside dashcam-only mode;
- preserve any already-validated Crosstrek 2025 behavior.

On the tested July base, the exact change was:

```diff
-    if ret.flags & SubaruFlags.LKAS_ANGLE and candidate != CAR.SUBARU_CROSSTREK_2025:
+    if ret.flags & SubaruFlags.LKAS_ANGLE and candidate not in (CAR.SUBARU_FORESTER_2022, CAR.SUBARU_CROSSTREK_2025):
       ret.dashcamOnly = True
```

After editing, confirm the safety parameter remains present:

```bash
rg -n "dashcamOnly|safetyParam|LKAS_ANGLE" \
  opendbc_repo/opendbc/car/subaru/interface.py
```

## 5. Split Gen1 and Gen2 cruise state in CarState

Open:

```text
opendbc_repo/opendbc/car/subaru/carstate.py
```

The July base changed all angle cars to use `ES_Status`. That is correct for the
newer Gen2 angle cars but does not match the working Gen1 Forester. Production
CarState and panda safety must derive engagement from the same generation-
specific message.

The tested logic is:

```python
    cp_es_brake = cp_alt if self.CP.flags & SubaruFlags.GLOBAL_GEN2 else cp_cam

    if self.CP.flags & SubaruFlags.LKAS_ANGLE:
      if self.CP.flags & SubaruFlags.GLOBAL_GEN2:
        # ES_Brake->Cruise_Activated stays high on brake at standstill on newer Gen2 cars.
        ret.cruiseState.enabled = cp_es_brake.vl["ES_Status"]["Cruise_Activated"] != 0
      else:
        # Gen1 Forester uses ES_Brake from the camera bus.
        ret.cruiseState.enabled = cp_es_brake.vl["ES_Brake"]["Cruise_Activated"] != 0
      ret.cruiseState.available = cp_cam.vl["ES_DashStatus"]["Cruise_On"] != 0
```

Preserve the new base's hybrid handling. Do not combine the Forester with the
hybrid case merely because both use `ES_Brake`; their reasons and bus/message
availability differ.

## 6. Split Gen1 and Gen2 panda safety buses

Open:

```text
opendbc_repo/opendbc/safety/modes/subaru.h
```

### RX checks

The angle RX-check configuration must accept generation-specific cruise message
parameters. On the tested base, the macro became:

```c
#define SUBARU_LKAS_ANGLE_RX_CHECKS(alt_bus, cruise_msg, cruise_bus, cruise_frequency) \
  /* retain all current throttle, steering, wheel-speed, brake-status, and angle checks */ \
  {.msg = {{cruise_msg, cruise_bus, 8, cruise_frequency, .max_counter = 15U, .ignore_quality_flag = true}, { 0 }, { 0 }}}, \
  /* retain the current Steering_2 check */
```

Use the current file's full macro rather than replacing it with this abbreviated
illustration. Only parameterize the cruise row.

Instantiate the checks as follows:

```c
  static RxCheck subaru_lkas_angle_rx_checks[] = {
    SUBARU_LKAS_ANGLE_RX_CHECKS(SUBARU_MAIN_BUS, MSG_SUBARU_ES_Brake, SUBARU_CAM_BUS, 50U)
  };

  static RxCheck subaru_lkas_angle_gen2_rx_checks[] = {
    SUBARU_LKAS_ANGLE_RX_CHECKS(SUBARU_ALT_BUS, MSG_SUBARU_ES_Status, SUBARU_ALT_BUS, 20U)
  };
```

### Cruise engagement decoding

The tested RX-hook logic is:

```c
  if (subaru_lkas_angle) {
    if (subaru_gen2 && (msg->addr == MSG_SUBARU_ES_Status) && (msg->bus == SUBARU_ALT_BUS)) {
      bool cruise_engaged = (msg->data[3] >> 5) & 1U;
      pcm_cruise_check(cruise_engaged);
    } else if (!subaru_gen2 && (msg->addr == MSG_SUBARU_ES_Brake) && (msg->bus == SUBARU_CAM_BUS)) {
      bool cruise_engaged = (msg->data[4] >> 7) & 1U;
      pcm_cruise_check(cruise_engaged);
    }
  }
```

### TX buses

Verify rather than assume:

- Gen1 `ES_LKAS_ANGLE` TX must be on `SUBARU_MAIN_BUS`.
- Gen2 `ES_LKAS_ANGLE` TX must be on `SUBARU_ALT_BUS`.

The tested July base already had this TX split, so no TX change was needed.

### VM angle safety

Locate the angle command check:

```bash
rg -n "steer_angle_cmd_checks_vm|AngleSteeringLimits|AngleSteeringParams" \
  opendbc_repo/opendbc/safety/modes/subaru.h
```

Keep the new base's VM-based implementation. Never restore historical
speed-breakpoint angle limits merely to make an old patch apply.

## 7. Update Subaru safety tests

Open:

```text
opendbc_repo/opendbc/safety/tests/test_subaru.py
```

Keep the shared VM tests, including lateral acceleration and jerk. For the Gen1
angle safety class, route its cruise fixture through `ES_Brake` on the camera
bus:

```python
class TestSubaruGen1AngleStockLongitudinalSafety(TestSubaruStockLongitudinalSafetyBase, TestSubaruAngleSafetyBase):
  ALT_CAM_BUS = SUBARU_CAM_BUS
  FLAGS = SubaruSafetyFlags.LKAS_ANGLE
  # Keep the current TX, relay-malfunction, and forwarding expectations.

  def _pcm_status_msg(self, enable):
    values = {"Cruise_Activated": enable}
    return self.packer.make_can_msg_safety("ES_Brake", self.ALT_CAM_BUS, values)
```

Do not change the Gen2 fixture: it must continue producing `ES_Status` on the
alternate bus.

## 8. Review the complete change set

The tested port changed only these functional files:

```text
launch_env.sh
opendbc_repo/opendbc/car/subaru/carstate.py
opendbc_repo/opendbc/car/subaru/interface.py
opendbc_repo/opendbc/safety/modes/subaru.h
opendbc_repo/opendbc/safety/tests/test_subaru.py
```

Deployment also changed `.gitmodules` so a fresh comma installation can fetch
the custom opendbc commit.

Review every diff:

```bash
git diff --check
git diff -- launch_env.sh .gitmodules
git -C opendbc_repo diff --check
git -C opendbc_repo diff -- \
  opendbc/car/subaru/carstate.py \
  opendbc/car/subaru/interface.py \
  opendbc/safety/modes/subaru.h \
  opendbc/safety/tests/test_subaru.py
git status --short
git -C opendbc_repo status --short
```

If other files appear, stop and determine why.

## 9. Run the focused safety tests

Use the project's normal environment when available. To create an isolated
temporary environment like the one used for the tested port:

```bash
python3 -m venv /tmp/opendbc-forester-test-venv
/tmp/opendbc-forester-test-venv/bin/pip install \
  "numpy==1.26.4" cffi pycapnp pycryptodome tqdm

cd /path/to/openpilot/opendbc_repo
PYTHONPATH="$PWD" \
  /tmp/opendbc-forester-test-venv/bin/python \
  opendbc/safety/tests/test_subaru.py
```

NumPy 1.26.4 was used because the newest wheel required CPU instructions not
available on the development machine. A newer machine may use the version from
the current lock file instead.

The known-good result was:

```text
Ran 197 tests in 8.345s

OK (skipped=30)
```

The exact test count can change on a newer base. The important requirement is
that all collected Subaru safety tests pass and that Gen1 and Gen2 angle tests,
lateral acceleration, and lateral jerk are still collected.

Also run any newer base-specific Subaru interface/controller tests, build, and
lint commands required by that branch.

## 10. Commit and publish opendbc first

The submodule is often checked out at detached HEAD. Create a branch from the
exact commit selected by the new openpilot base:

```bash
cd /path/to/openpilot/opendbc_repo
git switch --create forester-NEW-BASE
git add \
  opendbc/car/subaru/carstate.py \
  opendbc/car/subaru/interface.py \
  opendbc/safety/modes/subaru.h \
  opendbc/safety/tests/test_subaru.py
git diff --cached --check
git diff --cached
git commit -m "Subaru: port Forester Gen1 angle support"
```

Add or verify the personal remote, then push:

```bash
git remote add poshache https://github.com/Poshache/opendbc.git
# If it already exists, verify it instead:
git remote get-url poshache
git push --set-upstream poshache forester-NEW-BASE
```

Record the new commit:

```bash
git rev-parse HEAD
```

Confirm that exact commit is visible on GitHub before continuing.

## 11. Point fresh installations at the personal opendbc fork

In the top-level `.gitmodules`, change only the opendbc URL:

```diff
 [submodule "opendbc"]
   path = opendbc_repo
-  url = ../../commaai/opendbc.git
+  url = ../../Poshache/opendbc.git
```

This is required because the new submodule commit exists in
`Poshache/opendbc`, not `commaai/opendbc`. Without it, a fresh comma installer
can clone openpilot but fails when fetching the custom opendbc commit.

## 12. Commit and publish openpilot

Return to the top-level repository. Confirm the submodule pointer is the exact
commit just pushed:

```bash
cd /path/to/openpilot
git -C opendbc_repo rev-parse HEAD
git diff --submodule=short -- launch_env.sh .gitmodules opendbc_repo
```

Then commit and push:

```bash
git add launch_env.sh .gitmodules opendbc_repo
git diff --cached --check
git diff --cached --submodule=short
git commit -m "Subaru: port Forester support to angle steering"

git remote add poshache https://github.com/Poshache/openpilot.git
# If it already exists, verify it instead:
git remote get-url poshache
git push --set-upstream poshache forester-NEW-BASE
```

openpilot's Git LFS configuration may try to write to comma.ai's GitLab LFS
store. If the new commit contains no LFS file changes, it is safe to skip only
that pre-push hook:

```bash
git push --no-verify --set-upstream poshache forester-NEW-BASE
```

Do not skip the hook if the commit adds or changes LFS-tracked files.

Verify local and remote tips match:

```bash
git rev-parse HEAD
git ls-remote --heads poshache forester-NEW-BASE
git -C opendbc_repo rev-parse HEAD
git -C opendbc_repo ls-remote --heads poshache forester-NEW-BASE
git status --short --branch
git -C opendbc_repo status --short --branch
```

## 13. Install on comma 4

Both GitHub repositories must be public so the installer can fetch them without
credentials.

1. Park safely and keep the comma 4 powered.
2. Connect it to reliable Wi-Fi.
3. Open **Settings**, then **Software**.
4. Select **UNINSTALL** and confirm.
5. Wait for the device to reboot to setup.
6. Reconnect Wi-Fi if requested.
7. Select custom software installation.
8. Enter `installer.comma.ai/Poshache/BRANCH_NAME`.
9. Keep power and Wi-Fi connected through installation and reboot.

For the known-good build, use:

```text
installer.comma.ai/Poshache/forester-jul-port
```

Installation may remain near 92% while compiling. Do not interrupt it merely
because progress pauses for several minutes.

## 14. Validate on the car

Before driving, confirm:

- the UI starts normally;
- the detected platform is Subaru Forester;
- there are no CAN, panda safety, or vehicle-recognition errors;
- controls are disabled while parked until cruise is engaged normally;
- brake, accelerator, cruise cancel, and steering-wheel disengagement inputs
  behave normally.

For the first drive:

- use a low-speed, low-traffic road with good lane markings;
- keep both hands ready on the wheel;
- be ready to brake and disengage immediately;
- first test engagement and disengagement without relying on steering;
- verify braking disengages correctly, including near standstill;
- then verify gentle angle steering in both directions;
- watch for steering faults, unexpected cruise state, CAN errors, oscillation,
  excessive lateral acceleration, or delayed disengagement;
- stop testing immediately if behavior differs from the known-good build.

Passing unit tests does not replace in-car validation.

## 15. Roll back

If installation fails or vehicle behavior is unexpected:

1. Do not continue driving with the custom build engaged.
2. Park safely.
3. Uninstall the custom software from **Settings** > **Software** when the UI is
   available.
4. Reinstall the last known-good Forester branch:

   ```text
   installer.comma.ai/Poshache/forester-jul-port
   ```

5. To restore stock software, reflash using `https://flash.comma.ai`, then use
   the stock openpilot option during setup.

comma support generally requires reproducing hardware issues on current stock
openpilot rather than on a community fork.

## 16. Updating an existing Forester branch versus porting again

There are two different operations:

### Small update with unchanged Subaru architecture

If the new upstream commits do not modify Subaru angle control, Subaru safety,
CarState cruise logic, the opendbc pointer, or relevant tests, merge or rebase
carefully and rerun the full validation process.

Inspect first:

```bash
git diff OLD_BASE..NEW_BASE -- \
  launch_env.sh .gitmodules opendbc_repo

git -C opendbc_repo diff OLD_OPENDBC..NEW_OPENDBC -- \
  opendbc/car/subaru \
  opendbc/safety/modes/subaru.h \
  opendbc/safety/tests/test_subaru.py
```

### New controller or safety architecture

If any relevant logic changed, start a fresh branch from the new base and port
the Forester behavior again using this guide. Do not resolve conflicts by always
choosing the old Forester side. Preserve the new controller and safety design,
then add only the Gen1 Forester-specific behavior.

Reconfirm these invariants after every future port:

- Forester remains Gen1 `LKAS_ANGLE`.
- Forester angle TX is on the main bus.
- Forester cruise engagement uses camera-bus `ES_Brake` unless new route data
  proves the vehicle behavior changed.
- Gen2 angle cars retain their correct alternate-bus paths and `ES_Status`.
- production CarState and panda safety use matching engagement sources.
- the current base's strongest angle-limit and vehicle-model safety checks are
  preserved.
- tests cover both generations and the current lateral acceleration/jerk rules.
- the published openpilot submodule pointer is fetchable from the URL committed
  in `.gitmodules`.

## Known-good change summary

For reference, the tested port consisted of:

- `launch_env.sh`: force `SUBARU_FORESTER_2022`;
- `interface.py`: validate Forester outside dashcam-only mode while keeping
  Crosstrek 2025 validated;
- `carstate.py`: Gen1 angle uses `ES_Brake`, Gen2 angle uses `ES_Status`;
- `subaru.h`: Gen1 angle cruise RX on camera bus and Gen2 on alternate bus,
  while preserving VM angle safety;
- `test_subaru.py`: Gen1 angle cruise fixture uses camera-bus `ES_Brake`;
- `.gitmodules`: fetch custom opendbc from `Poshache/opendbc`;
- top-level submodule pointer: publish the exact tested opendbc commit.

That is the behavior to reproduce—not necessarily a permanent line-for-line
patch—when moving to a newer openpilot or opendbc base.
