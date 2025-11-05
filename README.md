# Contexto y objetivos
En este taller tomamos el rol de un equipo de ingenieros enfocados en el control y la automatización dentro de una empresa. La empresa tiene como objetivo desarrollar sistemas que supervisen y controlen entornos domésticos en tiempo real. Como parte de un producto, el equipo tiene la tarea de crear un sistema de supervisión y alerta de humedad para integrarse con el ecosistema IoT.
Diseñamos un prototipo que mide humedad y genera alertas remotas mediante IoT. Se implementó una arquitectura I2C maestro-esclavo  y un dashboard en ThingSpeak para ver la humedad actual, alertas y la historia de datos.
# Requisitos del sistema
* __Maestro:__ Recibe la medida de humedad, decide si mandar una alerta según un umbral propuesto (humedad > 20%) y pública a ThingSpeak el valor actual, la alerta y la gráfica.
* __Esclavo:__ Mide la humedad y envía el dato por I2C al maestro.
* __Entregables clave:__ Diagrama de actividad, documentación del código y resultados de pruebas.
