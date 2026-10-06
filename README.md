# OCR Workspace

Entorno local para desarrollar y ejecutar juntos los servicios OCR, gateway e interpretación de documentos.

## Servicios

| Servicio | Directorio | Uso | Puerto |
|---|---|---|---:|
| OCR | `tiny-ocr/` | Extrae texto de PDF e imágenes. | 3000 |
| Gateway | `tiny-ocr-gateway/` | Interfaz web y API que coordina los servicios. | 3001 |
| Interpretador | `ai-interpreter/` | Convierte el texto OCR a JSON validado con un schema. | 3002 interno |

Cada directorio de servicio conserva su propio repositorio Git. Este `compose.yaml` los arranca juntos.

## Requisitos

- Docker con Docker Compose.
- PocketBase y RustFS ejecutándose fuera de este Compose y accesibles desde los contenedores.
- Una clave de OpenRouter para usar la interpretación.
- El bucket de RustFS indicado en `RUSTFS_BUCKET` debe existir.

Compose usa `host.docker.internal` para acceder a servicios que corren en el host. `RUSTFS_ENDPOINT` también aparece en URLs firmadas: debe ser accesible desde los contenedores y desde quien descargue el resultado. No uses `localhost` para RustFS si corre en el host.

## Configuración

Copia el archivo de ejemplo y reemplaza las credenciales de muestra:

```sh
cp .env.example .env
```

| Variable | Valor en `.env.example` / defecto Compose | Uso |
|---|---|---|
| `OCR_PORT` | `3000` | Puerto del OCR publicado en el host. |
| `OCR_FRONT_PORT` | `3001` | Puerto del gateway publicado en el host. |
| `PB_URL` | `http://host.docker.internal:8090` | URL de PocketBase; guarda el estado de los jobs. |
| `RUSTFS_ENDPOINT` | `http://host.docker.internal:9000` | Endpoint S3 de RustFS; almacena resultados y sirve descargas firmadas. |
| `RUSTFS_BUCKET` | `ocr-results` | Bucket donde se guardan los resultados. |
| `RUSTFS_ACCESS_KEY` | `change-me` | Credencial de RustFS; reemplázala. |
| `RUSTFS_SECRET_KEY` | `change-me` | Secreto de RustFS; reemplázalo. |
| `RUSTFS_REGION` | `us-east-1` | Región S3 configurada para RustFS. |
| `RUSTFS_URL_EXPIRES_SECONDS` | `300` | Vigencia, en segundos, de las URLs firmadas. |
| `OCR_RENDER_SCALE` | `1.5` | Escala de renderizado de páginas PDF. |
| `OCR_MEMORY_TRIM` | `1` | Libera memoria entre páginas OCR (`1` activado). |
| `TESSERACT_LANG` | `spa+eng` | Idiomas de Tesseract. |
| `TESSERACT_PSM` | `3` | Modo de segmentación de página de Tesseract. |
| `OCR_ISOLATE_PROCESS` | `1` | Procesa cada documento en un proceso hijo (`1` activado). |
| `OPENROUTER_API_KEY` | `replace-me` (Compose: vacío si falta) | Clave obligatoria de OpenRouter; reemplaza el marcador. |
| `OPENROUTER_MODEL` | `openai/gpt-5-nano` | Modelo usado para interpretar documentos. |
| `INTERPRETER_MAX_TEXT_BYTES` | `524288` | Tamaño máximo del texto OCR que procesa el interpretador. |
| `OPENROUTER_MAX_TOKENS` | `4096` | Máximo de tokens solicitado a OpenRouter. |

## Ejecución

Desde este directorio:

```sh
docker compose up --build -d
docker compose ps
```

- Gateway: <http://localhost:3001>
- OCR y estado: <http://localhost:3000/alive>
- El interpretador solo es accesible dentro de la red de Compose.

Para ver logs y detener los contenedores:

```sh
docker compose logs -f
docker compose down
```

`docker compose down` no borra los datos de PocketBase ni RustFS, que son servicios externos.

## Flujo y API

Para interpretar un PDF, envía `POST /interpret/async` al gateway como `multipart/form-data`, con los campos `file` y `schema` (JSON Schema serializado). La respuesta incluye un `job_id`.

Consulta `GET /interpret/jobs/{job_id}` hasta que el job termine. Después, `GET /interpret/jobs/{job_id}/result` redirige a una URL firmada de RustFS con el JSON. Los resultados de interpretación se guardan bajo `interpretations/` en el bucket.

El gateway coordina OCR e interpretación y guarda el estado de los jobs en PocketBase. Los jobs de OCR también se pueden usar mediante la API del servicio OCR.

## Comprobación

```sh
docker compose config --quiet
```

Los comandos de pruebas de cada servicio están en el README de su directorio. No expongas PocketBase o RustFS a Internet con credenciales de ejemplo.
