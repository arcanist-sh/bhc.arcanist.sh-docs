+++
title = "Runtime Profiles"
description = "bhc runtime profiles for different use cases"
sort_by = "weight"
+++

A profile is chosen per build with `--profile`, and it selects which optimisation passes and runtime settings bhc uses. The same source compiles under every profile. Four profiles are available on the command line:

- **Default** - General-purpose, lazy evaluation, the profile everything is tested against
- **Server** - Reserved for structured concurrency and observability; today it compiles like the default profile
- **Numeric** - Enables the numeric list-fusion rewrites, the Tensor IR pipeline and `--kernel-report`
- **Edge** - Smaller memory limits for WebAssembly modules

Each page below describes what the profile does in compiled programs today and which parts of its design are still runtime-only. Two further profiles, realtime and embedded, exist in the runtime crates but cannot be selected from the command line.

Profiles are selected only with the `--profile` flag. Per-module selection (an `OPTIONS_BHC` pragma) and per-package selection are part of the design but are not read by the compiler yet.
