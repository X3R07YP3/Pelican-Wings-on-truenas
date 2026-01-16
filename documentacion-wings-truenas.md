# Guía de Instalación: Pelican Wings en TrueNAS Scale

Esta guía detalla cómo configurar Pelican Wings en TrueNAS Scale usando Docker Compose, resolviendo los problemas comunes de bind mounts.

## Índice

1. [Requisitos Previos](#requisitos-previos)
2. [Estructura de Directorios](#estructura-de-directorios)
3. [Creación de Directorios](#creación-de-directorios)
4. [Configuración de Docker Compose](#configuración-de-docker-compose)
5. [Problema Común: Bind Mounts](#problema-común-bind-mounts)
6. [Despliegue](#despliegue)
7. [Verificación](#verificación)
8. [Solución de Problemas](#solución-de-problemas)

---

## Requisitos Previos

- TrueNAS Scale server
- portainer (si quieres ver los logs del wing y ver los contenedores creados por el wing)
- Acceso SSH al servidor
- Panel de Pelican instalado y funcionando
- Node creado en el panel (para obtener UUID, token_id y token)

---

## Estructura de Directorios

Wings necesita los siguientes directorios en el host:

```
/mnt/<tu-pool>/<tu-Dataset>/wings/
├── varlib/                    # Datos persistentes de Wings
│   └── volumes/               # Volúmenes de servidores de juego (CRÍTICO)
├── logs/                      # Logs de Wings
└── etc/                       # Configuración de Wings

/tmp/pelican/                  # Directorio temporal para instalaciones (CRÍTICO)
```

### Descripción de cada directorio

| Directorio | Propósito | Persistencia |
|------------|-----------|--------------|
| `varlib/` | Datos internos de Wings | Persistente |
| `varlib/volumes/` | Archivos de cada servidor de juego | Persistente |
| `logs/` | Logs de Wings | Persistente |
| `etc/` | Configuración | Persistente |
| `/tmp/pelican/` | Scripts de instalación temporales | Temporal (se limpia en reboot) |

---

## Creación de Directorios

Ejecuta los siguientes comandos en TrueNAS (vía SSH):

```bash
# Crear estructura principal (ajusta la ruta a tu pool)
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/varlib/volumes
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/logs
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/etc

# Crear directorio temporal (CRÍTICO para instalaciones)
mkdir -p /tmp/pelican

# Verificar que se crearon correctamente
ls -la /mnt/<tu-pool>/<tu-Dataset>/wings/
ls -la /tmp/pelican
```

### Permisos (si es necesario)

```bash
# Si tienes problemas de permisos
chmod -R 755 /mnt/<tu-pool>/<tu-Dataset>/wings/
chmod -R 755 /tmp/pelican
```

---

## Configuración de Docker Compose

Crea el archivo `wing.yml` con la siguiente estructura:

```yaml
configs:
  wings_config:
    content: |
      debug: false
      uuid: <UUID-DEL-NODE>
      token_id: <TOKEN-ID>
      token: <TOKEN>
      api:
        host: 0.0.0.0
        port: 8080
        ssl:
          enabled: false
          cert: /etc/letsencrypt/live/<tu-dominio>/fullchain.pem
          key: /etc/letsencrypt/live/<tu-dominio>/privkey.pem
        upload_limit: 256
      system:
        data: /mnt/<tu-pool>/<tu-Dataset>/wings/varlib/volumes
        sftp:
          bind_port: 2022
      allowed_mounts: []
      remote: "http://<IP-DEL-PANEL>:<PUERTO>"

networks:
  wings0:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: wings0
    ipam:
      config:
        - subnet: 172.21.0.0/16
    name: wings0

services:
  wings:
    configs:
      - source: wings_config
        target: /etc/pelican/config.yml
    container_name: pelican_wings
    environment:
      TZ: Europe/Madrid
      WINGS_GID: 0
      WINGS_UID: 0
      WINGS_USERNAME: pelican
    image: ghcr.io/pelican-dev/wings:latest
    networks:
      - wings0
    ports:
      - '8080:8080'
      - '2022:2022'
    restart: always
    tty: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/<tu-pool>/<tu-Dataset>/wings/varlib:/var/lib/pelican
      - /mnt/<tu-pool>/<tu-Dataset>/wings/logs:/var/log/pelican
      - /mnt/<tu-pool>/<tu-Dataset>/wings/etc:/etc/pelican
      - /etc/ssl/certs:/etc/ssl/certs:ro
      # Path-through mounts: CRÍTICO - Wings usa estos paths exactos al crear contenedores
      - /mnt/<tu-pool>/<tu-Dataset>/wings/varlib/volumes:/mnt/<tu-pool>/<tu-Dataset>/wings/varlib/volumes
      - /tmp/pelican:/tmp/pelican

version: '3.9'
```

### Variables a reemplazar

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `<tu-pool>` | Nombre de tu pool de almacenamiento | `Disco3tb` |
| `<tu-Dataset>` | Nombre de dataset | `pelicanweb` |
| `<UUID-DEL-NODE>` | UUID generado por el panel al crear el node | `f227d6dd-c2ef-...` |
| `<TOKEN-ID>` | ID del token de autenticación | `IzHReNj35nAZudPJ` |
| `<TOKEN>` | Token de autenticación | (cadena larga) |
| `<IP-DEL-PANEL>` | IP donde corre el panel de Pelican | `192.168.1.140` |
| `<PUERTO>` | Puerto del panel | `8088` |
| `<tu-dominio>` | Dominio para SSL (si usas HTTPS) | `panel.ejemplo.com` |

---

## Problema Común: Bind Mounts

### El Problema

Cuando Wings corre en un contenedor Docker y crea servidores de juego, ocurre lo siguiente:

1. Wings crea directorios dentro del contenedor (ej: `/tmp/pelican/UUID/`)
2. Wings luego crea contenedores de servidores que montan esos directorios
3. Docker busca esos paths en el **HOST**, no dentro del contenedor de Wings
4. Si el path no existe en el host → **ERROR: bind source path does not exist**

### La Solución: Path-Through Mounts

Los **path-through mounts** mapean el mismo path dentro y fuera del contenedor:

```yaml
# ❌ INCORRECTO - Wings ve /tmp/pelican, pero el host tiene otro path
- /mnt/pool/tmp:/tmp/pelican

# ✅ CORRECTO - El mismo path existe en ambos lugares
- /tmp/pelican:/tmp/pelican
- /mnt/pool/wings/varlib/volumes:/mnt/pool/wings/varlib/volumes
```

### Por qué funciona

```
Wings Container                    Host (TrueNAS)
─────────────────                  ───────────────
/tmp/pelican/UUID/    ←────────→   /tmp/pelican/UUID/
       │                                  │
       └── Wings crea aquí                └── Docker encuentra aquí
```

---

## Despliegue

### Paso 1: Crear directorios

```bash
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/varlib/volumes
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/logs
mkdir -p /mnt/<tu-pool>/<tu-Dataset>/wings/etc
mkdir -p /tmp/pelican
```

### Paso 2: Importar aplicación en TrueNAS Scale

1. Ve a **Aplicaciones** en el menú principal de TrueNAS
2. Haz clic en el botón azul **"Descubrir aplicaciones"**
3. En la nueva pestaña, busca en la esquina superior derecha el botón **"Custom App"**
4. Haz clic en los **3 puntitos (...)** junto a "Custom App"
5. Selecciona **"Install via YAML"**
6. Pega el contenido completo de tu archivo `wing.yml`
7. Haz clic en **"Install"**

### Paso 3: Verificar despliegue
Para este paso necesitas portainer ya que el log no se muestra bien en truenas (o al menos a mi no me mostraba bien el log)

---

## Verificación

### Verificar desde el panel

1. Ve a **Admin → Nodes** en el panel de Pelican
2. El node debería mostrar estado "Connected" (verde)

---

## Solución de Problemas

### Error: "bind source path does not exist"

**Causa**: Falta un path-through mount o el directorio no existe en el host.

**Solución**:
1. Identifica qué path falta en el error
2. Crea el directorio en el host: `mkdir -p /path/que/falta`
3. Asegúrate de tener el path-through mount en `wing.yml`
4. Recrea el contenedor: `docker compose -f wing.yml down && docker compose -f wing.yml up -d`

### Error: "connection refused" al panel

**Causa**: Wings no puede conectar con el panel.

**Solución**:
1. Verifica que la URL en `remote:` sea correcta
2. Asegúrate de que el panel esté accesible desde TrueNAS
3. Verifica firewalls/puertos

### Los servidores no inician

**Causa**: Problemas de permisos o red.

**Solución**:
1. Verifica que la red `wings0` existe: `docker network ls | grep wings0`
2. Revisa permisos de los directorios
3. Revisa logs: `docker logs pelican_wings`

### /tmp/pelican se borra en reboot

**Nota**: Esto es normal. El directorio `/tmp` se limpia en reinicios. Solo necesitas recrearlo:

```bash
mkdir -p /tmp/pelican
```

Puedes agregar esto a un script de inicio o crear un cronjob `@reboot`.

---

## Resumen de Puertos

| Puerto | Servicio | Descripción |
|--------|----------|-------------|
| 8080 | API Wings | Comunicación con el panel |
| 2022 | SFTP | Acceso a archivos de servidores |

---

## Notas Adicionales

- **SSL**: Si usas SSL, asegúrate de que los certificados estén accesibles y configura `ssl.enabled: true`
- **Timezone**: Ajusta `TZ` según tu ubicación
- **Red**: La subnet `172.21.0.0/16` puede cambiarse si hay conflictos

---

*Documentación creada para Pelican Panel + Wings en TrueNAS Scale*
