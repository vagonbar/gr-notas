### GNU Radio Notas


# Módulos "out-of-tree"


## Guías y utilitarios

Out-of-tree modules: Extending GNU Radio with own functionality and blocks, 
  http://gnuradio.org/redmine/projects/gnuradio/wiki/OutOfTreeModules 

Configuring GNU Radio and Out-of-tree (OOT) Modules:
  http://gnuradio.org/redmine/projects/gnuradio/wiki/OutOfTreeModulesConfig

"An out-of-tree module is a GNU Radio component that does not live within the GNU Radio source tree". 

`gr_modtool`

Script para generar un módulo, y agregarle bloques. Crea estructura, archivos, makefiles. Usa plantillas, asume ajustes típicos; módulos muy personalizados pueden requerir ajustes; es una buena forma de comenzar. 

`cmake`

Suele estar instalado. CMake es un software multi plataforma, libre y de código abierto para manejar el proeceso de construcción de software usando métodos independientes del compilador  (Wikipedia: cmake). 


## Crear nuevo módulo 

    $ gr_modtool newmod <nuevo_modulo> 

crea directorio gr-<nuevo_modulo> y varios subdirectorios.

    $ gr_modtool newmod howto

crea directorio gr-howto, el nuevo módulo usado en estos ejemplos.


## Crear bloques

Agregar un nuevo bloque, ejemplo: square_ff : recibe un número en punto flotante (float), lo eleva al cuadrado, lo devuelve como número en punto flotante (float); observar notación: sufijo _ff indica "recibe float, devuelve float".
Este bloque hereda directamente de gr::block. Requiere: 

- cambiar signatura del constructor, sustituir los parámetros indicados. 
- ajustar función `forecast()`, indica `ninput_items requeridos para producir noutput_items`.
- sobreescribir función `general_work()`.

**Agregar un bloque, `square_ff`**

    $ cd gr-howto/                           # “howto” es el nombre del módulo
    $ gr_modtool add -t general square_ff    # agrega bloque 
    Language (python/cpp): cpp
    Block/code identifier: square_ff 
    Enter valid argument list, including default arguments: 
    Add Python QA code? [Y/n] y     # QA Quality Assurance, tests en Python 
    Add C++ QA code? [y/N] n        # QA Quality Assurance, tests en C++ 

Sugerencia: aceptar siempre crear el bloque QA en Python, o en ambos (Python y C++), aunque no se codifiquen pruebas en el momento. Si se crean luego pueden no quedar incluidos en los scripts para  recompilar o instalar.

**Agregar código de prueba unitaria en Python**

    $ vi python/qa_square_ff.py 

Agregar pruebas unitarias para este módulo; sufijo qa_ indica tests "quality assurance".   
`gr_modtool` modifica `python/CMakeLists.txt` para incluir las pruebas. 

Ejemplo de código de prueba para el bloque square_ff:

    from gnuradio import gr, gr_unittest
    from gnuradio import blocks
    import howto_swig as howto
    
    class qa_square_ff (gr_unittest.TestCase):
    
        def setUp (self):
            self.tb = gr.top_block ()
    
        def tearDown (self):
            self.tb = None
    
        def test_001_square_ff(self):
            src_data = (-3, 4, -5.5, 2, 3)
            expected_result = (9, 16, 30.25, 4, 9)
            src = blocks.vector_source_f(src_data)
            sqr = howto.square_ff()
            dst = blocks.vector_sink_f()
            self.tb.connect(src, sqr)
            self.tb.connect(sqr, dst)
            self.tb.run()
            result_data = dst.data()
            self.assertFloatTuplesAlmostEqual(expected_result, result_data, 6)
    
    if __name__ == '__main__':
        gr_unittest.run(qa_square_ff, "qa_square_ff.xml")
    
**Código C++**

Ver Blocks Coding Guide.

Creados por gr_modtool: 

    lib/square_ff_impl.h, 
    lib/square_ff_impl.cc, 
    include/howto/square_ff.h, 

completo si no hay que agregar métodos públicos e.g. set(), get(). 

**Agregar código funcional del bloque**

    $ vi lib/square_ff_impl.cc 

Código de bloque `square_ff como gr::block`:

    square_ff_impl::square_ff_impl()
      : gr::block("square_ff",
                  gr::io_signature::make(1, 1, sizeof (float)), // input signature
                  gr::io_signature::make(1, 1, sizeof (float))) // output signature
    {
        // empty constructor
    }
    
    void square_ff_impl::forecast (int noutput_items, 
                                   gr_vector_int &ninput_items_required)
    {
      ninput_items_required[0] = noutput_items;
    }
    
    int square_ff_impl::general_work (int noutput_items,
                                      gr_vector_int &ninput_items,
                                      gr_vector_const_void_star &input_items,
                                      gr_vector_void_star &output_items)
    {
        const float *in = (const float *) input_items[0];
        float *out = (float *) output_items[0];
        for(int i = 0; i < noutput_items; i++) {
            out[i] = in[i] * in[i];
        }

        // Tell runtime system how many input items we consumed on
        // each input stream.
        consume_each (noutput_items);
    
        // Tell runtime system how many output items we produced.
        return noutput_items;
    }

Subclases de gr::block para situaciones comunes:

- `gr::sync_block` deriva de gr::block e implementa un bloque 1:1, con historia opcional.
- `gr::sync_decimator`  deriva de gr::block e implementa un bloque N:1, con historia opcional.
- `gr::sync_interpolator`  deriva de gr::block e implementa un bloque 1:N, con historia opcional.

Estas subclases no requieren función forecast(), y sobreescriben la función work().


**Construcción del bloque**

    $ mkdir build    # We're currently in the module's top directory 

Crea el directorio de construcción, por única vez para cada módulo.

    $ cd build/ 
    $ cmake ../      # Tell CMake that all its config files are one dir up 

Asume instalación de GNU Radio a nivel de sistema.

    $ cmake -DCMAKE_INSTALL_PREFIX=/home/posiie1/target ../

Indica el directorio de instalación, necesario para GNU Radio instalado en directorio de usuario (recomendación actual). En este ejemplo el usuario es posiie1, directorio propio /home/posiie1.

Mensajes:

    ... 
      -- Build files have been written to: /home/posiie1/gr-howto/build 

    $ make          # And start building (should work after the previous section) 
    ... 
    [100%] Generating documentation with doxygen 
    [100%] Built target doxygen_target 

    $ ls ../cmake/Modules/    # para ver lo que armó 

**Correr tests**

    $ make test    # para correr los tests de unidad 

invoca el programa ctest de CMake, que admite varias opciones; ver man ctest. 

    $ ctest -V -R square 

da salida más detallada, para usar si hubo errores; V verboso, R expresión regular para seleccionar los tests. 

**Agregar bloque square2_ff basado en `gr::sync`**

Requerimientos comunes para un nuevo bloque: 

- relación de entradas a salidas, forecast() y consume() para rate variable; 1:1, 1:N, N:1. 
- cantidad de entradas examinadas para producir una salida, set_history(). 

Bloques tipo para heredar: `gr::sync 1:1, gr::sync_decimator N:1, gr::sync_interpolator 1:N`. 

Agregar un bloque que hereda de `gr::sync_block`. Requiere: 

- cambiar signatura del constructor, sustituir parámetros indicados. 
-  sobreescribir work(). 

`gr::sync_block` deriva de `gr::block` e implementa un bloque 1:1; la historia es opcional. Define `work()` en lugar de `general_work()`; no requiere `ninput_items`, e invoca `consume_each()`. 

    $ vi ../python/qa_square_ff.py  # agregar otro test 
    
    $ cd .. 
    $ gr_modtool add -t sync square2_ff

agrega bloque; no incluye Python QA, porque usa el archivo de test ya creado `qa_square_ff.py`.

    $ vi lib/square2_ff_impl.cc 

Corregir entradas y salidas, agregar código para generar i**3. La función `work()` viene incluida por heredar de `gr::sync`.

Código de bloque `square2_ff` como `gr::sync`:

    square2_ff_impl::square2_ff_impl()
      : gr::sync_block("square2_ff",
                       gr::io_signature::make(1, 1, sizeof (float)),
                       gr::io_signature::make(1, 1, sizeof (float)))
    {}
    
    // [...] skip some lines ...
    
    int
    square2_ff_impl::work(int noutput_items,
                          gr_vector_const_void_star &input_items,
                          gr_vector_void_star &output_items)
    {
      const float *in = (const float *) input_items[0];
      float *out = (float *) output_items[0];
    
      for(int i = 0; i < noutput_items; i++) {
        out[i] = in[i] * in[i];
      }

      // Tell runtime system how many output items we produced.
      return noutput_items;
    }

    $ cd build   # compila
    $ make       # cmake corre automático porque gr_modtools los CMakeLists.txt 

**Agregar bloque en Python**

Escribir bloques en Python puede implicar menor eficiencia en situaciones donde el tiempo de respuesta es crítico, en cuyo caso se usa sobre todo en prototipos. El proceso es análogo.

Disponibles en Python: `gr.sync_block, gr.decim_block, gr.interp_block, gr.basic_block (versión Python de gr::block)`. 

Diferentes signaturas en funciones de trabajo, usa arrays de numpy: 

      def work(self, input_items, output_items): 
        # Do stuff 
      ...
      def general_work(self, input_items, output_items): 
          # Do stuff 

Tipos de bloque en Python:

- `gr.sync_block` 
- `gr.decim_block` 
- `gr.interp_block` 
- `gr.basic_block` – La versión Python de `gr::block` 

Código de bloque python/square3_ff.py como gr::sync_block:

    import numpy
    from gnuradio import gr
    
    class square3_ff(gr.sync_block):
        " Squaring block " 
        def __init__(self):
            gr.sync_block.__init__(
                self,
                name = "square3_ff",
                in_sig = [numpy.float32], # Input signature: 1 float at a time
                out_sig = [numpy.float32], # Output signature: 1 float at a time
            )
    
        def work(self, input_items, output_items):
            output_items[0][:] = input_items[0] * input_items[0] # Only works because numpy.array
            return len(output_items[0])

## Colocar los nuevos bloques en GRC 

**Crear los XML** 

    $ cd .. 
    $ gr_modtool makexml square2_ff 
      ... 
      Overwrite existing GRC file? [y/N] y    # es un esqueleto, sobreescribir
    
      $ ls grc 
      CMakeLists.txt  howto_square2_ff.xml  howto_square_ff.xml 

Los xml pueden requerir retoques para incluir parámetros.

Para colocar los bloques en subcategorías, visibles en GRC:

    <category>GWN/ARQ</category>

El bloque aparece bajo GWN / ARQ, con ARQ como subcategoría de GWN.

**Poner disponibles en GRC**

    $ cd build 
    $ make install 
    $ sudo ldconfig

Este último paso el importante pues le da visibilidad en Python a lo que armamos. Un síntoma de este olvido es que tenemos un módulo python con el nombre correcto, pero esta vacío. 

Para desinstalar:

    $ make uninstall

**Eliminar bloques de GRC**

    $ gr_modtool rm REGEX

donde REGEX es una expresión regular que incluye los nombres de bloques a eliminar (y solo ellos...).

Este comando elimina los archivos de construcción del bloque, pero no quita el xml del Companion, donde el bloque se sigue viendo. Para quitar el bloque del Companion:

    rm target/share/gnuradio/grc/blocks/<nombre_modulo>_<nombre_bloque>.xml

Deshabilitar bloques, quitándolos del los archivos Cmake:

    $ gr_modtool disable REGEX

No parece haber una forma sencilla de revertir esta acción, i.e. de habilitar el bloque deshabilitado.


*GNU Radio en directorio de usuario*

Cuando GNU Radio se instaló a nivel de sistema (con sudo), el procedimiento anterior no funciona: los proyectos intentan instalar sus archivos en /usr/local, y no tienen permiso; hacer sudo make install los coloca en /usr/local, pero luego GRC no encuentra los bloques ni las bibliotecas.

El directorio de instalación puede cambiarse a través de cmake. Para GNU Radio instalado en `/home/posiie1/target`:

    $ cmake -DCMAKE_INSTALL_PREFIX=/home/posiie1/target ../

cambia directorio de instalación en archivo build/cmake_install.cmake, línea  
`SET(CMAKE_INSTALL_PREFIX "/home/posiie1/target")`

    $ make

actualiza rutas en el archivo `build/install_manifest.txt`.

    $ make install

instala en la ubicación especificada.

    $ make uninstall

desinstala.


## Resumen 

Crear módulo, agregar bloques C++, añadir a GRC.

    $ gr_modtool newmod <nuevo_modulo>

- crea directorio gr-<nuevo_modulo> y en él varios subdirectorios y archivos.
- los bloques quedarán en el módulo Python <nuevo_modulo>.
- quedan disponibles con import <nuevo_modulo>, aunque se codifiquen en C++.

    $ cd gr-<nuevo_modulo>/

Bloque genérico: 

    $ gr_modtool add -t general <bloque>_ff 

- agrega bloque heredero de gr::block, en C++. 
- sufijo ff es float-float, indica tipo de entrada y salida, por convención de nombres.
- pregunta por lista de parámetros, y si agrega código de prueba Python y C++. 

Bloque heredero de sync, decimator, interpolator: 

    $ gr_modtool add -t sync <bloque>_ff 

- agrega bloque heredero de gr::sync, en C++.
   
    $ vi python/qa_<bloque>_ff.py 

- para agregar código de prueba correspondiente a este bloque.
- ver modelo de test en instructivo Out of tree modules.

    $ vi lib/<bloque>_ff_impl.cc 

- modificar signaturas de funciones para entradas, salidas. 
- agregar código de constructor, si es necesario. 

Bloque genérico:

- en forecast() definir cantidad de items de entrada y salida.
- en general_work() agregar código procesamiento.

Bloque heredero `sync`, `decimator`, `interpolator`:

- en `work()` agregar código procesamiento. 

    $ mkdir build 
    $ cd build 
    $ cmake -DCMAKE_INSTALL_PREFIX=/home/posiie1/target ../

- arregla archivos de cmake para instalar en el directorio indicado de instalación de GNU Radio.

    $ make  

construye el módulo. 

    $ make test 

- corre los tests 

    $ ctest -V -R <regexp> 

- corre tests con salida verbosa, selecciona tests por <regexp>. 

    $ gr_modtool makexml <bloque>_ff 

- crea código XML para invocar el bloque. 
- sobreescribir GRC? sí, el existente es solo una plantilla. 

    $ vi ../grc/<nuevo_modulo>_<bloque>_ff.xml
 
- modificar si es necesario. 

    $ make install 
    $ sudo ldconfig   # para Ubuntu 

- permite usar los bloques en GRC.

**Agregar un bloque Python**

Para el código de prueba, puede haber un único archivo QA para el módulo, con pruebas para distintos bloques de ese módulo, ya sean bloques C++ o Python. Ver modelo en el tutorial Out of tree modules. 

    $ gr_modtool add -t sync -l python <bloque_python>_ff 

- agrega bloque Python, hereda de bloque sync en Python. 
- ajustar signaturas de entradas y salidas. 
- en `work()` agregar código de procesamiento. 



