---
title: "ZAPP Lab"
description: "Writeup y documentación técnica completa y extendida del laboratorio ZAPP."
author: "deox420"
date: "25-11-2025"
difficulty: "Principiante"
tags: ["CTF", "Linux", "FTP", "SSH", "Web Enumeration", "Privilege Escalation", "Forensics"]
services: ["ftp", "ssh", "http"]
platform: "Debian"
environment: "The Hackers Labs / VirtualBox"
source: "zapp.md"
---

# ZAPP — Writeup Completo (Versión extendida y pedagógica)

> Nota: este documento reconstruye y amplía el análisis original del laboratorio ZAPP para convertirlo en un recurso didáctico. He mantenido la evidencia técnica presente en el material fuente y la he complementado con explicaciones, hipótesis y razonamientos del analista. Las flags y credenciales sensibles aparecen **ocultadas** para publicación segura.

## 🧭 Entorno y contexto
Máquina objetivo: **192.168.77.71** (Debian).  
Servicios relevantes detectados: **FTP anónimo (21)**, **HTTP (80)** y **SSH (22)**.  
Escenario: pista oculta en la web que conduce a un directorio con un archivo RAR protegido; cracking del RAR proporciona credenciales que permiten acceso SSH; el usuario resultante tiene permisos `sudo` sin contraseña, lo que facilita una escalada inmediata a root.

Contextualmente, este laboratorio está diseñado para practicar:  
- Enumeración web y descubrimiento de contenido oculto.  
- Trabajo con FTP anónimo para recuperar pistas.  
- Manejo de archivos comprimidos protegidos y cracking de contraseñas.  
- Análisis de configuraciones inseguras de `sudo`.

---

## 📝 Resumen ejecutivo (versión extendida)
- **Objetivo:** obtener `user.txt` y `root.txt`.  
- **Vector de entrada:** enumeración web → mensaje Base64 escondido → directorio secreto → `Sup3rP4ss.rar`.  
- **Método:** extracción del RAR, conversión a hash con `rar2john`, cracking con `john` usando `rockyou.txt`; las credenciales reveladas permiten acceso SSH como `zappskred`.  
- **Escalada:** privilegios de root mediante `sudo -i` sin pedir contraseña.  
- **Impacto:** compromiso total de la máquina; exposición de credenciales y de información sensible en áreas pública/FTP.  
- **Estado para publicación:** flags ocultadas.

---

# 1) 🔍 Reconocimiento — superficie de ataque (detalle y pedagogía)

### 1.1. Comando y objetivo
Ejecuté un escaneo de puertos/services para identificar vectores activos:

```bash
nmap -p- -T4 -sV -sC 192.168.77.71
```

**Salida relevante (extracto):**
```text
21/tcp open ftp (anon allowed)
22/tcp open ssh
80/tcp open http Apache 2.4.65 (Debian)
```

### 1.2. Análisis de la superficie
- **FTP anónimo**: indica posibilidad inmediata de obtener archivos o pistas depositadas por administradores (intencionadas) o por usuarios (por error). Los labs suelen usar FTP anónimo para colocar hints; en producción, esto es un riesgo.
- **HTTP con contenido estático/dinámico**: la presencia de `zapp.jpg` y contenido HTML que incluye datos ocultos implica que el atacante debe inspeccionar el código fuente y buscar elementos no visibles al usuario (comentarios HTML, `display:none`, `data-*`, etc.).
- **SSH**: vector objetivo para obtener shell interactiva. Debe ser protegido con credenciales robustas y sin accesos directos como `sudo NOPASSWD`.

**Hipótesis inicial del atacante/analista:** comenzar por la web (fácil, no requiere credenciales) y combinando datos del FTP obtener la ruta o la confirmación de pistas.

---

# 2) 📡 Enumeración / descubrimiento (paso a paso razonado)

### 2.1. Enumeración web profunda
Herramienta usada:

```bash
feroxbuster -u http://192.168.77.71/
```

**Observación clave:** se encontró un `<div style="display:none">` que contenía una cadena Base64 prefijada por `4444`. La práctica de ocultar datos en elementos no visibles es común en CTFs para enseñar técnicas de **steganografía ligera** y descubrimiento de contenido.

**Cadena encontrada:**
```
4444 VjFST1YyRkhVa2xUYmxwYVRURmFiMXBGYUV0a2JWSjBWbTF3WVZkRk1VeERaejA5Q2c9PQo=
```

**Decodificación:** aplicar Base64 repetidamente (4 veces en este caso) produjo la palabra `cuatrocuatroveces`, que funciona como nombre de directorio.  
Este patrón enseña dos lecciones:
1. Comprueba contenido escondido en HTML además de directorios.
2. Si encuentras datos codificados, prueba decodificaciones iterativas (a veces se encadenan).

### 2.2. Exploración del directorio secreto
Ruta detectada:

```
http://192.168.77.71/cuatrocuatroveces/
```

Archivo relevante:

```
Sup3rP4ss.rar
```

![/cuatrocuatroveces](zapp_1.png)

**Interpretación:** el autor del lab quería combinar técnicas: descubrir la ruta (enum web) y resolver un reto criptográfico/práctico (crack RAR) para obtener credenciales.

### 2.3. Correlación con FTP
Accediendo al FTP anónimo (`ftp 192.168.77.71`) se localizaron `login.txt` y `secret.txt`. El contenido de estos archivos confirma la necesidad de la decodificación múltiple y aporta pistas que reducen el tiempo de búsqueda.  
**Correlación práctica:** siempre correlacionar hallazgos web con recursos accesibles por FTP/SMB/etc.; a menudo los labs distribuyen pistas entre servicios.

---

# 3) 🚪 Ataque inicial — descarga y preparación del cracking

### 3.1. Descarga del recurso
```bash
wget http://192.168.77.71/cuatrocuatroveces/Sup3rP4ss.rar
```

### 3.2. Preparación para cracking
Transformamos el RAR a formato crackeable:

```bash
rar2john Sup3rP4ss.rar > rar.hash
```

**Nota pedagógica:** `rar2john` extrae el hash compatible con John; siempre valida la versión de `rar` y si es RAR5 (el mecanismo de hash cambia). En este laboratorio la herramienta y la wordlist fueron suficientes.

### 3.3. Ejecución del cracking
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
```

**Resultado:** password `reema` (extraído por John).  
**Análisis del resultado:** la contraseña es corta/estándar, lo que ilustra la importancia de no usar contraseñas débiles en archivos sensibles.

---

# 4) 💻 Acceso inicial y enumeración post-login (análisis extendido)

### 4.1. Extracción del contenido
```bash
unrar x Sup3rP4ss.rar
```

Contenido extraído: `Sup3rP4ss.txt` con la pista final `3spuM4`.

### 4.2. Generación de credenciales
Concatenando pistas y archivos (FTP + RAR + texto) se forma la credencial:

```
usuario: zappskred
password: 3spuM4
```

**Razonamiento del analista:** los labs frecuentemente dividen la información en varias capas para obligar al atacante a correlacionar y a automatizar pequeñas tareas. Aquí, la palabra obtenida del RAR apuntó directamente a la contraseña, y el FTP confirmó el usuario objetivo.

### 4.3. Acceso SSH
```bash
ssh zappskred@192.168.77.71
```

![ssh](zapp_2.png)

Una vez dentro como `zappskred`, la enumeración local habitual debe realizarse: `id`, `whoami`, `pwd`, `ls -la`, revisar home, `sudo -l`, revisar `~/.ssh`, historial (`.bash_history`) y ficheros en `/home` o `/var/www`.

**Buenas prácticas del analista:** priorizar `sudo -l` y `id` para detectar vectores de escalada rápidos (como en este laboratorio).

---

# 5) 🔎 Búsqueda de vectores de escalada — análisis y alternativas

### 5.1. Resultado observado
```bash
sudo -l
```
Indica `NOPASSWD` para ciertos comandos o `ALL`, lo que permite escalar sin necesidad de explotar bugs.

### 5.2. Evaluación de vectores alternativos (análisis lateral)
Aunque en este lab la escalada fue por `sudo -i`, el analista debe verificar sistemáticamente:

- **SUIDs y capabilities:** `find / -perm -4000 -type f 2>/dev/null` y `getcap -r / 2>/dev/null`. Incluso si `sudo` funciona, estas rutas pueden existir en otros escenarios.
- **Ficheros con credenciales:** revisar `/etc/passwd`, `/etc/shadow` (si accesible), ficheros de configuración bajo `/var/www` o backups.
- **Cron jobs y scripts ejecutados por root:** `ls -la /etc/cron*`, `cat /etc/crontab`.
- **Servicios con credenciales embebidas:** `systemctl list-units` y revisar unit files por comandos invocados con permisos.

### 5.3. Hipótesis del atacante
- **Hipótesis primaria:** obtengo credenciales SSH y compruebo `sudo -l`; si NOPASSWD está presente, las ventajas son inmediatas.  
- **Hipótesis secundaria (si sudo no existiera):** buscaría SUIDs o capacidades, o intentaría pivotar mediante servicios web con upload (no aplicable aquí).

---

# 6) ⬆️ Explotación / escalada a root — evidencia y explicación

### 6.1. Escalada efectiva en este laboratorio
```bash
sudo -i
```

![user y root](zapp_3.png)

Resultado: shell `root`.

### 6.2. Evidencia de compromiso
- `whoami` → `root`
- Lectura de `root.txt` (ocultada en este informe)

### 6.3. Valoración del riesgo e impacto
- **Nivel de compromiso:** crítico — control total de la máquina.  
- **Impacto en un entorno real:** exfiltración de datos, persistencia, pivoting hacia la red interna, despliegue de malware/ransomware.  
- **Causa raíz:** combinación de información expuesta y mala configuración de privilegios (`sudo NOPASSWD`), además de archivos con contraseñas débiles accesibles desde recursos públicos.

---

# 7) 📁 Post-explotación y evidencias (análisis forense breve)

### 7.1. Acciones típicas del atacante tras obtener root
- **Recopilar credenciales** (`/etc/shadow`, archivos de configuración`, `/root/.ssh/`).
- **Crear usuario de persistencia** o añadir clave pública a `authorized_keys`.
- **Limpiar logs** o modificar timestamps (aunque en CTF esto suele omitirse para dejar evidencia).
- **Revisar la red** para movimientos laterales (`arp -a`, `netstat -tulpn`).

### 7.2. Evidencias relevantes guardadas
- Hashes y outputs de `rar2john` / `john`.  
- Salida de `sudo -l`.  
- Comandos clave ejecutados (listados más abajo).

---

# 8) 🔧 Reproducibilidad — comandos clave (con placeholders)

```bash
# Reconocimiento
nmap -p- -T4 -sV -sC 192.168.77.71

# Enumeración web
feroxbuster -u http://192.168.77.71/

# FTP
ftp 192.168.77.71

# Descarga y cracking
wget http://192.168.77.71/cuatrocuatroveces/Sup3rP4ss.rar
rar2john Sup3rP4ss.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash

# Acceso
ssh zappskred@192.168.77.71

# Escalada
sudo -l
sudo -i
```

> Nota: sustituye la IP por `<TARGET_IP>` cuando vayas a reutilizar la guía como plantilla.

---

## 🧰 Herramientas utilizadas (breve descripción pedagógica)
- **Nmap:** descubrimiento y fingerprinting de servicios.
- **Feroxbuster:** enumeración de directorios/archivos en web.
- **FTP client:** para extraer recursos accesibles de forma anónima.
- **RAR2John:** conversión de RAR a hash para cracking offline.
- **John The Ripper:** recuperación de contraseñas a partir de hashes.
- **SSH:** acceso remoto y administración.
- **sudo:** control de privilegios del sistema.

---

## 🛡️ Mitigaciones recomendadas (priorizadas y accionables)
1. **Deshabilitar FTP anónimo** o restringir el contenido disponible públicamente.  
2. **No dejar pistas sensibles en HTML/FTP**; si se usan para ejercicios, mantenerlos fuera de entornos de producción.  
3. **Eliminar archivos comprimidos con contraseñas débiles** o almacenarlos con control de acceso.  
4. **Eliminar entradas `NOPASSWD` en sudoers**; usar políticas RBAC y least privilege.  
5. **Aplicar inspección de logs y alertas** para accesos inusuales a rutas ocultas y descargas masivas.  
6. **Políticas de contraseñas y rotación** para evitar cracking rápido con wordlists públicas.

---

## 🔍 Análisis pedagógico — lecciones aprendidas y ejercicios propuestos
1. **Ejercicio de detección**: buscar elementos ocultos en HTML y automatizar decodificaciones (script que detecta Base64 y lo intenta varias veces).  
2. **Ejercicio de correlación**: practicar cómo combinar hallazgos entre FTP y web para reconstruir credenciales.  
3. **Reto de hardening**: preparar un `sudoers` seguro y exponer las diferencias (antes/después) con logs.  
4. **Práctica de cracking responsable**: aprender cuándo usar cracking offline y cómo proteger archivos sensibles.


---

**Documento generado y compilado por deox420.**

