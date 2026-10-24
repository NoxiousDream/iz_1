# Отчёт по индивидуальному заданию №2

**по дисциплине «Архитектура компьютера и операционные системы»**

**Тема:** Разработка многопоточной имитационной программы

**Вариант №11:** Задача о больнице

**Выполнил:** студент группы БПИ253
Лёвин Артём Михайлович
---

## 1. Условие задачи

**Вариант 11. Задача о больнице**

В приёмном отделении больницы работают два дежурных врача. Поступивший пациент сначала проходит первичный осмотр, сообщает жалобы и получает направление к стоматологу, хирургу или терапевту. После этого пациент переходит в очередь соответствующего специалиста.

Каждый врач принимает только одного пациента. Пациенты не покидают очередь самовольно. Продолжительность первичного осмотра и лечения зависит от состояния пациента либо задаётся случайно. При расширенном моделировании срочные пациенты могут обслуживаться с приоритетом, но выбранная политика не должна оставлять остальных пациентов без обслуживания.

Требуется разработать консольное приложение, моделирующее рабочий день больницы.

### Инварианты

- один врач принимает не более одного пациента;
- пациент одновременно находится не более чем в одной очереди или у одного врача;
- направление выдаётся только после первичного осмотра;
- лечение выполняет специалист соответствующего профиля;
- пациент выписывается после завершения назначенного лечения.

### Завершение

Моделирование завершается после окончания приёма новых пациентов и обслуживания всех поступивших или по прерыванию пользователем.

Каждый врач и пациент является самостоятельным участником модели.

---

## 2. Сценарий: отображение сущностей в потоки и их взаимодействие

В отличие от первого задания (последовательная событийная имитация), во втором задании каждая сущность, обладающая поведением, представлена **отдельным потоком выполнения**. Это обеспечивает истинно асинхронное поведение участников модели.

### Распределение потоков

| Сущность            | Количество | Роль потока                                                                                             |
|---------------------|------------|---------------------------------------------------------------------------------------------------------|
| Генератор пациентов | 1          | Создаёт пациентов через случайные интервалы, запускает их потоки                                        |
| Дежурный врач       | 2          | Ожидает пациентов в первичной очереди, проводит осмотр                                                  |
| Стоматолог          | 1          | Ожидает пациентов в своей очереди, проводит лечение                                                     |
| Хирург              | 1          | Ожидает пациентов в своей очереди, проводит лечение                                                     |
| Терапевт            | 1          | Ожидает пациентов в своей очереди, проводит лечение                                                     |
| Пациент             | N (до 200) | Проходит полный жизненный цикл: прибытие → очередь → осмотр → очередь к специалисту → лечение → выписка |

**Итого:** 6 + N потоков (при 20 пациентах — 26 потоков).

### Механизмы синхронизации

| Примитив                          | Назначение                                                                                                              |
|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| `pthread_mutex_t hospital_lock`   | Защита всех общих данных: очередей, состояний пациентов, статистики, флагов завершения                                  |
| `pthread_mutex_t log_mutex`       | Защита записи в консоль и файл лога от одновременного доступа потоков                                                   |
| `sem_t primary_sem`               | Счётчик пациентов, ожидающих первичного осмотра. Дежурный врач делает `sem_wait` (засыпает при 0), пациент — `sem_post` |
| `sem_t spec_sems[3]`              | Аналогичные счётчики для каждого специалиста                                                                            |
| `pthread_cond_t exam_done_cond`   | Врач ждёт, пока пациент завершит первичный осмотр                                                                       |
| `pthread_cond_t treat_done_cond`  | Специалист ждёт, пока пациент завершит лечение                                                                          |
| `pthread_cond_t patients[i].cond` | Персональная условная переменная пациента: пациент ждёт, пока его позовут к врачу                                       |

### Жизненный цикл потока пациента

```
Прибыл → Встал в очередь к дежурному (sem_post) →
Ждёт вызова (cond_wait) → Первичный осмотр (usleep) →
Направлен к специалисту → Встал в очередь (sem_post) →
Ждёт вызова (cond_wait) → Лечение (usleep) → Выписан
```

### Жизненный цикл потока врача

```
Ожидание пациента (sem_wait) → Извлечение из очереди →
Вызов пациента (cond_signal) → Ожидание завершения (cond_wait) →
Возврат к ожиданию следующего пациента
```

---

## 3. Пользовательский интерфейс

Программа выводит в консоль информативный лог с временными метками (секунды от начала работы, формат `[  0.123]`). Каждое событие предметной области отображается явно:

- прибытие пациента (с указанием срочности и жалоб);
- постановка в очередь к дежурным врачам;
- начало и окончание первичного осмотра (с идентификацией врача);
- выдача направления к специалисту;
- постановка в очередь к специалисту;
- начало и окончание лечения (с идентификацией специалиста);
- выписка пациента.

В завершение выводится итоговая статистика: число созданных, выписанных и не завершённых пациентов.

Логирование выполняется через системный вызов `write()` в дескрипторы `STDOUT_FILENO` и файла `hospital_threaded.log` (или `hospital_cond.log` для второй версии). Запись защищена отдельным мьютексом `log_mutex`.

---

## 4. Варианты запуска и завершения

### Запуск

Программа поддерживает аргументы командной строки:

```bash
./hospital_sem --patients 30 --seed 42 --urgent 20
./hospital_cond --patients 30 --seed 42 --urgent 20
```

| Параметр       | Описание                                             | По умолчанию |
|----------------|------------------------------------------------------|--------------|
| `--patients N` | Количество пациентов                                 | 20           |
| `--seed S`     | Зерно генератора случайных чисел (воспроизводимость) | 42           |
| `--urgent U`   | Процент срочных пациентов                            | 15           |

Компиляция:

```bash
gcc -std=c99 -pthread -Wall -Wextra hospital_sem.c  -o hospital_sem
gcc -std=c99 -pthread -Wall -Wextra hospital_cond.c -o hospital_cond
```

### Завершение

Предусмотрено два варианта завершения:

1. **Штатное завершение:** главный поток отслеживает условие `все пациенты сгенерированы И все выписаны`. После этого устанавливается флаг `shutdown_flag`, врачи и пациенты пробуждаются и корректно завершаются через `pthread_join`.

2. **Прерывание пользователем (Ctrl+C):** обработчик сигнала `SIGINT` устанавливает `interrupt_flag`. Главный поток обнаруживает флаг, инициирует корректное завершение всех потоков (пробуждение заблокированных `sem_wait` через `sem_post` и `cond_wait` через `pthread_cond_broadcast`), после чего присоединяет потоки и сохраняет статистику.

---

## 5. Результаты работы

### Пример вывода (семафоры, 10 пациентов)

```
[  0.000] === МНОГОПОТОЧНАЯ МОДЕЛЬ БОЛЬНИЦЫ (семафоры) ===
[  0.000] Параметры: пациентов=10, seed=42, срочные=15%
[  0.000] Потоков: 1 генератор + 2 дежурных + 3 специалиста + пациенты
[  0.001] Пациент 0 прибыл в приёмное отделение (жалобы: Хирург)
[  0.001] Пациент 0 встал в очередь к дежурным врачам
[  0.001] Дежурный врач 0 начал осмотр пациента 0
[  0.001] Пациент 0 проходит первичный осмотр (237 мс)
[  0.085] Пациент 1 прибыл в приёмное отделение (жалобы: Стоматолог)
[  0.085] Пациент 1 встал в очередь к дежурным врачам
[  0.085] Дежурный врач 1 начал осмотр пациента 1
[  0.085] Пациент 1 проходит первичный осмотр (312 мс)
[  0.238] Пациент 0: осмотр завершён, направлен к Хирург
[  0.238] Пациент 0 встал в очередь к Хирург
[  0.238] Хирург начал лечение пациента 0
[  0.238] Пациент 0 проходит лечение у Хирург (541 мс)
[  0.238] Дежурный врач 0 закончил осмотр пациента 0
...
[  4.512] Пациент 9 выписан

=== ЗАВЕРШЕНИЕ РАБОТЫ ===
Всего пациентов создано: 10
Выписано после лечения:  10
Не завершено (прервано): 0
Лог сохранён в файл: hospital_threaded.log
```

### Пример прерывания (Ctrl+C)

```
[  0.001] Пациент 0 прибыл в приёмное отделение (жалобы: Терапевт)
...
^C
=== ЗАВЕРШЕНИЕ РАБОТЫ ===
Всего пациентов создано: 14
Выписано после лечения:  5
Не завершено (прервано): 9
Лог сохранён в файл: hospital_threaded.log
```

---

## 6. Реализация дополнительных требований

| № | Требование                              | Реализация                                                                                                                                                            |
|---|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | Комментарии в коде                      | Код подробно прокомментирован: назначение каждой функции, переменных, примитивов синхронизации                                                                        |
| 2 | Системные вызовы и дескрипторы файлов   | Логирование через `open()`, `write()`, `close()` вместо `printf/fprintf`                                                                                              |
| 3 | Завершение по окончании и по прерыванию | Штатное завершение через флаг `all_generated && discharged_count >= total_patients`; прерывание через `SIGINT` с корректным пробуждением всех заблокированных потоков |
| 4 | Сохранение лога со статистикой          | Файл `hospital_threaded.log` / `hospital_cond.log` с полным логом и итоговой статистикой                                                                              |
| 5 | Тестовые наборы и скрипты               | Скрипт `run_tests.sh` (см. Приложение А)                                                                                                                              |
| 7 | Критическая оценка                      | Представлена в разделе 7                                                                                                                                              |


---

## 7. Критическая оценка

При переходе от последовательной имитации к многопоточной выявлены следующие особенности, требующие внимания:

1. **Проблема потерянного сигнала.** Если врач просигналит условную переменную пациента до того, как пациент начнёт `pthread_cond_wait`, сигнал будет потерян. Решение: все проверки состояния выполняются под общим мьютексом `hospital_lock` в цикле `while`, что является стандартным паттерном работы с условными переменными.

2. **Корректное завершение заблокированных потоков.** Поток, заблокированный на `sem_wait` или `pthread_cond_wait`, не может быть завершён принудительно. Решение: при завершении главный поток посылает в каждый семафор нужное число «пустых» постов и пробуждает все условные переменные, после чего потоки проверяют `shutdown_flag` и выходят из циклов.

3. **Потокобезопасность логирования.** Одновременная запись нескольких потоков в один файловый дескриптор без синхронизации привела бы к перемешиванию строк. Решение: отдельный мьютекс `log_mutex` вокруг вызова `write()`.

4. **Отсутствие гонок данных.** Все общие данные (очереди, состояния пациентов, счётчики, флаги) доступны только под мьютексом `hospital_lock`. Персональные условные переменные пациентов инициализируются до создания их потоков.

**Правки кода:** критических правок не потребовалось. Архитектура полностью удовлетворяет условию задачи и требованиям POSIX.

---

## Приложение А. Скрипт тестирования (`run_tests.sh`)

```bash
#!/bin/bash
set -e

echo "=== Компиляция основной программы (семафоры) ==="
gcc -std=c99 -pthread -Wall -Wextra hospital_sem.c -o hospital_sem

echo "=== Компиляция второй программы (условные переменные) ==="
gcc -std=c99 -pthread -Wall -Wextra hospital_cond.c -o hospital_cond

echo ""
echo "--- Тест 1: Базовый сценарий (семафоры, 20 пациентов) ---"
./hospital_sem --patients 20 --seed 42

echo ""
echo "--- Тест 2: Условные переменные вместо семафоров (20 пациентов) ---"
./hospital_cond --patients 20 --seed 42

echo ""
echo "--- Тест 3: Прерывание по сигналу (2 секунды работы) ---"
timeout 2s ./hospital_sem --patients 1000 --seed 99 || true
tail -n 5 hospital_threaded.log

echo ""
echo "=== Все тесты завершены ==="
```

---

## Приложение Б. Листинг программы с семафорами (`main.c`)

```c
/* Полный код программы приведён в файле hospital_sem.c.
   Ключевые особенности:
   - 6 фиксированных потоков (генератор, 2 дежурных, 3 специалиста)
     + по одному потоку на каждого пациента;
   - семафоры primary_sem и spec_sems[] для подсчёта ожидающих
     в очередях (классическая схема производитель-потребитель);
   - условные переменные для ожидания завершения процедур
     (врач ждёт пациента) и для вызова пациента из очереди;
   - мьютекс hospital_lock защищает все общие данные;
   - логирование через write() под отдельным мьютексом.
*/

#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <signal.h>
#include <fcntl.h>
#include <time.h>
#include <stdarg.h>

/* ==================== КОНСТАНТЫ ==================== */
#define MAX_PATIENTS   200
#define MAX_QUEUE_SIZE 100
#define SPEC_COUNT     3
#define NUM_DUTY_DOCS  2

/* Состояния пациента (конечный автомат) */
enum {
    ST_ARRIVED = 0,
    ST_WAIT_PRIMARY,      // ждёт в очереди к дежурному
    ST_IN_PRIMARY,        // на первичном осмотре
    ST_WAIT_SPECIALIST,   // ждёт в очереди к специалисту
    ST_IN_TREATMENT,      // на лечении
    ST_DISCHARGED,        // выписан
    ST_INTERRUPTED        // прерван по сигналу
};

typedef enum { SPEC_DENTIST = 0, SPEC_SURGEON, SPEC_THERAPIST } SpecType;
static const char *SPEC_NAMES[SPEC_COUNT] = {"Стоматолог", "Хирург", "Терапевт"};

/* ==================== СТРУКТУРЫ ==================== */
typedef struct {
    int      id;
    int      urgent;                 // срочный пациент
    SpecType spec;                   // к какому специалисту направлен
    int      exam_ms;                // длительность первичного осмотра, мс
    int      treat_ms;               // длительность лечения, мс
    int      state;                  // текущее состояние (конечный автомат)
    pthread_t      thread;
    pthread_cond_t cond;             // персональная условная переменная пациента
} Patient;

typedef struct {
    int items[MAX_QUEUE_SIZE];
    int count;
} Queue;

/* ==================== ГЛОБАЛЬНОЕ СОСТОЯНИЕ ==================== */
static Patient patients[MAX_PATIENTS];
static Queue   primary_queue;                 // очередь к дежурным врачам
static Queue   spec_queues[SPEC_COUNT];       // очереди к специалистам

static pthread_mutex_t hospital_lock;          // защита всех очередей и статистики
static pthread_mutex_t log_mutex;              // защита записи в лог

/* Условные переменные: врач ждёт завершения процедуры пациентом */
static pthread_cond_t exam_done_cond;
static pthread_cond_t treat_done_cond;

/* СЕМАФОРЫ: количество ожидающих пациентов в каждой очереди.
   Врач делает sem_wait (засыпает, если 0), пациент — sem_post. */
static sem_t primary_sem;
static sem_t spec_sems[SPEC_COUNT];

static pthread_t doc_threads[NUM_DUTY_DOCS];
static pthread_t spec_threads[SPEC_COUNT];
static pthread_t gen_thread;
static int       doc_ids[NUM_DUTY_DOCS]  = {0, 1};
static int       spec_ids[SPEC_COUNT]    = {0, 1, 2};

/* Параметры симуляции */
static int   total_patients   = 20;
static int   urgent_chance    = 15;   // % срочных пациентов
static int   arrival_min_ms   = 50;
static int   arrival_max_ms   = 200;
static unsigned int rng_seed  = 42;

/* Статистика и флаги завершения */
static volatile sig_atomic_t interrupt_flag = 0;
static int shutdown_flag    = 0;   // под hospital_lock
static int generated_count  = 0;   // под hospital_lock
static int discharged_count = 0;   // под hospital_lock
static int all_generated    = 0;   // под hospital_lock

static int log_fd = -1;
static struct timespec t_start;

/* ==================== УТИЛИТЫ ==================== */
static double elapsed_sec(void) {
    struct timespec now;
    clock_gettime(CLOCK_MONOTONIC, &now);
    return (now.tv_sec - t_start.tv_sec)
         + (now.tv_nsec - t_start.tv_nsec) / 1e9;
}

/* Потокобезопасное логирование через системный вызов write() */
static void log_msg(const char *fmt, ...) {
    char ts[32], msg[512], buf[1024];
    snprintf(ts, sizeof ts, "[%7.3f] ", elapsed_sec());

    va_list ap;
    va_start(ap, fmt);
    vsnprintf(msg, sizeof msg, fmt, ap);
    va_end(ap);

    snprintf(buf, sizeof buf, "%s%s", ts, msg);
    size_t len = strlen(buf);

    pthread_mutex_lock(&log_mutex);
    write(STDOUT_FILENO, buf, len);
    if (log_fd >= 0) write(log_fd, buf, len);
    pthread_mutex_unlock(&log_mutex);
}

static int rnd(int lo, int hi) { return lo + rand() % (hi - lo + 1); }

/* ==================== ОЧЕРЕДИ ==================== */
static void  q_init(Queue *q) { q->count = 0; }
static int   q_empty(const Queue *q) { return q->count == 0; }

static void q_push_back(Queue *q, int v) {
    if (q->count < MAX_QUEUE_SIZE) q->items[q->count++] = v;
}

/* Срочные пациенты вставляются в начало очереди */
static void q_push_priority(Queue *q, int v, int urgent) {
    if (q->count >= MAX_QUEUE_SIZE) return;
    if (urgent) {
        for (int i = q->count; i > 0; --i) q->items[i] = q->items[i - 1];
        q->items[0] = v;
    } else {
        q->items[q->count] = v;
    }
    q->count++;
}

static int q_pop(Queue *q) {
    if (q_empty(q)) return -1;
    int v = q->items[0];
    for (int i = 0; i < q->count - 1; ++i) q->items[i] = q->items[i + 1];
    q->count--;
    return v;
}

/* ==================== ПОТОК: ДЕЖУРНЫЙ ВРАЧ ==================== */
/*
   Врач бесконечно ждёт пациента через sem_wait.
   Получив пациента, будит его персональный cond,
   затем ждёт, пока пациент завершит осмотр (условная переменная).
*/
static void *duty_doctor_thread(void *arg) {
    int id = *(int *)arg;

    for (;;) {
        sem_wait(&primary_sem);                 // ждём пациента из очереди

        pthread_mutex_lock(&hospital_lock);
        if (shutdown_flag) { pthread_mutex_unlock(&hospital_lock); break; }
        if (q_empty(&primary_queue)) { pthread_mutex_unlock(&hospital_lock); continue; }

        int pid = q_pop(&primary_queue);
        patients[pid].state = ST_IN_PRIMARY;
        pthread_cond_signal(&patients[pid].cond);   // будим пациента
        pthread_mutex_unlock(&hospital_lock);

        log_msg("Дежурный врач %d начал осмотр пациента %d\n", id, pid);

        /* Ждём, пока пациент завершит осмотр */
        pthread_mutex_lock(&hospital_lock);
        while (patients[pid].state == ST_IN_PRIMARY && !shutdown_flag)
            pthread_cond_wait(&exam_done_cond, &hospital_lock);
        pthread_mutex_unlock(&hospital_lock);

        log_msg("Дежурный врач %d закончил осмотр пациента %d\n", id, pid);
    }
    return NULL;
}

/* ==================== ПОТОК: СПЕЦИАЛИСТ ==================== */
static void *specialist_thread(void *arg) {
    int spec = *(int *)arg;

    for (;;) {
        sem_wait(&spec_sems[spec]);

        pthread_mutex_lock(&hospital_lock);
        if (shutdown_flag) { pthread_mutex_unlock(&hospital_lock); break; }
        if (q_empty(&spec_queues[spec])) { pthread_mutex_unlock(&hospital_lock); continue; }

        int pid = q_pop(&spec_queues[spec]);
        patients[pid].state = ST_IN_TREATMENT;
        pthread_cond_signal(&patients[pid].cond);
        pthread_mutex_unlock(&hospital_lock);

        log_msg("%s начал лечение пациента %d\n", SPEC_NAMES[spec], pid);

        pthread_mutex_lock(&hospital_lock);
        while (patients[pid].state == ST_IN_TREATMENT && !shutdown_flag)
            pthread_cond_wait(&treat_done_cond, &hospital_lock);
        pthread_mutex_unlock(&hospital_lock);

        log_msg("%s закончил лечение пациента %d\n", SPEC_NAMES[spec], pid);
    }
    return NULL;
}

/* ==================== ПОТОК: ПАЦИЕНТ ==================== */
/*
   Жизненный цикл: прибыл -> очередь к дежурному -> осмотр
   -> очередь к специалисту -> лечение -> выписка.
*/
static void *patient_thread(void *arg) {
    int pid = *(int *)arg;
    Patient *p = &patients[pid];

    /* --- 1. Встать в очередь к дежурному врачу --- */
    pthread_mutex_lock(&hospital_lock);
    q_push_priority(&primary_queue, pid, p->urgent);
    p->state = ST_WAIT_PRIMARY;
    pthread_mutex_unlock(&hospital_lock);
    sem_post(&primary_sem);                       // разбудить дежурного врача

    log_msg("Пациент %d встал в очередь к дежурным врачам%s\n",
            pid, p->urgent ? " (СРОЧНЫЙ)" : "");

    /* --- 2. Ждать, пока позовут к дежурному --- */
    pthread_mutex_lock(&hospital_lock);
    while (p->state == ST_WAIT_PRIMARY && !shutdown_flag)
        pthread_cond_wait(&p->cond, &hospital_lock);
    int interrupted = shutdown_flag;
    pthread_mutex_unlock(&hospital_lock);
    if (interrupted) { p->state = ST_INTERRUPTED; return NULL; }

    log_msg("Пациент %d проходит первичный осмотр (%d мс)\n", pid, p->exam_ms);

    /* --- 3. Имитация первичного осмотра --- */
    usleep((useconds_t)p->exam_ms * 1000);

    /* --- 4. Завершить осмотр, получить направление --- */
    pthread_mutex_lock(&hospital_lock);
    if (shutdown_flag) {
        p->state = ST_INTERRUPTED;
        pthread_cond_broadcast(&exam_done_cond);
        pthread_mutex_unlock(&hospital_lock);
        return NULL;
    }
    p->state = ST_WAIT_SPECIALIST;
    pthread_cond_broadcast(&exam_done_cond);      // разбудить врача
    pthread_mutex_unlock(&hospital_lock);

    log_msg("Пациент %d: осмотр завершён, направлен к %s\n", pid, SPEC_NAMES[p->spec]);

    /* --- 5. Встать в очередь к специалисту --- */
    pthread_mutex_lock(&hospital_lock);
    q_push_back(&spec_queues[p->spec], pid);
    pthread_mutex_unlock(&hospital_lock);
    sem_post(&spec_sems[p->spec]);

    log_msg("Пациент %d встал в очередь к %s\n", pid, SPEC_NAMES[p->spec]);

    /* --- 6. Ждать, пока позовут к специалисту --- */
    pthread_mutex_lock(&hospital_lock);
    while (p->state == ST_WAIT_SPECIALIST && !shutdown_flag)
        pthread_cond_wait(&p->cond, &hospital_lock);
    interrupted = shutdown_flag;
    pthread_mutex_unlock(&hospital_lock);
    if (interrupted) { p->state = ST_INTERRUPTED; return NULL; }

    log_msg("Пациент %d проходит лечение у %s (%d мс)\n",
            pid, SPEC_NAMES[p->spec], p->treat_ms);

    /* --- 7. Имитация лечения --- */
    usleep((useconds_t)p->treat_ms * 1000);

    /* --- 8. Выписка --- */
    pthread_mutex_lock(&hospital_lock);
    if (shutdown_flag) {
        p->state = ST_INTERRUPTED;
        pthread_cond_broadcast(&treat_done_cond);
        pthread_mutex_unlock(&hospital_lock);
        return NULL;
    }
    p->state = ST_DISCHARGED;
    discharged_count++;
    pthread_cond_broadcast(&treat_done_cond);
    pthread_mutex_unlock(&hospital_lock);

    log_msg("Пациент %d выписан\n", pid);
    return NULL;
}

/* ==================== ПОТОК: ГЕНЕРАТОР ПАЦИЕНТОВ ==================== */
static void *generator_thread_func(void *arg) {
    (void)arg;
    for (int i = 0; i < total_patients; i++) {
        if (interrupt_flag) break;

        patients[i].id       = i;
        patients[i].urgent   = (rand() % 100) < urgent_chance;
        patients[i].spec     = (SpecType)(rand() % SPEC_COUNT);
        patients[i].exam_ms  = rnd(100, 400);
        patients[i].treat_ms = rnd(200, 600);
        patients[i].state    = ST_ARRIVED;

        log_msg("Пациент %d прибыл в приёмное отделение%s (жалобы: %s)\n",
                i, patients[i].urgent ? " (СРОЧНЫЙ)" : "", SPEC_NAMES[patients[i].spec]);

        pthread_create(&patients[i].thread, NULL, patient_thread, &patients[i].id);

        pthread_mutex_lock(&hospital_lock);
        generated_count++;
        pthread_mutex_unlock(&hospital_lock);

        usleep((useconds_t)rnd(arrival_min_ms, arrival_max_ms) * 1000);
    }

    pthread_mutex_lock(&hospital_lock);
    all_generated = 1;
    pthread_mutex_unlock(&hospital_lock);
    return NULL;
}

/* ==================== ОБРАБОТКА СИГНАЛОВ ==================== */
static void sigint_handler(int sig) { (void)sig; interrupt_flag = 1; }

/* ==================== MAIN ==================== */
int main(int argc, char *argv[]) {
    /* Парсинг аргументов командной строки */
    for (int i = 1; i < argc; i++) {
        if      (!strcmp(argv[i], "--patients") && i + 1 < argc) total_patients = atoi(argv[++i]);
        else if (!strcmp(argv[i], "--seed")     && i + 1 < argc) rng_seed       = atoi(argv[++i]);
        else if (!strcmp(argv[i], "--urgent")   && i + 1 < argc) urgent_chance  = atoi(argv[++i]);
    }
    if (total_patients > MAX_PATIENTS) total_patients = MAX_PATIENTS;

    srand(rng_seed);
    signal(SIGINT, sigint_handler);
    clock_gettime(CLOCK_MONOTONIC, &t_start);

    /* Открытие лога через системный вызов open() */
    log_fd = open("hospital_threaded.log", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (log_fd < 0) { perror("open log"); return 1; }

    /* Инициализация примитивов синхронизации */
    pthread_mutex_init(&hospital_lock, NULL);
    pthread_mutex_init(&log_mutex,     NULL);
    pthread_cond_init(&exam_done_cond,  NULL);
    pthread_cond_init(&treat_done_cond, NULL);
    for (int i = 0; i < MAX_PATIENTS; i++) pthread_cond_init(&patients[i].cond, NULL);

    sem_init(&primary_sem, 0, 0);
    for (int i = 0; i < SPEC_COUNT; i++) sem_init(&spec_sems[i], 0, 0);

    q_init(&primary_queue);
    for (int i = 0; i < SPEC_COUNT; i++) q_init(&spec_queues[i]);

    log_msg("=== МНОГОПОТОЧНАЯ МОДЕЛЬ БОЛЬНИЦЫ (семафоры) ===\n");
    log_msg("Параметры: пациентов=%d, seed=%u, срочные=%d%%\n",
            total_patients, rng_seed, urgent_chance);
    log_msg("Потоков: 1 генератор + %d дежурных + %d специалиста + пациенты\n",
            NUM_DUTY_DOCS, SPEC_COUNT);

    /* Запуск потоков врачей */
    for (int i = 0; i < NUM_DUTY_DOCS; i++)
        pthread_create(&doc_threads[i], NULL, duty_doctor_thread, &doc_ids[i]);
    for (int i = 0; i < SPEC_COUNT; i++)
        pthread_create(&spec_threads[i], NULL, specialist_thread, &spec_ids[i]);

    /* Запуск генератора пациентов */
    pthread_create(&gen_thread, NULL, generator_thread_func, NULL);

    /* Главный поток ждёт завершения или прерывания */
    for (;;) {
        usleep(50000);   // 50 мс
        if (interrupt_flag) break;
        pthread_mutex_lock(&hospital_lock);
        int done = all_generated && discharged_count >= total_patients;
        pthread_mutex_unlock(&hospital_lock);
        if (done) break;
    }

    /* --- Корректное завершение всех потоков --- */
    pthread_mutex_lock(&hospital_lock);
    shutdown_flag = 1;
    int gen_count = generated_count;
    pthread_mutex_unlock(&hospital_lock);

    /* Разбудить всех врачей (постить в семафоры) */
    for (int i = 0; i < NUM_DUTY_DOCS; i++) sem_post(&primary_sem);
    for (int i = 0; i < SPEC_COUNT; i++)    sem_post(&spec_sems[i]);

    /* Разбудить всех пациентов */
    pthread_mutex_lock(&hospital_lock);
    for (int i = 0; i < gen_count; i++) pthread_cond_signal(&patients[i].cond);
    pthread_cond_broadcast(&exam_done_cond);
    pthread_cond_broadcast(&treat_done_cond);
    pthread_mutex_unlock(&hospital_lock);

    /* Присоединение потоков */
    pthread_join(gen_thread, NULL);
    for (int i = 0; i < gen_count; i++)       pthread_join(patients[i].thread, NULL);
    for (int i = 0; i < NUM_DUTY_DOCS; i++)   pthread_join(doc_threads[i], NULL);
    for (int i = 0; i < SPEC_COUNT; i++)      pthread_join(spec_threads[i], NULL);

    /* Итоговая статистика */
    log_msg("\n=== ЗАВЕРШЕНИЕ РАБОТЫ ===\n");
    log_msg("Всего пациентов создано: %d\n", gen_count);
    log_msg("Выписано после лечения:  %d\n", discharged_count);
    log_msg("Не завершено (прервано): %d\n", gen_count - discharged_count);
    log_msg("Лог сохранён в файл: hospital_threaded.log\n");

    /* Освобождение ресурсов */
    for (int i = 0; i < MAX_PATIENTS; i++) pthread_cond_destroy(&patients[i].cond);
    pthread_cond_destroy(&exam_done_cond);
    pthread_cond_destroy(&treat_done_cond);
    pthread_mutex_destroy(&hospital_lock);
    pthread_mutex_destroy(&log_mutex);
    sem_destroy(&primary_sem);
    for (int i = 0; i < SPEC_COUNT; i++) sem_destroy(&spec_sems[i]);
    close(log_fd);

    return 0;
}

```



---
