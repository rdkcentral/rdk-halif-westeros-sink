# Westeros SoC HAL Specification

RDK SOUTHBOUND COMPONENT SPECIFICATION 

*Southbound contract for the platform-specific Westeros video sink layer*

| Document field | Value |
|---|---|
| Pattern | Aligned to the `rdk-halif-libdrm` specification structure |
| Source branch assessed | `westeros-sink` develop branch and public repository landing page |
| Prepared for | RDK Core / SoC vendor architecture review |
| Document version | 1.0 |
| Date | 16 October 2026 |

> **Scope note:** This document describes the current in-tree Westeros SoC adaptation contract and a target standalone HAL boundary. 

[Source repository: westeros-sink](https://github.com/rdkcentral/westeros-sink) 

## Table of Contents

1. Acronyms, Terms and Abbreviations
2. Description
3. Introduction
4. Interface API Documentation


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

The Westeros SoC HAL is the platform adaptation boundary used by the common Westeros GStreamer sink to initialize platform resources, negotiate capabilities, process lifecycle transitions, handle buffers and events, coordinate video positioning, and release resources. In the current repository, the common sink includes a platform-selected `westeros-sink-soc.h` and calls `gst_westeros_sink_soc_*` entry points implemented in platform directories.

> **Architectural intent:** Separate common sink behavior from silicon- and platform-specific implementation so each SoC module can evolve and compile independently behind a stable interface.

| Layer | Responsibility | Representative artifact |
|---|---|---|
| Caller / pipeline | Supplies caps, state changes, buffers, queries, and events | GStreamer pipeline |
| Common Westeros sink | Owns common element behavior, properties, Wayland surface integration, and dispatch to SoC hooks | `westeros-sink.c` / `westeros-sink.h` |
| SoC HAL / adaptation | Maps common requests to decoder, buffer, synchronization, and platform services | `<platform>/westeros-sink-soc.c/.h` |
| Kernel / vendor stack | Provides V4L2, DRM, decoder, secure-video, and display primitives | Vendor SDK and Linux drivers |

## 3. Introduction

Westeros is described by the repository as a lightweight Wayland compositor library supporting normal, nested, and embedded compositors. The `westeros-sink` repository contains the GStreamer sink implementation and multiple platform-specific SoC variants. 

The specification serves the following purposes:

- Document the observable contract between common sink code and the SoC implementation.
- Provide a baseline for extracting the SoC module into a standalone library without changing common sink behavior.
- Give SoC vendors a consistent checklist for lifecycle, buffers, events, error handling, threading, and validation.


## 4. Interface API Documentation

**Current-state warning:** The interfaces below document the observable V4L2-oriented SoC contract. The source repository does not present them as a separately versioned public HAL ABI. The API names, signatures, and basic purposes are grounded in the current public header and RDK porting documentation; detailed ownership and postconditions below define the normative expectations for an implementation of this specification.

#### 4.0 Common parameter, lifetime, and error rules

- `GstWesterosSink *` and `GstWesterosSinkClass *` are current integration types owned by the common sink. They expose an embedded dependency that is accepted for the current contract but shall be removed in the target standalone HAL.
- Pointer parameters are required to be non-NULL unless a function description explicitly permits otherwise.
- Input objects are borrowed for the duration of a call unless the SoC implementation explicitly acquires its own reference. Any acquired reference shall be released by flush or termination.
- `gboolean` functions return `TRUE` for successful completion and `FALSE` when the requested operation cannot be completed. State-transition functions additionally set `passToDefault` to state whether common/default handling shall continue.
- `void` functions have no direct status channel. Implementations shall use GStreamer error reporting and diagnostics for failures and shall leave the instance in a safe, recoverable state.
- Optional hooks shall be identified by the selected platform implementation and shall not silently change mandatory lifecycle semantics.

#### 4.1 `gst_westeros_sink_soc_class_init`

**Prototype**
```c
void gst_westeros_sink_soc_class_init(GstWesterosSinkClass *klass);
```

- **Purpose:** Called during GObject class initialization to register SoC-specific properties, signals, caps, and class behavior.
- **Parameters:** `klass`: common sink class to extend; non-NULL and valid for class initialization.
- **Return values / postconditions:** No return value. Registration failures must be reported through initialization diagnostics.

#### 4.2 `gst_westeros_sink_soc_init`

**Prototype**
```c
gboolean gst_westeros_sink_soc_init(GstWesterosSink *sink);
```

- **Purpose:** Creates or initializes per-instance SoC state before platform-dependent playback activity.
- **Parameters:** `sink`: sink instance whose platform state is initialized; non-NULL.
- **Return values / postconditions:** `TRUE` when initialization succeeds; `FALSE` when required SoC state cannot be initialized.

#### 4.3 `gst_westeros_sink_soc_term`

**Prototype**
```c
void gst_westeros_sink_soc_term(GstWesterosSink *sink);
```

- **Purpose:** Stops SoC activity and releases resources owned by the implementation. Must tolerate partial initialization.
- **Parameters:** `sink`: initialized sink instance to terminate.
- **Return values / postconditions:** No return value. After return, SoC-owned callbacks and worker activity must no longer reference the instance.

#### 4.4 `gst_westeros_sink_soc_set_property`

**Prototype**
```c
void gst_westeros_sink_soc_set_property(GObject *object, guint prop_id, const GValue *value, GParamSpec *pspec);
```

- **Purpose:** Processes writes to properties owned or extended by the SoC implementation.
- **Parameters:** `object`: sink object; `prop_id`: registered property identifier; `value`: new value; `pspec`: property metadata.
- **Return values / postconditions:** No return value. Unsupported IDs or invalid values are reported using normal GObject/GStreamer diagnostics.

#### 4.5 `gst_westeros_sink_soc_get_property`

**Prototype**
```c
void gst_westeros_sink_soc_get_property(GObject *object, guint prop_id, GValue *value, GParamSpec *pspec);
```

- **Purpose:** Returns the current value of a SoC-specific property.
- **Parameters:** `object`: sink object; `prop_id`: requested property identifier; `value`: output container; `pspec`: property metadata.
- **Return values / postconditions:** No return value. On a supported property, `value` is populated with a value compatible with `pspec`.

#### 4.6 `gst_westeros_sink_soc_null_to_ready`

**Prototype**
```c
gboolean gst_westeros_sink_soc_null_to_ready(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Prepares platform services and device context for NULL-to-READY.
- **Parameters:** `sink`: transitioning instance; `passToDefault`: output flag selecting whether common/default transition handling also runs.
- **Return values / postconditions:** `TRUE` on success; `FALSE` if the transition must fail. `*passToDefault` is set to `TRUE` to continue default handling or `FALSE` when fully handled by the SoC layer.

#### 4.7 `gst_westeros_sink_soc_ready_to_paused`

**Prototype**
```c
gboolean gst_westeros_sink_soc_ready_to_paused(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Configures negotiated stream, decoder, buffers, and preroll resources for READY-to-PAUSED.
- **Parameters:** Same ownership and output semantics as `null_to_ready`.
- **Return values / postconditions:** `TRUE` on success; `FALSE` on failure; `*passToDefault` controls default transition processing.

#### 4.8 `gst_westeros_sink_soc_paused_to_playing`

**Prototype**
```c
gboolean gst_westeros_sink_soc_paused_to_playing(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Starts or resumes the platform video path for PAUSED-to-PLAYING.
- **Parameters:** Same ownership and output semantics as `null_to_ready`.
- **Return values / postconditions:** `TRUE` on success; `FALSE` on failure; `*passToDefault` controls default transition processing.

#### 4.9 `gst_westeros_sink_soc_playing_to_paused`

**Prototype**
```c
gboolean gst_westeros_sink_soc_playing_to_paused(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Pauses presentation and quiesces platform activity required by PLAYING-to-PAUSED.
- **Parameters:** Same ownership and output semantics as `null_to_ready`.
- **Return values / postconditions:** `TRUE` on success; `FALSE` on failure; `*passToDefault` controls default transition processing.

#### 4.10 `gst_westeros_sink_soc_paused_to_ready`

**Prototype**
```c
gboolean gst_westeros_sink_soc_paused_to_ready(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Stops stream-scoped activity and releases resources no longer required in READY.
- **Parameters:** Same ownership and output semantics as `null_to_ready`.
- **Return values / postconditions:** `TRUE` on success; `FALSE` on failure; `*passToDefault` controls default transition processing.

#### 4.11 `gst_westeros_sink_soc_ready_to_null`

**Prototype**
```c
gboolean gst_westeros_sink_soc_ready_to_null(GstWesterosSink *sink, gboolean *passToDefault);
```

- **Purpose:** Releases platform/device resources for READY-to-NULL.
- **Parameters:** Same ownership and output semantics as `null_to_ready`.
- **Return values / postconditions:** `TRUE` on success; `FALSE` on failure; `*passToDefault` controls default transition processing.

#### 4.12 `gst_westeros_sink_soc_registryHandleGlobal`

**Prototype**
```c
void gst_westeros_sink_soc_registryHandleGlobal(GstWesterosSink *sink, struct wl_registry *registry, uint32_t id, const char *interface, uint32_t version);
```

- **Purpose:** Handles discovery of a Wayland global required by the SoC video/display path.
- **Parameters:** `registry`: borrowed Wayland registry; `id`: announced global name; `interface`: borrowed interface name valid for the callback; `version`: announced version.
- **Return values / postconditions:** No return value. The implementation binds only supported interfaces/versions and retains no borrowed string pointer after return.

#### 4.13 `gst_westeros_sink_soc_registryHandleGlobalRemove`

**Prototype**
```c
void gst_westeros_sink_soc_registryHandleGlobalRemove(GstWesterosSink *sink, struct wl_registry *registry, uint32_t name);
```

- **Purpose:** Invalidates and releases SoC state associated with a removed Wayland global.
- **Parameters:** `name`: identifier of the removed Wayland global; other pointers are borrowed for the callback.
- **Return values / postconditions:** No return value.

#### 4.14 `gst_westeros_sink_soc_accept_caps`

**Prototype**
```c
gboolean gst_westeros_sink_soc_accept_caps(GstWesterosSink *sink, GstCaps *caps);
```

- **Purpose:** Determines whether the selected SoC implementation can support the proposed media caps.
- **Parameters:** `caps`: borrowed caps to evaluate; the function shall not modify or unref it.
- **Return values / postconditions:** `TRUE` if supported; `FALSE` if unsupported, malformed, or inconsistent with available capability.

#### 4.15 `gst_westeros_sink_soc_set_startPTS`

**Prototype**
```c
void gst_westeros_sink_soc_set_startPTS(GstWesterosSink *sink, gint64 pts);
```

- **Purpose:** Supplies the initial presentation timestamp used to align stream timing.
- **Parameters:** `pts`: stream start presentation timestamp in the units expected by the current implementation.
- **Return values / postconditions:** No return value.

#### 4.16 `gst_westeros_sink_soc_render`

**Prototype**
```c
void gst_westeros_sink_soc_render(GstWesterosSink *sink, GstBuffer *buffer);
```

- **Purpose:** Submits encoded or raw media data to the selected platform path.
- **Parameters:** `buffer`: borrowed input buffer for the duration of the call unless the implementation takes its own reference.
- **Return values / postconditions:** No return value. Failures are surfaced through GStreamer error/message state because the current signature cannot return a flow status.

#### 4.17 `gst_westeros_sink_soc_flush`

**Prototype**
```c
void gst_westeros_sink_soc_flush(GstWesterosSink *sink);
```

- **Purpose:** Discards queued data, releases retained frames, resets stream-scoped state, and unblocks pending operations.
- **Parameters:** `sink`: active or partially active sink instance.
- **Return values / postconditions:** No return value. On completion, stale pre-flush frames shall not be presented.

#### 4.18 `gst_westeros_sink_soc_start_video`

**Prototype**
```c
gboolean gst_westeros_sink_soc_start_video(GstWesterosSink *sink);
```

- **Purpose:** Starts the platform video output after required negotiation and resource setup.
- **Parameters:** `sink`: configured sink instance.
- **Return values / postconditions:** `TRUE` when video output starts successfully; `FALSE` when required decoder/display activity cannot be started.

#### 4.19 `gst_westeros_sink_soc_eos_event`

**Prototype**
```c
void gst_westeros_sink_soc_eos_event(GstWesterosSink *sink);
```

- **Purpose:** Notifies the SoC implementation that no additional stream buffers are expected and initiates platform EOS handling.
- **Parameters:** `sink`: active sink receiving end-of-stream.
- **Return values / postconditions:** No return value.

#### 4.20 `gst_westeros_sink_soc_set_video_path`

**Prototype**
```c
void gst_westeros_sink_soc_set_video_path(GstWesterosSink *sink, bool useGfxPath);
```

- **Purpose:** Selects the rendering path required by negotiated media and composition mode.
- **Parameters:** `useGfxPath`: `true` selects the graphics-oriented path; `false` selects the platform video path.
- **Return values / postconditions:** No return value.

#### 4.21 `gst_westeros_sink_soc_update_video_position`

**Prototype**
```c
void gst_westeros_sink_soc_update_video_position(GstWesterosSink *sink);
```

- **Purpose:** Applies current video position, size, visibility, and related geometry to the platform path.
- **Parameters:** `sink`: instance containing the effective source and destination geometry.
- **Return values / postconditions:** No return value.

#### 4.22 `gst_westeros_sink_soc_query`

**Prototype**
```c
gboolean gst_westeros_sink_soc_query(GstWesterosSink *sink, GstQuery *query);
```

- **Purpose:** Handles queries that require SoC-specific timing, position, capability, or allocation information.
- **Parameters:** `query`: mutable borrowed query to inspect and, when handled, populate.
- **Return values / postconditions:** `TRUE` when the query was handled successfully; `FALSE` when unsupported or not handled.

#### 4.23 `gst_westeros_sink_soc_set_compositor_lock`

**Prototype**
```c
void gst_westeros_sink_soc_set_compositor_lock(GstWesterosSink *sink, bool lock);
```

- **Purpose:** Applies the platform-specific compositor lock state where this optional hook is implemented.
- **Parameters:** `lock`: requested compositor lock state.
- **Return values / postconditions:** No return value.

#### 4.24 `gst_westeros_sink_soc_set_speed`

**Prototype**
```c
void gst_westeros_sink_soc_set_speed(GstWesterosSink *sink, const GValue *value);
```

- **Purpose:** Applies a platform-specific playback rate where this optional hook is implemented.
- **Parameters:** `value`: borrowed playback-speed value with the type expected by the property/segment contract.
- **Return values / postconditions:** No return value.

#### 4.25 Target standalone interface rule

The current API passes `GstWesterosSink` and `GstWesterosSinkClass` directly. In the next refactoring stage, the public boundary shall replace these embedded common-sink dependencies with an opaque SoC context, explicit input/output parameter structures, interface version negotiation, stable status values, and capability discovery. Vendor SDK types and private sink structure members shall not appear in the public header.

