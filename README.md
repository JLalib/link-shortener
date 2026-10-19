# 🔗 Link Shortener

[![GitHub](https://img.shields.io/badge/GitHub-pablokbs%2Flink--shortener-181717?logo=github)](https://github.com/pablokbs/link-shortener)
[![Docker](https://img.shields.io/badge/Docker-ghcr.io%2Fpablokbs%2Flink--shortener-2496ED?logo=docker)](https://github.com/pablokbs/link-shortener/pkgs/container/link-shortener)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/pablokbs/link-shortener/blob/main/LICENSE)

## 📋 Descripción general

**Link Shortener** es un acortador de URLs autohospedado, open source, construido con **Node.js 22**, **Fastify 5**, **PostgreSQL 16** y **Drizzle ORM**. Permite crear enlaces cortos con **dominio propio**, ofrece un **panel administrativo seguro** con autenticación basada en sesiones y hash Argon2id, y proporciona **analíticas detalladas** de clics (series temporales, países, referrers). Se despliega mediante **Docker Compose** con **Caddy** como proxy inverso que gestiona certificados **TLS de Let's Encrypt** de forma automática. El proyecto está licenciado bajo **MIT**.

## ✨ Características principales

- 🌐 **Enlaces cortos con dominio propio** — Hostname personalizable mediante `SHORTENER_DOMAIN`
- 🔀 **Redirecciones configurables** — Códigos HTTP 301, 302, 307, 308 (global o por enlace)
- 🔐 **Panel administrativo seguro** — Autenticación por sesiones, hash Argon2id, protección CSRF
- 📝 **Gestión completa de enlaces** — Crear, editar, desactivar, archivar; vencimiento, título, descripción, metadatos
- 📊 **Analíticas globales y por enlace** — Series temporales, distribución por países, referrers, clics recientes
- 🤖 **API JSON para automatización** — Endpoints REST protegidos por sesión o token heredado (`ADMIN_TOKEN`)
- 🛡️ **Captura de IP respetuosa con la privacidad** — Solo se almacena hash de IP para análisis geográfico
- 🐳 **Despliegue listo para producción** — Docker Compose con app, PostgreSQL (red interna), Caddy, TLS automático
- 💻 **Desarrollo local simplificado** — `docker-compose.yml` expone puertos y permite crear admin vía variable de entorno
- ✅ **Validación estricta de configuración** — Falla al arranque si falta alguna variable obligatoria

## 📋 Requisitos del sistema

- VPS con **Ubuntu 22.04, 24.04 o 26.04** y acceso SSH como root
- Registro **DNS tipo A** apuntando el subdominio deseado a la IPv4 pública del VPS
- Puertos **TCP 80 y 443** abiertos en el firewall del proveedor y del servidor
- **Docker Engine** y **Docker Compose** (plugin `docker-compose-plugin`) instalados desde el repositorio oficial de Docker para Ubuntu
- Clonar el repositorio y configurar archivo `.env.production` con valores seguros generados mediante `openssl`

## 🐳 Instalación

### 1. Preparar el VPS y el DNS

Crear un VPS con Ubuntu, anotar su IPv4 pública y crear un registro A en el proveedor DNS para el subdominio elegido (ej. `go.example.com - 203.0.113.10`). Verificar que los puertos 80 y 443 estén permitidos.

```bash
# Ejemplo de registro DNS
go.example.com - 203.0.113.10
```

### 2. Instalar Docker Engine y Docker Compose

Utilizar el repositorio oficial de Docker para Ubuntu para instalar `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` y `docker-compose-plugin`. Verificar la instalación con `hello-world` y `docker compose version`.

```bash
apt update
apt install -y ca-certificates curl git
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

docker run --rm hello-world
docker compose version
```

### 3. Descargar y configurar Link Shortener

Clonar el repositorio, copiar el archivo de ejemplo de producción a `.env.production`, restringir permisos y editar las variables requeridas: `SHORTENER_DOMAIN` (solo hostname), `POSTGRES_PASSWORD` (hexadecimal de 32 bytes) y `SESSION_SECRET` (Base64 de al menos 32 caracteres).

```bash
git clone https://github.com/pablokbs/link-shortener.git
cd link-shortener
cp .env.production.example .env.production
chmod 600 .env.production
nano .env.production

# Generar valores seguros
openssl rand -hex 32
openssl rand -base64 48
```

### 4. Levantar la aplicación con HTTPS automático

Ejecutar `docker compose` con el archivo `compose.production.yml` y el archivo de entorno `.env.production`. El stack incluye la app, PostgreSQL (solo red interna), Caddy y certificados Let's Encrypt automáticos. Verificar el estado con `curl` al endpoint `/healthz`.

```bash
docker compose --env-file .env.production \
  -f compose.production.yml up --build -d

docker compose --env-file .env.production \
  -f compose.production.yml ps

curl "https://$(grep '^SHORTENER_DOMAIN=' .env.production | cut -d= -f2)/healthz"
```

### 5. Crear el primer usuario administrador

Ejecutar el script `create-admin` dentro del contenedor de la app, proporcionando el email. La contraseña se solicita de forma interactiva y oculta. Luego acceder al panel en `https://<dominio>/admin/login`.

```bash
docker compose --env-file .env.production \
  -f compose.production.yml exec app \
  npm run create-admin -- \
  --email=admin@example.com

# Acceder a:
# https://go.example.com/admin/login
```

## ⚙️ Configuración

1. **SHORTENER_DOMAIN** (obligatorio) — Solo el hostname, ej. `go.example.com`
2. **SHORTENER_SCHEME** (default `https`) — Esquema para generar URLs
3. **DATABASE_URL** (obligatorio) — Armado por Compose en producción: `postgres://shortener:password@db:5432/shortener`
4. **SESSION_SECRET** (obligatorio, mínimo 32 chars Base64) — Secreto para firmar cookies de sesión
5. **SESSION_MAX_AGE_SECONDS** (default `604800`) — Duración de la sesión en segundos (7 días)
6. **SESSION_COOKIE_NAME** — Nombre de la cookie de sesión
7. **CSRF_COOKIE_NAME** — Nombre de la cookie CSRF
8. **REDIRECT_STATUS_CODE** (default `302`) — Código global de redirección (301, 302, 307, 308)
9. **ADMIN_TOKEN** (opcional, heredado) — Token para autenticación API legacy
10. **NODE_ENV** (production) — Entorno de ejecución

```bash
# Ejemplo .env.production
SHORTENER_DOMAIN=go.example.com
SHORTENER_SCHEME=https
DATABASE_URL=postgres://shortener:password@db:5432/shortener
SESSION_SECRET=base64_string_min_32_chars
SESSION_MAX_AGE_SECONDS=604800
REDIRECT_STATUS_CODE=302
NODE_ENV=production
```

## 🚀 Primeros pasos

1. **Acceder al panel administrativo** — Navegador a `https://<SHORTENER_DOMAIN>/admin/login` tras crear el usuario administrador
2. **Verificar health check** — Endpoint público `GET /healthz` (ej. `curl https://go.example.com/healthz`)
3. **Crear tu primer enlace corto** — Desde el panel o vía API JSON
4. **Probar redirección** — Visitar `https://<SHORTENER_DOMAIN>/<slug>` y confirmar redirección
5. **Revisar analíticas** — Panel muestra series temporales, países, referrers y clics recientes

## 💡 Casos de uso

- 🏷️ Acortador de URLs personal para marca o proyecto propio con dominio propio
- 📈 Campañas de marketing con seguimiento de clics, países y referrers
- 🤖 Automatización de creación y gestión de enlaces mediante API JSON
- 🔄 Reemplazo autohospedado de servicios acortadores de terceros manteniendo control total de datos
- 🛡️ Entornos donde se requiere privacidad: solo se almacena hash de IP, no la IP real
- 🧪 Desarrollo y pruebas de integraciones que requieren un servicio de redirección configurable

## 🔒 Acceso remoto seguro

El despliegue de producción incluye **Caddy** como reverse proxy que obtiene y renueva certificados **TLS de Let's Encrypt** automáticamente. No se requiere configuración manual de certificados. El tráfico HTTP (puerto 80) se redirige a HTTPS (puerto 443). La base de datos PostgreSQL solo es accesible desde la red interna de Docker.

## 🛠️ Gestión y mantenimiento

### Ver logs en tiempo real

```bash
docker compose --env-file .env.production -f compose.production.yml logs -f
```

### Actualizar a la última versión

```bash
git pull --ff-only
docker compose --env-file .env.production \
  -f compose.production.yml up --build -d
```

### Detener el stack sin borrar datos

```bash
docker compose --env-file .env.production \
  -f compose.production.yml down
```
> ⚠️ No usar `down -v` a menos que se quiera borrar todo (base de datos, certificados TLS).

### Backup de datos persistentes

Los datos viven en volúmenes Docker: `shortener_pgdata` (PostgreSQL), `shortener_caddy_data` y `shortener_caddy_config` (Caddy).

```bash
# Ejemplo backup volumen PostgreSQL
docker run --rm \
  -v shortener_pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata_$(date +%F).tar.gz /data
```

### Rotación de secretos de sesión

```bash
docker compose --env-file .env.production \
  -f compose.production.yml exec app \
  npm run rotate-credentials
```

## 📝 Licencia

Este proyecto está licenciado bajo la **Licencia MIT** — ver el archivo [LICENSE](https://github.com/pablokbs/link-shortener/blob/main/LICENSE) para más detalles.

---

> 📖 **Artículo original:** [Cómo instalar y configurar Link Shortener: Acortador de URLs Autohospedado en Docker](https://genbyte.blogspot.com/2026/10/link-shortener-acortador-de-urls.html)