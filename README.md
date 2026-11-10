# Anonimizer — Air-Gapped AI Data Pseudonymization & Restoration Tool

[![Security: Air-Gapped](https://img.shields.io/badge/Security-Air--Gapped%20%2F%20RAM--only-success?style=flat-square)](#model-bezpiecze%C5%84stwa--security-model)
[![CSP: default-src 'none'](https://img.shields.io/badge/CSP-default--src%20'none'-blue?style=flat-square)](#polityka-bezpiecze%C5%84stwa-csp)
[![Zero Dependencies](https://img.shields.io/badge/Dependencies-0%20(Pure%20Vanilla)-brightgreen?style=flat-square)](#architektura)
[![Author: hatterp](https://img.shields.io/badge/Author-hatterp-orange?style=flat-square)](#autor)

Maksymalnie bezpieczna, lekka i samowystarczalna aplikacja webowa (działająca jako pojedynczy, niezależny plik HTML) przeznaczona do lokalnej pseudonimizacji wrażliwych danych (kod źródłowy, konfiguracje sieciowe i serwerowe, logi, hasła, klucze API, adresy IP, dane osobowe / PII) przed wysłaniem ich do chmurowych modeli sztucznej inteligencji (LLM, np. ChatGPT, Claude, Gemini), z możliwością natychmiastowego przywrócenia oryginalnych danych (deanonimizacji 1:1) po powrocie odpowiedzi z AI.

Dostępna w dwóch niezależnych wersjach językowych:
* 🇵🇱 **Polska**: [`index.html`](index.html)
* 🇬🇧 **Angielska**: [`index-en.html`](index-en.html)

---

## 🔒 Kluczowe priorytety bezpieczeństwa (Zero Network Footprint)

1. **Pełna izolacja sieciowa (Content Security Policy):**
   * W sekcji `<head>` osadzona jest restrykcyjna polityka:
     ```html
     <meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline';">
     ```
   * **Bezwzględny brak jakichkolwiek zasobów zewnętrznych** — brak CDN-ów (brak zewnętrznych bibliotek, Tailwind, Google Fonts, ikon SVG czy zewnętrznych skryptów JS). Całość stylów CSS i logiki JavaScript znajduje się w jednym pliku HTML.
   * **Wbudowany automatyczny autotest CSP**: aplikacja w czasie rzeczywistym wykonuje testową próbę połączenia i informuje zielonym/czerwonym wskaźnikiem w nagłówku o stanie izolacji sieciowej.
   * **Runtime lockdown API sieciowych i dyskowych**: w środowisku uruchomieniowym wyłączane i unieważniane są funkcje `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `Worker`, `sendBeacon` oraz `open`.

2. **Ochrona danych wyłącznie w pamięci ulotnej (RAM-only):**
   * **Zero zapisu na dysku**: aplikacja nie używa `localStorage`, `sessionStorage`, `IndexedDB` ani plików `cookies` (dostęp do tych API jest zablokowany).
   * **Bezpieczny słownik mapowań**: sekretne mapowania (`token <-> oryginalna wartość`) są przechowywane wyłącznie w ulotnej strukturze pamięci JavaScript (`Map()` / `Set()`) w bieżącej karcie przeglądarki. Po odświeżeniu lub zamknięciu karty wszystkie dane bezpowrotnie znikają z pamięci.
   * **Funkcja natychmiastowego czyszczenia (PANIC / Clear)**:
     * Dedykowany przycisk `PANIC · Wyczyść wszystko` (skrót klawiszowy `Alt` + `Shift` + `X`) oraz przyciski `Wyczyść` w obu kolumnach.
     * Natychmiast zeruje pola tekstowe, czyści historię cofania przeglądarki (`Undo` / `Ctrl+Z`), czyści schowek systemowy, nadpisuje i usuwa referencje do sekretów w pamięci RAM oraz wywiera nacisk na mechanizm Garbage Collectora przeglądarki.

---

## ⚙️ Możliwości i silnik detekcji

Aplikacja podzielona jest na 2 czytelne kroki (układ dwukolumnowy):

### Krok 1: Anonimizacja (Prompt OUT — przed wysłaniem do AI)
Automatycznie wykrywa i zamienia wrażliwe dane na czytelne, bezpieczne tokeny z możliwością włączania/wyłączania poszczególnych kategorii w czasie rzeczywistym:
* **IP (v4/v6, CIDR)**: pełne i skrócone adresy IPv6, IPv4 z maskami CIDR (`<IP_01>`), z zaawansowaną ochroną przed fałszywym maskowaniem numerów wersji oprogramowania (np. `'version' => '30.0.1.2'`, `v1.2.3.4`).
* **E-mail**: adresy poczty elektronicznej (`<EMAIL_01>`).
* **Sekrety**: hasła, klucze i tokeny przypisywane przez `=`, `:`, `:=`, `=>` (np. `password`, `secret`, `token`, `api_key`, `auth`, `dbpassword`, `cvc`/`cvv`), bloki PEM certyfikatów i kluczy prywatnych (`-----BEGIN PRIVATE KEY-----`), konfiguracje Cisco/BGP/OSPF/SNMP/RADIUS, poświadczenia w URL (`scheme://user:pass@host`).
* **Tokeny / hashe / klucze**: tokeny `Bearer`, klucze AWS (`AKIA...`), GitHub (`ghp_...`), GitLab, Slack, OpenAI/Anthropic/Stripe (`sk-...`), JWT, klucze publiczne SSH oraz ciągi o wysokiej entropii / base64 / hex powyżej 32 znaków (`<TOKEN_01>`).
* **FQDN / Hosty**: domeny internetowe i nazwy serwerów wewnątrz sieci (`<HOST_01>`), z wykluczeniem nazw przestrzeni konfiguracyjnych (np. `memcache.local`) oraz klas PHP.
* **MAC**: adresy fizyczne kart sieciowych (`<MAC_01>`).
* **ID / Konta / Karty**: identyfikatory UUID, identyfikatory chmurowe (AWS ARN, instancje EC2, GCP), numery kont IBAN (w tym ze spacjami) oraz polskie 26-cyfrowe numery NRB, numery kart płatniczych (Visa, Mastercard, Amex z walidacją algorytmem Luhna), sygnatury akt komorniczych/sądowych (`KM ...`), numery dokumentów tożsamości (`<ID_01>`).
* **User / Login**: nazwy użytkowników w konfiguracjach (`dbuser`, `superuser`, `username`, `login`), ścieżki katalogów domowych `/home/user`, komendy CLI (`sudo`, `psql -U`), logi SSH (`Failed password for user`).
* **Porty**: parametry `port=`, definicje w URL/hostach (`:5432`, `[::1]:8080`), dyrektywy `Listen`, `Expose`, flagi kontenerów Docker `-p 8080:80`.
* **Dane osobowe / PII**: imiona i nazwiska (wraz z tytułami `Pan`, `dr`, `lekarz`, `konsultant`), daty urodzenia, adresy zamieszkania (`ul. ...`, kody pocztowe), numery telefonów stacjonarnych i komórkowych (`+48 ...`), nazwy spółek i organizacji (`Sp. z o.o.`, `S.A.`, placówki medyczne).

#### Ręczne maskowanie:
Zaznacz dowolny fragment tekstu w polu źródłowym, wybierz predefiniowaną etykietę z listy 10 najczęstszych typów lub wpisz własną (np. `PACJENT`, `SERWER`) i kliknij **Zamaskuj zaznaczenie jako token** — zamaskowane zostaną wszystkie wystąpienia tego ciągu w całym tekście.

#### Tabela aktywnych mapowań:
Podgląd wszystkich zamian: typ, identyfikator tokena, długość ciągu, liczba użyć, bezpieczne odsłanianie wartości gwiazdkowanych na żądanie oraz możliwość usunięcia pojedynczej reguły (zabezpieczenie przed atakami XSS).

### Krok 2: Przywracanie danych (Prompt IN — po powrocie z AI)
* Wklej surową odpowiedź zwróconą przez model AI zawierającą tokeny (np. `<IP_01>`, `<SECRET_01>`).
* Kliknij **Przywróć oryginalne dane** (lub naciśnij `Ctrl` + `Enter`).
* Aplikacja wykonuje bezpieczne jednoetapowe mapowanie 1:1, wstawiając oryginalne wartości z pamięci RAM w miejsce wygenerowanych placeholderów, bez wysyłania czegokolwiek do sieci.

---

## 🚀 Jak uruchomić?

Aplikacja **nie wymaga instalacji, serwera, Node.js ani połączenia z Internetem**:

1. Pobierz repozytorium lub pojedynczy plik:
   * [Wersja polska (`index.html`)](index.html)
   * [Wersja angielska (`index-en.html`)](index-en.html)
2. Kliknij dwukrotnie plik `index.html` lub przeciągnij go do dowolnej nowoczesnej przeglądarki (Chrome, Firefox, Brave, Safari, Edge).
3. Aplikacja natychmiast działa w 100% lokalnie i offline.

> **Wskazówka (Maksymalne bezpieczeństwo / Air-Gap):**  
> Możesz odłączyć komputer od sieci (wyłączyć Wi-Fi / odpiąć kabel Ethernet) lub uruchomić aplikację w trybie incognito/prywatnym przeglądarki. Działa dokładnie tak samo, dając 100% gwarancji poufności.

---

## 📋 Skróty klawiszowe

| Skrót | Działanie |
|---|---|
| `Ctrl` + `Enter` (w polu Kroku 1) | Skanuj i zamaskuj tekst |
| `Ctrl` + `Enter` (w polu Kroku 2) | Przywróć oryginalne dane |
| `Alt` + `Shift` + `X` | **PANIC**: Natychmiastowe wyczyszczenie pól, RAM-u i schowka |

---

## 👨‍💻 Autor

Projekt stworzony przez: **hatterp**  
Architektura: *Air-Gapped / RAM-Only / Zero Network Footprint*

---

## 📄 Licencja

Projekt udostępniony na licencji [MIT](LICENSE). Możesz go swobodnie używać, modyfikować i wdrażać w projektach prywatnych oraz komercyjnych.

