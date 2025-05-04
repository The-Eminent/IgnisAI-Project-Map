# Project Map - Ignis AI

## ignis-ai-backend/

```
├─ models/
│  ├─ Wildfire.js             # Mongoose schema for real-time fire detections
│  ├─ Weather.js              # Schema for NOAA weather forecasts
│  ├─ Topography.js           # Schema for terrain/elevation data
│  ├─ HumanFactors.js         # Schema for human factors (population, roads, stations)
│  └─ HistoricalWildfire.js   # Schema for archival fire statistics
├─ routes/
│  ├─ fireData.js             # GET  /api/wildfires → fetch NASA FIRMS CSV, parse & store
│  ├─ weather.js              # GET  /api/weather    → fetch & store NOAA forecasts
│  ├─ topography.js           # GET  /api/topography → pull terrain-RGB tiles & compute elevation
│  ├─ humanFactors.js         # GET  /api/human-factors → (placeholder) external data
│  └─ predictSpread.js        # POST /api/predict    → invoke ML inference (predict_spread.py)
├─ ml/                        # Python ML pipeline and inference
│  ├─ data/                   # (ignored) raw TFRecord files for training
│  ├─ process_data_dual.py    # Extract features (classification + regression) from TFRecords
│  ├─ train_classifier_advanced.py  # Train & save GradientBoostingClassifier
│  ├─ train_regressor_advanced.py   # Train & save GradientBoostingRegressor
│  ├─ wildfire_spread_classifier_advanced.joblib  # Pre-trained classifier model
│  ├─ wildfire_spread_regressor_advanced.joblib   # Pre-trained regressor model
│  └─ predict_spread.py       # Python inference: loads models, fetches real weather/elevation, returns GeoJSON
├─ db.js                      # MongoDB connection setup
├─ app.js                     # Express server, mounts all routes
├─ .env                       # API keys (NASA, Mapbox, MongoDB URI)
└─ package.json               # Node dependencies & start script
```

## ignis-ai-frontend/

```
└─ src/
   ├─ api.js                  # Axios wrapper: getWildfireData(), getWeather(), getTopography(), predictFireSpread()
   ├─ App.js                  # Top-level: holds filters, location, passes props
   ├─ App.css                 # Full-screen styles
   ├─ components/
   │  ├─ MapComponent.jsx     # Mapbox map: loads GeoJSON, popups, predict button, ML visualization
   │  ├─ FireControls.js      # Panel: refresh, filters, location search, nearby fires
   │  └─ LocationSearch.js    # (unused) optional autocomplete
   ├─ predictFireSpread.js    # Front-end helper to wrap POST /api/predict
   ├─ index.js                # Renders <App />
   └─ index.css
```

---
### 🔄 Data Flow End-to-End

1. *Backend: Data Ingestion*  
   - *fireData.js* calls NASA FIRMS CSV API every time /api/wildfires is hit.  
   - CSV rows parsed (latitude, longitude, brightness, confidence, satellite, timestamp).  
   - Parsed objects inserted into Mongo wildfires collection.  
   - JSON response returned: { message, count, data: [...] }.

2. *Frontend: Fetch & Display*  
   - *api.js* exports getWildfireData() which does axios.get('/api/wildfires').  
   - *MapComponent.jsx* (inside App) calls getWildfireData() → receives data.data array.  
     - Stores in component state, builds a GeoJSON source → Mapbox circle layer.  
     - Applies brightness & confidence filters passed from *FireControls*.  
     - Popups show brightness category, confidence as exact % (e.g. "41%"), timestamp and reverse-geocoded address.
     - Popups include a "Predict Fire Spread" button.

3. *User Controls & Nearby Fires*  
   - *FireControls.js* lets user:  
     - Refresh the data.  
     - Select brightness/confidence filters (passed up to App, then into MapComponent).  
     - Search for a location or "Use My Location" → sets userLocation.  
     - Enter a radius → MapComponent computes Haversine distances to all fires.  
     - Reverse-geocodes the nearest fires, returns enriched list via onNearbyFiresUpdate.  
     - Panel shows top 10 in‐range fires, sorted by "Closest" or "Most Dangerous".

4. *Fire Spread Prediction*
   - When user clicks "Predict Fire Spread" button in a fire popup:
     - *MapComponent.jsx* calls predictFireSpread() from *api.js* with fire data.
     - *api.js* sends POST request to /api/predict-fire-spread with fire location and brightness.
     - Backend *predictFireSpread.js* route calls Python *predict_spread.py* script.
     - Python script:
       - Loads trained ML models (classifier and regressor).
       - Fetches real-time weather data for the fire location.
       - Predicts fire spread probability, direction, and distance.
       - Returns GeoJSON visualization data and environmental factors.
     - *MapComponent.jsx* displays:
       - Fire spread polygon (yellow-orange gradient based on probability).
       - Direction arrow showing primary spread direction.
       - Points around perimeter showing spread probability.
       - Popup with prediction details and environmental data.

5. *Styling & UX*  
   - *App.css* / *index.css* provide full-screen layout & basic resets.  
   - The sliding panel is a fixed <div> over the map, fully responsive in width/height.  
   - Map markers animate on hover; popups have a clean card style.
   - Fire spread visualization uses color gradients to show probability.

---

### 📌 Key Connections

- *API endpoint* /api/wildfires ←→ *api.js* ←→ *MapComponent.jsx*  
- *API endpoint* /api/predict-fire-spread ←→ *api.js* ←→ *MapComponent.jsx*
- *App.js* holds global state:  
  - brightnessFilter, confidenceFilter → passed to MapComponent  
  - userLocation, range → passed to MapComponent  
  - nearbyFires ← from MapComponent → passed to FireControls
  - selectedFire, firePrediction → for fire spread prediction
- *FireControls.js* UI ↔ callbacks to App.js (onChangeBrightness, onNearbyFiresUpdate, etc.)
- *ML models* ↔ *predict_spread.py* ↔ *predictFireSpread.js* route ↔ frontend

---

### 🔥 ML Component Details

- **Models**: Two machine learning models work together:
  - *Classifier*: Predicts if a fire will spread significantly (yes/no).
  - *Regressor*: Predicts how much a fire will spread (spread ratio).

- **Features Used**:
  - Environmental data (elevation, wind, temperature, humidity)
  - Fire characteristics (brightness, location)
  - Derived features (wind components, drought-vegetation interaction)

- **Visualization Logic**:
  - Spread probability < 10%: Simple message, no visualization
  - Spread probability ≥ 10%: Full visualization with spread polygon
  - Spread probability 10-20%: "Possibly" will spread
  - Spread probability ≥ 20%: "Yes" will spread

- **Real-time Data**:
  - Weather API provides current conditions at fire location
  - Location-based estimates for drought and vegetation

