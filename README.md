#include <WiFi.h>
#include <WebServer.h>
#include <PZEM004Tv30.h>
#include <math.h>

// ---------------- WiFi Credentials ----------------

const char* ssid = "D";
const char* password = "debjit2356";

// ---------------- ESP32 UART2 Pins ----------------

#define RXD2 16
#define TXD2 17

// ---------------- PZEM Sensor ----------------

PZEM004Tv30 pzem(Serial2, RXD2, TXD2);

// ---------------- Web Server ----------------

WebServer server(80);

// ---------------- Measurement Variables ----------------

float voltage = NAN;
float current = NAN;
float power = NAN;
float energy = NAN;
float frequency = NAN;
float pf = NAN;

// Minimum current threshold
const float LOAD_THRESHOLD = 0.02;

// Sensor reading interval
unsigned long lastRead = 0;
const unsigned long READ_INTERVAL = 2000;

// Load status
bool loadConnected = false;

// ---------------- Dashboard Webpage ----------------

const char webpage[] PROGMEM = R"rawliteral(

<!DOCTYPE html>
<html>

<head>

<meta name="viewport" content="width=device-width, initial-scale=1">

<title>ESP32 AC Energy Monitor</title>

<style>

body {
  margin: 0;
  font-family: Arial;
  background: #101827;
  color: white;
  text-align: center;
}

header {
  background: #172554;
  padding: 20px;
}

h1 {
  color: #38bdf8;
}

.container {
  display: grid;
  grid-template-columns: repeat(auto-fit,minmax(145px,1fr));
  gap: 15px;
  padding: 20px;
}

.card {
  background: #1e293b;
  padding: 22px 10px;
  border-radius: 14px;
}

.card h3 {
  color: #94a3b8;
}

.value {
  font-size: 26px;
  color: #22c55e;
  font-weight: bold;
}

.status {
  padding: 15px;
  font-size: 17px;
  font-weight: bold;
}

footer {
  color: #94a3b8;
  padding: 15px;
}

</style>

</head>

<body>

<header>

<h1>AC ENERGY MONITOR</h1>

<p>ESP32 + PZEM-004T V3.0</p>

</header>

<div class="status" id="status">

Connecting to sensor...

</div>

<div class="container">

<div class="card">
<h3>Voltage</h3>
<div class="value" id="voltage">--</div>
<p>Volts</p>
</div>

<div class="card">
<h3>Current</h3>
<div class="value" id="current">--</div>
<p>Ampere</p>
</div>

<div class="card">
<h3>Active Power</h3>
<div class="value" id="power">--</div>
<p>Watts</p>
</div>

<div class="card">
<h3>Energy</h3>
<div class="value" id="energy">--</div>
<p>kWh</p>
</div>

<div class="card">
<h3>Frequency</h3>
<div class="value" id="frequency">--</div>
<p>Hz</p>
</div>

<div class="card">
<h3>Power Factor</h3>
<div class="value" id="pf">--</div>
<p>PF</p>
</div>

</div>

<footer>

ESP32 IoT Energy Monitoring System

<br><br>

Auto Refresh: 2 Seconds

</footer>

<script>

async function updateData() {

  try {

    const response = await fetch('/api');

    if (!response.ok)
      throw new Error("Server error");

    const data = await response.json();

    document.getElementById('voltage').innerText =
      data.voltage === null ? '--' : data.voltage;

    document.getElementById('current').innerText =
      data.current === null ? '--' : data.current;

    document.getElementById('power').innerText =
      data.power === null ? '--' : data.power;

    document.getElementById('energy').innerText =
      data.energy === null ? '--' : data.energy;

    document.getElementById('frequency').innerText =
      data.frequency === null ? '--' : data.frequency;

    document.getElementById('pf').innerText =
      data.pf === null ? '--' : data.pf;

    const status = document.getElementById('status');

    if (data.loadConnected) {

      status.innerText = 'AC LOAD CONNECTED';

      status.style.color = '#22c55e';

    } else {

      status.innerText = 'NO AC LOAD CONNECTED';

      status.style.color = '#f87171';

    }

  }

  catch(error) {

    document.getElementById('status').innerText =
      'Connection Error';

  }

}

updateData();

setInterval(updateData, 2000);

</script>

</body>

</html>

)rawliteral";


// ---------------- JSON Value Function ----------------

String jsonValue(float value, int decimals) {

  if (!isfinite(value)) {
    return "null";
  }

  return String(value, decimals);

}


// ---------------- Dashboard Page ----------------

void handleRoot() {

  server.send_P(200, "text/html", webpage);

}


// ---------------- API Function ----------------

void handleAPI() {

  String json = "{";

  json += "\"voltage\":" + jsonValue(voltage, 1) + ",";

  json += "\"current\":" + jsonValue(current, 3) + ",";

  json += "\"power\":" + jsonValue(power, 1) + ",";

  json += "\"energy\":" + jsonValue(energy, 3) + ",";

  json += "\"frequency\":" + jsonValue(frequency, 1) + ",";

  json += "\"pf\":" + jsonValue(pf, 2) + ",";

  json += "\"loadConnected\":";
  json += loadConnected ? "true" : "false";

  json += "}";

  server.send(200, "application/json", json);

}


// ---------------- Setup ----------------

void setup() {

  Serial.begin(115200);

  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);

  Serial.println();

  Serial.println("ESP32 AC ENERGY MONITOR");

  // Connect WiFi

  WiFi.begin(ssid, password);

  Serial.print("Connecting to WiFi");

  while (WiFi.status() != WL_CONNECTED) {

    delay(500);

    Serial.print(".");

  }

  Serial.println();

  Serial.println("WiFi Connected!");

  Serial.print("Dashboard IP: ");

  Serial.println(WiFi.localIP());

  // Web server routes

  server.on("/", handleRoot);

  server.on("/api", handleAPI);

  server.begin();

  Serial.println("Web Dashboard Started");

}


// ---------------- Main Loop ----------------

void loop() {

  server.handleClient();

  if (millis() - lastRead >= READ_INTERVAL) {

    lastRead = millis();

    // Always read supply voltage

    voltage = pzem.voltage();

    // Read current

    float measuredCurrent = pzem.current();

    // Read energy and frequency

    energy = pzem.energy();

    frequency = pzem.frequency();


    // ---------------- Load Detection ----------------

    if (isfinite(measuredCurrent) &&
        measuredCurrent >= LOAD_THRESHOLD) {

      // AC load connected

      loadConnected = true;

      current = measuredCurrent;

      power = pzem.power();

      // Actual measured power factor

      pf = pzem.pf();

    }

    else {

      // No AC load connected

      loadConnected = false;

      current = NAN;

      power = NAN;

      pf = NAN;

    }


    // ---------------- Serial Monitor ----------------

    Serial.println("\n----------------------------");

    Serial.println("AC ENERGY MONITOR");

    Serial.println("----------------------------");

    Serial.print("Voltage: ");
    Serial.println(voltage);

    Serial.print("Current: ");
    Serial.println(current);

    Serial.print("Power: ");
    Serial.println(power);

    Serial.print("Energy: ");
    Serial.println(energy);

    Serial.print("Frequency: ");
    Serial.println(frequency);

    Serial.print("Power Factor: ");
    Serial.println(pf);

    Serial.print("Load Status: ");

    Serial.println(loadConnected ?
                   "CONNECTED" : "NOT CONNECTED");

    Serial.println("----------------------------");

  }

}
