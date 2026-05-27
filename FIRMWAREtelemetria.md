# extensometro_esp32
// Ponte completa com 4 extensometros ativos
// HX711 RATE em GND = 10 Hz; RATE em VCC = 80 Hz
// Para ESP32: cuidado com nivel logico do DOUT se o HX711 estiver em 5 V
/*filtro ativado para eliminação de ruídos de uma aplicação em chassis de carro, para fazer uma telemetria de deformação de longo prazo, para picos de deformação momentanea desativar o filtro para melhor telemetria*/
// Ajuste conforme a ligacao no ESP32
const uint8_t PIN_HX711_DOUT = 16;
const uint8_t PIN_HX711_SCK  = 17;

HX711 scale;

// Configuracoes da medicao
const float Vexc = 5.0f;          // medir a tensao real aplicada na ponte
const float GF = 2.1f;            // gauge factor medio dos extensometros
const float HX711_FS = 0.020f;    // ganho 128: faixa aprox. +/-20 mV
const float BRIDGE_FACTOR = 1.0f; // ponte completa com 4 extensometros ativos
const float SIGNAL_SIGN = 1.0f;   // use -1.0f se o sinal sair invertido

// Telemetria
const uint16_t TX_HZ = 80;        // use 10 se o RATE estiver em GND
const uint32_t TX_PERIOD_US = 1000000UL / TX_HZ;

// Filtro simples
const bool USE_FILTERED_OUTPUT = false;
const float FILTER_ALPHA = 0.20f; // maior = responde mais rapido; menor = mais suave

float filteredMicrostrain = 0.0f;
bool filterInitialized = false;

uint32_t lastTxUs = 0;

void tareScale() {
  Serial.println("# tare_iniciando");

  scale.tare(30);

  filterInitialized = false;

  Serial.println("# tare_ok");
}

void setup() {
  Serial.begin(115200);
  delay(500);

  scale.begin(PIN_HX711_DOUT, PIN_HX711_SCK);
  scale.set_gain(128);

  if (!scale.wait_ready_timeout(1000)) {
    Serial.println("# erro_hx711_nao_respondeu");
  } else {
    tareScale();
  }

  Serial.println("t_ms,raw_zeroed,vout_mV,microstrain,percent");
}

void loop() {
  // Envie 't' pela serial para refazer o zero com o carro sem carga dinamica
  if (Serial.available()) {
    char cmd = Serial.read();

    if (cmd == 't' || cmd == 'T') {
      tareScale();
    }
  }

  // Leitura somente quando o HX711 tiver uma conversao pronta
  if (!scale.is_ready()) {
    return;
  }

  long rawZeroed = scale.get_value(1);

  float Vout = SIGNAL_SIGN * ((float)rawZeroed) * (HX711_FS / 8388608.0f);

  float strain = BRIDGE_FACTOR * Vout / (GF * Vexc);
  float microstrain = strain * 1000000.0f;

  if (!filterInitialized) {
    filteredMicrostrain = microstrain;
    filterInitialized = true;
  } else {
    filteredMicrostrain =
      FILTER_ALPHA * microstrain + (1.0f - FILTER_ALPHA) * filteredMicrostrain;
  }

  uint32_t nowUs = micros();

  if ((uint32_t)(nowUs - lastTxUs) < TX_PERIOD_US) {
    return;
  }

  lastTxUs = nowUs;

  float microstrainOut = USE_FILTERED_OUTPUT ? filteredMicrostrain : microstrain;
  float percent = (microstrainOut / 1000000.0f) * 100.0f;

  Serial.print(millis());
  Serial.print(",");

  Serial.print(rawZeroed);
  Serial.print(",");

  Serial.print(Vout * 1000.0f, 6);
  Serial.print(",");

  Serial.print(microstrainOut, 2);
  Serial.print(",");

  Serial.println(percent, 6);
}
