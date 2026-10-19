# Архитектура (план)

> Черновик плана до начала реализации. Обновляется по мере работы. Требования: [task.md](task.md).

## Главная идея

- **Шаг** описывает только *что* сделать.
- **Режим** задаёт *когда* шаг активен и *что делать с ошибкой*. Это данные в инспекторе, а не код.
- **Анализатор** решает *как оценить*. Он читает только журнал.

Все части связаны одним контрактом — записью журнала `LogEntry`.

```
ActionSource ──событие──▶ ScenarioRunner ──LogEntry──▶ ScenarioLogger[] (JSON-файл, Console)
 (Button/Trigger/            │  ▲                          │
  Collision/UnityEvent)      │  │ правила + план            ▼
                    Step.Handle() ScenarioMode         RunRecord (журнал прогона)
                                                            │
                                         ScenarioAnalyzer.Analyze(entries) ──▶ AnalysisResult + warnings
```

## Компоненты

### 1. Источники событий — `ActionSource : MonoBehaviour`

- В базовом классе `protected void Emit()` и статическое событие `Raised(ActionSource source, GameObject actor)`.
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
    public string id, title;
    public List<ActionSource> sequence;      // 1 элемент — простое действие, N — «сначала A, затем B»
    public bool strictOrder = true;          // B раньше A → нарушение
    public List<ActionSource> forbidden;     // явные нарушения
    public List<StateRequirement> requires;  // контекст: «объект X активен/неактивен»
    public virtual Verdict Handle(ActionSource s);   // None | Progress | Success | Violation(reason)
    public virtual bool IsRelated(ActionSource s);
    public virtual void ResetProgress();
}
```

- Одного класса хватает на все случаи демо, включая составные шаги.
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

- **Group** (порог ошибок экзамена) и **Stage** (пакет одновременно активных шагов) — разные настройки.
- Следующий Stage открывается, когда все шаги текущего завершены (Completed или Failed).
- Смешанная активация из задания: `[1,2,3,4] [5] [6] [7] [8] [9,10] [11,12]`.
  - Последовательная активация — один шаг на Stage.
  - Одновременная — все шаги в одном Stage.
- Перетаскивание даёт стандартный ReorderableList.

| | Тренировка | Экзамен |
|---|---|---|
| Группы | 1 | 3 |
| `onViolation` | `RetryStep` | `FailStep` |
| `skipThreshold` | 0 | 2 |

- Отдельных классов под режимы нет. Оба режима ссылаются на одни и те же компоненты `Step`.
- Опционально: `Group.startWithPrevious` для перекрывающихся групп. Делаем последним.
- `OnValidate` предупреждает, если шаг не попал в план режима или встречается в нём дважды.

### 4. Исполнитель — `ScenarioRunner : MonoBehaviour`

- **`Begin(ScenarioMode)`:** сбрасывает состояние, пишет `ScenarioStarted` со списком stepId и активирует первый Stage первой группы.
- **`HandleAction(source)`:**
  1. Действие относится к активному шагу → вызывается `step.Handle`.
     - `Success` → шаг получает Completed.
     - `Violation` → по `onViolation`:
       - `RetryStep`: attempt + 1, `ResetProgress`;
       - `FailStep`: шаг получает Failed, счётчик ошибок группы + 1.
  2. Действие относится к неактивному шагу:
     - `outOfTurn == Violation` → нарушение этого шага;
     - `Ignore` → действие игнорируется.
  3. Счётчик ошибок группы ≥ `skipThreshold` → незавершённые шаги группы получают Skipped, начинается следующая группа.
- Наружу отдаёт C#-события `StepChanged`, `ViolationRaised`, `Finished` для HUD.
- Пишет `LogEntry` в `RunRecord` и во все логгеры.
- Публичные методы не зависят от `Update`, поэтому их можно покрыть EditMode-тестами без сцены.
- Другой исполнитель — наследник или замена раннера, который пишет тот же контракт `LogEntry`.

### 5. Журнал — контракт данных

```csharp
[Serializable] public class LogEntry {
    public float time;
    public string type;                      // string, не enum — новые типы не ломают старый код
    public string stepId, groupId, result, message;
    public int attempt;
    public List<Field> extra;                // key/value для расширений
}
```

- Типы записей: `ScenarioStarted`, `GroupStarted`, `StepActivated`, `Action`, `Violation`, `StepCompleted`, `StepFailed`, `StepSkipped`, `ScenarioFinished`.
- `ScenarioLogger : ScriptableObject` с одним методом `Write(LogEntry)`. Реализации:
  - `JsonFileLogger` — файл на прогон в `persistentDataPath/Runs`;
  - `ConsoleLogger` — вывод в Debug-консоль.
- Раннер держит массив `ScenarioLogger[]`, логгеры подключаются перетаскиванием ассетов.
- `RunRecord { modeName, entries }` — журнал прогона. Его же сериализует JSON-логгер.

### 6. Анализ — `ScenarioAnalyzer : ScriptableObject`

```csharp
public abstract AnalysisResult Analyze(IReadOnlyList<LogEntry> log); // score, max, строки по шагам, warnings
```

- **`TrainingAnalyzer`**
  - Баллы 10/5/0 по `attempt` из `StepCompleted`.
  - Максимум = 10 × число шагов.
  - Нет `attempt` → считает попытки по `Violation` и выдаёт предупреждение.
  - Нет `ScenarioStarted` → берёт stepId из журнала и выдаёт предупреждение.
- **`ExamAnalyzer`:** `max(0, 100 − 10·Failed − 5·Skipped)`.
- Баллы и штрафы — поля ассета, правятся в инспекторе.
- Анализатор не знает про режим и раннер, поэтому любая пара «режим × анализатор» работает через общий контракт.
- Неизвестные типы записей пропускаются.

### 7. UI и Editor

- **`ScenarioHud`:**
  - список активных заданий;
  - всплывающее сообщение о нарушении;
  - кнопки запуска режимов и повторного прохождения;
  - панель результата с выбором анализатора (повторная оценка);
  - прокручиваемый журнал.
- **`LogViewerWindow`** (EditorWindow): открывает JSON прогона, даёт выбрать анализатор и показывает результат без Play Mode. Это доказательство, что анализ отделён от исполнения.
- **Демо-сцена:**
  - капсула на WASD (Input System);
  - триггеры A и B;
  - ящик для Collision;
  - uGUI-кнопки;
  - Toggle с `EventSource`.
- **Составные шаги в демо:**
  - «войти в A, затем в B»;
  - «нажать "Открыть клапан", только когда лампа включена».

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

## Сознательно не делаем

- **Интерфейс под каждую часть.** Точки расширения — 4 базовых класса: `ActionSource`, `Step` (virtual), `ScenarioLogger`, `ScenarioAnalyzer`.
- **Классы под режимы.** Режим — это данные.
- **Строковые ID для связи сцены и шагов.** Только прямые ссылки.

## План коммитов

1. Источники событий + `Step`.
2. `ScenarioMode` + `ScenarioRunner` + тесты активации.
3. Журнал + логгеры.
4. Анализаторы + тесты контрольных примеров.
5. Демо-сцена + HUD.
6. `LogViewerWindow`.
7. README (инструкция, техописание, «что улучшить», затраченное время).
