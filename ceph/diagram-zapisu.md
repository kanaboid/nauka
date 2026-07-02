# Ceph — diagramy architektury

## 1. Komponenty klastra

```mermaid
flowchart TB
    subgraph clients [Warstwa kliencka]
        RBD[RBD - dyski blokowe]
        RGW[RGW - S3/Swift]
        CephFS[CephFS - pliki]
    end

    subgraph cluster [Klaster RADOS]
        MON[MON x3 - cluster map, Paxos]
        MGR[MGR - metryki, dashboard]
        OSD1[OSD - dysk 1]
        OSD2[OSD - dysk 2]
        OSD3[OSD - dysk N]
    end

    RBD --> librados
    RGW --> librados
    CephFS --> librados
    librados[librados] -->|1. pobierz mapę| MON
    librados -->|2. CRUSH → bezpośredni I/O| OSD1
    librados --> OSD2
    librados --> OSD3
    OSD1 <-->|replikacja / peering| OSD2
    OSD2 <-->|replikacja / peering| OSD3
    MGR -.->|odczyt stanu| MON
    OSD1 -.->|heartbeat| MON
    OSD2 -.->|heartbeat| MON
    OSD3 -.->|heartbeat| MON
```

---

## 2. Hierarchia danych

```mermaid
flowchart LR
  subgraph logical [Warstwa logiczna]
    Pool[Pula - polityka, size, CRUSH rule]
    PG[Placement Group]
    Obj[Obiekt - ID + dane + metadata]
  end

  subgraph physical [Warstwa fizyczna]
  CRUSH[CRUSH - topologia host/rack/DC]
    OSDa[Primary OSD]
    OSDB[Secondary OSD]
    OSDc[Secondary OSD]
  end

  Pool --> PG
  PG --> Obj
  Obj -->|CRUSH| CRUSH
  CRUSH --> OSDa
  CRUSH --> OSDB
  CRUSH --> OSDc
```

---

## 3. Ścieżka zapisu (krok po kroku)

```mermaid
sequenceDiagram
    participant C as Klient (RBD/RGW/librados)
    participant MON as MON
    participant CRUSH as CRUSH (lokalnie)
    participant P as Primary OSD
    participant S1 as Secondary OSD 1
    participant S2 as Secondary OSD 2

    Note over C: Klient zna: pool + object_id + klucz cephx

    C->>MON: Pobierz cluster map
    MON-->>C: OSD map, CRUSH map, PG map

    C->>CRUSH: hash(object_id) % num_pg → PG ID
    CRUSH-->>C: PG 4.58 + Acting Set [OSD5, OSD10, OSD15]
    Note over CRUSH: Primary = pierwszy w zestawie (OSD5)

    C->>P: WRITE object (OSD5 = primary)
    P->>P: Zapis lokalny (BlueStore)
    P->>S1: Replikuj na OSD10
    P->>S2: Replikuj na OSD15
    S1-->>P: ACK
    S2-->>P: ACK
    P-->>C: ACK (zapis zakończony)
```

---

## 4. Ścieżka odczytu

```mermaid
sequenceDiagram
    participant C as Klient
    participant MON as MON
    participant CRUSH as CRUSH (lokalnie)
    participant P as Primary OSD

    C->>MON: Pobierz cluster map (jeśli nieaktualna)
    MON-->>C: Aktualna mapa

    C->>CRUSH: pool + object_id → PG + primary OSD
    CRUSH-->>C: np. OSD10 (nowy primary po awarii OSD5)

    C->>P: READ object
    P-->>C: Dane obiektu
```

---

## 5. Awaria OSD i recovery

```mermaid
stateDiagram-v2
    [*] --> active_clean: Wszystkie repliki OK

    active_clean --> degraded: Primary lub secondary pada
    degraded --> peering: OSD uzgadniają stan PG
    peering --> recovering: Brakująca kopia się odtwarza
    recovering --> active_clean: Wszystkie repliki zsynchronizowane

    active_clean --> backfilling: Dodano nowy OSD do klastra
    backfilling --> active_clean: CRUSH przeniósł część PG (~1/N danych)
```

---

## 6. Acting Set vs Up Set

```mermaid
flowchart TB
    subgraph acting [Acting Set dla PG 1.a]
        direction LR
        A["OSD25 (Primary)"]
        B["OSD32 (Secondary)"]
        C["OSD61 (Secondary)"]
    end

    subgraph up [Up Set - tylko działające OSD]
        direction LR
        B2["OSD32 → nowy Primary"]
        C2["OSD61"]
    end

    A -.->|OSD25 down| X[wyłączony]
    A -->|awaria| B2
```

**Zasada:** Tylko **Primary** przyjmuje zapisy od klienta. Po awarii primary kolejny OSD w Acting Set przejmuje rolę.

---

## Legenda skrótów

| Skrót | Znaczenie |
|-------|-----------|
| RADOS | Reliable Autonomic Distributed Object Store |
| MON | Monitor — trzyma cluster map |
| OSD | Object Storage Daemon — jeden dysk |
| PG | Placement Group — jednostka replikacji |
| CRUSH | Algorytm rozmieszczania danych |
| EC | Erasure Coding — K+M chunków zamiast pełnych kopii |
