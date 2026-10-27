# Medi Mind

Medi Mind is a full-stack educational web app that predicts a likely disease category from free-text symptoms and combines it with a live heart-rate reading from an ESP32 + MAX30102 pulse sensor. It returns a confidence score, an emergency level, a suggested specialist, home-care and diet guidance, and nearby hospital suggestions.

## Features

- Symptom-text disease prediction with a confidence score and the top alternative diagnoses
- Patient questionnaire: age, sex, symptom group, duration, severity and pre-existing conditions
- Live heart-rate reading from an ESP32 + MAX30102 sensor over a serial port
- Vital triage levels: `EMERGENCY`, `CLINICIAN_REVIEW`, `RECHECK`, `NORMAL`, `UNAVAILABLE`
- Emergency-level detection (high / medium / low) from the symptom text
- Specialist recommendation, remedies, diet plan (recommended and avoid) and general advice
- Hospital suggestions with phone and map links (demo data, see Limitations)
- Feedback loop: users can correct a prediction, the correction is added to the dataset and the model is retrained
- Model statistics page and the last 25 predictions of the current session
- Optional image preview in the form (context only, not used by the model)

## Tech Stack

| Layer | Technology |
| --- | --- |
| ML | scikit-learn: TF-IDF (1 to 3-grams, sublinear TF) + Logistic Regression |
| Backend | Python, FastAPI, Uvicorn, pandas |
| Hardware | ESP32 + MAX30102, read with pyserial |
| Frontend | React 19, Vite 6, lucide-react |
| Storage | CSV dataset on disk; prediction history held in memory |

## How the Model Works

1. Load `patient_symptoms_textual.csv` (columns `patient_concern` and `diagnosis`).
2. Lowercase and clean the text, then make an 80/20 stratified train/test split (`random_state=42`).
3. Train a pipeline: `TfidfVectorizer(ngram_range=(1, 3), sublinear_tf=True)` followed by `LogisticRegression(C=5.0, max_iter=1000, random_state=42)`.
4. Return the predicted disease, its probability as confidence, and the top alternatives from `predict_proba`.
5. If a heart rate is available and outside the normal range, tachycardia or bradycardia terms are added to the model input, and the vital triage runs alongside the symptom result.

The dataset has 1,151 records across 20 classes (about 54 to 60 per class): Acidity, Allergy, Arthritis, Asthma, Blood Pressure Issue, Cardiac Issue, Common Cold, Dengue, Diabetes, Flu, Food Poisoning, Gastroenteritis, Infection, Kidney Stone, Malaria, Migraine, Pneumonia, Throat Infection, Typhoid and UTI. Much of it is template-based text generated with `expand_dataset.py`.

## Heart-Rate Hardware

- Board: ESP32 with a MAX30102 sensor, streaming over USB serial at **115200 baud**
- The serial port is set in `backend/app/sensor_service.py` as `ESP32_PORT = "COM3"`. Change it to match your machine (for example `/dev/ttyUSB0` on Linux).
- Accepted line formats: `BPM: <x> | Average BPM: <y>` or `VITAL,BPM,<x>,AVG,<y>`
- Readings outside **45 to 180 BPM** are rejected as unstable, and a **5-sample median filter** smooths the rest
- If no data arrives for 6 seconds, the sensor is reported as disconnected
- Close the Arduino Serial Monitor before starting the backend, since only one program can hold the port

The app still works without the sensor. Heart rate is then shown as unavailable and the prediction uses symptoms only.

## Project Structure

```text
medi_mind2/
  backend/
    app/
      main.py            FastAPI app and routes
      model_service.py   training, prediction, triage, advice, feedback
      sensor_service.py  ESP32 / MAX30102 serial reader
      hospitals.py       hospital suggestions by specialty
      schemas.py         request and response models
    requirements.txt
  frontend/
    src/
      services/api.js
      App.jsx
      main.jsx
      styles.css
    index.html
    package.json
  Advance_Disease_Detector_AI.py   original CLI version
  disease_deletctor_ai.py          original CLI version
  expand_dataset.py                dataset generation script
  patient_symptoms_textual.csv
  README.md
```

## Getting Started

You need Python 3.10 or newer and Node.js 18 or newer.

### Backend

```bash
cd backend
python -m venv .venv

# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
uvicorn app.main:app --reload
```

The API runs at `http://127.0.0.1:8000` and the interactive docs are at `http://127.0.0.1:8000/docs`.

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The app runs at `http://127.0.0.1:5173`. CORS in the backend only allows `localhost` and `127.0.0.1` on ports `5170` to `5179`. To point the frontend at a different backend, set `VITE_API_BASE_URL`.

## API

| Method | Endpoint | Description |
| --- | --- | --- |
| GET | `/health` | Status and current model accuracy |
| GET | `/api/vitals` | Latest heart-rate reading, sensor state and vital triage |
| POST | `/api/predict` | Predict a disease from symptoms (at least 3 words) |
| POST | `/api/feedback` | Submit a corrected label, then retrain the model |
| GET | `/api/stats` | Model accuracy, dataset statistics, top diseases and recent predictions |

Example request:

```json
POST /api/predict
{
  "symptoms": "severe headache with nausea and sensitivity to light",
  "age": 24,
  "severity": "moderate",
  "heart_rate": 78
}
```

Only `symptoms` is required.

## Results

- About 97% cross-validated accuracy on the public Symptom2Disease dataset
- About 74% on real, free-form patient descriptions
- Near-perfect accuracy on this repository's own dataset, which is mostly template-generated text. This reflects the repetitive wording of the data, not real-world performance.

The accuracy shown by `/health` and the dashboard is measured on a split of this same template-style dataset, so it is an optimistic figure.

## Limitations

- Educational prototype only, and not clinically validated
- No dataset links a live MAX30102 reading to confirmed diagnoses. Heart rate drives triage and care routing, not the disease label.
- The sensor port is hard-coded to `COM3`
- Hospital entries are demo data, not live or location-based results
- Feedback is written straight into the CSV and retrains the model, with no review step. Bad labels will change the model.
- Prediction history is kept in memory and resets when the server restarts
- Image upload is a preview only

## Team

Developed as a four-member group project under the guidance of Prof. Paras Narkhede.

## Author

Manav Parikh
