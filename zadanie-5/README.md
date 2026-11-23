# Zadanie 5 – migracja portfolio.local na HTTPS

- Certyfikat dla `portfolio.local` wystawiony własnym urzędem z zadania 4 (`zadanie-4/procedura.md`).
- Port 80 tylko przekierowuje (301) na HTTPS; dotychczasowa konfiguracja z L13–L14 bez zmian.
- `ssl_certificate` wskazuje `fullchain.pem` (certyfikat + pośredni CA).
- Profil intermediate: TLS 1.2 i 1.3, szyfry ECDHE z AEAD.
- HSTS etapami: najpierw `max-age=300`, po sprawdzeniu `max-age=86400`.

## Pliki
- `portfolio.local.przed` / `portfolio.local.po` – konfiguracja przed i po
- `zmiany.diff` – różnice (tylko dopisane linie, oryginalne bez zmian)
- `plan-wycofania.md` – jak wrócić i czego nie da się cofnąć

## Sprawdzenie
```
curl -I http://portfolio.local                       → 301, Location: https://portfolio.local/
curl --cacert korzen.crt -I https://portfolio.local  → HTTP/2 200, strict-transport-security: max-age=86400
openssl s_client ... -tls1_3                         → TLSv1.3, Verify return code: 0 (ok)
openssl s_client ... -tls1_1 -cipher 'DEFAULT:@SECLEVEL=0' → alert protocol version (TLS 1.1 odrzucony)
```

Chrome (F12 → Network): `Strict-Transport-Security: max-age=300`, po zmianie `max-age=86400`,
pseudo-nagłówki `:authority`, `:path` = HTTP/2.
`chrome://net-internals/#hsts` → Chrome zapamiętał politykę dla `portfolio.local`.

## Mixed content
`grep -rn "http://" /var/www/portfolio.local --include='*.html' --include='*.css' --include='*.js'`
→ brak wyników. Strona używa ścieżek względnych (`css/styl.css`), konsola Chrome bez ostrzeżeń.
