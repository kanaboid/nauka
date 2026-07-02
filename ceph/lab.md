# Lab Ceph — Tydzień 1 (cephadm, jeden host)

Cel: postawić działający klaster na **jednej maszynie Linux**, przećwiczyć komendy ze ściągawki i zobaczyć stany PG na żywo.

> **Uwaga:** to lab edukacyjny, nie produkcja. Na jednym hoście używamy `--single-host-defaults` (replikacja size=2, brak wymogu racków).

---

## Lab na Windows 11 (zdalny dostęp)

**Ceph nie działa natywnie na Windows.** `cephadm` wymaga Linuxa: systemd, Docker/Podman, LVM2.

| Opcja | Verdict |
|-------|---------|
| **Linux VM** (Hyper-V / VirtualBox) | **Zalecane** — najbliżej produkcji |
| **WSL2** | **Nie polecam** — problemy z systemd, LVM, loop, uprzywilejowany Docker |
| **Dual-boot Linux** | OK, jeśli nie chcesz VM |

### Schemat pracy zdalnej

```
Twój Linux (Mint) ──SSH/RDP──► Win11 ──► VM z Ubuntu Server
                              lub SSH bezpośrednio do VM (sieć bridged)
```

### Krok W1 — utwórz VM

**Specyfikacja minimalna:**

| Zasób | Minimum | Zalecane |
|-------|---------|----------|
| vCPU | 4 | 4 |
| RAM | 8 GiB | 12–16 GiB |
| Dysk systemowy | 40 GiB | 50 GiB |
| Dyski OSD | 3× 10 GiB (osobne wirtualne dyski) | 3× 20 GiB |
| OS | Ubuntu Server 24.04 LTS | ten sam |

**Hyper-V** (Win11 Pro/Enterprise): Menedżer Hyper-V → Nowa maszyna wirtualna → Generation 2 → pamięć dynamiczna wyłączona (stałe 8+ GiB).

**VirtualBox** (działa też na Win11 Home): pobierz [Ubuntu Server 24.04](https://ubuntu.com/download/server), zamontuj ISO, zainstaluj z SSH server (zaznacz przy instalacji).

**Sieć — ważne:** ustaw adapter na **Bridged** (mostkowany), żeby VM dostała IP z Twojej sieci LAN (np. `192.168.1.x`). Tego IP użyjesz w `--mon-ip`.

### Krok W2 — przygotuj Ubuntu w VM

Po instalacji, w VM (przez konsolę Hyper-V/VirtualBox lub SSH):

```bash
# sprawdź IP — zapisz go na bootstrap
ip -4 addr show | grep inet

# aktualizacja + wymagania cephadm
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl docker.io lvm2 chrony openssh-server

sudo systemctl enable --now docker chrony
sudo usermod -aG docker $USER   # wyloguj i zaloguj ponownie

# sklonuj materiały (albo skopiuj cephadm ręcznie)
git clone <twoje-repo> ~/nauka   # lub scp z maszyny Mint
```

Sprawdź, czy widać dyski OSD (jeśli dodałeś 3 osobne wirtualne dyski):

```bash
lsblk
# powinny być np. sdb, sdc, sdd (puste, bez partycji)
```

### Krok W3 — zdalna praca z Mint

Jeśli VM ma IP bridged (np. `192.168.1.50`):

```bash
# z Twojej maszyny Mint — bez RDP do Windows
ssh uzytkownik@192.168.1.50
```

Albo: RDP do Win11 → terminal → `ssh localhost` / konsola VM.

**Reszta labu** (Krok 0–5 poniżej) działa identycznie na VM — zamień IP w `--mon-ip` na IP VM.

### OSD na VM: wirtualne dyski vs loop

Jeśli dodałeś **3 osobne dyski** w Hyper-V/VirtualBox (lepsze do nauki):

```bash
HOST=$(hostname)
sudo cephadm shell -- ceph orch device ls
# zap + dodaj każdy dysk, np.:
sudo cephadm shell -- ceph orch device zap $HOST /dev/sdb --force
sudo cephadm shell -- ceph orch daemon add osd ${HOST}:/dev/sdb
# powtórz dla sdc, sdd
```

Jeśli masz tylko jeden dysk systemowy — użyj **Kroku 3** (loop devices) jak w sekcji poniżej.

---

## Audyt hosta Linux (przed bootstrapem)

Na maszynie, gdzie stawiasz klaster, sprawdź:

```bash
ip -4 addr show scope global    # IP do --mon-ip (LAN, nie VPN)
free -h                         # min. ~4 GiB wolne przed bootstrapem
df -h /                         # miejsce na obrazy Docker + OSD
docker info                     # Docker działa
groups                          # użytkownik w grupie docker (opcjonalnie)
```

Przykład audytu (maszyna Mint — **nie** docelowa dla labu):

| Parametr | Wartość | Ocena |
|----------|---------|-------|
| IP LAN | `192.168.1.93` | użyj IP **VM**, nie tego |
| RAM | 7,5 GiB | za mało na produkcyjny host + inne usługi |

---

## Krok 0 — pobierz cephadm

Stary URL z GitHub (`raw/main/src/cephadm/cephadm`) **już nie działa**. Aktualna metoda:

```bash
cd ~/nauka/ceph

CEPH_RELEASE=18.2.7   # Reef (stable) — sprawdź https://docs.ceph.com/en/latest/releases/
curl --silent --remote-name --location \
  https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
./cephadm version
```

Binarka `cephadm` jest już w tym katalogu (pobrana podczas przygotowania materiałów).

---

## Krok 1 — bootstrap klastra

```bash
cd ~/nauka/ceph

# zamień na IP Twojej VM / hosta Linux (ip -4 addr show)
MON_IP=192.168.1.50

sudo ./cephadm bootstrap \
  --mon-ip $MON_IP \
  --single-host-defaults \
  --skip-firewalld \
  --skip-monitoring-stack
```

Co robi każda flaga:

| Flaga | Po co |
|-------|-------|
| `--mon-ip` | adres LAN VM — MON musi być osiągalny z sieci |
| `--single-host-defaults` | `size=2`, `chooseleaf_type=0` — sensowne na jednym hoście |
| `--skip-firewalld` | Mint często nie ma firewalld; bez tego bootstrap może się wyłożyć |
| `--skip-monitoring-stack` | oszczędza RAM (Prometheus/Grafana — masz już Netdata) |

**Zapisz output!** Na końcu zobaczysz:
- URL dashboardu + login/hasło
- komendę `cephadm shell`
- `fsid` klastra

Typowy czas: 5–15 min (pobieranie obrazów Docker).

### Jeśli coś pójdzie nie tak

```bash
# logi bootstrapu
sudo journalctl -u ceph-<fsid>@ceph-mon.$(hostname).service -e

# sprzątanie po nieudanym bootstrapie
sudo ./cephadm rm-cluster --fsid <FSID> --force
```

---

## Krok 2 — pierwsze komendy (Tydzień 1, ćwiczenie A)

```bash
sudo cephadm shell -- ceph -s
sudo cephadm shell -- ceph health detail
sudo cephadm shell -- ceph osd tree
sudo cephadm shell -- ceph osd pool ls detail
sudo cephadm shell -- ceph pg stat
sudo cephadm shell -- ceph df
```

**Co sprawdzić w `ceph -s`:**
- `health: HEALTH_OK` lub `HEALTH_WARN` (na początku często WARN — brak OSD)
- sekcja `mon`, `mgr` — powinny być `up`
- `osd: 0 osds: 0 up, 0 in` — normalne **przed** dodaniem dysków

---

## Krok 3 — dodaj „dyski” (loop devices)

Na tej maszynie nie ma osobnych dysków na OSD. Tworzymy pliki + loop — wystarczy do nauki.

```bash
# 3 wirtualne dyski po 10 GiB
for i in 0 1 2; do
  sudo truncate -s 10G /var/lib/ceph-osd${i}.img
  sudo losetup -f /var/lib/ceph-osd${i}.img
done

# sprawdź które loop powstały
losetup -a | grep ceph-osd
```

Dodaj OSD przez orchestrator (zamień `loopX` na rzeczywiste nazwy z `losetup -a`):

```bash
HOST=$(hostname)

sudo cephadm shell -- ceph orch device ls

# dla każdego loop (przykład):
sudo cephadm shell -- ceph orch device zap $HOST /dev/loop4 --force
sudo cephadm shell -- ceph orch daemon add osd ${HOST}:/dev/loop4

sudo cephadm shell -- ceph orch device zap $HOST /dev/loop5 --force
sudo cephadm shell -- ceph orch daemon add osd ${HOST}:/dev/loop5

sudo cephadm shell -- ceph orch device zap $HOST /dev/loop6 --force
sudo cephadm shell -- ceph orch daemon add osd ${HOST}:/dev/loop6
```

Poczekaj aż PG przejdą w `active+clean`:

```bash
watch -n2 'sudo cephadm shell -- ceph -s'
```

**Ćwiczenie B — mapowanie teorii na praktykę:**

```bash
sudo cephadm shell -- ceph osd tree          # 3 OSD na jednym hoście
sudo cephadm shell -- ceph pg dump pgs_brief | head -20   # stany PG
sudo cephadm shell -- ceph osd pool create testpool 32 32 # mała pula testowa
sudo cephadm shell -- ceph osd pool ls detail
```

---

## Krok 4 — symulacja awarii (Tydzień 1, ćwiczenie C)

```bash
# zatrzymaj jeden OSD
sudo cephadm shell -- ceph orch daemon stop osd.0

sudo cephadm shell -- ceph -s          # powinno pokazać degraded
sudo cephadm shell -- ceph pg stat

# przywróć
sudo cephadm shell -- ceph orch daemon start osd.0

sudo cephadm shell -- ceph -s          # recovering → active+clean
```

To jest żywa wersja diagramów z [diagram-zapisu.md](diagram-zapisu.md).

---

## Krok 5 — dashboard (opcjonalnie)

Jeśli nie użyłeś `--skip-dashboard`, wejdź na URL z outputu bootstrapu.

```bash
sudo cephadm shell -- ceph mgr services
```

---

## Porządki po labie

```bash
# usuń klaster (zamień FSID)
sudo ./cephadm rm-cluster --fsid <FSID> --force

# usuń pliki loop
for i in 0 1 2; do
  sudo losetup -d $(losetup -j /var/lib/ceph-osd${i}.img -n -O NAME) 2>/dev/null
  sudo rm -f /var/lib/ceph-osd${i}.img
done
```

---

## Checklist Tydzień 1

- [ ] Bootstrap zakończony, `ceph -s` pokazuje MON + MGR
- [ ] 3 OSD `up`, pule w stanie `active+clean`
- [ ] Rozumiesz output `ceph osd tree` (host → OSD)
- [ ] Widziałeś `degraded` → `recovering` → `active+clean`
- [ ] Utworzyłeś pulę testową i widzisz PG w `ceph pg stat`

---

## Następny krok (Tydzień 2)

Gdy klaster jest stabilny, wybierz jedną ścieżkę:

**RBD (dysk blokowy):**
```bash
sudo cephadm shell -- rbd create demo --size 1G --pool testpool
sudo cephadm shell -- rbd map demo --pool testpool
# ... mkfs, mount, zapis pliku, rbd unmap
```

**RGW (S3):**
```bash
sudo cephadm shell -- ceph orch apply rgw foo --placement="1 $(hostname)"
```

---

## Szybka ściąga komend

```bash
sudo cephadm shell                          # interaktywna powłoka
sudo cephadm shell -- ceph <komenda>        # jedna komenda
sudo cephadm ls                             # kontenery Ceph na hoście
sudo cephadm logs --name mon.$(hostname)    # logi MON
```

Więcej teorii: [sciagawka.md](sciagawka.md) · Diagramy: [diagram-zapisu.md](diagram-zapisu.md)
