# Equipo 4 · Cloudflare · Borrador de apartados

Unidad II · Exposición · Fundamentos de Arquitectura de Software y Desarrollo Backend
Entrega del borrador: miércoles 21 de octubre de 2026 · Exposición: lunes 26 de octubre de 2026

## Equipo

| # | Integrante | Apartados que expone |
|---|---|---|
| 01 | Josue Enmanuel Poot Mateo | El problema · Qué resuelve |
| 02 | Juan Jose Zapata Buenfil | Las piezas · Cómo se usa |
| 03 | Sergio Orion Huerta Martinez | Con lo que ya vimos · Ejemplo |
| 04 | Diego Ramirez Magaña | Ganancias y límites · Cuándo no conviene · Demo |

## Tecnología elegida

**Cloudflare**: poner la API detrás de un proxy para tener HTTPS, caché y protección del servidor.
Para conectar nuestro servidor usamos **Cloudflare Tunnel** (`cloudflared`).

## 1. El problema que resuelve

- Nuestra API (FastAPI + Strawberry GraphQL en `/graphql`) corre con `uvicorn` en el puerto 8000.
- Si la publicamos tal cual, el token JWT viaja sin cifrar porque no hay HTTPS.
- La IP del servidor queda a la vista: cualquiera puede mandar miles de peticiones.
- Cada petición, aunque se repita, llega hasta FastAPI y MySQL.
- Cloudflare se pone en medio como **proxy inverso**: el cliente habla con la red de Cloudflare y Cloudflare habla con nuestro servidor.

## 2. Las piezas con las que se trabaja

- **Servicio en la nube:** la cuenta de Cloudflare con el dominio dado de alta (DNS, certificado, caché, reglas del WAF).
- **Servidor:** `cloudflared`, un programa (o contenedor) que corre junto a la API y abre un túnel *de salida* hacia Cloudflare. No se abre ningún puerto de entrada.
- **Archivo de configuración:** el `docker-compose.yml` con el servicio `tunnel` y su `TUNNEL_TOKEN`. La API deja de publicar `ports`.
- **Cliente:** GraphiQL o la app web. No cambian; solo usan `https://ingresos.example.com/graphql`.

## 3. Cómo se usa, en pasos

1. **Dominio:** dar de alta el dominio en Cloudflare y cambiar sus *nameservers*.
2. **Túnel:** en el panel, *Networking › Tunnels › Create Tunnel*. Guardar el token.
3. **Conector:** correr `cloudflared` con ese token (en nuestro caso, como contenedor del compose).
4. **Ruta:** agregar una *published application*: `ingresos.example.com → http://api:8000`.
5. **Reglas:** WAF administrado, límite de peticiones a `/graphql` y caché solo para contenido público.

## 4. En qué se parece o se diferencia de lo que vimos en el curso

- **Docker:** se parece porque es otra capa de infraestructura y `cloudflared` es un contenedor más. Se diferencia en que la API ya no publica puertos.
- **El token y los permisos:** Cloudflare **no** los reemplaza. Protege el camino, pero quién puede hacer qué lo siguen decidiendo `leer_actor` y `exigir_permiso`.
- **GraphQL:** todo va por `POST /graphql`, y Cloudflare no guarda en caché peticiones que no sean `GET`. La caché ayuda poco con nuestras consultas; sirve para `GET` y archivos estáticos.
- **MySQL:** nunca pasa por Cloudflare. Se queda en la red interna de Docker.

## 5. Ejemplo concreto

La API del Sistema General de Ingresos detrás de un túnel (solo como ejemplo de exposición, no se implementa en el sistema):

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
# Ruta en el panel de Cloudflare: ingresos.example.com → http://api:8000
```

El camino de una petición `accounts` con `Authorization: Bearer TOKEN`:

1. GraphiQL manda `POST https://ingresos.example.com/graphql`.
2. El borde de Cloudflare atiende el HTTPS y aplica WAF y límite de peticiones.
3. `cloudflared` la entrega por el túnel a `http://api:8000`.
4. FastAPI valida el JWT con `leer_actor` y consulta MySQL.

## 6. Qué gana el equipo y qué costo o límite tiene

- **Gana:** HTTPS sin manejar certificados, la IP del servidor oculta, protección contra DDoS y menos tráfico basura llegando a FastAPI.
- **Costo en dinero:** el plan gratuito incluye lo que usamos (HTTPS, DDoS y Tunnel).
- **Límites:** hace falta un dominio propio; Cloudflare termina el HTTPS y puede ver el tráfico descifrado; una regla de caché mal hecha puede servir la respuesta de un usuario a otro.

## 7. Cuándo no conviene usarla

- Solo desarrollo local: basta `localhost` (o un túnel rápido temporal para enseñarlo).
- Datos que no pueden pasar por un tercero.
- Comunicación entre servicios internos en la misma red.

## 8. Fuentes que vamos a consultar

1. Cloudflare Docs. *Cloudflare Tunnel · Get started*. https://developers.cloudflare.com/tunnel/get-started/
2. Cloudflare Docs. *Default cache behavior*. https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
3. Cloudflare Docs. *Proxy status*. https://developers.cloudflare.com/dns/proxy-status/
4. Cloudflare Docs. *Quick Tunnels* (para la demo). https://developers.cloudflare.com/tunnel/get-started/quick-tunnels/

## Opcional: guion de la demo (máximo 3 minutos)

Usamos un **Quick Tunnel**: no necesita cuenta ni dominio y da una URL temporal `trycloudflare.com`.

| Tiempo | Qué se ve en pantalla |
|---|---|
| 0:00–0:30 | La API corriendo local: `uvicorn app.main:app` y `http://localhost:8000/health` responde `{"status":"ok"}`. |
| 0:30–1:00 | En otra terminal: `cloudflared tunnel --url http://localhost:8000`. Se señala la URL `https://….trycloudflare.com` que imprime. |
| 1:00–1:45 | Desde el celular (otra red) se abre `https://….trycloudflare.com/graphql`: el candado de HTTPS y GraphiQL cargando. |
| 1:45–2:30 | `login` con un usuario de prueba y una consulta protegida con `Authorization: Bearer`. El token viajó cifrado. |
| 2:30–3:00 | Se detiene `cloudflared` con Ctrl+C y la URL deja de responder. Cierre: en producción se usa un túnel con dominio propio. |

Cuidado al grabar: no mostrar el `JWT_SECRET`, contraseñas reales ni tokens completos.
