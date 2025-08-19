# Propuesta de API 2: El Enfoque Dirigido por Datos (Data-Driven Store)

Este enfoque, basado en las ideas de `store.md`, separa completamente el estado de la aplicación de su representación en el DOM. La UI es una "función" del estado: cuando el estado cambia, la UI se actualiza para reflejarlo. Es ideal para aplicaciones con un estado complejo o compartido entre muchos componentes.

## Principios Clave

- **Fuente Única de Verdad (Single Source of Truth)**: El estado de toda la aplicación se almacena en una estructura de datos centralizada (el `Store`).
- **UI Reactiva**: La vista es una representación del estado. Las actualizaciones del DOM son un efecto secundario de los cambios en el `Store`.
- **Inmutabilidad (Recomendada)**: Los cambios de estado se realizan creando nuevos estados en lugar de mutar los existentes, lo que simplifica el seguimiento de cambios.
- **TinyGo-Friendly**: La implementación del `Store` utiliza `slices` para ser compatible con TinyGo.

---

## Definiciones de la API

### 1. El `Store` Genérico

Esta es la base del sistema, una versión pulida de la propuesta en `store.md`.

```go
package store

// Item define cualquier objeto que pueda ser guardado en el store.
type Item interface {
    GetID() string
}

// Store define una interfaz genérica para colecciones de datos.
type Store[T Item] interface {
    Create(item T) error
    Update(item T) error // Actualiza o inserta (Upsert)
    Get(id string) (T, error)
    Delete(id string) error
    GetAll() []T
}

// SliceStore es una implementación de Store compatible con TinyGo.
// (El código es el mismo que el propuesto en store.md, con findIndex, etc.)
type SliceStore[T Item] struct {
    items []T
    // ...
}

func NewSliceStore[T Item]() Store[T] {
    return &SliceStore[T]{items: make([]T, 0)}
}
```

### 2. El `Renderer`

El `Renderer` es el motor que traduce el estado del `Store` a operaciones del DOM. Escucha los cambios en el estado y actualiza la vista.

```go
package renderer

import "github.com/your-repo/store"

// Template es una función que toma un item y devuelve su representación HTML.
type Template[T store.Item] func(item T) string

// Renderer se encarga de dibujar y actualizar los datos del store en el DOM.
type Renderer[T store.Item] struct {
    store      store.Store[T]
    container  dom.Adapter // Adaptador para el elemento contenedor en el DOM
    template   Template[T]
    // ...
}

func NewRenderer[T store.Item](s store.Store[T], container dom.Adapter, t Template[T]) *Renderer[T] {
    return &Renderer[T]{
        store:      s,
        container:  container,
        template:   t,
    }
}

// RenderAll borra el contenedor y renderiza todos los items del store.
func (r *Renderer[T]) RenderAll() {
    items := r.store.GetAll()
    var html strings.Builder
    for _, item := range items {
        html.WriteString(r.template(item))
    }
    r.container.SetInnerHTML(html.String())
    r.attachEventListeners() // Vuelve a asociar eventos si es necesario
}

// Métodos para actualizaciones más granulares (ej: UpdateItem, RemoveItem)
// podrían optimizarse para no redibujar todo.
```

### 3. El `App` (El Orquestador)

El `App` une el `Store`, el `Renderer` y los manejadores de eventos.

```go
package main

type ToDo struct {
    ID        string
    Text      string
    Completed bool
}
func (t ToDo) GetID() string { return t.ID }

// App encapsula la lógica de la aplicación de lista de tareas.
type App struct {
    store    store.Store[ToDo]
    renderer *renderer.Renderer[ToDo]
}

func NewApp(container dom.Adapter) *App {
    s := store.NewSliceStore[ToDo]()

    // La plantilla define cómo se ve cada ToDo.
    todoTemplate := func(item ToDo) string {
        checked := ""
        if item.Completed {
            checked = "checked"
        }
        return fmt.Sprintf(
            `<div class="todo-item" data-id="%s">
                <input type="checkbox" %s>
                <span>%s</span>
                <button class="delete-btn">X</button>
            </div>`,
            item.ID, checked, item.Text,
        )
    }

    return &App{
        store:    s,
        renderer: renderer.NewRenderer(s, container, todoTemplate),
    }
}

func (a *App) AddToDo(text string) {
    newID := fmt.Sprintf("todo-%d", time.Now().UnixNano())
    todo := ToDo{ID: newID, Text: text, Completed: false}
    a.store.Create(todo)
    a.renderer.RenderAll() // Re-renderiza la lista
}

// ... otros métodos como ToggleToDo, DeleteToDo, etc.
```

---

## Ejemplo de Uso: Aplicación de Lista de Tareas

### `main.go` (Punto de entrada WASM)

```go
package main

import (
    "syscall/js"
    "github.com/your-repo/dom"
)

var app *App

func main() {
    c := make(chan struct{}, 0)

    // Crear el adaptador para el contenedor principal de la app
    hostElement := js.Global().Get("document").Call("getElementById", "app-container")
    adapter := dom.NewBrowserAdapter(hostElement)

    // Inicializar la aplicación
    app = NewApp(adapter)

    // Renderizado inicial
    app.store.Create(ToDo{ID: "todo-1", Text: "Aprender Go + WASM", Completed: true})
    app.renderer.RenderAll()

    // Exponer funciones a JavaScript para manejar la entrada del usuario
    js.Global().Set("addToDo", js.FuncOf(addToDoHandler))

    <-c
}

func addToDoHandler(this js.Value, args []js.Value) interface{} {
    text := args[0].String()
    if text != "" {
        app.AddToDo(text)
    }
    return nil
}
```

### `index.html`

```html
<body>
    <h1>Lista de Tareas (Data-Driven)</h1>
    <input type="text" id="new-todo-input" placeholder="Nueva tarea...">
    <button onclick="addToDo(document.getElementById('new-todo-input').value)">Añadir</button>

    <div id="app-container"></div>

    <script src="wasm_exec.js"></script>
    <script>
        // Carga y ejecuta el módulo WASM
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject).then(result => {
            go.run(result.instance);
        });
    </script>
</body>
```

## Ventajas y Desventajas

- **Ventajas**:
    - Excelente para gestionar estados complejos y compartidos.
    - El flujo de datos es unidireccional y predecible.
    - Los componentes se vuelven "tontos", simplemente renderizan datos, lo que los hace más fáciles de probar.
    - Facilita la implementación de funcionalidades como "deshacer/rehacer" al trabajar con snapshots del estado.
- **Desventajas**:
    - Puede ser excesivo para componentes muy simples y aislados.
    - Requiere una disciplina más estricta en la gestión del estado.
    - El rendimiento puede verse afectado si `RenderAll` es ineficiente y se llama con frecuencia (requiere optimizaciones como el renderizado parcial o "diffing").
```
