# Fiszka komend — Ceph lab

Wszystko z `~/nauka/ceph` na hoście `ceph-lab`.

```bash
cd ~/nauka/ceph
alias cephsh='sudo ./cephadm shell -- ceph'
alias rbdsh='sudo ./cephadm shell -- rbd'
alias rados='sudo ./cephadm shell -- rados'
alias rgwadmin='sudo ./cephadm shell -- radosgw-admin'
```

---

## cephadm — opakowanie

| Komenda | Co robi |
|---------|---------|
| `sudo ./cephadm shell` | powłoka z narzędziami Ceph |
| `sudo ./cephadm shell -- ceph -s` | jedna komenda bez wchodzenia w shell |
| `sudo ./cephadm ls` | kontenery Ceph na hoście |
| `sudo ./cephadm logs --name <daemon>` | logi daemona |
| `sudo ./cephadm install` | cephadm + ceph w PATH (opcjonalnie) |

---

## ceph — klaster

### Status

```bash
cephsh -s
cephsh health detail
cephsh df
cephsh versions
cephsh mgr services          # URL dashboardu, RGW
```

### OSD i topologia

```bash
cephsh osd tree
cephsh osd stat
cephsh osd df
cephsh device ls             # dyski z health
```

### Pule

```bash
cephsh osd pool ls
cephsh osd pool ls detail
cephsh osd pool create <nazwa> <pg_num> <pgp_num>
cephsh osd pool application enable <nazwa> rbd    # lub rgw
cephsh osd pool get <nazwa> all
```

### PG (placement groups)

```bash
cephsh pg stat
cephsh pg dump pgs_brief | head -20
cephsh pg ls-by-pool testpool
cephsh pg map <pool_id.pg>                    # np. 2.7
cephsh osd map testpool <nazwa_obiektu>         # obiekt → PG → OSD
```

### Orchestrator (cephadm)

```bash
cephsh orch device ls
cephsh orch device zap <host> /dev/sdX --force
cephsh orch daemon add osd <host>:/dev/sdX
cephsh orch daemon stop osd.0
cephsh orch daemon start osd.0
cephsh orch ps
cephsh orch apply rgw <nazwa> --placement="1 ceph-lab"
cephsh orch rm rgw.<nazwa>
```

---

## rbd — dysk blokowy

```bash
rbdsh create <pool>/<obraz> --size 1G
rbdsh ls <pool>
rbdsh info <pool>/<obraz>
rbdsh map <pool>/<obraz>              # → /dev/rbd0 na hoście
rbdsh unmap <pool>/<obraz>

# na hoście (nie w kontenerze)
sudo mkfs.ext4 /dev/rbd0
sudo mount /dev/rbd0 /mnt/ceph-demo
sudo umount /mnt/ceph-demo

# snapshot
rbdsh snap create <pool>/<obraz>@<snap>
rbdsh snap ls <pool>/<obraz>
rbdsh snap rollback <pool>/<obraz>@<snap>   # wymaga unmap!
rbdsh snap protect <pool>/<obraz>@<snap>
rbdsh snap unprotect <pool>/<obraz>@<snap>  # po usunięciu klona
rbdsh snap rm <pool>/<obraz>@<snap>

# klon
rbdsh clone <pool>/<obraz>@<snap> <pool>/<klon>
rbdsh children <pool>/<obraz>@<snap>
rbdsh rm <pool>/<obraz>
```

---

## rados — surowy RADOS (obiekty)

```bash
rados -p <pool> ls                          # lista obiektów
rados -p testpool ls | head
rados -p <pool> put <obj> <plik_lokalny>    # zapis obiektu
rados -p <pool> get <obj> <plik_lokalny>    # odczyt
rados -p <pool> rm <obj>                    # usuń
rados df                                    # zużycie per pool
```

Pule RGW: `.rgw.root`, `.rgw.buckets.data`, itd.

```bash
rados -p .rgw.buckets.data ls | head
```

---

## radosgw-admin — użytkownicy S3

```bash
rgwadmin user create --uid=<id> --display-name="Nazwa"
rgwadmin user list
rgwadmin user info --uid=<id>
rgwadmin user modify --uid=<id> --caps='buckets=*;users=*;metadata=*'
rgwadmin key create --uid=<id> --key-type=s3
rgwadmin user rm --uid=<id>
```

---

## s3cmd — klient S3 (na hoście VM)

Plik: `~/.s3cfg` — `host_base = 192.168.0.200`, `use_https = False`, `signature_v2 = True`

```bash
s3cmd ls
s3cmd mb s3://<bucket>
s3cmd rb s3://<bucket>              # usuń bucket (pusty)
s3cmd rb s3://<bucket> --force      # z zawartością
s3cmd put <plik> s3://<bucket>/
s3cmd get s3://<bucket>/<klucz> <plik>
s3cmd ls s3://<bucket>/
s3cmd del s3://<bucket>/<klucz>
```

---

## Szybka mapa: co czym

| Narzędzie | Warstwa | Typowe użycie |
|-----------|---------|---------------|
| **ceph** | administracja klastra | status, pule, PG, OSD |
| **rbd** | blok | dyski VM, mount |
| **rados** | obiekt (niskopoziomowy) | debug, bezpośredni zapis obiektów |
| **radosgw-admin** | S3 users | klucze access/secret |
| **s3cmd** | klient S3 | buckety, pliki przez HTTP |
| **cephadm** | wdrożenie | bootstrap, kontenery, orch |

---

## Najczęściej używane (top 10)

```bash
cephsh -s
cephsh osd tree
cephsh osd pool ls detail
cephsh pg stat
rbdsh ls testpool
rbdsh map testpool/demo
rados -p testpool ls | head
rgwadmin user info --uid=kanaboid
s3cmd ls s3://lab-bucket/
cephsh mgr services
```

---

Zobacz też: [sciagawka.md](sciagawka.md) (teoria) · [podsumowanie-laba.md](podsumowanie-laba.md) (przebieg labu)
