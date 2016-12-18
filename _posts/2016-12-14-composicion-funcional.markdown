---
layout: post
title:  "Composición funcional en JavaScript"
date:   2016-12-14 09:54:26 -0300
categories: javascript patrones
---
Vamos a tomar un ejemplo bastante simple, para comenzar a refactorearlo con el patrón de composición funcional. Aviso desde ya todo el código estará en ES6.

En el ejemplo tenemos algunas funciones que reciben un argumento y generan un resultado, todo muy simple.

{% highlight javascript %}
let doble = numero => numero * 2;
let triple = numero => numero * 3;
let quadruple = numero => numero * 4;
let numero = 5;

numero = doble(numero);
numero = triple(numero);
numero = quadruple(numero);

console.log(numero) // => 120
{% endhighlight %}


En este código hay algo que no cierra del todo, por ejemplo tenemos a la variable numero reasignada una y otra vez, ademas que no es claro al momento de leerlo. Podríamos refactorearlo de esta forma:

{% highlight javascript %}
let doble = numero => numero * 2;
let triple = numero => numero * 3;
let quadruple = numero => numero * 4;
let numero = 5;

numero = quadruple(triple(doble(5)));

console.log(numero) // => 120
{% endhighlight %}

Ahora estaríamos metiendo funciones dentro de funciones, continua siendo problemático de leer y si quisiéramos agregar otras funciones terminaríamos con una cantidad de paréntesis infernal.

Podriamos refactorear los métodos de arriba para que el retorno sea encadenable, es decir que `return` sea el mismo objeto y se le pueda llamar otro método inmediatamente. Un patrón de diseño conocido como [Builder][builder]

{% highlight javascript %}
let MultiplicadorBuilder = function(numeroInicial) {
  let num = numeroInicial;

  return {
      doble: () => {
        num = num * 2;
        return this; //--> se retorna a si mismo
      },
      triple: () => {
        num = num * 3;
        return this; //--> se retorna a si mismo
      },
      quadruple: () => {
        num = num * 4;
        return this; //--> se retorna a si mismo
      },
      total: () => num
  };
};

let multiB = new MultiplicadorBuilder(5);
let numero = multiB.doble().triple().quadruple().total();

console.log(numero) // => 120
{% endhighlight %}

En este ejemplo usamos el patrón Builder para aplicar un esquema de [azucar sintactica][azucar] como forma de reasignar el valor a la variable num. Todavía no se ve del todo correcto, ya que ahora tenemos que generar una instanciación del builder para asignar el valor inicial.

Nuestras funciones están ahora todas juntas dentro del Builder, podríamos abstraerlas pero sería sumar aun mas complejidad. Además tenemos ahora una función con un poco de estado también (num), ya que iniciamos el constructor con el valor de partida, quitando legibilidad a nuestro código y haciendo que nuestra solución inicial parezca mucho más simple (aunque mucho menos escalable).

El siguiente ejemplo es composición funcional en JavaScript, muchas librerías de JS incluyen la misma funcionalidad, por ejemplo [lodash][lodash] tiene un método llamado `_.flow`. El procedimiento técnico es conocido como [pipe][pipe] (tubería), que reduce las funciones de izquierda a derecha (en donde el resultado de una es el valor inicial de la próxima). Se puede usar también reduceRight para que nuestro operador aplique las funciones de derecha a izquierda de una forma mas funcional.

Cada función simplemente pasará el return de la anterior y dará el resultado a la siguiente, así hasta que se obtenga el valor final.

{% highlight javascript %}
let doble = numero => numero * 2;
let triple = numero => numero * 3;
let quadruple = numero => numero * 4;
let componer = (...funciones) => (valor) => funciones.reduce((v,fn) => fn(v), valor);

let multiplicador = componer(
    doble,
    triple,
    quadruple
);

let numero = multiplicador(5);

console.log(numero);

{% endhighlight %}

Vamos de a parte, en el inicio declaramos nuestras funciones como lo hicimos en el primer ejemplo, sin embargo también declaramos una llamada `componer`. Analicemos que es lo que hace:

{% highlight javascript %}
// Declaramos la función, que recibe
// como argumento otras funciones
let componer = function(...funciones){

    // Devuelve una función
    // que recibe un valor
    return function(valor){

        // Reduce el argumento que pasamos
        // al inicio, el array de funciones
        return funciones.reduce(function(val, funIndividual){

            // Agarra la función individual,
            // aplica los cambios y pasa el valor
            return funIndividual(val)

        // Pasa el valor al reductor para que
        // sea introducido en la siguiente función
        }, value);
    }
}
{% endhighlight %}

Para finalizar, llamamos a la función componer, pasamos todas las funciones que queremos que formen parte y obtenemos como resultado otra función a la que solo hay que pasarle el valor que tomará la primera de la funciones a ser ejecutada.

La composición funcional no afecta la testeabilidad ya que es puramente genérica y no afecta ningún detalle especifico de la implementación. Otorga un código mas claro para leer sin sobrecargar de complejidad.

[azucar]:    https://es.wikipedia.org/wiki/Az%C3%BAcar_sint%C3%A1ctico
[builder]:   https://es.wikipedia.org/wiki/Builder_(patr%C3%B3n_de_dise%C3%B1o)
[pipe]:      https://es.wikipedia.org/wiki/Tuber%C3%ADa_(inform%C3%A1tica)
[lodash]:    https://lodash.com/
