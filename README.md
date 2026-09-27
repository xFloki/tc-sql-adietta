# tc-sql-adietta

Práctica Obligatoria — SQL y Bases de Datos (Team Challenge SQL, The Bridge).

La práctica tiene dos partes independientes:

1. **SQL Murder Mystery:** resolver un caso de asesinato con queries SQL sobre una base de datos SQLite.
2. **Modelo de datos en BigQuery:** diseñar, implementar y poblar una base de datos normalizada (3NF) para un e-commerce de tecnología.

## Estructura del repositorio

```
tc-sql-adietta/
├── parte_1_sql_murder_mystery/
│   ├── data/sql-murder-mystery.db        # Base de datos del juego (SQLite)
│   └── investigacion.ipynb               # Resolución del caso
├── parte_2_modelo_bigquery/
│   ├── data/                             # Vacío: los datos están en BigQuery
│   ├── docs/
│   │   ├── er_diagram.png                # Diagrama entidad-relación
│   │   ├── er_diagram.dbml               # Código fuente del diagrama (dbdiagram.io)
│   │   └── normalizacion.md              # Modelo y justificación de 3NF
│   └── notebooks/
│       ├── 01_setup_bigquery.ipynb       # Crea el dataset y las tablas
│       ├── 02_generate_data.ipynb        # Genera los datos con Faker y los carga
│       └── 03_queries_verification.ipynb # Queries analíticas
├── .env.example                          # Plantilla de variables de entorno
├── .gitignore
├── README.md
└── requirements.txt
```

## Parte I — SQL Murder Mystery

El notebook [`investigacion.ipynb`](parte_1_sql_murder_mystery/investigacion.ipynb) reproduce la investigación del juego [SQL Murder Mystery](http://mystery.knightlab.com) (Joon Park y Cathy He) paso a paso, atacando la base de datos local `data/sql-murder-mystery.db`. No necesita credenciales.

**Solución:** el asesino es **Jeremy Bowers**, contratado por **Miranda Priestly**.

## Parte II — Modelo de datos en BigQuery

**Smart Solutions Dietta** es un e-commerce de productos tecnológicos que vende en varios países de Europa. El modelo tiene 7 tablas: `customers`, `categories`, `products`, `orders`, `order_items`, `payments` y `reviews`.

- Diagrama ER: [`docs/er_diagram.png`](parte_2_modelo_bigquery/docs/er_diagram.png)
- Modelo y justificación de la normalización (3NF): [`docs/normalizacion.md`](parte_2_modelo_bigquery/docs/normalizacion.md)

## Setup

### 1. Clonar el repositorio

```bash
git clone https://github.com/xFloki/tc-sql-adietta.git
cd tc-sql-adietta
```

### 2. Entorno virtual y dependencias

```bash
# Crear el entorno virtual
python -m venv venv

# Activarlo
source venv/bin/activate        # Mac/Linux
venv\Scripts\activate           # Windows (CMD / PowerShell)
source venv/Scripts/activate    # Windows (Git Bash)

# Instalar dependencias
pip install -r requirements.txt
```

En VS Code, seleccionar el entorno `venv` como kernel de los notebooks.

### 3. Credenciales de Google Cloud (solo Parte II)

1. Crear un proyecto en [Google Cloud](https://console.cloud.google.com) y activar la **BigQuery API**.
2. En *IAM & Admin → Service Accounts*, crear una service account con el rol **BigQuery Admin**.
3. Crear una clave **JSON** para la service account y guardarla como `credentials/service-account.json` en la raíz del repositorio.

### 4. Variables de entorno

Copiar `.env.example` como `.env` y rellenar los valores:

```
GCP_PROJECT_ID=tu-proyecto-gcp
BQ_DATASET_ID=smart_solutions_dietta
GOOGLE_APPLICATION_CREDENTIALS=./credentials/service-account.json
```

El fichero `.env` y la carpeta `credentials/` están en `.gitignore` y **nunca** se suben al repositorio.

## Ejecución de la Parte II

Ejecutar los notebooks de `parte_2_modelo_bigquery/notebooks/` en orden:

1. **`01_setup_bigquery.ipynb`:** crea el dataset (región EU) y las 7 tablas.
2. **`02_generate_data.ipynb`:** genera los datos sintéticos con Faker y los carga en BigQuery, validando cada carga.
3. **`03_queries_verification.ipynb`:** ejecuta las queries analíticas.

Los datos se generan con una semilla fija, así que cada ejecución produce los mismos datos. La carga sobrescribe las tablas, por lo que se puede repetir sin duplicar filas.

| Tabla | Filas |
|---|---|
| categories | 8 |
| customers | 500 |
| products | 70 |
| orders | 2000 |
| order_items | 4538 |
| payments | 2000 |
| reviews | 1115 |
