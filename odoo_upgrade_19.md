# Upgrade Odoo na verzi 19 (Docker Deployment)

Tento návod popisuje proces aktualizace Odoo běžícího v Dockeru na novější verzi (např. z 17 na 18 nebo 19).

> [!WARNING]
> **Kritické upozornění pro Community Edition (verze zdarma)**
> Odoo Community **neumí** automaticky aktualizovat strukturu databáze při přechodu na vyšší verzi (např. z 17.0 na 18.0).
> Pokud pouze změníte verzi v Dockeru, **Odoo se nespustí** s chybou nekompatibility databáze.
>
> **Máte dvě možnosti:**
> 1.  **Začít na čisto:** Nová instalace, nová prázdná databáze (vhodné pro testování).
> 2.  **Migrace dat:** Použít nástroj OpenUpgrade (složitější) nebo export/import dat (CSV). Návod níže popisuje *technickou* změnu verze serveru.

---

## 1. Zálohování (Naprosto klíčové!)

Před jakoukoliv změnou musíte mít zálohu.

### A. Záloha databáze (Doporučeno)
1.  Jděte na `https://odoo.scholartech.eu/web/database/manager`.
2.  Klikněte na **Backup** u vaší databáze.
3.  Zadejte Master Password.
4.  Formát: `zip` (zahrnuje i soubory/filestore).
5.  Stáhněte soubor do počítače.

### B. Záloha celého serveru (DigitalOcean Snapshot)
Toto vás zachrání, pokud rozbijete konfiguraci serveru.
1.  V DigitalOcean panelu jděte na váš Droplet -> **Snapshots**.
2.  Pojmenujte snapshot (např. "pred-upgradem-na-19").
3.  Klikněte na **Take Live Snapshot**.

---

## 2. Technický postup upgrade (změna Docker Image)

Tento postup zajistí, že server bude používat novější kód Odoo.

### 1. Připojení k serveru
```bash
ssh root@vase_ip_adresa
cd /opt/odoo  # Nebo složka, kde máte docker-compose.yml
```

### 2. Úprava konfigurace
Otevřete `docker-compose.yml`:
```bash
nano docker-compose.yml
```

Najděte řádek s `image` v sekci `web` (nebo `odoo`) a změňte číslo verze:

```yaml
services:
  web:
    image: odoo:19.0  # Změna z 17.0 nebo 18.0
    # ... zbytek souboru beze změny
```

*(Doporučuji zkontrolovat, zda nová verze Odoo nevyžaduje i novější PostgreSQL. Pokud ano, můžete změnit i `image: postgres:16` na novější, ale to také vyžaduje migraci databáze!)*

Uložte (`Ctrl+O`) a zavřete (`Ctrl+X`).

### 3. Aplikace změn
Nyní musíme Dockeru říct, aby stáhl novou verzi a restartoval kontejnery.

```bash
# Stáhne novou verzi obrazu odoo:19.0
docker compose pull

# Zastaví staré kontejnery a spustí nové
# (Krátký výpadek služby - cca 1 minuta)
docker compose down
docker compose up -d
```

### 4. Kontrola stavu
Podívejte se, zda kontejnery běží:
```bash
docker compose ps
```

Podívejte se do logů, co se děje při startu:
```bash
docker compose logs -f web
```
*(Vyskočíte pomocí `Ctrl+C`)*

---

## 3. Co dělat po restartu?

### Scénář A: Enterprise Edition
Pokud máte licenci Enterprise, Odoo vám nabídne formulář pro nahrání databáze k migraci na jejich servery (nebo použijte `upgrade.odoo.com`).

### Scénář B: Community Edition (Chyba database compatibility)
V logu uvidíte chybu typu `Incompatible version`.
To je v pořádku - technický upgrade proběhl, ale data jsou stará.

**Řešení:**
1.  Jděte na `/web/database/manager`.
2.  Vytvořte **NOVOU** databázi (Create Database).
3.  Tato nová databáze bude verze 19 a bude fungovat.
4.  Data ze staré verze (z bodu 1) můžete zkusit importovat pomocí CSV exportů, ale je to ruční práce.
