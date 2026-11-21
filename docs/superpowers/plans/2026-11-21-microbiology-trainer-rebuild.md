# Microbiology Trainer Rebuild Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Переписать тренажёр на React + TypeScript + Vite с новым интерфейсом, сохранив текущий путь «модули → теория → тест → результат», весь контент и локальный прогресс.

**Architecture:** SPA с hash-маршрутизацией, типизированными учебными данными и feature-компонентами. Учебные данные извлекаются из фактической v2.5.2 в JSON и отдельные изображения; состояние хранится через Context/useReducer и версионированный localStorage-адаптер.

**Tech Stack:** React, TypeScript strict, Vite, React Router, Vitest, React Testing Library, Playwright.

## Global Constraints

- Работа ведётся только в ветке `rebuild-from-scratch`; `main` не изменяется.
- Основной путь: **Главная со списком модулей → теория → тест → результат**.
- Не добавлять ежедневные планы, адаптивные алгоритмы, карту знаний, сервер, регистрацию и облачную синхронизацию.
- Сохранить 10 активных подмодулей, 77 занятий, 228 вопросов, 6 заблокированных разделов, 9 микропрепаратов, 24 микрозадания и 7 комплексных случаев.
- Не добавлять функции из описания v2.6, отсутствующие в исходной commit-версии `index.html`.
- Во время планирования в рабочем `index.html` появились внешние незакоммиченные изменения v2.6: сохранять их как есть, не staging и не использовать; extractor обязан читать `git show HEAD:index.html`.
- Старые ключи прогресса не удаляются; публикация, push и удаление старых файлов запрещены.
- После ручной проверки dev-сервер останавливается; финальный preview открывается через `npm run dev -- --host 0.0.0.0 --port 7100` по требованию клиента.

---

### Task 1: Каркас приложения и тестовая инфраструктура

**Files:**
- Create: `web/package.json`, `web/tsconfig.json`, `web/vite.config.ts`, `web/index.html`, `web/src/main.tsx`, `web/src/app/App.tsx`, `web/src/styles/global.css`.
- Test: `web/src/app/App.test.tsx`, `web/src/test/setup.ts`.

**Interfaces:**
- Produces: команды `npm run dev|build|test|test:e2e` из `web/`; корневой legacy `index.html` остаётся нетронутым до финальной сборки.

- [ ] **Step 1: Создать независимый Vite-проект в `web/`**

```bash
mkdir -p web/src/app web/src/styles web/src/test
cd web
npm init -y
npm install react react-dom react-router-dom
npm install -D typescript vite @vitejs/plugin-react vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event @playwright/test @types/react @types/react-dom @types/node
npm pkg set type=module scripts.dev='vite' scripts.build='tsc --noEmit && vite build' scripts.test='vitest run' scripts.test:e2e='playwright test'
cd ..
```

Expected: зависимости установлены, legacy-файлы в корне не изменены.

- [ ] **Step 2: Добавить конфигурацию и failing smoke-test**

`web/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "types": ["vitest/globals", "@testing-library/jest-dom", "node"]
  },
  "include": ["src", "vite.config.ts", "playwright.config.ts"]
}
```

`web/vite.config.ts`:

```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  base: './',
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    css: true,
  },
});
```

`web/src/test/setup.ts`:

```ts
import '@testing-library/jest-dom/vitest';
```

`web/src/app/App.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { App } from './App';

test('показывает название курса', () => {
  render(<App />);
  expect(screen.getByRole('heading', { name: 'Микробиология' })).toBeInTheDocument();
});
```

- [ ] **Step 3: Запустить smoke-test и убедиться в FAIL**

Run: `cd web && npm test -- src/app/App.test.tsx`  
Expected: FAIL, потому что `App.tsx` отсутствует.

- [ ] **Step 4: Реализовать минимальный запускаемый App**

`web/index.html`:

```html
<!doctype html>
<html lang="ru">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Тренажёр по общей медицинской микробиологии" />
    <title>Микробиология · учебный атлас</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

`web/src/app/App.tsx`:

```tsx
export function App() {
  return (
    <main>
      <h1>Микробиология</h1>
      <p>Учебный атлас и тренажёр к экзамену.</p>
    </main>
  );
}
```

`web/src/main.tsx`:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { HashRouter } from 'react-router-dom';
import { App } from './app/App';
import './styles/global.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <HashRouter>
      <App />
    </HashRouter>
  </React.StrictMode>,
);
```

`web/src/styles/global.css`:

```css
:root {
  font-family: Inter, ui-sans-serif, system-ui, sans-serif;
  color: #21352e;
  background: #f4f0e7;
  font-synthesis: none;
  --bg: #f4f0e7;
  --surface: #fffdf8;
  --ink: #21352e;
  --muted: #68746e;
  --line: #ded6c7;
  --accent: #285e4d;
  --accent-soft: #e3eee8;
  --danger: #a34236;
  --warning: #8b5e3c;
}
* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; }
button, input, select { font: inherit; }
button, a { -webkit-tap-highlight-color: transparent; }
button:focus-visible, a:focus-visible, select:focus-visible {
  outline: 3px solid #c98a22;
  outline-offset: 3px;
}
h1, h2, h3 { font-family: Georgia, 'Times New Roman', serif; }
```

- [ ] **Step 5: Проверить PASS и production-сборку**

Run: `cd web && npm test -- src/app/App.test.tsx && npm run build`  
Expected: 1 test PASS; TypeScript и Vite build завершаются без ошибок.

- [ ] **Step 6: Commit**

```bash
git add web .gitignore
git commit -m "chore: scaffold React trainer application"
```

---

### Task 2: Типизированный перенос всего учебного контента

**Files:**
- Create: `web/scripts/extract-legacy-content.mjs`.
- Create: `web/src/data/types.ts`, `web/src/data/content.ts`, generated `web/src/data/legacy-content.json`.
- Create: generated images in `web/public/images/`.
- Test: `web/src/data/content.test.ts`.

**Interfaces:**
- Produces: `content: Content`, `allMicroTasks: MicroTaskView[]`; типы `CourseModule`, `Lesson`, `QuizQuestion`, `MicroCase`, `ComplexCase`, `ComplexField`.

- [ ] **Step 1: Написать failing contract-test на объём контента**

`web/src/data/content.test.ts`:

```ts
import { content, allMicroTasks } from './content';

test('сохраняет весь подтверждённый учебный контент', () => {
  const active = content.modules.filter((module) => !module.locked);
  expect(active).toHaveLength(10);
  expect(active.reduce((sum, module) => sum + module.theory.length, 0)).toBe(77);
  expect(active.reduce((sum, module) => sum + module.quiz.length, 0)).toBe(228);
  expect(content.modules.filter((module) => module.locked)).toHaveLength(6);
  expect(content.microCases).toHaveLength(9);
  expect(allMicroTasks).toHaveLength(24);
  expect(content.complexCases).toHaveLength(7);
});

test('каждый вопрос имеет допустимый правильный ответ', () => {
  const questions = content.modules.flatMap((module) => module.quiz);
  for (const question of questions) {
    expect(question.o.length).toBeGreaterThanOrEqual(2);
    expect(question.a).toBeGreaterThanOrEqual(0);
    expect(question.a).toBeLessThan(question.o.length);
  }
});
```

Run: `cd web && npm test -- src/data/content.test.ts`  
Expected: FAIL, потому что `content.ts` отсутствует.

- [ ] **Step 2: Определить единые доменные типы**

`web/src/data/types.ts`:

```ts
export interface Lesson {
  h: string;
  body: string;
  img?: string;
}

export interface QuizQuestion {
  q: string;
  o: string[];
  a: number;
  e: string;
  h?: string;
  topic?: string;
  img?: string;
}

export interface CourseModule {
  id: string;
  title: string;
  qs: string;
  theory: Lesson[];
  quiz: QuizQuestion[];
  locked: boolean;
}

export interface MicroTask {
  q: string;
  o: string[];
  a: number;
  e: string;
}

export interface MicroCase {
  id: string;
  title: string;
  img: string;
  source: string;
  tasks: MicroTask[];
}

export interface MicroTaskView extends MicroTask {
  key: string;
  caseId: string;
  caseTitle: string;
  img: string;
  source: string;
  taskNo: number;
  taskTotal: number;
}

export type ComplexFieldId = 'form' | 'arrangement' | 'stain' | 'group';

export interface ComplexCase {
  id: string;
  title: string;
  img: string;
  source: string;
  answers: Record<ComplexFieldId, string>;
  why: Record<ComplexFieldId, string>;
}

export interface ComplexField {
  id: ComplexFieldId;
  label: string;
  options: string[];
}

export interface Content {
  modules: CourseModule[];
  microCases: MicroCase[];
  complexCases: ComplexCase[];
  complexFields: ComplexField[];
}
```

- [ ] **Step 3: Реализовать детерминированный extractor**

`web/scripts/extract-legacy-content.mjs`:

```js
import fs from 'node:fs';
import path from 'node:path';
import vm from 'node:vm';
import { execFileSync } from 'node:child_process';

const webRoot = path.resolve(import.meta.dirname, '..');
const repoRoot = path.resolve(webRoot, '..');
const html = execFileSync('git', ['show', 'HEAD:index.html'], { cwd: repoRoot, encoding: 'utf8' });
const match = html.match(/<script>([\s\S]*)<\/script>/);
if (!match) throw new Error('Legacy script not found');

const app = { innerHTML: '' };
const context = {
  console,
  window: {},
  document: { getElementById: () => app },
  localStorage: { getItem: () => null, setItem: () => {} },
  scrollTo: () => {},
  alert: () => {},
  confirm: () => true,
};
vm.createContext(context);
vm.runInContext(`${match[1]}
globalThis.__content = {
  modules: MODULES.map((m) => ({
    id: String(m.id), title: m.title, qs: m.qs,
    theory: Array.isArray(m.theory) ? m.theory : [],
    quiz: Array.isArray(m.quiz) ? m.quiz : [], locked: Boolean(m.locked),
  })),
  microCases: MICRO_CASES,
  complexCases: COMPLEX_CASES,
  complexFields: COMPLEX_FIELDS,
};`, context);

const imageDir = path.join(webRoot, 'public', 'images');
fs.mkdirSync(imageDir, { recursive: true });
let imageNumber = 0;
const extensions = { jpeg: 'jpg', jpg: 'jpg', png: 'png', webp: 'webp', gif: 'gif', 'svg+xml': 'svg' };

function extractImages(value) {
  if (typeof value === 'string') {
    return value.replace(/src="data:image\/([a-zA-Z0-9+.-]+);base64,([^"]+)"/g, (_, kind, data) => {
      const extension = extensions[kind];
      if (!extension) throw new Error(`Unsupported image type: ${kind}`);
      const name = `legacy-${String(++imageNumber).padStart(3, '0')}.${extension}`;
      fs.writeFileSync(path.join(imageDir, name), Buffer.from(data, 'base64'));
      return `src="./images/${name}"`;
    });
  }
  if (Array.isArray(value)) return value.map(extractImages);
  if (value && typeof value === 'object') {
    return Object.fromEntries(Object.entries(value).map(([key, item]) => [key, extractImages(item)]));
  }
  return value;
}

const content = extractImages(context.__content);
const output = path.join(webRoot, 'src', 'data');
fs.mkdirSync(output, { recursive: true });
fs.writeFileSync(path.join(output, 'legacy-content.json'), `${JSON.stringify(content, null, 2)}\n`);
console.log(`Extracted ${imageNumber} images and ${content.modules.length} modules.`);
```

- [ ] **Step 4: Добавить адаптер и выполнить перенос**

`web/src/data/content.ts`:

```ts
import raw from './legacy-content.json';
import type { Content, MicroTaskView } from './types';

export const content = raw as Content;

export const allMicroTasks: MicroTaskView[] = content.microCases.flatMap((item) =>
  item.tasks.map((task, index) => ({
    ...task,
    key: `${item.id}_${index + 1}`,
    caseId: item.id,
    caseTitle: item.title,
    img: item.img,
    source: item.source,
    taskNo: index + 1,
    taskTotal: item.tasks.length,
  })),
);

export function getModule(id: string | undefined) {
  return content.modules.find((module) => module.id === id);
}
```

Run: `cd web && node scripts/extract-legacy-content.mjs && npm test -- src/data/content.test.ts`  
Expected: extractor сообщает о переносе; оба теста PASS; JSON не содержит `data:image`.

- [ ] **Step 5: Commit**

```bash
git add web/scripts/extract-legacy-content.mjs web/src/data web/public/images
git commit -m "feat: migrate complete microbiology course content"
```

---

### Task 3: Версионированное хранение и миграция прогресса

**Files:**
- Create: `web/src/storage/progress.ts`, `web/src/storage/ProgressContext.tsx`.
- Test: `web/src/storage/progress.test.ts`, `web/src/storage/ProgressContext.test.tsx`.

**Interfaces:**
- Produces: `loadProgress`, `saveProgress`, `emptyProgress`, `ProgressProvider`, `useProgress`; типы `ProgressState`, `Draft`, `CompletionInput`.

- [ ] **Step 1: Написать failing tests для новой схемы и старых ключей**

`web/src/storage/progress.test.ts`:

```ts
import { emptyProgress, loadProgress, saveProgress, PROGRESS_KEY } from './progress';

function memoryStorage(initial: Record<string, string> = {}): Storage {
  const values = new Map(Object.entries(initial));
  return {
    get length() { return values.size; },
    clear: () => values.clear(),
    key: (index) => [...values.keys()][index] ?? null,
    getItem: (key) => values.get(key) ?? null,
    setItem: (key, value) => { values.set(key, value); },
    removeItem: (key) => { values.delete(key); },
  };
}

test('возвращает пустой прогресс при первом запуске', () => {
  expect(loadProgress(memoryStorage())).toEqual({ state: emptyProgress });
});

test('мигрирует результаты, статистику и ошибки v2.5.2', () => {
  const storage = memoryStorage({
    micro_trainer_v24_state: JSON.stringify({ version: 1, S: { results: { '1.1': 84 } } }),
    micro_trainer_v24_stats: JSON.stringify({ completed: 3, answers: 30, correct: 24, sessions: { quiz: 2 }, lastActivity: 123 }),
    micro_wrong_v1: JSON.stringify(['gram_1']),
    micro_complex_wrong_v1: JSON.stringify(['cx_ecoli']),
  });
  const { state } = loadProgress(storage);
  expect(state.results['1.1']).toBe(84);
  expect(state.stats).toMatchObject({ completed: 3, answers: 30, correct: 24, lastActivity: 123 });
  expect(state.microWrong).toEqual(['gram_1']);
  expect(state.complexWrong).toEqual(['cx_ecoli']);
  expect(storage.getItem('micro_trainer_v24_state')).not.toBeNull();
});

test('переживает повреждённый JSON и сохраняет новую версию', () => {
  const storage = memoryStorage({ [PROGRESS_KEY]: '{bad' });
  expect(loadProgress(storage).notice).toMatch(/поврежд/);
  saveProgress(storage, { ...emptyProgress, results: { '1.1': 90 } });
  expect(JSON.parse(storage.getItem(PROGRESS_KEY)!).results['1.1']).toBe(90);
});
```

Run: `cd web && npm test -- src/storage/progress.test.ts`  
Expected: FAIL, модуль отсутствует.

- [ ] **Step 2: Реализовать схему и миграцию**

`web/src/storage/progress.ts`:

```ts
export const PROGRESS_KEY = 'microbiology_trainer.progress.v3';

export interface Stats {
  completed: number;
  answers: number;
  correct: number;
  sessions: Record<string, number>;
  lastActivity: number | null;
}

export type Draft =
  | { kind: 'theory'; moduleId: string; slide: number; updatedAt: number }
  | { kind: 'quiz'; moduleId: string; updatedAt: number }
  | { kind: 'micro'; updatedAt: number }
  | { kind: 'complex'; updatedAt: number };

export interface ProgressState {
  version: 3;
  results: Record<string, number>;
  stats: Stats;
  microWrong: string[];
  complexWrong: string[];
  draft: Draft | null;
}

export const emptyProgress: ProgressState = {
  version: 3,
  results: {},
  stats: { completed: 0, answers: 0, correct: 0, sessions: {}, lastActivity: null },
  microWrong: [],
  complexWrong: [],
  draft: null,
};

function strings(value: unknown): string[] {
  return Array.isArray(value) ? value.filter((item): item is string => typeof item === 'string') : [];
}

function parse(raw: string | null): any {
  return raw ? JSON.parse(raw) : null;
}

export function loadProgress(storage: Storage): { state: ProgressState; notice?: string } {
  try {
    const current = parse(storage.getItem(PROGRESS_KEY));
    if (current?.version === 3) {
      return { state: { ...emptyProgress, ...current, stats: { ...emptyProgress.stats, ...current.stats } } };
    }
    const legacy = parse(storage.getItem('micro_trainer_v24_state'));
    const stats = parse(storage.getItem('micro_trainer_v24_stats'));
    const state: ProgressState = {
      ...emptyProgress,
      results: legacy?.S?.results && typeof legacy.S.results === 'object' ? legacy.S.results : {},
      stats: stats && typeof stats === 'object' ? { ...emptyProgress.stats, ...stats } : emptyProgress.stats,
      microWrong: strings(parse(storage.getItem('micro_wrong_v1'))),
      complexWrong: strings(parse(storage.getItem('micro_complex_wrong_v1'))),
    };
    return { state };
  } catch {
    return { state: emptyProgress, notice: 'Сохранённый прогресс повреждён. Начата новая сессия; старые данные не удалены.' };
  }
}

export function saveProgress(storage: Storage, state: ProgressState): boolean {
  try {
    storage.setItem(PROGRESS_KEY, JSON.stringify(state));
    return true;
  } catch {
    return false;
  }
}
```

- [ ] **Step 3: Добавить Context с единичной записью результата**

`web/src/storage/ProgressContext.tsx`:

```tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState, type ReactNode } from 'react';
import { emptyProgress, loadProgress, saveProgress, type Draft, type ProgressState } from './progress';

export interface CompletionInput {
  type: string;
  score: number;
  total: number;
  moduleId?: string;
}

interface ProgressContextValue {
  progress: ProgressState;
  notice: string | null;
  complete: (input: CompletionInput) => void;
  setMicroWrong: (keys: string[]) => void;
  setComplexWrong: (keys: string[]) => void;
  setDraft: (draft: Draft | null) => void;
}

const ProgressContext = createContext<ProgressContextValue | null>(null);

export function ProgressProvider({ children }: { children: ReactNode }) {
  const [initial] = useState(() => loadProgress(window.localStorage));
  const [progress, setProgress] = useState(initial.state);
  const [notice, setNotice] = useState(initial.notice ?? null);

  useEffect(() => {
    if (!saveProgress(window.localStorage, progress)) {
      setNotice('Хранилище недоступно: прогресс сохранится только до закрытия вкладки.');
    }
  }, [progress]);

  const complete = useCallback(({ type, score, total, moduleId }: CompletionInput) => setProgress((old) => ({
    ...old,
    results: moduleId ? { ...old.results, [moduleId]: Math.round(score / Math.max(total, 1) * 100) } : old.results,
    stats: {
      completed: old.stats.completed + 1,
      answers: old.stats.answers + Math.max(0, total),
      correct: old.stats.correct + Math.max(0, score),
      sessions: { ...old.stats.sessions, [type]: (old.stats.sessions[type] ?? 0) + 1 },
      lastActivity: Date.now(),
    },
  })), []);
  const setMicroWrong = useCallback((microWrong: string[]) => setProgress((old) => ({ ...old, microWrong })), []);
  const setComplexWrong = useCallback((complexWrong: string[]) => setProgress((old) => ({ ...old, complexWrong })), []);
  const setDraft = useCallback((draft: Draft | null) => setProgress((old) => ({ ...old, draft })), []);
  const value = useMemo<ProgressContextValue>(() => ({ progress, notice, complete, setMicroWrong, setComplexWrong, setDraft }), [progress, notice, complete, setMicroWrong, setComplexWrong, setDraft]);

  return <ProgressContext.Provider value={value}>{children}</ProgressContext.Provider>;
}

export function useProgress() {
  const value = useContext(ProgressContext);
  if (!value) throw new Error('useProgress must be used inside ProgressProvider');
  return value;
}

export { emptyProgress };
```

`web/src/storage/ProgressContext.test.tsx`:

```tsx
import { renderHook, act } from '@testing-library/react';
import { ProgressProvider, useProgress } from './ProgressContext';

const wrapper = ({ children }: { children: React.ReactNode }) => <ProgressProvider>{children}</ProgressProvider>;

test('один вызов complete добавляет одну попытку', () => {
  const { result } = renderHook(() => useProgress(), { wrapper });
  act(() => result.current.complete({ type: 'quiz:1.1', moduleId: '1.1', score: 8, total: 10 }));
  expect(result.current.progress.stats.completed).toBe(1);
  expect(result.current.progress.results['1.1']).toBe(80);
});
```

Run: `cd web && npm test -- src/storage`  
Expected: все storage tests PASS.

- [ ] **Step 4: Commit**

```bash
git add web/src/storage
git commit -m "feat: add versioned progress storage and migration"
```

---

### Task 4: App Shell, маршруты и главный каталог

**Files:**
- Modify: `web/src/app/App.tsx`, `web/src/main.tsx`, `web/src/styles/global.css`.
- Create: `web/src/app/AppLayout.tsx`, `web/src/features/course/CourseHome.tsx`, `web/src/components/ProgressBar.tsx`.
- Test: `web/src/features/course/CourseHome.test.tsx`.

**Interfaces:**
- Consumes: `content`, `ProgressProvider`.
- Produces: маршруты `/`, `/module/:moduleId/theory/:slide?`, `/module/:moduleId/quiz`, `/module/:moduleId/result`, `/micro`, `/complex`, `/mistakes`, `/progress`.

- [ ] **Step 1: Написать failing test каталога**

`web/src/features/course/CourseHome.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { CourseHome } from './CourseHome';

test('показывает активные и заблокированные модули', () => {
  render(<MemoryRouter><CourseHome /></MemoryRouter>);
  expect(screen.getByRole('link', { name: /Морфологические группы бактерий/ })).toHaveAttribute('href', '/module/1.1/theory/0');
  expect(screen.getByText('Бактериофаги: биология и практическое применение').closest('[aria-disabled="true"]')).not.toBeNull();
});
```

Run: `cd web && npm test -- src/features/course/CourseHome.test.tsx`  
Expected: FAIL, компонент отсутствует.

- [ ] **Step 2: Реализовать каталог без изменения учебного сценария**

`web/src/features/course/CourseHome.tsx`:

```tsx
import { Link } from 'react-router-dom';
import { content } from '../../data/content';

export function CourseHome() {
  const active = content.modules.filter((module) => !module.locked);
  const locked = content.modules.filter((module) => module.locked);
  return (
    <section aria-labelledby="course-title">
      <p className="eyebrow">Общая медицинская микробиология</p>
      <h1 id="course-title">Учебные модули</h1>
      <p className="lede">Теория, вопросы с объяснениями и повторение ошибок.</p>
      <div className="module-list">
        {active.map((module) => (
          <Link className="module-row" key={module.id} to={`/module/${module.id}/theory/0`}>
            <span className="module-number">{module.id}</span>
            <span><strong>{module.title}</strong><small>{module.qs} · {module.theory.length} занятий · {module.quiz.length} вопросов</small></span>
            <span aria-hidden="true">→</span>
          </Link>
        ))}
      </div>
      <h2>Следующие модули</h2>
      <div className="module-list">
        {locked.map((module) => (
          <div className="module-row is-locked" aria-disabled="true" key={module.id}>
            <span className="module-number">{module.id}</span>
            <span><strong>{module.title}</strong><small>{module.qs} · скоро</small></span>
          </div>
        ))}
      </div>
    </section>
  );
}
```

`web/src/components/ProgressBar.tsx`:

```tsx
export function ProgressBar({ value, label }: { value: number; label: string }) {
  const safe = Math.max(0, Math.min(100, value));
  return <div className="progress-track" role="progressbar" aria-label={label} aria-valuemin={0} aria-valuemax={100} aria-valuenow={safe}><span style={{ width: `${safe}%` }} /></div>;
}
```

- [ ] **Step 3: Подключить layout и маршруты**

`web/src/app/AppLayout.tsx`:

```tsx
import { NavLink, Outlet } from 'react-router-dom';

const links = [
  ['/', 'Курс'], ['/micro', 'Практикум'], ['/mistakes', 'Мои ошибки'], ['/progress', 'Прогресс'],
];

export function AppLayout() {
  return (
    <div className="app-shell">
      <header className="topbar"><NavLink to="/" className="brand">Микро · атлас</NavLink><span>Тренажёр к экзамену</span></header>
      <nav className="side-nav" aria-label="Основная навигация">
        {links.map(([to, label]) => <NavLink key={to} to={to} end={to === '/'}>{label}</NavLink>)}
      </nav>
      <main className="workspace"><Outlet /></main>
    </div>
  );
}
```

`web/src/app/App.tsx`:

```tsx
import { Navigate, Route, Routes } from 'react-router-dom';
import { AppLayout } from './AppLayout';
import { CourseHome } from '../features/course/CourseHome';

export function App() {
  return (
    <Routes>
      <Route element={<AppLayout />}>
        <Route index element={<CourseHome />} />
        <Route path="micro" element={<h1>Практикум</h1>} />
        <Route path="mistakes" element={<h1>Мои ошибки</h1>} />
        <Route path="progress" element={<h1>Прогресс</h1>} />
        <Route path="*" element={<Navigate to="/" replace />} />
      </Route>
    </Routes>
  );
}
```

В `web/src/main.tsx` обернуть `App` в `ProgressProvider` внутри `HashRouter`:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { HashRouter } from 'react-router-dom';
import { App } from './app/App';
import { ProgressProvider } from './storage/ProgressContext';
import './styles/global.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <HashRouter>
      <ProgressProvider><App /></ProgressProvider>
    </HashRouter>
  </React.StrictMode>,
);
```

- [ ] **Step 4: Добавить новую адаптивную сетку**

Append to `web/src/styles/global.css`:

```css
.app-shell { min-height: 100vh; display: grid; grid-template-columns: 220px minmax(0, 1fr); grid-template-rows: 72px 1fr; }
.topbar { grid-column: 1 / -1; display: flex; align-items: center; justify-content: space-between; padding: 0 32px; border-bottom: 1px solid var(--line); background: var(--surface); color: var(--muted); }
.brand { color: var(--ink); font: 700 23px Georgia, serif; text-decoration: none; }
.side-nav { display: flex; flex-direction: column; gap: 6px; padding: 28px 18px; border-right: 1px solid var(--line); background: #ece6da; }
.side-nav a { color: var(--muted); text-decoration: none; padding: 11px 14px; border-radius: 9px; }
.side-nav a.active { color: var(--accent); background: #d8e5dc; font-weight: 700; }
.workspace { width: min(980px, 100%); padding: 38px 34px 80px; }
.eyebrow { color: var(--warning); font-size: 12px; font-weight: 700; letter-spacing: .09em; text-transform: uppercase; }
.lede { color: var(--muted); }
.module-list { display: grid; gap: 10px; margin: 22px 0 34px; }
.module-row { display: flex; align-items: center; gap: 16px; padding: 17px; border: 1px solid var(--line); border-radius: 14px; background: var(--surface); color: var(--ink); text-decoration: none; }
.module-row > span:nth-child(2) { flex: 1; }
.module-row strong, .module-row small { display: block; }
.module-row small { margin-top: 5px; color: var(--muted); }
.module-number { width: 42px; height: 42px; display: grid; place-items: center; border-radius: 50%; background: var(--accent-soft); color: var(--accent); font-weight: 700; }
.module-row.is-locked { opacity: .58; }
.progress-track { height: 6px; overflow: hidden; border-radius: 6px; background: var(--line); }
.progress-track span { display: block; height: 100%; background: var(--accent); }
@media (max-width: 720px) {
  .app-shell { display: block; }
  .topbar { height: 62px; padding: 0 16px; }
  .topbar > span { display: none; }
  .side-nav { position: sticky; top: 0; z-index: 5; flex-direction: row; overflow-x: auto; padding: 8px 12px; border-right: 0; border-bottom: 1px solid var(--line); }
  .side-nav a { white-space: nowrap; padding: 8px 10px; }
  .workspace { padding: 25px 15px 60px; }
}
@media (prefers-reduced-motion: reduce) { *, *::before, *::after { scroll-behavior: auto !important; transition: none !important; } }
```

Обновить `web/src/app/App.test.tsx`, чтобы smoke-test использовал router и новый главный экран:

```tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { App } from './App';
import { ProgressProvider } from '../storage/ProgressContext';

test('показывает главный каталог курса', () => {
  render(<MemoryRouter><ProgressProvider><App /></ProgressProvider></MemoryRouter>);
  expect(screen.getByRole('heading', { name: 'Учебные модули' })).toBeInTheDocument();
});
```

Run: `cd web && npm test -- src/features/course/CourseHome.test.tsx src/app/App.test.tsx && npm run build`  
Expected: tests PASS; build PASS.

- [ ] **Step 5: Commit**

```bash
git add web/src
git commit -m "feat: add responsive app shell and course catalog"
```

---

### Task 5: Последовательная теория и учебные материалы

**Files:**
- Create: `web/src/components/RichHtml.tsx`, `web/src/features/course/TheoryPage.tsx`.
- Modify: `web/src/app/App.tsx`, `web/src/styles/global.css`.
- Test: `web/src/features/course/TheoryPage.test.tsx`.

**Interfaces:**
- Consumes: `getModule`, `useProgress().setDraft`.
- Produces: маршрут `/module/:moduleId/theory/:slide?`; `RichHtml` для доверенного bundled HTML.

- [ ] **Step 1: Написать failing tests навигации по занятиям**

`web/src/features/course/TheoryPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { ProgressProvider } from '../../storage/ProgressContext';
import { TheoryPage } from './TheoryPage';

function renderTheory(route = '/module/1.1/theory/0') {
  return render(<MemoryRouter initialEntries={[route]}><ProgressProvider><Routes>
    <Route path="/module/:moduleId/theory/:slide?" element={<TheoryPage />} />
    <Route path="/module/:moduleId/quiz" element={<h1>Тест</h1>} />
  </Routes></ProgressProvider></MemoryRouter>);
}

test('переходит к следующему занятию', async () => {
  renderTheory();
  expect(screen.getByText(/Теория 1 из 6/)).toBeInTheDocument();
  await userEvent.click(screen.getByRole('button', { name: 'Дальше' }));
  expect(screen.getByText(/Теория 2 из 6/)).toBeInTheDocument();
});

test('не открывает заблокированный модуль', () => {
  renderTheory('/module/3/theory/0');
  expect(screen.getByRole('heading', { name: 'Раздел не найден' })).toBeInTheDocument();
});
```

Run: `cd web && npm test -- src/features/course/TheoryPage.test.tsx`  
Expected: FAIL, компонент отсутствует.

- [ ] **Step 2: Реализовать безопасный рендер только локального HTML**

`web/src/components/RichHtml.tsx`:

```tsx
import { useState, type SyntheticEvent } from 'react';

export function RichHtml({ html, className = '' }: { html: string; className?: string }) {
  const [imageFailed, setImageFailed] = useState(false);
  function handleError(event: SyntheticEvent<HTMLDivElement>) {
    if ((event.target as HTMLElement).tagName === 'IMG') {
      (event.target as HTMLElement).style.display = 'none';
      setImageFailed(true);
    }
  }
  return (
    <div className={`rich-html ${className}`} onErrorCapture={handleError}>
      <div dangerouslySetInnerHTML={{ __html: html }} />
      {imageFailed && <p role="alert">Изображение не загрузилось. Текст задания остаётся доступным.</p>}
    </div>
  );
}
```

HTML поступает только из сгенерированного `legacy-content.json`; пользовательский ввод в `dangerouslySetInnerHTML` не передаётся.

- [ ] **Step 3: Реализовать TheoryPage**

`web/src/features/course/TheoryPage.tsx`:

```tsx
import { useEffect } from 'react';
import { Link, useNavigate, useParams } from 'react-router-dom';
import { getModule } from '../../data/content';
import { RichHtml } from '../../components/RichHtml';
import { ProgressBar } from '../../components/ProgressBar';
import { useProgress } from '../../storage/ProgressContext';

export function TheoryPage() {
  const { moduleId, slide: rawSlide = '0' } = useParams();
  const navigate = useNavigate();
  const { setDraft } = useProgress();
  const module = getModule(moduleId);
  const slide = Number.parseInt(rawSlide, 10);

  useEffect(() => {
    if (module && !module.locked && slide >= 0 && slide < module.theory.length) {
      setDraft({ kind: 'theory', moduleId: module.id, slide, updatedAt: Date.now() });
    }
  }, [module, slide, setDraft]);

  if (!module || module.locked || !Number.isInteger(slide) || slide < 0 || slide >= module.theory.length) {
    return <section><h1>Раздел не найден</h1><Link to="/">Вернуться к модулям</Link></section>;
  }
  const lesson = module.theory[slide];
  const last = slide === module.theory.length - 1;
  return (
    <section className="lesson-view">
      <Link to="/">← Все модули</Link>
      <p className="eyebrow">Подмодуль {module.id} · Теория {slide + 1} из {module.theory.length}</p>
      <ProgressBar value={(slide + 1) / module.theory.length * 100} label="Прогресс теории" />
      <article className="paper-card">
        <h1>{lesson.h}</h1>
        {lesson.img && <RichHtml html={lesson.img} className="lesson-illustration" />}
        <RichHtml html={lesson.body} />
      </article>
      <div className="lesson-actions">
        <button disabled={slide === 0} onClick={() => navigate(`/module/${module.id}/theory/${slide - 1}`)}>Назад</button>
        {last
          ? <button className="primary" onClick={() => navigate(`/module/${module.id}/quiz`)}>К тесту →</button>
          : <button className="primary" onClick={() => navigate(`/module/${module.id}/theory/${slide + 1}`)}>Дальше</button>}
      </div>
    </section>
  );
}
```

Добавить route перед `*`:

```tsx
<Route path="module/:moduleId/theory/:slide?" element={<TheoryPage />} />
```

и импорт:

```tsx
import { TheoryPage } from '../features/course/TheoryPage';
```

- [ ] **Step 4: Стилизовать чтение и таблицы**

Append to `web/src/styles/global.css`:

```css
.paper-card { margin-top: 20px; padding: clamp(20px, 4vw, 42px); border: 1px solid var(--line); border-radius: 18px; background: var(--surface); }
.rich-html { line-height: 1.65; overflow-wrap: anywhere; }
.rich-html img, .rich-html svg { display: block; max-width: 100%; height: auto; margin: 14px auto; }
.rich-html table { width: 100%; border-collapse: collapse; display: block; overflow-x: auto; }
.rich-html th, .rich-html td { padding: 10px; border: 1px solid var(--line); text-align: left; vertical-align: top; }
.rich-html th { background: var(--accent-soft); }
.lesson-illustration { margin: 18px 0; }
.lesson-actions { display: flex; justify-content: space-between; gap: 12px; margin-top: 18px; }
.lesson-actions button, .primary { border: 1px solid var(--line); border-radius: 9px; padding: 11px 18px; background: var(--surface); color: var(--ink); cursor: pointer; }
.lesson-actions button.primary, button.primary { border-color: var(--accent); background: var(--accent); color: white; }
.lesson-actions button:disabled { opacity: .4; cursor: default; }
@media (max-width: 560px) { .paper-card { padding: 18px 15px; } .rich-html { font-size: 15px; } }
```

Run: `cd web && npm test -- src/features/course/TheoryPage.test.tsx && npm run build`  
Expected: 2 tests PASS; build PASS.

- [ ] **Step 5: Commit**

```bash
git add web/src
git commit -m "feat: rebuild sequential theory experience"
```

---

### Task 6: Универсальный движок тестов и повторения ошибок

**Files:**
- Create: `web/src/features/practice/run.ts`.
- Test: `web/src/features/practice/run.test.ts`.

**Interfaces:**
- Produces: `RunState<K>`, `createRun`, `answerRun`, `advanceRun`, `shuffle`; используется обычными вопросами и микропрепаратами.

- [ ] **Step 1: Написать failing tests для первой попытки и циклов ошибок**

`web/src/features/practice/run.test.ts`:

```ts
import { advanceRun, answerRun, createRun, shuffle } from './run';

test('учитывает правильный ответ только с первой попытки', () => {
  let run = createRun([0, 1, 2]);
  run = answerRun(run, 0, true);
  expect(run.first).toEqual([0]);
  run = advanceRun(run);
  run = answerRun(run, 1, false);
  run = advanceRun(run);
  run = answerRun(run, 2, true);
  run = advanceRun(run);
  expect(run.phase).toBe('repeat');
  expect(run.order).toEqual([1]);
  expect(run.first).toEqual([0, 2]);
});

test('завершается после исправления ошибки', () => {
  let run = createRun([0]);
  run = answerRun(run, 0, false);
  run = advanceRun(run);
  run = answerRun(run, 0, true);
  run = advanceRun(run);
  expect(run.done).toBe(true);
  expect(run.wrong).toEqual([]);
  expect(run.first).toEqual([]);
});

test('shuffle не теряет элементы', () => {
  const input = [1, 2, 3, 4, 5];
  expect(shuffle(input).sort()).toEqual(input);
  expect(input).toEqual([1, 2, 3, 4, 5]);
});
```

Run: `cd web && npm test -- src/features/practice/run.test.ts`  
Expected: FAIL, модуль отсутствует.

- [ ] **Step 2: Реализовать чистый generic-движок**

`web/src/features/practice/run.ts`:

```ts
export interface RunState<K extends string | number> {
  order: K[];
  index: number;
  answered: boolean;
  lastCorrect: boolean | null;
  wrong: K[];
  first: K[];
  phase: 'main' | 'repeat';
  done: boolean;
}

export function createRun<K extends string | number>(order: K[]): RunState<K> {
  return { order: [...order], index: 0, answered: false, lastCorrect: null, wrong: [], first: [], phase: 'main', done: order.length === 0 };
}

export function answerRun<K extends string | number>(run: RunState<K>, key: K, correct: boolean): RunState<K> {
  if (run.answered || run.done) return run;
  return {
    ...run,
    answered: true,
    lastCorrect: correct,
    first: correct && run.phase === 'main' && !run.first.includes(key) ? [...run.first, key] : run.first,
    wrong: correct ? run.wrong.filter((item) => item !== key) : run.wrong.includes(key) ? run.wrong : [...run.wrong, key],
  };
}

export function advanceRun<K extends string | number>(run: RunState<K>): RunState<K> {
  if (!run.answered || run.done) return run;
  if (run.index < run.order.length - 1) {
    return { ...run, index: run.index + 1, answered: false, lastCorrect: null };
  }
  if (run.wrong.length > 0) {
    return { ...run, order: [...run.wrong], index: 0, answered: false, lastCorrect: null, wrong: [], phase: 'repeat' };
  }
  return { ...run, done: true, answered: false, lastCorrect: null };
}

export function shuffle<T>(items: T[]): T[] {
  const result = [...items];
  for (let index = result.length - 1; index > 0; index -= 1) {
    const other = Math.floor(Math.random() * (index + 1));
    [result[index], result[other]] = [result[other], result[index]];
  }
  return result;
}
```

Run: `cd web && npm test -- src/features/practice/run.test.ts`  
Expected: 3 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add web/src/features/practice
git commit -m "feat: add reusable quiz and mistake-repeat engine"
```

---

### Task 7: Экраны теста и результата

**Files:**
- Create: `web/src/features/course/QuizPage.tsx`, `web/src/features/course/ResultPage.tsx`.
- Modify: `web/src/app/App.tsx`, `web/src/styles/global.css`.
- Test: `web/src/features/course/QuizPage.test.tsx`, `web/src/features/course/ResultPage.test.tsx`.

**Interfaces:**
- Consumes: `RunState<number>`, `complete`, `setDraft`.
- Produces: `/module/:moduleId/quiz` и `/module/:moduleId/result`; location state `{ first: number[] }` для разбора результата.

- [ ] **Step 1: Написать failing integration-test**

`web/src/features/course/QuizPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { ProgressProvider } from '../../storage/ProgressContext';
import { QuizPage } from './QuizPage';

test('показывает объяснение после ответа и запрещает второй ответ', async () => {
  render(<MemoryRouter initialEntries={['/module/1.1/quiz']}><ProgressProvider><Routes>
    <Route path="/module/:moduleId/quiz" element={<QuizPage />} />
    <Route path="/module/:moduleId/result" element={<h1>Результат</h1>} />
  </Routes></ProgressProvider></MemoryRouter>);
  const options = screen.getAllByRole('button', { name: /^[а-я] / });
  await userEvent.click(options[0]);
  expect(screen.getByRole('status')).toHaveTextContent(/Верно|Неверно/);
  expect(options[0]).toBeDisabled();
  expect(screen.getByRole('button', { name: /Следующий вопрос|Итоги|повторению/ })).toBeEnabled();
});
```

Run: `cd web && npm test -- src/features/course/QuizPage.test.tsx`  
Expected: FAIL, компонент отсутствует.

- [ ] **Step 2: Реализовать QuizPage с защитой от двойной записи**

`web/src/features/course/QuizPage.tsx`:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Link, useNavigate, useParams } from 'react-router-dom';
import { getModule } from '../../data/content';
import { ProgressBar } from '../../components/ProgressBar';
import { RichHtml } from '../../components/RichHtml';
import { useProgress } from '../../storage/ProgressContext';
import { advanceRun, answerRun, createRun } from '../practice/run';

const letters = ['а', 'б', 'в', 'г', 'д'];

export function QuizPage() {
  const { moduleId } = useParams();
  const navigate = useNavigate();
  const { complete, setDraft } = useProgress();
  const module = getModule(moduleId);
  const [run, setRun] = useState(() => createRun(module?.quiz.map((_, index) => index) ?? []));
  const [hint, setHint] = useState(false);
  const recorded = useRef(false);

  useEffect(() => {
    if (module && !run.done) setDraft({ kind: 'quiz', moduleId: module.id, updatedAt: Date.now() });
  }, [module, run.done, setDraft]);

  useEffect(() => {
    if (module && run.done && !recorded.current) {
      recorded.current = true;
      complete({ type: `quiz:${module.id}`, moduleId: module.id, score: run.first.length, total: module.quiz.length });
      setDraft(null);
      navigate(`/module/${module.id}/result`, { state: { first: run.first } });
    }
  }, [module, run, complete, setDraft, navigate]);

  if (!module || module.locked || module.quiz.length === 0) {
    return <section><h1>Тест недоступен</h1><Link to="/">К модулям</Link></section>;
  }
  const key = run.order[run.index];
  const question = module.quiz[key];
  if (!question) return null;
  const last = run.index === run.order.length - 1;
  return (
    <section>
      <Link to={`/module/${module.id}/theory/0`}>← Выйти к теории</Link>
      <p className="eyebrow">{run.phase === 'repeat' ? 'Повторение ошибок' : `Подмодуль ${module.id} · тест`}</p>
      <p>Вопрос {run.index + 1} из {run.order.length}</p>
      <ProgressBar value={(run.index + (run.answered ? 1 : 0)) / run.order.length * 100} label="Прогресс теста" />
      <article className="paper-card">
        <h1 className="question-title">{question.q}</h1>
        {question.img && <RichHtml html={question.img} />}
        <div className="answer-list">
          {question.o.map((option, index) => (
            <button key={option} disabled={run.answered} className="answer-option" onClick={() => setRun(answerRun(run, key, index === question.a))}>
              <span>{letters[index]}</span>{option}
            </button>
          ))}
        </div>
        {!run.answered && question.h && <button className="text-button" onClick={() => setHint(true)}>Показать подсказку</button>}
        {hint && !run.answered && <p className="hint">{question.h}</p>}
        {run.answered && <div role="status" className="feedback">
          <strong>{run.lastCorrect ? 'Верно.' : 'Неверно.'}</strong>
          <p><b>Правильный ответ: {letters[question.a]}.</b> {question.e}</p>
        </div>}
        {run.answered && <button className="primary" onClick={() => { setHint(false); setRun(advanceRun(run)); }}>
          {last ? (run.wrong.length ? 'К повторению ошибок →' : 'Итоги →') : 'Следующий вопрос →'}
        </button>}
      </article>
    </section>
  );
}
```

- [ ] **Step 3: Реализовать результат по темам**

`web/src/features/course/ResultPage.tsx`:

```tsx
import { Link, useLocation, useParams } from 'react-router-dom';
import { content, getModule } from '../../data/content';
import { ProgressBar } from '../../components/ProgressBar';
import { useProgress } from '../../storage/ProgressContext';

export function ResultPage() {
  const { moduleId } = useParams();
  const location = useLocation();
  const { progress } = useProgress();
  const module = getModule(moduleId);
  if (!module) return <h1>Результат не найден</h1>;
  const first = (location.state as { first?: number[] } | null)?.first ?? [];
  const percent = progress.results[module.id] ?? Math.round(first.length / module.quiz.length * 100);
  const topics = [...new Set(module.quiz.map((question) => question.topic ?? 'Общее'))];
  const active = content.modules.filter((item) => !item.locked);
  const next = active[active.findIndex((item) => item.id === module.id) + 1];
  return (
    <section>
      <p className="eyebrow">Подмодуль {module.id}</p><h1>Тест завершён</h1>
      <div className="result-score">{percent}%</div>
      <ProgressBar value={percent} label="Результат теста" />
      <h2>Результат по темам</h2>
      {topics.map((topic) => {
        const indexes = module.quiz.map((question, index) => ({ question, index })).filter(({ question }) => (question.topic ?? 'Общее') === topic);
        const correct = indexes.filter(({ index }) => first.includes(index)).length;
        return <p key={topic}>{topic}: <b>{correct}/{indexes.length}</b></p>;
      })}
      {first.length < module.quiz.length && <><h2>Разбор вопросов с ошибками</h2>{module.quiz.filter((_, index) => !first.includes(index)).map((question) => <details key={question.q}><summary>{question.q}</summary><p>{question.o[question.a]} — {question.e}</p></details>)}</>}
      <div className="lesson-actions"><Link className="primary" to={`/module/${module.id}/quiz`}>Пройти заново</Link><Link to={`/module/${module.id}/theory/0`}>Перечитать теорию</Link>{next && <Link to={`/module/${next.id}/theory/0`}>Следующий подмодуль →</Link>}</div>
    </section>
  );
}
```

`web/src/features/course/ResultPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { ProgressProvider } from '../../storage/ProgressContext';
import { ResultPage } from './ResultPage';

test('показывает итог и разбор', () => {
  render(<MemoryRouter initialEntries={[{ pathname: '/module/1.1/result', state: { first: [] } }]}><ProgressProvider><Routes><Route path="/module/:moduleId/result" element={<ResultPage />} /></Routes></ProgressProvider></MemoryRouter>);
  expect(screen.getByRole('heading', { name: 'Тест завершён' })).toBeInTheDocument();
  expect(screen.getByText('0%')).toBeInTheDocument();
  expect(screen.getByRole('heading', { name: 'Разбор вопросов с ошибками' })).toBeInTheDocument();
});
```

Добавить imports и routes:

```tsx
<Route path="module/:moduleId/quiz" element={<QuizPage />} />
<Route path="module/:moduleId/result" element={<ResultPage />} />
```

- [ ] **Step 4: Добавить состояния ответов и запустить проверки**

Append to `web/src/styles/global.css`:

```css
.question-title { font-size: clamp(22px, 3vw, 30px); line-height: 1.25; }
.answer-list { display: grid; gap: 9px; margin: 20px 0; }
.answer-option { display: flex; gap: 12px; width: 100%; padding: 13px; border: 1px solid var(--line); border-radius: 10px; background: white; color: var(--ink); text-align: left; cursor: pointer; }
.answer-option span { color: var(--warning); font-weight: 700; text-transform: uppercase; }
.answer-option:disabled { cursor: default; }
.feedback { margin: 16px 0; padding: 15px; border-radius: 10px; background: var(--accent-soft); }
.hint { padding: 12px; background: #f4e8d3; border-radius: 9px; }
.text-button { border: 0; background: none; color: var(--accent); text-decoration: underline; cursor: pointer; }
.result-score { font: 56px Georgia, serif; color: var(--accent); }
details { margin: 8px 0; padding: 12px; border: 1px solid var(--line); border-radius: 9px; background: var(--surface); }
summary { cursor: pointer; font-weight: 600; }
```

Run: `cd web && npm test -- src/features/course/QuizPage.test.tsx src/features/course/ResultPage.test.tsx src/features/practice/run.test.ts && npm run build`  
Expected: tests PASS; build PASS.

- [ ] **Step 5: Commit**

```bash
git add web/src
git commit -m "feat: rebuild quizzes and detailed results"
```

---

### Task 8: Практикум по микропрепаратам

**Files:**
- Create: `web/src/features/microscopy/MicroscopyPage.tsx`.
- Modify: `web/src/app/App.tsx`, `web/src/styles/global.css`.
- Test: `web/src/features/microscopy/MicroscopyPage.test.tsx`.

**Interfaces:**
- Consumes: `allMicroTasks`, `RunState<string>`, `progress.microWrong`, `setMicroWrong`, `complete`.
- Produces: полный `/micro` с режимами всех заданий и повторения ошибок.

- [ ] **Step 1: Написать failing test запуска и пустых ошибок**

`web/src/features/microscopy/MicroscopyPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { ProgressProvider } from '../../storage/ProgressContext';
import { MicroscopyPage } from './MicroscopyPage';

function renderPage() {
  return render(<MemoryRouter><ProgressProvider><MicroscopyPage /></ProgressProvider></MemoryRouter>);
}

test('запускает полный набор из 24 заданий', async () => {
  renderPage();
  await userEvent.click(screen.getByRole('button', { name: 'Начать все задания' }));
  expect(screen.getByText('Задание 1 из 24')).toBeInTheDocument();
});

test('показывает пустое состояние повторения ошибок', async () => {
  renderPage();
  await userEvent.click(screen.getByRole('button', { name: 'Повторить только ошибки' }));
  expect(screen.getByRole('status')).toHaveTextContent('Ошибок для повторения пока нет');
});
```

Run: `cd web && npm test -- src/features/microscopy/MicroscopyPage.test.tsx`  
Expected: FAIL, компонент отсутствует.

- [ ] **Step 2: Реализовать практикум**

`web/src/features/microscopy/MicroscopyPage.tsx`:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Link } from 'react-router-dom';
import { allMicroTasks, content } from '../../data/content';
import { ProgressBar } from '../../components/ProgressBar';
import { RichHtml } from '../../components/RichHtml';
import { useProgress } from '../../storage/ProgressContext';
import { advanceRun, answerRun, createRun, shuffle, type RunState } from '../practice/run';

const letters = ['а', 'б', 'в', 'г', 'д'];

export function MicroscopyPage() {
  const { progress, complete, setMicroWrong, setDraft } = useProgress();
  const [run, setRun] = useState<RunState<string> | null>(null);
  const [tasks, setTasks] = useState(allMicroTasks);
  const [random, setRandom] = useState(true);
  const [hints, setHints] = useState(true);
  const [hint, setHint] = useState(false);
  const [notice, setNotice] = useState('');
  const recorded = useRef(false);

  function start(mistakesOnly: boolean) {
    const selected = mistakesOnly ? allMicroTasks.filter((task) => progress.microWrong.includes(task.key)) : allMicroTasks;
    if (!selected.length) {
      setNotice('Ошибок для повторения пока нет. Сначала пройди полный набор.');
      return;
    }
    const ordered = random ? shuffle(selected) : selected;
    setTasks(ordered);
    setRun(createRun(ordered.map((task) => task.key)));
    setNotice('');
    setHint(false);
    recorded.current = false;
    setDraft({ kind: 'micro', updatedAt: Date.now() });
  }

  useEffect(() => {
    if (run?.done && !recorded.current) {
      recorded.current = true;
      complete({ type: 'micro', score: run.first.length, total: tasks.length });
      const allKeys = new Set(allMicroTasks.map((task) => task.key));
      setMicroWrong([...new Set([...progress.microWrong.filter((key) => !allKeys.has(key)), ...run.wrong])]);
      setDraft(null);
    }
  }, [run, tasks.length, complete, setMicroWrong, setDraft, progress.microWrong]);

  if (!run) return (
    <section><p className="eyebrow">Практикум</p><h1>Распознавание микропрепаратов</h1>
      <p className="lede">{content.microCases.length} учебных изображений · {allMicroTasks.length} заданий. Не делай видовой вывод по одному признаку.</p>
      <div role="status">{notice}</div>
      <label><input type="checkbox" checked={hints} onChange={() => setHints(!hints)} /> Подсказки</label>
      <label><input type="checkbox" checked={random} onChange={() => setRandom(!random)} /> Случайный порядок</label>
      <div className="stack-actions"><button className="primary" onClick={() => start(false)}>Начать все задания</button><button onClick={() => start(true)}>Повторить только ошибки</button><Link to="/complex">Комплексное описание препарата →</Link></div>
      <h2>Что встретится</h2>{content.microCases.map((item) => <details key={item.id}><summary>{item.title}</summary><p>{item.tasks.length} заданий · {item.source}</p></details>)}
    </section>
  );

  if (run.done) return <section><h1>Практикум завершён</h1><div className="result-score">{Math.round(run.first.length / tasks.length * 100)}%</div><p>{run.first.length} из {tasks.length} верно с первой попытки.</p><button className="primary" onClick={() => start(false)}>Пройти снова</button><button onClick={() => setRun(null)}>К практикуму</button></section>;

  const key = run.order[run.index];
  const task = tasks.find((item) => item.key === key)!;
  return (
    <section><button onClick={() => setRun(null)}>← К практикуму</button><p className="eyebrow">{run.phase === 'repeat' ? 'Повторение ошибок' : task.caseTitle}</p>
      <p>Задание {run.index + 1} из {run.order.length}</p><ProgressBar value={(run.index + (run.answered ? 1 : 0)) / run.order.length * 100} label="Прогресс практикума" />
      <article className="paper-card"><RichHtml html={task.img} /><p className="micro-source">{task.source}</p><h1 className="question-title">{task.q}</h1>
        <div className="answer-list">{task.o.map((option, index) => <button className="answer-option" disabled={run.answered} key={option} onClick={() => setRun(answerRun(run, key, index === task.a))}><span>{letters[index]}</span>{option}</button>)}</div>
        {hints && !run.answered && <button className="text-button" onClick={() => setHint(true)}>Показать ориентир</button>}{hint && !run.answered && <p className="hint">Сначала определи тип изображения, затем оценивай признак. Не делай видовой вывод по одному признаку.</p>}
        {run.answered && <><div className="feedback" role="status"><strong>{run.lastCorrect ? 'Верно.' : 'Неверно.'}</strong><b>Правильный ответ: {task.o[task.a]}.</b><p>{task.e}</p></div><button className="primary" onClick={() => { setHint(false); setRun(advanceRun(run)); }}>Далее →</button></>}
      </article>
    </section>
  );
}
```

- [ ] **Step 3: Подключить route, стили и проверить**

В `App.tsx` заменить placeholder `/micro` на:

```tsx
<Route path="micro" element={<MicroscopyPage />} />
```

с импортом:

```tsx
import { MicroscopyPage } from '../features/microscopy/MicroscopyPage';
```

Append to `web/src/styles/global.css`:

```css
.stack-actions { display: flex; flex-wrap: wrap; gap: 10px; margin: 22px 0; }
.stack-actions > * { padding: 11px 16px; border-radius: 9px; }
.micro-source { color: var(--muted); font-size: 13px; }
```

Run: `cd web && npm test -- src/features/microscopy/MicroscopyPage.test.tsx src/features/practice/run.test.ts && npm run build`  
Expected: tests PASS; build PASS.

- [ ] **Step 4: Commit**

```bash
git add web/src
git commit -m "feat: rebuild microscopy practice and mistake review"
```

---

### Task 9: Комплексное описание препарата

**Files:**
- Create: `web/src/features/complex-cases/ComplexCasesPage.tsx`.
- Modify: `web/src/app/App.tsx`, `web/src/styles/global.css`.
- Test: `web/src/features/complex-cases/ComplexCasesPage.test.tsx`.

**Interfaces:**
- Consumes: `content.complexCases`, `content.complexFields`, `progress.complexWrong`, `setComplexWrong`, `complete`.
- Produces: `/complex` с полным набором и повторением ошибок.

- [ ] **Step 1: Написать failing test обязательных полей и проверки**

`web/src/features/complex-cases/ComplexCasesPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { ProgressProvider } from '../../storage/ProgressContext';
import { ComplexCasesPage } from './ComplexCasesPage';

test('не проверяет описание, пока не заполнены все поля', async () => {
  render(<MemoryRouter><ProgressProvider><ComplexCasesPage /></ProgressProvider></MemoryRouter>);
  const check = screen.getByRole('button', { name: 'Проверить описание' });
  expect(check).toBeDisabled();
  for (const select of screen.getAllByRole('combobox')) {
    const options = Array.from(select.querySelectorAll('option')).filter((option) => option.value);
    await userEvent.selectOptions(select, options[0] as HTMLOptionElement);
  }
  expect(check).toBeEnabled();
  await userEvent.click(check);
  expect(screen.getByRole('status')).toHaveTextContent('Разбор препарата');
});
```

Run: `cd web && npm test -- src/features/complex-cases/ComplexCasesPage.test.tsx`  
Expected: FAIL, компонент отсутствует.

- [ ] **Step 2: Реализовать комплексные задания**

`web/src/features/complex-cases/ComplexCasesPage.tsx`:

```tsx
import { useRef, useState } from 'react';
import { Link } from 'react-router-dom';
import { content } from '../../data/content';
import type { ComplexCase, ComplexFieldId } from '../../data/types';
import { RichHtml } from '../../components/RichHtml';
import { useProgress } from '../../storage/ProgressContext';

type Values = Partial<Record<ComplexFieldId, string>>;

export function ComplexCasesPage() {
  const { progress, complete, setComplexWrong, setDraft } = useProgress();
  const [cases, setCases] = useState<ComplexCase[]>(content.complexCases);
  const [index, setIndex] = useState(0);
  const [values, setValues] = useState<Values>({});
  const [checked, setChecked] = useState(false);
  const [wrong, setWrong] = useState<string[]>([]);
  const [first, setFirst] = useState(0);
  const [mainTotal, setMainTotal] = useState(content.complexCases.length);
  const [repeat, setRepeat] = useState(false);
  const [done, setDone] = useState(false);
  const [notice, setNotice] = useState('');
  const recorded = useRef(false);

  function start(items: ComplexCase[], isRepeat = false) {
    if (!items.length) {
      setNotice('Ошибок комплексного описания пока нет.');
      return;
    }
    setCases(items); setIndex(0); setValues({}); setChecked(false); setWrong([]); setRepeat(isRepeat); setDone(false); setNotice('');
    if (!isRepeat) { setFirst(0); setMainTotal(items.length); }
    recorded.current = false;
    setDraft({ kind: 'complex', updatedAt: Date.now() });
  }

  function check() {
    const current = cases[index];
    const incorrect = content.complexFields.filter((field) => values[field.id] !== current.answers[field.id]);
    setChecked(true);
    if (incorrect.length) setWrong((old) => old.includes(current.id) ? old : [...old, current.id]);
    else {
      setWrong((old) => old.filter((id) => id !== current.id));
      if (!repeat) setFirst((old) => old + 1);
    }
  }

  function next() {
    if (index < cases.length - 1) {
      setIndex(index + 1); setValues({}); setChecked(false); return;
    }
    if (wrong.length && !repeat) {
      start(content.complexCases.filter((item) => wrong.includes(item.id)), true);
      return;
    }
    setDone(true);
    if (!recorded.current) {
      recorded.current = true;
      complete({ type: 'complex', score: first, total: mainTotal });
      const currentIds = new Set(content.complexCases.map((item) => item.id));
      setComplexWrong([...new Set([...progress.complexWrong.filter((id) => !currentIds.has(id)), ...wrong])]);
      setDraft(null);
    }
  }

  if (done) return <section><h1>Комплексный разбор завершён</h1><div className="result-score">{Math.round(first / Math.max(mainTotal, 1) * 100)}%</div><p>{first} из {mainTotal} описаны полностью верно с первой попытки.</p><button className="primary" onClick={() => start(content.complexCases)}>Пройти снова</button><Link to="/micro">К практикуму</Link></section>;

  const current = cases[index];
  const allFilled = content.complexFields.every((field) => values[field.id]);
  return (
    <section><Link to="/micro">← Практикум</Link><p className="eyebrow">{repeat ? 'Работа над ошибками' : 'Комплексное описание'}</p>
      <h1>{current.title}</h1><p>Препарат {index + 1} из {cases.length}</p><div role="status">{notice}</div>
      <div className="stack-actions"><button onClick={() => start(content.complexCases.filter((item) => progress.complexWrong.includes(item.id)), true)}>Повторить сохранённые ошибки</button></div>
      <article className="paper-card"><RichHtml html={current.img} /><div className="complex-grid">
        {content.complexFields.map((field) => <label key={field.id}>{field.label}<select disabled={checked} value={values[field.id] ?? ''} onChange={(event) => setValues({ ...values, [field.id]: event.target.value })}><option value="">Выбери вариант…</option>{field.options.map((option) => <option key={option} value={option}>{option}</option>)}</select></label>)}
      </div>
      {!checked ? <button className="primary" disabled={!allFilled} onClick={check}>Проверить описание</button> : <div role="status" className="feedback"><h2>Разбор препарата</h2>{content.complexFields.map((field) => { const correct = values[field.id] === current.answers[field.id]; return <div key={field.id}><strong>{correct ? '✓ Верно' : '✕ Нужно исправить'} · {field.label}</strong><p>{current.answers[field.id]}</p><small>{current.why[field.id]}</small></div>; })}<button className="primary" onClick={next}>{index === cases.length - 1 ? 'Завершить' : 'Следующий препарат →'}</button></div>}
      </article>
    </section>
  );
}
```

- [ ] **Step 3: Подключить route, стили и проверить**

В `App.tsx` добавить:

```tsx
<Route path="complex" element={<ComplexCasesPage />} />
```

и импорт:

```tsx
import { ComplexCasesPage } from '../features/complex-cases/ComplexCasesPage';
```

Append to `web/src/styles/global.css`:

```css
.complex-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 14px; margin: 20px 0; }
.complex-grid label { display: grid; gap: 6px; font-weight: 600; }
.complex-grid select { width: 100%; min-height: 46px; padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: white; color: var(--ink); }
@media (max-width: 600px) { .complex-grid { grid-template-columns: 1fr; } }
```

Run: `cd web && npm test -- src/features/complex-cases/ComplexCasesPage.test.tsx && npm run build`  
Expected: test PASS; build PASS.

- [ ] **Step 4: Commit**

```bash
git add web/src
git commit -m "feat: rebuild complex microscopy descriptions"
```

---

### Task 10: Восстановление занятий, статистика и устойчивость интерфейса

**Files:**
- Modify: `web/src/storage/progress.ts`, `web/src/storage/ProgressContext.tsx`.
- Modify: `web/src/features/course/QuizPage.tsx`, `web/src/features/microscopy/MicroscopyPage.tsx`, `web/src/features/complex-cases/ComplexCasesPage.tsx`, `web/src/features/course/CourseHome.tsx`.
- Create: `web/src/features/progress/ProgressPage.tsx`, `web/src/features/mistakes/MistakesPage.tsx`, `web/src/app/AppErrorBoundary.tsx`.
- Modify: `web/src/app/App.tsx`, `web/src/main.tsx`.
- Test: `web/src/storage/drafts.test.tsx`, `web/src/features/progress/ProgressPage.test.tsx`.

**Interfaces:**
- Produces: восстановление `RunState` после reload, `/progress`, `/mistakes`, Error Boundary и заметные storage-уведомления.

- [ ] **Step 1: Расширить failing tests для сохранения незавершённой попытки**

`web/src/storage/drafts.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { ProgressProvider } from './ProgressContext';
import { QuizPage } from '../features/course/QuizPage';

test('восстанавливает номер вопроса и состояние ответа после remount', async () => {
  const first = render(<MemoryRouter initialEntries={['/module/1.1/quiz']}><ProgressProvider><Routes><Route path="/module/:moduleId/quiz" element={<QuizPage />} /></Routes></ProgressProvider></MemoryRouter>);
  await userEvent.click(screen.getAllByRole('button', { name: /^[а-я] / })[0]);
  const raw = JSON.parse(localStorage.getItem('microbiology_trainer.progress.v3')!);
  expect(raw.draft.kind).toBe('quiz');
  expect(raw.draft.run.answered).toBe(true);
  first.unmount();
  render(<MemoryRouter initialEntries={['/module/1.1/quiz']}><ProgressProvider><Routes><Route path="/module/:moduleId/quiz" element={<QuizPage />} /></Routes></ProgressProvider></MemoryRouter>);
  expect(screen.getByRole('status')).toBeInTheDocument();
  expect(screen.getAllByRole('button', { name: /^[а-я] / })[0]).toBeDisabled();
});
```

Run: `cd web && npm test -- src/storage/drafts.test.tsx`  
Expected: FAIL, draft пока не содержит `run`.

- [ ] **Step 2: Расширить Draft для полного восстановления попыток**

В `web/src/storage/progress.ts` добавить imports:

```ts
import type { RunState } from '../features/practice/run';
import type { ComplexFieldId } from '../data/types';
```

Заменить `Draft` на:

```ts
export type Draft =
  | { kind: 'theory'; moduleId: string; slide: number; updatedAt: number }
  | { kind: 'quiz'; moduleId: string; run: RunState<number>; updatedAt: number }
  | { kind: 'micro'; taskKeys: string[]; run: RunState<string>; updatedAt: number }
  | { kind: 'complex'; caseIds: string[]; index: number; values: Partial<Record<ComplexFieldId, string>>; checked: boolean; wrong: string[]; first: number; repeat: boolean; updatedAt: number };
```

Стабильные actions уже созданы в Task 3; здесь дополнительно проверить, что effects используют их в dependency arrays без циклических обновлений.

- [ ] **Step 3: Восстанавливать quiz и micro**

В `QuizPage.tsx` получить `progress` из `useProgress`, инициализировать и сохранять run:

```tsx
const { progress, complete, setDraft } = useProgress();
const [run, setRun] = useState(() => {
  const draft = progress.draft;
  return draft?.kind === 'quiz' && draft.moduleId === moduleId ? draft.run : createRun(module?.quiz.map((_, index) => index) ?? []);
});
```

Заменить effect сохранения:

```tsx
useEffect(() => {
  if (module && !run.done) setDraft({ kind: 'quiz', moduleId: module.id, run, updatedAt: Date.now() });
}, [module, run, setDraft]);
```

В `MicroscopyPage.tsx` инициализировать `tasks` и `run` из draft:

```tsx
const saved = progress.draft?.kind === 'micro' ? progress.draft : null;
const [tasks, setTasks] = useState(() => saved ? saved.taskKeys.map((key) => allMicroTasks.find((task) => task.key === key)).filter((task): task is typeof allMicroTasks[number] => Boolean(task)) : allMicroTasks);
const [run, setRun] = useState<RunState<string> | null>(() => saved?.run ?? null);
```

В `start` сохранять полный snapshot:

```tsx
setDraft({ kind: 'micro', taskKeys: ordered.map((task) => task.key), run: createRun(ordered.map((task) => task.key)), updatedAt: Date.now() });
```

Добавить effect для каждого изменения run:

```tsx
useEffect(() => {
  if (run && !run.done) setDraft({ kind: 'micro', taskKeys: tasks.map((task) => task.key), run, updatedAt: Date.now() });
}, [run, tasks, setDraft]);
```

- [ ] **Step 4: Восстанавливать complex**

В `ComplexCasesPage.tsx` при монтировании читать draft и восстанавливать все поля одним effect:

```tsx
useEffect(() => {
  const draft = progress.draft;
  if (draft?.kind !== 'complex') return;
  const restored = draft.caseIds.map((id) => content.complexCases.find((item) => item.id === id)).filter((item): item is ComplexCase => Boolean(item));
  if (!restored.length) return;
  setCases(restored); setIndex(draft.index); setValues(draft.values); setChecked(draft.checked); setWrong(draft.wrong); setFirst(draft.first); setRepeat(draft.repeat);
}, []);
```

Сохранять snapshot после изменений:

```tsx
useEffect(() => {
  if (!done) setDraft({ kind: 'complex', caseIds: cases.map((item) => item.id), index, values, checked, wrong, first, repeat, updatedAt: Date.now() });
}, [done, cases, index, values, checked, wrong, first, repeat, setDraft]);
```

Добавить `useEffect` в React import.

- [ ] **Step 5: Реализовать статистику и раздел ошибок**

`web/src/features/progress/ProgressPage.tsx`:

```tsx
import { content } from '../../data/content';
import { useProgress } from '../../storage/ProgressContext';

export function ProgressPage() {
  const { progress, notice } = useProgress();
  const accuracy = progress.stats.answers ? Math.round(progress.stats.correct / progress.stats.answers * 100) : 0;
  return <section><p className="eyebrow">Локально на этом устройстве</p><h1>Мой прогресс</h1>{notice && <p role="alert">{notice}</p>}<div className="stats-grid"><div><b>{progress.stats.completed}</b><span>завершено</span></div><div><b>{progress.stats.answers}</b><span>ответов</span></div><div><b>{accuracy}%</b><span>точность</span></div></div><h2>Результаты по подмодулям</h2>{content.modules.filter((module) => !module.locked).map((module) => <p key={module.id}>{module.id} · {module.title}: <b>{progress.results[module.id] ?? '—'}{progress.results[module.id] !== undefined ? '%' : ''}</b></p>)}</section>;
}
```

`web/src/features/progress/ProgressPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { ProgressProvider } from '../../storage/ProgressContext';
import { ProgressPage } from './ProgressPage';

test('показывает нулевую статистику без ошибок', () => {
  render(<ProgressProvider><ProgressPage /></ProgressProvider>);
  expect(screen.getByRole('heading', { name: 'Мой прогресс' })).toBeInTheDocument();
  expect(screen.getByText('0%')).toBeInTheDocument();
});
```

`web/src/features/mistakes/MistakesPage.tsx`:

```tsx
import { Link } from 'react-router-dom';
import { allMicroTasks, content } from '../../data/content';
import { useProgress } from '../../storage/ProgressContext';

export function MistakesPage() {
  const { progress } = useProgress();
  const micro = allMicroTasks.filter((task) => progress.microWrong.includes(task.key));
  const complex = content.complexCases.filter((item) => progress.complexWrong.includes(item.id));
  return <section><p className="eyebrow">Повторение</p><h1>Мои ошибки</h1>{!micro.length && !complex.length && <p>Сохранённых ошибок пока нет.</p>}{micro.length > 0 && <><h2>Микропрепараты · {micro.length}</h2><Link className="primary" to="/micro">Повторить в практикуме</Link></>}{complex.length > 0 && <><h2>Комплексные описания · {complex.length}</h2><Link className="primary" to="/complex">Повторить описания</Link></>}</section>;
}
```

В `CourseHome.tsx` внутри `Link` после `small` добавить результат:

```tsx
{progress.results[module.id] !== undefined && <span className="score-pill">{progress.results[module.id]}%</span>}
```

Добавить `const { progress } = useProgress();` и соответствующий import.

- [ ] **Step 6: Добавить Error Boundary и routes**

`web/src/app/AppErrorBoundary.tsx`:

```tsx
import { Component, type ErrorInfo, type ReactNode } from 'react';

interface State { failed: boolean }

export class AppErrorBoundary extends Component<{ children: ReactNode }, State> {
  state: State = { failed: false };
  static getDerivedStateFromError(): State { return { failed: true }; }
  componentDidCatch(error: Error, info: ErrorInfo) { console.error(error, info); }
  render() {
    if (this.state.failed) return <main className="paper-card"><h1>Не удалось открыть раздел</h1><p>Старые данные прогресса не удалены.</p><button className="primary" onClick={() => { window.location.hash = '#/'; this.setState({ failed: false }); }}>Вернуться на главную</button></main>;
    return this.props.children;
  }
}
```

В `main.tsx` обернуть `<App />` в `<AppErrorBoundary>`. В `App.tsx` заменить placeholders:

```tsx
<Route path="mistakes" element={<MistakesPage />} />
<Route path="progress" element={<ProgressPage />} />
```

Добавить imports обоих компонентов.

- [ ] **Step 7: Добавить стили и выполнить проверки**

Append to `web/src/styles/global.css`:

```css
.stats-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; margin: 24px 0; }
.stats-grid > div { padding: 20px; border: 1px solid var(--line); border-radius: 14px; background: var(--surface); }
.stats-grid b, .stats-grid span { display: block; }
.stats-grid b { font: 36px Georgia, serif; color: var(--accent); }
.stats-grid span { color: var(--muted); }
.score-pill { padding: 5px 9px; border-radius: 20px; background: var(--accent-soft); color: var(--accent); font-weight: 700; }
@media (max-width: 480px) { .stats-grid { gap: 7px; } .stats-grid > div { padding: 12px 8px; } .stats-grid b { font-size: 28px; } }
```

Run: `cd web && npm test -- src/storage src/features/progress && npm run build`  
Expected: storage/draft tests PASS; progress test PASS; build PASS.

- [ ] **Step 8: Commit**

```bash
git add web/src
git commit -m "feat: restore sessions and add progress resilience"
```

---

### Task 11: Сквозные тесты, корневой preview и финальная проверка

**Files:**
- Create: `web/playwright.config.ts`, `web/e2e/trainer.spec.ts`.
- Create: root `package.json` с командами-делегатами в `web/`.
- Modify: `README.md` только описанием запуска новой ветки без публикации.
- Test: все unit/integration/e2e и production build.

**Interfaces:**
- Produces: `npm run dev`, `npm run build`, `npm test` из корня; проверенный `web/dist/`.

- [ ] **Step 1: Написать e2e-сценарии**

`web/playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  use: { baseURL: 'http://127.0.0.1:4173', trace: 'retain-on-failure' },
  projects: [
    { name: 'desktop', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['Pixel 5'] } },
  ],
  webServer: {
    command: 'npm run build && npm run dev -- --host 127.0.0.1 --port 4173',
    url: 'http://127.0.0.1:4173',
    reuseExistingServer: !process.env.CI,
  },
});
```

`web/e2e/trainer.spec.ts`:

```ts
import { test, expect } from '@playwright/test';

test('модуль → теория → тест', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('link', { name: /Морфологические группы бактерий/ }).click();
  await expect(page.getByText(/Теория 1 из 6/)).toBeVisible();
  await page.getByRole('button', { name: 'Дальше' }).click();
  await expect(page.getByText(/Теория 2 из 6/)).toBeVisible();
  await page.goto('/#/module/1.1/quiz');
  await page.locator('.answer-option').first().click();
  await expect(page.getByRole('status')).toBeVisible();
});

test('практикум запускается и не имеет горизонтального overflow', async ({ page }) => {
  await page.goto('/#/micro');
  await page.getByRole('button', { name: 'Начать все задания' }).click();
  await expect(page.getByText('Задание 1 из 24')).toBeVisible();
  const overflow = await page.evaluate(() => document.documentElement.scrollWidth > document.documentElement.clientWidth);
  expect(overflow).toBe(false);
});

test('пустые ошибки и статистика доступны', async ({ page }) => {
  await page.goto('/#/mistakes');
  await expect(page.getByText('Сохранённых ошибок пока нет.')).toBeVisible();
  await page.goto('/#/progress');
  await expect(page.getByRole('heading', { name: 'Мой прогресс' })).toBeVisible();
});
```

- [ ] **Step 2: Добавить корневые команды**

Root `package.json`:

```json
{
  "name": "microbiology-trainer-workspace",
  "private": true,
  "scripts": {
    "dev": "npm --prefix web run dev --",
    "build": "npm --prefix web run build",
    "test": "npm --prefix web run test",
    "test:e2e": "npm --prefix web run test:e2e"
  }
}
```

Run from repository root: `npm run dev -- --host 127.0.0.1 --port 7100`  
Expected: Vite запускается на указанном порту; после проверки процесс остановлен.

- [ ] **Step 3: Обновить README без изменения публикации**

Добавить в начало `README.md`:

```markdown
## Новая версия в ветке rebuild-from-scratch

Исходники: `web/`. Локальный запуск из корня: `npm run dev`. Тесты: `npm test`; сквозные тесты: `npm run test:e2e`; сборка: `npm run build` → `web/dist/`.

Старая версия и настройки публикации в этой ветке не изменяются автоматически.
```

- [ ] **Step 4: Установить браузер и выполнить полный набор проверок**

```bash
cd web
npx playwright install chromium
cd ..
npm test
npm run test:e2e
npm run build
```

Expected: все unit/integration tests PASS; desktop/mobile e2e PASS; build PASS. Проверить `test-results/` и `playwright-report/` при любой ошибке, исправить причину и повторить весь набор.

- [ ] **Step 5: Проверить границы изменений и чистоту ветки**

```bash
git diff main --stat
git diff --check
git status --short --branch
```

Expected: новые изменения ограничены `web/`, root `package.json`, `README.md`, `.gitignore`, `docs/`; `microbiology_trainer_v2_4.html` и `microbiology_trainer_v2_6.html` не изменены. Extractor использует `git show HEAD:index.html`, поэтому любые внешние незакоммиченные изменения рабочего `index.html` сохраняются, не попадают в JSON и не staged. После commit статус чистый, кроме заранее существовавших внешних изменений; `.superpowers/` игнорируется.

- [ ] **Step 6: Commit**

```bash
git add package.json README.md web
git commit -m "test: verify complete rebuilt trainer"
```

- [ ] **Step 7: Финальная ручная приёмка**

Запустить `npm run dev -- --host 0.0.0.0 --port 7100`, проверить в браузере полный путь одного подмодуля, практикум, комплексный случай, повтор ошибок, reload незавершённого занятия, клавиатурную навигацию и мобильную ширину. Остановить сервер перед ответом. Не выполнять `git push`, не публиковать и не удалять legacy-файлы.
