# React Native Senior — полный гайд подготовки 2025

> Основан на реальных вопросах с собеседований, Reddit-тредах и топ-100 списках.
> Каждый вопрос — с подробным объяснением **почему именно так**.

---

## ⚡ ТОП-10 вопросов которые задают ВСЕГДА

1. Как работает архитектура React Native (Bridge vs New Arch)?
2. Чем отличается `useMemo` от `useCallback` и когда они вредят?
3. Как избежать лишних ре-рендеров?
4. В чём разница FlatList и FlashList?
5. Как работает `useEffect` — когда вызывается, когда cleanup?
6. Как хранить токены безопасно?
7. Что такое Hermes и зачем он нужен?
8. Как организован навигационный стек для авторизации?
9. Как написать кастомный хук?
10. Как дебажишь RN приложение?

---

## БЛОК 1. Архитектура React Native

### Q1: Объясни как работает React Native под капотом

**Старая архитектура — Bridge:**
```
JS Thread  ──→  Bridge (JSON serialization)  ──→  Native Thread
                     ↕ асинхронно
Native Thread ──→  Bridge (JSON serialization) ──→  JS Thread
```

Проблемы Bridge:
- Всё общение через JSON → сериализация/десериализация на каждый вызов
- Асинхронный — нельзя синхронно получить layout из JS
- Узкое место при большом количестве событий (scroll, анимации)

**Новая архитектура — JSI (JavaScript Interface):**
```
JS Thread ──→ JSI (C++ прослойка) ──→ Native напрямую, синхронно
```

Что входит в новую архитектуру:
- **JSI** — C++ API, даёт JS прямой доступ к native объектам без JSON
- **TurboModules** — нативные модули через JSI, загружаются лениво
- **Fabric** — новый рендерер, работает синхронно, поддерживает Concurrent React
- **Codegen** — генерирует типизированный C++ код из TypeScript/Flow спецификаций

**Почему это важно на интервью:** работодатели проверяют понимаешь ли ты реальные trade-offs, а не просто выучил определение.

---

### Q2: Что такое Hermes и почему Meta его создала?

**Проблема:** V8/JSC выполняют JS через JIT (Just-In-Time) компиляцию. На мобильных устройствах JIT-компиляция при запуске медленная и потребляет много памяти.

**Решение — Hermes:**
- AOT (Ahead-Of-Time) компиляция: JS → bytecode **при сборке**, не при запуске
- На устройстве уже готовый bytecode → парсинг не нужен → быстрый старт
- Оптимизированный Garbage Collector для мобильных (меньше пауз)
- Меньше потребление RAM

**Реальные результаты (Meta):** TTI (Time To Interactive) сократился на 30-40% на низкобюджетных Android-устройствах.

**Минус:** Hermes без JIT → "горячий" код не оптимизируется как в V8. Но для большинства мобильных приложений это неощутимо.

---

### Q3: Что такое Fabric?

Fabric — новый рендерер React Native, замена старого UIManager.

**Стар рендерер:**
```
React (JS) → Shadow Tree (JS) → Bridge → Shadow Thread (C++) → UI Thread
```
3 копии данных, всё асинхронно.

**Fabric:**
```
React (JS) → C++ Shadow Tree → синхронно → UI Thread
```

Преимущества:
- Concurrent режим React 18 (Suspense, transitions, startTransition)
- Синхронный layout — можно измерить размер view в том же кадре
- Меньше копий данных — меньше памяти
- Приоритезация рендеринга (urgent vs non-urgent updates)

---

### Q4: Что такое TurboModules?

TurboModules — замена старых Native Modules.

**Старые Native Modules:**
- Все инициализируются при старте приложения — замедляет startup
- Общение через Bridge (асинхронно, JSON)

**TurboModules:**
- Lazy initialization — модуль загружается только когда первый раз вызван
- Общение через JSI (синхронно, без JSON)
- Codegen генерирует типобезопасный интерфейс из TypeScript спецификации

```typescript
// Спецификация (NativeCalculator.ts)
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): number; // синхронный вызов!
}

export default TurboModuleRegistry.getEnforcing<Spec>('Calculator');
```

---

## БЛОК 2. JavaScript и React Hooks

### Q5: Объясни Event Loop — как работает в контексте RN?

```
Call Stack → пустой?
    → Microtask Queue (Promise.then, queueMicrotask)
        → пустой?
            → Macrotask Queue (setTimeout, setInterval, fetch callbacks)
```

**Порядок выполнения:**
```javascript
console.log('1');

setTimeout(() => console.log('4 (macrotask)'), 0);

Promise.resolve()
  .then(() => console.log('2 (microtask)'))
  .then(() => console.log('3 (microtask)'));

// Output: 1, 2, 3, 4
```

**Почему важно в RN:** JS Thread — один поток. Если ты заблокировал его тяжёлым синхронным кодом — нет frame updates — jank. `setTimeout(fn, 0)` — способ разбить тяжёлую задачу на части.

---

### Q6: Как работает useEffect? Все случаи.

```javascript
// 1. Без deps — каждый рендер
useEffect(() => { ... });

// 2. Пустой массив — один раз (componentDidMount)
useEffect(() => { ... }, []);

// 3. С deps — при изменении deps
useEffect(() => { ... }, [userId, filters]);

// 4. С cleanup (componentWillUnmount или перед следующим вызовом)
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe(); // cleanup
}, []);
```

**Порядок выполнения:**
1. React рендерит (вычисляет Virtual DOM)
2. Обновляет DOM/Native Views
3. **Потом** запускает useEffect (после paint)

**useLayoutEffect** — запускается до paint (синхронно после DOM mutations). Нужен когда надо измерить layout перед тем как пользователь увидит.

**Классическая ловушка — closure:**
```javascript
// ❌ проблема: count "застрял" в closure
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1); // всегда будет 0 + 1 = 1
  }, 1000);
  return () => clearInterval(timer);
}, []); // count не в deps

// ✅ fix: функциональное обновление
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1); // всегда актуальный prev
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

---

### Q7: useMemo vs useCallback — разница и когда каждый

```javascript
// useMemo — кэширует ЗНАЧЕНИЕ
const sortedList = useMemo(
  () => heavySort(items),
  [items]
); // пересчитывается только если items изменился

// useCallback — кэширует ФУНКЦИЮ (== useMemo(() => fn, deps))
const handlePress = useCallback(
  () => navigate(itemId),
  [itemId]
); // новая функция только если itemId изменился
```

**Когда нужны:**
- `useMemo`: тяжёлые вычисления (sort/filter большого массива, сложные трансформации)
- `useCallback`: передаёшь callback в дочерний компонент обёрнутый в `React.memo`

**Когда ВРЕДЯТ (антипаттерн):**
```javascript
// ❌ overhead больше чем benefit — простое умножение
const doubled = useMemo(() => count * 2, [count]);

// ❌ бессмысленно — компонент не в React.memo
<Button onPress={useCallback(() => doSomething(), [])} />

// ❌ новые объекты в deps каждый рендер — никогда не кэшируется
const result = useMemo(() => compute(data), [{ ...data }]); // новый объект каждый раз
```

**Правило:** профилируй сначала, оптимизируй потом. Преждевременная мемоизация = читаемость↓, производительность иногда даже хуже.

---

### Q8: Как избежать лишних ре-рендеров?

**5 инструментов:**

```javascript
// 1. React.memo — не ре-рендерит если props не изменились
const Card = React.memo(({ title, onPress }) => (
  <TouchableOpacity onPress={onPress}>
    <Text>{title}</Text>
  </TouchableOpacity>
));

// 2. useCallback — стабильный callback для React.memo компонентов
const handlePress = useCallback(() => {
  navigation.navigate('Details', { id });
}, [id]);

// 3. Стабильные ссылки — не создавай объекты/массивы inline
// ❌ новый объект каждый рендер родителя
<Card style={{ padding: 10 }} />

// ✅ стабильный объект
const cardStyle = { padding: 10 }; // вне компонента
<Card style={cardStyle} />

// 4. Context — разбивай на мелкие контексты
// ❌ один большой контекст = все подписчики ре-рендерятся
const AppContext = createContext({ user, theme, settings, ... });

// ✅ отдельные контексты
const UserContext = createContext(user);
const ThemeContext = createContext(theme);

// 5. State вниз — держи state как можно ближе к потребителю
// ❌ state в App → ре-рендерит всё дерево
// ✅ state в компоненте который его использует
```

**Как диагностировать:** React DevTools Profiler → "Highlight updates when components render" → видишь что мигает.

---

### Q9: Напиши кастомный хук — useFetch

```typescript
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    const controller = new AbortController();

    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(json => {
        if (!cancelled) {
          setData(json);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled && err.name !== 'AbortError') {
          setError(err);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [url]);

  return { data, loading, error };
}
```

---

### Q10: Напиши useDebounce хук

```typescript
function useDebounce<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebounced(value);
    }, delayMs);

    return () => clearTimeout(timer); // cleanup при каждом изменении value
  }, [value, delayMs]);

  return debounced;
}

// Использование:
const [query, setQuery] = useState('');
const debouncedQuery = useDebounce(query, 500);

useEffect(() => {
  if (debouncedQuery) searchAPI(debouncedQuery);
}, [debouncedQuery]);
```

---

## БЛОК 3. Производительность

### Q11: FlatList vs FlashList — в чём реальная разница?

**FlatList (встроенный):**
- При скролле: JS вычисляет что видно → рендерит → Native показывает
- При быстром скролле: JS Thread не успевает → белые блоки
- Каждый item создаётся и уничтожается при уходе с экрана

**FlashList (Shopify, open source):**
- RecycledItemPool: item не уничтожается, а переиспользуется с новыми данными
- Определяет размеры заранее через `estimatedItemSize` → меньше layout recalculations
- 10x меньше JS работы при быстром скролле (benchmarks Shopify)

```javascript
// FlatList
<FlatList
  data={items}
  keyExtractor={item => item.id}
  renderItem={({ item }) => <Card item={item} />}
/>

// FlashList
<FlashList
  data={items}
  keyExtractor={item => item.id}
  renderItem={({ item }) => <Card item={item} />}
  estimatedItemSize={80} // обязательно! высота среднего item
/>
```

**Когда FlashList обязателен:**
- Более 100 items
- Сложные карточки (изображения, много текста)
- Infinite scroll
- Пользователи на низкобюджетных Android устройствах

---

### Q12: Как профилировать RN приложение?

**Шаг 1 — React DevTools Profiler:**
- Какой компонент рендерится и сколько раз
- Сколько времени занимает каждый рендер
- Какой prop/state вызвал ре-рендер

**Шаг 2 — Flipper (Performance плагин):**
- JS Thread FPS и UI Thread FPS отдельно
- Если UI FPS нормальный, JS FPS плохой → проблема в JS
- Если оба плохие → нативная проблема

**Шаг 3 — Platform-specific:**
```
iOS:   Xcode Instruments → Time Profiler, Allocations, Leaks
Android: Android Studio Profiler → CPU, Memory, Network
```

**Как читать результат:**
```
Frame budget = 16ms (60fps) или 8ms (120fps)
JS frame > 16ms → jank
JS frame долго = что-то тяжёлое в рендере или бизнес-логике
```

**Типичные находки:**
- `console.log` в продакшне (убирает ~20% производительности)
- `JSON.parse(JSON.stringify(obj))` для deep copy в hot path
- `useSelector` подписка на большой объект целиком вместо нужного поля

---

### Q13: Почему нельзя делать тяжёлые операции синхронно в рендере?

```javascript
// ❌ тяжёлая операция в рендере — блокирует JS Thread
function ProductList({ items }) {
  // filterAndSort пересчитывается при каждом ре-рендере
  const sorted = items.filter(i => i.active).sort(...);
  return <FlatList data={sorted} ... />;
}

// ✅ мемоизация — пересчёт только при изменении items
function ProductList({ items }) {
  const sorted = useMemo(
    () => items.filter(i => i.active).sort(...),
    [items]
  );
  return <FlatList data={sorted} ... />;
}

// ✅✅ для очень тяжёлых операций — JSI/Worklets (Reanimated)
// или разбивка через InteractionManager
InteractionManager.runAfterInteractions(() => {
  // выполнится после завершения анимаций/жестов
  setData(heavyComputation(rawData));
});
```

---

## БЛОК 4. Анимации

### Q14: Animated API vs Reanimated — когда что?

**Animated API (встроенный):**
```javascript
const opacity = useRef(new Animated.Value(0)).current;

Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // обязательно для opacity/transform
}).start();
```

Ограничения `useNativeDriver: true`:
- Только `transform` и `opacity`
- Нельзя анимировать `width`, `height`, `backgroundColor`, `padding`
- После запуска нельзя менять конфигурацию

**Reanimated 3 (библиотека):**
```javascript
import { useSharedValue, withTiming, useAnimatedStyle } from 'react-native-reanimated';

function Card() {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Animated.View
      style={animatedStyle}
      onTouchStart={() => { scale.value = withTiming(0.95, { duration: 100 }); }}
      onTouchEnd={() => { scale.value = withTiming(1, { duration: 100 }); }}
    />
  );
}
```

Преимущества Reanimated:
- Worklets выполняются в UI Thread — JS может быть заблокирован, анимация идёт
- Почти все CSS свойства доступны
- Нативная интеграция с Gesture Handler
- `useAnimatedScrollHandler`, `useDerivedValue`, `useAnimatedReaction`

**Когда что:**
```
Простой fade/slide без жестов → Animated API с useNativeDriver: true
Интерактивные анимации (swipe, pinch, drag) → Reanimated 3 + Gesture Handler
Shared Element Transitions → React Navigation Shared Element или Reanimated
```

---

## БЛОК 5. State Management

### Q15: Когда Redux, когда Zustand, когда Context?

```
Context API:
  ✅ Редко меняющиеся данные: тема, локаль, user profile
  ✅ Простые приложения
  ❌ Часто обновляемые данные → все подписчики ре-рендерятся

Zustand:
  ✅ Маленькие и средние приложения
  ✅ Минимальный boilerplate
  ✅ Встроенная оптимизация через selectors
  ✅ Просто в тестировании
  ❌ Нет встроенного DevTools time-travel (есть плагин)

Redux Toolkit:
  ✅ Большие приложения, большие команды
  ✅ Строгая структура → предсказуемость
  ✅ Excellent DevTools (time-travel, diff)
  ✅ RTK Query для server state
  ❌ Больше кода

TanStack Query (React Query):
  ✅ Server state (API данные, кэш, инвалидация)
  ✅ Автоматический refetch, staleTime, cacheTime
  ✅ Optimistic updates из коробки
  ❌ Не для клиентского UI state
```

**Моя формула:**
```
Server state → TanStack Query
Client/UI state → Zustand (или useState для локального)
Глобальный редко меняющийся state → Context
Сложные state machines → XState
```

---

### Q16: Как работает Zustand?

```typescript
import { create } from 'zustand';

interface AuthStore {
  user: User | null;
  token: string | null;
  login: (user: User, token: string) => void;
  logout: () => void;
}

const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  token: null,
  login: (user, token) => set({ user, token }),
  logout: () => set({ user: null, token: null }),
}));

// В компоненте — подписка только на нужное поле
const user = useAuthStore(state => state.user); // ре-рендер только при изменении user
const logout = useAuthStore(state => state.logout); // функция стабильна
```

Персистентность:
```typescript
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

const useStore = create(persist(
  (set) => ({ ... }),
  {
    name: 'auth-storage',
    storage: createJSONStorage(() => AsyncStorage),
  }
));
```

---

## БЛОК 6. Навигация

### Q17: Как правильно сделать Auth flow?

**Правило: условный рендер, не redirect.**

```javascript
// ✅ правильно
function RootNavigator() {
  const { user, isLoading } = useAuthStore();

  if (isLoading) return <SplashScreen />;

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {user ? (
        // Авторизованный стек
        <>
          <Stack.Screen name="Home" component={HomeScreen} />
          <Stack.Screen name="Profile" component={ProfileScreen} />
        </>
      ) : (
        // Гостевой стек
        <>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}
```

**Почему не `navigation.navigate('Login')` на logout:**
React Navigation автоматически анимирует переход между стеками и очищает history. Нет риска что пользователь нажмёт "back" и вернётся на защищённый экран.

---

### Q18: Как передавать параметры между экранами типобезопасно?

```typescript
// navigation/types.ts
export type RootStackParamList = {
  Home: undefined;
  UserProfile: { userId: string; username: string };
  Settings: { section?: 'notifications' | 'privacy' };
};

// В компоненте
type ProfileScreenProps = NativeStackScreenProps<RootStackParamList, 'UserProfile'>;

function UserProfileScreen({ route, navigation }: ProfileScreenProps) {
  const { userId, username } = route.params; // автодополнение, типы
  
  const goToSettings = () => {
    navigation.navigate('Settings', { section: 'privacy' });
  };
}

// Хук
const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
```

---

## БЛОК 7. Native Modules

### Q19: Когда писать кастомный Native Module?

Когда писать:
- Нужна функциональность без готовой библиотеки (кастомный SDK от вендора)
- Существующая библиотека не поддерживает новую архитектуру
- Критична производительность (тяжёлые вычисления на нативном языке)
- Нужен background processing

Когда не писать:
- Есть хорошая библиотека (react-native-camera, react-native-keychain)
- Можно решить через fetch/WebSocket
- Бюджет на поддержку iOS + Android кода ограничен

**Минимальный TurboModule (Swift + Kotlin):**
```typescript
// NativeLogger.ts (спецификация)
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  logEvent(name: string, params: Object): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('Logger');
```

---

## БЛОК 8. TypeScript

### Q20: Напиши сложный тип — DeepPartial

```typescript
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

// Использование
type Config = {
  api: { url: string; timeout: number };
  theme: { primary: string; dark: boolean };
};

type PartialConfig = DeepPartial<Config>;
// { api?: { url?: string; timeout?: number }; theme?: { ... } }
```

### Q21: Conditional types и infer

```typescript
// Получить тип элемента массива
type ArrayElement<T> = T extends (infer Item)[] ? Item : never;

type StringArrayElement = ArrayElement<string[]>; // string

// Получить возвращаемый тип Promise
type Unpromise<T> = T extends Promise<infer R> ? R : T;

type ApiResult = Unpromise<Promise<{ users: User[] }>>; // { users: User[] }

// Discriminated union
type Result<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function handleResult<T>(result: Result<T>) {
  if (result.status === 'success') {
    console.log(result.data); // T — TypeScript знает
  } else {
    console.log(result.error); // string
  }
}
```

---

## БЛОК 9. Тестирование

### Q22: Что тестировать и как?

**Пирамида тестирования для RN:**
```
        E2E (Detox/Maestro)     ← 10%, только критичные флоу
      Integration (RTL)          ← 20%, экраны с логикой
    Unit (Jest)                   ← 70%, хуки, утилиты, стор
```

**Unit тест кастомного хука:**
```javascript
import { renderHook, act } from '@testing-library/react-hooks';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('increments counter', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });
});
```

**Integration тест компонента:**
```javascript
import { render, fireEvent, waitFor } from '@testing-library/react-native';

describe('LoginScreen', () => {
  it('shows error on invalid credentials', async () => {
    mockLoginApi.mockRejectedValueOnce(new Error('Invalid credentials'));

    const { getByTestId, getByText } = render(<LoginScreen />);

    fireEvent.changeText(getByTestId('email-input'), 'bad@email.com');
    fireEvent.changeText(getByTestId('password-input'), 'wrong');
    fireEvent.press(getByTestId('login-button'));

    await waitFor(() => {
      expect(getByText('Invalid credentials')).toBeTruthy();
    });
  });
});
```

---

## БЛОК 10. Безопасность

### Q23: Как безопасно хранить данные на устройстве?

```
Уровни безопасности:
❌ AsyncStorage — открытый текст, виден через adb backup
⚠️  MMKV — быстрый, шифрование есть но настройка сложнее
✅ react-native-keychain — iOS Keychain / Android Keystore
✅ expo-secure-store — то же самое через Expo API
```

```javascript
import * as Keychain from 'react-native-keychain';

// Сохранить
await Keychain.setGenericPassword(
  'access_token',
  token,
  {
    accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED,
    // Android: требует биометрию или PIN для доступа
    securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE,
  }
);

// Получить
const credentials = await Keychain.getGenericPassword();
const token = credentials?.password;

// Удалить
await Keychain.resetGenericPassword();
```

**Дополнительно:**
- Обфускация JS bundle через `@react-native/metro-babel-transformer` (не надёжна, но усложняет реверс)
- Jailbreak/Root detection: `react-native-jail-monkey`
- Certificate pinning: `react-native-ssl-pinning`

---

## БЛОК 11. CI/CD и деплой

### Q24: Как организован release process в продакшн проекте?

```
develop ─────────── staging build (EAS/Fastlane) ──→ QA
    ↓ release branch
main ─────────────── production build ──→ App Store / Play Store
```

**EAS Build (рекомендую):**
```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "staging": {
      "distribution": "internal",
      "env": { "API_URL": "https://staging.api.com" }
    },
    "production": {
      "autoIncrement": true
    }
  }
}
```

**OTA Updates (EAS Update / CodePush):**
```bash
# Отправить обновление (только JS, не нативный код)
eas update --branch production --message "Fix crash on iOS 17"

# Staged rollout — сначала 10% пользователей
eas update --branch production --rollout-percentage 10
```

Когда нельзя использовать OTA:
- Изменения в native коде (новый Native Module, Podfile)
- Изменения в `android/` или `ios/` директориях
- Обновление React Native версии

---

## БЛОК 12. Platform-specific код

### Q25: Как писать код для iOS и Android отдельно?

**Способ 1 — Platform.select:**
```javascript
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 } },
      android: { elevation: 4 },
    }),
  },
});
```

**Способ 2 — Platform-specific файлы (рекомендую для большой логики):**
```
Button.ios.tsx   ← импортируется на iOS
Button.android.tsx ← импортируется на Android
Button.tsx ← fallback для обоих, если .ios/.android нет
```
```javascript
import Button from './Button'; // Metro автоматически выбирает нужный файл
```

**Способ 3 — условие:**
```javascript
const isIOS = Platform.OS === 'ios';
const statusBarHeight = Platform.OS === 'ios'
  ? 44
  : StatusBar.currentHeight ?? 0;
```

---

## БЛОК 13. Реальные задачи с собеседований

### Задача 1: Infinite Scroll с pagination
```typescript
function useInfiniteScroll<T>(
  fetchFn: (page: number) => Promise<T[]>,
  pageSize = 20
) {
  const [items, setItems] = useState<T[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;

    setLoading(true);
    try {
      const newItems = await fetchFn(page);
      if (newItems.length < pageSize) setHasMore(false);
      setItems(prev => [...prev, ...newItems]);
      setPage(p => p + 1);
    } catch (e) {
      setError(e as Error);
    } finally {
      setLoading(false);
    }
  }, [loading, hasMore, page, fetchFn, pageSize]);

  return { items, loadMore, loading, hasMore, error };
}

// Использование с FlashList
const { items, loadMore, loading } = useInfiniteScroll(fetchProducts);

<FlashList
  data={items}
  renderItem={({ item }) => <ProductCard product={item} />}
  estimatedItemSize={120}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  ListFooterComponent={loading ? <ActivityIndicator /> : null}
/>
```

### Задача 2: Throttle хук
```typescript
function useThrottle<T>(value: T, limit: number): T {
  const [throttled, setThrottled] = useState(value);
  const lastRan = useRef(Date.now());

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= limit) {
        setThrottled(value);
        lastRan.current = Date.now();
      }
    }, limit - (Date.now() - lastRan.current));

    return () => clearTimeout(handler);
  }, [value, limit]);

  return throttled;
}
```

### Задача 3: Race condition в fetch
```typescript
// Проблема: быстро меняешь userId → два запроса в лёте
// Второй может прийти раньше первого → показываем старые данные

function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let isCurrent = true; // флаг актуальности

    fetchUser(userId).then(data => {
      if (isCurrent) setUser(data); // игнорируем устаревший ответ
    });

    return () => { isCurrent = false; };
  }, [userId]);

  return user;
}
```

---

## 📋 Финальный чеклист

### Обязательно уметь объяснить:
- [ ] Bridge vs JSI своими словами (без заучивания)
- [ ] Почему Hermes быстрее на cold start
- [ ] Event loop: порядок microtasks и macrotasks
- [ ] useEffect зависимости и closure trap
- [ ] Когда useMemo — антипаттерн
- [ ] Разница FlashList и FlatList внутри

### Обязательно написать на доске:
- [ ] useDebounce хук
- [ ] useFetch с AbortController и cleanup
- [ ] Infinite scroll хук
- [ ] Простой Zustand store
- [ ] Race condition fix в useEffect

### Из реального опыта (готовь истории):
- [ ] Самый сложный performance проблема и как решал
- [ ] Как внедрял новую технологию в команду
- [ ] Самый большой рефакторинг который делал
- [ ] Как находил и фиксил memory leak

---

*Интервью на Senior — это разговор равных. Объясняй trade-offs, спрашивай контекст, не давай ответ который "всегда верный" когда его нет.*
