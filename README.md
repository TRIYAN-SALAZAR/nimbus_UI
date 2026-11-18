# Nimbus UI

Interfaz web administrativa para Nimbus API, el backend de gestión de la cafetería. Está construida con Vue 3 y Vite, y consume la API REST mediante `fetch`.

## Requisitos

- Node.js `22.18+` o Node.js `24.12+`.
- Nimbus API ejecutándose y accesible desde el navegador.
- CORS habilitado en el backend para el origen del frontend.

## Instalación

```bash
npm install
```

Crear un archivo `.env` en la raíz:

```env
VITE_API_URL=http://localhost:3000
```

También se incluye `.env.example` como referencia. Si no se define `VITE_API_URL`, la aplicación utiliza `http://localhost:3000`.

> **Seguridad:** nunca colocar `SUPABASE_KEY`, `DATABASE_URL` ni ninguna credencial del backend en este proyecto. La clave service role debe permanecer exclusivamente en Nimbus API.

## Ejecución

Desarrollo:

```bash
npm run dev
```

La aplicación estará disponible normalmente en `http://localhost:5173`.

Producción:

```bash
npm run build
npm run preview
```

## Funcionalidades actuales

La interfaz incluye navegación y conexión con los siguientes módulos:

- Resumen con estado de conexión y estadísticas.
- Productos.
- Productos y variantes en un catálogo combinado.
- Extras.
- Insumos.
- Stock y alertas de reabastecimiento.
- Proveedores.
- Clientes.
- Métodos y orígenes de pago.
- Recetas de variantes y extras.
- Compras.
- Ventas.
- Tickets con historial de ventas.

Los formularios disponibles permiten crear productos, variantes, extras, insumos, proveedores y clientes. También permiten configurar recetas, registrar compras y registrar ventas. La administración de métodos y orígenes de pago está oculta en la navegación porque actualmente se utilizan catálogos definidos; sus datos siguen disponibles para las operaciones.

La sección de Productos combina la consulta de productos y variantes en una tabla que muestra nombre, categoría, variante abreviada, precio y estado. Una pestaña independiente conserva los formularios separados para crear productos y variantes.

## Compras y ventas con múltiples detalles

Una compra puede incluir varios insumos. Cada detalle contiene:

```json
{
  "id_insumo": 1,
  "cantidad": 10,
  "precio": 180
}
```

Una venta puede incluir varias variantes y extras independientes por producto:

```json
{
  "id_variante": 1,
  "cantidad": 2,
  "extras": [
    {
      "id_extra": 1,
      "cantidad": 1
    }
  ]
}
```

El frontend envía estos arreglos dentro de `detalles`. Los precios de venta y el total son responsabilidad del backend; el frontend no envía `total` ni `precio_cobrado`.

## Punto de venta

La sección de ventas presenta las variantes como tarjetas seleccionables y mantiene el ticket en un panel lateral. Permite ajustar cantidades, agregar extras, calcular el cambio al recibir efectivo y guardar tickets pendientes en `localStorage` sin enviarlos todavía al backend.

Los tickets pendientes pueden recuperarse o eliminarse desde el navegador. Las ventas confirmadas se muestran en la nueva sección `Tickets`.

La sección de compras tiene dos vistas: `Compras registradas` y `Nueva compra`. En la nueva compra se seleccionan insumos desde tarjetas, se agregan cantidades y se captura el precio unitario de cada detalle antes de enviarlo al backend.

## Estructura principal

```text
src/
├── App.vue
├── main.js
└── services/
    └── api.js
```

`src/services/api.js` centraliza las solicitudes HTTP, interpreta las respuestas `{ ok, data }` de Nimbus API y expone la URL configurada.

## Validación

Ejecutar build:

```bash
npm run build
```

Ejecutar lint:

```bash
npm run lint
```

## Limitaciones actuales

- Respaldos y administración de usuarios siguen siendo scripts administrativos del backend, no módulos HTTP del frontend.
- Las operaciones dependen de que Nimbus API esté activa y de que los catálogos relacionados ya existan.
- La autenticación HTTP todavía no está implementada en el backend; no exponer la API públicamente sin agregar autenticación y restringir CORS.
