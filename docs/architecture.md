# Архитектура (план)

> Черновик плана до начала реализации. Обновляется по мере работы. Требования: [task.md](task.md). Термины: [GLOSSARY.md](../GLOSSARY.md).

## Главная идея

- **Шаг** описывает только *что* сделать.
- **Режим** задаёт *когда* шаг активен и *что делать с ошибкой*. Это данные в инспекторе, а не код.
- **Анализатор** решает *как оценить*. Он читает только журнал.

Все части связаны одним контрактом — записью журнала `LogEntry`.

```
ActionSource ──событие──▶ ScenarioRunner ──LogEntry──▶ ScenarioLogger[] (JSONL-файл, Console)
 (Button/Trigger/            │  ▲                          │
  Collision/UnityEvent)      │  │ правила + план            ▼
                    Step.Handle() ScenarioMode        List<LogEntry> (журнал прогона)
                                                            │
                                         ScenarioAnalyzer.Analyze(entries) ──▶ AnalysisResult + warnings
```

## Компоненты

### 1. Источники событий — `ActionSource : MonoBehaviour`

- В базовом классе `protected void Emit()` и статическое событие `Raised(ActionSource source, GameObject actor)`.
- Domain reload при входе в Play Mode выключен, поэтому `Raised` обнуляется в `[RuntimeInitializeOnLoadMethod(SubsystemRegistration)]`.
- Реализации:
  - `ButtonSource` — `Button.onClick`;
  - `TriggerSource` — Enter или Exit, опционально фильтр по тегу;
  - `CollisionSource` — столкновение;
  - `EventSource` — публичный `Raise()`, вешается на любой UnityEvent (Toggle, Slider и т.д.).
- **Один компонент = одно действие.** Условие в шаге задаётся перетаскиванием нужного источника, без строковых ключей. Если у триггера нужны и Enter, и Exit, это два компонента.
- Свой источник: наследник `ActionSource`, который вызывает `Emit()`.

### 2. Шаг — `Step : MonoBehaviour`

Шаги лежат на дочерних объектах `Scenario/Steps`. MonoBehaviour, а не ScriptableObject, потому что шагу нужны ссылки на объекты сцены.

```csharp
public class Step : MonoBehaviour {
    public string Id => name;                // идентификатор — имя GameObject
    public string title;                     // текст задания в HUD
    public List<ActionSource> sequence;      // 1 элемент — простое действие, N — «сначала A, затем B»
    public bool strictOrder = true;          // B раньше A → нарушение
    public List<ActionSource> forbidden;     // явные нарушения, проверяются только у активного шага
    public List<StateRequirement> requires;  // контекст: GameObject X activeSelf == true/false
    public virtual Verdict Handle(ActionSource s);   // None | Progress | Success | Violation(reason)
    public virtual bool IsRelated(ActionSource s);   // только источники из sequence
    public virtual void ResetProgress();             // сброс прогресса целиком
}
```

- Одного класса хватает на все случаи демо, включая составные шаги.
- Невыполненное условие `requires` в момент действия даёт `Violation` с причиной (например, «Лампа выключена»).
- Новый тип шага — наследник с переопределённым `Handle`.
- Статус, попытки и нарушения хранит раннер, а не шаг. Шаг — это определение, результат шага живёт в журнале.

### 3. Режим — `ScenarioMode : MonoBehaviour`

Данные без логики. Лежат на `Scenario/Modes/Training` и `Scenario/Modes/Exam`.

```csharp
public class ScenarioMode : MonoBehaviour {
    public OnViolation onViolation;        // RetryStep | FailStep
    public OutOfTurn outOfTurn;            // Violation | Ignore (шаг «неактивен до очереди»)
    public int skipThreshold = 2;          // 0 — без досрочного пропуска
    public List<Group> groups;             // Group { name; List<Stage> stages; }
    public ScenarioAnalyzer defaultAnalyzer;
}

[Serializable] public class Stage { public List<Step> steps; } // шаги Stage активны одновременно
```

- **Group** (порог ошибок экзамена) и **Stage** (набор одновременно активных шагов) — разные настройки.
- Правила ошибок задаются только в режиме, без переопределения на шаге.
- Следующий Stage открывается, когда все шаги текущего завершены (Completed или Failed).
- Смешанная активация из задания: `[1,2,3,4] [5] [6] [7] [8] [9,10] [11,12]`.
  - Последовательная активация — один шаг на Stage.
  - Одновременная — все шаги в одном Stage.
- Перетаскивание даёт стандартный ReorderableList.

| | Тренировка | Экзамен |
|---|---|---|
| Группы | 1 | 3 |
| `onViolation` | `RetryStep` | `FailStep` |
| `outOfTurn` | `Violation` | `Violation` |
| `skipThreshold` | 0 | 2 |

- Отдельных классов под режимы нет. Оба режима ссылаются на одни и те же компоненты `Step`.
- `OnValidate` предупреждает (не блокирует), если шаг не попал в план режима, встречается в нём дважды или если у двух шагов одинаковые имена.

### 4. Исполнитель — `ScenarioRunner : MonoBehaviour`

Поля в инспекторе: `List<ScenarioMode> modes`, `List<ScenarioAnalyzer> analyzers`, `ScenarioLogger[] loggers`.

- **`Begin(ScenarioMode)`:**
  - если прохождение уже идёт, пишет `ScenarioFinished` с `result = Aborted`;
  - сбрасывает состояние и прогресс шагов, пишет `ScenarioStarted`, активирует первый Stage первой группы.
- **`HandleAction(source)`** (вне прохождения действия игнорируются и не пишутся):
  1. Источник есть у **активного** шага → `step.Handle`; остальные шаги это действие не видят.
     - `Success` → шаг получает Completed.
     - `Violation` → по `onViolation`:
       - `RetryStep`: attempt + 1, `ResetProgress`;
       - `FailStep`: шаг получает Failed, счётчик ошибок его группы + 1.
  2. Иначе действие **вне очереди** для самого раннего незавершённого шага с этим источником:
     - `outOfTurn == Violation` → нарушение этого шага по тем же правилам (Тренировка: attempt + 1; Экзамен: Failed, счётчик ошибок **группы этого шага** + 1);
     - `Ignore` → действие игнорируется.
  3. Шаги с итоговым статусом (Completed / Failed / Skipped) действий не получают.
  4. Счётчик ошибок группы ≥ `skipThreshold` → незавершённые шаги группы получают Skipped. Если это текущая группа, начинается следующая; если будущая — она пропускается сразу при старте.
- **Инвариант:** в завершённом прохождении у каждого шага плана ровно один `StepFinished`.
- Наружу отдаёт C#-события `StepChanged`, `ViolationRaised`, `Finished` для HUD. `ViolationRaised` срабатывает в любом режиме.
- Пишет `LogEntry` в журнал прогона (`List<LogEntry>`) и во все логгеры.
- Публичные методы не зависят от `Update`, поэтому их можно покрыть EditMode-тестами без сцены.
- Другой исполнитель — наследник или замена раннера, который пишет тот же контракт `LogEntry`.

### 5. Журнал — контракт данных

```csharp
[Serializable] public class LogEntry {
    public float time;                       // секунды от начала прохождения
    public string type;                      // string, не enum — новые типы не ломают старый код
    public string stepId, groupId, result, message;
    public int attempt;
    public List<Field> extra;                // key/value для расширений, не меняющих LogEntry
}
```

| Тип | Что значит |
|---|---|
| `ScenarioStarted` | `message` — имя режима, `extra.startedAt` — дата и время ISO |
| `GroupStarted` | `groupId` |
| `StepActivated` | шаг стал активным |
| `Action` | действие, принятое шагом |
| `Violation` | `message` — причина, `attempt` — номер попытки |
| `StepFinished` | `result` — Completed / Failed / Skipped, `attempt` — номер успешной попытки |
| `ScenarioFinished` | `result` — Completed / Aborted |

- `JsonUtility` пропускает лишние поля и ставит значения по умолчанию для отсутствующих, поэтому новые поля не ломают старые журналы.
- `ScenarioLogger : ScriptableObject` с одним методом `Write(LogEntry)`. Реализации:
  - `JsonFileLogger` — JSONL, одна строка на запись, дописывается сразу; новый файл в `persistentDataPath/Runs` на каждый `ScenarioStarted`;
  - `ConsoleLogger` — вывод в Debug-консоль.
- Раннер держит массив `ScenarioLogger[]`, логгеры подключаются перетаскиванием ассетов.

### 6. Анализ — `ScenarioAnalyzer : ScriptableObject`

```csharp
public abstract AnalysisResult Analyze(IReadOnlyList<LogEntry> log); // score, max, строки по шагам, warnings
```

- Набор шагов — уникальные stepId из `StepFinished`. Плана режима анализатор не знает.
- **`TrainingAnalyzer`**
  - Баллы 10/5/0 по `attempt` из `StepFinished` с `result = Completed`; Failed и Skipped — 0.
  - Максимум = 10 × число шагов.
  - Нет `attempt` → считает попытки по `Violation` и выдаёт предупреждение.
- **`ExamAnalyzer`:** `max(0, 100 − 10·Failed − 5·Skipped)`.
- Общие предупреждения: нет `ScenarioFinished` или `result = Aborted` → «прохождение не завершено, максимум посчитан по шагам из журнала».
- Баллы и штрафы — поля ассета, правятся в инспекторе.
- Анализатор не знает про режим и раннер, поэтому любая пара «режим × анализатор» работает через общий контракт без спецкода (например, `ExamAnalyzer` на Тренировке даёт 100).
- Неизвестные типы записей пропускаются.

### 7. UI и Editor

Интерфейс, задания и причины нарушений — на русском.

- **`ScenarioHud`:**
  - кнопка запуска на каждый режим из `runner.modes` (она же — повтор);
  - список активных заданий;
  - всплывающее сообщение о нарушении (в обоих режимах);
  - панель результата: первая оценка по `mode.defaultAnalyzer`, выпадающий список `runner.analyzers` для повторной оценки, предупреждения;
  - прокручиваемый журнал.
- При повторе сбрасывается только состояние сценария, игрок возвращается на точку старта. Остальные объекты сцены (лампа, ящик) остаются как есть.
- **`LogViewerWindow`** (EditorWindow, последним и по остаточному принципу): открывает JSONL прогона, даёт выбрать анализатор и показывает результат без Play Mode.

### 8. Демо-сцена

- Капсула на WASD через `Keyboard.current` (Input System, без `.inputactions`), триггеры, ящик для Collision, uGUI-кнопки, Toggle с `EventSource`. Лампа — объект, который Toggle включает через `SetActive`.

| # | Шаг | Источник |
|---|---|---|
| 1 | Включить питание | Button «Питание» |
| 2 | Включить лампу | Toggle «Лампа» → `EventSource` |
| 3 | Встать на площадку | Trigger «Площадка» |
| 4 | Толкнуть ящик | Collision «Ящик» |
| 5 | **Пройти ворота A, затем B** | Trigger A → Trigger B (strict) |
| 6 | **Открыть клапан при горящей лампе** | Button «Клапан» + лампа `activeSelf`; forbidden: Button «Аварийный сброс» |
| 7 | Сообщить диспетчеру | Button «Рация» |
| 8 | Выйти в зону эвакуации | Trigger «Выход» |

- **Тренировка:** `[1,2,3,4] → [5] → [6] → [7] → [8]`.
- **Экзамен:** G1 `[1,2,3,4]`, G2 `[5] → [6]`, G3 `[7,8]`. G1 воспроизводит контрольный пример 2 из §7.

## Структура

```
Assets/Scenario/
  Runtime/ (asmdef)  Sources/*.cs  Step.cs  ScenarioMode.cs  ScenarioRunner.cs
                     Log/LogEntry.cs, ScenarioLogger.cs, JsonFileLogger.cs, ConsoleLogger.cs
                     Analysis/ScenarioAnalyzer.cs, TrainingAnalyzer.cs, ExamAnalyzer.cs
  Editor/  (asmdef)  LogViewerWindow.cs
  Tests/   (asmdef)  AnalyzerTests.cs, RunnerTests.cs   ← контрольные примеры из task.md §7
  Demo/              Demo.unity, ScenarioHud.cs, PlayerMover.cs, *.asset
```

Тестам нужен `com.unity.test-framework` явно в `Packages/manifest.json` (сейчас он только транзитивный).

## Сознательно не делаем

- **Интерфейс под каждую часть.** Точки расширения — 4 базовых класса: `ActionSource`, `Step` (virtual), `ScenarioLogger`, `ScenarioAnalyzer`.
- **Классы под режимы.** Режим — это данные.
- **Строковые ID для связи сцены и шагов.** Только прямые ссылки.
- **Правила ошибок на уровне шага.** Только на уровне режима.
- **Перекрывающиеся группы.** Описать в README в «что улучшил бы».
- **Перезагрузку сцены при повторе.** Иначе пришлось бы трогать Build Settings.

## План коммитов

1. Источники событий + `Step`.
2. `ScenarioMode` + `ScenarioRunner` + тесты активации (+ test-framework в manifest).
3. Журнал + логгеры.
4. Анализаторы + тесты контрольных примеров.
5. Демо-сцена + HUD.
6. `LogViewerWindow` (если останется время).
7. README (инструкция, техописание, «что улучшить», затраченное время).
