# Medalion NYC Taxi — Analiza Danych Yellow Taxi (2014–2024)

## Opis Projektu

Celem projektu jest analiza **wydajności i rentowności** usług Yellow Taxi w Nowym Jorku
na przestrzeni dekady (2014–2024). Przetwarzając **15 GB surowych danych**, potok
identyfikuje najbardziej dochodowe strefy oraz trendy przychodowe. Projekt demonstruje
budowę skalowalnej architektury typu **Lakehouse w chmurze Azure** przy użyciu standardu
Medallion, rozwiązując realne problemy niespójności danych historycznych.

---

## Architektura i Przepływ Danych

Projekt realizuje wzorzec **Architektury Medalionowej**, zapewniając czystość i spójność
danych na każdym etapie przetwarzania.
```
[Źródło: NYC Open Data]
        │
        ▼
┌───────────────┐
│  🥉 BRONZE    │  Surowe pliki Parquet (15 GB)
│  Warstwa RAW  │  Bezpośrednia ingestia z NYC Open Data
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  🥈 SILVER    │  Dane oczyszczone i zwalidowane
│  Warstwa      │  Filtry jakości, deduplikacja
│  CLEANED      │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  🥇 GOLD      │  Zagregowane dane biznesowe
│  Warstwa      │  Przychody wg dzielnic Nowego Jorku
│  CURATED      │
└───────────────┘
```

---

## Stos Technologiczny

| Warstwa       | Technologia                                        |
|---------------|----------------------------------------------------|
| **Obliczenia**| Azure Databricks (Runtime 14.3 LTS, Spark 3.5.0)  |
| **Storage**   | Azure Blob Storage (protokół WASBS)                |
| **Język**     | PySpark, Spark SQL, Python                         |
| **Bezpieczeństwo** | Databricks Secrets API                        |

---

## Szczegóły Architektury Danych

### 🥉 Warstwa Bronze — Surowe Dane
- Bezpośrednia ingestia **15 GB plików Parquet** z NYC Open Data
- Dane przechowywane w oryginalnej, niezmienionej postaci

### 🥈 Warstwa Silver — Dane Oczyszczone
Zastosowane filtry jakości danych:
- `fare_amount > 0` — eliminacja nieprawidłowych opłat
- `passenger_count > 0` — eliminacja pustych przejazdów
- Walidacja chronologiczna (`dropoff > pickup`)
- Usunięcie zduplikowanych rekordów

### 🥇 Warstwa Gold — Dane Biznesowe
- Agregacje na poziomie biznesowym
- Złączenie faktów z tabelą słownikową `taxi_zones`
- Analiza przychodów w podziale na dzielnice (borough)

---

## Wyzwania Techniczne i Rozwiązania

### ⚠️ Kryzys Ewolucji Schematu
**Problem:** `ClassCastException` spowodowany zmianami typów danych w plikach Parquet
na przestrzeni lat (zmiana `INT64` → `DOUBLE`).

**Rozwiązanie:** Wdrożenie iteracyjnego procesu łączenia plików (`union`) z
jawnym wymuszaniem schematu i ręcznym rzutowaniem typów podczas ingestii.

---

### ⚡ Przetwarzanie na Dużą Skalę
**Problem:** Obsługa ponad **150 milionów rekordów (15 GB)** w jednym potoku.

**Rozwiązanie:** Wyłączenie Wektoryzowanego Czytnika Parquet (`Vectorized Parquet
Reader`), co zapewniło elastyczność typów i stabilność przetwarzania.

---

### 🔒 Bezpieczeństwo Danych
**Problem:** Ryzyko ujawnienia kluczy dostępowych Azure w publicznym repozytorium.

**Rozwiązanie:** Integracja z **Databricks Secrets API** — klucze dostępowe są
maskowane i nigdy nie pojawiają się w kodzie źródłowym.

---

## Wnioski Biznesowe

Końcowa analiza SQL identyfikuje **najbardziej dochodowe strefy taksówkowe** w Nowym
Jorku, dostarczając informacji o:
- łącznych przychodach w podziale na strefy i dzielnice
- liczbie przejazdów w każdej lokalizacji
- średniej odległości przejazdu na borough

---

### ERD + High Level
<img width="2816" height="1536" alt="erd+highlevel" src="https://github.com/user-attachments/assets/b90ae488-9a24-4bce-8375-714afc357aa0" />

---

## Jak Uruchomić Projekt

**Wymagania wstępne:**
- Aktywna subskrypcja Azure z zasobem Databricks
- Konto Azure Blob Storage z danymi NYC Taxi

**Kroki:**

1. **Skonfiguruj klaster Databricks**
   - Tryb: Single User (zalecany)
   - Runtime: 14.3 LTS

2. **Skonfiguruj Secret Scope**
   - Nazwa scope: `azure-storage`
   - Dodaj klucz dostępowy do Azure Blob Storage

3. **Uruchom notebooki w kolejności:**
```
01_Ingestion.ipynb        →  Ingestia surowych danych (Bronze)
02_Silver_Transformation  →  Czyszczenie i walidacja (Silver)
03_Gold_Analysis          →  Agregacje biznesowe (Gold)
```
