#include <Keypad.h>

// Настройка клавиатуры
const byte ROWS = 4; // 4 строки
const byte COLS = 4; // 4 колонки
char keys[ROWS][COLS] = {
  {'7', '8', '9', '/'},
  {'4', '5', '6', '*'},
  {'1', '2', '3', '-'},
  {'C', '0', '=', '+'}
};
byte rowPins[ROWS] = {22, 23, 24, 25}; // Пины строк
byte colPins[COLS] = {26, 27, 28, 29}; // Пины колонок
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Переменные для хранения введенного числа
int32_t enteredNumber = 0; // Хранит введенное число
byte numDigits = 0;     // Количество введенных цифр

// Переменные для хранения состояния калькулятора
int32_t firstOperand = 0;  // Первый операнд
int32_t secondOperand = 0; // Второй операнд
char operation = '\0';     // Текущая операция
bool isSecondOperand = false; // Флаг, указывающий, что вводится второй операнд

// Определяем пины для сегментов A-G
const int segmentPins[] = {30, 31, 32, 33, 34, 35, 36}; // A, B, C, D, E, F, G

// Определяем пины для разрядов (Digit 1-6)
const int digitPins[] = {38, 39, 40, 41, 42, 43}; // Digit 1, Digit 2, Digit 3, Digit 4, Digit 5, Digit 6

// Определяем, какие сегменты должны быть включены для каждой цифры (0-9)
const byte digitSegments[10] = {
  B1111110, // 0 (A, B, C, D, E, F, G)
  B0110000, // 1 (B, C)
  B1101101, // 2 (A, B, G, E, D)
  B1111001, // 3 (A, B, G, C, D)
  B0110011, // 4 (F, G, B, C)
  B1011011, // 5 (A, F, G, C, D)
  B1011111, // 6 (A, F, G, C, D, E)
  B1110000, // 7 (A, B, C)
  B1111111, // 8 (A, B, C, D, E, F, G)
  B1111011  // 9 (A, B, C, D, F, G)
};

/**
 * Применяет переполнение для 6-значного числа.
 * Если число отрицательное, возвращает его дополнение до 999999.
 * Если число положительное, возвращает его без изменений.
 * Входные данные:
 * - number: число (int32_t), к которому нужно применить переполнение.
 * Выходные данные:
 * - int32_t: число после применения переполнения.
 */
int32_t applyOverflow(int32_t number) {
  if (number < 0) {
    return 999999 + number + 1; // Дополнение до 999999
  }
  return number;
}

/**
 * Отображает цифру на конкретном разряде дисплея.
 * Входные данные:
 * - digit: номер разряда (0-5), на котором нужно отобразить цифру.
 * - number: цифра (0-9) для отображения. Если number = 10, разряд отключается.
 * Выходные данные: отсутствуют.
 */
void displayDigit(int digit, int number) {
  // Включаем только текущий разряд
  for (int i = 0; i < 6; i++) {
    digitalWrite(digitPins[i], LOW); // Выключаем все разряды (HIGH > LOW)
  }
  if (number == 10) return;
  digitalWrite(digitPins[digit], HIGH); // Включаем текущий разряд (LOW > HIGH)

  // Включаем сегменты для отображения цифры
  for (int i = 0; i < 7; i++) {
    digitalWrite(segmentPins[i], !bitRead(digitSegments[number], 6 - i)); // Инвертируем биты (HIGH - LOW)
  }
}

/**
 * Отображает число на шестиразрядном дисплее.
 * Входные данные:
 * - number: число (int32_t), которое нужно отобразить.
 * Выходные данные: отсутствуют.
 */
void displayNumber(int32_t number) {
  number = applyOverflow(number); // Применяем переполнение
  int digits[6] = {0}; // Массив для хранения цифр числа
  int numDigits = 0;   // Количество цифр в числе

  // Если число равно 0, отображаем 0
  if (number == 0) {
    digits[5] = 0;
    numDigits = 1;
  } else {
    // Разбиваем число на цифры
    while (number > 0 && numDigits < 6) {
      digits[5 - numDigits] = number % 10; // Заполняем массив 
      number /= 10;
      numDigits++;
    }
  }

  // Отображаем цифры на дисплее
  for (int i = 0; i < 6; i++) {
    if (i >= 6 - numDigits) { // Отображаем только нужные разряды
      displayDigit(i, digits[i]);
    } else {
      displayDigit(i, 10); // 10 означает "пусто" (ничего не отображать)
    }
    delay(7); // Небольшая задержка для стабильного отображения
  }
}

/**
 * Вычисляет результат операции между двумя операндами.
 * Входные данные:
 * - first: первый операнд (int32_t).
 * - second: второй операнд (int32_t).
 * - op: операция (char), может быть '+', '-', '*', '/'.
 * Выходные данные:
 * - int32_t: результат вычисления.
 */
int32_t calculateResult(int32_t first, int32_t second, char op) {
  switch (op) {
    case '+': return first + second;
    case '-': return first - second;
    case '*': return first * second;
    case '/': 
      if (second == 0) {
        return 999999; // Деление на ноль: возвращаем переполнение
      }
      return first / second;
    default: return 0;
  }
}

/**
 * Настройка пинов и начальное состояние калькулятора.
 * Входные данные: отсутствуют.
 * Выходные данные: отсутствуют.
 */
void setup() {
  // Настраиваем пины сегментов и разрядов как выходы
  for (int i = 0; i < 7; i++) {
    pinMode(segmentPins[i], OUTPUT);
    digitalWrite(segmentPins[i], HIGH); // Выключаем все сегменты (LOW > HIGH)
  }
  for (int i = 0; i < 6; i++) {
    pinMode(digitPins[i], OUTPUT);
    digitalWrite(digitPins[i], LOW); // Выключаем все разряды (HIGH > LOW)
  }
}

/**
 * Основной цикл программы. Обрабатывает ввод с клавиатуры и управляет логикой калькулятора.
 * Входные данные: отсутствуют.
 * Выходные данные: отсутствуют.
 */
void loop() {
  char key = keypad.getKey(); // Считываем нажатую клавишу

  if (key) {
    if (key >= '0' && key <= '9') { // Если нажата цифра
      if (numDigits < 6) { // Ограничение на 6 цифр
        enteredNumber = enteredNumber * 10 + (key - '0'); // Добавляем цифру к числу
        numDigits++; // Увеличиваем счетчик цифр
      }
    } else if (key == 'C') { // Если нажата клавиша 'C' (очистка)
      enteredNumber = 0; // Сбрасываем число
      numDigits = 0; // Сбрасываем счетчик цифр
      firstOperand = 0;
      secondOperand = 0;
      operation = '\0';
      isSecondOperand = false;
    } else if (key == '+' || key == '-' || key == '*' || key == '/') { // Если нажата операция
      if (!isSecondOperand) {
        firstOperand = enteredNumber;
        operation = key;
        isSecondOperand = true;
      } else {
        secondOperand = enteredNumber;
        firstOperand = calculateResult(firstOperand, secondOperand, operation);
        operation = key;
        secondOperand = 0;
      }
      enteredNumber = 0;
      numDigits = 0;
    } else if (key == '=') { // Если нажата клавиша '='
      if (isSecondOperand) {
        secondOperand = enteredNumber;
        enteredNumber = calculateResult(firstOperand, secondOperand, operation);
        firstOperand = enteredNumber;
        isSecondOperand = false;
        numDigits = 0;
      }
    }
  }
  displayNumber(enteredNumber); // Отображаем число на дисплее
}