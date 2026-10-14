# AGENTS.md — bandadau-ml

Repositorio de ML (ADR-0017). Lee primero `../AGENTS.md` (workspace), `bandadau-docs/docs/03-arquitectura/decisiones-clave.md` (D7),
`bandadau-docs/docs/03-arquitectura/ml-dl.md` y `bandadau-docs/docs/04-patrones-y-reglas/definition-of-done.md` §6.

## Identidad

Entrenamiento, evaluación y **export a ONNX** de los modelos de ML/DL (T9). Aquí no se sirve
inferencia en producción: los servicios (`media-service`, `social-service`, `insights-service`) y la
app (`nsfw-mobile`) consumen los artefactos publicados en MinIO/S3.

## Stack

Python 3.12+ (verificar) · `uv` (o `pip-tools`) con dependencias fijadas · Jupyter · PyTorch +
`timm`/`transformers` (DL) · scikit-learn + `skl2onnx` (ML clásico) · `onnx`, `onnxruntime` · pandas ·
Ruff (format + lint) · pytest.

## Carpetas

```
bandadau-ml/
├── pyproject.toml  uv.lock  .env.example
├── notebooks/
│   ├── 01-nsfw-finetune.ipynb         → nsfw-classifier (+ derivado nsfw-mobile)
│   ├── 02-violence.ipynb              → violence-classifier
│   ├── 03-feed-ranker.ipynb           → feed-ranker (coeficientes JSON para social-service)
│   ├── 04-suggestions.ipynb           → follow-suggester
│   └── 05-anomalies.ipynb             → anomaly-detector (Isolation Forest → ONNX)
├── src/bandada_ml/                    utilidades reutilizables (datasets, export, validate, publish)
├── scripts/                           export_onnx.py, validate_onnx.py, publish_model.py
├── models/<nombre>/<versión>/model-card.md   (los .onnx NO van a git si > 50 MB; ver abajo)
├── data/                              solo README con origen/licencia; datos crudos fuera de git
└── tests/
```

## Comandos (propuestos)

```bash
uv sync
uv run jupyter lab
uv run python scripts/export_onnx.py --model nsfw-classifier --version 1.0.0
uv run python scripts/validate_onnx.py --model nsfw-classifier --version 1.0.0   # original vs ONNX
uv run python scripts/publish_model.py --model nsfw-classifier --version 1.0.0   # sube a MinIO/S3
uv run ruff format --check . && uv run ruff check . && uv run pytest
```

## Variables de entorno (`.env.example`)

```
S3_ENDPOINT=http://localhost:9000
S3_BUCKET_MODELS=bandada-models
S3_ACCESS_KEY_ID=minio
S3_SECRET_ACCESS_KEY=change-me
MLFLOW_TRACKING_URI=            # opcional
RANDOM_SEED=42
```

## Contratos con los consumidores

| Modelo | Artefacto | Entrada / salida | Consumidor (variable de versión) |
|---|---|---|---|
| `nsfw-classifier` | `models/nsfw-classifier/<v>/model.onnx` | imagen 224×224 RGB normalizada → probabilidades por clase | media-service (`MODEL_NSFW_VERSION`) |
| `violence-classifier` | idem | idem | media-service (`MODEL_VIOLENCE_VERSION`) |
| `clip-zero-shot` | preentrenado exportado | imagen + textos de etiquetas (`apuntes`, `evento`, `deporte`, `meme`…) | media-service |
| `feed-ranker` | `coefficients.json` | features documentadas → score | social-service (`FEED_MODEL_PATH`) |
| `follow-suggester` | parámetros JSON | — | social-service |
| `anomaly-detector` | `model.onnx` | serie normalizada → score de anomalía | insights-service (`ANOMALY_MODEL_VERSION`) |
| `nsfw-mobile` | `model.onnx` ~5 MB | igual que nsfw (cuantizado) | bandadau-frontend |

La forma exacta de entradas/salidas, normalización y etiquetas se documenta en cada `model-card.md`
y es un **contrato**: cambiarla exige versión mayor del modelo.

## Reglas específicas y gotchas

- Reproducibilidad: semilla fija, versión de dataset y dependencias fijadas; notebooks ejecutables de principio a fin.
- Todo export se **valida** contra el modelo original (tolerancia 1e-4 o documentada) antes de publicar.
- Umbrales de D7 (`≥ 0.85` BLOCK, `0.5–0.85` REVIEW) se recalibran aquí y se reportan; cambiarlos exige ADR.
- **Nunca** commitear modelos > 50 MB, datasets ni imágenes no aptas reales; usar MinIO/S3 y fixtures sintéticos.
- Datos personales de la plataforma solo agregados/anonimizados para `feed-ranker` y `anomaly-detector`.
- Licencias de modelos preentrenados y datasets registradas en el `model-card.md`.
- Identificadores y código en inglés; notebooks con explicaciones en español.

## Casos de uso

UC-05, UC-18 (moderación DL), UC-06 (feed), UC-08 (sugerencias), UC-20 (anomalías), UC-05 on-device.

## Definición de hecho

`definition-of-done.md` §6 + prueba de integración del servicio consumidor con la nueva versión.
