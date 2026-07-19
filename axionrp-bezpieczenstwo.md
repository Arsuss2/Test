# AxionRP — plan zabezpieczeń i utwardzania (hardening)

> Dokument roboczy dla lokalnej sesji. Zawiera wyniki przeglądu bezpieczeństwa
> strony **axionrp.com** oraz konkretne kroki do wdrożenia.
> Stack: **ASP.NET Core** za reverse-proxy **nginx**. Logowanie przez **Steam OpenID**.
> Data przeglądu: 2026-07-19. Zakres: przegląd pasywny + lekkie testy aktywne (nieniszczące).

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
