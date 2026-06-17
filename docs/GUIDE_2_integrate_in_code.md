# Guideline 2 — Integrate the validation library in code

How to consume the generated parameter library inside your node: build a `ParamListener`, wire the
override layer, read the validated typed struct, and react to live changes. Assumes the package is
already set up per **Guideline 1 — Add to package** (deps, `config/params.yaml`, CMake codegen).

> **Copy-paste starting point:** `config_manager/examples/rkse_example_driver/` (full template:
> spec + node + launch + CMake). Real in-tree example: `rkse_hsal_depth_bluerobotics_driver/src/depth_node.cpp`.
> Flag contract: fork README §1.

---

## What codegen gives you

From `config/params.yaml` (root key `depth_node:`) + `generate_parameter_library(depth_parameters …)`:

- header — `#include "<your_pkg>/depth_parameters.hpp"`  (target name = `depth_parameters`)
- namespace — **`depth_node`** (the YAML **root key**, NOT the CMake target)
- types — `depth_node::Params` (validated struct) and `depth_node::ParamListener`

> **The #1 gotcha:** namespace = YAML root key, header name = CMake target. Mixing them up is the
> most common build error.

---

> **Rekise convention — the node writes NO override code; the generated library handles it.**
> The generated `ParamListener` auto-reads `CONFIG_OVERRIDE_FILE` (set per-node by the launch),
> applies the user override on top of the resolved baseline, and **persists accepted runtime changes
> there during validation**. So persistence is automatic and durable across relaunch — with zero
> override code in your node. You just build the listener + `get_params()`.
> (One env per node *process*; composition is deferred — see the gotchas.)

## Plain `rclcpp::Node` — listener in the constructor

```cpp
#include "rkse_hsal_depth_bluerobotics_driver/depth_parameters.hpp"

class DepthNode : public rclcpp::Node {
public:
  DepthNode() : rclcpp::Node("depth_node") {
    // namespace == YAML root key == node name. The listener loads the baseline params, then
    // auto-applies the override layer (CONFIG_OVERRIDE_FILE, set by the launch) on top.
    param_listener_ = std::make_shared<depth_node::ParamListener>(
        get_node_parameters_interface(), get_logger());
    params_ = param_listener_->get_params();      // validated, typed struct (baseline + override)
    // use params_.publish_rate_hz, params_.frame_id, params_.filter_enabled, ...
  }
private:
  std::shared_ptr<depth_node::ParamListener> param_listener_;
  depth_node::Params params_;
};
```

## `rclcpp_lifecycle::LifecycleNode` — listener in `on_configure`

The design's load point: params are loaded + validated on configure.

```cpp
CallbackReturn on_configure(const State &) override {
  if (!param_listener_) {                         // create once; params declared on first configure
    param_listener_ = std::make_shared<depth_node::ParamListener>(
        get_node_parameters_interface(), get_logger());
  }
  params_ = param_listener_->get_params();        // baseline + auto-applied override layer
  // build publishers / timers from params_
  return CallbackReturn::SUCCESS;
}
```

## React to live changes

`get_params()` returns the latest validated struct on each call. To pick up runtime changes in a
loop (poll), or via a push callback:

```cpp
// (poll) e.g. in a timer tick:
if (param_listener_->is_old(params_)) {           // a validated change landed
  params_ = param_listener_->get_params();
}

// (push) callback after each accepted change:
param_listener_->setUserCallback([this](const depth_node::Params & p){ params_ = p; });
```

## What the flags do at runtime (no extra code — the library enforces it)

With the spec from Guideline 1, `ros2 param set` behaves like this automatically:

| `ros2 param set /depth_node …` | flag | result |
|---|---|---|
| `publish_rate_hz 2.0` | `volatile: false` | accepted, live, **persisted** to the override file (durable) |
| `read_failure_threshold 0` | bounds/`gt_eq` | **rejected** (validation) — live value unchanged |
| `filter_enabled false` | `required_restart` | **rejected live**, **persisted** → applied on next launch |
| `frame_id x` | `read_only` | **rejected** (native) |
| a no-flag param | *(none)* | accepted, live, **not** persisted (volatile) |

Your code just reads `params_` — validation, persistence and rejection all happen inside the
generated listener. You write none of it.

## The override file (`user_config`) — auto-loaded + auto-written by the library

A small ROS-format file with only the changed values. The launch points the library at it via the
`CONFIG_OVERRIDE_FILE` env (per node); the library loads it on top of the baseline at construct and
writes accepted runtime changes back to it.

```yaml
/depth1/depth_node:                # FQN key (namespace/node) when launched under a namespace
  ros__parameters:
    fluid_density: 1050.0
    filter_enabled: false          # required_restart -> applied on next (re)launch
```

A **bad value** in this file is **skipped per-key** at load (out-of-range / read-only / malformed) —
the node keeps the default and does not crash. The file is the durable layer; runtime changes land
here automatically.

## Build & run (smoke test)

```bash
source install/setup.bash
# standalone — compiled defaults (no config manager)
ros2 run rkse_hsal_depth_bluerobotics_driver depth_node
ros2 lifecycle set /depth_node configure          # lifecycle: loads + validates here

# via the config-manager launch (library auto-loads override_file + persists into it)
ros2 launch <vessel>_bringup/drivers/depth/depth1/launch.py \
  namespace:=depth1 \
  params_file:=<vessel>_bringup/drivers/depth/depth1/params.yaml \
  override_file:=<user_config>/drivers/depth/depth1/override.yaml \
  vessel_config_dir:=<vessel>_bringup
ros2 lifecycle set /depth1/depth_node configure
ros2 param set /depth1/depth_node fluid_density 1080.0   # live + auto-persisted to override_file
cat <user_config>/drivers/depth/depth1/override.yaml     # -> fluid_density: 1080.0 (durable; relaunch reloads)
```

---

## Gotchas (all verified — fork TEST_RESULTS)

- **Namespace = YAML root key**, header = CMake target. `depth_node:` → `depth_node::ParamListener`,
  include `"<pkg>/depth_parameters.hpp"`.
- **Prefixed listeners** (`ParamListener(itf, logger, "prefix")`, ros2_control style): the override
  file stores **relative** (unprefixed) keys; the loader re-applies the prefix.
- **Nested groups**: override file accepts nested (`filtering: {enabled: false}`) or flat dotted
  (`filtering.enabled: false`).
- **A bad override file never crashes the node** — unknown / invalid / read-only / malformed keys are
  skipped per-key; valid keys still apply; defaults kept otherwise.
- **`ros2 param set` CLI quirks** (not the library): negative values need a leading space (`' -1'`);
  `#` in a string is read as a YAML comment.

## Checklist

- [ ] `#include "<pkg>/<cmake_target>.hpp"`
- [ ] `<node_name>::ParamListener` built (ctor for plain node, `on_configure` for lifecycle)
- [ ] `params_ = listener->get_params();` then use the typed struct
- [ ] **no** `set_override_file` / `CONFIG_OVERRIDE_FILE` in node code — the generated library auto-wires it from the launch env
- [ ] live changes handled via `is_old()` poll or `setUserCallback`
- [ ] smoke-tested: standalone defaults + override auto-loaded + `ros2 param set` persists + reject per flag
