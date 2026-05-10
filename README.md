# Medalion NYC Taxi — Analiza Danych Yellow Taxi (2014–2024)

## Opis Projektu

Celem projektu jest analiza **wydajności i rentowności** usług Yellow Taxi w Nowym Jorku
na przestrzeni dekady (2014–2024). Przetwarzając **15 GB surowych danych**, potok
identyfikuje najbardziej dochodowe strefy oraz trendy przychodowe. Projekt demonstruje
budowę skalowalnej architektury typu **Lakehouse w chmurze Azure** przy użyciu standardu
Medallion, rozwiązując realne problemy niespójności danych historycznych.

Projekt implementuje dwa tryby przetwarzania:
- **Batch Pipeline** — hurtowe przetwarzanie danych historycznych (Bronze → Silver → Gold)
- **Streaming Pipeline** — przetwarzanie strumieniowe w czasie rzeczywistym z użyciem kolejki Azure Event Hubs

---

## Architektura i Przepływ Danych

Projekt realizuje wzorzec **Architektury Medalionowej**, zapewniając czystość i spójność
danych na każdym etapie przetwarzania.

```
                        ┌─────────────────────────────────────────────────────┐
                        │          BATCH PIPELINE (dane historyczne)          │
                        │                                                     │
[Źródło: NYC Open Data] │  ┌──────────┐    ┌──────────┐    ┌──────────┐      │
        │               │  │ 🥉 BRONZE│───▶│ 🥈 SILVER│───▶│ 🥇 GOLD  │      │
        └──────────────▶│  │  RAW     │    │ CLEANED  │    │ CURATED  │      │
                        │  └──────────┘    └──────────┘    └──────────┘      │
                        └─────────────────────────────────────────────────────┘

                        ┌─────────────────────────────────────────────────────┐
                        │       STREAMING PIPELINE (dane real-time)           │
                        │                                                     │
[Producer: symulacja    │  ┌──────────┐    ┌──────────┐                      │
 danych taksówek]       │  │ 📨 EVENT │    │ 🥉 BRONZE│                      │
        │               │  │   HUBS   │───▶│ STREAMING│                      │
        └──────────────▶│  │  (Queue) │    │  (Delta) │                      │
                        │  └──────────┘    └──────────┘                      │
                        └─────────────────────────────────────────────────────┘
```

---

## Stos Technologiczny

| Warstwa            | Technologia                                        |
|--------------------|----------------------------------------------------|
| **Obliczenia**     | Azure Databricks (Runtime 14.3 LTS, Spark 3.5.0)  |
| **Storage**        | Azure Blob Storage (protokół WASBS)                |
| **Kolejka**        | Azure Event Hubs (Kafka-compatible)                |
| **Streaming**      | Spark Structured Streaming                         |
| **Format danych**  | Parquet (batch), Delta Lake (streaming)            |
| **Język**          | PySpark, Spark SQL, Python                         |
| **Bezpieczeństwo** | Azure Key Vault + Databricks Secret Scope          |
| **Orkiestracja**   | Databricks Workflows + Asset Bundles (YAML)        |

---

## Szczegóły Architektury Danych

### 🥉 Warstwa Bronze — Surowe Dane
- Bezpośrednia ingestia **15 GB plików Parquet** z NYC Open Data
- Dane przechowywane w oryginalnej, niezmienionej postaci
- Check idempotentności — pomija ingestię jeśli dane już istnieją

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

## Przetwarzanie Strumieniowe (Streaming Pipeline)

Oprócz batch pipeline'u, projekt implementuje **przetwarzanie strumieniowe** 
z użyciem kolejki komunikatów i Spark Structured Streaming.

### Architektura Streamingu

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  04_Stream        │     │  Azure Event     │     │  05_Stream        │
│  Producer         │────▶│  Hubs            │────▶│  Consumer         │
│                   │     │  (kolejka)       │     │                   │
│  Czyta dane z     │     │  Buforuje i      │     │  Spark Structured │
│  Silver, wysyła   │     │  kolejkuje       │     │  Streaming czyta  │
│  JSON do kolejki  │     │  wiadomości      │     │  i zapisuje do    │
│                   │     │                  │     │  Bronze (Delta)   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

### Komponenty

**Producer (`04_Stream_Producer`)** — symuluje źródło danych taksówkowych. 
Czyta rekordy z warstwy Silver i wysyła je jako JSON do kolejki Azure Event Hubs. 
W środowisku produkcyjnym rolę producenta pełniłyby systemy GPS w taksówkach.

**Queue — Azure Event Hubs** — zarządzana kolejka komunikatów kompatybilna z protokołem 
Apache Kafka. Buforuje wiadomości między producentem a konsumentem, zapewniając 
odporność na awarie i skalowalność. Namespace: `nyc-taxi-streaming`, 
Event Hub: `taxi-rides`.

**Consumer (`05_Stream_Consumer`)** — Spark Structured Streaming nasłuchuje 
Event Hubs w trybie ciągłym. Automatycznie odbiera nowe wiadomości, parsuje JSON 
i zapisuje dane do Streaming Bronze w formacie Delta Lake. Wykorzystuje checkpointy 
do zapewnienia exactly-once processing.

### Dlaczego Delta Lake?
Streaming zapisuje dane w formacie **Delta Lake** (nie Parquet) ponieważ:
- Obsługuje transakcje ACID — bezpieczny zapis z wielu strumieni
- Wspiera checkpointy — odporność na awarie
- Schema enforcement — automatyczna walidacja schematu
- Time travel — możliwość przejrzenia historii zmian

---

## Orkiestracja Pipeline'u (Batch)

Pipeline batch jest orkiestrowany przez **Databricks Workflows** jako Job z trzema 
zależnymi taskami:

```
bronze_ingestion → silver_cleaning → gold_analysis
```

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

## Wyzwania Techniczne i Rozwiązania

### ⚠️ Kryzys Ewolucji Schematu
**Problem:** `ClassCastException` spowodowany zmianami typów danych w plikach Parquet
na przestrzeni lat (zmiana `INT64` → `DOUBLE`).

**Rozwiązanie:** Wdrożenie iteracyjnego procesu łączenia plików (`union`) z
jawnym wymuszaniem schematu i ręcznym rzutowaniem typów podczas ingestii.

### ⚡ Przetwarzanie na Dużą Skalę
**Problem:** Obsługa ponad **150 milionów rekordów (15 GB)** w jednym potoku.

**Rozwiązanie:** Wyłączenie Wektoryzowanego Czytnika Parquet (`Vectorized Parquet
Reader`), co zapewniło elastyczność typów i stabilność przetwarzania.

### 🔒 Bezpieczeństwo Danych
**Problem:** Ryzyko ujawnienia kluczy dostępowych Azure w publicznym repozytorium.

**Rozwiązanie:** Integracja z **Azure Key Vault + Databricks Secret Scope** — 
klucze dostępowe (storage, Event Hubs) są maskowane i nigdy nie pojawiają się 
w kodzie źródłowym.

---

## Wnioski Biznesowe

Końcowa analiza SQL identyfikuje **najbardziej dochodowe strefy taksówkowe** w Nowym
Jorku, dostarczając informacji o:
- łącznych przychodach w podziale na strefy i dzielnice
- liczbie przejazdów w każdej lokalizacji
- średniej odległości przejazdu na borough

---

## ERD + High Level

<img width="2816" height="1536" alt="erd+highlevel" src="https://github.com/user-attachments/assets/b90ae488-9a24-4bce-8375-714afc357aa0" />

---

## Architektura Pipeline'u

<img width="1322" height="840" alt="nycpipelinearchitecture" src="https://github.com/user-attachments/assets/2b914c48-a0e1-4baa-9150-16218dda63d8" />

---

## Struktura Repozytorium

```
├── 01_Ingestion_Bronze.ipynb      # Ingestia danych historycznych (batch)
├── 02_Silver_cleaning.ipynb       # Czyszczenie i walidacja danych
├── 03_Gold_Analysis.ipynb         # Agregacje biznesowe (JOIN, GROUP BY)
├── 04_Stream_Producer.ipynb       # Producent — wysyła dane do Event Hubs
├── 05_Stream_Consumer.ipynb       # Konsument — Structured Streaming z Event Hubs
├── databricks.yml                 # Databricks Asset Bundle — konfiguracja
├── resources/
│   └── nyc_taxi_pipeline.yml      # Definicja Workflow (pipeline as code)
├── pipeline_architecture.drawio   # Diagram architektury (edytowalny)
├── workflow_dag.png               # Screenshot DAG z Databricks Workflows
└── README.md
```

---

## Jak Uruchomić Projekt

**Wymagania wstępne:**
- Aktywna subskrypcja Azure z zasobem Databricks
- Konto Azure Blob Storage z danymi NYC Taxi
- Azure Event Hubs namespace (dla streamingu)

**Kroki — Batch Pipeline:**

1. Skonfiguruj klaster Databricks (Single User, Runtime 14.3 LTS)
2. Zainstaluj bibliotekę Maven: `com.microsoft.azure:azure-eventhubs-spark_2.12:2.3.22`
3. Skonfiguruj Secret Scope (`azure-storage`) z kluczami:
   - `storagekeybdg` — klucz dostępowy Azure Blob Storage
   - `eventhubs-connection-string` — connection string Event Hubs
4. Uruchom Workflow "Run Now" lub notebooki w kolejności:
```
01_Ingestion_Bronze  →  02_Silver_cleaning  →  03_Gold_Analysis
```

**Kroki — Streaming Pipeline:**

1. Uruchom `04_Stream_Producer` — wysyła dane do Event Hubs
2. Uruchom `05_Stream_Consumer` — Structured Streaming czyta z kolejki
3. Streaming działa ciągle do momentu zatrzymania (`query.stop()`)
