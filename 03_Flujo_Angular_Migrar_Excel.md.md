# Módulo `prcMigrarExcel` (Migrar Excel)

Documentación técnica del feature ubicado en
`src/app/features/procesos/prcMigrarExcel/migrar-excel`.

## 1. Propósito

Este módulo permite registrar **series** (números de serie de producto,
ej. IMEI, número de lote, etc.) en el sistema por dos vías:

1. **Carga masiva desde archivo Excel** (`.xlsx`): el usuario sube un
   archivo, selecciona la hoja a procesar y el backend extrae e inserta las
   series contenidas en ella.
2. **Ingreso individual por lectura de escáner / teclado** (pestaña
   "Ingresar Serie"): el usuario apunta un lector de código de barras (o
   escribe manualmente) hacia un campo de texto y cada serie leída se envía
   e inserta una por una, en tiempo real.

Ambas vías requieren primero seleccionar un **Producto** y una **Bodega**,
ya que toda serie insertada queda asociada a esa combinación.

## 2. Arquitectura y tecnologías

- **Lazy loading** de rutas mediante `loadComponent()` (ver
  [migrar-excel.routes.ts](../src/app/features/procesos/prcMigrarExcel/migrar-excel/migrar-excel.routes.ts)).
- **RxJS** (`Observable`, operadores `map`/`catchError`) para las llamadas HTTP.
- **`HttpClient`** de `@angular/common/http` con `FormData` (multipart) para
  el envío de archivos y parámetros.
- **`@ngx-translate/core`** (`TranslatePipe`) para internacionalización de
  textos de la UI (claves como `labelsExcel.*`, `buttonsLabelsExcel.*`).
- **TailwindCSS** para el estilado (clases utilitarias en los templates).
- **Font Awesome** para iconografía (`fa-solid fa-barcode`, `fa-file-excel`, etc.).
- Componentes **compartidos** del proyecto (`shared/components/...`):
  `SidebarCommonComponent`, `HeaderCommonComponent`, `SettingsDrawerComponent`,
  `BarraEstadoComponent`, `BodegaProductoSelectorComponent`, `ButtonComponent`,
  `SpinnerCargaComponent`, `TablaPaginadaComponent`, `MessagesComponent`.
- Servicios **core** (`core/services/...`): `SharedService`, `LoginService`,
  `UtilidadService`, `ErrorModalService`.

> Nota: aunque el proyecto tiene instaladas librerías de manejo de Excel en
> el cliente (`exceljs`, `xlsx`), **este módulo no las usa**. El archivo se
> envía tal cual al backend (`FormData`) y es el servidor quien procesa el
> contenido del Excel (lectura de hojas y extracción de series).

## 3. Estructura de archivos

```
prcMigrarExcel/
└── migrar-excel/
    ├── migrar-excel.component.ts/html/css   # Shell/layout de la vista
    ├── migrar-excel.routes.ts               # Definición de rutas del feature
    ├── models/
    │   ├── documento-estructura.ts          # Interfaz DocumentoEstructura
    │   └── tbl-serie.ts                     # Interfaz PaTblSerieM
    ├── services/
    │   └── migrar-excel.service.ts          # Cliente HTTP hacia las APIs GD01/GD02
    └── components/
        └── traslado/
            ├── traslado.component.ts        # Lógica principal (carga Excel + escáner)
            ├── traslado.component.html
            └── traslado.component.css
```

## 4. Rutas (`migrar-excel.routes.ts`)

```ts
export const MigrarExcelRoutes: Routes = [
  {
    path: 'proceso/migrar-excel',
    loadComponent: () => import('./migrar-excel.component').then(m => m.MigrarExcelComponent),
    title: 'Migrar Excel',
    canActivate: [AuthGuard, AppSelectionGuard],
    children: [
      { path: '', redirectTo: 'trasladoDatos', pathMatch: 'full' },
      { path: 'trasladoDatos', loadComponent: () => import('./components/traslado/traslado.component').then(m => m.TrasladoComponent) },
    ],
  },
];
```

- Registrado globalmente en `app-routing.module.ts` mediante
  `...MigrarExcelRoutes`.
- Protegido por `AuthGuard` (sesión iniciada) y `AppSelectionGuard`
  (empresa/aplicación seleccionada).
- URL resultante: `/proceso/migrar-excel/trasladoDatos`.

## 5. `MigrarExcelComponent` (shell)

Es el contenedor de layout de la página: sidebar, header, drawer de
configuración, barra de estado inferior y un `<router-outlet>` donde se
renderiza `TrasladoComponent`. Obtiene del `LoginService` el usuario,
estación de trabajo y empresa para mostrarlos en la barra de estado, y se
suscribe a `SharedService` para la visibilidad del drawer y el estado
minimizado del sidebar (ajusta márgenes dinámicamente).

No contiene lógica de negocio propia del proceso de migración.

## 6. `TrasladoComponent` (lógica principal)

Es el componente donde ocurre todo el flujo funcional. Se estructura en dos
secciones dentro de la plantilla:

### 6.1 Selección previa obligatoria: Producto y Bodega

Usa `<app-bodega-producto-selector>`:

1. El usuario escribe un criterio y busca (`obtenerProducto()` →
   `MigrarExcelService.getProducto()` → API `GD01/Apis/api/Producto/buscar`).
2. Al elegir un producto, se consultan automáticamente las bodegas donde
   existe disponibilidad (`obtenerProductoBodega()` →
   `MigrarExcelService.getProductoBodega()` → API `GD01/Apis/api/producto/Bodega`).
3. Al elegir la bodega, el componente emite `seleccionCompleta` con
   `{ bodega, producto }`.
4. `TrasladoComponent.recibirSeleccion()` guarda `bodegaSeleccionada` /
   `productoSeleccionado` y activa `mostrarInputSerie = true`, habilitando
   las pestañas de trabajo.

Si el usuario cambia la selección, `onResetSeleccion()` limpia todo el
estado (archivo cargado, hojas, tabla, series) mediante `removerFile()`.

### 6.2 Pestañas de trabajo (`activeTab`)

Solo visibles cuando hay `productoSeleccionado`:

| Pestaña | Valor `activeTab` | Función |
|---|---|---|
| Traslado de excel | `trasladoArchivo` | Carga masiva desde archivo Excel |
| Ingresar Serie | `inputSerie` | Ingreso manual / **por escáner** (ver sección 7) |

#### Traslado de excel

1. `cargarFile(event)`: toma el archivo del `<input type="file" accept=".xlsx, .xls">`.
   - Si ya había un archivo cargado, muestra advertencia (auto-oculta a los 7s) y no permite reemplazarlo sin antes quitarlo.
   - Si es válido, llama a `MigrarExcelService.obtenerHojasExcel(file)` →
     `POST GD02/Apis/api/ObtenerHojasExcelCtrl` (multipart, campo `archivoExcel`).
   - Si el Excel tiene una sola hoja, se autoselecciona (`seleccionarHoja`).
2. El usuario elige la hoja en un dropdown personalizado (`toggleDropdown` / `seleccionarHoja`).
3. Botón "Trasladar Datos" abre un modal de confirmación (`mostrarModalTraslado` → `app-messages`).
4. Al confirmar, `trasladarDatosSeries()` arma el modelo y llama a
   `MigrarExcelService.trasladarDatos(model)` →
   `POST GD02/Apis/api/PaTblSerieCtrl` (multipart) con:
   `ArchivoExcel`, `NombreHojaExcel`, `tAccion=1`, `pUserName`,
   `pConsecutivo_Interno=0`, `pTipo_Estructura=1`, `pEstado=1`,
   `pBodega`, `pProducto`, `pUnidad_Medida`.
5. Éxito: refresca la tabla de series (`obtenerSeries()`), muestra mensaje de
   éxito y reinicia `hojaSeleccionada` a los 3s. Error: modal de error vía
   `ErrorModalService`.

#### Ingresar Serie (ver detalle completo en sección 7)

### 6.3 Tabla de series registradas

Se muestra con `<app-tabla-paginada>` usando `datosTablaSerie` (tipo
`PaTblSerieM[]`), columnas `serie` y `fecha_Hora`, con una columna especial
de checkbox (`templateSelChk`) para selección múltiple.

- `actualizarSeriesSeleccionadas()`: recalcula `seriesSeleccionadas` filtrando por `seleccionado === true`.
- `eliminarSerie()`: por cada serie seleccionada llama a
  `MigrarExcelService.deleteSerie({ tAccion: 3, serie })` →
  `DELETE GD02/Apis/api/PaDelSerieCtrl`, y refresca la tabla tras cada
  eliminación exitosa.
- `obtenerSeries()`: refresca el historial vía
  `MigrarExcelService.obtenerSeriesHistorial({ tAccion: 2, pUserName, pConsecutivoInterno: 0, pTipoEstructura: 1, pEstado: 1 })` →
  `GET GD02/Apis/api/PaBscSeriesCtrl`.

### 6.4 Manejo de mensajes

`manejarMensajeExito` / `manejarMensajeError` controlan banners temporales
(5s éxito, 8s error) mostrados por `<app-messages>`, el mismo componente que
maneja el modal de confirmación de traslado.

## 7. Funcionalidad de lectura de serie por escáner (foco de esta documentación)

### 7.1 Cómo funciona técnicamente el "escaneo"

**No existe integración con ninguna librería o API de cámara/lector.** El
mecanismo se apoya en el comportamiento estándar de los **lectores de
código de barras tipo HID** (Human Interface Device — USB o Bluetooth en
modo teclado), que son la inmensa mayoría de lectores físicos usados en
bodega/almacén:

1. El lector, al escanear un código, **emula un teclado**: "escribe"
   rápidamente cada carácter del código y finaliza enviando la tecla
   **Enter** (comportamiento configurable de fábrica en casi todo lector,
   usado como "terminador de trama").
2. Angular no necesita saber que el origen es un escáner: solo necesita que
   el **foco del navegador** esté puesto sobre el `<input>` de texto para
   que los caracteres emitidos por el lector lleguen a él como si se
   hubiesen tecleado.

### 7.2 Elementos involucrados en el template (`traslado.component.html`)

```html
<div *ngIf="mostrarInputSerie">
  <h2>Ingrese o escanee una serie:</h2>

  <input
    #serieInputRef
    type="text"
    name="serie"
    [placeholder]="'Ingrese la serie o escanee' | translate"
    [(ngModel)]="serieInput"
    (keydown.enter)="ingresarSerie()"
    required
  />
  <app-button [text]="'Agregar serie'" (onClick)="ingresarSerie()"></app-button>

  <p class="text-xs ...">
    Escanee la serie o escríbala manualmente. Asegúrese de que el cursor
    esté en este campo antes de escanear.
  </p>
</div>
```

- `#serieInputRef`: referencia de plantilla capturada en el componente con
  `@ViewChild('serieInputRef') serieInputRef!: ElementRef;`, usada para
  **re-enfocar** el input programáticamente.
- `[(ngModel)]="serieInput"`: binding bidireccional; el texto que va
  "tecleando" el escáner se refleja en `serieInput`.
- `(keydown.enter)="ingresarSerie()"`: el evento clave — como el escáner
  termina cada lectura con Enter, este binding dispara automáticamente el
  guardado **sin que el usuario tenga que hacer clic en nada**. El botón
  "Agregar serie" es solo un respaldo para ingreso manual con mouse.
- El texto de ayuda bajo el input recuerda al operario que **el cursor debe
  estar posicionado en el campo** antes de escanear (requisito indispensable
  del enfoque basado en teclado emulado).

### 7.3 Lógica en `TrasladoComponent.ingresarSerie()`

```ts
ingresarSerie(): void {
  const user = this.loginService.getUser();
  this.isVisibleModal = false;
  this.tempMostrarTablaBod = false;

  if (!this.serieInput) {
    this.manejarMensajeError('Se ha ingresado o escaneado una serie.');
    return;
  }

  const model = {
    tAccion: 1,
    userName: user,
    serie: this.serieInput,
    unidadMedida: this.productoSeleccionado?.unidad_Medida,
    bodega: this.bodegaSeleccionada?.bodega,
    producto: this.productoSeleccionado?.producto,
  };
  this.cargandoTraslado = true;

  this.migrarExcelService.insertSerie(model).subscribe({
    next: (response) => {
      this.cargandoTraslado = false;
      this.tempMostrarTablaBod = true;
      this.obtenerSeries();                      // refresca la tabla
      this.manejarMensajeExito('Datos trasladados correctamente.');
    },
    error: (err) => {
      const detalles = err.detallesError || {};
      this.errorModalService.mostrarError(detalles);
      this.cargandoTraslado = false;
      this.tempMostrarTablaBod = false;
    },
  });

  this.serieInput = '';                            // limpia el campo
  setTimeout(() => {
    this.serieInputRef.nativeElement.focus();       // reenfoca el input
  }, 0);
}
```

Puntos clave del flujo, en orden:

1. **Validación**: si `serieInput` está vacío no hace nada (mensaje de
   alerta).
2. **Construcción del modelo**: la serie leída se combina con
   `productoSeleccionado` y `bodegaSeleccionada` (obligatorios, elegidos
   previamente en el paso 6.1) y con el usuario autenticado
   (`loginService.getUser()`).
3. **Envío al backend**: `MigrarExcelService.insertSerie(model)` hace un
   `POST` a `GD02/Apis/api/PaInsertTblSerieCtrl/insertar` con `FormData`
   (`tAccion`, `userName`, `serie`, `bodega`, `producto`, `unidadMedida`) y
   token `Authorization: Bearer`.
4. **Limpieza y reenfoque inmediatos y optimistas**: nótese que
   `this.serieInput = ''` y el `setTimeout(..., 0)` que reenfoca el input
   se ejecutan **fuera del callback `subscribe`**, es decir, **inmediatamente
   después de disparar la petición HTTP, sin esperar la respuesta**. Esto es
   una decisión de UX deliberada: permite que el operario escanee la
   siguiente serie de inmediato, en cadena, sin esperar el round-trip de red
   de la serie anterior — crítico para procesos de escaneo masivo en bodega
   donde se leen muchas series por minuto.
5. **Confirmación visual**: al resolver la petición (éxito), se refresca la
   tabla de series (`obtenerSeries()`) y se muestra un mensaje de éxito
   temporal; en caso de error se abre el modal de error
   (`ErrorModalService`), aunque el input ya está libre para seguir
   escaneando.
6. El `setTimeout(fn, 0)` (en vez de llamar `focus()` directo) se usa para
   encolar el reenfoque **después** del ciclo de detección de cambios de
   Angular que procesa el `[(ngModel)]` recién vaciado, evitando condiciones
   de carrera donde el DOM aún no refleja el valor limpio del input.

### 7.4 Prerrequisitos para que el escaneo funcione

- Debe existir `productoSeleccionado` y `bodegaSeleccionada` (de lo
  contrario `mostrarInputSerie` es `false` y la pestaña "Ingresar Serie" no
  muestra el input).
- El foco del navegador debe estar sobre `#serieInputRef` al momento de
  disparar el lector (el propio componente lo garantiza tras cada
  inserción vía `focus()`, pero la primera vez el usuario debe hacer clic
  manualmente en el campo).
- El lector debe estar configurado en **modo teclado (HID)** y con
  **sufijo Enter/CR** habilitado — configuración de fábrica en la mayoría
  de escáneres USB/Bluetooth estándar.

## 8. `MigrarExcelService` — API HTTP consumida

| Método | Verbo / Endpoint | Uso |
|---|---|---|
| `obtenerHojasExcel(archivo)` | `POST GD02/Apis/api/ObtenerHojasExcelCtrl` | Lista las hojas de un Excel subido |
| `trasladarDatos(model)` | `POST GD02/Apis/api/PaTblSerieCtrl` | Migra series masivamente desde el Excel |
| `insertSerie(model)` | `POST GD02/Apis/api/PaInsertTblSerieCtrl/insertar` | Inserta **una** serie (ingreso manual o **escaneada**) |
| `obtenerSeriesHistorial(model)` | `GET GD02/Apis/api/PaBscSeriesCtrl` | Lista/actualiza la tabla de series registradas |
| `deleteSerie(model)` | `DELETE GD02/Apis/api/PaDelSerieCtrl` | Elimina una serie |
| `getProducto(model)` | `GET GD01/Apis/api/Producto/buscar` | Búsqueda de productos (usa token `jwtTokenGD01`) |
| `getProductoBodega(model)` | `GET GD01/Apis/api/producto/Bodega` | Bodegas disponibles para un producto |
| `conectarBaseDeDatos(model)` | `POST GD02/Apis/api/ConexionCtrl` | (utilitario, no usado directamente en `TrasladoComponent`) |
| `subirArchivos(files, urlCarpeta)` | `POST GD02/Apis/api/FilesCtrl` | (utilitario genérico de subida de archivos, no usado en este flujo) |

Detalles comunes:

- La URL base (`baseUrl`) se lee de `localStorage`/`sessionStorage`
  (`urlApi`), configurable en tiempo de ejecución vía `setBaseUrl()`.
- Todas las peticiones a `GD02` usan el token `jwtToken`; las de `GD01`
  (catálogo de productos) usan `jwtTokenGD01`.
- Manejo de errores centralizado con `catchError`, normalizando la
  respuesta a `{ detallesError: { message, error, procedimiento, timestamp } }`,
  que luego consume `ErrorModalService.mostrarError()` para mostrar un modal
  (con opción de reintentar la operación original, pasando el callback
  `.bind(this)`).

## 9. Modelos

```ts
// documento-estructura.ts — estructura de un documento migrado (histórico/plantilla)
export interface DocumentoEstructura {
  consecutivo_Interno: number;
  estructura: string;
  userName: string;
  fecha_Hora: Date;
  tipo_Estructura: number;
  estado: number;
  m_UserName: string;
  m_Fecha_Hora: Date;
}

// tbl-serie.ts — fila de la tabla de series (Excel o escáner)
export interface PaTblSerieM {
  serie: string;
  fecha_Hora: Date;
  seleccionado?: boolean; // estado de checkbox en la UI, no viene del backend
}
```

## 10. Resumen del flujo de usuario

```
1. Ingresar a /proceso/migrar-excel/trasladoDatos (requiere sesión + empresa seleccionada)
2. Buscar y seleccionar un Producto
3. Seleccionar una Bodega (filtradas automáticamente según el producto)
4. Elegir pestaña:
   a) "Traslado de excel":
      - Subir archivo .xlsx/.xls
      - Elegir hoja
      - Confirmar traslado → API procesa el Excel e inserta las series
   b) "Ingresar Serie" (manual o ESCÁNER):
      - Posicionar el cursor en el input
      - Escanear (o teclear) la serie → Enter dispara el guardado automáticamente
      - El input se limpia y se reenfoca de inmediato para el siguiente escaneo
      - Cada serie se inserta individualmente contra el backend
5. La tabla de series registradas se actualiza tras cada operación,
   permitiendo seleccionar (checkbox) y eliminar series existentes.
```