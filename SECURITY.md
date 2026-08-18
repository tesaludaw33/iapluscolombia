# Seguridad — IA Plus Colombia

## Principios aplicados

- Las credenciales de servicios no se leen directamente desde Firestore por el navegador.
- `activationCodes`, `activationSecrets`, `coupons` y `referrals` están bloqueados para acceso directo del cliente.
- El cliente no puede cambiar su código de referido ni falsear el correo de su perfil mediante Firestore.
- Las URLs de retorno del checkout se restringen al dominio configurado (o localhost de desarrollo).
- La creación/canje de códigos, administración, pagos y soporte sensible pasan por Cloud Functions.
- Los totales de compra se recalculan en el servidor; el navegador no decide el valor final.
- La activación por pago solo se ejecuta después de una confirmación de pago válida.
- El webhook de Wompi verifica el checksum del evento antes de actualizar un pedido.
- Los secretos de Wompi deben guardarse en Google Cloud Secret Manager mediante Firebase CLI.

## Acciones obligatorias antes de producción

1. Rota cualquier contraseña o secreto 2FA que haya estado alguna vez dentro de versiones HTML antiguas.
2. Publica `firestore.rules` y `firestore.indexes.json`.
3. Restringe los dominios autorizados de Firebase Authentication.
4. Restringe la API key web de Firebase por HTTP referrer al dominio real y localhost de desarrollo.
5. Usa HTTPS.
6. Configura Wompi primero en Sandbox y prueba pago aprobado, rechazado, webhook duplicado y stock agotado.
7. No subas `.env`, `.secret.local`, claves privadas ni secretos de integridad/eventos a Git.

## Nota sobre credenciales de terceros

Este proyecto implementa un mecanismo técnico genérico de entrega de información de acceso. Úsalo únicamente para servicios y licencias cuya distribución estés autorizado a realizar y de acuerdo con los términos del proveedor correspondiente.
