https://www.npmjs.com/package/classicube-protocol

================================================================
  classicube-protocol — Tutorial completo v4
  Node.js · Sin dependencias externas · Protocol version 7
================================================================


----------------------------------------------------------------
INDICE
----------------------------------------------------------------

  1.  Instalacion
  2.  Estructura de archivos
  3.  Flujo de conexion
  4.  createServer — servidor en 5 lineas
  5.  createClient — cliente en 5 lineas
  6.  ping — consultar info de un servidor
  7.  Servidor completo con todos los features
  8.  Cliente completo con todos los features
  9.  Level — mundos y bloques sin zlib manual
  10. client.write() y server.broadcast()
  11. Autenticacion con ClassiCube
  12. DEBUG mode
  13. Referencia: Server
  14. Referencia: ServerConnection
  15. Referencia: ClientConnection
  16. Referencia: Level
  17. IDs de bloques
  18. Colores en el chat
  19. Movimiento: cuando usar cada paquete
  20. Errores comunes


================================================================
1. INSTALACION
================================================================

Desde npm:
  npm install classicube-protocol

O copiando la carpeta al proyecto:
  const cc = require('./classicube-protocol');

Node.js 14+ requerido. Sin dependencias externas.


================================================================
2. ESTRUCTURA DE ARCHIVOS
================================================================

classicube-protocol/
├── index.js                      <- Exporta todo
├── package.json
├── PacketIDs.js                  <- IDs 0x00..0x0F
├── Binary.js                     <- BinaryWriter / BinaryReader
├── server/
│   ├── Server.js                 <- Clase Server de alto nivel
│   └── ServerConnection.js      <- Un jugador conectado
├── client/
│   └── ClientConnection.js      <- Conexion al servidor
├── auth/
│   └── ClassiCubeAuth.js        <- Verificacion con classicube.net
├── packets/                      <- 16 paquetes del protocolo
│   ├── PlayerIdentification.js   0x00 C->S
│   ├── ServerIdentification.js   0x00 S->C
│   ├── Ping.js                   0x01
│   ├── LevelInitialize.js        0x02
│   ├── LevelDataChunk.js         0x03
│   ├── LevelFinalize.js          0x04
│   ├── SetBlock.js               0x05 / 0x06
│   ├── SpawnPlayer.js            0x07
│   ├── PlayerTeleport.js         0x08
│   ├── PlayerUpdate.js           0x09
│   ├── PlayerMove.js             0x0A
│   ├── PlayerRotate.js           0x0B
│   ├── DespawnPlayer.js          0x0C
│   ├── ChatMessage.js            0x0D
│   ├── Disconnect.js             0x0E
│   └── UpdateUserType.js         0x0F
└── utils/
    ├── Level.js                  <- Mundos + GZIP automatico
    └── Debug.js                  <- Logs con DEBUG=cc-protocol


================================================================
3. FLUJO DE CONEXION
================================================================

Cliente                              Servidor
  |                                    |
  |-- PlayerIdentification (0x00) ---->|  Login
  |<-- ServerIdentification (0x00) ----|  Nombre + MOTD
  |<-- LevelInitialize (0x02) ---------|  Inicio del mapa
  |<-- LevelDataChunk (0x03) x N ------|  Chunks gzip 1024B c/u
  |<-- LevelFinalize (0x04) -----------|  Dimensiones X Y Z
  |<-- SpawnPlayer (0x07) x N ---------|  Jugadores existentes
  |<-- SpawnPlayer (0x07) id=-1 -------|  Tu propio spawn
  |                                    |
  |         -- Juego activo --          |
  |                                    |
  |-- PlayerTeleport (0x08) ---------->|  Tu posicion (absoluta)
  |-- SetBlock (0x05) ---------------->|  Colocar / romper bloque
  |-- ChatMessage (0x0D) ------------->|  Mensaje de chat
  |<-- PlayerUpdate (0x09) ------------|  Delta pos + orientacion
  |<-- PlayerMove (0x0A) --------------|  Solo delta posicion
  |<-- PlayerRotate (0x0B) ------------|  Solo orientacion
  |<-- SetBlock (0x06) ----------------|  Bloque cambiado
  |<-- ChatMessage (0x0D) -------------|  Chat de otros
  |<-- Ping (0x01) --------------------|  Keep-alive ~5s
  |<-- DespawnPlayer (0x0C) -----------|  Jugador salio
  |<-- UpdateUserType (0x0F) ----------|  Cambio de permisos op


================================================================
4. createServer — SERVIDOR EN 5 LINEAS
================================================================

--------------------------------------------------------------
const { createServer, Level } = require('classicube-protocol');

const server = createServer({ port: 25565, serverName: 'Mi Servidor' });
const world  = Level.createFlat(64, 64, 64);

server.on('login', client => {
  client.write('serverIdentification', { name: server.serverName, motd: 'Hola!' });
  client.sendLevel(world);
  client.sendLevelFinalize({ xSize: 64, ySize: 64, zSize: 64 });
  client.write('spawnPlayer', { playerId: -1, name: client.username, x: 32, y: 36, z: 32 });
});

server.on('chat', (client, message) => {
  server.broadcastChat('<' + client.username + '> ' + message);
});

await server.listen();
--------------------------------------------------------------


================================================================
5. createClient — CLIENTE EN 5 LINEAS
================================================================

--------------------------------------------------------------
const { createClient, Level } = require('classicube-protocol');

const client = await createClient({
  host: 'localhost',
  port: 25565,
  username: 'Steve',
});

client.on('chatMessage', ({ message }) => console.log(message));

// blocks ya viene decodificado, sin zlib manual
client.on('levelFinalize', ({ xSize, ySize, zSize, blocks }) => {
  console.log('Mapa:', xSize + 'x' + ySize + 'x' + zSize);
  const tipo = Level.getBlock(blocks, 32, 33, 32, xSize, zSize);
  console.log('Bloque en (32,33,32):', tipo);
});

client.write('chat', { message: 'Hola!' });
--------------------------------------------------------------


================================================================
6. ping — CONSULTAR INFO DE UN SERVIDOR
================================================================

--------------------------------------------------------------
const { ping } = require('classicube-protocol');

const info = await ping({ host: 'localhost', port: 25565 });
console.log('Servidor:', info.name);
console.log('MOTD:',     info.motd);
console.log('Op:',       info.op);

// Con timeout personalizado (default 5000ms)
const info2 = await ping({ host: 'classicube.net', port: 25565, timeout: 3000 });
--------------------------------------------------------------


================================================================
7. SERVIDOR COMPLETO
================================================================

Guarda como server.js

--------------------------------------------------------------
const { createServer, Level } = require('classicube-protocol');

const WORLD = { xSize: 64, ySize: 64, zSize: 64 };
const world = Level.createFlat(WORLD.xSize, WORLD.ySize, WORLD.zSize);

const server = createServer({
  host:         '0.0.0.0',
  port:         25565,
  serverName:   'Mi Servidor ClassiCube',
  motd:         'Bienvenido!',
  maxPlayers:   20,
  online:       false,   // true = verificar con classicube.net
  pingInterval: 5000,    // ms entre pings, 0 = desactivar
});

// ── Login ─────────────────────────────────────────────────
server.on('login', client => {
  console.log('[+]', client.username);

  // 1. Handshake
  client.write('serverIdentification', {
    name: server.serverName,
    motd: server.motd,
    op:   false,
  });

  // 2. Mapa — sendLevel acepta raw, gzip o Level.createFlat()
  client.sendLevel(world);
  client.sendLevelFinalize(WORLD);

  // 3. Spawnear jugadores ya conectados para el nuevo
  for (const [id, other] of server.players) {
    if (id !== client.id) {
      client.write('spawnPlayer', {
        playerId: id,   name: other.username,
        x: other.x,    y: other.y,    z: other.z,
        yaw: other.yaw, pitch: other.pitch,
      });
    }
  }

  // 4. Spawn del propio jugador (playerId -1 = self)
  client.write('spawnPlayer', {
    playerId: -1, name: client.username,
    x: 32, y: 36, z: 32,
  });

  // 5. Notificar a los demas del nuevo jugador
  server.broadcast('spawnPlayer', {
    playerId: client.id, name: client.username,
    x: 32, y: 36, z: 32,
  }, client.id);

  // 6. Mensajes de bienvenida
  client.write('chat', { playerId: -1, message: '&aBienvenido ' + client.username + '!' });
  server.broadcastChat('&e' + client.username + ' &fentra al servidor', -1, client.id);
});

// ── Chat ──────────────────────────────────────────────────
server.on('chat', (client, message) => {
  if (message.startsWith('/')) { handleCommand(client, message); return; }
  const msg = '<' + client.username + '> ' + message;
  console.log(msg);
  server.broadcastChat(msg, client.id);
});

// ── Movimiento ────────────────────────────────────────────
server.on('playerMove', (client, { x, y, z, yaw, pitch }) => {
  // client.x/y/z es la posicion ANTERIOR (se actualiza despues)
  const dx = x - client.x;
  const dy = y - client.y;
  const dz = z - client.z;

  if (Math.abs(dx) < 4 && Math.abs(dy) < 4 && Math.abs(dz) < 4) {
    const moved   = dx !== 0 || dy !== 0 || dz !== 0;
    const rotated = yaw !== client.yaw || pitch !== client.pitch;

    if      (moved && rotated) server.broadcast('playerUpdate',  { playerId: client.id, dx, dy, dz, yaw, pitch }, client.id);
    else if (moved)            server.broadcast('playerMove',    { playerId: client.id, dx, dy, dz }, client.id);
    else if (rotated)          server.broadcast('playerRotate',  { playerId: client.id, yaw, pitch }, client.id);
  } else {
    server.broadcast('playerTeleport', { playerId: client.id, x, y, z, yaw, pitch }, client.id);
  }
});

// ── Bloques ───────────────────────────────────────────────
server.on('setBlock', (client, { x, y, z, mode, blockType }) => {
  const block = mode === 0 ? 0 : blockType;
  server.broadcast('setBlock', { x, y, z, blockType: block });
});

// ── Desconexion ───────────────────────────────────────────
server.on('playerLeave', client => {
  console.log('[-]', client.username);
  server.broadcast('despawnPlayer', { playerId: client.id });
  server.broadcastChat('&e' + client.username + ' &fsale del servidor');
});

server.on('clientError', (client, err) => {
  console.error('Error en', client.username || '???', ':', err.message);
});

// ── Comandos ──────────────────────────────────────────────
function handleCommand(client, message) {
  const args = message.trim().split(/\s+/);
  const cmd  = args[0].toLowerCase();

  switch (cmd) {
    case '/tp': {
      const [x, y, z] = args.slice(1).map(Number);
      if ([x, y, z].some(isNaN)) {
        client.write('chat', { playerId: -1, message: 'Uso: /tp <x> <y> <z>' });
      } else {
        client.write('playerTeleport', { playerId: -1, x, y, z });
        client.write('chat', { playerId: -1, message: 'Teletransportado a ' + x + ' ' + y + ' ' + z });
      }
      break;
    }
    case '/op':
      client.sendUpdateUserType(true);
      client.write('chat', { playerId: -1, message: '&aAhora eres operador' });
      break;
    case '/deop':
      client.sendUpdateUserType(false);
      client.write('chat', { playerId: -1, message: 'Ya no eres operador' });
      break;
    case '/kick': {
      const target = args[1];
      const reason = args.slice(2).join(' ') || 'Kicked';
      let found = false;
      for (const other of server.players.values()) {
        if (other.username === target) {
          other.kick(reason);
          client.write('chat', { playerId: -1, message: '&c' + target + ' fue kickeado: ' + reason });
          found = true; break;
        }
      }
      if (!found) client.write('chat', { playerId: -1, message: 'Jugador no encontrado: ' + target });
      break;
    }
    case '/players': {
      const list = [...server.players.values()].map(p => p.username).join(', ');
      client.write('chat', { playerId: -1, message: 'Online (' + server.players.size + '): ' + list });
      break;
    }
    default:
      client.write('chat', { playerId: -1, message: 'Comando desconocido: ' + cmd });
  }
}

// ── Iniciar ───────────────────────────────────────────────
server.listen().then(() => {
  console.log('Servidor en :' + server.port);
});
--------------------------------------------------------------

Ejecutar:
  node server.js

Con debug de paquetes:
  DEBUG=cc-protocol:packets node server.js


================================================================
8. CLIENTE COMPLETO
================================================================

Guarda como client.js

--------------------------------------------------------------
const { createClient, Level } = require('classicube-protocol');

async function main() {
  const client = await createClient({
    host:            'localhost',
    port:            25565,
    username:        'MiBot',
    verificationKey: '-',   // '-' para servidores sin auth
  });

  console.log('Conectado! state:', client.state);

  // ── Handshake ─────────────────────────────────────────
  client.on('serverIdentification', ({ name, motd, op }) => {
    console.log('Servidor:', name, '|', motd, '| op:', op);
  });

  client.on('levelInitialize', () => process.stdout.write('Cargando mapa...'));
  client.on('levelDataChunk',  ({ percent }) => process.stdout.write('\rCargando mapa... ' + percent + '%'));

  client.on('levelFinalize', ({ xSize, ySize, zSize, blocks, blockCount }) => {
    console.log('\nMapa: ' + xSize + 'x' + ySize + 'x' + zSize + ' (' + blockCount + ' bloques)');
    // blocks ya viene decodificado — sin zlib.gunzipSync manual
    const tipo = Level.getBlock(blocks, 32, 33, 32, xSize, zSize);
    console.log('Bloque en (32,33,32):', tipo);
  });

  // ── Jugadores ─────────────────────────────────────────
  client.on('spawnPlayer', ({ playerId, name, x, y, z }) => {
    if (playerId === -1) {
      console.log('Spawneado en', x, y, z);
      console.log('Mi username:', client.username);
    } else {
      console.log('Jugador', name, 'en', x, y, z);
    }
  });

  // client.players se actualiza automaticamente en todos
  // los eventos de spawn/despawn/movimiento
  client.on('despawnPlayer', ({ playerId }) => {
    console.log('Jugador', playerId, 'salio');
  });

  // ── Permisos ──────────────────────────────────────────
  client.on('updateUserType', ({ op }) => {
    console.log('Permisos:', op ? 'operador' : 'normal');
    // client.op ya se actualizo automaticamente
  });

  // ── Chat y bloques ────────────────────────────────────
  client.on('chatMessage', ({ message }) => console.log(message));
  client.on('setBlock', ({ x, y, z, blockType }) => { /* actualizar copia local */ });

  // ── Desconexion ───────────────────────────────────────
  client.on('disconnect', ({ reason }) => console.log('Kickeado:', reason));
  client.on('close',      ()           => console.log('Conexion cerrada'));
  client.on('error',      err          => console.error('Error:', err.message));

  // ── Acciones del bot ──────────────────────────────────
  setTimeout(() => client.write('chat', { message: 'Hola!' }), 2000);

  setTimeout(() => {
    // client.x, client.y, client.z se actualizan automaticamente
    client.sendTeleport({ x: 35, y: 36, z: 35, yaw: 64, pitch: 0 });
  }, 5000);

  setTimeout(() => {
    client.write('setBlock', { x: 30, y: 33, z: 30, mode: 1, blockType: Level.BlockType.STONE });
  }, 8000);

  setTimeout(() => {
    client.write('setBlock', { x: 30, y: 33, z: 30, mode: 0, blockType: 0 });
  }, 11000);
}

main().catch(console.error);
--------------------------------------------------------------

Ejecutar (con el servidor corriendo):
  node client.js


================================================================
9. Level — MUNDOS Y BLOQUES SIN ZLIB MANUAL
================================================================

Level resuelve todo el GZIP automaticamente en ambas
direcciones: al enviar el mapa y al recibirlo.

--------------------------------------------------------------
const { Level } = require('classicube-protocol');

// ── Generar mundos listos para sendLevel() ────────────────

Level.createFlat(64, 64, 64)     // piedra / dirt / grass + aire
Level.createEmpty(64, 64, 64)    // todo aire
Level.createBedrock(64, 64, 64)  // suelo bedrock y=0, resto aire

// Todos devuelven Buffer gzipeado, default 64x64x64

// ── Mundo personalizado ───────────────────────────────────

const blocks = Buffer.alloc(64 * 64 * 64, Level.BlockType.AIR);

// Poner suelo de piedra en y=0
for (let z = 0; z < 64; z++)
  for (let x = 0; x < 64; x++)
    Level.setBlock(blocks, x, 0, z, 64, 64, Level.BlockType.STONE);

// Poner una caja de vidrio en el centro
for (let y = 5; y <= 10; y++)
  for (let z = 28; z <= 36; z++)
    for (let x = 28; x <= 36; x++)
      Level.setBlock(blocks, x, y, z, 64, 64, Level.BlockType.GLASS);

// Level.encode() convierte los bloques a gzip
conn.sendLevel(Level.encode(blocks));
conn.sendLevelFinalize({ xSize: 64, ySize: 64, zSize: 64 });

// ── sendLevel acepta cualquier forma ─────────────────────

conn.sendLevel(Level.createFlat());      // gzip de createFlat
conn.sendLevel(Level.encode(blocks));    // gzip de encode()
conn.sendLevel(blocks);                  // Buffer crudo (lo gzipea solo)
conn.sendLevel(bufferYaGzipeado);        // gzip manual (sigue funcionando)

// ── Leer bloques en el cliente ────────────────────────────

client.on('levelFinalize', ({ xSize, ySize, zSize, blocks, blockCount, levelData }) => {
  // blocks     = Buffer de IDs ya descomprimido (SIN zlib.gunzipSync)
  // blockCount = xSize * ySize * zSize
  // levelData  = Buffer gzipeado crudo (compatibilidad con codigo viejo)

  const tipo = Level.getBlock(blocks, x, y, z, xSize, zSize);
  if (tipo === Level.BlockType.GRASS) console.log('Es cesped!');
});

// ── Decodificar manualmente (si tienes el gzip guardado) ──

const { blocks, blockCount } = Level.decode(levelData);
const tipo = Level.getBlock(blocks, 10, 5, 10, xSize, zSize);

// ── Detectar si un buffer ya esta gzipeado ────────────────

Level.isGzipped(buffer)        // true / false (magic bytes 1F 8B)
Level.ensureGzipped(buffer)    // gzipea si no lo esta, si ya lo es lo devuelve igual
--------------------------------------------------------------


================================================================
10. client.write() Y server.broadcast()
================================================================

client.write(packetName, data)
──────────────────────────────
Disponible en ServerConnection y ClientConnection.
Serializa y envia cualquier paquete por nombre.
Lanza Error si el nombre no existe.

  Nombres validos en ServerConnection (servidor -> cliente):
  ┌──────────────────────┬─────────────────────────────────────────┐
  │ Nombre               │ Campos de data                          │
  ├──────────────────────┼─────────────────────────────────────────┤
  │ serverIdentification │ name, motd, op                          │
  │ ping                 │ (ninguno)                               │
  │ levelInitialize      │ (ninguno)                               │
  │ levelDataChunk       │ chunkData, chunkLength, percent         │
  │ levelFinalize        │ xSize, ySize, zSize                     │
  │ spawnPlayer          │ playerId, name, x, y, z, yaw, pitch     │
  │ playerTeleport       │ playerId, x, y, z, yaw, pitch           │
  │ playerUpdate         │ playerId, dx, dy, dz, yaw, pitch        │
  │ playerMove           │ playerId, dx, dy, dz                    │
  │ playerRotate         │ playerId, yaw, pitch                    │
  │ despawnPlayer        │ playerId                                │
  │ setBlock             │ x, y, z, blockType                      │
  │ chat / chatMessage   │ playerId, message                       │
  │ disconnect           │ reason                                  │
  │ updateUserType       │ op                                      │
  └──────────────────────┴─────────────────────────────────────────┘

  Nombres validos en ClientConnection (cliente -> servidor):
  ┌──────────────────────┬─────────────────────────────────────────┐
  │ playerIdentification │ username, verificationKey               │
  │ setBlock             │ x, y, z, mode, blockType                │
  │ playerTeleport       │ x, y, z, yaw, pitch                     │
  │ chat / chatMessage   │ message                                 │
  └──────────────────────┴─────────────────────────────────────────┘

server.broadcast(packetName, data, exceptId)
────────────────────────────────────────────
Envia el paquete a todos en server.players.
exceptId es opcional: excluye ese id.

  server.broadcast('chat',     { playerId: -1, message: 'Hola!' });
  server.broadcast('setBlock', { x, y, z, blockType }, client.id);
  server.broadcast('despawnPlayer', { playerId: client.id });

server.broadcastChat(message, playerId, exceptId)
  Atajo para broadcast de chat. playerId default -1.

  server.broadcastChat('Servidor reiniciando en 30s');
  server.broadcastChat('<Steve> hola', steveId, steveId);


================================================================
11. AUTENTICACION CON CLASSICUBE
================================================================

Modo offline (default):
  Acepta cualquier username sin verificar.
  Perfecto para servidores LAN o desarrollo.

  const server = createServer({ online: false });

Modo online:
  Verifica cada login contra la API de classicube.net.
  Si falla, kickea al jugador automaticamente.

  const server = createServer({
    online: true,
    serverName: 'MiServidor',  // debe coincidir con classicube.net
  });

Uso manual del modulo auth:
--------------------------------------------------------------
const { auth } = require('classicube-protocol');

// Verificar contra la API de classicube.net
const result = await auth.verify({
  serverName:      'MiServidor',
  username:        'Steve',
  verificationKey: keyDelCliente,
  ip:              '123.45.67.89',  // opcional
});

if (result.valid) {
  console.log('Bienvenido,', result.username);
} else {
  conn.kick('Authentication failed: ' + result.error);
}

// Verificacion offline (siempre acepta)
const r = auth.verifyOffline('Steve');
// => { valid: true, username: 'Steve' }

// Generar salt para sistemas propios de auth
const salt = auth.generateSalt(16);
// => String aleatorio de 16 chars alfanumericos
--------------------------------------------------------------


================================================================
12. DEBUG MODE
================================================================

Activa logs internos sin tocar el codigo:

  # Todo
  DEBUG=cc-protocol node server.js

  # Solo paquetes enviados/recibidos
  DEBUG=cc-protocol:packets node server.js

  # Solo eventos del servidor
  DEBUG=cc-protocol:server node server.js

  # Solo el cliente
  DEBUG=cc-protocol:client node server.js

  # Solo autenticacion
  DEBUG=cc-protocol:auth node server.js

  # Multiples namespaces
  DEBUG=cc-protocol:server,cc-protocol:auth node server.js

Ejemplo de salida:
  [12:34:56.789] [cc-protocol:server]  Nueva conexion desde 127.0.0.1
  [12:34:56.791] [cc-protocol:packets] <- playerIdentification username=Steve
  [12:34:56.792] [cc-protocol:auth]    Modo offline, aceptando: Steve
  [12:34:56.793] [cc-protocol:packets] -> serverIdentification { name: 'Mi Servidor' }
  [12:34:56.800] [cc-protocol:packets] -> chat { message: 'Bienvenido Steve!' }


================================================================
13. REFERENCIA: Server
================================================================

Crear:
  const server = createServer(opts);

Opciones de createServer(opts):
  host          String   '0.0.0.0'               IP de escucha
  port          Number   25565                   Puerto TCP
  serverName    String   'ClassiCube Server'
  motd          String   'Powered by classicube-protocol'
  maxPlayers    Number   20
  online        Boolean  false                   Verificar classicube.net
  pingInterval  Number   5000                    Ms entre pings (0=off)

Propiedades:
  server.players    Map<id, ServerConnection>   Jugadores activos
  server.port       Number
  server.host       String
  server.serverName String
  server.motd       String
  server.online     Boolean

Metodos:
  await server.listen()
    Inicia el servidor. Emite 'listening' al terminar.

  await server.close(reason)
    Kickea a todos y cierra. reason default: 'Server closed'

  server.broadcast(packetName, data, exceptId)
    Envia paquete a todos. exceptId opcional.

  server.broadcastChat(message, playerId, exceptId)
    Atajo de broadcast para chat. playerId default -1.

Eventos:
  'listening'    ({ host, port })
  'connect'      (client)          Socket conectado, antes del login
  'login'        (client)          Autenticado, client.username listo
  'chat'         (client, message) Mensaje de chat recibido
  'setBlock'     (client, { x, y, z, mode, blockType })
  'playerMove'   (client, { x, y, z, yaw, pitch })
  'playerLeave'  (client)          Desconectado, ya fuera de server.players
  'disconnect'   (client)          Cualquier desconexion
  'clientError'  (client, err)


================================================================
14. REFERENCIA: ServerConnection
================================================================

Propiedades:
  conn.id          Number    ID asignado por el servidor
  conn.username    String    Username del jugador
  conn.name        String    Alias de username
  conn.state       String    'connecting' | 'playing' | 'disconnected'
  conn.op          Boolean   Tiene permisos de operador
  conn.x/y/z       Number    Posicion actual (auto-actualizado)
  conn.yaw/pitch   Number    Orientacion actual (auto-actualizado)

Metodos:
  conn.write(packetName, data)         Ver seccion 10
  conn.sendServerIdentification({ name, motd, op })
  conn.sendPing()
  conn.sendLevel(data)                 Raw, gzip o Level.createFlat()
  conn.sendLevelFinalize({ xSize, ySize, zSize })
  conn.sendSpawnPlayer({ playerId, name, x, y, z, yaw, pitch })
  conn.sendTeleport({ playerId, x, y, z, yaw, pitch })
  conn.sendPlayerUpdate({ playerId, dx, dy, dz, yaw, pitch })
  conn.sendPlayerMove({ playerId, dx, dy, dz })
  conn.sendPlayerRotate({ playerId, yaw, pitch })
  conn.sendDespawnPlayer({ playerId })
  conn.sendSetBlock({ x, y, z, blockType })
  conn.sendChat(message, playerId)
  conn.sendUpdateUserType(op)          Actualiza conn.op automaticamente
  conn.sendDisconnect(reason)
  conn.kick(reason)                    Alias de sendDisconnect()
  conn.end(reason)                     Alias de kick()
  conn.startPingLoop(intervalMs)       Default 5000ms, auto-cancela al cerrar

Eventos (uso directo sin Server):
  'playerIdentification'  { protocolVersion, username, verificationKey }
  'playerTeleport'        { playerId, x, y, z, yaw, pitch }
  'setBlock'              { x, y, z, mode, blockType }
  'chatMessage'           { playerId, message }
  'close', 'error', 'unknownPacket'


================================================================
15. REFERENCIA: ClientConnection
================================================================

Crear:
  const client = await createClient({ host, port, username });
  // o manual:
  const client = new ClientConnection();
  await client.connect(host, port);
  client.sendIdentification({ username, verificationKey });

Propiedades:
  client.state       String    'disconnected' | 'connecting' | 'playing'
  client.username    String    Se asigna en spawnPlayer id=-1
  client.name        String    Alias de username
  client.op          Boolean   Auto-actualizado en updateUserType
  client.x/y/z       Number    Auto-actualizado en sendTeleport
  client.yaw/pitch   Number    Auto-actualizado en sendTeleport
  client.players     Map<playerId, { name, x, y, z, yaw, pitch }>
                     Auto-actualizado en spawn/despawn/movimiento

Metodos:
  await client.connect(host, port)
  client.disconnect()
  client.end()                          Alias de disconnect()
  client.write(packetName, data)        Ver seccion 10
  client.sendIdentification({ username, verificationKey })
  client.sendTeleport({ x, y, z, yaw, pitch })
  client.sendSetBlock({ x, y, z, mode, blockType })
  client.sendChat(message)

Eventos:
  'serverIdentification'  { protocolVersion, name, motd, op }
  'ping'                  ()
  'levelInitialize'       ()
  'levelDataChunk'        { chunkLength, chunkData, percent }
  'levelFinalize'         { xSize, ySize, zSize,
                            blocks,     <- Buffer de IDs decodificado
                            blockCount, <- xSize*ySize*zSize
                            levelData } <- gzip crudo (compatibilidad)
  'spawnPlayer'           { playerId, name, x, y, z, yaw, pitch }
  'playerTeleport'        { playerId, x, y, z, yaw, pitch }
  'playerUpdate'          { playerId, dx, dy, dz, yaw, pitch }
  'playerMove'            { playerId, dx, dy, dz }
  'playerRotate'          { playerId, yaw, pitch }
  'despawnPlayer'         { playerId }
  'setBlock'              { x, y, z, blockType }
  'chatMessage'           { playerId, message }
  'disconnect'            { reason }
  'updateUserType'        { userType, op }
  'close', 'error', 'unknownPacket'


================================================================
16. REFERENCIA: Level
================================================================

Generadores (devuelven Buffer gzipeado):
  Level.createFlat(xSize, ySize, zSize)
    Piedra / dirt / grass. Default 64x64x64.
    La mitad inferior es solida, la superior es aire.

  Level.createEmpty(xSize, ySize, zSize)
    Todo aire.

  Level.createBedrock(xSize, ySize, zSize)
    y=0 bedrock, resto aire. Para arenas / minijuegos.

Encode / decode:
  Level.encode(blocks)
    Buffer de IDs (xSize*ySize*zSize bytes) -> Buffer gzipeado.

  Level.decode(levelData)
    Buffer gzipeado -> { blocks: Buffer, blockCount: Number }

Leer / escribir bloques:
  Level.getBlock(blocks, x, y, z, xSize, zSize)
    Lee el ID en la posicion. Indice: y*zSize*xSize + z*xSize + x

  Level.setBlock(blocks, x, y, z, xSize, zSize, blockId)
    Escribe el ID en la posicion.

Utilidades:
  Level.isGzipped(buf)       true si empieza con magic bytes 1F 8B
  Level.ensureGzipped(data)  Gzipea si es raw, devuelve igual si ya es gzip

IDs:
  Level.BlockType            Objeto con los 50 IDs. Ver seccion 17.


================================================================
17. IDS DE BLOQUES (Level.BlockType)
================================================================

  AIR           0    GOLD_ORE     14    TEAL_CLOTH   26
  STONE         1    IRON_ORE     15    CYAN_CLOTH   27
  GRASS         2    COAL_ORE     16    BLUE_CLOTH   28
  DIRT          3    LOG          17    PURPLE_CLOTH 29
  COBBLESTONE   4    LEAVES       18    INDIGO_CLOTH 30
  WOOD          5    SPONGE       19    VIOLET_CLOTH 31
  SAPLING       6    GLASS        20    MAGENTA_CLOTH 32
  BEDROCK       7    RED_CLOTH    21    PINK_CLOTH   33
  WATER_FLOW    8    ORANGE_CLOTH 22    BLACK_CLOTH  34
  WATER         9    YELLOW_CLOTH 23    GRAY_CLOTH   35
  LAVA_FLOW    10    LIME_CLOTH   24    WHITE_CLOTH  36
  LAVA         11    GREEN_CLOTH  25    DANDELION    37
  SAND         12                       ROSE         38
  GRAVEL       13                       BROWN_MUSH   39
                                        RED_MUSH     40
                                        GOLD         41
                                        IRON         42
                                        DOUBLE_SLAB  43
                                        SLAB         44
                                        BRICK        45
                                        TNT          46
                                        BOOKSHELF    47
                                        MOSSY_STONE  48
                                        OBSIDIAN     49

Usar por nombre:
  Level.BlockType.STONE      // 1
  Level.BlockType.GLASS      // 20
  Level.BlockType.OBSIDIAN   // 49
  Level.BlockType.AIR        // 0


================================================================
18. COLORES EN EL CHAT
================================================================

  &0  Negro        &8  Gris oscuro
  &1  Azul oscuro  &9  Azul
  &2  Verde oscuro &a  Verde
  &3  Cian oscuro  &b  Cian
  &4  Rojo oscuro  &c  Rojo
  &5  Purpura      &d  Rosado
  &6  Naranja      &e  Amarillo
  &7  Gris         &f  Blanco

Ejemplos:
  client.write('chat', { playerId: -1, message: '&aServidor listo!' });
  client.write('chat', { playerId: -1, message: '&c' + name + ' fue kickeado' });
  server.broadcastChat('&e[Anuncio] &fReinicio en 5 minutos');
  server.broadcastChat('&bNuevo record: &f' + player + ' llego al nivel ' + level);


================================================================
19. MOVIMIENTO: CUANDO USAR CADA PAQUETE
================================================================

Regla:
  Delta < 4 bloques y gira     ->  playerUpdate   (0x09)  7 bytes
  Delta < 4 bloques y no gira  ->  playerMove     (0x0A)  5 bytes
  Solo gira, no se mueve       ->  playerRotate   (0x0B)  4 bytes
  Delta >= 4 bloques           ->  playerTeleport (0x08) 10 bytes

Implementacion:
  server.on('playerMove', (client, { x, y, z, yaw, pitch }) => {
    // client.x/y/z es la posicion ANTERIOR: perfecta para el delta
    const dx = x - client.x;
    const dy = y - client.y;
    const dz = z - client.z;

    if (Math.abs(dx) < 4 && Math.abs(dy) < 4 && Math.abs(dz) < 4) {
      const moved   = dx !== 0 || dy !== 0 || dz !== 0;
      const rotated = yaw !== client.yaw || pitch !== client.pitch;

      if      (moved && rotated) server.broadcast('playerUpdate',  { playerId: client.id, dx, dy, dz, yaw, pitch }, client.id);
      else if (moved)            server.broadcast('playerMove',    { playerId: client.id, dx, dy, dz }, client.id);
      else if (rotated)          server.broadcast('playerRotate',  { playerId: client.id, yaw, pitch }, client.id);
    } else {
      server.broadcast('playerTeleport', { playerId: client.id, x, y, z, yaw, pitch }, client.id);
    }
    // IMPORTANTE: client.x/y/z se actualizan DESPUES de este handler
  });


================================================================
20. ERRORES COMUNES
================================================================

Desconexion inmediata al conectar
  -> Enviaste datos antes del evento 'login'
  -> verificationKey mayor a 64 caracteres
  -> Protocol version no es 7

El cliente no recibe el mundo
  -> Falta llamar sendLevelFinalize() despues de sendLevel()
     sendLevel() NO lo llama automaticamente

Error: Unknown packet name "X"
  -> Nombre invalido en write()
  -> Ver seccion 10 para la lista completa

Error: Unknown client/server packet ID 0xXX
  -> Paquetes TCP desalineados
  -> El cliente usa CPE (extensiones no soportadas)

El jugador spawneado no se ve
  -> Envia SpawnPlayer con playerId -1 al propio jugador
  -> Envia SpawnPlayer con el id real a los demas
  -> No confundas los dos: self siempre es -1

El chat no llega al remitente
  -> No uses exceptId en el broadcast de chat
  -> El remitente debe ver su propio mensaje

client.players esta vacio
  -> Se llena en el evento 'spawnPlayer'
  -> Antes de levelFinalize no hay jugadores spawneados aun

client.username es null
  -> Se asigna en el evento spawnPlayer con playerId -1
  -> Espera ese evento antes de leer client.username

sendLevel muy lento con mundos grandes
  -> Cachea el Buffer gzipeado, no lo regeneres en cada login
  -> const world = Level.createFlat(256, 64, 256); // generar 1 vez
  -> server.on('login', client => client.sendLevel(world)); // reusar


================================================================
  classicube-protocol · MIT License · Node.js 14+
================================================================
