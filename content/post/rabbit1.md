---
title: Usando rabbitmq en node.js
date: 2018-04-03
---

Hacía tiempo que quería hacer algo con sistemas de colas más allá del ejemplo básico. Recientemente en mashme.io hemos necesitado usarlas para integrar dos servicios distintos y como no había nada hecho, hemos tenido que empezar desde cero.

Conforme iba leyendo documentación y guías sobre colas me daba la sensación de que no había un método de organización o de buenas prácticas definido como podría ser el caso de express.js en el que por un lado se definen las rutas y por otro los controladores y queda muy claro desde el principio cómo se construye cada uno.

En esta serie de posts intentaré describir el estándar que he intentado establecer después de leer y preguntar mucho sobre cómo lo estaban haciendo otros. Personalmente no he encontrado nada que hablara sobre cómo implementar un sistema de colas consistente con rabbitmq en Node.js. El objetivo de escribir esto es que le pueda servir de ayuda a alguien en mi misma situación y por supuesto como autoaprendizaje.

Voy a utilizar como ejemplo el funcionamiento de una cocina domótica. Esta cocina se controlará desde un servidor central al que enviaremos las órdenes usando una API REST. El servidor se comunicará con el resto de elementos de la cocina a través de colas.

### Conceptos
- Exchange: elemento encargado de enrutar los mensajes a las diferentes colas que tenga conectadas
- Cola: pila de mensajes que se consumen bajo demanda

### Exchange tipo fanout
Un exchange de tipo fanout envia una copia del mensaje recibido a todas las colas que tenga enganchadas. 
En nuestro caso vamos a usar este tipo de exchange para comunicarnos con los hornos de la cocina.

Para mantener el código lo más desacoplado posible vamos a crear una nueva clase llamada `WorkerFanout` que usaremos para conectarnos con el servidor de rabbit, crear el exchange y consumir mensajes. Esta clase nos permitirá crear nuevos servicios de tipo fanout sin necesidad de replicar código.

Al constructor de esta clase le pasaremos el host del servidor de rabbitmq, el nombre del exchange y un procesador que se encargará de procesar los mensajes que consuma esta clase.
El unico requisto de un procesador es que debe tener un método llamado `processMessage` que será usado por el worker.
A continuación se muestra un esquema de esta jerarquía de clases:

![](/Users/edumoyano/Downloads/WorkerFanoutScheme.png)

En el constructor de la clase simplemente estableceremos los atributos recibidos:

```js
class WorkerFanout {
  constructor(options) {
    this.exchange = options.exchange || '';
    this.processor = options.processor || null;
    this.host = options.host || 'localhost';
    this.ready = null;
  }
}
```

Para empezar a consumir mensajes crearemos un método `start` que inicializará la conexión con el servidor, creará el exchange y comenzará a consumir.

```js
start() {
    this.ready = amqplib.connect(`amqp://${this.host}`)
    .then((connection) => {
      this.connection = connection;
      return connection.createChannel();
    })
    .then((channel) => {
      this.channel = channel;
      channel.assertExchange(this.exchange, 'fanout');
      return channel.assertQueue('')
      .then((response) => {
        const createdQueue = response.queue;
        channel.bindQueue(createdQueue, this.exchange, '');
        return channel.consume(
          createdQueue,
          message => this.processor.processMessage(message, this._ackMessage.bind(this))
        )
        .catch(err => console.error(`Error receiving from queue ${this.queue}`));
      });
    })
    .catch(err => console.error(`Error connecting to queue ${this.queue}`));
}
```

Lo que hemos hecho en este método es:

1. Conectarnos con el servidor
2. Crear un exchange de tipo fanout con el nombre que se pasó en el constructor
3. Crear una cola y conectarla con el exchange. Si no se especifica nombre al crear la cola, se creara un nombre aleatorio.
4. Se inicia el consumidor

Como se puede ver, el worker no hace nada en concreto con el mensaje recibido, simplemente lo delega en el procesador.

Es importante destacar que ademas del mensaje, tambien pasa al procesador una función llamada `_ackMessage`, que el procesador usará para informar de que ha completado satisfactoriamente (o no) la tarea. Esta función pinta asi:

```js
_ackMessage(result, message) {
  if(result) {
    this.channel.ack(message);
  } else {
    this.channel.nack(message, false, false);
  }
}
```
Como se puede observar, recibe como primer argumento un booleano que informa sobre si ha terminado correctamente o no. El segundo argumento es el mensaje recibido tal cual.

Paralelo al worker, crearemos una clase `PublisherFanout`. Esta clase la usaremos para enviar mensajes a un exchange de tipo fanout.
En el constructor de esta clase abriremos la conexión con el servidor y con el exchange. 

```javascript
class PublisherFanout {
  constructor(options) {
    this.exchange = options.exchange ? options.exchange : '';
    const rabbitHost = options.host || 'localhost';
    console.debug(`Connection publisher to ${rabbitHost}`);
    this.ready = amqplib.connect(`amqp://${rabbitHost}`)
    .then((connection) => {
      this.connection = connection;
      return connection.createChannel();
    })
    .then((channel) => {
      this.channel = channel;
      return channel.assertExchange(this.exchange, 'fanout');
    })
    .catch(err => console.error({ err: err }, `Error connecting to exchange ${this.exchange}`));
  }
}
```

Los mensajes se enviarán usando el método `publishMessage` de esta clase:

```javascript
  publishMessage(payload) {
    const message = typeof payload === 'string' ? new Buffer(payload) : new Buffer(JSON.stringify(payload));
    return this.ready.then(() => this.channel.publish(this.exchange, '', message))
    .catch(err => console.error({ err: err }, `Error sending to queue ${this.queue}`));
  }
```

Ahora crearemos el procesador específico que ejecutarán los hornos. Los hornos tendrán tres acciones basicas: 

* iniciar: calentará durante el tiempo especificado
* parar: dejará de calentar aunque el tiempo no haya acabado
* añadir más tiempo: se ampliará el tiempo de funcionamiento

Al mismo tiempo y gracias a que hemos elegido el exchange de tipo fanout, podremos enviar tareas a varios hornos usando el mismo mensaje. Para conseguir esto, cada horno llevará un identificador único y el procesador decidirá si el mensaje es para él o no en función de este identificador.

```javascript
class OvenProcessor {
  constructor(options) {
    this.id = options.id;
    this.processors = {
      start: this._start,
      stop: this._stop,
      'add-time': this._addTime
    };
  }

  processMessage(message, callback) {
    const payload = JSON.parse(message.content.toString());
    const processor = this.processors[payload.action];
    if(typeof processor === 'function' && payload.ovenIDs.indexOf(this.id) !== -1) {
      console.debug('Processing message');
      processor.call(this, payload, message, callback);
    }
  }

  _start(info, message, callback) {
    console.log(`Oven ${this.id} cooking for ${info.time} seconds`);
    callback(true, message);
  }
}
```

Como se puede ver, en el constructor se asocian las funciones a cada posible acción. El método `processMessage` comprobará que puede atender a esa acción y que esa acción va dirigida para él.

Para probar esto crearemos nuestro consumidor (consumer.js):

```js
const WorkerDirect = require('./WorkerFanout');
const OvenProcessor = require('./OvenProcessor');

const oven = new OvenProcessor({ id: 'oven1' });
const worker = new WorkerDirect({ exchange: 'ovens', processor: oven });
worker.start();

```

Y el emisor (publisher.js):

```js
const PublisherDirect = require('./PublisherFanout');

const publisher = new PublisherDirect({ exchange: 'ovens' });
console.log('Setting oven1 to cook for 10 seconds');
publisher.publishMessage({ action: 'start', ovenIDs: ['oven1'], time: 10 });
```

Si los ejecutamos obtendremos algo asi:

```
node publisher.js
Setting oven1 to cook for 10 seconds
--------------------
node consumer.js
Oven oven1 cooking for 10 seconds
```

Y esto sería todo en cuanto a los exchanges de tipo fanout. El código completo de los 5 archivos de ejemplo se puede encontrar en:

https://gist.github.com/vosgaust/11862edf45ba6820506832e2a0e2c389

En el siguiente blog seguiré con el ejemplo del tipo de exchange "direct".

### Referencias

- https://www.cloudamqp.com/blog/2017-07-25-RabbitMQ-and-AMQP-concepts-glossary.html

- https://www.cloudamqp.com/blog/2015-09-03-part4-rabbitmq-for-beginners-exchanges-routing-keys-bindings.html
