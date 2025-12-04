# Sistema de Caché para Tablas de Análisis

## Problema Resuelto
Las tablas del modal "Ver Análisis" tardaban mucho en generarse cada vez que se abría el modal, especialmente cuando se accedía desde GitHub Pages. Dado que los datos del GeoJSON son estables (se actualizan mensualmente), no es necesario recalcular las tablas en cada apertura.

## Solución Implementada

### Variables de Caché
```javascript
var tablaCiudadanosCache = null;  // HTML de la tabla "Por Ciudadano"
var tablaTiposCache = null;        // HTML de la tabla "Por Tipo de Solicitud"
var solicitudesCount = 0;          // Contador para detectar cambios en datos
```

### Funcionamiento

1. **Primera apertura del modal:**
   - Se calcula `solicitudesCount = solicitudes.length`
   - Se generan ambas tablas (ciudadanos y tipos)
   - Se guarda el HTML en `tablaCiudadanosCache` y `tablaTiposCache`
   - Se muestran los resultados al usuario

2. **Aperturas subsecuentes (mismo mes):**
   - Se compara `solicitudesCount` con `solicitudes.length`
   - Si son iguales → datos no han cambiado
   - Se usa el HTML cacheado directamente (sin recálculo)
   - **Carga instantánea** sin overlay de loading

3. **Cuando cambian los datos (nuevo mes):**
   - `solicitudes.length` es diferente a `solicitudesCount`
   - Se invalida el caché (`tablaCiudadanosCache = null`)
   - Se recalculan las tablas
   - Se actualiza `solicitudesCount`
   - Se guarda el nuevo HTML en caché

### Lazy Loading
La tabla "Por Tipo de Solicitud" solo se genera cuando el usuario hace clic en esa pestaña por primera vez. Esto reduce el tiempo de carga inicial del modal.

### Funciones Modificadas

#### `generarTablaCiudadanos(forzarRecalculo = false)`
```javascript
// Inicio de la función - verifica caché
if (!forzarRecalculo && tablaCiudadanosCache && solicitudesCount === solicitudes.length) {
  document.getElementById('contenidoTablaCiudadanos').innerHTML = tablaCiudadanosCache;
  adjuntarListenersCiudadanos();
  return; // Salida temprana - no recalcula
}

// ... lógica de generación ...

// Final de la función - guarda caché
tablaCiudadanosCache = html;
document.getElementById('contenidoTablaCiudadanos').innerHTML = html;
adjuntarListenersCiudadanos();
```

#### `generarTablaTipos(forzarRecalculo = false)`
Misma lógica que `generarTablaCiudadanos()`, pero para la tabla inversa.

#### Event Listener del Botón "Ver Análisis"
```javascript
btnVerCiudadanos.addEventListener('click', () => {
  // Detectar cambios en datos
  const datosHanCambiado = solicitudesCount !== solicitudes.length;
  
  if (datosHanCambiado) {
    solicitudesCount = solicitudes.length;
    tablaCiudadanosCache = null;
    tablaTiposCache = null;
  }
  
  // Carga instantánea si hay caché válido
  if (tablaCiudadanosCache && !datosHanCambiado) {
    modalCiudadanos.style.display = 'block';
    generarTablaCiudadanos(); // Usa caché interno
  } else {
    showLoading();
    modalCiudadanos.style.display = 'block';
    setTimeout(() => {
      generarTablaCiudadanos();
      hideLoading();
    }, 50);
  }
});
```

#### Event Listener de Cambio de Pestaña
```javascript
// Lazy loading para tabla de tipos
if (targetTab === 'tabTipos' && !tablaTiposCache) {
  showLoading();
  setTimeout(() => {
    generarTablaTipos();
    hideLoading();
  }, 50);
}
```

## Beneficios

1. **Rendimiento:** Carga instantánea en aperturas subsecuentes del modal
2. **Experiencia de usuario:** No más esperas después de la primera carga
3. **Eficiencia:** Solo recalcula cuando los datos realmente cambian
4. **Inteligente:** Detecta automáticamente actualizaciones mensuales del GeoJSON
5. **Lazy loading:** La segunda pestaña solo se genera cuando se necesita

## Ciclo de Actualización de Datos

El sistema está diseñado para el ciclo mensual de actualización de datos:

1. **Diciembre 2025:** Usuario abre análisis → se generan tablas → se cachean
2. **Durante diciembre:** Usuario abre análisis 10 veces más → usa caché (instantáneo)
3. **Enero 2026:** Se actualiza archivo a `Enero2026.geojson`
4. **Primera apertura en enero:** Sistema detecta cambio → invalida caché → regenera tablas
5. **Durante enero:** Usuario abre análisis múltiples veces → usa nuevo caché

## Consideraciones Técnicas

- El caché está en memoria (variables JavaScript)
- Se pierde al recargar la página (comportamiento esperado)
- No ocupa espacio en localStorage o IndexedDB
- Es seguro porque los datos son de solo lectura
- La comparación `solicitudes.length` es suficiente para detectar cambios de archivo
