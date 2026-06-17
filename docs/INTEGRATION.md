# Using the Rekise fork in a ROS 2 package (overview)

How to add `generate_parameter_library` (Rekise fork) to **any** ROS 2 package and get:
typed + validated parameters and the three behaviour flags (`read_only` / `volatile` / `required_restart`).

> **The two focused guidelines are authoritative — use those:**
> - **[GUIDE_1_add_to_package.md](GUIDE_1_add_to_package.md)** — add the library to a package (deps, spec, CMake, build).
> - **[GUIDE_2_integrate_in_code.md](GUIDE_2_integrate_in_code.md)** — use it in the node.
>
> **Rekise convention (since 2026-06-17):** the node writes **no** override code. The generated
> `ParamListener` auto-reads `CONFIG_OVERRIDE_FILE` (set per-node by the launch), applies the
> `user_config` override on top of the resolved baseline, and **persists accepted runtime changes
> there during validation** — automatic + durable, zero node code. §4–§5 below describe the same
> mechanism with the (now-unnecessary) explicit `set_override_file` call; the library does it for you.
>
> README: §2 = param-def keys, §2a = validators, §1 = flag contract.

Worked end-to-end example: `vessel_demo/vessel_ws/src/rkse_hsal_depth_bluerobotics_driver` (lifecycle).

---

## 0. The 5 steps

1. Put the fork in your workspace `src/`.
2. Depend on `generate_parameter_library` (package.xml + CMakeLists).
3. Write `config/params.yaml` (the spec) — root key = your node name.
4. `generate_parameter_library(<lib> config/params.yaml)` in CMake; link it.
5. In the node: build a `ParamListener`, call `set_override_file(...)`, read `get_params()`.

---

## 1. Get the fork into your workspace

The fork overlays the one in `/opt/ros`. Clone the feature branch into your workspace `src/`:

```bash
cd <your_ws>/src
git clone -b rekise/param-flags-override-persistence \
    git@github.com:pavithran-rekise/generate_parameter_library.git
```

Or pin it in a `.repos` file (recommended for shared workspaces):

```yaml
# my_ws.repos  — vcs import src < my_ws.repos
repositories:
  generate_parameter_library:
    type: git
    url: git@github.com:pavithran-rekise/generate_parameter_library.git
    version: rekise/param-flags-override-persistence
```

> When both the fork and the stock package are present, colcon warns about overriding.
> Building the fork in your overlay is what you want — it takes precedence.

## 2. Declare the dependency

**`package.xml`**
```xml
<depend>rclcpp</depend>
<depend>rclcpp_lifecycle</depend>   <!-- only if you use a lifecycle node -->
<depend>generate_parameter_library</depend>
```

**`CMakeLists.txt`**
```cmake
find_package(generate_parameter_library REQUIRED)

# Codegen the param library from the spec.
# 1st arg = the cmake target/header name; produces "<name>.hpp".
# The C++ namespace is the YAML ROOT KEY (your node name), NOT this target name.
generate_parameter_library(my_node_parameters config/params.yaml)

add_executable(my_node src/my_node.cpp)
target_link_libraries(my_node my_node_parameters)
ament_target_dependencies(my_node rclcpp)   # + rclcpp_lifecycle if used

install(TARGETS my_node DESTINATION lib/${PROJECT_NAME})
# Ship the spec so the config manager / CI can read it:
install(FILES config/params.yaml DESTINATION share/${PROJECT_NAME})
```

## 3. Write the spec — `config/params.yaml`

Root key = the **node name**. It becomes the generated C++ namespace and the param-listener scope.

```yaml
my_node:                       # <- node name == namespace my_node::
  publish_rate_hz:             # tunable + persisted
    type: double
    default_value: 1.0
    description: "Publish rate (Hz)"
    volatile: false            # opt in to persistence (volatile defaults true)
    validation:
      bounds<>: [0.1, 50.0]

  frame_id:                    # structural — never changes at runtime
    type: string
    default_value: "base_link"
    read_only: true

  filter_window:               # changing it rebuilds buffers -> needs a restart
    type: int
    default_value: 5
    volatile: false            # required alongside required_restart
    required_restart: true
    validation:
      gt_eq<>: [1]

  debug_enabled:               # no flags -> volatile default: live, never persisted
    type: bool
    default_value: false

  filtering:                   # nested group -> params.filtering.enabled, etc.
    enabled:
      type: bool
      default_value: true
      volatile: false
```

Flag choice cheat-sheet (full table in README §1):

| want | flags |
|---|---|
| live tweak, **persist** across relaunch | `volatile: false` |
| live tweak, **don't** persist (default) | *(no flag)* |
| change is saved but only takes effect next launch/configure | `volatile: false` + `required_restart: true` |
| never changeable at runtime | `read_only: true` |

Validators: see README §2a (scalar / array-element / size, each with when-to-use).

## 4. Use it in the node

### Plain `rclcpp::Node` — listener in the constructor

```cpp
#include "my_pkg/my_node_parameters.hpp"   // generated header (target name from CMake)

class MyNode : public rclcpp::Node {
public:
  MyNode() : rclcpp::Node("my_node") {
    // namespace == YAML root key == node name
    param_listener_ = std::make_shared<my_node::ParamListener>(
        get_node_parameters_interface(), get_logger());

    // override layer: persist accepted runtime changes here + reload them on start
    if (const char * f = std::getenv("CONFIG_OVERRIDE_FILE")) {
      param_listener_->set_override_file(f);
    }

    params_ = param_listener_->get_params();   // validated, typed struct
    // use params_.publish_rate_hz, params_.frame_id, params_.filtering.enabled, ...
  }
private:
  std::shared_ptr<my_node::ParamListener> param_listener_;
  my_node::Params params_;
};
```

### `rclcpp_lifecycle::LifecycleNode` — listener in `on_configure`

This is the design's load point: a `cleanup → configure` re-reads the override file, so a
`required_restart` value applies **without a process restart**.

```cpp
CallbackReturn on_configure(const State &) override {
  if (!param_listener_) {                     // create once; params declared on first configure
    param_listener_ = std::make_shared<my_node::ParamListener>(
        get_node_parameters_interface(), get_logger());
  }
  if (const char * f = std::getenv("CONFIG_OVERRIDE_FILE")) {
    param_listener_->set_override_file(f);    // reload latest persisted values
  }
  params_ = param_listener_->get_params();
  // build publishers/timers from params_
  return CallbackReturn::SUCCESS;
}
```

### Reacting to live changes (optional)

`get_params()` returns the latest validated struct each call. For a push callback:

```cpp
param_listener_->setUserCallback([this](const my_node::Params & p){
  // runs after a validated runtime change is applied (volatile or volatile:false)
  params_ = p;
});
```

## 5. The override file (`user_config`)

A small ROS-format file holding only changed values. The node persists accepted
`volatile: false` / `required_restart` changes into it, and reloads it on start/configure.

```yaml
my_node:
  ros__parameters:
    publish_rate_hz: 2.0
    filter_window: 9        # required_restart -> takes effect this (re)launch
```

How a node finds it (two common wirings):
- **env var** `CONFIG_OVERRIDE_FILE=/path/user_config.yaml` (used by the examples / config-manager launch);
- a normal ROS launch arg you pass to `set_override_file(...)`.

If the env var is unset, `set_override_file` is never called → the node runs on compiled defaults,
nothing persists. Safe standalone default.

## 6. Build & run

```bash
cd <your_ws>
colcon build --packages-up-to my_pkg
source install/setup.bash

# standalone, compiled defaults (no persistence)
ros2 run my_pkg my_node

# with persistence/override layer
echo 'my_node: {ros__parameters: {}}' > /tmp/user_config.yaml
CONFIG_OVERRIDE_FILE=/tmp/user_config.yaml ros2 run my_pkg my_node
```

Drive it:
```bash
ros2 param set /my_node publish_rate_hz 2.0   # volatile:false -> live + written to user_config
ros2 param set /my_node debug_enabled true    # no flag -> live, NOT written
ros2 param set /my_node filter_window 9       # required_restart -> rejected live, saved for next launch
ros2 param set /my_node frame_id x            # read_only -> rejected
cat /tmp/user_config.yaml                      # see exactly what persisted
```
Relaunch with the same `CONFIG_OVERRIDE_FILE` → persisted values (incl. `required_restart`) are reapplied.

## 7. Gotchas (all verified — see TEST_RESULTS.md)

- **Namespace = YAML root key**, not the CMake target. Root `my_node:` → `my_node::ParamListener`.
- **Prefixed listeners** (ros2_control style, `ParamListener(itf, logger, "prefix")`): the override
  file stores **relative** (unprefixed) keys; the loader re-applies the prefix. Works transparently.
- **Nested groups**: the override file accepts both a nested map (`filtering: {enabled: false}`) and a
  flat dotted key (`filtering.enabled: false`).
- **Fixed-size arrays** (`double_array_fixed_3`) persist + reload fine; over-capacity sets are rejected.
- **Bad override file never crashes the node** — unknown/invalid/read-only/malformed keys are skipped
  per-key; valid keys still apply; a totally garbage file is ignored and defaults are kept.
- **`ros2 param set` CLI quirks** (not the library): negative values need a leading space (`' -1'`),
  empty arrays `'[]'` aren't cleanly settable, `#` in a string is read as a YAML comment.

## 8. Checklist

- [ ] fork cloned in `src/` on branch `rekise/param-flags-override-persistence`
- [ ] `generate_parameter_library` in package.xml + CMakeLists
- [ ] `config/params.yaml` root key == node name; flags + validators set
- [ ] `generate_parameter_library(<target> config/params.yaml)`; `target_link_libraries(node <target>)`
- [ ] include `"<pkg>/<target>.hpp"`; `<node_name>::ParamListener` in ctor (or `on_configure`)
- [ ] `set_override_file(...)` wired from `CONFIG_OVERRIDE_FILE` (or a launch arg)
- [ ] `params_ = listener->get_params();` then use the typed struct
- [ ] spec installed to `share/<pkg>` for the config manager / CI
