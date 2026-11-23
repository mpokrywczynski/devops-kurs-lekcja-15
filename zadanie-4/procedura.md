# Wewnętrzne PKI – procedura wystawienia certyfikatu serwera

## Budowa urzędu

```
Lab DevOps Root CA          korzen.crt    RSA 4096, 10 lat, CA:TRUE, pathlen:1
  └─ Lab DevOps Intermediate CA   posredni.crt  EC P-256, 5 lat, CA:TRUE, pathlen:0
       ├─ api.wewnetrzna.test     200 dni, serverAuth
       └─ panel.wewnetrzna.test   200 dni, serverAuth
```

- Certyfikaty serwerów podpisuje **pośredni** urząd. Klucz korzenia służył tylko do podpisania pośredniego.
- `pathlen:1` na korzeniu pozwala na jeden poziom pośredni. Przy `pathlen:0` łańcuch się nie domyka
  (`openssl verify`: `path length constraint exceeded`).
- Klucze prywatne (`*.key`) mają prawa 600 i nie trafiają do Gita (`.gitignore`).

## Wystawienie nowego certyfikatu

Wszystkie polecenia wykonuję w katalogu urzędu:

```bash
cd ~/devops-kurs/lekcja-15/zadanie-4
```

### 1. Nazwa strony

```bash
N=nowa.wewnetrzna.test
```

### 2. Klucz prywatny serwera

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out "$N.key"
chmod 600 "$N.key"
```

### 3. Wniosek o certyfikat (CSR)

```bash
openssl req -new -utf8 -key "$N.key" -out "$N.csr" -subj "/C=PL/O=Lab DevOps/CN=$N"
```

### 4. Rozszerzenia certyfikatu

```bash
printf 'basicConstraints = critical, CA:FALSE\nkeyUsage = critical, digitalSignature\nextendedKeyUsage = serverAuth\nsubjectAltName = DNS:%s\n' "$N" > "$N-ext.cnf"
```

SAN jest obowiązkowy – przeglądarki sprawdzają nazwę tylko w SAN, pole CN ignorują.
`openssl x509 -req` nie kopiuje rozszerzeń z CSR, dlatego podaję je w osobnym pliku.

### 5. Podpisanie certyfikatu urzędem pośrednim (200 dni)

```bash
openssl x509 -req -in "$N.csr" -CA posredni.crt -CAkey posredni.key -CAcreateserial \
  -out "$N.crt" -days 200 -sha256 -extfile "$N-ext.cnf"
```

Plik `posredni.srl` trzyma licznik numerów seryjnych – nie kasować.

### 6. Pełny łańcuch dla serwera

```bash
cat "$N.crt" posredni.crt > "$N-fullchain.pem"
```

Najpierw certyfikat serwera, pod nim pośredni. Korzenia nie dokładam – klient ma go u siebie.

### 7. Sprawdzenie

```bash
openssl verify -CAfile korzen.crt -untrusted posredni.crt "$N.crt"   # → OK
openssl x509 -in "$N.crt" -noout -ext subjectAltName                 # → DNS:$N
```

### 8. Wdrożenie w Nginksie

```bash
sudo mkdir -p /etc/nginx/ssl "/var/www/$N"
sudo cp "$N-fullchain.pem" "$N.key" /etc/nginx/ssl/
sudo chmod 600 "/etc/nginx/ssl/$N.key"
echo "<h1>$N</h1>" | sudo tee "/var/www/$N/index.html"
sudo sed "s/api\.wewnetrzna\.test/$N/g" /etc/nginx/sites-available/api.wewnetrzna.test \
  | sudo tee "/etc/nginx/sites-available/$N" > /dev/null
sudo ln -sf "/etc/nginx/sites-available/$N" /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Konfiguracja powstaje z działającego pliku `api.wewnetrzna.test` – zamieniam w nim nazwę
(5 miejsc: 2× `server_name`, `root`, `ssl_certificate`, `ssl_certificate_key`).
Wzorcowe pliki są w katalogu `nginx/`.

### 9. Test

```bash
echo "127.0.0.1 $N" | sudo tee -a /etc/hosts
curl --cacert korzen.crt -I "https://$N"    # → HTTP/2 200
curl -I "http://$N"                          # → 301 na https://
openssl s_client -connect "$N:443" -servername "$N" -CAfile korzen.crt -showcerts </dev/null 2>/dev/null \
  | grep -E ' s:|Verify return'              # pozycje 0 i 1, Verify return code: 0 (ok)
```

## Zaufanie do urzędu

- **WSL (curl, git):** `sudo cp korzen.crt /usr/local/share/ca-certificates/lab-devops-root.crt && sudo update-ca-certificates`
- **Chrome w Windowsie:** import `korzen.crt` do „Zaufane główne urzędy certyfikacji”
  (bieżący użytkownik) + wpis w hosts Windowsa na IP WSL.
- Po kursie korzeń usuwam z magazynów – kto zdobędzie `korzen.key`, może wystawić certyfikat na dowolną nazwę.
