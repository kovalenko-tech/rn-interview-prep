# Senior React Native — питання та детальні відповіді

---

## БЛОК 1. React Native Internals

### Q: Як працює React Native архітектура? Стара vs нова?

**Стара архітектура (Bridge):**
```
JS Thread → JSON серіалізація → Bridge → Native Thread
```
- JS і Native — два окремих потоки, спілкуються через Bridge
- Кожна взаємодія → серіалізація в JSON → асинхронно → десеріалізація
- Проблема: затримки, неможливо синхронно викликати нативний код
- `UIManager.measure()` — асинхронний, не можна заблокувати

**Нова архітектура (JSI — JavaScript Interface):**
```
JS Thread → JSI (C++ прошарок) → Native (синхронно, без JSON)
```
- JSI дає JS прямий доступ до C++ об'єктів
- TurboModules: нативні модулі через JSI, lazy-loaded
- Fabric: новий renderer, concurrent режим, синхронний доступ до shadow tree
- Bridgeless mode: повністю без старого Bridge

**Відповідь одним реченням:**
> "Стара архітектура — асинхронний JSON Bridge між JS і Native. Нова — синхронний JSI прошарок на C++, що дає прямий виклик нативного коду без серіалізації."

---

### Q: Що таке Hermes і навіщо він?

**Hermes** — JS рушій від Meta, оптимізований для мобільних пристроїв.

Що робить:
- **AOT компіляція**: перетворює JS в bytecode при білді, а не при запуску
- **Менше RAM**: менший heap, garbage collector оптимізований
- **Швидший cold start**: bytecode вже готовий, не потрібен JIT при старті

Порівняння:
```
V8/JavaScriptCore: парсинг JS → AST → bytecode → JIT → виконання
Hermes:            bytecode вже готовий → одразу виконання
```

Ціна: Hermes без JIT → повторний код не оптимізується так добре як V8. Але для мобільних додатків trade-off вигідний.

---

### Q: Що таке Fabric?

Fabric — новий рендерер React Native.

Стара схема:
```
React → Shadow Tree (JS) → Bridge → Native Layout Thread
```
Нова (Fabric):
```
React → C++ Shadow Tree → синхронно → Native View
```

Переваги:
- Concurrent rendering (React 18 features — Suspense, transitions)
- Синхронний доступ — layout можна виміряти в тому ж кадрі
- Менше копій даних між потоками

---

### Q: Як працює JavaScript Thread і чому він важливий?

В RN є кілька потоків:
- **JS Thread** — весь React код, бізнес-логіка, state
- **UI Thread** (Main) — нативні view, layout, touch events
- **Shadow Thread** — розрахунок Yoga layout (Flexbox)
- **Native Modules Thread** — нативні виклики

JS Thread — вузьке місце. Якщо він зайнятий > 16ms → frame drop → jank.

Що блокує JS Thread:
- Важкі обчислення в рендері
- Синхронні операції з AsyncStorage
- Великі JSON.parse
- Занадто часті setState в циклі

Рішення: `InteractionManager.runAfterInteractions()`, `requestAnimationFrame()`, або винести в WorkerThread / Reanimated worklets.

---

## БЛОК 2. Анімації

### Q: В чому різниця useNativeDriver: true і false?

**`useNativeDriver: false`** (JS-driven):
```
JS Thread: обчислює значення → Bridge → Native: оновлює view
```
Кожен кадр проходить через JS Thread → якщо він зайнятий → jank.

**`useNativeDriver: true`** (Native-driven):
```
JS Thread: надсилає конфігурацію → Native Thread: самостійно виконує
```
Анімація живе повністю в Native Thread — JS може бути зайнятий, анімація йде плавно.

Обмеження `useNativeDriver: true`:
- Тільки transform і opacity — не можна анімувати width, height, backgroundColor
- Не можна динамічно змінювати значення через setValue після старту

**Reanimated 2/3** вирішує це — worklets виконуються в UI Thread, доступні майже всі властивості.

---

### Q: Коли використовувати Reanimated vs Animated API?

| | Animated API | Reanimated 3 |
|---|---|---|
| Простота | ✅ простіше | ❌ складніше |
| Продуктивність | ⚠️ JS-driven | ✅ UI Thread |
| Gesture інтеграція | ❌ складно | ✅ нативно |
| Інтерполяція | обмежена | повна |

**Правило:** Animated для простих fade/slide, Reanimated для всього що пов'язане з жестами або складними інтерактивними анімаціями.

---

## БЛОК 3. Продуктивність

### Q: Як профілювати RN додаток?

**Інструменти:**
1. **Flipper** + Performance plugin — JS/Native frame rate, heap
2. **React DevTools Profiler** — який компонент рендериться і скільки
3. **Chrome DevTools** — CPU profiling JS Thread
4. **Xcode Instruments** — Memory, CPU для iOS native частини
5. **Android Studio Profiler** — те саме для Android

**Що шукати:**
- Довгі JS frames (> 16ms) → що виконується в рендері?
- Часті re-renders одного компонента
- Memory leaks — heap зростає і не падає після navigation.goBack()
- Blocked main thread на iOS/Android

---

### Q: FlatList vs FlashList — що краще?

**FlatList:**
- Recyclerview-подібний, але JS-driven
- При скролі: JS бачить visible items → рендерить → Native показує
- Проблема: blank areas при швидкому скролі, JS Thread перевантажений

**FlashList (від Shopify):**
- Recycled component pool — замість destroy/create перевикористовує
- Визначає розміри заздалегідь якщо `estimatedItemSize` задано
- 10x менше JS роботи при скролі
- Вимога: `estimatedItemSize` + стабільні `keyExtractor`

**Коли що:**
- < 100 items простих → FlatList нормально
- > 100 items або складні cards → FlashList
- Infinite scroll → FlashList завжди

---

### Q: Коли useMemo / useCallback шкодять?

`useMemo` і `useCallback` — не безкоштовні:
- Зберігають closure в пам'яті
- При кожному рендері перевіряють deps array (shallow compare)

**Коли корисні:**
```javascript
// Важке обчислення
const sorted = useMemo(() => heavySort(data), [data]);

// Callback передається дочірньому компоненту обгорнутому в React.memo
const onPress = useCallback(() => navigate(id), [id]);
```

**Коли шкодять:**
```javascript
// Просте значення — overhead більший ніж benefit
const doubled = useMemo(() => count * 2, [count]); // ❌ не треба

// Компонент не в React.memo — useCallback марний
<Component onPress={useCallback(() => {}, [])} /> // ❌ якщо Component не memo
```

**Правило:** спочатку профілюй, потім оптимізуй. React.memo + useCallback тільки для доведеної проблеми.

---

### Q: Як уникнути зайвих re-renders?

1. **React.memo** для чистих компонентів
2. **Винести state вниз** — якщо стейт потрібен тільки одному компоненту, не піднімай вгору
3. **Context + useMemo** — розбивай великий контекст на маленькі
4. **Selector pattern** (Zustand, Redux) — підписуватись тільки на потрібну частину стейту
5. **Стабільні посилання** — не створюй об'єкти/масиви inline в render

```javascript
// ❌ новий об'єкт кожен рендер
<Component style={{ marginTop: 10 }} />

// ✅ стабільне посилання
const styles = StyleSheet.create({ container: { marginTop: 10 } });
<Component style={styles.container} />
```

---

## БЛОК 4. State Management

### Q: Redux Toolkit vs Zustand — коли що?

**Redux Toolkit:**
- Великі команди, складний стан, потрібна структура
- DevTools, time-travel debugging
- Middleware (saga, thunk) для складних async флоу
- Вербозний навіть з RTK

**Zustand:**
- Мінімальний boilerplate
- Відмінна продуктивність (вбудований selector)
- Простий API
- Менше структури → ризик для великих команд

**Мій вибір:**
```
Маленький проект / стартап → Zustand
Середній+ проект / велика команда → Redux Toolkit
Сервер стан (API) → TanStack Query (окремо від UI стану)
```

---

### Q: Як організувати optimistic updates?

```javascript
const addItem = useMutation({
  mutationFn: api.addItem,
  onMutate: async (newItem) => {
    // 1. Відміняємо pending запити
    await queryClient.cancelQueries(['items']);

    // 2. Зберігаємо попередній стан
    const previous = queryClient.getQueryData(['items']);

    // 3. Оптимістично оновлюємо
    queryClient.setQueryData(['items'], old => [...old, newItem]);

    return { previous };
  },
  onError: (err, newItem, context) => {
    // 4. При помилці — відкат
    queryClient.setQueryData(['items'], context.previous);
  },
  onSettled: () => {
    // 5. Завжди рефетчимо після
    queryClient.invalidateQueries(['items']);
  },
});
```

---

## БЛОК 5. Navigation

### Q: Як організувати auth flow в React Navigation?

Правильний підхід — умовний рендер navigator, а не redirect:

```javascript
function RootNavigator() {
  const { isLoggedIn, isLoading } = useAuth();

  if (isLoading) return <SplashScreen />;

  return (
    <Stack.Navigator>
      {isLoggedIn ? (
        // Authenticated stack
        <Stack.Screen name="Home" component={HomeScreen} />
      ) : (
        // Auth stack
        <Stack.Screen name="Login" component={LoginScreen} />
      )}
    </Stack.Navigator>
  );
}
```

Чому не redirect: при logout React Navigation автоматично чистить history — неможливо повернутись назад на захищений екран через кнопку "back".

---

### Q: Deep linking + Universal Links — як реалізувати?

```javascript
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: 'home',
      Product: {
        path: 'product/:id',
        parse: { id: Number },
      },
    },
  },
};

<NavigationContainer linking={linking}>
```

Universal Links (iOS) / App Links (Android) потребують:
- iOS: `apple-app-site-association` файл на сервері
- Android: `assetlinks.json` на сервері
- Обидва верифікуються OS при встановленні

---

## БЛОК 6. Тестування

### Q: Що тестуєш в першу чергу?

```
Пріоритет:
1. Бізнес-логіка / use cases / утиліти → чисті unit тести
2. Кастомні хуки → renderHook з @testing-library/react-hooks  
3. Критичні user flows → E2E (Detox / Maestro)
4. Компоненти → тільки складні з логікою, не кожен дамбний компонент
```

Snapshot тести — обережно: вони ламаються при будь-якій зміні UI і не тестують поведінку.

---

### Q: Як мокати native modules в Jest?

```javascript
// __mocks__/@react-native-async-storage/async-storage.js
export default {
  getItem: jest.fn(() => Promise.resolve(null)),
  setItem: jest.fn(() => Promise.resolve()),
  removeItem: jest.fn(() => Promise.resolve()),
};

// jest.config.js
moduleNameMapper: {
  '@react-native-async-storage/async-storage':
    '<rootDir>/__mocks__/@react-native-async-storage/async-storage.js',
}
```

Або використовуй готові: `@react-native-async-storage/async-storage/jest/async-storage-mock`.

---

## БЛОК 7. TypeScript

### Q: Як типізувати Navigation params?

```typescript
// types/navigation.ts
export type RootStackParamList = {
  Home: undefined;
  Profile: { userId: string };
  Product: { id: number; title: string };
};

// Компонент
type Props = NativeStackScreenProps<RootStackParamList, 'Product'>;

function ProductScreen({ route, navigation }: Props) {
  const { id, title } = route.params; // типізовано
}

// useNavigation з типом
const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
```

---

### Q: Utility types — поясни і покажи приклад

```typescript
// Partial — всі поля опціональні
type UpdateUser = Partial<User>;

// Required — всі поля обов'язкові
type FullUser = Required<User>;

// Pick — вибрати поля
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit — виключити поля
type UserWithoutPassword = Omit<User, 'password'>;

// Record — словник
type Translations = Record<string, string>;

// ReturnType — тип повернення функції
type AuthResult = ReturnType<typeof login>;

// Parameters — тип аргументів функції
type LoginParams = Parameters<typeof login>[0];
```

---

## БЛОК 8. Безпека

### Q: Як безпечно зберігати токени?

```
❌ AsyncStorage — відкритий текст, доступний через бекап на Android
✅ react-native-keychain — iOS Keychain / Android Keystore
✅ expo-secure-store — те саме через Expo
```

```javascript
import * as Keychain from 'react-native-keychain';

// Зберегти
await Keychain.setGenericPassword('token', accessToken);

// Отримати
const creds = await Keychain.getGenericPassword();
if (creds) const { password: token } = creds;

// Видалити (logout)
await Keychain.resetGenericPassword();
```

На Android додатково: `setSecurityLevel(Keychain.SECURITY_LEVEL.SECURE_HARDWARE)` — зберігає в hardware-backed Keystore.

---

### Q: Certificate Pinning — що це і як реалізувати?

Certificate Pinning = додаток перевіряє не тільки що сертифікат валідний (CA), але і що це саме той сертифікат або публічний ключ який він очікує.

Захищає від: MITM атак навіть якщо атакуючий має валідний CA сертифікат.

```javascript
// react-native-ssl-pinning
import { fetch } from 'react-native-ssl-pinning';

fetch('https://api.myapp.com/data', {
  sslPinning: {
    certs: ['api_cert'] // api_cert.cer в assets
  }
});
```

Ризик: якщо сертифікат протермінується і ти не оновиш додаток → 100% поломка. Краще пінити публічний ключ (він рідше змінюється).

---

## БЛОК 9. CI/CD і деплой

### Q: Як організований release process?

Типовий flow:
```
feature branch → PR → CI (lint, test, build) → merge to develop
develop → staging build (EAS / Fastlane) → QA
develop → release branch → production build → App Store / Play Store
```

Versioning:
- `major.minor.patch` + build number
- build number автоінкремент в CI (`git rev-list --count HEAD`)

OTA Updates (CodePush):
```javascript
// Безпечне котення: staged rollout 10% → 50% → 100%
// Rollback: codepush rollback MyApp Production
// Обмеження: тільки JS bundle, не native код
```

---

## БЛОК 10. Типові задачі на whiteboard

### Реалізуй debounce хук
```javascript
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}
```

### Реалізуй infinite scroll
```javascript
function useInfiniteList(fetchFn) {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    if (!hasMore) return;
    const newItems = await fetchFn(page);
    if (newItems.length === 0) { setHasMore(false); return; }
    setItems(prev => [...prev, ...newItems]);
    setPage(p => p + 1);
  }, [page, hasMore, fetchFn]);

  return { items, loadMore, hasMore };
}
```

### Поясни memory leak в useEffect
```javascript
// ❌ leak: якщо компонент unmount поки fetch іде
useEffect(() => {
  fetch('/api/data').then(data => setState(data)); // setState на unmounted!
}, []);

// ✅ fix з AbortController
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal })
    .then(data => setState(data))
    .catch(err => { if (err.name !== 'AbortError') throw err; });
  return () => controller.abort();
}, []);
```

---

## Чекліст перед інтервью

- [ ] Міг пояснити JSI vs Bridge своїми словами
- [ ] Знаю різницю FlashList і FlatList
- [ ] Можу написати debounce, infinite scroll, optimistic update
- [ ] Знаю useNativeDriver обмеження
- [ ] Розумію коли useMemo/useCallback — антипаттерн
- [ ] Можу типізувати navigation params
- [ ] Знаю де зберігати токени і чому не AsyncStorage
- [ ] Маю приклад з реального проекту для кожного блоку

---

*З 13 роками досвіду — тебе питатимуть не "що це", а "чому саме так і які trade-offs". Готуй приклади зі своїх проектів.*
