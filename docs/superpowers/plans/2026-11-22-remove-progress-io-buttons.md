# Remove Progress Import/Export Buttons Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Убрать кнопки экспорта и импорта из карточки «Мой прогресс» и удалить связанное с ними пояснение, не меняя остальное поведение тренажёра.

**Architecture:** Интерфейс формируется в `build/ui_patch.js`; после точечного изменения этого источника существующий `build_v26.py` заново создаёт `repo/index.html`. Функции импорта и экспорта остаются в коде, но становятся недоступны из интерфейса.

**Tech Stack:** HTML, JavaScript, Python 3, существующий скрипт сборки `build_v26.py`.

## Global Constraints

- Убрать обе кнопки: «Экспорт прогресса (JSON)» и «Импорт прогресса».
- Удалить предложение «Экспорт прогресса — в карточке «Мой прогресс» выше.».
- Сохранить функции `exportProgress()` и `importProgress()`, их стили и остальные файлы без изменений.
- Не менять блок статистики и сообщение о локальном сохранении прогресса.
- Изменения ограничены `build/ui_patch.js`, новой сборкой `repo/index.html` и документацией этой задачи.

---

### Task 1: Удалить кнопки из исходного UI-патча и пересобрать приложение

**Files:**
- Modify: `build/ui_patch.js:63`
- Modify: `build/ui_patch.js:131`
- Modify: `repo/index.html` — только через `python build_v26.py`
- Test: временная проверка через `python -c`; новых файлов тестов не создавать

**Interfaces:**
- Consumes: существующий рендер `renderProgress()` и главный `render()` в `build/ui_patch.js`.
- Produces: собранный `repo/index.html` без кнопок импорта/экспорта и без устаревшего пояснения; публичные функции и форматы данных не меняются.

- [ ] **Step 1: Запустить проверку отсутствия кнопок до правки и убедиться, что она падает**

Run from `/Users/pay/Documents/Kimi/Workspaces/микра`:

```bash
python -c "from pathlib import Path; p=Path('repo/index.html').read_text(encoding='utf-8'); assert '⭳ Экспорт прогресса (JSON)' not in p; assert '⭱ Импорт прогресса' not in p; assert 'Экспорт прогресса — в карточке «Мой прогресс» выше.' not in p"
```

Expected: FAIL с `AssertionError`, потому что кнопки и пояснение пока присутствуют.

- [ ] **Step 2: Удалить строку с обеими кнопками из `build/ui_patch.js`**

Удалить ровно эту строку:

```html
  <div class="io-row"><button class="btn ghost" onclick="exportProgress()">⭳ Экспорт прогресса (JSON)</button><button class="btn ghost" onclick="importProgress()">⭱ Импорт прогресса</button></div>
```

Соседний блок должен стать:

```javascript
  <p class="sub" style="margin-bottom:0">Прогресс и незавершённые занятия сохраняются на этом устройстве.</p>
  </section>`;
};
```

- [ ] **Step 3: Удалить устаревшее предложение из пояснения в `build/ui_patch.js`**

Заменить:

```javascript
  <p class="legend">Итоговые проценты, незавершённые занятия и отметки задач сохраняются локально на этом устройстве. Экспорт прогресса — в карточке «Мой прогресс» выше. Микрофотографии встроены в файл и доступны без интернета.</p>`;
```

на:

```javascript
  <p class="legend">Итоговые проценты, незавершённые занятия и отметки задач сохраняются локально на этом устройстве. Микрофотографии встроены в файл и доступны без интернета.</p>`;
```

- [ ] **Step 4: Пересобрать `index.html` существующим скриптом**

Run from `/Users/pay/Documents/Kimi/Workspaces/микра`:

```bash
python build_v26.py
```

Expected: exit code 0 и строка, начинающаяся с `OK, размер:`.

- [ ] **Step 5: Проверить исходник и собранный файл**

Run from `/Users/pay/Documents/Kimi/Workspaces/микра`:

```bash
python -c "from pathlib import Path; files=[Path('build/ui_patch.js'),Path('repo/index.html')]; texts=[p.read_text(encoding='utf-8') for p in files]; forbidden=['⭳ Экспорт прогресса (JSON)','⭱ Импорт прогресса','Экспорт прогресса — в карточке «Мой прогресс» выше.']; assert all(s not in t for t in texts for s in forbidden); p=texts[1]; assert 'Прогресс и незавершённые занятия сохраняются на этом устройстве.' in p; assert 'Микрофотографии встроены в файл и доступны без интернета.' in p; assert 'window.exportProgress = () =>' in p; assert 'window.importProgress = () =>' in p"
```

Expected: PASS без вывода и с exit code 0.

- [ ] **Step 6: Проверить сборку production и diff на ошибки пробелов**

Run from `/Users/pay/Documents/Kimi/Workspaces/микра/repo`:

```bash
npm run build && git diff --check && git diff --stat
```

Expected: Vite завершает сборку успешно; `git diff --check` ничего не выводит; среди изменений присутствует `index.html`, а ранее существовавшие изменения других файлов не перезаписаны и не включены в commit.

- [ ] **Step 7: Зафиксировать изменение при наличии явного разрешения на commit**

```bash
git add index.html docs/superpowers/specs/2026-11-22-remove-progress-io-buttons-design.md docs/superpowers/plans/2026-11-22-remove-progress-io-buttons.md
git commit -m "feat: убрать кнопки импорта и экспорта прогресса"
```

Expected: создан commit с перечисленными файлами. Без явного разрешения пользователя этот шаг пропустить и сообщить, что изменения не закоммичены.
