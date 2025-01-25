# Na MACu

## Inštalácie Z INTERNETU

```
https://code.visualstudio.com
https://www.mongodb.com/try/download/compass
https://tableplus.com
```

---

## Inštalácie CEZ TERMINAL

### inštalácia Homebrew

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

echo >> /Users/<USER>/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/<USER>/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

brew --version

brew update
```

### inštalácia Git

```
brew install git

git --version

git config --global user.name "<USERNAME>"
git config --global user.email "<USEREMAIL>"
```

### SSH kľúč

```
ssh-keygen -t ed25519 -C "<EMAIL>"

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

pbcopy < ~/.ssh/id_ed25519.pub
cat ~/.ssh/id_ed25519.pub
```

## inštalácia NODE

```
brew install node

node --version
npm --version
```

## inštalácia Yarn

```
brew install yarn

yarn --version
```

### inštalácia MongoDB

```
brew tap mongodb/brew
brew update

brew install mongodb-community@8.0
```

### inštalácia Mailpit

```
brew install mailpit

brew services start mailpit
```

### služba MongoDB

```
brew services start mongodb-community@8.0
brew services stop mongodb-community@8.0

brew services list
```

---

## Vývojové prostredie

### inštalácia NGINX

```
brew install nginx
```

```
sudo nano /opt/homebrew/etc/nginx/nginx.conf
```

```
sudo nginx                   # spustenie
sudo nginx -s reload         # reštart
sudo nginx -s stop           # okamžité zastavenie
sudo nginx -s quit           # umozní procesom dokončiť aktuálnej požiadavky
sudo nginx -t                # kontrola konfiguracie

ps aux | grep nginx         # kontrola spustených procesov
sudo kill -9 <PID>          # vynútené zastavenie

brew services start nginx   # spustenie služby
brew services stop nginx    # zastavenie služby
brew services list          # zobrazenie služieb

sudo lsof -i :80        # kontrola portu
sudo kill <PID>         # zastavenie
```

### nastavenie lokálnych domén

nastaviť domény

```
sudo nano /etc/hosts

127.0.0.1   local.com
127.0.0.1   local.sk
```

### nastavenie HTTPS

konfiguračný súbor `openssl-san.cnf` na generovanie certifikátov s podporou rozšírenia SAN - `Subject Alternative Name`

```
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[dn]
CN = local.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = local.com
DNS.2 = local.sk
```

popis: `[req]` základné paramentre žiadosti o certifikát, `DN` definuje hlavnú doménu CN - Common Name, `[req_ext]` definuje rozšírenia certifikátu subjectAltName, `[alt_names]` definuje alternatívne názvy domén alebo subdomén

vytvorenie samo-podpísaného certifikátu

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout local.key \
    -out local.crt \
    -config openssl-san.cnf
```

pridať `local.crt` do `Kľúčenka` pre Systém a nastaviť `Vždy dôverovať`

```
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain local.crt
```

popis: `-d -r trustRoot` nastaví ako doveryhodný certifikát pre všetky účty, uloží do systémovej kľúčenky

overenie

```
security find-certificate -c "local.com" /Library/Keychains/System.keychain
```

vymazať DNS cache

```
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

skopirovanie do `nginx`, nastavenie oprávnení (plný prístup pre len pre vlastníka, nastavenie vlastníka ako `root`, skupiny ako `wheel`, ktorá je predvolená pre administračné účely, čím sa zabezpečí že adresár je riadený len administrátorom systému)

```
sudo mkdir -p /opt/homebrew/etc/nginx/certs

sudo chmod 700 /opt/homebrew/etc/nginx/certs
sudo chown root:wheel /opt/homebrew/etc/nginx/certs

sudo cp local.crt local.key /opt/homebrew/etc/nginx/certs
```

konfigurácia `nginx`

```
cp /opt/homebrew/etc/nginx/nginx.conf /opt/homebrew/etc/nginx/nginx.conf.backup

sudo nano /opt/homebrew/etc/nginx/nginx.conf
```

```
http {
    server {
        listen 443 ssl;
        server_name local.com local.sk;

        ssl_certificate /opt/homebrew/etc/nginx/certs/local.crt;
        ssl_certificate_key /opt/homebrew/etc/nginx/certs/local.key;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name local.com local.sk;

        return 301 https://$host$request_uri;
    }
}
```
