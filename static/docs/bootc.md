---
author: Adam Piaseczny
date: dd.mm.YYYY
---

# Bootable containers

Czyli ciekawe podejście do "immutable" systemów

## Jak mogę zobaczyć tą prezentację sam

```sh
go install github.com/maaslalani/slides@v0.9.0
curl -L piaseczny.dev/docs/bootc.md | slides
```

albo

```sh
docker run -it --rm typicalam/presentation-viewer piaseczny.dev/docs/bootc.md
```

## Meta

- Dziś: 19.11.2025
- Przerywaj mi - lubię o tym gadać

---

# Moje podejście do konfiguracji

## Stan na dziś

- Zepsułem już wiele maszyn (obecna nazywa się tygrys20 a to autoincrement id)
- Pracuję głównie na dwóch maszynach - thinkpad (mały) oraz lenovo legion 5 (duży)
    - Chcę minimalizować ryzyko bycia w miejscu, gdzie nie mogę działać
- Dotfiles istnieją, ale to nie wystarcza
- Centralizacja bazowego stanu maszyny (nixos-config, ansible playbook, etc)
- Nextcloud na najważniejsze ścieżki
- Używam fedory od ~6 lat

---

# Moje podejście do konfiguracji

## Jakie to ma problemy?

- Zamrożenie stanu systemu jest trudne
    - Overhead full-system backupów (subwolumeny ruchliwe/nieruchliwe)
    - Brak możliwości "specjalizacji" - NVIDIA, wirtualizacja
- Dotfiles mogą ciągnąć ze sobą zbiór wymaganych aplikacji oraz kroków instalacyjnych, a są user-local
- Nextcloud jest wolny przy full-disk pullach i trudno mu ogarnąć szybko zmienne pliki np. `bash_history`
- Nix - błędy są nieczytelne, trudno skonfigurować tak jak się chce bez robienia warstwy translacji DSL konfiguracji apki -> nix. Przeklinam ludzi, którzy nie dają możliwości configu apki w yaml/lua/xml/json/etc. Sparzyłem się na robieniu desktopów NixOS, serwery są OK.
- Chciałbym coś z rozruchem typu "A/B" (albo mechanizmem rollback)

---

# Moje podejście do konfiguracji

## Jak żyć?

Jak zepsuję maszynę (a zepsuję) to potrzebuje zwykle jednego dnia na regenerację plików z nextclouda, instalowanie programów, dotfilesów oraz prasowanie nierównych rogów. Przez ten jeden dzień jestem przyklejony do komputera, na studiach robie SSH do serwera i tam klikam, ale jestem wtedy zależny od internetu. Czasem robie full-partition mirrory (używając `dd`, hell yeah) mając nadzieję, że jak czegoś nie będzie na nextcloudzie to tam zajrze (nie zajrze, zapomnę po tygodniu). 

---

# Fedora atomic i universal blue

## Co to jest?

- Fedora oparta na "atomowej bazie", używająca immutable systemu bazowego, którego nie można ruszać*
- Dalej używa RPM do paczek więc super :D
- DNF nie działa :(
- Ciekawe projekty downstreamowe: "Bazzite" dla steam decka, instalujesz i działa
- Możliwość "przełączania się" między różnymi systemami
- Rollbacki do poprzedniej "atomowej bazy" z poziomu bootloadera
- Btrfs by default

## Czy w chwili uniesienia po godzinie czytania zaryłem thinkpada?

Tak lol

---

# Fedora atomic i universal blue

## Co to ma pod spodem?

Trzy technologie:
- Bootc (czyli "Bootable containers")
- Podman (czyli "Docker niedocker")
- OSTree (czyli ???)

Te trzy rzeczy są ze sobą ściśle powiązane, ale po kolei.

---

# OSTree

## Czyli sposób zarządzania wersjami systemu operacyjnego

- Nasza maszyna ma repozytorium ostree (**mocno** inspirowane gitem)
    - `ostree --repo=repo init`
    - `ostree --repo=repo commit --branch=foo myfile`
    - `ostree --repo=repo refs`
    - `ostree --repo=repo ls foo`
    - `ostree --repo=repo checkout foo tree-checkout/`
- Jak sama nazwa wskazuje to repozytorium ma zarządzać naszym drzewem systemu (`/`)
- Tylko `/etc` and `/var` są writable, reszta nie* (nie ma klasycznego `/home` oraz `/mnt`)
- Initramfs bierze nasze argumenty kernela, znajduje gdzie jest target commit/branch oraz montuje go
- Identyczne pliki w tym samym branchu ostree są hardlinkowane

---

# OSTree

## Atomowość tranzakcji systemowych

- Jak wprowadzamy zmianę w systemie (nowy commit), to możemy z niego stworzyć deployment 
- Ten deployment siedzi jako bootentry w naszym GRUBie z innym argumentem `ostree`
- Przerucamy się do zmienionego systemu podczas następnego rozruchu 
- Z racji tego, że kernel jest zawarty w drzewie ostree jest automatycznie kopiowany, aby mógł go znaleźć bootloader

## A co jak zepsuję?

- Jak coś nie działa to możemy w bootloaderze po prostu wybrać drugą opcję analogicznie do
    - Pokoleń w NixOS (gdzie pokoleń może być tyle co w kombii)
    - Partycji rozruchu A/B w androdzie
- Domyślnie w fedorze atomic deployowane są dwa ostatnie commity, ale można to zmienić, albo na stałe zapamiętać jeden commit (pinning).

---

# OSTree

## Co z `/etc`

`/etc` jest dość specjalną ścieżką, może być modyfikowana przez użytkownika jak i posiadać zmiany instalowane przez paczki. Jeżeli paczka instaluje serwis systemd to musi być jakoś obsłużone po stronie OSTree.
- `ostree admin config-diff` pokazuje zmiany między bazą a naszymi ustawieniami
- Każdy deployment oprócz read-only ostree ma zawartość katalogu `/etc`
- Podczas tworzenia nowego deploymentu jest robiony merge między
    - Katalogiem `/etc` starego deploymentu
    - Katalogiem `/etc` aktualnego systemu
    - Katalogiem `/etc` bazowego OSTree

## Jak jest robiony home? mnt? /usr/local/bin?

---

# OSTree

## Brzmi fajnie co? Ale co z paczkami

Jak już wspomniałem dnf nie działa - zamiast tego jest rpm-ostree czyli warstwa integracyjna między RPM a OSTree (duh).
- Package manager tworzy kolejne commity z nowymi paczkami jak je instalujemy/usuwamy
- Deployuje nowy commit
- Po każdej zmianie trzeba zrobić reboot (łee)
- 3-way merge, checkout i nakładanie paczek kosztują czas, więc instalacja jest toporna

---

# Bootc + rpm-ostree

## Rozprowadzanie read-only części deploymentu jako obraz dockerowy

Bootable containers, czyli kontenery rozruchowe pozwala nam rozprowadzać commity/deploymenty OSTree jako obrazy OCI (dockerowe).

## Zalety

- Obraz można wstawić na rejestr OCI (np DockerHub) i dzielić stan OSTree między różnymi maszynami!
- To pozwala też nam w bardzo łatwy sposób trackować zmiany nie trzymając na jakiś webowym repozytorium OSTree dałego drzewa systemu, tylko używać technik znanych ze świata cloud-native oraz CI/CD do budowania obrazów.
- Nagle możemy używać gita oraz Dockerfile!!!

```sh
sudo rpm-ostree status
sudo rpm-ostree rebase ostree-unverified-registry:docker.piaseczny.dev/machine/tygrys20:latest
```

---

# Bootc + rpm-ostree

## Moje flow teraz

Jeżeli robię zmianę systemową to oceniam czy ma ona wylądować w obrazie bazowym:
- Jeżeli tak to backportuje ją do `Containerfile` na repo, robie rebase u siebie na nową wersję obrazu i mam tą zmianę natywnie (np część mojego etc staje się taka sama jest w bazowym obrazie, więc OSTree już nie będzie jej mergował).
- Jeżeli nie to zostawiam ją jako "specjalizację" mojego systemu. Czyli mój system:
    - Na thinkpadzie to obraz bazowy z OCI registry
    - Na thinkpadzie to obraz bazowy + commity z NVIDIA/virt-manager/zaktualizowanymi paczkami

Jak potrzebuje użyć laptopa to po prostu robię na nim `rpm-ostree rebase`, zaciągam obraz i wszystko jest cacy

---

# Powrót do moich problemów

## Jakie to ma problemy?

- Zamrożenie stanu systemu jest trudne
    - Overhead full-system backupów (subwolumeny ruchliwe/nieruchliwe)
    - Brak możliwości "specjalizacji" - NVIDIA, wirtualizacja
- Dotfiles mogą ciągnąć ze sobą zbiór wymaganych aplikacji oraz kroków instalacyjnych, a są user-local
- Nextcloud jest wolny przy full-disk pullach i trudno mu ogarnąć szybko zmienne pliki np. `bash_history`
- Nix - Nie będę więcej nixa hejtował.
- Chciałbym coś z rozruchem typu "A/B" (albo mechanizmem rollback)

---

# Bootc + rpm-ostree

## Co ssie?

Moje przemyślenia po portowaniu wszystkiego:

- Użytkownicy i drift stanu, brak mechanizmu %post
- Czasy budowania i rebaseowania
- Cachowanie obrazów
- Restarty oraz mounty
- Brak wsparcia dla ESP only bootowania, musiałem oskryptować
- Hyprland nie działa, a działał (xD)
- SELinux zawsze enforced
- Działa praktycznie tylko na RPM, nie DEB

---

# Końcówka

## Co jest fajne?

- Rollbacki, można mieć system do gamingu (np bazzite) i system do robienia poważnych rzeczy
- Mam spójny stan na obu maszynach
- Github actions buduje mi system bazowy i mam to w jednym miejscu kodem
- Mogę rozprowadzać system używając DockerHub/mojego OCI registry
- YubiKey działa out of the box
- Ścieżki ze stanem zmiennym systemu można łatwo backupować przyrostowo:
    - Używamy `snapper`, żeby robić btrfs snapshoty diffa `/etc`, oraz `/var` (albo `/var/home`)
    - Robimy skrypt, który bierze taki snapshot i puszcza na nim `restic`
    - Yay!

## EOF
