# WEBCON BPS Docker Desktop – kontekst dla AI

## Co to jest

Projekt uruchamia lokalną instancję WEBCON BPS 2026 w Docker Desktop (Windows).
Składa się z dwóch plików compose:
- `linux-services.yml` – kontenery Linux: SQL Server, Solr (search), Caddy (reverse proxy)
- `windows-services.yml` – kontenery Windows: portal i workflow service

Startuje się przez `start.ps1` (PowerShell), który:
1. Uruchamia Linux containers (`docker compose -f linux-services.yml up -d`)
2. Czeka aż SQL Server będzie gotowy
3. Przełącza Docker na Windows containers
4. Uruchamia Windows containers (`docker compose -f windows-services.yml up -d`)
5. Importuje certyfikaty Caddy i czeka na HTTP 200 z portalu

## Stan na dzień migracji (2026-03-12)

### Co zostało zmienione względem poprzedniej wersji (2023.1.2.123)

#### linux-services.yml
- `bps-search`: obraz zmieniony z `webconbps/search:2023.1.2.123` na `webconbps/search:2026.1.4.131`

#### windows-services.yml – DUŻA ZMIANA ARCHITEKTURY
W WEBCON BPS 2026 obrazy `webconbps/init`, `webconbps/service` i `webconbps/portal`
**zostały wycofane** – ostatnie wersje:
- `webconbps/init` → tylko do 2023.1.2.121
- `webconbps/service` → tylko do 2025.2.x
- `webconbps/portal` → tylko do 2025.2.x

Zastąpił je **jeden obraz**: `webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022`
Obraz ten uruchamia się w dwóch trybach przez zmienną `STARTUP_MODE`:
- `STARTUP_MODE=service` → zastępuje stary bps-service
- `STARTUP_MODE=portal` → zastępuje stary bps-portal + bps-init (init jest teraz wbudowany)

Szczegóły obrazu bootstrapper:
- .NET 9 / ASP.NET Core 9
- Port wewnętrzny: **8080** (było 80 w starym portalu) – ustawione przez `ASPNETCORE_HTTP_PORTS=8080`
- Entrypoint: `powershell -ExecutionPolicy RemoteSigned -File C:\bps\Entrypoint.ps1`
- Workdir: `C:\bps`

### appsettings.json wewnątrz obrazu (C:\bps\appsettings.json)

```json
{
  "ConnectionStrings": {
    "ConfigDb": "Server=;Database=;",
    "LogsDb": "Server=;Database=;"
  },
  "Serilog": {
    "WriteTo": {
      "MSSqlServerSink": {
        "Name": "MSSqlServer",
        "Args": {
          "connectionString": "LogsDb",
          "tableName": "Logs",
          "schemaName": "infrastructure",
          "autoCreateSqlTable": true
        }
      }
    }
  }
}
```

**KLUCZOWE**: Serilog czyta connection string z klucza `ConnectionStrings:LogsDb`.
W ASP.NET Core ten klucz nadpisuje się zmienną środowiskową `ConnectionStrings__LogsDb`.
Bez tej zmiennej Serilog próbuje połączyć się z pustym serwerem → Named Pipes błąd → crash.

## Aktualny windows-services.yml

```yaml
version: '3.9'
services:
  bps-service:
    image: webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022
    container_name: bps-service
    hostname: bps-service
    environment:
      - STARTUP_MODE=service
      - ConnectionStrings__ConfigDb=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - ConnectionStrings__LogsDb=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - Configuration__BpsDbConfigRaw=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - Configuration__ExternalWebService__Host=bps-service
      - Configuration__ExternalWebService__LicenseServicePort=8002
      - Configuration__ExternalWebService__Port=8003
      - Configuration__BpsSelfHost=true
      - Configuration__Init__DoInit=true
      - Configuration__Init__WebService__LicenseServicePort=8002
      - Configuration__Init__WebService__Port=8003
      - Configuration__Init__ServiceRoles__LicenseService=true
      - Configuration__Init__ServiceRoles__BasicFeatures=true
      - Configuration__Init__ServiceRoles__SolrIndexing=true
    ports:
      - "8002:8002"
      - "8003:8003"
    restart: unless-stopped
    networks:
      - bps

  bps-portal:
    image: webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022
    container_name: bps-portal
    hostname: bps-portal
    environment:
      - STARTUP_MODE=portal
      - ConnectionStrings__ConfigDb=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - ConnectionStrings__LogsDb=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - App__ConfigConnection__Value=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - App__LogsConnection__Value=Server=tcp:10.5.0.1,8433;Database=BPS_Config;User ID=sa;Password=P@ssw0rd;TrustServerCertificate=True
      - App__IISIntegration=false
      - App__ForceHttpsOnProxy=true
      - App__Kestler__Port=8080
      - App__Kestler__UseSSL=false
      - App__Kestler__Protocol=http
      - App__LogLevel__Value=Warn
    ports:
      - "8080:8080"
    restart: unless-stopped
    depends_on:
      - bps-service
    networks:
      - bps

networks:
  bps:
    driver: nat
    ipam:
      config:
        - subnet: 10.5.0.0/16
          gateway: 10.5.0.1
```

## Architektura sieciowa

- Windows containers używają sieci `nat` (driver: nat) z gateway `10.5.0.1`
- `10.5.0.1` to gateway nat = adres hosta Windows widziany z Windows kontenera
- Linux containers mają porty opublikowane na hoście: SQL=8433, Solr=8983
- Windows kontenery łączą się z SQL przez `10.5.0.1,8433`
- Caddy (Linux) łączy się z portalem przez `host.docker.internal:8080`

## Diagnostyka – co sprawdzić

### 1. Czy SQL jest osiągalny z Windows kontenera?
```powershell
# Na hoście Windows – sprawdzenie czy SQL jest widoczny
Test-NetConnection -ComputerName 10.5.0.1 -Port 8433

# Lub spróbuj exec do działającego kontenera (jeśli nie pada od razu):
docker exec bps-service powershell -Command "Test-NetConnection -ComputerName 10.5.0.1 -Port 8433"
```

### 2. Odczyt appsettings z obrazu (bez uruchamiania)
```powershell
docker create --name bps-temp webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022 cmd
docker cp bps-temp:C:\bps\appsettings.json .\appsettings.json
docker rm bps-temp
type appsettings.json
```

### 3. Sprawdzenie wszystkich plików konfiguracyjnych w obrazie
```powershell
docker create --name bps-temp webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022 cmd
docker cp bps-temp:C:\bps\ .\bps-contents\
docker rm bps-temp
dir bps-contents\
```

### 4. Logi kontenerów
```powershell
docker logs bps-service --tail 50
docker logs bps-portal --tail 50
```

### 5. Sprawdzenie zmiennych środowiskowych wewnątrz kontenera
```powershell
docker exec bps-service powershell -Command "Get-ChildItem Env: | Sort-Object Name"
```

## Znane problemy i ich przyczyny

| Problem | Przyczyna | Fix |
|---------|-----------|-----|
| `No startup mode selected` | Brak `STARTUP_MODE` env var | Dodać `STARTUP_MODE=portal` lub `STARTUP_MODE=service` |
| `Named Pipes Provider, error: 40` przy starcie | Serilog czyta `ConnectionStrings:LogsDb` z appsettings.json (domyślnie puste) | Dodać `ConnectionStrings__LogsDb=Server=tcp:...` |
| `${hostname}` puste w compose | `start.ps1` ustawia `$env:hostname` ale jeśli nie uruchamiasz przez start.ps1, zmienna jest pusta | Użyj bezpośrednio `10.5.0.1` zamiast `${hostname}` |

## Potencjalnie brakujące zmienne środowiskowe

Są to zmienne których **NIE znamy** a mogą być potrzebne (sprawdź Entrypoint.ps1):
- Klucze dla Serilog ConfigDb – może bootstrap potrzebuje oddzielnej bazy dla konfiguracji
- Konfiguracja Solr/Search URL – w nowej wersji może być konfigurowana przez env var
- Klucze licencyjne – WEBCON 2026 zmienił model licencjonowania

## Jak sprawdzić Entrypoint.ps1
```powershell
docker create --name bps-temp webconbps/bootstrapper:2026.1.4.131-windowsservercore-ltsc2022 cmd
docker cp bps-temp:C:\bps\Entrypoint.ps1 .\Entrypoint.ps1
docker rm bps-temp
type Entrypoint.ps1
```
Ten skrypt pokaże dokładnie jakich zmiennych środowiskowych oczekuje obraz.

## Repozytorium

- Fork z konfiguracją: https://github.com/adideusz/bps-for-docker-desktop
- Oryginalne repo (stara wersja): https://github.com/WEBCON-BPS/bps-for-docker-desktop
