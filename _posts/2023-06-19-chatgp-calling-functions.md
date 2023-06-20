---
title: ChatGPT Calling Functions ahora la AI puede llamar funciones de código 
author: Benjamin
date: 2023-06-19 10:32:00 -0500
categories: [Programacion, Python, AI, ChatGPT ]
tags: [Programacion, Python, AI, ChatGPT ]
---

![img](https://i.ibb.co/BgZPvw0/Screenshot-2023-06-19-at-20-19-46.png)

Con Chatgpt dimos un salto enorme en el desarrollo de aplicaciones basadas en AI. Anteriormente, necesitábamos recolectar información y procesarla para entrenar nuestros modelos, pero ahora podemos incluso utilizar instrucciones simples para crear asistentes o realizar tareas más complejas mediante técnicas de "prompt engineering". No obstante, OpenAI ha llevado este progreso un paso más allá con la introducción de las "Function calling". Esta nueva funcionalidad permite que ChatGPT ejecute funciones personalizadas que hayamos definido previamente.

Es importante destacar que ChatGPT no ejecutará directamente estas funciones por nosotros. En cambio, nos indicará cuándo es necesario ejecutarlas y qué parámetros se requieren para su funcionamiento. Nosotros seremos responsables de ejecutar la función y devolver el resultado al modelo en formato de texto. A partir de ahí, ChatGPT generará una respuesta amigable para el usuario.

Sin más bla bla, vamos a un ejemplo con código:

La forma base de ejecutar las function callings es la siguiente:

```python
import openai
import json


# Example dummy function hard coded to return the same weather
# In production, this could be your backend API or an external API
def get_current_weather(location, unit="fahrenheit"):
    """Get the current weather in a given location"""
    weather_info = {
        "location": location,
        "temperature": "72",
        "unit": unit,
        "forecast": ["sunny", "windy"],
    }
    return json.dumps(weather_info)

def run_function_calls():
    # Step 1: send the conversation and available functions to GPT
    messages = [{"role": "user", "content": "What's the weather like in Boston?"}]
    functions = [
        {
            "name": "get_current_weather",
            "description": "Get the current weather in a given location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    },
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["location"],
            },
        }
    ]
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo-0613",
        messages=messages,
        functions=functions,
        function_call="auto",  # auto is default, but we'll be explicit
    )
```

Definimos nuestra función custom y le decimos qué estructura de argumentos tiene. Esto no tiene mayor ciencia, así que haremos un pequeño ejemplo con una aplicación de terminal donde consultaremos los productos de una botillería.

En base a un objeto JSON con las siguientes propiedades:

```json
{
    "name": "CACHANTUN CON GAS", 
    "description": "Agua Mineral de gran pureza, embotellada desde 1920 directamente en su vertiente de origen.", 
    "category": "Aguas"
}
```
creamos las siguientes funciones:

```python
products = []
with open('src/calling_functions_chatgpt/liquors.json', 'r') as file:
    products = json.load(file)

def get_products(search: str,limit: int = 10, offset: int = 0):
    filtered = []
    for p in products:
        if search.lower() in p['name'].lower() or search in p['description'].lower() or search.lower() in p['category'].lower():
            filtered.append(p)
    return json.dumps(filtered[offset:limit])

def get_categories():
    categories = []
    for c in list(map(lambda p: p['category'], products)):
        if c not in categories:
            categories.append(c)
    return json.dumps(categories)
```
get_products() obtiene los productos y get_categories() las categorias disponibles de la tienda de licores.


## Implementación de CallFunction 

Definiremos 2 clases principales: `CallFunction` y `ChatGPT`.

```python
@dataclass
class CallFunction(ABC):

    @property
    def manifest(self) -> Dict:
        ...

    @abstractclassmethod
    def execute(self, **kwargs):
        ...

    @property
    def function_name(self) -> str:
        return self.manifest["name"]
```

La clase `CallFunction` es una clase abstracta que actúa como una plantilla para definir funciones personalizadas que pueden ser llamadas desde el modelo de chat `ChatGPT`. Tiene los siguientes métodos y propiedades:

* `manifest`: Es una propiedad que devuelve un diccionario que representa la información de la función.
* `execute`: Es un método abstracto que debe ser implementado por las subclases. Representa la ejecución de la función personalizada y toma como argumentos palabras clave (**kwargs).
* `function_name`: Es una propiedad que devuelve el nombre de la función obtenido del diccionario manifest.


## Implementación de ChatGPT 
```python
class ChatGPT:
    model = "gpt-3.5-turbo-0613"
    tokens = 0
    chat_status: Any = None
    
    def __init__(self, prompt: str, call_functions: list[CallFunction] ):
        self.call_functions = call_functions
        self.functions = list(map(lambda fn: fn.manifest, call_functions))
        self.messages = [{ 'role': 'system', 'content': prompt }]

    def add_message(self, message):
        self.messages.append(message)

    def update_token_usage(self, response):
        self.tokens += response['usage']['total_tokens']

    def execute_function(self, function_name, arguments):
        function_arguments = json.loads(arguments)
        for cf in self.call_functions:
            if cf.function_name == function_name:
                return cf.execute(**function_arguments)

    def get_answer(self, response):
        return response["choices"][0]["message"]["content"]

    def init_chat(self):
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=self.messages,
            functions=self.functions,
            function_call="auto",
            temperature=0
        )
        self.update_token_usage(response)
        return self.get_answer(response)
    
    def ask(self, input_message, temperature=0):
        self.add_message({ 'role': 'user', 'content': input_message })
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=self.messages,
            functions=self.functions,
            function_call="auto",
            temperature=temperature
        )
        self.update_token_usage(response)
        message = response["choices"][0]["message"]
        # verify function calling
        function_call = message.get("function_call")
        if function_call:
            # execute function
            function_name = function_call["name"]
            self.chat_status.status(f"Llamando a la funcion {function_name} {function_call['arguments']}")
            function_response = self.execute_function(function_name, function_call['arguments'])
            self.chat_status.status(f"{function_name}() ejecutada.")
            # update messages
            self.add_message(message)
            self.add_message({ "role": "function", "name": function_name, "content": function_response })
            # send messages to chatgpt
            second_response = openai.ChatCompletion.create(
                model=self.model,
                messages=self.messages,
                temperature=temperature
            )
            self.update_token_usage(second_response)
            return self.get_answer(second_response)
        # save message to context
        self.add_message(message)
        # no function calling 
        return self.get_answer(response)
    
    def progress(self, msg):
        self.chat_status = log.progress(msg)

    def status(self, msg):
        if self.chat_status:
            self.chat_status.status(msg)

    def success(self, msg):
        if self.chat_status:
            self.chat_status.success(msg)

```

La clase ChatGPT es la clase principal que representa un modelo de chat basado en el modelo `GPT-3.5-turbo-6013`. Aquí hay una descripción de sus métodos y propiedades:

* `model`: Una variable de clase que almacena el nombre del modelo `GPT-3.5-turbo-6013` (este es el modelo que soporta function calling) utilizado.
* `tokens`: Una variable de clase que realiza un seguimiento del número total de tokens utilizados.
* `chat_status`: Una variable de instancia que almacena el estado actual del chat.

El método __init__ es el constructor de la clase ChatGPT. Recibe una cadena de texto prompt y una lista de objetos CallFunction llamada call_functions como argumentos. Inicializa las variables de instancia y crea una lista functions que contiene los diccionarios manifest de las funciones proporcionadas.

El método `add_message()` agrega un mensaje a la lista messages que se utiliza para almacenar el historial de mensajes del chat.

El método `update_token_usage()` actualiza el contador de tokens utilizando la información proporcionada en la respuesta del modelo de chat.

El método `execute_function()` toma el nombre de una función y sus argumentos como entrada y busca la función correspondiente en la lista de `call_functions`. Si encuentra una coincidencia, llama al método execute de la función correspondiente con los argumentos proporcionados y devuelve el resultado.

El método `get_answer()` toma la respuesta del modelo de chat y extrae el contenido del primer mensaje de la respuesta.

El método `init_chat()` inicializa el chat enviando una solicitud al modelo de chat. Utiliza el modelo model, la lista messages, la lista functions, y establece la llamada a la función en "auto" con una temperatura de 0. Actualiza el contador de tokens y devuelve la respuesta del modelo de chat.

El método `ask()` se utiliza para hacer una pregunta o enviar un mensaje al modelo de chat. Toma un mensaje de entrada y una temperatura (opcional) como argumentos. Agrega el mensaje del usuario a la lista messages y envía una solicitud al modelo de chat. Al igual que en `init_chat()`, utiliza el modelo model, la lista messages, la lista functions, y establece la llamada a la función en "auto" con la temperatura proporcionada. Actualiza el contador de tokens y procesa la respuesta. Si la respuesta contiene un mensaje de llamada a función, extrae el nombre de la función y sus argumentos, ejecuta la función correspondiente utilizando `execute_function()`, agrega los mensajes relevantes a la lista messages, y envía una segunda solicitud al modelo de chat. Finalmente, devuelve la respuesta del modelo de chat.

Los métodos `progress()`, `status()`, y `success()` nos ayudan a generar mensajes de información al usuario.

## Aplicación de terminal

Ahora crearemos una aplicación de terminal en la que ejecutaremos las llamadas a la API de ChatGPT. Esta será una función que le preguntará al usuario las acciones a realizar.

```python
def command_line(chatgpt: ChatGPT):
    try:
        print()
        assistant_progress = log.progress('AI Assistant')
        assistant_progress.status('Iniciando asistente...')
        assistant_input = chatgpt.init_chat()
        assistant_progress.success('Asistente listo!')
        while True:
            user_input = input(f"{green_color('[Assistant]')} {assistant_input}\n\n{green_color('[User]')} ")
            print()
            chatgpt.progress('Estado chat context')
            # exit command
            if user_input == 'exit':
                break
            # connecting with openai 
            chatgpt.status('Pensando...')
            assistant_input = chatgpt.ask(user_input)
            chatgpt.success("Listo!")
    except KeyboardInterrupt:
        log.info('Saliendo...')
    log.info(f'Total tokens: {chatgpt.tokens}\n')
```

La explicaión de la función `command_line()` es la siguiente:

* La función `command_line()` toma un objeto `ChatGPT` como argumento.

* Dentro de la función, se inicializa una barra de progreso llamada `assistant_progress` que muestra el estado del asistente. Se establece un mensaje de estado inicial indicando que el asistente se está iniciando.

* Se llama al método `init_chat()` del objeto chatgpt para iniciar el chat con el asistente. El resultado se asigna a la variable `assistant_input`, que contiene la respuesta inicial del asistente.

* La barra de progreso se actualiza para indicar que el asistente está listo.

* Se inicia un bucle while que se ejecutará continuamente hasta que se ingrese el comando "exit".

* Se solicita la entrada del usuario con el mensaje [User] y se asigna a la variable `user_input`.

* Si el valor de user_input es igual a "exit", se rompe el bucle y se sale de la función.

* Si no es un comando de salida, se actualiza la barra de progreso.

* Se llama al método `ask()` del objeto chatgpt pasando la `user_input` como argumento. Esto envía la entrada del usuario al asistente y devuelve la respuesta del asistente. La respuesta se asigna a `assistant_input`.

* La barra de progreso se actualiza para indicar que el asistente ha completado su tarea y está listo para responder.

* Si se produce una excepción de interrupción de teclado (por ejemplo, cuando se presiona Ctrl+C), se imprime un mensaje indicando que se está saliendo del programa.

* Finalmente Se imprime la cantidad total de tokens utilizados durante la ejecución del asistente.


Ahora definiremos nuestras CallFunction que ChatGPT ejecutará según corresponda.

```python
class GetProductsCallFunction(CallFunction):
    manifest = {
        "name": "get_products",
        "description": "Obtiene los productos disponibles de la botilleria",
        "parameters": {
            "type": "object",
            "properties": {
                "search": {
                    "type": "string",
                    "description": "busqueda por palabra clave",
                },
                "limit": {
                    "type": "number",
                    "description": "cantidad limite de datos a traer"
                },
                "offset": {
                    "type": "number",
                    "description": "indice donde empieza a traer datos"
                }
            }
          # "required": ["search"],
        }
    }

    def execute(self, **kwargs):
        return get_products(**kwargs)
    

class GetCategoriesCallFunction(CallFunction):
    manifest = {
        "name": "get_categories",
        "description": "Obtiene las categorias disponibles de la botilleria",
        "parameters": {
            "type": "object",
            "properties": {}
        }
    }

    def execute(self, **kwargs):
        return get_categories()

```

Implementamos las clases `GetProductsCallFunction` y `GetCategoriesCallFunction` que heredan de `CallFunction` con sus correspondientes descripciones de los argumentos y las funciones que ChatGPT podrá usar. Ya con estas clases definidas instanciaremos la clase `ChatGPT` con la siguiente configuración:

```python
chatgpt = ChatGPT(
    prompt="Se un util y amable asistente de una botilleria, que resolvera consultas sobre sus productos existentes. trabajaras con un limite de 15 items por consulta",
    call_functions=[
        GetProductsCallFunction(),
        GetCategoriesCallFunction()
    ]
)
```

A la clase `ChatGPT` le entregamos el parámetro `prompt`, donde definimos las instrucciones deseadas, y definimos el parámetro `call_functions`, donde les entregamos las `CallFunctions` definidas previamente.

Finalmente hacemos uso de la función `command_line()`:

```python

command_line(chatgpt=chatgpt)

```

Ahora ejecutamos
```bash
#!/bin/bash
python src/calling_function_chatgpt/app.py
```

Nuestro asistente inicia:

![img2](https://i.ibb.co/B2P52NJ/Screenshot-2023-06-19-at-19-38-42.png)

Y le realizaremos 2 preguntas:
  * ¿Qué categorías de bebidas existen?
  * ¿Qué cervezas tienes disponibles?

Obtendremos las siguientes respuestas:

Preguntamos por las categorias.

![img2](https://i.ibb.co/jWQjvRw/Screenshot-2023-06-19-at-19-39-14.png)

y preguntamos por los productos de la categoria que queramos en este caso preguntamos por cervezas.

![img3](https://i.ibb.co/r6FqcMN/Screenshot-2023-06-19-at-19-39-46.png)


## Conclusiones

ChatGPT ha demostrado ser una herramienta valiosa al responder nuestras preguntas utilizando las funciones que le hemos indicado. Sin embargo, sus capacidades van más allá de simplemente proporcionar respuestas. Esta nueva característica de `calling function` le brinda una mayor versatilidad al desarrollo de aplicaciones basadas en IA.

Con el uso de `calling function` podemos lograr lo siguiente:

* Orquestar acciones fuera de ChatGPT: Ahora podemos utilizar ChatGPT para coordinar y controlar acciones en otros sistemas o servicios, lo que amplía sus posibilidades de uso.

* Ejecutar acciones que ChatGPT no puede hacer: Gracias a la capacidad de integrar funciones externas, podemos realizar tareas complejas que están fuera del alcance de ChatGPT por sí solo. Esto permite abordar una variedad más amplia de problemas y escenarios.

* Integraciones más rápidas y sencillas: La funcionalidad de "calling function" facilita la integración de ChatGPT con otros sistemas y servicios. Esto agiliza el proceso de desarrollo de aplicaciones y permite crear soluciones más completas de manera más eficiente.

* Evitar dependencias con librerías de terceros: Al poder ejecutar acciones externas directamente desde ChatGPT, se reduce la necesidad de depender de librerías o herramientas adicionales. Esto simplifica el proceso de desarrollo y reduce posibles problemas de compatibilidad.

 Este ejemplo simple y fácil demuestra cómo podemos aprovechar la nueva funcionalidad de "calling function" para crear aplicaciones innovadoras más allá del típico asistente de chat. Las posibilidades son amplias y prometen un futuro emocionante para el desarrollo de aplicaciones basadas en IA.


## [Github repository](https://github.com/nullpointer-excelsior/python-examples/tree/master/src/calling_function_chatgpt)

Meme de cortesía

![meme](https://i.ibb.co/PQYfyDY/Screenshot-2023-06-19-at-19-58-50.png)



