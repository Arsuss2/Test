# AxionRP — plan zabezpieczeń i utwardzania (hardening)

> Dokument roboczy dla lokalnej sesji. Zawiera wyniki przeglądu bezpieczeństwa
> strony **axionrp.com** oraz konkretne kroki do wdrożenia.
> Stack: **ASP.NET Core** za reverse-proxy **nginx**. Logowanie przez **Steam OpenID**.
> Data przeglądu: 2026-07-19 (I) + 2026-07-20 (ponowny, po wdrożeniu poprawek).
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

### 2.2 `community.json` / `status.json` → 502 Bad Gateway
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

## Endpointy wykryte podczas przeglądu (mapa aplikacji)
- `/` (strona główna, statyczny HTML, 31 KB)
- `/login` → OpenID Steam (302)
- `/logout` → czyści `axion_auth` (302 na `/`)
- `/panel` → wymaga logowania (302 na `/login?r=/panel`)
- `/api/me` → JSON, bez logowania `{"loggedIn":false}`
- `/api/announcements` → JSON publiczny (ogłoszenia)
- `/community.json`, `/status.json` → obecnie 502 (backend down)
- `/admin/` → HTTP Basic Auth (401)

## Kolejność wdrożenia (sugerowana)
1. 1.1 open-redirect `r=`  → 1.2 SPF/DMARC
2. 2.1 rate-limit `/admin/`  → 2.2 naprawa 502
3. 2.3 CSP nonce  → 2.4 security.txt
4. Priorytet 3 (higiena) — sukcesywnie.
