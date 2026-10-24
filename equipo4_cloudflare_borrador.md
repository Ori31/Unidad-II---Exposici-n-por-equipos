# Equipo 4 · Cloudflare · Borrador de apartados

Unidad II · Exposición · Fundamentos de Arquitectura de Software y Desarrollo Backend
Entrega del borrador: miércoles 21 de octubre de 2026 · Exposición: lunes 26 de octubre de 2026

## Equipo

| # | Integrante | Apartados que expone |
|---|---|---|
| 01 | Josue Enmanuel Poot Mateo | El problema · Qué resuelve |
| 02 | Juan Jose Zapata Buenfil | Las piezas · Cómo se usa |
| 03 | Sergio Orion Huerta Martinez | Con lo que ya vimos · Ejemplo con nuestra API |
| 04 | Diego Ramirez Magaña | Ganancias y límites · Cuándo no · Fuentes · Demo |

## Tecnología elegida

**Cloudflare**: un proxy delante de nuestra API para tener HTTPS, caché y protección del servidor.
Para conectar nuestro servidor usamos **Cloudflare Tunnel** (`cloudflared`).

## 01. El problema

Nuestra API del Sistema General de Ingresos (FastAPI + Strawberry GraphQL en `/graphql`) corre con `uvicorn` en el puerto `8000`. Si la publicamos tal cual a internet sin ningún filtro intermedio (`Cliente [GraphiQL o la app] → Internet sin filtro → Servidor [FastAPI :8000 + MySQL]`):

- **Puerto abierto directamente:** `uvicorn` queda expuesto directamente en el puerto `8000`.
- **HTTP y token en claro:** sin HTTPS, el token JWT (`Authorization: Bearer`) viaja en texto plano por la red y puede ser interceptado.
- **IP pública expuesta:** conociendo la IP real del servidor, cualquiera puede saturarlo con miles de peticiones abusivas.
- **Sin caché intermedia:** cada petición repetida atraviesa internet y llega hasta FastAPI y MySQL, consumiendo recursos de cómputo y base de datos innecesariamente.

## 02. Qué resuelve

Cloudflare se pone en medio como **proxy inverso**: el cliente habla primero con la red de Cloudflare y de ahí Cloudflare se comunica con nuestro servidor, resolviendo los tres puntos clave del tema:

- **HTTPS automático:** otorga un certificado TLS gratuito que se renueva solo; el token JWT ya no viaja en claro entre el cliente y el borde de Cloudflare.
- **Caché en el borde:** guarda respuestas `GET` y recursos estáticos en servidores cercanos al usuario para responder de inmediato sin que la petición llegue a la API.
- **Protección del servidor:** oculta la IP pública real del servidor detrás de la red de Cloudflare, frena ataques DDoS y filtra tráfico malicioso con reglas del WAF (Web Application Firewall).

## 03. Las piezas

Para poner nuestra API detrás de Cloudflare trabajamos con cuatro elementos:

- **Nube (Panel de Cloudflare):** la cuenta de Cloudflare donde se da de alta el dominio y se administran el DNS, el certificado HTTPS, la caché y las reglas del WAF.
- **Conector (`cloudflared`):** un programa ligero (o contenedor) que corre junto a la API y abre un **túnel de salida** cifrado hacia la red de Cloudflare. Así no se abre ningún puerto de entrada en el firewall del servidor.
- **Configuración (`docker-compose.yml`):** el archivo de orquestación donde se define el servicio `tunnel` con su variable `TUNNEL_TOKEN`, y donde el servicio `api` deja de publicar puertos (`ports`).
- **Cliente (GraphiQL o la app web):** no requiere cambios en su código; únicamente apunta al dominio con HTTPS (`https://ingresos.example.com/graphql`).

## 04. Cómo se usa

Primero Cloudflare toma el DNS del dominio y después nuestro servidor se conecta a Cloudflare desde adentro, sin abrir ningún puerto de entrada:

1. **Dominio:** dar de alta el dominio en el panel de Cloudflare y cambiar sus *nameservers* en el registrador para que Cloudflare administre el DNS.
2. **Túnel:** en el panel de Cloudflare (*Networking › Tunnels › Create Tunnel*), crear el túnel, copiar el token generado (`TUNNEL_TOKEN`) y levantar el conector `cloudflared` en nuestro servidor (como contenedor en Docker Compose).
3. **Ruta:** registrar una *Published application* que conecte el subdominio público con el servicio interno de la red de Docker: `ingresos.example.com → http://api:8000`. Con esto la API ya responde en `https://` y la IP del servidor deja de publicarse.
4. **Reglas:** activar el WAF administrado, configurar un límite de peticiones (*rate limiting*) hacia `/graphql` y asegurar que la caché aplique únicamente para contenido público.

## 05. Con lo que ya vimos

Comparando Cloudflare con las piezas que ya trabajamos en el curso:

- **Docker:** se parece porque añade una capa más de infraestructura donde `cloudflared` corre como un contenedor adicional en el `docker-compose.yml`. Se diferencia en que hoy nuestro compose publica puertos al exterior, y con el túnel la API no expone ninguno.
- **Token y permisos:** Cloudflare **no** los reemplaza. Cloudflare protege el camino de red (transporte y filtrado perimetral), pero dentro de la API nuestras funciones `leer_actor` y `exigir_permiso` siguen decidiendo quién está autenticado y qué permisos tiene.
- **GraphQL:** en nuestra API todas las operaciones viajan por `POST /graphql`, y por defecto Cloudflare no guarda en caché peticiones `POST`. Por tanto, la caché ayuda sobre todo con peticiones `GET` y archivos estáticos, no con las consultas o mutaciones de GraphQL.
- **MySQL:** nunca pasa por Cloudflare ni sale a internet; permanece aislada dentro de la red interna de Docker, comunicándose únicamente con el contenedor `api`.

## 06. Ejemplo con nuestra API

### 01 · La API de ingresos en `docker-compose.yml`

Agregamos el servicio `cloudflared` al `docker-compose.yml` del Sistema General de Ingresos (como ejemplo arquitectónico de exposición). La API deja de publicar el puerto `8000` y solo el túnel puede alcanzarla dentro de la red de Docker:

```yaml
services:
  db:
    image: mysql:8.4          # igual que hoy
  api:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0
    # sin "ports": nadie entra directo
  tunnel:
    image: cloudflare/cloudflared
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
# Ruta en el panel de Cloudflare:
# ingresos.example.com → http://api:8000
```

### 02 · El camino de una petición

Una consulta real del curso como `accounts`, enviada con su encabezado `Authorization: Bearer TOKEN`, pasa por cuatro paradas antes de leer la base de datos:

1. **01 · GraphiQL:** el cliente envía `POST https://ingresos.example.com/graphql` con el `Bearer token`.
2. **02 · Borde de Cloudflare:** termina la conexión HTTPS, evalúa las reglas del WAF y verifica el límite de peticiones (*rate limit*).
3. **03 · `cloudflared`:** recibe la petición a través del túnel cifrado y la entrega internamente a `http://api:8000`.
4. **04 · FastAPI + MySQL:** `leer_actor` valida la firma del JWT, verifica permisos, consulta MySQL y devuelve la respuesta por el mismo camino.

## 07. Ganancias y límites

- **Lo que gana el equipo:** HTTPS automático sin gestionar ni renovar certificados manualmente, la IP real del servidor oculta, protección contra ataques DDoS y mucho menos tráfico basura llegando a FastAPI.
- **Costo en dinero (`$0`):** el plan gratuito de Cloudflare ya incluye todo lo que utilizamos en este esquema (HTTPS, protección DDoS básica, reglas esenciales y Cloudflare Tunnel).
- **Lo que cuesta (límites y riesgos):**
  - Hace falta contar con un dominio propio para producción (salvo al usar túneles temporales de prueba).
  - Se depende de un tercero que termina el HTTPS en su borde, por lo que Cloudflare puede ver el tráfico descifrado antes de reenviarlo al túnel.
  - Una regla de caché mal configurada (por ejemplo, forzar caché en respuestas autenticadas) podría almacenar y mostrar datos privados de un usuario a otro.

## 08. Cuándo no · Fuentes

### Cuándo no conviene usarla

1. **01 · Solo desarrollo local:** para programar y probar en clase basta con `localhost`; si solo se quiere mostrar un avance rápido, basta un túnel temporal sin configurar dominio ni reglas.
2. **02 · Datos que no deben pasar por terceros:** como Cloudflare termina el HTTPS en su red, existen escenarios con regulaciones estrictas donde ningún intermediario externo puede inspeccionar las peticiones en texto claro.
3. **03 · Servicios internos:** dos backends o microservicios que se comunican dentro de la misma red privada (o red interna de Docker) no necesitan salir a un proxy público en internet para hablar entre sí.

### Fuentes

1. Cloudflare Docs. *Cloudflare Tunnel · Get started*. https://developers.cloudflare.com/tunnel/get-started/
2. Cloudflare Docs. *Default cache behavior*. https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
3. Cloudflare Docs. *DNS · Proxy status*. https://developers.cloudflare.com/dns/proxy-status/
4. Cloudflare Docs. *Quick Tunnels* (para la demo). https://developers.cloudflare.com/tunnel/get-started/quick-tunnels/

## Guion de la demo (máximo 3 minutos)

Para la demostración en vivo (o video) usamos un **Quick Tunnel**, que no requiere cuenta ni dominio propio y genera una URL temporal bajo `trycloudflare.com`:

```bash
cloudflared tunnel --url http://localhost:8000
```

| Tiempo | Qué se ve en pantalla |
|---|---|
| 0:00–0:30 | La API corriendo en local con `uvicorn app.main:app` y `http://localhost:8000/health` respondiendo `{"status":"ok"}`. |
| 0:30–1:00 | En otra terminal se ejecuta `cloudflared tunnel --url http://localhost:8000` y se señala la URL pública `https://….trycloudflare.com` que imprime el conector. |
| 1:00–1:45 | Desde un celular (conectado a otra red) se abre `https://….trycloudflare.com/graphql`, mostrando el candado de HTTPS activo y la interfaz de GraphiQL cargando. |
| 1:45–2:30 | Se ejecuta `login` con un usuario de prueba y luego una consulta protegida enviando `Authorization: Bearer`. El token viaja cifrado por HTTPS hasta la API local. |
| 2:30–3:00 | Se detiene `cloudflared` con `Ctrl+C` y se comprueba que la URL pública deja de responder de inmediato. Cierre: en producción se reemplaza por un túnel fijo con dominio propio en `docker-compose.yml`. |

> **Cuidado al grabar:** no mostrar el `JWT_SECRET`, contraseñas reales ni tokens completos en pantalla.
