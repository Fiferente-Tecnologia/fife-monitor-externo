# fife-monitor-externo

Sonda de disponibilidad **desde fuera** de la infraestructura: un workflow de
GitHub Actions (`.github/workflows/monitor.yml`) corre cada ~5 minutos en
runners de GitHub, verifica que la plataforma responda desde internet y avisa
por Telegram si no.

## Por qué existe

Incidente del 28/07/2026: durante ~50 minutos la plataforma fue inaccesible
desde internet por un filtro en el borde del proveedor, mientras el servidor
estaba sano por dentro. El monitoreo interno (Uptime Kuma, en el mismo
servidor) vio todo verde y no alertó — 629 latidos, 0 caídas. Esta clase de
corte solo se detecta sondeando desde un punto externo.

## Qué vigila

| URL | Señal de vida |
|---|---|
| `https://mapeador.93-188-163-79.sslip.io/` | HTTP 401 (nginx + TLS + basic auth contestando) |
| `https://estado.93-188-163-79.sslip.io/status/fife` | HTTP 200 (página de estado de Kuma) |

3 intentos con pausa de 10 s antes de declarar caída (un blip del runner no
alerta). La alerta va al grupo de Telegram del equipo.

## Este repo es público a propósito

Los minutos de Actions son ilimitados solo en repos públicos (en privado un
cron cada 5 min agota la cuota de la organización). Acá **no hay ningún
secreto**: solo URLs ya públicas. El token del bot y el chat van en
Secrets del repo (`TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`).

## Probar a mano

Actions → monitor-externo → Run workflow → marcar `prueba` → debe llegar
"✅ Monitor externo activo" al grupo.

## Limitaciones conocidas

- El cron de GitHub no es puntual: los ~5 min pueden ser 5–20 según carga de
  GitHub. Para esta clase de fallo (cortes de decenas de minutos) alcanza.
- Sin estado entre corridas: mientras dure una caída, alerta en cada corrida
  (cada ~5–20 min). Es deliberado — simple y sin falsos "recuperado".
