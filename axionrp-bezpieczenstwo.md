# AxionRP — plan zabezpieczeń i utwardzania (hardening)

> Dokument roboczy dla lokalnej sesji. Zawiera wyniki przeglądu bezpieczeństwa
> strony **axionrp.com** oraz konkretne kroki do wdrożenia.
> Stack: **ASP.NET Core** za reverse-proxy **nginx**. Logowanie przez **Steam OpenID**.
> Data przeglądu: 2026-07-19 (I) + 2026-07-20 (II) + 2026-07-29 (III).
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

## 🔁 Przegląd III (2026-07-29) — stan aktualny

**Strona działa** — `/` → `200 OK`, backend statusowy odpowiada, brak błędów 5xx.

### ✅ Naprawione od poprzedniego przeglądu
| Pozycja | Stan |
|---|---|
| `/api/me` bez `Cache-Control` (drobiazg 1) | ✅ jest `no-store, no-cache, must-revalidate` + `Pragma: no-cache` |
| `robots.txt` zdradzał `/admin/` i `/api/` (drobiazg 2) | ✅ oba wpisy usunięte (został `/panel`, `/panel/`, `/account`) |
| SPF / DMARC (1.2) | ✅ `v=spf1 include:mx.ovh.com -all` oraz `v=DMARC1; p=quarantine; pct=100; sp=quarantine` |
| `community.json` / `status.json` → 502 (2.2) | ✅ oba `200`, zwracają wyłącznie publiczne statystyki |
| CSP nonce dla `script-src` | ✅ potwierdzone ponownie — nonce **rotuje per-request** (3/3 różne) |
| `server_tokens` | ✅ `Server: nginx` bez wersji, także na stronie 404 |
| Cache na stronie głównej | ✅ doszedł `Cache-Control: no-store` |

Nadal trzyma się poprzedni stan: HSTS, `nosniff`, `X-Frame-Options`, `Referrer-Policy`,
`Permissions-Policy`, HTTP→HTTPS 301, brak listowania katalogów, wrażliwe pliki 404
(`.env`, `.git/config`, `config.php`, `backup.zip`, `web.config`, `appsettings*.json`,
`server-status` — wszystkie 404), `/.well-known/` → 403, `/panel` → 302 na `/login`,
`/admin/` → 401, `/api/me` bez sesji → `{"loggedIn":false}`.
Metody zapisu zablokowane: `POST/PUT/DELETE/PATCH/OPTIONS/TRACE` na `/api/announcements` → **405**.
Ciasteczko korelacyjne OpenID: `secure; samesite=lax; httponly`, `path=/signin-steam` — wzorcowo.

### 🟠 Nadal otwarte
1. **Rate-limiting `/admin/` prawdopodobnie NIE działa.** 12 szybkich żądań pod rząd →
   12× `401`, ani jednego `503`/`429`. Przy `rate=5r/m burst=5` limit powinien się odezwać.
   Wdrożyć `limit_req` z pkt 2.1 i zweryfikować, że blok `location /admin/` faktycznie go używa.
2. **CSP `style-src` wciąż z `'unsafe-inline'`** (`script-src` już czysty). Ryzyko niskie,
   ale to ostatnia dziura w CSP — docelowo nonce/hash także dla stylów.
3. **`security.txt` ujawnia prywatnego Gmaila** (`wosmateusz611@gmail.com`). Ten sam adres
   siedzi publicznie w `rua=` rekordu DMARC. Założyć `security@axionrp.com` i podmienić w obu miejscach.
4. **Brak `Cross-Origin-Opener-Policy` / `Cross-Origin-Resource-Policy`** (pkt 2.3, nadal do dodania).
5. **DMARC do zaostrzenia:** `p=quarantine` → docelowo `p=reject`, gdy potwierdzisz, że
   legalna poczta przechodzi. Rozważyć `aspf=s` zamiast `aspf=r`. **DKIM niezweryfikowany**
   (selektor nieznany z zewnątrz) — sprawdzić w panelu OVH, czy podpisywanie jest włączone.
6. **HSTS bez `preload`** — jeśli chcesz wejść na listę preload, dodać dyrektywę i zgłosić domenę.

### ⚪ Wciąż niemożliwe do potwierdzenia z zewnątrz
- **Open-redirect `r=`** — sprawdzone 5 wariantów (`//evil`, `https://evil`, `/\evil`,
  `%2F%2Fevil`, `/panel`). W każdym przypadku `openid.return_to` wskazuje na
  `https://axionrp.com/signin-steam`, a wartość `r` jedzie w zaszyfrowanym `state`.
  Z zewnątrz wygląda dobrze, ale **finalny redirect po powrocie ze Steam widać tylko w kodzie** —
  nadal trzeba potwierdzić `Url.IsLocalUrl(returnUrl)` (pkt 1.1).
- **Certyfikat TLS i otwarte porty** — środowisko przeglądu ma proxy terminujące TLS,
  więc widziany certyfikat i wynik skanu portów są bezwartościowe (kontrola potwierdziła,
  że proxy podmienia issuer i blokuje surowy TCP). Certyfikat sprawdzić z własnej maszyny
  lub na SSL Labs; porty przez `nmap` spoza VPS-a.

### ⚠️ Uwaga architektoniczna (bez zmian)
`axionrp.com` i `www.axionrp.com` rozwiązują się na **81.210.88.68** — origin wystawiony
bezpośrednio, bez CDN/Cloudflare przed nim. Cała sekcja "Ochrona przed DDoS" niżej pozostaje aktualna.

### Drobiazg poza bezpieczeństwem
`HEAD` na `/panel` i `/login` zwraca `405`. Nie jest to luka, ale monitoring uptime
i część crawlerów używa `HEAD` — warto obsłużyć.

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

### 1.2 Poczta — SPF / DMARC / DKIM (anti-spoofing) — ✅ ZROBIONE (2026-07-29)
Rekordy są na miejscu:
```
axionrp.com.        TXT  "v=spf1 include:mx.ovh.com -all"
_dmarc.axionrp.com. TXT  "v=DMARC1; p=quarantine; pct=100; rua=...; sp=quarantine; aspf=r"
```
Zostaje tylko dostrojenie: `p=reject`, `aspf=s`, weryfikacja DKIM w panelu OVH
oraz podmiana prywatnego Gmaila w `rua=` na adres roli.

<details><summary>Oryginalna rekomendacja (archiwum)</summary>

Minimalny zestaw rekordów DNS (TXT):
```
# SPF (dostosuj do realnego nadawcy poczty)
axionrp.com.        TXT  "v=spf1 -all"        # jeśli domena NIE wysyła maili
# _dmarc
_dmarc.axionrp.com. TXT  "v=DMARC1; p=reject; rua=mailto:admin@axionrp.com"
```
Jeśli domena wysyła maile (np. z hostingu) — dodać serwery do SPF i skonfigurować DKIM.
</details>

---

## 🟠 PRIORYTET 2 — utwardzanie

### 2.1 Panel `/admin/` — rate-limiting + ograniczenie po IP — ⚠️ NADAL DO ZROBIENIA
Test 2026-07-29: 12 żądań pod rząd → 12× `401`, zero `503`. Limit nie działa.
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

### 2.2 `community.json` / `status.json` → ✅ NAPRAWIONE (oba `200`)
Backend statusowy działa. Zostaje opcjonalnie własna strona błędu na wypadek
kolejnej awarii (`/error.html` obecnie zwraca 404, czyli nie jest skonfigurowana):
```nginx
error_page 502 503 504 /error.html;
location = /error.html { internal; root /var/www/errors; }
```

### 2.3 Nagłówki bezpieczeństwa — dopięcie (`script-src` ✅, `style-src` ⚠️)
- **CSP:** ~~`script-src`~~ ✅ zrobione (nonce per-request). Zostaje `'unsafe-inline'` w `style-src`.
  Docelowo używać `nonce` generowanego per-request, np.:
  `Content-Security-Policy: script-src 'self' 'nonce-XYZ'`.
  Przenieść inline `<script>`/`<style>` do zewnętrznych plików.
- Rozważyć `Cross-Origin-Opener-Policy: same-origin` oraz
  `Cross-Origin-Resource-Policy: same-origin`.

### 2.4 `security.txt` — ✅ istnieje (⚠️ do poprawki adres kontaktowy)
Plik `/.well-known/security.txt` odpowiada `200`, `Expires` ustawione na 2027-07-19.
Do zmiany: `Contact` wskazuje na prywatnego Gmaila. Docelowo:
```
Contact: mailto:security@axionrp.com
Expires: 2027-07-19T00:00:00Z
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

## Endpointy wykryte podczas przeglądu (mapa aplikacji)
- `/` (strona główna, statyczny HTML, 31 KB)
- `/login` → OpenID Steam (302)
- `/logout` → czyści `axion_auth` (302 na `/`)
- `/panel` → wymaga logowania (302 na `/login?r=/panel`)
- `/api/me` → JSON, bez logowania `{"loggedIn":false}`
- `/api/announcements` → JSON publiczny (ogłoszenia)
- `/community.json`, `/status.json` → 200 (backend działa)
- `/signin-steam` → callback OpenID (ustawia ciasteczko korelacyjne)
- `/admin/` → HTTP Basic Auth (401)
- `/account` → wymieniony w `robots.txt`
- `/robots.txt`, `/sitemap.xml`, `/.well-known/security.txt` → 200

## Kolejność wdrożenia — stan na 2026-07-29
1. ⚠️ **2.1 rate-limit `/admin/`** — potwierdzone, że nie działa. Najpilniejsze.
2. ⚠️ **1.1 open-redirect `r=`** — sprawdzić `Url.IsLocalUrl` w kodzie (z zewnątrz OK).
3. 🟡 2.4 podmiana kontaktu w `security.txt` + `rua=` DMARC na adres roli.
4. 🟡 2.3 `style-src` bez `'unsafe-inline'` + COOP/CORP.
5. 🟡 DMARC `p=reject`, weryfikacja DKIM w OVH.
6. ✅ 1.2 SPF/DMARC, 2.2 backend statusowy, CSP `script-src` — zrobione.
7. Priorytet 3 (higiena) — sukcesywnie.
8. 🌊 Sekcja DDoS — origin nadal wystawiony bezpośrednio, temat otwarty.
