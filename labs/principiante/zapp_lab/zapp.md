---
title: "ZAPP Lab"
description: "Writeup técnico del laboratorio ZAPP."
author: "deox420"
date: "25-11-2025"
difficulty: "Principiante"
tags: ["CTF", "Linux", "FTP", "SSH", "Privilege Escalation"]
services: ["ftp", "ssh", "http"]
platform: "Debian"
environment: "The Hackers Labs / VirtualBox"
---

# ZAPP — Writeup Completo

## 🧭 Entorno y contexto
Laboratorio con servicios FTP anónimo, SSH y HTTP.  
Pistas codificadas llevan a credenciales para acceso inicial.

---

## 📝 Resumen ejecutivo
- **Acceso inicial:** usuario `zappskred` tras decodificar pistas y romper un archivo RAR.  
- **Escalada:** sudo sin contraseña.  
- **Flags:** ocultadas.

---

# 1. 🔍 Reconocimiento

```bash
nmap -p- -T4 -sV -sC 192.168.77.71
21/tcp ftp (anon allowed)
22/tcp ssh
80/tcp http Apache 2.4.65
```

---

# 2. 📡 Enumeración Web

`feroxbuster -u http://192.168.77.71/` revela un mensaje oculto en Base64 que tras 4 decodificaciones revela:

```
4444 VjFST1YyRkhVa2xUYmxwYVRURmFiMXBGYUV0a2JWSjBWbTF3WVZkRk1VeERaejA5Q2c9PQo=
```

Tras decodificar 4 veces →

```
cuatrocuatroveces
```

En la ruta aparece `Sup3rP4ss.rar`.

---

![/cuatrocuatroveces](zapp_1.png)

# 3. 🚪 FTP Anónimo

Se encuentran archivos con pistas (`secret.txt`, `login.txt`).

```
ftp 192.168.77.71
# usuario: anonymous
```

---

# 4. 💥 Cracking del RAR

```bash
wget http://192.168.77.71/cuatrocuatroveces/Sup3rP4ss.rar
rar2john Sup3rP4ss.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
```

Contraseña descubierta:

```
reema
```
```
unrar x Sup3rP4ss.rar
Sup3rP4ss.txt
```

Contenido extraído revela pista final: `3spuM4`

---

# 5. 💻 Acceso inicial

Credenciales:

```
usuario: zappskred
password: 3spuM4
```

![ssh](zapp_2.png)

---

# 6. ⬆️ Escalada de privilegios

```bash
sudo -i
```

Acceso a root inmediato.

![user/root flags](zapp_3.png)

---

# 7. 📁 Lectura de flags (OCULTADAS)

```
user.txt
[FLAG USER REDACTADA]

root.txt
[FLAG ROOT REDACTADA]
```

---

# 8. 🔧 Reproducibilidad — comandos clave

```bash
nmap ...
feroxbuster ...
ftp ...
rar2john ...
john ...
ssh zappskred@target
sudo -i
```

---

# 🧰 Herramientas utilizadas
Nmap, Feroxbuster, FTP, RAR2John, John, SSH.

---

# 🛡️ Mitigaciones recomendadas
Deshabilitar FTP anónimo, no dejar pistas, proteger sudo, usar contraseñas fuertes.

---

# 📝 Changelog
- Documento reconstruido desde cero con plantilla maestro.
- Flags ocultadas.
