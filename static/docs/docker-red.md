---
author: Adam Piaseczny
date: dd.mm.YYYY
---

# Docker - Sekcja RED

Czyli szybki wstęp do środowiska wykonywania dockera, podatności oraz implikacje w świecie cyberbezpieczeństwa.

## Jak dostać się do tej prezentacji

```
go install github.com/maaslalani/slides@latest
curl -L piaseczny.dev/docs/docker-red.md | slides
```

albo bez użycia `go`, obchodząc `eduroam`

```sh
ssh -p imap red.piaseczny.dev
```

## Meta

- Data: 28.11.2024
- Przerywaj mi jak masz pytania, lubię o tym gadać

---

# Rozkład jazdy

## Teoria

- Po co komu kontenery?
- Przegląd działania dockera
- Izolacja
- Dockerfile jako przepis na sukces
- Wektory ataku

## Praktyka

- Ekstrakcja sekretów - Git
- Ekstrakcja sekretów - ENV
- Wywrotka

---

# Po co komu kontenery?

## U mnie działa

Kontenery rozwiązują "u mnie działa", czyli pozwalają nam zagwarantować, że kod uruchamiany lokalnie będzie działał tak samo na innej maszynie.

## Modularność

Zwyczajowo odpala się tylko jeden kontener na jeden logiczny proces więc możemy przez to dynamicznie podmieniać komponenty dużego systemu.

## Izolacja

Kontenery izolują działanie aplikacji, więc jeżeli ktoś się dostanie na maszynę to nie może dużo popsuć.

---

# Przegląd działania dockera

## Typowe flow użytkownika:

- Zdobywamy obraz dockera
    - Pobieramy z repozytorium, na przykład domyślnego [dockerhub](https://hub.docker.com) albo
    - Budujemy obraz samemu
- Uruchamiamy obraz za pomocą środowiska wykonywania docker

## Etykiety

Obrazy dockera są opisane etykietą (`tag`), etykieta musi być unikalna w kontekście repozytorium. Etykiety zwykle wyglądają w następujący sposób - `typicalam/goread:v1.7.1` czyli `użytkownik/program:wersja`. Jest również specjalna wersja `latest`, która pobiera najnowszy obraz dostępny w repozytorium. Dodatkowo w przypadku `dockerhub` istnieje użytkownik `_`, którego wpisywanie można pominąć - na przykład `_/ubuntu:24.04` to `ubuntu:24.04`.

---

# Przegląd działania dockera

## Przykład budowania obrazu

```sh
git clone https://github.com/TypicalAM/goread
docker build --tag typicalam/goread:v1.7.1 . # Po tym obraz jest już lokalnie dostępny
```

## Przykład uruchamiania obrazu

```sh
docker run --interactive --rm --tty typicalam/goread:v1.7.1
```

- `--interactive` - Użyj standardowego wejścia
- `--rm` - Usuń kontener po użyciu
- `--tty` - Użyj pseudo-terminala

---

# Przegląd działania dockera

## Przykład uruchamiania obrazu

### Spróbujmy odpalić sobie najnowszą stabilną wersję `ubuntu`:

```sh
docker run --rm -it -v $HOME:/RED ubuntu:24.04
cat /etc/os-release # WOW, to rzeczywiście ubuntu
ls /RED # Folder z hosta jest zamontowany na kontenerze!
```

### Realistyczny przykład

```sh
docker run --rm -it -p 2137:8888 -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN=mojehaslo -v $PWD:/home/jovyan jupyter/datascience-notebook:latest
```

Temu kontenerowi podajemy nasz aktualny folder jako home użytkownika, robimy mapowanie portów (`localhost:2137` to tak naprawdę `kontener:8888`) oraz zmiennych środowiskowych programu.

---

# Izolacja

## Czy kontener to VM?

Wiele osób mylnie sądzi, że konteneryzowane aplikacje tracą na szybkości wykonywania, tak jak w przypadku maszyn wirtualnych. Nic bardziej mylnego. Kontenery mają swój własny system plików (wypakowywany z lokalnego obrazu) + opcjonalnie voluminy, ale używają kernela hosta!

## Demonstracja izolacji

```sh 
docker run --rm -it ubuntu:24.04 sleep 500
ps aux | grep "sleep 500" # W innym terminalu
```

Dlatego docker działa gorzej na Windowsie oraz MacOS - potrzebuje VM z linuksem jako hosta kontenerów. Chwila, czy to znaczy, że kontener też widzi aktywność hosta?

---

# Izolacja

## Demonstracja izolacji

```sh 
sleep 500 &
docker run --rm -it ubuntu:24.04 ps aux | grep "sleep 500"
```

Jak widać guest nie widzi aktywności hosta, te dwie aplikacje są w odmiennych przestrzeniach nazw kernela, w tym przypadku chodzi o przestrzeń nazw `PID` czyli pierwszym elementem izolacji. Izolacja w dockerze jest egzekwowana za pomocą trzech komponentów kernela:
- Przestrzenie nazw - `man namespaces` - Ogranicza co kontener "widzi", na przykład `docker run --rm --pid=host -it ubuntu:24.04` już zobaczy naszego `sleep 500`
- Grupy kontrolne - `man cgroups` - Ogranicza ile zasobów maszyny kontener może zabrać, na przykład `docker run --rm --memory="300m" -it ubuntu:24.04`
- Capabilities - `man capabilities` - Ogranicza ile operacji systemowych kontener może wykonać, jest to rozbicie możliwości użytkowika `root`

---

# Dockerfile jako przepis na sukces

## Skąd docker wie jak zbudować obraz?

Sposób budowania kontenera definiuje specjalny plik `Dockerfile`:

- Instrukcje w nim są wykonywane krok po kroku przez silnik budulcowy `buildkit` i kończą się gotowym obrazem.
- Każdy obraz w dockerze jest budowany na podstawie innego obrazu, więc
- `Dockerfile` zawsze rozpoczyna się instrukcją `FROM`, która deklaruje obraz bazowy.
- Podczas budowania `buildkit` "widzi" tylko folder, w którym budujemy obraz.

Plik `Dockerfile` jest analogiczny do innych charakterystycznych plików budowania takich jak `Makefile`, `CMakeLists.txt`, `go.mod`, czy nawet `package.json` - jak go widzimy to od razu wiadomo jak zbudować projekt! Jest on napisany customowym językiem, dość łatwym do zrozumienia, wytłumaczmy składnię poprzez przykład:

---

# Dockerfile jako przepis na sukces

## Skąd docker wie jak zbudować obraz?

```Dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update && apt-get install -y neofetch
ENTRYPOINT ["neofetch"]
CMD ["--ascii_distro", "ArchLinux"]

```

Budujemy i uruchamiamy:
```sh
docker build --tag myfakearch . && docker run --rm -it myfakearch
```

Możemy też zmodyfikować też argumenty `neofetch`, nadpisać `CMD`:

```sh
docker run --rm -it myfakearch --ascii_distro Ubuntu
```

---

# Dockerfile jako przepis na sukces

## Skąd docker wie jak zbudować obraz?

Możemy uogólnić obraz używając argumentów budowy, podajemy je podczas wywoływania `docker build`

```Dockerfile
FROM ubuntu:24.04
ARG MESSAGE
ENV MESSAGE=$MESSAGE
COPY . /home/nobody
CMD cat /home/nobody/hello.txt && echo $MESSAGE
```

```sh
docker build --tag hello:adam --build-arg MESSAGE=adam . && docker run hello:adam
```

Patrząc na to możemy wpaść na dużo głupich pomysłów:
- Możemy za pomocą `ARG` podawać na przykład token do GitLab.
- Dodatkowo za pomocą `COPY` możemy kopiować klucze SSH.

---

# Dockerfile jako przepis na sukces

## Warstwy

- Każda instrukcja, która może modyfikować pliki jest dokładana jako warstwa obrazu.
- Możemy zobaczyć warstwy obrazu używając `docker history <tag>`.
- Każda warstwa dodawana jest przyrostowo na postawie poprzednich.

Zobaczmy instrukcje do stworzenia naszego obrazu:

```sh
docker history myfakearch
docker history hello:adam
```

Warstwy opisane są w pliku manifestu obrazu, do wglądu w te rzeczy przyda nam się `docker inspect <tag>`. Każda warstwa jest osobnym archiwum `tar`, żeby je zobaczyć zapiszmy jeden z obrazów:

```sh
docker save hello:adam > obraz.tar
```

---

# Dockerfile jako przepis na sukces

## Warstwy

Ekstrakcja archiwum:

```sh
mkdir obraz
tar xvf obraz.tar --directory=obraz
tree obraz
```

Czy widać warstwy zauważone w `docker inspect hello:adam`? Spróbuj znaleźć nasz dodany plik! Istnieje też narzędzie do przeglądania wastw `dive`, możemy je odpalić używając dockera:

```sh
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest hello:adam
```

Czy oprócz naszego pliku `hello.txt` coś tam jeszcze jest?

---

# Dockerfile jako przepis na sukces

## Docker ignore

Podczas instrukcji `COPY <host> <guest>` może się okazać, że kopiujemy więcej niż myślimy. W naszym poprzednim przykładzie okazało się, że pliki z naszego lokalnego folderu wyciekły do kontenera. Z tego powodu istnieje plik `.dockerignore`, który pozwala nam oznaczać pliki wykluczone z kontekstu budowy:

```
obraz.tar
obraz
```

Deweloperzy bardzo często zapominają podczas tworzenia instrukcji budowy o potędze `COPY` - warto podczas inspekcji warstw szukać rzeczy, które mogą nam dać dodatkowe metadane:
- Jeżeli nie ma pliku `.dockerignore` to szukajmy rzeczy, które są w `.gitignore`
- Pliki `.env`, być może posiadają klucze dostępowe
- Foldery `.git` z projektów closed source mogą nam dać dużo informacji o deweloperach

---

# Wektory ataku

## Przykładowe wektory ataku

### Łamanie izolacji (container breakout)

Jeżeli kernel ma podatność albo kontener ma za duże uprawnienia, na przykład przy użyciu `--priviledged`, jest szansa na eskalacja uprawnień do konta administratora hosta.

### Wyciek wrażliwych informacji

Serwer budowania obrazów może powodować włączenie potencjalnie niebezpiecznych informacji do obrazu w warstwach obrazu/instrukcjach.

### Ataki łańcucha dostaw

Polecenie `docker pull` pobiera najnowszą wersję obrazu z dowolnego repozytorium, popularnym atakiem jest wstawienie obrazu na `dockerhub` z wyższą wersją niż ta w wewnętrznym repozytorium firmy.

---

# Wektory ataku

## Przykładowa obrona

### Łamanie izolacji (container breakout)

- Ogarniczanie możliwości kontenera używając `--cap-drop=ALL --cap-add=<nazwa>`
- Uruchamianie dockera w trybie rootless, budując obraz pod dane `UID` i `GID` użytkownika.

### Wyciek wrażliwych informacji

- Używanie plików `.dockerignore` i inspekcja obrazów
- Używanie skanerów bezpieczeństwa w CI/CD

### Ataki łańcucha dostaw

- Version pinning, używanie konkretnej etykiety zamiast `latest`, na przykład `ubuntu:24.04`
- Używanie wersji sha256 obrazu aby się upewnić, że obraz o tej samej etykiecie nie został zmieniony

---

# Praktyka

## Ekstrakcja sekretów - Git

Twoim zadaniem jest dostanie się do flagi zawartej w prywatnym repozytorium GitHub. Jedyne co masz to obraz na `dockerhub`:

```sh
docker run --rm -it -p 2137:8080 typicalam/myawesomeapp
```

Przydatne polecenia:

```sh
docker inspect <tag>
docker history --no-trunc <tag>
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest <tag>
```

---

# Praktyka

## Ekstrakcja sekretów - ENV

Twoim zadaniem jest dostanie się do flagi będącej kluczem deweloperskim. Jedyne co masz to obraz na `dockerhub`:

```sh
docker run --rm -it -p 2137:8080 -e HELLO_USER=imie typicalam/mysecondapp
```

Przydatne polecenia:

```sh
docker inspect <tag>
docker history --no-trunc <tag>
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest <tag>
```

---

# Praktyka

## Zadanie od unknow - naprawa kontenera

Zasady:
- Nie modyfikujesz kontenera
- Sukces jest wtedy, gdy aplikacja wydrukuje `SUKCES`

```sh
docker run --rm -it unknow/wywrotka
```

Przydatne polecenia:

```sh
docker inspect <tag>
docker history --no-trunc <tag>
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive:latest <tag>
```

---

# Koniec

## Dzięki za przejście przez warsztaty!

Najlepszą nauką jest praktyka, więc łatwiej jest zrozumieć jak sam eksperymentujesz!

## Dodatkowe materiały

- [Docker Engine Security](https://docs.docker.com/engine/security)
- [Mitre Att&ck surface](https://attack.mitre.org/matrices/enterprise/containers)
- [Moja prezentacja na Poznań Security Meetup](https://opensecurity.pl/wp-content/uploads/2024/06/hardening-dockera.pdf)
- [Rootless docker](https://docs.docker.com/engine/security/rootless/)
