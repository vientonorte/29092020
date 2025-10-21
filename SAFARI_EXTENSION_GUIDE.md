# Safari ChatGPT Extension Setup and Deployment Guide

Esta guía profundiza en el proceso completo para crear, probar y distribuir una extensión de Safari que integre la API de OpenAI ChatGPT. Incluye comandos, fragmentos de código y recomendaciones prácticas para que puedas partir de un proyecto funcional.

## 1. Configura el entorno
1. En macOS, instala la versión más reciente de **Xcode** desde la App Store.
2. Asegúrate de tener instaladas las **Command Line Tools** (`xcode-select --install`).
3. Activa una cuenta de **Apple Developer** (personal o de equipo) para acceder a certificados de firma y herramientas de distribución.
4. Verifica que tienes acceso a la API de OpenAI y genera una **API key** desde <https://platform.openai.com/account/api-keys>.

## 2. Genera el proyecto base
1. Abre Xcode y crea un nuevo proyecto con la plantilla **"Safari Web Extension App"**.
2. Asigna un identificador de equipo y elige un nombre de organización adecuado.
3. El proyecto generado incluye una **app contenedora** y la carpeta `Resources` con `manifest.json`, scripts de contenido y la UI de popup.
4. Desde la barra lateral de Xcode, haz clic derecho en la carpeta **Resources → Add Files to…** para añadir imágenes o scripts adicionales que necesites.
5. Ejecuta la app contenedora una vez para generar el esquema base de la extensión dentro de `~/Library/Containers/<bundle-id>/Data/Library/Application Support/`.

## 3. Declara permisos y endpoints
1. En `Resources/manifest.json`, añade permisos de host para `https://api.openai.com/*` y los sitios donde se inyectará el panel.
2. Configura los elementos `background`, `content_scripts` y `action` para definir el flujo de la extensión y la experiencia de usuario.
3. Si planeas usar almacenamiento persistente, añade `"permissions": ["storage"]`.

Ejemplo mínimo de `manifest.json` (manifest version 3):

```json
{
  "manifest_version": 3,
  "name": "Safari ChatGPT Helper",
  "version": "1.0.0",
  "description": "Consulta a ChatGPT desde Safari.",
  "permissions": ["storage"],
  "host_permissions": [
    "https://api.openai.com/*",
    "https://*.tusitio.com/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png"
    }
  },
  "content_scripts": [
    {
      "matches": ["https://*.tusitio.com/*"],
      "js": ["content.js"],
      "run_at": "document_idle"
    }
  ]
}
```

## 4. Implementa la lógica de ChatGPT
1. En `background.js` o `service_worker.js`, gestiona el estado global y las peticiones a la API de OpenAI.
2. Implementa `browser.runtime.sendMessage` y `browser.runtime.onMessage` para la comunicación entre scripts de fondo, scripts de contenido y la UI.
3. En `popup.js` y `content.js`, crea la interfaz de usuario (popup o panel flotante) para mostrar respuestas de ChatGPT y recopilar entradas del usuario.
4. Añade manejo de errores y retroalimentación visual (p. ej. estados de cargando y fallos de red).

Fragmentos de ejemplo:

**background.js**

```javascript
const API_URL = "https://api.openai.com/v1/chat/completions";

async function callChatGPT(apiKey, conversation) {
  const response = await fetch(API_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      model: "gpt-3.5-turbo",
      messages: conversation,
      temperature: 0.2
    })
  });

  if (!response.ok) {
    throw new Error(`OpenAI error: ${response.status}`);
  }

  const data = await response.json();
  return data.choices?.[0]?.message?.content ?? "Sin respuesta";
}

browser.runtime.onMessage.addListener(async (request) => {
  if (request.type !== "chatgpt:prompt") return;

  try {
    const { apiKey } = await browser.storage.local.get("apiKey");
    if (!apiKey) {
      return { error: "Configura tu API key en las opciones." };
    }

    const reply = await callChatGPT(apiKey, request.messages);
    return { success: true, reply };
  } catch (error) {
    console.error(error);
    return { error: error.message };
  }
});
```

**popup.js**

```javascript
const form = document.querySelector("form");
const output = document.querySelector("#output");

form.addEventListener("submit", async (event) => {
  event.preventDefault();
  const userPrompt = event.target.elements.prompt.value;

  output.textContent = "Consultando…";

  const { success, reply, error } = await browser.runtime.sendMessage({
    type: "chatgpt:prompt",
    messages: [{ role: "user", content: userPrompt }]
  });

  output.textContent = success ? reply : `Error: ${error}`;
});
```

**content.js**

```javascript
const toolbar = document.createElement("div");
toolbar.id = "chatgpt-toolbar";
toolbar.innerHTML = `
  <button id="ask-chatgpt">Preguntar a ChatGPT</button>
  <div id="chatgpt-answer"></div>
`;
document.body.appendChild(toolbar);

document
  .querySelector("#ask-chatgpt")
  .addEventListener("click", async () => {
    const selection = window.getSelection().toString();
    const { success, reply, error } = await browser.runtime.sendMessage({
      type: "chatgpt:prompt",
      messages: [
        { role: "system", content: "Responde de forma concisa." },
        { role: "user", content: selection || "Resumen de la página" }
      ]
    });

    const answer = document.querySelector("#chatgpt-answer");
    answer.textContent = success ? reply : `Error: ${error}`;
  });
```

## 5. Gestiona secretos
1. Almacena la clave de API de OpenAI usando `browser.storage.local` con cifrado adicional o mediante la app contenedora para protegerla.
2. Evita hardcodear la clave directamente en los scripts distribuibles.
3. Ofrece un formulario en `popup.html` u opciones dedicadas para que el usuario ingrese y actualice su clave.
4. Usa el llavero del sistema desde la app contenedora si necesitas mayor protección.

Ejemplo para guardar la clave desde `popup.js`:

```javascript
const apiKeyInput = document.querySelector("#api-key");
const saveButton = document.querySelector("#save-api-key");

saveButton.addEventListener("click", async () => {
  await browser.storage.local.set({ apiKey: apiKeyInput.value.trim() });
  alert("Clave guardada correctamente");
});
```

## 6. Prueba en Safari
1. Ejecuta la app contenedora desde Xcode (⌘ + R) para instalar la extensión.
2. Habilita el modo de desarrollo en Safari (**Preferencias → Avanzado → "Mostrar menú Desarrollar"**).
3. Desde el menú **Develop → Allow Unsigned Extensions** durante la depuración.
4. Instala la extensión generada, prueba la funcionalidad y depura con **Web Inspector**.
5. Usa la consola del inspector (`⌘ + option + C`) para observar los mensajes enviados entre scripts.

## 7. Adapta para Chrome
1. Reutiliza la carpeta WebExtension para empaquetar un `.zip` e instalar la extensión en Chrome desde `chrome://extensions` en modo desarrollador.
2. Ajusta `manifest_version` y APIs específicas que difieran entre Safari y Chrome.
3. Sustituye el espacio de nombres `browser.*` por `chrome.*` o añade un polyfill (`browser-polyfill.js`).
4. Verifica los permisos adicionales que Chrome requiere (p. ej. `"scripting"` para ejecutar scripts dinámicos).

## 8. Empaqueta y publica
1. Utiliza **Xcode Organizer** para firmar la app contenedora y la extensión.
2. Crea un archivo `.pkg` o `.app` firmado si vas a distribuir fuera de la Mac App Store.
3. Distribuye a través de la **Mac App Store** o comparte un paquete ad-hoc.
4. Documenta el proceso de instalación para los usuarios finales e incluye capturas de pantalla.

## 9. Mantenimiento
1. Monitorea cambios en las políticas de OpenAI y Apple.
2. Gestiona cuotas y errores de la API.
3. Planifica actualizaciones periódicas con mejoras de UI/UX y soporte de nuevas funcionalidades.
4. Implementa telemetría opcional (con consentimiento del usuario) para saber qué funciones se usan con más frecuencia.
5. Añade pruebas automatizadas para scripts de contenido usando herramientas como **Jest** o **Vitest**.

## 10. Flujos de depuración recomendados
1. Activa `console.log` detallados en `background.js`, `popup.js` y `content.js` durante el desarrollo.
2. Usa el menú **Develop → Show Extension Builder** para recargar rápidamente la extensión en Safari.
3. Aprovecha los paneles **Network** y **Storage** del Web Inspector para verificar llamadas a la API y el contenido guardado en `browser.storage.local`.

## 11. Recursos adicionales
Además de la documentación oficial, considera revisar ejemplos de código abiertos y repositorios de referencia:

- [Documentación de Safari Web Extensions](https://developer.apple.com/documentation/safariservices/safari_web_extensions)
- [Guía de distribución de extensiones](https://developer.apple.com/documentation/safariservices/publishing_a_safari_web_extension)
- [API de OpenAI](https://platform.openai.com/docs/api-reference)
- [WebExtension Polyfill](https://github.com/mozilla/webextension-polyfill)
- [Ejemplo oficial de Apple "Reading List" Extension](https://developer.apple.com/documentation/safariservices/safari_web_extensions/adding_a_safari_web_extension_to_your_mac_app)
