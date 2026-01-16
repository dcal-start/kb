---
tags: [wifi, vpn, fortigate, forticlient, windows11, intel-wifi, ax211, dell, troubleshooting, driver, packet-loss]
created: 2026-01-16
updated: 2026-01-16
---

# Troubleshooting WiFi - Laptop Dell con Intel WiFi AX211

## Revisioni

| Versione | Data | Autore | Descrizione |
|----------|------|--------|-------------|
| 1.0 | 2026-01-16 | Dan | Creazione documento |

---

## Scenario

Utente con laptop Dell (2024+) con scheda WiFi **Intel Wi-Fi 6E AX211** presenta anomalie di rete, tra cui:

- Connessioni VPN (IPsec o SSL-VPN) che falliscono verso FortiGate tramite FortiClient VPN
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

## Note

- Le modifiche sopravvivono ai riavvii
- Dell SupportAssist e Windows Update possono ripristinare i valori default dopo aggiornamenti driver
- In caso di problemi persistenti: rollback driver a versione precedente
- Test con cavo ethernet conferma che il problema è isolato alla scheda WiFi
- **Applicare queste impostazioni preventivamente su tutti i laptop Dell con Intel AX211**

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
