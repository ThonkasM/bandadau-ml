# bandadau-ml

Notebooks y scripts Python para entrenar, evaluar y exportar a ONNX los modelos de Bandada U:
clasificadores NSFW y de violencia, CLIP zero-shot, ranker del feed, sugerencias de seguimiento,
detector de anomalías y el clasificador pequeño on-device.

```bash
uv sync
uv run jupyter lab
uv run python scripts/validate_onnx.py --model nsfw-classifier --version 1.0.0
uv run pytest
```

Los modelos se publican en MinIO/S3 (`models/<nombre>/<versión>/model.onnx`), nunca en git si superan 50 MB.
Cada modelo tiene su `model-card.md`. Reglas y contratos con los servicios: **[`AGENTS.md`](AGENTS.md)**.
Decisión de arquitectura: `bandadau-docs/docs/03-arquitectura/decisiones-clave.md` (D7).
