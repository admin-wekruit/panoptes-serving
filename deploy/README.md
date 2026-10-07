# Model services on the GPU boxes

One model = one FastAPI service = one container; one compose stack per card (`serving/registry.yaml` says which services sit on
which card). Contract: `docs/BACKENDS-v1.md`. Config: `.env` only (`.env.example`, the "model services" block).

```sh
git clone https://github.com/admin-wekruit/panoptes-serving.git && cd panoptes-serving
git clone <ehs-spatial> ../ehs-spatial                       # the function bodies and the audited docker recipes
cp .env.example .env                                         # PANOPTES_SERVICE_API_KEY, WEIGHTS
python3 ../ehs-spatial/scripts/onprem/fetch_weights_sam3d.py    --cache $WEIGHTS   # card A (HF_TOKEN; gated facebook/sam-3d-objects)
python3 ../ehs-spatial/scripts/onprem/fetch_weights_geometry.py --cache $WEIGHTS   # card B (da3-base, roma_outdoor, roma_dinov2, moge3)
make images GPU=a && make up GPU=a && make smoke GPU=a       # sam3d :8805 + sam3 :8801
make images GPU=b && make up GPU=b && make smoke GPU=b       # geometry-mvs :8804 + moge :8803 + mapanything :8802
```

- `make context` assembles the build context (`deploy/context.sh`): ehs-spatial's `stage_context.sh` layout (`SRC/workcell`,
  `SRC/serving`, its `.dockerignore` whitelist) plus this repo's `serving/` and `deploy/`.
- `deploy/Dockerfile.sam3d` and `deploy/Dockerfile.geometry` build FROM the ehs-spatial images (`docker/sam3d.Dockerfile`,
  `docker/{da3,geometry,workcell-gpu}.Dockerfile`), so every pin is the audited one and lives in one place; `make images` builds
  the bases first. `deploy/Dockerfile.serving` is the three `docs/BACKENDS.md` services as `serving/HANDOFF.md` installs them.
- Air-gapped boxes: build on a connected machine, then ehs-spatial `scripts/onprem/airgap.sh save / load`.
- Verify: `make test` (fake backends, no GPU) and `serving/smoke_v1.sh URL` (a running service; the real model when deployed).
  `PANOPTES_CONTRACT_URL=http://<gpu-a>:8805 pytest tests/contract/live.py` runs the contract tests against the live service.
