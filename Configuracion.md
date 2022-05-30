### GNU Radio Notas

# Configuración


## Módulos out-of-tree

Los módulos out-of-tree suelen construirse desde el código fuente; esto requiere:

1. Conocer dónde están instaladas las bibliotecas y archivos de encabezado de GNU Radio (e.g. libgnuradio-runtime.so, gnuradio/block.h).
1. Bloques de cierta complejidad pueden requerir usar otras partes de GNU Radio, como filtros de kernel FIR o bloques FFT (libgnuradio-filter.so, libgnuradio-fft.so, encabezados).
1. GNU Radio cambia; el módulo out-of-tree debe reflejar los cambios para poder seguir funcionando. Debe indicarse con qué versión de GNU Radio funciona el módulo.

El script CMake 

    $prefix/lib/cmake/gnuradio/GnuradioConfig.cmake

ayuda a resolver estos problemas. La variable `$prefix` designa la ubicación de instalación de GNU Radio, por defecto `/usr/local`, pero la recomendación reciente es un directorio de usuario, e.g. `/home/posiie1/target`.

Variables a ajustar:

- `CMAKE_INSTALL_PREFIX` : directorio de instalación GNU Radio, para CMake.
- `PATH` : ruta hacia ejecutables de GNU Radio.
- `LD_LIBRARY_PATH` : ruta hacia las bibliotecas de GNU Radio.
- `PYTHONPATH` : ruta hacia los módulos de Python.
- `PKG_CONFIG_PATH` : para construir con GNU Radio, ruta hacia  configuraciones de paquetes, archivos de  Package Config, extensión .pc (pkg-config).

Para GNU Radio instalado en `/home/posiie1/target`:

    $ export PATH=/home/posiie1/target/bin:$PATH
    $ export PYTHONPATH=/home/posiie1/target/lib/python2.7/dist-packages:$PYTHONPATH
    $ export LD_LIBRARY_PATH=/home/posiie1/target/lib:$LD_LIBRARY_CONFIG
    $ export PKG_CONFIG_PATH=/home/posiie1/target/lib/pkgconfig:$PKG_CONFIG_PATH
    $ pkg-config --modversion gnuradio-runtime 
    3.7.6git

Indica: 1) el runtime de GNU Radio está instalado; 2) qué versión está instalada; 3) que puede usarse el script `GnuradioConfig`.

En máquina `posiie1`:

    PYTHONPATH:
    /home/posiie1/GWN/GNUnetwork/gwn     # desarrollo GWN nuestro
    /home/posiie1/target/lib64/python2.6/dist-packages/     # no existe
    /home/posiie1/target/lib64/python2.6/site-packages/     # existe, vacío
    /home/posiie1/target/lib64/python2.7/dist-packages/     # no existe
    /home/posiie1/target/lib64/python2.7/site-packages/     # no existe
    /home/posiie1/target/lib/python2.6/dist-packages/       # existe
    /home/posiie1/target/lib/python2.6/site-packages/       # existe, vacío
    /home/posiie1/target/lib/python2.7/dist-packages/       # existe
    /home/posiie1/target/lib/python2.7/site-packages/       # existe, vacío
    /home/posiie1/target/python/
    PATH:
    /home/posiie1/target/bin/
    LD_LIBRARY_PATH:
    /home/posiie1/target/lib/ 
    /home/posiie1/target/lib64/
    PKG_CONFIG_PATH:
    /home/posiie1/target/lib64/pkgconfig/ 
    /home/posiie1/target/lib/pkgconfig/
    
    $ which gnuradio-companion 
    /home/posiie1/target/bin//gnuradio-companion
    
    $ pkg-config --modversion gnuradio-runtime 
    3.7.6git


## Usar bibliotecas de GNU Radio

Referencia: Configuring GNU Radio and Out-of-tree (OOT) modules. Describe los ajustes necesarios para que un nuevo módulo out-of-tree pueda usar las bibliotecas de GNU Radio. Varios de los ajustes descritos en esta referencia ya vienen hechos desde la versión 3.7.3 de GNU Radio.

El archivo donde se realizan los ajustes es `gr-<nuevo_modulo>/CMakeLists.txt`:

1) En las líneas

    set(GR_REQUIRED_COMPONENTS RUNTIME FILTER)
    find_package(Gnuradio "3.7.6" REQUIRED)

sustituir por la versión apropiada de GNU Radio. Las palabras RUNTIME y FILTER indican dos de las bibliotecas de GNU Radio a enlazar en el proyecto; agregar las necesarias, dejar solo las necesarias. Otras bibliotecas:  FILTER, FFT, PMT, AUDIO, ANALOG, CHANNELS. \[Buscar una referencia donde estén descritos estos valores\].


2) Desde la versión 3.7.3 gr_modtool usa GnuradioConfig.cmake, y ya incluye los cambios:

    include_directories, variable ${GNURADIO_ALL_INCLUDE_DIRS}.
    target_link_libraries, variable ${GNURADIO_ALL_LIBRARIES}.

Luego de estos cambios se debe correr `make`, que a su vez corre automáticamente `cmake` porque se modificó `CMakeLists.txt`. Si no se instaló GNU Radio en la localización por defecto (`/usr/local`), la reconstrucción debe indicar el destino:

    $ cd build
    $ cmake -DCMAKE_INSTALL_PREFIX=/home/posiie1/target ../
    $ make

También es posible cambiar el destino de instalación en el momento mismo de instalar, haciendo

    make DESTDIR=/home/posiie1/target install

como indica `man cmake`. Parece mejor indicar el lugar de instalación a `cmake`, que configura las rutas de instalación en el archivo `build/install_manifest.txt`.


## Crear bloque usando bibliotecas de GNU Radio

Crea un bloque para calcular la derivada de una señal usando un filtro FIR de GNU Radio, dentro del módulo howto creado anteriormente.

    $ cd gr-howto                            // directorio del módulo howto
    $ gr_modtool add -t sync derivative_ff
    $ cd build

Crea el nuevo bloque, se traslada al directorio de construcción.

    $ vi python/qa_derivative_ff.py

Crea los tests de verificación.

    $ vi lib/derivative_ff_impl.h

Agrega acceso a las bibliotecas de GNU Radio, define un miembro privado para accederlas:

    #include <gnuradio/filter/fir_filter.h>
    ...
    private:
      gr::filter::kernel::fir_filter_fff *d_fir;
    
    $ vi lib/derivative_ff_impl.cc

En el constructor: ajusta tipos de entradas y salidas, define un vector llamado taps, indica taps a usar ("To calculate a derivative, we use the taps [1, -1]"), invoca el filtro FIR, ajusta historia para tomar 2 muestras para calcular salida de 1 item.

En el destructor: como usó punteros, libera memoria.

    derivative_ff_impl::derivative_ff_impl()
      : gr::sync_block("derivative_ff",
                       gr::io_signature::make(1, 1, sizeof(float)),
                       gr::io_signature::make(1, 1, sizeof(float)))
    {
      std::vector<float> taps;
      taps.push_back(1);
      taps.push_back(-1);
      d_fir = new gr::filter::kernel::fir_filter_fff(1, taps);
      set_history(2);
    }

    derivative_ff_impl::~derivative_ff_impl()
    {
      delete d_fir;
    }

En la función work, invoca el filtro con la función filterN que procesa N muestras simultáneas:

    int
    derivative_ff_impl::work(int noutput_items,
                             gr_vector_const_void_star &input_items,
                             gr_vector_void_star &output_items)
    {
        const float *in = (const float *) input_items[0];
        float *out = (float *) output_items[0];

        d_fir->filterN(out, in, noutput_items);

        // Tell runtime system how many output items we produced.
        return noutput_items;
    }

    $ cd ..    # estaba en build, sube al directorio del módulo howto, gr-howto
    $ vi CMakeLists.txt

Declara acceso a la biblioteca de filtros como componente requerido:

    set(GR_REQUIRED_COMPONENTS RUNTIME FILTER)
    find_package(Gnuradio "3.7.2" REQUIRED)

    $ cd build
    $ cmake -DCMAKE_INSTALL_PREFIX=/home/posiie1/target ../
    $ make

Como alteró `CMakeLists.txt`, corre de nuevo `cmake`.

    $ vi howto_derivative_ff.xml
    $ make test
    $ make install

Ajusta el xml para usar desde GRC (GNU Radio Companion); corre tests; instala.


## Agregar módulo out-of-tree a GNU Radio

Esta sección describe el procedimiento para colocar el nuevo módulo en el repositorio de GNU Radio de modo que pueda instalarse vía PyBOMBS. La instalación de un módulo se realiza conforme lo establecido en una receta (recipe).

1. Create a recipe (an \*.lwr file) for your project. The easiest way is to take a recipe from  https://github.com/gnuradio/recipes and modify as required.
2. Go to the recipes site, https://github.com/gnuradio/recipes, use the Fork button to fork the project.
3. In your forked project, add new recipe (the \*.lwr file) using button “+” to add new file; indicate name of file, content, commit message and extended descripition.
4. In your forked project, go to Pull requests and create New pull request with corresponding button.
5. See your pull request in https://github.com/gnuradio/recipes/pulls.
6. Wait for pull to be done; your new recipe must eventually appear in  https://github.com/gnuradio/recipes.

Referencias:

- Contributing to PyBOMBS, cómo agregar parches o código nuevo,
http://gnuradio.org/redmine/projects/pybombs/wiki/Contributing

Las recetas están en https://github.com/gnuradio/recipes, ahí es donde deben ponerse las recetas de módulos out-of-tree nuevos.

