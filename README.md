# AgriSense Backend

FastAPI service that serves the trained crop-disease-detection model over a REST API.
Hand this whole folder (or its repo) to your frontend developer along with the section
below titled **"API contract for the frontend developer."**

## What this does

Given a leaf photo, it runs the exact same pipeline as the training notebook:

1. **Leaf-color check** — rejects images that clearly aren't plant photos.
2. **Model prediction** — runs the trained CNN.
3. **Confidence calibration** — applies temperature scaling so confidence numbers are honest.
4. **Confidence gate** — if calibrated confidence is too low, returns "not available" instead of a guess.

## Setup

### 1. Add your trained model files

Drop these three files into `artifacts/` (see `artifacts/README.md` for details):
- `model.keras`
- `class_names.json`
- `temperature.json`

Generate them by running `scripts/export_artifacts.py`'s contents as a cell in your
training notebook.

### 2. Run it

**With Docker (recommended):**
```bash
docker compose up --build
```

**Without Docker:**
```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\\Scripts\\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Either way, the API is now at `http://localhost:8000`, with interactive docs at
`http://localhost:8000/docs`.

### 3. Verify it's working

```bash
curl http://localhost:8000/api/health
```

Should return `{"status": "ok", "model_loaded": true, "num_classes": <N>}`. If it says
`model_not_loaded`, your `artifacts/` files aren't in place yet — check the server logs
for exactly what's missing.

---

## API contract for the frontend developer

Base URL: `http://localhost:8000` (or wherever this gets deployed).

### `POST /api/predict`

Send a leaf photo, get back a diagnosis (or a clear "couldn't tell" response).

**Request:** `multipart/form-data` with a single field:

| Field | Type | Notes |
|---|---|---|
| `file` | image file | jpg/jpeg/png/webp, max 8 MB |

Example with `curl`:
```bash
curl -X POST http://localhost:8000/api/predict \\
  -F "file=@/path/to/leaf.jpg"
```

Example with `fetch` (JS):
```js
const formData = new FormData();
formData.append("file", fileInput.files[0]);

const res = await fetch("http://localhost:8000/api/predict", {
  method: "POST",
  body: formData,
});
const result = await res.json();
```

**Response:** always `200 OK` with a JSON body if the image was processed. The `status`
field tells you which of three cases you're in — **build your UI around all three**, not
just the happy path:

**1. `status: "predicted"`** — model is confident enough to show a result.
```json
{
  "status": "predicted",
  "crop_disease": "Tomato___Late_blight",
  "crop": "Tomato",
  "disease": "Late_blight",
  "raw_confidence": 91.4,
  "calibrated_confidence": 84.2
}
```
Show `crop` + `disease` (format `disease` for display, e.g. replace underscores with
spaces) and `calibrated_confidence` as the "confidence" number — not `raw_confidence`,
which runs artificially high. A `disease` value of `"healthy"` means no disease detected.

**2. `status: "not_a_leaf"`** — image doesn't look like a plant photo at all.
```json
{
  "status": "not_a_leaf",
  "message": "This doesn't look like a plant leaf photo. Please upload a clear, well-lit photo of a leaf.",
  "leaf_color_fraction": 0.003
}
```
Show `message` directly to the user and prompt them to re-upload.

**3. `status: "low_confidence"`** — it's a leaf, but the model isn't sure enough to commit
to an answer.
```json
{
  "status": "low_confidence",
  "message": "This leaf doesn't confidently match any disease this model was trained on. Please consult a local agricultural expert or extension officer.",
  "top_guess": "Blueberry___healthy",
  "raw_confidence": 88.4,
  "calibrated_confidence": 68.5
}
```
Show `message` — **do not show `top_guess` as if it were the answer**, it's below the
confidence bar on purpose. It's included only for your own debugging/logging.

**Error responses** (no image / bad file / model not loaded) return a normal HTTP error
code (`400`, `415`, `422`, `503`) with `{"detail": "..."}`. Handle these as network/app
errors, not as prediction results.

### `GET /api/health`

For a startup check or status indicator in your app.
```json
{"status": "ok", "model_loaded": true, "num_classes": 38}
```

### `GET /api/classes`

All crop-disease labels the model knows — useful if you want to build something like a
"supported crops" list in the UI.
```json
{"classes": ["Apple___Apple_scab", "Apple___Black_rot", "..."], "count": 38}
```

---

## Known limitations (tell your frontend dev / demo audience honestly)

- **Crop misidentification**: a real leaf can occasionally get matched to the wrong crop
  species if its disease has no matching class for the correct crop (e.g. a tomato leaf
  with powdery mildew, since PlantVillage has no `Tomato___Powdery_mildew` class). The
  confidence gate does NOT reliably catch this, because the visual match to the wrong
  class can still be genuinely strong. Fixing this needs a two-stage crop→disease model
  (see notebook discussion) — not yet implemented.
- **Thresholds are heuristic, not exhaustively tuned.** `CONFIDENCE_THRESHOLD` (70%) and
  `MIN_LEAF_COLOR_FRACTION` (0.10) are reasonable starting points, not validated numbers.
  Adjust them in `.env` / `docker-compose.yml` based on real testing.
- **Advisory text (IPM recommendations, safe pesticide dosage, etc.) is not implemented
  yet.** The API currently returns only the crop + disease + confidence. Any treatment
  advice shown in the frontend should be clearly marked as needing expert review before
  being presented as authoritative — this was called out as a requirement in the original
  problem statement ("expert validation").

## Project structure

```
agrisense_backend/
├── app/
│   ├── main.py         # FastAPI routes
│   ├── inference.py     # model loading + prediction pipeline
│   ├── schemas.py        # request/response models
│   └── config.py          # thresholds, paths, env vars
├── artifacts/               # put model.keras, class_names.json, temperature.json here
├── scripts/
│   └── export_artifacts.py    # notebook cell to generate the artifacts
├── Dockerfile
├── docker-compose.yml
└── requirements.txt
```
