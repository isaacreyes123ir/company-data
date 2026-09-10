# company-data

┌─────────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐     ┌─────────────┐
│  FUENTES OFICIALES  │────▶│  PROCESADORES  │────▶│   AWS S3    │────▶│  INYECTORES │────▶│  RDS POSTGRES  │
│  (SRI, SERCOP,  │     │  (5 scripts) │     │(CSV/Parquet)│     │(Blue/Green)│   │(4 tablas)   │
│   SuperCías)    │     │              │     │             │     │          │     │             │
└─────────────────┘     └──────────────┘     └─────────────┘     └──────────┘     └─────────────┘
                                                                                          │
                                                                                          ▼
                                                                                  ┌───────────────┐
                                                                                  │   API FASTAPI │
                                                                                  │   (api.py)    │
                                                                                  └───────────────┘
                                                                                          │
                                                                                          ▼
                                                                                  ┌───────────────┐
                                                                                  │  FRONTEND HTML│
                                                                                  │ (company-data)│
                                                                                  └───────────────┘

---


https://isaacreyes123ir.github.io/company-data/company-data.html
