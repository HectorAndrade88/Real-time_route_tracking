# Real-time route tracking — MVP (recogida de vehículos, PMI)

Aplicación web de autoseguimiento de ruta en un único archivo (`index.html`). Cada sesión
se ve a sí misma sobre la ruta activa: no hay backend ni observador de otros vehículos.

Caso aplicado: **Terminal de llegadas del aeropuerto de Palma (PMI) → punto de recogida
de vehículos de alquiler**, con tres opciones de ruta, zonificación aeroportuaria y
consola en cuatro idiomas.

## Ejecutar

Abrir `index.html` en el navegador funciona para todo salvo la geolocalización real
(`file://` no es contexto seguro, así que arranca en modo simulación automáticamente).

Para probar la geolocalización real hace falta https (o `localhost`):

```bash
npx --yes serve .
```

## Qué incluye

- **Tres opciones de ruta** calculadas por OSRM con puntos de paso definidos por la
  operación (directa Ma-19, litoral Ca'n Pastilla, Coll d'en Rabassa). Se eligen desde el
  panel lateral o pinchando la traza en el mapa; la activa se dibuja en verde y las otras
  en gris discontinuo.
- **Detección de cambio de ruta**: si el vehículo sale del corredor de la ruta activa pero
  entra en el corredor de otra opción, la consola cambia la ruta activa sola y lo avisa.
  Si no coincide con ninguna, se mantiene el estado de desvío.
- **Zonas del aeropuerto** con dos niveles de exigencia:
  - `prohibited` → estado rojo (Zona de Seguridad Restringida, lado aire/plataforma).
  - `regulated` → la zona se anuncia y se resalta su reglamento, sin marcar infracción
    (vial de llegadas, vial perimetral, transporte público, entrega de alquiler).
- **Reglamentos y recomendaciones** en el panel lateral. Al entrar en una zona, su ficha
  se resalta y se desplaza a la vista.
- **Cuatro idiomas** (Español, English, Deutsch, Français) con selector en la barra
  superior. Se traduce toda la interfaz: reglamentos, recomendaciones, rutas, zonas,
  avisos y etiquetas del mapa. La preferencia se guarda en `localStorage`.

## Stack

- **Leaflet 1.9.4** para el render del mapa (BSD, sin cuenta ni API key).
- **Tiles OpenStreetMap** raster, atribución "© OpenStreetMap contributors".
- **Geolocation API** nativa (`watchPosition`, `enableHighAccuracy: true`).
- **OSRM público** (`router.project-osrm.org`) para la geometría por calles.
- Vanilla JS + CSS, sin framework ni paso de build.

> Leaflet se carga desde unpkg: el archivo abre sin build, pero necesita red la primera
> vez. Para uso 100 % offline hay que embeber `leaflet.js`/`leaflet.css` y usar
> `precomputedPath` en cada opción de ruta.

## Configuración

Todo se controla desde `routeConfiguration` y `localization`, al inicio del `<script>`.

| Campo | Uso |
| --- | --- |
| `origin`, `destination` | Puntos fijos de la ruta, `[lat, lng]`. |
| `routeOptions[]` | Opciones de ruta: `id`, `viaPoints`, `isRecommended`, `precomputedPath`. |
| `routeOptions[].precomputedPath` | Polilínea precalculada. Si está presente (≥ 2 puntos), esa opción no llama a OSRM. |
| `corridorMeters` | Ancho del corredor para el estado "en ruta". |
| `accuracyThresholdMeters` | Lecturas de GPS con `accuracy` peor que esto se descartan. |
| `movingAverageWindow` | Tamaño de la media móvil sobre las últimas N posiciones. |
| `regulations[]` | Reglamentos. Con `polygon` son geovallas; `enforcement` es `prohibited`, `regulated` o `route`. |
| `recommendations[]` | Recomendaciones operativas (código + icono). |
| `operator.logoImageDataUri` | Logotipo del operador como data URI. Si es `null` se usa un glifo genérico. |
| `simulationSpeedKmh`, `simulationStepMeters`, `simulationEmitIntervalMs` | Modo simulación. |
| `vehicleFocusZoom`, `markerAnimationDurationMs` | Encuadre y deslizamiento del marcador. |
| `localization.dictionaries` | Textos por idioma, indexados por código de reglamento, recomendación y opción de ruta. |

El origen de la geometría es intercambiable sin tocar el resto del código: `RoutePlanner`
devuelve `{ coordinates, source, distanceMeters, durationSeconds }` y `source` puede ser
`embedded`, `routing-service` o `straight-fallback`.

## Estados

| Estado | Condición | Color |
| --- | --- | --- |
| En ruta | distancia a la polilínea activa ≤ `corridorMeters` | `#00843D` |
| Fuera del corredor | distancia > `corridorMeters` | `#B36A00` |
| Zona restringida | el punto cae dentro de una zona `prohibited` | `#B3261E` |

La distancia punto-a-polilínea (proyección local a metros + punto-a-segmento) y el
punto-en-polígono (ray casting) están implementados a mano en `GeoMath`, sin librería
geoespacial.

## Marca

El distintivo del operador que se dibuja es un glifo genérico de vehículo, **no** el
logotipo registrado de ninguna compañía de alquiler. Para usar el asset oficial (con los
derechos correspondientes), colócalo en `routeConfiguration.operator.logoImageDataUri`.

## Restricciones conocidas

1. **Sin seguimiento en segundo plano.** Al bloquear el teléfono o mandar la pestaña a
   background, el sistema operativo suspende los timers y corta la geolocalización.
   Mitigación en este MVP: Screen Wake Lock API mientras se hace seguimiento.
   Camino de producción: app nativa (.NET MAUI) con permisos de ubicación en segundo plano.

2. **OSRM público es un servidor de demo**, con límites de uso y sin SLA. Cada carga hace
   una petición por opción de ruta (tres). No usar en producción: precalcula las rutas y
   embébelas en `precomputedPath`, o auto-hospeda OSRM / Valhalla.

3. **La geolocalización real exige contexto seguro** y suele estar bloqueada dentro de
   iframes cross-origin. Si falla o se deniega, la simulación arranca sola y lo avisa.

4. **Las coordenadas de las zonas son aproximaciones** para la demo. Siguen la
   nomenclatura del zonificado aeroportuario (ZSR, lado aire, viales de servicio,
   transporte público, entrega de alquiler), pero deben sustituirse por la cartografía
   oficial de AENA antes de cualquier uso operativo. Los textos de los reglamentos son
   contenido de demostración, no una publicación oficial.
