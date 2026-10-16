# CLAUDE.md — Karinca Robotics fork of `opennav_coverage`

Fork of [open-navigation/opennav_coverage](https://github.com/open-navigation/opennav_coverage),
used by the Ari V1 floor cleaner (`kr_robot_ws`, `karinca_ari_v1`) for complete-coverage cleaning.
Decisions and setup live in the workspace: `karinca_ari_v1/DECISIONS.md` (D1 Fields2Cover, D2
branches) and `kr_ros2_docs/docs/setup/` (`CLAUDE.md` → *Fields2Cover v2*, `03_overlays.md`).

## Why the fork exists

We do not build against the Nav2 of the ROS distro. Our Nav2 is a source overlay
(`~/temporary_overlays/nav2_ws`) that tracks **Nav2 `main` (HEAD)** on every distro, Jazzy
included. Upstream's distro branches (`jazzy`, `jazzy-v2`, `lyrical`) are written for that distro's
Nav2 API and do not compile against HEAD. Upstream `main` is written for Rolling, which uses the
same Nav2 API as HEAD, so it is the closest base: it needs only the small delta below.

## Branch model

- **`kr-jazzy-v2`** = upstream `main` + the kr delta. Built on Jazzy against Nav2 HEAD.
- Upstream `main` is on Fields2Cover **v2**; we install the patched v2.1.1 into `/usr/local`
  (never `ros-$ROS_DISTRO-fields2cover`).
- `kr-lyrical` (upstream `lyrical` + the same delta) is expected to be replaced by this branch once
  it is verified on Lyrical.

## The kr delta on top of upstream `main`

Keep it this small. Each item is required by Nav2 HEAD, not a feature:

1. **`nullptr` for `completion_callback`** in the `create_action_server(...)` calls of
   `opennav_coverage/src/coverage_server.cpp` and `opennav_row_coverage/src/row_coverage_server.cpp`.
   Nav2 HEAD added that argument to `nav2::SimpleActionServer` (Nav2 #6279) before
   `server_timeout`; without it the `std::chrono::milliseconds` lands in the callback slot and
   fails to compile.
2. **`uint16 GOAL_REJECTED=1` and `uint16 SEND_GOAL_FAILURE=2`** in the result of both
   `NavigateCompleteCoverage.action` and `ComputeCoveragePath.action` (Nav2 #6279 / #6289).
   - `NavigateCompleteCoverage`: required to **compile** (`BtActionServer` uses them).
   - `ComputeCoveragePath`: compiles without them, but `nav2_behavior_tree::BtActionNode` sets
     `error_code_id` on a rejected / unsendable goal only `if constexpr (requires
     {Result::GOAL_REJECTED;})` (falling back to `UNKNOWN`, which this action lacks too). Without
     them the BT node fails with `error_msg` but **no error code**. No test covers this path.
3. **Explicit `ament_index_cpp` dependency** in `opennav_coverage_navigator` (`find_package`,
   `dependencies`, `dep_targets`, `package.xml`, and `#include
   "ament_index_cpp/get_package_share_directory.hpp"` in `src/coverage_navigator.cpp` and
   `test/test_coverage_navigator.cpp`). Upstream `main` relies on a transitive include that works on
   Jazzy; upstream `lyrical` carries the explicit one, so it is kept for a single cross-distro
   branch.

## Merging upstream

1. `git fetch origin && git merge origin/main` on `kr-jazzy-v2`.
2. Conflicts are expected only in the delta's lines. Re-apply the three items above; check whether
   upstream has already absorbed one, and drop it from this file if so.
3. If Nav2 HEAD changed an API in the meantime, fix by hand from the compiler output and add the
   fix to the delta list above with the reason.
4. Build and test as below. Do not cherry-pick from the upstream distro branches (`jazzy*`,
   `lyrical`): they target other Nav2 APIs.

## Build and test

```bash
source ~/ros2init.bash
source ~/temporary_overlays/nav2_ws/install/local_setup.bash
rosdep install --from-paths . ~/temporary_overlays/nav2_ws/src --ignore-src --skip-keys fields2cover -y
colcon build --symlink-install
RMW_IMPLEMENTATION=rmw_fastrtps_cpp colcon test && colcon test-result
ldd install/opennav_coverage/lib/libopennav_coverage_core.so | grep -iE 'fields2cover|ortools|not found'
```

Traps:
- **Run the tests with `RMW_IMPLEMENTATION=rmw_fastrtps_cpp`.** Under `rmw_zenoh_cpp` every gtest
  passes and then aborts at process exit (rc 134, `The Thread Local Storage inside Tokio is
  destroyed`), so CTest reports failures that are not ours.
- **`export Eigen3_DIR=/usr/share/eigen3/cmake`** (in `~/ros2init.bash`): `ortools-vendor` otherwise
  makes CMake pick its bundled Eigen. A build directory configured before that was set keeps the
  cached value; pass `--cmake-args -DEigen3_DIR=/usr/share/eigen3/cmake` once.
- `ldd` must show `libFields2Cover.so => /usr/local/lib/...` and `libortools.so.9 =>
  /opt/ros/$ROS_DISTRO/opt/ortools_vendor/lib/...`. `undefined reference to 'absl::lts_…'` at link
  time means the shell sourced ROS before `ortools-vendor` was installed: open a fresh shell.
