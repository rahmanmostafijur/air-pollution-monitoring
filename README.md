# Air Pollution Monitoring and Forecasting

An end-to-end IoT and machine-learning system for air quality: an Arduino sensor node streams live readings to ThingSpeak, a browser dashboard shows those channels alongside current weather, and a trained classifier predicts whether conditions count as polluted.

Undergraduate final-year project. Solo build - hardware, firmware, dashboard and models.

---

## How it works

```
MQ-2 / MQ-135 / LM35  ->  Arduino + ESP8266  ->  ThingSpeak  ->  Dashboard
   (gas, air quality,        (sampling,          (time-series      (live charts
    temperature)              WiFi upload)        storage)          + weather)
                                                       |
                                              Labelled readings
                                                       |
                                            Classifier (Keras MLP)
                                                       |
                                              Polluted / Not polluted
```

**Sensor node** - an Arduino reads three analogue sensors and pushes each sample to a ThingSpeak channel over WiFi through an ESP8266 module on software serial.

| Sensor | Pin | Measures |
| --- | --- | --- |
| MQ-135 | A0 | Air quality - CO2, ammonia, benzene, NOx |
| MQ-2 | A1 | Combustible gases, smoke |
| LM35 | A2 | Ambient temperature |

**Dashboard** - a Bootstrap 5 page that embeds the three live ThingSpeak charts, pulls current conditions from the OpenWeatherMap and WeatherAPI REST APIs (temperature, humidity, wind, UV, pressure, visibility, precipitation), and renders the labelled dataset into a table.

**Models** - a Jupyter notebook that trains and compares three classifiers on the labelled air-quality dataset.

---

## Dataset

`Final Air Quality Dataset.csv` - 4,530 labelled readings.

| Feature | Unit |
| --- | --- |
| O3 | ppb |
| CO | ppm |
| SO2 | ug/m3 |
| NO2 | ug/m3 |
| PM2.5 | ug/m3 |
| PM10 | ug/m3 |

Target: `Polluted` (binary). Split 80/20 with `random_state=42` - 3,624 train, 906 test.

---

## Results

| Model | Accuracy | Weighted F1 | Notes |
| --- | --- | --- | --- |
| **Feedforward NN (Keras MLP)** | **0.89** | **0.89** | Best - 20 epochs, batch 32, standardised features |
| Logistic Regression | 0.81 | 0.80 | Solid baseline |
| SVM (RBF, unscaled) | 0.66 | 0.53 | Collapsed to the majority class - see below |

The neural network was the clear winner: 0.94 recall on the *polluted* class and 0.80 on *clean*, against a 66% majority-class baseline.

The SVM result is a finding, not a typo. Trained on raw unscaled features it predicted `Polluted` for all 906 test samples - 0.00 precision and recall on the clean class - landing exactly on the majority-class rate. RBF kernels are distance-based, and with PM10 spanning hundreds while CO sits below 1, the large-magnitude features swamped the rest. The neural network avoided this because its inputs went through `StandardScaler` first.

The trained network is exported two ways: the fitted model via `joblib`, and the architecture as JSON.

---

## Repository

| File | Contents |
| --- | --- |
| `thingspeak.ino` | Arduino firmware - sensor sampling and ThingSpeak upload |
| `main.html` | Dashboard page - ThingSpeak chart embeds, weather panel, data table |
| `main.js` | Weather API calls and CSV table rendering |
| `Air_Quality_Detection.ipynb` | Preprocessing, EDA, and the three models |
| `Final Air Quality Dataset.csv` | Labelled dataset |

---

## Configuration

All credentials in this repository are placeholders. Replace them with your own:

| Placeholder | Where | Get it from |
| --- | --- | --- |
| `YOUR_WIFI_SSID` / `YOUR_WIFI_PASSWORD` | `thingspeak.ino` | Your network |
| `YOUR_THINGSPEAK_WRITE_API_KEY` | `thingspeak.ino` | ThingSpeak channel -> API Keys |
| `YOUR_OPENWEATHERMAP_API_KEY` | `main.js` | openweathermap.org |
| `YOUR_WEATHERAPI_KEY` | `main.js` | weatherapi.com |

The ThingSpeak chart embeds in `main.html` point at a channel ID - swap it for yours.

---

## Running it

**Notebook** - open `Air_Quality_Detection.ipynb` in Colab or Jupyter. The original loads the CSV from Google Drive; point that path at the local file instead.

```
pip install numpy pandas matplotlib scikit-learn tensorflow joblib
```

**Dashboard** - fill in the API keys in `main.js`, then open `main.html` in a browser.

**Firmware** - fill in the WiFi and ThingSpeak values in `thingspeak.ino`, wire MQ-135 to A0, MQ-2 to A1 and LM35 to A2, connect the ESP8266 to the software serial pins, and flash.

---

## Stack

Arduino - ESP8266 - ThingSpeak - Python - scikit-learn - TensorFlow/Keras - pandas - Matplotlib - Bootstrap 5 - JavaScript

---

## What I would do differently

- **Scale the features before the SVM.** A single `StandardScaler` in the pipeline would likely have put it near the neural network instead of at the majority-class floor.
- **Report a confusion matrix and ROC-AUC**, not just accuracy - on a 66/34 split, accuracy alone flatters a weak model.
- **Cross-validate.** One 80/20 split on 4,530 rows leaves the numbers noisier than they look.
- **Keep credentials out of source from day one.** They belong in a gitignored config file or environment variables - this version uses placeholders for exactly that reason.
