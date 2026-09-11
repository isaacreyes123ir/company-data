# 🏛️ Plataforma Societaria — Inteligencia Corporativa Ecuador

> **API + Dashboard** para consultar información societaria, tributaria, contractual y financiera de empresas ecuatorianas en tiempo real.

---

## 📸 Demo

| Dashboard | Detalle Financiero (DuPont + 6 Gráficos) | Compras Públicas |
|-----------|------------------------------------------|------------------|
| ![Dashboard](https://via.placeholder.com/400x200/2c2c2c/d9f95d?text=Dashboard+Preview) | ![Finanzas](https://via.placeholder.com/400x200/2c2c2c/d9f95d?text=Modelo+DuPont) | ![SERCOP](https://via.placeholder.com/400x200/2c2c2c/d9f95d?text=Contratos+Estado) |

> **Live:** [company-data.duckdns.org](https://company-data.duckdns.org) (GitHub Pages + API en EC2)

---

## 🗂️ Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ETL PIPELINE (Diario / Semanal)                     │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────────────────┤
│   SRI       │  SuperCías  │  SERCOP     │  SuperCías  │  Resultado          │
│ Catastro    │ Directorio  │ Compras     │ Ranking     │  4 CSV en S3        │
│ Nacional    │ Compañías   │ Públicas    │ Financiero  │  (particionados)    │
└──────┬──────┴──────┬──────┴──────┬──────┴──────┬──────┴──────────┬─────────┘
       ▼             ▼             ▼             ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INYECCIÓN BLUE/GREEN (Zero Downtime)                      │
│  S3 → EC2 → COPY a tabla CLON → SWAP atómico (RENAME) → ANALYZE → DROP old  │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API LAYER (FastAPI)                             │
│  • /buscar?q=...           → Autocompletado RUC/Razón Social                │
│  • /perfil/{ruc}           → Ficha maestra (SRI + Directorio)               │
│  • /compras-publicas/{ruc} → Historial contratos como proveedor             │
│  • /finanzas/{expediente}  → Serie histórica + ratios DuPont/liquidez/solv. │
│  • Auth: API Key (Header X-API-Key)  • CORS restringido  • Healthcheck      │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND (GitHub Pages)                            │
│  • HTML/JS/Chart.js estático  • Glassmorphism UI  • Carga paralela          │
│  • Autocompletado debounced  • Modelo DuPont en vivo  • 6 gráficos          │
│  • Modal SERCOP agrupado por año/entidad                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Fuentes de Datos

| Fuente | Qué aporta | Frecuencia | Tamaño aprox. |
|--------|------------|------------|---------------|
| **SRI Catastro Nacional** | RUC, razón social, CIIU, provincia/cantón/parroquia, estado contribuyente, obligado contabilidad, fecha inicio | Mensual | ~4.5M registros |
| **SuperCías Directorio** | Expediente, representante legal, capital suscrito, teléfono, dirección completa, situación legal, fecha constitución | Mensual | ~500k registros |
| **SERCOP Compras Públicas** | OCID, monto, moneda, fecha firma, entidad compradora, objeto, tipo contrato, categoría (Bienes/Servicios/Obras) | Anual (2015–2026) | ~2M contratos |
| **SuperCías Ranking Financiero** | Estados financieros completos + 35+ ratios calculados (DuPont, liquidez, solvencia, gestión, rotaciones) | Anual | ~1.2M registros |

---

## 🚀 Inicio Rápido

### Prerrequisitos
- Python 3.11+
- PostgreSQL 16+ (RDS o local)
- AWS CLI configurado (para S3)
- Cuenta AWS con bucket S3

### 1. Clonar y configurar entorno
```bash
git clone https://github.com/tu-usuario/plataforma-societaria.git
cd plataforma-societaria
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Variables de entorno
```bash
cp .env.example .env
# Edita .env con tus credenciales reales
```

```env
# .env
RDS_BD=nombre_base
RDS_USUARIO=usuario_bd
RDS_PASSWORD=tu_password_seguro
RDS_ENDPOINT=tu-instancia.rds.amazonaws.com
RDS_PUERTO=5432
API_SECRET_KEY=generar_clave_aleatoria_64_chars
S3_BUCKET_NAME=api-empresas-ecuador-datos
```

### 3. Crear esquema en PostgreSQL
```bash
psql -h $RDS_ENDPOINT -U $RDS_USUARIO -d $RDS_BD -f schema.sql
```

### 4. Ejecutar ETL completo (orden recomendado)
```bash
# 1. SRI Catastro Nacional (~15-30 min)
python procesador_sri_nacional.py

# 2. SuperCías Directorio (~5 min)
python procesador_supercias.py

# 3. SERCOP Compras Públicas (~30-60 min según años)
python automatizador_sercop.py

# 4. SuperCías Ranking Financiero (~10 min)
python procesador_financiero.py
```

### 5. Inyectar a RDS (Blue/Green swap)
```bash
# Orden independiente, cada uno ~2-5 min
python inyeccion_sri_rds.py
python inyector_directorio.py
python inyeccion_sercop_rds.py
python inyector_financieros.py
```

### 6. Levantar API local
```bash
uvicorn api:app --reload --port 8000
# Docs: http://localhost:8000/docs
```

### 7. Frontend (GitHub Pages)
```bash
# Solo sube company-data.html a tu repo GitHub Pages
# O sirve localmente:
npx serve .
```

---

## 📁 Estructura del Proyecto

```
plataforma-societaria/
├── 📄 README.md                    # Este archivo
├── 📄 requirements.txt             # Dependencias Python (pip-compile)
├── 📄 .env.example                 # Template de variables de entorno
├── 📄 .gitignore                   # Ignora .env, __pycache__, CSV, ZIP
├── 📄 schema.sql                   # DDL PostgreSQL (tablas + índices optimizados)
├── 📄 openapi.json                 # Spec OpenAPI 3.1 generada
├── 📄 api.py                       # FastAPI app (endpoints + auth + CORS)
├── 📄 company-data.html            # Dashboard estático (Chart.js + Glassmorphism)
│
├── 🔄 ETL - EXTRACCIÓN Y TRANSFORMACIÓN
│   ├── procesador_sri_nacional.py      # Descarga 24 provincias → 1 CSV maestro (pipe-delimited)
│   ├── procesador_supercias.py         # Descarga Excel Directorio → CSV limpio
│   ├── automatizador_sercop.py         # Descarga ZIPs anuales → cruce contracts/suppliers/tender → CSV
│   └── procesador_financiero.py        # Descarga CSV ranking → limpieza + fecha auditoría → CSV
│
├── 💉 INYECCIÓN BLUE/GREEN A RDS
│   ├── inyeccion_sri_rds.py            # Limpieza comillas/filas rotas + COPY + SWAP + ANALYZE
│   ├── inyector_directorio.py          # COPY CSV + SWAP atómico + ANALYZE
│   ├── inyeccion_sercop_rds.py         # SET datestyle DMY + COPY + SWAP + ANALYZE
│   └── inyector_financieros.py         # SET datestyle YMD + COPY + SWAP + ANALYZE
│
└── 📎 OTROS
    └── EJEMPLO - HOLCIM ECUADOR S.A..pdf   # Ejemplo de reporte (revisar si sensible)
```

---

## 🔐 Seguridad

| Capa | Implementación |
|------|----------------|
| **API Key** | Header `X-API-Key` validado en middleware (FastAPI `Security`) |
| **CORS** | Restringido a dominio GitHub Pages configurable por ENV |
| **Secrets** | `.env` en EC2 (no en repo). `.env.example` como plantilla |
| **DB** | RDS en VPC privada, Security Group solo desde EC2 API |
| **S3** | Bucket versionado, bloqueo acceso público, lifecycle rules |
| **⚠️ Pendiente** | Rate limiting, validación Pydantic estricta, proxy para API key en frontend |

> **⚠️ IMPORTANTE:** El archivo `company-data.html` **tiene la API key hardcodeada** (línea 237). **No usar en producción así**. Ver sección [Despliegue Frontend Seguro](#-despliegue-frontend-seguro).

---

## 🛠️ Endpoints API

| Método | Ruta | Descripción | Parámetros |
|--------|------|-------------|------------|
| `GET` | `/health` | Healthcheck sin auth (para ALB/CloudWatch) | — |
| `GET` | `/api/v1/buscar` | Autocompletado empresa | `q` (query, min 3 chars) |
| `GET` | `/api/v1/perfil/{ruc}` | Ficha maestra SRI + Directorio | `ruc` (path, 13 dígitos) |
| `GET` | `/api/v1/compras-publicas/{ruc}` | Contratos como proveedor | `ruc` (path) |
| `GET` | `/api/v1/finanzas/{expediente}` | Histórico financiero + ratios | `expediente` (path) |

**Auth:** Header `X-API-Key: <tu_clave>`

**Ejemplo:**
```bash
curl -H "X-API-Key: TU_CLAVE" \
  "https://company-data.duckdns.org/api/v1/perfil/1791234567001"
```

---

## 🎨 Frontend — Características

- **Glassmorphism UI** con backdrop-filter, gradientes aurora, badges neón
- **Autocompletado** debounced (300ms) contra `/buscar`
- **Carga paralela**: Perfil inmediato → Finanzas/SERCOP en background
- **Modelo DuPont en vivo**: Margen × Rotación × Apalancamiento = ROE
- **6 Gráficos Chart.js**:
  1. Evolución Ingresos vs Utilidad (eje dual)
  2. Márgenes Bruto/Operacional/Neto (recorte inteligente eje X)
  3. Estructura Capital (stacked bar 100%)
  4. Liquidez Corriente + Prueba Ácida
  5. Solvencia: Endeudamiento Activo % + Patrimonial (eje dual)
  6. Gestión: Rotación Cartera / Activo Fijo / Ventas
- **Modal SERCOP**: Agrupado por año + entidad, montos formateados USD

---

## 📈 Modelo de Datos (Resumen)

### `directorio_completo` (SuperCías)
```sql
ruc, expediente, nombre, situacion_legal, fecha_constitucion, tipo,
pais, region, provincia, canton, ciudad, calle, numero, interseccion,
barrio, telefono, representante, cargo, capital_suscrito,
ciiu_nivel_1, ciiu_nivel_6, ultimo_balance, ...
-- 18 índices (btree + gin_trgm para búsqueda fuzzy)
```

### `sri_catastro_nacional` (SRI)
```sql
numero_ruc, razon_social, codigo_jurisdiccion, estado_contribuyente,
clase_contribuyente, fecha_inicio_actividades, obligado,
tipo_contribuyente, numero_establecimiento, nombre_fantasia_comercial,
codigo_ciiu, actividad_economica, agente_retencion, ...
-- 10 índices (gin_trgm en ruc y razon_social)
```

### `compras_publicas_sercop` (SERCOP)
```sql
ocid, estado_contrato, monto_contrato, moneda, fecha_firma,
ruc_proveedor_limpio, tipo_proveedor, nombre_proveedor,
ruc_comprador_limpio, entidad_compradora, objeto_contrato,
tipo_contrato, categoria_compra, fecha_actualizacion_estado
-- 3 índices (fecha_firma, ruc_proveedor, ruc_comprador)
```

### `financieros_historial_completo` (SuperCías Ranking)
```sql
anio, expediente, posicion_general, cia_imvalores,
ingresos_ventas, activos, patrimonio, utilidad_neta, n_empleados,
-- 35+ ratios: liquidez_corriente, prueba_acida, end_activo, roe, roa,
-- rot_cartera, rot_activo_fijo, rot_ventas, cobertura_interes, ...
-- 12 índices compuestos para queries analíticas
```

---

## ⚙️ Detalles Técnicos Clave

### Blue/Green Swap (Zero Downtime)
```sql
-- 1. Crear clon con MISMOS índices/constraints
CREATE TABLE tabla_clon (LIKE tabla_original INCLUDING ALL);

-- 2. COPY masivo (psycopg2 copy_expert) → ~50k rows/sec
COPY tabla_clon FROM STDIN WITH CSV HEADER DELIMITER ',' ENCODING 'UTF8';

-- 3. Swap atómico (milisegundos, sin bloquear lectores)
BEGIN;
DROP TABLE IF EXISTS tabla_vieja;
ALTER TABLE tabla_original RENAME TO tabla_vieja;
ALTER TABLE tabla_clon RENAME TO tabla_original;
COMMIT;

-- 4. Actualizar estadísticas del planner
ANALYZE tabla_original;

-- 5. Limpiar
DROP TABLE tabla_vieja;
```

### Chunking Pandas (Anti-OOM)
```python
for chunk in pd.read_csv(archivo, chunksize=50000, dtype=str):
    chunk.columns = chunk.columns.str.strip().str.replace(' ', '_').str.upper()
    chunk = chunk.where(pd.notnull(chunk), None)
    chunk.to_csv(salida, mode='a', header=primero, index=False)
    del chunk
    gc.collect()
```

### Limpieza SERCOP (filas rotas + comillas)
```python
# Regex detecta inicio de registro válido (RUC de 13 dígitos + pipe)
regex_ruc = re.compile(r'^\d{13}\|')

# Solda líneas rotas (descripciones con saltos de línea)
if regex_ruc.match(linea):
    # Escribir registro anterior y empezar nuevo
else:
    # Concatenar al registro actual
```

---

## 🌐 Despliegue Frontend Seguro

**Problema:** GitHub Pages es estático → no puede ocultar la API key.

**Solución recomendada: Cloudflare Workers (gratis, edge, 100k req/día)**

```javascript
// worker.js - Despliega en Cloudflare Workers
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Solo proxy las llamadas a /api/
    if (url.pathname.startsWith('/api/')) {
      const apiUrl = `https://company-data.duckdns.org${url.pathname}${url.search}`;
      
      const response = await fetch(apiUrl, {
        method: request.method,
        headers: {
          'X-API-Key': env.API_SECRET_KEY,  // Secret en Cloudflare Dashboard
          'Content-Type': 'application/json',
        },
      });
      
      // CORS para tu GitHub Pages
      const newHeaders = new Headers(response.headers);
      newHeaders.set('Access-Control-Allow-Origin', 'https://isaacreyes123ir.github.io');
      newHeaders.set('Access-Control-Allow-Methods', 'GET, OPTIONS');
      newHeaders.set('Access-Control-Allow-Headers', 'Content-Type');
      
      return new Response(response.body, {
        status: response.status,
        headers: newHeaders,
      });
    }
    
    // Sirve el HTML estático desde KV o Assets
    return env.ASSETS.fetch(request);
  },
};
```

**Alternativas:**
- **Netlify Functions** / **Vercel Edge Functions** (similar)
- **API Gateway + Lambda** (AWS nativo)
- **NGINX en EC2** con `proxy_set_header X-API-Key $api_key;`

Luego en `company-data.html`:
```javascript
// Cambia API_BASE a tu worker
const API_BASE = "https://tu-worker.workers.dev/api/v1";
// ELIMINA const API_KEY y fetchOptions (el worker la inyecta)
```

---

## 🧪 Testing

```bash
# Instalar deps de test
pip install pytest pytest-asyncio httpx

# Ejecutar tests (cuando existan)
pytest tests/ -v --cov=api --cov=procesador_*
```

**Tests sugeridos:**
- `test_api.py`: Healthcheck, auth, 404, 422, response schema
- `test_procesadores.py`: Columnas esperadas, tipos, nulos, RUCs válidos
- `test_inyectores.py`: Mock psycopg2, verificar COPY + SWAP + ANALYZE

---

## 📦 CI/CD (GitHub Actions Sugerido)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: [5432:5432]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt pytest pytest-asyncio httpx
      - run: pytest -v
  
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff mypy
      - run: ruff check . && mypy api.py
```

---

## ⏰ Cron / Orquestación (Producción)

El archivo [`crontab.txt`](crontab.txt) define la ejecución automática en el servidor EC2 (Ubuntu, usuario `ubuntu`, venv en `/home/ubuntu/escudo_datos`):

### Fase 1: ETL → Data Lake (S3)

| Tarea | Frecuencia | Hora (UTC) | Script | Log |
|-------|------------|------------|--------|-----|
| Directorio SuperCías | Diario | 03:00 | `procesador_supercias.py` | `log_directorio.txt` |
| Estados Financieros | Diario | 03:15 | `procesador_financiero.py` | `log_financiero.txt` |
| SERCOP Contratos | Domingos | 03:30 | `automatizador_sercop.py` | `log_sercop.txt` |
| SRI Catastro Nacional | Día 16/mes | 04:00 | `procesador_sri_nacional.py` | `log_sri.txt` |

### Fase 2: Inyección Blue/Green → RDS

| Tarea | Frecuencia | Hora (UTC) | Script | Log |
|-------|------------|------------|--------|-----|
| Directorio SuperCías | Diario | 03:10 | `inyector_directorio.py` | `log_inyeccion_directorio.txt` |
| Estados Financieros | Diario | 03:20 | `inyector_financieros.py` | `log_inyeccion_financieros.txt` |
| SERCOP Contratos | Domingos | 04:30 | `inyeccion_sercop_rds.py` | `log_inyeccion_sercop.txt` |
| SRI Catastro Nacional | Día 16/mes | 05:30 | `inyeccion_sri_rds.py` | `log_inyeccion_sri.txt` |

**Orden temporal garantizado:** Cada inyección corre **después** de su ETL correspondiente (buffer 10-60 min).
Los domingos SERCOP tiene ventana dedicada (03:30→04:30). El día 16 SRI tiene ventana dedicada (04:00→05:30).

### Instalar en crontab del servidor

```bash
# En el EC2 (usuario ubuntu)
crontab crontab.txt

# Verificar
crontab -l

# Ver logs en tiempo real
tail -f /home/ubuntu/log_*.txt
```

---

## 🐳 Docker (Opcional)

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY api.py .
EXPOSE 8000
CMD ["uvicorn", "api:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t plataforma-societaria-api .
docker run -d -p 8000:8000 --env-file .env plataforma-societaria-api
```

---

## 🗺️ Roadmap / Próximos Pasos

- [ ] **Rate limiting** (slowapi + Redis)
- [ ] **Validación Pydantic** en todos los endpoints
- [ ] **Proxy serverless** para API key (Cloudflare Workers)
- [ ] **Tests unitarios + integración** (pytest)
- [ ] **CI/CD** GitHub Actions
- [ ] **Docker + Docker Compose** para dev local
- [ ] **Métricas** (Prometheus `/metrics` + Grafana)
- [ ] **Logs estructurados** (structlog + JSON)
- [ ] **Documentación OpenAPI** en `/docs` (ya está en FastAPI)
- [ ] **Cache Redis** para `/buscar` y `/perfil`
- [ ] **Particionar tablas grandes** por año (pg_partman)

---

## 🤝 Contribuir

1. Fork el repo
2. Crea branch: `git checkout -b feature/mi-mejora`
3. Commit: `git commit -m 'feat: descripción clara'`
4. Push: `git push origin feature/mi-mejora`
5. Abre Pull Request

**Convenciones:**
- Commits: [Conventional Commits](https://www.conventionalcommits.org/)
- Code style: `ruff` + `black` (config en `pyproject.toml` futuro)
- Type hints obligatorios en código nuevo

---

## 📄 Licencia

**MIT License** — Úsalo libremente, pero **no subas credenciales reales**.

```
MIT License

Copyright (c) 2025 Isaac Reyes

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 🙏 Agradecimientos

- **SRI Ecuador** — Datos abiertos catastro nacional
- **Superintendencia de Compañías** — Directorio + Ranking financiero
- **SERCOP** — Portal de compras públicas OCDS
- **FastAPI / Pydantic / Starlette** — Framework API moderno
- **Chart.js** — Gráficos reactivos
- **psycopg2 / pandas / boto3** — Data engineering stack

---

> **¿Preguntas?** Abre un *Issue* o revisa la [documentación de la API](https://company-data.duckdns.org/docs) (Swagger UI).


https://isaacreyes123ir.github.io/company-data/company-data.html
