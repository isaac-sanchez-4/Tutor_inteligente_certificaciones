# Tutor Inteligente RAG para Microsoft DP-800

Asistente técnico local basado en RAG (Retrieval-Augmented Generation) para preparar la certificación Microsoft DP-800. El proyecto combina SQL Server 2025 con búsqueda vectorial nativa, FastAPI, Ollama y dos clientes independientes: una interfaz web con Streamlit y un bot de Telegram con OCR para analizar capturas de pantalla.

## Arquitectura

- **Backend (`backend/api.py`):** API FastAPI que clasifica cada petición como `CHAT` o `BUSCAR`, genera el embedding, consulta SQL Server y devuelve respuestas en streaming junto con metadatos.
- **Base de datos:** SQL Server 2025 almacena fragmentos y embeddings nativos `VECTOR(768)`. El procedimiento `sp_BuscarFragmentosSimilares` busca mediante distancia coseno.
- **Modelos locales:** Ollama con `nomic-embed-text` para embeddings y `llama3` con temperatura cero para las respuestas.
- **Cliente web (`frontend/app.py`):** Chat de Streamlit con streaming, historial y telemetría de latencia, fragmentos, distancia y páginas.
- **Cliente móvil (`bot_telegram.py`):** Bot de Telegram que admite texto y fotos. Las imágenes se procesan con Pillow y Tesseract OCR antes de enviarse a la API.

## Estructura

```text
dp800-mini-project/
├── ai/
│   └── INGESTA.py              # Carga el PDF, crea fragmentos y genera embeddings
├── backend/
│   └── api.py                  # API FastAPI, clasificación de intención y motor RAG
├── database/
│   ├── 01_crear_bbdd.sql       # Crea la base de datos DP800_TutorRAG
│   ├── 02_crear_tablas.sql     # Crea tablas y vectores VECTOR(768)
│   └── 03_procedimiento.sql    # Crea sp_BuscarFragmentosSimilares
├── frontend/
│   └── app.py                  # Interfaz web Streamlit
├── documentos/
│   └── DP800.pdf               # Temario utilizado como fuente
├── bot_telegram.py             # Bot de Telegram con OCR
├── config.py                   # Configuración gráfica del bot mediante Tkinter
├── INICIO.bat                  # Inicio automático de Ollama, FastAPI y Streamlit
├── requirements.txt
└── readme.md
```
## Arquitectura 
![Arquitectura del Sistema](docs\arquitectura.webp)

## Requisitos

- Windows con Python 3.10 o superior.
- SQL Server 2025 con soporte nativo para `VECTOR` y ODBC Driver 17 for SQL Server.
- Ollama instalado y ejecutándose.
- Tesseract-OCR instalado en `C:\Program Files\Tesseract-OCR\tesseract.exe` para el bot.
- Entorno virtual `.venv` para usar `INICIO.bat`.

### Descargar los modelos de Ollama

```powershell
ollama pull llama3
ollama pull nomic-embed-text
```

### Instalar las dependencias

```powershell
python -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
```

## Configuración de SQL Server

Los scripts de `database/` deben ejecutarse en este orden:

1. `01_crear_bbdd.sql`
2. `02_crear_tablas.sql`
3. `03_procedimiento.sql`

> **Advertencia:** `01_crear_bbdd.sql` elimina y vuelve a crear `DP800_TutorRAG` si ya existe. No lo ejecutes sobre una base de datos con información que quieras conservar.

La aplicación está configurada actualmente para la instancia `localhost\SQLEXPRESS01` y la base de datos `DP800_TutorRAG`, usando autenticación de Windows. Si tu servidor es diferente, actualiza `SERVER`, `DATABASE` y `CONNECTION_STRING` en `backend/api.py` y `ai/INGESTA.py`.

## Ingesta del documento

La ingesta solo es necesaria al crear o actualizar la base de datos. Coloca el PDF en `documentos/DP800.pdf` y ejecuta desde la raíz:

```powershell
python ai/INGESTA.py
```

El script divide el PDF en fragmentos de 1000 caracteres con solapamiento de 200, genera embeddings de 768 dimensiones y los guarda en SQL Server.

## Ejecución

### Inicio automático en Windows

Con el entorno virtual creado y Ollama en ejecución, ejecuta:

```powershell
INICIO.bat
```

El script abre FastAPI en `http://localhost:8000` y Streamlit en `http://localhost:8501`.

### Inicio manual

Abre una terminal en la raíz para iniciar el backend:

```powershell
python -m uvicorn backend.api:app --reload --port 8000
```

Y otra terminal para iniciar la interfaz web:

```powershell
streamlit run frontend/app.py
```

Puedes comprobar el estado de la API en `http://localhost:8000/` y abrir la interfaz en `http://localhost:8501`.

### Bot de Telegram

Configura el token y la URL de la API mediante el asistente gráfico:

```powershell
python config.py
```

Con FastAPI activo, inicia el bot en una segunda terminal:

```powershell
python bot_telegram.py
```

El bot acepta preguntas de texto y fotografías. Para las fotografías usa Tesseract en inglés (`--psm 6`) y muestra un botón para consultar las páginas fuente.

## Solución de problemas

- **No se puede conectar con SQL Server:** comprueba que el servicio esté iniciado, que exista la instancia `localhost\SQLEXPRESS01`, que esté instalado ODBC Driver 17 y que la base de datos se haya creado ejecutando los scripts en orden.
- **No se puede conectar con Ollama:** inicia Ollama y verifica los modelos con `ollama list`. Si faltan, ejecuta los comandos `ollama pull` de la sección de requisitos.
- **La ingesta no encuentra el PDF:** ejecuta `python ai/INGESTA.py` desde la raíz del proyecto y confirma que el archivo esté en `documentos/DP800.pdf`.
- **El frontend indica que FastAPI no responde:** inicia primero el backend con Uvicorn y comprueba `http://localhost:8000/` antes de abrir Streamlit.
- **La búsqueda no devuelve resultados:** revisa que la ingesta haya terminado correctamente y que la tabla `Fragmentos` contenga datos.
- **El OCR no funciona:** verifica que Tesseract esté instalado en la ruta configurada y que el paquete `pytesseract` esté instalado.
- **El bot no inicia:** ejecuta `python config.py`, guarda un `TELEGRAM_TOKEN` válido y una `API_URL` que termine en `/chat`.

## Limitaciones actuales

- La configuración de conexión está definida directamente en `backend/api.py` y `ai/INGESTA.py`.
- La aplicación está pensada para uso local y no incluye autenticación de usuarios.
- El script `01_crear_bbdd.sql` recrea la base de datos; realiza una copia de seguridad antes de usarlo en un entorno con datos.
- El OCR está configurado para texto inglés y depende de una instalación local de Tesseract.
- `INICIO.bat` inicia Ollama, FastAPI y Streamlit; el bot de Telegram debe iniciarse por separado.

## Funcionalidades

- Recuperación de fragmentos mediante embeddings y búsqueda vectorial en SQL Server.
- Respuestas en streaming desde FastAPI hacia Streamlit.
- Historial de conversación con reescritura de preguntas breves y ambiguas.
- Telemetría de la consulta SQL, distancia de los resultados y páginas del PDF utilizadas.
- Entrada por texto o imagen en Telegram mediante OCR.