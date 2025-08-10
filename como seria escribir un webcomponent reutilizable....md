Claro que sí. Crear un WebComponent reutilizable en Go compilado a WebAssembly (WASM) es una excelente manera de encapsular lógica compleja y de alto rendimiento directamente en el frontend, minimizando la dependencia de JavaScript.

La clave es usar Go para toda la lógica interna y el estado del componente, y una capa muy delgada de JavaScript para registrar el Custom Element y conectar sus eventos del ciclo de vida (como cuando se añade al DOM) con funciones exportadas desde Go.

El enfoque que describes, usando syscall/js de forma mínima solo para actualizar el DOM directamente, es el más performante y puro para esta tarea, evitando por completo la sobrecarga de un Virtual DOM.

---

### **⚙️ Arquitectura General**

1. **Código Go (.go):** Contiene la lógica principal, el estado y una función render. Esta función recibe una referencia al elemento del DOM y lo modifica directamente. Las funciones que necesiten ser llamadas desde JS se exportan.  
2. **Compilador Go:** Genera el archivo WebAssembly (.wasm) a partir del código Go.  
3. **Archivo "puente" (wasm\_exec.js):** Es un archivo proporcionado por Go que permite cargar y ejecutar el módulo WASM en el navegador.  
4. **Código JavaScript "pegamento":** Una pequeña clase que define el CustomElement. Su única responsabilidad es:  
   * Cargar el WASM.  
   * En el connectedCallback (cuando el elemento se inserta en el DOM), llama a la función render de Go.  
   * Conecta eventos del usuario (como clics) a otras funciones de Go.  
5. **HTML:** Simplemente usa la etiqueta personalizada, como \<mi-componente-go\>\</mi-componente-go\>.

---

### **🚀 Ejemplo Práctico: Un Contador en Go**

Vamos a crear un componente \<go-counter\> que muestra un número y un botón para incrementarlo. Toda la lógica del contador y la actualización del DOM se harán en Go.

#### **Paso 1: Escribir el Código Go (component.go)**

Este código define la lógica del contador y la función render que manipula el DOM.

Go

package main

import (  
	"fmt"  
	"syscall/js"  
)

var (  
	// Estado interno del componente  
	count \= 0  
	// Mantenemos una referencia al elemento anfitrión para no tener que buscarlo cada vez  
	hostElement js.Value  
)

// render se encarga de reescribir el innerHTML del componente.  
// No usa Virtual DOM, manipula el DOM directamente.  
func render() {  
	if \!hostElement.Truthy() {  
		return // Aún no se ha adjuntado el elemento  
	}

	html := fmt.Sprintf(\`  
		\<p\>Contador Go (WASM): \<strong\>%d\</strong\>\</p\>  
		\<button id="go-counter-btn"\>Incrementar \+\</button\>  
	\`, count)

	// Reescribe el contenido del componente  
	hostElement.Set("innerHTML", html)

	// Añadimos el listener al nuevo botón creado.  
	// Usamos \`QuerySelector\` desde el \`hostElement\` para asegurar que estamos en nuestro componente.  
	btn := hostElement.Call("querySelector", "\#go-counter-btn")  
	if btn.Truthy() {  
		btn.Call("addEventListener", "click", js.FuncOf(increment))  
	}  
}

// increment es la función que se ejecuta al hacer clic.  
// Es un \`js.Func\` que no recibe argumentos y no devuelve nada.  
func increment(this js.Value, args \[\]js.Value) interface{} {  
	count++  
	render() // Vuelve a renderizar el componente con el nuevo estado  
	return nil  
}

// setup es la función inicial que se llama desde JS cuando el componente se conecta al DOM.  
func setup(this js.Value, args \[\]js.Value) interface{} {  
	// args\[0\] es el elemento \<go-counter\> que nos pasa el JS  
	hostElement \= args\[0\]  
	render() // Renderizado inicial  
	return nil  
}

func main() {  
	fmt.Println("Módulo Go-WASM cargado.")  
	c := make(chan bool)

	// Exportamos la función \`setup\` para que JavaScript pueda llamarla.  
	// La ponemos en el objeto \`window\` para que sea globalmente accesible.  
	js.Global().Set("setupGoComponent", js.FuncOf(setup))

	\<-c // Mantenemos el programa Go en ejecución  
}

#### **Paso 2: Crear el "Pegamento" JavaScript y el HTML**

Necesitamos un index.html y un main.js para cargar nuestro componente.

Primero, copia wasm\_exec.js desde tu instalación de Go a tu directorio de proyecto:

Bash

cp "$(go env GOROOT)/misc/wasm/wasm\_exec.js" .

**index.html**

HTML

\<\!DOCTYPE **html**\>  
\<html lang\="es"\>  
\<head\>  
    \<meta charset\="UTF-8"\>  
    \<title\>Go WebAssembly Component\</title\>  
    \<script src\="wasm\_exec.js" defer\>\</script\>  
    \<script src\="main.js" type\="module" defer\>\</script\>  
    \<style\>  
        go-counter {  
            display: block;  
            border: 1px solid \#007d9c;  
            padding: 1rem;  
            border-radius: 8px;  
            font-family: sans-serif;  
        }  
    \</style\>  
\</head\>  
\<body\>  
    \<h1\>Ejemplo de WebComponent con Go y WASM\</h1\>  
    \<p\>El siguiente componente está 100% controlado por Go:\</p\>  
      
    \<go-counter\>\</go-counter\>

    \<hr\>  
    \<p\>Puedes reutilizarlo fácilmente:\</p\>  
    \<go-counter\>\</go-counter\>

\</body\>  
\</html\>

**main.js**

JavaScript

// Carga y ejecuta nuestro módulo WASM  
const go \= new Go();  
WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject).then((result) \=\> {  
    go.run(result.instance);  
});

// Definimos la clase para nuestro Custom Element  
class GoCounter extends HTMLElement {  
    constructor() {  
        super();  
        // Podrías inicializar el Shadow DOM aquí si quisieras un encapsulamiento más fuerte  
        // this.attachShadow({ mode: 'open' });   
    }  
      
    // Este es el callback clave del ciclo de vida.  
    // Se ejecuta cuando el elemento es insertado en el DOM.  
    connectedCallback() {  
        console.log("Componente \<go-counter\> conectado al DOM.");  
          
        // Esperamos a que la función de Go esté disponible en \`window\`  
        // y la llamamos, pasándole \`this\` (el propio elemento) como argumento.  
        if (window.setupGoComponent) {  
            window.setupGoComponent(this);  
        } else {  
            // Reintento por si el WASM no ha cargado todavía  
            setTimeout(() \=\> window.setupGoComponent(this), 50);  
        }  
    }  
      
    disconnectedCallback() {  
        // Aquí podrías limpiar recursos si fuera necesario  
        console.log("Componente \<go-counter\> desconectado.");  
    }  
}

// Registramos nuestro nuevo elemento en el navegador.  
customElements.define('go-counter', GoCounter);

#### **Paso 3: Compilar y Servir**

1. **Compila el código Go a WASM:**  
   Bash  
   GOOS=js GOARCH=wasm \-o main.wasm component.go

   Esto creará el archivo main.wasm en tu directorio.  
2. **Inicia un servidor web local:** No puedes abrir index.html directamente desde el sistema de archivos (file://) porque la carga de WASM requiere el protocolo HTTP. La forma más sencilla es usar el módulo http de Python:  
   Bash  
   python3 \-m http.server

   O si tienes Node.js:  
   Bash  
   npx serve

Ahora, abre tu navegador y ve a http://localhost:8000. Verás tus dos contadores funcionando de forma independiente, ¡controlados completamente por Go\!

---

### **💡 Puntos Clave del Enfoque**

* **Mínima Interfaz syscall/js:** Solo usamos js.Global().Set para exportar una función y elemento.Set("innerHTML", ...) para la actualización. Es directo y eficiente.  
* **Sin Virtual DOM:** La función render de Go analiza el estado actual (count) y genera el HTML necesario, sobreescribiendo el contenido del componente. Para componentes complejos, podrías hacer actualizaciones más granulares (ej. elemento.Call("querySelector", ...).Set("textContent", ...)), pero para este caso, innerHTML es simple y efectivo.  
* **Estado en Go:** El estado (count) vive enteramente dentro del entorno de Go. JavaScript no sabe ni necesita saber sobre él. Esto hace que el componente sea muy robusto.  
* **Reutilizable:** Cada \<go-counter\> en el HTML es una instancia separada. El ejemplo actual usa una variable global en Go, lo cual no es ideal para múltiples instancias. Para un componente verdaderamente reutilizable, deberías manejar un mapa de instancias en Go, usando un ID único generado en el connectedCallback para gestionar el estado de cada componente por separado.