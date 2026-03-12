# classicube-protocol



----------------------------------------------------------------
1. INSTALACION
----------------------------------------------------------------

Desde npm (cuando lo subas):
  npm install protocol-classicube



O copiando la carpeta directo al proyecto:
  const { ServerConnection } = require('./protocol-classicube');


----------------------------------------------------------------
2. ESTRUCTURA DE ARCHIVOS
----------------------------------------------------------------

protocol-classicube/
├── index.js              <- Punto de entrada, exporta todo
├── package.json
├── PacketIDs.js          <- IDs de paquetes (SERVER / CLIENT)
├── Binary.js             <- BinaryWriter y BinaryReader
├── server/
│   └── ServerConnection.js  <- Wrapper socket lado servidor
├── client/
│   └── ClientConnection.js  <- Wrapper socket lado cliente
└── packets/
    ├── PlayerIdentification.js  <- 0x00 C->S
    ├── ServerIdentification.js  <- 0x00 S->C
    ├── Ping.js                  <- 0x01
    ├── LevelInitialize.js       <- 0x02
    ├── LevelDataChunk.js        <- 0x03
    ├── LevelFinalize.js         <- 0x04
    ├── SetBlock.js              <- 0x05 y 0x06
    ├── SpawnPlayer.js           <- 0x07
    ├── PlayerTeleport.js        <- 0x08
    ├── PlayerUpdate.js          <- 0x09
    ├── PlayerMove.js            <- 0x0A
    ├── PlayerRotate.js          <- 0x0B
    ├── DespawnPlayer.js         <- 0x0C
    ├── ChatMessage.js           <- 0x0D
    ├── Disconnect.js            <- 0x0E
    └── UpdateUserType.js        <- 0x0F


----------------------------------------------------------------
3. FLUJO DE CONEXION
----------------------------------------------------------------

Cliente                              Servidor
  |                                    |
  |-- PlayerIdentification (0x00) ---->|  Login
  |<-- ServerIdentification (0x00) ----|  Nombre + MOTD
  |<-- LevelInitialize (0x02) ---------|  "Prepárate para el mapa"
  |<-- LevelDataChunk (0x03) x N ------|  Datos gzip en chunks 1024B
  |<-- LevelFinalize (0x04) -----------|  Dimensiones X, Y, Z
  |<-- SpawnPlayer (0x07) x N ---------|  Spawn jugadores existentes
  |<-- SpawnPlayer (0x07) id=-1 -------|  Tu propio spawn
  |                                    |
  |         -- Juego en curso --        |
  |                                    |
  |-- PlayerTeleport (0x08) ---------->|  Tu posicion
  |-- SetBlock (0x05) ---------------->|  Colocar/romper bloque
  |-- ChatMessage (0x0D) ------------->|  Mensaje de chat
  |<-- PlayerUpdate (0x09) ------------|  Movimiento delta de otros
  |<-- PlayerMove (0x0A) --------------|  Solo posicion delta
  |<-- PlayerRotate (0x0B) ------------|  Solo orientacion
  |<-- ChatMessage (0x0D) -------------|  Chat de otros jugadores
  |<-- SetBlock (0x06) ----------------|  Cambios de bloque
  |<-- Ping (0x01) --------------------|  Keep-alive cada ~5s
  |<-- DespawnPlayer (0x0C) -----------|  Jugador salio
  |<-- UpdateUserType (0x0F) ----------|  Cambio de permisos


----------------------------------------------------------------
4. SERVIDOR COMPLETO
----------------------------------------------------------------

Guarda esto como server.js en la raiz de tu proyecto.

--------------------------------------------------------------
const net  = require('net');
const zlib = require('zlib');
const { ServerConnection, packets } = require('protocol-classicube');

// ── Generar un mundo flat 64x64x64 ───────────────────────
function buildWorld(xSize, ySize, zSize) {
  const blockCount = xSize * ySize * zSize;
  const raw = Buffer.alloc(4 + blockCount);
  raw.writeInt32BE(blockCount, 0);
  for (let y = 0; y < ySize / 2; y++) {
    for (let z = 0; z < zSize; z++) {
      for (let x = 0; x < xSize; x++) {
        const idx = 4 + (y * zSize * xSize + z * xSize + x);
        raw[idx] = y < ySize / 2 - 1 ? 1 : 2; // piedra / cesped
      }
    }
  }
  return zlib.gzipSync(raw);
}

const WORLD = { x: 64, y: 64, z: 64 };
const gzWorld = buildWorld(WORLD.x, WORLD.y, WORLD.z);
const players = new Map(); // id => ServerConnection
let   nextId  = 0;

function broadcast(buf, exceptId = null) {
  for (const [id, conn] of players) {
    if (id !== exceptId) conn._write(buf);
  }
}

// ── Servidor TCP ──────────────────────────────────────────
const server = net.createServer(socket => {
  const conn     = new ServerConnection(socket);
  const playerId = nextId++;
  conn.id        = playerId;

  conn.on('playerIdentification', ({ username }) => {
    console.log('[+]', username, '(id=' + playerId + ')');
    conn.name = username;
    players.set(playerId, conn);

    // 1. Handshake
    conn.sendServerIdentification({
      name: 'Mi Servidor ClassiCube',
      motd: 'Bienvenido!',
      op: false,
    });

    // 2. Mapa
    conn.sendLevel(gzWorld);
    conn.sendLevelFinalize({ xSize: WORLD.x, ySize: WORLD.y, zSize: WORLD.z });

    // 3. Spawn jugadores existentes para el nuevo
    for (const [id, other] of players) {
      if (id !== playerId) {
        conn.sendSpawnPlayer({
          playerId: id, name: other.name,
          x: other.x, y: other.y, z: other.z,
          yaw: other.yaw, pitch: other.pitch,
        });
      }
    }

    // 4. Spawn del propio jugador (id -1 = self)
    conn.sendSpawnPlayer({
      playerId: -1, name: username,
      x: 32, y: 36, z: 32,
    });

    // 5. Notificar a los demas
    for (const [id, other] of players) {
      if (id !== playerId) {
        other.sendSpawnPlayer({
          playerId, name: username,
          x: 32, y: 36, z: 32,
        });
      }
    }

    // 6. Ping automatico
    conn.startPingLoop(5000);

    // 7. Mensajes de bienvenida
    conn.sendChat('&aBienvenido ' + username + '!');
    broadcast(packets.ChatMessage.serialize({
      playerId: -1,
      message: '&e' + username + ' &fentra al servidor',
    }), playerId);
  });

  // ── Chat ──────────────────────────────────────────────
  conn.on('chatMessage', ({ message }) => {
    if (message.startsWith('/')) {
      handleCommand(conn, message);
      return;
    }
    const msg = '<' + conn.name + '> ' + message;
    console.log(msg);
    const buf = packets.ChatMessage.serialize({ playerId, message: msg });
    for (const other of players.values()) other._write(buf);
  });

  // ── Movimiento ────────────────────────────────────────
  conn.on('playerTeleport', ({ x, y, z, yaw, pitch }) => {
    const dx = x - conn.x;
    const dy = y - conn.y;
    const dz = z - conn.z;

    if (Math.abs(dx) < 4 && Math.abs(dy) < 4 && Math.abs(dz) < 4) {
      const moved   = dx !== 0 || dy !== 0 || dz !== 0;
      const rotated = yaw !== conn.yaw || pitch !== conn.pitch;
      if (moved && rotated) {
        broadcast(packets.PlayerUpdate.serialize({
          playerId, dx, dy, dz, yaw, pitch,
        }), playerId);
      } else if (moved) {
        broadcast(packets.PlayerMove.serialize({
          playerId, dx, dy, dz,
        }), playerId);
      } else if (rotated) {
        broadcast(packets.PlayerRotate.serialize({
          playerId, yaw, pitch,
        }), playerId);
      }
    } else {
      broadcast(packets.PlayerTeleport.serialize({
        playerId, x, y, z, yaw, pitch,
      }), playerId);
    }
    // conn.x/y/z/yaw/pitch se actualizan solos
  });

  // ── Bloques ───────────────────────────────────────────
  conn.on('setBlock', ({ x, y, z, mode, blockType }) => {
    const block = mode === 0 ? 0 : blockType;
    const buf = packets.SetBlock.server.serialize({ x, y, z, blockType: block });
    for (const other of players.values()) other._write(buf);
  });

  // ── Desconexion ───────────────────────────────────────
  conn.on('close', () => {
    players.delete(playerId);
    console.log('[-]', conn.name || '???');
    if (!conn.name) return;
    broadcast(packets.DespawnPlayer.serialize({ playerId }));
    broadcast(packets.ChatMessage.serialize({
      playerId: -1,
      message: '&e' + conn.name + ' &fsale del servidor',
    }));
  });

  conn.on('error', err => console.error('Error:', err.message));
});

// ── Comandos ──────────────────────────────────────────────
function handleCommand(conn, message) {
  const args = message.trim().split(/\s+/);
  const cmd  = args[0].toLowerCase();

  switch (cmd) {
    case '/tp': {
      const x = parseFloat(args[1]);
      const y = parseFloat(args[2]);
      const z = parseFloat(args[3]);
      if (isNaN(x) || isNaN(y) || isNaN(z)) {
        conn.sendChat('Uso: /tp <x> <y> <z>');
      } else {
        conn.sendTeleport({ playerId: -1, x, y, z });
        conn.sendChat('Teletransportado a ' + x + ' ' + y + ' ' + z);
      }
      break;
    }
    case '/op':
      conn.sendUpdateUserType(true);
      conn.sendChat('&aAhora eres operador');
      break;
    case '/deop':
      conn.sendUpdateUserType(false);
      conn.sendChat('Ya no eres operador');
      break;
    case '/kick': {
      const target = args[1];
      const reason = args.slice(2).join(' ') || 'Kicked';
      let found = false;
      for (const other of players.values()) {
        if (other.name === target) {
          other.kick(reason);
          conn.sendChat('&c' + target + ' fue kickeado: ' + reason);
          found = true;
          break;
        }
      }
      if (!found) conn.sendChat('Jugador ' + target + ' no encontrado');
      break;
    }
    case '/players': {
      const list = [...players.values()].map(p => p.name).join(', ');
      conn.sendChat('Conectados (' + players.size + '): ' + list);
      break;
    }
    default:
      conn.sendChat('Comando desconocido: ' + cmd);
  }
}

server.listen(25565, () => console.log('Servidor en :25565'));
--------------------------------------------------------------

Ejecutar:
  node server.js


----------------------------------------------------------------
5. CLIENTE COMPLETO
----------------------------------------------------------------

Guarda esto como client.js.

--------------------------------------------------------------
const zlib = require('zlib');
const { ClientConnection } = require('protocol-classicube');

async function main() {
  const client = new ClientConnection();

  await client.connect('localhost', 25565);
  console.log('Conectado!');

  client.sendIdentification({
    username: 'MiBot',
    verificationKey: '-',
  });

  client.on('serverIdentification', ({ name, motd }) => {
    console.log('Servidor:', name);
    console.log('MOTD:', motd);
  });

  client.on('levelInitialize', () => {
    process.stdout.write('Cargando mapa...');
  });

  client.on('levelDataChunk', ({ percent }) => {
    process.stdout.write('\rCargando mapa... ' + percent + '%');
  });

  client.on('levelFinalize', ({ xSize, ySize, zSize, levelData }) => {
    console.log('\nMapa: ' + xSize + 'x' + ySize + 'x' + zSize);
    const raw = zlib.gunzipSync(levelData);
    // raw.readInt32BE(0) = cantidad de bloques
    // raw.slice(4)       = array de IDs de bloques
  });

  client.on('spawnPlayer', ({ playerId, name, x, y, z }) => {
    if (playerId === -1) console.log('Spawneado en', x, y, z);
    else console.log('Jugador', name, 'en', x, y, z);
    // client.players.get(playerId) disponible automaticamente
  });

  client.on('playerUpdate',  ({ playerId, dx, dy, dz, yaw, pitch }) => {});
  client.on('playerMove',    ({ playerId, dx, dy, dz }) => {});
  client.on('playerRotate',  ({ playerId, yaw, pitch }) => {});
  client.on('playerTeleport',({ playerId, x, y, z }) => {});

  client.on('despawnPlayer', ({ playerId }) => {
    console.log('Jugador', playerId, 'salio');
    // client.players ya fue actualizado automaticamente
  });

  client.on('updateUserType', ({ op }) => {
    console.log('Permisos:', op ? 'operador' : 'normal');
  });

  client.on('chatMessage',  ({ message }) => console.log(message));
  client.on('setBlock',     ({ x, y, z, blockType }) => {});
  client.on('disconnect',   ({ reason }) => console.log('Kickeado:', reason));
  client.on('close',        () => console.log('Conexion cerrada'));
  client.on('error',        err => console.error('Error:', err.message));

  // Acciones del bot
  setTimeout(() => client.sendChat('Hola!'), 2000);
  setTimeout(() => client.sendTeleport({ x: 35, y: 36, z: 35 }), 5000);
  setTimeout(() => client.sendSetBlock({ x: 30, y: 33, z: 30, mode: 1, blockType: 1 }), 8000);
  setTimeout(() => client.sendSetBlock({ x: 30, y: 33, z: 30, mode: 0, blockType: 0 }), 11000);
}

main().catch(console.error);
--------------------------------------------------------------

Ejecutar (con el servidor corriendo):
  node client.js


----------------------------------------------------------------
6. METODOS DE ServerConnection
----------------------------------------------------------------

sendServerIdentification({ name, motd, op })
  Handshake. Primer paquete que envias al cliente. op default false.

sendPing()
  Keep-alive manual.

startPingLoop(intervalMs = 5000)
  Ping automatico. Se cancela solo al cerrar el socket.

sendLevel(gzBuffer)
  LevelInitialize + todos los LevelDataChunks automatico.
  Llama a sendLevelFinalize() por separado despues.

sendLevelFinalize({ xSize, ySize, zSize })
  Cierra el stream del mapa.

sendSpawnPlayer({ playerId, name, x, y, z, yaw, pitch })
  Spawna jugador. playerId -1 = self. yaw/pitch opcional.

sendTeleport({ playerId, x, y, z, yaw, pitch })
  Posicion absoluta.

sendPlayerUpdate({ playerId, dx, dy, dz, yaw, pitch })
  Delta posicion + orientacion. Deltas < 4 bloques.

sendPlayerMove({ playerId, dx, dy, dz })
  Solo delta posicion.

sendPlayerRotate({ playerId, yaw, pitch })
  Solo orientacion.

sendDespawnPlayer({ playerId })
  Elimina la entidad del cliente.

sendSetBlock({ x, y, z, blockType })
  Notifica cambio de bloque. 0 = aire.

sendChat(message, playerId = -1)
  Chat. -1 = mensaje del servidor.

sendUpdateUserType(op)
  Da/quita op. Actualiza conn.op automaticamente.

sendDisconnect(reason) / kick(reason)
  Kickea y cierra socket.


----------------------------------------------------------------
7. METODOS DE ClientConnection
----------------------------------------------------------------

await connect(host, port)
  Conecta. Lanza error si falla.

disconnect()
  Cierra el socket.

sendIdentification({ username, verificationKey })
  Login. verificationKey es '-' sin auth.

sendTeleport({ x, y, z, yaw, pitch })
  Envia tu posicion. Actualiza client.x/y/z automaticamente.

sendSetBlock({ x, y, z, mode, blockType })
  mode 1 = colocar, mode 0 = romper.

sendChat(message)
  Envia chat al servidor.


----------------------------------------------------------------
8. ESTADO AUTOMATICO
----------------------------------------------------------------

ServerConnection actualiza automaticamente:
  conn.name         <- en playerIdentification
  conn.x/y/z        <- en playerTeleport
  conn.yaw/pitch    <- en playerTeleport
  conn.op           <- en sendUpdateUserType()

ClientConnection actualiza automaticamente:
  client.x/y/z      <- en sendTeleport()
  client.yaw/pitch  <- en sendTeleport()
  client.op         <- en updateUserType
  client.players    <- Map con todos los jugadores online
                       se actualiza en spawn/despawn/teleport/
                       playerUpdate/playerMove/playerRotate


----------------------------------------------------------------
9. LEER BLOQUES DEL MUNDO
----------------------------------------------------------------

  const zlib = require('zlib');

  client.on('levelFinalize', ({ xSize, ySize, zSize, levelData }) => {
    const raw    = zlib.gunzipSync(levelData);
    const count  = raw.readInt32BE(0);
    const blocks = raw.slice(4);

    function getBlock(x, y, z) {
      return blocks[y * zSize * xSize + z * xSize + x];
    }

    console.log(getBlock(32, 33, 32)); // bloque en esa posicion
  });


----------------------------------------------------------------
10. IDS DE BLOQUES CLASICOS
----------------------------------------------------------------

  0  Aire          13  Grava         26  Tela aguamarina
  1  Piedra        14  Mineral oro   27  Tela azul
  2  Cesped        15  Mineral hiero 28  Tela violeta
  3  Tierra        16  Carbon        29  Tela indigo
  4  Adoquin       17  Tronco        30  Tela violeta osc.
  5  Madera        18  Hojas         31  Tela magenta
  6  Arbusto       19  Esponja       32  Tela rosa
  7  Bedrock       20  Vidrio        33  Tela negra
  8  Agua (fluye)  21  Tela roja     34  Tela gris
  9  Agua (stat.)  22  Tela naranja  35  Tela blanca
  10 Lava (fluye)  23  Tela amarilla 36  Flor amarilla
  11 Lava (stat.)  24  Tela lima     37  Flor roja
  12 Arena         25  Tela verde    48  Obsidiana


----------------------------------------------------------------
11. COLORES EN EL CHAT
----------------------------------------------------------------

  &0 Negro     &8 Gris oscuro
  &1 Azul osc. &9 Azul
  &2 Verde osc. &a Verde
  &3 Cian osc. &b Cian
  &4 Rojo osc. &c Rojo
  &5 Purpura   &d Rosado
  &6 Naranja   &e Amarillo
  &7 Gris      &f Blanco

  Ejemplo:
    conn.sendChat('&aServidor iniciado!');
    conn.sendChat('&c' + name + ' fue kickeado');
    conn.sendChat('&e[Info] &fBienvenido');


----------------------------------------------------------------
12. ERRORES COMUNES
----------------------------------------------------------------

Desconexion inmediata al conectar
  -> Enviaste datos antes del evento playerIdentification
  -> verificationKey mayor a 64 caracteres
  -> Protocol version no es 7

El cliente no recibe el mundo
  -> Falta llamar sendLevelFinalize() despues de sendLevel()
     sendLevel() NO lo llama automaticamente

Error "Unknown client packet ID: 0xXX"
  -> Paquetes desalineados o cliente usa CPE no soportado

El jugador spawneado no se ve
  -> Envia SpawnPlayer con id -1 al propio jugador
  -> Envia SpawnPlayer con el id real a los demas

El chat no llega al remitente
  -> El broadcast debe incluir al remitente
  -> De lo contrario el no ve sus propios mensajes


----------------------------------------------------------------


⚠️ Beta Notice

protocol-classicube is currently in beta.

The core implementation of the ClassiCube / Classic Minecraft protocol is mostly complete, but the public API and internal structures may still change in future versions.

You may encounter bugs or incomplete features while the project continues to evolve
