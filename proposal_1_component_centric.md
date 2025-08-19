# Propuesta de API 1: El Enfoque Centrado en Componentes (Component-Centric)

Este enfoque se inspira en el ejemplo `go-counter` proporcionado y formaliza la idea de que cada elemento de la UI es un objeto autónomo que gestiona su propio estado y renderizado. Es ideal para componentes encapsulados y reutilizables.

## Principios Clave

- **Encapsulación**: Cada componente agrupa su estado, su lógica y su representación visual.
- **Ciclo de Vida Simple**: Los componentes tienen métodos claros para su creación (`Mount`), actualización (`Update`) y destrucción (`Unmount`).
- **Control Directo del DOM**: El componente manipula directamente el DOM a través de una interfaz inyectada, sin un Virtual DOM.
- **Estado Local**: El estado se gestiona dentro de cada instancia del componente, permitiendo múltiples instancias independientes.

---

## Definiciones de la API

### 1. La Interfaz `DOMAdapter`

Esta es la abstracción clave que permite el desacoplamiento del DOM, facilitando las pruebas y el renderizado en el lado del servidor (SSR).

```go
package dom

// DOMAdapter define las operaciones mínimas para manipular el DOM.
// La implementación para el navegador usará `syscall/js`.
// La implementación para SSR generará una cadena HTML.
type DOMAdapter interface {
    // GetElement devuelve un manejador para un elemento del DOM (ej: js.Value en cliente).
    GetElement() interface{}

    // SetInnerHTML reemplaza el contenido del elemento.
    SetInnerHTML(html string)

    // QuerySelector busca un elemento dentro del componente.
    QuerySelector(selector string) interface{}

    // AddEventListener asocia un evento a un elemento.
    AddEventListener(element interface{}, eventType string, handler func())

    // GetCSS devuelve los estilos del componente como bytes.
    GetCSS() []byte
}
```

### 2. La Interfaz `Component`

Todo componente web debe implementar esta interfaz.

```go
package component

import "github.com/your-repo/dom"

// Component define el ciclo de vida de un componente web.
type Component interface {
    // Mount se llama cuando el componente se añade al DOM.
    // Recibe el adaptador del DOM para el elemento anfitrión.
    Mount(adapter dom.DOMAdapter)

    // Update se llama para solicitar una nueva renderización del componente.
    Update()

    // Unmount se llama cuando el componente se elimina del DOM.
    Unmount()
}
```

### 3. El Gestor de Componentes (`ComponentManager`)

Para manejar múltiples instancias de componentes de forma segura, un gestor se encarga de asociar elementos del DOM con sus instancias de componentes en Go.

```go
package main

import (
    "sync"
    "syscall/js"
    "github.com/your-repo/component"
)

// manager mantiene un registro de las instancias de componentes activas.
// Es compatible con TinyGo al usar un slice en lugar de un mapa.
type instance struct {
    id        string
    component component.Component
}

var (
    instances []instance
    mutex     sync.Mutex
    nextID    int
)

// RegisterComponent se exporta a JavaScript.
// Se llama desde el `connectedCallback` del Custom Element.
//go:export RegisterComponent
func RegisterComponent(hostElement js.Value, componentName string) {
    mutex.Lock()
    defer mutex.Unlock()

    // Genera un ID único para esta instancia.
    id := fmt.Sprintf("go-component-%d", nextID)
    nextID++
    hostElement.Set("id", id)

    var comp component.Component
    switch componentName {
    case "go-counter":
        comp = NewCounter()
    // case "go-profile-card":
    //     comp = NewProfileCard()
    default:
        fmt.Println("Componente desconocido:", componentName)
        return
    }

    // Crea el adaptador del DOM para esta instancia
    adapter := NewBrowserAdapter(hostElement)

    // Monta el componente y lo guarda
    comp.Mount(adapter)
    instances = append(instances, instance{id: id, component: comp})
}

// ... funciones para UnregisterComponent, etc.
```

---

## Ejemplo de Uso: `<go-counter>`

Así se implementaría el contador con esta API.

### `counter.go`

```go
package components

import (
    "fmt"
    "github.com/your-repo/dom"
)

type Counter struct {
    count   int
    adapter dom.DOMAdapter
}

func NewCounter() *Counter {
    return &Counter{count: 0}
}

func (c *Counter) Mount(adapter dom.DOMAdapter) {
    c.adapter = adapter
    c.Update()
}

func (c *Counter) Update() {
    html := fmt.Sprintf(
        `<p>Contador: <strong>%d</strong></p>
         <button class="increment-btn">Incrementar</button>`,
        c.count,
    )
    c.adapter.SetInnerHTML(html)

    // Añadir listener al botón recién creado
    btn := c.adapter.QuerySelector(".increment-btn")
    if btn != nil {
        c.adapter.AddEventListener(btn, "click", c.increment)
    }
}

func (c *Counter) Unmount() {
    // Limpiar recursos si es necesario
    c.adapter = nil
}

// increment es el manejador del evento de clic.
func (c *Counter) increment() {
    c.count++
    c.Update()
}

// GetCSS devuelve los estilos del componente.
func (c *Counter) GetCSS() []byte {
    return []byte(`
        .go-counter { border: 1px solid blue; padding: 10px; }
        .go-counter button { background-color: blue; color: white; }
    `)
}
```

### `glue.js` (JavaScript de Conexión)

```javascript
// Carga y ejecuta el módulo WASM
const go = new Go();
WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject).then(result => {
    go.run(result.instance);
});

// Define el Custom Element
class GoCounter extends HTMLElement {
    connectedCallback() {
        // Llama a la función Go exportada para registrar esta instancia
        window.RegisterComponent(this, 'go-counter');
    }

    disconnectedCallback() {
        // Aquí se llamaría a UnregisterComponent(this.id)
    }
}

customElements.define('go-counter', GoCounter);
```

## Ventajas y Desventajas

- **Ventajas**:
    - Muy intuitivo para desarrolladores con experiencia en OOP.
    - Fomenta la creación de componentes verdaderamente autocontenidos.
    - El estado local es simple de razonar.
- **Desventajas**:
    - La comunicación entre componentes puede ser compleja (requiere un bus de eventos o callbacks).
    - Para aplicaciones con estado global complejo, puede llevar a la duplicación de datos o a un "prop drilling" excesivo.
    - La gestión manual del ciclo de vida (`Mount`, `Unmount`) puede ser propensa a errores si no se maneja cuidadosamente.
```
