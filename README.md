# tiny-ocr — entorno local

Este directorio reúne los servicios para desarrollarlos y probarlos juntos:

- `tiny-ocr/`: API Python OCR (puerto 3000).
- `tiny-ocr-gateway/`: gateway Bun y pequeña interfaz web (puerto 3001).
- `ai-interpreter/`: servicio Bun de interpretación con OpenRouter y AJV (puerto interno 3002).
- `compose.yaml`: arranque conjunto; el gateway coordina OCR e interpretación.

Los tres directorios de servicios (`ai-interpreter/`, `tiny-ocr-gateway/` y `tiny-ocr/`) conservan sus propios repositorios Git.

## Arranque

Copia `.env.example` a `.env`, configura las credenciales y ejecuta desde esta carpeta:

```sh
cp .env.example .env
docker compose up --build
```

- Gateway: <http://localhost:3001>
- OCR: <http://localhost:3000/alive>

- Interpretación: `POST /interpret/async` en el gateway (PDF + campo `schema` con JSON Schema). Devuelve `{job_id}`.

Consulta `GET /interpret/jobs/{job_id}` hasta que `status` sea `done`; entonces `GET /interpret/jobs/{job_id}/result` redirige a una URL firmada de RustFS con el JSON. El gateway guarda temporalmente el estado del job en PocketBase y el resultado en el prefijo `interpretations/` del bucket configurado.

El interpretador no publica puerto en Compose. Configura `OPENROUTER_API_KEY` en `.env`; el modelo por defecto es `openai/gpt-5-nano` y se puede cambiar con `OPENROUTER_MODEL`.

PocketBase y RustFS deben estar ejecutándose fuera de Compose y ser accesibles desde los contenedores. Los valores iniciales usan los puertos 8090 y 9000 del host. Cambia las credenciales de ejemplo por las de tu entorno.

`RUSTFS_ENDPOINT` se usa también en las URLs firmadas entre gateway, interpretador y cliente, por lo que debe ser accesible desde los contenedores y, para la descarga final, desde el servicio cliente. No uses `localhost` si RustFS corre en el host, porque dentro del contenedor apuntaría al propio contenedor.

Tras cambiar código, reconstruye los servicios con `docker compose up --build -d`. Para ver logs o detenerlos:

```sh
docker compose logs -f
docker compose down
```

`docker compose down` no elimina los datos de PocketBase ni RustFS, que siguen siendo servicios externos.
