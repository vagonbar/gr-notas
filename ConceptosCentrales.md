### GNU Radio Notas


## Conceptos centrales de GNU Radio

Flow graph 

- es un grafo orientado. 
- nodos = blocks. 
- arcos = las conexiones entre bloques. 
- ports = puntos de entrada o salida de un bloque. 
- items = lo que fluye por las conexiones 

Blocks 

- una sola tarea, modularidad, reutilización ("atomicity"); "a performance vs. modularity tradeoff". 
- escritos en C++; puede ser en Python. 
- source: bloque sin entradas (sin input ports); genera datos o los toma de un driver (acceso al hardware) o del sistema operativo. 
- sink: bloque sin salidas (sin output ports); produce datos o los envía a un driver o al sistema operativo. 

Items 

- todo lo que sale de un bloque. 
- puede ser cualquier cosa expresable en forma digital. 
- lo más común: 
    tipos escalares: real, complex, integer; 
    vectores de estos tipos escalares. 

GNU Radio 

- provee los bloques. 
- el usuario arma el grafo de flujo. 
- ejecuta el grafo de flujo invocando los bloques uno atrás del otro asegurando que los items pasen de un bloque al otro. 
 
Relaciones de muestreo en bloques 

- relación de muestreo:  <cantidad recibida> : <cantidad enviada>, relación entre cantidad de items entrados (puertos de entrada) a cantidad de items salidos (puertos de salida).
- decimator: ingresa 1024 muestras, devuelve un vector, relación N:1. 
- interpolator: envía más items de los que recibe, relación 1:N. 
- sync: recibe y envía la misma cantidad, relación 1:1. 
- general: N:M. 

No hay un reloj de hardware, no hay una "base sampling rate", solo importan las "relative rates", las relaciones entre cantidad de items a la entrada y cantidad de items a la salida. Limitación: capacidad de la CPU, puede bloquearse por asignar el 100% de la capacidad al flow graph. 

Metadata 

- información adicional al flujo de muestras: hora de recepción, frecuencia central, identificación de nodo (protocol specific). 
- stream tag: objeto conectado a un item específico, por ejemplo una muestra. Puede ser un escalar, vector, diccionario u otro. 
- los metadata se pueden guardar a disco junto con el flujo. 

Metadata Documentation page: http://gnuradio.org/doc/doxygen/page_metadata.html 

Protocol Data Units, PDUs 

- identificar bordes: primer byte y largo. 
- dos mecanismos: message passing, tagged stream blocks. 
- Message passing: 

    - método asíncrono para pasar PDUs de un bloque a otro (e.g. para MAC). 
    - Message passing manual page: http://gnuradio.org/doc/doxygen/page_msg_passing.html 
    - Implementado en `gr::basic_block`. Cada bloque tiene colas para mensajes entrantes y puede enviar mensajes a las colas de mensajes de entrada a otros bloques. 

- Tagged stream blocks:

    - usa stream tags para identificar bordes de PDU. 
    - permite mezclar bloques de flujo continuo con bloques de PDU. 

- Conversión entre message passing y tagged stream: 

    - hay bloques para convertir de uno a otro. 


