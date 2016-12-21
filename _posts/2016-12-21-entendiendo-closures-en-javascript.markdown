---
layout: post
title: "Entendiendo Closures en Javascript"
date: "2016-12-21 12:10:22 -0300"
categories: javascript
comments: true
---
En nuestro proceso de convertirnos en desarrollador avanzado de JavaScript seguro que nos cruzamos con "closures". Antes de leer alguna explicación técnica sobre el asunto posiblemente quieras entenderlo de una forma mas metafórica.
Una cosa asombrosa sobre closures: te permiten escribir funciones con un paso intermedio, el de capturar la información en un momento específico de tiempo, como si fuera una pausa. Podes ejecutar esa función y guardar los valores en una variable para utilizarlos luego, a pesar de que la variable original ya tenga otros valores en ese momento.

![Challenge accepted]({{ site.baseurl }}/assets/img/cha.jpg)

### ¿Cuándo usar Closures?
Supongamos que estas programando un mapa interactivo con los lugares turísticos de Córdoba usando GoogleMaps API. Tenes un JSON con un listado de los markers que querés agregar al mapa (peatonal, terminal, cañada, etc). Lograste agregarlos pero también querés agregar un evento 'click' a cada marker para mostrar información extra sobre ese lugar en particular.

```javascript
let lugaresTuristicos = {…};
for(let i=0; i< lugaresTuristicos.length; i++){
  let marker= lugaresTuristicos[i];
  $(marker).click(function(){
    mostrarInfo(i);
  });
}
```
Acá hay un problema, si lo escribís de esta forma no va a funcionar. El loop 'for' va a terminar antes del callback, y el evento click no va a poder registrar apropiadamente el valor de 'i'. Necesitas capturar el valor de 'i' en el momento inmediato para poder utilizarlo después.

## ¿Qué es lo que necesito conocer?
* Scope de las variables
* El concepto de callback

Podemos entender Closures comparándolo con el proceso de enviar un paquete por correo (el viejo correo, no email).

Miremos este código básico que usa closure para enviar un paquete:

```javascript
function empaquetar(item){
    //Codigo que pone al item en una caja
    console.log('Pongo ' +item+ ' en una caja');

    function agregarDestinatario(direccion){
        // Código que agrega la direccion a la caja
        console.log('Enviar caja de ' + item + ' a ' + direccion);
    }

    return agregarDestinatario;
}

var miEnvio = empaquetar('tornillos');
//Pongo tornillos en una caja

miEnvio('123 San Martín, Córdoba')
// Enviar caja de tornillos a 123 San Martín, Córdoba
```

La función *agregarDestinatario* es un closure!. Puede ser llamada en cualquier momento después de que la función *empaquetar* fuese llamada. Y siempre va a poder acceder a las variables y argumentos del momento en que empaquetar fue originalmente llamado.

En este caso el proceso de enviar el paquete no se termina hasta que no tiene la dirección escrita. Por eso la función *empaquetar* espera hasta que el closure sea llamado. Vamos a ver lo que pasa línea por línea.

* **Línea 13**: Creas la variable miEnvio, que es una instancia de empaquetar. Estoy empaquetando tornillos.
* **Línea 3**: El código logea sobre el proceso de meter tornillos en una caja.
* **Línea 10**: empaquetar retorna… otra función? Huh?

![Wait for it]({{ site.baseurl }}/assets/img/wfi.png)

Hagamos una pausa, asumamos que la línea 16 todavía no se ejecutó. Esto es lo que está pasando; *empaquetar()* no va a hacer un return con un "listo para enviar" hasta que no se llame tambien agregarDestinatario() con el argumento de la dirección. Como los pasos comunes al momento de enviar un paquete, primero metes las cosas dentro de una caja y después le agregar el destinatario. No es un paquete enviable hasta no tener dirección! Podrías pensar que no es necesario agregar la dirección inmediatamente después de llenar la caja, pueden pasar otras cosas entremedio. Podes tener que ir a buscar la dirección a otro lado, o llenar todas las cajas y luego agregarle las direcciones una por una por ejemplo.

Si no pones la dirección inmediatamente después de empaquetar no significa que el contenido de la caja desaparezca mágicamente. El contenido va a estar ahí hasta que le pongas una dirección al paquete, por eso cada vez que llamamos *miEnvio*, el argumento original, tornillos, va a estar todavía disponible.

Ahora pongamos play y continuemos con la línea 16...

* **Línea 16**: Estamos listos para agregar la dirección, entonces llamemos a miEnvio con la dirección como argumento. Recordemos la línea 10, miEnvio es una instancia de empaquetar() con 'tornillos' como argumento. Entonces cuando la llamemos estaremos ofreciendo otro argumento el cual será enviado con el closure: agregarDestinatario().

* **Línea 5**: Enviamos un argumento (la dirección) a agregarDestinatario()
* **Línea 7**: agregarDestinatario logea el resultado, utilizando el primer argumento (item) y la dirección recientemente enviada.

Si quisiéramos podríamos *consumir* empaquetar en una sola línea: empaquetar('tornillos')('123 San Martín, Córdoba')

## Un ejemplo más
Digamos que queremos enviar paquetes a cada miembro de nuestro equipo, prepararíamos cada paquete antes de agregarles las direcciones
En código esto sería:

```javascript
let paqueteAna = empaquetar('pinturas');
//Pongo pinturas en una caja
let paqueteLibertad = empaquetar('juguete');
//Pongo juguete en una caja
let paqueteMarcos = empaquetar('teclado');
//Pongo teclado en una caja
let paqueteTita = empaquetar('ratón');
//Pongo ratón en una caja

paqueteAna('123 San Martín, Córdoba')
// Enviar caja de pinturas a 123 San Martín, Córdoba
paqueteLibertad('123 Belgrano, Córdoba')
// Enviar caja de juguete a 123 Belgrano, Córdoba
paqueteMarcos('Calle Pública s/n, José de la Quintana')
// Enviar caja de teclado a Calle Pública s/n, José de la Quintana
paqueteTita('Av. Corrientes 2020, CABA')
// Enviar caja de ratón a Av. Corrientes 2020, CABA
```

¡Otra característica mágica de los closures! Cada instancia es capaz de usar el item correcto con la dirección correcta, incluso cuando ejecutamos 4 pares separados de item/dirección. En una función tradicional no hay ningún concepto de memoria. Si quisieras hacerlo de un modo tradicional necesitarías explícitamente volver a incluir el item en agregarDestinatario

## ¿Dónde se usa esto?
Te vas a encontrar frecuentemente con closures en NodeJs. Si tan sólo estas interesado en front-end recuerda el problema del comienzo. Si querés escribir una función que tenga en cuenta el input del usuario en dos estados diferentes de tu aplicación considera usar closure.
