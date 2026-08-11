<div align="center">
  <img src="./eturlia.png" alt="Eturlia" width="900">
  <h1>Eturlia</h1>
  <p>Гибридное серверное ядро Minecraft <strong>1.21.1</strong><br>
  региональная многопоточность Folia · загрузка модов NeoForge</p>
  <p><a href="#русский">Русский</a> · <a href="#english">English</a></p>
</div>

---

<div id="русский"></div>

# Русский

> [!WARNING]
> Экспериментальный проект. **Не для продакшена.** Делайте бэкапы миров.
> Многие моды рассчитаны на один серверный поток Vanilla/Forge и ломаются на региональной модели Folia.

## Что такое Eturlia

**Eturlia** — одно jar-ядро, в котором одновременно живут два стека:

| Слой | Роль |
|------|------|
| **[Folia](https://github.com/PaperMC/Folia)** | Форк Paper: мир режется на независимые тик-регионы, которые крутятся на разных ядрах CPU |
| **[NeoForge](https://github.com/neoforged/NeoForge) 21.1.248** | Модлоадер (FancyModLoader 4.0.43): `mods/`, RegisterEvent, lifecycle, datapacks, mixins |

Цель — масштабирование Folia без отказа от экосистемы NeoForge (Create, Farmers Delight и соседние техпаки), по мере закрытия runtime-разрывов между региональной моделью и ожиданиями модов.

| | |
|---|---|
| **Артефакт** | `eturlia-1.21.1-neoforge-21.1.248.jar` |
| **Launch target** | `eturliaserver` |
| **Точка входа** | `eturlia.EturliaServer` → `org.bukkit.craftbukkit.Main` (через `EturliaServerLaunchHandler`) |
| **Релиз** | [v0.2.5](https://github.com/eturnercus/Core/releases/tag/v0.2.5) |

## Как это работает

### Загрузка (runtime)

```text
java -jar eturlia-1.21.1-neoforge-21.1.248.jar
        │
        ▼
 Eturlia launcher
   · распаковка nested libs (Folia, FML, NeoForge, runtime)
   · подготовка classpath и launch services
        │
        ▼
 ModLauncher / FancyModLoader
   · scan каталога mods/
   · bootstrap NeoForge, coremods, mixins
        │
        ▼
 Folia / Paper server
   · регионы, чанки, сущности, плагины (folia-supported)
        │
        ▼
 NeoForge hooks в NMS
   · RegisterEvent, game lifecycle, datapack reload, API-расширения
```

1. **Launcher** поднимает вложенные библиотеки из fat-jar и передаёт управление ModLauncher.
2. **FML** находит моды, прогоняет discovery / loading / common setup и подключает coremods Eturlia (редирект `main` → `EturliaServer` и служебные трансформы).
3. **Folia** тикает мир по регионам: соседние чанки могут жить на разных потоках; кросс-регионные обращения требуют правильных планировщиков.
4. **Патчи ядра** вшивают в Folia/Paper API и поведение, которые ожидает NeoForge (регистры, food/crafting/fire, reload, entity-list фасады и т.д.).

### Сборка (dev)

1. **paperweight** накатывает `patches/api` и `patches/server` (апстрим Folia + слой Eturlia/NeoForge) на Paper.
2. **Шимы** (`build-data/eturlia-neoforge-shims`) — справочные stub-сигнатуры NeoForge.
   Сборка их **не использует**: Folia-Server компилируется против опубликованного
   NeoForge universal (`compileOnly` в `build.gradle.kts`).
3. В runtime в jar вложен опубликованный **NeoForge universal 21.1.248** — тот же major, что у целевых модов.
4. Опциональные модули `eturlia-compat-create` / `eturlia-compat-sable` закрывают узкие region-bridges; они не заменяют пробелы в самом ядре.

### Почему моды всё ещё могут падать

Стартовый путь FML уже проходит. Текущий слой работы — **runtime gaps**: мод вызывает API или полагается на однопоточный NMS, которого нет или оно ведёт себя иначе под Folia. Типичные классы проблем:

- holders / DeferredHolder / порядок RegisterEvent;
- Folia entity tick list / section managers вместо vanilla-структур;
- remapping / refmap у mixin-модов;
- однопоточные допущения в Create, Moonlight и соседних библиотеках.

## Версии

| Компонент | Версия |
|-----------|--------|
| Minecraft | 1.21.1 |
| NeoForge | **21.1.248** |
| FancyModLoader | 4.0.43 |
| Paper upstream | `84281cee…` (Folia `dev/1.21.1`) |
| Java | **21** |

## Сборка

Нужны **git clone** (не ZIP), **JDK 21**, доступ к Maven PaperMC и NeoForged.

```bash
./gradlew applyPatches                 # Folia + Eturlia/NeoForge патчи
./gradlew :folia-server:build          # Folia-Server
./gradlew :folia-server:eturliaStandaloneJar
```

Результат: `build/libs/eturlia-1.21.1-neoforge-21.1.248.jar`

```bash
./patch.sh   # applyPatches
./rb.sh      # rebuildServerPatches
```

## Запуск

```bash
java -Xmx8G -jar build/libs/eturlia-1.21.1-neoforge-21.1.248.jar --nogui
# или jar с GitHub Releases (v0.2.5+)
```

- Jar — обёртка: она распаковывает библиотеки и запускает **дочернюю JVM** с сервером.
  Флаги JVM обёртки (`-Xmx`, `-XX:*`, `-Deturlia.*`) пробрасываются в дочерний процесс,
  а SIGTERM/Ctrl+C обёртки останавливает и сервер.
- Первый запуск: примите `eula.txt`.
- Моды NeoForge → `mods/`.
- Плагины Bukkit/Paper → только с `folia-supported: true`.
- Консоль по умолчанию **calm** (INFO без зелёного). `-Deturlia.console.color=full|off` при необходимости.
- Плагины с `libraries:` в `plugin.yml` могут не резолвить Maven под ModLauncher; без `libraries:` — ок.

### Пак модов — текущий процесс (не «убери мод»)

Eturlia закрывает Folia↔NeoForge gaps **патчами ядра**. Не надо выкидывать контент-моды из ASIC-списка ради boot.

| Действие | Когда |
|----------|--------|
| Поставить jar **v0.2.5+** (патчи через **0094**) | libjf / respackopts / WorldWeaver больше не FATAL на `Main` |
| **Обновить** Lithostitched до **≥ 1.7.13** | Бета `1.7.10+beta4` — тот же мод, битый jar; гейт останавливает boot |
| Оставить `spark-neoforge` / Arclight sable patch | Ядро **soft-skip** → `*.jar.eturlia-skipped` (не удаляет файл; bundled `/spark` работает). Для Sable предпочтителен `arclight_sable_patch-*-eturlia-shim.jar` |
| **WorldEdit / FAWE как NeoForge-мод** в `mods/` | Soft-skip — ставьте WorldEdit/FAWE **Folia-плагином** в `plugins/` |
| Перекачать `easy_npc`, убрать `.jar1` / `.bak*` | Битые/мусорные файлы на диске, не пробел ядра |
| BetterEnd + BCLib + WorldWeaver | Нужен **wunderlib**; тяжёлый worldgen = RISK на Folia |
| Оптимизаторы (FerriteCore и т.п.) | **Не обязательны** для boot |

Аудит: [`docs/PACK_COMPAT_ASIC_2026-08.md`](./docs/PACK_COMPAT_ASIC_2026-08.md) · smoke: [`docs/SMOKE_ASIC_2026-08-07.md`](./docs/SMOKE_ASIC_2026-08-07.md) · релиз-ноты: [`docs/RELEASE_NOTES_0.2.5.md`](./docs/RELEASE_NOTES_0.2.5.md).

## Smoke-статус / whitelist matrix

| Набор | Статус | Совместимость |
|-------|--------|---------------|
| Пустой / лёгкий стек (Cloth, Curios, GeckoLib, JEI, …) | SML + Folia `Done` | whitelist |
| Farmers Delight (+ Cloth) | SML + `Done` | whitelist |
| Create 6.0.10 | SML + `Done` + вход клиента держится; tick gaps под нагрузкой ещё ловятся | whitelist (best-effort) |
| Moonlight Lib | SML + `Done` + worlds; SoftFluid/FluidType bridged | whitelist (best-effort) |
| ASIC core (FD, CreativeCore, Farm&Charm, amendments, TF, Supplementaries, …) | `Done` + worlds (0084–0093) | whitelist (best-effort) |
| libjf + respackopts (+ litho ≥1.7.13) | MainMixin OK + `Done` (0094) | whitelist |
| Bukkit/Paper плагины (`folia-supported`) | discovery + load | whitelist (без `libraries:`) |
| Bundled spark | `spark tps` OK; TPS с Folia global tick | whitelist |
| FerriteCore / полный ASIC gameplay | не certified | **at your own risk** |

Политика моддеров: [`docs/MODDER_POLICY.md`](./docs/MODDER_POLICY.md).

### Crash-reports

- Vanilla/Paper → `crash-reports/`
- Eturlia-отчёты с region id → `eturlia-crash-reports/` (`-Deturlia.crash.dir=…`).
  Пишет `eturlia.EturliaServer`, который launch handler ставит перед передачей управления
  в `org.bukkit.craftbukkit.Main`.

### Конфиг ядра — `config/eturlia.yml`

Всё, что специфично для Eturlia, настраивается здесь. Файл создаётся при первом запуске.
Настройки Paper/Spigot/Bukkit **не дублируются** — для них свои файлы (ссылки в секции
`reference:`).

| Секция | Что настраивает |
|--------|-----------------|
| `threads` | потоки region-тика Folia и размер сетки регионов, chunk worker / IO, пул чата |
| `jvm` | флаги **дочерней JVM**: `-Xmx`/`-Xms`, `Paper.WorkerThreadCount`, `Paper.IOThreadCount`, произвольные аргументы |
| `chunks` | view/simulation distance, лимиты отправки и генерации чанков, автосейв |
| `region` | режим region guard и валидации событий (`STRICT`/`WARN`/`PERMISSIVE`) |
| `validation` | проверка region-потоков в coremod-хуках, строгий режим |
| `logging` | файл диагностики ядра, печатать ли преды в консоль |
| `crash` | каталог region-отчётов, ставить ли обработчик |
| `hygiene` | что делать с несовместимыми jar'ами в `mods/` (`skip`/`warn`/`off`) |
| `mods` | минимальная версия Lithostitched, строгая блокировка модов из манифеста |
| `lod`, `spark`, `watchdog`, `console` | LOD-мост, spark, watchdog, цвет консоли |

Количество потоков задаётся в двух местах, потому что применяется на разных этапах:
`threads.region-tick-threads` и `region-grid-exponent` уходят в Folia
(`GlobalConfiguration.threadedRegions`), а `jvm.worker-threads` / `jvm.io-threads` —
на командную строку дочерней JVM, потому что Moonrise читает их до загрузки конфига.

### Консоль и логи

При старте ядро печатает баннер ETURLIA и несколько строк статуса. Предупреждения и ошибки
Eturlia остаются в консоли, но **строго одной строкой**:

```text
[Eturlia] WARN config/eturlia.yml has _version=2 but this build expects 3
[Eturlia] ERROR mod compatibility check failed — IllegalStateException: c2me is excluded
```

Полный текст со стектрейсами уходит в **`logs/eturlia.log`**. Логи самого сервера (Log4j2)
не трогаются.

| Флаг | Что делает |
|------|------------|
| `-Deturlia.console.errors=off` | вообще не печатать предупреждения Eturlia в консоль (в файл всё равно пишутся) |
| `-Deturlia.console.color=off` | без ANSI-цвета (также уважается `NO_COLOR`) |
| `-Deturlia.log.file=<path>` | другое расположение файла диагностики |

### Вход игрока: что работает

**Клиент NeoForge заходит на сервер и остаётся в мире** — проверено настоящим клиентом
(NeoForge 21.1.248, headless, автоподключение), не эмулятором протокола:

```
There are 1 out of maximum 20 players online
```

Рукопожатие NeoForge собрано из 13 точек интеграции, которые NeoForge добавляет своими
патчами к ванильным классам, а в патче `0095` их не было. Каждая найдена по реальной
попытке входа:

| Симптом у клиента | Чего не хватало |
|---|---|
| «сервер не использует NeoForge» | пейлоады согласования в `startConfiguration` |
| `Internal Exception: io.netty.handler.codec…` | кодек не знал модовых пейлоадов |
| `SocketException: Connection reset` | `PacketEncoder.getProtocolInfo()` |
| `UnsupportedOperationException` | `MinecraftServer.execute()` бросал исключение — на Folia нет главного потока, задачи идут в глобальный регион |
| сервер **падал** при входе | `ConfigurationTask$Type(ResourceLocation)` |
| обрыв на синхронизации реестров | `PacketFlow.isClientbound()`, `IServerConfigurationPacketListenerExtension`, `CustomPacketPayload.toVanillaClientbound()`, `FeatureFlagRegistry.getAllFlags()` |
| обрыв сразу после входа | `minecraft:register` кодировался типом NeoForge вместо `DiscardedPayload` |
| `Sending unknown packet clientbound/minecraft:bundle` | распаковщик bundle стоял в пайплайне позже сплиттера NeoForge |
| `No value with id -1` при чтении чанка (с модами) | модовые блоки не попадали в `Block.BLOCK_STATE_REGISTRY` — см. [Create](#create-текущее-состояние) |

Сбой согласования или конфигурации теперь отключает **одно соединение**, а не валит процесс:
Folia убивает сервер при исключении в region-тике, и кривой клиент дважды этим пользовался.

### Create: текущее состояние

Create **загружается, не роняет сервер, и с ним можно зайти в игру**. Проверено на
тестовом стенде настоящим headless NeoForge-клиентом: `EturliaTester joined the game`,
`join=1 left=0`. Чинить пришлось две независимые вещи.

**1. Падение тика региона.** Миксин Create `ProjectileUtilMixin` оборачивает
`Entity.canRiderInteract()` внутри `ProjectileUtil` — метод и вызов добавляет NeoForge.
Без них инъекция падала критически, это валило тик региона, а с ним весь процесс.
Оба добавлены.

**2. Обрыв клиента через пару секунд после входа.**

```
IllegalArgumentException: No value with id -1
  at IdMap.byIdOrThrow → LinearPalette.read → LevelChunkSection.read
```

Диагностика в записи палитры назвала виновника поимённо: `Block{create:scoria}` уходил
на провод с идентификатором `-1`.

`Block.BLOCK_STATE_REGISTRY` заполняется один раз — статическим блоком `Blocks`, на
бутстрапе ванильного контента. Дальше апстримный NeoForge поддерживает карту через
`NeoForgeRegistryCallbacks.BlockCallbacks`, который вешается на `BuiltInRegistries.BLOCK`;
этот патч в Eturlia не переносился. Поэтому **ни один модовый блок в карту не попадал**, и
кодировщик чанка писал `-1` — на любом модовом блоке в прогруженном чанке.

Вместо переноса всего registry-callback API карта дозаполняется один раз, сразу после
`ServerModLoader` — `Block.eturlia$rebuildStateIds()`. Реестр к этому моменту заморожен,
обход идёт в порядке идентификаторов — ровно в том же порядке клиент перестраивает свою
карту, так что стороны сходятся. На наборе Create + Aeronautics + lithostitched + Sable
это 38 909 состояний.

Тем же способом восстановлены ещё две производные карты, которые в апстриме ведут те же
колбэки:

- `Item.BY_BLOCK` (`Item.eturlia$rebuildBlockItemMap()`) — без неё `Item.byBlock()`
  отвечает `AIR` на любой модовый блок (ломает выбор блока, дропы, часть рецептов);
- `PoiTypes.TYPE_BY_STATE` (`PoiTypes.eturlia$rebuildStateMap()`) — без неё модовые
  points of interest не привязываются к блокам.

Запись палитры теперь навсегда сообщает о таком `-1` одной строкой с именем блока, вместо
того чтобы оставлять клиенту неатрибутируемое `No value with id -1`.

### Большие сборки: почему мод может не завестись

На наборе из 90 модов (прод-сборка NoteBuns) выяснилось: **самая частая причина падения —
не NeoForge, а Paper**. Folia наследует переписанные Paper ванильные методы, а мод целится в
ванильную форму. Mixin считает это фатальным, и **один мод не даёт стартовать всему серверу**.

Что чинится на стороне ядра:

| Симптом | Причина | Что сделано |
|---|---|---|
| `world_folder.MainMixin from mod wover ... (0/1) succeeded` | в `net.minecraft.server.Main` было **два** метода `main`; моды целятся в мойанговский по имени, а Mixin разрешал имя в CraftBukkit-перегрузку | CraftBukkit-вход переименован в `eturlia$bootFromOptions`; `main` снова один |
| `@Shadow field existingFileHelper was not located in TagsProvider` | NeoForge добавляет это поле, патч не переносился | поле добавлено (на сервере всегда `null` — datagen не запускается) |
| `@ModifyArg ... knockback(DDD)V ... Scanned 0 target(s)` | Paper заменил вызов внутри `hurt` на 5-аргументный ради `EntityKnockbackEvent` | вызов вернул ванильную форму; attacker и cause передаются полем, событие Paper не потеряно |
| `AbstractMethodError: ... does not define ... 'abstract int getIdFor'` в `Block.<init>` | Paper добавил в `Property` **абстрактный** `getIdFor(T)` для своей таблицы состояний. Любой мод со своим `Property` компилировался против ванили, где метода нет — и падал в конструкторе `Block`, то есть не мог зарегистрировать вообще ничего | метод больше не абстрактный: по умолчанию возвращает индекс значения в `getPossibleValues()` (с ленивым кэшем — он на горячем пути `setValue`). Ванильные подклассы по-прежнему переопределяют его, их быстрый путь не тронут |
| `NullPointerException: ... "snapshots" is null` при загрузке модов | апстримный NeoForge снимает `GameData.vanillaSnapshot()` в патче `Bootstrap`; Eturlia вызывает сток. Снимка нет — и откат реестров падает, **пряча настоящую ошибку мода** | снимок берётся между `Bootstrap.validate()` и `ServerModLoader.load()` |

Что **не** чинится и требует отключить мод:

- **FerriteCore** — заменяет карту соседей блок-состояний своей структурой, а Paper уже имеет
  свою (`ZeroCollidingReferenceStateTable`). Итог — `NullPointerException: index_table is null`
  в `Blocks.<clinit>`. На Paper/Folia он не нужен: экономию памяти там уже сделали.
- **tt20** — переписывает серверный тик-луп, которого у Folia нет в ванильном виде.
- Моды, чьи инъекции целятся в места, где Paper заменил вызов или добавил локальную
  переменную (`LVT ... has incompatible changes`): BetterEnd, bclib, WorldWeaver,
  letsdo-vinery, horseman и подобные.

Зарегистрирован `EturliaMixinErrorHandler`: `InvalidMixinException` от **чужого** мода
понижается до предупреждения с именем мода в консоли, вместо остановки сервера. Ошибки в
mixin'ах Eturlia, Folia и NeoForge остаются фатальными. Отключается
`-Deturlia.mixin.errors=fatal`, тихий режим — `=quiet`. Это размен, а не починка: не
применённый mixin означает, что мод недополучил свою логику.

`InjectionError` (провал самой инъекции) обработчик перехватить не может — Mixin вызывает
обработчики только для `InvalidMixinException`. Такой мод придётся отключать.

### Известные ограничения

- `mods/`-гигиена **переименовывает** конфликтные jar'ы (`spark-*neoforge*`, оригинальный
  `arclight_sable_patch`) в `*.jar.eturlia-skipped` при каждом старте. Отключается:
  `-Deturlia.mods.hygiene=warn` (только сообщать) или `=off` (не сканировать).

- `ServerTickEvent.Pre/Post` (и `LevelTickEvent`) фактически шлются **из каждого
  region-потока**, а не один раз за глобальный тик. Моды с однопоточными допущениями
  в обработчиках тика получат конкурентные вызовы.
- Модули `compat/eturlia-compat-create` и `compat/eturlia-compat-sable` — **скелеты**:
  все хендлеры пустые, миксинов нет, зависимости не запинены. Они не собираются
  root-проектом и не проверяются CI.
- Region guard / thread validator ловят вызовы, но их ещё не подключили ко всем
  NMS точкам входа — покрытие частичное.

### Semver

Теги `vMAJOR.MINOR.PATCH` (сейчас **[v0.2.5](https://github.com/eturnercus/Core/releases/tag/v0.2.5)**). Линия `0.x` — экспериментальная; ломающие NMS-патчи между минорными ожидаемы. Артефакт: `eturlia-1.21.1-neoforge-21.1.248.jar`.

## Структура репозитория

| Путь | Назначение |
|------|------------|
| `patches/server`, `patches/api` | Folia + NeoForge/Eturlia патчи |
| `build-data/eturlia-core` | launch handler, region guard, mods hygiene, mixins ядра |
| `build-data/eturlia-server-templates` | `EturliaServer` |
| `build-data/eturlia-neoforge-shims` | compile-time stubs (документация) |
| `neoforge/` | extras, resources, coremods |
| `compat/` | опциональные compat-модули |
| `docs/MODDER_POLICY.md` | whitelist / unsupported / region API |
| `docs/PACK_COMPAT_ASIC_2026-08.md` | аудит списка модов + Folia-плагины |
| `docs/RELEASE_NOTES_0.2.5.md` | релиз-ноты v0.2.5 |
| `.github/workflows/eturlia-ci.yml` | applyPatches + jar + headless smoke |

## Апстрим и лицензии

- [PaperMC/Folia](https://github.com/PaperMC/Folia) · [Paper](https://github.com/PaperMC/Paper) · [NeoForge](https://github.com/neoforged/NeoForge)
- Патчи Folia/Paper: [`PATCHES-LICENSE`](./PATCHES-LICENSE)
- Код NeoForge — LGPL / заголовки апстрима

## Статус патчей

Активные server-патчи: Folia `0001`–`0019`, NeoForge/Eturlia `0020`–`0099` (ASIC BLOCK bridges 0084–0086, amendments/TF/Moonlight/quality_food 0087–0093, Main.main+LevelStorageSource для libjf/WorldWeaver 0094, datapack/Create/Sable, сетевые идентификаторы модового контента 0097, совместимость с модами на больших сборках 0098–0099).  
Черновики остальных WIP — в `patches/server-wip/`.

---

<details open>
<summary><strong>English</strong></summary>

<div id="english"></div>

# English

> [!WARNING]
> Experimental. **Not for production.** Back up worlds.
> Many mods assume a single Vanilla/Forge server thread and break under Folia’s region threading.

## What Eturlia is

**Eturlia** is a single fat-jar server kernel that runs two stacks together:

| Layer | Role |
|-------|------|
| **[Folia](https://github.com/PaperMC/Folia)** | Paper fork: the world is split into independent tick regions across CPU cores |
| **[NeoForge](https://github.com/neoforged/NeoForge) 21.1.248** | Mod loader (FancyModLoader 4.0.43): `mods/`, RegisterEvent, lifecycle, datapacks, mixins |

The goal is Folia’s multi-core scaling without giving up the NeoForge ecosystem (Create, Farmers Delight, and similar packs), as Folia↔NeoForge runtime gaps are closed.

| | |
|---|---|
| **Artifact** | `eturlia-1.21.1-neoforge-21.1.248.jar` |
| **Launch target** | `eturliaserver` |
| **Entry point** | `eturlia.EturliaServer` → `org.bukkit.craftbukkit.Main` (via `EturliaServerLaunchHandler`) |
| **Release** | [v0.2.5](https://github.com/eturnercus/Core/releases/tag/v0.2.5) |

## How it works

### Boot (runtime)

```text
java -jar eturlia-1.21.1-neoforge-21.1.248.jar
        │
        ▼
 Eturlia launcher
   · extract nested libs (Folia, FML, NeoForge, runtime)
   · prepare classpath and launch services
        │
        ▼
 ModLauncher / FancyModLoader
   · scan mods/
   · bootstrap NeoForge, coremods, mixins
        │
        ▼
 Folia / Paper server
   · regions, chunks, entities, plugins (folia-supported)
        │
        ▼
 NeoForge hooks in NMS
   · RegisterEvent, game lifecycle, datapack reload, API extensions
```

1. **Launcher** unpacks nested libraries from the fat jar and hands off to ModLauncher.
2. **FML** discovers mods, runs discovery / loading / common setup, and applies Eturlia coremods (`main` → `EturliaServer` plus support transforms).
3. **Folia** ticks the world by region: neighboring chunks may run on different threads; cross-region work needs the correct schedulers.
4. **Core patches** teach Folia/Paper the APIs and behaviors NeoForge mods expect (registries, food/crafting/fire, reload, entity-list facades, and so on).

### Build (dev)

1. **paperweight** applies `patches/api` and `patches/server` (upstream Folia + the Eturlia/NeoForge layer) onto Paper.
2. **Shims** (`build-data/eturlia-neoforge-shims`) are reference stubs only. The build does **not** use them: Folia-Server compiles against the published NeoForge universal jar (`compileOnly` in `build.gradle.kts`).
3. Runtime embeds published **NeoForge universal 21.1.248** — the same major line as target mods.
4. Optional `eturlia-compat-create` / `eturlia-compat-sable` modules cover narrow region bridges; they do not replace gaps in the core itself.

### Why mods can still fail

The FML boot path already works. The current work layer is **runtime gaps**: a mod calls an API or assumes single-threaded NMS that is missing or behaves differently under Folia. Typical failure classes:

- holders / DeferredHolder / RegisterEvent ordering;
- Folia entity tick list / section managers vs vanilla structures;
- remapping / refmap issues in mixin mods;
- single-thread assumptions in Create, Moonlight, and related libraries.

## Versions

| Component | Version |
|-----------|---------|
| Minecraft | 1.21.1 |
| NeoForge | **21.1.248** |
| FancyModLoader | 4.0.43 |
| Paper upstream | `84281cee…` (Folia `dev/1.21.1`) |
| Java | **21** |

## Build

Requires a **Git clone** (not a ZIP), **JDK 21**, and PaperMC + NeoForged Maven access.

```bash
./gradlew applyPatches
./gradlew :folia-server:build
./gradlew :folia-server:eturliaStandaloneJar
```

Output: `build/libs/eturlia-1.21.1-neoforge-21.1.248.jar`

```bash
./patch.sh   # applyPatches
./rb.sh      # rebuildServerPatches
```

## Run

```bash
java -Xmx8G -jar build/libs/eturlia-1.21.1-neoforge-21.1.248.jar --nogui
# or the jar from GitHub Releases (v0.2.5+)
```

The jar is a wrapper: it unpacks the libraries and starts a **child JVM** that runs the
server. The wrapper's JVM options (`-Xmx`, `-XX:*`, `-Deturlia.*`) are forwarded to that
child, and terminating the wrapper (SIGTERM / Ctrl+C) shuts the server down with it.
Accept `eula.txt` on first boot. NeoForge mods go in `mods/`. Plugins need `folia-supported: true`.  
Console defaults to **calm** (no green INFO). Override with `-Deturlia.console.color=full|off`.  
Plugins that declare `libraries:` in `plugin.yml` may fail Maven resolve under ModLauncher; plugins without `libraries:` load fine.

### Mod pack — current process (not “delete the mod”)

Eturlia closes Folia↔NeoForge gaps with **kernel patches**. Do not strip ASIC content mods just to boot.

| Action | When |
|--------|------|
| Install jar **v0.2.5+** (patches through **0094**) | libjf / respackopts / WorldWeaver no longer FATAL on `Main` |
| **Upgrade** Lithostitched to **≥ 1.7.13** | Beta `1.7.10+beta4` is the same mod, broken jar; gate aborts boot |
| Leave `spark-neoforge` / Arclight sable patch | Kernel **soft-skips** → `*.jar.eturlia-skipped` (file kept; bundled `/spark` works). Prefer `arclight_sable_patch-*-eturlia-shim.jar` for Sable |
| Re-download `easy_npc`; drop `.jar1` / `.bak*` | Corrupt/junk files, not a kernel gap |
| BetterEnd + BCLib + WorldWeaver | Need **wunderlib**; heavy worldgen = RISK on Folia |
| Optimizers (FerriteCore, etc.) | **Not required** for boot |

Audit: [`docs/PACK_COMPAT_ASIC_2026-08.md`](./docs/PACK_COMPAT_ASIC_2026-08.md) · smoke: [`docs/SMOKE_ASIC_2026-08-07.md`](./docs/SMOKE_ASIC_2026-08-07.md) · notes: [`docs/RELEASE_NOTES_0.2.5.md`](./docs/RELEASE_NOTES_0.2.5.md).

## Smoke status / whitelist matrix

| Set | Status | Compatibility |
|-----|--------|---------------|
| Empty / light stack (Cloth, Curios, GeckoLib, JEI, …) | SML + Folia `Done` | whitelist |
| Farmers Delight (+ Cloth) | SML + `Done` | whitelist |
| Create 6.0.10 | SML + `Done` + client join holds; tick gaps under load still tracked | whitelist (best-effort) |
| Moonlight Lib | SML + `Done` + worlds; SoftFluid/FluidType bridged | whitelist (best-effort) |
| ASIC core (FD, CreativeCore, Farm&Charm, amendments, TF, Supplementaries, …) | `Done` + worlds (0084–0093) | whitelist (best-effort) |
| libjf + respackopts (+ litho ≥1.7.13) | MainMixin OK + `Done` (0094) | whitelist |
| Bukkit/Paper plugins (`folia-supported`) | discovery + load | whitelist (no `libraries:`) |
| Bundled spark | `spark tps` OK; TPS from Folia global tick | whitelist |
| FerriteCore / full ASIC gameplay | not certified | **at your own risk** |

Modder policy: [`docs/MODDER_POLICY.md`](./docs/MODDER_POLICY.md).

### Crash reports

- Vanilla/Paper → `crash-reports/`
- Region-annotated Eturlia reports → `eturlia-crash-reports/` (`-Deturlia.crash.dir=…`),
  written by `eturlia.EturliaServer`, which the launch handler installs before handing
  control to `org.bukkit.craftbukkit.Main`.

### Core config — `config/eturlia.yml`

Everything Eturlia-specific is configured here; the file is created on first boot. Paper /
Spigot / Bukkit settings are **not duplicated** — see the `reference:` section for where those
live.

| Section | What it controls |
|---------|------------------|
| `threads` | Folia region tick threads and grid exponent, chunk worker / IO threads, chat pool |
| `jvm` | flags for the **child JVM**: `-Xmx`/`-Xms`, `Paper.WorkerThreadCount`, `Paper.IOThreadCount`, arbitrary extra args |
| `chunks` | view/simulation distance, chunk send and generation limits, autosave |
| `region` | region guard and event validation mode (`STRICT`/`WARN`/`PERMISSIVE`) |
| `validation` | region-thread checking in the coremod hooks, strict mode |
| `logging` | core diagnostics file, whether warnings also print to the console |
| `crash` | region-annotated report directory, whether to install the handler |
| `hygiene` | what to do with incompatible jars in `mods/` (`skip`/`warn`/`off`) |
| `mods` | minimum Lithostitched version, strict enforcement of the compatibility manifest |
| `lod`, `spark`, `watchdog`, `console` | LOD bridge, spark, watchdog, console colour |

Thread counts live in two places because they apply at different times:
`threads.region-tick-threads` and `region-grid-exponent` are pushed into Folia's
`GlobalConfiguration.threadedRegions`, while `jvm.worker-threads` / `jvm.io-threads` go on the
child JVM's command line — Moonrise reads those before any config is parsed.

### Console and logs

On startup the core prints the ETURLIA banner and a few status lines. Eturlia warnings and
errors stay on the console but are **strictly one line each**:

```text
[Eturlia] WARN config/eturlia.yml has _version=2 but this build expects 3
[Eturlia] ERROR mod compatibility check failed — IllegalStateException: c2me is excluded
```

The full record, stack trace included, goes to **`logs/eturlia.log`**. The server's own Log4j2
logging is untouched.

| Flag | Effect |
|------|--------|
| `-Deturlia.console.errors=off` | keep Eturlia warnings off the console entirely (still logged to the file) |
| `-Deturlia.console.color=off` | no ANSI colour (`NO_COLOR` is honoured too) |
| `-Deturlia.log.file=<path>` | move the diagnostics file |

### Player join: what works

**A NeoForge client joins the server and stays in the world** — verified with a real client
(NeoForge 21.1.248, headless, auto-connect), not a protocol emulator:

```
There are 1 out of maximum 20 players online
```

The handshake needed 13 integration points that NeoForge adds to vanilla classes with its own
patches and that patch `0095` was missing. Every one was found from an actual join attempt,
not guessed: `PacketEncoder.getProtocolInfo()`, per-phase payload codecs,
`MinecraftServer.execute()` routed to Folia's global region (Folia has no main thread and
throws there by design), `ConfigurationTask.Type(ResourceLocation)`,
`PacketFlow.isClientbound()`, `IServerConfigurationPacketListenerExtension`,
`CustomPacketPayload.toVanillaClientbound()`, `FeatureFlagRegistry.getAllFlags()`,
`SavedData.Factory(Supplier, BiFunction)`, codec selection by payload class rather than id,
`RegistryFriendlyByteBuf` carrying the connection type, and the bundle unpacker ordered ahead
of NeoForge's packet splitter.

With mods installed one more thing was needed: modded blocks never reached
`Block.BLOCK_STATE_REGISTRY`, so reading a chunk killed the client with `No value with id -1` —
see [Create](#create-current-state).

A failed negotiation or configuration now drops that one connection instead of the process:
Folia halts the server when a region tick throws, and a malformed client exploited that twice.

### Create: current state

Create **loads, no longer crashes the server, and can be joined**. Verified on the test rig with
a real headless NeoForge client: `EturliaTester joined the game`, `join=1 left=0`. Two unrelated
things had to be fixed.

**1. A region tick died.** Create's `ProjectileUtilMixin` wraps `Entity.canRiderInteract()`
inside `ProjectileUtil`; NeoForge adds both the method and the call site, and without them the
injection failed hard, which killed a region tick and with it the whole process. Both are in
place now.

**2. The client was dropped a couple of seconds after joining.**

```
IllegalArgumentException: No value with id -1
  at IdMap.byIdOrThrow → LinearPalette.read → LevelChunkSection.read
```

A diagnostic on the palette write named the culprit outright: `Block{create:scoria}` went on the
wire with id `-1`.

`Block.BLOCK_STATE_REGISTRY` is filled exactly once, by `Blocks`' static initialiser, while
vanilla content is bootstrapped. Upstream NeoForge keeps it current from there on through
`NeoForgeRegistryCallbacks.BlockCallbacks`, installed on `BuiltInRegistries.BLOCK` — a patch
Eturlia never carried. **No modded block ever entered the map**, so the chunk encoder wrote `-1`
for any modded block in a loaded chunk.

Rather than porting NeoForge's whole registry-callback API, the map is topped up once, right
after `ServerModLoader` — `Block.eturlia$rebuildStateIds()`. The registry is frozen by then and
iteration runs in id order, which is exactly the order the client rebuilds its own map in, so
both sides agree. On Create + Aeronautics + lithostitched + Sable that is 38,909 states.

The same treatment restores two more derived maps the upstream callbacks maintain:

- `Item.BY_BLOCK` (`Item.eturlia$rebuildBlockItemMap()`) — without it `Item.byBlock()` answers
  `AIR` for every modded block, breaking block picking, drops and some recipe lookups;
- `PoiTypes.TYPE_BY_STATE` (`PoiTypes.eturlia$rebuildStateMap()`) — without it modded points of
  interest never bind to their block states.

The palette write now permanently reports such a `-1` as a single line naming the block, instead
of leaving the client with an unattributable `No value with id -1`.

### Large packs: why a mod may refuse to load

Bringing up the 90-mod production pack (NoteBuns) showed that **the usual culprit is Paper, not
NeoForge**. Folia inherits Paper's rewrites of vanilla methods; a mod aims at the vanilla shape,
Mixin calls the mismatch fatal, and **one mod stops the entire server from starting**.

Fixed in the core:

| Symptom | Cause | Fix |
|---|---|---|
| `world_folder.MainMixin from mod wover ... (0/1) succeeded` | `net.minecraft.server.Main` had **two** methods named `main`; mods target Mojang's by name and Mixin resolved the name to the CraftBukkit overload | the CraftBukkit entry is now `eturlia$bootFromOptions`; there is exactly one `main` again |
| `@Shadow field existingFileHelper was not located in TagsProvider` | NeoForge adds that field; the patch was never carried | field added (always `null` on a server — data generation does not run) |
| `@ModifyArg ... knockback(DDD)V ... Scanned 0 target(s)` | Paper rewrote the call inside `hurt` to the five-argument form for `EntityKnockbackEvent` | the call is Mojang-shaped again; attacker and cause travel in a field, so Paper's event is unchanged |
| `AbstractMethodError: ... 'abstract int getIdFor'` in `Block.<init>` | Paper made `getIdFor(T)` **abstract** on `Property` for its state table. Any mod with its own `Property` was compiled against vanilla, where the method does not exist, and died in `Block`'s constructor — so it could not register anything at all | the method is no longer abstract: it defaults to the value's index in `getPossibleValues()`, cached lazily because it sits on `setValue`'s hot path. Vanilla subclasses still override it |
| `NullPointerException: ... "snapshots" is null` during mod loading | upstream NeoForge takes `GameData.vanillaSnapshot()` in its `Bootstrap` patch; Eturlia calls stock `Bootstrap`. With no snapshot the registry rollback throws, **hiding the mod failure it was reporting** | the snapshot is taken between `Bootstrap.validate()` and `ServerModLoader.load()` |
| `Ingredient does not have member field 'MapCodec MAP_CODEC_NONEMPTY'` | NeoForge's own `SizedIngredient` reads an ingredient inline and takes that codec from `Ingredient`; the field comes from a NeoForge patch we lacked. `SizedIngredient` is common currency in NeoForge recipes | field added, built from the existing `Ingredient.Value.MAP_CODEC` |

Not fixable — the mod has to go:

- **FerriteCore** — replaces the block-state neighbour map with its own while Paper already has
  one (`ZeroCollidingReferenceStateTable`), giving `NullPointerException: index_table is null` in
  `Blocks.<clinit>`. It is redundant on Paper/Folia, which made that memory saving already.
- **tt20** — rewrites the server tick loop, which Folia does not have in vanilla form.
- Mods whose injectors target a place where Paper replaced a call or added a local
  (`LVT ... has incompatible changes`): BetterEnd, bclib, WorldWeaver, letsdo-vinery, horseman
  and similar.

`EturliaMixinErrorHandler` is registered: an `InvalidMixinException` from a **third-party** mod
is downgraded to a warning naming that mod, instead of stopping the server. Failures in Eturlia,
Folia and NeoForge mixins stay fatal. `-Deturlia.mixin.errors=fatal` restores stock behaviour,
`=quiet` keeps the downgrade without the per-mixin line. This is a trade, not a repair: a mixin
that did not apply means the mod is missing part of itself.

`InjectionError` — the injection itself failing — cannot be intercepted, because Mixin only
consults error handlers for `InvalidMixinException`. Those mods must be disabled.

### Known limitations

- The `mods/` hygiene pass **renames** conflicting jars (`spark-*neoforge*`, the original
  `arclight_sable_patch`) to `*.jar.eturlia-skipped` on every boot. Opt out with
  `-Deturlia.mods.hygiene=warn` (report only) or `=off` (do not scan).

- `ServerTickEvent.Pre/Post` (and `LevelTickEvent`) are fired **per region tick**, not once
  per global tick, so listeners are invoked concurrently from every region thread. Mods with
  single-threaded assumptions in tick handlers will misbehave.
- `compat/eturlia-compat-create` and `compat/eturlia-compat-sable` are **skeletons**: every
  handler is a stub, no mixins are applied, dependencies are not pinned. They are not built
  by the root project and not covered by CI.
- The region guard / thread validator catch violations but are not yet wired into every NMS
  entry point — coverage is partial.

### Semver

Tags `vMAJOR.MINOR.PATCH` (currently **[v0.2.5](https://github.com/eturnercus/Core/releases/tag/v0.2.5)**). The `0.x` line is experimental; breaking NMS patches between minors are expected. Artifact: `eturlia-1.21.1-neoforge-21.1.248.jar`.

## Repository layout

| Path | Role |
|------|------|
| `patches/server`, `patches/api` | Folia + NeoForge/Eturlia patches |
| `build-data/eturlia-core` | launch handler, region guard, mods hygiene, core mixins |
| `build-data/eturlia-server-templates` | `EturliaServer` |
| `build-data/eturlia-neoforge-shims` | compile-time stubs (docs) |
| `neoforge/` | extras, resources, coremods |
| `compat/` | optional compat modules |
| `docs/MODDER_POLICY.md` | whitelist / unsupported / region API |
| `docs/PACK_COMPAT_ASIC_2026-08.md` | pack audit (mods + Folia plugins) |
| `docs/RELEASE_NOTES_0.2.5.md` | v0.2.5 release notes |
| `.github/workflows/eturlia-ci.yml` | applyPatches + jar + headless smoke |

## Upstream & license

- [PaperMC/Folia](https://github.com/PaperMC/Folia) · [Paper](https://github.com/PaperMC/Paper) · [NeoForge](https://github.com/neoforged/NeoForge)
- Folia/Paper patches: [`PATCHES-LICENSE`](./PATCHES-LICENSE)
- NeoForge code: upstream LGPL / file headers

## Patch status

Active server patches: Folia `0001`–`0019`, NeoForge/Eturlia `0020`–`0099` (ASIC BLOCK bridges 0084–0086, amendments/TF/Moonlight/quality_food 0087–0093, Main.main+LevelStorageSource for libjf/WorldWeaver 0094, datapack/Create/Sable, network ids for modded content 0097, large-pack mod compatibility 0098–0099).  
Remaining WIP drafts: `patches/server-wip/`.

</details>
