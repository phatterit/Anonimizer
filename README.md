# Anonimizer — Air-Gapped AI Data Pseudonymization & Restoration Tool

[![Security: Air-Gapped](https://img.shields.io/badge/Security-Air--Gapped%20%2F%20RAM--only-success?style=flat-square)](#model-bezpiecze%C5%84stwa--security-model)
[![CSP: default-src 'none'](https://img.shields.io/badge/CSP-default--src%20'none'-blue?style=flat-square)](#polityka-bezpiecze%C5%84stwa-csp)
[![Zero Dependencies](https://img.shields.io/badge/Dependencies-0%20(Pure%20Vanilla)-brightgreen?style=flat-square)](#architektura)
[![Author: hatterp](https://img.shields.io/badge/Author-hatterp-orange?style=flat-square)](#autor)

Maksymalnie bezpieczna, lekka i samowystarczalna aplikacja webowa (działająca jako pojedynczy, niezależny plik HTML) przeznaczona do lokalnej pseudonimizacji wrażliwych danych (kod źródłowy, konfiguracje sieciowe i serwerowe, logi, hasła, klucze API, adresy IP, dane osobowe / PII) przed wysłaniem ich do chmurowych modeli sztucznej inteligencji (LLM, np. ChatGPT, Claude, Gemini), z możliwością natychmiastowego przywrócenia oryginalnych danych (deanonimizacji 1:1) po powrocie odpowiedzi z AI.

Dostępna jako pojedyncze, niezależne pliki HTML:
* 🇵🇱 **Polska (główna / rozwojowa)**: [`anonimizer.html`](anonimizer.html) oraz zsynchronizowany [`index.html`](index.html)
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
   * **Runtime lockdown API sieciowych, WebRTC i dyskowych**: w środowisku uruchomieniowym wyłączane i unieważniane są funkcje `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `Worker`, `SharedWorker`, `RTCPeerConnection`, `webkitRTCPeerConnection`, `navigator.sendBeacon` oraz `window.open`.

2. **Ochrona danych wyłącznie w pamięci ulotnej (RAM-only):**
   * **Zero zapisu na dysku**: aplikacja nie używa `localStorage`, `sessionStorage`, `IndexedDB`, `caches`, `cookieStore` ani plików `cookies` (dostęp do tych API jest zablokowany).
   * **Bezpieczny słownik mapowań**: sekretne mapowania (`token <-> oryginalna wartość / zestaw wariantów`) są przechowywane wyłącznie w ulotnej strukturze pamięci JavaScript (`Map()` / `Set()`) w bieżącej karcie przeglądarki. Po odświeżeniu lub zamknięciu karty wszystkie dane bezpowrotnie znikają z pamięci.
   * **Ochrona metadanych w tabeli aktywnych mapowań**: długość sekretu pozostaje ukryta (`—`) aż do momentu świadomego kliknięcia przycisku `Pokaż` / `Show`.
   * **Skaner ryzyka rezydualnego (`residualRisks`)**: po każdym maskowaniu analizuje tekst wynikowy pod kątem potencjalnie pozostawionych śladów (np. nieoczekiwany adres IP, e-mail, PESEL, XML/SQL z sekretem, webhook lub wartość przy kluczu hasła) i wyświetla ostrzeżenie ze wskazaniem numeru linii.
   * **Funkcja natychmiastowego czyszczenia (PANIC / Clear)**:
     * Dedykowany przycisk `PANIC · Wyczyść wszystko` (skrót klawiszowy `Alt` + `Shift` + `X`) oraz przyciski `Wyczyść` w obu kolumnach.
     * Natychmiast zeruje pola tekstowe (łącznie z polem wyszukiwania fraz), czyści historię cofania przeglądarki (`Undo` / `Ctrl+Z`), czyści schowek systemowy, nadpisuje i usuwa referencje do sekretów w pamięci RAM oraz wywiera nacisk na mechanizm Garbage Collectora przeglądarki.

---

## ⚙️ Możliwości i silnik detekcji

Aplikacja podzielona jest na 2 czytelne kroki (układ dwukolumnowy):

### Krok 1: Anonimizacja (Prompt OUT — przed wysłaniem do AI)
Przed skanowaniem `normalizeScanText` bezpiecznie usuwa niewidoczne znaki o zerowej szerokości (zero-width chars) oraz kontekstowo normalizuje zapis *defanged* (`10[.]0[.]0[.]1`, `user@firma[.]pl`, `[dot]`), zachowując w stanie nienaruszonym wyrażenia regularne w kodzie (np. `/a[.]b/`). Następnie silnik wykrywa i zamienia wrażliwe dane na spójne tokeny z możliwością włączania/wyłączania poszczególnych kategorii w czasie rzeczywistym:
* **IP (v4/v6, CIDR)**: pełne i skrócone adresy IPv6, IPv4 z maskami CIDR (`<IP_01>`), z zaawansowaną ochroną przed fałszywym maskowaniem numerów wersji oprogramowania (np. `'version' => '30.0.1.2'`, `v1.2.3.4`).
* **E-mail**: adresy poczty elektronicznej (`<EMAIL_01>`).
* **Sekrety**: hasła, klucze i tokeny przypisywane przez `=`, `:`, `:=`, `=>` (w tym polskie nazwy kluczy z diakrytykami np. `hasło_użytkownika`, `klucz_główny`, a także flagi CLI bez sklejania kolejnych flag `--user`/`--port`), listy YAML, tagi XML z sekretami, klauzule SQL (`IDENTIFIED BY`, `PASSWORD`), ciasteczka sesyjne (`Set-Cookie`), webhooki (Slack/Discord/Teams), frazy seed (BIP-39), bloki PEM certyfikatów i kluczy prywatnych (`-----BEGIN PRIVATE KEY-----`), konfiguracje Cisco/BGP/OSPF/SNMP/RADIUS oraz poświadczenia w URL (`scheme://user:pass@host`).
* **Tokeny / hashe / klucze**: tokeny `Bearer`, klucze AWS (`AKIA...`), GitHub (`ghp_...`), GitLab, Slack, OpenAI/Anthropic/Stripe (`sk-...`), JWT, klucze publiczne SSH oraz ciągi o wysokiej entropii / base64 / hex powyżej 32 znaków (`<TOKEN_01>`).
* **FQDN / Hosty**: domeny internetowe i nazwy serwerów wewnątrz sieci (`<HOST_01>`), z wykluczeniem nazw przestrzeni konfiguracyjnych (np. `memcache.local`), schematów URI i klas PHP.
* **MAC**: adresy fizyczne kart sieciowych (`<MAC_01>`).
* **ID / Konta / Karty**: PESEL, NIP i REGON z **walidacją sum kontrolnych** (także z separatorami), księgi wieczyste (`KW`), identyfikatory UUID, identyfikatory chmurowe (AWS ARN, instancje EC2, GCP), numery kont IBAN (w tym ze spacjami) oraz polskie 26-cyfrowe numery NRB, numery kart płatniczych (Visa, Mastercard, Amex z walidacją algorytmem Luhna), sygnatury akt komorniczych/sądowych (`KM ...`), numery dowodów osobistych (`<ID_01>`).
* **User / Login**: nazwy użytkowników w konfiguracjach (`dbuser`, `superuser`, `username`, `login`), ścieżki katalogów domowych `/home/user`, komendy CLI (`sudo`, `psql -U`), logi SSH (`Failed password for user`).
* **Porty**: parametry `port=`, definicje w URL/hostach (`:5432`, `[::1]:8080`), dyrektywy `Listen`, `Expose`, flagi kontenerów Docker `-p 8080:80`.
* **Dane osobowe / PII**: imiona i nazwiska (wraz z tytułami `Pan`, `dr`, `lekarz`, `konsultant`, automatyczną **odmianą polskich nazwisk przez przypadki** oraz obsługą nazwisk dwuczłonowych z dywizem np. `Nowak-Wiśniewska` / `Nowak-Wiśniewskiej`), daty urodzenia, adresy zamieszkania (`ul. ...`, kody pocztowe), współrzędne GPS, dane medyczne, numery telefonów stacjonarnych i komórkowych (`+48 ...`), nazwy spółek i organizacji (`Sp. z o.o.`, `S.A.`, placówki medyczne).

#### Wysuwana wyszukiwarka fraz, podgląd w oknie modalnym i maskowanie ręczne:
* **Wysuwana wyszukiwarka fraz w tekście źródłowym (`#findToggleBtn` → `#findBar`)**:
  * Domyślnie zwinięta, aby nie zajmować miejsca przy codziennym maskowaniu — rozwija się po kliknięciu przycisku **„🔍 Wyszukaj frazę w tekście”**.
  * Natychmiastowe zliczanie wystąpień w tekście źródłowym oraz nawigacja `◀` / `▶` (`Enter` / `Shift+Enter`) podświetlająca i przewijająca pole do kolejnych trafień.
  * Przełączniki **„Ignoruj wielkość liter (a=A)”** oraz **„Tylko całe słowa”** pozwalające znaleźć i zgrupować różne warianty zapisu (np. `TajnaFirma`, `tajnafirma`, `TAJNAFIRMA`).
  * Kliknij **Zamaskuj wszystkie znalezione jako wybrany token** (lub `Ctrl+Enter`), aby przypisać wszystkie trafienia naraz do wskazanego tokena.
* **Podgląd „Zanonimizowany tekst — bezpieczny prompt” w oknie modalnym (`#openModalBtn` → `#previewModal`)**:
  * Pełnoekranowe okno podglądu z numeracją wierszy, wizualnym wyróżnieniem tokenów (`<TOKEN_NN>` jako niebieskie pigułki) oraz podświetleniem linii wykrytych przez skaner ryzyka rezydualnego.
  * Zaznaczenie dowolnego pominiętego fragmentu myszką w podglądzie automatycznie wpisuje go do paska szybkiego maskowania u góry modala i pozwala jednym kliknięciem (**„Zamaskuj w tekście”**) przypisać wszystkie jego wystąpienia do nowego lub istniejącego tokena bez zamykania okna.
* **Wskazanie docelowego tokena (`#targetTok` / `#lbl`)**:
  * Wygenerowanie **nowego tokena** z predefiniowanej listy 11 etykiet lub własnej etykiety (np. `HASLO` → `<HASLO_01>`, `NAZWA` → `<NAZWA_01>`),
  * Wskazanie **istniejącego tokena** z listy rozwijanej **„Wskaż określony token”** (aby dopiąć kolejne frazy lub warianty do tego samego placeholdera),
  * Lub wpisanie jawnego tokena z numerem bezpośrednio w polu etykiety (np. `<USER_05>` lub `USER_05`).

#### Tabela aktywnych mapowań:
Podgląd wszystkich zamian: typ (z oznaczeniem liczby wariantów np. `×3`), identyfikator tokena, długość ciągu (ukryta jako `—` do czasu odsłonięcia), liczba użyć, bezpieczne odsłanianie wartości gwiazdkowanych na żądanie (`Pokaż` / `Ukryj`) oraz możliwość usunięcia pojedynczej reguły (zabezpieczenie przed atakami XSS przez `textContent`).

### Krok 2: Przywracanie danych (Prompt IN — po powrocie z AI)
* Wklej surową odpowiedź zwróconą przez model AI zawierającą tokeny (np. `<IP_01>`, `<SECRET_01>`).
* Kliknij **Przywróć oryginalne dane** (lub naciśnij `Ctrl` + `Enter`).
* Aplikacja wykonuje bezpieczne jednoetapowe mapowanie 1:1, wstawiając oryginalne wartości z pamięci RAM w miejsce wygenerowanych placeholderów, bez wysyłania czegokolwiek do sieci.

---

## 🚀 Jak uruchomić?

Aplikacja **nie wymaga instalacji, serwera, Node.js ani połączenia z Internetem**:

1. Pobierz repozytorium lub pojedynczy plik:
   * [Wersja polska (`anonimizer.html` / `index.html`)](anonimizer.html)
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
| `Enter` / `Shift` + `Enter` (w polu wyszukiwania) | Przejdź do następnego / poprzedniego wystąpienia frazy |
| `Ctrl` + `Enter` (w polu wyszukiwania) | Zamaskuj wszystkie znalezione wystąpienia jako wybrany token |
| `Esc` (w oknie podglądu modal) | Zamknij okno podglądu bezpiecznego promptu |
| `Ctrl` + `Enter` (w polu Kroku 2) | Przywróć oryginalne dane |
| `Alt` + `Shift` + `X` | **PANIC**: Natychmiastowe wyczyszczenie pól, RAM-u i schowka |

---

## 👨‍💻 Autor

Projekt stworzony przez: **hatterp**  
Architektura: *Air-Gapped / RAM-Only / Zero Network Footprint*

---

## 📄 Licencja

Projekt udostępniony na licencji [MIT](LICENSE). Możesz go swobodnie używać, modyfikować i wdrażać w projektach prywatnych oraz komercyjnych.


---

## 🇬🇧 English Overview (Quick Reference)

**Anonimizer** is an ultra-secure, zero-dependency, single-file web application for local pseudonymization and 1:1 de-anonymization of sensitive data (source code, server configs, network logs, passwords, API keys, IP addresses, financial IDs, and PII) before sending prompts to AI/LLM services.

* **Standalone English Version**: [`index-en.html`](index-en.html)
* **Standalone Polish Version**: [`anonimizer.html`](anonimizer.html) / [`index.html`](index.html)
* **Full Technical & Security Documentation**: [`SYSTEM_OVERVIEW.md`](SYSTEM_OVERVIEW.md)

### Key Security Guarantees:
* **Strict CSP (`default-src 'none'`)**: Blocks all outbound connections, external scripts, fonts, images, and stylesheets. Includes an automatic real-time CSP self-test badge.
* **Runtime API Lockdown**: Overrides and disables `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `Worker`, `SharedWorker`, `RTCPeerConnection`, `webkitRTCPeerConnection`, `sendBeacon`, `localStorage`, `sessionStorage`, `indexedDB`, `caches`, `cookieStore`, and `document.cookie`.
* **Volatile RAM-Only Vault**: Secret mappings live exclusively in JavaScript `Map`/`Set` structures inside the current browser tab. Secret lengths in the active mappings table remain hidden (`—`) until explicitly revealed.
* **Residual Risk Scanner**: Post-masking heuristic scan (`residualRisks`) highlights any obvious traces (IPs, emails, PESEL, XML/SQL secrets, webhooks, key-value leftovers) by line number before you send the prompt.
* **PANIC Button (`Alt + Shift + X`)**: Instantly wipes all textareas (including search input), destroys native browser undo history (`Ctrl+Z`), clears the system clipboard, overwrites secret references in RAM, and pressures the Garbage Collector.

### Search & Assign to Token, Preview Modal + Detection Categories:
* **Expandable Source Phrase Search & Bulk Assignment (`#findToggleBtn` → `#findBar`)**: Collapsed by default and expanded on click. Finds occurrences in `#src` (case-insensitive / whole-word), steps through matches (`◀`/`▶` or `Enter`/`Shift+Enter`), and assigns all matches at once to a new token from a preset/custom label, an existing token from the **Target specific token** (`#targetTok`) dropdown, or an explicit numbered token (e.g. `<USER_05>`).
* **Safe Prompt Preview Modal (`#openModalBtn` → `#previewModal`)**: Full-window preview of the anonymized output with line numbers, highlighted token pills (`<TOKEN_NN>`), amber markers on residual-risk lines, and an inline masking bar — highlighting any missed phrase copies it to the top bar for one-click masking.
1. **IP (`#tg_IP`)**: IPv4 (with CIDR masks & version-number false-positive protection) and IPv6.
2. **E-mail (`#tg_EMAIL`)**: Email addresses (plus context-aware defanged normalization for `user@host[.]tld`).
3. **Secrets (`#tg_SECRET`)**: Key-value secrets (`password`, `secret`, `token`, `api_key`, `auth`, `cvc`/`cvv`, Polish keys with diacritics), PEM private keys/certificates, YAML lists, XML secret tags, SQL `IDENTIFIED BY`/`PASSWORD`, `Set-Cookie`, webhooks, BIP-39 seed phrases, URL credentials, Cisco/BGP/OSPF/SNMP secrets.
4. **Tokens (`#tg_TOKEN`)**: `Bearer` tokens, JWT, AWS, GitHub, GitLab, Slack, OpenAI/Anthropic/Stripe keys, SSH public keys, and high-entropy hex/base64 strings (>= 32 chars).
5. **FQDN / Hosts (`#tg_HOST`)**: Domain names and internal URLs (with config-key & URI scheme exclusion).
6. **MAC (`#tg_MAC`)**: Hardware MAC addresses.
7. **IDs / Accounts / Cards (`#tg_ID`)**: Checksum-validated PESEL, NIP & REGON, land registry (`KW`), UUIDs, Cloud ARNs/IDs, IBAN & 26-digit bank accounts, Luhn-validated payment cards, document numbers, and case signatures.
8. **User / Login (`#tg_USER`)**: Config usernames (`dbuser`, `superuser`, `login`), `/home/` paths, CLI user flags, and SSH/sudo logs.
9. **Ports (`#tg_PORT`)**: `port=` assignments, `host:port`, `[ipv6]:port`, server directives (`Listen`, `Expose`), and Docker `-p` mappings.
10. **PII (`#tg_PII`)**: Full names & titles (with Polish case inflection & hyphenated surnames), dates of birth, phone numbers, street/postal addresses, GPS coordinates, medical terms, and company/organization names.
