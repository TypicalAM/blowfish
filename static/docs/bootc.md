---
author: Adam Piaseczny
date: dd.mm.YYYY
---

# Bootable containers

Czyli ciekawe podejście do linuksowych systemów "tylko do odczytu".

## Jak mogę zobaczyć tą prezentację?

```sh
go install github.com/maaslalani/slides@v0.9.0
curl -L piaseczny.dev/docs/bootc.md | slides
```

albo

```sh
docker run -it --rm typicalam/presentation-viewer piaseczny.dev/docs/bootc.md
```

## Meta

- Wszystko o czym dziś mówie to wolne oprogramowanie, ale mówić będę raczej szybko

---

# Bootable containers

## `whoami`

- DevSecOps @ Secawa
- PUT - Linux Academy Group
- P.I.W.O.
- Koneser świnek morskich

---

# Bootable containers

## Co to jest?

Bootable containers to transport dla repozytorium OSTree. Koniec.

---

# Bootable containers

## Co to jest?

Super. Idziemy do domu.

---

# OSTree

## Co to jest?

OSTree to system kontroli wersji dla systemów operacyjnych:

- Każda wersja systemu to osobny commit
- Upgrade polega na restarcie systemu i wybrania nowego commitu w bootloaderze
- Aktualizacje systemu są atomowe - wszystko albo nic

---

# OSTree

## Przykładowe polecenia

- `ostree --repo=repo init`
- `ostree --repo=repo commit --branch=foo myfile`
- `ostree --repo=repo refs`
- `ostree --repo=repo ls foo`
- `ostree --repo=repo checkout foo tree-checkout/`

---

# OSTree

## Czemu nie git?

- Git nie śledzi atrybutów rozszerzonych (xattr)
- Git nie śledzi pustych folderów
- W gicie pracujemy i edytujemy sobie drzewo swobodnie, w OSTree checkoutujemy do folderu i lądują tam hardlinki

---

# OSTree

## Desktryptywne zmiany

- "Installed fastfetch"
- "Modified the default kernel args"
- "Introduced plymouth in initramfs"

Notka na temat wprowadzania zmian - model server-side zamiast client-side.

---

# OSTree

## Jak to działa?

Mamy repo OSTree w `/ostree`, ostatnie dwa commity posiadają "deployment", czyli są dodane do bootloadera:
- Fedora 43 commit a60ad395341 (OSTree 0) ma pozycję z argumentem kernela `ostree=/ostree/boot.0/fedora/a60ad395341`
- Fedora 43 commit 2e2f9d38e11 (OSTree 1) ma pozycję z argumentem kernela `ostree=/ostree/boot.0/fedora/2e2f9d38e11`

Podczas rozruchu, jeszcze w initramfs, system znajduje wybrany przez nas commit oraz montuje go jako `/`. Jeżeli ten commit nie działa, to możemy zrestartować maszynę w poprzedni, działający stan. 

---

# OSTree

## Używanie tego w praktyce

W Fedorze istnieje helper o nazwie `rpm-ostree`, pozwalający na ułatwioną pracę z drzewem OSTree i robieniem takich rzeczy jak:

- Instalowanie paczek
- Upgrade systemu
- Rebase na inny remote (możemy dosłownie mieć pobrane 3 dystrybucje naraz i przemieszczać się między nimi przez bootloader, mając ten sam `/home`)

Niestety po każdej operacji potrzebny jest reboot.

---

# OSTree

## Gdzie jest mój home skoro wszystko jest immutable?

Wszystko jest w `/var` i `/etc`, reszta jest read-only:

- `/home/adam` -> `/var/home/adam`
- `/srv` -> `/var/srv`
- `/opt` -> `/var/opt`

Te ścieżki są dynamicznie podmontowane podczas rozruchu systemu. Dużą zaletą tego podejścia jest to, że cały "stan" systemu i wszystkie pliki użytkownika są w `/var`, więc łatwo to backupować.

---

# Bootable containers

## Co to jest?

Bootable containers to transport dla repozytorium OSTree. Koniec.

---

# Bootable containers

## Co to jest?

Super!!!

---

# Bootable containers

## Co to jest?

Ale nie tylko transport - bootc pozwala nam na używanie rejestru obrazów OCI jako remote w OSTree:

[Rysunek](https://piaseczny.dev/posts/bootc/ostree-overview.png)

Oficjalnie wchodzimy w strefę zieloną

---

# Bootable containers

## Przykład użycia

```Dockerfile
FROM quay.io/fedora/fedora-kinoite:43

# Add tailscale to the mix
RUN dnf config-manager addrepo --from-repofile=https://pkgs.tailscale.com/stable/fedora/tailscale.repo
RUN dnf install -y tailscale
RUN systemctl enable tailscaled
# Look for common issues, like stray files in /var
RUN bootc container lint
```

---

# Bootable containers

## Przykład użycia

```bash
docker build -t typicalam/bootc:latest --push .
sudo rpm-ostree rebase ostree-unverified-registry:docker.io/typicalam/bootc:latest
```

Po reboocie będziemy już w naszej nowej dystrybucji. Rebase robi nam lokalny "branch" `docker.io/typicalam/bootc:latest` i każdy następny "rpm-ostree upgrade" ściągnie nowy obraz z rejestru oraz doda go jako nowy commit.

---

# Bootable containers

## Instalacja na metal

**`bootc install to-disk`** — instalacja bezpośrednio z działającego kontenera:

```bash
sudo podman run --rm --privileged \
  --pid=host --security-opt label=type:unconfined_t \
  -v /dev:/dev -v /var/lib/containers:/var/lib/containers \
  localhost/nuclear:latest \
  bootc install to-disk /dev/sda
```

---

# Bootable containers

## Instalacja na metal

---

# Bootable containers

## Podpisywanie obrazów

```bash
# Podpisujemy obraz kluczem prywatnym
cosign sign --key cosign.key docker.io/typicalam/nuclear:latest
```

W `/etc/containers/policy.json` definiujemy politykę weryfikacji podpisu:

```json
{"transports": {
  "docker": {
    "docker.io/typicalam/nuclear": [{
      "type": "sigstoreSigned",
      "keyPath": "/etc/pki/containers/nuclear.pub"
    }]
  }
}}
```

`bootc upgrade` odmówi aktualizacji jeżeli podpis jest nieprawidłowy lub brakujący.

---

# Bootable containers

## Zalety

Teraz mamy:
- Swoją własną dystrubucję Fedory 44 zbudowaną za pomocą prostego Dockerfile
- Łatwy sposób na hostowanie naszej dystrybucji używając infrastruktury OCI
- Jedno źródło prawdy dla naszej maszyny (lub stu maszyn)

Ogólne zalety:
- Rozszerzamy podejście infrastructure as code do maszyn desktopowych
- Łatwe skanowanie obrazów używając gotowych narzędzi bezpieczeństwa
- Brak potrzeby uczenia się nowych, skomplikowanych narzędzi (oczywiście jeżeli znamy dockera).

---

# Bootable containers

## Wady

- System jest domyślnie w trybie read-only - istnieje `sudo ostree admin unlock`, który pozwala nam jednorazowo mieć user-writable overlayfs (zmiany trwają tylko do ponownego uruchomienia).
- System trzeba restartować przy aktualizacjach systemu
- Instalowanie paczek jest całkiem wolne w porównaniu do `dnf` oraz `apt`. Jeszcze wolniej jak robimy to w obrazie kontenerowym.
- Skupione wokół dystrybucji około-RHELowych, chociaż są projekty odpalające archa oraz debiana z bootc.

---

# Bootable containers

## Jak ja tego używam

- Mam repozytorium na [githubie](https://github.com/TypicalAM/nuclear) z CI
- Co tydzień system pulluje nowy obraz i aktualizuje się w nocy
- Backupy są bardzo proste. BTRFS snapshot z całego `/var` co godzinę i przyrostowe backupy na snapshocie używając `restic` 
- Profit!

---

# Bootable containers

## Czy ktoś w ogóle tego używa?

Tak:

- [Fedora Atomic](https://www.fedoraproject.org/atomic-desktops/)
- [Bazzite](https://bazzite.gg/) - Dystrybucja na desktopy oraz steam decki do grania
- [Moje repo](https://github.com/TypicalAM/nuclear) - Przykład mojego użycia w praktyce
- [Bluefin](https://projectbluefin.io/) - Dystrybucja bootc dla deweloperów
- [+ mój post z przykładami](https://piaseczny.dev/posts/bootc/)

## EOF
