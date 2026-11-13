# Anonimizer — Kompleksowa Dokumentacja Architektury i Silnika Pseudonimizacji

> **Autor projektu:** `hatterp`  
> **Model wdrożeniowy:** Air-Gapped / RAM-Only / Zero Network Footprint  
> **Dystrybucja:** Pojedynczy, niezależny plik HTML ([`anonimizer.html`](anonimizer.html) / [`index.html`](index.html) oraz wersja angielska [`index-en.html`](index-en.html))

---

## 1. Wprowadzenie i Cel Projektu

W dobie powszechnego wykorzystywania modeli sztucznej inteligencji (LLM, np. ChatGPT, Claude, Gemini, modele lokalne) jednym z największych zagrożeń bezpieczeństwa informacji jest **niekontrolowany wyciek danych poufnych**. Programiści, administratorzy i analitycy codziennie przesyłają do modeli AI fragmenty kodu źródłowego, konfiguracje serwerowe, logi systemowe oraz zgłoszenia klientów zawierające:
* Hasła, klucze API, tokeny JWT, klucze prywatne PEM,
* Adresy IP, nazwy hostów infrastruktury wewnętrznej,
* Identyfikatory bazodanowe, numery kont IBAN, dane kart płatniczych,
* Dane osobowe (PII) podlegające RODO/GDPR (imiona, nazwiska, adresy, telefony, PESEL).

**Anonimizer** rozwiązuje ten problem w sposób bezkompromisowy:
1. **Lokalnie i bezśladowo** zamienia wszystkie wrażliwe dane na spójne tokeny zastępcze (np. `<SECRET_01>`, `<IP_01>`, `<PERSON_01>`).
2. Użytkownik przekazuje do modelu AI **bezpieczny prompt**.
3. Po wygenerowaniu odpowiedzi przez AI, aplikacja wykonuje **deanonimizację 1:1**, przywracając oryginalne dane w wygenerowanym kodzie lub konfiguracji.

---

## 2. Model Bezpieczeństwa (Air-Gapped / RAM-Only)

Aplikacja została zaprojektowana zgodnie z regułą zerowego zaufania do środowiska przeglądarki (*Zero Trust Browser Client*):

```
+-----------------------------------------------------------------------------------+
| PRZEGLĄDARKA INTERNETOWA (KLIENT)                                                 |
|                                                                                   |
|  +--------------------+        Krok 1: Maskowanie         +--------------------+  |
|  | Tekst Źródłowy     | --------------------------------> | Bezpieczny Prompt  |  |
|  | (Hasła, IP, PII)   |                                   | (<SECRET_01>, itp) |  |
|  +--------------------+                                   +--------------------+  |
|            |                                                         |            |
|            | Zapis mapowań:                                          |            |
|            v                                                         v            |
|  +------------------------------------------------------+   DO MODELU AI          |
|  | ULOTNY SKARBIEC RAM (Map / Set w pamięci JS)         |   (np. ChatGPT)         |
|  | - byToken: <SECRET_01> => "TajneHaslo123"            |                         |
|  | - byValue: "TajneHaslo123" => <SECRET_01>            |            |            |
|  | - ZERO zapisu na dysku (brak cookies, localStorage)  |            |            |
|  +------------------------------------------------------+            |            |
|            ^                                                         |            |
|            | Odczyt mapowań:                                         v            |
|            |                   Krok 2: Przywracanie       +--------------------+  |
|  +--------------------+ <-------------------------------- | Odpowiedź z AI     |  |
|  | Odtworzony Wynik   |                                   | (zawiera tokeny)   |  |
|  | (Oryginalne dane)  |                                   +--------------------+  |
|  +--------------------+                                                           |
|                                                                                   |
|  BLOKADA SIECIOWA: CSP default-src 'none' | fetch/XHR = BLOCKED | Zero wycieków   |
+-----------------------------------------------------------------------------------+
```

### 2.1. Twarda Izolacja Sieciowa (CSP)
W nagłówku `<head>` zdefiniowana jest restrykcyjna dyrektywa Content Security Policy:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline';">
```
* `default-src 'none'`: zabrania przeglądarce wysyłania jakichkolwiek zapytań HTTP/HTTPS, ładowania skryptów, obrazów, czcionek, stylów z sieci oraz łączenia się przez WebSockets czy WebRTC.
* **Brak zewnętrznych zależności**: style CSS i silnik JavaScript są w 100% wbudowane w plik HTML.
* **Autotest CSP w czasie rzeczywistym**: funkcja `securitySelfTest()` wykonuje lokalną próbę zapytania (`fetch('data:...')`), która w poprawnie zabezpieczonym środowisku zostaje zablokowana przez CSP, co potwierdza zielony badge bezpieczeństwa.

### 2.2. Runtime Lockdown API
Po zainicjowaniu strony funkcja `lockNetworkAPIs()` nadpisuje i unieważnia w obiekcie `window` funkcje komunikacyjne, kanały WebRTC i magazyny trwałe:
* Blokada sieci i WebRTC: `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `Worker`, `SharedWorker`, `RTCPeerConnection`, `webkitRTCPeerConnection`, `navigator.sendBeacon`, `window.open`.
* Blokada pamięci trwałej: `localStorage`, `sessionStorage`, `indexedDB`, `caches`, `cookieStore` oraz `document.cookie`.

### 2.3. Przechowywanie Danych Wyłącznie w Pamięci RAM
* Słownik mapowań istnieje wyłącznie w obiekcie struktury danych JavaScript (`Map()` / `Set()`).
* **Procedura PANIC (`Alt` + `Shift` + `X`)**:
  1. Wszystkie referencje tekstowe do wartości sekretów (oraz zestawów wariantów `en.values`) w słowniku są czyszczone i nadpisywane pustymi ciągami znaków (`en.value = ''`).
  2. Mapy i zbiory są czyszczone (`.clear()`), a obiekt skarbca zastępowany nową, pustą instancją.
  3. Zawartość pól `<textarea>` oraz pola wyszukiwania `#findInput` jest nadpisywana blokami znaków `█`, zerowana, a same elementy węzłów DOM są odpinane i wpinane ponownie, co niszczy natywny bufor historii cofania przeglądarki (`Undo` / `Ctrl+Z`).
  4. Schowek systemowy zostaje wyczyszczony.
  5. Skrypt generuje kontrolowaną alokację buforów pamięci w celu wywarcia nacisku na Garbage Collector przeglądarki.

### 2.4. Skaner Ryzyka Rezydualnego i Ochrona Metadanych
* **Ukryta długość sekretu**: w tabeli aktywnych mapowań kolumna „Długość” wyświetla `—`, a kolumna „Wartość” pokazuje `••••••••` do czasu świadomego kliknięcia przycisku `Pokaż` / `Show`.
* **Skaner ryzyka rezydualnego (`residualRisks(out)`)**: po każdym maskowaniu analizuje tekst wynikowy linia po linii pod kątem potencjalnie pominiętych śladów (adresy e-mail, IPv4, webhooki, XML/SQL z sekretem, PESEL, nieujęte wartości przy kluczach haseł) i raportuje dokładny numer linii w pasku ostrzeżeń.

---

## 3. Silnik Detekcji Heurystycznej (10 Kategorii)

Przed uruchomieniem detektorów funkcja `normalizeScanText(text)` usuwa niewidoczne znaki o zerowej szerokości (`\u200B–\u200D`, `\uFEFF`, `\u2060`, `\u180E`, `\u00AD`) oraz kontekstowo normalizuje zapis *defanged* (`10[.]0[.]0[.]1`, `user@host[.]pl`, `[dot]`) w czasie $O(N)$ — z ochroną wyrażeń regularnych w kodzie typu `/a[.]b/`. Silnik operuje następnie deterministycznie przy użyciu zoptymalizowanych detektorów, entropii Shannona oraz walidatorów sum kontrolnych:

| Kategoria | Przełącznik UI | Generowany Token | Przykłady Wykrywanych Danych |
|---|---|---|---|
| **IP** | `#tg_IP` | `<IP_01>` | IPv4 z maskami CIDR (`192.168.1.1/24`), IPv6 pełne i skrócone (`fe80::1`, `2001:db8::/32`). Zabezpieczenie przed maskowaniem numerów wersji (`'version' => '30.0.1.2'`, `v1.2.3.4`). |
| **EMAIL** | `#tg_EMAIL` | `<EMAIL_01>` | Adresy poczty elektronicznej (`jan.kowalski@firma.pl`), również po kontekstowej normalizacji zapisu `[.]` / `[dot]`. |
| **SECRET** | `#tg_SECRET` | `<SECRET_01>` | Pary klucz-wartość (`password=...`, `secret: ...`, `api_key := ...`, `dbpassword => ...`, `CVC: 123`, polskie nazwy kluczy np. `hasło_użytkownika`, flagi `--password` bez połykania kolejnych flag CLI), listy YAML (`detectYamlBlocks`), tagi XML z sekretami, klauzule SQL (`IDENTIFIED BY`, `PASSWORD`), ciasteczka sesyjne (`Set-Cookie`), frazy seed (BIP-39), bloki certyfikatów/kluczy PEM (`BEGIN RSA PRIVATE KEY`), poświadczenia w URL (`user:pass@host`), konfiguracje Cisco/BGP/OSPF/SNMP. |
| **TOKEN** | `#tg_TOKEN` | `<TOKEN_01>` | Nagłówki `Bearer <token>`, webhooki (Slack, Discord, MS Teams), tokeny OpenAI (`sk-...`), AWS (`AKIA...`), GitHub (`ghp_...`), GitLab, Slack, tokeny JWT (`eyJ...`), klucze publiczne SSH (`ssh-ed25519`), hashe i ciągi o wysokiej entropii (min. 32 znaki base64/hex). |
| **HOST** | `#tg_HOST` | `<HOST_01>` | Domeny internetowe, nazwy FQDN (`baza.corp.local`, `api.firma.pl`) oraz hosty jednoczłonowe (`host = api_srv`). Ochrona przestrzeni konfiguracyjnych (`memcache.local`), schematów URI i klas programistycznych. |
| **MAC** | `#tg_MAC` | `<MAC_01>` | Fizyczne adresy MAC w formatach standardowych (`00:1A:2B:3C:4D:5E`, `00-1A-2B-...`, `001a.2b3c.4d5e`). |
| **ID** | `#tg_ID` | `<ID_01>` | PESEL, NIP i REGON z walidacją sum kontrolnych (`peselOk`, `nipOk`, `regonOk`), księgi wieczyste (`KW`), identyfikatory UUID/GUID, AWS ARN, instancje chmurowe (`i-0123456789abcdef0`), numery rachunków IBAN oraz polskie 26-cyfrowe NRB, karty płatnicze (`luhnOk`), sygnatury spraw (`KM 1234/26`), dowody osobiste (`ABA 123456`). |
| **USER** | `#tg_USER` | `<USER_01>` | Loginy i nazwy użytkowników (`dbuser = 'postgres'`, `superuser = admin`, `username: root`), ścieżki katalogów domowych (`/home/janusz`), flagi CLI (`sudo -u admin`, `psql -U postgres`), logi SSH (`Failed password for invalid user admin`). |
| **PORT** | `#tg_PORT` | `<PORT_01>` | Przypisania portów (`port = 5432`, `PORT: 8080`), notacje `host:port` i `[ipv6]:port`, dyrektywy serwerowe `Listen 443`, mapowania portów Docker (`-p 8080:80`). |
| **PII** | `#tg_PII` | `<PERSON_01>`, `<PHONE_01>`, `<ADDRESS_01>`, `<ORG_01>`, `<DATE_01>` | Dane osobowe: imiona i nazwiska (z tytułami `Pan`, `dr`, `lekarz`, dwukierunkową odmianą polskich nazwisk `inflectSingleWord` / `addPersonInflections` i obsługą nazwisk dwuczłonowych z dywizem), daty urodzenia (`14.05.1985 r.`), numery telefonów, adresy (`ul. ...`), współrzędne GPS (`GPS_CTX`), nazwy firm i placówek. |

---

## 4. Architektura Silnika (Algorytm Maskowania i Przywracania)

### 4.1. Rozwiązywanie konfliktów nakładania się reguł (Collision Resolution)
Tekst wejściowy może zawierać nakładające się dopasowania (np. ciąg znaków będący jednocześnie adresem URL, hostem i portem). Silnik rozwiązuje to trójstopniowo:
1. **Priorytetyzacja reguł (`priority`)**: reguły o wyższym rygorze kontekstowym (np. jawne bloki PEM lub tokeny w cudzysłowach) mają pierwszeństwo przed ogólnymi wyrażeniami regularnymi.
2. **Sortowanie kandydatów**: kandydaci sortowani są według priorytetu, następnie według długości dopasowania (dłuższe dopasowanie wygrywa), a w ostateczności według pozycji w tekście.
3. **Tablica zajętości (`Uint8Array`)**: silnik rezerwuje zakresy indeksów znaków w tablicy bajtowej. Dopasowania kolidujące z już zajętym zakresem są bezpiecznie odrzucane, zapobiegając uszkodzeniu struktury danych.

### 4.2. Spójność Tokenów (Identyczne Wartości = Ten Sam Token)
Wszystkie wystąpienia tej samej wartości w całym tekście otrzymują ten sam identyfikator tokena (np. wielokrotnie powtórzony adres IP `10.0.0.1` w całym pliku otrzyma zawsze token `<IP_01>`). Dzięki temu model sztucznej inteligencji zachowuje logiczny kontekst relacji między elementami konfiguracji lub kodu.

### 4.3. Odporność na Ataki ReDoS (Regular Expression Denial of Service)
Wszystkie wyrażenia regularne zostały zoptymalizowane pod kątem unikania niebezpiecznego nawrotu (*catastrophic backtracking*):
* Zakotwiczenia liniowe i pozycjonujące,
* Konstrukcje bezwzględnie dopasowujące (*atomic-like matching*),
* Testy obciążeniowe potwierdzają przetwarzanie złośliwych ciągów o długości 60 000+ znaków w czasie poniżej 60 ms.

### 4.4. Jednoetapowa Deanonimizacja 1:1
Przywracanie oryginalnych danych skanuje wyłącznie identyfikatory tokenów wygenerowane przez aplikację:
```javascript
const TOKRE = /(?:<|&lt;)([A-Za-z][A-Za-z0-9_]{0,30}_\d{1,6})(?:>|&gt;)/g;
```
Podmiana odbywa się w pojedynczym przejściu bez ponownego skanowania już wstawionych wartości, co eliminuje ryzyko zapętlenia deanonimizacji (*injection attacks*).

### 4.5. Wysuwana wyszukiwarka fraz, modal weryfikacji i wielowartościowe wiązanie do tokena
* **Wysuwana wyszukiwarka w tekście źródłowym (`setFindOpen` + `findMatchesInSource`)**: panel `#findBar` jest domyślnie zwinięty i otwiera się na żądanie po kliknięciu `#findToggleBtn`. Skanuje pole `#src` z obsługą flag `ignoreCase` oraz `wholeWord`, pomijając już wygenerowane placeholdery `<...>`. Funkcja `stepFind(dir)` nawiguje po kolejnych dopasowaniach i automatycznie przewija pole tekstowe do odpowiedniego wiersza.
* **Okno modalne weryfikacji bezpiecznego promptu (`#previewModal` / `renderModalView`)**: prezentuje wynik `#out1` z numeracją wierszy, podświetlonymi pigułkami tokenów (`span.tok-pill` tworzony bezpiecznie przez DOM bez `innerHTML`), oznaczeniem linii z ostrzeżeniami z `residualRisks` (`.risk-line`) oraz paskiem szybkiego domaskowywania (`maskFromModal`), do którego trafia tekst zaznaczony bezpośrednio w modalu.
* **Wielowartościowy skarbiec (`en.values`)**: każdy wpis w `vault.byToken` przechowuje zbiór `values: Set(...)` (obok kanonicznej wartości głównej `en.value` przywracanej w Kroku 2). Umożliwia to przypięcie wielu wariantów tej samej frazy (np. `TajnaFirma`, `tajnafirma`, `TAJNAFIRMA`, `ProjektOmega`) do jednego wspólnego tokena z najwyższym priorytetem (`pri = 0`).
* **Wybór docelowego tokena (`parseExplicitToken` + `bindValuesToToken`)**: użytkownik może utworzyć nowy token z wybranej etykiety, wskazać istniejący token z listy rozwijanej `#targetTok` / `#modalTargetTok` albo wpisać jawny token numerowany w polu `#lbl` / `#modalLbl` (np. `<USER_05>` lub `USER_05`). Przepięcie wartości już zamaskowanej automatycznie przenosi ją ze starego tokena do wskazanego bez pozostawiania osieroconych reguł.

---

## 5. Dostępne Wersje i Wielojęzyczność

Repozytorium zawiera zsynchronizowane wersje aplikacji, współdzielące identyczną architekturę i silnik logiczny:

| Plik | Język UI | Predefiniowane Presety Etykiet (Krok 1) |
|---|---|---|
| [`anonimizer.html`](anonimizer.html) / [`index.html`](index.html) | 🇵🇱 Polski | `USER`, `IMIE`, `NAZWISKO`, `ADRES`, `NAZWA`, `PESEL`, `ROK_URODZENIA`, `TELEFON`, `ID`, `PORT` |
| [`index-en.html`](index-en.html) | 🇬🇧 Angielski | `USER`, `FIRST_NAME`, `LAST_NAME`, `NAME`, `ADDRESS`, `NATIONAL_ID`, `BIRTH_YEAR`, `PHONE`, `ID`, `PORT` |

---

## 6. Instrukcja Obsługi Krok po Kroku

```
+-----------------------------------------------------------------------------------------------------+
| KROK 1: WKLEJ TEKST W LEWE OKNO -> KLIKNIJ "SKANUJ I ZAMASKUJ" -> SKOPIUJ BEZPIECZNY PROMPT        |
|                                                                                                     |
| [Dane Źródłowe]                                   [Bezpieczny Prompt do AI]                         |
| dbuser = 'admin'           ====== Silnik =====>   dbuser = '<USER_01>'                              |
| password = 'Tajne'         === Anonimizera ===>   password = '<SECRET_01>'                          |
| host = '10.0.0.5:5432'                            host = '<IP_01>:<PORT_01>'                        |
+-----------------------------------------------------------------------------------------------------+
                                                  |
                                                  | Przesłanie promptu do AI (ChatGPT / Claude / Gemini)
                                                  v
+-----------------------------------------------------------------------------------------------------+
| KROK 2: ODPOWIEDŹ Z AI WKLEJ W PRAWE OKNO -> KLIKNIJ "PRZYWRÓĆ ORYGINALNE DANE" -> SKOPIUJ WYNIK    |
|                                                                                                     |
| [Odpowiedź od AI]                                 [Oryginalny Wynik z Przywróconymi Danymi]         |
| "Połącz się z bazą:                               "Połącz się z bazą:                               |
| psql -h <IP_01> -p <PORT_01> === Przywrócenie ===> psql -h 10.0.0.5 -p 5432                         |
| -U <USER_01>"                === danych 1:1 ===>  -U admin"                                         |
+-----------------------------------------------------------------------------------------------------+
```

1. **Otwórz plik** `index.html` (lub `index-en.html`) w dowolnej przeglądarce internetowej (bez konieczności połączenia z Internetem).
2. **Wklej poufny tekst** w pole *Dane źródłowe* po lewej stronie.
3. Wybierz aktywne kategorie detekcji (lub pozostaw domyślnie zaznaczone wszystkie).
4. Kliknij **Skanuj i zamaskuj** (lub naciśnij `Ctrl` + `Enter`).
5. Skopiuj bezpieczny prompt za pomocą przycisku **Kopiuj bezpieczny prompt** i wyślij go do modelu AI.
6. Po otrzymaniu odpowiedzi od AI, wklej ją do pola *Surowa odpowiedź z modelu* po prawej stronie.
7. Kliknij **Przywróć oryginalne dane** (lub naciśnij `Ctrl` + `Enter`) i skopiuj gotowy, zrekonstruowany kod/tekst.
8. Po zakończeniu pracy naciśnij **PANIC · Wyczyść wszystko** (`Alt` + `Shift` + `X`) lub po prostu zamknij kartę przeglądarki.

---

## 7. Skróty Klawiszowe

| Skrót | Miejsce wywołania | Funkcja |
|---|---|---|
| `Ctrl` + `Enter` | Pole *Dane źródłowe* (Krok 1) | Wykonanie skanowania i maskowania tekstu |
| `Enter` / `Shift` + `Enter` | Pole *Wyszukaj frazę* (`#findInput`) | Przejście do następnego / poprzedniego wystąpienia frazy |
| `Ctrl` + `Enter` | Pole *Wyszukaj frazę* (`#findInput`) | Przypisanie wszystkich znalezionych wystąpień do wybranego tokena |
| `Esc` | Okno modalne podglądu (`#previewModal`) | Zamknięcie okna podglądu bezpiecznego promptu |
| `Ctrl` + `Enter` | Pole *Odpowiedź z modelu* (Krok 2) | Wykonanie deanonimizacji i przywrócenia danych |
| `Alt` + `Shift` + `X` | Dowolne miejsce na stronie | **PANIC**: Natychmiastowe usunięcie danych z pamięci, pól i schowka |

---

## 8. Licencja i Autorstwo

* **Autor:** `hatterp`
* **Licencja:** [MIT](LICENSE) — pełna swoboda wykorzystania w projektach prywatnych, komercyjnych i instytucjonalnych.

