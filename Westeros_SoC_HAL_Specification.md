# Westeros SoC HAL Specification

**RDK SOUTHBOUND COMPONENT SPECIFICATION 

*Southbound contract for the platform-specific Westeros video sink layer*

| Document field | Value |
|---|---|
| Pattern | Aligned to the `rdk-halif-libdrm` specification structure |
| Source branch assessed | `westeros-sink` develop branch and public repository landing page |
| Prepared for | RDK Core / SoC vendor architecture review |
| Date | 31 August 2026 |

> **Scope note:** This document describes the current in-tree Westeros SoC adaptation contract and a target standalone HAL boundary. It does not claim that a separately versioned public HAL interface is already available.

[Source repository: westeros-sink](https://github.com/rdkcentral/westeros-sink) | [Pattern repository: rdk-halif-libdrm](https://github.com/rdkcentral/rdk-halif-libdrm)

## Table of Contents

1. Acronyms, Terms and Abbreviations
2. Description
3. Introduction
4. References
5. Component Runtime Execution Requirements
6. Non-functional Requirements
7. Licensing and Build Requirements
8. Variability Management
9. Interface API Documentation
10. Theory of Operation and Key Concepts
11. General Westeros SoC Code Flow
12. SoC Implementation Requirements
13. Compliance and Validation Checklist


## 1. Acronyms, Terms and Abbreviations

| Term | Meaning |
|---|---|
| AIDL | Android Interface Definition Language |
| API | Application Programming Interface |
| AV | Audio / Video |
| DMA-BUF | Linux buffer-sharing mechanism using file descriptors |
| DRM | Direct Rendering Manager |
| EOS | End of Stream |
| GStreamer | Multimedia framework hosting the `GstBaseSink`-derived element |
| HAL | Hardware Abstraction Layer |
| SoC | System on Chip |
| V4L2 | Video4Linux2 |
| Wayland | Display-server protocol used by Westeros |
| Westeros sink | GStreamer sink component that connects pipeline video to the Westeros display environment |

## 2. Description

The Westeros SoC HAL is the platform adaptation boundary used by the common Westeros GStreamer sink to initialize platform resources, negotiate capabilities, process lifecycle transitions, handle buffers and events, coordinate video positioning, and release resources. In the current repository, the common sink includes a platform-selected `westeros-sink-soc.h` and calls `gst_westeros_sink_soc_*` entry points implemented in platform directories such as `v4l2`, `drm`, `brcm`, `rpi`, `emu`, `raw`, and `icegdl`.

> **Architectural intent:** Separate common sink behavior from silicon- and platform-specific implementation so each SoC module can evolve and compile independently behind a stable interface.

| Layer | Responsibility | Representative artifact |
|---|---|---|
| Caller / pipeline | Supplies caps, state changes, buffers, queries, and events | GStreamer pipeline |
| Common Westeros sink | Owns common element behavior, properties, Wayland surface integration, and dispatch to SoC hooks | `westeros-sink.c` / `westeros-sink.h` |
| SoC HAL / adaptation | Maps common requests to decoder, buffer, synchronization, and platform services | `<platform>/westeros-sink-soc.c/.h` |
| Kernel / vendor stack | Provides V4L2, DRM, decoder, secure-video, and display primitives | Vendor SDK and Linux drivers |

## 3. Introduction

Westeros is described by the repository as a lightweight Wayland compositor library supporting normal, nested, and embedded compositors. The `westeros-sink` repository contains the GStreamer sink implementation and multiple platform-specific SoC variants. The current code exposes a compile-time platform adaptation rather than a language-neutral or binder-based HAL contract.

The specification serves the following purposes:

- Document the observable contract between common sink code and the SoC implementation.
- Provide a baseline for extracting the SoC module into a standalone library without changing common sink behavior.
- Give SoC vendors a consistent checklist for lifecycle, buffers, events, error handling, threading, and validation.

## 4. References

- [Westeros sink repository](https://github.com/rdkcentral/westeros-sink)
- [Common sink header](https://github.com/rdkcentral/westeros-sink/blob/develop/westeros-sink.h)
- [Common sink implementation](https://github.com/rdkcentral/westeros-sink/blob/develop/westeros-sink.c)
- [V4L2 SoC header](https://github.com/rdkcentral/westeros-sink/blob/develop/v4l2/westeros-sink-soc.h)
- [V4L2 SoC implementation](https://github.com/rdkcentral/westeros-sink/blob/develop/v4l2/westeros-sink-soc.c)
- [LibDRM HAL documentation pattern](https://github.com/rdkcentral/rdk-halif-libdrm)

## 5. Component Runtime Execution Requirements

### 5.1 Initialization and Startup

- The common sink class initialization shall invoke the SoC class initialization hook so platform properties, signals, capabilities, and class behavior can be registered.
- Per-instance initialization shall establish the SoC state required before the element enters READY.
- SoC resources shall be acquired only when required by the state transition or negotiated media path.
- Initialization failures shall be returned synchronously through the declared return value and translated by the caller into appropriate GStreamer state or flow results.

### 5.2 State Transition Contract

| Transition hook | Purpose | Pass-through control |
|---|---|---|
| `gst_westeros_sink_soc_null_to_ready` | Open or prepare platform services and device context | May control whether default parent transition runs |
| `gst_westeros_sink_soc_ready_to_paused` | Configure negotiated media path, buffers, decoder, or preroll resources | May control default transition |
| `gst_westeros_sink_soc_paused_to_playing` | Start or resume platform video processing | May control default transition |
| `gst_westeros_sink_soc_playing_to_paused` | Pause platform video processing | May control default transition |
| `gst_westeros_sink_soc_paused_to_ready` | Stop media path and release stream-scoped resources | May control default transition |
| `gst_westeros_sink_soc_ready_to_null` | Release device and platform resources | May control default transition |

### 5.3 Threading Model

The SoC implementation shall be safe under the calling and callback behavior of the GStreamer element. Shared mutable state shall be protected consistently. The V4L2 implementation declares a dedicated SoC mutex and may operate video-output, EOS-detection, dispatch, first-frame, and underflow threads depending on build options and mode. Thread creation and termination shall be paired, and termination shall not leave callbacks referencing released sink state.

### 5.4 Process Model

The interface is designed for use within the process hosting the GStreamer pipeline. The current contract is a C source/header integration. If extracted into `libwesterossink_soc.so`, ABI ownership, symbol visibility, and supported concurrent instances must be explicitly versioned by the standalone HAL package.

### 5.5 Memory and Buffer Ownership

| Object / resource | Expected ownership rule |
|---|---|
| `GstBuffer` passed to render | Borrowed for the duration of the call unless explicitly referenced by the implementation |
| DMA-BUF / DRM file descriptors | Ownership transfer or duplication must be documented for every path |
| V4L2 buffers and mapped planes | Allocated, queued, dequeued, unmapped, and closed by the SoC implementation that created them |
| Codec data | Copied or referenced with clearly paired cleanup |
| Last/preroll buffers | If retained, hold a GStreamer reference and release during flush/teardown |
| Wayland/SoC protocol objects | Destroy before the associated display/registry lifetime ends |

### 5.6 Power Management Requirements

No independent power-management API is exposed by the current SoC header. The implementation shall release or quiesce decoder, buffer, and display resources on downward state transitions and termination so platform power policy can operate correctly.

### 5.7 Asynchronous Notification Model

- First-frame, underflow, decode-error, time-code, and related notifications may be surfaced as GStreamer signals or messages where supported by the selected SoC implementation.
- Wayland registry add/remove notifications are delegated through the SoC registry hooks.
- Asynchronous callbacks shall validate instance lifetime and shall not call into released resources.

### 5.8 Blocking Calls

The current public code does not define a formal maximum blocking duration per interface call. Implementations should avoid unbounded blocking in render, query, property, and state-change paths. Operations that wait for decoder or server events should use bounded waits and terminate promptly during flush or shutdown. This is a proposed requirement for the extracted HAL contract.

### 5.9 Internal Error Handling

- Return initialization, negotiation, start, query, and state-transition failures through the declared `gboolean` or state/flow result path.
- Convert platform `errno` or vendor status to stable, diagnosable GStreamer error messages.
- Do not leak buffers, mappings, threads, sockets, file descriptors, or protocol objects after partial initialization failure.
- Treat malformed caps, unsupported formats, and invalid property values as controlled errors rather than undefined behavior.

### 5.10 Persistence Model

The SoC adaptation has no documented requirement to persist settings. Runtime properties and negotiated media state are instance-scoped unless a vendor platform explicitly documents otherwise.

## 6. Non-functional Requirements

### 6.1 Logging and Debugging

- Use the Westeros sink GStreamer debug category for component diagnostics where applicable.
- Provide ERROR-level diagnostics for failed device, buffer, synchronization, protocol, and decoder operations.
- Avoid frame-by-frame logging by default. Gate verbose frame and pipeline diagnostics behind explicit runtime or build controls.
- Logs shall identify the failing operation and stable context without exposing protected media content.

### 6.2 Memory and Performance

- Support zero-copy buffer paths when the platform and negotiated mode allow DMA-BUF sharing.
- Avoid per-frame allocation in steady state where buffers can be pooled.
- Bound internal queues and release retained frames on flush, EOS, state change, and termination.
- Maintain frame scheduling and video positioning without excessive CPU wakeups.

### 6.3 Quality Control

- Build with warnings treated as errors for owned code.
- Run static analysis and resolve high-confidence defects before release.
- Run memory and descriptor leak analysis across repeated play, pause, seek, flush, EOS, and teardown cycles.
- Provide unit tests for failure paths and vendor-layer tests for required lifecycle and buffer scenarios.
- Validate secure and non-secure paths separately when secure video is supported.

## 7. Licensing and Build Requirements

The `westeros-sink` repository states LGPL-2.1 licensing. A separated SoC HAL library shall retain license and notice obligations applicable to the extracted source and any linked dependencies. Vendor-specific dependencies shall be declared explicitly.

| Build item | Requirement |
|---|---|
| Common library | Build `libgstwesterossink.so` from common sink code and interface dispatch |
| SoC library target | Target design builds `libwesterossink_soc.so` from platform-specific source |
| Dependency direction | Common sink depends on the SoC interface library, not on private vendor headers |
| Headers | Expose only stable interface declarations; keep SoC structure private or opaque |
| Build independence | Allow native/standalone compilation and testing outside a complete Yocto image where dependencies are available |
| Feature flags | Document all flags that alter ABI, caps, signals, threading, or buffer behavior |

## 8. Variability Management

The repository contains multiple platform directories. Platform selection currently changes data structures, capabilities, properties, and implementation behavior at compile time. A stable HAL extraction should separate mandatory core behavior from optional capability groups.

| Capability group | Examples | Contract treatment |
|---|---|---|
| Mandatory lifecycle | init, term, state transitions, caps, render, flush, query | Stable required API |
| Display integration | registry hooks, video positioning, graphics/video path selection | Required where Wayland video-plane integration is used |
| Decode path | V4L2, vendor decoder, tunnelled decode | Capability-declared implementation choice |
| Buffer sharing | MMAP, DMA-BUF, DRM/GEM | Capability flags plus explicit ownership rules |
| Synchronization | generic AV sync or vendor sync | Optional extension with defined fallback |
| Protected media | secure-video path and protected-buffer handling | Optional, separately validated capability |
| Raw video | `video/x-raw` and graphics path | Optional caps-driven capability |

## 9. Interface API Documentation

> **Current-state warning:** These functions are extracted from the current V4L2 SoC header and are not yet presented by the source repository as a separately versioned public HAL ABI.

| Function | Return | Responsibility |
|---|---|---|
| `gst_westeros_sink_soc_class_init` | `void` | Register SoC-specific class behavior, properties, signals, and capabilities |
| `gst_westeros_sink_soc_init` | `gboolean` | Initialize per-instance SoC state |
| `gst_westeros_sink_soc_term` | `void` | Terminate instance and release SoC resources |
| `gst_westeros_sink_soc_set_property` | `void` | Handle SoC-specific property writes |
| `gst_westeros_sink_soc_get_property` | `void` | Handle SoC-specific property reads |
| `gst_westeros_sink_soc_null_to_ready` | `gboolean` | Handle NULL to READY transition |
| `gst_westeros_sink_soc_ready_to_paused` | `gboolean` | Handle READY to PAUSED transition |
| `gst_westeros_sink_soc_paused_to_playing` | `gboolean` | Handle PAUSED to PLAYING transition |
| `gst_westeros_sink_soc_playing_to_paused` | `gboolean` | Handle PLAYING to PAUSED transition |
| `gst_westeros_sink_soc_paused_to_ready` | `gboolean` | Handle PAUSED to READY transition |
| `gst_westeros_sink_soc_ready_to_null` | `gboolean` | Handle READY to NULL transition |
| `gst_westeros_sink_soc_registryHandleGlobal` | `void` | Handle Wayland registry global discovery |
| `gst_westeros_sink_soc_registryHandleGlobalRemove` | `void` | Handle Wayland registry global removal |
| `gst_westeros_sink_soc_accept_caps` | `gboolean` | Validate caps support |
| `gst_westeros_sink_soc_set_startPTS` | `void` | Provide stream start PTS |
| `gst_westeros_sink_soc_render` | `void` | Submit or process a media buffer |
| `gst_westeros_sink_soc_flush` | `void` | Flush queued/decode/render state |
| `gst_westeros_sink_soc_start_video` | `gboolean` | Start the platform video path |
| `gst_westeros_sink_soc_eos_event` | `void` | Handle EOS |
| `gst_westeros_sink_soc_set_video_path` | `void` | Select video or graphics-oriented path |
| `gst_westeros_sink_soc_update_video_position` | `void` | Apply current video geometry/position |
| `gst_westeros_sink_soc_query` | `gboolean` | Handle SoC-specific GStreamer query |

### 9.1 Interface Design Rules for Standalone HAL

- Replace direct exposure of `struct _GstWesterosSinkSoc` with an opaque context handle.
- Do not expose vendor SDK types in the public interface header.
- Define ABI version negotiation and a size/version field for extensible function tables or context parameters.
- Return a stable Westeros SoC status enum and preserve platform detail through a diagnostic accessor.
- Declare mandatory and optional entry points explicitly.
- Define thread-safety, callback context, lifetime, and ownership for every parameter.

## 10. Theory of Operation and Key Concepts

### 10.1 Common-to-SoC Dispatch

The common sink derives from `GstBaseSink`, owns common element and Wayland state, includes the selected SoC header, embeds SoC state, and delegates platform-sensitive operations to `gst_westeros_sink_soc_*` functions. The SoC layer supplies media caps and platform-specific properties, handles device and decoder setup, manages buffers, and coordinates output with the compositor or video server.

### 10.2 Encoded and Raw Paths

The V4L2 header declares encoded H.264/MPEG caps and raw NV12, NV21, I420, and YU12 caps. It defines both V4L2 buffer information and DRM/GEM-style buffer information. The implementation therefore supports distinct encoded and raw/graphics flows selected by caps, sink mode, compile-time configuration, and platform capabilities.

### 10.3 Buffer Flow

- Encoded data is accepted after caps negotiation, queued to the decoder input path, and matched with platform-managed output buffers.
- Output may use mapped memory or DMA-BUF depending on platform support and configuration.
- Raw/graphics frames may be represented through DRM-related handles/file descriptors and delivered to a compositor/video-server path.
- Buffers are retained only when required for preroll, keep-last-frame, frame stepping, or asynchronous output, with explicit release during flush/teardown.

### 10.4 Video Geometry and Visibility

The common sink tracks window coordinates, size, opacity, z-order, visibility, transforms, output size, and source dimensions. The SoC update-video-position hook maps the effective geometry to the platform video path. Common Wayland shell/VPC behavior and SoC video-plane behavior must remain synchronized.

### 10.5 Resource Management

The common header includes Essos resource-manager state and acquire/release hooks. The target architecture should keep resource policy in the common or centralized RDK resource manager and limit the SoC HAL to resource execution and capability reporting. The internal refactoring material also identifies movement toward a unified RDK resource manager and standalone SoC module.

## 11. General Westeros SoC Code Flow

1. **Pipeline selects Westeros sink:** Application or player constructs a GStreamer pipeline and negotiates video caps.
2. **Common class and instance initialization:** `westeros-sink` registers common behavior and invokes SoC class/instance initialization.
3. **NULL to READY:** SoC implementation opens or prepares device, protocol, synchronization, and platform context.
4. **READY to PAUSED and caps acceptance:** Validate format, choose encoded/raw path, configure decoder and buffer model, and prepare preroll.
5. **PAUSED to PLAYING:** Start or resume decode/output and asynchronous event handling.
6. **Render loop:** Common sink receives `GstBuffer`; SoC layer queues, imports, maps, or submits it and updates counters/timestamps.
7. **Display coordination:** Apply video rectangle/visibility and coordinate buffers with Wayland/video server or platform composition.
8. **Events and queries:** Handle position, EOS, flush, first frame, underflow, decode error, registry, and platform queries.
9. **Teardown:** Stop threads, flush buffers, release decoder/display resources, close descriptors, and return to NULL.

## 12. SoC Implementation Requirements

| ID | Requirement | Level |
|---|---|---|
| SOC-WST-001 | Provide all mandatory lifecycle entry points | Required |
| SOC-WST-002 | Declare supported caps and reject unsupported caps deterministically | Required |
| SOC-WST-003 | Support clean repeated initialization and termination | Required |
| SOC-WST-004 | Pair all allocations, mappings, references, descriptors, sockets, and threads with cleanup | Required |
| SOC-WST-005 | Define buffer ownership at each API boundary | Required |
| SOC-WST-006 | Implement flush so blocked operations terminate and stale frames are not displayed | Required |
| SOC-WST-007 | Handle EOS without leaking or deadlocking resources | Required |
| SOC-WST-008 | Apply video geometry changes consistently with visibility and path selection | Required |
| SOC-WST-009 | Report platform failures through stable caller-visible diagnostics | Required |
| SOC-WST-010 | Protect shared state used by callbacks and worker threads | Required |
| SOC-WST-011 | Expose protected-content capability only when end-to-end secure handling is implemented | Conditional |
| SOC-WST-012 | Support DMA-BUF import/export only with documented fd ownership and synchronization | Conditional |
| SOC-WST-013 | Do not publish vendor SDK types in the standalone public header | Target architecture |
| SOC-WST-014 | Provide interface version and capability discovery | Target architecture |
| SOC-WST-015 | Pass vendor-layer lifecycle, negative, leak, and stress tests | Required |

## 13. Compliance and Validation Checklist

| Area | Minimum validation |
|---|---|
| Build | Clean build, warnings as errors, supported configurations enumerated |
| Lifecycle | Repeated NULL to PLAYING to NULL cycles; failure injection at each transition |
| Caps | Supported encoded/raw formats accepted; malformed and unsupported caps rejected |
| Playback | First frame, steady playback, pause/resume, rate and position behavior as supported |
| Events | Flush-start/stop, seek/segment changes, EOS, decoder errors, underflow |
| Buffers | MMAP and/or DMA-BUF paths; ownership; exhaustion; requeue; teardown |
| Geometry | Window move/resize, hide/show, z-order/opacity where supported |
| Concurrency | Rapid state changes, callback races, shutdown during wait |
| Security | Secure and non-secure paths separated; protected buffers never mapped by unauthorized code |
| Robustness | No memory, fd, mapping, socket, thread, or protocol-object leaks |
| Integration | Run through RDK middleware and representative application-level playback scenarios |


