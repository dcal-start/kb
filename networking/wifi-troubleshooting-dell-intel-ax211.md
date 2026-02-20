---
tags: [wifi, vpn, fortigate, forticlient, windows11, intel-wifi, ax211, dell, troubleshooting, driver, packet-loss, ipv6, dns, sticky-dns, dhcp, kerberos, smb, share, domain, gpo]
created: 2026-01-16
updated: 2026-02-20
---

# Troubleshooting WiFi - Laptop Dell con Intel WiFi AX211

## Revisioni

| Versione | Data | Autore | Descrizione |
|----------|------|--------|-------------|
| 1.0 | 2026-01-16 | Dan | Creazione documento |
| 1.1 | 2026-01-22 | Dan | Aggiunta sezione IPv6/DNS e procedure disabilitazione massiva |
| 1.2 | 2026-02-10 | Dan | Aggiunta sezione FortiClient Sticky DNS bug |
| 1.3 | 2026-02-20 | Dan | Aggiunta sezione accesso negato share via VPN |

---

## Scenario

Utente con laptop Dell (2024+) con scheda WiFi **Intel Wi-Fi 6E AX211** presenta anomalie di rete, tra cui:

- Connessioni VPN (IPsec o SSL-VPN) che falliscono verso FortiGate tramite FortiClient VPN
- DNS che non risolve nomi interni (domain controller, file server) mentre la VPN è connessa
- Perdita di pacchetti anomala durante semplici ping
- Disconnessioni WiFi intermittenti
- Latenza elevata o instabile
- Qualsiasi altro comportamento anomalo della connessione WiFi

**Nota:** Questo intervento è consigliato come primo step diagnostico ogni volta che un utente con laptop Dell recente segnala problemi WiFi.

## Sintomi

- VPN fallisce sia in modalità IPsec che SSL-VPN
- Ping con packet loss intermittente
- Navigazione web lenta o a singhiozzo
- Connessione WiFi che cade e si riconnette
- Altri utenti sulla stessa rete non hanno problemi

---

## VERIFICA IMMEDIATA: IPv6

**Controllare SEMPRE per primo se IPv6 è attivo sulla scheda Wi-Fi.** Questo check richiede pochi secondi e può risolvere immediatamente il problema.

### Perché IPv6 causa problemi con le VPN

In Italia, IPv6 è raramente necessario e può causare gravi problemi con le VPN aziendali:

- Windows fa query DNS sia IPv4 (A) che IPv6 (AAAA) **in parallelo**
- Se il router di casa/ufficio assegna un DNS IPv6 via DHCPv6/SLAAC, Windows lo usa
- Il DNS IPv6 del router non conosce i nomi del dominio aziendale → risponde **NXDOMAIN**
- Questa risposta arriva PRIMA di quella del DNS aziendale via VPN → Windows considera il nome inesistente
- Risultato: `nslookup` funziona, ma `ping` e applicazioni falliscono

### Sintomi tipici problema IPv6

- `nslookup dc01.dominio.local` → **funziona** (usa il primo DNS configurato)
- `Resolve-DnsName dc01.dominio.local` → **funziona**
- `ping dc01.dominio.local` → **"Impossibile trovare l'host"** (usa tutti i DNS in parallelo)
- Mappatura dischi di rete fallisce
- GPO non si applicano
- Folder redirection non funziona

### Diagnosi rapida IPv6

```powershell
# Check DNS IPv6 su interfacce attive
Get-DnsClientServerAddress -AddressFamily IPv6 | Where-Object {$_.ServerAddresses} | Format-Table InterfaceAlias, ServerAddresses

# Se vedi indirizzi DNS IPv6 su Wi-Fi (tipo 2001:xxx o fe80:xxx) → PROBLEMA!
```

### Diagnosi approfondita con ETW DNS Client Logging

Questo è lo strumento che ha permesso di identificare il problema dopo numerosi tentativi con altri tool. **Packet Capture con pktmon** è stato provato ma si è rivelato troppo dispersivo; **ETW DNS Client Logging** è stato risolutivo.

```powershell
# ============================================================
# ETW DNS Client Logging - Diagnostica DNS Windows
# ============================================================

# Abilita logging
wevtutil set-log Microsoft-Windows-DNS-Client/Operational /enabled:true

# Pulisci log precedenti
wevtutil cl Microsoft-Windows-DNS-Client/Operational

# Pulisci cache DNS
ipconfig /flushdns

# Genera il problema (es. ping che fallisce)
ping dc01.dominio.local

# Leggi IMMEDIATAMENTE il log
Get-WinEvent -LogName "Microsoft-Windows-DNS-Client/Operational" -MaxEvents 30 | Format-List TimeCreated, Id, Message

# Filtra per nome specifico
Get-WinEvent -LogName "Microsoft-Windows-DNS-Client/Operational" -MaxEvents 50 | Where-Object {$_.Message -like "*dc01*"} | Format-List TimeCreated, Id, Message
```

**Codici evento importanti:**

| ID | Descrizione |
|----|-------------|
| 3006 | Query DNS chiamata (inizio richiesta) |
| 3008 | Query DNS completata (fine richiesta con stato) |
| 3009 | Query di rete avviata - **mostra interfaccia e DNS server usati** |
| 3010 | Query DNS inviata al server - **mostra QUALE server riceve la query** |
| 3011 | Risposta ricevuta dal server |
| 3016 | Ricerca nella cache chiamata |
| 3018 | Ricerca nella cache completata |
| 3019 | Chiamata wire di query |
| 3020 | Risposta alla query |
| 1016 | **ERRORE "nome non trovato"** - mostra quale server ha risposto NXDOMAIN |

**Cosa cercare nei log:**

| Valore | Significato |
|--------|-------------|
| tipo 1 | Query A (IPv4) |
| tipo 28 | Query AAAA (IPv6) ← se va a un DNS sbagliato, questo è il problema |
| stato 0 | Successo |
| stato 9003 | NXDOMAIN (nome non trovato) |
| stato 9701 | Cache miss |
| stato 87 | ERROR_INVALID_PARAMETER |
| stato 1223 | ERROR_CANCELLED |

**Red flag nel log:** Se vedi query AAAA (tipo 28) inviate a un DNS IPv6 pubblico/locale (non aziendale) con stato 9003 (NXDOMAIN), hai trovato il problema.

### Soluzione: Disabilitare IPv6 sulla Wi-Fi

```powershell
# Disabilita IPv6 sulla scheda Wi-Fi
Disable-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip6

# Verifica
Get-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip6

# Flush cache e test
ipconfig /flushdns
ping dc01.dominio.local
```

---

## VERIFICA: FortiClient Sticky DNS (DNS statici residui dopo VPN)

Quando FortiClient si connette alla VPN, il FortiGate pusha i DNS aziendali e FortiClient li scrive nella configurazione della NIC attiva (di solito Wi-Fi). Alla disconnessione, dovrebbe ripristinarli a DHCP, ma **in alcuni casi non lo fa** (bug noto Fortinet, [articolo 279430](https://community.fortinet.com/t5/FortiGate/Technical-Tip-FortiClient-Sticky-DNS/ta-p/279430)).

I DNS aziendali restano "martellati" nella NIC → fuori dalla VPN non si naviga più e la VPN successiva non si connette.

**Trigger comuni:** spegnimento PC senza disconnettere la VPN, crash di FortiClient, cambio rete Wi-Fi durante la VPN, sospensione/ibernazione con VPN attiva.

### Diagnosi

```powershell
# Se NameServer contiene IP (es. 10.1.1.11,10.x.x.x) e EnableDHCP = 1 → DNS forzati da FortiClient
$if = Get-NetAdapter -Name "Wi-Fi"
$cfg = Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object {$_.InterfaceIndex -eq $if.ifIndex}
$guid = $cfg.SettingID.Trim('{}')
$rk = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{$guid}"
Get-ItemProperty $rk | Select-Object NameServer, DhcpNameServer, EnableDHCP
```

### Soluzione utente (senza admin)

"Dimenticare" la rete Wi-Fi e riconnettersi. Questo forza Windows a ricreare il profilo di rete da zero, eliminando i DNS residui.

1. **Impostazioni → Rete e Internet → Wi-Fi → Gestisci reti note**
2. Selezionare la rete problematica → **Dimentica**
3. Riconnettersi inserendo la password

Oppure da riga di comando (non richiede admin):

```cmd
netsh wlan delete profile name="Nome-SSID"
```

### Soluzione admin

```powershell
netsh interface ip set dns "Wi-Fi" dhcp
ipconfig /flushdns
```

### Prevenzione

Istruire gli utenti a **disconnettere sempre la VPN** prima di spegnere il PC o cambiare rete Wi-Fi.

---

## Accesso negato alle share con VPN attiva ma connettività funzionante

Laptop domain-joined (Windows 11) collegato tramite VPN. La connettività verso il Domain Controller è corretta, ma l'utente riceve **"Accesso negato"** aprendo cartelle condivise o unità mappate in Esplora File. Paradossalmente, `dir \\server\share` da PowerShell funziona.

**Causa:** sessione Kerberos incoerente. L'utente ha fatto logon con credenziali cached (offline), poi ha connesso la VPN → i ticket Kerberos e la sessione SMB non sono allineati con le policy/gruppi AD correnti.

### Procedura standard di accesso al dominio via VPN

Questa è la procedura corretta da seguire **ogni volta** che ci si collega al dominio aziendale tramite VPN:

1. Dopo il login al computer, avviare e collegare la VPN
2. Dopo che la VPN si è connessa, premere **CTRL+ALT+CANC** e **bloccare il computer**
3. Riaccedere con le proprie credenziali, forzando il dialogo di autenticazione tra client e domain controller
4. Attendere qualche istante e verificare se le mappature di rete e i percorsi UNC sono disponibili
5. Se non lo sono, eseguire `gpupdate` e verificare nuovamente

### Se la procedura standard non basta

Se dopo i passi precedenti le share risultano ancora inaccessibili con errore "Accesso negato", e la diagnostica conferma che tutto è regolare:

```powershell
ping dc01                              # OK
Resolve-DnsName dc01.dominio.local     # OK
Test-ComputerSecureChannel             # True
klist                                  # TGT e ticket CIFS presenti e validi
Get-SmbConnection                      # (come admin) connessioni visibili
dir \\dc01\share                       # funziona da PowerShell
# Ma Esplora File → "Accesso negato"
```

Eseguire il purge dei ticket Kerberos e rieseguire le GPO:

```powershell
klist purge
gpupdate
```

Attendere qualche istante e verificare nuovamente l'accesso alle share da Esplora File.

---

## Causa

Driver Intel WiFi AX211 (versioni 23.x) ha funzionalità di risparmio energetico e ottimizzazione pacchetti che interferiscono con il traffico di rete. Queste funzionalità aggregano o ritardano pacchetti per risparmiare energia, causando problemi con:

- Tunnel VPN (richiedono timing preciso)
- ICMP/ping (pacchetti ritardati o scartati)
- Applicazioni real-time (VoIP, video)

**Evidenza nel log eventi:**
```
Intel(R) Wi-Fi 6E AX211 160MHz: la scheda di rete ha restituito un valore non valido al driver.
```

---

## Diagnosi

### Verifica errori nel log eventi

```powershell
Get-WinEvent -LogName System -MaxEvents 50 | Where-Object {$_.ProviderName -match "WLAN|Netw"} | Format-Table TimeCreated, Message -Wrap
```

### Verifica driver WiFi

```powershell
Get-NetAdapter -Name "Wi-Fi" | Format-List DriverVersion, DriverDate, DriverProvider
```

### Verifica impostazioni avanzate scheda

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi" | Format-Table Name, DisplayName, DisplayValue
```

### Verifica MTU e stato interfacce

```powershell
netsh interface ipv4 show subinterfaces
```

### Verifica offload

```powershell
netsh int ip show offload
```

### Test packet loss

```powershell
ping -n 100 8.8.8.8
```

---

## Soluzione

### Disabilita offload e coalescing

```powershell
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" -DisplayName "Packet Coalescing" -DisplayValue "Disabled"
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" -DisplayName "ARP offload for WoWLAN" -DisplayValue "Disabled"
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" -DisplayName "NS offload for WoWLAN" -DisplayValue "Disabled"
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" -DisplayName "MIMO Power Save Mode" -DisplayValue "No SMPS"
```

### Riavvia la scheda

```powershell
Restart-NetAdapter -Name "Wi-Fi"
```

---

## Riepilogo Impostazioni

| Impostazione | Default | Valore Corretto |
|--------------|---------|-----------------|
| Packet Coalescing | Enabled | **Disabled** |
| ARP offload for WoWLAN | Enabled | **Disabled** |
| NS offload for WoWLAN | Enabled | **Disabled** |
| MIMO Power Save Mode | Auto SMPS | **No SMPS** |

---

## Disabilitazione Massiva IPv6

### Metodo 1: GPO per PC in dominio Active Directory

Non esiste una GPO nativa per disabilitare IPv6. Usare una delle seguenti alternative:

#### Opzione A: GPO con Registry Preference

1. Aprire **Group Policy Management Console**
2. Creare una nuova GPO o modificare una esistente
3. Navigare a: `Computer Configuration → Preferences → Windows Settings → Registry`
4. Creare nuova voce Registry:

| Campo | Valore |
|-------|--------|
| Action | Update |
| Hive | HKEY_LOCAL_MACHINE |
| Key Path | SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters |
| Value name | DisabledComponents |
| Value type | REG_DWORD |
| Value data | 0xFF (255 decimale) |

**Valori DisabledComponents:**

| Valore | Effetto |
|--------|---------|
| 0xFF (255) | Disabilita IPv6 completamente su tutte le interfacce |
| 0x20 (32) | Preferisce IPv4 su IPv6 (meno aggressivo) |
| 0x10 (16) | Disabilita IPv6 su interfacce non-tunnel |
| 0x01 (1) | Disabilita IPv6 su tutte le interfacce tunnel |

**Nota:** Richiede riavvio del PC.

#### Opzione B: GPO con PowerShell Startup Script

1. Creare script `Disable-IPv6-WiFi.ps1`:

```powershell
# Disable-IPv6-WiFi.ps1
# Disabilita IPv6 solo sulla scheda Wi-Fi

$wifiAdapter = Get-NetAdapter | Where-Object {$_.InterfaceDescription -like "*Wi-Fi*" -or $_.InterfaceDescription -like "*Wireless*"}

if ($wifiAdapter) {
    foreach ($adapter in $wifiAdapter) {
        $binding = Get-NetAdapterBinding -Name $adapter.Name -ComponentID ms_tcpip6 -ErrorAction SilentlyContinue
        if ($binding -and $binding.Enabled) {
            Disable-NetAdapterBinding -Name $adapter.Name -ComponentID ms_tcpip6
            Write-Output "$(Get-Date) - IPv6 disabilitato su $($adapter.Name)" | Out-File -Append "C:\Windows\Logs\IPv6-Disable.log"
        }
    }
}
```

2. Configurare GPO:
   - `Computer Configuration → Policies → Windows Settings → Scripts → Startup`
   - Aggiungere lo script PowerShell

### Metodo 2: Script Atera per PC gestiti via RMM

#### Script Atera - Disabilita IPv6 su Wi-Fi

**Nome script:** `Disable-IPv6-WiFi`
**Tipo:** PowerShell
**Esecuzione:** Come Sistema

```powershell
<#
.SYNOPSIS
    Disabilita IPv6 sulla scheda Wi-Fi per evitare problemi DNS con VPN
.DESCRIPTION
    - Identifica tutte le schede Wi-Fi
    - Disabilita il binding IPv6
    - Opzionalmente configura anche Packet Coalescing per Intel AX211
    - Scrive log in C:\ProgramData\Atera\Logs
.NOTES
    Autore: Dan
    Versione: 1.0
    Data: 2026-01-22
#>

# Configurazione
$LogPath = "C:\ProgramData\Atera\Logs"
$LogFile = Join-Path $LogPath "IPv6-WiFi-Disable.log"
$ApplyIntelFixes = $true  # Imposta $false per saltare le fix Intel

# Crea cartella log se non esiste
if (-not (Test-Path $LogPath)) {
    New-Item -ItemType Directory -Path $LogPath -Force | Out-Null
}

function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - $Message" | Out-File -Append $LogFile
    Write-Output $Message
}

Write-Log "=== Inizio script Disable-IPv6-WiFi ==="
Write-Log "Computer: $env:COMPUTERNAME"

# Trova schede Wi-Fi
$wifiAdapters = Get-NetAdapter | Where-Object {
    $_.InterfaceDescription -like "*Wi-Fi*" -or
    $_.InterfaceDescription -like "*Wireless*" -or
    $_.InterfaceDescription -like "*WLAN*"
}

if (-not $wifiAdapters) {
    Write-Log "Nessuna scheda Wi-Fi trovata"
    exit 0
}

foreach ($adapter in $wifiAdapters) {
    Write-Log "Elaborazione: $($adapter.Name) - $($adapter.InterfaceDescription)"

    # Disabilita IPv6
    try {
        $ipv6Binding = Get-NetAdapterBinding -Name $adapter.Name -ComponentID ms_tcpip6 -ErrorAction Stop
        if ($ipv6Binding.Enabled) {
            Disable-NetAdapterBinding -Name $adapter.Name -ComponentID ms_tcpip6
            Write-Log "  IPv6 DISABILITATO"
        } else {
            Write-Log "  IPv6 già disabilitato"
        }
    } catch {
        Write-Log "  ERRORE disabilitazione IPv6: $_"
    }

    # Fix Intel AX211 se richiesto
    if ($ApplyIntelFixes -and $adapter.InterfaceDescription -like "*Intel*Wi-Fi*") {
        Write-Log "  Rilevata scheda Intel - applicazione fix aggiuntive"

        $intelSettings = @{
            "Packet Coalescing" = "Disabled"
            "ARP offload for WoWLAN" = "Disabled"
            "NS offload for WoWLAN" = "Disabled"
            "MIMO Power Save Mode" = "No SMPS"
        }

        foreach ($setting in $intelSettings.GetEnumerator()) {
            try {
                $currentValue = Get-NetAdapterAdvancedProperty -Name $adapter.Name -DisplayName $setting.Key -ErrorAction SilentlyContinue
                if ($currentValue -and $currentValue.DisplayValue -ne $setting.Value) {
                    Set-NetAdapterAdvancedProperty -Name $adapter.Name -DisplayName $setting.Key -DisplayValue $setting.Value
                    Write-Log "    $($setting.Key): $($currentValue.DisplayValue) -> $($setting.Value)"
                } elseif ($currentValue) {
                    Write-Log "    $($setting.Key): già configurato correttamente"
                }
            } catch {
                Write-Log "    ERRORE $($setting.Key): $_"
            }
        }
    }
}

Write-Log "=== Fine script ==="
exit 0
```

#### Script Atera - Verifica stato IPv6 (Audit)

**Nome script:** `Audit-IPv6-Status`
**Tipo:** PowerShell
**Esecuzione:** Come Sistema

```powershell
<#
.SYNOPSIS
    Verifica lo stato IPv6 su tutte le schede di rete
.DESCRIPTION
    Genera un report dello stato IPv6 per audit
#>

Write-Output "=== Audit IPv6 - $env:COMPUTERNAME ==="
Write-Output "Data: $(Get-Date)"
Write-Output ""

# Stato generale IPv6
$ipv6Disabled = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" -Name "DisabledComponents" -ErrorAction SilentlyContinue
if ($ipv6Disabled) {
    Write-Output "Registry DisabledComponents: $($ipv6Disabled.DisabledComponents) (0xFF=completamente disabilitato)"
} else {
    Write-Output "Registry DisabledComponents: Non configurato (IPv6 abilitato)"
}
Write-Output ""

# Stato per interfaccia
Write-Output "=== Stato binding IPv6 per interfaccia ==="
Get-NetAdapter | ForEach-Object {
    $binding = Get-NetAdapterBinding -Name $_.Name -ComponentID ms_tcpip6 -ErrorAction SilentlyContinue
    $status = if ($binding.Enabled) { "ABILITATO" } else { "Disabilitato" }
    Write-Output "$($_.Name): $status ($($_.InterfaceDescription))"
}
Write-Output ""

# DNS IPv6 configurati
Write-Output "=== Server DNS IPv6 configurati ==="
$ipv6Dns = Get-DnsClientServerAddress -AddressFamily IPv6 | Where-Object {$_.ServerAddresses}
if ($ipv6Dns) {
    $ipv6Dns | Format-Table InterfaceAlias, ServerAddresses -AutoSize
} else {
    Write-Output "Nessun DNS IPv6 configurato"
}
```

---

## Note

### Persistenza delle modifiche

- Disabilitazione IPv6 via `Disable-NetAdapterBinding`: sopravvive ai riavvii
- Disabilitazione IPv6 via Registry: sopravvive ai riavvii
- Modifiche Packet Coalescing: sopravvivono ai riavvii

### Attenzione

- **Dell SupportAssist** e **Windows Update** possono ripristinare driver e impostazioni default dopo aggiornamenti
- Schedulare verifiche periodiche tramite Atera o task schedulato
- In caso di problemi persistenti: rollback driver a versione precedente
- Test con cavo ethernet conferma che il problema è isolato alla scheda WiFi

### Raccomandazioni

1. **Applicare preventivamente** queste impostazioni su tutti i laptop Dell con Intel AX211
2. **Disabilitare IPv6** di default sui PC aziendali in Italia (raramente necessario)
3. **Documentare** le eccezioni dove IPv6 è richiesto
4. **ETW DNS Client Logging** è lo strumento più efficace per diagnosticare problemi DNS complessi

---

## Ambiente di Riferimento

| Componente | Versione |
|------------|----------|
| Sistema Operativo | Windows 11 Pro |
| Scheda WiFi | Intel Wi-Fi 6E AX211 160MHz |
| Driver problematico | 23.x (es. 23.160.0.4) |
| VPN Client | FortiClient VPN 7.4.x |
| Firewall | FortiGate (FortiOS 7.4.x) |
| Protocolli VPN | IPsec, SSL-VPN |
