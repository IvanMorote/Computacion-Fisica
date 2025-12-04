Equipo: Ivan Morote Aguilar

Roles: Ivan Morote Aguilar (Montaje proyecto entero)

Plan de Sprints

## Sprint 1:

Definir el objetivo del proyecto (Juego Snake en Arduino).

Hacer un boceto en Tinkercad del circuito inicial:
  - Arduino UNO
  - Pantalla OLED o LCD
  - Botones / joystick para controlar dirección

Crear y subir las primeras versiones del código:
  - Representar la serpiente como coordenadas básicas
  - Mostrar un punto en pantalla
  - Permitir control básico (arriba, abajo, izquierda, derecha)

## Boceto en Tinkercad

<img width="1367" height="727" alt="image" src="https://github.com/user-attachments/assets/fe3104a1-05e6-406e-b8e3-ee67037d7611" />

Mi minijuego tiene componentes que no encuentro en Tinkercad, te dejo una imagen para que veas el boceto/prototipo de lo que quiero hacer:

<img width="954" height="1395" alt="89ddd7c5-84b3-40ad-ad84-9ac2448d9f09" src="https://github.com/user-attachments/assets/a3e3a576-9247-4085-bf78-13ced8ce6fba" />

## Code

```cpp
// The MAX7219 uses SPI communication protocol...Hence, import SPI.h library
#include <SPI.h>

// The chip select pin
#define CS 10

// Few necessary registers for configuring the MAX7219 chip
#define DECODE_MODE 9
#define INTENSITY 0x0A
#define SCAN_LIMIT 0x0B
#define SHUTDOWN 0x0C
#define DISPLAY_TEST 0x0F

// Buttons used for controlling the snake
#define left_button 2
#define right_button 3

volatile byte move_left = 0;
volatile byte move_right = 0;

// Varibles for snake
int snake_l = 2;
const int max_len = 15;
int snake[max_len][2];
byte cur_heading = 0;

// Variable for food blob
int blob[2] = { 0, 0 };
int is_eaten = 1;

// The game scene
byte scene[8] = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };

// A general function to send data to the MAX7219
void SendData(uint8_t address, uint8_t value) {
  digitalWrite(CS, LOW);
  SPI.transfer(address);   // Send address.
  SPI.transfer(value);     //   Send the value.
  digitalWrite(CS, HIGH);  // Finish transfer.
}

// Function to initialize the game variables
void init_game() {
  is_eaten = 1;
  move_left = 0;
  move_right = 0;
  cur_heading = 0;
  snake_l = 2;
  for (int i = 0; i < max_len; i++)
    for (int j = 0; j < 2; j++)
      snake[i][j] = 0;
  snake[max_len - 1][0] = 2;
  snake[max_len - 1][1] = 5;
  snake[max_len - 2][0] = 1;
  snake[max_len - 2][1] = 5;
  refresh_scene();
  while ((move_left || move_right) == 0)
    ;
  move_left = 0;
  move_right = 0;
}

// Function to draw snake on gamescene
void spawn_snake() {
  // If the snake goes out of the scene, it enters from the other side
  for (int j = max_len - snake_l; j < max_len; j++) {
    if (snake[j][0] <= 0)
      snake[j][0] = 8 + snake[j][0];
    else if (snake[j][0] >= 9)
      snake[j][0] = snake[j][0] - 8;
    if (snake[j][1] <= 0)
      snake[j][1] = 8 + snake[j][1];
    else if (snake[j][1] >= 9)
      snake[j][1] = snake[j][1] - 8;

    // Draw the snake on the LED matrix
    scene[snake[j][0] - 1] |= (1 << (snake[j][1] - 1));
  }
}

// Function to update the position and length of the snake
void snake_move() {
  // If snake eats a blob...Increase length
  if (snake[max_len - 1][0] == blob[0] && snake[max_len - 1][1] == blob[1]) {
    is_eaten = 1;
    snake_l += 1;
  }

  // Move each pixel forward
  for (int i = snake_l - 1; i >= 1; i--) {
    snake[max_len - 1 - i][0] = snake[max_len - i][0];
    snake[max_len - 1 - i][1] = snake[max_len - i][1];
  }

  // Move the head according to button input
  if (move_left == 1) {
    if (cur_heading == 0) {
      cur_heading = 1;
      snake[max_len - 1][1] -= 1;
    } else if (cur_heading == 1) {
      cur_heading = 2;
      snake[max_len - 1][0] -= 1;
    } else if (cur_heading == 2) {
      cur_heading = 3;
      snake[max_len - 1][1] += 1;
    } else if (cur_heading == 3) {
      cur_heading = 0;
      snake[max_len - 1][0] += 1;
    }
    move_left = 0;
  } else if (move_right == 1) {
    if (cur_heading == 0) {
      cur_heading = 3;
      snake[max_len - 1][1] += 1;
    } else if (cur_heading == 1) {
      cur_heading = 0;
      snake[max_len - 1][0] += 1;
    } else if (cur_heading == 2) {
      cur_heading = 1;
      snake[max_len - 1][1] -= 1;
    } else if (cur_heading == 3) {
      cur_heading = 2;
      snake[max_len - 1][0] -= 1;
    }
    move_right = 0;
  } else {
    if (cur_heading == 0) {
      snake[max_len - 1][0] += 1;
    } else if (cur_heading == 1) {
      snake[max_len - 1][1] -= 1;
    } else if (cur_heading == 2) {
      snake[max_len - 1][0] -= 1;
    } else if (cur_heading == 3) {
      snake[max_len - 1][1] += 1;
    }
  }
}

// Function to generate a blob and draw the blob on the gamescene
void blob_generator() {
  // If blob is eaten by the snake, generate one
  if (is_eaten) {
    blob[0] = random(1, 9);
    blob[1] = random(1, 9);
  }

  // Draw the blob on the gamescene
  scene[blob[0] - 1] |= (1 << (blob[1] - 1));
  is_eaten = 0;
}

// Function to redraw the gamescene to the LED matrix with updated variables
void refresh_scene() {
  for (int i = 0; i < 8; i++)
    scene[i] = 0x00;
  snake_move();
  spawn_snake();
  blob_generator();
  for (int i = 1; i < 9; i++)
    SendData(i, scene[i - 1]);
}

// Callback for interrupt attached to left button
void update_left() {
  move_left = 1;
}

// Callback for interrupt attached to right button
void update_right() {
  move_right = 1;
}

// Setup function
void setup() {
  // GPIO Configuration
  pinMode(left_button, INPUT_PULLUP);
  pinMode(right_button, INPUT_PULLUP);
  pinMode(CS, OUTPUT);
  
  // SPI configuration
  SPI.setBitOrder(MSBFIRST);     // Most significant bit first
  SPI.begin();                   // Start SPI
  SendData(DISPLAY_TEST, 0x00);  // Finish test mode.
  SendData(DECODE_MODE, 0x00);   // Disable BCD mode.
  SendData(INTENSITY, 0x01);     // Use lowest intensity.
  SendData(SCAN_LIMIT, 0x0f);    // Scan all digits.
  SendData(SHUTDOWN, 0x01);      // Turn on chip.

  // Random seed generation...Uses the noise in the analog channel 0 to create a random seed
  randomSeed(analogRead(0));

  // Attach interrupts to the buttons
  attachInterrupt(digitalPinToInterrupt(left_button), update_left, FALLING);
  attachInterrupt(digitalPinToInterrupt(right_button), update_right, FALLING);

  // Enable interrupt
  sei();

  // Start the game
  init_game();
}

void loop() {
  // Check if snake is at max length
  if (snake_l == max_len) {
    // If yes, display win and restart
    byte win_scene[8] = { B11100011, B00100100, B01000010, B11100100, B00000011, 0, B00011100, 0 };
    for (int i = 1; i < 9; i++)
      SendData(i, win_scene[i - 1]);
    delay(5000);
    init_game();
  }

  // Check if snake has collided with itself
  for (int i = 0; i < max_len - 1; i++) {
    if (snake[i][0] == snake[max_len - 1][0] && snake[i][1] == snake[max_len - 1][1]) {
      // If yes, blink all leds and restart the game
      delay(1000);
      for (int j = 0; j < 4; j++) {
        SendData(DISPLAY_TEST, 0x01);
        delay(500);
        SendData(DISPLAY_TEST, 0x00);
        delay(500);
      }
      init_game();
      break;
    }
  }
  
  // Keep refreshing the matrix with updated data....
  refresh_scene();
  // ...Every 0.5 secs
  delay(500);
}

## Sprint 2:

- Mejorar montaje

-Actualizar codigo

- Montaje

## Boceto en tinkercad 2:

<img width="779" height="915" alt="image" src="https://github.com/user-attachments/assets/66aa98b6-43db-4af3-ba14-ac60915e7fba" />

## Code actualizado

#define PIN_ROW1 2
#define PIN_ROW2 3
#define PIN_ROW3 4
#define PIN_ROW4 5
#define PIN_ROW5 17
#define PIN_ROW6 16
#define PIN_ROW7 15
#define PIN_ROW8 14

#define PIN_COL1 6
#define PIN_COL2 7
#define PIN_COL3 8
#define PIN_COL4 9
#define PIN_COL5 10
#define PIN_COL6 11
#define PIN_COL7 12
#define PIN_COL8 13

#define PIN_INPUT_LEFT 18
#define PIN_INPUT_RIGHT 19

#define MAX_BODY_LENGTH 12
#define GAME_AREA_WIDTH 8
#define GAME_AREA_HEIGHT 8
#define GAME_OVER_TIME 3000

#define DIRECTION_UP 0
#define DIRECTION_RIGHT 1
#define DIRECTION_DOWN 2
#define DIRECTION_LEFT 3

typedef struct p
{
  int x;
  int y;
} Position;

Position body[MAX_BODY_LENGTH];
int lastInput;
int head;
int tail;
int bodyLength;
int direction;
Position food;
int gameover;
int score;
int elapsedTime;
bool readInput;
unsigned long previousTime;


void setPixel(int row, int column)
{
  digitalWrite(PIN_ROW1, LOW);
  digitalWrite(PIN_ROW2, LOW);
  digitalWrite(PIN_ROW3, LOW);
  digitalWrite(PIN_ROW4, LOW);
  digitalWrite(PIN_ROW5, LOW);
  digitalWrite(PIN_ROW6, LOW);
  digitalWrite(PIN_ROW7, LOW);
  digitalWrite(PIN_ROW8, LOW);
  digitalWrite(PIN_COL1, HIGH);
  digitalWrite(PIN_COL2, HIGH);
  digitalWrite(PIN_COL3, HIGH);
  digitalWrite(PIN_COL4, HIGH);
  digitalWrite(PIN_COL5, HIGH);
  digitalWrite(PIN_COL6, HIGH);
  digitalWrite(PIN_COL7, HIGH);
  digitalWrite(PIN_COL8, HIGH);
  
    switch(column)
    {
      case 0: digitalWrite(PIN_COL1, LOW); break;
      case 1: digitalWrite(PIN_COL2, LOW); break;
      case 2: digitalWrite(PIN_COL3, LOW); break;
      case 3: digitalWrite(PIN_COL4, LOW); break;
      case 4: digitalWrite(PIN_COL5, LOW); break;
      case 5: digitalWrite(PIN_COL6, LOW); break;
      case 6: digitalWrite(PIN_COL7, LOW); break;
      case 7: digitalWrite(PIN_COL8, LOW); break;
      default: break;
    }

    switch(row)
    {
      case 0: digitalWrite(PIN_ROW1, HIGH); break;
      case 1: digitalWrite(PIN_ROW2, HIGH); break;
      case 2: digitalWrite(PIN_ROW3, HIGH); break;
      case 3: digitalWrite(PIN_ROW4, HIGH); break;
      case 4: digitalWrite(PIN_ROW5, HIGH); break;
      case 5: digitalWrite(PIN_ROW6, HIGH); break;
      case 6: digitalWrite(PIN_ROW7, HIGH); break;
      case 7: digitalWrite(PIN_ROW8, HIGH); break;
      default: break;
   }
}

bool foodPositionIsValid()
{
  if (food.x < 0 || food.y < 0) return false;

  for (int i = tail; i <= (head > tail ? head : MAX_BODY_LENGTH - 1); i++)
  {
    if (body[i].x == food.x && body[i].y == food.y) return false;
  }

  if (head < tail)
  {
    for (int i = 0; i <= head; i++)
    {
      if (body[i].x == food.x && body[i].y == food.y) return false;
    }
  }
  return true;
}

void checkGameover()
{
  for (int i = tail; i <= (head > tail ? head - 1 : MAX_BODY_LENGTH - 1); i++)
  {
    if (body[head].x == body[i].x && body[head].y == body[i].y)
    {
      gameover = GAME_OVER_TIME;
        return;
    }
  }

  if (head < tail)
  {
    for (int i = 0; i < head; i++)
    {
      if (body[head].x == body[i].x && body[head].y == body[i].y)
      {
        gameover = GAME_OVER_TIME;
        return;
      }
    }
  }
}

void spawnFood()
{
  while (!foodPositionIsValid())
    food = { random(GAME_AREA_WIDTH), random(GAME_AREA_HEIGHT) };
}


void draw()
{
  for (int i = tail; i <= (head > tail ? head : MAX_BODY_LENGTH - 1); i++)
  {
    setPixel(body[i].x, body[i].y);
  }
  if (head < tail)
  {
    for (int i = 0; i <= head; i++)
    {
      setPixel(body[i].x, body[i].y);
    }
  }
  setPixel(food.x, food.y);
}

void move()
{
  tail = tail + 1 == MAX_BODY_LENGTH ? 0 : tail + 1;
  Position prevHead = body[head];
  head = head + 1 == MAX_BODY_LENGTH ? 0 : head + 1;
  body[head] = { prevHead.x + (direction == DIRECTION_LEFT ? -1 : (direction == DIRECTION_RIGHT ? 1 : 0)), prevHead.y + (direction == DIRECTION_UP ? -1 : (direction == DIRECTION_DOWN ? 1 : 0)) };
  body[head].x = body[head].x < 0 ? GAME_AREA_WIDTH - 1 : (body[head].x >= GAME_AREA_WIDTH ? 0 : body[head].x);
  body[head].y = body[head].y < 0 ? GAME_AREA_HEIGHT - 1 : (body[head].y >= GAME_AREA_HEIGHT ? 0 : body[head].y);
}

void eat()
{
  if (body[head].x == food.x && body[head].y == food.y)
  {
    if (bodyLength < MAX_BODY_LENGTH)
    {
      bodyLength++;
      tail--;
      if (tail < 0) tail = MAX_BODY_LENGTH - 1;
    }
    score++;
    food = { -1, -1 };
    spawnFood();
  }
}

void reset()
{
  body[0] = { 3,3 };
  body[1] = { 3,4 };
  body[2] = { 3,5 };
  bodyLength = 3;
  head = 2;
  tail = 0;
  direction = DIRECTION_DOWN;
  food = {-1, -1};
  gameover = 0;
  elapsedTime = 0;
  score = 0;
  spawnFood();
  readInput = true;
}

void setup(){    
 
  pinMode(PIN_ROW1, OUTPUT);
  pinMode(PIN_ROW2, OUTPUT);
  pinMode(PIN_ROW3, OUTPUT);
  pinMode(PIN_ROW4, OUTPUT);
  pinMode(PIN_ROW5, OUTPUT);
  pinMode(PIN_ROW6, OUTPUT);
  pinMode(PIN_ROW7, OUTPUT);
  pinMode(PIN_ROW8, OUTPUT);
  pinMode(PIN_COL1, OUTPUT);
  pinMode(PIN_COL2, OUTPUT);
  pinMode(PIN_COL3, OUTPUT);
  pinMode(PIN_COL4, OUTPUT);
  pinMode(PIN_COL5, OUTPUT);
  pinMode(PIN_COL6, OUTPUT);
  pinMode(PIN_COL7, OUTPUT);
  pinMode(PIN_COL8, OUTPUT);

  pinMode(PIN_INPUT_LEFT, INPUT);
  pinMode(PIN_INPUT_RIGHT, INPUT);
  
  reset(); 
}


void loop(){   

    unsigned long currentTime = millis(); 

    if(!gameover)
    {
        draw();
        elapsedTime += currentTime - previousTime;
        if(elapsedTime > 500)
        {
            move();
            eat();
            checkGameover();
            elapsedTime = 0;
            readInput = true;
        }
        if(readInput)
        {
          if(digitalRead(PIN_INPUT_RIGHT) && !lastInput) {  direction = (direction + 1) % 4; readInput = false; }
          if(digitalRead(PIN_INPUT_LEFT) && !lastInput) { direction = (4 + direction-1) % 4; readInput = false; }
        }
        lastInput = digitalRead(PIN_INPUT_RIGHT) || digitalRead(PIN_INPUT_LEFT);
        
    }
    else
    {
        if(currentTime % 800 > 400) draw();
        gameover -= currentTime - previousTime;
        
        if(gameover <= 0) 
           reset();
    }
      
    previousTime = currentTime;
} 

## sprint 2

- Mejorar codigo

- Actualizar montaje

- Montaje en tinkercad de nuevo

## montaje fisico

![WhatsApp Image 2025-12-04 at 09 19 27](https://github.com/user-attachments/assets/aec7747d-5a22-4a33-ac25-ffc45dcdbd05)

## Tinkercad Actualizado

<img width="791" height="893" alt="image" src="https://github.com/user-attachments/assets/ff111592-2277-4fa0-a95d-3bda81536e35" />

## Code actualizado
#define PIN_ROW1 2
#define PIN_ROW2 3
#define PIN_ROW3 4
#define PIN_ROW4 5
#define PIN_ROW5 17
#define PIN_ROW6 16
#define PIN_ROW7 15
#define PIN_ROW8 14

#define PIN_COL1 6
#define PIN_COL2 7
#define PIN_COL3 8
#define PIN_COL4 9
#define PIN_COL5 10
#define PIN_COL6 11
#define PIN_COL7 12
#define PIN_COL8 13

#define PIN_INPUT_LEFT 18
#define PIN_INPUT_RIGHT 19

#define MAX_BODY_LENGTH 12
#define GAME_AREA_WIDTH 8
#define GAME_AREA_HEIGHT 8
#define GAME_OVER_TIME 3000

#define DIRECTION_UP 0
#define DIRECTION_RIGHT 1
#define DIRECTION_DOWN 2
#define DIRECTION_LEFT 3

typedef struct p
{
  int x;
  int y;
} Position;

Position body[MAX_BODY_LENGTH];
int lastInput;
int head;
int tail;
int bodyLength;
int direction;
Position food;
int gameover;
int score;
int elapsedTime;
bool readInput;
unsigned long previousTime;


void setPixel(int row, int column)
{
  digitalWrite(PIN_ROW1, LOW);
  digitalWrite(PIN_ROW2, LOW);
  digitalWrite(PIN_ROW3, LOW);
  digitalWrite(PIN_ROW4, LOW);
  digitalWrite(PIN_ROW5, LOW);
  digitalWrite(PIN_ROW6, LOW);
  digitalWrite(PIN_ROW7, LOW);
  digitalWrite(PIN_ROW8, LOW);
  digitalWrite(PIN_COL1, HIGH);
  digitalWrite(PIN_COL2, HIGH);
  digitalWrite(PIN_COL3, HIGH);
  digitalWrite(PIN_COL4, HIGH);
  digitalWrite(PIN_COL5, HIGH);
  digitalWrite(PIN_COL6, HIGH);
  digitalWrite(PIN_COL7, HIGH);
  digitalWrite(PIN_COL8, HIGH);
  
    switch(column)
    {
      case 0: digitalWrite(PIN_COL1, LOW); break;
      case 1: digitalWrite(PIN_COL2, LOW); break;
      case 2: digitalWrite(PIN_COL3, LOW); break;
      case 3: digitalWrite(PIN_COL4, LOW); break;
      case 4: digitalWrite(PIN_COL5, LOW); break;
      case 5: digitalWrite(PIN_COL6, LOW); break;
      case 6: digitalWrite(PIN_COL7, LOW); break;
      case 7: digitalWrite(PIN_COL8, LOW); break;
      default: break;
    }

    switch(row)
    {
      case 0: digitalWrite(PIN_ROW1, HIGH); break;
      case 1: digitalWrite(PIN_ROW2, HIGH); break;
      case 2: digitalWrite(PIN_ROW3, HIGH); break;
      case 3: digitalWrite(PIN_ROW4, HIGH); break;
      case 4: digitalWrite(PIN_ROW5, HIGH); break;
      case 5: digitalWrite(PIN_ROW6, HIGH); break;
      case 6: digitalWrite(PIN_ROW7, HIGH); break;
      case 7: digitalWrite(PIN_ROW8, HIGH); break;
      default: break;
   }
}

bool foodPositionIsValid()
{
  if (food.x < 0 || food.y < 0) return false;

  for (int i = tail; i <= (head > tail ? head : MAX_BODY_LENGTH - 1); i++)
  {
    if (body[i].x == food.x && body[i].y == food.y) return false;
  }

  if (head < tail)
  {
    for (int i = 0; i <= head; i++)
    {
      if (body[i].x == food.x && body[i].y == food.y) return false;
    }
  }
  return true;
}

void checkGameover()
{
  for (int i = tail; i <= (head > tail ? head - 1 : MAX_BODY_LENGTH - 1); i++)
  {
    if (body[head].x == body[i].x && body[head].y == body[i].y)
    {
      gameover = GAME_OVER_TIME;
        return;
    }
  }

  if (head < tail)
  {
    for (int i = 0; i < head; i++)
    {
      if (body[head].x == body[i].x && body[head].y == body[i].y)
      {
        gameover = GAME_OVER_TIME;
        return;
      }
    }
  }
}

void spawnFood()
{
  while (!foodPositionIsValid())
    food = { random(GAME_AREA_WIDTH), random(GAME_AREA_HEIGHT) };
}


void draw()
{
  for (int i = tail; i <= (head > tail ? head : MAX_BODY_LENGTH - 1); i++)
  {
    setPixel(body[i].x, body[i].y);
  }
  if (head < tail)
  {
    for (int i = 0; i <= head; i++)
    {
      setPixel(body[i].x, body[i].y);
    }
  }
  setPixel(food.x, food.y);
}

void move()
{
  tail = tail + 1 == MAX_BODY_LENGTH ? 0 : tail + 1;
  Position prevHead = body[head];
  head = head + 1 == MAX_BODY_LENGTH ? 0 : head + 1;
  body[head] = { prevHead.x + (direction == DIRECTION_LEFT ? -1 : (direction == DIRECTION_RIGHT ? 1 : 0)), prevHead.y + (direction == DIRECTION_UP ? -1 : (direction == DIRECTION_DOWN ? 1 : 0)) };
  body[head].x = body[head].x < 0 ? GAME_AREA_WIDTH - 1 : (body[head].x >= GAME_AREA_WIDTH ? 0 : body[head].x);
  body[head].y = body[head].y < 0 ? GAME_AREA_HEIGHT - 1 : (body[head].y >= GAME_AREA_HEIGHT ? 0 : body[head].y);
}

void eat()
{
  if (body[head].x == food.x && body[head].y == food.y)
  {
    if (bodyLength < MAX_BODY_LENGTH)
    {
      bodyLength++;
      tail--;
      if (tail < 0) tail = MAX_BODY_LENGTH - 1;
    }
    score++;
    food = { -1, -1 };
    spawnFood();
  }
}

void reset()
{
  body[0] = { 3,3 };
  body[1] = { 3,4 };
  body[2] = { 3,5 };
  bodyLength = 3;
  head = 2;
  tail = 0;
  direction = DIRECTION_DOWN;
  food = {-1, -1};
  gameover = 0;
  elapsedTime = 0;
  score = 0;
  spawnFood();
  readInput = true;
}

void setup(){    
 
  pinMode(PIN_ROW1, OUTPUT);
  pinMode(PIN_ROW2, OUTPUT);
  pinMode(PIN_ROW3, OUTPUT);
  pinMode(PIN_ROW4, OUTPUT);
  pinMode(PIN_ROW5, OUTPUT);
  pinMode(PIN_ROW6, OUTPUT);
  pinMode(PIN_ROW7, OUTPUT);
  pinMode(PIN_ROW8, OUTPUT);
  pinMode(PIN_COL1, OUTPUT);
  pinMode(PIN_COL2, OUTPUT);
  pinMode(PIN_COL3, OUTPUT);
  pinMode(PIN_COL4, OUTPUT);
  pinMode(PIN_COL5, OUTPUT);
  pinMode(PIN_COL6, OUTPUT);
  pinMode(PIN_COL7, OUTPUT);
  pinMode(PIN_COL8, OUTPUT);

  pinMode(PIN_INPUT_LEFT, INPUT);
  pinMode(PIN_INPUT_RIGHT, INPUT);
  
  reset(); 
}


void loop(){   

    unsigned long currentTime = millis(); 

    if(!gameover)
    {
        draw();
        elapsedTime += currentTime - previousTime;
        if(elapsedTime > 500)
        {
            move();
            eat();
            checkGameover();
            elapsedTime = 0;
            readInput = true;
        }
        if(readInput)
        {
          if(digitalRead(PIN_INPUT_RIGHT) && !lastInput) {  direction = (direction + 1) % 4; readInput = false; }
          if(digitalRead(PIN_INPUT_LEFT) && !lastInput) { direction = (4 + direction-1) % 4; readInput = false; }
        }
        lastInput = digitalRead(PIN_INPUT_RIGHT) || digitalRead(PIN_INPUT_LEFT);
        
    }
    else
    {
        if(currentTime % 800 > 400) draw();
        gameover -= currentTime - previousTime;
        
        if(gameover <= 0) 
           reset();
    }
      
    previousTime = currentTime;
} 
