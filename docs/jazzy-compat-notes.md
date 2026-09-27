# ROS 2 Jazzy compatibility notes

Findings from porting `avular_origin_simulation` (branch `ros2-jazzy`, based on
upstream `main` = release `origin_1.0/3`) to ROS 2 Jazzy / Ubuntu 24.04
(`osrf/ros:jazzy-desktop-full`). Diagnosed on 2026-09-27. The `.deb` files were
only inspected (package metadata, file lists, ELF `NEEDED` entries); nothing was
decompiled.

## TL;DR

| Package (`cmd_vel_controller/*.deb`) | Jazzy status | Resolution |
|---|---|---|
| `origin-msgs` 2.3.0 | Humble binary, not installable | **Solved**: build from public source (`avular-robotics/example_origin_ros`, `origin_msgs/`) |
| `loggercxx` 3.3.0 | Humble binary, not installable | No public source. Needs a Jazzy build from Avular |
| `rclcpp-avular` 3.3.0 | Humble binary, not installable | No public source. Needs a Jazzy build from Avular |
| `rclcpp-mocks` 3.3.1 | Humble binary, not installable | No public source. Needs a Jazzy build from Avular |
| `cmd-vel-controller` 1.7.2 | Humble binary, not installable | No public source. Simulation runs without it (`use_cmd_vel_controller:=False`, the default); real robot needs a Jazzy build from Avular |

The workspace (`origin_one_description` + `origin_one_gazebo`) **builds on Jazzy
without the `.deb`s**: `colcon build --symlink-install` succeeds for both packages.

## Why the `.deb`s cannot be used on Jazzy

1. **They target Humble / Ubuntu 22.04.** `Depends:` lists `ros-humble-*`
   packages, `libfmt8` (Noble has `libfmt9`) and `libpython3.10` (Noble has
   3.12). They also depend on the Avular-internal packages `cmake-avular` and
   `ament-copyright-avular`, which are not in any public apt source.
2. **Everything installs under `/opt/ros/humble/`** (`lib/`, `include/`,
   `share/`, `local/lib/python3.10/dist-packages`), so nothing is on a Jazzy
   environment's search paths.
3. **They are linked against Humble's ABI.** ROS 2 does not keep ABI
   compatibility between distros, so forcing the install (`dpkg --force-depends`)
   or copying the files into `/opt/ros/jazzy` would not produce working
   binaries. Examples of `NEEDED` entries:
   - `cmd_vel_controller`: `librclcpp.so`, `librcl.so`, `librmw.so`,
     `libclass_loader.so`, `libfmt.so.8`, `librclcpp_avular.so.3`,
     `liborigin_msgs__rosidl_typesupport_cpp.so`
   - `librclcpp_avular.so.3.3.0`: `librclcpp.so`, `librclcpp_lifecycle.so`
   - `liborigin_msgs__rosidl_typesupport_fastrtps_cpp.so`: `libfastcdr.so.1`
     (Jazzy ships Fast-CDR 2.x)
   - `libloggercxx.so.3.3.0`: only libc/libstdc++. The library itself is
     portable, but the package still installs into `/opt/ros/humble` and
     depends on `cmake-avular`.

`apt-get install -s` of all five `.deb`s on Noble fails with 30+ unmet
dependencies (every `ros-humble-*` entry, plus `cmake-avular`,
`ament-copyright-avular`, `libfmt8` and `libpython3.10`).

### Control check: the same `.deb`s on Humble

In `osrf/ros:humble-desktop-full`, all `ros-humble-*` dependencies resolve.
Only `cmake-avular` and `ament-copyright-avular` are missing. They look like
build/lint tooling declared as runtime `Depends:`, so the README's
`sudo apt install ./*.deb` fails on a clean Humble install too. The workaround
works:

```bash
sudo apt install libfmt-dev ros-humble-diagnostic-updater ros-humble-ament-cmake-pytest \
  ros-humble-launch-testing-ament-cmake ros-humble-ament-cmake-gtest ros-humble-ament-lint-auto
sudo dpkg -i --ignore-depends=cmake-avular,ament-copyright-avular \
  loggercxx_3.3.0_amd64.deb rclcpp-avular_3.3.0_amd64.deb origin-msgs_2.3.0_amd64.deb \
  cmd-vel-controller_1.7.2_amd64.deb rclcpp-mocks_3.3.1_amd64.deb
```

After that, `ldd` reports no missing libraries. `cmd_vel_controller` runs with
`origin_one_gazebo/config/cmd_vel_controller.yaml` and brings up
`/robot/cmd_vel`, `/robot/control_mode` and the
`set_control_mode` / `reset_control_mode` / `previous_control_mode` services.
So the binaries are fine; the incompatibility is purely Humble vs Jazzy.

## Upstream availability (checked 2026-09-27)

- `avular-robotics` GitHub org: no Jazzy branches, tags or releases in
  `avular_origin_simulation`, `avular_origin_description`, `example_origin_ros`
  or `user-container`. Upstream README states "We currently only support ROS2
  Humble on Ubuntu 22.04."
- **`origin_msgs` source is public** in
  [`example_origin_ros`](https://github.com/avular-robotics/example_origin_ros)
  (`origin_msgs/`, `main` @ `bc9b47c`). All 7 `.msg` and 3 `.srv` files are
  identical (whitespace-insensitive diff) to the interfaces shipped in
  `origin-msgs_2.3.0`. It builds with plain `colcon build` on Jazzy.
  - Caveats: its `package.xml` says `<version>1.0.0</version>` (the `.deb` says
    2.3.0) and `<license>Avular Confidential</license>`, although the repo
    itself is MIT-licensed. Worth confirming with Avular.
- No public source for `loggercxx`, `rclcpp-avular`, `rclcpp-mocks` or
  `cmd-vel-controller`.

## rosdep

`rosdep check --from-paths src --ignore-src` on Noble reports two unresolvable
keys:

- `ament_copyright_avular` (in `origin_one_description`, a test dependency)
- `cmd_vel_controller` (`<depend>` in `origin_one_gazebo/package.xml`)

colcon ignores both, so the build is unaffected. Use `rosdep install ... --skip-keys
"ament_copyright_avular cmd_vel_controller"` when installing dependencies.

## What `cmd_vel_controller` does (from its config, not its binary)

`origin_one_gazebo/config/cmd_vel_controller.yaml` describes a control-mode
multiplexer in namespace `/robot`:

- It reads twist inputs per mode: `cmd_vel_joy` (manual, manual_mapping),
  `cmd_vel_teleop` (teleops), `cmd_vel_aut` (autopilot, docking) and
  `cmd_vel_user` (user).
- It publishes the selected one on `/robot/cmd_vel`. The current mode goes out
  on `/robot/control_mode` (`origin_msgs/ControlMode`, latched).
- Per-mode velocity/acceleration limits apply, and a 1.0 s timeout zeroes a
  stale input.
- The mode is switched through `origin_msgs` services (`SetControlMode`,
  `ReturnControlMode`).

In simulation the Gazebo bridge listens on `/robot/cmd_vel` directly, so
without the controller any node can drive the robot by publishing there
(Nav2 can remap its `cmd_vel` output to `/robot/cmd_vel`). A Jazzy-native
stand-in, if needed, can be built from stock packages (`twist_mux` +
`nav2_velocity_smoother`, or a small node). No Avular code is needed for that.

## Open items

- Ask Avular support for Jazzy/Noble builds (or source) of `loggercxx`,
  `rclcpp-avular`, `rclcpp-mocks` and `cmd-vel-controller`. This matters for
  the real robot, which uses `cmd_vel_controller` on board.
- Confirm the licence status of `origin_msgs` (repo is MIT, `package.xml` says
  "Avular Confidential").
- Upstream simulation targets Gazebo Fortress (`IGN_GAZEBO_RESOURCE_PATH`,
  Fortress-era `ros_gz`). Jazzy ships Gazebo Harmonic, so the launch and SDF
  changes are part of the Gazebo simulation baseline work, not of this
  diagnosis.
