# Podsumowanie laba Ceph + fiszka komend

Materiał do powtórki po Tygodniu 1 (klaster) i części Tygodnia 2 (RBD).  
Środowisko: **VM `ceph-lab`**, Ubuntu Server 24.04, VMware Workstation (Win11), statyczny IP LAN.

Powiązane pliki: [sciagawka.md](sciagawka.md) (teoria) · [lab.md](lab.md) (cephadm) · [rbd-lab.md](rbd-lab.md) (RBD) · [diagram-zapisu.md](diagram-zapisu.md)

---

## Co zrobiliśmy — chronologia

### Infrastruktura
- [x] VM: 4 vCPU, 8 GiB RAM, dysk 50 GiB + 3× 20 GiB (sdb/sdc/sdd)
- [x] Sieć bridged w VMware (VMnet0) — DHCP nie działał → **statyczny IP** (np. `192.168.0.200/24`)
- [x] Docker, chrony, lvm2, git

### Tydzień 1 — klaster cephadm
- [x] Pobranie `cephadm` (Reef 18.2.7 z `download.ceph.com`)
- [x] Bootstrap: `--mon-ip`, `--single-host-defaults`, `--skip-firewalld`, `--skip-monitoring-stack`
- [x] 3 OSD na `/dev/sdb`, `/dev/sdc`, `/dev/sdd` (zap + `orch daemon add osd`)
- [x] Pula `testpool` (32 PG)
- [x] Symulacja awarii: `stop osd.0` → `degraded` / `undersized` → `start osd.0` → recovery

### Tydzień 2 — RBD (częściowo)
- [x] Obraz `testpool/demo` (1 GiB)
- [x] `rbd map` → `/dev/rbd0` → `mkfs.ext4` → mount `/mnt/ceph-demo`
- [x] Zapis pliku, persystencja po unmap/map
- [x] Snapshot `demo@snap1`, rollback (po `umount` + `unmap`)
- [x] Klon: `snap protect` → `clone` → `demo-clone`
- [x] Porządki: `rm` klona → `snap unprotect` (kolejność!)

### Jeszcze przed nami
- [ ] RGW (S3)
- [ ] CRUSH / PG / rebalancing (Tydzień 3)
- [ ] Monitoring, backup (Tydzień 4)

---

## Kluczowe pojęcia z labu

### Hierarchia

```
Klaster (fsid)
└── testpool              ← pula: polityka replikacji, PG, CRUSH
    └── demo              ← obraz RBD (wirtualny dysk 1 GiB)
        └── obiekty RADOS (striping)
            └── PG → osd.0, osd.1, osd.2
```

| Nazwa | Czym jest | Twoja wartość |
|-------|-----------|---------------|
| **testpool** | Pula — kontener z regułami storage | 32 PG, `size=2`, app `rbd` |
| **demo** | Obraz RBD w puli | 1 GiB, mapowany jako `/dev/rbd0` |
| **demo@snap1** | Snapshot obrazu w chwili T | copy-on-write, nie kopiuje całego 1 GiB |
| **demo-clone** | Osobny obraz sklonowany ze snapa | zależność od `snap1` (protect) |

### Pool vs obraz — jednym zdaniem
- **testpool** = *gdzie* i *jak* przechowywane są dane (replikacja, PG)
- **demo** = *konkretny dysk* w tej puli

### Stany PG (widziane w labie)
| Stan | Kiedy |
|------|-------|
| `active+clean` | OK, repliki zsynchronizowane |
| `active+undersized` | Mniej replik niż `size` (np. osd down, size=2) |
| `active+undersized+degraded` | Brakuje kopii + za mało OSD |

---

## Fiszka komend

> Wszystkie komendy Ceph/RBD z katalogu `~/nauka/ceph`.  
> Klient `rbd` **nie jest na hoście** — używaj `sudo ./cephadm shell --`.

```bash
cd ~/nauka/ceph
export HOST=$(hostname)   # ceph-lab
```

### Alias (opcjonalnie)

```bash
alias cephsh='sudo ./cephadm shell -- ceph'
alias rbdsh='sudo ./cephadm shell -- rbd'
```

---

### Klaster — status i diagnostyka

```bash
sudo ./cephadm shell -- ceph -s
sudo ./cephadm shell -- ceph health detail
sudo ./cephadm shell -- ceph osd tree
sudo ./cephadm shell -- ceph osd pool ls
sudo ./cephadm shell -- ceph osd pool ls detail
sudo ./cephadm shell -- ceph pg stat
sudo ./cephadm shell -- ceph df
sudo ./cephadm ls
```

### Pule

```bash
# lista pul
sudo ./cephadm shell -- ceph osd pool ls detail

# utworzenie puli (pg_num, pgp_num)
sudo ./cephadm shell -- ceph osd pool create testpool 32 32

# włączenie aplikacji (usuwa WARN)
sudo ./cephadm shell -- ceph osd pool application enable testpool rbd
```

### OSD — dodawanie i awaria

```bash
# lista dysków dostępnych dla OSD
sudo ./cephadm shell -- ceph orch device ls

# dodanie OSD
sudo ./cephadm shell -- ceph orch device zap $HOST /dev/sdb --force
sudo ./cephadm shell -- ceph orch daemon add osd ${HOST}:/dev/sdb

# symulacja awarii
sudo ./cephadm shell -- ceph orch daemon stop osd.0
sudo ./cephadm shell -- ceph orch daemon start osd.0
```

### RBD — obraz blokowy

```bash
# utworzenie / lista / info
sudo ./cephadm shell -- rbd create testpool/demo --size 1G
sudo ./cephadm shell -- rbd ls testpool
sudo ./cephadm shell -- rbd info testpool/demo

# mapowanie (tworzy /dev/rbd0 na hoście)
sudo modprobe rbd
sudo ./cephadm shell -- rbd map testpool/demo
lsblk | grep rbd

# system plików (na hoście, nie w shellu)
sudo mkfs.ext4 /dev/rbd0
sudo mkdir -p /mnt/ceph-demo
sudo mount /dev/rbd0 /mnt/ceph-demo

# odmontowanie
sudo umount /mnt/ceph-demo
sudo ./cephadm shell -- rbd unmap testpool/demo
```

### RBD — snapshot i klon

```bash
# snapshot (obraz może być zamapowany)
sudo ./cephadm shell -- rbd snap create testpool/demo@snap1
sudo ./cephadm shell -- rbd snap ls testpool/demo

# rollback — WYMAGA odmapowania!
sudo umount /mnt/ceph-demo
sudo ./cephadm shell -- rbd unmap testpool/demo
sudo ./cephadm shell -- rbd snap rollback testpool/demo@snap1
sudo ./cephadm shell -- rbd map testpool/demo
sudo mount /dev/rbd0 /mnt/ceph-demo

# klon (wymaga protect)
sudo ./cephadm shell -- rbd snap protect testpool/demo@snap1
sudo ./cephadm shell -- rbd clone testpool/demo@snap1 testpool/demo-clone
sudo ./cephadm shell -- rbd children testpool/demo@snap1

# usuwanie — kolejność!
sudo ./cephadm shell -- rbd unmap testpool/demo-clone
sudo ./cephadm shell -- rbd rm testpool/demo-clone
sudo ./cephadm shell -- rbd snap unprotect testpool/demo@snap1
sudo ./cephadm shell -- rbd snap rm testpool/demo@snap1
```

---

## Pułapki z labu (ważne!)

| Problem | Przyczyna | Rozwiązanie |
|---------|-----------|-------------|
| Brak IP w VM (bridged) | VMware / Wi‑Fi / Hyper-V | Statyczny IP w netplan |
| `rbd: command not found` na hoście | Brak `ceph-common` | `sudo ./cephadm shell -- rbd ...` |
| `rollback: Read-only file system` | Obraz nadal zamapowany | `umount` + `rbd unmap`, potem rollback |
| `unprotect: Device busy` | Istnieje klon (`demo-clone`) | Najpierw `rbd rm` klona |
| Dane po rollback zniknęły | Rollback nadpisuje HEAD | Bez `snap2` nie odzyskasz — robić snap przed ryzykiem |
| Wklejanie outputu do terminala | Bash wykonuje linie logów | Wklejaj tylko komendy |
| `sudo cephadm` nie działa | cephadm tylko w `~/nauka/ceph` | `./cephadm` lub `sudo ./cephadm install` |

---

## Bootstrap — komenda referencyjna

```bash
cd ~/nauka/ceph
MON_IP=192.168.0.200   # Twój statyczny IP

sudo ./cephadm bootstrap \
  --mon-ip $MON_IP \
  --single-host-defaults \
  --skip-firewalld \
  --skip-monitoring-stack
```

**fsid klastra (Twój lab):** `0a91220a-7634-11f1-82fb-000c299ee879`

---

## Ścieżka zapisu — RBD (co ćwiczyliśmy)

```
test.txt
  → ext4 na /mnt/ceph-demo
  → /dev/rbd0
  → obraz testpool/demo (klient RBD)
  → CRUSH → PG w testpool
  → primary OSD → replika na drugi OSD (size=2)
```

---

## Szybka powtórka „na pamięć”

1. **MON** trzyma mapę, **OSD** trzyma dane, **MGR** — dashboard/metryki.
2. **Pool** = polityka; **PG** = jednostka replikacji; **OSD** = dysk.
3. **testpool** = Twoja pula; **demo** = Twój dysk RBD w puli.
4. Tylko **primary OSD** przyjmuje zapisy.
5. `ceph -s` — pierwsza komenda przy każdym problemie.
6. RBD: **unmap** przed rollback; **rm clone** przed unprotect.

---

## Następny krok nauki

**RGW (S3)** — ten sam RADOS, interfejs obiektowy zamiast blokowego:

```bash
sudo ./cephadm shell -- ceph orch apply rgw myrgw --placement="1 ceph-lab"
```

Powodzenia!
