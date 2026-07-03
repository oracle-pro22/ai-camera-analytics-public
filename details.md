# Project Details

This repository is a multi-service video analytics system. It takes camera or video frames, runs person detection, applies CLIP-based role classification, tracks objects over time, evaluates rules, and exposes several visualization and insight UIs.

The main idea is:

1. `frame_service` captures and buffers frames.
2. `inference_engine` polls frames, batches them, runs RT-DETR, then uses CLIP.
3. `state_layer` tracks objects across time and builds demo and insight views.
4. `rule_engine` reads state and raises alerts based on rules.

The current code is designed around one configured camera, `cam1`, but the structure supports multiple cameras.

## Big Picture

The live pipeline is:

`frame_service -> inference_engine -> state_layer -> rule_engine`

The display pipeline is:

`state_layer -> demo UI / viz UI / insight UI`

The inference pipeline is:

`frame_service frames -> RT-DETR -> CLIP -> state_layer`

The rules pipeline is:

`state_layer state -> rule_engine rules -> alerts`

## Folder Overview

### `frame_service`

This layer is responsible for ingesting frames from a source, buffering them, serving them to downstream services, and exposing live stream endpoints.

Main responsibilities:

- open camera or video sources
- capture frames at a controlled rate
- keep a latest-frame copy for live streaming
- keep a rolling frame buffer for polling
- expose frame and video-session APIs
- store inference detections for quick access

### `inference_engine`

This layer is responsible for taking buffered frames, batching them dynamically, running detection and CLIP classification, then posting results back to `state_layer`.

Main responsibilities:

- poll buffered frames from `frame_service`
- select the best frame per camera
- group selected frames into batches
- run RT-DETR detection in batch mode
- classify detected people with CLIP
- attach session metadata and post results

### `state_layer`

This layer is the temporal brain of the system. It tracks detections across time, assigns zones, builds demo payloads, creates insight dashboards, and prepares visualization output.

Main responsibilities:

- track objects across frames
- preserve first-seen and last-seen state
- compute duration, speed, and zone membership
- merge CLIP authorization metadata into tracked state
- build visual debug output
- build the demo UI payload
- build the insight UI payload

### `rule_engine`

This layer watches the current state and converts it into rules and alerts.

Main responsibilities:

- poll the state layer at a fixed interval
- evaluate matching rules
- create active and recent alert lists
- expose rule and alert APIs

## `frame_service` Details

### What It Does

`frame_service` reads a camera or video source using OpenCV, encodes each frame as JPEG, stores the frame in memory, and exposes it through API routes.

It also maintains a video-session store so the system can tell whether a source is queued, processing, completed, or stopped.

### Important Files

- `frame_service/main.py`
- `frame_service/api/server.py`
- `frame_service/core/camera_source.py`
- `frame_service/core/config_loader.py`
- `frame_service/core/ingestion_engine.py`
- `frame_service/core/frame_envelope.py`
- `frame_service/core/frame_buffer.py`
- `frame_service/core/latest_frame_store.py`
- `frame_service/core/video_session_store.py`

### Current Features

- live streaming through `/stream/{camera_id}`
- buffered polling through `/frames/{camera_id}`
- video session metadata through `/videos`
- detection storage through `/detections/*`
- adaptive frame pacing using queue pressure
- adaptive resolution control
- per-camera metrics

### Current Camera Setup

The repo currently uses `cam1` in `frame_service/config/cameras.yaml`.

Depending on the config, the source may be:

- a local video file
- a webcam index
- a network stream URL

### Frame Envelope

Each captured frame is wrapped in a `FrameEnvelope`.

This object carries:

- `frame_uuid`
- `camera_id`
- `frame_id`
- capture timestamp
- ingest timestamp
- resolution
- effective FPS
- queue depth
- health state
- session status
- video name
- source type
- total frames
- progress ratio
- source metadata

This gives downstream services enough context to show progress and source details.

### Frame Service APIs

Current routes include:

- `GET /`
- `GET /health`
- `GET /metrics`
- `GET /cameras`
- `GET /videos`
- `GET /videos/{camera_id}`
- `GET /stream/{camera_id}`
- `GET /frames/{camera_id}`
- `POST /detections/receive`
- `GET /detections/{camera_id}`
- `GET /detections`

## `inference_engine` Details

### What It Does

This service polls recent frames, picks the best candidate frame per camera, groups frames into dynamic batches, runs RT-DETR, then runs CLIP on the detected people.

It does not expose HTTP routes. It is a worker process.

### Important Files

- `inference_engine/main.py`
- `inference_engine/config/model.yaml`
- `inference_engine/core/batch_scheduler.py`
- `inference_engine/core/base_detector.py`
- `inference_engine/core/rtdetr_detector.py`
- `inference_engine/core/clip_person_classifier.py`
- `inference_engine/core/frame_poller.py`
- `inference_engine/core/frame_selector.py`
- `inference_engine/core/frame_store.py`
- `inference_engine/core/frame_decoder.py`
- `inference_engine/core/result_poster.py`
- `inference_engine/core/inference_result.py`

### Batching

The batching system is dynamic, not fixed.

The scheduler:

- keeps at most one pending frame per camera by default
- replaces older pending frames with newer ones
- drains when batch size is reached
- drains when a wait threshold is reached
- prunes stale frames
- protects against queue overflow

This means the system prefers fresh frames and bounded latency instead of blindly processing every single frame.

### Detection Flow

The detection flow is:

1. poll frames from `frame_service`
2. store them in the local `FrameStore`
3. select the best frame per camera
4. enqueue selected frames in `BatchScheduler`
5. drain a ready batch
6. decode JPEG into numpy images
7. run RT-DETR with `detect_batch`
8. run CLIP on each frame's detections
9. build `InferenceResult`
10. post results to `state_layer`

### RT-DETR

RT-DETR is the main object detector.

Current configuration:

- backend: `rtdetr`
- device: `cpu`
- threshold-based filtering
- allowlist-based label filtering
- currently focused on `person`

### CLIP

CLIP is used as a second-stage classifier for detected people.

It is not doing general object detection. It is used to classify detected persons into role-like categories, mainly:

- `doctor`
- `non_doctor`

CLIP outputs fields such as:

- `is_doctor`
- `doctor_score`
- `authorization_status`
- `authorization_reason`
- `excluded_from_alerts`

This metadata is carried downstream so the state layer can suppress or raise alerts correctly.

### Inference Result

Each processed frame produces one `InferenceResult`.

It includes:

- `frame_id`
- `camera_id`
- `timestamp`
- `detections`
- `inference_time_ms`
- session metadata
- source metadata

### Inference Engine APIs

This service does not expose HTTP endpoints.

It is controlled through config and runs as a background worker.

## `state_layer` Details

### What It Does

This is the layer that turns per-frame detections into tracked, time-aware, zone-aware object state.

It also powers the demo UI, the visualization UI, and the insight UI.

### Important Files

- `state_layer/main.py`
- `state_layer/api/server.py`
- `state_layer/core/tracker.py`
- `state_layer/core/state_store.py`
- `state_layer/core/debug_store.py`
- `state_layer/core/visualization_store.py`
- `state_layer/core/visualizer.py`
- `state_layer/core/demo_service.py`
- `state_layer/core/dashboard_service.py`
- `state_layer/core/frame_service_client.py`
- `state_layer/core/insight_store.py`
- `state_layer/core/pipeline_metrics.py`
- `state_layer/core/gemini_client.py`

### Tracking

`tracker.py` matches new detections to existing tracks.

It computes:

- `first_seen`
- `last_seen`
- duration
- average confidence
- movement speed
- idle time
- zone membership
- intrusion candidates

It uses matching logic based on:

- IoU threshold
- center-distance threshold
- duplicate filtering
- max track age

### CLIP Metadata in State

The tracker preserves CLIP-derived metadata so the system can make policy decisions without losing the original classification.

Useful fields include:

- `is_doctor`
- `doctor_score`
- `authorization_status`
- `authorization_reason`
- `excluded_from_alerts`

This means the tracking logic still behaves normally, but each object carries extra semantic context.

### Zones and Intrusions

The state layer knows which zones are restricted and which labels are allowed.

It can detect:

- person in restricted zone
- doctor in zone
- unauthorized object in zone
- intrusion duration threshold exceeded

This is the bridge between visual detection and policy enforcement.

### Demo UI

`demo_service.py` generates the current `/demo` pages.

It shows:

- tracked objects
- current frame preview
- overlay boxes
- zone information
- active alerts
- CLIP doctor/non-doctor summaries

The demo UI is used to visually confirm that inference and tracking are aligned.

### Visualization UI

`visualizer.py` generates SVG and debug visualizations.

It is useful for:

- seeing bounding boxes on a blank canvas
- checking zone overlap
- checking track updates
- inspecting IoU matching

### Insight UI

The insight stack adds a higher-level dashboard for video sessions.

It includes:

- `/home`
- `/ui/videos`
- `/ui/videos/{camera_id}`
- `/ui/{camera_id}/frame`
- `/debug/pipeline-metrics`
- `/debug/pipeline-metrics/{camera_id}`

This is the screenshot-style UI intended for a more product-like experience.

### Insight Store

`insight_store.py` accumulates session-level information across the video.

It builds:

- playlist cards
- AI insight chips
- coordinate rows
- metadata blocks
- GenAI placeholder content

It can infer a rough profile from the video name and metadata.

### Pipeline Metrics

`pipeline_metrics.py` tracks the timing of the pipeline across frames and insights.

It helps answer:

- when the first frame result arrived
- when the first meaningful AI result arrived
- when GenAI completed
- how long the video processing took
- how fast the pipeline ran relative to source video length

### State Layer APIs

Current routes include:

- `GET /`
- `GET /health`
- `POST /state/receive`
- `GET /state/{camera_id}`
- `GET /state`
- `GET /viz`
- `GET /viz/{camera_id}`
- `GET /viz/{camera_id}/json`
- `GET /viz/{camera_id}/history`
- `GET /viz/{camera_id}/iou-matrix`
- `GET /demo`
- `GET /demo/{camera_id}`
- `GET /demo/{camera_id}/json`
- `GET /demo/{camera_id}/frame`
- `GET /home`
- `GET /ui/videos`
- `GET /ui/videos/{camera_id}`
- `GET /ui/{camera_id}/frame`
- `GET /debug/pipeline-metrics`
- `GET /debug/pipeline-metrics/{camera_id}`

## `rule_engine` Details

### What It Does

The rule engine polls current state and evaluates rule conditions.

It turns tracked object state into alerts.

### Important Files

- `rule_engine/main.py`
- `rule_engine/api/server.py`
- `rule_engine/config/engine.yaml`
- `rule_engine/config/rules.yaml`
- `rule_engine/core/rule_evaluator.py`
- `rule_engine/core/alert_store.py`

### Rule Logic

The evaluator checks fields like:

- label
- zone
- restricted zone
- authorization status
- excluded from alerts
- minimum duration
- minimum average confidence

This means rules do not need to understand raw frame data.
They operate on the already-structured tracked state.

### Alert Flow

1. `state_layer` produces current tracked state.
2. `rule_engine` polls that state.
3. Matching rules create alerts.
4. Alerts are exposed through alert routes.

### Rule Engine APIs

Current routes include:

- `GET /`
- `GET /health`
- `GET /rules`
- `GET /alerts`
- `GET /alerts/active`
- `GET /alerts/recent`

## Why This Design Works

This architecture separates concerns well:

- frame capture is isolated
- inference is isolated
- tracking is isolated
- rules are isolated
- UI is isolated

That makes it easier to debug, test, and extend each part independently.

It also lets us add features without rewriting the entire system.

## Current Key Features

- buffered video processing
- dynamic inference batching
- RT-DETR detection
- CLIP-based doctor/non-doctor classification
- track persistence across frames
- restricted zone logic
- unauthorized person alerting
- live stream and buffered frame APIs
- visual demo route
- SVG visualization route
- insight dashboard route
- pipeline timing metrics

## Important Notes

- The repository currently focuses on `cam1`.
- The pipeline is designed for local execution first.
- `inference_engine` is a worker, not a web server.
- The insight dashboard depends on both `frame_service` session routes and `state_layer` insight services.

## Quick Route Summary

- `http://localhost:8000` - frame service
- `http://localhost:8002` - state layer
- `http://localhost:8003` - rule engine

## Final Summary

In simple words, this project watches a video source, detects people, classifies whether they look like doctors, tracks them over time, checks whether they are in restricted zones, and raises alerts if unauthorized people are present.

The extra UI layers make it easy to inspect what the system saw, how it tracked it, and why an alert fired.

