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


## Orkiestracja Pipeline'u

Pipeline jest orkiestrowany przez **Databricks Workflows** jako Job z trzema 
zależnymi taskami:

bronze_ingestion → silver_cleaning → gold_analysis

### Single Entry Point
Pipeline uruchamia się jednym kliknięciem "Run Now" w Databricks Workflows, 
albo komendą:
```bash
databricks bundle deploy
databricks bundle run nyc_taxi_pipeline
```

### Idempotentność
Notebook `01_Ingestion_Bronze` zawiera check, który pomija ingestię jeśli 
dane już istnieją w warstwie Bronze. Dzięki temu wielokrotne uruchomienie 
pipeline'u nie powoduje duplikacji ani ponownego pobierania 15 GB danych.

### Pipeline as Code
Definicja workflow jest wersjonowana w Git jako Databricks Asset Bundle 
(`databricks.yml` + `resources/nyc_taxi_pipeline.yml`).

<img width="1128" height="215" alt="workflow_dag" src="https://github.com/user-attachments/assets/ecf9f248-f286-4e59-b276-8ef36ac7e7e7" />


---

### ERD + High Level
<img width="2816" height="1536" alt="erd+highlevel" src="https://github.com/user-attachments/assets/b90ae488-9a24-4bce-8375-714afc357aa0" />

---

## Architektura Pipeline'u


<img width="1261" height="550" alt="pipeline_architecture drawio" src="https://github.com/user-attachments/assets/e08a9426-bc35-44d4-8b54-5b1817574939" />


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
