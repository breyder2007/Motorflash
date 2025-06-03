Scraper de Coches de Segunda Mano - Motorflash
Este repositorio contiene un script Python diseñado para extraer información detallada de coches de segunda mano de concesionarios específicos en la plataforma Motorflash. El programa recopila datos como URL, título, precio, detalles y enlaces a imágenes, para luego organizar esta información en formatos CSV y JSON, y descargar las imágenes en un archivo ZIP.

🚀 Características Principales
Extracción de Datos Completa: Recopila URL, título, precio, detalles y enlaces de imágenes de cada vehículo.

Manejo de Paginación: Navega automáticamente a través de múltiples páginas de listados de coches.

Extracción Robusta de Precios: Incluye lógica avanzada para identificar y extraer precios, incluso si su ubicación en el HTML varía ligeramente.

Guardado de Datos Versátil: Exporta la información extraída a archivos .csv (para hojas de cálculo) y .json (para uso programático).

Descarga y Organización de Imágenes: Descarga todas las imágenes de los vehículos y las organiza en carpetas individuales por coche, comprimiéndolas finalmente en un archivo .zip.

Modo de Ejecución Visible/Invisible (Headless): Permite ver el navegador en acción durante el scraping para depuración (headless=False) o ejecutarlo en segundo plano (headless=True).

🛠️ Requisitos Previos
Antes de ejecutar el script, asegúrate de tener instalado lo siguiente en tu sistema:

Python 3.x: Puedes descargarlo desde python.org. Durante la instalación en Windows, asegúrate de marcar la opción "Add Python to PATH".

📦 Instalación de Dependencias
El script requiere varias librerías de Python. Puedes instalarlas fácilmente usando pip.

Abre tu terminal o línea de comandos.

Navega hasta el directorio donde has guardado el archivo del script (por ejemplo, cd C:\Users\TuUsuario\MisProyectos\ScraperMotorflash).

Ejecuta los siguientes comandos para instalar las librerías necesarias:

pip install beautifulsoup4
pip install playwright
pip install requests
pip install Pillow

Importante: Playwright necesita descargar los navegadores que utilizará. Ejecuta este comando después de instalar playwright:

playwright install

⚙️ Configuración
El script está diseñado para ser flexible con los enlaces de Motorflash.

URL Base del Concesionario:
Abre el archivo de tu script (.py) en un editor de texto. Al principio del archivo, encontrarás la variable MOTORFLASH_BASE_URL:

MOTORFLASH_BASE_URL = "https://www.motorflash.com/concesionario/grupo-cobendai/coches-segunda-mano/204237/"

Para usarlo con otro concesionario de Motorflash: Simplemente cambia esta URL por la URL del listado de coches de segunda mano del concesionario que te interese en Motorflash. Por ejemplo:

MOTORFLASH_BASE_URL = "https://www.motorflash.com/concesionario/deysa-ford-ribera-del-loira/coches-segunda-mano/200293/"

Nota: Este script está optimizado para la estructura HTML de motorflash.com. Si intentas usarlo con un dominio completamente diferente, es probable que necesites ajustar los selectores CSS dentro del código.

Modo Headless (Opcional):
Si deseas que el navegador se ejecute en segundo plano (sin abrir una ventana visible), cambia headless=False a headless=True en la línea browser = p.chromium.launch(headless=False).

🚀 Cómo Ejecutar el Programa
Una vez que hayas configurado la URL base y tengas todas las dependencias instaladas, puedes ejecutar el script:

Abre tu terminal o línea de comandos.

Navega hasta el directorio donde guardaste tu script Python (el mismo lugar donde instalaste las dependencias).

Ejecuta el script usando el siguiente comando:

python tu_script_principal.py

(Reemplaza tu_script_principal.py con el nombre real de tu archivo, por ejemplo, scraper_coches.py).

Límite de URLs (Opcional):
Si quieres probar el script con un número limitado de coches (útil para pruebas rápidas), puedes modificar la variable url_limit_to_extract en la función main() de tu script. Por ejemplo, para extraer solo 10 coches:

# En la función main():
url_limit_to_extract = 10  # Establece un número (ej. 10) para limitar la extracción

Si la dejas como None, el script intentará extraer todos los coches disponibles en la paginación.

📊 Output Esperado
Al finalizar la ejecución del script, se generarán los siguientes archivos en el mismo directorio donde se ejecutó:

coches_motorflash.csv: Un archivo CSV que contiene la marca, modelo, precio, detalles y el enlace (URL) de cada coche. Puedes abrirlo con programas de hoja de cálculo como Microsoft Excel, Google Sheets o LibreOffice Calc.

coches_motorflash.json: Un archivo JSON con la misma información, pero en un formato estructurado, ideal para ser procesado por otros programas o APIs.

imagenes_coches.zip: Un archivo comprimido que contiene todas las imágenes de los coches descargadas. Las imágenes se organizan en subcarpetas dentro del ZIP, con cada subcarpeta nombrada según el título del coche.

Durante la ejecución, verás mensajes en la terminal que te informarán sobre el progreso (ej. "Iniciando la extracción...", "Processing page...", "Descargando imagen..."). Si headless=False, también observarás una ventana de navegador abriéndose y navegando por la web.

🔗 Contexto de Uso: Salón del Vehículo de Ocasión de IFEMA
Este código fue utilizado específicamente para extraer información de vehículos y marcas del "Salón del Vehículo de Ocasión de IFEMA", y los datos resultantes se integraron en la siguiente hoja de cálculo de Google Sheets:

Link a la Hoja de Cálculo: Salón del Vehículo de Ocasión de IFEMA - Google Sheets

Esto demuestra la capacidad del script para recopilar datos de listados de vehículos y su utilidad en la gestión y análisis de información de grandes eventos o concesionarios.

⚠️ Notas Importantes
Velocidad de Extracción: El script incluye pausas aleatorias (time.sleep) para simular el comportamiento humano y evitar ser bloqueado por el sitio web. No modifiques estas pausas a valores muy bajos, ya que podrías ser detectado como un bot.

Cambios en la Web: Las estructuras de las páginas web pueden cambiar con el tiempo. Si el script deja de funcionar, es probable que los selectores CSS (div.item-listado, span.price, etc.) necesiten ser actualizados para coincidir con la nueva estructura de la página de Motorflash.

Errores de Red/Imágenes: El script incluye manejo de errores básico para descargas de imágenes y problemas de red. Si una imagen no se descarga, se imprimirá un mensaje de error en la consola.
