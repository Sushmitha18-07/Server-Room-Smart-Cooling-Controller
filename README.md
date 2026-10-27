# Server-Room-Smart-Cooling-Controller
Occupancy and thermal-sensing based cooling controller for server rooms and small data centers.
// Occupancy + temperature based cooling controller (ESP32)
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22
const int PIN_PIR = 13, PIN_RELAY = 26;

const float SET_OCCUPIED = 24.0f;    // setpoint when someone is inside
const float SET_EMPTY = 27.0f;       // relaxed setpoint when empty
const float HYST = 1.0f;
const float HOT_OVERRIDE = 32.0f;    // always cool above this
const unsigned long OCC_HOLD_MS = 10UL * 60000;  // stay "occupied" 10 min after last motion
const unsigned long MIN_ON_MS = 2UL * 60000;     // protect compressor
const unsigned long MIN_OFF_MS = 3UL * 60000;

DHT dht(DHTPIN, DHTTYPE);
bool cooling = false;
unsigned long lastMotion = 0, tOn = 0, tOff = 0, tRead = 0;

void setCooling(bool on, unsigned long now) {
  if (on == cooling) return;
  cooling = on;
  digitalWrite(PIN_RELAY, on ? HIGH : LOW);
  if (on) tOn = now; else tOff = now;
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_PIR, INPUT);
  pinMode(PIN_RELAY, OUTPUT);
  digitalWrite(PIN_RELAY, LOW);
  dht.begin();
}

void loop() {
  unsigned long now = millis();
  if (digitalRead(PIN_PIR)) lastMotion = now;
  if (now - tRead < 2000) return;   // DHT22 needs 2 s between reads
  tRead = now;

  float t = dht.readTemperature();
  if (isnan(t)) { setCooling(true, now); Serial.println("Sensor fault: cooling ON (fail-safe)"); return; }

  bool occupied = lastMotion != 0 && (now - lastMotion) < OCC_HOLD_MS;
  float setpoint = occupied ? SET_OCCUPIED : SET_EMPTY;

  bool minOffOk = (tOff == 0) || (now - tOff >= MIN_OFF_MS);
  bool minOnOk = (now - tOn >= MIN_ON_MS);

  if (t >= HOT_OVERRIDE) setCooling(true, now);                       // safety override
  else if (!cooling && t >= setpoint + HYST && minOffOk) setCooling(true, now);
  else if (cooling && t <= setpoint - HYST && minOnOk) setCooling(false, now);

  Serial.printf("T=%.1fC | %s | setpoint %.1f | cooling %s\n",
                t, occupied ? "OCCUPIED" : "EMPTY", setpoint, cooling ? "ON" : "OFF");
}
