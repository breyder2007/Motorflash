Explicación del código  

Antes de ejecutar el código, necesitas tener instalado estas librerías:
 pip install beautifulsoup4
pip install playwright
pip install requests
 pip install Pillow
playwright install   

 MOTORFLASH_BASE_URL = "https://www.motorflash.com/concesionario/grupo-cobendai/coches-segunda-mano/204237/"
MOTORFLASH_BASE_URL: Esta es la dirección web (URL) de donde el script empezará a extraer la información. Este script sirve para cualquier link que sea para motorflash. 

with sync_playwright() as p: Esto inicia Playwright. Piensa en ello como si abrieras un navegador web invisible (o visible, si headless=False).

browser = p.chromium.launch(headless=False): Lanza el navegador Chrome.

headless=False: Esto es importante para entender. Si lo pones en True, el navegador se ejecutará en segundo plano, sin que lo veas. Si lo pones en False (como está ahora), verás una ventana de Chrome abriéndose y navegando por la página. Esto es genial para depurar y ver qué está haciendo el script.
    
wait_until="networkidle": Espera hasta que la red esté "inactiva", lo que significa que la página y sus recursos (imágenes, scripts) se han cargado. Esto es crucial para evitar errores.
time.sleep(random.uniform(5, 10)): Hace que el script espere un tiempo aleatorio entre 5 y 10 segundos. Esto es para parecerse más a un humano navegando y evitar que la página nos bloquee por hacer peticiones demasiado rápido.
all_vehicle_locators = page.locator(vehicle_card_selector): Encuentra todas las tarjetas de coche en la página actual.

URL:
 card_locator.locator('p.h2-style a').get_attribute('href') busca el enlace dentro del título.

Título:
 title_locator.text_content().strip() obtiene el texto del título.

Precio:
Primero, intenta encontrar un <span> con la clase price.
Si no lo encuentra, o el texto no parece un precio, busca un <span> con la clase price dentro de un <div> con la clase price-box.
Si aún no encuentra el precio, usa una "expresión regular" (re.search(r'\d{1,3}(?:\.\d{3})*(?:,\d+)?\s*€', text_content)) para buscar cualquier texto que se parezca a un número con puntos, comas y el símbolo "€" en cualquier <span> o <p> dentro de la tarjeta del coche. Esto lo hace muy robusto.

Descripción:
Busca los detalles en las listas (<ul>) con las clases general y extras

Navegación a la Siguiente Página (a.nxtpage):
next_button_locator = page.locator('a.nxtpage'): Busca el botón de "Siguiente página".
if next_button_locator.is_enabled(): Comprueba si el botón está activo.
next_button_locator.click(): Hace clic en el botón.
page.wait_for_load_state('networkidle', ...): Espera a que la nueva página cargue.
Función para Guardar Datos
Esta función toma la lista de diccionarios con la información de los coches y la guarda en dos formatos:
CSV (.csv): Un archivo de texto plano donde los datos están separados por comas. Ideal para abrir en Excel o programas de hojas de cálculo.
csvwriter.writerow(['MARCA', 'MODELO', 'PRECIO', 'DETALLES', 'LINK']): Define los encabezados de las columnas.
marca, modelo = 'N/A', 'N/A': Intenta separar la marca y el modelo del título del coche. Esto es una suposición basada en cómo se suelen estructurar los títulos. Podría necesitar ajustes si los títulos en otra web son muy diferentes.
JSON (.json): Un formato de datos estructurado, muy común en programación. Es fácil de leer para las máquinas y también para los humanos (aunque menos que un CSV para grandes tablas).
json.dump(car_data_list, jsonfile, ensure_ascii=False, indent=4): Guarda la lista de diccionarios como JSON, con formato legible (indent=4).

Funciones para Descargar y Comprimir Imágenes

Esta función descarga una imagen de una URL y la convierte a JPG.
requests.get(img_url, stream=True, timeout=10): Descarga la imagen. timeout=10 es importante para que no se quede esperando indefinidamente si una imagen no carga.
Image.open(temp_path) as img: img.convert("RGB").save(jpg_path, "JPEG"): Usa la librería Pillow para abrir la imagen descargada (que podría estar en formato WEBP, PNG, etc.) y la guarda como JPG.
Maneja errores de red y de procesamiento de imágenes.
Esta función orquesta la descarga de todas las imágenes y la creación del ZIP.
temp_dir = "temp_imagenes": Crea una carpeta temporal para guardar las imágenes antes de comprimirlas.
sanitized_title = re.sub(r'[\\/:*?"<>|]', '_', titulo): Limpia el título del coche para que pueda usarse como nombre de carpeta (elimina caracteres que no están permitidos en los nombres de archivos).
Para cada coche, crea una subcarpeta dentro de temp_imagenes con el nombre del coche.
Llama a descargar_y_convertir_imagen para cada imagen del coche.
with zipfile.ZipFile(zip_filename, 'w', zipfile.ZIP_DEFLATED) as zipf:: Crea el archivo ZIP y añade todas las imágenes.
finally:: Esta sección es crucial. Después de crear el ZIP, limpia la carpeta temporal (temp_imagenes) para no dejar archivos basura en tu sistema.


Qué Debes Hacer para que Funcione con cualquier link de motorflash

Cambia la URL Base:
Modifica la variable MOTORFLASH_BASE_URL al principio del script con la URL de la página que quieres scrapear.
MOTORFLASH_BASE_URL = "https://www.la-nueva-web-de-coches.com/listado/" 


puede servir también 

MOTORFLASH_BASE_URL = "https://www.motorflash.com/concesionario/deysa-ford-ribera-del-loira/coches-segunda-mano/200293/"  


Donde utilice este código fue en el excel de “Salón del Vehículo de Ocasión de IFEMA” para sacar cada coche y marca

Link:https://docs.google.com/spreadsheets/d/1LqS0QWpIj7C_XvmhVhoeuU_anclY1upAe6o7JwMGJGE/edit?usp=sharing
