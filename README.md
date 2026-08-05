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

Cada corrida **vigila en bucle ~5,5 h**, sondeando cada 60 s, y avisa al grupo
de Telegram solo al *cruzar* de sano a caído (y otra vez al recuperarse), no en
cada ciclo.

Antes de dar algo por caído hacen falta **tres cosas a la vez**:

1. que los 3 intentos contra el objetivo fallen,
2. que eso se repita **3 ciclos seguidos** (~3 min sostenidos) — un blip corto
   no despierta a nadie,
3. que el runner **sí** llegue a internet, comprobado contra dos referencias
   ajenas a nuestra infra (`google.com/generate_204` y `api.github.com`). Si el
   runner no llega ni a los controles, el problema es suyo: el ciclo se ignora.

El punto 3 es la lección del 05/08: el monitor estuvo avisando de caídas que no
existían — la plataforma respondía desde otras redes mientras el runner de Azure
no la alcanzaba. Con un solo punto de vista, un runner con la red degradada es
indistinguible de un servidor caído.

Los dos objetivos cuelgan del mismo dominio, así que si `sslip.io` deja de
resolver caen juntos y parece una caída de la plataforma. Ese caso se detecta
aparte y el mensaje lo dice. Es el argumento para migrar a un dominio propio.

## Este repo es público a propósito

Los minutos de Actions son ilimitados solo en repos públicos (en privado un
cron cada 5 min agota la cuota de la organización). Acá **no hay ningún
secreto**: solo URLs ya públicas. El token del bot y el chat van en
Secrets del repo (`TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`).

## Probar a mano

Actions → monitor-externo → Run workflow → marcar `prueba` → debe llegar
"✅ Monitor externo activo" al grupo.

## Limitaciones conocidas

- El cron de GitHub no es puntual: declara `*/5` pero dispara cada 1–3 h. Por
  eso cada corrida vigila 5,5 h en bucle y la nueva reemplaza a la anterior
  (`cancel-in-progress`): siempre hay una cubriendo, aunque el scheduler se
  atrase.
- Sin estado **entre** corridas: el bucle recuerda si ya avisó, pero al
  reemplazarse la corrida ese estado se pierde. Una caída que dure más que la
  ventana puede repetir el aviso una vez cada 1–3 h. Preferible a no avisar.
- Un solo punto de observación (runners de GitHub, red de Azure). Los controles
  descartan que el runner esté aislado, pero no un corte de ruta que afecte solo
  a Azure↔Hostinger. Para eso haría falta una segunda sonda en otra red.
