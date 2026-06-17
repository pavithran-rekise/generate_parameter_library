# Guideline 1 — Add the validation library to a ROS 2 package

How to pull the Rekise **`generate_parameter_library`** (GPL) fork into a package so the package gets
typed + validated parameters, the three behaviour flags (`read_only` / `volatile` / `required_restart`),
and the persistent override layer.

This guide is **package setup only** (deps, CMake, the spec file, build). Using the generated code in
your node is **Guideline 2 — Integrate in code**.

> Reference: fork [`README.md`](../README.md) §2 (param-def keys), §2a (full validator list), §1 (flag contract).
> Real building example to copy from: `rkse_hsal_depth_bluerobotics_driver` (lifecycle node + GPL).

---

## Steps at a glance

1. Make the fork available in your workspace.
2. `package.xml` — depend on `generate_parameter_library`.
3. `config/params.yaml` — write the spec (root key = node name).
4. `CMakeLists.txt` — `find_package` + `generate_parameter_library(...)` + link + install the spec.
5. Build with `--allow-overriding generate_parameter_library`.

---

## 1. Make the fork available

The fork **overlays** the stock `generate_parameter_library` in `/opt/ros`. Get it into your workspace
`src/` one of three ways:

```bash
# (a) clone the feature branch
cd <your_ws>/src
git clone -b rekise/param-flags-override-persistence \
    git@github.com:pavithran-rekise/generate_parameter_library.git
```
```yaml
# (b) pin it in a .repos file  (vcs import src < my_ws.repos)
repositories:
  generate_parameter_library:
    type: git
    url: git@github.com:pavithran-rekise/generate_parameter_library.git
    version: rekise/param-flags-override-persistence
```
```bash
# (c) symlink an existing checkout (how vessel_demo/vessel_ws does it)
ln -s /abs/path/rkse_param/generate_parameter_library/generate_parameter_library    src/generate_parameter_library
ln -s /abs/path/rkse_param/generate_parameter_library/generate_parameter_library_py src/generate_parameter_library_py
```

> When both the fork and the stock package are present, colcon warns about overriding — that's
> expected; building the fork in your overlay is what gives you the flags + override layer (step 5).

## 2. `package.xml` — declare the dependency

```xml
<depend>rclcpp</depend>
<depend>rclcpp_lifecycle</depend>          <!-- only if you use a lifecycle node -->
<depend>generate_parameter_library</depend>
```
(Real example: `rkse_hsal_depth_bluerobotics_driver/package.xml`.)

## 3. `config/params.yaml` — write the spec

The library is **generated from this file**. The **root key = your node name**; it becomes the C++
namespace and the param-listener scope. Each leaf needs `type` + `default_value`; add validators and
behaviour flags as needed.

```yaml
depth_node:                       # <- node name == generated namespace `depth_node::`
  publish_rate_hz:                # tunable + persisted
    type: double
    default_value: 1.0
    volatile: false               # opt in to persistence (volatile defaults true)
    validation:
      bounds<>: [0.1, 50.0]

  frame_id:                       # structural — cannot change at runtime
    type: string
    default_value: "pressure_altitude"
    read_only: true

  filter_enabled:                 # change takes effect only on next (re)launch/configure
    type: bool
    default_value: true
    volatile: false               # required alongside required_restart
    required_restart: true

  read_failure_threshold:
    type: int
    default_value: 5
    volatile: false
    validation:
      gt_eq<>: [1]
```

Flag cheat-sheet (full table in README §1):

| want | flags |
|---|---|
| live tweak, **persist** across relaunch | `volatile: false` |
| live tweak, **don't** persist (default) | *(no flag)* |
| saved now, applies next launch/configure | `volatile: false` + `required_restart: true` |
| never changeable at runtime | `read_only: true` |

Validators (scalar / array-element / size) — see README §2a for the full list + when-to-use. Common:
`bounds<>` · `gt<>` · `gt_eq<>` · `lt<>` · `lt_eq<>` · `one_of<>` · `fixed_size<>` · `not_empty<>`.

> Keep param names **flat scalars** (no dotted keys). Nested groups are allowed and become
> `params.<group>.<name>`.

## 4. `CMakeLists.txt`

```cmake
find_package(generate_parameter_library REQUIRED)

# Codegen the param library from the spec.
#   1st arg = cmake target / header name -> produces "depth_parameters.hpp"
#   the C++ NAMESPACE is the YAML root key (depth_node), NOT this target name.
generate_parameter_library(depth_parameters config/params.yaml)

add_executable(depth_node src/depth_node.cpp)
target_link_libraries(depth_node depth_parameters)
ament_target_dependencies(depth_node rclcpp rclcpp_lifecycle std_msgs)

install(TARGETS depth_node DESTINATION lib/${PROJECT_NAME})
# Ship the spec so the Configuration Manager / CI can read it:
install(FILES config/params.yaml DESTINATION share/${PROJECT_NAME})
```
(Real example: `rkse_hsal_depth_bluerobotics_driver/CMakeLists.txt`.)

## 5. Build

Always pass `--allow-overriding` so your fork wins over the stock package:

```bash
cd <your_ws>
source /opt/ros/humble/setup.bash
colcon build --packages-up-to <your_pkg> --allow-overriding generate_parameter_library
source install/setup.bash
```

If you edited the spec and codegen looks stale, clean-rebuild the package:
```bash
rm -rf build/<your_pkg> install/<your_pkg> && colcon build --packages-select <your_pkg> --allow-overriding generate_parameter_library
```

## Verify the library was generated

```bash
find build/<your_pkg> -name "depth_parameters.hpp"     # the generated header exists
```
The header lives under your package's namespace — included as
`"<your_pkg>/depth_parameters.hpp"` (used in Guideline 2).

---

## Checklist

- [ ] fork available in `src/` (clone / .repos / symlink), branch `rekise/param-flags-override-persistence`
- [ ] `package.xml` depends on `generate_parameter_library`
- [ ] `config/params.yaml` written; root key == node name; flags + validators set
- [ ] `CMakeLists.txt`: `find_package` + `generate_parameter_library(<target> config/params.yaml)` + `target_link_libraries(<node> <target>)`
- [ ] spec installed to `share/<pkg>` (for the config manager / CI)
- [ ] builds with `--allow-overriding generate_parameter_library`; generated `.hpp` present

→ Next: **Guideline 2 — Integrate in code.**
