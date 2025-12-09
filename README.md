KELOMPOK	
Nama Anggota	: 1. Icha Agustin		(0903
		          2. Marshanda
		          3. Atiya Melvina
		          4. Ivan Saputra
		          5. Citra Putri Firdawati
Kelas		: TK5A
Mata Kuliah	: Keamanan Komputer
_____________________________________________________________________________________

KOTAK SAMPAH PINTAR DENGAN SERANGAN DOS

Deskripsi Proyek
Proyek ini mengembangkan kotak sampah otomatis berbasis sensor, kemudian melakukan simulasi serangan DoS untuk melihat bagaimana perangkat IoT bereaksi terhadap beban trafik yang tinggi. Seluruh proses pengujian dilakukan secara legal dalam lingkungan laboratorium dan jaringan tertutup, sehingga aman untuk penelitian.

1. Kotak Sampah Pintar
Pada bagian ini dibuat sebuah perangkat kotak sampah yang dapat membuka dan menutup secara otomatis menggunakan sensor jarak (ultrasonic). Sistem ini dirancang agar lebih higienis dan memudahkan pengguna

2. Pengujiuan Serangan ( DoS )
Sebagai bagian dari analisis keamanan, dilakukan pengujian berupa simulasi gangguan lalu lintas jaringan. Tujuan dari pengujian ini adalah untuk melihat bagaimana sistem bereaksi ketika menerima trafik berlebih.


Cara Instalasi serangan DoS menggunakan tools hping3 

- Pertama kami menginstall hping3 menggunakan command : sudo apt install hping3 -y
- Kemudian setelah di install kami cek version menggunakan command : hping3 --version 

Cara Menjalankan Serangan DoS Tools hping3 menggunakan command : hping3 -c 20000 -d 120 -S -w 64 -p 80 --flood --rand-source [IP]

3. dampak serangan doS
- Wifi Lost
- Web Down
- Sensor tidak terdeteksi

4. mitigasi serangan Dos, menggunakan ttols evillimiter sebagai alat untuk memblokir/membatasi bandwitch trafic

5. # code program

   #include <Servo.h>
#include <ESP8266WiFi.h>
#include <WiFiClientSecureBearSSL.h>
#include <UniversalTelegramBot.h>
#include <ESP8266WebServer.h>

// ==== WEB MONITOR ====
ESP8266WebServer server(80);
String serialBuffer = "";

// Interval baca sensor (lebih cepat biar halus)
unsigned long sensorReadInterval = 200;   // 0.2 detik
unsigned long lastSensorRead = 0;

// Interval cek Telegram
unsigned long botMessageInterval = 1000;
unsigned long lastBotUpdate = 0;

// === WiFi & Telegram ===
const char* ssid     = "ABCgenggong";
const char* password = "matchalatte2";
const char* botToken = "8558012574:AAHspoZrOn9pI2_iOLBG71Owqf9FSzEJDQE";
int64_t chatID       = 8520396645;

BearSSL::WiFiClientSecure client;
UniversalTelegramBot bot(botToken, client);

// === Sensor Ultrasonik ===
#define trigPin 14   // D5
#define echoPin 12   // D6
int distanceCM = 0;
const int MAX_DISTANCE_CM = 400;

// === Servo ===
Servo servo;
#define servoPin 13  // D7
const int THRESHOLD_CM = 25;   // <= 25 cm → buka

// === DoS Detector (berbasis hit web) ===
unsigned long totalRequests      = 0;
unsigned long lastWindowRequests = 0;
unsigned long lastWindowTime     = 0;

bool dosDetected          = false;
unsigned long dosStartTime       = 0;
unsigned long lastLowTrafficTime = 0;

const unsigned long DOS_THRESHOLD_RPS = 10;    // >10 req/s → DoS
const unsigned long DOS_COOLDOWN      = 3000;  // 3 detik normal → stop

// ========== TAMBAHAN: ADVANCED DOS DETECTION ==========
// Deteksi berdasarkan LOOP FREQUENCY (CPU overload detection)
unsigned long loopCounter = 0;
unsigned long lastLoopCheck = 0;
unsigned long normalLoopFreq = 0;
int consecutiveSlowLoops = 0;
int consecutiveNormalLoops = 0;

// Threshold detection
#define NORMAL_LOOP_THRESHOLD 3000    // Normal: >3000 loops/sec
#define ATTACK_LOOP_THRESHOLD 800     // Attack: <800 loops/sec
#define CONSECUTIVE_CHECKS 3          // Perlu 3x berturut untuk confirm

// WiFi monitoring
bool wifiWasConnected = true;
bool attackConfirmedByLoop = false;
bool attackConfirmedByRPS = false;
bool attackConfirmedByWiFi = false;   // <<< FIX: flag serangan via WiFi

// Visual & logging
String getTimeString() {
  unsigned long seconds = millis() / 1000;
  unsigned long minutes = seconds / 60;
  unsigned long hours = minutes / 60;
  seconds = seconds % 60;
  minutes = minutes % 60;
  
  String timeStr = "";
  if (hours < 10) timeStr += "0";
  timeStr += String(hours) + ":";
  if (minutes < 10) timeStr += "0";
  timeStr += String(minutes) + ":";
  if (seconds < 10) timeStr += "0";
  timeStr += String(seconds);
  
  return timeStr;
}

void addToSerialBuffer(String msg) {
  String timestamped = "[" + getTimeString() + "] " + msg;
  Serial.println(msg);  // Tetap ke Serial Monitor
  serialBuffer += timestamped + "\n";
  
  // Batasi buffer
  if (serialBuffer.length() > 5000) {
    serialBuffer = serialBuffer.substring(2000);
  }
}

// ===================================================================
// ================ FUNGSI DOS DETECTION YANG LEBIH AKURAT ==========
// ===================================================================

void notifyAttackDetected(String detectionMethod); // forward
void notifyAttackStopped();                        // forward

void checkSystemPerformance(unsigned long loops) {
  // Deteksi berdasarkan LOOP FREQUENCY
  if (loops < ATTACK_LOOP_THRESHOLD) {
    consecutiveSlowLoops++;
    consecutiveNormalLoops = 0;
    
    Serial.print("[Performance] Loops: ");
    Serial.print(loops);
    Serial.print(" | Status: ⚠ SLOW (");
    Serial.print(consecutiveSlowLoops);
    Serial.println("/3)");
    
    // Confirm attack setelah 3x consecutive slow
    if (consecutiveSlowLoops >= CONSECUTIVE_CHECKS && !attackConfirmedByLoop) {
      attackConfirmedByLoop = true;
      
      if (!dosDetected) {
        dosDetected = true;
        dosStartTime = millis();
        notifyAttackDetected("CPU Overload");
      }
    }
    
  } else if (loops > NORMAL_LOOP_THRESHOLD) {
    consecutiveNormalLoops++;
    consecutiveSlowLoops = 0;
    
    Serial.print("[Performance] Loops: ");
    Serial.print(loops);
    Serial.println(" | Status: ✓ Normal");
    
    // Confirm recovery setelah 3x consecutive normal
    if (consecutiveNormalLoops >= CONSECUTIVE_CHECKS && attackConfirmedByLoop) {
      attackConfirmedByLoop = false;
      
      // Jika tidak ada attack lain yang aktif
      if (dosDetected && !attackConfirmedByRPS && !attackConfirmedByWiFi) {
        notifyAttackStopped();
      }
    }
    
  } else {
    Serial.print("[Performance] Loops: ");
    Serial.print(loops);
    Serial.println(" | Status: ⚠ Degraded");
  }
}

void monitorWiFiConnection() {
  bool currentlyConnected = (WiFi.status() == WL_CONNECTED);
  
  // Detect WiFi disconnection
  if (wifiWasConnected && !currentlyConnected) {
    addToSerialBuffer("⚠ WiFi CONNECTION LOST - Possible Attack!");
    
    attackConfirmedByWiFi = true;   // <<< tandai attack via WiFi
    
    if (!dosDetected) {
      dosDetected = true;
      dosStartTime = millis();
      notifyAttackDetected("WiFi Disconnection");
    }
  }
  
  // Detect WiFi reconnection
  if (!wifiWasConnected && currentlyConnected) {
    addToSerialBuffer("✓ WiFi RECONNECTED");
    
    // Kalau serangan hanya dari WiFi (RPS & Loop sudah tidak aktif)
    if (attackConfirmedByWiFi) {
      attackConfirmedByWiFi = false;
      
      if (dosDetected && !attackConfirmedByLoop && !attackConfirmedByRPS) {
        notifyAttackStopped();      // <<< sekarang serangan bisa “berhenti”
      }
    }
  }
  
  wifiWasConnected = currentlyConnected;
}

void notifyAttackDetected(String detectionMethod) {
  // ========== NOTIFIKASI KE SERIAL MONITOR ==========
  Serial.println("\n╔════════════════════════════════════════╗");
  Serial.println("║   🚨 DOS ATTACK DETECTED! 🚨          ║");
  Serial.println("╚════════════════════════════════════════╝");
  Serial.print("Time: ");
  Serial.println(getTimeString());
  Serial.print("Method: ");
  Serial.println(detectionMethod);
  Serial.println("Status: SYSTEM UNDER ATTACK");
  Serial.println("════════════════════════════════════════\n");
  
  // ========== NOTIFIKASI KE WEB SERIAL ==========
  serialBuffer += "\n";
  serialBuffer += "╔════════════════════════════════════════╗\n";
  serialBuffer += "║   🚨 DOS ATTACK DETECTED! 🚨          ║\n";
  serialBuffer += "╚════════════════════════════════════════╝\n";
  serialBuffer += "[" + getTimeString() + "] Detection: " + detectionMethod + "\n";
  serialBuffer += "[" + getTimeString() + "] Status: UNDER ATTACK\n";
  serialBuffer += "[" + getTimeString() + "] Performance: CRITICAL\n";
  serialBuffer += "════════════════════════════════════════\n\n";
  
  // ========== NOTIFIKASI KE TELEGRAM ==========
  if (WiFi.status() == WL_CONNECTED) {
    String telegramMsg = "🚨 DOS ATTACK DETECTED!\n\n";
    telegramMsg += "⏰ Time: " + getTimeString() + "\n";
    telegramMsg += "🎯 Method: " + detectionMethod + "\n";
    telegramMsg += "📡 IP: " + WiFi.localIP().toString() + "\n";
    telegramMsg += "💀 Status: System Under Attack\n";
    telegramMsg += "⚠ Performance: CRITICAL\n\n";
    telegramMsg += "Defense systems monitoring...";
    
    bot.sendMessage(String(chatID), telegramMsg, "Markdown");
  }
  
  // Visual: blink servo sedikit sebagai indicator
  int currentPos = servo.read();
  servo.write(currentPos + 10);
  delay(100);
  servo.write(currentPos);
}

void notifyAttackStopped() {
  unsigned long attackDuration = (millis() - dosStartTime) / 1000;
  dosDetected = false;
  
  // ========== NOTIFIKASI KE SERIAL MONITOR ==========
  Serial.println("\n╔════════════════════════════════════════╗");
  Serial.println("║   ✅ ATTACK STOPPED - RECOVERED       ║");
  Serial.println("╚════════════════════════════════════════╝");
  Serial.print("Time: ");
  Serial.println(getTimeString());
  Serial.print("Duration: ");
  Serial.print(attackDuration);
  Serial.println(" seconds");
  Serial.println("Status: SYSTEM RECOVERED");
  Serial.println("Performance: BACK TO NORMAL");
  Serial.println("════════════════════════════════════════\n");
  
  // ========== NOTIFIKASI KE WEB SERIAL ==========
  serialBuffer += "\n";
  serialBuffer += "╔════════════════════════════════════════╗\n";
  serialBuffer += "║   ✅ ATTACK STOPPED - RECOVERED       ║\n";
  serialBuffer += "╚════════════════════════════════════════╝\n";
  serialBuffer += "[" + getTimeString() + "] Duration: " + String(attackDuration) + "s\n";
  serialBuffer += "[" + getTimeString() + "] Status: RECOVERED\n";
  serialBuffer += "[" + getTimeString() + "] Performance: NORMAL\n";
  serialBuffer += "════════════════════════════════════════\n\n";
  
  // ========== NOTIFIKASI KE TELEGRAM ==========
  if (WiFi.status() == WL_CONNECTED) {
    String telegramMsg = "✅ ATTACK STOPPED\n\n";
    telegramMsg += "⏰ Ended: " + getTimeString() + "\n";
    telegramMsg += "⏱ Duration: " + String(attackDuration) + " seconds\n";
    telegramMsg += "🔄 Status: System Recovered\n";
    telegramMsg += "💚 Performance: Normal\n";
    telegramMsg += "📡 IP: " + WiFi.localIP().toString() + "\n\n";
    telegramMsg += "All systems operational";
    
    bot.sendMessage(String(chatID), telegramMsg, "Markdown");
  }
  
  // Reset counters & flags
  consecutiveSlowLoops = 0;
  consecutiveNormalLoops = 0;
  attackConfirmedByLoop = false;
  attackConfirmedByRPS = false;
  attackConfirmedByWiFi = false;
  lastLowTrafficTime = 0;
  dosStartTime = 0;
}

// ===================================================================
// ======================== FUNGSI BACA JARAK ========================
// ===================================================================

int bacaJarak() {
  long total = 0;
  long maxDuration = MAX_DISTANCE_CM * 58;

  for (int i = 0; i < 3; i++) {
    digitalWrite(trigPin, LOW);
    delayMicroseconds(2);
    digitalWrite(trigPin, HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPin, LOW);

    total += pulseIn(echoPin, HIGH, maxDuration);
    delay(10);
  }

  long duration = total / 3;
  if (duration == 0) return -1;
  return (int)(duration * 0.034 / 2.0);   // cm
}

// ===================================================================
// ========================= HALAMAN WEB =============================
// ===================================================================

// Root: langsung tampilkan serial log
void handleRoot() {
  handleMonitor();
}

// /monitor: mirip Serial Monitor Arduino IDE dengan STATUS DoS
void handleMonitor() {
  totalRequests++;

  String html = "<!DOCTYPE html><html><head><meta charset='UTF-8'>";
  html += "<meta http-equiv='refresh' content='0.5'>"; // refresh tiap 0.5 detik
  html += "<title>ESP8266 Serial Log</title>";
  
  // ========== TAMBAHAN: CSS untuk visual DoS ==========
  html += "<style>";
  html += "body{font-family:monospace;background:#000;color:#0f0;padding:20px;margin:0;}";
  html += ".header{background:#111;padding:15px;border:2px solid #0f0;margin-bottom:20px;}";
  html += ".status-normal{color:#0f0;font-size:20px;font-weight:bold;}";
  html += ".status-attack{color:#f00;font-size:24px;font-weight:bold;animation:blink 0.5s infinite;}";
  html += "@keyframes blink{0%,50%{opacity:1}51%,100%{opacity:0.3}}";
  html += "pre{background:#000;color:#0f0;padding:15px;border:1px solid #0f0;height:400px;overflow-y:scroll;font-size:14px;}";
  html += ".info{background:#111;padding:10px;border:1px solid #0f0;margin:10px 0;}";
  html += "</style>";
  
  html += "</head><body>";
  
  // ========== TAMBAHAN: Header dengan Status ==========
  html += "<div class='header'>";
  html += "<h1>🛡 ESP8266 Serial Monitor</h1>";
  html += "<div class='" + String(dosDetected ? "status-attack" : "status-normal") + "'>";
  html += dosDetected ? "🚨 SYSTEM UNDER DOS ATTACK!" : "✅ SYSTEM NORMAL";
  html += "</div>";
  html += "</div>";
  
  // ========== TAMBAHAN: Info Panel ==========
  html += "<div class='info'>";
  html += "<b>📊 System Info:</b><br>";
  html += "Distance: " + String(distanceCM) + " cm | ";
  html += "Servo: " + String(distanceCM <= THRESHOLD_CM ? "OPEN" : "CLOSED") + " | ";
  html += "Uptime: " + String(millis()/1000) + "s | ";
  html += "Loops: " + String(loopCounter) + "/s";
  html += "</div>";
  
  if (dosDetected) {
    unsigned long attackTime = (millis() - dosStartTime) / 1000;
    html += "<div class='info status-attack'>";
    html += "⚠ <b>Attack Duration: " + String(attackTime) + " seconds</b>";
    html += "</div>";
  }
  
  html += "<h2>📜 Serial Log:</h2>";
  html += "<pre>";
  html += serialBuffer;
  html += "</pre>";
  html += "<p style='color:#0f0;'><i>Auto-refresh every 0.5s</i></p>";
  html += "</body></html>";

  server.send(200, "text/html", html);
}

void handleNotFound() {
  totalRequests++;
  server.send(404, "text/plain", "404 Not Found");
}

// ===================================================================
// ======================== TELEGRAM BOT =============================
// ===================================================================

void handleNewMessages(int num) {
  for (int i = 0; i < num; i++) {
    String text    = bot.messages[i].text;
    int64_t fromID = atoll(bot.messages[i].chat_id.c_str());

    if (fromID != chatID) {
      bot.sendMessage(String(fromID), "❌ Unauthorized user", "");
      continue;
    }

    if (text == "/status") {
      String reply = "🗑 Smart Trash Status\n\n";
      reply += "📏 Distance: " + String(distanceCM) + " cm\n";
      reply += "🚪 Servo: " + String(distanceCM <= THRESHOLD_CM ? "OPEN" : "CLOSED") + "\n";
      reply += "🛡 DoS: " + String(dosDetected ? "⚠ UNDER ATTACK" : "✅ Normal") + "\n";
      reply += "🔢 Loop Freq: " + String(loopCounter) + "/s\n";
      reply += "📡 IP: " + WiFi.localIP().toString() + "\n";
      reply += "⏱ Uptime: " + String(millis()/1000) + "s";
      
      if (dosDetected && dosStartTime != 0) {
        unsigned long dur = (millis() - dosStartTime) / 1000;
        reply += "\n⚠ Attack Duration: " + String(dur) + "s";
      }
      
      bot.sendMessage(String(chatID), reply, "Markdown");
    }
  }
}

// ===================================================================
// ============================== SETUP ==============================
// ===================================================================

void setup() {
  Serial.begin(115200);
  delay(200);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  servo.attach(servoPin);
  servo.write(0);

  Serial.println("=======================================");
  Serial.println(" SMART TRASH MONITOR + DOS DETECTOR");
  Serial.println("=======================================\n");

  Serial.print("Connecting WiFi");

  WiFi.begin(ssid, password);
  client.setInsecure();

  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }

  Serial.println("\nWiFi Connected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  // 🔔 Notifikasi startup
  String msg = "🟢 Smart Trash Online!\n\n";
  msg += "📡 IP: " + WiFi.localIP().toString() + "\n";
  msg += "🌐 Web: http://" + WiFi.localIP().toString() + "\n";
  msg += "🛡 DoS Detection: Active\n\n";
  msg += "System ready for monitoring";
  bot.sendMessage(String(chatID), msg, "Markdown");

  // Web server routes
  server.on("/",        handleRoot);
  server.on("/monitor", handleMonitor);
  server.onNotFound(    handleNotFound);
  server.begin();

  lastWindowTime = millis();
  lastSensorRead = millis();
  lastLoopCheck = millis();
  
  // ========== TAMBAHAN: Calibrate normal loop frequency ==========
  Serial.println("\n[System] Calibrating loop frequency...");
  delay(1000);
  unsigned long calibStart = millis();
  unsigned long calibCount = 0;
  while (millis() - calibStart < 1000) {
    calibCount++;
    ESP.wdtFeed();
  }
  normalLoopFreq = calibCount;
  Serial.print("[System] Normal frequency: ");
  Serial.print(normalLoopFreq);
  Serial.println(" loops/sec");
  Serial.println("[System] DoS Detection: ACTIVE\n");
  
  addToSerialBuffer("✓ System initialized - DoS detection active");
}

// ===================================================================
// =============================== LOOP ==============================
// ===================================================================

void loop() {
  // ========== TAMBAHAN: Count loops untuk performance monitoring ==========
  loopCounter++;
  
  if (WiFi.status() == WL_CONNECTED) {
    server.handleClient();
  }

  unsigned long now = millis();

  // ========== TAMBAHAN: Check LOOP FREQUENCY setiap 1 detik ==========
  if (now - lastLoopCheck >= 1000) {
    checkSystemPerformance(loopCounter);
    monitorWiFiConnection();
    
    loopCounter = 0;
    lastLoopCheck = now;
  }

  // ---------- DoS DETECTION (Request-based) ----------
  if (now - lastWindowTime >= 1000) {
    unsigned long reqNow = totalRequests - lastWindowRequests;
    lastWindowRequests   = totalRequests;
    lastWindowTime       = now;

    if (reqNow > DOS_THRESHOLD_RPS) {
      lastLowTrafficTime = 0;
      attackConfirmedByRPS = true;

      if (!dosDetected) {
        dosDetected  = true;
        dosStartTime = now;
        notifyAttackDetected("High Request Rate (" + String(reqNow) + " RPS)");
      }
    } else {
      if (attackConfirmedByRPS) {
        if (lastLowTrafficTime == 0) lastLowTrafficTime = now;

        if (now - lastLowTrafficTime >= DOS_COOLDOWN) {
          attackConfirmedByRPS = false;
          
          // Jika tidak ada attack lain yang aktif
          if (dosDetected && !attackConfirmedByLoop && !attackConfirmedByWiFi) {
            notifyAttackStopped();
          }
        }
      }
    }
  }

  // ---------- SENSOR & SERVO ----------
  if (now - lastSensorRead > sensorReadInterval) {
    lastSensorRead = now;

    distanceCM = bacaJarak();

    if (distanceCM != -1) {
      // Tampilkan seperti di Serial Monitor
      Serial.print("Jarak: ");
      Serial.print(distanceCM);
      Serial.println(" cm");

      if (distanceCM <= THRESHOLD_CM && distanceCM > 0) {
        servo.write(180);
        Serial.println("Servo: BUKA");
      } else {
        servo.write(0);
        Serial.println("Servo: TUTUP");
      }

      Serial.println("------------------");

      // Di web kita log minimal
      serialBuffer += "Jarak: " + String(distanceCM) + " cm\n";
      if (serialBuffer.length() > 5000) {
        serialBuffer = serialBuffer.substring(2000);
      }

    } else {
      Serial.println("❌ Sensor Error!");
      serialBuffer += "❌ Sensor Error!\n";
      if (serialBuffer.length() > 5000) {
        serialBuffer = serialBuffer.substring(2000);
      }
    }
  }

  // ---------- TELEGRAM POLLING ----------
  if (WiFi.status() == WL_CONNECTED) {
    if (now - lastBotUpdate > botMessageInterval) {
      lastBotUpdate = now;

      int num = bot.getUpdates(bot.last_message_received + 1);
      while (num) {
        handleNewMessages(num);
        num = bot.getUpdates(bot.last_message_received + 1);
      }
    }
  }
  
  // ========== TAMBAHAN: Feed watchdog ==========
  ESP.wdtFeed();
}






