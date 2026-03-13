Tutorial Classicube-Protocol v4 – Servidor y Cliente Node.js

1️⃣ Requisitos

Node.js 14+

npm

classicube-protocol (npm install classicube-protocol)


2️⃣ Instalación

npm install classicube-protocol

3️⃣ Estructura de la librería

classicube-protocol/
├── index.js
├── PacketIDs.js
├── Binary.js
├── server/
│   ├── Server.js
│   └── ServerConnection.js
├── client/
│   └── ClientConnection.js
├── auth/
│   └── ClassiCubeAuth.js
├── packets/
│   └── 16 paquetes (PlayerIdentification.js, ServerIdentification.js…)
└── utils/
    ├── Level.js
    └── Debug.js

4️⃣ Flujo de conexión cliente-servidor

Cliente → Servidor: PlayerIdentification → ChatMessage → Movimiento

Servidor → Cliente: ServerIdentification → LevelInitialize → LevelDataChunk → LevelFinalize → SpawnPlayer → Updates


5️⃣ Servidor básico

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

6️⃣ Cliente básico

const { createClient, Level } = require('classicube-protocol');
const client = await createClient({ host: 'localhost', port: 25565, username: 'Steve' });
client.on('chatMessage', ({ message }) => console.log(message));
client.on('levelFinalize', ({ xSize, ySize, zSize, blocks }) => console.log('Mapa:', xSize + 'x' + ySize + 'x' + zSize));
client.write('chat', { message: 'Hola desde el bot!' });

7️⃣ Mundo plano con bloques

const { Level } = require('classicube-protocol');
const blocks = Buffer.alloc(64*64*64, Level.BlockType.AIR);
for (let z=0; z<64; z++)
  for (let x=0; x<64; x++)
    Level.setBlock(blocks, x, 0, z, 64, 64, Level.BlockType.STONE);
for (let y=5; y<=10; y++)
  for (let z=28; z<=36; z++)
    for (let x=28; x<=36; x++)
      Level.setBlock(blocks, x, y, z, 64, 64, Level.BlockType.GLASS);
conn.sendLevel(Level.encode(blocks));
conn.sendLevelFinalize({ xSize:64, ySize:64, zSize:64 });

8️⃣ Comandos de servidor

/tp x y z

/op

/deop

/kick jugador [razon]

/players


9️⃣ Autenticación

const server = createServer({ online: true, serverName: 'MiServidor' });
const { auth } = require('classicube-protocol');
const result = await auth.verify({ serverName: 'MiServidor', username: 'Steve', verificationKey: keyDelCliente });
if(result.valid) console.log('Bienvenido,', result.username);
else conn.kick('Auth failed');

🔟 Debug mode

DEBUG=cc-protocol node server.js          # logs generales
DEBUG=cc-protocol:packets node server.js # paquetes TCP
DEBUG=cc-protocol:auth node server.js    # auth