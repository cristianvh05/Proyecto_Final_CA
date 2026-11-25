# Proyecto final — Pronóstico de consumo eléctrico con SARIMA

**Computación Avanzada (ET0197) · Institución Universitaria Pascual Bravo**

## Video de sustentación

**[Ver demostración del proyecto en YouTube](https://youtu.be/HRkBc9Et_hk)**

El video presenta la aplicación con datos reales, la ejecución sin Docker y su funcionamiento dentro de un contenedor Docker.

## Descripción del proyecto

Este proyecto implementa una aplicación web que consulta registros históricos de consumo eléctrico almacenados en **InfluxDB**, genera pronósticos horarios mediante un modelo **SARIMA** entrenado en **Google Colab** y presenta los resultados en un dashboard interactivo.

La API está desarrollada con **FastAPI** y se ejecuta en **GitHub Codespaces**, tanto directamente con Uvicorn como dentro de un contenedor **Docker**. El proyecto utiliza como punto de partida la plantilla y la API proporcionadas en el curso; sobre esa base se configuró la fuente de datos, se entrenó y exportó el modelo, se desarrolló la interfaz gráfica y se preparó la contenerización.

## Tecnologías utilizadas

- **Python, Pandas, NumPy y Statsmodels:** tratamiento de la serie temporal y entrenamiento SARIMA.
- **InfluxDB 2.x:** almacenamiento y consulta de mediciones históricas.
- **Google Colab:** análisis, evaluación y exportación del modelo.
- **FastAPI y Uvicorn:** servicio web y endpoints de pronóstico.
- **HTML, CSS, JavaScript y Chart.js:** dashboard, gráfica y controles.
- **GitHub Codespaces:** entorno de desarrollo y ejecución.
- **Docker:** empaquetado y ejecución de la aplicación en un contenedor.

## Datos y entrenamiento del modelo

| Elemento | Configuración |
|---|---|
| Bucket de InfluxDB | `Consumo_elec` |
| Measurement | `Server` |
| Field | `Consumo_total` |
| Observaciones del entrenamiento final | 720 horas |
| Modelo seleccionado | SARIMA(1, 0, 2) × (0, 1, 1, 24) |
| Estacionalidad | 24 horas |
| Archivo exportado | `models/sarima_config.json` |

En el notebook `notebooks/entrenamiento_sarima_colab.ipynb` se realizó la consulta de datos, el análisis de estacionalidad, el tratamiento de valores extremos, la selección de órdenes y la evaluación del modelo sobre las últimas 24 horas reservadas para prueba.

### Resultados de evaluación del entrenamiento final

| Métrica | SARIMA | Línea base ingenua estacional |
|---|---:|---:|
| MAE | **57.0992** | 58.1551 |
| RMSE | 84.7692 | **83.2935** |
| MAPE | **17.9492 %** | 18.4067 % |

SARIMA obtuvo menor error en **MAE** y **MAPE**; la línea base obtuvo menor **RMSE**. Por tanto, el resultado no representa una mejora uniforme en todas las métricas.

La prueba de reproducción del modelo en Colab devolvió:

```text
Diferencia máxima API vs. modelo original: 0.0
```

Esta prueba compara 12 pronósticos del modelo original con los de la reconstrucción realizada a partir de los parámetros exportados en JSON, sin reentrenarlos.

> La fuente contiene mediciones denominadas `Consumo_total`; la unidad física y el dispositivo de adquisición no se especifican en este repositorio. No se deben interpretar los valores como kWh, kW u otra unidad sin esa confirmación.

## Funcionalidades del dashboard

La interfaz `frontend/index.html` permite:

- Visualizar los valores históricos y los pronósticos en una gráfica interactiva.
- Mostrar una banda de **intervalo de confianza del 95 %**.
- Presentar las fechas y horas en la zona **America/Bogota**.
- Consultar indicadores: próxima hora pronosticada, máximo estimado y última observación real.
- Elegir el horizonte de pronóstico (6, 12, 24 o 48 horas) y el histórico visible (24, 48 o 72 horas).
- Consultar una tabla con los valores pronosticados y sus límites.
- Mostrar mensajes de error cuando la API no devuelve una respuesta válida o no se puede completar la consulta.

## Estructura del repositorio

```text
Proyecto_Final_CA/
├── .devcontainer/
├── app/
│   ├── main.py                # Endpoints y servidor FastAPI
│   ├── config.py              # Configuración desde variables de entorno
│   ├── influx.py              # Consultas a InfluxDB
│   └── model.py               # Carga y uso del modelo SARIMA
├── frontend/
│   └── index.html             # Dashboard interactivo
├── models/
│   └── sarima_config.json     # Parámetros del modelo entrenado
├── notebooks/
│   └── entrenamiento_sarima_colab.ipynb
├── scripts/
│   ├── probar_conexion.py
│   └── crear_modelo_demo.py
├── Dockerfile
├── .dockerignore
├── .env.example
├── requirements.txt
└── README.md
```

## Configuración de la conexión a InfluxDB

Las credenciales **no se almacenan en el repositorio ni se copian dentro de la imagen Docker**. Se configuran mediante *Codespaces secrets* o variables de entorno:

| Variable | Descripción |
|---|---|
| `INFLUX_URL` | URL del servidor InfluxDB |
| `INFLUX_TOKEN` | Token de acceso (secreto) |
| `INFLUX_ORG` | Organización |
| `INFLUX_BUCKET` | Bucket de datos |
| `INFLUX_MEASUREMENT` | `Server` |
| `INFLUX_FIELD` | `Consumo_total` |

En GitHub Codespaces: **Settings → Secrets and variables → Codespaces**. La configuración del measurement y field también puede recuperarse de `models/sarima_config.json` cuando se inicia la API.

Para comprobar la conexión sin revelar el token:

```bash
INFLUX_MEASUREMENT=Server INFLUX_FIELD=Consumo_total python scripts/probar_conexion.py
```

## Ejecutar la aplicación sin Docker

Desde la raíz del repositorio, con las dependencias instaladas y las variables de InfluxDB configuradas:

```bash
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Abre el puerto **8000** desde GitHub Codespaces. La página principal (`/`) muestra el dashboard. Para consultar el estado utiliza `/api/salud`.

Detén Uvicorn con `Ctrl + C` antes de usar el mismo puerto con Docker.

## Ejecutar la aplicación con Docker

### Construir la imagen

```bash
docker build -t pronostico:1.0 .
```

### Iniciar el contenedor

Las cuatro variables de conexión deben existir previamente en el entorno desde el cual se ejecuta el comando. Se pasan sus valores al contenedor sin escribirlos en la terminal:

```bash
docker run -d \
  --name pronostico \
  -p 8000:8000 \
  -e INFLUX_URL \
  -e INFLUX_TOKEN \
  -e INFLUX_ORG \
  -e INFLUX_BUCKET \
  -e INFLUX_MEASUREMENT=Server \
  -e INFLUX_FIELD=Consumo_total \
  -e DEMO_MODE=false \
  pronostico:1.0
```

Si ya existe un contenedor llamado `pronostico`, deténlo y elimínalo antes de crear uno nuevo:

```bash
docker stop pronostico
docker rm pronostico
```

### Verificar la ejecución

```bash
docker ps
docker logs --tail 30 pronostico
```

Al abrir `/api/salud`, se comprobó que la API devuelve `estado: ok`, `influx_configurado: true`, `modelo_cargado: true`, `modo_demo: false` y `modelo_demo: false`. El campo `atendido_por` coincidió con el **CONTAINER ID** mostrado en `docker ps`, confirmando la ejecución desde el contenedor.

También se comprobó que los pronósticos de **12 y 24 horas** respondieron con HTTP `200 OK` dentro de Docker.

### Exclusión de archivos sensibles

El archivo `.dockerignore` excluye credenciales (`.env`), historial Git, notebooks y archivos de caché. Se realizó una prueba comparativa con un contexto ficticio: con `.dockerignore` esos archivos no aparecieron en la imagen de prueba; sin él, sí aparecieron. Se comprobó además que la imagen principal no incorporaba variables InfluxDB al ejecutar una inspección sin pasarle credenciales.

## Endpoints de la API

| Ruta | Función |
|---|---|
| `/` | Dashboard web |
| `/docs` | Documentación interactiva de FastAPI |
| `GET /api/salud` | Estado del servicio y del modelo |
| `GET /api/historico?horas=48` | Últimas observaciones históricas |
| `GET /api/prediccion?horas=24&historia=48&confianza=0.95` | Pronóstico e intervalo de confianza |
| `GET /api/modelo` | Información y métricas del modelo exportado |

## Entregables y estado

- [x] Modelo propio exportado en `models/sarima_config.json`.
- [x] Notebook de entrenamiento ejecutado y guardado en `notebooks/`.
- [x] Verificación del modelo reconstruido con diferencia máxima `0.0`.
- [x] Dashboard desarrollado en `frontend/index.html`.
- [x] Ejecución de FastAPI en Codespaces sin Docker.
- [x] `Dockerfile` y `.dockerignore` incluidos.
- [x] Imagen Docker construida y contenedor probado con datos reales.
- [x] Coincidencia de `atendido_por` con el ID del contenedor.
- [x] Video de sustentación enlazado al inicio de este README.

## Nota de seguridad

El token de InfluxDB no debe publicarse en código, archivos `.env`, historial de Git, imágenes Docker ni grabaciones. El repositorio utiliza configuración por variables de entorno para evitar incluir las credenciales en los archivos del proyecto.

---

**Proyecto final · Computación Avanzada · Institución Universitaria Pascual Bravo · 2026**
