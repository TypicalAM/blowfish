---
author: Adam Piaseczny
date: dd.mm.YYYY
---

# Docker - Sekcja RED 2

Eksploracja izolacji w środowisku wykonywania docker, wykorzystanie podatności wynikających z przyznania nadmiernych uprawnień. 

## Jak dostać się do tej prezentacji

```
go install github.com/maaslalani/slides@v0.8
curl -L piaseczny.dev/docs/docker-red-2.md | slides
```

albo bez użycia `go`, obchodząc `eduroam`

```sh
ssh -p imap red.piaseczny.dev
```

## Meta

- Data: 05.12.2024
- Przerywaj mi jak masz pytania, lubię o tym gadać

---

# Rozkład jazdy

## Teoria

- Izolacja
- Przestrzenie nazw (`namespaces`)
- Grupy kontrolne (`cgroups`)
- Możliwości (`capabilities`)
- Kontenery uprzywilejowane
- Docker socket
- Container breakout

## Praktyka

- Ucieczka z dockera (`ssh`)

---

# Izolacja

Izolacja w Dockerze to mechanizm oddzielający kontenery od siebie oraz od systemu operacyjnego hosta.

## Zalety

- Niezależność między kontenerami
- Ograniczenie powierzchni ataku (attack surface)
- Skalowalność rozwiązań

Mimo ogólności środowiska wykonywania w środowiskach produkcyjnych powinniśmy konfigurować dodatkowe opcje zabezpieczeń. Spójrzmy na poszczególne mechanizmy i możliwości ich nadużycia.

---

# Przestrzenie nazw

Mechanizm jądra Linuksa izolujący zasoby dla procesów w kontenerze, ogranicza to co kontener "widzi".

```sh
man namespaces
```

## Rodzaje przestrzeni nazw
- PID (Process ID): Każdy kontener ma własne ID procesów.
- NET (Networking): Każdy kontener ma odrębny stos sieciowy.
- MNT (Mount): Izolacja systemów plików.
- UTS (Unix Timesharing): Unikalne nazwy hostów dla kontenerów.
- IPC (Interprocess Communication): Izolacja pamięci współdzielonej.
- USER: Izolacja identyfikatorów użytkowników.

## Zmiana przestrzeni nazw kontenera 

```sh
man docker run
```

---

# Przestrzenie nazw

## Zmiana przestrzeni nazw kontenera 

### Przetrzeń `PID`

Zobaczenie procesów kontenera na hoście

```sh
sleep 1234 # Na hoście
docker run -it --rm ubuntu:24.04 ps aux | grep sleep
```

vs

```sh
sleep 1234 # Na hoście
docker run -it --rm --pid=host ubuntu:24.04 ps aux | grep sleep
```

Najczęściej jednak modyfikowana jest przestrzeń nazw `NET` używajać `--net=host`, aby pozwolić kontenerowi wystawiać porty na hoście. To rozwiązanie nie pozwala jednak pingować urządzień w sieci lokalnej hosta.

---

# Grupy kontrolne

Ograniczają ile zasobów dany kontener może użyć, powstrzymują ataki typu DDoS.

```
man cgroups
```

## Rodzaje grup kontrolnych

- CPU
- RAM
- IO dysku
- Sieć (limity przepustowości)

---

# Grupy kontrolne

W celu symulacji obciążenia procesora/pamięci/IO większość systemów *NIX (w tym Linux) udostępnia przydatne narzędzie o nazwie `stress`. Przykład nadmiernej alokacji pamięci w kontenerze używając `stress`:

## Stress testing

```Dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y && apt-get install stress
```

```sh
docker build -t demo . && docker run --rm -it demo stress --vm 1 --vm-bytes 512M --vm-hang 100
```

Argumenty do `stress`: 
- `--vm 1` - Jeden wątek
- `--vm-bytes 512M` - Zaalokuj `512MB` pamięci w każdym wątku
- `--vm-hang 100` - Poczekaj 100 sekund przed realokacją

---

# Grupy kontrolne

## Stress testing

Jak widać kontener może sobie pozwolić na zabranie 512 megabajtów pamięci, teraz spróbujmy uruchomić go ograniczając zasoby kontenerowi używając `cgroups`:

```sh
docker run --rm -it --memory=300MB demo stress --vm 1 --vm-bytes 512M --vm-hang 100
```

Zasoby używane przez kontenery można sprawdzić używając polecenia `docker stats`

```
CONTAINER ID   NAME             CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O        PIDS
a76d0c8b46ce   elastic_jepsen   0.00%     299.8MiB / 300MiB   99.94%    2.49kB / 0B   98.3kB / 224MB   2
```

Dodatkowe przykłady:

```sh
docker run --rm -it --cpus=3 demo stress --cpu 5
```

---

# Możliwości

Rozbijamy uprawnienia administratora systemu na atomowe jednostki. Zamiast pełnych uprawnień kontener dostaje tylko te wymagane:

```
man capabilities
```

Ta funkcjonalność kernela powoduje, że `root` w kontenerze nie jest równoważny `root`owi na hoście. Zobaczmy domyślne możliwości kontenera w [kodzie źródłowym dockera](https://github.com/moby/moby/blob/master/oci/caps/defaults.go#L6-L19). 

---

# Możliwości

```go
func DefaultCapabilities() []string { return []string {
        "CAP_CHOWN",            // Allows changing the owner of files.
        "CAP_DAC_OVERRIDE",     // Bypasses file read, write, and execute permission checks.
        "CAP_FSETID",           // Allows setting the file system User ID (UID) and Group ID (GID) bits.
        "CAP_FOWNER",           // Allows bypassing permission checks (file owner).
        "CAP_MKNOD",            // Allows the creation of special files using the mknod system call.
        "CAP_NET_RAW",          // Allows the use of RAW and PACKET sockets.
        "CAP_SETGID",           // Allows setting the GID (Group ID) of a process.
        "CAP_SETUID",           // Allows setting the UID (User ID) of a process.
        "CAP_SETFPCAP",         // Allows setting file capabilities.
        "CAP_SETPCAP",          // Allows modifying process capabilities.
        "CAP_NET_BIND_SERVICE", // Allows binding to network ports below 1024.
        "CAP_SYS_CHROOT",       // Allows the use of chroot() system call.
        "CAP_KILL",             // Allows sending signals to processes.
        "CAP_AUDIT_WRITE",      // Allows writing to the audit logs.
}}
```

---

# Możliwości

## Dodawanie i usuwanie możliwości

Kontenerowi można usunąć `capability` za pomocą flagi `--cap-drop=<nazwa>`, wszystkie usuwamy używając `--cap-drop=ALL`. Dodawanie możliwości robimy za pomocą `--cap-add=<nazwa>`.

```sh
docker run --rm -it -p 80:80 \
        --cap-drop=ALL --cap-add=CAP_NET_BIND_SERVICE python:3.13-slim python3 -m http.server 80
```

## Niebezpieczne możliwości

- `CAP_SYS_ADMIN` - Administracja systemu, bardzo wiele dostępnych opcji
- `CAP_SYS_PTRACE` - Debugowanie arbitralnych programów
- `CAP_SYS_BOOT` - Używanie syscalla `kexec_load`

Wspólny element między tymi możliwościami jest taki, że pozwalają na nadanie sobie samemu większych możliwości oraz wyłączenie innych zabezpieczeń i dostanie się do hosta.

---

# Możliwości

Możliwości danego kontenera możemy wyświetlić uzywając programu `capsh`:

```sh
docker run --rm -it ubuntu:24.04
root@18a87c42e0e1:/# apt-get update -y
root@18a87c42e0e1:/# apt-get install -y libcap2-bin
root@18a87c42e0e1:/# capsh --print
```

W sekcji "Current" znajdują się możliwości kontenera:

```
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,
cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep
```

Daje nam to dobry punkt startowy do szukania sposobu ucieczki z kontenera za pomocą nadmiarowych możliwości.

---

# Kontenery uprzywilejowane

Kontekst przywilejów `privileged` zezwala kontenerowi na ominięcie wielu aspektów izolacji. Kontenery uprzywilejowane:
- Posiadają dostęp do wszystkich urządzeń hosta
- Mają wyłączony SELinux oraz AppArmor
- Mają włączone WSZYSTKIE możliwości

W branży mówi się [Priviledged Containers are not containers](https://ericchiang.github.io/post/privileged-containers), bo wyłamanie się z takiego kontenera jest banalnie proste. Możliwości doboru parametrów `docker run` powodują, że NIGDY nie ma prawdziwej potrzeby do uruchamiania uprzywilejowanych kontenerów "na prodzie".

```sh
docker run --rm -it --privileged ubuntu:24.04
```

---

# Docker socket

Program linii poleceń `docker` wykonuje swoje zadania za pomocą komunikacji z serwerem HTTP, który jest osadzony przez demon dockera w pliku `/var/run/docker.sock`. Możemy sprawdzić dostępne obrazy używając `curl`:

```sh
curl --silent --unix-socket /var/run/docker.sock http://localhost/images/json
```

Jeżeli wystawimy ten socket do kontenera, kontener może go użyć do stworzenia innego kontenera z większymi uprawnieniami - na przykład dodać `--priviledged`.

```sh
docker run --rm -it -v /var/run/docker.sock:/exposed.sock typicalam/ubuntu:24.04
root@18a87c42e0e1:/# docker --host unix:///exposed.sock run --rm -it \
        --privileged -v /home:/RED typicalam/ubuntu:24.04
root@ffbf863c07ba:/# ls /RED
adam  agatka
```

Jak widać dostaliśmy uprzywilejowany kontener, oraz mamy dostęp do katalogu domowego HOSTA. Tak się dzieje dlatego, że gniazdo dockera jest tak naprawdę obsługiwane przez docker demon hosta!

---

# Praktyka

## Skok w bok

Twoim zadaniem jest zdobycie flagi `root`, środowisko jest autorskie, więc może się chwilę odpalać:

```sh
ssh -p pop3 redentry@piaseczny.dev # Zastanów się jakie jest hasło ;)
```

Przydatne:

```sh
capsh
find / -name "docker.sock"
cat /run/FLAGA.txt
mount
```

---

# Praktyka

## Cap

Twoim zadaniem jest zdobycie flagi `root`, środowisko jest autorskie, więc może się chwilę odpalać:

```sh
ssh -p pop3 red@piaseczny.dev # Zastanów się jakie jest hasło ;)
```

Przydatne:

```sh
capsh
find / -type s
chroot
mount
```

---

# Koniec

## Dzięki za przejście przez warsztaty!

Najlepszą nauką jest praktyka, więc łatwiej jest zrozumieć jak sam eksperymentujesz!

## Dodatkowe materiały

- [Docker Engine Security](https://docs.docker.com/engine/security)
- [Mitre Att&ck surface](https://attack.mitre.org/matrices/enterprise/containers)
- [Privileged containers aren't containers](https://ericchiang.github.io/post/privileged-containers)
- [CAP_SYS_ADMIN: the new root](https://lwn.net/Articles/486306)
- [Moja prezentacja na Poznań Security Meetup](https://opensecurity.pl/wp-content/uploads/2024/06/hardening-dockera.pdf)
- [Rootless docker](https://docs.docker.com/engine/security/rootless/)
