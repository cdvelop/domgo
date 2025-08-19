# Propuesta de API 3: El Enfoque Funcional y Declarativo

Este enfoque se inspira en bibliotecas como `gomponents` y se centra en el uso de funciones de Go para describir la estructura del HTML de una manera declarativa y segura en cuanto a tipos. En lugar de construir componentes como `structs` con estado, se construyen árboles de UI componiendo funciones.

## Principios Clave

- **Declarativo**: El código describe *qué* debe mostrar la UI, no *cómo* actualizarla.
- **Componibilidad**: Interfaces complejas se construyen anidando y combinando funciones simples.
- **Inmutable por Naturaleza**: Las funciones de renderizado son puras; reciben datos y devuelven una descripción de la UI, sin estado interno ni efectos secundarios.
- **Seguridad de Tipos**: Aprovecha el sistema de tipos de Go para evitar errores comunes de HTML (como atributos mal escritos).

---

## Definiciones de la API

### 1. Nodos y Atributos

El núcleo de la API son los tipos que representan nodos HTML y sus atributos.

```go
package gdom

import "io"

// Attr representa un atributo HTML (ej: class="...", href="...").
type Attr func(w io.Writer)

// Node representa un nodo HTML (un elemento, un texto, etc.).
type Node func(w io.Writer)

// Render escribe la representación en cadena de un nodo a un io.Writer.
func Render(w io.Writer, n Node) {
    n(w)
}
```

### 2. Constructores de Elementos

Se proporcionan funciones para cada etiqueta HTML común. Estas funciones aceptan `Nodes` y `Attrs` como hijos.

```go
package gdom

// Div crea un elemento <div>.
func Div(children ...interface{}) Node {
    return element("div", children...)
}

// P crea un elemento <p>.
func P(children ...interface{}) Node {
    return element("p", children...)
}

// Button crea un elemento <button>.
func Button(children ...interface{}) Node {
    return element("button", children...)
}

// Text crea un nodo de texto.
func Text(content string) Node {
    return func(w io.Writer) {
        w.Write([]byte(content)) // Escapado de HTML debería añadirse aquí
    }
}

// element es una función auxiliar interna que construye un nodo de elemento.
func element(name string, children ...interface{}) Node {
    return func(w io.Writer) {
        w.Write([]byte("<" + name))
        // Aplica atributos
        for _, child := range children {
            if attr, ok := child.(Attr); ok {
                w.Write([]byte(" "))
                attr(w)
            }
        }
        w.Write([]byte(">"))
        // Renderiza nodos hijos
        for _, child := range children {
            if node, ok := child.(Node); ok {
                node(w)
            }
        }
        w.Write([]byte("</" + name + ">"))
    }
}
```

### 3. Constructores de Atributos

Funciones para crear atributos de forma segura.

```go
package gdom

// Class añade un atributo 'class'.
func Class(name string) Attr {
    return attribute("class", name)
}

// ID añade un atributo 'id'.
func ID(name string) Attr {
    return attribute("id", name)
}

// OnClick define un manejador de eventos. En el cliente, esto registraría
// una función Go a través de `syscall/js`. En el servidor, podría omitirse.
func OnClick(handlerName string) Attr {
    // La implementación real necesitaría un mecanismo para registrar
    // el handler en el cliente.
    return attribute("onclick", handlerName+"(event)")
}
```

---

## Ejemplo de Uso: Una Tarjeta de Perfil Simple

### `profile.go` (Definición del componente funcional)

```go
package components

import (
    . "github.com/your-repo/gdom"
)

type UserProfile struct {
    Name     string
    Bio      string
    ImageURL string
}

// ProfileCard es una función que toma datos y devuelve un `Node` que describe la UI.
func ProfileCard(profile UserProfile) Node {
    return Div(Class("profile-card"),
        Img(Src(profile.ImageURL), Alt("Foto de perfil")),
        H2(Text(profile.Name)),
        P(Class("bio"), Text(profile.Bio)),
        Button(OnClick("handleFollow"), Text("Seguir")),
    )
}
```

### `main.go` (Renderizado en el cliente)

```go
package main

import (
    "bytes"
    "syscall/js"
    "github.com/your-repo/components"
    "github.com/your-repo/gdom"
)

func main() {
    c := make(chan struct{}, 0)

    // 1. Datos del modelo
    user := components.UserProfile{
        Name:     "Jules, el Ingeniero",
        Bio:      "Especialista en resolver problemas de código con Go y WASM.",
        ImageURL: "https://example.com/jules.png",
    }

    // 2. Describir la UI usando la función componente
    ui := components.ProfileCard(user)

    // 3. Renderizar la descripción de la UI a una cadena HTML
    var buf bytes.Buffer
    gdom.Render(&buf, ui)
    html := buf.String()

    // 4. Inyectar el HTML en el DOM
    container := js.Global().Get("document").Call("getElementById", "app-container")
    container.Set("innerHTML", html)

    // 5. Registrar los manejadores de eventos (esto requeriría un mecanismo más robusto)
    js.Global().Set("handleFollow", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        js.Global().Call("alert", "Ahora estás siguiendo a Jules!")
        return nil
    }))

    <-c
}
```

## Renderizado en el Servidor (SSR)

Este enfoque es **excelente para SSR**. Dado que las funciones de los componentes simplemente escriben en un `io.Writer`, se pueden usar directamente en un manejador HTTP para generar HTML sobre la marcha.

```go
// En un servidor Go:
func handleProfilePage(w http.ResponseWriter, r *http.Request) {
    user := components.UserProfile{ /* ... cargar datos ... */ }

    // Escribe el HTML directamente en la respuesta HTTP
    gdom.Render(w, components.ProfileCard(user))
}
```

## Ventajas y Desventajas

- **Ventajas**:
    - Código muy expresivo y fácil de leer para describir estructuras HTML.
    - Promueve la creación de funciones puras, lo que facilita las pruebas y el razonamiento.
    - El renderizado del lado del servidor (SSR) es trivialmente simple y eficiente.
    - La composición de funciones es una forma poderosa y natural de construir UI en Go.
- **Desventajas**:
    - La gestión de eventos en el cliente (la "hidratación") es más compleja. Se necesita un sistema para conectar los atributos `onclick` del HTML renderizado con las funciones de Go en el WASM.
    - Para actualizaciones de UI muy dinámicas y de grano fino, re-renderizar todo el HTML y hacer un `Set("innerHTML", ...)` puede no ser lo más eficiente. Se necesitaría un mecanismo de "diffing" (como `morphdom`) para aplicar parches inteligentes, lo que añade complejidad.
```
