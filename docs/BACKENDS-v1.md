# Model service contract v1 (2026-10-07)

Extends `docs/BACKENDS.md` (the SAM 3 / MapAnything / MoGe-3 one-POST contract stays as is). Everything new lives under `/v1/`.
One model = one FastAPI service = one container (images from `ehs-spatial/docker/*.Dockerfile`). Weights come from a mounted
volume and are SHA-256-checked at start. No service ever reaches the internet.

## Common

| Endpoint | Returns |
|---|---|
| `GET /healthz` | `{"ok": true, "model": "sam3d", "revision": "<weights revision>", "gpu": "NVIDIA A100-SXM4-80GB", "vram_free_mb": 61234, "queue_depth": 0, "running": 1}` — 200 even when busy; `ok:false` + 503 only when the model is not loaded |
| `GET /v1/info` | `{"model_id": "facebook/sam-3d-objects", "model_revision": "2e73555…", "code_revision": "f91db41…", "weights_sha256": {"ss_generator.ckpt": "…"}, "licence": "SAM License", "code_sha": "<git sha of this service>", "seed_policy": "seed in request; cuDNN deterministic on"}` |
| `POST /v1/<model>/jobs` | `202 {"job_id": "…", "input_sha256": "…", "cached": false}` — long calls; the body's `input_sha256` (client-computed over the canonical input bytes) is the idempotency key: a job with the same key returns the finished result |
| `GET /v1/jobs/{job_id}` | `{"status": "queued|running|done|error", "result": {...}|null, "error": {"code","message"}|null, "seconds": 12.3, "model_info": {...as /v1/info...}, "gpu": "..."}` |
| `POST /v1/<model>` | synchronous convenience for calls under ~60 s (sam3d single candidate): same body as the job, returns the job result directly |
| auth | header `X-API-Key: <PANOPTES_SERVICE_API_KEY>` on every request; 401 `{"error":{"code":"unauthorized"}}` |
| errors | `{"error": {"code": "unauthorized|bad_input|model_error|busy|not_found", "message": "..."}}`; 429 `busy` when `queue_depth >= PANOPTES_SERVICE_MAX_QUEUE` |
| inputs | binary fields accept either `"<name>_b64"` (base64 of the file bytes) or `"<name>_uri"` (`s3://bucket/key`, read with the service's own credentials); every request carries `input_sha256` |
| outputs | small: inline JSON / base64; large: `"result_uri": "s3://…/<input_sha256>.tar.gz"` when `PANOPTES_RESULT_BUCKET` is set, else `"result_b64"` of the tar.gz |

Every response (`/v1/jobs/{id}`, synchronous calls) carries `model_info` = the `/v1/info` document. Clients write it into their ledgers.

## `sam3d` (port 8805) — SAM 3D Objects, one candidate per call

Body = exactly what `modal_apps/sam3d_research.SAM3DObjects.run(rgb, mask, pointmap, seed)` takes:
```json
{"input_sha256": "…", "seed": 42,
 "image_b64": "<PNG, RGB, the whole photo at 1/factor>", "mask_b64": "<PNG, 0/255, same size>",
 "pointmap_npz_b64": "<npz: 'pointmap' float32 HxWx3, PyTorch3D camera (x, y negated), NaN = no depth>"}
```
Result (= `SAM3DObjects.run`'s return, as the research A/B stored it in `gen/<variant>/<object>.npz`):
```json
{"mesh_npz_b64": "<npz: vertices float32 Nx3, faces int32 Mx3, colors uint8 Nx3, object_to_camera_p3d float64 4x4>",
 "pins": {...}, "vertices": 52624, "faces": 104000, "seconds": 9.8, "gpu": "NVIDIA A100-SXM4-80GB", "model_info": {...}}
```

## `geometry-mvs` (port 8804) — the GPU half of the licence-clean geometry route

Input = the frozen canonical frames of a run; output = the three directories the CPU route reads, as one tar.gz:
```json
{"input_sha256": "…", "cell": "090",
 "frames": [{"frame_id": "frame_0001", "canonical_png_b64": "<518x518 RGB>", "alpha_npy_b64": "<518x518 bool>"}, ...],
 "options": {"start": "da3-base", "roma": "outdoor", "pairs": "all"}}
```
Result tar.gz layout (what `research/module-swap-2026-10-07/notes/geometry-backbone-ab-2026-10-06/mvs_route.py` consumes):
```
checks/clean-gpu/<cell>/roma-<i>-<j>.npz, dense-<i>-<j>.npz, moge-frame_000N.npz      # RoMa matches + dense warps (fp16 autocast), MoGe-3 depth
checks/da3fair-geom/<cell>-da3-base-padded/geometry/frames/<f>/{pts3d,conf,valid_mask,intrinsics,camera_to_world}.npy + candidate_manifest.json
checks/clean-geom/<cell>-da3-base-ba-f/geometry/...                                   # BA-f cameras (geometry_clean_ab.refine, numpy LM)
```
Function bodies: `ehs-spatial/modal_apps/geometry_clean_ab.py` (`infer`, `refine`, `dense-infer` stages) and `moge3_app.py`, unchanged.

## Client side (`ehs_spatial/providers/`)

| env | values |
|---|---|
| `SAM3D_BACKEND`, `GEOMETRY_MVS_BACKEND` | `http` (these services) \| `local` (same function bodies in this process through `scripts/onprem/run_stage.py`'s stub) \| `modal` (our Modal apps) |
| `SAM3D_HTTP_URLS`, `GEOMETRY_MVS_HTTP_URLS` | comma-separated service roots, e.g. `http://10.21.72.251:8805,http://10.21.72.251:8815`; the provider polls `/healthz` and sends to the smallest `queue_depth` (two A100s = two entries) |
| `PANOPTES_SERVICE_API_KEY` | sent as `X-API-Key` |
| `PANOPTES_SERVICE_TIMEOUT_S`, `PANOPTES_SERVICE_RETRIES` | defaults 1800 / 3 (exponential backoff; retried only on connection errors and 429/503, never on model_error) |

The pipeline code calls `providers.sam3d.generate(...)` / `providers.geometry_mvs.run(...)` only. No stage imports `modal`.

## Rules

1. Breaking change → `/v2/` beside `/v1/`; `/v1/` kept one release cycle.
2. Determinism inputs (seed, flags) are part of the request and of `input_sha256`.
3. Services log one JSON line per job: `{job_id, input_sha256, seconds, gpu, status}`; no payloads, no keys.
4. Contract tests (`tests/contract/`) run the same requests against a fake backend in CI and against a live service when `PANOPTES_CONTRACT_URL` is set.
