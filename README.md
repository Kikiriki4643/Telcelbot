import os
import re
import telebot
from playwright.sync_api import sync_playwright

# Inicializa tu bot con el token de entorno seguro
TOKEN = os.environ.get('8556063338:AAFzycqinzWiL6UAW610kWOukMm0z-7fkoE')
bot = telebot.TeleBot(TOKEN)

def scraping_telcel(telefono, password):
    """Navega de forma invisible en Mi Telcel para extraer saldo y descargar el PDF."""
    with sync_playwright() as p:
        # Lanzamiento optimizado para servidores en la nube como Render
        browser = p.chromium.launch(
            headless=True,
            args=[
                "--no-sandbox", 
                "--disable-setuid-sandbox", 
                "--disable-blink-features=AutomationControlled"  # Oculta el rastro de automatización
            ]
        )
        context = browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
            viewport={"width": 1280, "height": 720}
        )
        page = context.new_page()
        
        try:
            # 1. Acceso al inicio de sesión de Mi Telcel
            page.goto("https://mitelcel.com", timeout=45000)
            
            # Espera a que los campos de texto estén disponibles
            page.wait_for_selector("input[name='username']", timeout=15000)
            page.fill("input[name='username']", telefono)
            page.fill("input[name='password']", password)
            
            # Clic en el botón principal para iniciar sesión
            page.click("button[type='submit']")
            
            # 2. Esperar a que cargue el Dashboard Principal del usuario
            page.wait_for_url("**/mitelcel/dashboard**", timeout=30000)
            page.wait_for_load_state("networkidle")
            
            # Extraer Saldo usando un selector inteligente que busque el símbolo "\$"
            saldo_elemento = page.locator("div:has-text('\$')").last
            saldo_texto = saldo_elemento.inner_text().strip()
            saldo_limpio = re.sub(r'\s+', ' ', saldo_texto) # Limpia espacios o saltos de línea basura
            
            # 3. Navegación directa a la sección del Estado de Cuenta
            page.goto("https://mitelcel.com", timeout=30000)
            page.wait_for_load_state("networkidle")
            
            # Ruta temporal en el servidor para almacenar el PDF mientras se envía a Telegram
            path_pdf = f"/tmp/estado_cuenta_{telefono}.pdf"
            
            # Intercepta el inicio de la descarga al hacer clic en el botón de PDF
            with page.expect_download(timeout=20000) as download_info:
                # Encuentra el botón de descarga basándose en su texto visible
                page.locator("button:has-text('Descargar'), a:has-text('Descargar'), button:has-text('PDF')").first.click()
            
            download = download_info.value
            download.save_as(path_pdf)
            
            browser.close()
            return {"status": "success", "saldo": saldo_limpio, "pdf": path_pdf}
            
        except Exception as e:
            browser.close()
            error_msg = str(e)
            if "timeout" in error_msg.lower():
                error_msg = "Tiempo de espera agotado. Verifica tu señal o que tu contraseña de Mi Telcel sea correcta."
            return {"status": "error", "message": error_msg}

# Controlador del comando de bienvenida
@bot.message_handler(commands=['start', 'ayuda'])
def send_welcome(message):
    texto_ayuda = (
        "👋 *¡Bienvenido al Consultor Telcel Gratuito!*\n\n"
        "Para obtener tu saldo actual y tu estado de cuenta oficial en PDF, envíame un mensaje usando el siguiente formato:\n\n"
        "`/consultar TU_TELEFONO TU_CONTRASENA`\n\n"
        "💡 _Ejemplo: `/consultar 5512345678 MiPass123`_\n\n"
        "🔒 *Seguridad:* Tus credenciales se procesan en memoria bajo demanda y nunca se guardan en ninguna base de datos."
    )
    bot.reply_to(message, texto_ayuda, parse_mode="Markdown")

# Controlador principal de consulta
@bot.message_handler(commands=['consultar'])
def consultar_telcel(message):
    try:
        datos = message.text.split()
        if len(datos) < 3:
            bot.reply_to(message, "❌ *Formato incorrecto.*\nUsa: `/consultar TU_NUMERO TU_CONTRASENA`", parse_mode="Markdown")
            return
            
        telefono = datos[1]
        password = datos[2]
        
        # Validar longitud de número celular en México
        if not telefono.isdigit() or len(telefono) != 10:
            bot.reply_to(message, "❌ El número de teléfono debe constar exactamente de 10 dígitos numéricos.")
            return
            
        msg_espera = bot.reply_to(message, "🤖 Iniciando navegador seguro y conectando con Mi Telcel... (Esto puede tomar entre 30 y 45 segundos)")
        
        # Ejecución del Web Scraper gratuito
        resultado = scraping_telcel(telefono, password)
        
        if resultado["status"] == "success":
            bot.edit_message_text("✅ ¡Datos obtenidos con éxito! Enviando archivos...", message.chat.id, msg_espera.message_id)
            
            # Enviar Saldo en un formato estético
            bot.send_message(message.chat.id, f"💰 *Resumen de tu Línea*\n\n📱 *Número:* `{telefono}`\n💵 *Saldo Disponible:* {resultado['saldo']}", parse_mode="Markdown")
            
            # Enviar el archivo de Estado de Cuenta PDF adjunto
            with open(resultado["pdf"], 'rb') as pdf:
                bot.send_document(
                    message.chat.id, 
                    pdf, 
                    caption=f"📄 Estado de cuenta oficial de la línea {telefono}."
                )
                
            # Borrado inmediato del archivo del servidor por políticas de privacidad del usuario
            if os.path.exists(resultado["pdf"]):
                os.remove(resultado["pdf"])
        else:
            bot.edit_message_text(f"❌ *Error al consultar:* \n{resultado['message']}", message.chat.id, msg_espera.message_id, parse_mode="Markdown")
            
    except Exception as e:
        bot.reply_to(message, f"💥 Ocurrió un error inesperado en el sistema del bot: {str(e)}")

if __name__ == "__main__":
    print("🤖 El bot gratuito de Telegram está encendido y escuchando peticiones...")
    bot.infinity_polling()
