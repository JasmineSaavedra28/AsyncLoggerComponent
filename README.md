# README - Propuesta de Componente Concurrente: AsyncLoggerComponent

**Actividad Asincrónica**  
**Docente:** Gabriel Nicolás González Ferreira  
**Materia:** Modelo de Componente Concurrente en C++  
**Proyecto:** Propuesta y Especificación de Requerimientos de un Nuevo Componente Concurrente  

---

## 1. Visión General del Componente

**AsyncLoggerComponent** es un módulo de registro (*logging*) asíncrono y concurrente de alto rendimiento diseñado para integrarse dinámicamente en el **Modelo de Componente Concurrente en C++**. Su propósito principal es permitir que la aplicación Host (`HostApp`) y otros componentes del sistema registren eventos, métricas y errores de manera no bloqueante, evitando que las operaciones de Entrada/Salida (E/S) a disco o consola afecten los hilos de ejecución principales.

El componente se distribuye como una biblioteca compartida dinámica (`lib/async_logger.so` en Linux / `lib/async_logger.dll` en Windows) y respeta de forma estricta los contratos del sistema: la interfaz base (`IComponent`), la frontera binaria pura C (`extern "C"` y `noexcept`), el manejo de memoria RAII mediante *custom deleters* en el `ModuleManager`, y las convenciones de codificación del proyecto.

---

## 2. Justificación de Independencia como Componente

Para justificar que este módulo constituya un componente dinámico independiente (y no una simple clase interna del Host), se cumplen tres principios clave:

1. **Desacoplamiento Operativo y E/S No Bloqueante**: La escritura en disco involucra llamadas al sistema (*syscalls*) de almacenamiento que introducen latencias impredecibles. Convertir el *logger* en un componente independiente desacopla la E/S en un hilo de trabajo (*worker thread*) dedicado.
2. **Ciclo de Vida y Despliegue Independiente**: Permite modificar o actualizar la estrategia de registro (por ejemplo, agregar rotación de archivos, compresión o formateo en JSON) recompilando únicamente la biblioteca compartida `lib/async_logger.so`, sin detener ni volver a compilar el binario del Host.
3. **Reutilización y Reemplazabilidad**: Múltiples aplicaciones Host o componentes secundarios pueden instanciar y compartir el *logger* a través de la interfaz `ILogger`. Asimismo, en fases de prueba se puede sustituir por un componente simulado (*mock*) cargado dinámicamente.

---

## 3. Listado de Requerimientos (Especificación Técnica)

### 3.1. Requerimientos Funcionales (RF)

| ID | Nombre | Descripción |
| :--- | :--- | :--- |
| **RF-01** | **Registro Asíncrono** | El componente debe proveer el método `log_message` para encolar mensajes de forma no bloqueante desde múltiples hilos concurrentes. |
| **RF-02** | **Niveles de Severidad** | Debe soportar 6 niveles de severidad: `LOG_TRACE`, `LOG_DEBUG`, `LOG_INFO`, `LOG_WARN`, `LOG_ERROR`, y `LOG_FATAL`. |
| **RF-03** | **Filtrado por Nivel Mínimo** | Debe permitir configurar un nivel mínimo de registro (`set_min_level`). Todo mensaje con severidad inferior al nivel configurado se descartará inmediatamente sin costo de E/S. |
| **RF-04** | **Salida Configurable** | Debe permitir redirigir la salida hacia un archivo en disco (`set_log_file`) o enviar a la salida estándar/error por defecto. |
| **RF-05** | **Formateo Estructurado** | Cada registro escrito debe incluir la marca de tiempo (precisión de ms), el ID del hilo emisor, el nivel de severidad y el cuerpo del mensaje. |
| **RF-06** | **Vaciado Explícito (*Flush*)** | Debe ofrecer el método `flush()` que bloquea al cliente hasta garantizar que todos los mensajes encolados se hayan escrito y vaciado en el flujo de salida (para persistencia física real se requeriría `fsync`, fuera del alcance mínimo). |
| **RF-07** | **Cierre Ordenado (*Graceful Shutdown*)** | Al invocarse `destroy_component`, el componente debe procesar los mensajes pendientes en la cola antes de finalizar su hilo de trabajo y cerrar el archivo. |
| **RF-08** | **Retorno de Estado C-API** | Cada operación debe retornar un código de estado tipo `ComponentResult` indicando éxito (`SUCCESS`) o el error específico (`ERROR_INVALID_ARGUMENT`, `ERROR_INTERNAL`). |
| **RF-09** | **Política de Saturación** | Si la cola alcanza `MAX_QUEUE_CAPACITY`, `log_message` debe descartar el mensaje nuevo y retornar `ERROR_INTERNAL` sin bloquear al hilo cliente. *(Alternativa a evaluar: agregar `ERROR_QUEUE_FULL` a `ComponentResult`, lo que exigiría incrementar la versión del ABI).* |
| **RF-10** | **Validación de Argumentos** | `log_message` debe retornar `ERROR_INVALID_ARGUMENT` si `message` es `nullptr`; `set_log_file` debe hacerlo si la ruta es nula o vacía. |
| **RF-11** | **Reapertura Segura de Archivo** | `set_log_file` debe ser *thread-safe* y no interrumpir al *worker*: la reapertura del flujo se realiza bajo exclusión mutua y, si falla, se conserva la salida anterior y se retorna `ERROR_INTERNAL`. |
| **RF-12** | **Sin `std::exit` ni Excepciones Cruzadas** | El componente no debe invocar `std::exit`; `create_component` retorna `nullptr` ante cualquier fallo interno y ninguna excepción C++ puede atravesar el ABI. |

### 3.2. Requerimientos No Funcionales (RNF)

| ID | Nombre | Descripción |
| :--- | :--- | :--- |
| **RNF-01** | **Seguridad de Hilos (*Thread Safety*)** | Todas las llamadas públicas de `ILogger` deben ser *thread-safe*, soportando invocaciones simultáneas desde múltiples hilos sin *race conditions*. |
| **RNF-02** | **Frontera ABI C y `noexcept`** | Las funciones exportadas deben usar `extern "C"` y `noexcept`. Ninguna excepción de C++ debe cruzar el límite del componente. |
| **RNF-03** | **Incompatibilidad de Tipos C++ Evitada** | No se deben pasar objetos de la STL (`std::string`) a través del ABI. Se utilizarán únicamente tipos primitivos y buffers C (`const char*`, `size_t`). |
| **RNF-04** | **Gestión RAII de Memoria** | Prohibido el uso de `new`/`delete` desnudos expuestos al Host. `ModuleManager` administra las instancias mediante `std::shared_ptr` con un *custom deleter* que llama a `destroy_component`. |
| **RNF-05** | **Rendimiento en Encolado** | La llamada a `log_message` debe ser de complejidad $O(1)$ amortizada (sin contar la copia del mensaje) y no debe realizar operaciones síncronas de disco ni de consola. |
| **RNF-06** | **Cumplimiento de Nomenclatura** | Archivos en `snake_case`, clases en `PascalCase`, métodos en `snake_case`, atributos privados con sufijo `_` (`snake_case_`), y constantes en `UPPER_SNAKE_CASE`. |
| **RNF-07** | **Capacidad Acotada de Cola** | La cola interna contará con un límite de capacidad (`MAX_QUEUE_CAPACITY = 10000`) para evitar el agotamiento de memoria bajo sobrecarga extrema. |
| **RNF-08** | **Estándar C++17** | El código se compilará bajo el estándar C++17 con las opciones de código independiente de posición (`-fPIC`) y creación de librería compartida (`-shared`). |
| **RNF-09** | **Portabilidad Linux/Windows** | El mismo código fuente debe compilar con `g++`/MinGW; `SharedLibrary` resuelve la extensión `.so` o `.dll` según la plataforma. |
| **RNF-10** | **Sin Dependencias Externas** | Solo se permite la biblioteca estándar de C++17 y `pthread` en Linux; no se admiten frameworks de *logging* de terceros. |

---

## 4. Contrato de Interfaz y Especificación del ABI

### 4.1. Archivo de Cabecera de Interfaz: `include/i_logger.hpp`

El contrato de negocio extiende la interfaz base `IComponent`:

```cpp
#ifndef I_LOGGER_HPP
#define I_LOGGER_HPP

#include "i_component.hpp"
#include <cstddef>

/**
 * @brief Niveles de severidad para el sistema de logging.
 * @details Se usa un tipo subyacente explícito (int) para mantener un ABI estable.
 */
enum class LogLevel : int
{
    LOG_TRACE = 0,
    LOG_DEBUG = 1,
    LOG_INFO  = 2,
    LOG_WARN  = 3,
    LOG_ERROR = 4,
    LOG_FATAL = 5
};

/**
 * @brief Interfaz de negocio del componente de registro asíncrono.
 * Hereda de IComponent para garantizar compatibilidad con el ModuleManager.
 */
class ILogger : public IComponent
{
public:
    virtual ~ILogger() noexcept = default;

    /**
     * @brief Encola un mensaje para su procesamiento asíncrono.
     * @param level Nivel de severidad del mensaje.
     * @param message Cadena C-style finalizada en nulo con el contenido.
     * @return ComponentResult::SUCCESS si se encoló correctamente.
     */
    virtual ComponentResult log_message(LogLevel level, const char* message) noexcept = 0;

    /**
     * @brief Configura la ruta del archivo de salida.
     * @param file_path Ruta del archivo de log.
     */
    virtual ComponentResult set_log_file(const char* file_path) noexcept = 0;

    /**
     * @brief Establece el nivel mínimo de severidad para registrar.
     * @param level Nivel de severidad mínimo.
     */
    virtual ComponentResult set_min_level(LogLevel level) noexcept = 0;

    /**
     * @brief Bloquea hasta que todos los mensajes encolados sean procesados.
     */
    virtual ComponentResult flush() noexcept = 0;
};

#endif // I_LOGGER_HPP
```

### 4.2. Funciones C-API Exportadas (Puntos de Entrada DLL)

Cada binario dinámico exporta externamente los tres símbolos requeridos por el `ModuleManager`. La firma debe coincidir exactamente con los *typedefs* de `i_component.hpp`:

```cpp
extern "C" {
    int get_api_version() noexcept;              // Retorna CURRENT_API_VERSION (constexpr int)
    IComponent* create_component() noexcept;     // Retorna nullptr si falla la construcción
    void destroy_component(IComponent* instance) noexcept;
}
```

> **Nota de versionado:** si se modifica la interfaz `ILogger` (se agregan, quitan o cambian métodos virtuales), debe incrementarse `CURRENT_API_VERSION` en `i_component.hpp` para que `ModuleManager` rechace binarios incompatibles.

---

## 5. Diseño Arquitectónico y Concurrencia

### 5.1. Flujo Productor-Consumidor Concurrente

El componente implementa internamente el patrón **Productor-Consumidor**:

1. **Productores (Hilos del Host)**: Invocan `log_message()`. Adquieren el cerrojo (`queue_mutex_`), insertan la entrada en `log_queue_` y notifican mediante `queue_cv_.notify_one()`.
2. **Consumidor (`worker_thread_`)**: Permanece a la espera en `queue_cv_.wait()`. Al recibir notificación, extrae las entradas pendientes de la cola y realiza la E/S física hacia el archivo o consola.

```
       [ Hilo Host 1 ] ──┐
                         ├──> [ log_message() ] ──> (std::lock_guard) ──> [ log_queue_ ]
       [ Hilo Host 2 ] ──┘                                                      │
                                                                        (cv.notify_one())
                                                                                │
                                                                                ▼
                                                                  [ worker_thread_ ]
                                                                                │
                                                                                ▼
                                                                  Escritura en Disco / Consola
```

### 5.2. Declaración de la Clase Concreta: `src/async_logger_component.hpp`

```cpp
#ifndef ASYNC_LOGGER_COMPONENT_HPP
#define ASYNC_LOGGER_COMPONENT_HPP

#include "i_logger.hpp"
#include <cstddef>
#include <cstdint>
#include <queue>
#include <string>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <atomic>
#include <fstream>

class AsyncLoggerComponent : public ILogger
{
public:
    AsyncLoggerComponent();
    virtual ~AsyncLoggerComponent() noexcept override;

    virtual ComponentResult log_message(LogLevel level, const char* message) noexcept override;
    virtual ComponentResult set_log_file(const char* file_path) noexcept override;
    virtual ComponentResult set_min_level(LogLevel level) noexcept override;
    virtual ComponentResult flush() noexcept override;

private:
    struct LogEntry
    {
        LogLevel level;
        std::string text;
        std::thread::id thread_id;
        uint64_t timestamp_ms;
    };

    void worker_loop();
    void write_entry_to_stream(const LogEntry& entry);
    void close_output_stream();

    std::queue<LogEntry> log_queue_;
    mutable std::mutex queue_mutex_;
    std::condition_variable queue_cv_;
    std::thread worker_thread_;

    std::atomic<bool> is_running_{true};
    std::atomic<LogLevel> min_level_{LogLevel::LOG_INFO};
    std::atomic<size_t> pending_count_{0};

    std::mutex flush_mutex_;
    std::condition_variable flush_cv_;

    std::mutex file_mutex_;
    std::ofstream output_file_stream_;
    std::string current_file_path_;

    static constexpr size_t MAX_QUEUE_CAPACITY = 10000;
};

#endif // ASYNC_LOGGER_COMPONENT_HPP
```

---

## 6. Verificación de Cumplimiento de Normas y Convenciones

El diseño responde rigurosamente a los **Acuerdos de Codificación y Nomenclatura en C++**:

- **Archivos del Sistema (`snake_case`)**: `i_logger.hpp`, `async_logger_component.hpp`, `async_logger_component.cpp`.
- **Clases y Estructuras (`PascalCase`)**: `ILogger`, `AsyncLoggerComponent`, `LogEntry`, `LogLevel`, `ComponentResult`.
- **Funciones y Métodos (`snake_case`)**: `log_message()`, `set_log_file()`, `set_min_level()`, `flush()`, `worker_loop()`.
- **Atributos Privados (`snake_case_`)**: `log_queue_`, `queue_mutex_`, `queue_cv_`, `worker_thread_`, `is_running_`, `min_level_`, `pending_count_`, `flush_mutex_`, `flush_cv_`, `file_mutex_`, `output_file_stream_`.
- **Constantes y Macros (`UPPER_SNAKE_CASE`)**: `MAX_QUEUE_CAPACITY`, `CURRENT_API_VERSION`.
- **Constructores / Destructores (`PascalCase`)**: `AsyncLoggerComponent()`, `~AsyncLoggerComponent() noexcept`.
- **Gestión RAII y Excepciones**:
  - El destructor finaliza limpiamente el `worker_thread_`, vacía la cola pendiente y cierra el archivo antes de liberar descriptores.
  - Métodos marcados con `noexcept` y envueltos en bloques `try-catch (...)` para impedir la fuga de excepciones a través del ABI.
  - `create_component` captura cualquier excepción y retorna `nullptr`; `destroy_component` delega en `delete` (que admite `nullptr`).

---

## 7. Estructura del Proyecto y Compilación

### 7.1. Árbol de Directorios

```
cpp-component-model/
├── include/
│   ├── application.hpp            # Orquestador de la lógica de negocio del Host
│   ├── i_component.hpp            # Interfaz base y C-API
│   ├── i_logger.hpp               # Interfaz de negocio del componente Logger
│   ├── module_manager.hpp         # Gestor de módulos dinámicos
│   └── shared_library.hpp         # Cargador RAII de bibliotecas (.so/.dll)
├── src/
│   ├── async_logger_component.hpp # Cabecera privada del componente
│   └── async_logger_component.cpp # Implementación concreta del componente
├── lib/
│   └── async_logger.so            # Biblioteca compartida compilada
├── build.sh                       # Script de compilación Linux
├── build.bat                      # Script de compilación Windows
└── main.cpp                       # Punto de entrada del Host
```

### 7.2. Comandos de Compilación (`g++` C++17)

```bash
# 1. Compilar el componente AsyncLogger como biblioteca compartida (DSO)
g++ -std=c++17 -c -fPIC -I./include src/async_logger_component.cpp -o async_logger_component.o
g++ -std=c++17 -shared -o lib/async_logger.so async_logger_component.o -pthread

# 2. Compilar la aplicación Host (enlazando la librería de carga dinámica -ldl)
g++ -std=c++17 -I./include main.cpp -o host.app -ldl -pthread

# 3. Ejecutar la aplicación principal
./host.app
```

> En Windows/MinGW el componente se genera como `lib\async_logger.dll` y el Host como `host.exe`. `SharedLibrary` agrega automáticamente la extensión `.so` o `.dll`, por lo que la ruta que recibe `load_module` debe ir **sin extensión**: `./lib/async_logger`.

---

## 8. Análisis de Viabilidad y Factibilidad (Para Debate en Clase)

1. **Viabilidad Técnica**: **Alta**. Utiliza exclusivamente características nativas de C++17 (`std::thread`, `std::mutex`, `std::condition_variable`, `std::atomic`) compatibles en Linux y Windows.
2. **Factibilidad de Integración**: **Alta**. Se acopla sin fricción al `ModuleManager` y `SharedLibrary` del proyecto, requiriendo únicamente invocar `load_module("./lib/async_logger")` y `create_instance<ILogger>("async_logger")`.
3. **Impacto en Rendimiento**: **Positivo**. Al eliminar el bloqueo por E/S síncrona en el hilo principal del Host, se incrementa notablemente la tasa de procesamiento (*throughput*) y la capacidad de respuesta de la aplicación.
**4. Riesgos y Mitigaciones:**

| Riesgo | Impacto | Mitigación propuesta |
| :--- | :--- | :--- |
| Saturación de la cola bajo carga extrema | Pérdida de mensajes o crecimiento de memoria | Cola acotada (`MAX_QUEUE_CAPACITY`) y política de descarte con `ERROR_INTERNAL` (RF-09). |
| Latencia impredecible del disco | Retraso en el *worker* | E/S exclusiva en el hilo *worker*; el Host nunca espera la escritura. |
| Bloqueo indefinido en `flush()` | *Deadlock* del Host | `pending_count_` atómico + `flush_cv_`; el *worker* notifica al llegar a cero. |
| Incompatibilidad de ABI | Corrupción de pila o *crash* | `CURRENT_API_VERSION` validado por `ModuleManager` antes de crear la instancia. |
| Diferencias de exportación en Windows | Símbolos no encontrados por `GetProcAddress` | Mantener `extern "C"` y verificar la exportación con `-Wl,--export-all-symbols` o `__declspec(dllexport)` si fuera necesario. |
| Excepciones internas (por ejemplo, `std::bad_alloc`) | Fuga a través del ABI | `try-catch (...)` en todos los métodos y en `create_component`; retorno de `ComponentResult`/`nullptr`. |

**5. Conclusión preliminar:** el componente es **viable y factible** como nuevo ejemplo del modelo, siempre que se respeten la cola acotada, la sincronización de `flush()` y la validación de versión del ABI.

---

## 9. Ejemplo de Integración en el Host

La capa `Application` del modelo absorbe la infraestructura y deja `main()` limpio:

```cpp
#include "module_manager.hpp"
#include "i_logger.hpp"
#include <cstdlib>

class Application
{
private:
    ModuleManager module_manager_;

public:
    void initialize()
    {
        // La ruta va sin extensión: SharedLibrary agrega .so o .dll
        module_manager_.load_module("./lib/async_logger");
    }

    int run()
    {
        auto logger = module_manager_.create_instance<ILogger>("async_logger");

        logger->set_min_level(LogLevel::LOG_INFO);
        logger->set_log_file("./logs/app.log");

        logger->log_message(LogLevel::LOG_INFO, "Aplicación iniciada");
        logger->log_message(LogLevel::LOG_ERROR, "Ejemplo de error controlado");

        // Garantiza que los mensajes pendientes lleguen a disco antes de salir
        logger->flush();
        return EXIT_SUCCESS;
    }
};
```

`main.cpp` solo conserva el manejo global de errores de infraestructura:

```cpp
#include "application.hpp"
#include <iostream>
#include <stdexcept>

int main()
{
    try
    {
        Application app;
        app.initialize();
        return app.run();
    }
    catch (const std::exception& e)
    {
        std::cerr << "[FATAL] " << e.what() << std::endl;
        return EXIT_FAILURE;
    }
}
```

---

## 10. Checklist de Ajuste al Modelo de Componentes

| Requisito del modelo | ¿Cumple? | Evidencia en esta propuesta |
| :--- | :---: | :--- |
| Hereda de `IComponent` | ✅ | `class ILogger : public IComponent` |
| Exporta `get_api_version` | ✅ | C-API con `int get_api_version() noexcept` |
| Exporta `create_component` | ✅ | Retorna `IComponent*` o `nullptr` |
| Exporta `destroy_component` | ✅ | `void destroy_component(IComponent*) noexcept` |
| Valida versión del ABI | ✅ | `CURRENT_API_VERSION` y `ModuleManager` |
| Frontera `extern "C"` + `noexcept` | ✅ | Sección 4.2 |
| Sin STL a través del ABI | ✅ | `const char*`, `size_t`, `LogLevel` con tipo subyacente fijo |
| RAII con *custom deleter* | ✅ | `ModuleManager::create_instance<ILogger>` |
| Sin excepciones cruzadas | ✅ | `try-catch (...)` y `ComponentResult` |
| Sin `std::exit` en el componente | ✅ | RF-12 |
| *Thread safety* | ✅ | `std::mutex`, `std::condition_variable`, `std::atomic` |
| Nomenclatura del proyecto | ✅ | Sección 6 |
| Distribución como DSO | ✅ | `lib/async_logger.so` / `lib/async_logger.dll` |
| Integración con `Application` | ✅ | Sección 9 |

---

## 11. Referencias

- Repositorio oficial del modelo: <https://github.com/gabrielinuz/cpp-component-model>
- Documentación de GitHub Markdown: <https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax>
- Sintaxis original de Markdown: <https://daringfireball.net/projects/markdown/syntax>
- Estándar C++17 (ISO/IEC 14882:2017).
