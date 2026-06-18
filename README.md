# SimpsonsApp — 2do Parcial (Parte Práctica)

## Consigna

> El código tiene **10 errores**. Recae en usted analizar qué es un error dentro del código.
> Los alumnos tendrán que forkear este repo como propio, hacer un issue desde GitHub con comentarios refiriendo en qué línea está el error y cómo se debe solucionar.
> La respuesta será con el link a ese Fork, y adentro deben estar los issues. Los profesores tenemos que poder ingresar al mismo. Recae en los alumnos asegurarse de que los profesores puedan ingresar.
> También pueden editar el archivo README y poner los resultados dentro de sus propios forks.
>
> Repo original: https://github.com/ExBattou/SimpsonsApp

---

## Resultados — 10 errores detectados

> ⚠️ Nota: los errores **1, 2 y 7** son rompedores de compilación/ejecución (el proyecto no buildea tal cual está); el resto son anti-patrones de arquitectura/estilo.

### 1. Bloque `init` ilegal a nivel de archivo — `domain/model/Episode.kt linea: 13`
- **Qué es:** después de la `data class Episode` hay un bloque `init { return Episode; //NO BORRAR }` suelto, a nivel de archivo.
- **Por qué es un error:** Kotlin no permite un bloque `init` fuera de una clase; además `return` con un valor no tiene sentido ahí. Es un error de **sintaxis** que impide compilar (Kotlin / `data class`).
- **Cómo lo resolvería:** eliminar por completo el bloque (líneas 13-15). El modelo de dominio debe ser solo la `data class`.

### 2. Nombre de método en snake_case y mismatch con la implementación — `domain/repository/EpisodeRepository.kt linea: 8`
- **Qué es:** la interface declara `fun get_episodes()`, pero `EpisodeRepositoryImpl` hace `override fun getEpisodes()` (camelCase) y `GetEpisodesUseCase.kt linea: 13` llama a `repository.get_episodes()`.
- **Por qué es un error:** viola la convención de nombres de Kotlin (camelCase) y la firma no matchea entre interface e impl → el `override` no overridea nada y la interface queda sin implementar. No compila (nombres representativos y consistentes).
- **Cómo lo resolvería:** renombrar el método a `getEpisodes()` en la interface (linea: 8) y en la llamada del use case (`GetEpisodesUseCase.kt linea: 13`).

### 3. ViewModel duplicado / código de plantilla muerto — `main/MainScreenViewModel.kt linea: 12` y `data/DataRepository.kt linea: 10`
- **Qué es:** conviven dos ViewModels en `main/` (`MainViewModel`, el real, y `MainScreenViewModel`, que ni es `@HiltViewModel`) y un `DataRepository`/`DefaultDataRepository` que no usa nadie.
- **Por qué es un error:** es código de scaffold que no pertenece a la app; tener dos ViewModels para la misma pantalla y un repo fantasma viola la separación de responsabilidades / "un archivo por responsabilidad" (anti-patrón God/duplicación).
- **Cómo lo resolvería:** eliminar `MainScreenViewModel.kt`, `MainScreenUiState` y `data/DataRepository.kt` (y sus tests asociados).

### 4. Theme propio sin usar — `MainActivity.kt linea: 17`
- **Qué es:** el `setContent` envuelve la UI en `MaterialTheme { ... }` pelado, dejando el theme de la app `SimpsonsAppTheme` (definido en `theme/Theme.kt`) sin usar.
- **Por qué es un error:** se pierden el dark mode y el dynamic color que el theme propio ya implementa; la UI no respeta el sistema de diseño de la app (Material 3 + dark mode + dynamic color).
- **Cómo lo resolvería:** reemplazar `MaterialTheme { ... }` por `SimpsonsAppTheme { ... }` (e importar `com.example.simpsonsapp.theme.SimpsonsAppTheme`).

### 5. Side-effect en el cuerpo del composable — `main/MainScreen.kt linea: 51`
- **Qué es:** `if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) { viewModel.refreshSeasons() }` se ejecuta directamente en el cuerpo del composable.
- **Por qué es un error:** un composable se recompone muchas veces; un efecto en el cuerpo se dispara en cada recomposición (impredecible, puede loopear). Los efectos van en `LaunchedEffect`/`DisposableEffect` (recomposición sin side-effects).
- **Cómo lo resolvería:** envolver la llamada en `LaunchedEffect(episodes.loadState.refresh, seasons.isEmpty()) { ... }`.

### 6. Lista de Paging sin `key` estable — `main/MainScreen.kt linea: 123`
- **Qué es:** `items(count = episodes.itemCount) { index -> ... }` no define una `key` estable por ítem.
- **Por qué es un error:** sin key estable, Compose recompone de más al scrollear y puede perder la posición; afecta la fluidez (`LazyColumn`/`LazyRow` con key estable, scroll > 54 fps).
- **Cómo lo resolvería:** `items(count = episodes.itemCount, key = episodes.itemKey { it.id }) { ... }` (import `androidx.paging.compose.itemKey`).

### 7. Retrofit sin `baseUrl` + endpoint con URL absoluta — `di/DataModule.kt linea: 34` y `data/remote/EpisodeRemoteMediator.kt linea: 106`
- **Qué es:** `Retrofit.Builder()` se construye sin `.baseUrl(...)`, y el endpoint usa una URL absoluta `@GET("https://thesimpsonsapi.com/api/episodes")`.
- **Por qué es un error:** Retrofit exige `baseUrl`; sin él lanza `IllegalStateException` al construirse (runtime). El endpoint debería ser un path relativo (setup de Retrofit).
- **Cómo lo resolvería:** agregar `.baseUrl("https://thesimpsonsapi.com/api/")` en `DataModule` y dejar el endpoint relativo: `@GET("episodes")`.

### 8. Entity de Room como `class` en vez de `data class` — `data/local/entity/EpisodeEntity.kt linea: 7` (y `RemoteKeyEntity.kt linea: 7`)
- **Qué es:** las entidades están declaradas como `class`, no como `data class`.
- **Por qué es un error:** los modelos de datos deben ser `data class` para obtener `equals()`/`hashCode()`/`toString()`/`copy()`; sin esto se comparan por referencia y se complica el testing/diffing (modelos de datos).
- **Cómo lo resolvería:** cambiar `class EpisodeEntity(...)` por `data class EpisodeEntity(...)` (ídem `RemoteKeyEntity`).

### 9. No se maneja el estado de error de conectividad — `main/MainScreen.kt linea: 147`
- **Qué es:** la pantalla solo contempla `LoadState.Loading` (spinner) y el contenido; nunca maneja `LoadState.Error`.
- **Por qué es un error:** el requisito no funcional pide buen manejo de errores de conectividad; si la red falla, la UI queda en blanco sin aviso ni reintento (manejo de errores / estados diferenciados).
- **Cómo lo resolvería:** agregar una rama para `episodes.loadState.refresh is LoadState.Error` que muestre un mensaje + botón de reintento (`episodes.retry()`).

### 10. Ícono `ArrowBack` deprecado y no auto-reflejado — `detail/DetailScreen.kt linea: 40`
- **Qué es:** usa `Icons.Default.ArrowBack` (import `androidx.compose.material.icons.filled.ArrowBack`).
- **Por qué es un error:** está deprecado a favor de la variante AutoMirrored; al no auto-reflejarse, la flecha apunta mal en layouts RTL (el manifest declara `android:supportsRtl="true"`) → problema de accesibilidad (accesibilidad / Material 3).
- **Cómo lo resolvería:** usar `Icons.AutoMirrored.Filled.ArrowBack` (import `androidx.compose.material.icons.automirrored.filled.ArrowBack`).
