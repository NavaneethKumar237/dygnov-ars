V1
code 

```

#include <SPI.h>
#include <TFT_eSPI.h>

// ============================================================
// DYGNOVA ARS - MULTI PAGE TOUCH HMI
// ESP32 + 3.5" SPI TFT 480x320 + HR2046 TOUCH
//
// PAGE 1: WELCOME
// PAGE 2: MODE SELECTION
// PAGE 3: CYCLE CONFIGURATION
// PAGE 4: REVIEW
// PAGE 5: RUNNING (SIMULATION)
// PAGE 6: COMPLETE
//
// UI SIMULATION ONLY - NO HEATER / PUMP CONTROL
// ============================================================

TFT_eSPI tft = TFT_eSPI();

// -------------------- HARDWARE --------------------
#define TFT_CS_PIN  5
#define TOUCH_CS   22
#define TOUCH_IRQ  27

#define SPI_CLK   18
#define SPI_MOSI  23
#define SPI_MISO  19

// -------------------- COLORS ---------------------
#define BG        0x0841
#define HEADER    0x10A2
#define PANEL    0x18E3
#define PANEL2   0x2124
#define WHITE    0xFFFF
#define GREY    0xBDF7
#define BLUE     0x249F
#define CYAN     0x07FF
#define GREEN    0x07E0
#define RED      0xF800
#define ORANGE  0xFD20
#define BLACK    0x0000
#define DARKBLUE 0x0320

// ============================================================
// TOUCH CALIBRATION - YOUR PREVIOUS VALUES
// ============================================================

#define RAW_LEFT    3610
#define RAW_RIGHT   390

#define RAW_TOP    3450
#define RAW_BOTTOM  390

// ============================================================
// PAGE SYSTEM
// ============================================================

enum Page {
  PAGE_WELCOME,
  PAGE_MODE,
  PAGE_CYCLE,
  PAGE_REVIEW,
  PAGE_RUNNING,
  PAGE_COMPLETE
};

Page currentPage = PAGE_WELCOME;

// ============================================================
// USER SETTINGS
// ============================================================

enum Mode {
  MODE_DEFAULT,
  MODE_CUSTOM
};

Mode selectedMode = MODE_DEFAULT;

const int MAX_CYCLES = 5;

int selectedCycles = 1;
int currentSetupCycle = 0;
int currentRunningCycle = 0;

int cycleTemp[MAX_CYCLES] = {
  121, 121, 121, 121, 121
};

int cycleTime[MAX_CYCLES] = {
  20, 20, 20, 20, 20
};

int completedCycles = 0;
int remainingSeconds = 0;

// ============================================================
// PROCESS STATES
// ============================================================

bool processRunning = false;
bool processStopped = false;

String statusText = "IDLE";

// ============================================================
// TIMING
// ============================================================

unsigned long lastTouchTime = 0;
unsigned long lastTimerTick = 0;
unsigned long lastAnimationTime = 0;

const unsigned long TOUCH_DELAY = 250;
const unsigned long TIMER_INTERVAL = 1000;
const unsigned long ANIMATION_INTERVAL = 35;

// ============================================================
// ANIMATION
// ============================================================

int logoAngle = 0;
int logoPulse = 0;
bool pulseDirection = true;

// ============================================================
// TOUCH FUNCTIONS
// ============================================================

uint16_t readTouchRaw(uint8_t command) {

  digitalWrite(TFT_CS_PIN, HIGH);

  SPI.beginTransaction(
    SPISettings(500000, MSBFIRST, SPI_MODE0)
  );

  digitalWrite(TOUCH_CS, LOW);

  SPI.transfer(command);
  delayMicroseconds(10);

  uint16_t value = SPI.transfer16(0x0000);

  digitalWrite(TOUCH_CS, HIGH);

  SPI.endTransaction();

  return (value >> 3) & 0x0FFF;
}

uint16_t readTouchAverage(uint8_t command) {

  long total = 0;

  for (int i = 0; i < 5; i++) {
    total += readTouchRaw(command);
  }

  return total / 5;
}

bool getTouch(int &screenX, int &screenY) {

  if (digitalRead(TOUCH_IRQ) == HIGH) {
    return false;
  }

  uint16_t rawX = readTouchAverage(0xD0);
  uint16_t rawY = readTouchAverage(0x90);

  screenX = map(rawY, RAW_LEFT, RAW_RIGHT, 30, 450);
  screenY = map(rawX, RAW_TOP, RAW_BOTTOM, 30, 290);

  screenX = constrain(screenX, 0, 479);
  screenY = constrain(screenY, 0, 319);

  return true;
}

void waitTouchRelease() {

  while (digitalRead(TOUCH_IRQ) == LOW) {
    delay(10);
  }

  delay(50);
}

// ============================================================
// GRAPHIC HELPERS
// ============================================================

void centerText(
  String text,
  int x,
  int y,
  int size,
  uint16_t color
) {

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(color, BG);
  tft.setTextSize(size);
  tft.drawString(text, x, y);
}

void drawButton(
  int x,
  int y,
  int w,
  int h,
  String label,
  uint16_t color,
  int textSize = 2
) {

  tft.fillRoundRect(
    x, y, w, h, 8, color
  );

  tft.drawRoundRect(
    x, y, w, h, 8, WHITE
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, color);
  tft.setTextSize(textSize);

  tft.drawString(
    label,
    x + w / 2,
    y + h / 2
  );
}

void drawPanel(
  int x,
  int y,
  int w,
  int h,
  String title
) {

  tft.fillRoundRect(
    x, y, w, h, 8, PANEL
  );

  tft.drawRoundRect(
    x, y, w, h, 8, BLUE
  );

  tft.setTextDatum(TL_DATUM);
  tft.setTextColor(CYAN, PANEL);
  tft.setTextSize(2);

  tft.drawString(
    title,
    x + 10,
    y + 8
  );
}

void drawHeader(String title) {

  tft.fillRect(0, 0, 480, 45, HEADER);

  tft.setTextDatum(TL_DATUM);
  tft.setTextColor(CYAN, HEADER);
  tft.setTextSize(2);

  tft.drawString("DYGNOVA ARS", 12, 12);

  tft.setTextDatum(TR_DATUM);
  tft.setTextColor(WHITE, HEADER);
  tft.drawString(title, 468, 12);
}

void drawFooter(String text) {

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(GREY, BG);
  tft.setTextSize(1);

  tft.drawString(text, 240, 315);
}

// ============================================================
// PAGE 1 - WELCOME
// ============================================================

void drawWelcomeBase() {

  tft.fillScreen(BG);

  tft.fillRect(0, 0, 480, 5, CYAN);
  tft.fillRect(0, 315, 480, 5, CYAN);

  // Logo core
  tft.fillCircle(240, 100, 42, BLUE);
  tft.drawCircle(240, 100, 48, CYAN);
  tft.drawCircle(240, 100, 55, BLUE);

  tft.fillCircle(240, 100, 30, BG);

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, BG);
  tft.setTextSize(4);
  tft.drawString("D", 240, 100);

  // Brand
  tft.setTextColor(WHITE, BG);
  tft.setTextSize(4);
  tft.drawString("DYGNOVA ARS", 240, 170);

  tft.setTextColor(CYAN, BG);
  tft.setTextSize(2);
  tft.drawString(
    "ANTIGEN RETRIEVAL SYSTEM",
    240, 205
  );

  tft.setTextColor(GREY, BG);
  tft.setTextSize(1);
  tft.drawString(
    "SOFTWARE VERSION 1.0.0",
    240, 228
  );

  drawButton(
    140, 250, 200, 55,
    "START", BLUE, 3
  );

  tft.setTextColor(GREY, BG);
  tft.setTextSize(1);
  tft.drawString(
    "INNOMAYI TECHNOLOGIES",
    240, 305
  );
}

void drawWelcomeAnimation() {

  if (millis() - lastAnimationTime < ANIMATION_INTERVAL) {
    return;
  }

  lastAnimationTime = millis();

  logoAngle += 8;

  if (logoAngle >= 360) {
    logoAngle = 0;
  }

  if (pulseDirection) {
    logoPulse++;
    if (logoPulse >= 8) pulseDirection = false;
  }
  else {
    logoPulse--;
    if (logoPulse <= 0) pulseDirection = true;
  }

  // Clear animated outer ring only
  tft.fillCircle(240, 100, 58, BG);

  tft.drawCircle(240, 100, 48 + logoPulse, CYAN);
  tft.drawCircle(240, 100, 42, BLUE);

  float angleRad = logoAngle * 0.0174533;

  int px = 240 + cos(angleRad) * (48 + logoPulse);
  int py = 100 + sin(angleRad) * (48 + logoPulse);

  tft.fillCircle(px, py, 4, WHITE);

  tft.fillCircle(240, 100, 30, BG);

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, BG);
  tft.setTextSize(4);
  tft.drawString("D", 240, 100);
}

void openWelcomePage() {

  currentPage = PAGE_WELCOME;

  logoAngle = 0;
  logoPulse = 0;

  drawWelcomeBase();
}

// ============================================================
// PAGE 2 - MODE SELECTION
// ============================================================

void drawModePage() {

  tft.fillScreen(BG);
  drawHeader("MODE SELECT");

  // Default mode card
  uint16_t defaultColor =
    (selectedMode == MODE_DEFAULT) ? GREEN : BLUE;

  tft.fillRoundRect(
    25, 65, 205, 90, 10, PANEL
  );

  tft.drawRoundRect(
    25, 65, 205, 90, 10, defaultColor
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(2);
  tft.drawString("DEFAULT MODE", 127, 94);

  tft.setTextColor(GREY, PANEL);
  tft.setTextSize(1);
  tft.drawString("Preset parameters", 127, 123);

  // Custom mode card
  uint16_t customColor =
    (selectedMode == MODE_CUSTOM) ? GREEN : BLUE;

  tft.fillRoundRect(
    250, 65, 205, 90, 10, PANEL
  );

  tft.drawRoundRect(
    250, 65, 205, 90, 10, customColor
  );

  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(2);
  tft.drawString("CUSTOM MODE", 352, 94);

  tft.setTextColor(GREY, PANEL);
  tft.setTextSize(1);
  tft.drawString("Manual parameters", 352, 123);

  // Cycle selection
  drawPanel(25, 170, 430, 90, "NUMBER OF CYCLES");

  drawButton(55, 205, 70, 42, "-", BLUE, 3);
  drawButton(355, 205, 70, 42, "+", BLUE, 3);

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(4);
  tft.drawString(
    String(selectedCycles),
    240, 225
  );

  // Navigation
  drawButton(15, 278, 130, 32, "BACK", PANEL2);
  drawButton(335, 278, 130, 32, "NEXT", GREEN);

  drawFooter("SELECT MODE AND CYCLES");
}

// ============================================================
// PAGE 3 - CYCLE CONFIGURATION
// ============================================================

void drawCyclePage() {

  tft.fillScreen(BG);

  drawHeader("CYCLE SETUP");

  // Cycle heading
  tft.fillRoundRect(
    115, 53, 250, 35, 8, PANEL2
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(CYAN, PANEL2);
  tft.setTextSize(2);

  tft.drawString(
    "CYCLE " + String(currentSetupCycle + 1),
    240, 70
  );

  // Temperature
  drawPanel(20, 100, 440, 82, "TEMPERATURE");

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(4);

  tft.drawString(
    String(cycleTemp[currentSetupCycle]) + " C",
    240, 140
  );

  drawButton(35, 145, 90, 30, "-", BLUE);
  drawButton(355, 145, 90, 30, "+", BLUE);

  // Time
  drawPanel(20, 190, 440, 82, "HOLD TIME (MIN)");

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(4);

  tft.drawString(
    String(cycleTime[currentSetupCycle]) + " min",
    240, 230
  );

  drawButton(35, 235, 90, 30, "-", BLUE);
  drawButton(355, 235, 90, 30, "+", BLUE);

  // Navigation
  drawButton(10, 282, 110, 32, "BACK", PANEL2);

  drawButton(
    130, 282, 100, 32,
    "PREV", BLUE
  );

  drawButton(
    240, 282, 100, 32,
    "NEXT", BLUE
  );

  drawButton(
    350, 282, 120, 32,
    "REVIEW", GREEN
  );

  drawFooter(
    "CYCLE " + String(currentSetupCycle + 1) +
    " OF " + String(selectedCycles)
  );
}

// ============================================================
// PAGE 4 - REVIEW
// ============================================================

void drawReviewPage() {

  tft.fillScreen(BG);

  drawHeader("REVIEW");

  // Column titles
  tft.fillRoundRect(
    15, 52, 450, 28, 5, BLUE
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, BLUE);
  tft.setTextSize(2);

  tft.drawString("CYCLE", 70, 66);
  tft.drawString("TEMP", 230, 66);
  tft.drawString("TIME", 380, 66);

  // Cycle rows
  for (int i = 0; i < selectedCycles; i++) {

    int rowY = 85 + i * 37;

    uint16_t rowColor = PANEL;

    tft.fillRoundRect(
      15, rowY, 450, 32, 5, rowColor
    );

    tft.drawRoundRect(
      15, rowY, 450, 32, 5, BLUE
    );

    tft.setTextDatum(MC_DATUM);
    tft.setTextColor(WHITE, rowColor);
    tft.setTextSize(2);

    tft.drawString(
      String(i + 1),
      70, rowY + 16
    );

    tft.drawString(
      String(cycleTemp[i]) + " C",
      230, rowY + 16
    );

    tft.drawString(
      String(cycleTime[i]) + " min",
      380, rowY + 16
    );
  }

  // Navigation
  drawButton(15, 278, 130, 34, "BACK", PANEL2);
  drawButton(175, 278, 130, 34, "EDIT", BLUE);
  drawButton(335, 278, 130, 34, "START", GREEN);

  drawFooter("TO EDIT: TOUCH A CYCLE ROW");
}

// ============================================================
// PAGE 5 - RUNNING
// ============================================================

void drawRunningPage() {

  tft.fillScreen(BG);

  drawHeader("PROCESS");

  // Running status
  tft.fillRoundRect(
    95, 52, 290, 32, 8, ORANGE
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(BLACK, ORANGE);
  tft.setTextSize(2);

  tft.drawString("PROCESS RUNNING", 240, 68);

  // Current cycle
  drawPanel(20, 95, 440, 65, "CURRENT CYCLE");

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(3);

  tft.drawString(
    String(currentRunningCycle + 1) +
    " / " + String(selectedCycles),
    240, 130
  );

  // Temperature target
  drawPanel(20, 170, 210, 75, "SET TEMP");

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(CYAN, PANEL);
  tft.setTextSize(3);

  tft.drawString(
    String(cycleTemp[currentRunningCycle]) + " C",
    125, 210
  );

  // Hold time remaining
  drawPanel(250, 170, 210, 75, "TIME REMAINING");

  int mins = remainingSeconds / 60;
  int secs = remainingSeconds % 60;

  char timeString[10];

  sprintf(
    timeString,
    "%02d:%02d",
    mins, secs
  );

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(WHITE, PANEL);
  tft.setTextSize(3);

  tft.drawString(
    timeString,
    355, 210
  );

  // Pressure placeholder
  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(GREY, BG);
  tft.setTextSize(2);

  tft.drawString(
    "Pressure: -- bar",
    240, 260
  );

  // Stop
  drawButton(
    140, 282, 200, 34,
    "STOP", RED, 2
  );

  drawFooter("SIMULATION ONLY");
}

// ============================================================
// PAGE 6 - COMPLETE
// ============================================================

void drawCompletePage() {

  tft.fillScreen(BG);

  tft.fillRect(0, 0, 480, 6, GREEN);

  // Success symbol
  tft.drawCircle(240, 95, 45, GREEN);
  tft.drawCircle(240, 95, 35, CYAN);

  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(GREEN, BG);
  tft.setTextSize(5);

  tft.drawString("OK", 240, 95);

  // Title
  tft.setTextColor(WHITE, BG);
  tft.setTextSize(3);

  tft.drawString("PROCESS COMPLETE", 240, 165);

  // Result
  tft.setTextColor(CYAN, BG);
  tft.setTextSize(2);

  tft.drawString(
    String(completedCycles) + " CYCLES COMPLETED",
    240, 205
  );

  // Warning
  tft.setTextColor(ORANGE, BG);
  tft.setTextSize(1);

  tft.drawString(
    "WAIT FOR SAFE COOLING BEFORE OPENING",
    240, 235
  );

  // Home
  drawButton(
    140, 270, 200, 45,
    "HOME", BLUE, 2
  );

  tft.setTextColor(GREY, BG);
  tft.setTextSize(1);

  tft.drawString(
    "INNOMAYI TECHNOLOGIES",
    240, 310
  );
}

// ============================================================
// PAGE DRAW ROUTER
// ============================================================

void drawCurrentPage() {

  switch (currentPage) {

    case PAGE_WELCOME:
      drawWelcomeBase();
      break;

    case PAGE_MODE:
      drawModePage();
      break;

    case PAGE_CYCLE:
      drawCyclePage();
      break;

    case PAGE_REVIEW:
      drawReviewPage();
      break;

    case PAGE_RUNNING:
      drawRunningPage();
      break;

    case PAGE_COMPLETE:
      drawCompletePage();
      break;
  }
}

// ============================================================
// MODE SELECTION
// ============================================================

void selectDefaultMode() {

  selectedMode = MODE_DEFAULT;

  for (int i = 0; i < MAX_CYCLES; i++) {
    cycleTemp[i] = 121;
    cycleTime[i] = 20;
  }

  drawModePage();
}

void selectCustomMode() {

  selectedMode = MODE_CUSTOM;

  drawModePage();
}

// ============================================================
// PROCESS START
// ============================================================

void startProcessSimulation() {

  completedCycles = 0;
  currentRunningCycle = 0;

  remainingSeconds =
    cycleTime[currentRunningCycle] * 60;

  processRunning = true;
  processStopped = false;

  statusText = "RUNNING";

  lastTimerTick = millis();

  currentPage = PAGE_RUNNING;

  drawRunningPage();
}

// ============================================================
// PROCESS TIMER - SIMULATION
// ============================================================

void updateProcessTimer() {

  if (!processRunning) return;

  if (millis() - lastTimerTick < TIMER_INTERVAL) {
    return;
  }

  lastTimerTick = millis();

  if (remainingSeconds > 0) {
    remainingSeconds--;
  }

  if (remainingSeconds <= 0) {

    completedCycles++;

    // All selected cycles completed
    if (completedCycles >= selectedCycles) {

      processRunning = false;
      processStopped = false;

      statusText = "COMPLETE";

      currentPage = PAGE_COMPLETE;

      drawCompletePage();

      return;
    }

    // Move to next cycle
    currentRunningCycle++;

    if (currentRunningCycle < selectedCycles) {

      remainingSeconds =
        cycleTime[currentRunningCycle] * 60;
    }
  }

  drawRunningPage();
}

// ============================================================
// STOP PROCESS
// ============================================================

void stopProcessSimulation() {

  processRunning = false;
  processStopped = true;

  statusText = "STOPPED";

  currentPage = PAGE_REVIEW;

  drawReviewPage();
}

// ============================================================
// RESET PROCESS SETTINGS
// ============================================================

void resetSettings() {

  selectedMode = MODE_DEFAULT;

  selectedCycles = 1;
  currentSetupCycle = 0;
  currentRunningCycle = 0;
  completedCycles = 0;
  remainingSeconds = 0;

  processRunning = false;
  processStopped = false;

  for (int i = 0; i < MAX_CYCLES; i++) {
    cycleTemp[i] = 121;
    cycleTime[i] = 20;
  }

  statusText = "IDLE";

  currentPage = PAGE_WELCOME;

  drawWelcomeBase();
}

// ============================================================
// TOUCH - PAGE 1
// ============================================================

void handleWelcomeTouch(int x, int y) {

  if (x >= 140 && x <= 340 &&
      y >= 245 && y <= 305) {

    currentPage = PAGE_MODE;

    drawModePage();
  }
}

// ============================================================
// TOUCH - PAGE 2
// ============================================================

void handleModeTouch(int x, int y) {

  // Default mode
  if (x >= 25 && x <= 230 &&
      y >= 65 && y <= 155) {

    selectDefaultMode();
    return;
  }

  // Custom mode
  if (x >= 250 && x <= 455 &&
      y >= 65 && y <= 155) {

    selectCustomMode();
    return;
  }

  // Cycle decrease
  if (x >= 55 && x <= 125 &&
      y >= 205 && y <= 247) {

    if (selectedCycles > 1) {
      selectedCycles--;
    }

    drawModePage();
    return;
  }

  // Cycle increase
  if (x >= 355 && x <= 430 &&
      y >= 205 && y <= 247) {

    if (selectedCycles < MAX_CYCLES) {
      selectedCycles++;
    }

    drawModePage();
    return;
  }

  // Back
  if (x >= 15 && x <= 145 &&
      y >= 278 && y <= 312) {

    currentPage = PAGE_WELCOME;
    drawWelcomeBase();
    return;
  }

  // Next
  if (x >= 335 && x <= 465 &&
      y >= 278 && y <= 312) {

    currentSetupCycle = 0;

    // Default mode presets
    if (selectedMode == MODE_DEFAULT) {

      for (int i = 0; i < selectedCycles; i++) {
        cycleTemp[i] = 121;
        cycleTime[i] = 20;
      }
    }

    currentPage = PAGE_CYCLE;
    drawCyclePage();
    return;
  }
}

// ============================================================
// TOUCH - PAGE 3
// ============================================================

void handleCycleTouch(int x, int y) {

  // Temperature decrease
  if (x >= 35 && x <= 125 &&
      y >= 145 && y <= 175) {

    if (cycleTemp[currentSetupCycle] > 1) {
      cycleTemp[currentSetupCycle]--;
    }

    drawCyclePage();
    return;
  }

  // Temperature increase
  if (x >= 355 && x <= 445 &&
      y >= 145 && y <= 175) {

    if (cycleTemp[currentSetupCycle] < 130) {
      cycleTemp[currentSetupCycle]++;
    }

    drawCyclePage();
    return;
  }

  // Time decrease
  if (x >= 35 && x <= 125 &&
      y >= 235 && y <= 265) {

    if (cycleTime[currentSetupCycle] > 1) {
      cycleTime[currentSetupCycle]--;
    }

    drawCyclePage();
    return;
  }

  // Time increase
  if (x >= 355 && x <= 445 &&
      y >= 235 && y <= 265) {

    if (cycleTime[currentSetupCycle] < 99) {
      cycleTime[currentSetupCycle]++;
    }

    drawCyclePage();
    return;
  }

  // Back to mode selection
  if (x >= 10 && x <= 120 &&
      y >= 282 && y <= 314) {

    currentPage = PAGE_MODE;
    drawModePage();
    return;
  }

  // Previous cycle
  if (x >= 130 && x <= 230 &&
      y >= 282 && y <= 314) {

    if (currentSetupCycle > 0) {
      currentSetupCycle--;
    }

    drawCyclePage();
    return;
  }

  // Next cycle
  if (x >= 240 && x <= 340 &&
      y >= 282 && y <= 314) {

    if (currentSetupCycle < selectedCycles - 1) {
      currentSetupCycle++;
    }
    else {
      // Last cycle -> Review
      currentPage = PAGE_REVIEW;
      drawReviewPage();
      return;
    }

    drawCyclePage();
    return;
  }

  // Review
  if (x >= 350 && x <= 470 &&
      y >= 282 && y <= 314) {

    currentPage = PAGE_REVIEW;
    drawReviewPage();
    return;
  }
}

// ============================================================
// TOUCH - PAGE 4
// ============================================================

void handleReviewTouch(int x, int y) {

  // Select cycle row to edit
  for (int i = 0; i < selectedCycles; i++) {

    int rowY = 85 + i * 37;

    if (x >= 15 && x <= 465 &&
        y >= rowY && y <= rowY + 32) {

      currentSetupCycle = i;

      currentPage = PAGE_CYCLE;
      drawCyclePage();

      return;
    }
  }

  // Back
  if (x >= 15 && x <= 145 &&
      y >= 278 && y <= 312) {

    currentPage = PAGE_CYCLE;
    drawCyclePage();
    return;
  }

  // Edit
  if (x >= 175 && x <= 305 &&
      y >= 278 && y <= 312) {

    currentSetupCycle = 0;

    currentPage = PAGE_CYCLE;
    drawCyclePage();
    return;
  }

  // Start
  if (x >= 335 && x <= 465 &&
      y >= 278 && y <= 312) {

    startProcessSimulation();
    return;
  }
}

// ============================================================
// TOUCH - PAGE 5
// ============================================================

void handleRunningTouch(int x, int y) {

  // STOP
  if (x >= 140 && x <= 340 &&
      y >= 282 && y <= 316) {

    stopProcessSimulation();
  }
}

// ============================================================
// TOUCH - PAGE 6
// ============================================================

void handleCompleteTouch(int x, int y) {

  // HOME
  if (x >= 140 && x <= 340 &&
      y >= 270 && y <= 315) {

    resetSettings();
  }
}

// ============================================================
// MASTER TOUCH ROUTER
// ============================================================

void handleTouch(int x, int y) {

  Serial.print("Touch: ");
  Serial.print(x);
  Serial.print(", ");
  Serial.println(y);

  switch (currentPage) {

    case PAGE_WELCOME:
      handleWelcomeTouch(x, y);
      break;

    case PAGE_MODE:
      handleModeTouch(x, y);
      break;

    case PAGE_CYCLE:
      handleCycleTouch(x, y);
      break;

    case PAGE_REVIEW:
      handleReviewTouch(x, y);
      break;

    case PAGE_RUNNING:
      handleRunningTouch(x, y);
      break;

    case PAGE_COMPLETE:
      handleCompleteTouch(x, y);
      break;
  }
}

// ============================================================
// SETUP
// ============================================================

void setup() {

  Serial.begin(115200);

  pinMode(TFT_CS_PIN, OUTPUT);
  pinMode(TOUCH_CS, OUTPUT);
  pinMode(TOUCH_IRQ, INPUT_PULLUP);

  digitalWrite(TFT_CS_PIN, HIGH);
  digitalWrite(TOUCH_CS, HIGH);

  SPI.begin(
    SPI_CLK,
    SPI_MISO,
    SPI_MOSI,
    TFT_CS_PIN
  );

  tft.init();
  tft.setRotation(1);

  tft.fillScreen(BG);

  currentPage = PAGE_WELCOME;

  drawWelcomeBase();

  Serial.println();
  Serial.println("==============================");
  Serial.println("DYGNOVA ARS MULTI-PAGE UI");
  Serial.println("6 PAGE NAVIGATION");
  Serial.println("TOUCH READY");
  Serial.println("==============================");
}

// ============================================================
// LOOP
// ============================================================

void loop() {

  // Welcome animation
  if (currentPage == PAGE_WELCOME) {
    drawWelcomeAnimation();
  }

  // Running timer simulation
  if (currentPage == PAGE_RUNNING) {
    updateProcessTimer();
  }

  // Touch
  int x = 0;
  int y = 0;

  if (getTouch(x, y)) {

    if (millis() - lastTouchTime > TOUCH_DELAY) {

      lastTouchTime = millis();

      handleTouch(x, y);

      waitTouchRelease();
    }
  }
}


```
pinout
<img width="997" height="581" alt="image" src="https://github.com/user-attachments/assets/287404bc-e67c-44c1-b7ae-dbcb641c503b" />



===== CALIBRATION RESULTS =====
|position|value|raw|
|------|------|---------|
|TOP LEFT | Screen: 30,30 | Raw: 3421,3675 |
|TOP CENTER | Screen: 240,30 | Raw: 3471,1995|
|TOP RIGHT | Screen: 450,30 | Raw: 3454,384|
|MIDDLE LEFT | Screen: 30,160 | Raw: 1887,3607|
|CENTER | Screen: 240,160 | Raw: 1881,2005|
|MIDDLE RIGHT | Screen: 450,160 | Raw: 1817,399|
|BOTTOM LEFT | Screen: 30,290 | Raw: 459,3615|
|BOTTOM CENTER | Screen: 240,290 | Raw: 385,2008|
|BOTTOM RIGHT | Screen: 450,290 | Raw: 325,367|
===== END =====

