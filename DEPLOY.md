# InsForge — Guía de Despliegue y Configuración

> **Entorno detectado:** Ubuntu 20.04.6 LTS · Docker 26.1.3 · Docker Compose v2.27.0 · Node 24  
> **Servidor coexistente:** `naughty-backend` + Caddy (80/443/8080) — **no hay conflicto de puertos** con InsForge.

---

## Tabla de Contenidos

1. [Puertos que usa InsForge](#1-puertos-que-usa-insforge)
2. [Modo de despliegue recomendado](#2-modo-de-despliegue-recomendado)
3. [Configuración del archivo .env](#3-configuración-del-archivo-env)
4. [Levantar InsForge](#4-levantar-insforge)
5. [Verificar que funciona](#5-verificar-que-funciona)
6. [Exponer InsForge con dominio + HTTPS (Caddy)](#6-exponer-insforge-con-dominio--https-caddy)
7. [Gestión diaria](#7-gestión-diaria)
8. [Actualizaciones](#8-actualizaciones)
9. [Backup y restauración](#9-backup-y-restauración)
10. [Variables de entorno: referencia completa](#10-variables-de-entorno-referencia-completa)

---

## 1. Puertos que usa InsForge

| Puerto | Servicio | Descripción |
|--------|----------|-------------|
| `7130` | `insforge` | API principal + dashboard |
| `7131` | `insforge` | Servidor de autenticación |
| `5432` | `postgres` | PostgreSQL |
| `5430` | `postgrest` | PostgREST (API automática) |
| `7133` | `deno` | Runtime de edge functions |

**Tu servidor actual** ocupa los puertos `80`, `443`, `4096`, `8080`, `8443`.  
No hay solapamiento — podés levantar InsForge sin tocar nada de lo que ya está corriendo.

---

## 2. Modo de despliegue recomendado

Hay dos formas de desplegar InsForge. Elegí según tu caso:

### Opción A — Imágenes pre-construidas (recomendado para producción)

Usa imágenes oficiales publicadas en GitHub Container Registry.  
No requiere compilar nada, arranca en minutos.

```bash
cd /home/ubuntu/InsForge
cp deploy/docker-compose/docker-compose.yml ./docker-compose.deploy.yml
```

> El archivo `deploy/docker-compose/docker-compose.yml` ya usa imágenes oficiales  
> (`ghcr.io/insforge/insforge-oss`, `ghcr.io/insforge/postgres-all`, etc.)

### Opción B — Build desde el código fuente (desarrollo / personalización)

Usa el `docker-compose.prod.yml` de la raíz del repo, que compila la imagen localmente.  
Tarda más la primera vez (compilación de ~5 minutos), pero te da control total del código.

```bash
cd /home/ubuntu/InsForge
# ya tenés el docker-compose.prod.yml listo
```

---

## 3. Configuración del archivo .env

Este es el paso más importante. Todo lo de seguridad se configura acá.

### 3.1 Crear el archivo

```bash
cd /home/ubuntu/InsForge
cp .env.example .env
```

### 3.2 Configurar los secretos obligatorios

Abrí `.env` y reemplazá **mínimamente** estos valores:

```bash
# Generá dos claves independientes (una para JWT, otra para cifrado)
openssl rand -base64 32   # copiá el resultado en JWT_SECRET
openssl rand -base64 32   # copiá el resultado en ENCRYPTION_KEY
```

Editá `.env`:

```dotenv
# ── Seguridad ────────────────────────────────────────────────
JWT_SECRET=<resultado del primer openssl>
ENCRYPTION_KEY=<resultado del segundo openssl>

# ── Admin inicial ────────────────────────────────────────────
ROOT_ADMIN_USERNAME=admin
ROOT_ADMIN_PASSWORD=<contraseña segura — cámbiala siempre>

# ── URLs públicas (ajustá si usás dominio) ───────────────────
API_BASE_URL=http://localhost:7130
VITE_API_BASE_URL=http://localhost:7130

# ── Base de datos (podés dejar los defaults) ─────────────────
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=insforge
```

> ⚠️ `JWT_SECRET` y `ENCRYPTION_KEY` **deben ser distintos**.  
> Si algún día rotás `JWT_SECRET` sin actualizar `ENCRYPTION_KEY`, los secretos almacenados (OAuth tokens, API keys) se corrompen de forma permanente.

---

## 4. Levantar InsForge

### Con imágenes pre-construidas (Opción A)

```bash
cd /home/ubuntu/InsForge
docker compose -f deploy/docker-compose/docker-compose.yml --env-file .env up -d
```

### Desde el código fuente (Opción B)

```bash
cd /home/ubuntu/InsForge
docker compose -f docker-compose.prod.yml --env-file .env up -d --build
```

### Verificar que los contenedores estén saludables

```bash
docker compose -f docker-compose.prod.yml ps
```

Debería mostrar todos los servicios con estado `healthy` o `running`:

```
NAME                   STATUS
insforge-postgres-1    healthy
insforge-postgrest-1   running
insforge-insforge-1    running
insforge-deno-1        running
```

---

## 5. Verificar que funciona

```bash
# Health check de la API
curl http://localhost:7130/api/health

# Respuesta esperada:
# {"status":"ok"}
```

Abrí en el navegador:

- **Dashboard:** `http://<ip-del-server>:7131`
- **API:** `http://<ip-del-server>:7130/api`

Iniciá sesión con las credenciales que pusiste en `ROOT_ADMIN_USERNAME` / `ROOT_ADMIN_PASSWORD`.

---

## 6. Exponer InsForge con dominio + HTTPS (Caddy)

Ya tenés Caddy corriendo. No necesitás un segundo Caddy ni parar nada — solo agregás un nuevo sitio a la config existente.

### 6.1 Encontrar la config de Caddy del servidor existente

```bash
# Ubicación típica
docker exec naughty-backend-caddy-1 cat /etc/caddy/Caddyfile 2>/dev/null
# o si usás un archivo montado:
find /home/ubuntu -name "Caddyfile" 2>/dev/null
```

### 6.2 Agregar InsForge como nuevo sitio

En el `Caddyfile` existente, agregá un bloque para InsForge:

```caddyfile
# Reemplazá insforge.tudominio.com por tu dominio real
insforge.tudominio.com {
    reverse_proxy localhost:7130

    # Websockets y timeouts para la API
    reverse_proxy /api/* localhost:7130 {
        transport http {
            dial_timeout 30s
            response_header_timeout 120s
        }
    }
}

# Auth en subdominio separado (opcional pero más limpio)
auth.tudominio.com {
    reverse_proxy localhost:7131
}
```

### 6.3 Recargar Caddy sin reiniciar

```bash
docker exec naughty-backend-caddy-1 caddy reload --config /etc/caddy/Caddyfile
```

### 6.4 Actualizar las URLs en .env

```dotenv
API_BASE_URL=https://insforge.tudominio.com
VITE_API_BASE_URL=https://insforge.tudominio.com
```

Luego reiniciá el contenedor de InsForge:

```bash
docker compose -f docker-compose.prod.yml restart insforge
```

---

## 7. Gestión diaria

Todos los comandos asumen que estás en `/home/ubuntu/InsForge`.  
Reemplazá `-f docker-compose.prod.yml` por `-f deploy/docker-compose/docker-compose.yml` si usás la Opción A.

```bash
# Ver estado de todos los servicios
docker compose -f docker-compose.prod.yml ps

# Ver logs en tiempo real
docker compose -f docker-compose.prod.yml logs -f

# Logs de un servicio específico
docker compose -f docker-compose.prod.yml logs -f insforge
docker compose -f docker-compose.prod.yml logs -f postgres

# Reiniciar un servicio
docker compose -f docker-compose.prod.yml restart insforge

# Parar todo (sin borrar datos)
docker compose -f docker-compose.prod.yml down

# Parar y borrar volúmenes (⚠️ DESTRUYE LA BASE DE DATOS)
docker compose -f docker-compose.prod.yml down -v
```

---

## 8. Actualizaciones

### Opción A (imágenes pre-construidas)

```bash
cd /home/ubuntu/InsForge

# Bajar nuevas imágenes
docker compose -f deploy/docker-compose/docker-compose.yml pull

# Reiniciar con las nuevas imágenes
docker compose -f deploy/docker-compose/docker-compose.yml --env-file .env up -d

# Las migraciones de base de datos corren automáticamente al iniciar el contenedor
```

### Opción B (desde el código fuente)

```bash
cd /home/ubuntu/InsForge

# Actualizar el código
git pull origin main

# Reconstruir y reiniciar
docker compose -f docker-compose.prod.yml --env-file .env up -d --build
```

---

## 9. Backup y restauración

### Backup de la base de datos

```bash
# Crear backup
docker exec insforge-postgres-1 \
  pg_dump -U postgres insforge > backup-$(date +%Y%m%d-%H%M%S).sql

# O comprimir:
docker exec insforge-postgres-1 \
  pg_dump -U postgres insforge | gzip > backup-$(date +%Y%m%d-%H%M%S).sql.gz
```

### Restaurar un backup

```bash
# Restaurar desde .sql
cat backup-20260606-120000.sql | docker exec -i insforge-postgres-1 \
  psql -U postgres insforge

# Restaurar desde .sql.gz
gunzip -c backup-20260606-120000.sql.gz | docker exec -i insforge-postgres-1 \
  psql -U postgres insforge
```

### Backup del storage local (si no usás S3)

```bash
# El volumen de Docker se llama insforge_storage-data
# Para exportar:
docker run --rm \
  -v insforge_storage-data:/source \
  -v $(pwd)/backups:/dest \
  alpine tar czf /dest/storage-backup-$(date +%Y%m%d).tar.gz -C /source .
```

---

## 10. Variables de entorno: referencia completa

### Obligatorias (siempre)

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `JWT_SECRET` | Clave para firmar tokens JWT. Mín. 32 chars. | `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | Clave para cifrar secretos en DB. Mín. 32 chars. **Distinta de JWT_SECRET.** | `openssl rand -base64 32` |
| `ROOT_ADMIN_USERNAME` | Nombre del admin inicial | `admin` |
| `ROOT_ADMIN_PASSWORD` | Contraseña del admin inicial | `my-strong-password` |

### Opcionales frecuentes

| Variable | Descripción | Default |
|----------|-------------|---------|
| `API_BASE_URL` | URL pública de la API | `http://localhost:7130` |
| `VITE_API_BASE_URL` | URL pública para el frontend | `http://localhost:7130` |
| `POSTGRES_PASSWORD` | Contraseña de PostgreSQL | `postgres` |
| `MAX_FILE_SIZE` | Tamaño máximo de upload (bytes) | `52428800` (50 MB) |
| `OPENROUTER_API_KEY` | API key para funciones de IA | — |

### Storage (elegir uno)

| Variable | Descripción |
|----------|-------------|
| `STORAGE_DIR` | Path local para almacenamiento de archivos (default: `/insforge-storage`) |
| `AWS_S3_BUCKET` + `AWS_REGION` + `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | Usar AWS S3 |
| `S3_ENDPOINT_URL` + `S3_ACCESS_KEY_ID` + `S3_SECRET_ACCESS_KEY` | Usar S3-compatible (Wasabi, MinIO, R2) |

### OAuth (opcionales — para habilitar login social)

| Variable | Proveedor |
|----------|-----------|
| `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` | Google |
| `GITHUB_CLIENT_ID` + `GITHUB_CLIENT_SECRET` | GitHub |
| `DISCORD_CLIENT_ID` + `DISCORD_CLIENT_SECRET` | Discord |
| `MICROSOFT_CLIENT_ID` + `MICROSOFT_CLIENT_SECRET` | Microsoft |

**Redirect URI para todos:** `http(s)://<tu-host>:7130/auth/<provider>/callback`  
Ejemplo para GitHub: `https://insforge.tudominio.com/auth/github/callback`

### Edge Functions (opcionales)

| Variable | Descripción |
|----------|-------------|
| `DENO_SUBHOSTING_TOKEN` | Token de Deno Deploy para subhosting |
| `DENO_SUBHOSTING_ORG_ID` | Org ID en Deno Deploy |

### Compute / Fly.io (opcional)

| Variable | Descripción |
|----------|-------------|
| `FLY_API_TOKEN` | Token de Fly.io para deploy de containers |
| `FLY_ORG` | Tu org slug en Fly.io (NO usar "insforge") |
| `COMPUTE_DOMAIN` | Wildcard domain para apps desplegadas en Fly |

---

## Nota sobre el servidor existente

Tu stack actual (`naughty-backend` + Caddy) corre en puertos **80, 443, 4096, 8080, 8443**.  
InsForge usa **7130, 7131, 7133, 5432, 5430** — no hay conflicto.

**No necesitás bajar nada** para instalar InsForge.  
Si en algún momento querés exponer InsForge por el mismo Caddy que ya tenés, seguí la sección [6](#6-exponer-insforge-con-dominio--https-caddy) — solo agregás un bloque al Caddyfile existente y recargás, sin interrumpir los otros servicios.

---

*Generado el 2026-06-06 — InsForge monorepo en `/home/ubuntu/InsForge`*
