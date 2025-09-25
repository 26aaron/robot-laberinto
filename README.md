/*
  Startlight - Código base Arduino Nano
  Robot laberinto (2x HC-SR04, 2 motores DC con L298N)
  Algoritmo: wall-following (derecha) simple y reactivo.

  Wiring (ejemplo):
  - Sensor frontal HC-SR04:
    trigFront -> D2
    echoFront -> D3
  - Sensor derecha HC-SR04:
    trigRight -> D4
    echoRight -> D5

  - Motor A (rueda izquierda) conectado a OUT1/OUT2 del L298N:
    in1 -> D8
    in2 -> D9
    ena (PWM) -> D5  (si usás D5 como PWM, ajustar pines si hay conflicto con sensores)
  - Motor B (rueda derecha) conectado a OUT3/OUT4 del L298N:
    in3 -> D10
    in4 -> D11
    enb (PWM) -> D6

  NOTA: Si tus pines PWM o digital difieren en la Nano, adaptalos.
*/

const int trigFront = 2;
const int echoFront = 3;
const int trigRight = 4;
const int echoRight = 5;

// Motor left (A)
const int in1 = 8;
const int in2 = 9;
const int ena = 5; // PWM

// Motor right (B)
const int in3 = 10;
const int in4 = 11;
const int enb = 6; // PWM

// Parámetros (ajustar)
const int SPEED = 160;         // 0 - 255 PWM velocidad base
const int TURN_SPEED = 150;    // velocidad durante giros
const int DIST_FRONT = 18;     // distancia mínima frontal en cm (si < => obstáculo)
const int DIST_SIDE = 12;      // distancia deseada a la pared derecha en cm
const int TURN_DELAY = 300;    // ms para giros cortos
const int BACKUP_TIME = 250;   // ms para retroceder cuando esté atrapado

// Funcion lectura HC-SR04
long readUltrasonicCM(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duration = pulseIn(echoPin, HIGH, 30000); // timeout 30 ms
  long cm = duration / 58; // fórmula típica
  if (cm == 0) cm = 300; // si no lee, devolver valor grande
  return cm;
}

// Movimientos básicos
void forwardMotors(int pwm) {
  // Left forward
  digitalWrite(in1, HIGH);
  digitalWrite(in2, LOW);
  analogWrite(ena, pwm);
  // Right forward
  digitalWrite(in3, HIGH);
  digitalWrite(in4, LOW);
  analogWrite(enb, pwm);
}

void backwardMotors(int pwm) {
  digitalWrite(in1, LOW);
  digitalWrite(in2, HIGH);
  analogWrite(ena, pwm);
  digitalWrite(in3, LOW);
  digitalWrite(in4, HIGH);
  analogWrite(enb, pwm);
}

void stopMotors() {
  analogWrite(ena, 0);
  analogWrite(enb, 0);
  digitalWrite(in1, LOW);
  digitalWrite(in2, LOW);
  digitalWrite(in3, LOW);
  digitalWrite(in4, LOW);
}

void turnRightInPlace(int pwm) {
  // Left forward, right backward => gira a la derecha
  digitalWrite(in1, HIGH);
  digitalWrite(in2, LOW);
  analogWrite(ena, pwm);
  digitalWrite(in3, LOW);
  digitalWrite(in4, HIGH);
  analogWrite(enb, pwm);
}

void turnLeftInPlace(int pwm) {
  // Left backward, right forward => gira a la izquierda
  digitalWrite(in1, LOW);
  digitalWrite(in2, HIGH);
  analogWrite(ena, pwm);
  digitalWrite(in3, HIGH);
  digitalWrite(in4, LOW);
  analogWrite(enb, pwm);
}

void setup() {
  // Sensores
  pinMode(trigFront, OUTPUT);
  pinMode(echoFront, INPUT);
  pinMode(trigRight, OUTPUT);
  pinMode(echoRight, INPUT);

  // Motores
  pinMode(in1, OUTPUT);
  pinMode(in2, OUTPUT);
  pinMode(ena, OUTPUT);
  pinMode(in3, OUTPUT);
  pinMode(in4, OUTPUT);
  pinMode(enb, OUTPUT);

  Serial.begin(9600);
  stopMotors();
  delay(200);
}

unsigned long lastPrint = 0;

void loop() {
  long distF = readUltrasonicCM(trigFront, echoFront);   // frontal
  long distR = readUltrasonicCM(trigRight, echoRight);   // derecha

  // debug
  if (millis() - lastPrint > 300) {
    Serial.print("Front: "); Serial.print(distF);
    Serial.print(" cm  | Right: "); Serial.print(distR);
    Serial.println(" cm");
    lastPrint = millis();
  }

  // Lógica simple:
  // 1) Si la derecha está libre (distR > DIST_SIDE + margen) => girar a la derecha y avanzar
  // 2) Si delante libre (distF > DIST_FRONT) => avanzar
  // 3) Si delante obstáculo => girar a la izquierda
  // Caso extremo: si está atrapado, retroceder y girar

  if (distR > (DIST_SIDE + 8) && distF > DIST_FRONT) {
    // oportunidad de girar a la derecha y seguir pared derecha
    turnRightInPlace(TURN_SPEED);
    delay(TURN_DELAY);
    forwardMotors(SPEED);
    delay(200);
  } else if (distF > DIST_FRONT) {
    // delante libre -> avanza
    forwardMotors(SPEED);
  } else {
    // obstáculo frontal
    stopMotors();
    delay(50);
    // pequeña maniobra: retroceder un poco
    backwardMotors(SPEED);
    delay(BACKUP_TIME);
    stopMotors();
    delay(50);
    // ahora girar a la izquierda hasta que el frente esté libre
    turnLeftInPlace(TURN_SPEED);
    delay(250);
    // si sigue muy cerca, hacer giro mayor
    if (readUltrasonicCM(trigFront, echoFront) <= DIST_FRONT) {
      delay(200);
    }
    stopMotors();
    delay(50);
  }

  delay(30); // loop corto para estabilidad
}
