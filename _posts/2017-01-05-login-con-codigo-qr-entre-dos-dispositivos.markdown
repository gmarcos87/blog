---
layout: post
title: "Login con código Qr: Parte 1"
date: "2017-01-05 14:08:48 -0300"
comments: true
---
## La idea
Trabajando con Ionic2 y Firebase me encontré con la posibilidad de generar aplicaciones en tiempo real, pero que carecen de sentido si solo pueden ser utilizados en un solo dispositivo. En los momentos de desarrollo, trabajadando con el preview en el navegador y la app en el celular se observa la potencialidad de tener los dos entornos sincronizados en real-time.

Un claro ejemplo son las interfaces web de Whatsapp y Telegram.
En el caso de Telegram la vinculación entre el navegador y la app se genera mediante un código enviado al celular que ingresas en el navegador (útil si tu app es de mensajería o mediante una push-notification), Whatsapp lo hace de una forma similar pero en vez de un código corto enviado por mensaje utiliza un código Qr que muestra el navegador y que capturas desde la aplicación mobile.

## ¿Porque no poner usuario y contraseña?
El sistema de vinculación con códigos es útil en donde la autorización no se encuentra regida por nombre de usuario y contraseña, en mi caso la aplicación registra al usuario (nombre, apellido, número de teléfono) y listo. El resto de las interacciones suceden mediante un sistema de tokens. Lo mismo puede ocurrir cuando el login incluye un sistema de terceros como Facebook, Twitter, etc.

## La lógica
Básicamente necesitamos los siguientes elementos:
* Navegador web con la webapp
* Servidor web
* Firebase para la DB y administración de Usuarios
* App mobile en el celular con el usuario cargado

Procedimientos:
* El navegador dispone de un identificador único otorgado por el servidor
* La app captura ese identificador y envía la autorización al servidor
* El servidor confirma que los datos de autorización son correctos y entrega los datos de autorización al navegador
* El navegador se legea con los datos recibidos

Vamos a construir todas las etapas (utilizando node, mucho javascript y algo de html y css). Las tecnologías involucradas incluye: NodeJS, Express, Socket.io, Webpack, Ionic2 y Firebase. Un fullstack real-time en javascript.

## Lo primero... el servidor
El servidor va a ser el encargado de conectar todas las partes, por eso vamos a iniciar por acá. Su funciones son:
* Mostrar la pantalla de login-qr
* Utilizar sockets para mantener la conexión abierta con la pantalla (para los eventos de conexión, desconexión y login)
* Recibir los datos de la aplicación mobile
* Comprobar su autenticidad y comunicar el resultado al navegador

La parte de la pantalla de login la veremos mas tarde, ya que es un desarrollo en si, es un front y ahora nos tratamos de concentrar en el backend puramente. Empezamos por crear una carpeta nueva y hacer dentro de ella un `npm init`. Esto generará el archivo package.json en donde se registran las dependencias y los comandos personalizados para la ejecución del servidor.

Instalamos las dependencias:

```bash
npm install --save express body-parser socket.io ejs firebase-admin
```

y las de desarrollo:

```bash
npm install mocha chai chai-http webpack
```

Luego de un rato podemos comenzar a crear nuestros archivos y directorios:

```
qr-server
│    app.js
│    serviceAccountKey.json
|    firebase-admin.js
|    package.json (Creado por npm)
|
└───/node_modules (Creado por npm)
|
└─── /public
│   |    index.js
|   |
|   └───/css
|   |    style.css
|   |
|   └─── /js
|         app.js
|
└─── /views
       login.ejs

```
### app.js
Este es el punto de entrada de nuestra aplicación, cargamos las dependencias y el buildplate mínimo de express:
```javascript
const express = require('express')
const app = express();
const path = require('path');
const bodyParser = require('body-parser');
const server = require('http').Server(app);
const io = require('socket.io')(server);
const port = process.env.PORT || 3000;

// View engine
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

//Express config
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));
app.use(express.static(path.join(__dirname, 'public')));

//Express CROS
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

//Rutas
app.get('/', function(req, res) {
  res.render('login'); // Render pantalla de inicio
});

app.post('/auth', function(req, res) {
  res.send('Ok'); /
});

server.listen(port, function(){
  console.log("Listening on port " + port);
});
```
Listo, si ejecutamos `node app.js` deberíamos tener nuestro server corriendo en el puerto 3000  (http://localhost:3000), si entramos no veremos nada porque cargará el archivo login.ejs que está vacío.
El servidor cuenta con dos rutas, / via get y /auth via post. La primera mostrará la pantalla de login con el código Qr y la segunda receptará los datos enviados por la aplicación del celular.

Agregaremos ahora nuestro servidor de Sockets para conexión en tiempo real entre el server y el navegador. Al final del app.js escribimos:
```javascript
//Socket manager
io.on('connection', function (socket) {
  console.log('Usuario conectado!');
  //Envío un hash para el qr
  socket.emit('hash',{hash:'test'});
  //Escucho la desconexión de este socket
  socket.on('disconnect', function() {
      console.log('Usuario desconectado');
   });
});
```
Este es el bosquejo de los eventos que vamos a utilizar, cuando un navegador se conecta le emitimos un hash que lo va a identificar (en este caso es la palabra test). Para generar un identificador único vamos a incluir el modulo de node Crypto:
```javascript
const crypto = require('crypto');
```

Y segun este [post](http://stackoverflow.com/questions/9407892/how-to-generate-random-sha1-hash-to-use-as-id-in-node-js) la mejor forma de obtener un identificador random similar a SHA1 es:
```javascript
let randomId = function(){
  return crypto.randomBytes(20).toString('hex');
}
```

Además tenemos que ir guardando en algún sitio todas las conexiones activas y sus identificadores, para poder identificar posteriormente cuando una celular solicite vincularse con una en particular. Para esto creamos un array vacío y luego usamos el id como key y el socket como value en la conexión y un splice para removerlo en la desconexión.
Entonces quedaría:

```javascript
//Random generator SHA1
const randomId = function(){
  return crypto.randomBytes(20).toString('hex');
}

let socketConections = []; //Donde guardaremos las conexiones

//Socket manager
io.on('connection', function (socket) {
  //Creo el identificador único
  let id = randomId();
  //Agrego el socket al array de sockets
  console.log('Usuario conectado!', id);
  //Envio un hash para el qr
  socket.emit('hash',{hash:id});
  //Escucho la desconexión de este socket
  socket.on('disconnect', function() {
      //Quitamos la conexión del array
      let i = socketConections.indexOf(id);
      socketConections.splice(i, 1);
      console.log('Usuario desconectado',id);
   });
});
```
Casi está lista nuestra parte de sockets en el servidor, solo queda agregar un emit para cuando se vincula el navegador con el celular, eso sucede dentro del metodo /auth que explico a continuación

## Firebase-admin
Dentro de nuestro método */auth* tienen que suceder un par de acciones. Por este método la aplicación mobile envía el token de su usuario registrado Firebase junto con el hash SHA1 escaneado del qr, comprobamos su veracidad y en caso de que esté todo correcto emitimos un token de logeo al navegador correspondiente.

Antes que nada instalemos y configuremos [firebase-admin](https://firebase.google.com/docs/admin/setup)
```bash
npm install firebase-admin --save
```
Importamos el módulo en nuestro firebase-admin.js
```javascript
const admin = require("firebase-admin");
```
Necesitamos bajarnos de Firebase nuestra credencial, un JSON con datos privados que otorgan control total sobre nuestra DB, usuarios, archivos, etc en firebase, así que ojo con publicar este archivo en GitHub! En la [consola de Firebase](https://console.firebase.google.com/) ingresamos a nuestro proyecto o creamos uno y luego clickeamos en *Configuración de proyecto*.
![Firebase console]({{ site.baseurl }}/assets/qr-login/permisos.jpg)

En este panel hay que ir a *Cuentas de servicio* y luego debajo clickear *Generar nueva clave privada*. Esto descargará un archivo JSON al que movemos a nuestro proyecto bajo el nombre de "serviceAccountKey.json".
En firebase-admin.js importamos nuestra clave, inicializamos el admin y lo exportamos.
```javascript
const admin = require("firebase-admin");
const serviceAccount = require("./serviceAccountKey.json");

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
  databaseURL: "https://laUrlDeTuProyecto.firebaseio.com"
});

module.exports = admin;
```
La idea es tener toda la inisialización del admin en un solo archivo y luego importarlo cuando lo necesitemos ya listo para utilizar. En nuestro caso vamos a utilizarlo en app.js, así que debajo de todas las importaciones que ya tenemos agregamos:
```javascript
const admin = require('./firebase-admin')

```
Nota que al tener './' en el nombre buscará en nuestros archivos y no en el modulo de npm. Listo, ya tenemos la capacidad de administrar nuestra cuenta de Firebase desde la aplicación. Ahora vamos a utilizar dos de sus funciones: *createCustomToken* y *verifyIdToken*.

## **/auth** uniendo Firebase y Socket.io
El código del método /auth quedará así:
```javascript
app.post('/auth', function(req, res, next) {
  //Datos enviados por la aplicación mobile
  let clientToken = req.body.token;
  let clientHash = req.body.hash;
  //Verifico que los datos sean correctos
  admin.auth().verifyIdToken(clientToken)
  .then(function(decodedToken) {
    //Al ser verídicos genero un token nuevo que enviaremos al navegador
    admin.auth().createCustomToken(decodedToken.user_id)
    .then(function(customToken) {
      //Busco en socketConections el socket que coincide con el hash
      //y le hago un emit con el token recién creado
      socketConections[clientHash].emit('login',{token:customToken})
      //Respuesta que verá la app mobile
      res.send('ok');
    })
    .catch(function(error) {
      //Si no puedo crear el token
      console.log("Error creating custom token:", error);
      res.send('Error creating custom token')
    });

  }).catch(function(error) {
    //Si los datos recibidos no son validos
    res.send('Invalid token');
  });
});

```
El código está un poco espagueti, es un promise dentro de otro. Lo primero es registrar le token y hash enviado por la app mobile, luego verificar su autenticidad, si está todo correcto solicito un customToken. Cuando el token esté creado (la segunda promise) lo envío al navegador. En este código tenemos una promise que depende del resultado de otra promise, tal cual está escrito (que es la forma intuitiva) es conocido como un antipatrón, la explicación llevaría un post entero, pero acá dejo la forma "correcta" de redactarlo para que veas la diferencia:

```javascript
app.post('/auth', function(req, res, next) {
  let clientToken = req.body.token;
  let clientHash = req.body.hash;
  admin.auth().verifyIdToken(clientToken)
    .then(function(decodedToken) {
      return Promise.all([decodedToken, admin.auth().createCustomToken(decodedToken.user_id)]);
    })
    .then(function(customToken) {
      console.log(customToken)
      socketConections[clientHash].emit('login',{token:customToken[1]})
      res.send({status:'ok'})
    })
    .catch(function(err){
      res.send(err)
    });
});
```
La funcionalidad es la misma pero con un solo catch que permite administrar todos los errores de las promesas puestas en cadena, sea cual sea la que falle.

Cuando el navegador logre hacer login a Firebase ya no necesitaremos mas la conexión y todo quedará funcionando por su cuenta. Es momento de ponernos con la pantalla de login.

## Pantalla de login
La pantalla de login básicamente es la encargada de mostrar el qr con el hash asignado y de administrar los eventos del socket y firebase según corresponda. Hay que tener en claro que el javascript que escribamos de acá en adelante va a ser ejecutado en el cliente (navegador) y no está relacionado con el código que escribimos antes. La manipulación del DOM la voy a hacer con el viejo y querido jQuery, pero podrías usar lo que quieras. Abrimos */views/login-view.ejs* y comenzamos con el HTML:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Qr Login</title>
    <link rel="stylesheet" href="css/main.css">
  </head>
  <body>
    <h1>Login QR:</h1>
    <div><img id="qr"/></div>
    <p class="status" id="qr-value">
      Loading...
    </p>
    <!-- Firebase y jquery desde sus CDN -->
    <script src="https://code.jquery.com/jquery-3.1.1.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery.qrcode/1.0/jquery.qrcode.min.js"></script>
    <script src="https://www.gstatic.com/firebasejs/3.6.4/firebase.js"></script>
    <!-- Socket.io creado por nuestro servidor / cambiar por la url que corresponda -->
    <script src="socket.io/socket.io.js"></script>
    <!-- Nuestra app para el front -->
    <script src="js/front-app.js"></script>
  </body>
</html>
```
Los archivos estáticos que llamemos (como js/front-app.js o css/main.css) estarán dentro de la estructura de directorios de */public*. Antes de continuar añadimos un poco de estilo en main.css
```css
body {
  padding: 0 auto;
  margin: 0 auto;
  text-align: center;
  background-color: #f4f4f4;
  font-family: sans-serif;
}
h1,h2 {
  color: #1c9e7f;
  margin: 20px 0px;
}
```
El generador de código qr es un plugin para jQuery, existen varios todos bastantes buenos y simples de utilizar. Veamos ahora nuestro front-app.js.

## front-app.js
Solo tenemos que realizar tres tareas, una es hacer cambios en el DOM (cambiar textos, mostrar y ocultar código qr) y la segunda es establecer una conexión socket con el servidor y finalmente logearse con firebase desde el navegador.

```javascript
//Iniciamos socket
var socket = io.connect();

//Funciones para la manipulación del DOM
var changeImg = function(data){
  $('#qrcode').empty();
  $('#qrcode').qrcode(data);
}
var changeText = function(data){
  $('#qr-value').text(data);
}
var hiddeQr = function(){
  $('#qrcode').fadeOut();
}

//Iniciamos firebase
firebase.initializeApp({
  apiKey: "asfd123asdf123asdf123asdf123asdf123asdf123",
  authDomain: "tuApp.firebaseapp.com",
  databaseURL: "https://tuApp.firebaseio.com",
  storageBucket: "tuApp.appspot.com",
  messagingSenderId: "123456789123456"
});

//Wrapper de logeo en Firebase
var firebaseLogin = function(data){
  firebase.auth().signInWithCustomToken(data.token)
  .then(function(user){
    hiddeQr();
    changeText('Hola '+user.email);
  })
  .catch(function(error) {
      socket.emit('login-error',error);
  });
};

//Cuando jquery esté listo comenzamos
jQuery(document).ready(function($){
    //Escuchamos el hash via socket
    socket.on('token', function (data) {
      //Renderizamos el hash en qr
      changeImg(data.hash);
      changeText(data.hash);
    });
    //Esuchamos el customToken via socket
    socket.on('login',function(data){
      firebaseLogin(data);
    });
})
```
*firebaseLogin* es un wrapper para *signInWithCustomToken*, la función que ofrece el SDK de Firebase para logearse con token como el que generamos en el backend y que recibimos vía sockets. También estoy emitiendo un mensaje de error que en el servidor no estamos escuchando pero que podríamos hacerlo. Este es todo el código necesario en el front, no es mucho y podría ser mucho menos quitando comentarios, wrappers y minificando.
Ya tenemos el servidor, el frontend de logeo y la conexión con Firebase, solo nos queda la app mobile con Ionic2.
