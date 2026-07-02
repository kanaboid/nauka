# Lab RBD — Tydzień 2

**RBD** (RADOS Block Device) = wirtualny dysk blokowy w puli Ceph. Duży „dysk” to wiele obiektów RADOS ze stripingiem.

Wymagania: działający klaster z Tygodnia 1, pula `testpool`, `cephadm` w `~/nauka/ceph`.

---

## Przygotowanie

```bash
cd ~/nauka/ceph
export HOST=$(hostname)

# włącz moduł jądra (mapowanie RBD)
sudo modprobe rbd

# aplikacja puli (usuwa WARN w ceph -s)
sudo ./cephadm shell -- ceph osd pool application enable testpool rbd
```

---

## Ćwiczenie 1 — utwórz i obejrzyj obraz

```bash
# obraz 1 GiB w puli testpool
sudo ./cephadm shell -- rbd create testpool/demo --size 1G

sudo ./cephadm shell -- rbd ls testpool
sudo ./cephadm shell -- rbd info testpool/demo
```

W `rbd info` zwróć uwagę na:
- **order** — wielkość stripe (domyślnie 22 = 4 MiB)
- **size** — rozmiar widoczny dla klienta
- obraz = wiele obiektów w RADOS (nie jeden plik)

---

## Ćwiczenie 2 — zamapuj obraz jako dysk blokowy

Mapowanie tworzy urządzenie `/dev/rbd0` (lub `rbd1`, `rbd2`…).

```bash
# mapuj (w kontenerze cephadm z dostępem do hosta)
sudo ./cephadm shell -- rbd map testpool/demo
```

Jeśli `rbd map` w shellu nie działa, na hoście VM:

```bash
sudo apt install -y ceph-common
sudo rbd map testpool/demo --cluster ceph -m $(sudo ./cephadm shell -- ceph mon dump 2>/dev/null | grep -oP 'v1:\K[0-9.]+' | head -1)
# prościej — jeśli /etc/ceph istnieje po bootstrapie:
sudo rbd map testpool/demo
lsblk | grep rbd
```

Powinieneś zobaczyć np. `rbd0` ~ 1G.

---

## Ćwiczenie 3 — formatuj, zamontuj, zapisz plik

```bash
sudo mkfs.ext4 /dev/rbd0
sudo mkdir -p /mnt/ceph-demo
sudo mount /dev/rbd0 /mnt/ceph-demo

echo "Hello from RBD $(date)" | sudo tee /mnt/ceph-demo/test.txt
cat /mnt/ceph-demo/test.txt
df -h /mnt/ceph-demo
```

**Co się dzieje pod spodem:** zapis do `/mnt/ceph-demo/test.txt` → kernel RBD → klient Ceph → CRUSH → PG → primary OSD → replikacja.

---

## Ćwiczenie 4 — odmontuj i odmapuj

```bash
sudo umount /mnt/ceph-demo
sudo rbd unmap testpool/demo
# lub: sudo rbd unmap /dev/rbd0

lsblk | grep rbd   # pusto
```

Dane zostają w puli — obraz `demo` nadal istnieje:

```bash
sudo ./cephadm shell -- rbd ls testpool
```

Ponowne `rbd map` + `mount` — plik `test.txt` nadal tam jest.

---

## Ćwiczenie 5 — snapshot (backup w czasie)

```bash
sudo ./cephadm shell -- rbd snap create testpool/demo@snap1
sudo ./cephadm shell -- rbd snap ls testpool/demo

# zapisz coś nowego na zamontowanym dysku, potem porównaj z klona snapa
```

Snapshot = lista obiektów w chwili T; **copy-on-write** — nie kopiuje całego 1 GiB od razu.

```bash
# klon snapa → nowy obraz (opcjonalnie, zaawansowane)
sudo ./cephadm shell -- rbd clone testpool/demo@snap1 testpool/demo-clone
sudo ./cephadm shell -- rbd ls testpool
```

---

## Ćwiczenie 6 — benchmark (opcjonalnie)

```bash
sudo ./cephadm shell -- rbd bench --pool testpool --io-type write demo-bench --io-size 4M --io-threads 4 --io-total 100M
sudo ./cephadm shell -- rbd rm testpool/demo-bench
```

---

## Ćwiczenie 7 — RBD a stan klastra

Podczas zapisu na zamontowanym RBD, w drugim terminalu:

```bash
watch -n2 'sudo ./cephadm shell -- ceph -s'
sudo ./cephadm shell -- ceph osd pool stats testpool
```

---

## Porządki

```bash
sudo umount /mnt/ceph-demo 2>/dev/null
sudo rbd unmap testpool/demo 2>/dev/null

sudo ./cephadm shell -- rbd snap purge testpool/demo
sudo ./cephadm shell -- rbd rm testpool/demo
sudo ./cephadm shell -- rbd rm testpool/demo-clone 2>/dev/null
```

---

## Pojęcia do zapamiętania

| Pojęcie | Znaczenie |
|---------|-----------|
| **Image** | obraz RBD (= wirtualny dysk) |
| **Pool** | polityka replikacji / PG |
| **Striping** | obraz → wiele obiektów RADOS (szybszy równoległy I/O) |
| **Map** | podłączenie obrazu jako `/dev/rbdX` |
| **Exclusive lock** | jeden writer na obraz (ważne przy VM) |
| **Snapshot** | chwilowy stan, COW |

---

## Typowe problemy

| Problem | Rozwiązanie |
|---------|-------------|
| `modprobe: FATAL: Module rbd not found` | `sudo apt install linux-modules-extra-$(uname -r)` lub inny kernel |
| `rbd: sysfs write failed` | `sudo modprobe rbd`; mapuj z uprawnieniami root |
| `device still in use` | `sudo umount` przed `rbd unmap` |
| brak `/etc/ceph` na hoście | użyj `sudo ./cephadm shell -- rbd map` albo `sudo ./cephadm install` |

---

Następny krok po RBD: [RGW / S3](rgw-lab.md) (Tydzień 2, ścieżka B).
