# Agenda — contexto del proyecto

Aplicación web personal (PWA) de agenda diaria con lista de tareas y lienzo de
dibujo. Uso privado del autor. Prioridad: simplicidad, que funcione offline y
que se sincronice al instante entre iPhone y Mac. **Sin coste**: todo el stack
es gratuito.

## Cómo hablar con el autor
- Responde en **español**.
- Estilo directo y concreto, tono profesional.
- Flujo de trabajo iterativo: cambios pequeños, ciclos rápidos de refinamiento.

## Arquitectura (leer antes de tocar nada)

Toda la app vive en un **único archivo `index.html`** autocontenido: HTML, CSS
y JavaScript vanilla en el mismo fichero. **No hay frameworks, ni build, ni
dependencias de npm.** No introducir React, bundlers ni pasos de compilación
salvo petición explícita del autor. Mantener esa filosofía de un solo archivo.

Archivos del repositorio:
- `index.html` — toda la aplicación.
- `sw.js` — service worker (offline + auto-actualización).
- `manifest.webmanifest` — metadatos PWA.
- `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` — iconos.
- `LEEME.txt` — instrucciones de despliegue y de Firebase para el autor.

## Despliegue
- Alojado en **GitHub Pages** (rama `main`, carpeta raíz).
- **Cada `git push` a `main` redespliega automáticamente** en 1-2 minutos.
- Flujo esperado con Claude Code: editar → commit → push/PR. No hay que subir
  archivos a mano nunca más.

## Modelo de datos

Estado en memoria y en `localStorage` bajo la clave `agenda-days`:

    days = {
      "2026-07-07": {
        items:   [{ id, text, done }],           // tareas / notas
        strokes: [{ c, s, e, p:[[x,y]...] }],     // trazos de dibujo
        photos:  [{ id, img, u }],                // fotos (dataURL JPEG)
        u: timestamp                              // última modificación (merge)
      }
    }

## Sincronización (Firestore, plan gratuito Spark)
- **Local-first**: la app siempre guarda en el dispositivo y funciona offline.
  Firestore es la capa de sync opcional; si no está configurada, la app
  funciona igual pero cada dispositivo guarda lo suyo.
- La configuración (firebaseConfig + clave de sync) se guarda en `localStorage`
  (`agenda-fb-config`, `agenda-sync-key`), **no en el código**. Por eso
  actualizar el código NO obliga a reconfigurar la sincronización.
- Estructura en Firestore:
  - `agendas/{syncKey}/days/{diaISO}` = `{ items, s (strokes en JSON string), u }`
  - `agendas/{syncKey}/photos/{photoId}` = `{ day, img, u }`
- **Las fotos van en documentos separados a propósito**: Firestore limita cada
  documento a 1 MB. No fusionar fotos dentro del doc del día.
- Firestore no admite arrays anidados → los `strokes` viajan serializados como
  string JSON en el campo `s`.
- Merge por timestamp `u`: gana la versión más reciente. Al aplicar un día
  remoto hay que **preservar las fotos locales** (van por su propio listener).
- Dos listeners `onSnapshot`: uno para `days`, otro para `photos`. Ignorar los
  ecos de escrituras propias (`hasPendingWrites`).
- Reglas de seguridad ya publicadas en la consola de Firebase:

      rules_version = '2';
      service cloud.firestore {
        match /databases/{database}/documents {
          match /agendas/{syncKey}/{document=**} {
            allow read, write: if syncKey.size() >= 30;
          }
        }
      }

  Seguridad por clave secreta larga (>=30 caracteres), sin cuentas de usuario.
  Equilibrio deliberado para uso personal.
- El SDK de Firebase se carga por import dinámico desde
  `https://www.gstatic.com/firebasejs/12.15.0/`. El service worker cachea
  gstatic (cache-first) para que la sync funcione tras la primera carga.

## Dibujo (lienzo a pantalla completa)
- El `<canvas>` se superpone a toda la hoja del día (encima del texto y las
  fotos). En modo escritura tiene `pointer-events:none`; en modo dibujo captura
  los eventos.
- Coordenadas lógicas con ancho fijo `LW = 1000`. **La Y se guarda en la misma
  escala que la X (relativa al ancho)** para que el dibujo sea proporcional en
  iPhone y Mac. No cambiar a píxeles absolutos: rompería la consistencia entre
  dispositivos y los datos ya guardados.
- Gestos en modo dibujo: **un dedo/ratón dibuja, dos dedos desplazan (scroll)**.
- El canvas se reajusta con `ResizeObserver` cuando cambia la altura del
  contenido. No reajustar mientras se está dibujando.

## Otras piezas
- **Dictado**: Web Speech API (`webkitSpeechRecognition`), idioma `es-ES`. El
  botón del micro solo aparece si el navegador lo soporta. Al terminar de
  hablar, la frase se añade como tarea.
- **Fotos**: se reducen a máx. 1000 px y se exportan a JPEG calidad 0.65 antes
  de guardar. Rejilla + lightbox al tocar.
- **Parser de firebaseConfig tolerante** (`parseFirebaseConfig`): acepta el
  bloque tal cual lo da Firebase (formato JS con `import`, JSON, comillas
  tipográficas de macOS/iOS, comillas simples, comas finales). No simplificar a
  `JSON.parse` directo.
- **Cabecera**: solo la fecha en texto (incluye el número del día). No volver a
  añadir una línea de solo números.

## Service worker y auto-actualización
- Estrategia **red primero** para los archivos de la app: al abrir con conexión
  siempre trae la última versión; la caché es solo respaldo offline. gstatic
  (SDK Firebase) va cache-first.
- Auto-actualización: al detectar un service worker nuevo (`controllerchange`)
  la página se recarga sola una vez (con guarda para no recargar en la primera
  instalación ni entrar en bucle).
- **Importante**: si se cambian los archivos cacheados, subir la constante
  `CACHE` de versión en `sw.js` (`agenda-v3` → `agenda-v4`...).

## Estética
- Papel roto (`--paper #FBFBF9`), tinta (`--ink #1C1C1A`), acento **verde
  Manrique** (`--verde #1F7A4D`) como guiño a Lanzarote y como color de estado
  "completado". Tipografía del sistema; monoespaciada para detalles de interfaz.

## Al hacer cambios
1. No romper la compatibilidad con los datos ya guardados (estructura `days`,
   claves de `localStorage`, esquema de Firestore).
2. Mantener el archivo único y sin dependencias.
3. Probar mentalmente el caso offline y el caso con sync activa.
4. Subir `CACHE` en `sw.js` si cambian archivos cacheados.
