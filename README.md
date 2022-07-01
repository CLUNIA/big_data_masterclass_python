## Aplicación de seguimiento de vuelos con Python

Existen numerosas aplicaciones de seguimiento de vuelos que nos permiten hacer tracking o seguimiento de qué vuelos hay en qué momento y en qué área del mapa, como [flightradar24](https://www.flightradar24.com/), [FlightAware](https://es.flightaware.com/), [flightview](https://www.flightview.com/), etc, además algunas de ellas nos proporcionan algunos datos interesantes como a qué altitud se encuentra en ese momento un vuelo determinado o a qué velocidad va.

En esta Master Class vamos a ver cómo podemos explotar esta fuente de datos para su estudio o su representación en un mapa, y para ello utilizaremos el lenguaje de programación Python que hemos aprendido en este módulo.

Para esta Master Class utilizaremos como fuente de datos [Opensky Network](https://opensky-network.org/) y como librerías [Pandas](https://pandas.pydata.org/) y [Bokeh](https://docs.bokeh.org/en/latest/). Pandas ya lo conocemos, pero Bokeh no lo habíamos usado hasta ahora. Bokeh es una biblioteca de Python para crear visualizaciones interactivas para navegadores web modernos. Con esta librería podremos realizar gráficos muy vistosos y atractivos, que van desde gráficos simples hasta algunos gráficos más complejos con conjuntos de datos de transmisión. Las visualizaciones están basadas en JavaScript pero no necesitamos escribir ni una línea de código JavaScript, no necesitamos conocer este lenguaje, Bokeh ya hace el trabajo por nosotros.

Vamos a crear una aplicación de seguimiento de vuelos con python utilizando como fuente de datos estas APIs con información del tráfico aéreo en tiempo real. A continuación las diferentes fases que veremos en detalle:
* Obtener datos.
* Importar las bibliotecas necesarias.
* Cargar el mapa base.
* Trazar la posición de los aviones.
* Crear una aplicación de seguimiento de vuelos en "tiempo real".

Vamos a analizar cada una de estas fases y, al final, si la API está disponible y tenemos datos, veremos cómo funciona una aplicación que es capaz de hacer un seguimiento de vuelos en tiempo real (refresca la información cada pocos segundos) y que se ejecuta en el navegador.

<p align="center">
<img src="img/peninsula_iberica_00.png">
Figura 1. Vista de la app de seguimiento de vuelos
</p>
<br>

## Obtención de datos de tráfico aéreo
Estamos usando datos abiertos de tráfico aéreo de OpenSky Network. OpenSky Network es un consorcio sin fines de lucro que proporciona datos de tráfico aéreo abiertos al público, en particular para fines de investigación y no comerciales. Se puede acceder a los datos a través de la API REST.

Para recuperar los datos usando la API REST se puede hacer usando una solicitud HTTP de tipo GET. Se pueden utilizar dos tipos de solicitudes. La primera es la solicitud de un avión específico según la hora en la marca de tiempo UNIX o la dirección ICAO24. Por ejemplo: https://opensky-network.org/api/states/all?time=1458564121&icao24=3c6444

```json
{
    "time": 1458564121,
    "states": [
        [
            "3c6444",
            "DLH9LF  ",
            "Germany",
            1458564120,
            1458564120,
            6.1546,
            50.1964,
            9639.3,
            false,
            232.88,
            98.26,
            4.55,
            null,
            9547.86,
            "1000",
            false,
            0
        ]
    ]
}
```

El segundo, podemos obtener todos los datos del avión dentro de una extensión de área utilizando el sistema de coordenadas WGS84. Además el acceso a los datos puede hacerse de forma anónima o como usuario registrado. La diferencia es que para una solicitud anónima tiene 10 segundos de respuesta y 5 segundos para un usuario registrado.

Nosotros vamos a usar el segundo. Definiremos una extensión de área con coordenadas mínimas y máximas, y luego enviaremos la consulta para obtener todos los datos del avión dentro del área. Por ejemplo, queremos obtener los datos sobre la Península Ibérica con la coordenada mínima -9.9503,35.9425 y la coordenada máxima 4.5517,43.9617. La consulta tanto para el usuario anónimo como para el registrado será la siguiente.
* Solicitud anónima: https://opensky-network.org/api/states/all?lamin=35.9425&lomin=-9.9503&lamax=43.9617&lomax=4.5517


* Solicitud de usuario registrado: https://john_doe:abcd1234@opensky-network.org/api/states/all?lamin=35.9425&lomin=-9.9503&lamax=43.9617&lomax=4.5517

Vamos a asegurarnos de que la consulta es correcta, probemos. Si obtenemos una respuesta como la siguiente, entonces funciona. 

```json
{
  "time": 1655882833,
  "states": [
    [
      "51111d",
      "TOM35Y  ",
      "Estonia",
      1655882832,
      1655882832,
      2.445,
      41.2582,
      11247.12,
      false,
      204.76,
      165.3,
      -7.48,
      null,
      11727.18,
      "0536",
      false,
      0,
      0
    ],
    [
      "4952c9",
      "TAP822W ",
      "Portugal",
      1655882832,
      1655882832,
      -4.4104,
      41.9013,
      11582.4,
      false,
      250.05,
      71.16,
      0,
      null,
      11879.58,
      "2275",
      false,
      0,
      0
    ],
    ...
  ]
}
```
<p align="center">
Figura 2. Respuesta de datos de tráfico aéreo
</p>
<br>

La respuesta, como se muestra en la figura 2, tiene una estructura JSON con dos claves. El primero es time y el segundo es states que contiene datos para cada avión en una matriz de listas. Cada una de las listas almacena muchos datos, como: dirección ICAO24, distintivo de llamada del avión, país de origen, posición horaria, último contacto, longitud, latitud, altitud del barómetro, etc. Para obtener una explicación completa sobre la respuesta de los datos y también más información sobre la API de red de OpenSky, consulte la documentación de la API de OpenSky Network.

## Obtener datos de tráfico aéreo en Python

Ya recuperamos los datos de tráfico utilizando la API REST en un navegador. Ahora hagámoslo en Python. Usaremos Jupyter Notebook con Python 3 y algunas bibliotecas como Bokeh, Pandas, Requests, json y numpy.

A continuación se muestra el código para realizar una solicitud y obtener datos.
Primero importamos las bibliotecas requeridas, como json, pandas y requests.

```Python
import json
import pandas as pd
import requests
```

Luego definimos las coordenadas de extensión en WGS84 (es un sistema geodésico de coordenadas geográficas usado mundialmente, que permite localizar cualquier punto de la Tierra, sin necesitar otro de referencia, por medio de tres unidades dadas x, y, z) mediante las variables lon_min, lat_min, lon_max y lat_max. En este ejemplo pondremos las coordenadas de la península Ibérica, aunque quizás entren datos de alguna región cercana más, pues al final estamos trazando un rectángulo en el mapa.

```Python
# Longitud y latitud máxima y mínima de la Península Ibérica
lon_min, lat_min = -9.9503, 35.9425
lon_max, lat_max = 4.5517, 43.9617
```

Usaremos las coordenadas de extensión para construir la dirección URL a la que haremos la petición de datos. Si quisiéramos usar un usuario registrado, crearemos las variables user y password. Finalmente creamos la variable url en el que concatenamos todos los datos anteriormente especificados.

```Python
user = 'john_doe'
password = 'abcd1234'
query = f'lamin={lat_min}&lomin={lon_min}&lamax={lat_max}&lomax={lon_max}'
url = f'https://{user}:{password}@opensky-network.org/api/states/all?{query}'
```

Ahora es el momento de hacer la petición web a la URL mediante requests y el método get. La respuesta la almacenamos en formato JSON mediante la función json y almacenaremos los datos en la variable response.

```Python
response = requests.get(url).json()
```

Como ya he mencionado anteriormente, estamos utilizando la API de OpenSky Network, que al fin y al cabo es un servicio en internet gratuíto, sin ánimo de lucro y al que accede un gran número de personas constantemente para hacer casos de estudio o simplemente para recibir información en tiempo real, Esto quiere decir que es un servicio muy solicitado y es posible que en ocasiones no esté disponible porque el servidor en el que está alojado esté en mantenimiento o se haya caído, en cuyo caso habría que esperar a que se vuelva a levantar, y a veces pueden pasar horas.
Si en el momento de realizar esta práctica el servicio no estuviera disponible… no estaríamos perdidos del todo, pues yo ya he descargado en un archivo JSON la información de las coordenadas anteriormente citadas y en un momento muy concreto del tiempo, lo que podríamos llamar snapshot o captura de datos. Son datos estáticos, ya no tendríamos datos en tiempo real, pero tendríamos datos para poder seguir trabajando y seguir aprendiendo.
Así pues, única y exclusivamente si este fuera el caso, deberíamos descargar en una carpeta llamada data el archivo IberianPeninsula.json con los datos capturados, e importarlos de la siguiente manera.

```Python
with open('data/IberianPeninsula.json') as data:
    response = json.load(data)
```

Lo bueno de este método de carga de datos es que si ya tienes una captura de datos descargada en tu PC local podrías trabajar offline, es decir, sin acceso a internet.
Ahora, antes de volcar los datos obtenidos en un dataframe de Pandas estableceremos el nombre de todas las columnas con las que trabajaremos, estos nombres se corresponden a cada uno de los datos que obtenemos para cada vuelo.

```Python
col_name = [
  'icao24', 'callsign', 'origin_country', 'time_position', 'last_contact', 
  'long', 'lat', 'baro_altitude', 'on_ground', 'velocity', 'true_track',
  'vertical_rate', 'sensors', 'geo_altitude', 'squawk', 'spi', 'position_source'
]
```

De toda la información que obtuvimos en la respuesta inicial nos quedaremos con el valor del campo states, lo volcaremos a un dataframe de Pandas, obtenemos datos de las 16 columnas de datos y le añadiremos los nombres de las columnas que definimos antes.

```Python
flight_df = pd.DataFrame(response['states'])
flight_df = flight_df.loc[:, 0:16]
flight_df.columns = col_name
```

Ahora reemplazamos los datos en puedan llegar blanco o vacíos con el string 'No Data'.

```Python
flight_df = flight_df.fillna('No Data') # Replace NAN with No Data string.
```

Finalmente comprobamos que tenemos el dataframe usando el método head para ver las primeras 5 filas de datos.

```Python
flight_df.head()
```

Deberemos ver algo como la siguiente imagen:

<p align="center">
<img src="img/peninsula_iberica_dataframe.png">
Figura 3. Primeros datos del dataframe.
</p>
<br>

## Trazar avión en el mapa

Ahora que ya tenemos un dataframe con todos los datos cargados es el momento de ubicar todos los vuelos en un mapa. Para ello usaremos la biblioteca Bokeh, así que la tenemos que importar. También necesitaremos importar la biblioteca NumPy que utilizaremos más adelante en la conversión de coordenadas.

```Python
import numpy as np

from bokeh.plotting import figure, show
from bokeh.tile_providers import get_provider, STAMEN_TERRAIN
from bokeh.models import HoverTool, LabelSet, ColumnDataSource
```

La conversión del sistema de coordenadas la vamos a implementar mediante un par de funciones llamadas wgs84_to_web_mercator_point y wgs84_to_web_mercator.

```Python
def wgs84_to_web_mercator_point(lon, lat):
    """ Function to convert GCS WGS84 to web marcator system. """
    
    k = 6378137  # WGS84 oblate spheroid with this equatorial radius in meters.
    x = lon * (k * np.pi / 180.0)
    y = np.log(np.tan((90 + lat) * np.pi / 360.0)) * k
    
    return x, y
```

```Python
def wgs84_to_web_mercator(df, lon='long', lat='lat'):
    """ Function to get converted Dataframe. """
    
    k = 6378137  # WGS84 oblate spheroid with this equatorial radius in meters.
    df['x'] = df[lon] * (k * np.pi / 180.0)
    df['y'] = np.log(np.tan((90 + df[lat]) * np.pi / 360.0)) * k
    
    return df
```

Como su nombre indica, estas funciones las utilizaremos para transformar coordenadas del sistema WGS84 al sistema de coordenadas universal transversal de Mercator. Esta transformación es necesaria porque vamos a usar un mapa con base STAMEN_TERRAIN en un navegador web que tiene proyección web Mercator (EPSG: 3857).
Realizamos la transformación para el marco de datos y las coordenadas de extensión.

```Python
# COORDINATE CONVERSION
xy_min = wgs84_to_web_mercator_point(lon_min, lat_min)
xy_max = wgs84_to_web_mercator_point(lon_max, lat_max)

wgs84_to_web_mercator(flight_df)
```

A continuación configuramos el mapa especificando la extensión del área en función del rango de coordenadas x e y. Cada uno de los vuelos aparecerá en el mapa en función de las coordenadas x e y, con puntos de color rojo y también con la imagen del icono del avión que hayamos escogido en la variable icon_url.

```Python
flight_df['rot_angle'] = flight_df['true_track'] * -1  # Rotation angle
icon_url = 'https://www.iconsdb.com/icons/preview/red/airplane-4-xl.png'
flight_df['url'] = icon_url
```

En el siguiente código se configura y genera el mapa final en el que aparecerán todos los datos representados en forma de aviones con una etiqueta con el nombre del vuelo. Además se ha añadido la función de mostrar información relevante sobre el vuelo al pasar el puntero del ratón por encima.

```Python
# FIGURE SETTING
x_range, y_range = ([xy_min[0], xy_max[0]], [xy_min[1], xy_max[1]])
p = figure(
    x_range=x_range,
    y_range=y_range,
    x_axis_type='mercator',
    y_axis_type='mercator',
    sizing_mode='scale_width',
    plot_height=300
)

# PLOT BASEMAP AND AIRPLANE POINTS
flight_source = ColumnDataSource(flight_df)
tile_prov = get_provider(STAMEN_TERRAIN)
p.add_tile(tile_prov, level='image')
p.image_url(
    url='url',
    x='x',
    y='y',
    source=flight_source,
    anchor='center',
    angle_units='deg',
    angle='rot_angle',
    h_units='screen',
    w_units='screen',
    w=40,
    h=40
)
p.circle(
    'x',
    'y',
    source=flight_source,
    fill_color='red',
    hover_color='yellow',
    size=10,
    fill_alpha=0.8,
    line_width=0
)

# HOVER INFORMATION AND LABEL
my_hover = HoverTool()
my_hover.tooltips = [
    ('Call sign', '@callsign'),
    ('Origin Country', '@origin_country'),
    ('velocity(m/s)', '@velocity'),
    ('Altitude(m)', '@baro_altitude')
]
labels = LabelSet(
    x='x',
    y='y',
    text='callsign',
    level='glyph',
    x_offset=5,
    y_offset=5,
    source=flight_source,
    render_mode='canvas',
    background_fill_color='white',
    text_font_size='8pt'
)
p.add_tools(my_hover)
p.add_layout(labels)

show(p)
```

Si se ejecuta obtendremos un resultado como el de la siguiente imagen.

<p align="center">
<img src="img/peninsula_iberica_app_vuelos.gif">
Figura 4. Mapa con los vuelos tal como se ve en el navegador.
</p>
<br>

## Crear aplicación de seguimiento de vuelos

Hasta ahora habíamos discutido acerca de obtener datos de tráfico aéreo y ubicar la aeronave en un mapa. En esta última sección, veremos cómo crear una aplicación de seguimiento de vuelos que se ejecuta en un navegador web. La aplicación recuperará automáticamente nuevos datos en un intervalo específico y trazará los datos en el mapa. En esta sección, combinaremos el código del paso anterior y lo empaquetamos en una aplicación usando la biblioteca Bokeh. El código completo se puede encontrar al final de este tutorial.
Si observamos el código, necesitamos importar una biblioteca Bokeh adicional, como Server , Application y FunctionHandler . El código de la aplicación comienza en la línea 50. Aquí creamos la función principal de la aplicación que se llama seguimiento_de_vuelo . La función principal consta de todos los procesos que se tomarán cuando se ejecute la función principal, como actualizar los datos de vuelo, volcarlos en el marco de datos de Pandas, convertirlos en fuente de columna de datos Bokeh y transmitir, recuperar la actualización cada 5 segundos y trazar los datos en el mapa. Después de crear la función principal de la aplicación, al final determinamos algunas variables o parámetros para el servidor. Puedes encontrar todo el proceso con las etiquetas de comentarios en el código.

```python
"""
FLIGHT TRACKING WITH PYTHON AND OPEN AIR TRAFFIC DATA
by ideagora geomatics | www.geodose.com | @ideageo
updated by @JavDomGom
"""

import json
import numpy as np
import pandas as pd
import requests

from bokeh.application import Application
from bokeh.application.handlers.function import FunctionHandler
from bokeh.models import HoverTool,LabelSet,ColumnDataSource
from bokeh.plotting import figure
from bokeh.server.server import Server
from bokeh.tile_providers import get_provider, STAMEN_TERRAIN


def wgs84_to_web_mercator_point(lon, lat):
    """ Function to convert GCS WGS84 to web marcator point. """
    
    k = 6378137  # WGS84 oblate spheroid with this equatorial radius in meters.
    x= lon * (k * np.pi / 180.0)
    y= np.log(np.tan((90 + lat) * np.pi / 360.0)) * k
    
    return x,y


def wgs84_to_web_mercator(df, lon='long', lat='lat'):
    """ Function to get Dataframe. """
    
    k = 6378137  # WGS84 oblate spheroid with this equatorial radius in meters.
    df['x'] = df[lon] * (k * np.pi / 180.0)
    df['y'] = np.log(np.tan((90 + df[lat]) * np.pi / 360.0)) * k
    
    return df


# Iberian Peninsula min and max longitude & latitude.
lon_min, lat_min = -9.9503, 35.9425
lon_max, lat_max = 4.5517, 43.9617

# COORDINATE CONVERSION
xy_min = wgs84_to_web_mercator_point(lon_min, lat_min)
xy_max = wgs84_to_web_mercator_point(lon_max, lat_max)

# COORDINATE RANGE IN WEB MERCATOR
x_range, y_range=([xy_min[0], xy_max[0]], [xy_min[1], xy_max[1]])

# REST API QUERY
user = 'john_doe'
password = 'abcd1234'
query = f'lamin={lat_min}&lomin={lon_min}&lamax={lat_max}&lomax={lon_max}'
url = f'https://{user}:{password}@opensky-network.org/api/states/all?{query}'


def flight_tracking(doc):
    """ Init bokeh column data source. """
    
    flight_source = ColumnDataSource(
        {
            'icao24': [],
            'callsign': [],
            'origin_country': [],
            'time_position': [],
            'last_contact': [],
            'long': [],
            'lat': [],
            'baro_altitude': [],
            'on_ground': [],
            'velocity': [],
            'true_track': [],
            'vertical_rate': [],
            'sensors': [],
            'geo_altitude': [],
            'squawk': [],
            'spi': [],
            'position_source': [],
            'x': [],
            'y':[],
            'rot_angle': [],
            'url': []
        }
    )
    
    def update():
        """ Updating flight data. """
        
        response = requests.get(url).json()
        
        # CONVERT TO PANDAS DATAFRAME
        col_name=[
            'icao24', 'callsign', 'origin_country', 'time_position', 'last_contact', 'long', 'lat',
            'baro_altitude', 'on_ground', 'velocity', 'true_track', 'vertical_rate', 'sensors', 'geo_altitude',
            'squawk', 'spi', 'position_source'
        ]
       
        flight_df = pd.DataFrame(response['states']) 
        flight_df = flight_df.loc[:, 0:16] 
        flight_df.columns = col_name
        wgs84_to_web_mercator(flight_df)
        flight_df = flight_df.fillna('No Data')
        flight_df['rot_angle'] = flight_df['true_track'] * -1
        icon_url = 'https://www.iconsdb.com/icons/preview/red/airplane-4-xl.png'
        # icon_url = 'http://icons.iconarchive.com/icons/custom-icon-design/flatastic-11/512/User-red-icon.png'
        flight_df['url'] = icon_url
        
        # CONVERT TO BOKEH DATASOURCE AND STREAMING
        n_roll = len(flight_df.index)
        flight_source.stream(flight_df.to_dict(orient='list'), n_roll)
        
    # CALLBACK UPATE IN AN INTERVAL
    doc.add_periodic_callback(update, 5000)  # 5000 ms/10000 ms for registered user.
    
    # PLOT AIRCRAFT POSITION
    p = figure(
        x_range=x_range,
        y_range=y_range,
        x_axis_type='mercator',
        y_axis_type='mercator',
        sizing_mode='scale_width',
        plot_height=300
    )
    tile_prov = get_provider(STAMEN_TERRAIN)
    
    p.add_tile(
        tile_prov,
        level='image'
    )
    p.image_url(
        url='url',
        x='x',
        y='y',
        source=flight_source,
        anchor='center',
        angle_units='deg',
        angle='rot_angle',
        h_units='screen',
        w_units='screen',
        w=40,
        h=40
    )
    p.circle(
        'x',
        'y',
        source=flight_source,
        fill_color='red',
        hover_color='yellow',
        size=10,
        fill_alpha=0.8,
        line_width=0
    )

    # ADD HOVER TOOL AND LABEL
    my_hover=HoverTool()
    my_hover.tooltips=[
        ('Call sign', '@callsign'),
        ('Origin Country', '@origin_country'),
        ('velocity(m/s)', '@velocity'),
        ('Altitude(m)', '@baro_altitude')
    ]
    labels = LabelSet(
        x='x',
        y='y',
        text='callsign',
        level='glyph',
        x_offset=5,
        y_offset=5,
        source=flight_source,
        render_mode='canvas',
        background_fill_color='white',
        text_font_size='8pt'
    )
    p.add_tools(my_hover)
    p.add_layout(labels)
    doc.title='REAL TIME FLIGHT TRACKING'
    doc.add_root(p)
    
# SERVER CODE
apps = {'/': Application(FunctionHandler(flight_tracking))}
server = Server(apps, port=8084)  # Define an unused port.
server.start()
```

Ahora es el momento de probar la aplicación. Ejecutemos el código y abramos un navegador web. Escribimos localhost:puerto (por ejemplo, localhost:8084). Deberíamos ver la aplicación de seguimiento de vuelos ejecutándose en el navegador web.