# AxionRP — plan zabezpieczeń i utwardzania (hardening)

> Dokument roboczy dla lokalnej sesji. Zawiera wyniki przeglądu bezpieczeństwa
> strony **axionrp.com** oraz konkretne kroki do wdrożenia.
> Stack: **ASP.NET Core** za reverse-proxy **nginx**. Logowanie przez **Steam OpenID**.
> Data przeglądu: 2026-07-19 (I) + 2026-07-20 (II) + 2026-09-12 (III, po rozbudowie strony).
> Zakres: przegląd pasywny + lekkie testy aktywne (nieniszczące).

---

## ✅ Stan obecny — co JUŻ jest dobrze (nie psuć)

- HTTPS/TLS 1.3, certyfikat Let's Encrypt, przekierowanie HTTP→HTTPS (301).
- HSTS: `max-age=31536000; includeSubDomains`.
- Nagłówki: `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`,
  `Referrer-Policy`, `Permissions-Policy`, `Content-Security-Policy`.
- Brak wycieku wrażliwych plików (`.env`, `.git`, `config.php`, `backup.zip` → 404).
- Brak listowania katalogów.
- Metody HTTP ograniczone (`OPTIONS` → 405).
- Logowanie przez **Steam OpenID** → brak lokalnych haseł → brak ryzyka brute-force haseł.
- `/panel` egzekwuje autoryzację (bez logowania → 302 na `/login`).
- `/api/me` bez logowania zwraca tylko `{"loggedIn":false}` — brak wycieku danych.
- Ciasteczka `axion_auth`: `Secure; HttpOnly; SameSite=Lax` — wzorcowo.
- Panel `/admin/` chroniony (HTTP Basic Auth, realm "AxionRP Admin").

---

## 🔁 Aktualizacja po ponownym przeglądzie (2026-07-20)

**Potwierdzone naprawy (weszły):**
- CSP `script-src` używa `'nonce-...'`, a nonce **rotuje per-request** (zweryfikowane).
- `community.json` / `status.json` → 200 (backend działa), zwracają tylko publiczne statystyki.
- `security.txt`, `robots.txt`, `sitemap.xml` → 200.
- Zapis przez API bez logowania zablokowany: `POST/PUT` → 405, `DELETE` → 404.
- Wrażliwe pliki nadal 404; `/.well-known/` → 403 (brak listowania).
- `/login?r=...` — `return_to` trzyma się domeny `axionrp.com` niezależnie od `r`.

**Drobne, wciąż do dopięcia (nic krytycznego):**
1. `/api/me` bez `Cache-Control: no-store` — dane zalogowanego usera mogą być
   buforowane. Dodać na endpointach z danymi konta:
   `Cache-Control: no-store, no-cache, must-revalidate` + `Pragma: no-cache`.
2. `robots.txt` wypisuje `/admin/`, `/api/`, `/panel` — publicznie zdradza mapę
   wrażliwych ścieżek. Usunąć `/admin/` z listy (jest za Basic Auth); dla `/panel`
   użyć nagłówka `X-Robots-Tag: noindex` zamiast wpisu w robots.
3. CSP `style-src` nadal ma `'unsafe-inline'` (skrypty już naprawione) — docelowo
   nonce/hash też dla stylów. Ryzyko niskie.
4. `security.txt` ujawnia prywatny Gmail — rozważyć adres roli (`security@axionrp.com`).

**Do potwierdzenia po stronie serwera (nieweryfikowalne z zewnątrz):**
- Open-redirect `r=`: potwierdzić w kodzie `Url.IsLocalUrl(returnUrl)` (wartość `r`
  jedzie w zaszyfrowanym `state`, więc finalny redirect po loginie widać tylko w kodzie).
- Rate-limiting `/admin/`: potwierdzić, że `limit_req` w nginx jest aktywny
  (celowo nie testowane atakiem).

---

## 🔁 Aktualizacja po przeglądzie III (2026-09-12)

Strona mocno urosła od poprzedniego przeglądu: doszły `/account`, `/sklep`,
`/pojazdy`, `/rankings`, system podań i sklep za walutę **AC (Axion Credits)**.
To realnie zwiększa powierzchnię ataku (operacje zmieniające stan i saldo).

**Potwierdzone naprawy (weszły):**
- `Cache-Control: no-store, no-cache, must-revalidate` + `Pragma: no-cache`
  na `/api/me`, `/api/announcements`, `/api/applications/*`, `/api/shop/*`
  oraz `Cache-Control: no-store` na stronie głównej. (pkt 1 z listy II — zrobione)
- `robots.txt` nie zdradza już `/admin/` ani `/api/`. (pkt 2 — częściowo)
- CSP `script-src 'nonce-...'` — nonce rotuje per-request (3 kolejne żądania = 3 różne nonce).
- **SPF i DMARC istnieją** (pkt 1.2 — w dużej mierze zrobione):
  `v=spf1 include:mx.ovh.com -all` oraz `v=DMARC1; p=quarantine; pct=100; ...`.
- `community.json` / `status.json` → 200, backend działa. (pkt 2.2 — zrobione)
- Limit rozmiaru żądania działa: 2 MB body → `413`.
- Tylko JSON przyjmowany: `text/plain` i `application/x-www-form-urlencoded` → `415`
  (to samo w sobie blokuje CSRF z prostego formularza HTML; w parze z `SameSite=Lax`
  ryzyko CSRF jest niskie mimo braku tokenów anty-CSRF).
- Brak nagłówków CORS (nie ma odbicia `Origin`), preflight `OPTIONS` → `405`.
- Host header injection: żądanie z obcym `Host:` → `403`. Wejście po IP na HTTPS → odrzucone.
- Nadal 404 na `.env`, `.git/config`, `config.php`, `backup.zip`, `appsettings.json`,
  `web.config`, `swagger`, `elmah.axd`, `trace.axd`; katalogi → `403`; `server_tokens off`.
- Autoryzacja na nowych endpointach trzyma: `/account` → 302 na `/login`,
  `POST /api/shop/buy` i `POST /api/applications` → `401`,
  `GET /api/applications/mine` bez sesji → `{"loggedIn":false}` (brak wycieku).

### 🔴 Najważniejsze z tego przeglądu

**A. Brak rate-limitingu — potwierdzone testem (pkt 2.1 NIE wdrożony)**
- 12 kolejnych błędnych logowań Basic Auth na `/admin/` → 12× `401`, ani jednego `503`.
  `limit_req` na `/admin/` **nie działa** (albo jest ustawiony tak luźno, że nie łapie).
- 20 żądań pod rząd na `/api/me` → 20× `200`.
- 15 nieudanych `POST /api/shop/buy` → 15× `401`, bez spowolnienia.

Teraz waży to więcej niż w lipcu: sklep wydaje walutę, a podania można spamować.
Wdrożyć `limit_req` z sekcji 2.1 **oraz** limity na `/api/` (osobna, luźniejsza strefa),
plus fail2ban na powtarzające się `401` z `/admin/`.

**B. Prywatny Gmail w dwóch publicznych miejscach**
- `security.txt` → `Contact: mailto:wosmateusz611@gmail.com`
- DMARC → `rua=mailto:wosmateusz611@gmail.com`
Oba są publicznie czytelne i będą zbierane przez spam/scrapery. Założyć adres roli
(`security@axionrp.com`, `dmarc@axionrp.com`) i podmienić w obu miejscach.

**C. Poczta — dokończyć**
- DMARC jest na `p=quarantine`; docelowo `p=reject` (po okresie obserwacji raportów).
- `aspf=r` (relaxed) → rozważyć `aspf=s`.
- **DKIM niepotwierdzony** — selektor `default._domainkey` nie istnieje. Sprawdzić
  w panelu OVH, który selektor jest aktywny, i włączyć DKIM jeśli go nie ma.

### 🟠 Drobne do dopięcia

1. `robots.txt` nadal wypisuje `/panel` i `/account`, a nagłówka `X-Robots-Tag: noindex`
   na tych ścieżkach **nie ma**. Lepiej: wyrzucić je z `robots.txt` i dodać nagłówek.
2. CSP `style-src` wciąż z `'unsafe-inline'` (skrypty już na nonce). Ryzyko niskie.
3. Brak `Cross-Origin-Opener-Policy: same-origin` i `Cross-Origin-Resource-Policy: same-origin`.
4. Brak rekordu **CAA** — dodać `axionrp.com. CAA 0 issue "letsencrypt.org"`,
   żeby nikt inny nie wystawił certyfikatu na domenę.
5. HSTS bez `preload` — jeśli domena ma zostać na HTTPS na stałe, dodać `preload`
   i zgłosić na hstspreload.org (uwaga: trudno odwrócić).
6. Kolejność kontroli: nieuwierzytelnione żądanie dostaje `400` (zły JSON), `415`
   (zły content-type) i `413` (za duże body) **zanim** dostanie `401`. Czyli autoryzacja
   jest sprawdzana po sparsowaniu ciała. Nic groźnego, ale taniej i bezpieczniej
   odrzucać brak sesji jako pierwsze (`[Authorize]` na kontrolerze / filtr przed bindingiem).
7. `sitemap.xml` nie zawiera nowych podstron (kwestia SEO, nie bezpieczeństwa).

### 🟡 Front-end — escaping (przegląd kodu JS)

Dobrze: `esc()` (przez `textContent`) jest stosowane konsekwentnie do **wszystkich
pól tekstowych** z API — tytuły i treści ogłoszeń, nicki, nazwy pakietów, opisy,
nazwy frakcji, klasy pojazdów. Nie znalazłem miejsca, gdzie tekst z serwera trafia
do `innerHTML` bez escapowania.

Do utwardzenia (dziś bezpieczne, bo serwer zwraca tam liczby):
- Pola liczbowe wstawiane są surowo: `e.rank`, `x.members`, `x.onDuty`,
  `'<button data-buy="'+p.id+'">'`. Gdyby API kiedykolwiek zwróciło tam string,
  robi się z tego XSS (a `data-buy` to dodatkowo wstrzyknięcie w atrybut).
- `nf()` / `num()` to `(v||0).toLocaleString('pl-PL')` — dla stringa
  `toLocaleString()` zwraca ten string bez zmian, więc **nie escapuje**.
- Fix: albo `esc(...)` wszędzie, albo twarde rzutowanie `Number(v)||0` w `nf`/`num`
  i `parseInt` przy `p.id`.

### 🔵 Do sprawdzenia w kodzie (z zewnątrz się nie da)

Nowe funkcje operują na pieniądzach i uprawnieniach, więc to jest teraz najważniejsza
rzecz do przejrzenia po stronie serwera:

- **Sklep `/api/shop/buy`:** cena i saldo muszą być liczone **wyłącznie na serwerze**
  na podstawie `packageId` (klient wysyła tylko `packageId` — dobrze). Sprawdzić, czy
  serwer nie ufa żadnemu polu ceny/salda z żądania.
- **Podwójne wydanie (race condition):** dwa równoległe `POST /api/shop/buy` przy
  saldzie na jeden pakiet. Potrzebna transakcja + blokada wiersza konta
  (`UPDATE ... WHERE saldo >= cena` w jednej transakcji), nie „odczytaj i zapisz".
- **IDOR na podaniach:** `/api/applications/mine` musi filtrować po ID z sesji, nigdy
  po ID z żądania. Sprawdzić też, czy nie ma endpointu przyjmującego cudze `applicationId`.
- **`isAdmin` z `/api/shop` i „widok admina" na `/pojazdy`:** ukrycie w UI nic nie daje —
  każda akcja admina musi być autoryzowana po stronie serwera przy każdym żądaniu.
- **Open-redirect `r=`** (pkt 1.1): nadal nie do zweryfikowania z zewnątrz — `r` jedzie
  w zaszyfrowanym `state` OpenID, `return_to` zawsze wskazuje `axionrp.com/signin-steam`
  niezależnie od tego, co wstawię w `r` (`//evil`, `https://evil`, `/\evil`, `%2f%2f`).
  Potwierdzić `Url.IsLocalUrl(returnUrl)` w kodzie.
- **Limity na podaniach:** ile podań na konto na dobę, limit długości opisu, walidacja
  nazwy firmy/organizacji (i jej escapowanie przy wyświetlaniu innym graczom).

### ⚪ Czego NIE dało się sprawdzić z tego środowiska

- **TLS (wersje, szyfry, certyfikat)** — ruch z tej sesji idzie przez proxy, które
  podstawia własny certyfikat, więc wynik dotyczy proxy, a nie serwera. Wyniku
  „TLS 1.0/1.1 wyłączone" **nie traktować jako potwierdzonego** — przepuścić domenę
  przez ssllabs.com/ssltest.
- **Skan portów** — z kontenera odpowiadały tylko 80 i 443, reszta (22, 25, 3306,
  5432, 6379, 27015…) timeout. To najpewniej ograniczenie wyjścia po naszej stronie,
  więc **nie jest to dowód**, że firewall jest dobrze ustawiony. Sprawdzić lokalnie
  (`ss -tulpn`, `ufw status`) albo skanem z innej maszyny.

### 🌊 DDoS — bez zmian, problem nadal aktualny

`axionrp.com` i `www` rozwiązują się **wprost na `81.210.88.68`** — Cloudflare ani
żadne proxy nie jest włączone, origin IP jest publiczne. Cała sekcja „Ochrona przed
DDoS i architektura" niżej obowiązuje w całości i nic z niej nie zostało wdrożone.

---

## 🔴 PRIORYTET 1 — do zrobienia w pierwszej kolejności

### 1.1 Open-redirect w parametrze `r=` (logowanie)
Logowanie używa `/login?r=/panel`. Trzeba potwierdzić, że po zalogowaniu aplikacja
przekierowuje **tylko na ścieżki lokalne**. Inaczej link
`axionrp.com/login?r=//zly-serwer.pl` może posłużyć do phishingu.

**Fix (ASP.NET Core):**
```csharp
// zamiast Redirect(returnUrl):
if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
    return Redirect(returnUrl);
return Redirect("/"); // fallback
```
Reguła: akceptuj tylko wartości zaczynające się od pojedynczego `/`.
Odrzucaj `//`, `http://`, `https://`, `\`, `%2F%2F` itp.

### 1.2 Poczta — SPF / DMARC / DKIM (anti-spoofing)
Nie udało się sprawdzić z zewnątrz — **zweryfikować na https://mxtoolbox.com**.
Bez tych rekordów ktoś może podszyć się pod `@axionrp.com`.

Minimalny zestaw rekordów DNS (TXT):
```
# SPF (dostosuj do realnego nadawcy poczty)
axionrp.com.        TXT  "v=spf1 -all"        # jeśli domena NIE wysyła maili
# _dmarc
_dmarc.axionrp.com. TXT  "v=DMARC1; p=reject; rua=mailto:admin@axionrp.com"
```
Jeśli domena wysyła maile (np. z hostingu) — dodać serwery do SPF i skonfigurować DKIM.

---

## 🟠 PRIORYTET 2 — utwardzanie

### 2.1 Panel `/admin/` — rate-limiting + ograniczenie po IP
Basic Auth nie ma blokady po nieudanych próbach. Dodać w nginx:
```nginx
# w http { }
limit_req_zone $binary_remote_addr zone=adminlim:10m rate=5r/m;

# w server { }
location /admin/ {
    # opcjonalnie: dostęp tylko z Twojego IP
    # allow 1.2.3.4;
    # deny all;

    limit_req zone=adminlim burst=5 nodelay;
    auth_basic "AxionRP Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://backend;
}
```
Dodatkowo rozważyć **fail2ban** na logi nginx (401 na `/admin/`).
Docelowo: właściwy panel z sesjami + **2FA** zamiast Basic Auth.

### 2.2 `community.json` / `status.json` → 502 Bad Gateway ✅ NAPRAWIONE (2026-09-12: 200)
Usługa statusu serwera (backend) jest niedostępna. `502` zdradza reverse-proxy.
- Naprawić/uruchomić usługę w tle.
- Ustawić własną, dyskretną stronę błędu:
```nginx
error_page 502 503 504 /error.html;
location = /error.html { internal; root /var/www/errors; }
```

### 2.3 Nagłówki bezpieczeństwa — dopięcie
- **CSP:** usunąć `'unsafe-inline'` ze `script-src` i `style-src`.
  Docelowo używać `nonce` generowanego per-request, np.:
  `Content-Security-Policy: script-src 'self' 'nonce-XYZ'`.
  Przenieść inline `<script>`/`<style>` do zewnętrznych plików.
- Rozważyć `Cross-Origin-Opener-Policy: same-origin` oraz
  `Cross-Origin-Resource-Policy: same-origin`.

### 2.4 `security.txt`
Dodać `/.well-known/security.txt`:
```
Contact: mailto:admin@axionrp.com
Expires: 2027-01-01T00:00:00Z
Preferred-Languages: pl, en
```

---

## 🟡 PRIORYTET 3 — higiena / warstwa serwera i aplikacji

### Serwer / infrastruktura
- [ ] Firewall (ufw/iptables): otwarte tylko 80, 443 i SSH; reszta zamknięta.
- [ ] SSH: logowanie kluczem, `PasswordAuthentication no`, `PermitRootLogin no`, fail2ban.
- [ ] Automatyczne aktualizacje bezpieczeństwa (`unattended-upgrades`).
- [ ] Kopie zapasowe **offline** (ochrona przed ransomware). Test odtwarzania.
- [ ] Ochrona anty-DDoS / WAF przed serwerem (np. Cloudflare) — opcjonalnie.
- [ ] Nie ujawniać wersji nginx: `server_tokens off;`.

### Aplikacja (ASP.NET Core)
- [ ] Walidacja i sanityzacja WSZYSTKICH danych wejściowych (formularze, query, API).
- [ ] Zapytania do bazy tylko przez parametry / ORM (żadnej konkatenacji SQL).
- [ ] Kodowanie wyjścia (output encoding) przy renderowaniu danych użytkownika — chroni przed XSS. (W JS strony jest już funkcja `esc()` — dobrze, utrzymać wszędzie.)
- [ ] Anty-CSRF tokeny na wszystkich akcjach zmieniających stan (POST/PUT/DELETE).
- [ ] Autoryzacja per-endpoint: sprawdzać uprawnienia po stronie serwera przy KAŻDYM żądaniu (nie ufać ukrywaniu przycisków w UI).
- [ ] Rate-limiting na endpointach API (nie tylko `/admin/`).
- [ ] Limity rozmiaru requestów / uploadów; walidacja typów plików jeśli są uploady.
- [ ] Logi błędów bez ujawniania stack-trace użytkownikom (produkcja: generyczne komunikaty).

### Monitoring
- [ ] Centralne logi dostępu + alerty o anomaliach (masa 401/403/404).
- [ ] Monitoring uptime.
- [ ] (Opcjonalnie) monitoring integralności plików.

---

## 🌊 Ochrona przed DDoS i architektura (strona + serwer gry na jednym VPS)

**Problem:** strona WWW i serwer Unturned stoją na **jednym VPS z jednym publicznym IP**.

Ryzyka:
- **Współdzielone zasoby** — DDoS w którekolwiek z nich zapycha całą maszynę
  (łącze/CPU/RAM). Pada jedno → pada wszystko.
- **Wyciek IP** — każdy gracz łączący się z serwerem gry **zna IP**. Skoro strona
  jest na tym samym IP, atakujący automatycznie zna też IP strony.
- **Cloudflare nie pomoże**, dopóki to samo IP jest publiczne przez serwer gry —
  atakujący pomija proxy i wali prosto w origin.
  Wniosek: **na współdzielonym IP nie da się skutecznie ukryć origin strony.**

### Rekomendowana kolejność działań

**Poziom 1 — tanie/darmowe (od razu):**
- Strona za **Cloudflare** (darmowy plan, ochrona L7) — pod warunkiem, że origin IP
  jest inny niż to znane z gry (patrz Poziom 2).
- **Firewall:** otwarte tylko potrzebne porty (443 WWW + port gry + SSH), reszta DROP.
- **Rate-limiting nginx + fail2ban** — pomaga na słabsze ataki L7. NIE zatrzyma
  dużego ataku wolumetrycznego L3/L4.

**Poziom 2 — właściwe rozwiązanie (rekomendowane):**
- **Rozdzielić na dwa IP / dwie maszyny:** strona na jednym VPS (za Cloudflare),
  serwer gry na drugim. Atak na gadżet nie kładzie strony, a IP strony pozostaje ukryte.
- **Serwer gry u dostawcy z ochroną anty-DDoS L3/L4** (np. OVH Game/VAC lub usługa
  scrubbing/filtrująca). To najważniejsze — gry są celem ataków wolumetrycznych,
  których zwykły VPS nie wytrzyma.

**Poziom 3 — dla spokoju:**
- **Tunel GRE / reverse-proxy przez scrubbing provider** dla ruchu gry, żeby prawdziwe
  IP serwera nigdy nie było widoczne graczom.

### Do rozmowy z hostingodawcą (filtry firewall)
Zapytać, czy oferują / mogą włączyć:
- **Ochronę wolumetryczną L3/L4 na łączu** (scrubbing) — kluczowe, software tego nie
  zastąpi, bo łącze zapcha się zanim ruch dojdzie do filtrowania na VPS.
- **Filtry/ACL na brzegu sieci** dla portu gry (limit pakietów/s, ochrona przed
  amplifikacją UDP, blokada spoofowanych źródeł).
- **Osobne IP** dla serwera gry i dla strony.

Przykładowe filtry po stronie VPS (uzupełnienie, nie zamiennik ochrony na łączu):
```bash
# nftables — limit nowych połączeń TCP per IP (ochrona L7/SYN flood na WWW)
nft add rule inet filter input tcp dport {80,443} ct state new \
  meter conlimit { ip saddr limit rate 60/second burst 100 packets } accept

# iptables — limit nowych połączeń na port gry (dostosuj port!)
iptables -A INPUT -p udp --dport 27015 -m hashlimit \
  --hashlimit-name gamelim --hashlimit-mode srcip \
  --hashlimit-above 200/sec --hashlimit-burst 400 -j DROP

# SYN flood — podstawowa ochrona
iptables -A INPUT -p tcp --syn -m limit --limit 20/s --limit-burst 40 -j ACCEPT
```
```nginx
# nginx — limit żądań i połączeń (ochrona L7)
limit_req_zone  $binary_remote_addr zone=wwwlim:10m rate=20r/s;
limit_conn_zone $binary_remote_addr zone=connlim:10m;
server {
    limit_req  zone=wwwlim burst=40 nodelay;
    limit_conn connlim 20;
}
```
> Uwaga: filtry na VPS łagodzą małe/średnie ataki i L7. Przy dużym wolumenie
> (dziesiątki Gb/s) ratuje wyłącznie ochrona na łączu u dostawcy.

---

## 📌 Ważna uwaga — o "skopiowaniu strony"
Nie da się technicznie uniemożliwić skopiowania front-endu (HTML/CSS/JS trafia do
przeglądarki każdego odwiedzającego). Można jedynie:
- utrudnić (minifikacja/obfuskacja JS),
- chronić prawnie (regulamin + prawa autorskie),
- trzymać realną wartość po stronie serwera (logika, baza, API) — co już jest robione.

---

## Endpointy wykryte podczas przeglądu (mapa aplikacji, stan 2026-09-12)

Strony:
- `/` (strona główna, 43 KB), `/regulamin.html` → 200
- `/pojazdy`, `/rankings`, `/sklep` → 200 (publiczne)
- `/panel` → 302 na `/login?r=/panel` (wymaga logowania)
- `/account` → 302 na `/login?r=/account` (wymaga logowania)
- `/login` → OpenID Steam (302), `/logout` → czyści `axion_auth` (302 na `/`)
- `/admin/` → HTTP Basic Auth (401)

API:
- `/api/me` → bez logowania `{"loggedIn":false}`
- `/api/announcements` → GET publiczny; POST → 405
- `/api/rankings`, `/api/pojazdy` → GET publiczny
- `/api/shop` → GET, bez logowania `{"loggedIn":false,"isAdmin":false,...}` + cennik
- `/api/shop/buy` → POST, bez sesji `401`
- `/api/applications` → POST, bez sesji `401` (GET → 405)
- `/api/applications/mine` → GET, bez sesji `{"loggedIn":false}`
- `/community.json`, `/status.json` → 200 (tylko publiczne statystyki)

## Kolejność wdrożenia — zaktualizowana 2026-09-12
1. **Rate-limiting** `/admin/` + `/api/` i fail2ban (pkt A) — potwierdzone, że nie ma.
2. **Przegląd kodu sklepu i podań**: transakcja przy zakupie (double-spend),
   IDOR na podaniach, autoryzacja akcji admina po stronie serwera.
3. **Adres roli** zamiast prywatnego Gmaila w `security.txt` i w DMARC `rua`.
4. **Poczta:** DKIM (sprawdzić selektor w OVH), potem DMARC `p=reject`.
5. Potwierdzić `Url.IsLocalUrl` dla `r=` (pkt 1.1) — jedyne, co zostało z Priorytetu 1.
6. Drobiazgi: `X-Robots-Tag`, CAA, COOP/CORP, CSP `style-src`, kolejność 401 vs 400/415.
7. Front-end: rzutowanie liczb w `nf()`/`num()` i escapowanie `p.id` w atrybucie.
8. Test TLS na ssllabs.com i lokalna weryfikacja firewalla (`ufw status`, `ss -tulpn`).
9. Priorytet 3 (higiena) i sekcja DDoS — nadal nietknięte.
