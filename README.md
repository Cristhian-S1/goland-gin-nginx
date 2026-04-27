# Validador de Transacciones

Servicio crítico de validación para e-commerce. 3 réplicas Golang/Gin balanceadas con NGINX Round-Robin.

## Requisitos previos

- Go 1.21+
- NGINX instalado (`sudo pacman go nginx`)
- systemd (incluido en Ubuntu/Debian/Arch)

## Despliegue manual paso a paso

# Validador de Transacciones — Guía Completa de Despliegue

Marketplace e-commerce con validación de pagos distribuida. 3 réplicas Go/Gin balanceadas con NGINX Round-Robin, frontend estático servido por NGINX.

---

## Índice

1. [Arquitectura del sistema](#1-arquitectura-del-sistema)
2. [Estructura de archivos del proyecto](#2-estructura-de-archivos-del-proyecto)
3. [Debian/Ubuntu — Instalación completa](#3-debianubuntu--instalación-completa)
4. [Verificación del sistema](#4-verificación-del-sistema)
5. [Reversión completa — Debian/Ubuntu](#5-reversión-completa--debianubuntu)
6. [Referencia rápida de comandos](#6-referencia-rápida-de-comandos)

---

## 1. Arquitectura del sistema

```
Navegador (cliente)
        │
        ▼  puerto 80
┌───────────────────────────────┐
│            NGINX              │
│  ┌─────────────────────────┐  │
│  │  root /opt/.../static   │  │  ← Sirve index.html (frontend)
│  │  location /validar      │  │  ← Proxy hacia réplicas Go
│  └─────────────────────────┘  │
└──────────┬─────┬──────┬───────┘
           │     │      │   Round-Robin: 3001→3002→3003→3001...
           ▼     ▼      ▼
        :3001  :3002  :3003      ← Réplicas Go/Gin (mismo binario, PORT distinto)
```

**Niveles de la arquitectura N-Tier:**
- **Nivel 1** — Cliente (navegador, cualquier red)
- **Nivel 2** — Balanceador de carga (NGINX :80) — única puerta de entrada
- **Nivel 3** — Capa de lógica de negocio (3 réplicas Go/Gin)

**Endpoints disponibles:**

| Ruta | Descripción |
|------|-------------|
| `GET /` | Frontend del marketplace |
| `GET /validar` | Validación de transacción (80% aprobada, 20% rechazada) |
| `GET /health` | Health check de la réplica que responde |

---

## 2. Estructura de archivos del proyecto

```
proyecto/
├── main.go                    # Aplicación Go con Gin (lógica + rutas)
├── go.mod                     # Módulo Go y declaración de dependencias
├── index.html                 # Frontend del marketplace (va en static/)
├── validador-tx@.service      # Unit file systemd (template para 3 réplicas)
├── validador-tx.nginx.conf    # Configuración NGINX completa
├── demo.sh                    # Script de demostración para presentación
└── SIMULACRO.md               # Guía de presentación paso a paso
```

**Rutas de instalación en el servidor:**

| Archivo | Destino |
|---------|---------|
| Binario compilado | `/opt/validador-tx/validador-tx` |
| Frontend | `/opt/validador-tx/static/index.html` |
| Unit systemd | `/etc/systemd/system/validador-tx@.service` |
| Config NGINX (Arch) | `/etc/nginx/sites-enabled/validador-tx.conf` |
| Config NGINX (Ubuntu) | `/etc/nginx/sites-available/validador-tx` |

---

## 3. Debian/Ubuntu — Instalación completa

> Esta sección aplica a la VM asignada: **Ubuntu-vm-01 · IP 146.83.102.20**

### 3.1 Instalar dependencias

```bash
sudo apt update
sudo apt install -y golang nginx
```

Verifica:

```bash
go version
nginx -v
```

### 3.2 Compilar el proyecto

```bash
mkdir -p ~/validador-tx
cd ~/validador-tx
# Copia main.go y go.mod a esta carpeta (vía scp o editor)
go mod tidy
go build -o validador-tx .
```

Copiar archivos desde tu Arch Linux hacia la VM (ejecuta desde Arch):

```bash
# Crear directorio remoto
ssh usuario@146.83.102.20 "mkdir -p ~/validador-tx"

# Transferir todos los archivos
scp main.go go.mod index.html validador-tx.nginx.conf validador-tx@.service \
    usuario@146.83.102.20:~/validador-tx/
```

### 3.3 Instalar binario y frontend

En Ubuntu el usuario de NGINX es `www-data`. El `validador-tx@.service` viene configurado con `User=www-data` por defecto, así que no requiere modificación.

```bash
sudo mkdir -p /opt/validador-tx/static

sudo cp validador-tx /opt/validador-tx/
sudo chmod +x /opt/validador-tx/validador-tx

sudo cp index.html /opt/validador-tx/static/

sudo chown -R www-data:www-data /opt/validador-tx
```

### 3.4 Instalar el servicio systemd

No se necesita modificar el `User=` esta vez:

```bash
sudo cp validador-tx@.service /etc/systemd/system/
sudo systemctl daemon-reload

sudo systemctl enable --now validador-tx@3001
sudo systemctl enable --now validador-tx@3002
sudo systemctl enable --now validador-tx@3003
```

Verifica:

```bash
sudo systemctl status validador-tx@{3001,3002,3003}
```

### 3.5 Configurar NGINX en Debian/Ubuntu

Ubuntu incluye el sistema `sites-available` / `sites-enabled` de forma nativa. No hay que editar `nginx.conf` a mano.

```bash
# Instalar la configuración
sudo cp validador-tx.nginx.conf /etc/nginx/sites-available/validador-tx

# Activar con symlink
sudo ln -s /etc/nginx/sites-available/validador-tx \
           /etc/nginx/sites-enabled/validador-tx

# Deshabilitar el sitio por defecto que ocupa el puerto 80
sudo rm /etc/nginx/sites-enabled/default

# Verificar y recargar
sudo nginx -t
sudo systemctl reload nginx
```

### 3.6 Cambiar la URL de la API en el frontend

El frontend tiene la URL de la API hardcodeada. Para que funcione desde cualquier navegador que acceda a la VM, debe apuntar a la IP pública:

Edita `/opt/validador-tx/static/index.html` en la VM, busca la línea:

```js
const API_URL = 'http://localhost/validar';
```

Cámbiala por:

```js
const API_URL = 'http://146.83.102.20/validar';
```

O hazlo directamente desde Arch antes de subir el archivo:

```bash
# En tu Arch, antes del scp:
sed -i "s|http://localhost/validar|http://146.83.102.20/validar|" index.html
```

### 3.7 Verificar la instalación completa en Ubuntu

```bash
# En la VM
for p in 3001 3002 3003; do
  echo -n "Réplica :$p → "
  curl -s http://localhost:$p/health | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print('OK · port', d['port'])"
done

# Frontend
curl -s http://localhost | grep -o '<title>.*</title>'

# Round-Robin
for i in {1..6}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print(d['status'], '→', d['port'])"
done
```

Desde tu navegador en Arch: `http://146.83.102.20`

---

## 4. Verificación del sistema

Estos comandos funcionan igual en ambas distros:

```bash
# Estado compacto de las 3 réplicas
for p in 3001 3002 3003; do
  printf "validador-tx@%-6s → %s\n" "$p" "$(systemctl is-active validador-tx@$p)"
done

# Puertos en escucha (deben aparecer 80, 3001, 3002, 3003)
ss -tlnp | grep -E ':80|:300[123]'

# Distribución del Round-Robin (30 requests)
echo "Distribución Round-Robin (30 requests):"
for i in {1..30}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; print(json.load(sys.stdin)['port'])"
done | sort | uniq -c | sort -rn
# Resultado esperado: ~10 por cada réplica

# Simular caída y recuperación
sudo systemctl stop validador-tx@3002
echo "Con :3002 caída (solo 3001 y 3003 deben aparecer):"
for i in {1..4}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print(d['port'])"
done
sudo systemctl start validador-tx@3002
```

---
## 5. Reversión completa — Debian/Ubuntu

### 5.1 Detener y deshabilitar las réplicas

```bash
sudo systemctl stop validador-tx@3001
sudo systemctl stop validador-tx@3002
sudo systemctl stop validador-tx@3003

sudo systemctl disable validador-tx@3001
sudo systemctl disable validador-tx@3002
sudo systemctl disable validador-tx@3003
```

### 5.2 Eliminar el unit file y limpiar systemd

```bash
sudo rm /etc/systemd/system/validador-tx@.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

### 5.3 Eliminar binario y frontend

```bash
sudo rm -rf /opt/validador-tx
```

### 5.4 Revertir la configuración de NGINX en Ubuntu

Paso 1 — Elimina el symlink activo y el archivo de configuración:

```bash
sudo rm /etc/nginx/sites-enabled/validador-tx
sudo rm /etc/nginx/sites-available/validador-tx
```

Paso 2 — Restaura el sitio por defecto:

```bash
sudo ln -s /etc/nginx/sites-available/default \
           /etc/nginx/sites-enabled/default
```

Paso 3 — Verifica y recarga:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Confirma que NGINX volvió a su estado original:

```bash
curl http://localhost
# Debe devolver el HTML de bienvenida de NGINX
```

### 5.5 Desinstalar paquetes (opcional)

```bash
sudo apt remove nginx golang
sudo apt autoremove   # elimina dependencias huérfanas
sudo apt purge nginx  # elimina también archivos de configuración del paquete
```

## 6. Referencia rápida de comandos

### Control de réplicas

```bash
# Ver estado de las 3 de un vistazo
for p in 3001 3002 3003; do
  printf "%-30s %s\n" "validador-tx@$p" "$(systemctl is-active validador-tx@$p)"
done

# Parar una réplica (simular fallo)
sudo systemctl stop validador-tx@3002

# Iniciar una réplica (restaurar)
sudo systemctl start validador-tx@3002

# Reiniciar todas
sudo systemctl restart validador-tx@3001
sudo systemctl restart validador-tx@3002
sudo systemctl restart validador-tx@3003

# Ver logs en tiempo real de una réplica
sudo journalctl -fu validador-tx@3001

# Ver logs de las 3 simultáneamente
sudo journalctl -fu validador-tx@3001 &
sudo journalctl -fu validador-tx@3002 &
sudo journalctl -fu validador-tx@3003
```

### Control de NGINX

```bash
sudo nginx -t                      # verificar sintaxis
sudo systemctl reload nginx        # recargar sin cortar conexiones activas
sudo systemctl restart nginx       # reinicio completo
sudo systemctl status nginx        # ver estado
tail -f /var/log/nginx/error.log   # logs de error en tiempo real
```

### Pruebas de API

```bash
# Health check por réplica (directo, sin NGINX)
curl -s http://localhost:3001/health | python3 -m json.tool
curl -s http://localhost:3002/health | python3 -m json.tool
curl -s http://localhost:3003/health | python3 -m json.tool

# Verificar Round-Robin (el campo "port" debe rotar)
for i in {1..9}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print(f\"Req $i → :{d['port']} — {d['status']}\")"
done

# Distribución estadística (50 requests)
for i in {1..50}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; print(json.load(sys.stdin)['port'])"
done | sort | uniq -c | sort -rn

# Verificar puertos en escucha
ss -tlnp | grep -E ':80|:300[123]'
```

### Demo de tolerancia a fallos

```bash
# 1. Matar réplica 3002
sudo systemctl stop validador-tx@3002

# 2. Confirmar que el sistema sigue respondiendo
for i in {1..4}; do
  curl -s http://localhost/validar | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print('OK →', d['port'])"
done

# 3. Restaurar
sudo systemctl start validador-tx@3002

# 4. Demo automatizada completa (con pauses)
sudo bash demo.sh
# o para la VM:
sudo bash demo.sh http://146.83.102.20
```

## Verificación

```bash
# Estado de las réplicas
sudo systemctl status validador-tx@3001
sudo systemctl status validador-tx@3002
sudo systemctl status validador-tx@3003

# Logs en tiempo real
sudo journalctl -fu validador-tx@3001

# Probar balanceo Round-Robin (el campo "port" debe rotar entre 3001, 3002, 3003)
for i in {1..6}; do curl -s http://localhost/validar | python3 -m json.tool; done
```

## Endpoint

`GET /validar`

**Respuesta aprobada (200):**
```json
{
  "status": "aprobada",
  "message": "Transacción aprobada correctamente.",
  "port": "3001",
  "processed_at": "2025-04-20T14:32:10Z"
}
```

**Respuesta rechazada (422):**
```json
{
  "status": "rechazada",
  "message": "Transacción rechazada por política de riesgo.",
  "port": "3002",
  "processed_at": "2025-04-20T14:32:11Z"
}
```

## Algoritmo de balanceo

**Round-Robin** (default NGINX). Elegido porque las validaciones son stateless y de carga uniforme, por lo que la distribución secuencial es óptima sin overhead de monitoreo.

Guía de Mantenimiento y Actualización: Validador-TX

Este documento detalla los procedimientos necesarios para aplicar cambios en los distintos componentes del proyecto, clasificados según el impacto y las acciones requeridas para su despliegue.
⚡ Cambios Instantáneos (Archivos Estáticos y Configuración)

Estos archivos pueden modificarse sin afectar la ejecución de las réplicas de la aplicación.
1. index.html (Frontend)

NGINX sirve este archivo directamente. Los cambios son visibles de forma inmediata en el próximo request.
Bash

# Editar directamente en la VM
sudo nano /opt/validador-tx/static/index.html

# No necesitas reiniciar nada. Para verificar el cambio:
curl -s http://localhost | grep '<title>'

2. validador-tx.nginx.conf (Configuración NGINX)

Requiere recargar el servicio NGINX para aplicar nuevas rutas, puertos o reglas de balanceo.
Bash

# Editar la configuración
sudo nano /etc/nginx/sites-available/validador-tx

# 1. Verificar que la sintaxis sea correcta
sudo nginx -t

# 2. Aplicar cambios sin cortar conexiones activas
sudo systemctl reload nginx

🛠️ Cambios que requieren Acción (Recompilación y Reinicio)
1. main.go (Aplicación Core)

Al ser un lenguaje compilado, cualquier cambio en la lógica requiere generar un nuevo binario y reiniciar las réplicas para liberar el archivo en uso.

Pasos para el despliegue:

    Conectarse a la VM:
    Bash

    ssh usuario@146.83.102.20

    Preparar y Compilar:
    Bash

    cd ~/validador-tx
    nano main.go
    go build -o validador-tx .

    Detener réplicas y reemplazar binario:
    Bash

    # Parar réplicas para liberar el binario
    sudo systemctl stop validador-tx@3001 validador-tx@3002 validador-tx@3003

    # Reemplazar e instalar
    sudo cp validador-tx /opt/validador-tx/validador-tx
    sudo chown www-data:www-data /opt/validador-tx/validador-tx

    Reiniciar y Verificar:
    Bash

    sudo systemctl start validador-tx@3001 validador-tx@3002 validador-tx@3003

    # Test de salud
    for p in 3001 3002 3003; do
      echo -n ":$p → "; curl -s http://localhost:$p/health
    done

    [!TIP]
    Si no te importa un tiempo de inactividad (downtime) de milisegundos, puedes simplificar los pasos 3 y 4 usando:
    sudo systemctl restart validador-tx@{3001,3002,3003}

2. validador-tx@.service (Unidad de Systemd)

Si modificas variables de entorno, límites de recursos o parámetros de ejecución en el archivo de servicio.
Bash

sudo nano /etc/systemd/system/validador-tx@.service

# Recargar configuración de systemd y reiniciar réplicas
sudo systemctl daemon-reload
sudo systemctl restart validador-tx@3001 validador-tx@3002 validador-tx@3003

📊 Tabla Resumen
Archivo	Función	Acción tras editar
index.html	Frontend estático	Ninguna (Inmediato)
nginx.conf	Balanceo y rutas	nginx -t → systemctl reload nginx
main.go	Lógica del servidor	Recompilar + Reemplazar binario + Reiniciar réplicas
validador-tx@.service	Gestión de procesos	daemon-reload + Reiniciar réplicas
💡 Regla General

    Texto leído en runtime (HTML, Config de NGINX): Cambio inmediato o simple reload.

    Código fuente compilado (Go): Requiere recompilar el binario y reiniciar los procesos para que carguen el nuevo código en memoria.


