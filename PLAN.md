# Plan de Trabajo: Radar Familiar PWA

## Aplicación de Geolocalización bajo demanda para la familia

---

## 1. Estructura del Proyecto

```
radar-family/
├── public/
│   ├── admin.html              # Panel del Administrador
│   ├── device.html             # Vista del dispositivo objetivo
│   ├── app.js                  # Lógica frontend compartida
│   ├── admin.js                # Lógica específica del Admin
│   ├── device.js               # Lógica específica del Device
│   ├── style.css
│   ├── sw.js                   # Service Worker
│   ├── manifest.json           # PWA manifest
│   └── icons/
│       ├── icon-192.png
│       └── icon-512.png
├── server.js                   # Backend Express
├── push-utils.js               # Lógica web-push
├── firebase-config.js          # Config Firebase Admin SDK
├── .env                        # Variables de entorno
├── .env.example
├── package.json
└── README.md
```

---

## 2. Stack Tecnológico

| Capa               | Tecnología                              | Razón                                          |
|--------------------|-----------------------------------------|------------------------------------------------|
| Frontend           | HTML5 + Vanilla JS ES6                  | Sin frameworks, máximo control                 |
| Mapas              | Leaflet.js + OpenStreetMap tiles        | Gratuito, sin API key                           |
| Backend            | Node.js + Express                       | Familiar, simple                                |
| Push Notifications | `web-push` + VAPID                      | Push nativo PWA                                 |
| DB Tiempo Real     | **Firebase Firestore**                  | TTL nativo + listeners `onSnapshot`             |
| Auth               | API Key familiar (token compartido)     | Sin complejidad de OAuth                        |

### ¿Por qué Firebase Firestore y no Supabase?

- **TTL nativo**: Creas un campo `expireAt` y Firestore borra documentos automáticamente. Supabase requeriría pg_cron o Edge Functions programadas para limpieza periódica.
- **Listeners en tiempo real**: `onSnapshot` es nativo. Supabase requiere configurar Replication Slots y Realtime.
- **Plan Spark gratuito**: 1GB almacenados, 50K lecturas/día, 20K escrituras/día, 20K borrados/día. Para uso familiar (~180 registros/día, ~180 borrados/día por TTL) es suficiente.

---

## 3. Arquitectura de Datos (Firestore)

### Colección: `locations`

```
/locations/{docId}
  ├── deviceId: string       ← Identificador único del dispositivo
  ├── lat: number            ← Latitud
  ├── lng: number            ← Longitud
  ├── accuracy: number       ← Precisión en metros
  ├── sessionId: string      ← ID de la sesión de rastreo
  ├── timestamp: Timestamp   ← Marca de tiempo del servidor
  └── expireAt: Timestamp    ← createdAt + 24h (campo usado por TTL policy)
```

### Colección: `sessions`

```
/sessions/{sessionId}
  ├── adminId: string                ← Quién solicitó
  ├── targetDeviceId: string         ← Dispositivo objetivo
  ├── type: 'single' | 'continuous' ← Tipo de solicitud
  ├── status: 'pending' | 'active' | 'completed' | 'expired'
  ├── intervalSec: number            ← 60 para rastreo continuo
  ├── durationMin: number            ← Duración total en minutos
  ├── startedAt: Timestamp           ← Inicio
  ├── expiresAt: Timestamp           ← startedAt + durationMin
  └── pushCount: number              ← Contador de pushes enviados
```

### Colección: `devices`

```
/devices/{deviceId}
  ├── name: string                   ← Nombre del miembro
  ├── pushSubscription: object       ← Objeto de suscripción web-push
  └── lastSeen: Timestamp            ← Última conexión
```

### Colección: `families`

```
/families/{familyId}
  ├── apiKey: string                 ← Token compartido
  ├── members: [deviceId1, ...]      ← IDs de dispositivos
  └── name: string                   ← "Familia Pérez"
```

### TTL Policy (Firestore)

En Firebase Console:
- `Cloud Firestore` → `TTL Policies` → `Add TTL Policy`
- Colección: `locations`
- Campo: `expireAt`
- Los documentos se borran automáticamente dentro de las 24 horas posteriores al valor de `expireAt`

---

## 4. Estrategia de Geolocalización en Segundo Plano

### Realidad Técnica (verificada)

| Mecanismo               | ¿Funciona en background? | Limitación                                         |
|------------------------|--------------------------|----------------------------------------------------|
| Service Worker GPS     | ❌ No                     | `navigator.geolocation` NO existe en SW             |
| `watchPosition()`      | ⚠️ Parcial               | Android ~5 min, iOS se detiene al instante          |
| `periodicSync`         | ❌ No para GPS            | No da acceso a geolocation                          |
| `push` event           | ❌ No para GPS            | Solo despierta SW, SW no puede hacer geolocation    |
| Wake Lock `'screen'`   | ❌ No                     | Se libera automáticamente al background             |
| Background Sync        | ✅ Sí para re-envíos      | No captura GPS, solo reenvía peticiones fallidas    |

### Estrategia Multicapa

```
┌─────────────────────────────────────────────────────────────────┐
│ CAPA 1: watchPosition en página visible (tiempo real)           │
│   - Se ejecuta mientras la app está abierta/visible             │
│   - wakeLock 'screen' mantiene la pantalla encendida            │
│   - Envía ubicación a /api/location cada vez que cambia         │
├─────────────────────────────────────────────────────────────────┤
│ CAPA 2: Heartbeat del servidor (push cada minuto)               │
│   - Servidor envía push HEARTBEAT cada 60s durante la sesión    │
│   - SW recibe, postMessage a todos los clients abiertos         │
│   - Si hay client (aunque background en Android),                    │
│     ejecuta getCurrentPosition() y envía                        │
├─────────────────────────────────────────────────────────────────┤
│ CAPA 3: Notificación con acción (re-enganche manual)            │
│   - Si no hay clients abiertos, SW muestra notificación         │
│   - Botón "Abrir y rastrear" → usuario toca, app se abre       │
│   - Al abrirse, la app retoma watchPosition automáticamente    │
├─────────────────────────────────────────────────────────────────┤
│ CAPA 4: Background Sync (resiliencia de red)                    │
│   - Si fetch /api/location falla, se registra sync              │
│   - Cuando vuelva la red, se reenvía la ubicación               │
│   - Garantiza que no se pierden datos por conectividad          │
└─────────────────────────────────────────────────────────────────┘
```

### Flujo Detallado: Rastreo Continuo (cada 1 minuto, 60 minutos de duración)

```
FASE 0: Configuración
─────────────────────────────────────────────────────────────────────
Admin elige "Rastreo continuo" → 60 min → dispositivo objetivo
  ↓

FASE 1: Inicio (servidor)
─────────────────────────────────────────────────────────────────────
POST /api/request-location
  Body: { token, targetDeviceId, type: 'continuous', intervalSec: 60, durationMin: 60 }
  
Servidor:
  1. Valida API Key
  2. Crea documento en /sessions/ con status: 'pending'
  3. Obtiene pushSubscription del dispositivo objetivo desde /devices/
  4. Envía push INIT con datos:
     {
       type: 'continuous',
       sessionId: 'abc123',
       intervalSec: 60,
       durationMin: 60,
       expiresAt: <timestamp>
     }
  5. Actualiza session status → 'active', pushCount++
  6. Programa Heartbeat Loop:
     for i = 1 to 60:
       setTimeout(() => {
         sendPush(device, { type: 'heartbeat', sessionId, remainingPushes: 60-i })
       }, i * 60000)
  7. Programa Stop automático tras 60 minutos:
     setTimeout(() => {
       sendPush(device, { type: 'stop', sessionId })
       updateSession(sessionId, { status: 'completed' })
     }, 60 * 60000)

FASE 2: Recepción en Service Worker
─────────────────────────────────────────────────────────────────────
self.addEventListener('push', event => {
  const data = event.data.json()

  if (data.type === 'init' && data.type === 'continuous') {
    Guarda en Cache API:
      sessionId, startedAt, expiresAt, pushesRemaining: 60
    
    self.clients.matchAll({ includeUncontrolled: true, type: 'window' })
      .then(clients => {
        if (clients.length > 0) {
          // Hay cliente abierto → enviar mensaje
          clients[0].postMessage({
            type: 'START_TRACKING',
            sessionId: data.sessionId,
            intervalSec: data.intervalSec,
            durationMin: data.durationMin
          })
        } else {
          // No hay cliente → notificar al usuario
          self.registration.showNotification('Rastreo iniciado', {
            body: `Se solicitará tu ubicación cada ${data.intervalSec}s por ${data.durationMin}min`,
            actions: [{ action: 'open', title: 'Abrir app' }],
            tag: `tracking-${data.sessionId}`,
            requireInteraction: true
          })
        }
      })
  }

  if (data.type === 'heartbeat') {
    // Recordatorio: intentar obtener ubicación
    self.clients.matchAll({ includeUncontrolled: true, type: 'window' })
      .then(clients => {
        if (clients.length > 0) {
          clients[0].postMessage({
            type: 'HEARTBEAT',
            sessionId: data.sessionId,
            remainingPushes: data.remainingPushes
          })
        }
      })
  }

  if (data.type === 'stop') {
    // Detener rastreo
    // 1. Limpiar Cache API
    // 2. Notificar a clients
    self.clients.matchAll().then(clients => {
      clients.forEach(c => c.postMessage({ type: 'STOP_TRACKING', sessionId: data.sessionId }))
    })
    // 3. Cerrar notificaciones activas
    self.registration.getNotifications({ tag: `tracking-${data.sessionId}` })
      .then(notifs => notifs.forEach(n => n.close()))
  }
})

FASE 3: Recepción en Device (página visible)
─────────────────────────────────────────────────────────────────────
navigator.serviceWorker.addEventListener('message', event => {
  if (event.data.type === 'START_TRACKING') {
    sessionId = event.data.sessionId
    trackingActive = true
    
    // Adquirir wake lock
    if ('wakeLock' in navigator) {
      wakeLock = await navigator.wakeLock.request('screen')
    }
    
    // Iniciar watchPosition
    watchId = navigator.geolocation.watchPosition(
      position => {
        sendLocation(position.coords.latitude, position.coords.longitude, position.coords.accuracy)
      },
      error => {
        // Fallback: getCurrentPosition cada 55 segundos
        fallbackInterval = setInterval(() => {
          navigator.geolocation.getCurrentPosition(
            pos => sendLocation(pos.coords.latitude, pos.coords.longitude, pos.coords.accuracy),
            err => console.error('GPS error:', err),
            { enableHighAccuracy: false, timeout: 10000 }
          )
        }, 55000)
      },
      { enableHighAccuracy: false, timeout: 15000, maximumAge: 30000 }
    )
  }
  
  if (event.data.type === 'STOP_TRACKING') {
    trackingActive = false
    navigator.geolocation.clearWatch(watchId)
    clearInterval(fallbackInterval)
    if (wakeLock) wakeLock.release()
  }
  
  if (event.data.type === 'HEARTBEAT') {
    // Heartbeat del servidor: forzar getCurrentPosition
    navigator.geolocation.getCurrentPosition(
      pos => sendLocation(pos.coords.latitude, pos.coords.longitude, pos.coords.accuracy),
      null,
      { enableHighAccuracy: false, timeout: 10000 }
    )
  }
})

function sendLocation(lat, lng, accuracy) {
  fetch('/api/location', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'X-Api-Key': apiKey },
    body: JSON.stringify({ deviceId, lat, lng, accuracy, sessionId, timestamp: Date.now() })
  }).catch(() => {
    // Si falla, registrar background sync para re-intento
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      navigator.serviceWorker.ready.then(reg => {
        reg.sync.register('retry-location')
      })
    }
  })
}

FASE 4: Recepción en Admin (mapa en tiempo real)
─────────────────────────────────────────────────────────────────────
// Listener Firestore en colección locations filtrado por sessionId
db.collection('locations')
  .where('sessionId', '==', sessionId)
  .orderBy('timestamp')
  .onSnapshot(snapshot => {
    snapshot.docChanges().forEach(change => {
      if (change.type === 'added') {
        const data = change.doc.data()
        // Nuevo punto en el mapa
        L.marker([data.lat, data.lng]).addTo(map)
        // Si es rastreo continuo, dibujar polyline
        polyline.addLatLng([data.lat, data.lng])
        map.fitBounds(polyline.getBounds())
      }
    })
  })
```

---

## 5. Endpoints del Backend

| Método | Ruta                    | Auth        | Descripción                                                  |
|--------|-------------------------|-------------|--------------------------------------------------------------|
| POST   | `/api/request-location` | `X-Api-Key` | Admin solicita ubicación única o inicia rastreo continuo     |
| POST   | `/api/location`         | `X-Api-Key` | Device envía coordenadas con `{deviceId, lat, lng, accuracy, sessionId, timestamp}` |
| POST   | `/api/subscribe`        | `X-Api-Key` | Device registra suscripción push `{deviceId, subscription}`  |
| POST   | `/api/stop/:sessionId`  | `X-Api-Key` | Admin detiene una sesión activa                              |
| GET    | `/api/devices`          | `X-Api-Key` | Lista dispositivos de la familia                             |

### Detalle de cada endpoint

#### `POST /api/request-location`

```javascript
// Request
Headers: { 'X-Api-Key': 'token-familiar' }
Body: {
  targetDeviceId: 'device-phone-papa',
  type: 'single' | 'continuous',
  intervalSec?: 60,     // solo para continuous
  durationMin?: 60      // solo para continuous
}

// Response
{
  success: true,
  sessionId: 'abc-123-def',
  status: 'active'
}

// Lógica del servidor:
// 1. Validar API Key contra /families/
// 2. Validar targetDeviceId existe en /devices/
// 3. Crear documento en /sessions/
// 4. Obtener pushSubscription del device desde /devices/{targetDeviceId}/pushSubscription
// 5. Enviar push INIT mediante web-push
// 6. Si type === 'continuous':
//    a. Programar heartbeat loop (setInterval por duración)
//    b. Programar stop automático (setTimeout por duración)
// 7. Responder con sessionId
```

#### `POST /api/location`

```javascript
// Request
Headers: { 'X-Api-Key': 'token-familiar' }
Body: {
  deviceId: 'device-phone-papa',
  lat: 19.4326,
  lng: -99.1332,
  accuracy: 15,
  sessionId: 'abc-123-def',
  timestamp: 1716512345678  // epoch ms
}

// Response
{ success: true }

// Lógica del servidor:
// 1. Validar API Key
// 2. Validar sessionId existe y status === 'active'
// 3. Crear documento en /locations/ con expireAt = ahora + 24h
// 4. Incrementar contador en sesión
```

#### `POST /api/subscribe`

```javascript
// Request
Headers: { 'X-Api-Key': 'token-familiar' }
Body: {
  deviceId: 'device-phone-papa',
  subscription: { endpoint, keys: { p256dh, auth } }
}

// Response
{ success: true }

// Lógica:
// 1. Validar API Key
// 2. Actualizar /devices/{deviceId}/pushSubscription
```

#### `POST /api/stop/:sessionId`

```javascript
// Request
Headers: { 'X-Api-Key': 'token-familiar' }

// Response
{ success: true }

// Lógica:
// 1. Validar API Key
// 2. Actualizar /sessions/{sessionId}/status → 'completed'
// 3. Enviar push STOP al dispositivo objetivo
// 4. Limpiar heartbeats programados
```

#### `GET /api/devices`

```javascript
// Request
Headers: { 'X-Api-Key': 'token-familiar' }

// Response
{
  devices: [
    { id: 'device-phone-papa', name: 'Papá', lastSeen: '2026-05-24T...' },
    { id: 'device-phone-mama', name: 'Mamá', lastSeen: '2026-05-24T...' }
  ]
}
```

---

## 6. Seguridad

### API Key Familiar

- String único de tipo UUID v4 generado al crear la familia
- Se almacena en Firestore: `/families/{familyId}/apiKey`
- Se envía en cada petición HTTP como header: `X-Api-Key: <uuid>`
- Se valida en middleware Express:

```javascript
async function validateApiKey(req, res, next) {
  const key = req.headers['x-api-key']
  if (!key) return res.status(401).json({ error: 'API Key required' })

  const snapshot = await db.collection('families')
    .where('apiKey', '==', key)
    .limit(1)
    .get()

  if (snapshot.empty) return res.status(403).json({ error: 'Invalid API Key' })

  req.family = snapshot.docs[0]
  next()
}
```

### VAPID Keys (web-push)

- Generadas con `npx web-push generate-vapid-keys`
- Almacenadas en `.env`: `VAPID_PUBLIC_KEY` y `VAPID_PRIVATE_KEY`
- La clave pública se expone al frontend para generar suscripciones

### Firestore Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Solo denegado por defecto
    match /{document=**} {
      allow read, write: if false;
    }

    // Las ubicaciones pueden ser leídas por cualquiera con API Key
    match /locations/{doc} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;
      // No se permite update/delete (los maneja TTL)
    }

    // Solo el servidor (Admin SDK) escribe en sessions y devices
    match /sessions/{doc} {
      allow read: if request.auth != null;
      allow write: if request.auth.token.admin === true;
    }

    match /devices/{doc} {
      allow read: if request.auth != null;
      allow write: if request.auth.token.admin === true;
    }
  }
}
```

### HTTPS Obligatorio

- Las PWAs requieren HTTPS para Service Worker y Push API
- Desarrollo local: usar `ngrok` para exponer localhost con HTTPS
- Producción: desplegar en Railway, Render o Vercel (todos con HTTPS nativo)

---

## 7. Limitaciones y Decisiones de Diseño

### Limitaciones Técnicas (transparencia al usuario)

| Plataforma | Comportamiento en background | Qué mostramos en la UI |
|------------|------------------------------|------------------------|
| **Android Chrome** | `watchPosition` funciona ~5 min, luego throttling severo. Heartbeat push puede reactivar momentáneamente. | "Rastreo activo. Mantén Chrome abierto. La precisión puede reducirse si el teléfono está bloqueado." |
| **iOS Safari (PWA)** | Toda ejecución JS se detiene al background. Sin excepción. | "Rastreo activo solo mientras la app esté visible. iOS no permite GPS en segundo plano en PWAs." |
| **Desktop** | Funciona completo mientras la pestaña esté abierta. | Sin restricciones. |

### Decisiones de Diseño

1. **`enableHighAccuracy: false` por defecto**: Ahorra batería. El usuario puede activar alta precisión manualmente si necesita más exactitud.

2. **TTL de 24 horas**: Los datos de ubicación se borran automáticamente. No hay historial a largo plazo. El usuario no necesita limpiar nada manualmente.

3. **Sin autenticación de usuarios**: Solo una API Key familiar compartida. Simple de configurar. Si se necesita más seguridad en el futuro, se puede migrar a Firebase Auth.

4. **Leaflet.js en vez de Google Maps**: Sin API key, sin límites de uso, sin costos.

5. **El SW no almacena ubicaciones**: Solo almacena el estado de la sesión activa (sessionId, expiresAt). Las ubicaciones van directo al servidor.

6. **Heartbeat del servidor como mecanismo principal**: En lugar de confiar en `setInterval` del SW (que el OS puede matar), es el servidor quien orquesta los disparos cada minuto mediante push notifications.

---

## 8. Implementación Paso a Paso

### Fase 1: Proyecto base
```
1.1  Crear directorio y package.json
1.2  npm install express web-push firebase-admin dotenv cors uuid
1.3  Crear .env con VAPID keys y credenciales Firebase
1.4  Crear firebase-config.js
```

### Fase 2: Backend (server.js + push-utils.js)
```
2.1  Servidor Express básico con CORS y JSON parser
2.2  Middleware de validación de API Key
2.3  Endpoint POST /api/request-location
2.4  Endpoint POST /api/location
2.5  Endpoint POST /api/subscribe
2.6  Endpoint POST /api/stop/:sessionId
2.7  Endpoint GET /api/devices
2.8  push-utils.js: funciones sendPush, generateVapidKeys
2.9  Lógica de heartbeat loop para rastreo continuo
```

### Fase 3: Firebase
```
3.1  Crear proyecto en Firebase Console
3.2  Habilitar Firestore en modo nativo
3.3  Generar Service Account (Admin SDK)
3.4  Configurar TTL policy en colección 'locations' con campo 'expireAt'
3.5  Configurar Security Rules
3.6  Crear documento inicial en /families/ con apiKey generada
```

### Fase 4: Frontend Admin (admin.html + admin.js)
```
4.1  Admin.html: selector de dispositivo, botones de solicitud
4.2  Admin.js: inicializar Leaflet map
4.3  Obtener lista de dispositivos desde GET /api/devices
4.4  Llamar POST /api/request-location con tipo 'single' o 'continuous'
4.5  Listener Firestore onSnapshot para /locations/ filtrado por sessionId
4.6  Dibujar marcadores individuales o polyline según el tipo
4.7  Botón "Detener rastreo" para sesiones activas
```

### Fase 5: Frontend Device (device.html + device.js)
```
5.1  Device.html: mostrar estado, solicitar permiso GPS
5.2  Register Service Worker
5.3  Suscripción a push notifications (POST /api/subscribe)
5.4  Listener de mensajes del Service Worker
5.5  Función startTracking(): watchPosition + wakeLock
5.6  Función stopTracking(): clearWatch + release wakeLock
5.7  Función sendLocation(): fetch POST /api/location con fallback Background Sync
5.8  Manejo de errores: permisos denegados, GPS timeout, red caída
```

### Fase 6: Service Worker (sw.js)
```
6.1  Evento install: cachear assets estáticos
6.2  Evento activate: limpiar caches viejos
6.3  Evento push: manejar tipos 'init', 'heartbeat', 'stop', 'single'
6.4  postMessage a todos los clients abiertos
6.5  Notificaciones con action 'open' para re-enganche
6.6  Cache API para estado de sesión activa
6.7  Evento notificationclick: abrir/clients.openWindow
6.8  Evento sync: re-envío de ubicaciones fallidas
```

### Fase 7: PWA
```
7.1  manifest.json con name, short_name, icons, display: standalone
7.2  icons/icon-192.png y icon-512.png
7.3  Etiquetas <link> y <meta> en HTML para manifest y theme-color
```

### Fase 8: Pruebas y Despliegue
```
8.1  npm start para servidor local
8.2  ngrok http 3000 para exponer con HTTPS
8.3  Registro de VAPID keys
8.4  Prueba en Android Chrome:
     - Abrir device.html, registrar push
     - Desde admin, solicitar ubicación única → verificar marcador en mapa
     - Iniciar rastreo continuo → verificar llegada de ubicaciones cada 60s
     - Bloquear pantalla → verificar comportamiento
     - Detener rastreo → verificar fin de envíos
8.5  Prueba en iOS Safari (PWA añadida a pantalla de inicio)
8.6  Prueba en escritorio
```

---

## 9. Arquitectura de Componentes (Diagrama de Flujo)

```
┌─────────────────────────────────────────────────────────────┐
│                      NAVEGADOR ADMIN                        │
│                                                             │
│  admin.html ─── admin.js                                    │
│                   │                                         │
│                   ├── Leaflet map                           │
│                   ├── Firestore onSnapshot ←─────┐         │
│                   │     (escucha /locations/)     │         │
│                   ├── fetch POST /api/request     │         │
│                   └── fetch POST /api/stop        │         │
└───────────────────────────────────────────────────┼─────────┘
                                                    │
┌───────────────────────────────────────────────────┼─────────┐
│                    SERVIDOR (Node.js)             │         │
│                                                    │         │
│  server.js                                         │         │
│    ├── Middleware: validateApiKey                   │         │
│    ├── POST /api/request-location ─────→ web-push ─┤         │
│    ├── POST /api/location ───────────→ Firestore ──┼─────────┘
│    ├── POST /api/subscribe ─────────→ Firestore    │
│    ├── POST /api/stop/:sessionId ──→ web-push      │
│    └── Heartbeat Loop (setInterval) ──→ web-push   │
│                                                      │
│  push-utils.js                                      │
│    └── sendPush(device, payload)                    │
│                                                      │
│  firebase-config.js                                 │
│    └── Admin SDK initialization                     │
└──────────────────────────────────────────────────────┘
                                                    │
                                                    │ Push
                                                    │ Notification
                                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   NAVEGADOR DEVICE                          │
│                                                             │
│  device.html ─── device.js ──── Service Worker (sw.js)      │
│                   │                  │                      │
│                   │                  ├── Event: push        │
│                   │                  │   ├── init → postMessage START       │
│                   │                  │   ├── heartbeat → postMessage GPS    │
│                   │                  │   ├── stop → postMessage STOP       │
│                   │                  │   └── single → postMessage GET_GPS  │
│                   │                  │                                      │
│                   │                  ├── Event: sync (re-intentos)          │
│                   │                  └── Event: notificationclick           │
│                   │                                                         │
│                   ├── Recibe postMessage → watchPosition                    │
│                   ├── Wake Lock API                                         │
│                   ├── fetch POST /api/location                             │
│                   └── registerBackgroundSync (fallback)                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Variables de Entorno (.env)

```
# === FIREBASE ===
FIREBASE_PROJECT_ID=radar-familia-xxxxx
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@radar-familia.iam.gserviceaccount.com

# === WEB-PUSH (VAPID) ===
VAPID_PUBLIC_KEY=BDd3_hC7f...
VAPID_PRIVATE_KEY=YjNlZj...
VAPID_SUBJECT=mailto:admin@familia.com

# === APP ===
PORT=3000
FAMILY_API_KEY=550e8400-e29b-41d4-a716-446655440000
FAMILY_NAME=Familia Pérez

# === FRONTEND (se expone al cliente) ===
NEXT_PUBLIC_FIREBASE_API_KEY=AIzaSy...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=radar-familia-xxxxx
NEXT_PUBLIC_VAPID_PUBLIC_KEY=BDd3_hC7f...
```

---

## 11. Dependencias (package.json)

```json
{
  "name": "radar-family",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "firebase-admin": "^12.6.0",
    "web-push": "^3.6.7",
    "uuid": "^10.0.0"
  }
}
```

---

## 12. Resumen de Decisiones Clave

| Decisión | Opción Elegida | Alternativa | Razón |
|----------|---------------|-------------|-------|
| Base de datos | Firebase Firestore | Supabase | TTL nativo para auto-borrado 24h. Listeners en tiempo real sin configuración extra. |
| Background GPS | watchPosition + Heartbeat Push | periodicSync | periodicSync no da acceso a GPS y solo funciona en Android. Heartbeat push es multiplataforma. |
| Almacenamiento de ubicaciones | Servidor (Firestore) | IndexedDB local + sincronización | El usuario pidió persistencia mínima. En servidor se controla TTL centralizadamente. |
| Autenticación | API Key familiar | Firebase Auth | Simplicidad. Una sola llave para toda la familia. Fácil de compartir. |
| Framework frontend | Vanilla JS | React/Vue | Sin dependencias pesadas. La app es pequeña (2 vistas). |
| Mapa | Leaflet.js | Google Maps | Gratuito. Sin API key. Sin límites de uso. |
| Servidor heartbeat | setInterval en servidor | Cliente programa sus propios timers | El OS puede matar timers del cliente. El servidor tiene control absoluto del schedule. |

---

## 13. Preguntas Pendientes para el Usuario

1. **Firebase vs Supabase**: ¿Confirmas Firebase Firestore o prefieres Supabase? La recomendación es Firebase por el TTL nativo.

2. **Cantidad de miembros**: ¿Cuántos dispositivos/miembros estimas (2-3, 5-10, más de 10)? Para dimensionar el plan.

3. **Despliegue**: ¿Tienes servidor propio o prefieres que configure para Railway/Render (gratuito)?

4. **Android vs iOS**: ¿La mayoría usa Android o iOS? Para saber qué tan críticas son las limitaciones de background en iOS.

5. **Notificaciones push**: ¿Los dispositivos objetivo ya tienen la app añadida a pantalla de inicio? Las PWAs solo reciben push si están instaladas como "a launcher" (Android) o añadidas a pantalla de inicio (iOS 16.4+).
