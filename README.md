# Miniventa Admin

Panel web para consultar clientes, conceder o bloquear suscripciones, invalidar sesiones y revisar cambios de dispositivo. Usa Vue 3 y Vite.

## Desarrollo

1. Copia `.env.example` a `.env.local` y define `VITE_API_BASE_URL` con la URL base del API.
2. Instala dependencias con `npm install`.
3. Inicia el panel con `npm run dev` y abre la URL que muestra Vite. La pantalla solo solicita la clave administrativa; esta se mantiene en memoria mientras el panel está conectado.

`VITE_API_BASE_URL` se fija al iniciar Vite o compilar el panel. No se puede cambiar desde la interfaz. Si no está definida, en desarrollo se usa `http://localhost:3000`.

## Producción

Define `VITE_API_BASE_URL` con la URL fija del API y ejecuta `npm run build`; publica el contenido de `dist/` en un hosting HTTPS. Esta variable forma parte del frontend compilado, por lo que solo debe contener la URL pública del API, nunca secretos. Configura `ADMIN_API_KEY` únicamente en el entorno del API con un valor aleatorio de al menos 32 bytes; no la incluyas en el código del panel, en variables `VITE_*`, ni en el repositorio. Genera una con `openssl rand -hex 32` y entrega el valor solo a operadores autorizados. El panel y el API deben servirse por HTTPS.

El API debe permitir CORS desde el origen del panel y admitir la cabecera `X-Admin-Key`. La versión actual del API permite CORS de forma general; en despliegues públicos conviene restringir los orígenes permitidos al dominio del panel.

## Cuentas de clientes

El panel permite registrar cuentas de clientes, activar o bloquear licencias y guardar fecha de pago y vencimiento. No hay registro público ni cobros automáticos. Al guardar cambios de acceso se incrementa `sessionVersion`, por lo que las sesiones online anteriores quedan invalidadas. Aprobar un equipo nuevo reemplaza el dispositivo activo e invalida sus sesiones; rechazar una solicitud no afecta al equipo que ya estaba activo.
# Vue 3 + Vite

This template should help get you started developing with Vue 3 in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about IDE Support for Vue in the [Vue Docs Scaling up Guide](https://vuejs.org/guide/scaling-up/tooling.html#ide-support).
