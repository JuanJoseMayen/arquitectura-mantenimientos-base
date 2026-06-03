# Documentación Técnica: Estandarización de Módulos de Mantenimiento (CRUD)

> **NOTA DE DISEÑO GLOBAL:**
> Este proyecto utiliza **Tailwind CSS** a nivel global para todo el diseño, definición de estilos y maquetación de la interfaz de usuario. Cualquier nueva vista, componente o formulario dentro de los mantenimientos debe implementarse utilizando las clases utilitarias de Tailwind CSS.
### Instalación Rápida de Tailwind CSS
Si se necesita configurar Tailwind CSS desde cero o verificar la instalación en tu entorno de Angular, se puede seguir estos pasos basados en la documentación oficial:

**1. Instalar Tailwind y sus dependencias**
Abre la terminal en la raíz del proyecto y ejecuta el siguiente comando para instalar Tailwind junto con PostCSS y Autoprefixer. Luego, inicializa el archivo de configuración:
```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init
```
**2. Configurar las rutas de las plantillass**
Abre el archivo tailwind.config.js que se acaba de generar en la raíz del proyecto y asegúrate de incluir las rutas de todos tus archivos .html y .ts en la propiedad content. Esto le indica a Tailwind dónde buscar sus clases para compilarlas:
```typescript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/**/*.{html,ts}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```
**3. Agregar las directivas de Tailwind a tu CSS global**
Abre el archivo de estilos principales del proyecto (por lo general es src/styles.css o src/styles.scss) y añade las directivas de las capas de Tailwind al principio del documento:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
---

## 1. Componentes Centralizados

Para asegurar la coherencia visual y evitar la duplicidad de código, los módulos de mantenimiento hacen uso de una arquitectura basada en componentes compartidos.

* `app-messages`: Utilizado para la gestión de notificaciones y alertas globales (éxito, error, advertencias).
* `app-modal` *(Componente Base)*: Encapsula la estructura visual de las ventanas emergentes y cuadros de diálogo.
**Nota:** Este componente actúa como un contenedor abstracto integrado en el proyecto; por lo tanto, no es necesario recrearlo, sino únicamente invocarlo o adaptar las instancias existentes en tu módulo (como las confirmaciones de guardado o eliminación) para mantener la consistencia del flujo.

### Énfasis Exclusivo: El componente `items-list`
El componente `items-list` se creó **específicamente para estandarizar y manejar todos los listados de los mantenimientos** de la aplicación. Es el núcleo visual de lectura de datos. 
Su propósito es recibir dinámicamente un arreglo de columnas y los datos correspondientes, renderizando una tabla estandarizada. Toda la lógica de presentación tabular y emisión de eventos principales (como botones de "Agregar", "Editar" o "Eliminar") está abstraída dentro de este componente. **No se deben crear tablas manuales en los nuevos módulos; se debe invocar invariablemente a `items-list`.**

---

## 2. Lógica de Funcionamiento y Flujo

El flujo operativo estándar de cualquier módulo de mantenimiento se rige bajo la siguiente secuencia lógica:

1.  **Manejo y Navegación mediante Pestañas (Tabs):** La navegación entre la visualización de datos (Lista) y la inserción/edición de los mismos (Formulario) se controla mediante Tabs lógicos en el componente TypeScript.
2.  **Renderizado de la Lista (`items-list`):** Al inicializar el módulo, el componente padre solicita los datos al servicio y los pasa al `<app-items-list>`.
3.  **Acciones Disparadas desde la Lista:** El `items-list` emite eventos (`@Output`) para Crear, Editar y Eliminar. El componente padre reacciona cambiando el Tab o lanzando confirmaciones.

---

## 3. Arquitectura de Servicios y Payload Reutilizable

Para optimizar el mantenimiento y el funcionamiento a futuro, los servicios no separan las peticiones HTTP metódicamente en funciones aisladas (get, post, put, delete). En su lugar, utilizan un **patrón de métodos reutilizables centralizados**:

*   **Interfaz de Parámetros Dinámica (`RequestParams[Entidad]`):** Centraliza todos los campos posibles de la entidad y la acción a ejecutar.
*   **`buildPayload`:** Construye dinámicamente el cuerpo de la petición (JSON). Dependiendo de si es un INSERT (agrega datos de creación), UPDATE o DELETE (agrega datos de auditoría de modificación y el ID).
*   **`ejecutarOperacion`:** Método base que ejecuta la llamada HTTP genérica, enviando el endpoint, la acción y los parámetros.
*   **`crud[NombreEntidad]`:** Métodos públicos específicos (ej. `crudCatalogo`, `crudVersion`) que fungen como fachada para llamar a `ejecutarOperacion` con el endpoint correcto.

---

## 4. Implementación de Referencia (`mntVersionApp`)

A continuación, se detalla la plantilla de código de referencia completa, implementando el patrón centralizado. 

### 4.1. Interfaces y Tipados (`mnt-version-app.model.ts`)
```typescript
// Define el identificador de la operación a realizar en base de datos.
export enum CrudAccion {
  Select = 2,
  Insert = 1,
  Update = 3,
  Delete = 4
}

// ¿Para qué se usa?: Esta interfaz estructura el payload exacto que el servicio dinámico 
// necesita para procesar cualquier acción (Insert, Update, Delete).
export interface RequestParamsVersionApp {
  // ── 1. Parámetros Base de Control (Obligatorios para la lógica central) ──
  accion: CrudAccion;
  id?: number;          // Se envía únicamente en Update o Delete
  idFieldName?: string; // Nombre exacto de tu llave primaria en la BD (ej: 'IdUsuario')
  
  // ── 2. Payload de la Entidad (ADAPTAR) ──
  // Define AQUÍ únicamente los campos de tu tabla que son estrictamente 
  // necesarios para guardar o actualizar el registro. Evita agregar campos de relleno.
  // Ejemplo:
  // descripcion?: string;
  // estado?: number;
}
```
### 4.2. Servicio Adaptable (mnt-version-app.service.ts)
```typescript
@Injectable({
  providedIn: 'root'
})
export class TuEntidadService {
  private readonly baseUrl: string = 'api/v1/'; 

  // Configura las rutas relativas de tus controladores en el backend
  private readonly endpoints = {
    principal: 'MantenimientoEntidad/ControlPrincipal',
    catalogos: 'MantenimientoEntidad/CatalogosAuxiliares',
  };

  constructor(
    private http: HttpClient,
    private loginService: LoginService, // Para obtener el usuario activo
    private injector: Injector          // Inyección bajo demanda para evitar dependencias cíclicas
  ) {}

  // ── Helper Interno: Construcción dinámica del Payload de Auditoría y CRUD ──
  private buildPayload(accion: CrudAccion, params: RequestParamsEntidad): Record<string, any> {
    const usuarioActivo = this.loginService.getUser();
    
    // Base obligatoria: El backend siempre requiere la acción a ejecutar
    const body: Record<string, any> = { Accion: accion };

    // 1. Identificación (Requerido en UPDATE y DELETE)
    if ((accion === CrudAccion.Update || accion === CrudAccion.Delete) && params.id != null) {
      const primaryKey = params.idFieldName ?? 'Id';
      body[primaryKey] = params.id;
    }

    // 2. Mapeo de Contenido (Solo INSERT y UPDATE)
    if (accion === CrudAccion.Insert || accion === CrudAccion.Update) {
      // SUSTITUIR: Mapea únicamente las propiedades de tu interfaz que espera recibir el Backend
      // Ejemplo:
      // if (params.descripcion != null) body['Descripcion'] = params.descripcion;
      // if (params.estado != null) body['Estado'] = params.estado;
    }

    // 3. Bloque de Auditoría para Nuevos Registros (INSERT)
    if (accion === CrudAccion.Insert) {
      const utilidadService = this.injector.get(UtilidadService);
      body['Consecutivo_Interno'] = utilidadService.generarConsecutivoInterno();
      body['UserName'] = usuarioActivo;
      // ADAPTAR: Mapear campos adicionales requeridos por la base de datos (Empresa, Estación, etc.)
    }

    // 4. Bloque de Auditoría para Modificaciones (UPDATE / DELETE)
    if (accion === CrudAccion.Update || accion === CrudAccion.Delete) {
      body['M_UserName'] = usuarioActivo;
    }

    return body;
  }

  // ── Orquestador Genérico de Operaciones HTTP ──
  private ejecutarOperacion<T>(endpoint: string, accion: CrudAccion, params: RequestParamsEntidad): Observable<T> {
    const url = `${this.baseUrl}${endpoint}`;
    const payload = this.buildPayload(accion, params);
    return this.http.post<T>(url, payload);
  }

  // ── MÉTODOS PÚBLICOS (Los que consumirá tu componente) ──

  // SUSTITUIR: Nombra el método según tu contexto (ej: crudUsuarios, crudProductos)
  crudEntidadPrincipal<T = any>(accion: CrudAccion, params: RequestParamsEntidad = { accion } as any): Observable<T> {
    return this.ejecutarOperacion<T>(this.endpoints.principal, accion, params);
  }
}
```

### 4.3. Lógica del Componente (`mnt-version-app.component.ts`)
La lógica del componente debe manejar la transición entre el modo "Listado" y el modo "Formulario" mediante la propiedad `mostrarFormulario`. Se recomienda utilizar el enfoque _Template-Driven Forms_ (usando `[(ngModel)]`) para la vinculación de datos, tal como se plantea en la arquitectura base.

> **Componente `app-list-items` (List View):**
> Observa cómo en el componente TypeScript únicamente nos encargamos de obtener el arreglo de datos (`versionAppsData`) y de crear métodos para reaccionar a las interacciones (Editar/Eliminar) que dicho componente visual emitirá.

```typescript
@Component({
  selector: 'app-form-tu-entidad',
  templateUrl: './form-tu-entidad.component.html',
  styleUrl: './form-tu-entidad.component.css',
})
export class FormEntidadComponent implements OnInit {
  @Output() formularioCerrado = new EventEmitter<{ actualizado: boolean }>();

  // ── 1. Selectores y datos (ADAPTAR) ───────────────────────────────────────────
  listaDatos: TuEntidadModel[] = [];
  itemSeleccionado: TuEntidadModel | null = null;

  // ── 2. Control de Vistas ──────────────────────────────────────────────────────
  mostrarFormulario: boolean = false;
  accion: 'agregar' | 'actualizar' = 'agregar';

  // ── 3. Campos del formulario (ngModel) (ADAPTAR) ──────────────────────────────
  // Define aquí SOLO los campos necesarios para el CRUD de tu entidad
  campoEjemplo1: string = '';
  campoEjemplo2: string = '';
  estado: number = 1;

  // ── 4. Control de Errores y Modales (MANTENER) ────────────────────────────────
  erroresInputs: { [key: string]: string } = {};
  isVisibleModal: boolean = false;
  isVisibleModalEliminar: boolean = false;
  isVisibleAdvertencia: boolean = false;
  mensajeAdvertenciaM: string = '';
  isVisibleExito: boolean = false;
  mensajeExito: string = '';

  // ── 5. Estados de UI / Carga (MANTENER) ───────────────────────────────────────
  cargandoListado: boolean = false;
  cargandoGuardar: boolean = false;
  cargandoEliminar: boolean = false;

  constructor(
    private tuEntidadService: TuEntidadService, // ADAPTAR: Tu servicio
    private errorModalService: ErrorModalService,
    private utilidadService: UtilidadService,
  ) {}

  ngOnInit(): void {
    this.obtenerListado();
  }

  // ── GET: Cargar listado ───────────────────────────────────────────────────────
  obtenerListado() {
    this.cargandoListado = true;
    this.tuEntidadService
      .crudPrincipal(CrudAccion.Select) // ADAPTAR: Método de tu servicio
      .pipe(
        finalize(() => (this.cargandoListado = false)),
        catchError((err: ApiResponse<any>) => {
          this.errorModalService.mostrarError(err || {}, this.obtenerListado.bind(this));
          return of([] as TuEntidadModel[]);
        }),
      )
      .subscribe((data) => {
        this.listaDatos = data;
      });
  }

  // ── NAVEGACIÓN: Listado ↔ Formulario ──────────────────────────────────────────
  onCerrarFormulario(evento: { actualizado: boolean }) {
    this.mostrarFormulario = false;
    this.erroresInputs = {};
    this.formularioCerrado.emit(evento);
  }

  abrirFormulario(accion: 'agregar' | 'actualizar' = 'agregar') {
    this.accion = accion;
    if (accion === 'agregar') {
      // ADAPTAR: Limpiar tus campos a su estado inicial
      this.campoEjemplo1 = '';
      this.campoEjemplo2 = '';
      this.estado = 1;
      this.itemSeleccionado = null;
    }
    this.erroresInputs = {};
    this.mostrarFormulario = true;
  }

  onActualizar(item: TuEntidadModel) {
    this.itemSeleccionado = item;
    // ADAPTAR: Asignar los valores del registro seleccionado a las variables locales
    this.campoEjemplo1 = item.campoEjemplo1 ?? '';
    this.campoEjemplo2 = item.campoEjemplo2 ?? '';
    this.estado = item.estado ?? 1;
    
    this.abrirFormulario('actualizar');
  }

  onMostrarModalEliminar(item: TuEntidadModel) {
    this.itemSeleccionado = item;
    this.isVisibleModalEliminar = true;
  }

  // ── VALIDACIÓN ────────────────────────────────────────────────────────────────
  validarInputs(): boolean {
    this.erroresInputs = {};

    // ADAPTAR: Agrega tus reglas de validación obligatorias
    if (!this.campoEjemplo1.trim()) {
      this.erroresInputs['campoEjemplo1'] = 'El campo 1 es obligatorio.';
    }

    if (Object.keys(this.erroresInputs).length > 0) {
      this.mensajeAdvertenciaM = Object.values(this.erroresInputs).join(' ');
      this.isVisibleAdvertencia = true;
      setTimeout(() => (this.isVisibleAdvertencia = false), 7000);
      return false;
    }
    return true;
  }

  mostrarModalConfirmacion() {
    if (!this.validarInputs()) return;
    this.isVisibleModal = true;
  }

  // ── ORQUESTADOR: ADD / UPD ────────────────────────────────────────────────────
  guardarCambios() {
    this.isVisibleModal = false;
    this.cargandoGuardar = true;

    // ADAPTAR: Mapeo de parámetros genérico para Insertar o Actualizar
    const parametros = {
      accion: this.accion === 'agregar' ? CrudAccion.Insert : CrudAccion.Update,
      id: this.accion === 'actualizar' ? this.itemSeleccionado?.idTabla : undefined,
      idFieldName: 'IdTabla', // ADAPTAR: Nombre de tu PK en BD
      
      // ADAPTAR: Tus campos mapeados a la interfaz
      campoEjemplo1: this.campoEjemplo1,
      campoEjemplo2: this.campoEjemplo2,
      estado: this.estado,
    };

    this.tuEntidadService.crudPrincipal(parametros.accion, parametros).subscribe({
      next: (data: any) => {
        const respuesta = Array.isArray(data) ? data[0] : data;
        this.cargandoGuardar = false;
        this.manejarExito(respuesta?.mensaje ?? 'Registro guardado correctamente.');
        this.obtenerListado();
        this.onCerrarFormulario({ actualizado: true });
      },
      error: (err: ApiResponse<any>) => {
        this.cargandoGuardar = false;
        this.errorModalService.mostrarError(err || {});
      },
    });
  }

  // ── DEL: Eliminar ─────────────────────────────────────────────────────────────
  confirmarEliminar() {
    if (!this.itemSeleccionado?.idTabla) return; // ADAPTAR: Tu PK

    this.cargandoEliminar = true;
    
    // ADAPTAR: Parámetros exclusivos para Delete
    const parametros = {
      accion: CrudAccion.Delete,
      id: this.itemSeleccionado.idTabla,
      idFieldName: 'IdTabla',
    };

    this.tuEntidadService.crudPrincipal(CrudAccion.Delete, parametros).subscribe({
      next: (data: any) => {
        const respuesta = Array.isArray(data) ? data[0] : data;
        this.cargandoEliminar = false;
        this.isVisibleModalEliminar = false;
        this.manejarExito(respuesta?.mensaje ?? 'Registro eliminado correctamente.');
        this.obtenerListado();
      },
      error: (err: ApiResponse<any>) => {
        this.cargandoEliminar = false;
        this.isVisibleModalEliminar = false;
        this.errorModalService.mostrarError(err || {});
      },
    });
  }
}
```
### 4.4. Estructura HTML y Tailwind CSS (mnt-version-app.component.html)
El HTML se divide estrictamente en dos bloques controlados por *ngIf="mostrarFormulario". El bloque inicial es el encargado de instanciar la lista centralizada, mientras que el segundo construye el formulario utilizando las clases utilitarias de Tailwind CSS.
> **Componente `app-list-items` (List View):** Este componente encapsula toda la complejidad de generar una tabla iterativa, un buscador y la paginación. Nosotros solo debemos alimentar las propiedades estandarizadas (campoTitulo1, camposSub, etc.) e inyectarle un fragmento de código HTML (<ng-template>) con los botones de acción que deseamos que tenga cada fila.

```html
<!-- ─── 1. VISTA DE LISTADO (Aparece cuando mostrarFormulario es false) ────────── -->
<div *ngIf="!mostrarFormulario" class="w-full">
  
  <div class="mb-4 flex flex-col sm:flex-row items-start sm:items-center justify-end gap-3">
    <button
      (click)="abrirFormulario('agregar')"
      class="flex items-center px-4 py-2 color-primary rounded-lg shadow-md hover:shadow-lg transition-all duration-200"
      title="Agregar nuevo registro"
    >
      <i class="fas fa-plus mr-2"></i> Agregar
    </button>
  </div>
<!-- ─── LISTADO  ─── -->
  <app-list-items
    *ngIf="!cargandoListado"
    [datos]="listaDatos"
    campoTitulo1="campoEjemplo1"
    campoTitulo2="campoEjemplo2"
    [camposSub]="['estado']"
    [templateAcciones]="accionesEntidad"
    [mostrarBuscador]="true"
    [habilitarDetalle]="true"
    tituloModalDetalle="Detalle del Registro"
  >
  </app-list-items>

<!-- ─── TEMPLATE DE ACCIONES PARA EL LISTADO  ─── -->
  <ng-template #accionesEntidad let-item>
    
    <button
      (click)="onActualizar(item)"
      class="flex items-center justify-center w-10 h-10 bg-blue-100 hover:bg-blue-200 text-blue-700 rounded-full"
      title="Editar"
    >
      <i class="fas fa-edit"></i>
    </button>

    <button
      (click)="onMostrarModalEliminar(item)"
      class="flex items-center justify-center w-10 h-10 bg-red-100 hover:bg-red-200 text-red-700 rounded-full"
      title="Eliminar"
    >
      <i class="fas fa-trash"></i>
    </button>
  </ng-template>

</div>

<!-- ─── FORMULARIO PARA LOS CAMPOS (CRUD)  ─── -->
<div *ngIf="mostrarFormulario" class="mt-4">
  
  <div class="sticky top-0 bg-gray-100 dark:bg-gray-700 border-b border-gray-200 p-4 mb-6 z-10">
    <div class="flex items-center justify-between">
      
      <button type="button" (click)="onCerrarFormulario({ actualizado: false })" class="text-gray-600 hover:text-gray-800 font-medium">
        <i class="fas fa-arrow-left mr-2"></i> Regresar
      </button>

      <h3 class="text-xl font-semibold text-gray-900 text-center flex-1">
        {{ accion === 'agregar' ? 'Agregar nuevo registro' : 'Actualizar registro' }}
      </h3>

      <button type="submit" (click)="mostrarModalConfirmacion()" class="text-blue-600 hover:text-blue-700 font-medium">
        Guardar <i class="fas fa-save ml-2"></i>
      </button>

    </div>
  </div>

  <div class="max-h-[calc(100vh-120px)] overflow-y-auto p-6 grid grid-cols-1 sm:grid-cols-2 gap-6">
    
    <div class="flex flex-col gap-4">
      
      <div class="flex flex-col">
        <label class="mb-1 font-medium text-sm text-gray-700">
          Campo Ejemplo 1 <span *ngIf="accion === 'agregar'" class="text-red-500 font-bold">*</span>
        </label>
        <input
          type="text"
          [(ngModel)]="campoEjemplo1"
          name="campoEjemplo1"
          placeholder="Ej. Valor del campo"
          class="mt-1 block w-full pl-3 pr-3 py-2 rounded-md border border-gray-300 shadow-sm focus:ring-2 focus:ring-blue-500"
        />
      </div>

    </div>

    <div class="flex flex-col gap-4">

      <div class="flex flex-col">
        <label class="mb-1 font-medium text-sm text-gray-700">Campo Ejemplo 2</label>
        <input
          type="text"
          [(ngModel)]="campoEjemplo2"
          name="campoEjemplo2"
          class="mt-1 block w-full pl-3 pr-3 py-2 rounded-md border border-gray-300 shadow-sm"
        />
      </div>
      
      <div class="flex flex-col mt-4">
        <label class="mb-1 font-medium text-sm text-gray-700">Estado</label>
        <select
          [(ngModel)]="estado"
          name="estado"
          class="mt-1 block w-full pl-3 pr-3 py-2 rounded-md border border-gray-300 shadow-sm bg-white"
        >
          <option [ngValue]="1">Activo</option>
          <option [ngValue]="0">Inactivo</option>
        </select>
      </div>

    </div>

  </div>
</div>
```
# Documentación del Componente Estandarizado: `app-list-items`

El componente `<app-list-items>` es indispensable para la visualización de datos en los módulos de mantenimiento. Su diseño se enfoca en la reutilización, delegando la responsabilidad visual y estructural (Tailwind CSS) al componente padre, mientras que maneja internamente funcionalidades complejas como:
*   Renderizado responsivo en formato de tarjetas o listas.
*   Buscador interactivo en tiempo real integrado.
*   Inyección de templates dinámicos para acciones personalizadas.

---

## 1. Propiedades de Configuración (Inputs)

Para implementar el componente, es necesario conocer las propiedades que recibe para adaptar la vista a cualquier tabla/entidad de la base de datos:

*   **`[datos]`**: `any[]` - Arreglo principal de los datos a renderizar.
*   **`campoTitulo1`**: `string` - Llave del JSON a mostrar como el título principal de la fila (en negrita). *Obligatorio*.
*   **`campoTitulo2`**: `string` - Llave del JSON a mostrar como subtítulo secundario. *Opcional*.
*   **`[camposSub]`**: `string[]` - Arreglo con los nombres de las propiedades extra que se mostrarán apiladas bajo los títulos.
*   **`[mostrarBuscador]`**: `boolean` - Habilita una barra de búsqueda local que filtra el listado analizando los campos pasados en títulos y sub-campos.

### Plantillas Inyectables (`<ng-template>`)
*   **`[templateAcciones]`**: Recibe un template de Angular con los botones que dictan las acciones (Editar, Eliminar, etc.).
*   **`[templateInfo]`**: Recibe un template opcional para renderizar badges (etiquetas de estado) junto a la información principal.

---

## 2. Código de Referencia Interno del Componente

A continuación se expone la base lógica de cómo opera este componente genérico, útil para que el equipo entienda su funcionamiento interno.

### Lógica TypeScript (`list-items.component.ts`)
```typescript
@Component({
  selector: 'app-list-items',
  templateUrl: './list-items.component.html'
})
export class ListItemsComponent {
  // Arreglo de registros a mostrar
  @Input() datos: any[] = [];

  // Mapeo dinámico de campos
  @Input() campoTitulo1: string = '';
  @Input() campoTitulo2?: string;
  @Input() camposSub: string[] = [];

  // Templates inyectables
  @Input() templateAcciones: TemplateRef<any> | null = null;
  @Input() templateInfo: TemplateRef<any> | null = null;

  // Funcionalidades extras
  @Input() mostrarBuscador: boolean = false;
  terminoBusqueda: string = '';


  // Getter inteligente para el buscador: Filtra solo entre los campos mapeados
  get datosFiltrados(): any[] {
    if (!this.terminoBusqueda || this.terminoBusqueda.trim() === '') {
      return this.datos;
    }
    const term = this.terminoBusqueda.toLowerCase().trim();
    
    return this.datos.filter(item => {
      const camposAFiltrar = [this.campoTitulo1];
      if (this.campoTitulo2) camposAFiltrar.push(this.campoTitulo2);
      if (this.camposSub && this.camposSub.length > 0) camposAFiltrar.push(...this.camposSub);
      
      return camposAFiltrar.some(campo => {
         const valor = item[campo];
         return valor != null && String(valor).toLowerCase().includes(term);
      });
    });
  }
}
```
### Template HTML (`list-items.component.html`)
```html
<div class="flex flex-col gap-3">
  <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-2 mb-1">
    <!-- Buscador Discreto -->
    <div *ngIf="mostrarBuscador" class="flex justify-start">
      <div class="relative w-full sm:w-64">
        <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
          <svg class="w-4 h-4 text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/>
          </svg>
        </div>
        <input 
          type="text" 
          [(ngModel)]="terminoBusqueda"
          class="bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 text-gray-700 dark:text-gray-200 text-sm rounded-md focus:ring-blue-500 focus:border-blue-500 block w-full pl-9 pr-3 py-1.5 shadow-sm transition-all outline-none" 
          placeholder="Buscar..." 
        />
      </div>
    </div>

    <!-- Contador de registros -->
    <div *ngIf="datos" class="flex justify-start sm:justify-end text-xs text-gray-500 dark:text-gray-400">
      <span class="bg-gray-100 dark:bg-gray-800 border border-gray-200 dark:border-gray-700 px-2.5 py-1.5 rounded-md font-medium shadow-sm flex items-center">
        <svg class="w-3.5 h-3.5 mr-1.5 text-indigo-500" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 10h16M4 14h16M4 18h16" />
        </svg>
        Mostrando {{ datosFiltrados.length }} <ng-container *ngIf="datosFiltrados.length !== datos.length">de {{ datos.length }}</ng-container> registro(s)
      </span>
    </div>
  </div>

  <div class="max-h-[530px] overflow-y-auto">
    <ul class="space-y-3">
      <li
        *ngFor="let item of datosFiltrados"
        class="flex flex-col md:flex-row justify-between p-4 rounded-xl bg-white dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700 shadow-sm hover:shadow-md transition-all duration-200"
      >
        <!-- Contenido principal -->
        <div class="flex-1 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-3 md:gap-4">
          
          <!-- Título 1 -->
          <div class="flex flex-col" *ngIf="campoTitulo1">
            <span class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-1" [title]="campoTitulo1 | formatLabel">{{ campoTitulo1 | formatLabel }}</span>
            <div class="flex items-center">
              <div class="w-2 h-2 rounded-full bg-indigo-500 mr-2"></div>
              <span class="font-medium text-gray-800 dark:text-gray-100 truncate" [title]="item[campoTitulo1]">
                {{ item[campoTitulo1] || '—' }}
              </span>
            </div>
          </div>

          <!-- Título 2 -->
          <div class="flex flex-col" *ngIf="campoTitulo2">
            <span class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-1" [title]="campoTitulo2 | formatLabel">{{ campoTitulo2 | formatLabel }}</span>
            <div class="flex items-center">
              <div class="w-2 h-2 rounded-full bg-indigo-500 mr-2"></div>
              <span class="text-gray-700 dark:text-gray-200 truncate" [title]="item[campoTitulo2]">
                {{ item[campoTitulo2]  || '—'}}
              </span>
            </div>
          </div>

          <!-- Campos extras (sub1, sub2, subN) -->
          <ng-container *ngFor="let campo of camposSub">
            <div class="flex flex-col">
              <span class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-1" [title]="campo | formatLabel">{{ campo | formatLabel }}</span>
              <span class="text-gray-700 dark:text-gray-200 truncate" [title]="item[campo]">
                {{ item[campo]  || '—'}}
              </span>
            </div>
          </ng-container>

          <!-- Info extra con template -->
          <ng-container *ngIf="templateInfo">
            <ng-container *ngTemplateOutlet="templateInfo; context: { $implicit: item }"></ng-container>
          </ng-container>

        </div>

        <!-- Acciones -->
        <div class="flex justify-end md:justify-start items-center space-x-2 mt-4 md:mt-0 md:ml-4 md:pl-4 md:border-l md:border-gray-200 dark:md:border-gray-700">
          
          <!-- ACA PUEDEN IR MAS BOTONES INTERNOS DE ACCIONES (ESTOS APARECERAN SIEMPRE DENTRO DE ESTE COMPONENTE) -->

          <ng-container *ngTemplateOutlet="templateAcciones; context: { $implicit: item }"></ng-container>
        </div>
      </li>

      <!-- Mensaje cuando no hay resultados en la búsqueda -->
      <li *ngIf="datosFiltrados.length === 0 && datos.length > 0" class="p-8 text-center text-gray-500 dark:text-gray-400 bg-gray-50/50 dark:bg-gray-800/30 rounded-xl border border-dashed border-gray-200 dark:border-gray-700">
        <svg class="mx-auto h-8 w-8 text-gray-400 mb-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
        </svg>
        No se encontraron resultados para "{{ terminoBusqueda }}"
      </li>

    </ul>
  </div>
</div>
```
