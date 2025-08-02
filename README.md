# Оповещение о завершении стирки стиральной машины на Arduino

Этот проект Arduino воспроизводит мелодию после завершения цикла стирки, определяя состояние светодиода окончания стирки стиральной машины.
Считывает только стабильное свечение в течении 3 секунд непрерывно (у некоторых машинок светодиод окончания скомбинен с какой-то херней и он мигает)
Внимание! В некоторых машинках LED окончания стирки идет через ШИМ сигнал. Тут ничего не поделать и этот код не подойдет - надо лепить поверх такого светодиода фототранзистор/резистор, считывать свечение, гнать в операционник и кидать на аналоговый сигнал Arduino. 

## Характеристики

- Обнаружение постоянного свечения LED в течение 3 секунд
- Воспроизведение мелодии при завершении
- Аппаратная развязка через оптопару
- Поддержка отладки через Serial Monitor
- Срет в Serial Port для дебага

## Аппаратные требования

- Arduino Nano (или совместимая плата)
- Оптопара PC817 (или аналогичная 4-контактная)
- Резистор 1 кОм
- Зуммер или пьезо-динамик
- Макетная плата и провода

## Схема подключения

- **BUZZER_PIN (D4)** → зуммер
- **FINISH_TRIGGER_PIN (D12)** → выход оптопары (через подтяжку к +5V или INPUT_PULLUP)

## Код прошивки

```cpp
#define BUZZER_PIN 4
#define FINISH_TRIGGER_PIN 12

// Ноты (значения в Гц)
#define NOTE_D5  587
#define NOTE_E5  659
#define NOTE_FS5 740
#define NOTE_G5  784
#define NOTE_A5  880

const int melody[] = {
  NOTE_D5, -4, NOTE_A5, 8, NOTE_FS5, 8, NOTE_D5, 8,
  NOTE_E5, -4, NOTE_FS5, 8, NOTE_G5, 4,
  NOTE_FS5, -4, NOTE_E5, 8, NOTE_FS5, 4,
  NOTE_D5, -2
};

// Настройки
#define REQUIRED_TIME 3000    // 3 секунды
#define CHECK_INTERVAL 50     // Интервал проверки
#define TEMPO 75              // Скорость мелодии

bool melodyPlayed = false;
unsigned long ledOnTime = 0;

void setup() {
  pinMode(FINISH_TRIGGER_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("System initialized (FINAL FIX)");
}

void loop() {
  bool ledState = !digitalRead(FINISH_TRIGGER_PIN);
  unsigned long currentTime = millis();

  // Логика срабатывания
  if (ledState && !melodyPlayed) {
    if (ledOnTime == 0) {
      ledOnTime = currentTime;
      Serial.println("LED ON detected, starting timer");
    } 
    else if (currentTime - ledOnTime >= REQUIRED_TIME) {
      Serial.println("Condition met, playing melody");
      playMelody();
      melodyPlayed = true;
    }
  } 
  else if (!ledState) {
    ledOnTime = 0;
    melodyPlayed = false;
  }

  delay(CHECK_INTERVAL);
}

void playMelody() {
  int notesCount = sizeof(melody) / sizeof(melody[0]);
  long wholenote = (60000 * 4) / TEMPO;
  
  for (int i = 0; i < notesCount; i += 2) {
    // Проверка прерывания ДО воспроизведения ноты
    if (digitalRead(FINISH_TRIGGER_PIN) == HIGH) { // Если светодиод выключился
      noTone(BUZZER_PIN);
      Serial.println("Melody interrupted");
      return;
    }
    
    int noteDuration = wholenote / abs(melody[i+1]);
    tone(BUZZER_PIN, melody[i], noteDuration);
    delay(noteDuration * 1.1);
    noTone(BUZZER_PIN);
    
    // Короткая пауза между нотами
    delay(30);
  }
  Serial.println("Melody completed");
}
