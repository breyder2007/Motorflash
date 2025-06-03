import os
import re
import requests
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeoutError
import time
import random
import csv
import json
import zipfile
from PIL import Image

# --- Configuración Global ---
MOTORFLASH_BASE_URL = "https://www.motorflash.com/concesionario/grupo-cobendai/coches-segunda-mano/204237/"


def extraer_info_publicaciones_motorflash(url_limit=None):
    """
    Extrae información de publicaciones de coches de la página de Motorflash.

    Args:
        url_limit (int, optional): Límite de URLs de vehículos a extraer.
                                   Si es None, extrae todas las disponibles.

    Returns:
        list: Una lista de diccionarios, donde cada diccionario contiene
              información de un coche (URL, título, imágenes, precio, detalles).
    """
    print(f"Iniciando la extracción de información de publicaciones de: {MOTORFLASH_BASE_URL}")
    car_data_list = []
    extracted_urls = set()  # Keep track of extracted URLs to avoid duplicates

    with sync_playwright() as p:
        # Launch a Chromium browser in headless mode (set to False for visual debugging)
        browser = p.chromium.launch(headless=False)
        # Create a new page with a common user agent and viewport size
        page = browser.new_page(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36")
        page.set_viewport_size({"width": 1366, "height": 768})

        try:
            # Navigate to the base URL and wait until network is idle (all resources loaded)
            page.goto(MOTORFLASH_BASE_URL, wait_until="networkidle", timeout=90000)
            # Introduce a random delay to mimic human behavior and avoid being blocked
            time.sleep(random.uniform(5, 10))

            # Selector for individual vehicle cards on the listing page
            vehicle_card_selector = 'div.item-listado'
            # Wait for at least one vehicle card to be attached to the DOM
            page.wait_for_selector(vehicle_card_selector, state='attached', timeout=90000)

            page_num = 1
            while True:
                # Get all vehicle card locators on the current page
                all_vehicle_locators = page.locator(vehicle_card_selector)
                num_locators = all_vehicle_locators.count()
                print(f"Processing page {page_num}: Found {num_locators} vehicle cards.")

                for i in range(num_locators):
                    card_locator = all_vehicle_locators.nth(i)
                    car_info = {}

                    # Extraer URL del vehículo
                    link_locator = card_locator.locator('p.h2-style a')
                    href = link_locator.get_attribute('href') if link_locator.count() > 0 else None
                    if href:
                        full_url = requests.compat.urljoin(MOTORFLASH_BASE_URL, href)
                        # Add URL to extracted_urls set to prevent processing duplicates
                        if full_url in extracted_urls:
                            continue  # Skip if already processed
                        car_info['url'] = full_url
                        extracted_urls.add(full_url)
                    else:
                        continue  # Skip if no URL found for the card

                    # Extraer título del vehículo
                    title_locator = card_locator.locator('p.h2-style a')
                    car_info['titulo'] = title_locator.text_content().strip() if title_locator.count() > 0 else 'N/A'

                    # Extraer precio del vehículo - REVISED LOGIC
                    price_text = 'N/A'
                    # 1. Try to find a span with class 'price' first (original selector)
                    price_locator = card_locator.locator('span.price')
                    if price_locator.count() > 0:
                        price_text = price_locator.text_content().strip()

                    # 2. If not found or still 'N/A', try to find within a common 'price-box' structure
                    if price_text == 'N/A' or not re.search(r'\d',
                                                            price_text):  # Check if price_text actually contains digits
                        price_box_locator = card_locator.locator('div.price-box span.price')
                        if price_box_locator.count() > 0:
                            price_text = price_box_locator.text_content().strip()

                    # 3. If still not found, try a more general search for price-like text within any span or p tag
                    if price_text == 'N/A' or not re.search(r'\d', price_text):
                        all_text_elements = card_locator.locator('span, p').all()
                        for elem in all_text_elements:
                            text_content = elem.text_content().strip()
                            # Regex to find numbers possibly with dots/commas and a Euro symbol (€)
                            # This pattern looks for one or more digits, optionally followed by groups of three digits
                            # separated by dots, and optionally followed by a comma and more digits, then a space and '€'.
                            if re.search(r'\d{1,3}(?:\.\d{3})*(?:,\d+)?\s*€', text_content):
                                price_text = text_content
                                break
                    car_info['precio'] = price_text

                    # Extraer otros detalles del vehículo - REVISED LOGIC
                    # Combine text from 'ul.general' and 'ul.extras' as seen in the provided HTML snippet
                    general_details_locator = card_locator.locator('ul.general')
                    general_details = general_details_locator.text_content().strip() if general_details_locator.count() > 0 else ''

                    extras_details_locator = card_locator.locator('ul.extras')
                    extras_details = extras_details_locator.text_content().strip() if extras_details_locator.count() > 0 else ''

                    # Concatenate and clean up the details
                    combined_details = f"{general_details} {extras_details}".strip()
                    car_info['detalles'] = combined_details if combined_details else 'N/A'

                    # Extraer imágenes
                    # Look for img tags within the card and extract data-src, src, or srcset
                    image_locators = card_locator.locator('img').all()
                    car_info['imagenes'] = [
                        img.get_attribute('data-src') or img.get_attribute('src') or
                        (img.get_attribute('srcset').split(' ')[0] if img.get_attribute('srcset') else None)
                        for img in image_locators
                        if img.get_attribute('data-src') or img.get_attribute('src') or img.get_attribute('srcset')
                    ]

                    car_data_list.append(car_info)

                    # Check if the URL limit has been reached
                    if url_limit is not None and len(car_data_list) >= url_limit:
                        print(f"Reached URL limit of {url_limit}. Stopping extraction.")
                        break

                if url_limit is not None and len(car_data_list) >= url_limit:
                    break

                # Try to click the "Next" button to go to the next page
                try:
                    next_button_locator = page.locator('a.nxtpage')
                    if next_button_locator.is_enabled():
                        print(f"Navigating to next page...")
                        next_button_locator.click()
                        page_num += 1
                        # Wait for the next page to load
                        page.wait_for_load_state('networkidle', timeout=30000)
                        time.sleep(random.uniform(3, 7))  # Small delay after page load
                    else:
                        print("No more 'Next' button or it's disabled. Ending pagination.")
                        break  # No more pages
                except PlaywrightTimeoutError:
                    print("Timeout waiting for next page load. Ending pagination.")
                    break
                except Exception as e:
                    print(f"An error occurred while trying to paginate: {e}. Ending pagination.")
                    break

        finally:
            # Always close the browser when done
            browser.close()
            print("Browser closed.")

    return car_data_list


def guardar_csv_y_json(car_data_list):
    """
    Guarda la información de los coches en archivos CSV y JSON.

    Args:
        car_data_list (list): Lista de diccionarios con la información de los coches.
    """
    csv_filename = "coches_motorflash.csv"
    try:
        with open(csv_filename, 'w', encoding='utf-8', newline='') as csvfile:
            csvwriter = csv.writer(csvfile)
            # Encabezados del CSV
            csvwriter.writerow(['MARCA', 'MODELO', 'PRECIO', 'DETALLES', 'LINK'])

            for car in car_data_list:
                titulo = car.get('titulo', 'N/A')
                url = car.get('url', 'N/A')
                precio = car.get('precio', 'N/A')
                detalles = car.get('detalles', 'N/A')

                # Separar marca y modelo del título
                # Assuming the first word is the brand and the next is part of the model
                marca, modelo = 'N/A', 'N/A'
                if ' ' in titulo:
                    parts = titulo.split(' ', 1)
                    marca = parts[0].strip()
                    if len(parts) > 1:
                        # Take the first word after the brand as the model, or the whole rest if only one word
                        model_parts = parts[1].strip().split(' ', 1)
                        modelo = model_parts[0]
                else:
                    marca = titulo  # If only one word, consider it both brand and model (or just brand)

                csvwriter.writerow([marca, modelo, precio, detalles, url])

        print(f"Información de vehículos guardada exitosamente en '{csv_filename}'")
    except IOError as e:
        print(f"Error al guardar la información en el archivo CSV: {e}")

    json_filename = "coches_motorflash.json"
    try:
        with open(json_filename, 'w', encoding='utf-8') as jsonfile:
            json.dump(car_data_list, jsonfile, ensure_ascii=False, indent=4)
        print(f"Información de vehículos guardada exitosamente en '{json_filename}'")
    except IOError as e:
        print(f"Error al guardar la información en el archivo JSON: {e}")


def descargar_y_convertir_imagen(img_url, output_dir, nombre_archivo):
    """
    Descarga una imagen de una URL y la convierte a formato JPG.

    Args:
        img_url (str): La URL de la imagen a descargar.
        output_dir (str): Directorio donde se guardará la imagen.
        nombre_archivo (str): Nombre base del archivo (sin extensión).
    """
    try:
        print(f"Descargando imagen: {img_url}")
        # Add a timeout to the request to prevent indefinite waiting
        response = requests.get(img_url, stream=True, timeout=10)
        if response.status_code == 200:
            # Save the image to a temporary file (e.g., .temp) in its original format
            temp_path = os.path.join(output_dir, f"{nombre_archivo}.temp")
            with open(temp_path, 'wb') as temp_file:
                for chunk in response.iter_content(1024):  # Stream content in chunks
                    temp_file.write(chunk)

            # Open the downloaded image and convert it to JPG
            jpg_path = os.path.join(output_dir, f"{nombre_archivo}.jpg")
            try:
                with Image.open(temp_path) as img:
                    # Convert to RGB to ensure compatibility before saving as JPEG
                    img.convert("RGB").save(jpg_path, "JPEG")
                print(f"Imagen convertida y guardada en: {jpg_path}")
            except Exception as img_err:
                print(f"Error processing image with PIL (might be corrupted or unsupported format): {img_err}")
                print(f"Attempting to save original image as is: {jpg_path}")
                # If PIL fails, try to just copy the temp file to a .jpg extension
                os.rename(temp_path, jpg_path)  # This might not be a valid JPG, but keeps the file
            finally:
                if os.path.exists(temp_path):
                    os.remove(temp_path)  # Clean up the temporary file
        else:
            print(f"Error al descargar la imagen (Código: {response.status_code}): {img_url}")
    except requests.exceptions.RequestException as e:
        print(f"Error de red al descargar la imagen {img_url}: {e}")
    except Exception as e:
        print(f"Error general al procesar la imagen {img_url}: {e}")


def descargar_imagenes_y_crear_zip(car_data_list):
    """
    Descarga todas las imágenes de la lista de coches, las organiza en directorios
    temporales y luego las comprime en un archivo ZIP.

    Args:
        car_data_list (list): Lista de diccionarios con la información de los coches.
    """
    zip_filename = "imagenes_coches.zip"
    temp_dir = "temp_imagenes"

    # Create the temporary directory if it doesn't exist
    if not os.path.exists(temp_dir):
        os.makedirs(temp_dir)
    print(f"Imágenes temporales se guardarán en: {os.path.abspath(temp_dir)}")

    try:
        for car in car_data_list:
            # Sanitize the title to use it as a directory name (remove invalid characters)
            titulo = car.get('titulo', 'N/A')
            # Replace characters that are invalid in file paths
            sanitized_title = re.sub(r'[\\/:*?"<>|]', '_', titulo)
            # Limit length to avoid issues with long path names
            sanitized_title = sanitized_title[:100].strip() if sanitized_title else 'untitled_car'

            imagenes = car.get('imagenes', [])
            car_dir = os.path.join(temp_dir, sanitized_title)

            if not os.path.exists(car_dir):
                os.makedirs(car_dir)

            # Keep track of downloaded images for this car to avoid duplicates within one car's folder
            downloaded_car_images = set()

            print(f"Procesando imágenes para: {titulo}")
            for idx, img_url in enumerate(imagenes):
                if img_url and img_url not in downloaded_car_images:
                    descargar_y_convertir_imagen(img_url, car_dir, f"imagen_{idx + 1}")
                    downloaded_car_images.add(img_url)
                elif not img_url:
                    print(f"URL de imagen inválida o vacía encontrada para {titulo}.")
                else:
                    print(f"Skipping duplicate image URL for {titulo}: {img_url}")

        # Create the ZIP file after all images have been downloaded
        print(f"\nCreando archivo ZIP: {zip_filename}...")
        with zipfile.ZipFile(zip_filename, 'w', zipfile.ZIP_DEFLATED) as zipf:
            for root, dirs, files in os.walk(temp_dir):
                for file in files:
                    file_path = os.path.join(root, file)
                    # arcname is the path within the ZIP file
                    arcname = os.path.relpath(file_path, temp_dir)
                    zipf.write(file_path, arcname)
        print(f"Archivo ZIP '{zip_filename}' creado exitosamente en {os.path.abspath(zip_filename)}.")

    except Exception as e:
        print(f"Error al procesar las imágenes o crear el archivo ZIP: {e}")
    finally:
        # Clean up the temporary directory after creating the ZIP
        print(f"Limpiando directorio temporal: {temp_dir}...")
        if os.path.exists(temp_dir):
            for root, dirs, files in os.walk(temp_dir, topdown=False):
                for file in files:
                    try:
                        os.remove(os.path.join(root, file))
                    except OSError as e:
                        print(f"Error removing file {os.path.join(root, file)}: {e}")
                for dir_name in dirs:
                    try:
                        os.rmdir(os.path.join(root, dir_name))
                    except OSError as e:
                        print(f"Error removing directory {os.path.join(root, dir_name)}: {e}")
            try:
                os.rmdir(temp_dir)
            except OSError as e:
                print(f"Error removing main temporary directory {temp_dir}: {e}")
        print("Directorio temporal limpiado.")


def main():
    """
    Función principal para ejecutar el proceso de extracción, guardado y descarga.
    """
    # You can set a URL limit for quick testing, or None to extract everything
    url_limit_to_extract = None  # Set a number (e.g., 10) to limit extraction

    all_car_details = extraer_info_publicaciones_motorflash(url_limit=url_limit_to_extract)

    if all_car_details:
        print(f"\n--- Extracción finalizada. Total de vehículos encontrados: {len(all_car_details)} ---")
        guardar_csv_y_json(all_car_details)
        descargar_imagenes_y_crear_zip(all_car_details)
    else:
        print("No se encontró información de vehículos para guardar.")


if __name__ == "__main__":
    main()
