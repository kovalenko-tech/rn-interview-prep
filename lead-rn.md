# Lead React Native — подготовка к интервью

> Кирилл, 13 лет мобильной разработки — базу можно пропустить.
> Акцент на **глубину**, **архитектуру** и **лидерство**.

---

## 1. React Native — internals

### Вопросы
- Как работает **JS Bridge** (old arch) vs **JSI** (new arch)?
- Что такое **Hermes** и зачем он нужен?
- Как работает **Fabric** (новый renderer) и **TurboModules**?
- Чем отличается `useNativeDriver: true` от JS-анимации?
- Как RN обновляет UI — reconciliation, fiber, commit phase?
- Что такое **Metro bundler** — как настроить кастомный resolver?

### Что знать
```
Old arch: JS Thread → Bridge (serialization JSON) → Native Thread
New arch: JS Thread → JSI (sync, no JSON) → C++ → Native
Hermes: AOT bytecode компиляция → быстрый старт, меньше памяти
Fabric: concurrent rendering, синхронный доступ к shadow tree
TurboModules: lazy load native modules через JSI
```

---

## 2. JavaScript / TypeScript

### Вопросы
- Event loop, microtasks vs macrotasks — порядок выполнения?
- Как работает `Promise.all` vs `Promise.allSettled` vs `Promise.race`?
- Объясни замыкания и почему они проблема в `useEffect`
- Что такое `WeakMap` / `WeakRef` — где применять?
- Разница `type` vs `interface` в TS, когда что?
- Как типизировать HOC, render props, conditional types?
- `infer` в TypeScript — примеры

### Типичные задачи
```typescript
// Deeppartial
type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] }

// Exclude null/undefined
type NonNullableProps<T> = { [K in keyof T]-?: NonNullable<T[K]> }
```

---

## 3. Архитектура мобильного приложения

### Вопросы
- Как бы ты организовал структуру большого RN проекта (50+ экранов)?
- Feature-based vs layer-based — плюсы/минусы?
- Как реализовать **dependency injection** в RN без фреймворка?
- Monorepo (Nx / Turborepo) для RN + shared packages — опыт?
- Как разделить бизнес-логику от UI чтобы тестировать без рендеринга?

### Паттерны
```
Clean Architecture: Entity → UseCase → Repository (interface) → Data Source
Feature Sliced Design: shared / entities / features / widgets / pages
State: Zustand (малый), Redux Toolkit (средний), XState (сложные флоу)
```

---

## 4. Производительность

### Вопросы
- Как профилировать RN приложение? Какие инструменты?
- `FlatList` vs `FlashList` — в чём разница внутри?
- Когда использовать `useMemo`, `useCallback`, `React.memo`? Когда они вредят?
- Как избежать лишних re-renders в сложном дереве компонентов?
- Что такое **JS thread jank** — как диагностировать?
- Как оптимизировать cold start time?
- Image caching стратегии в RN

### Метрики
```
Цели для production:
- Cold start < 2s
- TTI (time to interactive) < 3s
- Frame drop < 1% (60fps → max 6 dropped/600 frames)
- JS bundle size: отслеживать через @bundle-analyzer
```

---

## 5. State Management

### Вопросы
- Когда Redux, когда Zustand, когда Context?
- Как организовать **optimistic updates**?
- Как решать race conditions в async actions?
- Нормализация данных — zod + normalizr / custom?
- Как синхронизировать серверный стейт с клиентским? (React Query / SWR)

### Пример правильного ответа
```
Серверный стейт (API данные) → React Query / TanStack Query
Клиентский стейт (UI, forms) → Zustand или useState
Глобальный синхронный стейт → Zustand
Сложные машины состояний → XState
```

---

## 6. Native modules / интеграция

### Вопросы
- Когда писать кастомный Native Module vs использовать библиотеку?
- Как сделать Native Module для iOS (Swift) и Android (Kotlin)?
- Что такое **New Architecture**-совместимый модуль (codegen, spec)?
- Как отлаживать нативный crash из RN?
- Bridgeless mode — что это?

---

## 7. Навигация

### Вопросы
- React Navigation vs Expo Router — разница в архитектуре?
- Как организовать **deep linking** + universal links?
- Authentication flow — как скрыть protected routes до загрузки сессии?
- Performance: почему нативный стек быстрее JS-навигации?

---

## 8. Тестирование

### Вопросы
- Пирамида тестирования для мобилок — как выглядит у тебя?
- `@testing-library/react-native` — философия, что тестировать?
- Mocking native modules в Jest
- E2E: Detox vs Maestro — когда что?
- Snapshot тесты — польза или вред?

### Стратегия
```
Unit: бизнес-логика, utils, hooks (без render)
Integration: компоненты с navigation, store
E2E: критичные user flows (login, purchase, onboarding)
Соотношение: 70% unit / 20% integration / 10% E2E
```

---

## 9. CI/CD и деплой

### Вопросы
- Как организован release process у тебя?
- Fastlane, Bitrise, GitHub Actions, EAS Build — что использовал?
- **CodePush / OTA updates** — как безопасно катить?
- Signing: Keystore management, certificates rotation
- Staged rollout — как реализовать для мобилок?

---

## 10. Безопасность

### Вопросы
- Как хранить sensitive данные на устройстве? (Keychain / Keystore)
- Certificate pinning — как реализовать, зачем?
- Obfuscation JS bundle — реально ли это защищает?
- Jailbreak / root detection — подходы?
- OWASP Mobile Top 10 — назови топ-3 для твоих приложений

---

## 11. Lead-специфика — самое важное

### Техническое лидерство
- Как ты проводишь code review? Что ищешь в первую очередь?
- Как вводишь новые технологии в команду? Процесс?
- Как принимаешь архитектурные решения — ADR (Architecture Decision Record)?
- Как балансируешь tech debt vs фичи?

### Людские вопросы
- Как онбордишь нового разработчика?
- Что делаешь если джун постоянно ломает продакшн?
- Конфликт в команде по техническому решению — как разрешаешь?
- Как мотивируешь команду работать над скучными задачами?

### Процессы
- Как оцениваешь задачи? (Planning poker, T-shirt sizing?)
- Как работаешь с PM когда дедлайн нереалистичный?
- Retrospective — как проводишь?

---

## 12. System Design (для senior+/lead)

### Типичные задачи
- Спроектируй offline-first приложение (синхронизация, conflict resolution)
- Спроектируй real-time чат на мобилке (WebSocket, reconnect, pagination)
- Спроектируй систему push-уведомлений (FCM/APNs, deep link, tracking)
- Как бы ты мигрировал legacy RN 0.63 приложение на new architecture?

### Фреймворк для ответа
```
1. Уточни требования (MAU, offline?, real-time?)
2. Опиши компоненты (Navigation, State, Network, Storage)
3. Trade-offs (почему это решение, а не то)
4. Scaling concerns
5. Testing strategy
```

---

## 13. Вопросы которые задать им

```
- Какой стек сейчас? New arch или старая?
- Какой размер команды, сколько RN разработчиков?
- Как выглядит release cycle?
- Есть ли E2E тесты? Какое покрытие?
- Какие главные технические проблемы сейчас?
- Буду ли я влиять на найм команды?
- Как выглядит онбординг нового разработчика?
```

---

## Быстрая самопроверка

| Тема | Уровень (1-5) | Подготовить |
|------|--------------|-------------|
| JS/TS advanced | | |
| RN new architecture | | |
| Performance profiling | | |
| State management | | |
| Native modules | | |
| Testing strategy | | |
| System design | | |
| Team leadership | | |
| CI/CD | | |

---

*Главный совет: Lead интервью — это не только "знаешь ли ты", но и "как ты думаешь". Объясняй trade-offs, не давай однозначные ответы там где их нет.*
