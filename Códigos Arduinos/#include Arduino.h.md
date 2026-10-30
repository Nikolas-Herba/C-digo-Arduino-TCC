Pra abrir as câmera: http://cam-tomate.local/config

# CAMAMERA WEB SERVER



\#include <Arduino.h>

\#include "esp\_camera.h"

\#include <WiFi.h>

\#include <HTTPClient.h>

\#include <Preferences.h>



\#include "board\_config.h"



// ============================================================

// W-IFI \& PERSISTÊNCIA

// ============================================================

Preferences preferences;



// Credenciais de emergência (Fallback Protegido)

const char\* fallback\_ssid = "NCA";

const char\* fallback\_password = "N21C30A19";



String current\_ssid = "";

String current\_password = "";



// Gestão de estado para alteração remota de Wi-Fi

bool testandoNovaRede = false;

String novo\_ssid\_temp = "";

String nova\_pass\_temp = "";

unsigned long tempoInicioTesteWifi = 0;



// ============================================================

// ESP32 + DHT22 (VALORES PROTEGIDOS)

// ============================================================

const char\* dht\_esp32\_ip = "192.168.15.10";



// ============================================================

// DJANGO (VALORES PROTEGIDOS)

// ============================================================

const char\* apiUrl = "http://192.168.15.5:8000//api/captures/";

const char\* authToken = "4f7e3d52612dd5fec4b9f7dcb3e4647a9250dee2";



// ============================================================

// GESTÃO DO INTERVALO DE FOTOGRAFIAS

// ============================================================

unsigned long intervaloCapturaMs = 600000; // 10 minutos por defeito

unsigned long ultimoTempoCaptura = 0;

bool modoTesteAtivo = false;



// ============================================================

// DECLARAÇÕES EXTERNAS (Para app\_httpd.cpp)

// ============================================================

void startCameraServer();

void setupLedFlash();



void aplicarNovoIntervalo(int minutos) {

&#x20; if(minutos <= 0) return;

&#x20; intervaloCapturaMs = (unsigned long)minutos \* 60000;

&#x20; preferences.begin("tcc", false);

&#x20; preferences.putUInt("intervalo", intervaloCapturaMs);

&#x20; preferences.end();

&#x20; ultimoTempoCaptura = millis(); // Reinicia o contador

&#x20; Serial.printf("\[SISTEMA] Novo intervalo aplicado: %d minutos\\n", minutos);

}



void agendarTesteWifi(String n\_ssid, String n\_pass) {

&#x20; novo\_ssid\_temp = n\_ssid;

&#x20; nova\_pass\_temp = n\_pass;

&#x20; testandoNovaRede = true;

&#x20; tempoInicioTesteWifi = millis();

}



// ============================================================

// OBTER DADOS DO DHT22 (Mantido Original)

// ============================================================

bool obterDadosDHT(float\& temperatura, float\& umidade) {

&#x20; if (WiFi.status() != WL\_CONNECTED) {

&#x20;   Serial.println("\[DHT22] Wi-Fi desconectado.");

&#x20;   return false;

&#x20; }



&#x20; WiFiClient client;

&#x20; HTTPClient http;

&#x20; String url = "http://" + String(dht\_esp32\_ip) + "/sensor";



&#x20; Serial.println("\\n================================");

&#x20; Serial.println("CONSULTANDO DHT22");

&#x20; Serial.print("\[DHT22] URL: ");

&#x20; Serial.println(url);



&#x20; client.setTimeout(5000);

&#x20; http.setConnectTimeout(5000);

&#x20; http.setTimeout(5000);



&#x20; if (!http.begin(client, url)) return false;



&#x20; int httpCode = http.GET();

&#x20; if (httpCode <= 0 || httpCode != HTTP\_CODE\_OK) {

&#x20;   http.end();

&#x20;   return false;

&#x20; }



&#x20; String resposta = http.getString();

&#x20; int tempInicio = resposta.indexOf("\\"temperature\\":");

&#x20; int humInicio = resposta.indexOf("\\"humidity\\":");



&#x20; if (tempInicio == -1 || humInicio == -1) {

&#x20;   http.end();

&#x20;   return false;

&#x20; }



&#x20; tempInicio += strlen("\\"temperature\\":");

&#x20; humInicio += strlen("\\"humidity\\":");

&#x20; int tempFim = resposta.indexOf(",", tempInicio);

&#x20; int humFim = resposta.indexOf("}", humInicio);



&#x20; temperatura = resposta.substring(tempInicio, tempFim).toFloat();

&#x20; umidade = resposta.substring(humInicio, humFim).toFloat();



&#x20; http.end();

&#x20; Serial.printf("\[OK] Temperatura: %.1f C | Umidade: %.1f %%\\n", temperatura, umidade);

&#x20; return true;

}



// ============================================================

// ENVIAR FOTO + DHT22 PARA DJANGO (Mantido Original)

// ============================================================

bool enviarParaDjango(camera\_fb\_t\* fb, float temperatura, float umidade) {

&#x20; if (WiFi.status() != WL\_CONNECTED) return false;



&#x20; WiFiClient client;

&#x20; HTTPClient http;

&#x20; if (!http.begin(client, apiUrl)) return false;



&#x20; String boundary = "----ESP32CAMBoundary";

&#x20; String bodyStart = "--" + boundary + "\\r\\n";

&#x20; bodyStart += "Content-Disposition: form-data; name=\\"temperature\\"\\r\\n\\r\\n" + String(temperatura, 1) + "\\r\\n";

&#x20; bodyStart += "--" + boundary + "\\r\\n";

&#x20; bodyStart += "Content-Disposition: form-data; name=\\"humidity\\"\\r\\n\\r\\n" + String(umidade, 1) + "\\r\\n";

&#x20; bodyStart += "--" + boundary + "\\r\\n";

&#x20; bodyStart += "Content-Disposition: form-data; name=\\"observation\\"\\r\\n\\r\\nCaptura automatizada TCC\\r\\n";

&#x20; bodyStart += "--" + boundary + "\\r\\n";

&#x20; bodyStart += "Content-Disposition: form-data; name=\\"image\\"; filename=\\"tomate.jpg\\"\\r\\n";

&#x20; bodyStart += "Content-Type: image/jpeg\\r\\n\\r\\n";



&#x20; String bodyEnd = "\\r\\n--" + boundary + "--\\r\\n";

&#x20; size\_t totalLength = bodyStart.length() + fb->len + bodyEnd.length();



&#x20; http.addHeader("Content-Type", "multipart/form-data; boundary=" + boundary);

&#x20; http.addHeader("Authorization", "Token " + String(authToken));

&#x20; http.addHeader("Content-Length", String(totalLength));



&#x20; String payload;

&#x20; payload.reserve(totalLength);

&#x20; payload = bodyStart;

&#x20; for (size\_t i = 0; i < fb->len; i++) payload += (char)fb->buf\[i];

&#x20; payload += bodyEnd;



&#x20; int httpCode = http.POST((uint8\_t\*)payload.c\_str(), payload.length());

&#x20; 

&#x20; if (httpCode > 0) {

&#x20;   Serial.println("\[Django] Resposta: " + http.getString());

&#x20; } else {

&#x20;   Serial.println("\[ERRO] Falha no envio para Django.");

&#x20; }

&#x20; 

&#x20; http.end();

&#x20; return (httpCode >= 200 \&\& httpCode < 300);

}



// ============================================================

// PROCESSAMENTO DO TCC (Chamado via HTTP ou Timer)

// ============================================================

bool processarCapturaTCC(camera\_fb\_t\* fb) {

&#x20; if (fb == nullptr) return false;

&#x20; float temperatura = 0.0, umidade = 0.0;



&#x20; if (!obterDadosDHT(temperatura, umidade)) {

&#x20;   Serial.println("\[TCC] Sem dados do DHT22. Cancelando envio para preservar integridade.");

&#x20;   return false;

&#x20; }

&#x20; return enviarParaDjango(fb, temperatura, umidade);

}



// ============================================================

// FUNÇÃO AUXILIAR DE LIGAÇÃO WI-FI

// ============================================================

bool conectarWiFi(String ssid, String pass, int timeoutSecs) {

&#x20; WiFi.disconnect();

&#x20; delay(100);

&#x20; WiFi.begin(ssid.c\_str(), pass.c\_str());

&#x20; Serial.printf("Tentando conectar a: %s ", ssid.c\_str());

&#x20; 

&#x20; int tentativas = 0;

&#x20; while (WiFi.status() != WL\_CONNECTED \&\& tentativas < (timeoutSecs \* 2)) {

&#x20;   delay(500);

&#x20;   Serial.print(".");

&#x20;   tentativas++;

&#x20; }

&#x20; Serial.println();

&#x20; return (WiFi.status() == WL\_CONNECTED);

}



// ============================================================

// SETUP

// ============================================================

void setup() {

&#x20; Serial.begin(115200);

&#x20; Serial.println("\\n\[SISTEMA] Inicializando TCC - Câmera + DHT22 + Django");



&#x20; // Carregar preferências persistentes

&#x20; preferences.begin("tcc", false);

&#x20; current\_ssid = preferences.getString("ssid", fallback\_ssid);

&#x20; current\_password = preferences.getString("password", fallback\_password);

&#x20; intervaloCapturaMs = preferences.getUInt("intervalo", 600000);

&#x20; preferences.end();



&#x20; // Configuração da Câmara

&#x20; camera\_config\_t config;

&#x20; config.ledc\_channel = LEDC\_CHANNEL\_0;

&#x20; config.ledc\_timer = LEDC\_TIMER\_0;

&#x20; config.pin\_d0 = Y2\_GPIO\_NUM; config.pin\_d1 = Y3\_GPIO\_NUM;

&#x20; config.pin\_d2 = Y4\_GPIO\_NUM; config.pin\_d3 = Y5\_GPIO\_NUM;

&#x20; config.pin\_d4 = Y6\_GPIO\_NUM; config.pin\_d5 = Y7\_GPIO\_NUM;

&#x20; config.pin\_d6 = Y8\_GPIO\_NUM; config.pin\_d7 = Y9\_GPIO\_NUM;

&#x20; config.pin\_xclk = XCLK\_GPIO\_NUM; config.pin\_pclk = PCLK\_GPIO\_NUM;

&#x20; config.pin\_vsync = VSYNC\_GPIO\_NUM; config.pin\_href = HREF\_GPIO\_NUM;

&#x20; config.pin\_sccb\_sda = SIOD\_GPIO\_NUM; config.pin\_sccb\_scl = SIOC\_GPIO\_NUM;

&#x20; config.pin\_pwdn = PWDN\_GPIO\_NUM; config.pin\_reset = RESET\_GPIO\_NUM;

&#x20; config.xclk\_freq\_hz = 20000000;

&#x20; config.pixel\_format = PIXFORMAT\_JPEG;



&#x20; if (psramFound()) {

&#x20;   config.frame\_size = FRAMESIZE\_SVGA;

&#x20;   config.jpeg\_quality = 12;

&#x20;   config.fb\_count = 2;

&#x20;   config.grab\_mode = CAMERA\_GRAB\_LATEST;

&#x20;   config.fb\_location = CAMERA\_FB\_IN\_PSRAM;

&#x20; } else {

&#x20;   config.frame\_size = FRAMESIZE\_VGA;

&#x20;   config.jpeg\_quality = 14;

&#x20;   config.fb\_count = 1;

&#x20;   config.fb\_location = CAMERA\_FB\_IN\_DRAM;

&#x20; }



&#x20; esp\_err\_t err = esp\_camera\_init(\&config);

&#x20; if (err != ESP\_OK) {

&#x20;   Serial.printf("\[ERRO] Câmera falhou: 0x%x\\n", err);

&#x20;   return;

&#x20; }



&#x20; // Otimização para Sensor Real (OV5640 / OV2640)

&#x20; sensor\_t\* s = esp\_camera\_sensor\_get();

&#x20; if (s != nullptr) {

&#x20;   // Reativar os automatismos que estavam comentados para melhoria em ambientes dinâmicos

&#x20;   s->set\_exposure\_ctrl(s, 1);

&#x20;   s->set\_gain\_ctrl(s, 1);

&#x20;   s->set\_whitebal(s, 1);

&#x20;   s->set\_awb\_gain(s, 1);

&#x20;   s->set\_lenc(s, 1); // Correção de lente 

&#x20; }



\#if defined(LED\_GPIO\_NUM)

&#x20; setupLedFlash();

\#endif



&#x20; // Ligar ao Wi-Fi

&#x20; WiFi.mode(WIFI\_STA);

&#x20; WiFi.setSleep(false);

&#x20; 

&#x20; if (!conectarWiFi(current\_ssid, current\_password, 15)) {

&#x20;   Serial.println("\[AVISO] Falha na rede gravada. Tentando Fallback...");

&#x20;   if (!conectarWiFi(fallback\_ssid, fallback\_password, 15)) {

&#x20;     Serial.println("\[ERRO CRÍTICO] Falha total de Wi-Fi. A reiniciar em 10 seg...");

&#x20;     delay(10000);

&#x20;     ESP.restart();

&#x20;   }

&#x20; }



&#x20; Serial.print("\[OK] Conectado! IP da ESP32-CAM: ");

&#x20; Serial.println(WiFi.localIP());



&#x20; startCameraServer();

&#x20; ultimoTempoCaptura = millis(); // Inicia o contador de fotos

}



// ============================================================

// LOOP PRINCIPAL (Automatização Não-Bloqueante)

// ============================================================

void loop() {

&#x20; 

&#x20; // 1. Tratamento da mudança de Wi-Fi pela interface

&#x20; if (testandoNovaRede \&\& (millis() - tempoInicioTesteWifi > 3000)) { // Aguarda resposta HTTP terminar

&#x20;   testandoNovaRede = false;

&#x20;   Serial.println("\\n\[REDE] Aplicando novas credenciais recebidas...");

&#x20;   

&#x20;   if (conectarWiFi(novo\_ssid\_temp, nova\_pass\_temp, 15)) {

&#x20;     Serial.println("\[REDE] Sucesso! Salvando novas configurações...");

&#x20;     preferences.begin("tcc", false);

&#x20;     preferences.putString("ssid", novo\_ssid\_temp);

&#x20;     preferences.putString("password", nova\_pass\_temp);

&#x20;     preferences.end();

&#x20;     current\_ssid = novo\_ssid\_temp;

&#x20;     current\_password = nova\_pass\_temp;

&#x20;   } else {

&#x20;     Serial.println("\[ERRO] Nova rede falhou. Revertendo para anterior...");

&#x20;     conectarWiFi(current\_ssid, current\_password, 15);

&#x20;   }

&#x20; }



&#x20; // 2. Sistema de Captura Automática

&#x20; // Verifica se o tempo decorrido atingiu o intervalo (e não está em modo offline)

&#x20; if (WiFi.status() == WL\_CONNECTED \&\& !testandoNovaRede) {

&#x20;   if (millis() - ultimoTempoCaptura >= intervaloCapturaMs) {

&#x20;     Serial.println("\\n\[TCC] Iniciando captura automatizada agendada...");

&#x20;     ultimoTempoCaptura = millis(); // Atualiza marca de tempo

&#x20;     

&#x20;     camera\_fb\_t\* fb = esp\_camera\_fb\_get();

&#x20;     if (fb) {

&#x20;       processarCapturaTCC(fb);

&#x20;       esp\_camera\_fb\_return(fb);

&#x20;     } else {

&#x20;       Serial.println("\[ERRO] Falha ao obter frame automático da câmara.");

&#x20;     }

&#x20;   }

&#x20; }



&#x20; delay(10); // Pequeno atraso para estabilidade do Watchdog

}





# APP\_HTTP.CPP



// Copyright 2015-2016 Espressif Systems (Shanghai) PTE LTD

//

// Licensed under the Apache License, Version 2.0 (the "License");

// you may not use this file except in compliance with the License.

// You may obtain a copy of the License at

//

//     http://www.apache.org/licenses/LICENSE-2.0

//

// Unless required by applicable law or agreed to in writing, software

// distributed under the License is distributed on an "AS IS" BASIS,

// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.

// See the License for the specific language governing permissions and

// limitations under the License.



\#include "Arduino.h"

\#include "esp\_http\_server.h"

\#include "esp\_timer.h"

\#include "esp\_camera.h"

\#include "img\_converters.h"

\#include "fb\_gfx.h"

\#include "esp32-hal-ledc.h"

\#include "sdkconfig.h"

\#include "camera\_index.h"

\#include "board\_config.h"



// ============================================================

// FUNÇÕES DO TCC DEFINIDAS NO ARQUIVO .ino

// ============================================================

extern bool processarCapturaTCC(camera\_fb\_t\* fb);



\#if defined(ARDUINO\_ARCH\_ESP32) \&\& defined(CONFIG\_ARDUHAL\_ESP\_LOG)

\#include "esp32-hal-log.h"

\#endif



// LED FLASH setup

\#if defined(LED\_GPIO\_NUM)

\#define CONFIG\_LED\_MAX\_INTENSITY 255

int led\_duty = 0;

bool isStreaming = false;

\#endif



typedef struct {

&#x20; httpd\_req\_t \*req;

&#x20; size\_t len;

} jpg\_chunking\_t;



\#define PART\_BOUNDARY "123456789000000000000987654321"

static const char \*\_STREAM\_CONTENT\_TYPE = "multipart/x-mixed-replace;boundary=" PART\_BOUNDARY;

static const char \*\_STREAM\_BOUNDARY = "\\r\\n--" PART\_BOUNDARY "\\r\\n";

static const char \*\_STREAM\_PART = "Content-Type: image/jpeg\\r\\nContent-Length: %u\\r\\nX-Timestamp: %d.%06d\\r\\n\\r\\n";



httpd\_handle\_t stream\_httpd = NULL;

httpd\_handle\_t camera\_httpd = NULL;



typedef struct {

&#x20; size\_t size;   //number of values used for filtering

&#x20; size\_t index;  //current value index

&#x20; size\_t count;  //value count

&#x20; int sum;

&#x20; int \*values;  //array to be filled with values

} ra\_filter\_t;



static ra\_filter\_t ra\_filter;



static ra\_filter\_t \*ra\_filter\_init(ra\_filter\_t \*filter, size\_t sample\_size) {

&#x20; memset(filter, 0, sizeof(ra\_filter\_t));



&#x20; filter->values = (int \*)malloc(sample\_size \* sizeof(int));

&#x20; if (!filter->values) {

&#x20;   return NULL;

&#x20; }

&#x20; memset(filter->values, 0, sample\_size \* sizeof(int));



&#x20; filter->size = sample\_size;

&#x20; return filter;

}



\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

static int ra\_filter\_run(ra\_filter\_t \*filter, int value) {

&#x20; if (!filter->values) {

&#x20;   return value;

&#x20; }

&#x20; filter->sum -= filter->values\[filter->index];

&#x20; filter->values\[filter->index] = value;

&#x20; filter->sum += filter->values\[filter->index];

&#x20; filter->index++;

&#x20; filter->index = filter->index % filter->size;

&#x20; if (filter->count < filter->size) {

&#x20;   filter->count++;

&#x20; }

&#x20; return filter->sum / filter->count;

}

\#endif



\#if defined(LED\_GPIO\_NUM)

void enable\_led(bool en) {  // Turn LED On or Off

&#x20; int duty = en ? led\_duty : 0;

&#x20; if (en \&\& isStreaming \&\& (led\_duty > CONFIG\_LED\_MAX\_INTENSITY)) {

&#x20;   duty = CONFIG\_LED\_MAX\_INTENSITY;

&#x20; }

&#x20; ledcWrite(LED\_GPIO\_NUM, duty);

&#x20; log\_i("Set LED intensity to %d", duty);

}

\#endif



static esp\_err\_t bmp\_handler(httpd\_req\_t \*req) {

&#x20; camera\_fb\_t \*fb = NULL;

&#x20; esp\_err\_t res = ESP\_OK;

\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20; int64\_t fr\_start = esp\_timer\_get\_time();

\#endif

&#x20; fb = esp\_camera\_fb\_get();

&#x20; if (!fb) {

&#x20;   log\_e("Camera capture failed");

&#x20;   httpd\_resp\_send\_500(req);

&#x20;   return ESP\_FAIL;

&#x20; }



&#x20; httpd\_resp\_set\_type(req, "image/x-windows-bmp");

&#x20; httpd\_resp\_set\_hdr(req, "Content-Disposition", "inline; filename=capture.bmp");

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");



&#x20; char ts\[32];

&#x20; snprintf(ts, 32, "%" PRIu32 ".%06" PRIu32, (uint32\_t)fb->timestamp.tv\_sec, (uint32\_t)fb->timestamp.tv\_usec);

&#x20; httpd\_resp\_set\_hdr(req, "X-Timestamp", (const char \*)ts);



&#x20; uint8\_t \*buf = NULL;

&#x20; size\_t buf\_len = 0;

&#x20; bool converted = frame2bmp(fb, \&buf, \&buf\_len);

&#x20; esp\_camera\_fb\_return(fb);

&#x20; if (!converted) {

&#x20;   log\_e("BMP Conversion failed");

&#x20;   httpd\_resp\_send\_500(req);

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; res = httpd\_resp\_send(req, (const char \*)buf, buf\_len);

&#x20; free(buf);

\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20; int64\_t fr\_end = esp\_timer\_get\_time();

\#endif

&#x20; log\_i("BMP: %" PRId32 "ms, %" PRIu32 "B", (int32\_t)((fr\_end - fr\_start) / 1000), (uint32\_t)buf\_len);

&#x20; return res;

}



static size\_t jpg\_encode\_stream(void \*arg, size\_t index, const void \*data, size\_t len) {

&#x20; jpg\_chunking\_t \*j = (jpg\_chunking\_t \*)arg;

&#x20; if (!index) {

&#x20;   j->len = 0;

&#x20; }

&#x20; if (httpd\_resp\_send\_chunk(j->req, (const char \*)data, len) != ESP\_OK) {

&#x20;   return 0;

&#x20; }

&#x20; j->len += len;

&#x20; return len;

}



static esp\_err\_t capture\_handler(httpd\_req\_t \*req) {

&#x20; camera\_fb\_t \*fb = NULL;

&#x20; esp\_err\_t res = ESP\_OK;



\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20; int64\_t fr\_start = esp\_timer\_get\_time();

\#endif



\#if defined(LED\_GPIO\_NUM)

&#x20; enable\_led(true);

&#x20; vTaskDelay(150 / portTICK\_PERIOD\_MS);

&#x20; fb = esp\_camera\_fb\_get();

&#x20; enable\_led(false);

\#else

&#x20; fb = esp\_camera\_fb\_get();

\#endif



&#x20; if (!fb) {

&#x20;   log\_e("Camera capture failed");

&#x20;   httpd\_resp\_send\_500(req);

&#x20;   return ESP\_FAIL;

&#x20; }



&#x20; Serial.println();

&#x20; Serial.println("================================");

&#x20; Serial.println("\[CAMERA] NOVA CAPTURA");

&#x20; Serial.println("================================");

&#x20; Serial.print("\[CAMERA] Tamanho: ");

&#x20; Serial.print(fb->len);

&#x20; Serial.println(" bytes");



&#x20; // PROCESSAMENTO DO TCC

&#x20; bool tccOK = processarCapturaTCC(fb);

&#x20; if (tccOK) {

&#x20;   Serial.println("\[TCC] Captura processada com sucesso.");

&#x20; } else {

&#x20;   Serial.println("\[TCC] Falha no processamento DHT22/Django.");

&#x20;   Serial.println("\[TCC] A imagem ainda será exibida no navegador.");

&#x20; }



&#x20; httpd\_resp\_set\_type(req, "image/jpeg");

&#x20; httpd\_resp\_set\_hdr(req, "Content-Disposition", "inline; filename=capture.jpg");

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");



&#x20; char ts\[32];

&#x20; snprintf(ts, 32, "%" PRIu32 ".%06" PRIu32, (uint32\_t)fb->timestamp.tv\_sec, (uint32\_t)fb->timestamp.tv\_usec);

&#x20; httpd\_resp\_set\_hdr(req, "X-Timestamp", (const char \*)ts);



\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20; size\_t fb\_len = 0;

\#endif



&#x20; if (fb->format == PIXFORMAT\_JPEG) {

\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20;   fb\_len = fb->len;

\#endif

&#x20;   res = httpd\_resp\_send(req, (const char \*)fb->buf, fb->len);

&#x20; } else {

&#x20;   jpg\_chunking\_t jchunk = {req, 0};

&#x20;   res = frame2jpg\_cb(fb, 80, jpg\_encode\_stream, \&jchunk) ? ESP\_OK : ESP\_FAIL;

&#x20;   httpd\_resp\_send\_chunk(req, NULL, 0);

\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20;   fb\_len = jchunk.len;

\#endif

&#x20; }



&#x20; esp\_camera\_fb\_return(fb);



\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20; int64\_t fr\_end = esp\_timer\_get\_time();

\#endif

&#x20; log\_i("JPG: %" PRIu32 "B %" PRId32 " ms", (uint32\_t)fb\_len, (int32\_t)((fr\_end - fr\_start) / 1000));

&#x20; return res;

}



static esp\_err\_t stream\_handler(httpd\_req\_t \*req) {

&#x20; camera\_fb\_t \*fb = NULL;

&#x20; struct timeval \_timestamp;

&#x20; esp\_err\_t res = ESP\_OK;

&#x20; size\_t \_jpg\_buf\_len = 0;

&#x20; uint8\_t \*\_jpg\_buf = NULL;

&#x20; char \*part\_buf\[128];



&#x20; static int64\_t last\_frame = 0;

&#x20; if (!last\_frame) {

&#x20;   last\_frame = esp\_timer\_get\_time();

&#x20; }



&#x20; res = httpd\_resp\_set\_type(req, \_STREAM\_CONTENT\_TYPE);

&#x20; if (res != ESP\_OK) {

&#x20;   return res;

&#x20; }



&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; httpd\_resp\_set\_hdr(req, "X-Framerate", "60");



\#if defined(LED\_GPIO\_NUM)

&#x20; isStreaming = true;

&#x20; enable\_led(true);

\#endif



&#x20; while (true) {

&#x20;   fb = esp\_camera\_fb\_get();

&#x20;   if (!fb) {

&#x20;     log\_e("Camera capture failed");

&#x20;     res = ESP\_FAIL;

&#x20;   } else {

&#x20;     \_timestamp.tv\_sec = fb->timestamp.tv\_sec;

&#x20;     \_timestamp.tv\_usec = fb->timestamp.tv\_usec;

&#x20;     if (fb->format != PIXFORMAT\_JPEG) {

&#x20;       bool jpeg\_converted = frame2jpg(fb, 80, \&\_jpg\_buf, \&\_jpg\_buf\_len);

&#x20;       esp\_camera\_fb\_return(fb);

&#x20;       fb = NULL;

&#x20;       if (!jpeg\_converted) {

&#x20;         log\_e("JPEG compression failed");

&#x20;         res = ESP\_FAIL;

&#x20;       }

&#x20;     } else {

&#x20;       \_jpg\_buf\_len = fb->len;

&#x20;       \_jpg\_buf = fb->buf;

&#x20;     }

&#x20;   }

&#x20;   if (res == ESP\_OK) {

&#x20;     res = httpd\_resp\_send\_chunk(req, \_STREAM\_BOUNDARY, strlen(\_STREAM\_BOUNDARY));

&#x20;   }

&#x20;   if (res == ESP\_OK) {

&#x20;     size\_t hlen = snprintf((char \*)part\_buf, 128, \_STREAM\_PART, \_jpg\_buf\_len, \_timestamp.tv\_sec, \_timestamp.tv\_usec);

&#x20;     res = httpd\_resp\_send\_chunk(req, (const char \*)part\_buf, hlen);

&#x20;   }

&#x20;   if (res == ESP\_OK) {

&#x20;     res = httpd\_resp\_send\_chunk(req, (const char \*)\_jpg\_buf, \_jpg\_buf\_len);

&#x20;   }

&#x20;   if (fb) {

&#x20;     esp\_camera\_fb\_return(fb);

&#x20;     fb = NULL;

&#x20;     \_jpg\_buf = NULL;

&#x20;   } else if (\_jpg\_buf) {

&#x20;     free(\_jpg\_buf);

&#x20;     \_jpg\_buf = NULL;

&#x20;   }

&#x20;   if (res != ESP\_OK) {

&#x20;     log\_e("Send frame failed");

&#x20;     break;

&#x20;   }

&#x20;   int64\_t fr\_end = esp\_timer\_get\_time();



&#x20;   int64\_t frame\_time = fr\_end - last\_frame;

&#x20;   last\_frame = fr\_end;



&#x20;   frame\_time /= 1000;

\#if ARDUHAL\_LOG\_LEVEL >= ARDUHAL\_LOG\_LEVEL\_INFO

&#x20;   uint32\_t avg\_frame\_time = ra\_filter\_run(\&ra\_filter, frame\_time);

\#endif

&#x20;   log\_i("MJPG: %" PRIu32 "B %" PRId32 "ms (%.1ffps)", (uint32\_t)\_jpg\_buf\_len, (int32\_t)frame\_time, 1000.0 / frame\_time);

&#x20; }



\#if defined(LED\_GPIO\_NUM)

&#x20; isStreaming = false;

&#x20; enable\_led(false);

\#endif

&#x20; return res;

}



static esp\_err\_t parse\_get(httpd\_req\_t \*req, char \*\*obuf) {

&#x20; char \*buf = NULL;

&#x20; size\_t buf\_len = 0;

&#x20; buf\_len = httpd\_req\_get\_url\_query\_len(req) + 1;

&#x20; if (buf\_len > 1) {

&#x20;   buf = (char \*)malloc(buf\_len);

&#x20;   if (!buf) {

&#x20;     httpd\_resp\_send\_500(req);

&#x20;     return ESP\_FAIL;

&#x20;   }

&#x20;   if (httpd\_req\_get\_url\_query\_str(req, buf, buf\_len) == ESP\_OK) {

&#x20;     \*obuf = buf;

&#x20;     return ESP\_OK;

&#x20;   }

&#x20;   free(buf);

&#x20; }

&#x20; httpd\_resp\_send\_404(req);

&#x20; return ESP\_FAIL;

}



static esp\_err\_t cmd\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; char variable\[32];

&#x20; char value\[32];



&#x20; if (parse\_get(req, \&buf) != ESP\_OK) {

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; if (httpd\_query\_key\_value(buf, "var", variable, sizeof(variable)) != ESP\_OK || httpd\_query\_key\_value(buf, "val", value, sizeof(value)) != ESP\_OK) {

&#x20;   free(buf);

&#x20;   httpd\_resp\_send\_404(req);

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; free(buf);



&#x20; int val = atoi(value);

&#x20; log\_i("%s = %d", variable, val);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = 0;



&#x20; if (!strcmp(variable, "framesize")) {

&#x20;   if (s->pixformat == PIXFORMAT\_JPEG) res = s->set\_framesize(s, (framesize\_t)val);

&#x20; } else if (!strcmp(variable, "quality")) res = s->set\_quality(s, val);

&#x20; else if (!strcmp(variable, "contrast")) res = s->set\_contrast(s, val);

&#x20; else if (!strcmp(variable, "brightness")) res = s->set\_brightness(s, val);

&#x20; else if (!strcmp(variable, "saturation")) res = s->set\_saturation(s, val);

&#x20; else if (!strcmp(variable, "gainceiling")) res = s->set\_gainceiling(s, (gainceiling\_t)val);

&#x20; else if (!strcmp(variable, "colorbar")) res = s->set\_colorbar(s, val);

&#x20; else if (!strcmp(variable, "awb")) res = s->set\_whitebal(s, val);

&#x20; else if (!strcmp(variable, "agc")) res = s->set\_gain\_ctrl(s, val);

&#x20; else if (!strcmp(variable, "aec")) res = s->set\_exposure\_ctrl(s, val);

&#x20; else if (!strcmp(variable, "hmirror")) res = s->set\_hmirror(s, val);

&#x20; else if (!strcmp(variable, "vflip")) res = s->set\_vflip(s, val);

&#x20; else if (!strcmp(variable, "awb\_gain")) res = s->set\_awb\_gain(s, val);

&#x20; else if (!strcmp(variable, "agc\_gain")) res = s->set\_agc\_gain(s, val);

&#x20; else if (!strcmp(variable, "aec\_value")) res = s->set\_aec\_value(s, val);

&#x20; else if (!strcmp(variable, "aec2")) res = s->set\_aec2(s, val);

&#x20; else if (!strcmp(variable, "dcw")) res = s->set\_dcw(s, val);

&#x20; else if (!strcmp(variable, "bpc")) res = s->set\_bpc(s, val);

&#x20; else if (!strcmp(variable, "wpc")) res = s->set\_wpc(s, val);

&#x20; else if (!strcmp(variable, "raw\_gma")) res = s->set\_raw\_gma(s, val);

&#x20; else if (!strcmp(variable, "lenc")) res = s->set\_lenc(s, val);

&#x20; else if (!strcmp(variable, "special\_effect")) res = s->set\_special\_effect(s, val);

&#x20; else if (!strcmp(variable, "wb\_mode")) res = s->set\_wb\_mode(s, val);

&#x20; else if (!strcmp(variable, "ae\_level")) res = s->set\_ae\_level(s, val);

\#if defined(LED\_GPIO\_NUM)

&#x20; else if (!strcmp(variable, "led\_intensity")) {

&#x20;   led\_duty = val;

&#x20;   if (isStreaming) enable\_led(true);

&#x20; }

\#endif

&#x20; else {

&#x20;   log\_i("Unknown command: %s", variable);

&#x20;   res = -1;

&#x20; }



&#x20; if (res < 0) return httpd\_resp\_send\_500(req);



&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, NULL, 0);

}



static int print\_reg(char \*p, char \*end, sensor\_t \*s, uint16\_t reg, uint32\_t mask) {

&#x20; return snprintf(p, end - p, "\\"0x%04x\\":%d,", reg, s->get\_reg(s, reg, mask));

}



static esp\_err\_t status\_handler(httpd\_req\_t \*req) {

&#x20; static char json\_response\[1024];



&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; char \*p = json\_response;

&#x20; char \*end = json\_response + sizeof(json\_response);

&#x20; \*p++ = '{';



&#x20; if (s->id.PID == OV5640\_PID || s->id.PID == OV3660\_PID) {

&#x20;   for (int reg = 0x3400; reg < 0x3406; reg += 2) p += print\_reg(p, end, s, reg, 0xFFF);

&#x20;   p += print\_reg(p, end, s, 0x3406, 0xFF);

&#x20;   p += print\_reg(p, end, s, 0x3500, 0xFFFF0);

&#x20;   p += print\_reg(p, end, s, 0x3503, 0xFF);

&#x20;   p += print\_reg(p, end, s, 0x350a, 0x3FF);

&#x20;   p += print\_reg(p, end, s, 0x350c, 0xFFFF);

&#x20;   for (int reg = 0x5480; reg <= 0x5490; reg++) p += print\_reg(p, end, s, reg, 0xFF);

&#x20;   for (int reg = 0x5380; reg <= 0x538b; reg++) p += print\_reg(p, end, s, reg, 0xFF);

&#x20;   for (int reg = 0x5580; reg < 0x558a; reg++) p += print\_reg(p, end, s, reg, 0xFF);

&#x20;   p += print\_reg(p, end, s, 0x558a, 0x1FF);

&#x20; } else if (s->id.PID == OV2640\_PID) {

&#x20;   p += print\_reg(p, end, s, 0xd3, 0xFF);

&#x20;   p += print\_reg(p, end, s, 0x111, 0xFF);

&#x20;   p += print\_reg(p, end, s, 0x132, 0xFF);

&#x20; }



&#x20; p += snprintf(p, end - p, "\\"xclk\\":%u,", s->xclk\_freq\_hz / 1000000);

&#x20; p += snprintf(p, end - p, "\\"pixformat\\":%u,", s->pixformat);

&#x20; p += snprintf(p, end - p, "\\"framesize\\":%u,", s->status.framesize);

&#x20; p += snprintf(p, end - p, "\\"quality\\":%u,", s->status.quality);

&#x20; p += snprintf(p, end - p, "\\"brightness\\":%d,", s->status.brightness);

&#x20; p += snprintf(p, end - p, "\\"contrast\\":%d,", s->status.contrast);

&#x20; p += snprintf(p, end - p, "\\"saturation\\":%d,", s->status.saturation);

&#x20; p += snprintf(p, end - p, "\\"sharpness\\":%d,", s->status.sharpness);

&#x20; p += snprintf(p, end - p, "\\"special\_effect\\":%u,", s->status.special\_effect);

&#x20; p += snprintf(p, end - p, "\\"wb\_mode\\":%u,", s->status.wb\_mode);

&#x20; p += snprintf(p, end - p, "\\"awb\\":%u,", s->status.awb);

&#x20; p += snprintf(p, end - p, "\\"awb\_gain\\":%u,", s->status.awb\_gain);

&#x20; p += snprintf(p, end - p, "\\"aec\\":%u,", s->status.aec);

&#x20; p += snprintf(p, end - p, "\\"aec2\\":%u,", s->status.aec2);

&#x20; p += snprintf(p, end - p, "\\"ae\_level\\":%d,", s->status.ae\_level);

&#x20; p += snprintf(p, end - p, "\\"aec\_value\\":%u,", s->status.aec\_value);

&#x20; p += snprintf(p, end - p, "\\"agc\\":%u,", s->status.agc);

&#x20; p += snprintf(p, end - p, "\\"agc\_gain\\":%u,", s->status.agc\_gain);

&#x20; p += snprintf(p, end - p, "\\"gainceiling\\":%u,", s->status.gainceiling);

&#x20; p += snprintf(p, end - p, "\\"bpc\\":%u,", s->status.bpc);

&#x20; p += snprintf(p, end - p, "\\"wpc\\":%u,", s->status.wpc);

&#x20; p += snprintf(p, end - p, "\\"raw\_gma\\":%u,", s->status.raw\_gma);

&#x20; p += snprintf(p, end - p, "\\"lenc\\":%u,", s->status.lenc);

&#x20; p += snprintf(p, end - p, "\\"hmirror\\":%u,", s->status.hmirror);

&#x20; p += snprintf(p, end - p, "\\"vflip\\":%u,", s->status.vflip);

&#x20; p += snprintf(p, end - p, "\\"dcw\\":%u,", s->status.dcw);

&#x20; p += snprintf(p, end - p, "\\"colorbar\\":%u", s->status.colorbar);

\#if defined(LED\_GPIO\_NUM)

&#x20; p += snprintf(p, end - p, ",\\"led\_intensity\\":%u", led\_duty);

\#else

&#x20; p += snprintf(p, end - p, ",\\"led\_intensity\\":%d", -1);

\#endif

&#x20; \*p++ = '}';

&#x20; \*p++ = 0;

&#x20; httpd\_resp\_set\_type(req, "application/json");

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, json\_response, strlen(json\_response));

}



static esp\_err\_t xclk\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; char \_xclk\[32];

&#x20; if (parse\_get(req, \&buf) != ESP\_OK) return ESP\_FAIL;

&#x20; if (httpd\_query\_key\_value(buf, "xclk", \_xclk, sizeof(\_xclk)) != ESP\_OK) {

&#x20;   free(buf);

&#x20;   httpd\_resp\_send\_404(req);

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; free(buf);

&#x20; int xclk = atoi(\_xclk);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = s->set\_xclk(s, LEDC\_TIMER\_0, xclk);

&#x20; if (res) return httpd\_resp\_send\_500(req);

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, NULL, 0);

}



static esp\_err\_t reg\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; char \_reg\[32], \_mask\[32], \_val\[32];

&#x20; if (parse\_get(req, \&buf) != ESP\_OK) return ESP\_FAIL;

&#x20; if (httpd\_query\_key\_value(buf, "reg", \_reg, sizeof(\_reg)) != ESP\_OK || 

&#x20;     httpd\_query\_key\_value(buf, "mask", \_mask, sizeof(\_mask)) != ESP\_OK || 

&#x20;     httpd\_query\_key\_value(buf, "val", \_val, sizeof(\_val)) != ESP\_OK) {

&#x20;   free(buf);

&#x20;   httpd\_resp\_send\_404(req);

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; free(buf);

&#x20; int reg = atoi(\_reg), mask = atoi(\_mask), val = atoi(\_val);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = s->set\_reg(s, reg, mask, val);

&#x20; if (res) return httpd\_resp\_send\_500(req);

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, NULL, 0);

}



static esp\_err\_t greg\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; char \_reg\[32], \_mask\[32];

&#x20; if (parse\_get(req, \&buf) != ESP\_OK) return ESP\_FAIL;

&#x20; if (httpd\_query\_key\_value(buf, "reg", \_reg, sizeof(\_reg)) != ESP\_OK || 

&#x20;     httpd\_query\_key\_value(buf, "mask", \_mask, sizeof(\_mask)) != ESP\_OK) {

&#x20;   free(buf);

&#x20;   httpd\_resp\_send\_404(req);

&#x20;   return ESP\_FAIL;

&#x20; }

&#x20; free(buf);

&#x20; int reg = atoi(\_reg), mask = atoi(\_mask);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = s->get\_reg(s, reg, mask);

&#x20; if (res < 0) return httpd\_resp\_send\_500(req);

&#x20; char buffer\[20];

&#x20; const char \*val = itoa(res, buffer, 10);

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, val, strlen(val));

}



static int parse\_get\_var(char \*buf, const char \*key, int def) {

&#x20; char \_int\[16];

&#x20; if (httpd\_query\_key\_value(buf, key, \_int, sizeof(\_int)) != ESP\_OK) return def;

&#x20; return atoi(\_int);

}



static esp\_err\_t pll\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; if (parse\_get(req, \&buf) != ESP\_OK) return ESP\_FAIL;

&#x20; int bypass = parse\_get\_var(buf, "bypass", 0);

&#x20; int mul = parse\_get\_var(buf, "mul", 0);

&#x20; int sys = parse\_get\_var(buf, "sys", 0);

&#x20; int root = parse\_get\_var(buf, "root", 0);

&#x20; int pre = parse\_get\_var(buf, "pre", 0);

&#x20; int seld5 = parse\_get\_var(buf, "seld5", 0);

&#x20; int pclken = parse\_get\_var(buf, "pclken", 0);

&#x20; int pclk = parse\_get\_var(buf, "pclk", 0);

&#x20; free(buf);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = s->set\_pll(s, bypass, mul, sys, root, pre, seld5, pclken, pclk);

&#x20; if (res) return httpd\_resp\_send\_500(req);

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, NULL, 0);

}



static esp\_err\_t win\_handler(httpd\_req\_t \*req) {

&#x20; char \*buf = NULL;

&#x20; if (parse\_get(req, \&buf) != ESP\_OK) return ESP\_FAIL;

&#x20; int startX = parse\_get\_var(buf, "sx", 0);

&#x20; int startY = parse\_get\_var(buf, "sy", 0);

&#x20; int endX = parse\_get\_var(buf, "ex", 0);

&#x20; int endY = parse\_get\_var(buf, "ey", 0);

&#x20; int offsetX = parse\_get\_var(buf, "offx", 0);

&#x20; int offsetY = parse\_get\_var(buf, "offy", 0);

&#x20; int totalX = parse\_get\_var(buf, "tx", 0);

&#x20; int totalY = parse\_get\_var(buf, "ty", 0);

&#x20; int outputX = parse\_get\_var(buf, "ox", 0);

&#x20; int outputY = parse\_get\_var(buf, "oy", 0);

&#x20; bool scale = parse\_get\_var(buf, "scale", 0) == 1;

&#x20; bool binning = parse\_get\_var(buf, "binning", 0) == 1;

&#x20; free(buf);

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; int res = s->set\_res\_raw(s, startX, startY, endX, endY, offsetX, offsetY, totalX, totalY, outputX, outputY, scale, binning);

&#x20; if (res) return httpd\_resp\_send\_500(req);

&#x20; httpd\_resp\_set\_hdr(req, "Access-Control-Allow-Origin", "\*");

&#x20; return httpd\_resp\_send(req, NULL, 0);

}



static esp\_err\_t index\_handler(httpd\_req\_t \*req) {

&#x20; httpd\_resp\_set\_type(req, "text/html");

&#x20; httpd\_resp\_set\_hdr(req, "Content-Encoding", "gzip");

&#x20; sensor\_t \*s = esp\_camera\_sensor\_get();

&#x20; if (s != NULL) {

&#x20;   if (s->id.PID == OV3660\_PID) return httpd\_resp\_send(req, (const char \*)index\_ov3660\_html\_gz, index\_ov3660\_html\_gz\_len);

&#x20;   else if (s->id.PID == OV5640\_PID) return httpd\_resp\_send(req, (const char \*)index\_ov5640\_html\_gz, index\_ov5640\_html\_gz\_len);

&#x20;   else return httpd\_resp\_send(req, (const char \*)index\_ov2640\_html\_gz, index\_ov2640\_html\_gz\_len);

&#x20; } else {

&#x20;   log\_e("Camera sensor not found");

&#x20;   return httpd\_resp\_send\_500(req);

&#x20; }

}



// ============================================================

// ADIÇÕES TCC: HANDLERS DE CONFIGURAÇÃO WEB

// ============================================================

extern void aplicarNovoIntervalo(int minutos);

extern void agendarTesteWifi(String n\_ssid, String n\_pass);



static esp\_err\_t config\_page\_handler(httpd\_req\_t \*req) {

&#x20; const char\* html = 

&#x20;   "<!DOCTYPE html><html lang=\\"pt-BR\\"><head><meta charset=\\"UTF-8\\">"

&#x20;   "<meta name=\\"viewport\\" content=\\"width=device-width, initial-scale=1.0\\">"

&#x20;   "<title>Configuração TCC</title>"

&#x20;   "<style>body{font-family: Arial; max-width: 400px; margin: 20px auto; padding: 20px; background: #f4f4f4;} "

&#x20;   "h2{color: #333;} .card{background: white; padding: 15px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 20px;} "

&#x20;   "input, button{width: 100%; padding: 10px; margin: 5px 0 15px 0; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box;} "

&#x20;   "button{background: #28a745; color: white; border: none; cursor: pointer; font-weight: bold;} "

&#x20;   "button:hover{background: #218838;}</style>"

&#x20;   "</head><body>"

&#x20;   "<h2>Gestão do Sistema - TCC</h2>"

&#x20;   

&#x20;   "<div class=\\"card\\">"

&#x20;   "<h3>Intervalo de Fotografias</h3>"

&#x20;   "<form method=\\"POST\\" action=\\"/api/interval\\">"

&#x20;   "<label>Tempo (minutos):</label>"

&#x20;   "<input type=\\"number\\" name=\\"interval\\" min=\\"1\\" max=\\"1440\\" value=\\"10\\" required>"

&#x20;   "<button type=\\"button\\" onclick=\\"document.getElementsByName('interval')\[0].value=2\\" style=\\"background:#007bff; margin-bottom:5px;\\">Modo de Testes (2 min)</button>"

&#x20;   "<button type=\\"submit\\">Guardar Intervalo</button>"

&#x20;   "</form></div>"



&#x20;   "<div class=\\"card\\">"

&#x20;   "<h3>Rede Wi-Fi</h3>"

&#x20;   "<form method=\\"POST\\" action=\\"/api/wifi\\">"

&#x20;   "<label>SSID (Nome da Rede):</label>"

&#x20;   "<input type=\\"text\\" name=\\"ssid\\" required>"

&#x20;   "<label>Senha:</label>"

&#x20;   "<input type=\\"password\\" name=\\"password\\" required>"

&#x20;   "<button type=\\"submit\\">Alterar e Conectar</button>"

&#x20;   "</form></div>"

&#x20;   "</body></html>";

&#x20;   

&#x20; httpd\_resp\_set\_type(req, "text/html");

&#x20; return httpd\_resp\_send(req, html, strlen(html));

}



static esp\_err\_t api\_interval\_handler(httpd\_req\_t \*req) {

&#x20; char buf\[100];

&#x20; int ret, remaining = req->content\_len;

&#x20; if (remaining >= sizeof(buf)) return httpd\_resp\_send\_500(req);

&#x20; ret = httpd\_req\_recv(req, buf, remaining);

&#x20; if (ret <= 0) return ESP\_FAIL;

&#x20; buf\[ret] = '\\0';



&#x20; char val\[16] = {0};

&#x20; if (httpd\_query\_key\_value(buf, "interval", val, sizeof(val)) == ESP\_OK) {

&#x20;   aplicarNovoIntervalo(atoi(val));

&#x20;   const char\* resp = "Intervalo alterado com sucesso! Regresse ao painel.";

&#x20;   httpd\_resp\_set\_type(req, "text/plain; charset=utf-8");

&#x20;   return httpd\_resp\_send(req, resp, strlen(resp));

&#x20; }

&#x20; return httpd\_resp\_send\_404(req);

}



static esp\_err\_t api\_wifi\_handler(httpd\_req\_t \*req) {

&#x20; char buf\[200];

&#x20; int ret, remaining = req->content\_len;

&#x20; if (remaining >= sizeof(buf)) return httpd\_resp\_send\_500(req);

&#x20; ret = httpd\_req\_recv(req, buf, remaining);

&#x20; if (ret <= 0) return ESP\_FAIL;

&#x20; buf\[ret] = '\\0';



&#x20; char ssid\[64] = {0};

&#x20; char pass\[64] = {0};

&#x20; if (httpd\_query\_key\_value(buf, "ssid", ssid, sizeof(ssid)) == ESP\_OK \&\&

&#x20;     httpd\_query\_key\_value(buf, "password", pass, sizeof(pass)) == ESP\_OK) {

&#x20;   

&#x20;   agendarTesteWifi(String(ssid), String(pass));

&#x20;   

&#x20;   const char\* resp = "Credenciais recebidas. A camera vai tentar conectar. Se falhar, revertera para a anterior em 15 segundos.";

&#x20;   httpd\_resp\_set\_type(req, "text/plain; charset=utf-8");

&#x20;   return httpd\_resp\_send(req, resp, strlen(resp));

&#x20; }

&#x20; return httpd\_resp\_send\_404(req);

}



// ============================================================

// SERVIDOR WEB (ORIGINAL + TCC)

// ============================================================

void startCameraServer() {

&#x20; httpd\_config\_t config = HTTPD\_DEFAULT\_CONFIG();

&#x20; config.max\_uri\_handlers = 20;



&#x20; httpd\_uri\_t index\_uri = { .uri = "/", .method = HTTP\_GET, .handler = index\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t status\_uri = { .uri = "/status", .method = HTTP\_GET, .handler = status\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t cmd\_uri = { .uri = "/control", .method = HTTP\_GET, .handler = cmd\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t capture\_uri = { .uri = "/capture", .method = HTTP\_GET, .handler = capture\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t stream\_uri = { .uri = "/stream", .method = HTTP\_GET, .handler = stream\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t bmp\_uri = { .uri = "/bmp", .method = HTTP\_GET, .handler = bmp\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t xclk\_uri = { .uri = "/xclk", .method = HTTP\_GET, .handler = xclk\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t reg\_uri = { .uri = "/reg", .method = HTTP\_GET, .handler = reg\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t greg\_uri = { .uri = "/greg", .method = HTTP\_GET, .handler = greg\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t pll\_uri = { .uri = "/pll", .method = HTTP\_GET, .handler = pll\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t win\_uri = { .uri = "/resolution", .method = HTTP\_GET, .handler = win\_handler, .user\_ctx = NULL };



&#x20; // URIs DO TCC

&#x20; httpd\_uri\_t config\_uri = { .uri = "/config", .method = HTTP\_GET, .handler = config\_page\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t interval\_uri = { .uri = "/api/interval", .method = HTTP\_POST, .handler = api\_interval\_handler, .user\_ctx = NULL };

&#x20; httpd\_uri\_t wifi\_uri = { .uri = "/api/wifi", .method = HTTP\_POST, .handler = api\_wifi\_handler, .user\_ctx = NULL };



&#x20; ra\_filter\_init(\&ra\_filter, 20);



&#x20; log\_i("Starting web server on port: '%u'", config.server\_port);

&#x20; if (httpd\_start(\&camera\_httpd, \&config) == ESP\_OK) {

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&index\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&cmd\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&status\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&capture\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&bmp\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&xclk\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&reg\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&greg\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&pll\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&win\_uri);

&#x20;   

&#x20;   // REGISTRO DAS URIs DO TCC

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&config\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&interval\_uri);

&#x20;   httpd\_register\_uri\_handler(camera\_httpd, \&wifi\_uri);

&#x20; }



&#x20; config.server\_port += 1;

&#x20; config.ctrl\_port += 1;

&#x20; log\_i("Starting stream server on port: '%u'", config.server\_port);

&#x20; if (httpd\_start(\&stream\_httpd, \&config) == ESP\_OK) {

&#x20;   httpd\_register\_uri\_handler(stream\_httpd, \&stream\_uri);

&#x20; }

}



void setupLedFlash() {

\#if defined(LED\_GPIO\_NUM)

&#x20; ledcAttach(LED\_GPIO\_NUM, 5000, 8);

\#else

&#x20; log\_i("LED flash is disabled -> LED\_GPIO\_NUM undefined");

\#endif

}

