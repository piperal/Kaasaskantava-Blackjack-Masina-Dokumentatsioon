# Projekti dokumentatsioon

## 1. Projekti eesmärk ja seadme lühikirjeldus
**Mis asi see on, mida ja miks me teeme? Millist praktilist probleemi see lahendab?**

- Minu projekti eesmärk oli luua kaasaskantev blackjacki mängumasin
- Ehitasin sellise masina kuna mulle väga meeldiv blackjacki mangida
- Peamine probleem mida see lahendab on igavus
- Peamised komponendid: Arduino mikrokontroller, 2 LCD ekraani, 1 nelja kohaline TM1637 ekraain ja 2 nuppu

---

## 2. Sisendite loetelu
**Millised on süsteemi poolt loetavad / mõõdetavad sisendid? Millega neid mõõdetakse / tuvastatakse?**


- Nupp "hit" ehk, vasakpoolne nupp mängu sees võtab kaarte juurde
- Nupp "hit" mängust väljas läheb järgmisele lehele (kui mäng läbi saab saab minna 'hit' nupuga edasi)
- Nupp "stand" ehk, parempoolne nupp mängu sees lõpetab mängija kaartide võtmise
- Nupp "stand" mängust väljas valib rastustaset (easy, normal, hard)

---

## 3. Väljundite loetelu
**Mida süsteem teeb / muudab? Millega väljund realiseeritakse?**

- Mängija võtab kaarte juurde → LCD ekraan
- Mängija kaotab/võidab → LCD ekraan
- Diiler võtab kaarte juurde → LCD ekraan
- Diiler kaotab/võidab → LCD ekraan
- Mängija võidab/kaotab → TM1637

---

## 4. Nõuded loodavale seadmele
**Mis peab toimuma, kui kasutaja teeb mingi toimingu? Kirjelda käitumisloogika.**

- Kui vajutatakse "hit" nuppu, vasak poolset nuppu, siis:
  - kui on mängus sees, siis antakse mängijale kaart juurde
  - kui ei ole mängus sees, siis alustatab algmenüüst mängu või peale mängu läbi saamist saab sama raskusatmega edasi mängida

- Kui vajutatakse "stand" nuppu, parem poolset nuppu:
  - kui on mängus sees, siis lõpetatakse kaartide juurde võtmine
  - Kui ei ole mängus sees, siis vahetatakse raskusastmete vahel või minnakse peale mängu läbi saamist raskusatme valimise menüsse


👉 _Pane siia KÕIK kokkulepitud reeglid. Need reeglid on alus, mille järgi hiljem hinnatakse, kas teie lahendus vastab eesmärgile._

---

## 5. Süsteemi füüsiliste komponentide loetelu
**Millest seade koosneb? Lisa lingid või täpsed nimed, et keegi teine saaks sama asja uuesti osta / teha.**

Tabelina või punktidena. Nt:

- Arduino Uno (mikrokontroller)
- 2 x LCD ekraan + I2C moodul 
- TM1637 moodul
- 2 mikrolülitit, Tactile Push Button Switch
- I2C driver (Wire)
- LCD driver (LiquidCrystal_I2C)
- 4 segmendiga TM1637 driver (TM1637)
- 3D prinditud ümbris
- Toiteallikas:  USB akupank, 5V ~2 A

👉 _Kui ise tegite 3D mudeli, lisage STL või Fusion faili `hardware/` alla. Kui kasutasite netist leitud mudelit, märkige allikas._

---

## 6. Ühendusskeem
**Kuidas kõik osad on omavahel ühendatud?**
![Ühendusskeem](/hardware/wiring-diagram.png)
- Vooluallikas akupank

---

## 7. Süsteemi juhtiv kood (või pseudokood)

Programmi loogika on üsna 'lihtne', kuna kogu kood käib mängija sisendi ümber.
Igale mängu osale on loodud funktsioon mida kutsutakse õigel ajal.
Enne igat toimingut ootab süsteem mängija sisendit ja käithub 
sisendist mis talle sai antud.
Koodis olevad süsteemid ja funktsioonid on kõik komenteeritud

# Kood
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <TM1637Display.h>

// --------------------------------
// DISPLAYS
// --------------------------------

// Dealer LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Player LCD
LiquidCrystal_I2C lcd2(0x26, 16, 2);

// 4-DIGIT DISPLAY
#define CLK A3
#define DIO A2

TM1637Display scoreDisplay(CLK, DIO);

// --------------------------------
// BUTTON PINS
// --------------------------------

const int HIT_BUTTON = 3;
const int STAND_BUTTON = 2;

// --------------------------------
// DECK AND CARDS
// --------------------------------

int deck[52];
int deckPosition = 0;

int playerCards[12];
int dealerCards[12];

int playerCount = 0;
int dealerCount = 0;

// --------------------------------
// GAME STATE
// --------------------------------

bool gameOver = false;
bool playerTurn = true;
bool selectingDifficulty = true;

// PLAYER TOTAL POINTS
int totalPoints = 0;

// --------------------------------
// DIFFICULTY SETTINGS
// --------------------------------

// 0 = Easy
// 1 = Medium
// 2 = Hard

int difficulty = 0;
int dealerStandScore = 15;

// --------------------------------
// BUTTON DEBOUNCE
// --------------------------------

int lastHitReading = HIGH;
int hitState = HIGH;
unsigned long hitTimer = 0;

int lastStandReading = HIGH;
int standState = HIGH;
unsigned long standTimer = 0;

const unsigned long debounceTime = 40;

// --------------------------------
// UPDATE 4-DIGIT SCORE DISPLAY
// --------------------------------

void updatePointsDisplay() {
  scoreDisplay.showNumberDec(totalPoints, false);
}

// --------------------------------
// CREATE AND SHUFFLE DECK
// --------------------------------

void shuffleDeck() {
  for (int i = 0; i < 52; i++) {
    deck[i] = i;
  }

  for (int i = 51; i > 0; i--) {
    int j = random(i + 1);

    int temp = deck[i];
    deck[i] = deck[j];
    deck[j] = temp;
  }

  deckPosition = 0;
}

// --------------------------------
// DRAW CARD
// --------------------------------

int drawCard() {
  return deck[deckPosition++];
}

// --------------------------------
// GET CARD VALUE
// --------------------------------

int cardValue(int card) {
  int rank = card % 13;

  if (rank == 0) return 11;
  if (rank >= 10) return 10;

  return rank + 1;
}

// --------------------------------
// DISPLAY CARD NAME
// --------------------------------

void printCard(LiquidCrystal_I2C &screen, int card) {
  int rank = card % 13;

  if (rank == 0) screen.print("A");
  else if (rank == 10) screen.print("J");
  else if (rank == 11) screen.print("Q");
  else if (rank == 12) screen.print("K");
  else screen.print(rank + 1);
}

// --------------------------------
// CALCULATE HAND SCORE
// --------------------------------

int getScore(int cards[], int count) {
  int total = 0;
  int aces = 0;

  for (int i = 0; i < count; i++) {
    total += cardValue(cards[i]);

    if (cards[i] % 13 == 0) {
      aces++;
    }
  }

  while (total > 21 && aces > 0) {
    total -= 10;
    aces--;
  }

  return total;
}

// --------------------------------
// CHECK NATURAL BLACKJACK
// --------------------------------

bool isBlackjack(int cards[], int count) {
  return count == 2 && getScore(cards, count) == 21;
}

// --------------------------------
// GET DIFFICULTY NAME
// --------------------------------

const char* getDifficultyName() {
  if (difficulty == 0) return "EASY";
  if (difficulty == 1) return "MEDIUM";
  return "HARD";
}

// --------------------------------
// UPDATE DIFFICULTY
// --------------------------------

void updateDifficulty() {
  if (difficulty == 0) {
    dealerStandScore = 19;
  }
  else if (difficulty == 1) {
    dealerStandScore = 17;
  }
  else {
    dealerStandScore = 15;
  }

  lcd.clear();
  lcd2.clear();

  lcd.setCursor(0, 0);
  lcd.print("SELECT DIFFICULTY");

  lcd.setCursor(0, 1);
  lcd.print(getDifficultyName());

  lcd2.setCursor(0, 0);
  lcd2.print("STAND: CHANGE");

  lcd2.setCursor(0, 1);
  lcd2.print("HIT: START GAME");
}

// --------------------------------
// SHOW DIFFICULTY SELECTION
// --------------------------------

void showDifficultySelection() {
  selectingDifficulty = true;
  gameOver = false;
  playerTurn = false;

  updateDifficulty();
}

// --------------------------------
// DISPLAY PLAYER SCREEN
// --------------------------------

void displayPlayer() {
  lcd2.clear();

  lcd2.setCursor(0, 0);
  lcd2.print("P:");

  int column = 2;

  for (int i = 0; i < playerCount; i++) {
    if (column > 13) break;

    lcd2.setCursor(column, 0);

    printCard(lcd2, playerCards[i]);

    if (playerCards[i] % 13 == 9) {
      column += 3;
    }
    else {
      column += 2;
    }
  }

  lcd2.setCursor(0, 1);
  lcd2.print("Score: ");
  lcd2.print(getScore(playerCards, playerCount));
}

// --------------------------------
// DISPLAY DEALER SCREEN
// --------------------------------

void displayDealerHidden() {
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("D:");

  printCard(lcd, dealerCards[0]);

  lcd.print(" ??");

  lcd.setCursor(0, 1);
  lcd.print("Dealer waiting");
}

// --------------------------------
// DISPLAY ALL DEALER CARDS
// --------------------------------

void displayDealerCards() {
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("Dealer:");

  lcd.setCursor(0, 1);

  int column = 0;

  for (int i = 0; i < dealerCount; i++) {
    if (column > 13) break;

    lcd.setCursor(column, 1);

    printCard(lcd, dealerCards[i]);

    if (dealerCards[i] % 13 == 9) {
      column += 3;
    }
    else {
      column += 2;
    }
  }

  lcd.setCursor(11, 0);
  lcd.print("=");
  lcd.print(getScore(dealerCards, dealerCount));
}

// --------------------------------
// DISPLAY ACTIVE GAME
// --------------------------------

void displayGame() {
  displayPlayer();
  displayDealerHidden();
}

// --------------------------------
// DISPLAY GAME RESULT
// --------------------------------

void displayResult(const char* result) {
  lcd.clear();
  lcd2.clear();

  lcd.setCursor(0, 0);
  lcd.print(result);

  lcd2.setCursor(0, 0);
  lcd2.print(result);

  lcd.setCursor(0, 1);
  lcd.print("HIT = NEXT GAME");

  lcd2.setCursor(0, 1);
  lcd2.print("STAND = DIFFICULTY");

  updatePointsDisplay();
}

// --------------------------------
// FINISH ROUND AND AWARD POINTS
// --------------------------------

void finishRound(const char* result, int pointsEarned) {
  gameOver = true;
  playerTurn = false;

  totalPoints += pointsEarned;

  updatePointsDisplay();

  displayResult(result);
}

// --------------------------------
// DEALER TURN
// --------------------------------

void dealerTurn() {
  playerTurn = false;

  displayDealerCards();

  delay(1200);

  while (getScore(dealerCards, dealerCount) < dealerStandScore &&
         dealerCount < 12) {

    dealerCards[dealerCount++] = drawCard();

    displayDealerCards();

    delay(2000);
  }

  int playerScore = getScore(playerCards, playerCount);
  int dealerScore = getScore(dealerCards, dealerCount);

  if (dealerScore > 21) {
    finishRound("DEALER BUST!", 1);
  }
  else if (playerScore > dealerScore) {
    finishRound("YOU WIN!", 1);
  }
  else if (playerScore < dealerScore) {
    finishRound("YOU LOSE!", -1);
  }
  else {
    finishRound("DRAW!", 0);
  }
}

// --------------------------------
// HIT BUTTON ACTION
// --------------------------------

void hit() {
  // On difficulty screen, start the game
  if (selectingDifficulty) {
    newGame();
    return;
  }

  // After a round, HIT starts the next game
  if (gameOver) {
    newGame();
    return;
  }

  if (!playerTurn || playerCount >= 12) return;

  playerCards[playerCount++] = drawCard();

  int score = getScore(playerCards, playerCount);

  if (score > 21) {
    lcd2.clear();

    lcd2.setCursor(0, 0);
    lcd2.print("BUST!");

    lcd2.setCursor(0, 1);
    lcd2.print("Too many points!");

    delay(1500);

    finishRound("BUST! YOU LOSE", -1);
  }
  else if (score == 21) {
    dealerTurn();
  }
  else {
    displayGame();
  }
}

// --------------------------------
// STAND BUTTON ACTION
// --------------------------------

void stand() {
  // During game over, STAND opens difficulty selection
  if (gameOver) {
    showDifficultySelection();
    return;
  }

  // On difficulty screen, change difficulty
  if (selectingDifficulty) {
    difficulty++;

    if (difficulty > 2) {
      difficulty = 0;
    }

    updateDifficulty();
    return;
  }

  // Normal stand during gameplay
  if (playerTurn) {
    dealerTurn();
  }
}

// --------------------------------
// START NEW GAME
// --------------------------------

void newGame() {
  selectingDifficulty = false;

  shuffleDeck();

  playerCount = 0;
  dealerCount = 0;

  gameOver = false;
  playerTurn = true;

  // Deal two cards each
  playerCards[playerCount++] = drawCard();
  dealerCards[dealerCount++] = drawCard();

  playerCards[playerCount++] = drawCard();
  dealerCards[dealerCount++] = drawCard();

  // Check natural Blackjack
  bool playerBJ = isBlackjack(playerCards, playerCount);
  bool dealerBJ = isBlackjack(dealerCards, dealerCount);

  displayPlayer();
  displayDealerHidden();

  if (playerBJ && dealerBJ) {
    finishRound("BOTH BLACKJACK", 0);
    return;
  }

  if (playerBJ) {
    finishRound("BLACKJACK WIN!", 5);
    return;
  }

  if (dealerBJ) {
    finishRound("DEALER BLACKJACK", 0);
    return;
  }

  displayGame();
}

// --------------------------------
// BUTTON DEBOUNCE
// --------------------------------

bool buttonPressed(
  int pin,
  int &lastReading,
  int &stableState,
  unsigned long &timer
) {
  int reading = digitalRead(pin);

  if (reading != lastReading) {
    timer = millis();
  }

  if (millis() - timer > debounceTime) {
    if (reading != stableState) {
      stableState = reading;

      if (stableState == LOW) {
        lastReading = reading;
        return true;
      }
    }
  }

  lastReading = reading;

  return false;
}

// --------------------------------
// SETUP
// --------------------------------

void setup() {
  pinMode(HIT_BUTTON, INPUT_PULLUP);
  pinMode(STAND_BUTTON, INPUT_PULLUP);

  // Start LCD screens
  lcd.init();
  lcd.backlight();

  lcd2.init();
  lcd2.backlight();

  // Start 4-digit display
  scoreDisplay.setBrightness(7);
  scoreDisplay.clear();

  updatePointsDisplay();

  randomSeed(analogRead(A0));

  lcd.setCursor(0, 0);
  lcd.print("DEALER SCREEN");

  lcd2.setCursor(0, 0);
  lcd2.print("PLAYER SCREEN");

  delay(1500);

  // Show difficulty selection instead of starting immediately
  showDifficultySelection();
}

// --------------------------------
// MAIN LOOP
// --------------------------------

void loop() {
  if (buttonPressed(
        HIT_BUTTON,
        lastHitReading,
        hitState,
        hitTimer
      )) {
    hit();
  }

  if (buttonPressed(
        STAND_BUTTON,
        lastStandReading,
        standState,
        standTimer
      )) {
    stand();
  }
}
