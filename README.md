# Miniventa Admin

Panel web para administrar clientes, precios de suscripción, pagos confirmados e ingresos, además de revisar cambios de dispositivo. Usa Vue 3 y Vite.

## Desarrollo

1. Copia `.env.example` a `.env.local` y define `VITE_API_BASE_URL` con la URL base del API.
2. Instala dependencias con `npm install`.
3. Inicia el panel con `npm run dev` y abre la URL que muestra Vite. La pantalla solo solicita la clave administrativa; esta se mantiene en memoria mientras el panel está conectado.

`VITE_API_BASE_URL` se fija al iniciar Vite o compilar el panel. No se puede cambiar desde la interfaz. Si no está definida, en desarrollo se usa `http://localhost:3000`.

## Producción

Define `VITE_API_BASE_URL` con la URL fija del API y ejecuta `npm run build`; publica el contenido de `dist/` en un hosting HTTPS. Esta variable forma parte del frontend compilado, por lo que solo debe contener la URL pública del API, nunca secretos. Configura `ADMIN_API_KEY` únicamente en el entorno del API con un valor aleatorio de al menos 32 bytes; no la incluyas en el código del panel, en variables `VITE_*`, ni en el repositorio. Genera una con `openssl rand -hex 32` y entrega el valor solo a operadores autorizados. El panel y el API deben servirse por HTTPS.

El API debe permitir CORS desde el origen del panel y admitir la cabecera `X-Admin-Key`. La versión actual del API permite CORS de forma general; en despliegues públicos conviene restringir los orígenes permitidos al dominio del panel.

## Cuentas de clientes

Las cuentas nuevas quedan bloqueadas y requieren confirmar el primer pago para activar el acceso. Cada cliente tiene un precio configurable en MXN. La acción “Pago / renovar” registra monto, fecha de pago y vencimiento elegidos por el operador; cada confirmación se conserva en un historial financiero separado de la cuenta. No hay cobros ni renovaciones automáticos. El resumen de ingresos muestra el total cobrado histórico y hasta 500 movimientos recientes; no representa utilidad neta porque no resta costos ni comisiones.

El registro de pagos requiere que la API use un replica set o clúster MongoDB compatible con transacciones. El historial solo incluye pagos confirmados mediante este flujo; las fechas de pago antiguas no contienen montos y no se suman automáticamente.

Desde la ficha se puede bloquear el acceso, lo que invalida las sesiones online, o eliminar la cuenta con doble confirmación. La eliminación de la cuenta conserva los movimientos en el historial de ingresos. Aprobar un equipo nuevo reemplaza el dispositivo activo e invalida sus sesiones; rechazar una solicitud no afecta al equipo que ya estaba activo.

## Catálogo de productos

En Productos se registran número, nombre y precio en MXN, y se pueden editar o eliminar del catálogo global. Los números son texto: `001` y `1` son diferentes. Los dispositivos descargan el catálogo al pulsar refrescar; el número coincidente reemplaza nombre y precio locales. La eliminación en la nube no borra el producto local y cambiar un número deja intacto el número anterior en los dispositivos que ya lo tenían.

## Android

La aplicación Android usa el identificador `com.miniventa.admin` y requiere Android 7.0 (API 24) o posterior. Instala JDK 21 y Android SDK 36, configura `JAVA_HOME` y `ANDROID_HOME`, y define `VITE_API_BASE_URL` con la URL HTTPS pública de la API. La clave administrativa se solicita al abrir el panel y no debe incluirse en la compilación.

Para generar una APK release firmada, coloca `keystore.properties` y el almacén de claves en `android/`, o configura `MINIVENTA_ANDROID_SIGNING_DIR` con la ruta a un directorio que los contenga. El archivo debe definir `storeFile`, `storePassword`, `keyAlias` y `keyPassword`; no compartas ni subas esos archivos. Usa siempre la misma clave para firmar las actualizaciones de una instalación existente.

Ejecuta `npm run android:release:apk`. La APK firmada queda en `android/app/build/outputs/apk/release/app-release.apk`; sin configuración de firma, Gradle genera `app-release-unsigned.apk`. También puedes ejecutar `npm run android:sync` y `npm run android:open` para abrir el proyecto en Android Studio.
# Vue 3 + Vite

This template should help get you started developing with Vue 3 in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about IDE Support for Vue in the [Vue Docs Scaling up Guide](https://vuejs.org/guide/scaling-up/tooling.html#ide-support).
