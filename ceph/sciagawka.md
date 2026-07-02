# Ceph — ściągawka (1 strona)

## Architektura w 10 sekund

**RADOS** = rozproszony magazyn obiektów. Na nim: **RBD** (blok), **RGW** (S3), **CephFS** (pliki). Klient liczy **CRUSH** lokalnie → idzie **bezpośrednio do OSD** (bez centralnego indeksu).

---

## Daemony

| Daemon | Rola | Minimum HA |
|--------|------|------------|
| **MON** | Cluster map, Paxos, kworum | **3** (nieparzysta liczba) |
| **OSD** | Dane na dysku, replikacja, recovery | **2+** (replikacja) |
| **MGR** | Metryki, dashboard, API | 1 aktywny + standby |
| **MDS** | Metadane CephFS | 1+ (tylko dla CephFS) |
| **RGW** | API S3/Swift | opcjonalnie |

---

## Ścieżka danych (zapis)

```
Klient → MON (cluster map) → CRUSH(pool + object_id) → PG + primary OSD
       → zapis do primary → replikacja na secondary OSD → ACK klientowi
```

**Klient zna:** pool, object ID, user + key. **Nie zna:** który OSD (liczy CRUSH).

Szczegółowy diagram: [diagram-zapisu.md](diagram-zapisu.md)

---

## Hierarchia storage

```
Pool → PG (placement group) → Obiekt → OSD (dysk)
         ↑
      CRUSH (topologia: host/rack/DC)
```

- **Pool** — polityka: replikacja/EC, liczba PG, CRUSH rule, ACL
- **PG** — jednostka replikacji, recovery, scrub (~100 PG/OSD/pool jako start)
- **Obiekt** — ID + dane + metadata (płaska przestrzeń, bez katalogów)

---

## CRUSH — 3 zdania

1. Hash(object) → numer PG w puli
2. PG → **Acting Set** OSD (primary + repliki)
3. Kopie w **różnych failure domains** (racki/hosty)

**CRUSH rule** przypisany do puli — **nie zmienia się** po utworzeniu.

---

## PG: stany i role

| Pojęcie | Znaczenie |
|---------|-----------|
| **Acting Set** | OSD przypisane do PG |
| **Up Set** | OSD z Acting Set, które są `up` |
| **Primary** | Pierwszy w zestawie — **jedyny przyjmuje zapisy** |
| **active+clean** | OK, wszystkie repliki zsynchronizowane |
| **degraded** | Brakuje repliki (awaria OSD) |
| **recovering** | Odtwarzanie kopii |

---

## Replikacja vs Erasure Coding

| | Replikacja | Erasure Coding |
|---|------------|----------------|
| **Przykład** | size=3 (3 pełne kopie) | K=4, M=2 (6 chunków) |
| **Zapis** | Szybszy | Wolniejszy (kodowanie) |
| **Miejsce** | Więcej (3×) | Mniej (~1.5×) |
| **Awaria** | Wytrzyma M-1 OSD (zależy od min_size) | Wytrzyma M OSD |

---

## BlueStore (OSD)

Dane **bezpośrednio na bloku** (nie ext4 na OSD). **Block DB** (metadane) + **WAL** (journal). Szybsze zapisy, checksumy, opcjonalna kompresja.

---

## Samonaprawa

| Proces | Kiedy |
|--------|-------|
| **Heartbeat** | OSD/MON sprawdzają `up`/`down` |
| **Peering** | OSD uzgadniają stan PG |
| **Recovery** | Po awarii — odtwarzanie kopii |
| **Backfill** | Po dodaniu OSD — ~1/N danych się przesuwa |
| **Scrub** | Codziennie light, tygodniowo deep |

---

## Autentykacja (cephx)

MON wydaje ticket → klient dostaje ticket do OSD. Jak Kerberos, ale bez jednego bottlenecku. **Nie szyfruje** danych na dysku/sieci.

---

## Komendy na start

```bash
ceph -s                      # status klastra
ceph health detail           # problemy
ceph osd tree                # topologia
ceph osd pool ls detail      # pule, PG, size
ceph pg stat                 # stany PG
ceph df                      # zużycie miejsca
```

---

## Pułapki / fakty do zapamiętania

- **1 MON** = działa, ale SPOF
- **2 MON** = zły pomysł (brak sensownego kworum)
- Tylko **primary** przyjmuje zapisy od klienta
- Brak **centralnego lookupu** obiekt→OSD — to siła Ceph
- Nowy OSD → **automatyczny** rebalance (nie ręczny)
- Za mało PG → duże skoki I/O; za dużo PG → CPU/RAM

---

## RBD (skrót)

Duży dysk = wiele obiektów (**striping**). **Exclusive lock** = jeden writer. **Object map** = szybsze operacje na rzadkich obrazach.

---

**Model mentalny:** `MON = mapa` · `CRUSH = GPS` · `PG = jednostka ruchu` · `OSD = dysk + robotnik`
