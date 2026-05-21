> **Clasificación:** Documento Técnico Interno — Uso Restringido  
> **Versión:** 1.0.0  
> **Fecha de emisión:** 2025-05-15  
> **Elaborado por:** Equipo de Desarrollo DAW  
> **Norma de referencia:** ISO/IEC/IEEE 26514:2022 — *Systems and software engineering — Design and development of information for users*  
> **Estado:** Aprobado para producción

---

## Tabla de Contenidos

1. [Introducción y Arquitectura](#1-introducción-y-arquitectura)
2. [Guía de Instalación y Reinstalación](#2-guía-de-instalación-y-reinstalación)
3. [Seguridad y Control de Acceso](#3-seguridad-y-control-de-acceso)
4. [Procedimiento de Backup y Restauración](#4-procedimiento-de-backup-y-restauración)
5. [Flujo Operativo de Facturación e Informes](#5-flujo-operativo-de-facturación-e-informes)
6. [Resolución de Problemas (Troubleshooting)](#6-resolución-de-problemas)
7. [Glosario](#7-glosario)

---

## 1. Introducción y Arquitectura

### 1.1 Propósito del documento

Este Manual de Explotación proporciona la documentación técnica completa necesaria para que los **administradores de sistemas** y **usuarios finales** puedan instalar, configurar, operar y mantener el sistema ERP/CRM de **WillmanTech S.L.** De conformidad con los requisitos de calidad y usabilidad del estándar **ISO/IEC/IEEE 26514:2022**, el documento cubre los ciclos de vida operativo y de mantenimiento del sistema.

**Audiencia objetivo:**

El **Administrador de Sistemas** debe consultar principalmente las secciones 1, 2, 3 y 4. El **Desarrollador Backend/Frontend** se centrará en las secciones 1, 2 y 5. El **Usuario Contable o Comercial** encontrará su guía operativa en la sección 5. El **Responsable de Seguridad** deberá revisar en detalle las secciones 3 y 4.

### 1.2 Descripción del sistema

El sistema ERP/CRM de WillmanTech S.L. está basado en **Odoo 17 Community/Enterprise** y se despliega mediante contenedores Docker sobre infraestructura Linux (Ubuntu 22.04 LTS). La plataforma gestiona los procesos críticos de negocio:

- **CRM:** gestión de leads, oportunidades y pipeline comercial.
- **Ventas (Sale):** presupuestos, pedidos y confirmación de contratos.
- **Facturación y Contabilidad (Accounting/Invoicing):** facturación electrónica, gestión de cobros/pagos, conciliaciones bancarias.
- **Inventario (Stock):** control de almacén, movimientos y valoración de existencias.
- **RRHH (HR):** gestión de empleados, contratos y nóminas.
- **Portal Cliente:** acceso externo para consulta de facturas y pedidos.

### 1.3 Arquitectura lógica y topología Docker Compose

El entorno de producción se basa en una arquitectura de **tres capas** completamente contenerizada:

```
┌─────────────────────────────────────────────────────────┐
│                    CAPA DE PRESENTACIÓN                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │     Nginx Reverse Proxy (puerto 80/443)         │    │
│  │     SSL/TLS: certificado Let's Encrypt          │    │
│  └──────────────────────┬──────────────────────────┘    │
└─────────────────────────┼───────────────────────────────┘
                          │ HTTP upstream
┌─────────────────────────┼───────────────────────────────┐
│                    CAPA DE APLICACIÓN                   │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │     Odoo 17 App Server (puerto interno 8069)    │    │
│  │     Workers: 4 | Longpolling: 8072             │    │
│  └──────────────────────┬──────────────────────────┘    │
└─────────────────────────┼───────────────────────────────┘
                          │ PostgreSQL wire protocol
┌─────────────────────────┼───────────────────────────────┐
│                    CAPA DE DATOS                        │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │     PostgreSQL 15 (puerto interno 5432)         │    │
│  │     Volumen persistente: /var/lib/postgresql    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Servicios Docker definidos en `docker-compose.yml`:**

El servicio `db` usa la imagen `postgres:15`, expone el puerto 5432 únicamente en la red interna y actúa como SGBD relacional. El servicio `odoo` usa la imagen `odoo:17`, expone los puertos 8069 (aplicación) y 8072 (longpolling) en localhost y ejecuta el servidor de aplicación. El servicio `nginx` usa la imagen `nginx:stable-alpine`, expone los puertos 80 y 443 al exterior y actúa como proxy inverso con terminación SSL.

### 1.4 Módulos Odoo activados

```
Módulos instalados en la base de datos willmantech_prod:
├── base                     (núcleo del framework)
├── web                      (interfaz web)
├── mail                     (mensajería y chatter)
├── account                  (contabilidad y facturación)
│   └── account_edi          (facturación electrónica UBL/PEPPOL)
├── sale_management          (gestión de ventas)
├── crm                      (CRM y pipeline)
├── stock                    (inventario)
├── hr                       (recursos humanos)
├── portal                   (portal de clientes)
└── willmantech_invoice      (plantilla QWeb personalizada ← custom)
```

---

## 2. Guía de Instalación y Reinstalación

### 2.1 Prerrequisitos del sistema

**Hardware mínimo recomendado para producción:**

Se requiere como mínimo 2 vCPU (recomendado 4), 4 GB de RAM (recomendado 8 GB), 40 GB de disco SSD (recomendado 100 GB) y sistema operativo Ubuntu 22.04 LTS.

**Software requerido en el host:**

```bash
# Verificar versiones instaladas
docker --version          # Docker Engine >= 24.0
docker compose version    # Docker Compose >= 2.20
git --version             # Git >= 2.34
openssl version           # OpenSSL >= 3.0
```

### 2.2 Variables de entorno

Crear el archivo `.env` en el directorio raíz del proyecto **antes** de iniciar los contenedores. Este archivo **nunca debe versionarse en Git** (añadir a `.gitignore`).

```dotenv
# ─── BASE DE DATOS ───────────────────────────────────────
POSTGRES_DB=willmantech_prod
POSTGRES_USER=odoo_db_user
POSTGRES_PASSWORD=<contraseña_segura_min_20_caracteres>
POSTGRES_HOST=db
POSTGRES_PORT=5432

# ─── ODOO APPLICATION ────────────────────────────────────
ODOO_DB_NAME=willmantech_prod
ODOO_ADMIN_PASSWD=<master_password_segura>
ODOO_WORKERS=4
ODOO_MAX_CRON_THREADS=2
ODOO_LIMIT_TIME_CPU=600
ODOO_LIMIT_TIME_REAL=1200
ODOO_PROXY_MODE=True

# ─── NGINX / SSL ─────────────────────────────────────────
DOMAIN=erp.willmantech.es
CERTBOT_EMAIL=admin@willmantech.es

# ─── PATHS DE VOLÚMENES ──────────────────────────────────
ODOO_DATA_PATH=./volumes/odoo-data
ODOO_ADDONS_PATH=./volumes/odoo-addons
POSTGRES_DATA_PATH=./volumes/postgres-data
NGINX_CERTS_PATH=./volumes/nginx-certs
```

### 2.3 Estructura de directorios del proyecto

```
willmantech-erp/
├── .env                          ← Variables de entorno (NO versionar)
├── .gitignore
├── docker-compose.yml            ← Orquestación principal
├── docker-compose.override.yml   ← Overrides para desarrollo local
├── config/
│   ├── odoo.conf                 ← Configuración de Odoo
│   └── nginx/
│       ├── nginx.conf
│       └── willmantech.conf      ← VirtualHost con SSL
├── addons/
│   └── willmantech_invoice/      ← Módulo QWeb personalizado
│       ├── __manifest__.py
│       ├── report/
│       │   └── report_invoice_willmantech.xml
│       └── views/
├── volumes/                      ← Datos persistentes (NO versionar)
│   ├── odoo-data/
│   ├── odoo-addons/
│   ├── postgres-data/
│   └── nginx-certs/
└── scripts/
    ├── backup.sh                 ← Script de respaldo automatizado
    ├── restore.sh                ← Script de restauración
    └── init.sh                   ← Script de inicialización
```

### 2.4 Archivo `docker-compose.yml`

```yaml
version: "3.8"

services:

  db:
    image: postgres:15
    container_name: willmantech_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ${POSTGRES_DATA_PATH}:/var/lib/postgresql/data
    networks:
      - willmantech_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  odoo:
    image: odoo:17
    container_name: willmantech_odoo
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      HOST: ${POSTGRES_HOST}
      PORT: ${POSTGRES_PORT}
      USER: ${POSTGRES_USER}
      PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ${ODOO_DATA_PATH}:/var/lib/odoo
      - ./addons:/mnt/extra-addons
      - ./config/odoo.conf:/etc/odoo/odoo.conf:ro
    networks:
      - willmantech_net
    ports:
      - "127.0.0.1:8069:8069"
      - "127.0.0.1:8072:8072"

  nginx:
    image: nginx:stable-alpine
    container_name: willmantech_nginx
    restart: unless-stopped
    depends_on:
      - odoo
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx:/etc/nginx/conf.d:ro
      - ${NGINX_CERTS_PATH}:/etc/letsencrypt:ro
    networks:
      - willmantech_net

networks:
  willmantech_net:
    driver: bridge
```

### 2.5 Procedimiento de instalación desde cero

Seguir los pasos en el orden indicado. **No omitir ningún paso** bajo pena de inconsistencia del entorno:

```bash
git clone https://github.com/willmantech/erp-infrastructure.git
cd erp-infrastructure

cp .env.example .env
nano .env         
mkdir -p volumes/{odoo-data,odoo-addons,postgres-data,nginx-certs}

docker compose up -d --build

docker compose ps

docker compose logs -f odoo        

docker compose exec odoo odoo \
    --db_host=db \
    --db_user=${POSTGRES_USER} \
    --db_password=${POSTGRES_PASSWORD} \
    -d willmantech_prod \
    -i base,account,sale_management,crm,stock,willmantech_invoice \
    --stop-after-init

docker compose exec odoo odoo \
    -d willmantech_prod \
    -u willmantech_invoice \
    --stop-after-init
```

### 2.6 Procedimiento de actualización del sistema

```bash
docker compose pull

docker compose up -d --force-recreate

docker compose exec odoo odoo \
    -d willmantech_prod \
    -u willmantech_invoice \
    --stop-after-init
```

---

## 3. Seguridad y Control de Acceso

### 3.1 Modelo de roles y permisos

El sistema implementa un modelo de **control de acceso basado en roles (RBAC)** con tres perfiles principales, siguiendo el principio de mínimo privilegio.

El rol **Administrador** (`base.group_system`) tiene acceso total al sistema: configuración, módulos, base de datos y gestión de usuarios.

El rol **Contable** (`account.group_account_manager`) puede gestionar facturas, pagos, conciliaciones, informes financieros y exportación UBL, pero no tiene acceso a la configuración técnica ni a RRHH.

El rol **Comercial** (`sales_team.group_sale_salesman`) puede operar el CRM, crear presupuestos y gestionar pedidos y clientes, pero no tiene acceso a cuentas bancarias, datos financieros sensibles ni a la configuración del sistema.

#### 3.1.1 Configuración de roles en Odoo

```
Ajustes → Usuarios y Compañías → Usuarios → [Seleccionar usuario]
→ Pestaña "Derechos de Acceso"
→ Asignar el grupo correspondiente por módulo
```

**Permisos específicos recomendados:**

El **Administrador** tiene habilitados la configuración técnica, la gestión de módulos, el acceso a la base de datos, la gestión de usuarios y grupos, y el acceso completo a todos los módulos.

El **Contable** tiene habilitados el perfil de Administrador de Contabilidad, la lectura de pedidos de venta, la generación de informes PDF y la exportación de datos en formato UBL/CSV. Tiene deshabilitado el acceso a la configuración del sistema y al módulo de RRHH.

El **Comercial** tiene habilitados el perfil Comercial de CRM y Ventas, y la lectura del portal de clientes. Tiene deshabilitada la facturación directa (requiere validación del contable) y el acceso a datos financieros sensibles.

### 3.2 Política de contraseñas

Configurar en `Ajustes → Permisos → Política de Contraseñas`:

```
Longitud mínima:          12 caracteres
Mayúsculas obligatorias:  Sí (mínimo 1)
Minúsculas obligatorias:  Sí (mínimo 1)
Números obligatorios:     Sí (mínimo 1)
Caracteres especiales:    Sí (mínimo 1)
Caducidad de contraseña:  90 días
Historial de contraseñas: Últimas 5 no reutilizables
Bloqueo por intentos:     5 intentos fallidos → bloqueo 30 min
```

### 3.3 Seguridad de la infraestructura Docker

```bash
docker compose ps
ufw default deny incoming
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 3.4 Configuración HTTPS con Let's Encrypt

```bash
docker run --rm \
  -v ${NGINX_CERTS_PATH}:/etc/letsencrypt \
  -p 80:80 \
  certbot/certbot certonly \
  --standalone \
  --email ${CERTBOT_EMAIL} \
  --agree-tos \
  --no-eff-email \
  -d ${DOMAIN}

0 3 * * * docker run --rm -v /path/certs:/etc/letsencrypt certbot/certbot renew --quiet
```

### 3.5 Auditoría y registros

Odoo registra automáticamente en el **chatter** de cada registro todas las modificaciones. Para activar el log de auditoría técnico completo:

```
Ajustes → Técnico → Seguridad → Reglas de Acceso a Registros
→ Activar "Logging" en los modelos críticos:
   - account.move (facturas)
   - res.partner (clientes)
   - sale.order (pedidos)
```

---

## 4. Procedimiento de Backup y Restauración

### 4.1 Estrategia de backup

Se implementa una estrategia **3-2-1**:
- **3** copias de los datos
- **2** soportes/medios distintos (disco local + almacenamiento en la nube)
- **1** copia fuera de las instalaciones (offsite)

Se realizan cuatro tipos de copia: backup completo de la base de datos diariamente a las 03:00 con retención de 30 días en `/backups/daily/`; backup semanal los lunes a las 02:00 con retención de 12 semanas en `/backups/weekly/`; backup mensual el día 1 de cada mes con retención de 12 meses en almacenamiento cloud (S3/B2); y backup diario del filestore (adjuntos) con retención de 30 días en `/backups/filestore/`.

### 4.2 Backup de la base de datos PostgreSQL

```bash
#!/bin/bash
set -euo pipefail

source "$(dirname "$0")/../.env"

BACKUP_DIR="/opt/willmantech-erp/backups/daily"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/willmantech_prod_${TIMESTAMP}.sql.gz"
ODOO_DATA_DIR="/opt/willmantech-erp/volumes/odoo-data"
FILESTORE_BACKUP="${BACKUP_DIR}/filestore_${TIMESTAMP}.tar.gz"
RETENTION_DAYS=30

mkdir -p "${BACKUP_DIR}"

echo "[INFO] Iniciando backup — $(date)"

docker compose exec -T db pg_dump \
    --username="${POSTGRES_USER}" \
    --no-password \
    --format=custom \
    --compress=9 \
    --file="/tmp/backup_${TIMESTAMP}.dump" \
    "${POSTGRES_DB}"

docker compose cp "db:/tmp/backup_${TIMESTAMP}.dump" "${BACKUP_FILE%.gz}"
gzip -9 "${BACKUP_FILE%.gz}"

echo "[OK] Backup BD completado: ${BACKUP_FILE}"
tar -czf "${FILESTORE_BACKUP}" \
    -C "${ODOO_DATA_DIR}" \
    filestore/

echo "[OK] Backup filestore completado: ${FILESTORE_BACKUP}"

find "${BACKUP_DIR}" -name "*.gz" -mtime +${RETENTION_DAYS} -delete
echo "[OK] Backups anteriores a ${RETENTION_DAYS} días eliminados"

echo "[INFO] Backup finalizado correctamente — $(date)"
```

```bash
0 3 * * * /opt/willmantech-erp/scripts/backup.sh >> /var/log/willmantech-backup.log 2>&1
```

### 4.3 Restauración de la base de datos

**ADVERTENCIA:** La restauración sobreescribe completamente la base de datos existente. Ejecutar **solo** en un entorno de pruebas o tras confirmación del responsable de sistemas.

```bash
#!/bin/bash
set -euo pipefail
source "$(dirname "$0")/../.env"

BACKUP_FILE="${1:?ERROR: Debes indicar la ruta al archivo de backup}"

if [ ! -f "${BACKUP_FILE}" ]; then
    echo "[ERROR] Archivo no encontrado: ${BACKUP_FILE}"
    exit 1
fi

echo "[WARN] Esta operación DESTRUIRÁ la base de datos actual."
read -rp "¿Confirmas la restauración? (escribe 'CONFIRMAR'): " CONFIRM
[ "${CONFIRM}" != "CONFIRMAR" ] && { echo "Operación cancelada."; exit 0; }

echo "[INFO] Deteniendo el servidor Odoo..."
docker compose stop odoo

echo "[INFO] Eliminando la base de datos actual..."
docker compose exec -T db dropdb \
    --username="${POSTGRES_USER}" \
    --if-exists \
    "${POSTGRES_DB}"

echo "[INFO] Creando nueva base de datos vacía..."
docker compose exec -T db createdb \
    --username="${POSTGRES_USER}" \
    --owner="${POSTGRES_USER}" \
    "${POSTGRES_DB}"

echo "[INFO] Restaurando desde ${BACKUP_FILE}..."
gunzip -c "${BACKUP_FILE}" | docker compose exec -T db pg_restore \
    --username="${POSTGRES_USER}" \
    --dbname="${POSTGRES_DB}" \
    --no-password \
    --verbose \
    --exit-on-error

echo "[INFO] Reiniciando Odoo..."
docker compose start odoo

echo "[OK] Restauración completada exitosamente — $(date)"
```

### 4.4 Backup vía interfaz web de Odoo

Para backups ad hoc sin acceso a la línea de comandos:

```
1. Acceder a: https://erp.willmantech.es/web/database/manager
2. Introducir la Master Password (ODOO_ADMIN_PASSWD del .env)
3. Clic en "Backup" junto a "willmantech_prod"
4. Seleccionar formato: "zip" (incluye filestore) o "dump" (solo SQL)
5. Guardar el archivo descargado en ubicación segura
```

---

## 5. Flujo Operativo de Facturación e Informes

### 5.1 Ciclo de vida de una factura en Odoo

```
[LEAD/OPORTUNIDAD CRM]
        │
        ▼
[PRESUPUESTO DE VENTA]  ──── Comercial crea presupuesto
        │ Confirmar pedido
        ▼
[PEDIDO DE VENTA]       ──── Sistema genera orden confirmada
        │ Crear factura
        ▼
[FACTURA EN BORRADOR]   ──── Contable revisa y completa datos
        │ Validar (Confirmar)
        ▼
[FACTURA CONFIRMADA]    ──── Estado: "Publicada" / Número asignado
        │ Imprimir / Enviar
        ▼
[GENERACIÓN DEL PDF]    ──── Pipeline HTML → wkhtmltopdf → PDF
        │
        ▼
[ENVÍO AL CLIENTE]      ──── Email automático con adjunto PDF
        │ Registrar pago
        ▼
[FACTURA PAGADA]        ──── Estado: "En Pago" → "Pagado"
```

### 5.2 Procedimiento paso a paso: generación de factura

#### PASO 1 — Acceder al módulo de Facturación

```
Menú principal → Facturación → Clientes → Facturas → [+ Nuevo]
```

#### PASO 2 — Completar los campos obligatorios

Los campos a rellenar son: **Cliente** (seleccionar de la libreta de direcciones), **Fecha de Factura** (se rellena automáticamente, es editable), **Diario** (diario contable de ventas, por ejemplo "Facturas de clientes"), **Fecha de Vencimiento** (calculada automáticamente según las condiciones de pago) y **Condiciones de Pago** (política de pago acordada con el cliente, por ejemplo "30 días").

#### PASO 3 — Añadir líneas de factura

```
Sección "Líneas de Factura" → [Añadir un producto]
→ Seleccionar producto/servicio del catálogo
→ Ajustar cantidad, precio unitario y descuento si aplica
→ Verificar que los impuestos se asignan automáticamente
→ Repetir para cada línea
```

#### PASO 4 — Validar la factura

```
Botón "Confirmar" (esquina superior izquierda)
→ Odoo asigna automáticamente el número de secuencia (ej: WTECH/2025/00142)
→ El estado cambia de "Borrador" a "Publicada"
→ Los asientos contables se generan automáticamente
```

#### PASO 5 — Imprimir la factura en PDF

```
Botón "Imprimir" → "Factura" → El sistema lanza el pipeline de renderizado
```

### 5.3 Pipeline de generación de PDF: HTML → wkhtmltopdf → PDF

Este pipeline es el proceso interno por el cual Odoo convierte una plantilla de informe en un documento PDF descargable. Se describe a continuación de forma didáctica:

```
┌─────────────────────────────────────────────────────────────────────┐
│  FASE 1: COMPILACIÓN DE LA PLANTILLA QWeb (Motor Python/Lxml)       │
│                                                                     │
│  El servidor Odoo carga el archivo XML de la plantilla              │
│  (report_invoice_willmantech.xml) y el motor QWeb lo procesa:       │
│                                                                     │
│  1. Recupera el objeto `doc` (account.move) de la BBDD PostgreSQL   │
│  2. Evalúa las directivas:                                          │
│     · t-field → sustituye el campo por su valor formateado          │
│     · t-foreach → itera sobre invoice_line_ids y genera <tr>s       │
│     · t-if → evalúa has_discount; oculta la columna si es False     │
│  3. Produce un documento HTML completo y válido                     │
│                                                                     │
│  INPUT:  Template XML + Datos de la BBDD                            │
│  OUTPUT: Cadena HTML en memoria (UTF-8)                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTML string (en memoria)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FASE 2: RENDERIZADO A PDF (wkhtmltopdf)                            │
│                                                                     │
│  wkhtmltopdf es un binario de línea de comandos que incorpora el    │
│  motor de renderizado WebKit (el mismo que usa Chrome/Safari).      │
│                                                                     │
│  Odoo invoca wkhtmltopdf pasándole:                                 │
│  · El HTML generado en la Fase 1 (por stdin o archivo temporal)    │
│  · Parámetros de página: tamaño A4, márgenes, orientación           │
│  · Cabecera/pie de página si están configurados en la plantilla     │
│                                                                     │
│  Comando equivalente (simplificado):                                │
│  $ wkhtmltopdf \                                                    │
│      --page-size A4 \                                               │
│      --margin-top 10mm \                                            │
│      --margin-bottom 15mm \                                         │
│      --encoding utf-8 \                                             │
│      --enable-local-file-access \                                   │
│      input.html \                                                   │
│      output.pdf                                                     │
│                                                                     │
│  WebKit parsea el HTML, aplica los estilos CSS (Bootstrap 5 +       │
│  estilos corporativos) y vuelca el resultado como bytes PDF         │
│                                                                     │
│  INPUT:  HTML string                                                │
│  OUTPUT: Bytes PDF en memoria                                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ PDF bytes
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FASE 3: ENTREGA AL CLIENTE (HTTP Response)                         │
│                                                                     │
│  El servidor Odoo recibe los bytes PDF de wkhtmltopdf y los         │
│  devuelve al navegador del usuario con las cabeceras HTTP:          │
│                                                                     │
│  Content-Type: application/pdf                                      │
│  Content-Disposition: inline; filename="WTECH-2025-00142.pdf"      │
│                                                                     │
│  El navegador muestra el visor de PDF integrado o descarga el       │
│  archivo, según la configuración del usuario.                       │
│                                                                     │
│  INPUT:  PDF bytes                                                  │
│  OUTPUT: Archivo PDF descargado / visualizado en el navegador       │
└─────────────────────────────────────────────────────────────────────┘
```

**Verificar que wkhtmltopdf está disponible en el contenedor:**

```bash
docker compose exec odoo wkhtmltopdf --version
```

### 5.4 Exportación de facturas en formato UBL (Interoperabilidad)

Para exportar una factura validada al formato UBL 2.1 / PEPPOL compatible con la Agencia Tributaria:

```
1. Abrir la factura validada
2. Menú de acción → "Descargar XML"
   → El sistema genera el archivo invoice_ubl.xml
   → Alternativamente, usar la API REST de Odoo:

GET /api/account.move/{id}/download_edi_document
Authorization: Bearer <token>
```

---

## 6. Resolución de Problemas

**El PDF de factura no se genera.**
Causa probable: wkhtmltopdf no está instalado o está dañado.
Solución: `docker compose exec odoo apt-get install -y wkhtmltopdf`

**Error "Connection refused" al arrancar.**
Causa probable: PostgreSQL no ha terminado de iniciarse.
Solución: Esperar 30 segundos y relanzar con `docker compose restart odoo`.

**El módulo no aparece en la interfaz.**
Causa probable: No está instalado o hay un error en `__manifest__.py`.
Solución: Revisar `docker compose logs odoo | grep ERROR`.

**El backup falla con "permission denied".**
Causa probable: El script no tiene permisos de ejecución.
Solución: `chmod +x scripts/backup.sh`

**Error 502 Bad Gateway en Nginx.**
Causa probable: Odoo no está respondiendo.
Solución: Ejecutar `docker compose ps` para verificar el estado y revisar los logs del contenedor.

**Campos de factura aparecen vacíos en el PDF.**
Causa probable: Error en el `t-field` del XML QWeb.
Solución: Verificar el nombre exacto del campo en el modelo `account.move`.

---

## 7. Glosario

**Docker Compose:** herramienta para definir y gestionar aplicaciones multicontenedor Docker mediante un archivo YAML.

**ERP:** Enterprise Resource Planning. Sistema de gestión integrado de los recursos de una empresa.

**CRM:** Customer Relationship Management. Sistema de gestión de relaciones con clientes.

**QWeb:** motor de plantillas XML/HTML propio del framework Odoo, utilizado para generar informes PDF e interfaces web.

**UBL:** Universal Business Language (OASIS). Estándar internacional XML para documentos de negocio electrónicos.

**PEPPOL:** Pan-European Public Procurement OnLine. Red europea de intercambio de documentos electrónicos entre empresas y administraciones.

**wkhtmltopdf:** herramienta de línea de comandos que convierte documentos HTML a PDF usando el motor WebKit.

**SGBD:** Sistema Gestor de Bases de Datos. En este contexto, PostgreSQL 15.

**Filestore:** almacén de archivos binarios adjuntos de Odoo (imágenes, documentos PDF, etc.), ubicado fuera de la base de datos.

**ISO/IEC/IEEE 26514:** estándar internacional para el diseño y desarrollo de documentación de usuario en sistemas y software.

**RBAC:** Role-Based Access Control. Modelo de control de acceso basado en roles de usuario.