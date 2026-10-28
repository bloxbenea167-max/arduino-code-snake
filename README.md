# arduino-code-snake
so do make it put a arduino uno r4 wifi and a joystic connected to 5v and gnd and a0 a1

// Arduino Uno R4 WiFi Snake Game with Joystick Control
// Joystick connections: VRx to A0, VRy to A1, +5V to 5V, GND to GND
// Uses built in LED matrix for game display

#include "Arduino_LED_Matrix.h"

ArduinoLEDMatrix matrix;

// Joystick pins
const int JOY_X = A0;  // Analog pin for X axis
const int JOY_Y = A1;  // Analog pin for Y axis

// Game constants
const int GRID_WIDTH = 8;
const int GRID_HEIGHT = 8;
const int MAX_SNAKE_LENGTH = 32;
const int MOVE_DELAY = 300;  // Milliseconds between snake moves

// Game state
struct SnakeSegment {
  int x;
  int y;
};

SnakeSegment snake[MAX_SNAKE_LENGTH];
int snakeLength = 3;
int dirX = 1;   // Current direction X
int dirY = 0;   // Current direction Y
int nextDirX = 1;   // Next direction X (buffered input)
int nextDirY = 0;   // Next direction Y (buffered input)

int foodX = 5;
int foodY = 5;

int score = 0;
bool gameOver = false;
unsigned long lastMoveTime = 0;

// LED matrix buffer (8x8 grid)
uint8_t frame[8][8];

void setup() {
  Serial.begin(9600);
  delay(1000);
  
  // Initialize LED matrix
  matrix.begin();
  
  // Set joystick pins as inputs
  pinMode(JOY_X, INPUT);
  pinMode(JOY_Y, INPUT);
  
  // Initialize snake in the middle
  snake[0].x = 4;
  snake[0].y = 4;
  snake[1].x = 3;
  snake[1].y = 4;
  snake[2].x = 2;
  snake[2].y = 4;
  
  Serial.println("Snake Game Started!");
  Serial.println("Use joystick to control the snake.");
  
  generateFood();
}

void loop() {
  // Read joystick input and update direction
  readJoystickInput();
  
  // Update game state at regular intervals
  unsigned long currentTime = millis();
  if (currentTime - lastMoveTime >= MOVE_DELAY) {
    lastMoveTime = currentTime;
    
    if (!gameOver) {
      updateSnakePosition();
      checkCollisions();
    } else {
      // Reset game on any input if game over
      if (dirX != 0 || dirY != 0) {
        resetGame();
      }
    }
  }
  
  // Render the game
  renderGame();
  
  delay(50);  // Small delay for stability
}

void readJoystickInput() {
  int xValue = analogRead(JOY_X);
  int yValue = analogRead(JOY_Y);
  
  // Joystick center is around 512, threshold for detection is about 300
  int xThreshold = 300;
  int yThreshold = 300;
  
  // Determine intended movement from joystick position
  int tempDirX = 0;
  int tempDirY = 0;
  
  if (xValue < 512 - xThreshold) {
    tempDirX = -1;  // Left
    tempDirY = 0;
  } else if (xValue > 512 + xThreshold) {
    tempDirX = 1;   // Right
    tempDirY = 0;
  } else if (yValue < 512 - yThreshold) {
    tempDirY = -1;  // Up
    tempDirX = 0;
  } else if (yValue > 512 + yThreshold) {
    tempDirY = 1;   // Down
    tempDirX = 0;
  }
  
  // Prevent the snake from reversing into itself
  if (tempDirX != 0 || tempDirY != 0) {
    // Only change direction if not going opposite to current direction
    if ((tempDirX != -dirX) || (tempDirY != -dirY)) {
      nextDirX = tempDirX;
      nextDirY = tempDirY;
    }
  }
}

void updateSnakePosition() {
  // Update direction
  dirX = nextDirX;
  dirY = nextDirY;
  
  // Calculate new head position
  int newHeadX = snake[0].x + dirX;
  int newHeadY = snake[0].y + dirY;
  
  // Check if head eats food
  if (newHeadX == foodX && newHeadY == foodY) {
    // Grow snake
    for (int i = snakeLength; i > 0; i--) {
      snake[i].x = snake[i - 1].x;
      snake[i].y = snake[i - 1].y;
    }
    snake[0].x = newHeadX;
    snake[0].y = newHeadY;
    snakeLength++;
    score += 10;
    
    Serial.print("Food eaten! Score: ");
    Serial.println(score);
    
    generateFood();
  } else {
    // Move snake (remove tail, add head)
    for (int i = snakeLength - 1; i > 0; i--) {
      snake[i].x = snake[i - 1].x;
      snake[i].y = snake[i - 1].y;
    }
    snake[0].x = newHeadX;
    snake[0].y = newHeadY;
  }
}

void checkCollisions() {
  int headX = snake[0].x;
  int headY = snake[0].y;
  
  // Check wall collision
  if (headX < 0 || headX >= GRID_WIDTH || headY < 0 || headY >= GRID_HEIGHT) {
    gameOver = true;
    Serial.print("Game Over! Final Score: ");
    Serial.println(score);
    return;
  }
  
  // Check self collision
  for (int i = 1; i < snakeLength; i++) {
    if (headX == snake[i].x && headY == snake[i].y) {
      gameOver = true;
      Serial.print("Game Over! Final Score: ");
      Serial.println(score);
      return;
    }
  }
}

void generateFood() {
  bool foodOnSnake = true;
  
  while (foodOnSnake) {
    foodX = random(0, GRID_WIDTH);
    foodY = random(0, GRID_HEIGHT);
    
    foodOnSnake = false;
    for (int i = 0; i < snakeLength; i++) {
      if (foodX == snake[i].x && foodY == snake[i].y) {
        foodOnSnake = true;
        break;
      }
    }
  }
}

void renderGame() {
  // Clear frame buffer
  for (int y = 0; y < 8; y++) {
    for (int x = 0; x < 8; x++) {
      frame[y][x] = 0;
    }
  }
  
  if (!gameOver) {
    // Draw snake
    for (int i = 0; i < snakeLength; i++) {
      if (snake[i].x >= 0 && snake[i].x < GRID_WIDTH &&
          snake[i].y >= 0 && snake[i].y < GRID_HEIGHT) {
        frame[snake[i].y][snake[i].x] = 255;  // Full brightness
      }
    }
    
    // Draw food (blink effect)
    if ((millis() / 200) % 2 == 0) {
      if (foodX >= 0 && foodX < GRID_WIDTH &&
          foodY >= 0 && foodY < GRID_HEIGHT) {
        frame[foodY][foodX] = 128;  // Half brightness for food
      }
    }
  } else {
    // Game over display - blinking pattern
    if ((millis() / 300) % 2 == 0) {
      for (int y = 0; y < 8; y++) {
        for (int x = 0; x < 8; x++) {
          if ((x + y) % 2 == 0) {
            frame[y][x] = 200;
          }
        }
      }
    }
  }
  
  // Send frame to matrix
  matrix.renderBitmap((uint8_t *)frame, 8, 8);
}

void resetGame() {
  snakeLength = 3;
  dirX = 1;
  dirY = 0;
  nextDirX = 1;
  nextDirY = 0;
  score = 0;
  gameOver = false;
  
  // Reset snake position
  snake[0].x = 4;
  snake[0].y = 4;
  snake[1].x = 3;
  snake[1].y = 4;
  snake[2].x = 2;
  snake[2].y = 4;
  
  generateFood();
  and if is not working just talk to me
  Serial.println("Game Reset!");
}
