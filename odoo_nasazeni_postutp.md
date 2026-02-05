# Podrobný průvodce nasazením Odoo na DigitalOcean (Docker Compose)

Tento návod vás provede nasazením Odoo Community Edition a PostgreSQL na virtuální server (Droplet) u DigitalOcean pomocí Docker Compose. Tato metoda je robustní, snadno udržovatelná a vhodná pro produkční nasazení.

## Předpoklady
1.  **DigitalOcean účet**: Pokud nemáte, vytvořte si ho.
2.  **Vlastní doména**: Např. `moje-firma.cz`. Budete potřebovat přístup k DNS záznamům.

---

## 1. Vytvoření Dropletu (Serveru)

1.  V DigitalOcean panelu klikněte na **Create** -> **Droplets**.
2.  **Region**: Vyberte datacentrum nejblíže vašim uživatelům (např. Frankfurt).
3.  **Choose an Image**: Vyberte **Ubuntu 24.04 (LTS)**.
4.  **Size**:
    *   Pro testování: **Basic**, Regular Disk Type, 2GB RAM / 1 CPU.
    *   **Pro produkci (Doporučeno):** **Basic**, Premium Intel/AMD, **2GB RAM / 1 CPU**.
    *   *Tip: Protože 2GB RAM je pro Odoo "tak akorát", v dalším kroku nastavíme SWAP (odkládací paměť), aby server nespadl při špičce.*
5.  **Additional Options**: Zaškrtněte **Backups** (Daily backups). Je to malý příplatek za klidné spaní.
6.  **Authentication Method**: Zvolte **Password**.
    *   Vytvořte si **silné root heslo** (musí mít velké písmeno, číslo a nesmí končit číslem/znakem).
    *   *Pozor: Heslo si ihned uložte do správce hesel, DigitalOcean vám ho nepošle mailem.*
6.  **Hostname**: Pojmenujte server, např. `odoo-server`.
7.  Klikněte na **Create Droplet**.

## 2. Příprava a Zabezpečení Serveru

Připojte se k serveru přes SSH (nebo přes konzoli na webu DO):
```bash
ssh root@104.248.244.187
```

### Aktualizace systému
```bash
apt update && apt upgrade -y
```

### Nastavení SWAP (Důležité pro 2GB RAM!)
Aby server nespadl, když mu dojde RAM, vytvoříme 2GB odkládacího souboru na disku.
```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
```

### Nastavení Firewallu (UFW)
Povolíme pouze SSH (22), HTTP (80) a HTTPS (443).
```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```
*(Potvrďte `y` když se zeptá, zda pokračovat.)*

## 3. Instalace Docker a Docker Compose

Nainstalujeme nejnovější verzi Dockeru.

```bash
# Instalace prerekvizit
apt install apt-transport-https ca-certificates curl software-properties-common -y

# Přidání GPG klíče Dockeru
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Přidání repozitáře
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalace Dockeru
apt update
apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

# Ověření instalace
docker compose version
```

## 4. Nasazení Odoo pomocí Docker Compose

Vytvoříme složku pro projekt a konfigurační soubor.

```bash
mkdir -p /opt/odoo
cd /opt/odoo
```

Vytvořte soubor `docker-compose.yml`:
```bash
nano docker-compose.yml
```

Vložte následující obsah (upravte hesla!):

```yaml
services:
  web:
    image: odoo:17.0
    depends_on:
      - db
    ports:
      - "127.0.0.1:8069:8069" # Otevřeme jen pro localhost, ven půjde přes Nginx
    volumes:
      - odoo-web-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=moje_supertajne_heslo_db
    restart: always

  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=moje_supertajne_heslo_db
      - POSTGRES_USER=odoo
    volumes:
      - odoo-db-data:/var/lib/postgresql/data
    restart: always

volumes:
  odoo-web-data:
  odoo-db-data:
```

Uložte (`Ctrl+O`, `Enter`) a zavřete (`Ctrl+X`).

**Spuštění Odoo:**
```bash
docker compose up -d
```
Zkontrolujte, zda běží: `docker compose ps`.

## 5. Nastavení Domény a Nginx (HTTPS)

### Nastavení DNS
U vašeho registrátora domény nastavte **A záznam**:
*   Host: `odoo` (nebo `@` pro hlavní doménu)
*   Value: `104.248.244.187`
*   TTL: 3600

### Instalace Nginx a Certbot
```bash
apt install nginx certbot python3-certbot-nginx -y
```

### Konfigurace Nginx
Vytvořte konfiguraci pro vaši doménu:
```bash
nano /etc/nginx/sites-available/odoo
```

Vložte (nahraďte `odoo.scholartech.eu` vaší skutečnou doménou):

```nginx
upstream odoo {
    server 127.0.0.1:8069;
}

server {
    server_name odoo.scholartech.eu;

    access_log /var/log/nginx/odoo.access.log;
    error_log /var/log/nginx/odoo.error.log;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://odoo;
    }

    # Podpora pro dlouhé spojení (longpolling) - chat v Odoo
    location /websocket {
        proxy_pass http://odoo;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Zvýšení limitu pro upload souborů
    client_max_body_size 100M;
    gzip on;
}
```

Aktivujte konfiguraci:
```bash
ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

### Získání SSL certifikátu (HTTPS)
Spusťte Certbot, který automaticky upraví Nginx a nastaví HTTPS.
```bash
certbot --nginx -d odoo.scholartech.eu
```
Postupujte podle pokynů na obrazovce (zadejte email, potvrďte podmínky). Vyberte možnost přesměrování HTTP na HTTPS (Redirect).

## 6. Dokončení

Nyní otevřete v prohlížeči `https://odoo.scholartech.eu`. Měli byste vidět úvodní obrazovku Odoo pro vytvoření databáze.

**Důležité pro první spuštění:**
*   **Master Password**: Toto heslo si **dobře uložte**! Slouží pro správu databází (zálohování, mazání).
*   Vytvořte první databázi, zadejte email a heslo pro admin účet.

---

## 7. Zálohování a údržba

### Aktualizace Odoo
Pro aktualizaci na novější "patch" verzi (např. z 17.0.20231001 na 17.0.20231101):
```bash
cd /opt/odoo
docker compose pull
docker compose down
docker compose up -d
```

### Zálohování
1.  **Databáze**: Odoo má vestavěný správce záloh (`/web/database/manager`). Stahujte pravidelně `.zip` zálohu (včetně filestore).
2.  **Server**: V DigitalOcean zapněte "Backups" (stojí 20% ceny Dropletu navíc), což vám dá týdenní snapshoty celého serveru.