#SalesForm upsell rendszer, ahol értékesítők hívják az ügyfeleket – Google Apps Script

Automatikus értékesítői rendszer Google Spreadsheet alapon. Szinkronizálja a SalesForm rendelési adatait több értékesítő saját munkafüzetébe, kezeli a foglalásokat, az upsell célokat és a jutalékokat.

---

## Tartalomjegyzék

- [Hogyan működik](#hogyan-működik)
- [Előfeltételek](#előfeltételek)
- [Telepítés](#telepítés)
- [Mit kell átírni a kódban](#mit-kell-átírni-a-kódban)
- [Triggerek beállítása](#triggerek-beállítása)
- [Upsell szabályok beállítása](#upsell-szabályok-beállítása)
- [Munkalapok leírása](#munkalapok-leírása)
- [Státuszok és színek](#státuszok-és-színek)
- [Foglalási rendszer](#foglalási-rendszer)
- [Jutalék rendszer](#jutalék-rendszer)
- [Több értékesítő beállítása](#több-értékesítő-beállítása)
- [Forrás táblázat oszlopai](#forrás-táblázat-oszlopai)
- [Hibaelhárítás](#hibaelhárítás)

---

## Hogyan működik

```
FORRÁS TÁBLÁZAT (SalesForm adatok)
│
├── Sikeres rendelések_2026-02-25
├── Sikeres előfizetések_2026-02-25
└── ... (Z oszlop = Tulajdonos URL)
         │
         ▼
   Google Apps Script
         │
    ┌────┴────┐
    ▼         ▼
 Értékesítő A   Értékesítő B
 Cél tábla      Cél tábla
 │               │
 ├── Sikeres rendelések
 ├── Sikeres előfizetések
 ├── Upsell
 ├── Termék_szűrő
 └── Jutalék
```

- Minden értékesítőnek **saját cél táblázata** van
- A forrás táblázat **közös** – mindenkinek ugyanaz
- Ha egy értékesítő lefoglal egy sort, a **forrás Z oszlopába** bekerül az ő táblázatának URL-je
- A többi értékesítő táblájából a sor **automatikusan eltűnik** a következő `quickClean` futáskor (max. 1 perc)

---

## Előfeltételek

- SalesForm fiók
- Google fiók
- Hozzáférés a forrás táblázathoz (szerkesztési jog szükséges a Z oszlophoz)
- Saját Google Spreadsheet a cél táblázatnak (üres, új fájl)

---

## Telepítés

### 1. Cél táblázat létrehozása

Hozz létre egy **új, üres Google Spreadsheetest** minden értékesítőnek. Nevezd el pl. `Értékesítő – Kiss János`.

### 2. Script bemásolása

1. Nyisd meg a cél spreadsheetet
2. Kattints: **Bővítmények → Apps Script**
3. Töröld az alapértelmezett `function myFunction() {}` kódot
4. Másold be a `script.gs` teljes tartalmát
5. Kattints a **💾 Mentés** gombra (`Ctrl+S`)

### 3. Spreadsheet ID-k beállítása

Keresd meg a kódban ezt a részt és írd be a saját ID-kat:

```javascript
function getSpreadsheetIds() {
  return {
    sourceSpreadsheetId: "IDE_IRD_A_FORRÁS_TÁBLA_ID-JÁT",
    targetSpreadsheetId: "IDE_IRD_A_CÉL_TÁBLA_ID-JÁT"
  };
}
```

**Hogyan találod meg a Spreadsheet ID-t?**

Nyisd meg a táblázatot és nézd meg az URL-t:
```
https://docs.google.com/spreadsheets/d/1ABC123XYZ.../edit
                                       ^^^^^^^^^^^
                                       Ez az ID
```

### 4. Jutalék beállítása

```javascript
// "fixed" = minden eladásnál ugyanannyi fix összeg
// "percentage" = a végösszeg százaléka
function getCommissionType() {
  return "percentage";
}

// Ha fix összeget választottál – ide írd a forint összeget
function getCommissionAmount() {
  return 10000; // 10 000 Ft
}

// Ha százalékot választottál – ide írd a százalékot (csak szám, pl. 15 = 15%)
function getCommissionRate() {
  return 15;
}
```

### 5. Első futtatás

1. Az Apps Script szerkesztőben válaszd ki a **`runAll`** függvényt a felső legördülőből
2. Kattints a **▶ Futtatás** gombra
3. Első alkalommal Google engedélyt kér → **Engedélyek áttekintése** → válaszd ki a fiókodat → **Speciális** → **Tovább** → **Engedélyezés**
4. Várd meg amíg lefut

Az első futtatás után a cél táblázatban megjelennek a munkalapok és feltöltődnek az adatok.

### 6. Upsell szabályok beállítása

Lásd: [Upsell szabályok beállítása](#upsell-szabályok-beállítása)

### 7. Triggerek beállítása

Lásd: [Triggerek beállítása](#triggerek-beállítása)

---

## Mit kell átírni a kódban

### Kötelező módosítások

| Függvény | Mit kell megváltoztatni |
|---|---|
| `getSpreadsheetIds()` | `sourceSpreadsheetId` és `targetSpreadsheetId` |

### Opcionális módosítások

| Függvény | Mit lehet megváltoztatni |
|---|---|
| `getCommissionType()` | `"fixed"` vagy `"percentage"` |
| `getCommissionAmount()` | Fix jutalék összege forintban |
| `getCommissionRate()` | Jutalék százaléka (csak szám, pl. `15`) |

### Ha a forrás táblázat oszlopai eltérnek

Ha a forrás táblázatodban az adatok más oszlopokban vannak mint az alapértelmezett, módosítsd az oszlopszámokat a megfelelő függvényben.

A `syncSuccessfulOrders` függvényben:

```javascript
// Oszlopszámok 1-alapúak: A=1, B=2, C=3, ... Z=26
var idColumn          = 1;   // A – Azonosító
var nameColumn        = 2;   // B – Név
var emailColumn       = 10;  // J – E-mail
var phoneColumn       = 11;  // K – Telefon
var dateColumn        = 12;  // L – Dátum
var productColumn     = 13;  // M – Termék
var bumpColumn        = 15;  // O – BUMP termék neve
var bumpQtyColumn     = 16;  // P – BUMP mennyiség
var resellerColumn    = 17;  // Q – Viszonteladó
var totalColumn       = 18;  // R – Végösszeg
```

A `syncSuccessfulSubscriptions` függvényben:

```javascript
var idColumn       = 1;   // A – Azonosító
var nameColumn     = 2;   // B – Név
var emailColumn    = 11;  // K – E-mail
var phoneColumn    = 12;  // L – Telefon
var productColumn  = 14;  // N – Termék
var resellerColumn = 18;  // R – Viszonteladó
var totalColumn    = 19;  // S – Végösszeg
```

> **Fontos:** az oszlopszámok **1-alapúak**. A=1, B=2, ... J=10, K=11, ... Z=26.

---

## Triggerek beállítása

Minden értékesítő cél táblázatában **3 triggert** kell beállítani.

### Trigger beállítás lépései

1. Az Apps Script szerkesztőben kattints a bal oldali **⏰ ikonra** (Triggerek)
2. Jobb alsó sarokban: **+ Trigger hozzáadása**
3. Állítsd be a mezőket az alábbi leírás szerint
4. **Mentés**

---

### 1. trigger – Napi teljes szinkronizálás

| Mező | Érték |
|---|---|
| Melyik függvényt futtassa | `runAll` |
| Eseményforrás | Időalapú timer |
| Típus | Nap timer |
| Időpont | 7:00–8:00 |

Mit csinál: szinkronizálja az összes új rendelést, frissíti az Upsell munkalapot, újragenerálja a Jutalék lapot, kitakarítja a foglalt sorokat.

---

### 2. trigger – Gyors foglalás-szinkron

| Mező | Érték |
|---|---|
| Melyik függvényt futtassa | `quickClean` |
| Eseményforrás | Időalapú timer |
| Típus | Perc timer |
| Időköz | Minden 1 perc |

Mit csinál: megnézi a forrás táblában ki foglalt le sorokat és törli azokat ebből a táblázatból. Gyors (~2-5 mp), percenként futtatható gond nélkül.

> Ha egy másik értékesítő lefoglal egy tételt, az **maximum 1 percen belül** eltűnik ebből a táblázatból is.

---

### 3. trigger – Azonnali státuszkezelés *(kötelező!)*

| Mező | Érték |
|---|---|
| Melyik függvényt futtassa | `onEditInstallable` |
| Eseményforrás | Táblázatból |
| Eseménytípus | **Szerkesztéskor** |

Mit csinál: amikor az értékesítő státuszt változtat, azonnal:
- Beírja a forrás táblába hogy ez a sor le van foglalva (Z oszlop)
- Kiszínezi a sort a státusznak megfelelő színre
- Kiszámítja a jutalékot ("Megmentve" vagy "Eladva" státusznál)
- Törli a sort a többi értékesítő táblájából

> **Miért kell "telepített" trigger és nem egyszerű `onEdit`?**
> Az egyszerű `onEdit` nem tud más Google Spreadsheetbe írni – nincs joga hozzá. A telepített trigger a te Google fiókoddal fut, így vissza tud írni a forrás táblába.

---

## Upsell szabályok beállítása

Az első `runAll` futtatás után megjelenik a **Termék_szűrő** munkalap. Ezt kell kitölteni.

### A Termék_szűrő munkalap szerkezete

| Termék | Db | Upsell 1 | Upsell 2 | Upsell 3 | Upsell 4 | Upsell 5 | Upsell 6 | Kiválasztva |
|---|---|---|---|---|---|---|---|---|
| Alapcsomag | 142 | Prémium | VIP | | | | | ☑ |
| Starter | 38 | Business | | | | | | ☐ |

**Lépések:**

1. A **Kiválasztva** oszlopban pipáld be azokat a termékeket, amelyek vásárlóit fel szeretnéd hívni
2. Az **Upsell 1–6** oszlopokban a legördülőből válaszd ki mit szeretnél eladni nekik (max. 6 cél termék termékenként)
3. Futtasd újra a `runAll`-t – az Upsell munkalap feltöltődik a megfelelő vásárlókkal

> Csak az elmúlt **60 napban** vásárolt ügyfelek kerülnek az Upsell munkalapra.

---

## Munkalapok leírása

### Sikeres rendelések

| Oszlop | Tartalom |
|---|---|
| Azonosító | Egyedi rendelésazonosító |
| Név | Vásárló neve |
| E-mail | Vásárló e-mail címe |
| Telefonszám | Vásárló telefonszáma |
| Dátum | Vásárlás dátuma (legfrissebb elöl) |
| Végösszeg | Rendelés összege |
| Termék | Vásárolt termék neve |
| BUMP | Order bump termék (ha volt) |
| Viszonteladó | Ki közvetítette az eladást |
| Státusz | Legördülő lista |
| Jutalék | Automatikusan számított ("Megmentve" státusznál) |

### Sikeres előfizetések

Megegyezik a Sikeres rendelések szerkezetével, BUMP oszlop nélkül.

### Upsell

| Oszlop | Tartalom |
|---|---|
| Azonosító | Egyedi azonosító |
| Név | Vásárló neve |
| E-mail | Vásárló e-mail címe |
| Telefonszám | Vásárló telefonszáma |
| Dátum | Eredeti vásárlás dátuma |
| Végösszeg | Eredeti rendelés összege |
| Termék | Megvásárolt belépő termék |
| BUMP | Order bump (ha volt) |
| Viszonteladó | Ki közvetítette az eredeti eladást |
| Upsell célok | Automatikusan kitöltve a Termék_szűrőből |
| Státusz | Legördülő lista |
| Megjegyzés | Az értékesítő saját jegyzetei (nem szinkronizálódik) |
| Jutalék | Automatikusan számított ("Eladva" státusznál) |

### Termék_szűrő

Az összes termék a forrásból, darabszámokkal és upsell beállításokkal.

### Jutalék

Automatikusan generált összesítő. Csak "Eladva" státuszú upsell sorok kerülnek ide, ahol van Viszonteladó adat.

> ⚠️ Ezt a munkalapot **ne szerkeszd kézzel** – minden `runAll` futtatáskor újragenerálódik.

---

## Státuszok és színek

### Sikeres rendelések és Sikeres előfizetések

| Státusz | Szín | Jutalék számítás |
|---|---|---|
| Válassz | ⬜ Fehér | – |
| Időpont küldve | 🟡 Sárga | – |
| Időpont emlékeztető | 🟠 Narancssárga | – |
| Foglalt | 🔵 Kék | – |
| **Megmentve** | 🟢 Zöld | ✅ Igen |
| Elvesztett | 🔴 Piros | – |

### Upsell munkalap

| Státusz | Szín | Jutalék számítás |
|---|---|---|
| Válassz | ⬜ Fehér | – |
| Időpont küldve | 🟡 Sárga | – |
| Időpont emlékeztető | 🟠 Narancssárga | – |
| Foglalt | 🔵 Kék | – |
| **Eladva** | 🟢 Zöld | ✅ Igen |
| Elvesztett | 🔴 Piros | – |

> Az "Eladva" státuszt a rendszer automatikusan állítja be ha észleli, hogy az ügyfél megvette a cél terméket. Manuálisan is beállítható.

---

## Foglalási rendszer

```
Ügyfél látható mindenkinél
        │
        ▼
Értékesítő státuszt állít
        │
        ▼
onEditInstallable lefut AZONNAL:
  ├── Forrás Z oszlop = értékesítő spreadsheet URL-je
  ├── Sor kiszínezése
  ├── Jutalék számítás (ha "Megmentve"/"Eladva")
  └── cleanTargetSpreadsheet() – törli a többi táblából
        │
        ▼
quickClean (percenként minden táblában):
  └── cleanTargetSpreadsheet() – megerősíti a törlést
```

- Ha visszaállítod **"Válassz"** státuszra → a foglalás felszabadul, mindenkinél visszajön
- Ha két értékesítő egyszerre foglal → az nyer aki **előbb** állított státuszt

---

## Jutalék rendszer

### Mikor számolódik?

- **Sikeres rendelések / Előfizetések:** "Megmentve" státusz beállításakor
- **Upsell:** "Eladva" státusz beállításakor

### Számítás módja

**Százalékos:** `Jutalék = Végösszeg × jutalék% / 100`

**Fix összeg:** minden eladásnál ugyanannyi

### Jutalék munkalap

Tételes lista és összesítő viszonteladónként. Csak akkor jelenik meg egy sor, ha van Viszonteladó adat a forrásban.

---

## Több értékesítő beállítása

Minden értékesítőnek **azonos lépésekkel** kell beállítani:

1. Új üres Google Spreadsheet létrehozása
2. Script bemásolása (ugyanaz a kód mindenkinek)
3. `sourceSpreadsheetId` = **ugyanaz** mindenkinek (közös forrás)
4. `targetSpreadsheetId` = **mindenkinél a saját** spreadsheet ID-ja
5. Jutalék beállítása (lehet értékesítőnként eltérő)
6. Mindhárom trigger beállítása
7. `runAll` futtatása egyszer manuálisan
8. Termék_szűrő kitöltése

### Jogosultságok a forrás táblázathoz

Minden értékesítőnek **szerkesztési jog** kell a forrás táblázathoz:

1. Nyisd meg a forrás táblázatot
2. **Megosztás** → add meg az értékesítő Google fiókját
3. Jogosultság: **Szerkesztő**

---

## Forrás táblázat oszlopai

### Sikeres rendelések

| Oszlop betű | Szám | Adat |
|---|---|---|
| A | 1 | Azonosító |
| B | 2 | Név |
| J | 10 | E-mail |
| K | 11 | Telefon |
| L | 12 | Dátum |
| M | 13 | Termék |
| O | 15 | BUMP termék |
| P | 16 | BUMP mennyiség |
| Q | 17 | Viszonteladó |
| R | 18 | Végösszeg |
| Z | 26 | Tulajdonos URL *(a script írja)* |

### Sikeres előfizetések

| Oszlop betű | Szám | Adat |
|---|---|---|
| A | 1 | Azonosító |
| B | 2 | Név |
| K | 11 | E-mail |
| L | 12 | Telefon |
| M | 13 | Dátum |
| N | 14 | Termék |
| R | 18 | Viszonteladó |
| S | 19 | Végösszeg |
| Z | 26 | Tulajdonos URL *(a script írja)* |

---

## Hibaelhárítás

### Naplók megtekintése

1. Apps Script szerkesztő → bal oldali **▶ Végrehajtások** ikon
2. Kattints bármelyik futtatásra → naplóbejegyzések listája

### Nem kerülnek át az adatok

A script `"Sikeres rendelések_"` előtagú munkalapot keres a forrásban (pl. `Sikeres rendelések_2026-02-25`). Ellenőrizd a forrás táblában a munkalapok pontos nevét.

### Az onEditInstallable nem fut le

Státuszváltáskor nem színeződik a sor és nem íródik a forrás Z oszlopa.

**Ok:** nincs beállítva a telepített trigger.

**Megoldás:** állítsd be `onEditInstallable`-t **"Szerkesztéskor"** eseménytípussal telepített triggerként. Lásd: [Triggerek beállítása](#triggerek-beállítása).

### A sor nem tűnik el a többi táblázatból

**Ok:** a `quickClean` trigger nincs beállítva a többi értékesítő táblájában.

**Megoldás:** minden értékesítő táblájában állítsd be a `quickClean` percenkénti triggert.

### Engedélyezési hiba

`"Exception: You do not have permission to..."`

**Megoldás:** az értékesítőnek **szerkesztési jogot** kell adni a forrás táblázathoz.

### Az upsell jutalék nem számolódik automatikusan

A `checkUpsellSales` az ügyfél e-mail címe alapján egyeztet. Ha az e-mail más formátumban szerepel a forrásban (extra szóköz, eltérő kis/nagybetű), nem találja meg.

---

## Függvények összefoglalója

| Függvény | Mit csinál | Mikor fut |
|---|---|---|
| `runAll` | Teljes szinkronizálás | Naponta 1x (trigger) |
| `quickClean` | Foglalt sorok gyors törlése | Percenként (trigger) |
| `onEditInstallable` | Státuszváltás azonnali kezelése | Szerkesztéskor (trigger) |
| `syncSuccessfulOrders` | Sikeres rendelések szinkron | `runAll` részeként |
| `syncSuccessfulSubscriptions` | Sikeres előfizetések szinkron | `runAll` részeként |
| `syncProductsToUpsell` | Termék_szűrő + Upsell lap | `runAll` részeként |
| `checkUpsellSales` | Automatikus Eladva státusz + jutalék | `runAll` részeként |
| `syncCommissionSheet` | Jutalék lap generálás | `runAll` részeként |
| `cleanTargetSpreadsheet` | Foglalt sorok törlése | `runAll` és `quickClean` részeként |
