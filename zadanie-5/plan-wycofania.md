# Plan wycofania migracji portfolio.local na HTTPS

## Kiedy wycofujemy
Strona nie działa po HTTPS: `nginx -t` nie przechodzi, błąd certyfikatu, odpowiedzi 5xx.

## Najpierw: naprawić zamiast wycofywać
Przy błędzie certyfikatu (wygasł, zła nazwa) wystawić nowy według `zadanie-4/procedura.md`
i przeładować Nginksa. To szybsze i nie psuje HSTS.

## Wycofanie konfiguracji
1. `sudo cp zadanie-5/portfolio.local.przed /etc/nginx/sites-available/portfolio.local`
2. `sudo nginx -t && sudo systemctl reload nginx`
3. Sprawdzenie: `curl -I http://portfolio.local` → 200 (bez przekierowania)

## Czego nie da się cofnąć: HSTS
Przeglądarka, która dostała `Strict-Transport-Security: max-age=86400`, przez dobę sama
zamienia http:// na https:// i nie pozwala kliknąć „kontynuuj mimo to” przy błędzie certyfikatu.
Powrót serwera do samego HTTP tego nie zmienia – u tych użytkowników strona przestanie działać.

Co można zrobić:
- przed wyłączeniem HTTPS wysłać przez HTTPS `max-age=0` – przeglądarka, która wejdzie, skasuje wpis;
- we własnym Chrome: `chrome://net-internals/#hsts` → „Delete domain security policies” → `portfolio.local`;
- u pozostałych użytkowników – tylko czekać do końca `max-age`.

Dlatego HSTS wdrażamy etapami: 300 s → 86400 s, a rok i `includeSubDomains` dopiero po
miesiącu bez problemów. `preload` nie włączamy wcale.
