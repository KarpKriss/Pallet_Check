# Pallet Check Tool — dokumentacja techniczna dla osób edytujących HTML

Wersja dokumentu: 1.0  
Zakres: jednoplikowy tool HTML `Pallet_Check_Tool.html`  
Cel dokumentu: pomóc osobie technicznej lub power-userowi bezpiecznie zmienić logikę, teksty, wygląd i zachowanie narzędzia bez przebudowy całego projektu.

---

## 1. Czym jest ten tool

`Pallet Check Tool` to lokalnie działające narzędzie HTML do kontroli zawartości palety. Użytkownik importuje dane z systemu, skanuje lub wpisuje Pallet ID, a następnie skanuje SKU albo EAN. Tool sprawdza, czy skanowany kod należy do aktywnej palety, pokazuje postęp i loguje błędne skany.

Najważniejsze cechy:

- działa jako pojedynczy plik HTML,
- nie wymaga backendu, instalacji ani bazy danych,
- przechowuje dane lokalnie w przeglądarce przez `localStorage`,
- pozwala dodać wiele batchy danych,
- pomija duplikaty w importowanych batchach,
- obsługuje dwa tryby kompletacji: po SKU i po ilości,
- rozpoznaje SKU oraz EAN,
- zapisuje historię ukończonych palet,
- zapisuje log błędnych skanów,
- posiada panele: Data Import, Settings, Instructions oraz dolne logi/narzędzia.

---

## 2. Najważniejsza zasada edycji

Nie zmieniaj losowo całego pliku. Ten tool ma kilka warstw:

1. **HTML** — struktura ekranu, panele, przyciski, tabele.
2. **CSS** — wygląd, layout, animacje, sticky elementy, kolory.
3. **JavaScript** — import danych, logika skanowania, localStorage, historia, walidacja.
4. **EAN map** — wbudowana baza mapowania EAN → SKU.

Najbezpieczniejszy sposób zmian:

1. Zrób kopię pliku `Pallet_Check_Tool.html`.
2. Zmień jedną rzecz naraz.
3. Uruchom plik lokalnie w Chrome/Edge.
4. Wykonaj test importu danych.
5. Wykonaj test palety znalezionej i nieznalezionej.
6. Wykonaj test skanowania poprawnego SKU/EAN i błędnego SKU/EAN.
7. Dopiero potem wrzuć zmianę do repozytorium.

---

## 3. Gdzie szukać najważniejszych elementów

W pliku używaj wyszukiwania `Ctrl + F` po nazwach funkcji lub ID elementów.

### 3.1. Konfiguracja stanu

Szukaj:

```js
const STORAGE_KEY
const DEFAULT_MAPPING
const DEFAULT_STATE
```

To jest konfiguracja startowa narzędzia.

Najważniejsze pola:

```js
const STORAGE_KEY = "pallet-check-tool-state-v2";
const DEFAULT_MAPPING = { qty: 6, sku: 8, pallet: 26, container: 27 };
```

`DEFAULT_MAPPING` określa domyślne kolumny importu:

- `qty` — kolumna ilości,
- `sku` — kolumna SKU,
- `pallet` — kolumna Pallet ID,
- `container` — kolumna Container ID.

Jeżeli system zmieni układ kolumn, można zmienić te wartości albo pozwolić użytkownikowi zmienić je w panelu `Settings`.

`DEFAULT_STATE` przechowuje domyślne wartości całego narzędzia, np. listę rows, batch count, historię, ustawienia skanowania i datę ostatniego importu.

---

## 4. Import danych

### 4.1. Funkcja parsowania importu

Szukaj:

```js
function parseRawData(rawText)
```

Ta funkcja bierze tekst wklejony z Excela/systemu i zamienia go na rekordy:

```js
{
  sku,
  qty,
  palletId,
  containerId
}
```

Aktualnie parser:

- dzieli dane po tabulatorze,
- czyta kolumny według `state.mapping`,
- pomija puste Pallet ID,
- pomija wiersze, gdzie SKU zawiera tekst `PALLET`.

Jeżeli format danych zmieni się z tabulatorów na CSV, separator trzeba zmienić tutaj.

### 4.2. Dodawanie batcha

Szukaj:

```js
function appendBatch(rows, sourceLabel)
```

Ta funkcja:

- sprawdza, czy batch ma prawidłowe rekordy,
- porównuje je z już zaimportowanymi rekordami,
- pomija duplikaty,
- dopisuje tylko unikalne wiersze,
- zwiększa `batchCount`,
- zapisuje datę ostatniego importu w `state.lastImportDate`,
- zapisuje stan do `localStorage`.

Duplikat jest rozpoznawany przez funkcję:

```js
function buildRowKey(row)
```

Aktualny klucz duplikatu:

```js
[row.sku, row.qty, row.palletId, row.containerId].join("||")
```

Jeżeli w przyszłości duplikat ma być rozpoznawany inaczej, np. tylko po `palletId + sku`, zmień `buildRowKey`.

---

## 5. LocalStorage i dane lokalne

Szukaj:

```js
function saveState()
function loadState()
```

Tool zapisuje dane w przeglądarce pod kluczem:

```js
pallet-check-tool-state-v2
```

Zapis obejmuje m.in.:

- załadowane rekordy,
- liczbę batchy,
- historię ukończonych palet,
- log błędnych skanów,
- ustawienia kolumn,
- tryb skanowania,
- datę ostatniego importu.

Jeżeli chcesz wymusić reset danych u wszystkich użytkowników po wdrożeniu dużej zmiany struktury danych, zmień `STORAGE_KEY`, np. na:

```js
pallet-check-tool-state-v3
```

Uwaga: zmiana klucza sprawi, że przeglądarka nie wczyta starego zapisu.

---

## 6. Wyszukiwanie palety

Szukaj:

```js
function searchPallet()
```

To jedna z najważniejszych funkcji w całym narzędziu.

Robi kolejno:

1. Odczytuje wartość z pola Pallet ID.
2. Sprawdza, czy są załadowane dane.
3. Sprawdza, czy obecna paleta nie jest niedokończona.
4. Obsługuje wyszukiwanie po suffixie, np. `*3128`.
5. Szuka dokładnego dopasowania w `palletId`.
6. Jeżeli nie znajdzie palety, szuka dopasowania w `containerId`.
7. Jeżeli nie znajdzie nic, sprawdza, czy dzisiaj był import danych.
8. Jeżeli dzisiaj nie było importu, pokazuje komunikat o braku dzisiejszych danych.
9. Jeżeli import dzisiaj był, zapisuje skan jako `Wrong Scan`.

### 6.1. Komunikat „brak dzisiejszego importu”

Szukaj:

```js
function hasImportedToday()
```

oraz w `searchPallet()` fragmentu:

```js
if (!hasImportedToday())
```

Tam można zmienić tekst komunikatu dla użytkownika, np. gdy paleta nie została znaleziona, a użytkownik nie zaimportował jeszcze danych danego dnia.

---

## 7. Wyniki palety i tabela pozycji

Szukaj:

```js
function renderResults(rows, palletId, options = {})
function renderCurrentMeta(rows, palletId, labelText)
function renderRowTable(rows)
```

### 7.1. `renderResults`

Odpowiada za pokazanie wyników po wyszukaniu palety lub SKU.

Jeżeli `rows.length === 0`, funkcja pokazuje pusty stan, np. `Pallet Not Found` albo inny tekst przekazany w `options`.

### 7.2. `renderCurrentMeta`

Liczy i wyświetla metadane aktualnej palety:

- liczba rekordów,
- suma quantity,
- liczba containerów.

### 7.3. `renderRowTable`

Buduje wiersze tabeli SKU/EAN/Quantity/Pallet ID/Container ID.

Jeżeli chcesz zmienić kolejność kolumn, dodawać nowe kolumny albo usunąć kolumny, zmiany będą potrzebne w dwóch miejscach:

1. nagłówki tabeli w HTML,
2. zawartość wiersza w `renderRowTable`.

---

## 8. Logika skanowania SKU/EAN

Najważniejsze funkcje:

```js
function startScanWorkflow(rows, palletId)
function handleSkuScan()
function updateScanStatus()
function evaluatePalletStatus()
function sumRequirementCounts()
function sumScannedCounts()
```

### 8.1. `startScanWorkflow`

Uruchamia proces skanowania dla aktywnej palety.

Tworzy mapę wymagań:

```js
scanState.requirements
```

Dla trybu `sku` każde SKU wymaga jednego skanu.  
Dla trybu `quantity` SKU wymaga tylu skanów, ile wynosi ilość w danych.

### 8.2. `handleSkuScan`

Obsługuje pojedynczy skan SKU/EAN.

Robi kolejno:

1. Pobiera wartość z inputa SKU/EAN.
2. Zamienia EAN na SKU przez `resolveSkuOrEan`.
3. Sprawdza, czy SKU należy do aktywnej palety.
4. Jeżeli nie należy, zapisuje błąd w `Wrong Scan Log`.
5. Jeżeli należy, zwiększa licznik skanów.
6. Aktualizuje status palety.
7. Odtwarza dźwięk sukcesu lub zakończenia.
8. Czyści input i ustawia focus.

### 8.3. Tryby kompletacji

Szukaj:

```js
scanMode: "sku"
```

oraz:

```js
if (scanState.mode === "quantity")
```

Aktualne tryby:

- `sku` — każde SKU trzeba potwierdzić raz,
- `quantity` — trzeba potwierdzić liczbę sztuk.

Jeżeli chcesz dodać trzeci tryb, np. „complete by line”, trzeba zmienić:

- select w HTML,
- `DEFAULT_STATE.scanMode`,
- `startScanWorkflow`,
- teksty progressu w `updateScanInstructionAndProgress`,
- ewentualnie eksport/historię.

---

## 9. EAN → SKU

Najważniejsze funkcje:

```js
function rebuildEanMaps()
function resolveSkuOrEan(value)
function parseEanCsv(rawText)
function reloadEanDatabase()
```

EAN map jest przechowywana w obiekcie:

```js
window.EAN_SKU_MAP
```

`resolveSkuOrEan` próbuje rozpoznać kod jako:

1. pełny EAN,
2. skrócony EAN bez pierwszych dwóch znaków,
3. zwykły SKU.

Jeżeli użytkownik ładuje własny plik CSV EAN, dane trafiają do:

```js
state.customEanMap
```

Jeżeli chcesz zmienić format CSV EAN, zmień `parseEanCsv`.

---

## 10. Wrong Scan Log

Najważniejsze funkcje:

```js
function addWrongScan(type, value, details)
function renderWrongScanLog()
function renderWrongScanSummary()
```

`addWrongScan` dodaje nowy wpis na początek listy i zapisuje go do `localStorage`.

Aktualny limit historii:

```js
state.wrongScans = state.wrongScans.slice(0, 100);
```

Jeżeli chcesz trzymać więcej wpisów, zmień `100` na większą wartość.

Widok logu ma paginację po 20 pozycji. Odpowiada za to:

```js
const ACTIVITY_PAGE_SIZE = 20;
function renderActivityPagination(...)
```

---

## 11. Completed Pallets History

Najważniejsze funkcje:

```js
function logCompletedPallet()
function renderCompletedHistory()
function exportCompletedHistoryCsv()
```

`logCompletedPallet` dopisuje paletę do historii po pełnym zakończeniu procesu.

Aktualny limit historii:

```js
state.completedHistory = state.completedHistory.slice(0, 100);
```

Historia również ma paginację po 20 pozycji. Jeżeli chcesz zmienić liczbę pozycji na stronie, zmień:

```js
const ACTIVITY_PAGE_SIZE = 20;
```

---

## 12. Komunikaty, popupy i warningi

Najważniejsze miejsca:

```js
function setStatus(message, tone)
function showWorkflowModal(type, title, message, kicker)
function ensureScannerAcknowledgement(message, tone)
function updateDuplicateWarning(rows)
```

Typowe `tone` statusu:

- `info`,
- `success`,
- `warn`,
- `error`.

Jeżeli chcesz zmienić tekst komunikatu po ukończeniu palety, szukaj:

```js
Pallet completed correctly
```

Jeżeli chcesz zmienić tekst o sprawdzeniu pozostałych SKU na palecie, szukaj:

```html
Before moving on, make sure there is no other SKU left on the pallet
```

---

## 13. UI, layout i CSS

Cały CSS jest w `<style>` w górnej części pliku HTML.

Najważniejsze klasy i obszary:

- `.hero` — tytuł i ikona narzędzia,
- `.side-tab-panel` — górne sticky zakładki Data Import / Settings / Instructions,
- `.tab-panel` — zawartość otwieranego panelu,
- `.search-card` — główny obszar pracy,
- `.scan-panel` — panel skanowania SKU/EAN,
- `.table-wrap` — tabela pozycji palety,
- `.meta-grid` i `.meta-card` — statystyki po prawej stronie,
- `.stacked-panels` — dolne przyciski Wrong Scan Log / Completed History / Advanced Tools,
- `.workflow-modal` — popup zakończenia palety,
- `.activity-pagination` — paginacja logów.

### 13.1. Kolory

Główne kolory są na początku CSS w zmiennych `:root`.

Szukaj:

```css
:root
```

Najważniejsze zmienne:

```css
--accent
--accent-strong
--accent-soft
--success
--warn
--danger
--ink
--muted
--line
```

Jeżeli chcesz zmienić główny kolor toola, najpierw zmień `--accent` i `--accent-strong`, a dopiero potem poprawiaj szczegóły.

### 13.2. Górne zakładki

Szukaj:

```css
.side-tab-panel
.tab-btn
.topDrawerIn
```

Zakładki są fixed przy górnej lewej krawędzi i otwierają panel pod sobą. Hover jest animowany przez `transform`, `box-shadow` i gradienty.

### 13.3. Tabela pozycji

Szukaj:

```css
.table-wrap
thead th
tbody td
```

Kolumny `Pallet ID` i `Container ID` są zwężone i wycentrowane przez selektory:

```css
th:nth-child(4)
td:nth-child(4)
th:nth-child(5)
td:nth-child(5)
```

Jeżeli dodasz lub usuniesz kolumny, trzeba zaktualizować te numery.

### 13.4. Podświetlenie skanowanych wierszy

Szukaj:

```css
scan-row-progress
scan-row-done
scan-row-latest
```

Wiersz ma zmienną CSS:

```css
--scan-fill
```

W trybie quantity wypełnienie wiersza zależy od procentu zeskanowanej ilości.

---

## 14. Dźwięki

Szukaj:

```js
function beep(type)
```

Obsługiwane typy:

- `success`,
- `error`,
- `complete`.

Każdy typ ma sekwencję częstotliwości i długości tonu. Jeżeli dźwięki są zbyt agresywne, zmniejsz wartości `gain` albo skróć `duration`.

---

## 15. Focus skanera

Najważniejsze funkcje:

```js
function getActiveInput()
function queueFocusActiveInput()
function queueFocusSkuInput()
```

Tool automatycznie kieruje focus do właściwego pola:

- Pallet ID na początku,
- SKU/EAN po aktywacji palety,
- Next Pallet ID po zakończeniu palety.

Jeżeli użytkownicy skarżą się, że focus „ucieka”, sprawdź te funkcje oraz ustawienia:

```js
state.scanLock
state.manualScanEntry
state.manualPalletEntry
```

---

## 16. Eksport CSV

Szukaj:

```js
function exportCsv(filename, rows, headers)
function exportCurrentView()
function exportCompletedHistoryCsv()
```

`exportCsv` jest funkcją bazową. Pozostałe funkcje przygotowują konkretne dane do eksportu.

Jeżeli chcesz dodać nowy eksport, najprościej stworzyć nową funkcję podobną do `exportCurrentView` i dodać przycisk w `Advanced Tools`.

---

## 17. Jak dodać nową instrukcję dla użytkownika

Szukaj w HTML:

```html
<div id="instructionsTab" class="tab-panel">
```

Każdy krok instrukcji jest kartą:

```html
<div class="field-card">
  <strong>1. Load system data</strong>
  <div class="footnote">...</div>
</div>
```

Aby dodać nowy krok, skopiuj jedną kartę i zmień numer oraz tekst.

---

## 18. Jak bezpiecznie dodać nową funkcję

Rekomendowany schemat:

1. Dodaj HTML przycisku lub pola.
2. Dodaj `const element = document.getElementById(...)` w sekcji referencji DOM.
3. Dodaj funkcję JS.
4. Podepnij event listener w `bindEvents()`.
5. Zapisz dane do `state`, jeżeli mają przetrwać odświeżenie.
6. Wywołaj `saveState()` po zmianie danych.
7. Zaktualizuj UI przez osobną funkcję renderującą.

Nie mieszaj logiki biznesowej bezpośrednio w event listenerze, jeżeli zmiana jest większa. Event listener powinien tylko wywoływać funkcję.

---

## 19. Typowe zmiany i gdzie je zrobić

### Zmienić domyślne kolumny importu

Szukaj:

```js
DEFAULT_MAPPING
```

### Zmienić tekst „No data loaded”

Szukaj:

```html
No data loaded.
```

### Zmienić komunikat po nieznalezionej palecie

Szukaj:

```js
No results found for pallet ID
```

oraz:

```js
Today's data not imported
```

### Zmienić limit historii wrong scans

Szukaj:

```js
state.wrongScans = state.wrongScans.slice(0, 100)
```

### Zmienić limit historii completed pallets

Szukaj:

```js
state.completedHistory = state.completedHistory.slice(0, 100)
```

### Zmienić liczbę pozycji na stronie w logach

Szukaj:

```js
ACTIVITY_PAGE_SIZE
```

### Zmienić kolor główny

Szukaj:

```css
--accent
```

### Zmienić wygląd górnych zakładek

Szukaj:

```css
Larger top-left tabs
```

### Zmienić wygląd statystyk po prawej

Szukaj:

```css
Compact sticky side stats
```

### Zmienić podświetlanie wiersza skanowania

Szukaj:

```css
Continuous row progress highlight
```

---

## 20. Testy po każdej zmianie

Po każdej zmianie wykonaj minimum taki test:

1. Odśwież stronę przez `Ctrl + F5`.
2. Wyczyść dane przez `Clear All`.
3. Zaimportuj mały batch testowy.
4. Wyszukaj istniejącą paletę.
5. Zeskanuj poprawne SKU.
6. Zeskanuj błędne SKU.
7. Sprawdź `Wrong Scan Log`.
8. Dokończ paletę.
9. Sprawdź `Completed Pallets History`.
10. Odśwież stronę i sprawdź, czy dane wróciły z `localStorage`.

---

## 21. Ryzyka przy edycji jednoplikowego HTML

Najczęstsze problemy:

- literówka w JS zatrzymuje cały tool,
- zmiana ID w HTML bez zmiany `getElementById` psuje eventy,
- zmiana liczby kolumn tabeli bez zmiany `renderRowTable` psuje layout,
- zmiana `STORAGE_KEY` resetuje dane użytkownika,
- zmiana `DEFAULT_MAPPING` działa tylko dla nowych importów,
- źle dodany CSS z `!important` może nadpisać inne widoki,
- zbyt agresywny focus lock może utrudnić kliknięcie przycisków.

---

## 22. Minimalny model mentalny narzędzia

Najprościej myśleć o toolu tak:

```text
Import danych -> state.rows
Search Pallet -> currentView.rows
Start Scan -> scanState.requirements
Scan SKU/EAN -> scanState.scannedCounts
Update Status -> table highlight + progress
Completed -> completedHistory
Wrong Scan -> wrongScans
Save -> localStorage
```

Jeżeli rozumiesz ten przepływ, większość zmian w toolu będzie bezpieczna i przewidywalna.
