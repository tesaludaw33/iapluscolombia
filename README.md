# IA Plus Colombia — plataforma completa

Esta versión evoluciona el HTML original a una tienda de servicios digitales con Firebase como backend y Wompi preparado para pagos en línea.

## Funciones incluidas

### Cliente
- Inicio de sesión con Google.
- Carrito persistente por usuario.
- Compra por WhatsApp con pedido registrado en servidor.
- Pago en línea mediante Wompi (cuando se configuran las llaves).
- Totales y descuentos validados en Cloud Functions.
- Cupones.
- Códigos de referido y seguimiento de referidos.
- Historial de pedidos y estados de pago/entrega.
- Activación manual con código.
- Activación automática después de pago aprobado y stock disponible.
- Renovaciones acumulables: una nueva vigencia se programa después de la cobertura existente.
- Historial de activaciones.
- Visualización de tiempo restante.
- Avisos de vencimiento dentro de la cuenta.
- Centro de soporte con tickets.
- Notificaciones internas.

### Administrador
- Precios y WhatsApp editables.
- Umbral de stock bajo.
- Configuración de días de beneficio por referido.
- Creación y control de códigos de activación.
- Credenciales separadas en `activationSecrets` y entregadas solo desde backend.
- Inventario por servicio/plan.
- Pedidos, estados de pago, entrega y reintento por falta de stock.
- Métricas de clientes, pedidos, pagos, ingresos y soporte.
- Gestión de tickets.
- Creación/activación/desactivación de cupones.
- Clientes recientes.
- Migración idempotente desde el esquema anterior (pedidos, vigencias y separación de credenciales).

## Estructura

```text
IA-Plus-Colombia-Completo/
├── index.html
├── firestore.rules
├── firestore.indexes.json
├── firebase.json
├── .firebaserc
├── SECURITY.md
├── README.md
└── functions/
    ├── index.js
    ├── package.json
    └── .env.example
```

## 1. Requisitos

Instala Node.js 20+ y Firebase CLI:

```bash
npm install -g firebase-tools
firebase login
```

Luego, desde esta carpeta:

```bash
cd functions
npm install
cd ..
```

## 2. Parámetros públicos

Copia los valores de `functions/.env.example` a un archivo de entorno del proyecto o deja que Firebase CLI los solicite al desplegar.

Los parámetros importantes son:

- `ADMIN_EMAIL`: correo administrador. Debe coincidir también con el correo definido en `firestore.rules`.
- `STORE_BASE_URL`: URL final de la tienda; el backend solo permite retornos de pago a ese origen (o localhost de desarrollo).
- `WOMPI_PUBLIC_KEY`: llave pública de Sandbox o Producción.
- `WOMPI_ENV`: `sandbox` o `production`.

La llave pública de Wompi no es el secreto de integridad.

## 3. Secretos de Wompi

Nunca los escribas en `index.html` ni en un archivo que vayas a publicar.

Configúralos con:

```bash
firebase functions:secrets:set WOMPI_INTEGRITY_SECRET
firebase functions:secrets:set WOMPI_EVENTS_SECRET
```

Ingresa el secreto correspondiente cuando Firebase CLI lo solicite.

## 4. Despliegue

Primero valida las funciones:

```bash
cd functions
npm run check
cd ..
```

Despliega reglas, índices, Functions y Hosting:

```bash
firebase deploy --only firestore:rules,firestore:indexes,functions,hosting
```

Después del despliegue, Firebase mostrará la URL pública de `wompiWebhook`. Configura esa URL como **Events URL** en el panel de Wompi del ambiente que estés usando.

## 4.1 Si ya estabas usando la versión anterior

Después de desplegar y entrar con la cuenta administradora, abre **Panel administrativo → Mantenimiento → Migrar datos anteriores**. La migración:

- copia pedidos antiguos de `users/{uid}/orders` al historial global;
- reconstruye `serviceStates` tomando la fecha de vencimiento máxima real;
- mueve credenciales antiguas a `activationSecrets`;
- elimina las credenciales embebidas de los documentos antiguos cuando puede hacerlo;
- puede ejecutarse más de una vez sin duplicar pedidos ya migrados.

La lógica de renovación también consulta activaciones antiguas aunque todavía no hayas ejecutado la migración, para no perder días vigentes.

## 5. Flujo de pago

1. El navegador envía al backend únicamente los IDs de servicio/plan, cupón y referido.
2. Cloud Functions vuelve a consultar los precios reales y calcula el total.
3. El backend crea un pedido con referencia única.
4. El backend genera la firma de integridad de Wompi.
5. El cliente pasa al Checkout de Wompi.
6. Wompi envía un webhook al backend.
7. El backend valida el checksum, monto, moneda y pedido.
8. Si el pago es `APPROVED`, se intenta asignar inventario.
9. Si hay stock, se crea la activación automáticamente.
10. Si no hay stock, el pedido queda `PAID_AWAITING_STOCK` para que el administrador agregue un código y pulse **Reintentar entrega**.

La redirección del navegador después de Wompi es solo informativa; la aprobación se verifica en servidor.

## 6. Renovaciones

La versión anterior guardaba una activación única por producto. Esta versión guarda un historial:

```text
users/{uid}/activations/{activationId}
users/{uid}/serviceStates/{productId}
```

Si el usuario renueva antes de vencer, la siguiente activación comienza al finalizar la cobertura existente. Así no pierde días.

## 7. Inventario

Cada código creado por administración genera:

```text
activationCodes/{code}      # metadatos, sin credenciales
activationSecrets/{code}    # información sensible, bloqueada por reglas
```

Cuando un pedido pagado se entrega, el backend toma un código disponible del producto/plan correspondiente y lo marca como usado.

## 8. Pruebas recomendadas antes de publicar

- Inicio/cierre de sesión Google.
- Usuario normal intentando abrir `#admin`.
- Cupón válido, vencido, desactivado y sobre límite de usos.
- Código de referido propio (debe rechazarse).
- Código de activación válido, usado y de producto incorrecto.
- Renovación antes y después del vencimiento.
- Pago Wompi Sandbox aprobado y rechazado.
- Reenvío duplicado del webhook: no debe crear dos activaciones.
- Pago aprobado sin stock: debe quedar esperando inventario.
- Agregar stock y usar **Reintentar entrega**.
- Ticket de soporte y respuesta del administrador.
- Vista móvil.

## Qué queda pendiente para que esté EN PRODUCCIÓN

El código está preparado, pero hay acciones externas que no se pueden inventar dentro del proyecto:

1. Instalar dependencias (`npm install` en `functions`).
2. Publicar reglas, índices, Functions y Hosting en tu proyecto Firebase.
3. Configurar la llave pública y los dos secretos reales de tu comercio Wompi.
4. Registrar la URL desplegada de `wompiWebhook` en Wompi.
5. Probar primero con Sandbox y después cambiar a Producción.
6. Rotar cualquier credencial que alguna vez haya quedado expuesta en HTML antiguo.

Sin las llaves reales de tu comercio, **el pago online queda intencionalmente deshabilitado**, mientras la compra por WhatsApp continúa disponible.

## Importante

Revisa `SECURITY.md` antes de producción. No reutilices credenciales que hayan quedado expuestas en versiones antiguas del sitio.


## Video de portada

La portada usa `assets/hero-bg.mp4` como video de fondo profesional y `assets/hero-poster.jpg` como imagen de respaldo mientras el video carga o cuando el usuario tiene activada la reducción de movimiento. El video está optimizado para web, se reproduce en silencio, en bucle y se pausa cuando la portada o la pestaña dejan de estar visibles.
