# IA Plus Colombia — Guía de seguridad y despliegue

Este documento resume **qué cambié en tu página** y **qué debes hacer tú** para que
quede realmente segura. Los cambios del código ya están aplicados en `index.html`
(se guardó una copia del original en `index.original.html`).

---

## 🔴 1. URGENTE — Rotar la cuenta que estaba filtrada

En el HTML había credenciales reales visibles para cualquiera que abriera el código
fuente (correo, contraseña y **secreto del autenticador 2FA**):

- Correo: `tiffanyhardy3201@gmail.com`
- Contraseña: `Chatgpt666888`
- Secreto 2FA (TOTP): `IHTLVWXAXCKB2MKNSLK54D3WD6W4BVY2`

Ya las quité del archivo, **pero como estuvieron publicadas debes considerarlas
comprometidas**. Haz esto cuanto antes:

1. Cambia la contraseña de esa cuenta.
2. Regenera / desactiva y vuelve a activar el 2FA (el secreto anterior ya no sirve).
3. Revisa la actividad reciente de esa cuenta por si hubo accesos no autorizados.

> A partir de ahora, las credenciales de cada servicio NO deben escribirse en el HTML.
> Se cargan únicamente desde Firestore cuando el cliente canjea su código, que es como
> ya está diseñado el flujo (`redeemActivationCode`).

---

## 🔴 2. Publicar las reglas de Firestore (esto es la seguridad real)

Tu control de "administrador" hoy vive solo en el navegador: cualquiera con la consola
del navegador podría intentar leer o escribir en tu base de datos si las reglas no lo
impiden. El archivo **`firestore.rules`** que agregué cierra eso.

**Cómo publicarlas (opción fácil, sin instalar nada):**

1. Entra a [Firebase Console](https://console.firebase.google.com/) → proyecto
   `iapluscolombia-f9d76`.
2. Menú **Firestore Database** → pestaña **Reglas**.
3. Borra lo que haya y pega **todo el contenido de `firestore.rules`**.
4. Clic en **Publicar**.

**Opción con CLI (si usas Firebase CLI):**

```bash
firebase deploy --only firestore:rules
```

Estas reglas garantizan que:
- Solo el correo administrador (`wannaia333@gmail.com`) puede cambiar precios/WhatsApp,
  crear códigos y ver la lista de clientes.
- **Nadie puede listar la colección `activationCodes`** (no se pueden "sacar" todas las
  credenciales). Un cliente solo puede leer el código puntual que le corresponde.
- Cada usuario solo puede ver sus propios pedidos y activaciones.

> Si algún día cambias el correo administrador, cámbialo **en los dos lados**:
> en `index.html` (constante `ADMIN_EMAIL`) y en `firestore.rules`.

---

## 🟠 3. Restringir la API key y los dominios de Firebase

La `apiKey` de Firebase en el HTML **no es un secreto** (es normal que sea pública en
apps web), pero conviene limitarla:

1. **Dominios autorizados:** Firebase Console → Authentication → Settings →
   *Authorized domains*. Deja solo tu dominio real (y `localhost` para pruebas).
   Quita cualquier dominio que no uses.
2. **Restringir la clave:** [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
   → tu API key → *Application restrictions* → **HTTP referrers** → agrega solo tu
   dominio (ej. `https://tudominio.com/*`). Así la clave no funciona desde otros sitios.

---

## 🟢 4. Mejoras ya aplicadas en `index.html`

- ❌ **Credenciales reales eliminadas** del HTML (ahora aparecen como `—`).
- 🛡️ **Protección anti-XSS:** se agregó `escapeHtml()` y se aplicó al render de pedidos,
  usuarios y códigos del panel admin. Antes, un nombre de usuario malicioso podía
  inyectar código que se ejecutaba en tu sesión de administrador.
- 🔗 **Enlace 2FA saneado:** solo se aceptan URLs `http/https` (bloquea `javascript:`).
- 📄 **Páginas legales:** los enlaces del pie (Términos, Privacidad, Garantías) ahora
  abren un modal con contenido base editable. Ajústalo con tu asesor legal.
- ⚡ **Página 60% más liviana:** las imágenes estaban repetidas 12 veces (4 únicas).
  Ahora se guarda una sola copia de cada una y se reutiliza. El archivo pasó de
  **1.58 MB a ~613 KB**, sin cambiar nada visual. Sigue siendo un único archivo
  autónomo (funciona en cualquier hosting o abriéndolo directamente).

---

## 🔵 5. Recomendaciones siguientes (opcionales)

1. **Entrega de credenciales vía Cloud Function (lo más seguro).**
   Hoy el navegador del cliente lee el documento del código para mostrar las
   credenciales. Lo ideal es que una Cloud Function valide el canje en el servidor y
   devuelva las credenciales, para que el cliente **nunca** lea `activationCodes`
   directamente. Requiere el plan Blaze de Firebase.

2. **Peso de la página (ya reducido a ~613 KB).** Se eliminó la duplicación de imágenes.
   El siguiente paso sería **recomprimir el logo** (hoy es un PNG de 1020×699 px que se
   muestra a ~230 px): recortarlo a su tamaño real lo dejaría en ~30 KB en vez de 240 KB.
   Requiere una herramienta de imágenes (no disponible en este equipo ahora). Alternativa:
   mover las imágenes a archivos `.webp` externos si decides un hosting fijo.

3. **Pago real.** Hoy el "pago" solo abre WhatsApp. Si quieres cobros en línea, se puede
   integrar una pasarela (Wompi, Bold, Mercado Pago) con confirmación por servidor.

4. **Cumplimiento de terceros.** Revender/compartir cuentas de ChatGPT Plus y Claude Pro
   va en contra de los Términos de OpenAI y Anthropic. Es un riesgo del negocio, no del
   código; conviene tenerlo presente.

---

### Archivos en esta carpeta

| Archivo | Qué es |
|---|---|
| `index.html` | Tu página, ya mejorada y con las correcciones de seguridad. |
| `index.original.html` | Copia del archivo tal como estaba antes de mis cambios. |
| `firestore.rules` | Reglas de seguridad para publicar en Firebase. |
| `LEEME-SEGURIDAD.md` | Este documento. |
