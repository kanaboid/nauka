# Lab PG — śledzenie na żywym klastrze

Wykonuj na `ceph-lab` z katalogu `~/nauka/ceph`.

```bash
cd ~/nauka/ceph
alias cephsh='sudo ./cephadm shell -- ceph'
```

---

## Krok 1 — ile PG ma testpool?

```bash
cephsh osd pool ls detail | grep -A20 testpool
cephsh pg ls-by-pool testpool | wc -l
```

Szukaj: `pg_num 32`, `size 2`, `pool ID` (np. `1` lub `2`).

PG mają format: `<pool_id>.<pg_num>` → np. `2.5` = PG 5 w puli o id 2.

---

## Krok 2 — losowa PG z puli

```bash
cephsh pg ls-by-pool testpool | head -5
```

Wybierz jedną linię, np. `2.7`.

---

## Krok 3 — gdzie leży ta PG? (acting set)

```bash
PG=2.7   # zamień na swoją

cephsh pg map $PG
cephsh pg dump pgs_brief | grep "^${PG}"
```

`pg map` pokazuje:
- **up** / **acting** — które OSD (numery) trzymają tę PG
- pierwszy w acting = **primary**
- **object_pg_split_bits** — techniczne; na start ignoruj

Porównaj z drzewem OSD:

```bash
cephsh osd tree
```

Mapowanie: `osd.0` → id `0`, `osd.1` → id `1`, itd.

---

## Krok 4 — od obiektu do PG

Obiekty w puli (w tym fragmenty RBD `demo`):

```bash
cephsh osd pool application get testpool
sudo ./cephadm shell -- rados ls -p testpool | head -10
```

Weź **jedną nazwę obiektu** (np. `rbd_data.0000000000000000`):

```bash
OBJ=rbd_data.0000000000000000   # zamień

cephsh osd map testpool $OBJ
```

Output: `osdmap eXXX pool testpool object <OBJ> -> pg <PG> (pool X) -> up [..], acting [..]`

To jest **CRUSH w praktyce**: znasz obiekt → dostajesz PG + OSD.

---

## Krok 5 — wiele obiektów, ta sama PG?

```bash
for o in $(sudo ./cephadm shell -- rados ls -p testpool | head -5); do
  echo -n "$o → "
  cephsh osd map testpool "$o" 2>/dev/null | grep -oP 'pg \K[0-9]+\.[0-9]+'
done
```

Różne obiekty → zwykle **różne PG** (rozłożenie obciążenia).

---

## Krok 6 — stan PG (peering)

```bash
cephsh pg stat
cephsh pg dump pgs_brief | grep active | head -5
cephsh pg dump pgs_brief | grep -v active+clean | head -5
```

Typowe stany: `active+clean`, `active+undersized`, `active+degraded`, `peering`.

Szczegóły jednej PG:

```bash
cephsh pg $PG query 2>/dev/null | head -40
# bez jq — wystarczy pg map + pgs_brief
```

---

## Krok 7 — awaria a konkretna PG (powtórka z rozumieniem)

```bash
# PG które używają osd.0 jako primary lub w acting set
cephsh pg dump pgs_brief | grep -E '\[0,' | head -5
```

Zatrzymaj osd.0:

```bash
cephsh orch daemon stop osd.0
sleep 10
cephsh pg stat
cephsh pg dump pgs_brief | grep degraded | head -5
```

Sprawdź **tę samą PG** co w Kroku 3:

```bash
cephsh pg map $PG
```

Przywróć:

```bash
cephsh orch daemon start osd.0
```

---

## Krok 8 — podsumowanie jednym rzutem

```bash
echo "=== POOL ==="
cephsh osd pool ls detail | grep -E "pool|size|pg_num" | head -20

echo "=== PG STAT ==="
cephsh pg stat

echo "=== PRZYKŁADOWA PG ==="
PG=$(cephsh pg ls-by-pool testpool | head -1 | awk '{print $1}')
echo "PG=$PG"
cephsh pg map $PG

echo "=== PRZYKŁADOWY OBIEKT ==="
OBJ=$(sudo ./cephadm shell -- rados ls -p testpool 2>/dev/null | head -1)
echo "OBJ=$OBJ"
cephsh osd map testpool "$OBJ"
```

---

## Co zapamiętać

```
obiekt --CRUSH--> PG X.Y --acting set--> [primary osd.A, replika osd.B]
                      ↑
              jednostka recovery / scrub / degraded
```

Dane = **obiekty**. PG = **przypisanie obiektów do par OSD**.
