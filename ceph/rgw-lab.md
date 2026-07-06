# Lab RGW — Tydzień 2 (S3 / object storage)

**RGW** (RADOS Gateway) = API **S3** i Swift na tym samym RADOS co RBD.  
Zamiast „dysku blokowego” — **buckety i obiekty** (jak AWS S3).

Środowisko: klaster `ceph-lab`, `~/nauka/ceph`.

```bash
cd ~/nauka/ceph
alias cephsh='sudo ./cephadm shell -- ceph'
alias rgwadmin='sudo ./cephadm shell -- radosgw-admin'
```

---

## RBD vs RGW — jednym zdaniem

| | RBD | RGW |
|---|-----|-----|
| Interfejs | dysk blokowy (`/dev/rbd0`) | HTTP S3 (bucket/key) |
| Jednostka | obraz (`testpool/demo`) | bucket + obiekt (`s3://moj-bucket/plik.txt`) |
| Typowy klient | kernel, QEMU | `aws cli`, `s3cmd`, aplikacje S3 |
| Pod spodem | ten sam RADOS (obiekty, PG, OSD) | ten sam RADOS |

---

## Krok 1 — wdroż RGW

```bash
cephsh orch apply rgw myrgw --placement="1 ceph-lab"
```

Czekaj aż daemon wstanie:

```bash
watch -n3 'cephsh orch ps | grep rgw'
# STATUS: running
```

Sprawdź endpoint (port i URL):

```bash
cephsh mgr services
cephsh orch ps --service_name rgw.myrgw --format json-pretty | head -30
```

Typowo RGW nasłuchuje na hoście `ceph-lab` na porcie **80** (HTTP).  
Zapisz: `http://192.168.0.200` (Twój statyczny IP).

---

## Krok 2 — użytkownik S3

```bash
rgwadmin user create --uid=kanaboid --display-name="Kanaboid Lab"
```

**Zapisz output:** `access_key` i `secret_key` (jak AWS credentials).

Lista użytkowników:

```bash
rgwadmin user list
rgwadmin user info --uid=kanaboid
```

---

## Krok 3 — klient S3 na VM

```bash
sudo apt install -y awscli
# lub: sudo apt install -y s3cmd
```

### AWS CLI

```bash
aws configure --profile ceph
# AWS Access Key ID:     <access_key z kroku 2>
# AWS Secret Access Key: <secret_key>
# Default region:        us-east-1   (dowolne — RGW ignoruje)
# Default output:        json

export AWS_PROFILE=ceph
export ENDPOINT=http://192.168.0.200   # Twój IP + port jeśli inny
```

Operacje:

```bash
aws --endpoint-url $ENDPOINT s3 mb s3://lab-bucket
aws --endpoint-url $ENDPOINT s3 ls
echo "Hello S3 $(date)" > /tmp/hello-s3.txt
aws --endpoint-url $ENDPOINT s3 cp /tmp/hello-s3.txt s3://lab-bucket/
aws --endpoint-url $ENDPOINT s3 ls s3://lab-bucket/
aws --endpoint-url $ENDPOINT s3 cp s3://lab-bucket/hello-s3.txt -
```

---

## Krok 4 — co się dzieje pod spodem?

```bash
# obiekty RGW w pulach systemowych (po pierwszym użyciu)
sudo ./cephadm shell -- rados ls -p .rgw.buckets.data 2>/dev/null | head -5
cephsh osd pool ls | grep rgw
cephsh -s
```

Zapis do S3 → RGW → **obiekty RADOS** w pulach RGW → PG → OSD (ta sama ścieżka co RBD).

---

## Krok 5 — porównanie z RBD (ćwiczenie)

| Co zrobiłeś | RBD | RGW |
|-------------|-----|-----|
| Utworzenie | `rbd create testpool/demo` | `s3 mb s3://lab-bucket` |
| Zapis | plik na `/mnt/ceph-demo` | `s3 cp plik s3://bucket/` |
| Nazwa w RADOS | `rbd_data.*` w `testpool` | hashe w `.rgw.buckets.data` |
| Widok | `rbd ls testpool` | `s3 ls` |

---

## Krok 6 — status i logi

```bash
cephsh orch ps --service_name rgw.myrgw
sudo ./cephadm logs --name rgw.myrgw.ceph-lab.<id>  # id z orch ps
```

---

## Porządki (opcjonalnie)

```bash
aws --endpoint-url $ENDPOINT s3 rb s3://lab-bucket --force
rgwadmin user rm --uid=kanaboid
cephsh orch rm rgw.myrgw
```

---

## Typowe problemy

| Problem | Rozwiązanie |
|---------|-------------|
| `Connection refused` na :80 | `ceph mgr services`; sprawdź firewall; `orch ps` czy `running` |
| `AccessDenied` | złe klucze; `user info --uid=...` |
| RGW nie startuje | `cephadm logs`; sprawdź RAM (`ceph -s`) |
| Port 80 zajęty | `ceph orch apply rgw myrgw --placement="1 ceph-lab" --port=8080` |

---

## Pojęcia do zapamiętania

| Pojęcie | Znaczenie |
|---------|-----------|
| **uid** | użytkownik RGW (nie to samo co Linux) |
| **access_key / secret_key** | credentials S3 |
| **bucket** | kontener na obiekty (jak „folder” w S3) |
| **object key** | nazwa pliku w buckecie |
| **endpoint** | URL RGW (`http://IP:port`) |

---

Powrót: [podsumowanie-laba.md](podsumowanie-laba.md) · Teoria: [sciagawka.md](sciagawka.md)
