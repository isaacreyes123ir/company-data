# 🏛️ Plataforma de Inteligencia Societaria Ecuador

> **API unificada + Dashboard interactivo** para consultar perfil corporativo, compras públicas, historial financiero y actividad económica de empresas ecuatorianas.

![Arquitectura](https://img.shields.io/badge/Arquitectura-ETL%20%2B%20API%20%2B%20Frontend-blue)
![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-0.109-009688?logo=fastapi)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql)
![Chart.js](https://img.shields.io/badge/Chart.js-4.4-FF6384?logo=chart.js)
![AWS](https://img.shields.io/badge/AWS-S3%20%2B%20RDS-FF9900?logo=amazon-aws)

---

## 📋 Tabla de Contenidos

- [Visión General](#-visión-general)
- [Arquitectura](#-arquitectura)
- [Fuentes de Datos](#-fuentes-de-datos)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Requisitos Previos](#-requisitos-previos)
- [Configuración](#-configuración)
- [Despliegue](#-despliegue)
- [Uso de la API](#-uso-de-la-api)
- [Pipelines ETL](#-pipelines-etl)
- [Frontend](#-frontend)
- [Seguridad](#-seguridad)
- [Roadmap](#-roadmap)
- [Licencia](#-licencia)

---

## 🎯 Visión General

Este proyecto consolida **cuatro fuentes oficiales ecuatorianas** en una sola plataforma consultable:

| Fuente | Qué aporta | Tabla destino |
|--------|------------|---------------|
| **SRI - Catastro Nacional** | RUC, razón social, actividad económica, ubicación, representante | `sri_catastro_nacional` |
| **SuperCías - Directorio** | Expediente, tipo sociedad, capital, situación legal, CIIU | `directorio_completo` |
| **SERCOP - Compras Públicas** | Contratos como proveedor del Estado (2015–presente) | `compras_publicas_sercop` |
| **SuperCías - Financieros** | Balances y resultados anuales (histórico completo) | `financieros_historial_completo` |

**Resultado:** Un dashboard que, dado un RUC o nombre, muestra en segundos la radiografía completa de cualquier empresa.

---

## 🏗️ Arquitectura

```
┌──────────────────┐     ┌──────────────────┐     ┌───────────────┐      ┌──────────────┐     ┌────────────────┐
│FUENTES OFICIALES │───▶│  PROCESADORES    │────▶│     AWS S3    │────▶│  INYECTORES  │────▶│  RDS POSTGRES  │
│  (SRI, SERCOP,   │     │  (5 scripts Py)  │     │  (CSV/Parquet)│      │ (Blue/Green) │     │   (4 tablas)   │
│   SuperCías)     │     │                  │     │               │      │              │     │                │
└──────────────────┘     └──────────────────┘     └───────────────┘      └──────────────┘     └────────────────┘
                                                                                                      │
                                                                                                      ▼
                                                                                              ┌─────────────────┐
                                                                                              │  API FASTAPI    │
                                                                                              │  (api.py)       │
                                                                                              └─────────────────┘
                                                                                                      │
                                                                                                      ▼
                                                                                              ┌─────────────────┐
                                                                                              │  FRONTEND HTML  │
                                                                                              │ (company-data)  │
                                                                                              └─────────────────┘
```

### Patrones clave

- **Blue/Green Deployment** en RDS: `CREATE TABLE _clon (LIKE original INCLUDING ALL)` → `COPY` → `RENAME` atómico → `ANALYZE`. Cero downtime.
- **Chunking streaming** (50k filas): Procesa archivos de GB sin OOM.
- **BFF (Backend For Frontend)**: Un solo endpoint `/perfil/{ruc}` une SRI + Directorio.
- **Cálculo DuPont en cliente**: Métricas financieras derivadas en el navegador (sin almacenar ratios).

---

## 📊 Fuentes de Datos

| Script | Fuente | Frecuencia | Formato | Notas |
|--------|--------|------------|---------|-------|
| `procesador_sri_nacional.py` | `descargas.sri.gob.ec` | Mensual | 24 CSV/zip (pipe `|`) | Une provincias; limpia filas rotas |
| `procesador_supercias.py` | `mercadodevalores.supercias.gob.ec` | Diario | XLSX (skiprows=4) | Directorio de compañías |
| `automatizador_sercop.py` | `datosabiertos.compraspublicas.gob.ec` | Semanal (2015+) | ZIP (3 CSV OCDS) | Une contracts+suppliers+tender |
| `procesador_financiero.py` | `appscvsmovil.supercias.gob.ec` | Diario | CSV (bi_ranking) | Histórico completo anual |

---

## 📁 Estructura del Proyecto

```
isaac/
├── api.py                      # FastAPI REST API (4 endpoints)
├── company-data.html           # Dashboard SPA (HTML+JS+Chart.js)
├── automatizador_sercop.py     # ETL SERCOP → S3 (consolida años)
├── inyeccion_sercop_rds.py     # Blue/Green: S3 → RDS (compras_publicas_sercop)
├── inyeccion_sri_rds.py        # Blue/Green: S3 → RDS (sri_catastro_nacional)
├── inyector_directorio.py      # Blue/Green: S3 → RDS (directorio_completo)
├── inyector_financieros.py     # Blue/Green: S3 → RDS (financieros_historial_completo)
├── procesador_financiero.py    # ETL Financieros SuperCías → S3
├── procesador_sri_nacional.py  # ETL Catastro SRI (24 provincias) → S3
├── procesador_supercias.py     # ETL Directorio SuperCías → S3
├── .env                        # NO COMMITEAR - ver .env.example
├── .env.example                # Plantilla de variables de entorno
├── requirements.txt            # Dependencias Python
└── README.md                   # Este archivo
```



https://isaacreyes123ir.github.io/company-data/company-data.html
