# 20 Problemas Críticos de Diseño en PAT

Tras analizar el código proporcionado, he identificado varios problemas arquitectónicos y de diseño significativos:

# Problema #1: Uso de Task.Delay Arbitrario para Sincronización Temporal

## Descripción del Problema

En la solución PAT se identificó un patrón problemático donde se utilizan tiempos de espera fijos arbitrarios mediante `Task.Delay` como mecanismo de sincronización, en lugar de implementar soluciones basadas en eventos o notificaciones.

### Código Problemático

```csharp
if (ViewModel.UserAuthorizationFailed && firstRender)
{
    await Task.Delay(TimeSpan.FromSeconds(5));
    await JSRuntime.InvokeVoidAsync("submitForm", "logoutForm");
}
```
**Archivo:** `~\Compass\Compass.BlazorComponents\Pages\Auth\AuthFailureHandler.razor.cs`

## Consecuencias

1. **Bloqueo del hilo de UI**: Espera 5 segundos bloqueando la interfaz sin feedback al usuario
2. **Tiempo fijo arbitrario**: No se adapta a las condiciones reales de operación
3. **Mala experiencia de usuario**: No hay indicación visual de lo que está sucediendo
4. **Acoplamiento temporal**: Asume que siempre 5 segundos es el tiempo adecuado para cualquier situación
5. **Falta de manejo de errores**: No verifica si la operación de logout fue exitosa

## Solución Recomendada

```csharp
if (ViewModel.UserAuthorizationFailed && firstRender)
{
    // Notificar al usuario inmediatamente
    await NotificationService.ShowWarningAsync("Autorización fallida. Redirigiendo al login...");
    
    // La tarea que realmente maneja la redirección
    var redirectTask = Task.Run(async () => {
        await Task.Delay(1000); // Tiempo para que el usuario lea el mensaje
        
        // Código real de preparación para logout (si es necesario)
        await PrepareForLogout(); // Esto podría ser guardar estado, limpiar sesión, etc.
        
        // Ejecutar la redirección (esto debería estar dentro de la tarea)
        await JSRuntime.InvokeVoidAsync("submitForm", "logoutForm");
        return true;
    });
    
    // Solo esperamos hasta 3 segundos como máximo
    // Si la tarea no se completa en ese tiempo, procedemos con la redirección forzada
    if (await Task.WhenAny(redirectTask, Task.Delay(3000)) != redirectTask)
    {
        // Si llegamos aquí es porque se agotó el tiempo de espera
        // Forzamos la redirección como plan B
        await JSRuntime.InvokeVoidAsync("submitForm", "logoutForm");
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar retroalimentación inmediata al usuario sobre lo que está sucediendo
- Usar Task.WhenAny para manejar adecuadamente tiempos de espera sin bloquear la UI
- Separar las preocupaciones de notificación y navegación
- Mejorar la experiencia de usuario con información clara del proceso
- Permitir que el proceso se complete tan pronto como esté listo, sin esperar un tiempo fijo

## Conclusión

El uso de Task.Delay con tiempos fijos es un antipatrón que debe evitarse en favor de mecanismos basados en eventos, callbacks o notificaciones asíncronas. En los casos donde sea inevitable esperar, se debe proporcionar feedback adecuado al usuario y utilizar técnicas como Task.WhenAny para manejar timeouts de manera elegante.

# Problema #2: Carga Excesiva en Caché sin Paginación

## Descripción del Problema

En la solución PAT se identificó un antipatrón donde se cargan conjuntos completos de datos en memoria caché, en lugar de implementar paginación para cargar solo los datos necesarios para la vista actual.

### Código Problemático

```csharp
public async Task<IEnumerable<OperationDto>> GetAllOperationsForCompany(Guid companyId)
{
    // Carga TODAS las operaciones de la compañía en memoria
    var operations = await _repository.GetAllOperationsForCompany(companyId);
    
    // Las almacena en caché con una clave global
    _cache.Set($"company_{companyId}_operations", operations, TimeSpan.FromMinutes(30));
    
    return operations;
}
```
**Archivo:** `~\Compass\Compass.Operations.Domain\Services\OperationService.cs`

## Consecuencias

1. **Consumo excesivo de memoria**: Cargar todos los registros en memoria puede agotar los recursos del servidor
2. **Tiempo de respuesta inicial lento**: La primera carga debe procesar todos los datos, no solo los visibles
3. **Ineficiencia en consultas**: La base de datos tiene que recuperar y transferir registros que no serán mostrados
4. **Problemas de escalabilidad**: El sistema no podrá manejar volúmenes de datos crecientes
5. **Experiencia de usuario deficiente**: Tiempos de carga largos y posible congelamiento de la interfaz

## Solución Recomendada

```csharp
public async Task<PagedResult<OperationDto>> GetOperationsForCompanyPaged(
    Guid companyId, 
    int page, 
    int pageSize,
    OperationsFilterCriteria filters)
{
    // Solo carga la página solicitada con filtros aplicados en la consulta
    var pagedOperations = await _repository.GetOperationsForCompanyPaged(
        companyId, 
        page, 
        pageSize, 
        filters
    );
    
    // Guarda en caché solo la página actual con una clave específica
    string cacheKey = $"company_{companyId}_operations_page{page}_size{pageSize}_{filters.GetHashCode()}";
    _cache.Set(cacheKey, pagedOperations, TimeSpan.FromMinutes(15));
    
    return pagedOperations;
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Reducir drásticamente el consumo de memoria al cargar solo los datos necesarios
- Mejorar los tiempos de respuesta al procesar conjuntos de datos más pequeños
- Optimizar las consultas a la base de datos aplicando filtros y paginación en el origen
- Permitir el escalamiento del sistema para manejar grandes volúmenes de datos
- Mejorar la experiencia de usuario con tiempos de carga más cortos

## Conclusión

La implementación de paginación es fundamental en aplicaciones empresariales que manejan grandes volúmenes de datos. Cargar todo en caché es una práctica desaconsejada que impacta negativamente el rendimiento, la escalabilidad y la experiencia del usuario. La paginación del lado del servidor, combinada con una estrategia de caché adecuada, ofrece la mejor solución para este problema.

# Problema #3: Violación del Principio de Responsabilidad Única (SRP)

## Descripción del Problema

En la solución PAT se encontraron clases que violan el Principio de Responsabilidad Única (SRP) al contener múltiples responsabilidades no relacionadas, como la clase `DocumentValidationService` que implementa diferentes tipos de validaciones en un solo componente.

### Código Problemático

```csharp
public class DocumentValidationService : IDocumentValidationService
{
    private readonly IDocumentRepository documentRepository;
    private readonly ICompaniesService companiesService;
    private readonly ICreditLinesService creditLinesService;
    private readonly ICompaniesAffiliationService companiesAffiliationService;
    private readonly ICurrencyExchangeService currencyExchangeService;
    private readonly IDateProvider dateProvider;
    
    public async Task<ValidationResult> ValidateDocument(Document document)
    {
        // Validación de fechas
        if (document.ExpirationDate < dateProvider.Now())
            return ValidationResult.Invalid("El documento ha expirado");
            
        // Validación de montos
        if (document.Amount <= 0)
            return ValidationResult.Invalid("El monto debe ser positivo");
            
        // Validación de tipo de documento
        if (!IsValidDocumentType(document.Type))
            return ValidationResult.Invalid("Tipo de documento no válido");
            
        // Validación de emisor
        if (string.IsNullOrEmpty(document.Issuer))
            return ValidationResult.Invalid("El emisor es requerido");
            
        // Validación de duplicados en base de datos
        if (await documentRepository.ExistsWithSameNumber(document.Number))
            return ValidationResult.Invalid("Ya existe un documento con el mismo número");
            
        // Más validaciones...
        
        return ValidationResult.Valid();
    }
    
    // Más métodos de validación...
}
```
**Archivo:** `~\Compass\Compass.Core.Domain\Services\DocumentValidationService.cs`

## Consecuencias

1. **Código difícil de mantener**: Cambios en una validación pueden afectar a otras
2. **Alta complejidad ciclomática**: Muchas ramas y condiciones en un solo método
3. **Dificultad para pruebas unitarias**: Probar una validación específica requiere configurar todas las dependencias
4. **Alto acoplamiento**: La clase depende de múltiples servicios externos
5. **Falta de cohesión**: Las diferentes validaciones no están relacionadas intrínsecamente entre sí

## Solución Recomendada

```csharp
// Definir una interfaz común para validadores
public interface IDocumentValidator
{
    Task<ValidationResult> Validate(Document document);
}

// Implementar validadores específicos con responsabilidad única
public class DocumentExpirationValidator : IDocumentValidator
{
    private readonly IDateProvider _dateProvider;
    
    public DocumentExpirationValidator(IDateProvider dateProvider)
    {
        _dateProvider = dateProvider;
    }
    
    public async Task<ValidationResult> Validate(Document document)
    {
        if (document.ExpirationDate < _dateProvider.Now())
            return ValidationResult.Invalid("El documento ha expirado");
        
        return ValidationResult.Valid();
    }
}

public class DocumentAmountValidator : IDocumentValidator
{
    public async Task<ValidationResult> Validate(Document document)
    {
        if (document.Amount <= 0)
            return ValidationResult.Invalid("El monto debe ser positivo");
        
        return ValidationResult.Valid();
    }
}

public class DocumentDuplicateValidator : IDocumentValidator
{
    private readonly IDocumentRepository _documentRepository;
    
    public DocumentDuplicateValidator(IDocumentRepository documentRepository)
    {
        _documentRepository = documentRepository;
    }
    
    public async Task<ValidationResult> Validate(Document document)
    {
        if (await _documentRepository.ExistsWithSameNumber(document.Number))
            return ValidationResult.Invalid("Ya existe un documento con el mismo número");
        
        return ValidationResult.Valid();
    }
}

// Compositor que ejecuta todos los validadores
public class CompositeDocumentValidator : IDocumentValidationService
{
    private readonly IEnumerable<IDocumentValidator> _validators;
    
    public CompositeDocumentValidator(IEnumerable<IDocumentValidator> validators)
    {
        _validators = validators;
    }
    
    public async Task<ValidationResult> ValidateDocument(Document document)
    {
        foreach (var validator in _validators)
        {
            var result = await validator.Validate(document);
            if (!result.IsValid)
                return result;
        }
        
        return ValidationResult.Valid();
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Asignar una única responsabilidad a cada clase validadora
- Facilitar el mantenimiento al aislar los cambios dentro de validadores específicos
- Simplificar las pruebas unitarias al poder probar cada validador por separado
- Reducir el acoplamiento al limitar las dependencias de cada validador
- Mejorar la cohesión al agrupar lógica relacionada
- Permitir la extensión del sistema añadiendo nuevos validadores sin modificar el código existente

## Conclusión

La implementación del Principio de Responsabilidad Única mediante el patrón Composite y Chain of Responsibility permite una arquitectura más limpia, mantenible y extensible. Cada validador se enfoca en una única tarea, facilitando las pruebas unitarias y permitiendo la reutilización de componentes. Este enfoque reduce la complejidad y mejora la calidad del código a largo plazo.

# Problema #4: Lógica de Negocio en ViewModels

## Descripción del Problema

En la solución PAT se encontró que los ViewModels contienen lógica de negocio compleja que debería estar en la capa de dominio, violando el principio de separación de responsabilidades y creando acoplamiento entre la presentación y la lógica de negocio.

### Código Problemático

```csharp
public class OperationListViewModel
{
    private readonly IBankInstitutionsService _bankInstitutionsService;
    private readonly Action<string> errorFunc;
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        // Validación de negocio en ViewModel
        if (itemsToAdvance.Any(ita => ita.EmissaryBankId() == BANK_NOT_SELECTED))
        {
            this.errorFunc.Invoke("Banco emisor no seleccionado para uno o más elementos");
            return;
        }
        
        // Cálculo de montos y tasas en ViewModel
        var totalAmount = itemsToAdvance.Sum(i => i.Amount);
        var effectiveRate = CalculateEffectiveRate(itemsToAdvance);
        
        // Reglas de negocio para determinar fechas
        var paymentDate = DeterminePaymentDate(itemsToAdvance);
        
        // Más lógica de negocio...
        await _itemToAdvanceHandler.ApplyAdvance(itemsToAdvance, totalAmount, effectiveRate, paymentDate);
    }
    
    private decimal CalculateEffectiveRate(IEnumerable<ItemToAdvanceRow> items)
    {
        // Lógica compleja de cálculo de tasa efectiva que debería estar en la capa de dominio
        // ...
        return calculatedRate;
    }
    
    private DateTime DeterminePaymentDate(IEnumerable<ItemToAdvanceRow> items)
    {
        // Lógica de negocio para determinar fechas de pago
        // ...
        return paymentDate;
    }
}
```
**Archivo:** `~\Compass\Compass.Confirming.BlazorComponents\Pages\OperationList\OperationListViewModel.cs`

## Consecuencias

1. **Violación de Separación de Responsabilidades**: Mezcla lógica de presentación con lógica de negocio
2. **Duplicación de código**: La misma lógica de negocio puede estar duplicada en múltiples ViewModels
3. **Dificultad de prueba**: La lógica de negocio es difícil de probar sin instanciar toda la UI
4. **Acoplamiento fuerte**: Cambios en la lógica de negocio requieren modificar componentes de UI
5. **Impedimento para reutilización**: La lógica de negocio no puede reutilizarse en otros contextos

## Solución Recomendada

```csharp
// Capa de dominio - Servicios de negocio
public class AdvanceService : IAdvanceService
{
    // Dependencias inyectadas
    
    public ValidationResult ValidateItemsForAdvance(IEnumerable<ItemToAdvance> items)
    {
        if (items.Any(item => item.EmissaryBankId == Guid.Empty))
            return ValidationResult.Invalid("Banco emisor no seleccionado para uno o más elementos");
        
        return ValidationResult.Valid();
    }
    
    public decimal CalculateEffectiveRate(IEnumerable<ItemToAdvance> items)
    {
        // Lógica compleja de cálculo de tasa efectiva
        // ...
        return calculatedRate;
    }
    
    public DateTime DeterminePaymentDate(IEnumerable<ItemToAdvance> items)
    {
        // Lógica de negocio para determinar fechas de pago
        // ...
        return paymentDate;
    }
    
    public async Task<AdvanceResult> ApplyAdvance(IEnumerable<ItemToAdvance> items)
    {
        var validationResult = ValidateItemsForAdvance(items);
        if (!validationResult.IsValid)
            return AdvanceResult.Failure(validationResult.Error);
            
        var totalAmount = items.Sum(i => i.Amount);
        var effectiveRate = CalculateEffectiveRate(items);
        var paymentDate = DeterminePaymentDate(items);
        
        // Aplicar el adelanto
        // ...
        
        return AdvanceResult.Success("Items procesados correctamente", totalAmount);
    }
}

// Capa de presentación - ViewModel limpio
public class OperationListViewModel
{
    private readonly IAdvanceService _advanceService;
    private readonly Action<string> _errorMessageHandler;
    
    public OperationListViewModel(IAdvanceService advanceService, Action<string> errorMessageHandler)
    {
        _advanceService = advanceService;
        _errorMessageHandler = errorMessageHandler;
    }
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvanceRows)
    {
        // Mapear de DTO de UI a entidad de dominio
        var itemsToAdvance = itemsToAdvanceRows.Select(r => new ItemToAdvance(r.Id, r.Amount, r.EmissaryBankId()));
        
        // Ejecutar lógica de negocio
        var result = await _advanceService.ApplyAdvance(itemsToAdvance);
        
        // Manejar resultado
        if (!result.Success)
        {
            _errorMessageHandler(result.ErrorMessage);
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Separar claramente las responsabilidades entre capas de presentación y dominio
- Centralizar la lógica de negocio en servicios específicos de la capa de dominio
- Facilitar las pruebas unitarias de la lógica de negocio sin dependencias de UI
- Permitir la reutilización de la lógica de negocio en diferentes contextos
- Reducir la duplicación de código al extraer lógica común a servicios compartidos
- Mejorar la mantenibilidad al aislar los cambios de lógica de negocio

## Conclusión

Mantener una clara separación entre la lógica de negocio y la presentación es fundamental para una arquitectura limpia. Los ViewModels deben enfocarse exclusivamente en coordinar la UI y los servicios de dominio, delegando toda la lógica de negocio a la capa de dominio. Este enfoque mejora la mantenibilidad, testabilidad y reutilización del código.

# Problema #5: Acoplamientos Temporales

## Descripción del Problema

En la solución PAT se identificaron varios acoplamientos temporales, donde el código asume que ciertas operaciones se completan en un tiempo fijo o se basa en el orden específico de ejecución sin garantías adecuadas.

### Código Problemático

```csharp
public async Task ProcessAdvance()
{
    // Envía comando a través del bus
    await _commandBus.SendAsync(new AdvanceItemsCommand(itemsToAdvance));
    
    // Asume que el procesamiento tarda exactamente 1 segundo
    await Task.Delay(1000);
    
    // Consulta resultados sin garantía de que estén listos
    var result = await _repository.GetAdvanceResult(itemsToAdvance.First().Id);
    
    // Actualización de UI que depende del resultado anterior
    UpdateUIWithAdvanceResult(result);
}

private void UpdateAdvanceStatus(ItemToAdvance itemToAdvance)
{
    if (itemToAdvance.AdvanceStatus = $"{(short)AdvanceStatus.Launched}") // This is shitty... but I can't find a cleaner solution
    {
        // Lógica que depende del estado...
    }
}
```
**Archivo:** `~\Compass\Compass.AdvanceProcessing.BlazorComponents\ViewModels\AdvanceProcessingViewModel.cs`

## Consecuencias

1. **Comportamiento impredecible**: No hay garantía de que las operaciones asíncronas se completen en el tiempo asumido
2. **Condiciones de carrera**: El código puede intentar acceder a datos que aún no están disponibles
3. **Fragilidad**: Cambios en rendimiento del sistema pueden romper la funcionalidad
4. **Mala experiencia de usuario**: Pueden mostrarse datos incompletos o desactualizados
5. **Errores difíciles de depurar**: Los problemas aparecen intermitentemente y son difíciles de reproducir

## Solución Recomendada

```csharp
public async Task ProcessAdvance()
{
    // Mostrar indicador de procesamiento
    ShowProcessingIndicator();
    
    try {
        // Genera un ID de correlación para seguimiento
        var correlationId = Guid.NewGuid();
        
        // Registra callback para actualización de UI
        _advanceNotificationService.RegisterForNotification(correlationId, result => {
            // Esta función se llama cuando el proceso termina
            UpdateUIWithAdvanceResult(result);
            HideProcessingIndicator();
        });
        
        // Envía comando con el ID de correlación
        await _commandBus.SendAsync(new AdvanceItemsCommand(itemsToAdvance, correlationId));
    }
    catch (Exception ex) {
        HideProcessingIndicator();
        ShowError("Error al iniciar el proceso de avance: " + ex.Message);
    }
}

private void UpdateAdvanceStatus(ItemToAdvance itemToAdvance)
{
    // Uso correcto de comparación en vez de asignación
    if (itemToAdvance.AdvanceStatus == AdvanceStatus.Launched) 
    {
        // Lógica que depende del estado...
    }
}

// En el servicio de notificaciones
public class AdvanceNotificationService : IAdvanceNotificationService
{
    private readonly ConcurrentDictionary<Guid, Action<AdvanceResult>> _callbacks = new();
    
    public void RegisterForNotification(Guid correlationId, Action<AdvanceResult> callback)
    {
        _callbacks.TryAdd(correlationId, callback);
    }
    
    // Este método es llamado por un handler que escucha eventos del sistema
    public void HandleAdvanceCompleted(AdvanceCompletedEvent @event)
    {
        if (_callbacks.TryRemove(@event.CorrelationId, out var callback))
        {
            callback(new AdvanceResult(@event.Success, @event.Message));
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Eliminar dependencias temporales usando un sistema basado en eventos
- Proporcionar retroalimentación adecuada al usuario durante el procesamiento
- Garantizar la actualización de la UI solo cuando los datos están disponibles
- Utilizar correlationIds para asociar solicitudes con sus respuestas
- Separar el envío de comandos de la recepción de resultados
- Manejar correctamente las comparaciones de estado

## Conclusión

Los acoplamientos temporales son una fuente común de errores en sistemas distribuidos y aplicaciones asíncronas. Las mejores prácticas incluyen utilizar sistemas basados en eventos, callbacks para notificaciones, y correlationIds para seguimiento. Estos enfoques crean sistemas más robustos, predecibles y con mejor experiencia de usuario.

# Problema #6: Strings Mágicos y Hardcodeados

## Descripción del Problema

En la solución PAT se identificó un uso extensivo de strings y textos hardcodeados directamente en el código, en lugar de utilizar recursos centralizados o constantes bien definidas, lo que dificulta el mantenimiento, la localización y causa inconsistencias.

### Código Problemático

```csharp
public static class TemplatesHtml
{
    public const string LegalAdviceText = @"*AVISO LEGAL COMPASS: La relación de Facturas...";
    
    public static string GenerateReportHTML(ReportDto report)
    {
        var html = @"<html>
            <head><title>Reporte de Operaciones</title></head>
            <body>
                <h1>Reporte de Operaciones</h1>
                <p>Generado el " + DateTime.Now.ToString("dd/MM/yyyy HH:mm") + @"</p>
                <!-- Más HTML hardcodeado... -->
            </body>
        </html>";
        
        return html;
    }
}

// En otro archivo:
public void ShowError()
{
    errorFunc.Invoke("Hay un proceso de adelanto en curso. Por favor espere a que termine.");
}

public void ValidateOperation(Operation operation)
{
    if (operation.Status != "ACTIVO")
    {
        throw new ValidationException("La operación debe estar en estado ACTIVO");
    }
}
```
**Archivo:** `~\Compass\Compass.Reports.Contracts\Templates\TemplatesHtml.cs` y otros

## Consecuencias

1. **Dificultad de internacionalización**: No hay manera fácil de adaptar la aplicación a diferentes idiomas
2. **Inconsistencias**: El mismo mensaje puede estar escrito de formas diferentes en distintas partes
3. **Duplicación**: Textos idénticos se repiten en múltiples archivos del proyecto
4. **Fragilidad ante cambios**: Un cambio en un texto requiere modificar múltiples archivos
5. **Falta de contexto**: Los strings codificados directamente no proporcionan contexto sobre su uso

## Solución Recomendada

```csharp
// Implementar una clase de recursos centralizada
public static class AppResources
{
    public static class LegalTexts
    {
        public static string CompassLegalAdvice => ResourceManager.GetString("CompassLegalAdvice");
    }
    
    public static class ErrorMessages
    {
        public static string AdvanceInProgress => ResourceManager.GetString("AdvanceInProgress");
        public static string OperationMustBeActive => ResourceManager.GetString("OperationMustBeActive");
    }
    
    public static class OperationStatus
    {
        public const string Active = "ACTIVO";
        public const string Inactive = "INACTIVO";
        public const string Pending = "PENDIENTE";
    }
}

// Usar recursos localizables con un ResourceManager
public class ResourceService : IResourceService
{
    private readonly IStringLocalizer<ErrorMessages> _errorLocalizer;
    private readonly IStringLocalizer<LegalTexts> _legalLocalizer;
    
    public ResourceService(
        IStringLocalizer<ErrorMessages> errorLocalizer,
        IStringLocalizer<LegalTexts> legalLocalizer)
    {
        _errorLocalizer = errorLocalizer;
        _legalLocalizer = legalLocalizer;
    }
    
    public string GetError(string key, params object[] args)
    {
        return _errorLocalizer[key, args];
    }
    
    public string GetLegalText(string key)
    {
        return _legalLocalizer[key];
    }
}

// Para las plantillas HTML, usar un motor de templates
public class TemplateService : ITemplateService
{
    private readonly IResourceService _resourceService;
    private readonly IRazorViewRenderer _viewRenderer;
    
    public TemplateService(IResourceService resourceService, IRazorViewRenderer viewRenderer)
    {
        _resourceService = resourceService;
        _viewRenderer = viewRenderer;
    }
    
    public async Task<string> GenerateReportHtml(ReportDto report)
    {
        // Usar plantillas Razor en lugar de strings hardcodeados
        return await _viewRenderer.RenderViewToStringAsync("Reports/OperationReport", report);
    }
}

// Ejemplo de uso
public void ShowError(IResourceService resourceService)
{
    errorFunc.Invoke(resourceService.GetError("AdvanceInProgress"));
}

public void ValidateOperation(Operation operation)
{
    if (operation.Status != AppResources.OperationStatus.Active)
    {
        throw new ValidationException(AppResources.ErrorMessages.OperationMustBeActive);
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Centralizar todos los textos y mensajes en un solo lugar para facilitar mantenimiento
- Proporcionar soporte para internacionalización y localización
- Eliminar la duplicación de textos en el código
- Facilitar cambios globales en mensajes y textos
- Añadir contexto semántico a los textos mediante clases y propiedades bien nombradas
- Permitir la gestión de plantillas HTML con un motor de templates apropiado

## Conclusión

La centralización de strings y el uso de recursos localizables es una práctica fundamental para aplicaciones empresariales. Esto no solo facilita la internacionalización, sino que también mejora significativamente la mantenibilidad del código y reduce inconsistencias. Para textos más complejos como plantillas HTML, es preferible utilizar un motor de templates en lugar de concatenar strings en el código.

# Problema #7: Comentarios que Indican Problemas de Diseño

## Descripción del Problema

En la solución PAT aparecen frecuentemente comentarios que indican problemas conocidos de diseño o implementación que no se han resuelto adecuadamente, revelando debilidades en la arquitectura del sistema.

### Código Problemático

```csharp
private void UpdateAdvanceStatus(ItemToAdvance itemToAdvance)
{
    if (itemToAdvance.AdvanceStatus = $"{(short)AdvanceStatus.Launched}") // This is shitty... but I can't find a cleaner solution
    {
        // Lógica que depende del estado...
    }
    
    // TODO - REFACTOR: pass filters dto object as parameter to repository and build query there
    var items = await _repository.GetItems(startDate, endDate, companyId);
    
    // Esto es un parche temporal, necesita refactorización
    var taxRate = 0.18; // Debería venir de configuración pero no hay tiempo
    
    // No sé por qué esto falla a veces
    try {
        ProcessTaxes();
    } catch (Exception) {
        // Ignoramos el error, no es crítico
    }
}
```
**Archivo:** `~\Compass\Compass.AdvanceProcessing.BlazorComponents\ViewModels\AdvanceProcessingViewModel.cs` y otros

## Consecuencias

1. **Deuda técnica no gestionada**: Problemas conocidos que se acumulan en el tiempo
2. **Fragilidad del código**: Soluciones improvisadas que pueden fallar en condiciones inesperadas
3. **Dificultad de mantenimiento**: Los nuevos desarrolladores tienen que lidiar con código problemático
4. **Reducción de la calidad**: Los "parches temporales" tienden a volverse permanentes
5. **Pérdida de conocimiento**: El contexto de por qué se implementó algo de cierta manera puede olvidarse

## Solución Recomendada

```csharp
// Documentar correctamente los problemas con tickets en el sistema de gestión
// TICKET-1234: Refactorizar comparación de estados para usar enums tipados

// Implementar una solución correcta en lugar de dejar comentarios
private void UpdateAdvanceStatus(ItemToAdvance itemToAdvance)
{
    // Uso correcto de comparación de enums tipados
    if (itemToAdvance.AdvanceStatus == AdvanceStatus.Launched)
    {
        // Lógica que depende del estado...
    }
    
    // Implementar el TODO mencionado en lugar de dejarlo como comentario
    var filters = new ItemFilterDto
    {
        StartDate = startDate,
        EndDate = endDate,
        CompanyId = companyId
    };
    var items = await _repository.GetItems(filters);
    
    // Utilizar configuración adecuada
    var taxRate = _configurationService.GetTaxRate();
    
    // Implementar manejo de errores adecuado con logging
    try {
        await ProcessTaxes();
    } catch (Exception ex) {
        _logger.LogError(ex, "Error al procesar impuestos: {Message}", ex.Message);
        _notificationService.NotifyAdministrator("Error en procesamiento de impuestos", ex.Message);
    }
}

// Añadir pruebas para detectar problemas temprano
[Fact]
public void AdvanceStatus_ShouldCompareCorrectly()
{
    // Arrange
    var item = new ItemToAdvance { AdvanceStatus = AdvanceStatus.Launched };
    
    // Act & Assert
    Assert.True(item.AdvanceStatus == AdvanceStatus.Launched);
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Eliminar código problemático en lugar de solo comentarlo
- Registrar problemas en un sistema de seguimiento para asegurar su resolución
- Implementar soluciones adecuadas para patrones conocidos
- Añadir pruebas que verifiquen la correcta implementación
- Proporcionar manejo de errores adecuado con logging y notificaciones
- Eliminar valores mágicos usando servicios de configuración apropiados

## Conclusión

Los comentarios que indican problemas conocidos son señales de alerta que deben atenderse, no ignorarse. En lugar de documentar deficiencias con comentarios, es preferible registrar los problemas en un sistema de seguimiento y priorizar su solución. Las soluciones temporales deben ser reemplazadas por implementaciones adecuadas que sigan las buenas prácticas de desarrollo. Un código limpio y bien diseñado generalmente no necesita comentarios para explicar por qué hace algo de cierta manera.

# Problema #8: Valores Mágicos Hardcodeados

## Descripción del Problema

En la solución PAT se utilizan frecuentemente valores numéricos y constantes hardcodeadas directamente en el código, sin explicación ni posibilidad de configuración externa, lo que dificulta el mantenimiento y la adaptación del sistema.

### Código Problemático

```csharp
public bool IsValidTermForCreditLine(CreditLine creditLine, int requestedDays)
{
    var creditLineMinTerm = creditLine?.MinCancelationDays ?? 14; // Valor mágico hardcodeado
    var creditLineMaxTerm = creditLine?.MaxCancelationDays ?? 180; // Valor mágico hardcodeado
    
    return requestedDays >= creditLineMinTerm && requestedDays <= creditLineMaxTerm;
}

// En otro archivo
public async Task GeneratePeriodicReports()
{
    int reportRetentionDays = 90; // Valor mágico hardcodeado
    decimal taxRate = 0.18M; // Valor mágico hardcodeado
    int gracePeriod = 5; // Valor mágico hardcodeado
    string defaultCurrency = "PEN"; // Valor mágico hardcodeado
    
    // Uso de los valores mágicos
    var cutoffDate = DateTime.Now.AddDays(-reportRetentionDays);
    // Más código...
}

// En otro archivo más
public async Task ProcessItems()
{
    int batchSize = 10; // Valor mágico hardcodeado
    int maxRetries = 3; // Valor mágico hardcodeado
    TimeSpan retryDelay = TimeSpan.FromSeconds(5); // Valor mágico hardcodeado
    
    // Uso de los valores
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            // Procesamiento por lotes
            for (int j = 0; j < items.Count; j += batchSize)
            {
                var batch = items.Skip(j).Take(batchSize);
                await ProcessBatch(batch);
            }
            break;
        }
        catch
        {
            if (i == maxRetries - 1) throw;
            await Task.Delay(retryDelay);
        }
    }
}
```
**Archivo:** `~\Compass\Compass.CreditLines.Domain\Services\CreditLineValidationService.cs` y otros

## Consecuencias

1. **Rigidez**: Los valores no pueden modificarse sin cambiar el código fuente
2. **Inconsistencia**: El mismo valor puede estar definido con diferentes cifras en distintas partes
3. **Falta de documentación**: No se explica el origen o la justificación de los valores
4. **Dificultad de mantenimiento**: Cambios en los requisitos de negocio requieren modificar el código
5. **Problemas de configuración**: No se pueden adaptar los valores a diferentes entornos o clientes

## Solución Recomendada

```csharp
// Definir clases de configuración tipadas
public class CreditLineSettings
{
    public int DefaultMinCancelationDays { get; set; } = 14;
    public int DefaultMaxCancelationDays { get; set; } = 180;
}

public class ReportSettings
{
    public int RetentionDays { get; set; } = 90;
    public decimal DefaultTaxRate { get; set; } = 0.18M;
    public int GracePeriod { get; set; } = 5;
    public string DefaultCurrency { get; set; } = "PEN";
}

public class ProcessingSettings
{
    public int BatchSize { get; set; } = 10;
    public int MaxRetries { get; set; } = 3;
    public TimeSpan RetryDelay { get; set; } = TimeSpan.FromSeconds(5);
}

// Implementar servicios que usen la configuración inyectada
public class CreditLineService
{
    private readonly CreditLineSettings _settings;
    
    public CreditLineService(IOptions<CreditLineSettings> settings)
    {
        _settings = settings.Value;
    }
    
    public bool IsValidTermForCreditLine(CreditLine creditLine, int requestedDays)
    {
        var creditLineMinTerm = creditLine?.MinCancelationDays ?? _settings.DefaultMinCancelationDays;
        var creditLineMaxTerm = creditLine?.MaxCancelationDays ?? _settings.DefaultMaxCancelationDays;
        
        return requestedDays >= creditLineMinTerm && requestedDays <= creditLineMaxTerm;
    }
}

// Configuración en archivo appsettings.json
{
  "CreditLineSettings": {
    "DefaultMinCancelationDays": 14,
    "DefaultMaxCancelationDays": 180
  },
  "ReportSettings": {
    "RetentionDays": 90,
    "DefaultTaxRate": 0.18,
    "GracePeriod": 5,
    "DefaultCurrency": "PEN"
  },
  "ProcessingSettings": {
    "BatchSize": 10,
    "MaxRetries": 3,
    "RetryDelay": "00:00:05"
  }
}

// Registro de configuración en el contenedor de DI
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<CreditLineSettings>(
        Configuration.GetSection("CreditLineSettings"));
    
    services.Configure<ReportSettings>(
        Configuration.GetSection("ReportSettings"));
    
    services.Configure<ProcessingSettings>(
        Configuration.GetSection("ProcessingSettings"));
}

// Uso en servicios inyectados
public class BatchProcessor
{
    private readonly ProcessingSettings _settings;
    
    public BatchProcessor(IOptions<ProcessingSettings> settings)
    {
        _settings = settings.Value;
    }
    
    public async Task ProcessItems(IEnumerable<Item> items)
    {
        for (int i = 0; i < _settings.MaxRetries; i++)
        {
            try
            {
                // Procesamiento por lotes
                var batchedItems = items.Chunk(_settings.BatchSize);
                foreach (var batch in batchedItems)
                {
                    await ProcessBatch(batch);
                }
                break;
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Error en el intento {Attempt} de {MaxRetries}", i + 1, _settings.MaxRetries);
                if (i == _settings.MaxRetries - 1) throw;
                await Task.Delay(_settings.RetryDelay);
            }
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Centralizar la configuración en archivos externos fácilmente modificables
- Proporcionar valores predeterminados sensatos cuando no se especifica configuración
- Documentar el propósito de cada valor a través de nombres significativos
- Permitir diferentes configuraciones para distintos entornos (desarrollo, pruebas, producción)
- Facilitar cambios en los requisitos de negocio sin modificar el código fuente
- Mejorar la testabilidad al poder inyectar diferentes configuraciones durante las pruebas

## Conclusión

La externalización de valores numéricos y constantes en archivos de configuración es una práctica fundamental para aplicaciones empresariales robustas. Esto proporciona flexibilidad para adaptar el comportamiento del sistema a diferentes entornos y requisitos sin necesidad de recompilar. Además, mejora la mantenibilidad al centralizar estos valores y dotarlos de contexto semántico mediante nombres significativos. En .NET Core y superiores, el sistema de configuración tipada ofrece una solución elegante para este problema.

# Problema #9: Gestión de Estado Manual en Lugar de Stateful Services

## Descripción del Problema

En la solución PAT se identificaron múltiples implementaciones de gestión de estado manual mediante variables booleanas y flags, en lugar de utilizar patrones y servicios diseñados específicamente para el manejo de estado.

### Código Problemático

```csharp
public class OperationListViewModel
{
    // Flag para evitar inicios múltiples del proceso
    private bool _isProcessing;
    // Control manual de estado
    private int _currentStep;
    private List<string> _errors = new List<string>();
    private bool _hasCompletedValidation;
    
    public async Task ProcessOperation()
    {
        if (_isProcessing) 
        {
            ShowError("Hay un proceso en curso");
            return;
        }
        
        _isProcessing = true;
        _currentStep = 1;
        _errors.Clear();
        
        try
        {
            // Paso 1
            _currentStep = 1;
            await DoStep1();
            
            // Paso 2 si el 1 fue exitoso
            if (_errors.Count == 0)
            {
                _currentStep = 2;
                await DoStep2();
            }
            
            // Finalización
            _hasCompletedValidation = _errors.Count == 0;
        }
        finally
        {
            _isProcessing = false;
        }
    }
}
```
**Archivo:** `~\Compass\Compass.AdvanceProcessing.BlazorComponents\ViewModels\AdvanceProcessingViewModel.cs`

## Consecuencias

1. **Condiciones de carrera**: Variables como `_isProcessing` pueden generar comportamientos inconsistentes en entornos concurrentes
2. **Gestión de estado frágil**: Olvidar restablecer flags en ciertas ramas de código puede bloquear el sistema
3. **Lógica de transición dispersa**: La lógica de cambio de estado está esparcida por todo el código
4. **Estado no persistente**: Si la aplicación se reinicia, se pierde todo el estado
5. **Dificultad para depurar**: Es difícil seguir la secuencia de cambios de estado

## Solución Recomendada

```csharp
// Definir estados posibles mediante una enumeración
public enum OperationProcessState
{
    Initial,
    Processing,
    ValidationComplete,
    ExecutionComplete,
    Failed
}

// Interfaz para comandos que pueden provocar transiciones
public interface IOperationCommand { }

// Comandos concretos
public class StartProcessCommand : IOperationCommand { }
public class CompleteValidationCommand : IOperationCommand { }
public class ExecuteOperationCommand : IOperationCommand { }
public class AbortOperationCommand : IOperationCommand { }

// Implementación de una máquina de estados
public class ProcessStateMachine<TState, TCommand>
    where TState : struct, Enum
    where TCommand : class
{
    private readonly Dictionary<(TState currentState, Type commandType), Func<TCommand, Task<TState>>> _transitions;
    private TState _currentState;
    
    public ProcessStateMachine(TState initialState)
    {
        _currentState = initialState;
        _transitions = new Dictionary<(TState, Type), Func<TCommand, Task<TState>>>();
    }
    
    public TState CurrentState => _currentState;
    
    public void AddTransition(TState fromState, Type commandType, Func<TCommand, Task<TState>> handler)
    {
        _transitions.Add((fromState, commandType), handler);
    }
    
    public async Task<TState> SendCommand(TCommand command)
    {
        var commandType = command.GetType();
        var transitionKey = (_currentState, commandType);
        
        if (!_transitions.TryGetValue(transitionKey, out var handler))
            throw new InvalidOperationException($"No hay transición definida para estado {_currentState} y comando {commandType.Name}");
        
        var newState = await handler(command);
        _currentState = newState;
        
        return newState;
    }
}

// ViewModel que utiliza la máquina de estados
public class OperationListViewModel
{
    private readonly ProcessStateMachine<OperationProcessState, IOperationCommand> _stateMachine;
    private readonly IOperationService _operationService;
    private readonly INotificationService _notificationService;
    
    public OperationListViewModel(
        IOperationService operationService, 
        INotificationService notificationService)
    {
        _operationService = operationService;
        _notificationService = notificationService;
        
        // Inicializar máquina de estados
        _stateMachine = new ProcessStateMachine<OperationProcessState, IOperationCommand>(OperationProcessState.Initial);
        
        // Configurar transiciones
        _stateMachine.AddTransition(
            OperationProcessState.Initial,
            typeof(StartProcessCommand),
            async cmd => {
                try {
                    await DoStep1();
                    return OperationProcessState.ValidationComplete;
                }
                catch (Exception ex) {
                    _notificationService.ShowError(ex.Message);
                    return OperationProcessState.Failed;
                }
            });
        
        _stateMachine.AddTransition(
            OperationProcessState.ValidationComplete,
            typeof(ExecuteOperationCommand),
            async cmd => {
                try {
                    await DoStep2();
                    return OperationProcessState.ExecutionComplete;
                }
                catch (Exception ex) {
                    _notificationService.ShowError(ex.Message);
                    return OperationProcessState.Failed;
                }
            });
        
        // Más transiciones...
    }
    
    public async Task ProcessOperation()
    {
        // La máquina de estados garantiza que no se inicien múltiples procesos
        if (_stateMachine.CurrentState != OperationProcessState.Initial &&
            _stateMachine.CurrentState != OperationProcessState.Failed)
        {
            _notificationService.ShowWarning("Hay un proceso en curso");
            return;
        }
        
        // Iniciar el proceso
        await _stateMachine.SendCommand(new StartProcessCommand());
        
        // Si la validación fue exitosa, continuar con la ejecución
        if (_stateMachine.CurrentState == OperationProcessState.ValidationComplete)
        {
            await _stateMachine.SendCommand(new ExecuteOperationCommand());
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Centralizar la lógica de gestión de estado en una máquina de estados explícita
- Definir claramente los estados posibles y las transiciones permitidas
- Evitar condiciones de carrera mediante un diseño thread-safe
- Proporcionar una forma coherente de representar y manipular el estado de la aplicación
- Facilitar la depuración al tener transiciones de estado explícitas y trazables
- Permitir persistencia del estado actual si fuera necesario
- Reducir errores al imposibilitar transiciones no definidas

## Conclusión

La gestión manual de estado con flags y variables booleanas es propensa a errores y difícil de mantener. Utilizar patrones como State Pattern, máquinas de estado o servicios stateful proporciona una forma más robusta y clara de manejar estados complejos. Estas soluciones estructuradas permiten que el sistema sea más predecible, mantenible y menos propenso a errores sutiles relacionados con la concurrencia y el ciclo de vida de los componentes.

# Problema #10: Repetición de Código Entre Componentes

## Descripción del Problema

En la solución PAT se identificó duplicación significativa de código entre componentes similares, especialmente en los ViewModels de diferentes módulos que implementan la misma funcionalidad (como Factoring, Confirming, Letras).

### Código Problemático

```csharp
// En ConfirmingOperationListViewModel.cs
public class ConfirmingOperationListViewModel
{
    private readonly IConfirmingService _confirmingService;
    private readonly Action<string> errorFunc;
    private bool _isProcessing;
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        if (_isProcessing) 
        {
            errorFunc.Invoke("Hay un proceso de adelanto en curso. Por favor espere a que termine.");
            return;
        }
        
        _isProcessing = true;
        
        try
        {
            var totalAmount = itemsToAdvance.Sum(item => item.Amount);
            await _confirmingService.AdvanceItems(itemsToAdvance.Select(i => i.Id).ToList(), totalAmount);
            successFunc.Invoke("Items procesados correctamente");
        }
        catch (Exception ex)
        {
            errorFunc.Invoke($"Error: {ex.Message}");
        }
        finally
        {
            _isProcessing = false;
        }
    }
}

// En FactoringOperationListViewModel.cs - Código casi idéntico
public class FactoringOperationListViewModel
{
    private readonly IFactoringService _factoringService;
    private readonly Action<string> errorFunc;
    private bool _isProcessing;
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        if (_isProcessing) 
        {
            errorFunc.Invoke("Hay un proceso de adelanto en curso. Por favor espere a que termine.");
            return;
        }
        
        _isProcessing = true;
        
        try
        {
            var totalAmount = itemsToAdvance.Sum(item => item.Amount);
            await _factoringService.AdvanceItems(itemsToAdvance.Select(i => i.Id).ToList(), totalAmount);
            successFunc.Invoke("Items procesados correctamente");
        }
        catch (Exception ex)
        {
            errorFunc.Invoke($"Error: {ex.Message}");
        }
        finally
        {
            _isProcessing = false;
        }
    }
}
```
**Archivo:** `~\Compass\Compass.Confirming.BlazorComponents\Pages\OperationList\OperationListViewModel.cs` y `~\Compass\Compass.Factoring.BlazorComponents\Pages\OperationList\OperationListViewModel.cs`

## Consecuencias

1. **Mantenimiento costoso**: Cualquier cambio debe implementarse en múltiples lugares
2. **Inconsistencia**: Los componentes pueden evolucionar de manera diferente con el tiempo
3. **Bugs duplicados**: Un error en la lógica compartida se repite en todos los componentes
4. **Código base inflado**: El tamaño del proyecto aumenta innecesariamente
5. **Violación del principio DRY**: Don't Repeat Yourself es un principio fundamental ignorado

## Solución Recomendada

```csharp
// Crear una clase base para los ViewModels
public abstract class BaseAdvanceService<T> where T : class
{
    protected readonly Action<string> _errorMessageHandler;
    protected readonly Action<string> _successMessageHandler;
    private readonly SemaphoreSlim _processingSemaphore = new SemaphoreSlim(1);
    
    protected BaseAdvanceService(Action<string> errorMessageHandler, Action<string> successMessageHandler)
    {
        _errorMessageHandler = errorMessageHandler;
        _successMessageHandler = successMessageHandler;
    }
    
    protected abstract Task<bool> ExecuteAdvance(IEnumerable<Guid> itemIds, decimal totalAmount);
    
    public async Task<bool> AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        if (itemsToAdvance == null || !itemsToAdvance.Any())
        {
            _errorMessageHandler("No hay elementos para avanzar");
            return false;
        }
        
        // Usar semáforo en vez de flag booleano
        if (!await _processingSemaphore.WaitAsync(0))
        {
            _errorMessageHandler("Hay un proceso en curso");
            return false;
        }
        
        try
        {
            var itemIds = itemsToAdvance.Select(i => i.Id).ToList();
            var totalAmount = itemsToAdvance.Sum(i => i.Amount);
            
            var result = await ExecuteAdvance(itemIds, totalAmount);
            
            if (result)
                _successMessageHandler("Items procesados correctamente");
                
            return result;
        }
        catch (Exception ex)
        {
            _errorMessageHandler($"Error en el proceso: {ex.Message}");
            return false;
        }
        finally
        {
            _processingSemaphore.Release();
        }
    }
}

// Implementación para Confirming
public class ConfirmingAdvanceService : BaseAdvanceService<ConfirmingItem>
{
    private readonly IConfirmingService _confirmingService;
    
    public ConfirmingAdvanceService(
        IConfirmingService confirmingService,
        Action<string> errorMessageHandler, 
        Action<string> successMessageHandler) 
        : base(errorMessageHandler, successMessageHandler)
    {
        _confirmingService = confirmingService;
    }
    
    protected override async Task<bool> ExecuteAdvance(IEnumerable<Guid> itemIds, decimal totalAmount)
    {
        return await _confirmingService.AdvanceItems(itemIds, totalAmount);
    }
}

// Implementación para Factoring
public class FactoringAdvanceService : BaseAdvanceService<FactoringItem>
{
    private readonly IFactoringService _factoringService;
    
    public FactoringAdvanceService(
        IFactoringService factoringService,
        Action<string> errorMessageHandler, 
        Action<string> successMessageHandler) 
        : base(errorMessageHandler, successMessageHandler)
    {
        _factoringService = factoringService;
    }
    
    protected override async Task<bool> ExecuteAdvance(IEnumerable<Guid> itemIds, decimal totalAmount)
    {
        return await _factoringService.AdvanceItems(itemIds, totalAmount);
    }
}

// ViewModels simplificados que usan los servicios
public class ConfirmingOperationListViewModel
{
    private readonly ConfirmingAdvanceService _advanceService;
    
    public ConfirmingOperationListViewModel(ConfirmingAdvanceService advanceService)
    {
        _advanceService = advanceService;
    }
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        await _advanceService.AdvanceAllItemsOnDate(itemsToAdvance);
    }
}

public class FactoringOperationListViewModel
{
    private readonly FactoringAdvanceService _advanceService;
    
    public FactoringOperationListViewModel(FactoringAdvanceService advanceService)
    {
        _advanceService = advanceService;
    }
    
    public async Task AdvanceAllItemsOnDate(IEnumerable<ItemToAdvanceRow> itemsToAdvance)
    {
        await _advanceService.AdvanceAllItemsOnDate(itemsToAdvance);
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Centralizar la lógica común en una clase base reutilizable
- Minimizar la duplicación de código entre diferentes módulos
- Garantizar un comportamiento consistente entre todos los tipos de avance
- Simplificar el mantenimiento al tener que modificar el código en un solo lugar
- Utilizar patrones como Template Method para personalizar comportamientos específicos
- Mejorar la estructura del código mediante una jerarquía clara
- Facilitar la adición de nuevos tipos de avance en el futuro

## Conclusión

La repetición de código es un anti-patrón que incrementa la complejidad, dificulta el mantenimiento y aumenta la probabilidad de errores. Mediante el uso de patrones como herencia, composición y plantillas, se puede extraer la funcionalidad común y crear una estructura que promueva la reutilización del código. Esto no solo reduce el tamaño del código base, sino que también mejora su calidad, consistencia y mantenibilidad a largo plazo.

# Problema #11: Manejo Incorrecto de Transacciones y Operaciones Distribuidas

## Descripción del Problema

En la solución PAT se identificó un patrón problemático en el manejo de transacciones, donde múltiples operaciones relacionadas y dependientes entre sí no se gestionan dentro de un contexto transaccional adecuado, lo que puede comprometer la integridad de datos en escenarios de fallos.

### Código Problemático

```csharp
public async Task<Result> ProcessPayment(Guid operationId, decimal amount)
{
    // Primera operación: Actualizar el estado del documento
    var document = await _documentRepository.GetById(operationId);
    document.Status = DocumentStatus.Processing;
    await _documentRepository.Update(document);
    
    // Segunda operación: Crear registro de transacción financiera
    var transaction = new FinancialTransaction
    {
        Id = Guid.NewGuid(),
        DocumentId = operationId,
        Amount = amount,
        Date = DateTime.UtcNow,
        Type = TransactionType.Payment
    };
    await _transactionRepository.Create(transaction);
    
    // Tercera operación: Actualizar el saldo del cliente
    var client = await _clientRepository.GetById(document.ClientId);
    client.Balance += amount;
    await _clientRepository.Update(client);
    
    // Cuarta operación: Enviar notificación
    await _notificationService.SendPaymentConfirmation(document.ClientId, amount);
    
    // Quinta operación: Finalizar actualización del documento
    document.Status = DocumentStatus.Paid;
    document.PaidAmount = amount;
    document.PaidDate = DateTime.UtcNow;
    await _documentRepository.Update(document);
    
    return Result.Success();
}
```
**Archivo:** `~\Compass\Compass.Payments.Domain\Services\PaymentProcessor.cs`

## Consecuencias

1. **Inconsistencia de datos**: Si el proceso falla después de algunas operaciones, el sistema queda en un estado inconsistente
2. **Operaciones huérfanas**: Se pueden crear registros de transacción sin completar el flujo completo
3. **Pérdida de confiabilidad**: No se garantiza que todo el proceso se complete o se deshaga completamente
4. **Imposibilidad de reintentos**: Sin un mecanismo de compensación, es difícil reanudar operaciones fallidas
5. **Dificultad para diagnosticar problemas**: Sin un registro claro de la transacción y su estado, es difícil entender fallos

## Solución Recomendada

```csharp
public async Task<Result> ProcessPayment(Guid operationId, decimal amount)
{
    // Registrar inicio del proceso para seguimiento
    var processId = Guid.NewGuid();
    _logger.LogInformation("Iniciando proceso de pago {ProcessId} para operación {OperationId}", processId, operationId);
    
    // Usar TransactionScope para operaciones de bases de datos
    using (var transactionScope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
    {
        try
        {
            // Primera operación: Actualizar el estado del documento
            var document = await _documentRepository.GetById(operationId);
            if (document == null)
                return Result.Failure("El documento no existe");
                
            document.Status = DocumentStatus.Processing;
            await _documentRepository.Update(document);
            
            // Segunda operación: Crear registro de transacción financiera
            var transaction = new FinancialTransaction
            {
                Id = Guid.NewGuid(),
                DocumentId = operationId,
                Amount = amount,
                Date = DateTime.UtcNow,
                Type = TransactionType.Payment,
                ProcessId = processId
            };
            await _transactionRepository.Create(transaction);
            
            // Tercera operación: Actualizar el saldo del cliente
            var client = await _clientRepository.GetById(document.ClientId);
            if (client == null)
                return Result.Failure("El cliente no existe");
                
            client.Balance += amount;
            await _clientRepository.Update(client);
            
            // Completar la transacción de base de datos
            transactionScope.Complete();
            
            // Operaciones que no pueden ser parte de la transacción principal
            // pero son idempotentes o pueden reintenarse
            await SendNotificationsAndFinalizePayment(document, amount, processId);
            
            _logger.LogInformation("Proceso de pago {ProcessId} completado exitosamente", processId);
            return Result.Success();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error en proceso de pago {ProcessId}: {Message}", processId, ex.Message);
            // La transacción se revertirá automáticamente por la disposición del TransactionScope
            
            // Registrar el error para análisis posterior
            await _paymentErrorRepository.RegisterError(processId, operationId, ex.Message);
            
            return Result.Failure($"Error al procesar el pago: {ex.Message}");
        }
    }
}

private async Task SendNotificationsAndFinalizePayment(Document document, decimal amount, Guid processId)
{
    try
    {
        // Usar un patrón de reintentos para operaciones no transaccionales
        await _retryPolicy.ExecuteAsync(async () =>
        {
            // Cuarta operación: Enviar notificación
            await _notificationService.SendPaymentConfirmation(document.ClientId, amount);
            
            // Quinta operación: Finalizar actualización del documento
            document.Status = DocumentStatus.Paid;
            document.PaidAmount = amount;
            document.PaidDate = DateTime.UtcNow;
            await _documentRepository.Update(document);
            
            // Registrar finalización correcta
            await _processTracker.CompleteProcess(processId);
        });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error en finalización de pago {ProcessId}: {Message}", processId, ex.Message);
        
        // Encolar tarea para procesamiento posterior
        await _backgroundJobClient.Enqueue<IPaymentFinalizer>(
            finalizer => finalizer.FinalizePaymentAsync(document.Id, amount, processId));
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Garantizar consistencia de datos mediante transacciones adecuadas para operaciones relacionadas
- Separar operaciones transaccionales de las no transaccionales o externas
- Implementar un sistema de logging estructurado para seguimiento de procesos
- Incorporar reintentos automáticos para operaciones idempotentes
- Proporcionar un mecanismo de compensación mediante trabajos en segundo plano para finalizar operaciones incompletas
- Facilitar la auditoría y diagnóstico mediante IDs de proceso únicos
- Asegurar que los recursos se liberan adecuadamente mediante bloques using

## Conclusión

El manejo correcto de transacciones es fundamental para mantener la integridad de los datos en aplicaciones empresariales. Utilizando patrones como Unit of Work, TransactionScope y separando las operaciones transaccionales de las externas, podemos crear sistemas más robustos que garanticen la consistencia incluso ante fallos. Complementar esto con mecanismos de compensación, reintentos y un sistema de seguimiento adecuado permite recuperarse de errores y facilitar la resolución de problemas cuando ocurren.

# Problema #12: Falta de Manejo de Errores Consistente

## Descripción del Problema

En la solución PAT se identificó la ausencia de un enfoque consistente para el manejo de errores. Existen múltiples patrones utilizados simultáneamente: excepciones no controladas, retorno de nulos, códigos de estado personalizados, y mensajes de error como strings, lo que dificulta el tratamiento uniforme de condiciones de error.

### Código Problemático

```csharp
// Ejemplo 1: Retorno de null sin información adicional
public async Task<Document> GetDocumentById(Guid id)
{
    var document = await _repository.GetById(id);
    return document; // Retorna null si no existe, sin contexto adicional
}

// Ejemplo 2: Lanzamiento de excepciones sin manejo estructurado
public async Task<decimal> CalculateAmount(Guid documentId)
{
    var document = await _repository.GetById(documentId);
    if (document == null)
        throw new Exception("Documento no encontrado"); // Excepción genérica
        
    try {
        return await _calculator.Calculate(document);
    }
    catch (Exception ex) {
        // Log simple sin estructuración
        _logger.LogError("Error al calcular monto: " + ex.Message);
        throw; // Re-lanza la excepción sin enriquecerla
    }
}

// Ejemplo 3: Uso de booleanos para señalizar éxito/fallo
public async Task<bool> ProcessOperation(OperationDto operation)
{
    try {
        await _operationService.Process(operation);
        return true; // No hay forma de obtener información sobre el resultado
    }
    catch {
        return false; // Sin detalles del error
    }
}

// Ejemplo 4: Uso de strings para comunicar errores
public async Task<string> ValidateDocument(DocumentDto document)
{
    if (string.IsNullOrEmpty(document.Number))
        return "El número de documento es requerido";
        
    if (document.Amount <= 0)
        return "El monto debe ser mayor que cero";
        
    // Si no hay error, retorna null (mezclando null con string de error)
    return null;
}
```
**Archivo:** `~\Compass\Compass.Documents.Domain\Services\DocumentService.cs` y otros

## Consecuencias

1. **Inconsistencia en manejo de errores**: Cada componente implementa su propio enfoque
2. **Pérdida de contexto**: Los errores originales pierden información valiosa al propagarse
3. **Dificultad para centralizar el tratamiento de errores**: No hay un formato estándar
4. **Complejidad en el código cliente**: Debe manejar múltiples tipos de respuestas de error
5. **Problemas de diagnóstico**: La falta de información estructurada dificulta identificar la causa raíz

## Solución Recomendada

```csharp
// Definir un tipo de resultado genérico
public class Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public Error Error { get; }
    
    private Result(bool isSuccess, T value, Error error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => new Result<T>(true, value, null);
    public static Result<T> Failure(Error error) => new Result<T>(false, default, error);
    
    // Pattern matching helpers
    public TResult Match<TResult>(
        Func<T, TResult> onSuccess,
        Func<Error, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value) : onFailure(Error);
        
    public bool TryGetValue(out T value)
    {
        value = IsSuccess ? Value : default;
        return IsSuccess;
    }
}

// Para operaciones sin valor de retorno
public class Result
{
    public bool IsSuccess { get; }
    public Error Error { get; }
    
    private Result(bool isSuccess, Error error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }
    
    public static Result Success() => new Result(true, null);
    public static Result Failure(Error error) => new Result(false, error);
    
    // Implicit conversion from Result<T> to Result
    public static implicit operator Result(Result<object> result) =>
        result.IsSuccess ? Success() : Failure(result.Error);
}

// Error estructurado con código y detalles
public class Error
{
    public string Code { get; }
    public string Message { get; }
    public string Details { get; }
    public Exception Exception { get; }
    
    public Error(string code, string message, string details = null, Exception exception = null)
    {
        Code = code;
        Message = message;
        Details = details;
        Exception = exception;
    }
    
    public static Error NotFound(string entity, string id) => 
        new Error("NotFound", $"{entity} con ID {id} no encontrado");
        
    public static Error Validation(string message) => 
        new Error("Validation", message);
        
    public static Error Unexpected(Exception ex) => 
        new Error("Unexpected", "Ocurrió un error inesperado", ex?.Message, ex);
}

// Implementación consistente en servicios
public class DocumentService
{
    private readonly IDocumentRepository _repository;
    private readonly IAmountCalculator _calculator;
    private readonly ILogger<DocumentService> _logger;
    
    // Constructor...
    
    public async Task<Result<Document>> GetDocumentById(Guid id)
    {
        try
        {
            var document = await _repository.GetById(id);
            if (document == null)
                return Result<Document>.Failure(Error.NotFound("Document", id.ToString()));
            
            return Result<Document>.Success(document);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al obtener documento {Id}", id);
            return Result<Document>.Failure(Error.Unexpected(ex));
        }
    }
    
    public async Task<Result<decimal>> CalculateAmount(Guid documentId)
    {
        try
        {
            // Reutilizar el método anterior y aprovechar su manejo de errores
            var documentResult = await GetDocumentById(documentId);
            
            // Pattern matching para manejar éxito/fallo
            return await documentResult.Match(
                async document => {
                    try
                    {
                        var amount = await _calculator.Calculate(document);
                        return Result<decimal>.Success(amount);
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, "Error al calcular monto para documento {Id}", documentId);
                        return Result<decimal>.Failure(
                            new Error("CalculationError", "Error al calcular monto", ex.Message, ex));
                    }
                },
                error => Task.FromResult(Result<decimal>.Failure(error))
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error inesperado al calcular monto para documento {Id}", documentId);
            return Result<decimal>.Failure(Error.Unexpected(ex));
        }
    }
    
    public async Task<Result> ProcessOperation(OperationDto operation)
    {
        try
        {
            await _operationService.Process(operation);
            return Result.Success();
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Error de validación al procesar operación {Id}", operation.Id);
            return Result.Failure(Error.Validation(ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al procesar operación {Id}", operation.Id);
            return Result.Failure(Error.Unexpected(ex));
        }
    }
    
    public Result ValidateDocument(DocumentDto document)
    {
        var validator = new DocumentValidator();
        var validationResult = validator.Validate(document);
        
        if (!validationResult.IsValid)
        {
            var errors = validationResult.Errors.Select(e => e.ErrorMessage).ToList();
            return Result.Failure(Error.Validation(string.Join(", ", errors)));
        }
        
        return Result.Success();
    }
}

// Ejemplo de uso en capa de presentación (Controller o ViewModel)
public class DocumentController
{
    private readonly IDocumentService _documentService;
    
    // Constructor...
    
    public async Task<IActionResult> GetDocument(Guid id)
    {
        var result = await _documentService.GetDocumentById(id);
        
        return result.Match<IActionResult>(
            document => Ok(document),
            error => error.Code switch
            {
                "NotFound" => NotFound(error.Message),
                _ => StatusCode(500, error.Message)
            }
        );
    }
    
    public async Task<IActionResult> CalculateAmount(Guid documentId)
    {
        var result = await _documentService.CalculateAmount(documentId);
        
        if (!result.IsSuccess)
        {
            // Manejo centralizado de errores
            return HandleError(result.Error);
        }
        
        return Ok(new { Amount = result.Value });
    }
    
    private IActionResult HandleError(Error error) => error.Code switch
    {
        "NotFound" => NotFound(error.Message),
        "Validation" => BadRequest(error.Message),
        "CalculationError" => BadRequest(error.Message),
        _ => StatusCode(500, "Se produjo un error interno. Por favor contacte al administrador.")
    };
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar un enfoque consistente y tipado para manejo de errores
- Preservar el contexto completo de los errores, incluyendo excepciones originales
- Diferenciar claramente entre diferentes tipos de errores (validación, no encontrado, etc.)
- Facilitar el manejo centralizado de errores en la capa de presentación
- Permitir la transformación adecuada de errores a respuestas HTTP o mensajes de UI
- Mejorar la capacidad de diagnóstico mediante registro estructurado de errores
- Eliminar el uso de valores nulos o booleanos como indicadores de error sin contexto

## Conclusión

Un manejo de errores consistente es crucial para crear aplicaciones empresariales robustas y mantenibles. El patrón Result proporciona una forma estructurada para representar tanto los resultados exitosos como los errores, preservando el contexto necesario para un adecuado diagnóstico y experiencia de usuario. Este enfoque evita los problemas asociados con excepciones no controladas, valores nulos sin contexto, y strings arbitrarios como mensajes de error.

# Problema #13: Acoplamiento Estático y Fuerte Entre Componentes

## Descripción del Problema

En la solución PAT se identificó un patrón problemático de acoplamiento fuerte y estático entre componentes, donde las clases dependen directamente de implementaciones concretas en lugar de abstracciones, y utilizan referencias directas o singleton estáticos para acceder a servicios. Esto dificulta la modularidad, el testing y la evolución del sistema.

### Código Problemático

```csharp
// Singleton estático con estado compartido
public static class GlobalConfiguration 
{
    public static string ApiUrl { get; set; }
    public static AuthenticationSettings AuthSettings { get; set; }
    
    public static void Initialize()
    {
        ApiUrl = "https://api.compass.com/";
        AuthSettings = new AuthenticationSettings { ClientId = "DefaultClient", Timeout = 30 };
    }
}

// Uso de tipo concreto en lugar de interfaz
public class CreditLineService
{
    // Dependencia directa en implementación concreta, no en interfaz
    private readonly SqlCreditLineRepository _repository;
    
    public CreditLineService()
    {
        // Instanciación directa de dependencias
        _repository = new SqlCreditLineRepository(GlobalConfiguration.ApiUrl);
    }
    
    public async Task<CreditLine> GetCreditLine(Guid id)
    {
        // Acceso directo a componentes estáticos globales
        var authHeader = AuthManager.Current.GetAuthHeader();
        // Uso de singleton estático
        var cache = CacheManager.Instance;
        
        // Intento de obtener de caché
        if (cache.TryGet($"creditLine_{id}", out CreditLine cachedCreditLine))
            return cachedCreditLine;
            
        var creditLine = await _repository.GetCreditLine(id);
        
        // Acceso directo a servicio compartido
        var logService = ServiceLocator.GetService<LogService>();
        logService.Log($"CreditLine {id} retrieved");
        
        return creditLine;
    }
}

// Uso de Service Locator antipatrón
public static class ServiceLocator
{
    private static Dictionary<Type, object> _services = new Dictionary<Type, object>();
    
    public static void Register<T>(T service)
    {
        _services[typeof(T)] = service;
    }
    
    public static T GetService<T>()
    {
        if (_services.TryGetValue(typeof(T), out var service))
            return (T)service;
            
        throw new InvalidOperationException($"Service {typeof(T).Name} not registered");
    }
}

// Clase que accede directamente a otra clase concreta
public class InvoiceProcessor
{
    public async Task Process(Invoice invoice)
    {
        // Acoplamiento directo a otros componentes del sistema
        var calculator = new TaxCalculator();
        var taxAmount = calculator.Calculate(invoice);
        
        // Dependencia y conocimiento directo de otra clase
        var notifier = new EmailNotifier("smtp.compass.com", 587);
        
        // Conocimiento de los detalles internos de otra clase
        if (invoice.Customer.Type == CustomerType.Premium)
        {
            notifier.SetPriority(NotificationPriority.High);
        }
        
        await notifier.SendInvoiceNotification(invoice);
    }
}
```
**Archivo:** `~\Compass\Compass.Core.Services\CreditLine\CreditLineService.cs` y otros

## Consecuencias

1. **Dificultad para pruebas unitarias**: Componentes con dependencias concretas son difíciles de aislar para testing
2. **Rigidez del código**: Los cambios en una dependencia requieren modificar todas las clases que la utilizan directamente
3. **Bajo grado de modularidad**: Los componentes no pueden reutilizarse en diferentes contextos fácilmente
4. **Problemas con estado compartido**: Los singletons estáticos conducen a estado global mutable y comportamiento no determinista
5. **Violación del principio de inversión de dependencias**: Las clases dependen de implementaciones en lugar de abstracciones

## Solución Recomendada

```csharp
// Definir interfaces para todas las dependencias
public interface ICreditLineRepository
{
    Task<CreditLine> GetCreditLine(Guid id);
}

public interface IAuthProvider
{
    string GetAuthHeader();
}

public interface ICacheService
{
    bool TryGet<T>(string key, out T value);
    void Set<T>(string key, T value, TimeSpan expiration);
}

public interface ILogService
{
    void Log(string message);
}

// Implementar clases concretas que implementan las interfaces
public class SqlCreditLineRepository : ICreditLineRepository
{
    private readonly string _connectionString;
    
    public SqlCreditLineRepository(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public async Task<CreditLine> GetCreditLine(Guid id)
    {
        // Implementación...
        return new CreditLine();
    }
}

// Refactorizar servicios para utilizar inyección de dependencias
public class CreditLineService
{
    private readonly ICreditLineRepository _repository;
    private readonly IAuthProvider _authProvider;
    private readonly ICacheService _cacheService;
    private readonly ILogService _logService;
    
    // Inyección de dependencias a través del constructor
    public CreditLineService(
        ICreditLineRepository repository,
        IAuthProvider authProvider,
        ICacheService cacheService,
        ILogService logService)
    {
        _repository = repository;
        _authProvider = authProvider;
        _cacheService = cacheService;
        _logService = logService;
    }
    
    public async Task<CreditLine> GetCreditLine(Guid id)
    {
        // Uso de interfaces inyectadas en lugar de accesos estáticos
        var authHeader = _authProvider.GetAuthHeader();
        
        // Uso de interfaz de caché inyectada
        if (_cacheService.TryGet($"creditLine_{id}", out CreditLine cachedCreditLine))
            return cachedCreditLine;
            
        var creditLine = await _repository.GetCreditLine(id);
        
        // Logging a través de la interfaz
        _logService.Log($"CreditLine {id} retrieved");
        
        return creditLine;
    }
}

// Configuración centralizada con opciones tipadas
public class CompassApiOptions
{
    public string ApiUrl { get; set; }
    public AuthenticationSettings AuthSettings { get; set; }
}

// Configuración de servicios en el contenedor DI
public class Startup
{
    private readonly IConfiguration _configuration;
    
    public Startup(IConfiguration configuration)
    {
        _configuration = configuration;
    }
    
    public void ConfigureServices(IServiceCollection services)
    {
        // Configuración tipada desde appsettings
        services.Configure<CompassApiOptions>(_configuration.GetSection("CompassApi"));
        
        // Registro de servicios en el contenedor de DI
        services.AddSingleton<ICacheService, MemoryCacheService>();
        services.AddScoped<IAuthProvider, JwtAuthProvider>();
        services.AddScoped<ILogService, StructuredLogService>();
        
        // Registro de repositorio con factory para acceder a opciones de configuración
        services.AddScoped<ICreditLineRepository>(provider => {
            var options = provider.GetRequiredService<IOptions<CompassApiOptions>>();
            return new SqlCreditLineRepository(options.Value.ApiUrl);
        });
        
        services.AddScoped<CreditLineService>();
    }
}

// Refactorizar InvoiceProcessor con inyección de dependencias
public interface ITaxCalculator
{
    decimal Calculate(Invoice invoice);
}

public interface INotificationService
{
    Task SendInvoiceNotification(Invoice invoice);
    void SetPriority(NotificationPriority priority);
}

public class InvoiceProcessor
{
    private readonly ITaxCalculator _taxCalculator;
    private readonly INotificationService _notificationService;
    
    public InvoiceProcessor(
        ITaxCalculator taxCalculator,
        INotificationService notificationService)
    {
        _taxCalculator = taxCalculator;
        _notificationService = notificationService;
    }
    
    public async Task Process(Invoice invoice)
    {
        var taxAmount = _taxCalculator.Calculate(invoice);
        
        if (invoice.Customer.Type == CustomerType.Premium)
        {
            _notificationService.SetPriority(NotificationPriority.High);
        }
        
        await _notificationService.SendInvoiceNotification(invoice);
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Facilitar las pruebas unitarias mediante el uso de mocks/stubs para las interfaces
- Reducir el acoplamiento entre componentes al depender de abstracciones, no implementaciones
- Aumentar la flexibilidad del sistema permitiendo cambiar implementaciones sin modificar el código cliente
- Eliminar el estado global mutable que dificulta el razonamiento sobre el comportamiento del código
- Mejorar la modularidad permitiendo la reutilización en diferentes contextos
- Seguir el principio de inversión de dependencias, permitiendo configurar el sistema desde el nivel más alto
- Centralizar la configuración de servicios en un único punto facilitando cambios en la composición del sistema

## Conclusión

El acoplamiento fuerte y estático entre componentes es uno de los mayores obstáculos para la mantenibilidad y evolución de aplicaciones empresariales. Implementar un sistema basado en interfaces y utilizar la inyección de dependencias permite crear un código mucho más flexible, testeable y modular. Los contenedores de inyección de dependencias modernos proporcionan una manera elegante de componer las aplicaciones sin crear dependencias rígidas entre componentes. Esta mejora arquitectónica facilita enormemente el desarrollo ágil, las pruebas automatizadas y la adaptación del sistema a nuevos requisitos.

# Problema #14: Modelos de Dominio Anémicos

## Descripción del Problema

En la solución PAT se identificó un patrón de diseño problemático donde las entidades del dominio actúan como simples contenedores de datos (POCOs) sin comportamiento ni lógica de negocio, delegando toda la lógica a servicios externos. Este patrón, conocido como "Modelo de Dominio Anémico", viola los principios de la Programación Orientada a Objetos y del Domain-Driven Design.

### Código Problemático

```csharp
// Entidad sin comportamiento - simplemente un contenedor de datos
public class Invoice
{
    public Guid Id { get; set; }
    public string Number { get; set; }
    public DateTime IssueDate { get; set; }
    public DateTime DueDate { get; set; }
    public decimal Amount { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal TotalAmount { get; set; }
    public string Status { get; set; }
    public Guid CustomerId { get; set; }
    public Customer Customer { get; set; }
    public List<InvoiceItem> Items { get; set; }
}

// Toda la lógica de dominio está en servicios
public class InvoiceService
{
    private readonly IInvoiceRepository _repository;
    
    public InvoiceService(IInvoiceRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<Invoice> CreateInvoice(InvoiceCreateDto dto)
    {
        var invoice = new Invoice
        {
            Id = Guid.NewGuid(),
            Number = GenerateInvoiceNumber(),
            IssueDate = DateTime.Now,
            DueDate = DateTime.Now.AddDays(dto.PaymentTermDays),
            CustomerId = dto.CustomerId,
            Status = "Draft"
        };
        
        invoice.Items = dto.Items.Select(item => new InvoiceItem
        {
            Id = Guid.NewGuid(),
            InvoiceId = invoice.Id,
            Description = item.Description,
            Quantity = item.Quantity,
            UnitPrice = item.UnitPrice,
            Amount = item.Quantity * item.UnitPrice
        }).ToList();
        
        // Cálculos que deberían estar en la entidad
        invoice.Amount = invoice.Items.Sum(i => i.Amount);
        invoice.TaxAmount = CalculateTax(invoice.Amount);
        invoice.TotalAmount = invoice.Amount + invoice.TaxAmount;
        
        await _repository.Create(invoice);
        return invoice;
    }
    
    public async Task<bool> ApproveInvoice(Guid invoiceId)
    {
        var invoice = await _repository.GetById(invoiceId);
        if (invoice == null)
            return false;
            
        if (invoice.Status != "Draft")
            return false;
            
        // Cambios de estado que deberían estar en la entidad
        invoice.Status = "Approved";
        
        await _repository.Update(invoice);
        return true;
    }
    
    private decimal CalculateTax(decimal amount)
    {
        // Lógica de cálculo de impuestos que debería estar en la entidad
        return amount * 0.18m;
    }
    
    private string GenerateInvoiceNumber()
    {
        // Lógica de generación de números que debería estar en la entidad
        return $"INV-{DateTime.Now:yyyyMMdd}-{Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
    }
}
```
**Archivo:** `~\Compass\Compass.Invoicing.Domain\Models\Invoice.cs` y `~\Compass\Compass.Invoicing.Domain\Services\InvoiceService.cs`

## Consecuencias

1. **Violación del encapsulamiento**: La lógica relacionada con las entidades está dispersa en servicios
2. **Dificultad para mantener invariantes de dominio**: No hay garantía de que las entidades siempre estén en un estado válido
3. **Alto acoplamiento**: Los servicios necesitan conocer detalles internos de las entidades
4. **Reglas de negocio duplicadas**: La misma lógica puede implementarse inconsistentemente en diferentes servicios
5. **Difícil testabilidad**: Probar reglas de negocio requiere instanciar servicios con todas sus dependencias

## Solución Recomendada

```csharp
// Entidad de dominio rica con comportamiento encapsulado
public class Invoice
{
    private readonly List<InvoiceItem> _items = new();
    
    // Constructor privado - uso de método factory para garantizar invariantes
    private Invoice(Guid customerId, int paymentTermDays)
    {
        Id = Guid.NewGuid();
        CustomerId = customerId;
        Number = GenerateInvoiceNumber();
        IssueDate = DateTime.Now;
        DueDate = IssueDate.AddDays(paymentTermDays);
        Status = InvoiceStatus.Draft;
    }
    
    // Propiedades con acceso controlado
    public Guid Id { get; private set; }
    public string Number { get; private set; }
    public DateTime IssueDate { get; private set; }
    public DateTime DueDate { get; private set; }
    public decimal Amount { get; private set; }
    public decimal TaxAmount { get; private set; }
    public decimal TotalAmount { get; private set; }
    public InvoiceStatus Status { get; private set; }
    public Guid CustomerId { get; private set; }
    
    // Colección inmutable desde el exterior
    public IReadOnlyCollection<InvoiceItem> Items => _items.AsReadOnly();
    
    // Método factory que garantiza que la entidad se cree en un estado válido
    public static Invoice Create(Guid customerId, int paymentTermDays)
    {
        if (customerId == Guid.Empty)
            throw new DomainException("El ID de cliente es requerido");
            
        if (paymentTermDays <= 0)
            throw new DomainException("El plazo de pago debe ser mayor a cero");
            
        return new Invoice(customerId, paymentTermDays);
    }
    
    // Métodos que encapsulan comportamiento del dominio
    public void AddItem(string description, int quantity, decimal unitPrice)
    {
        if (Status != InvoiceStatus.Draft)
            throw new DomainException("Solo se pueden agregar ítems a facturas en borrador");
            
        if (quantity <= 0)
            throw new DomainException("La cantidad debe ser mayor a cero");
            
        if (unitPrice <= 0)
            throw new DomainException("El precio unitario debe ser mayor a cero");
            
        var item = new InvoiceItem(Id, description, quantity, unitPrice);
        _items.Add(item);
        
        RecalculateAmounts();
    }
    
    public void Approve()
    {
        if (Status != InvoiceStatus.Draft)
            throw new DomainException("Solo se pueden aprobar facturas en estado borrador");
            
        if (!_items.Any())
            throw new DomainException("La factura debe tener al menos un ítem para aprobarla");
            
        Status = InvoiceStatus.Approved;
    }
    
    public void Cancel(string reason)
    {
        if (Status == InvoiceStatus.Cancelled)
            throw new DomainException("La factura ya está cancelada");
            
        if (string.IsNullOrEmpty(reason))
            throw new DomainException("Debe proporcionar un motivo para la cancelación");
            
        Status = InvoiceStatus.Cancelled;
    }
    
    private void RecalculateAmounts()
    {
        Amount = _items.Sum(i => i.Amount);
        TaxAmount = CalculateTax(Amount);
        TotalAmount = Amount + TaxAmount;
    }
    
    private decimal CalculateTax(decimal amount)
    {
        // Lógica de cálculo de impuestos encapsulada en la entidad
        return amount * 0.18m;
    }
    
    private string GenerateInvoiceNumber()
    {
        return $"INV-{DateTime.Now:yyyyMMdd}-{Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
    }
}

// Clase de valor inmutable
public class InvoiceItem
{
    public InvoiceItem(Guid invoiceId, string description, int quantity, decimal unitPrice)
    {
        Id = Guid.NewGuid();
        InvoiceId = invoiceId;
        Description = description;
        Quantity = quantity;
        UnitPrice = unitPrice;
        Amount = quantity * unitPrice;
    }
    
    public Guid Id { get; }
    public Guid InvoiceId { get; }
    public string Description { get; }
    public int Quantity { get; }
    public decimal UnitPrice { get; }
    public decimal Amount { get; }
}

// Enum para estados en lugar de strings
public enum InvoiceStatus
{
    Draft,
    Approved,
    Paid,
    Cancelled
}

// Excepción de dominio para errores relacionados con las reglas de negocio
public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}

// Servicio de aplicación que usa las entidades ricas
public class InvoiceApplicationService
{
    private readonly IInvoiceRepository _repository;
    private readonly ICustomerRepository _customerRepository;
    
    public InvoiceApplicationService(
        IInvoiceRepository repository,
        ICustomerRepository customerRepository)
    {
        _repository = repository;
        _customerRepository = customerRepository;
    }
    
    public async Task<Result<Invoice>> CreateInvoice(InvoiceCreateDto dto)
    {
        try
        {
            // Verificar que el cliente existe
            var customer = await _customerRepository.GetById(dto.CustomerId);
            if (customer == null)
                return Result<Invoice>.Failure(Error.NotFound("Cliente", dto.CustomerId.ToString()));
                
            // Crear la factura usando el método factory de la entidad
            var invoice = Invoice.Create(dto.CustomerId, dto.PaymentTermDays);
            
            // Agregar ítems a través de métodos de comportamiento de la entidad
            foreach (var item in dto.Items)
            {
                invoice.AddItem(item.Description, item.Quantity, item.UnitPrice);
            }
            
            // Persistir la entidad
            await _repository.Create(invoice);
            return Result<Invoice>.Success(invoice);
        }
        catch (DomainException ex)
        {
            return Result<Invoice>.Failure(Error.Validation(ex.Message));
        }
    }
    
    public async Task<Result> ApproveInvoice(Guid invoiceId)
    {
        try
        {
            var invoice = await _repository.GetById(invoiceId);
            if (invoice == null)
                return Result.Failure(Error.NotFound("Factura", invoiceId.ToString()));
                
            // La entidad encapsula las reglas para aprobar una factura
            invoice.Approve();
            
            await _repository.Update(invoice);
            return Result.Success();
        }
        catch (DomainException ex)
        {
            return Result.Failure(Error.Validation(ex.Message));
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Encapsular la lógica de negocio directamente en las entidades de dominio
- Garantizar la consistencia de las reglas de negocio en toda la aplicación
- Promover la cohesión al mantener juntos los datos y su comportamiento relacionado
- Reducir la duplicación de código al centralizar las reglas en un solo lugar
- Proteger los invariantes del dominio utilizando constructores privados y métodos factory
- Mejorar la legibilidad y expresividad del código reflejando conceptos del negocio
- Facilitar las pruebas unitarias al poder probar entidades aisladamente de infraestructura

## Conclusión

Los modelos de dominio ricos son una parte fundamental del Domain-Driven Design y una buena práctica de la Programación Orientada a Objetos. Al dotar a las entidades con comportamiento, garantizamos que las reglas de negocio se mantengan consistentes y encapsuladas, lo que mejora significativamente la mantenibilidad del código. Este enfoque hace que el modelo refleje con mayor precisión los conceptos del negocio y permite evitar la complejidad adicional que supone tener que sincronizar el estado de las entidades desde servicios externos.

# Problema #15: Uso Inadecuado de Mapeo de Objetos

## Descripción del Problema

En la solución PAT se identificó un patrón problemático en el manejo de conversiones entre diferentes representaciones de objetos (entidades de dominio, DTOs, ViewModels), con mapeos manuales repetitivos, conversiones implícitas que ocultan transformaciones importantes, y uso inconsistente de herramientas de mapeo.

### Código Problemático

```csharp
// Mapeo manual repetitivo en diferentes servicios
public class CustomerService
{
    public CustomerDto GetCustomerById(Guid id)
    {
        var customer = _repository.GetById(id);
        
        // Mapeo manual con duplicación de lógica
        return new CustomerDto
        {
            Id = customer.Id,
            Name = customer.Name,
            Email = customer.Email,
            Phone = customer.PhoneNumber, // Nótese el cambio de nombre
            IsActive = customer.Status == CustomerStatus.Active, // Transformación implícita
            Address = new AddressDto
            {
                Street = customer.Address.Street,
                City = customer.Address.City,
                PostalCode = customer.Address.PostalCode,
                Country = customer.Address.Country
            },
            // Se omite LastUpdated deliberadamente
        };
    }
}

// En otra parte de la aplicación, similar mapeo con inconsistencias
public class CustomerController
{
    public async Task<IActionResult> GetCustomer(Guid id)
    {
        var customer = await _customerRepository.GetById(id);
        
        // Lógica duplicada pero ligeramente diferente
        var dto = new CustomerViewModel
        {
            CustomerId = customer.Id, // Nombre diferente
            FullName = customer.Name, // Nombre diferente
            EmailAddress = customer.Email, // Nombre diferente
            Phone = customer.PhoneNumber,
            Active = customer.Status == CustomerStatus.Active, // Nombre diferente
            // La dirección se mapea de manera diferente
            FullAddress = $"{customer.Address.Street}, {customer.Address.City}, {customer.Address.Country}",
            LastModified = customer.LastUpdated // Aquí sí se incluye
        };
        
        return View(dto);
    }
}

// Uso inconsistente de AutoMapper en otros servicios
public class OrderService
{
    private readonly IMapper _mapper;
    
    public OrderService(IMapper mapper)
    {
        _mapper = mapper;
    }
    
    public OrderDto GetOrderById(Guid id)
    {
        var order = _repository.GetById(id);
        
        // Uso de AutoMapper pero sin configuración clara
        var dto = _mapper.Map<OrderDto>(order);
        
        // Ajustes manuales post-mapeo
        dto.TotalCost = CalculateTotalCost(order); // Lógica que debería estar en el perfil de mapeo
        dto.CustomerName = GetCustomerName(order.CustomerId); // Información relacionada
        
        return dto;
    }
}
```
**Archivos:** `~\Compass\Compass.Core.Application\Services\CustomerService.cs`, `~\Compass\Compass.Web\Controllers\CustomerController.cs`, `~\Compass\Compass.Orders.Application\Services\OrderService.cs`

## Consecuencias

1. **Código duplicado**: La misma lógica de conversión se implementa múltiples veces
2. **Inconsistencias en mapeos**: Los mismos conceptos se traducen de manera diferente en distintas partes
3. **Pérdida de información**: Propiedades olvidadas en el mapeo manual sin advertencia
4. **Difícil mantenimiento**: Los cambios en las entidades requieren actualizar múltiples puntos de mapeo
5. **Ocultación de transformaciones**: Las conversiones implícitas esconden lógica de negocio importante
6. **Mezcla de responsabilidades**: Los servicios contienen tanto lógica de negocio como de mapeo

## Solución Recomendada

```csharp
// Definir perfiles de mapeo claros y centralizados
public class DomainToDtoMappingProfile : Profile
{
    public DomainToDtoMappingProfile()
    {
        CreateMap<Customer, CustomerDto>()
            .ForMember(dest => dest.Phone, opt => opt.MapFrom(src => src.PhoneNumber))
            .ForMember(dest => dest.IsActive, opt => opt.MapFrom(src => src.Status == CustomerStatus.Active))
            .ForMember(dest => dest.LastModified, opt => opt.MapFrom(src => src.LastUpdated));
            
        CreateMap<Address, AddressDto>();
        
        CreateMap<Customer, CustomerSummaryDto>()
            .ForMember(dest => dest.FullAddress, opt => opt.MapFrom(src => 
                $"{src.Address.Street}, {src.Address.City}, {src.Address.Country}"));
            
        CreateMap<Order, OrderDto>()
            .ForMember(dest => dest.TotalCost, opt => opt.MapFrom(src => 
                src.OrderItems.Sum(item => item.Quantity * item.UnitPrice)))
            .ForMember(dest => dest.CustomerName, opt => opt.Ignore()); // Se completa después con información relacionada
    }
}

// Configuración para UI específica
public class DtoToViewModelMappingProfile : Profile
{
    public DtoToViewModelMappingProfile()
    {
        CreateMap<CustomerDto, CustomerViewModel>()
            .ForMember(dest => dest.CustomerId, opt => opt.MapFrom(src => src.Id))
            .ForMember(dest => dest.FullName, opt => opt.MapFrom(src => src.Name))
            .ForMember(dest => dest.EmailAddress, opt => opt.MapFrom(src => src.Email))
            .ForMember(dest => dest.Active, opt => opt.MapFrom(src => src.IsActive));
    }
}

// Registro centralizado de perfiles de mapeo
public static class AutoMapperConfig
{
    public static void Configure(IServiceCollection services)
    {
        services.AddAutoMapper(cfg => {
            cfg.AddProfile<DomainToDtoMappingProfile>();
            cfg.AddProfile<DtoToViewModelMappingProfile>();
            
            // Validación en tiempo de arranque
            cfg.AllowNullCollections = true;
            cfg.ValidateInlineMaps = false;
        }, typeof(AutoMapperConfig).Assembly);
    }
}

// Uso consistente en servicios
public class CustomerService
{
    private readonly ICustomerRepository _repository;
    private readonly IMapper _mapper;
    
    public CustomerService(ICustomerRepository repository, IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }
    
    public CustomerDto GetCustomerById(Guid id)
    {
        var customer = _repository.GetById(id);
        if (customer == null)
            return null;
            
        return _mapper.Map<CustomerDto>(customer);
    }
    
    public CustomerSummaryDto GetCustomerSummary(Guid id)
    {
        var customer = _repository.GetById(id);
        if (customer == null)
            return null;
            
        return _mapper.Map<CustomerSummaryDto>(customer);
    }
}

// Uso consistente en controladores
public class CustomerController
{
    private readonly ICustomerService _customerService;
    private readonly IMapper _mapper;
    
    public CustomerController(ICustomerService customerService, IMapper mapper)
    {
        _customerService = customerService;
        _mapper = mapper;
    }
    
    public async Task<IActionResult> GetCustomer(Guid id)
    {
        var customerDto = await _customerService.GetCustomerById(id);
        if (customerDto == null)
            return NotFound();
            
        var viewModel = _mapper.Map<CustomerViewModel>(customerDto);
        
        return View(viewModel);
    }
}

// Para casos complejos, crear servicios específicos de mapeo
public class OrderMappingService
{
    private readonly IMapper _mapper;
    private readonly ICustomerRepository _customerRepository;
    
    public OrderMappingService(IMapper mapper, ICustomerRepository customerRepository)
    {
        _mapper = mapper;
        _customerRepository = customerRepository;
    }
    
    public OrderDto MapToDto(Order order)
    {
        var dto = _mapper.Map<OrderDto>(order);
        
        // Completar información relacionada
        var customer = _customerRepository.GetById(order.CustomerId);
        dto.CustomerName = customer?.Name ?? "Cliente desconocido";
        
        return dto;
    }
}

// Uso del servicio de mapeo especializado
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly OrderMappingService _mappingService;
    
    public OrderService(IOrderRepository repository, OrderMappingService mappingService)
    {
        _repository = repository;
        _mappingService = mappingService;
    }
    
    public OrderDto GetOrderById(Guid id)
    {
        var order = _repository.GetById(id);
        if (order == null)
            return null;
            
        return _mappingService.MapToDto(order);
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Centralizar la lógica de mapeo en perfiles y configuraciones claras y explícitas
- Asegurar consistencia en las conversiones de datos a través de toda la aplicación
- Detectar errores de mapeo faltantes durante la compilación o en tiempo de arranque
- Simplificar el mantenimiento al tener la lógica de conversión en un solo lugar
- Separar claramente los aspectos de mapeo de la lógica de negocio principal
- Documentar explícitamente las transformaciones importantes realizadas durante el mapeo
- Facilitar pruebas unitarias específicas para validar los mapeos

## Conclusión

El mapeo de objetos es una operación frecuente que merece una estrategia clara y consistente. En lugar de implementaciones manuales dispersas, es recomendable utilizar bibliotecas como AutoMapper con configuraciones centralizadas y explícitas. Este enfoque no solo reduce la duplicación de código, sino que también mejora la mantenibilidad, evita errores sutiles y documenta claramente las transformaciones realizadas durante los mapeos. Para casos complejos que requieren acceso a datos relacionados, es preferible crear servicios especializados de mapeo que encapsulen esta lógica.

# Problema #16: Uso Incorrecto de Async/Await

## Descripción del Problema

En la solución PAT se identificó un patrón problemático en el manejo de código asincrónico, incluyendo bloqueo de hilos, mezcla inconsistente de código sincrónico y asincrónico, uso de `async void`, y ausencia de manejo adecuado de cancelaciones, lo que afecta el rendimiento y la estabilidad del sistema.

### Código Problemático

```csharp
// Uso de .Result o .Wait() bloqueando hilos
public class DocumentController
{
    public IActionResult GetDocument(Guid id)
    {
        // Bloqueo del hilo utilizando .Result
        var document = _documentService.GetDocumentByIdAsync(id).Result;
        return View(document);
    }
}

// Async void en métodos que no son eventos
public async void ProcessDocuments()
{
    // Las excepciones aquí no serán capturadas por el caller
    // y podrían derribar el proceso
    await _documentService.ProcessAllPendingDocumentsAsync();
}

// Falta de propagación de CancellationToken
public async Task<IEnumerable<Document>> SearchDocumentsAsync(string query)
{
    // Sin CancellationToken, no se puede cancelar la operación
    var results = await _repository.SearchAsync(query);
    
    // Procesamiento potencialmente largo sin posibilidad de cancelación
    var processedResults = await ProcessResultsAsync(results);
    
    return processedResults;
}

// Mezcla de código sincrónico y asincrónico
public async Task<Document> SaveDocumentAsync(DocumentDto dto)
{
    // Operación sincrónica que podría ser costosa
    ValidateDocument(dto);
    
    // Operación asincrónica
    var document = await _mapper.MapAsync<Document>(dto);
    
    // Operación sincrónica que bloquea
    _logger.Log($"Saving document {document.Id}");
    
    // Operación asincrónica
    return await _repository.SaveAsync(document);
}

// Await innecesario en retornos
public async Task<IEnumerable<Document>> GetRecentDocumentsAsync()
{
    // Uso innecesario de await en return
    return await _repository.GetRecentAsync();
}

// ConfigureAwait olvidado en bibliotecas
public async Task<bool> ValidateDocumentAsync(Document document)
{
    // Sin ConfigureAwait(false), puede causar deadlocks en contextos sincrónicos
    var rules = await _rulesRepository.GetRulesAsync();
    var validator = new DocumentValidator(rules);
    return await validator.ValidateAsync(document);
}
```
**Archivos:** `~\Compass\Compass.Web\Controllers\DocumentController.cs`, `~\Compass\Compass.Core.Application\Services\DocumentService.cs`

## Consecuencias

1. **Bloqueo de hilos y posibles deadlocks**: El uso de `.Result` o `.Wait()` puede causar deadlocks en entornos con contexto de sincronización
2. **Excepciones no manejadas**: Los métodos `async void` no permiten la captura de excepciones desde el código llamante
3. **Operaciones zombies**: Sin propagación de `CancellationToken`, las operaciones continúan ejecutándose incluso cuando ya no son necesarias
4. **Uso ineficiente de recursos**: La mezcla de código sincrónico y asincrónico reduce los beneficios del modelo asíncrono
5. **Sobrecarga innecesaria**: El uso de `await` para retornar una tarea simplemente añade trabajo para el compilador

## Solución Recomendada

```csharp
// Uso correcto de async/await en controladores
public class DocumentController
{
    public async Task<IActionResult> GetDocument(Guid id)
    {
        // Uso adecuado de async/await sin bloqueo
        var document = await _documentService.GetDocumentByIdAsync(id);
        return View(document);
    }
}

// Uso de Task en lugar de async void
public Task ProcessDocumentsAsync()
{
    try
    {
        // Retorna la tarea para permitir seguimiento
        return _documentService.ProcessAllPendingDocumentsAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error processing documents");
        return Task.FromException(ex);
    }
}

// Propagación correcta de CancellationToken
public async Task<IEnumerable<Document>> SearchDocumentsAsync(
    string query, 
    CancellationToken cancellationToken = default)
{
    // CancellationToken propagado a todas las operaciones asíncronas
    var results = await _repository.SearchAsync(query, cancellationToken);
    
    // Propagar token a todas las operaciones anidadas
    var processedResults = await ProcessResultsAsync(results, cancellationToken);
    
    return processedResults;
}

// Consistencia en operaciones asíncronas
public async Task<Document> SaveDocumentAsync(
    DocumentDto dto, 
    CancellationToken cancellationToken = default)
{
    // Convertir operaciones síncronas costosas a asíncronas
    await ValidateDocumentAsync(dto, cancellationToken);
    
    var document = await _mapper.MapAsync<Document>(dto, cancellationToken);
    
    // Usar métodos asincrónicos para todas las operaciones que podrían bloquear
    await _logger.LogAsync($"Saving document {document.Id}", cancellationToken);
    
    return await _repository.SaveAsync(document, cancellationToken);
}

// Retorno directo de tareas cuando no es necesario await
public Task<IEnumerable<Document>> GetRecentDocumentsAsync(
    CancellationToken cancellationToken = default)
{
    // Retorno directo de la tarea sin await innecesario
    return _repository.GetRecentAsync(cancellationToken);
}

// Uso correcto de ConfigureAwait en bibliotecas
public async Task<bool> ValidateDocumentAsync(
    Document document, 
    CancellationToken cancellationToken = default)
{
    // ConfigureAwait(false) para evitar volver al contexto de sincronización
    // original cuando no es necesario
    var rules = await _rulesRepository.GetRulesAsync(cancellationToken)
        .ConfigureAwait(false);
        
    var validator = new DocumentValidator(rules);
    
    return await validator.ValidateAsync(document, cancellationToken)
        .ConfigureAwait(false);
}

// Implementación de patrón de cancelación para operaciones de larga duración
public class DocumentProcessingService
{
    public async Task ProcessBatchAsync(
        IEnumerable<Document> documents, 
        IProgress<int> progress = null,
        CancellationToken cancellationToken = default)
    {
        int total = documents.Count();
        int processed = 0;
        
        foreach (var document in documents)
        {
            // Verificar cancelación antes de cada operación costosa
            cancellationToken.ThrowIfCancellationRequested();
            
            await ProcessDocumentAsync(document, cancellationToken);
            
            processed++;
            
            // Reportar progreso si se proporciona un IProgress
            progress?.Report((int)((float)processed / total * 100));
        }
    }
    
    // Implementación de timeout con CancellationToken
    public async Task<Document> GetDocumentWithTimeoutAsync(
        Guid id, 
        TimeSpan timeout)
    {
        using var cts = new CancellationTokenSource(timeout);
        try
        {
            return await _repository.GetByIdAsync(id, cts.Token);
        }
        catch (OperationCanceledException) when (cts.IsCancellationRequested)
        {
            throw new TimeoutException($"Operation to get document {id} timed out after {timeout.TotalSeconds} seconds");
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Evitar deadlocks y bloqueos de hilos eliminando el uso de `.Result` y `.Wait()`
- Mejorar el manejo de excepciones usando `Task` en lugar de `async void`
- Permitir la cancelación de operaciones propagando `CancellationToken` a través de la cadena de llamadas
- Incrementar la eficiencia aprovechando completamente el modelo asíncrono
- Reducir la sobrecarga evitando el uso innecesario de `await` al retornar tareas
- Evitar posibles deadlocks en bibliotecas mediante el uso adecuado de `ConfigureAwait(false)`
- Proporcionar retroalimentación durante operaciones largas usando el patrón `IProgress<T>`

## Conclusión

El uso adecuado del modelo de programación asincrónico requiere seguir ciertas prácticas y patrones para evitar problemas sutiles que pueden afectar seriamente el rendimiento y la estabilidad de la aplicación. La consistencia en la propagación de tokens de cancelación, el uso correcto de `ConfigureAwait(false)` en bibliotecas, y evitar el bloqueo de hilos son clave para desarrollar aplicaciones asincrónicas robustas. Adoptar estas prácticas mejora la escalabilidad, la capacidad de respuesta y la utilización de recursos del sistema.

# Problema #17: Inconsistencia y Deficiencias en el Manejo de Respuestas de API

## Descripción del Problema

En la solución PAT se identificó un manejo inconsistente de respuestas HTTP en la capa de API, donde diferentes controladores implementan sus propios enfoques de respuesta, códigos de estado, y formatos de errores. Esta falta de estandarización dificulta el consumo de la API por clientes externos y complica el tratamiento de errores en el frontend.

### Código Problemático

```csharp
// Controlador que devuelve diferentes tipos de respuestas sin consistencia
public class DocumentsController : ControllerBase
{
    private readonly IDocumentService _documentService;

    public DocumentsController(IDocumentService documentService)
    {
        _documentService = documentService;
    }

    // Devuelve 200 aunque ocurra un error (en el cuerpo)
    [HttpGet("search")]
    public async Task<ActionResult<List<DocumentDto>>> SearchDocuments(string query)
    {
        try
        {
            var results = await _documentService.SearchAsync(query);
            return Ok(results);
        }
        catch (Exception ex)
        {
            // Devuelve 200 OK con un objeto de error en el cuerpo
            return Ok(new { success = false, error = ex.Message });
        }
    }

    // Devuelve null cuando no encuentra resultados
    [HttpGet("{id}")]
    public async Task<ActionResult<DocumentDto>> GetDocument(Guid id)
    {
        var document = await _documentService.GetByIdAsync(id);
        
        // Devuelve 200 con null en vez de 404
        return Ok(document);
    }

    // No incluye detalles de errores de validación
    [HttpPost]
    public async Task<IActionResult> CreateDocument(CreateDocumentDto dto)
    {
        if (!ModelState.IsValid)
            return BadRequest(); // Sin detalles de errores de validación

        try
        {
            var result = await _documentService.CreateDocumentAsync(dto);
            if (result.Success)
                return Created($"/api/documents/{result.DocumentId}", null);
            else
                return BadRequest(result.Message);
        }
        catch (Exception ex)
        {
            // Log pero sin información útil en la respuesta
            _logger.LogError(ex, "Error creating document");
            return StatusCode(500, "An internal error occurred");
        }
    }
}

// Otro controlador con formato de respuesta diferente
public class PaymentsController : ControllerBase
{
    // Formato de respuesta diferente al de DocumentsController
    [HttpGet("{id}")]
    public async Task<IActionResult> GetPayment(Guid id)
    {
        try
        {
            var payment = await _paymentService.GetByIdAsync(id);
            
            if (payment == null)
                return NotFound(new { code = "PAYMENT_NOT_FOUND", message = "Payment not found" });
                
            return Ok(new { result = payment, status = "success" });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { code = "INTERNAL_ERROR", message = ex.Message });
        }
    }
    
    // Respuestas con estructura inconsistente
    [HttpPost("process")]
    public async Task<IActionResult> ProcessPayment(ProcessPaymentDto dto)
    {
        // Validación manual con formato diferente
        if (dto.Amount <= 0)
        {
            return BadRequest("Amount must be positive");  // String simple en vez de objeto
        }
        
        var result = await _paymentService.ProcessPaymentAsync(dto);
        
        if (result.Success)
            return Ok(result);  // Objeto completo de resultado
        else
            return BadRequest(result.Error);  // Solo el mensaje de error
    }
}
```
**Archivos:** `~\Compass\Compass.Api\Controllers\DocumentsController.cs`, `~\Compass\Compass.Api\Controllers\PaymentsController.cs`

## Consecuencias

1. **Inconsistencia para clientes**: Diferentes endpoints devuelven formatos de respuesta distintos
2. **Manejo de errores complejo**: Los clientes necesitan código específico para cada endpoint
3. **Problemas de integración**: Dificulta la creación de interceptores y middleware genéricos
4. **Errores ocultos**: Códigos de estado incorrectos (200 para errores) complican la detección
5. **Experiencia deficiente de API**: Falta estandarización en los patrones de respuesta
6. **Dificultad de depuración**: Mensajes de error inconsistentes complican la identificación de problemas

## Solución Recomendada

```csharp
// Definir una estructura estándar de respuesta para la API
public class ApiResponse<T>
{
    public bool Success { get; }
    public T Data { get; }
    public ApiError Error { get; }
    
    private ApiResponse(bool success, T data, ApiError error)
    {
        Success = success;
        Data = data;
        Error = error;
    }
    
    public static ApiResponse<T> SuccessResponse(T data) =>
        new ApiResponse<T>(true, data, null);
        
    public static ApiResponse<T> ErrorResponse(ApiError error) =>
        new ApiResponse<T>(false, default, error);
}

// Estructura estandarizada para errores
public class ApiError
{
    public string Code { get; set; }
    public string Message { get; set; }
    public Dictionary<string, string[]> ValidationErrors { get; set; }
    
    public static ApiError FromException(Exception ex, string code = "INTERNAL_ERROR")
    {
        return new ApiError
        {
            Code = code,
            Message = ex.Message
        };
    }
    
    public static ApiError FromModelState(ModelStateDictionary modelState)
    {
        var errors = modelState
            .Where(e => e.Value.Errors.Count > 0)
            .ToDictionary(
                kvp => kvp.Key,
                kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
            );
            
        return new ApiError
        {
            Code = "VALIDATION_ERROR",
            Message = "One or more validation errors occurred",
            ValidationErrors = errors
        };
    }
    
    public static ApiError NotFound(string resource, string id)
    {
        return new ApiError
        {
            Code = "NOT_FOUND",
            Message = $"{resource} with id {id} not found"
        };
    }
}

// Extensiones para simplificar el retorno de respuestas consistentes
public static class ControllerExtensions
{
    public static ActionResult<ApiResponse<T>> ApiOk<T>(this ControllerBase controller, T data)
    {
        return controller.Ok(ApiResponse<T>.SuccessResponse(data));
    }
    
    public static ActionResult<ApiResponse<T>> ApiNotFound<T>(this ControllerBase controller, string resource, string id)
    {
        var response = ApiResponse<T>.ErrorResponse(
            ApiError.NotFound(resource, id));
        return controller.NotFound(response);
    }
    
    public static ActionResult<ApiResponse<T>> ApiBadRequest<T>(this ControllerBase controller, string code, string message)
    {
        var response = ApiResponse<T>.ErrorResponse(
            new ApiError { Code = code, Message = message });
        return controller.BadRequest(response);
    }
    
    public static ActionResult<ApiResponse<T>> ApiValidationError<T>(this ControllerBase controller)
    {
        var response = ApiResponse<T>.ErrorResponse(
            ApiError.FromModelState(controller.ModelState));
        return controller.BadRequest(response);
    }
    
    public static ActionResult<ApiResponse<T>> ApiServerError<T>(this ControllerBase controller, Exception ex)
    {
        var response = ApiResponse<T>.ErrorResponse(
            ApiError.FromException(ex));
        return controller.StatusCode(500, response);
    }
}

// Implementación de un filtro global para manejar excepciones
public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ApiExceptionFilter> _logger;
    private readonly IHostEnvironment _env;
    
    public ApiExceptionFilter(
        ILogger<ApiExceptionFilter> logger,
        IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }
    
    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");
        
        var error = new ApiError
        {
            Code = "INTERNAL_SERVER_ERROR",
            Message = _env.IsDevelopment() 
                ? context.Exception.ToString() 
                : "An unexpected error occurred"
        };
        
        var response = ApiResponse<object>.ErrorResponse(error);
        
        context.Result = new ObjectResult(response)
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };
        
        context.ExceptionHandled = true;
    }
}

// Controlador refactorizado usando el enfoque estandarizado
public class DocumentsController : ControllerBase
{
    private readonly IDocumentService _documentService;

    public DocumentsController(IDocumentService documentService)
    {
        _documentService = documentService;
    }

    // Uso consistente de respuestas API
    [HttpGet("search")]
    public async Task<ActionResult<ApiResponse<List<DocumentDto>>>> SearchDocuments(string query)
    {
        try
        {
            var results = await _documentService.SearchAsync(query);
            return this.ApiOk(results);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error searching documents with query {Query}", query);
            return this.ApiServerError<List<DocumentDto>>(ex);
        }
    }

    // Manejo adecuado de recursos no encontrados
    [HttpGet("{id}")]
    public async Task<ActionResult<ApiResponse<DocumentDto>>> GetDocument(Guid id)
    {
        var document = await _documentService.GetByIdAsync(id);
        
        if (document == null)
            return this.ApiNotFound<DocumentDto>("Document", id.ToString());
            
        return this.ApiOk(document);
    }

    // Información detallada de errores de validación
    [HttpPost]
    public async Task<ActionResult<ApiResponse<DocumentCreatedDto>>> CreateDocument(CreateDocumentDto dto)
    {
        if (!ModelState.IsValid)
            return this.ApiValidationError<DocumentCreatedDto>();

        try
        {
            var result = await _documentService.CreateDocumentAsync(dto);
            if (result.Success)
            {
                // Crear DTO específico para la respuesta
                var responseDto = new DocumentCreatedDto
                {
                    Id = result.DocumentId,
                    CreatedAt = DateTime.UtcNow
                };
                
                // URI de ubicación en header + respuesta estandarizada
                return Created(
                    $"/api/documents/{result.DocumentId}", 
                    ApiResponse<DocumentCreatedDto>.SuccessResponse(responseDto));
            }
            else
            {
                return this.ApiBadRequest<DocumentCreatedDto>(
                    "DOCUMENT_CREATION_FAILED", 
                    result.Message);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating document");
            return this.ApiServerError<DocumentCreatedDto>(ex);
        }
    }
}

// Configuración global en Startup
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers(options =>
    {
        // Registrar el filtro global de excepciones
        options.Filters.Add<ApiExceptionFilter>();
    })
    .ConfigureApiBehaviorOptions(options =>
    {
        // Personalizar la respuesta de validación de modelos automática
        options.InvalidModelStateResponseFactory = context =>
        {
            var error = ApiError.FromModelState(context.ModelState);
            var response = ApiResponse<object>.ErrorResponse(error);
            return new BadRequestObjectResult(response);
        };
    });
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar un formato de respuesta unificado para todos los endpoints de la API
- Estandarizar la estructura de errores con códigos específicos y mensajes descriptivos
- Garantizar el uso correcto de códigos de estado HTTP según la situación
- Facilitar el manejo de errores del lado del cliente con formatos predecibles
- Incluir detalles de validación que permiten identificar rápidamente problemas en las solicitudes
- Centralizar la captura de excepciones no manejadas con el filtro global
- Mejorar la experiencia de desarrollo tanto para consumidores internos como externos de la API

## Conclusión

Una arquitectura de API bien diseñada debe proporcionar respuestas coherentes y predecibles para facilitar su integración y uso. Estandarizar el formato de respuestas, utilizar correctamente los códigos de estado HTTP y proporcionar mensajes de error significativos mejoran significativamente la calidad de la API. Este enfoque no solo beneficia a los consumidores externos, sino que también facilita la implementación de interceptores genéricos y middleware para manejar aspectos transversales como autenticación, registro y tratamiento de errores.

# Problema #18: Prácticas de Logging Impropias

## Descripción del Problema

En la solución PAT se identificó un patrón problemático en las prácticas de registro (logging), donde la información se registra de manera inconsistente, con niveles de log inadecuados, información sensible expuesta, y sin un contexto suficiente para diagnóstico efectivo en entornos de producción.

### Código Problemático

```csharp
// Uso de strings de interpolación directamente en logs
public class PaymentProcessor
{
    private readonly ILogger _logger;
    
    public async Task ProcessPayment(PaymentRequest payment)
    {
        try
        {
            _logger.LogInformation($"Iniciando procesamiento de pago para usuario {payment.UserId}. " +
                $"Tarjeta: {payment.CreditCardNumber}, Monto: {payment.Amount}");
            
            // Procesamiento del pago...
            
            _logger.LogInformation("Pago procesado correctamente");
        }
        catch (Exception ex)
        {
            // Excepción completa en logs de información
            _logger.LogInformation($"Error procesando pago: {ex}");
            throw;
        }
    }
}

// Logs inconsistentes o ausentes en flujos de control críticos
public class DocumentService
{
    private readonly ILogger<DocumentService> _logger;
    
    public async Task<List<Document>> GetDocumentsByStatus(DocumentStatus status)
    {
        // Sin registro al iniciar operación
        var documents = await _repository.GetByStatusAsync(status);
        
        // Condicionales sin logs apropiados
        if (!documents.Any())
        {
            // No se registra esta condición potencialmente anómala
            return new List<Document>();
        }
        
        // Log insuficiente al finalizar operación exitosa
        return documents;
    }
    
    public async Task<bool> ApproveDocument(Guid id)
    {
        try
        {
            var document = await _repository.GetByIdAsync(id);
            
            if (document == null)
            {
                // No hay registro de este error de negocio
                return false;
            }
            
            document.Status = DocumentStatus.Approved;
            document.ApprovedDate = DateTime.UtcNow;
            await _repository.UpdateAsync(document);
            
            // No hay registro del éxito de la operación
            return true;
        }
        catch (Exception ex)
        {
            // Mensaje genérico sin contexto
            _logger.LogError("Error al aprobar documento");
            return false;
        }
    }
}

// Falta de correlación entre logs relacionados
public class WorkflowOrchestrator
{
    private readonly ILogger _logger;
    
    public async Task ExecuteWorkflow(WorkflowDefinition workflow)
    {
        _logger.LogInformation("Iniciando workflow");
        
        foreach (var step in workflow.Steps)
        {
            // No hay correlación entre los logs de diferentes pasos
            _logger.LogInformation($"Ejecutando paso: {step.Name}");
            await ExecuteStep(step);
        }
        
        _logger.LogInformation("Workflow completado");
    }
    
    private async Task ExecuteStep(WorkflowStep step)
    {
        // No hay forma de correlacionar este log con el workflow principal
        _logger.LogDebug($"Detalles del paso: {step.Description}");
        
        try
        {
            await step.Execute();
        }
        catch (Exception ex)
        {
            // No incluye ID del workflow ni del paso
            _logger.LogError(ex, "Error ejecutando paso");
            throw;
        }
    }
}

// Uso incorrecto de niveles de log
public class UserService
{
    private readonly ILogger<UserService> _logger;
    
    public async Task<User> RegisterUser(UserRegistrationRequest request)
    {
        // Información crítica registrada como Debug
        _logger.LogDebug($"Nuevo usuario registrado: {request.Email}, Perfil: {request.ProfileType}");
        
        // Operación de rutina registrada como Warning
        _logger.LogWarning("Inicio del proceso de verificación de usuario");
        
        // Registro innecesario como Error para casos normales
        if (request.RequiresManagerApproval)
        {
            _logger.LogError("Usuario requiere aprobación del gerente");
        }
        
        return await _userRepository.CreateAsync(request.ToUser());
    }
}
```
**Archivos:** `~\Compass\Compass.Payments.Domain\Services\PaymentProcessor.cs`, `~\Compass\Compass.Documents.Domain\Services\DocumentService.cs`, `~\Compass\Compass.Workflows.Application\Services\WorkflowOrchestrator.cs`, `~\Compass\Compass.Users.Application\Services\UserService.cs`

## Consecuencias

1. **Dificultad para diagnosticar problemas**: Logs insuficientes o inadecuados que no proporcionan el contexto necesario
2. **Exposición de datos sensibles**: Información confidencial registrada en texto plano en los logs
3. **Ruido vs. señal**: Niveles de log incorrectos que dificultan distinguir eventos importantes de los rutinarios
4. **Incapacidad para seguir flujos completos**: Falta de correlación entre logs relacionados
5. **Degradación del rendimiento**: Logs excesivos o ineficientes que afectan el desempeño de la aplicación
6. **Problemas de cumplimiento normativo**: Registro inadecuado de eventos auditables requeridos por regulaciones

## Solución Recomendada

```csharp
// Definir una estrategia de logging estructurado
public class PaymentProcessor
{
    private readonly ILogger<PaymentProcessor> _logger;
    
    public async Task ProcessPayment(PaymentRequest payment)
    {
        // Creación de un scope para correlacionar todos los logs de esta operación
        using var scope = _logger.BeginScope(new Dictionary<string, object> 
        {
            ["PaymentId"] = payment.Id,
            ["UserId"] = payment.UserId,
            ["OperationType"] = "PaymentProcessing"
        });
        
        try
        {
            // Uso de mensajes de plantilla y no exposición de datos sensibles
            _logger.LogInformation("Iniciando procesamiento de pago {PaymentId} para usuario {UserId} por {Amount}",
                payment.Id, payment.UserId, payment.Amount);
            
            // Enmascarar información sensible
            _logger.LogDebug("Detalles de pago: {LastFourDigits}", 
                payment.CreditCardNumber.Substring(payment.CreditCardNumber.Length - 4).PadLeft(16, '*'));
            
            // Procesamiento del pago...
            
            _logger.LogInformation("Pago {PaymentId} procesado correctamente", payment.Id);
        }
        catch (Exception ex)
        {
            // Nivel de log apropiado para errores y con contexto enriquecido
            _logger.LogError(ex, "Error procesando pago {PaymentId}. Tipo: {ErrorType}", 
                payment.Id, ex.GetType().Name);
            throw;
        }
    }
}

// Uso consistente de logs para flujos de control
public class DocumentService
{
    private readonly ILogger<DocumentService> _logger;
    
    public async Task<List<Document>> GetDocumentsByStatus(DocumentStatus status)
    {
        _logger.LogDebug("Buscando documentos con estado {Status}", status);
        
        var documents = await _repository.GetByStatusAsync(status);
        
        if (!documents.Any())
        {
            _logger.LogInformation("No se encontraron documentos con estado {Status}", status);
            return new List<Document>();
        }
        
        _logger.LogDebug("Se encontraron {Count} documentos con estado {Status}", 
            documents.Count, status);
        
        return documents;
    }
    
    public async Task<Result<Document>> ApproveDocument(Guid id)
    {
        _logger.LogInformation("Intentando aprobar documento {DocumentId}", id);
        
        try
        {
            var document = await _repository.GetByIdAsync(id);
            
            if (document == null)
            {
                _logger.LogWarning("Documento {DocumentId} no encontrado para aprobación", id);
                return Result<Document>.Failure(Error.NotFound("Document", id.ToString()));
            }
            
            document.Status = DocumentStatus.Approved;
            document.ApprovedDate = DateTime.UtcNow;
            await _repository.UpdateAsync(document);
            
            _logger.LogInformation("Documento {DocumentId} aprobado exitosamente por {UserId} en {ApprovalDate}",
                id, _currentUserService.UserId, document.ApprovedDate);
            
            return Result<Document>.Success(document);
        }
        catch (Exception ex)
        {
            // Registro estructurado de la excepción con contexto significativo
            _logger.LogError(ex, "Error al aprobar documento {DocumentId}. Error: {ErrorMessage}",
                id, ex.Message);
            
            return Result<Document>.Failure(Error.Unexpected(ex));
        }
    }
}

// Uso de correlación entre logs relacionados
public class WorkflowOrchestrator
{
    private readonly ILogger<WorkflowOrchestrator> _logger;
    
    public async Task ExecuteWorkflow(WorkflowDefinition workflow)
    {
        // Generar un ID de correlación para todo el workflow
        var correlationId = Guid.NewGuid().ToString();
        
        // Crear un scope enriquecido con datos del contexto
        using var workflowScope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId,
            ["WorkflowId"] = workflow.Id,
            ["WorkflowType"] = workflow.Type
        });
        
        _logger.LogInformation("Iniciando workflow {WorkflowName} con {StepCount} pasos", 
            workflow.Name, workflow.Steps.Count);
        
        try
        {
            for (int i = 0; i < workflow.Steps.Count; i++)
            {
                var step = workflow.Steps[i];
                
                // El scope anidado hereda el correlationId del scope padre
                using var stepScope = _logger.BeginScope(new Dictionary<string, object>
                {
                    ["StepId"] = step.Id,
                    ["StepName"] = step.Name,
                    ["StepIndex"] = i + 1
                });
                
                _logger.LogInformation("Ejecutando paso {StepIndex}/{TotalSteps}: {StepName}", 
                    i + 1, workflow.Steps.Count, step.Name);
                
                await ExecuteStep(step, correlationId);
            }
            
            _logger.LogInformation("Workflow {WorkflowName} completado exitosamente", workflow.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error ejecutando workflow {WorkflowName}. Detalles: {ErrorDetails}", 
                workflow.Name, ex.Message);
            throw;
        }
    }
    
    private async Task ExecuteStep(WorkflowStep step, string correlationId)
    {
        _logger.LogDebug("Iniciando ejecución del paso {StepName} con parámetros: {@StepParameters}",
            step.Name, step.Parameters);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await step.Execute();
            stopwatch.Stop();
            
            _logger.LogInformation("Paso {StepName} ejecutado correctamente en {ElapsedMilliseconds}ms", 
                step.Name, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            _logger.LogError(ex, 
                "Error ejecutando paso {StepName} después de {ElapsedMilliseconds}ms. Error: {ErrorMessage}", 
                step.Name, stopwatch.ElapsedMilliseconds, ex.Message);
            
            throw;
        }
    }
}

// Uso correcto de niveles de log y formato estructurado
public class UserService
{
    private readonly ILogger<UserService> _logger;
    
    public async Task<User> RegisterUser(UserRegistrationRequest request)
    {
        // Usar nivel apropiado para operación de negocio normal
        _logger.LogInformation("Registrando nuevo usuario con email {Email} y perfil {ProfileType}", 
            request.Email, request.ProfileType);
        
        // Log de debug para información detallada
        _logger.LogDebug("Detalles de registro: {@RegistrationDetails}", 
            new { 
                Email = request.Email, 
                ProfileType = request.ProfileType,
                Source = request.RegistrationSource,
                Features = request.EnabledFeatures
            });
        
        // Warning para condiciones excepcionales pero esperadas
        if (request.RequiresManagerApproval)
        {
            _logger.LogWarning("Usuario {Email} requiere aprobación del gerente", request.Email);
        }
        
        var user = await _userRepository.CreateAsync(request.ToUser());
        
        // Registrar resultado con datos relevantes para auditoría
        _logger.LogInformation("Usuario {UserId} registrado exitosamente con email {Email}", 
            user.Id, user.Email);
        
        return user;
    }
}

// Configuración centralizada para logging
public class LoggingConfiguration
{
    public static void Configure(ILoggingBuilder loggingBuilder, IConfiguration config)
    {
        loggingBuilder.ClearProviders();
        
        // Configurar Serilog como proveedor
        Log.Logger = new LoggerConfiguration()
            .ReadFrom.Configuration(config)
            .Enrich.FromLogContext()
            .Enrich.WithMachineName()
            .Enrich.WithEnvironmentName()
            .Filter.ByExcluding(Matching.WithProperty<string>("CreditCardNumber", 
                p => !string.IsNullOrEmpty(p) && !p.All(c => c == '*')))
            .WriteTo.Console(outputTemplate: 
                "{Timestamp:yyyy-MM-dd HH:mm:ss} [{Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}")
            .WriteTo.File(new CompactJsonFormatter(), 
                path: "logs/compass-.log", 
                rollingInterval: RollingInterval.Day)
            .CreateLogger();
        
        loggingBuilder.AddSerilog(dispose: true);
    }
}

// Middleware para agregar CorrelationId automáticamente
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    
    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context, ILogger<CorrelationIdMiddleware> logger)
    {
        string correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault() ?? 
            Guid.NewGuid().ToString();
            
        // Agregar a los headers de respuesta
        context.Response.OnStarting(() => {
            context.Response.Headers.Add("X-Correlation-ID", correlationId);
            return Task.CompletedTask;
        });
        
        // Crear un scope con el correlationId que será heredado por todos los logs
        using (logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        {
            await _next(context);
        }
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar contexto consistente y estructurado en todos los logs de la aplicación
- Proteger datos sensibles mediante enmascaramiento y filtrado automático
- Utilizar correctamente los niveles de log según la severidad real de cada evento
- Facilitar el seguimiento de flujos completos mediante IDs de correlación
- Mejorar el diagnóstico con información detallada sobre excepciones y su contexto
- Optimizar el rendimiento utilizando mensajes de plantilla en lugar de interpolación de strings
- Permitir consultas y filtros eficientes en herramientas de análisis de logs
- Facilitar el cumplimiento de requisitos de auditoría y regulaciones de datos

## Conclusión

Una estrategia de logging bien implementada es crucial para la operación, mantenimiento y diagnóstico de aplicaciones empresariales. El logging estructurado, con niveles adecuados, datos de contexto enriquecidos y mecanismos de correlación, permite identificar y resolver problemas rápidamente en entornos de producción. Además, una práctica adecuada de logging debe balancear la necesidad de información diagnóstica con la protección de datos sensibles y el rendimiento del sistema. La inversión en una buena arquitectura de logging desde el inicio del proyecto paga dividendos significativos cuando se necesita investigar comportamientos anómalos en producción.

# Problema #19: Testings Automáticos y TDD Insuficientes

## Descripción del Problema

En la solución PAT se identificó una cobertura insuficiente de pruebas automatizadas y la ausencia de prácticas de desarrollo guiado por pruebas (TDD), lo que resulta en una baja confiabilidad del código, dificultad para refactorizar con seguridad, y un ciclo de retroalimentación lento durante el desarrollo.

### Código Problemático

```csharp
// Servicio complejo con lógica de negocio crítica pero sin pruebas
public class PaymentCalculationService
{
    private readonly IExchangeRateService _exchangeRateService;
    private readonly ITaxCalculator _taxCalculator;
    private readonly IDiscountService _discountService;
    
    public PaymentCalculationService(
        IExchangeRateService exchangeRateService,
        ITaxCalculator taxCalculator,
        IDiscountService discountService)
    {
        _exchangeRateService = exchangeRateService;
        _taxCalculator = taxCalculator;
        _discountService = discountService;
    }
    
    public async Task<decimal> CalculatePaymentAmount(Invoice invoice, string targetCurrency)
    {
        // Lógica compleja con múltiples caminos y condiciones
        if (invoice == null)
            throw new ArgumentNullException(nameof(invoice));
            
        if (string.IsNullOrEmpty(targetCurrency))
            throw new ArgumentException("Target currency is required", nameof(targetCurrency));
        
        decimal subtotal = invoice.Items.Sum(item => item.Quantity * item.UnitPrice);
        
        // Aplicar descuentos si corresponde
        decimal discount = 0;
        if (invoice.Customer.IsPreferred && subtotal > 1000)
        {
            discount = await _discountService.CalculateDiscount(invoice.Customer, subtotal);
        }
        
        // Calcular impuestos
        decimal taxableAmount = subtotal - discount;
        decimal taxes = await _taxCalculator.CalculateTax(taxableAmount, invoice.BillingAddress.Country);
        
        // Calcular total en moneda original
        decimal totalInOriginalCurrency = taxableAmount + taxes;
        
        // Convertir a moneda de destino si es diferente
        if (invoice.Currency != targetCurrency)
        {
            var rate = await _exchangeRateService.GetExchangeRate(invoice.Currency, targetCurrency);
            return totalInOriginalCurrency * rate;
        }
        
        return totalInOriginalCurrency;
    }
}

// Controlador con lógica de negocio pero sin pruebas
public class PaymentController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    
    public PaymentController(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
    
    [HttpPost("process")]
    public async Task<IActionResult> ProcessPayment(PaymentRequest request)
    {
        // Validación manual sin pruebas
        if (request.Amount <= 0)
            return BadRequest("Amount must be positive");
            
        if (string.IsNullOrEmpty(request.CreditCardNumber))
            return BadRequest("Credit card number is required");
            
        if (request.CreditCardNumber.Length < 15 || request.CreditCardNumber.Length > 16)
            return BadRequest("Invalid credit card number format");
            
        if (request.ExpirationYear < DateTime.Now.Year || 
            (request.ExpirationYear == DateTime.Now.Year && request.ExpirationMonth < DateTime.Now.Month))
            return BadRequest("Card has expired");
        
        // Lógica de negocio en controlador - difícil de probar aisladamente
        try
        {
            var result = await _paymentService.ProcessPaymentAsync(request);
            return Ok(result);
        }
        catch (InsufficientFundsException)
        {
            return BadRequest("Insufficient funds");
        }
        catch (InvalidCardException)
        {
            return BadRequest("Invalid card information");
        }
        catch (Exception ex)
        {
            return StatusCode(500, "An error occurred while processing your payment");
        }
    }
}

// Código que es difícil de probar por dependencias estáticas
public class DocumentGenerator
{
    public byte[] GenerateInvoicePdf(Invoice invoice)
    {
        // Dependencia estática difícil de simular en pruebas
        var license = LicenseManager.ValidateLicense();
        if (!license.IsValid)
            throw new InvalidOperationException("PDF generation license is not valid");
        
        // Acceso directo a sistema de archivos - difícil de probar
        var templatePath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData),
            "Templates", "InvoiceTemplate.html");
        
        var template = File.ReadAllText(templatePath);
        
        // Acceso a fecha actual - impredecible en pruebas
        var generatedDate = DateTime.Now;
        
        // Lógica de generación PDF usando dependencias difíciles de mockear
        // ...
        
        return null; // Simulación de retorno
    }
}
```
**Archivos:** `~\Compass\Compass.Payments.Domain\Services\PaymentCalculationService.cs`, `~\Compass\Compass.Api\Controllers\PaymentController.cs`, `~\Compass\Compass.Documents.Infrastructure\Services\DocumentGenerator.cs`

## Consecuencias

1. **Baja confiabilidad del código**: Cambios que introducen regresiones pueden pasar desapercibidos
2. **Dificultad para refactorizar**: Sin pruebas automatizadas, los desarrolladores evitan refactorizar por miedo a introducir errores
3. **Ciclo de retroalimentación lento**: Los errores se detectan tarde en el ciclo de desarrollo, aumentando el costo de corrección
4. **Documentación viva insuficiente**: Las pruebas sirven como documentación ejecutable de cómo debe comportarse el sistema
5. **Diseño deficiente**: La ausencia de TDD lleva a componentes con alta cohesión y bajo desacoplamiento
6. **Deuda técnica creciente**: La falta de pruebas facilita la acumulación de código de baja calidad

## Solución Recomendada

```csharp
// Implementación de pruebas unitarias para el servicio de cálculo de pagos
public class PaymentCalculationServiceTests
{
    private Mock<IExchangeRateService> _exchangeRateServiceMock;
    private Mock<ITaxCalculator> _taxCalculatorMock;
    private Mock<IDiscountService> _discountServiceMock;
    private PaymentCalculationService _service;
    
    [SetUp]
    public void SetUp()
    {
        _exchangeRateServiceMock = new Mock<IExchangeRateService>();
        _taxCalculatorMock = new Mock<ITaxCalculator>();
        _discountServiceMock = new Mock<IDiscountService>();
        
        _service = new PaymentCalculationService(
            _exchangeRateServiceMock.Object,
            _taxCalculatorMock.Object,
            _discountServiceMock.Object);
    }
    
    [Test]
    public async Task CalculatePaymentAmount_WithNullInvoice_ThrowsArgumentNullException()
    {
        // Arrange
        Invoice invoice = null;
        
        // Act & Assert
        var exception = Assert.ThrowsAsync<ArgumentNullException>(async () => 
            await _service.CalculatePaymentAmount(invoice, "USD"));
        
        Assert.That(exception.ParamName, Is.EqualTo("invoice"));
    }
    
    [Test]
    public async Task CalculatePaymentAmount_WithEmptyTargetCurrency_ThrowsArgumentException()
    {
        // Arrange
        var invoice = new Invoice { Items = new List<InvoiceItem>() };
        
        // Act & Assert
        var exception = Assert.ThrowsAsync<ArgumentException>(async () => 
            await _service.CalculatePaymentAmount(invoice, ""));
        
        Assert.That(exception.ParamName, Is.EqualTo("targetCurrency"));
    }
    
    [Test]
    public async Task CalculatePaymentAmount_WithPreferredCustomerAndLargeSubtotal_AppliesDiscount()
    {
        // Arrange
        var invoice = new Invoice 
        {
            Customer = new Customer { IsPreferred = true },
            Currency = "USD",
            BillingAddress = new Address { Country = "US" },
            Items = new List<InvoiceItem> 
            {
                new InvoiceItem { Quantity = 1, UnitPrice = 1200 }
            }
        };
        
        // Configure mocks
        _discountServiceMock.Setup(x => x.CalculateDiscount(invoice.Customer, 1200))
            .ReturnsAsync(100);
        
        _taxCalculatorMock.Setup(x => x.CalculateTax(1100, "US"))
            .ReturnsAsync(110);
        
        // Act
        var result = await _service.CalculatePaymentAmount(invoice, "USD");
        
        // Assert
        Assert.That(result, Is.EqualTo(1210));
        _discountServiceMock.Verify(x => x.CalculateDiscount(invoice.Customer, 1200), Times.Once);
        _taxCalculatorMock.Verify(x => x.CalculateTax(1100, "US"), Times.Once);
    }
    
    [Test]
    public async Task CalculatePaymentAmount_WithDifferentTargetCurrency_ConvertsCorrectly()
    {
        // Arrange
        var invoice = new Invoice 
        {
            Customer = new Customer(),
            Currency = "USD",
            BillingAddress = new Address { Country = "US" },
            Items = new List<InvoiceItem> 
            {
                new InvoiceItem { Quantity = 2, UnitPrice = 50 }
            }
        };
        
        // Configure mocks
        _taxCalculatorMock.Setup(x => x.CalculateTax(100, "US"))
            .ReturnsAsync(10);
            
        _exchangeRateServiceMock.Setup(x => x.GetExchangeRate("USD", "EUR"))
            .ReturnsAsync(0.85m);
        
        // Act
        var result = await _service.CalculatePaymentAmount(invoice, "EUR");
        
        // Assert
        Assert.That(result, Is.EqualTo(93.5m)); // (100 + 10) * 0.85 = 93.5
        _exchangeRateServiceMock.Verify(x => x.GetExchangeRate("USD", "EUR"), Times.Once);
    }
}

// Refactorización del controlador para facilitar las pruebas
public class PaymentController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    private readonly IPaymentRequestValidator _validator;
    
    public PaymentController(
        IPaymentService paymentService,
        IPaymentRequestValidator validator)
    {
        _paymentService = paymentService;
        _validator = validator;
    }
    
    [HttpPost("process")]
    public async Task<IActionResult> ProcessPayment(PaymentRequest request)
    {
        // Usar un validador separado para facilitar las pruebas
        var validationResult = await _validator.ValidateAsync(request);
        if (!validationResult.IsValid)
            return BadRequest(validationResult.Errors);
        
        try
        {
            var result = await _paymentService.ProcessPaymentAsync(request);
            return Ok(result);
        }
        catch (PaymentException ex) when 
            (ex is InsufficientFundsException || 
             ex is InvalidCardException)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            return StatusCode(500, "An error occurred while processing your payment");
        }
    }
}

// Pruebas para el controlador
public class PaymentControllerTests
{
    private Mock<IPaymentService> _paymentServiceMock;
    private Mock<IPaymentRequestValidator> _validatorMock;
    private PaymentController _controller;
    
    [SetUp]
    public void SetUp()
    {
        _paymentServiceMock = new Mock<IPaymentService>();
        _validatorMock = new Mock<IPaymentRequestValidator>();
        
        _controller = new PaymentController(
            _paymentServiceMock.Object,
            _validatorMock.Object);
    }
    
    [Test]
    public async Task ProcessPayment_WithInvalidRequest_ReturnsBadRequest()
    {
        // Arrange
        var request = new PaymentRequest();
        var validationErrors = new[] { "Amount must be positive" };
        
        _validatorMock.Setup(x => x.ValidateAsync(request))
            .ReturnsAsync(new ValidationResult { IsValid = false, Errors = validationErrors });
        
        // Act
        var result = await _controller.ProcessPayment(request);
        
        // Assert
        Assert.IsInstanceOf<BadRequestObjectResult>(result);
        var badRequestResult = (BadRequestObjectResult)result;
        Assert.That(badRequestResult.Value, Is.EqualTo(validationErrors));
    }
    
    [Test]
    public async Task ProcessPayment_WithValidRequest_ReturnsOkResult()
    {
        // Arrange
        var request = new PaymentRequest { Amount = 100 };
        var paymentResult = new PaymentResult { TransactionId = Guid.NewGuid() };
        
        _validatorMock.Setup(x => x.ValidateAsync(request))
            .ReturnsAsync(new ValidationResult { IsValid = true });
            
        _paymentServiceMock.Setup(x => x.ProcessPaymentAsync(request))
            .ReturnsAsync(paymentResult);
        
        // Act
        var result = await _controller.ProcessPayment(request);
        
        // Assert
        Assert.IsInstanceOf<OkObjectResult>(result);
        var okResult = (OkObjectResult)result;
        Assert.That(okResult.Value, Is.EqualTo(paymentResult));
    }
    
    [Test]
    public async Task ProcessPayment_WhenInsufficientFunds_ReturnsBadRequest()
    {
        // Arrange
        var request = new PaymentRequest { Amount = 999999 };
        
        _validatorMock.Setup(x => x.ValidateAsync(request))
            .ReturnsAsync(new ValidationResult { IsValid = true });
            
        _paymentServiceMock.Setup(x => x.ProcessPaymentAsync(request))
            .ThrowsAsync(new InsufficientFundsException("Insufficient funds for this transaction"));
        
        // Act
        var result = await _controller.ProcessPayment(request);
        
        // Assert
        Assert.IsInstanceOf<BadRequestObjectResult>(result);
        var badRequestResult = (BadRequestObjectResult)result;
        Assert.That(badRequestResult.Value, Is.EqualTo("Insufficient funds for this transaction"));
    }
}

// Refactorización del generador de documentos para permitir pruebas
public class DocumentGenerator
{
    private readonly ILicenseValidator _licenseValidator;
    private readonly ITemplateProvider _templateProvider;
    private readonly IDateTimeProvider _dateTimeProvider;
    private readonly IPdfRenderer _pdfRenderer;
    
    public DocumentGenerator(
        ILicenseValidator licenseValidator,
        ITemplateProvider templateProvider,
        IDateTimeProvider dateTimeProvider,
        IPdfRenderer pdfRenderer)
    {
        _licenseValidator = licenseValidator;
        _templateProvider = templateProvider;
        _dateTimeProvider = dateTimeProvider;
        _pdfRenderer = pdfRenderer;
    }
    
    public byte[] GenerateInvoicePdf(Invoice invoice)
    {
        // Verificar licencia a través de una abstracción
        var license = _licenseValidator.ValidateLicense();
        if (!license.IsValid)
            throw new InvalidOperationException("PDF generation license is not valid");
        
        // Obtener plantilla a través de una abstracción
        var template = _templateProvider.GetTemplate("InvoiceTemplate");
        
        // Acceder a la fecha actual a través de una abstracción
        var generatedDate = _dateTimeProvider.Now;
        
        // Renderizar PDF usando la abstracción
        var pdfContent = _pdfRenderer.RenderPdf(template, new
        {
            Invoice = invoice,
            GeneratedDate = generatedDate
        });
        
        return pdfContent;
    }
}

// Pruebas para el generador de documentos
public class DocumentGeneratorTests
{
    private Mock<ILicenseValidator> _licenseValidatorMock;
    private Mock<ITemplateProvider> _templateProviderMock;
    private Mock<IDateTimeProvider> _dateTimeProviderMock;
    private Mock<IPdfRenderer> _pdfRendererMock;
    private DocumentGenerator _generator;
    
    [SetUp]
    public void SetUp()
    {
        _licenseValidatorMock = new Mock<ILicenseValidator>();
        _templateProviderMock = new Mock<ITemplateProvider>();
        _dateTimeProviderMock = new Mock<IDateTimeProvider>();
        _pdfRendererMock = new Mock<IPdfRenderer>();
        
        _generator = new DocumentGenerator(
            _licenseValidatorMock.Object,
            _templateProviderMock.Object,
            _dateTimeProviderMock.Object,
            _pdfRendererMock.Object);
    }
    
    [Test]
    public void GenerateInvoicePdf_WithInvalidLicense_ThrowsException()
    {
        // Arrange
        var invoice = new Invoice();
        
        _licenseValidatorMock.Setup(x => x.ValidateLicense())
            .Returns(new LicenseInfo { IsValid = false });
        
        // Act & Assert
        Assert.Throws<InvalidOperationException>(() => _generator.GenerateInvoicePdf(invoice));
    }
    
    [Test]
    public void GenerateInvoicePdf_WithValidLicense_GeneratesPdf()
    {
        // Arrange
        var invoice = new Invoice { Id = Guid.NewGuid() };
        var fixedDate = new DateTime(2023, 1, 1);
        var expectedTemplate = "<html>Template</html>";
        var expectedPdfBytes = new byte[] { 1, 2, 3 };
        
        _licenseValidatorMock.Setup(x => x.ValidateLicense())
            .Returns(new LicenseInfo { IsValid = true });
            
        _templateProviderMock.Setup(x => x.GetTemplate("InvoiceTemplate"))
            .Returns(expectedTemplate);
            
        _dateTimeProviderMock.Setup(x => x.Now)
            .Returns(fixedDate);
            
        _pdfRendererMock.Setup(x => x.RenderPdf(
                expectedTemplate, 
                It.Is<object>(o => 
                    JsonConvert.SerializeObject(o).Contains(invoice.Id.ToString()) &&
                    JsonConvert.SerializeObject(o).Contains(fixedDate.ToString()))))
            .Returns(expectedPdfBytes);
        
        // Act
        var result = _generator.GenerateInvoicePdf(invoice);
        
        // Assert
        Assert.That(result, Is.EqualTo(expectedPdfBytes));
        _pdfRendererMock.Verify(
            x => x.RenderPdf(expectedTemplate, It.IsAny<object>()),
            Times.Once);
    }
}

// Implementación de pruebas de integración
public class PaymentIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public PaymentIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Reemplazar servicios reales con mocks para pruebas
                services.RemoveAll<IPaymentGateway>();
                services.AddScoped(_ => 
                {
                    var mock = new Mock<IPaymentGateway>();
                    mock.Setup(x => x.ProcessPayment(
                            It.Is<PaymentInfo>(p => p.Amount <= 1000),
                            It.IsAny<CreditCardInfo>()))
                        .ReturnsAsync(new GatewayResponse { Success = true, TransactionId = "TX123" });
                        
                    mock.Setup(x => x.ProcessPayment(
                            It.Is<PaymentInfo>(p => p.Amount > 1000),
                            It.IsAny<CreditCardInfo>()))
                        .ReturnsAsync(new GatewayResponse { Success = false, ErrorCode = "INSUFFICIENT_FUNDS" });
                    
                    return mock.Object;
                });
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task ProcessPayment_WithValidSmallAmount_ReturnsSuccess()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 100,
            CreditCardNumber = "4111111111111111",
            ExpirationMonth = 12,
            ExpirationYear = DateTime.Now.Year + 1,
            Cvv = "123"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/payment/process", request);
        
        // Assert
        response.EnsureSuccessStatusCode();
        var responseContent = await response.Content.ReadFromJsonAsync<PaymentResult>();
        Assert.NotNull(responseContent);
        Assert.Equal("TX123", responseContent.TransactionId);
    }
    
    [Fact]
    public async Task ProcessPayment_WithLargeAmount_ReturnsBadRequest()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 5000,
            CreditCardNumber = "4111111111111111",
            ExpirationMonth = 12,
            ExpirationYear = DateTime.Now.Year + 1,
            Cvv = "123"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/payment/process", request);
        
        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        var content = await response.Content.ReadAsStringAsync();
        Assert.Contains("Insufficient funds", content);
    }
}

// Test-Driven Development (TDD) para una nueva funcionalidad
public class DiscountCalculatorTests
{
    // Primero la prueba, luego la implementación (TDD)
    [Test]
    public void CalculateBulkDiscount_WithSmallQuantity_ReturnsZero()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        
        // Act
        var discount = calculator.CalculateBulkDiscount(5, 100);
        
        // Assert
        Assert.That(discount, Is.EqualTo(0));
    }
    
    [Test]
    public void CalculateBulkDiscount_WithMediumQuantity_Returns10Percent()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        
        // Act
        var discount = calculator.CalculateBulkDiscount(15, 100);
        
        // Assert
        Assert.That(discount, Is.EqualTo(10));
    }
    
    [Test]
    public void CalculateBulkDiscount_WithLargeQuantity_Returns20Percent()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        
        // Act
        var discount = calculator.CalculateBulkDiscount(30, 100);
        
        // Assert
        Assert.That(discount, Is.EqualTo(20));
    }
}

// Implementación después de las pruebas
public class DiscountCalculator
{
    public decimal CalculateBulkDiscount(int quantity, decimal unitPrice)
    {
        if (quantity >= 30)
            return unitPrice * 0.2m;
            
        if (quantity >= 10)
            return unitPrice * 0.1m;
            
        return 0;
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar cobertura de pruebas para componentes críticos del sistema
- Facilitar la detección temprana de errores mediante pruebas unitarias
- Mejorar el diseño del código mediante la inyección de dependencias para facilitar pruebas
- Permitir simulación de comportamientos complejos o impredecibles (fechas, licencias, etc.)
- Crear documentación viva y ejecutable que describe cómo debe comportarse el sistema
- Habilitar la refactorización segura con confianza de que los cambios no rompen la funcionalidad existente
- Introducir desarrollo guiado por pruebas (TDD) para nuevas características
- Implementar pruebas de integración para verificar el funcionamiento de componentes combinados

## Conclusión

La implementación de pruebas automatizadas y prácticas de TDD es fundamental para construir sistemas empresariales robustos y mantenibles. Las pruebas unitarias, de integración y de aceptación proporcionan diferentes capas de seguridad que permiten detectar errores temprano, documentar el comportamiento esperado del sistema y facilitar la refactorización segura. La inversión en pruebas automatizadas paga dividendos significativos a lo largo del tiempo en forma de mayor calidad del software, menor costo de mantenimiento y una experiencia de desarrollo más fluida. Adoptar TDD también mejora el diseño del código al fomentar interfaces claras, bajo acoplamiento y alta cohesión.

# Problema #20: Falta de Health Checks e Infraestructura de Monitoria

## Descripción del Problema
En la solución PAT se identificó una falta crítica de mecanismos de monitoreo y health checks, lo que dificulta la detección temprana de problemas, el diagnóstico de incidentes, y la medición del rendimiento del sistema. Esta ausencia impide una operación proactiva y genera dependencia en reportes de usuarios para identificar fallos.

## Código Problemático
```csharp
// Startup sin configuración de health checks
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddDbContext<CompassDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
        // Sin health checks configurados
        // Sin registro de métricas
        // Sin exposición de endpoints de diagnóstico
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
            // Sin endpoints de health o métricas
        });
    }
}

// Servicio crítico sin monitoreo de estado
public class PaymentProcessingService
{
    private readonly SqlConnection _dbConnection;
    private readonly HttpClient _httpClient;

    public PaymentProcessingService(string connectionString)
    {
        _dbConnection = new SqlConnection(connectionString);
        _httpClient = new HttpClient();
        
        // Sin verificación de disponibilidad de dependencias
        // Sin eventos o métricas para monitoreo
    }

    public async Task<bool> ProcessPayment(Payment payment)
    {
        try
        {
            // Dependencia externa sin timeout, retry o circuit breaker
            var gatewayResponse = await _httpClient.PostAsJsonAsync(
                "https://payment-gateway.com/api/process", 
                payment);
            
            if (gatewayResponse.IsSuccessStatusCode)
            {
                await _dbConnection.OpenAsync();
                // Procesamiento de pago...
                return true;
            }
            
            return false;
        }
        catch (Exception ex)
        {
            // Excepción registrada pero sin información de diagnóstico
            // o alertas que permitan monitoreo proactivo
            _logger.LogError(ex, "Error processing payment");
            return false;
        }
    }
    
    // Sin métodos para verificar el estado del servicio
    // Sin métricas de rendimiento ni contador de errores
}

// Controlador sin endpoints de estado o información
public class PaymentController : ControllerBase
{
    private readonly PaymentProcessingService _paymentService;
    
    [HttpPost]
    public async Task<IActionResult> Process(PaymentRequest request)
    {
        var result = await _paymentService.ProcessPayment(request.ToPayment());
        
        // Sin métricas ni instrumentación
        // Sin registro de tiempos de respuesta
        
        return result ? Ok() : BadRequest();
    }
    
    // Sin endpoints para verificar el estado del controlador
}

// Program.cs sin configuración de monitoreo
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
                // Sin configuración para Application Insights
                // Sin integración con servicios de monitoreo
            });
}
```

**Archivos:** `~\Compass\Compass.Api\Startup.cs, ~\Compass\Compass.Payments.Infrastructure\Services\PaymentProcessingService.cs, ~\Compass\Compass.Api\Controllers\PaymentController.cs, ~\Compass\Compass.Api\Program.cs`

## Consecuencias
1. **Detección tardía de problemas:** Los fallos solo se detectan cuando afectan a usuarios finales
2. **Tiempo de resolución prolongado:** Sin información de diagnóstico, la identificación de causas raíz es lenta y difícil
3. **Imposibilidad de monitoreo preventivo:** No hay forma de identificar tendencias de degradación antes de que sean críticas
4. **Falta de visibilidad operativa:** Los equipos de operaciones carecen de información sobre el estado del sistema
5. **Dificultad en la planificación de capacidad:** Sin métricas de rendimiento, es imposible prever necesidades de escalamiento
6. **Dependencias no monitorizadas:** Fallos en servicios externos pueden pasar desapercibidos hasta causar problemas mayores
7. **Recuperación manual:** La ausencia de auto-diagnóstico requiere intervención manual frecuente

## Solución Recomendada
```csharp
// Startup con configuración completa de health checks y monitoreo
public class Startup
{
    private readonly IConfiguration _configuration;
    
    public Startup(IConfiguration configuration)
    {
        _configuration = configuration;
    }
    
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        
        // Configuración de contexto de base de datos
        services.AddDbContext<CompassDbContext>(options =>
            options.UseSqlServer(_configuration.GetConnectionString("DefaultConnection")));
        
        // Configuración de Application Insights para telemetría
        services.AddApplicationInsightsTelemetry();
        
        // Configuración de métricas con Prometheus
        services.AddMetrics()
            .AddPrometheusExporter()
            .AddHealthChecks();
            
        // Configuración de health checks
        services.AddHealthChecks()
            // Verificación de base de datos
            .AddSqlServer(
                _configuration.GetConnectionString("DefaultConnection"),
                name: "sql-database",
                tags: new[] { "db", "data", "sql" })
            // Verificación de API externa de pagos
            .AddUrlGroup(
                new Uri(_configuration["PaymentGateway:BaseUrl"]),
                name: "payment-gateway-api",
                tags: new[] { "payment", "external" })
            // Verificación de Redis (caché)
            .AddRedis(
                _configuration.GetConnectionString("Redis"),
                name: "redis-cache",
                tags: new[] { "cache", "redis" })
            // Health check personalizado para servicio crítico
            .AddCheck<PaymentProcessingHealthCheck>(
                "payment-processing-service",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "payment", "core" })
            // Verificación de espacio en disco
            .AddDiskStorageHealthCheck(
                setup => setup.AddDrive("C:\\", 1024), // 1GB free minimum
                name: "disk-space-check",
                tags: new[] { "system" });
                
        // Configuración de Polly para resiliencia
        services.AddHttpClient("payment-gateway")
            .ConfigureHttpClient(client =>
            {
                client.BaseAddress = new Uri(_configuration["PaymentGateway:BaseUrl"]);
                client.Timeout = TimeSpan.FromSeconds(30);
            })
            .AddPolicyHandler(GetRetryPolicy())
            .AddPolicyHandler(GetCircuitBreakerPolicy())
            .AddHealthCheck();
            
        // Configuración del UI de Health Checks
        services.AddHealthChecksUI(setupSettings: setup =>
        {
            setup.SetEvaluationTimeInSeconds(60); // Evaluar cada 60 segundos
            setup.MaximumHistoryEntriesPerEndpoint(50); // Guardar 50 entradas por endpoint
            setup.SetApiMaxActiveRequests(1); // Limitar cantidad de requests paralelos
            
            // Agregar endpoints a monitorear
            setup.AddHealthCheckEndpoint("api", "/health");
            setup.AddHealthCheckEndpoint("payments-api", "/payments/health");
        })
        .AddInMemoryStorage(); // Almacenar resultados en memoria
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        // Middleware para capturar métricas de peticiones
        app.UseMetricServer();
        app.UseHttpMetrics();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
            
            // Endpoints de health check
            endpoints.MapHealthChecks("/health", new HealthCheckOptions
            {
                Predicate = _ => true, // Todos los health checks
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
            
            // Health check específico para disponibilidad (liveness)
            endpoints.MapHealthChecks("/health/live", new HealthCheckOptions
            {
                Predicate = check => check.Tags.Contains("live")
            });
            
            // Health check específico para componentes listos (readiness)
            endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions
            {
                Predicate = check => check.Tags.Contains("ready")
            });
            
            // Dashboard de UI para health checks
            endpoints.MapHealthChecksUI(options =>
            {
                options.UIPath = "/healthchecks-ui";
                options.ApiPath = "/health-ui-api";
            });
            
            // Endpoint para métricas de Prometheus
            endpoints.MapMetrics();
        });
    }
    
    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(msg => msg.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
    }
    
    private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
    }
}

// Implementación de Health Check personalizado
public class PaymentProcessingHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly IOptions<PaymentServiceOptions> _options;
    private readonly ILogger<PaymentProcessingHealthCheck> _logger;
    
    public PaymentProcessingHealthCheck(
        IHttpClientFactory httpClientFactory,
        IDbConnectionFactory connectionFactory,
        IOptions<PaymentServiceOptions> options,
        ILogger<PaymentProcessingHealthCheck> logger)
    {
        _httpClientFactory = httpClientFactory;
        _connectionFactory = connectionFactory;
        _options = options;
        _logger = logger;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        var data = new Dictionary<string, object>();
        var isHealthy = true;
        
        // Verificar conexión a base de datos
        try
        {
            using var connection = _connectionFactory.CreateConnection();
            await connection.OpenAsync(cancellationToken);
            
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = 5; // 5 segundos timeout
            
            await command.ExecuteScalarAsync(cancellationToken);
            
            data["Database"] = "Healthy";
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error en health check de base de datos");
            data["Database"] = "Unhealthy: " + ex.Message;
            isHealthy = false;
        }
        
        // Verificar servicio de pago externo
        try
        {
            var client = _httpClientFactory.CreateClient();
            client.Timeout = TimeSpan.FromSeconds(5);
            
            var response = await client.GetAsync(
                _options.Value.StatusEndpoint, 
                cancellationToken);
                
            response.EnsureSuccessStatusCode();
            
            data["PaymentGateway"] = "Healthy";
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error en health check de gateway de pagos");
            data["PaymentGateway"] = "Unhealthy: " + ex.Message;
            isHealthy = false;
        }
        
        // Verificar métricas de procesamiento
        var successRate = await GetSuccessRateMetric();
        data["SuccessRate"] = successRate;
        
        if (successRate < 95)
        {
            data["SuccessRateWarning"] = "Below threshold of 95%";
            isHealthy = false;
        }
        
        // Devolver resultado consolidado
        if (isHealthy)
            return HealthCheckResult.Healthy("Payment processing service is operational", data);
        else
            return HealthCheckResult.Degraded("Payment processing service has issues", null, data);
    }
    
    private async Task<double> GetSuccessRateMetric()
    {
        // Implementación real: obtener de sistema de métricas
        return 98.5; // Simulación para ejemplo
    }
}

// Servicio mejorado con health checks, métricas y resiliencia
public class PaymentProcessingService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly IMetricsRecorder _metrics;
    private readonly ILogger<PaymentProcessingService> _logger;
    
    public PaymentProcessingService(
        IHttpClientFactory httpClientFactory,
        IDbConnectionFactory connectionFactory,
        IMetricsRecorder metrics,
        ILogger<PaymentProcessingService> logger)
    {
        _httpClientFactory = httpClientFactory;
        _connectionFactory = connectionFactory;
        _metrics = metrics;
        _logger = logger;
    }

    public async Task<Result<PaymentResponse>> ProcessPayment(Payment payment, CancellationToken cancellationToken = default)
    {
        using (_metrics.MeasureDuration("payment_processing_duration"))
        using (_logger.BeginScope(new Dictionary<string, object> 
        {
            ["PaymentId"] = payment.Id,
            ["Amount"] = payment.Amount,
            ["MerchantId"] = payment.MerchantId
        }))
        {
            _metrics.IncrementCounter("payment_attempts_total", new[] { "method", payment.Method });
            _logger.LogInformation("Iniciando procesamiento de pago {PaymentId}", payment.Id);
            
            try
            {
                // Usar HttpClient con políticas de resiliencia configuradas
                var client = _httpClientFactory.CreateClient("payment-gateway");
                
                _metrics.IncrementCounter("payment_gateway_requests");
                var stopwatch = Stopwatch.StartNew();
                
                var gatewayResponse = await client.PostAsJsonAsync(
                    "api/process", 
                    payment,
                    cancellationToken);
                
                stopwatch.Stop();
                _metrics.RecordValue("payment_gateway_response_time", stopwatch.ElapsedMilliseconds);
                
                if (gatewayResponse.IsSuccessStatusCode)
                {
                    var response = await gatewayResponse.Content.ReadFromJsonAsync<GatewayResponse>(
                        cancellationToken: cancellationToken);
                    
                    // Registrar en base de datos con manejo adecuado de conexión
                    using var connection = await _connectionFactory.CreateOpenConnectionAsync(cancellationToken);
                    using var transaction = connection.BeginTransaction();
                    
                    try
                    {
                        // Registrar transacción en base de datos
                        _metrics.IncrementCounter("payment_success_total", new[] { "method", payment.Method });
                        _logger.LogInformation("Pago {PaymentId} procesado correctamente, ID de transacción: {TransactionId}",
                            payment.Id, response.TransactionId);
                        
                        transaction.Commit();
                        return Result<PaymentResponse>.Success(new PaymentResponse
                        {
                            TransactionId = response.TransactionId,
                            Status = "Completed",
                            ProcessedAt = DateTime.UtcNow
                        });
                    }
                    catch (Exception ex)
                    {
                        transaction.Rollback();
                        _metrics.IncrementCounter("database_error_total");
                        _logger.LogError(ex, "Error al registrar pago {PaymentId} en la base de datos", payment.Id);
                        return Result<PaymentResponse>.Failure(Error.Database("Error al registrar el pago"));
                    }
                }
                else
                {
                    var errorContent = await gatewayResponse.Content.ReadAsStringAsync(cancellationToken);
                    _metrics.IncrementCounter("payment_rejection_total", new[] { "status_code", ((int)gatewayResponse.StatusCode).ToString() });
                    _logger.LogWarning("Pago {PaymentId} rechazado por gateway con código {StatusCode}: {ErrorContent}",
                        payment.Id, (int)gatewayResponse.StatusCode, errorContent);
                    
                    return Result<PaymentResponse>.Failure(Error.PaymentRejected(errorContent));
                }
            }
            catch (Exception ex)
            {
                _metrics.IncrementCounter("payment_error_total", new[] { "error_type", ex.GetType().Name });
                _logger.LogError(ex, "Error procesando pago {PaymentId}: {ErrorMessage}", payment.Id, ex.Message);
                return Result<PaymentResponse>.Failure(Error.Unexpected(ex));
            }
        }
    }
    
    // Método específico para health check
    public async Task<HealthStatus> CheckHealthAsync(CancellationToken cancellationToken = default)
    {
        var status = new HealthStatus { IsHealthy = true };
        
        // Verificar conexión a base de datos
        try
        {
            using var connection = await _connectionFactory.CreateOpenConnectionAsync(cancellationToken);
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT COUNT(*) FROM Transactions";
            command.CommandTimeout = 5;
            var count = Convert.ToInt32(await command.ExecuteScalarAsync(cancellationToken));
            status.Components.Add("Database", new ComponentHealth { Status = "Healthy", Details = $"{count} transactions" });
        }
        catch (Exception ex)
        {
            status.IsHealthy = false;
            status.Components.Add("Database", new ComponentHealth { Status = "Unhealthy", Details = ex.Message });
        }
        
        // Verificar API externa
        try
        {
            var client = _httpClientFactory.CreateClient("payment-gateway");
            var response = await client.GetAsync("api/status", cancellationToken);
            response.EnsureSuccessStatusCode();
            
            status.Components.Add("PaymentGateway", new ComponentHealth { Status = "Healthy" });
        }
        catch (Exception ex)
        {
            status.IsHealthy = false;
            status.Components.Add("PaymentGateway", new ComponentHealth { Status = "Unhealthy", Details = ex.Message });
        }
        
        // Agregar métricas de sistema
        status.Metrics.Add("SuccessRate", await GetSuccessRate());
        status.Metrics.Add("AverageResponseTime", await GetAverageResponseTime());
        
        return status;
    }
    
    private async Task<double> GetSuccessRate()
    {
        // Implementación real obtendría esto de almacén de métricas
        return 99.2;
    }
    
    private async Task<double> GetAverageResponseTime()
    {
        // Implementación real obtendría esto de almacén de métricas
        return 235; // ms
    }
}

// Controlador con endpoints de health y mejores prácticas
public class PaymentController : ControllerBase
{
    private readonly PaymentProcessingService _paymentService;
    private readonly IMetricsRecorder _metrics;
    private readonly ILogger<PaymentController> _logger;
    
    public PaymentController(
        PaymentProcessingService paymentService,
        IMetricsRecorder metrics,
        ILogger<PaymentController> logger)
    {
        _paymentService = paymentService;
        _metrics = metrics;
        _logger = logger;
    }
    
    [HttpPost("process")]
    public async Task<IActionResult> Process(
        PaymentRequest request, 
        [FromHeader(Name = "X-Correlation-ID")] string correlationId = null,
        CancellationToken cancellationToken = default)
    {
        // Generar ID de correlación si no se proporciona
        correlationId ??= Guid.NewGuid().ToString();
        
        using (_logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        using (_metrics.MeasureDuration("http_request_duration", new[] { "endpoint", "process_payment" }))
        {
            _metrics.IncrementCounter("http_requests_total", new[] { "endpoint", "process_payment" });
            
            _logger.LogInformation("Solicitud de procesamiento de pago recibida: {Amount} {Currency}",
                request.Amount, request.Currency);
                
            var result = await _paymentService.ProcessPayment(
                request.ToPayment(correlationId), 
                cancellationToken);
                
            if (result.IsSuccess)
            {
                _metrics.IncrementCounter("http_success_total", new[] { "endpoint", "process_payment" });
                
                // Incluir Id de correlación en la respuesta
                Response.Headers.Add("X-Correlation-ID", correlationId);
                
                return Ok(result.Value);
            }
            else
            {
                _metrics.IncrementCounter("http_error_total", new[] { "endpoint", "process_payment", "error_code", result.Error.Code });
                
                _logger.LogWarning("Error procesando pago: {ErrorCode} {ErrorMessage}",
                    result.Error.Code, result.Error.Message);
                
                // Incluir Id de correlación en la respuesta de error
                Response.Headers.Add("X-Correlation-ID", correlationId);
                
                return result.Error.Code switch
                {
                    "PaymentRejected" => BadRequest(new { error = result.Error.Message, correlationId }),
                    "Database" => StatusCode(503, new { error = "Service temporarily unavailable", correlationId }),
                    _ => StatusCode(500, new { error = "An unexpected error occurred", correlationId })
                };
            }
        }
    }
    
    // Endpoint específico para verificación de estado
    [HttpGet("health")]
    public async Task<IActionResult> HealthCheck(CancellationToken cancellationToken)
    {
        var health = await _paymentService.CheckHealthAsync(cancellationToken);
        
        var status = health.IsHealthy ? StatusCodes.Status200OK : StatusCodes.Status503ServiceUnavailable;
        
        return StatusCode(status, health);
    }
    
    // Endpoint para métricas específicas del servicio de pagos
    [HttpGet("metrics")]
    public IActionResult GetMetrics()
    {
        var metrics = _metrics.GetCurrentMetrics();
        return Ok(metrics);
    }
}

// Program.cs con configuración para monitoreo
public class Program
{
    public static void Main(string[] args)
    {
        // Configurar registro de eventos temprano en el inicio
        Log.Logger = new LoggerConfiguration()
            .Enrich.FromLogContext()
            .Enrich.WithMachineName()
            .WriteTo.Console()
            .CreateBootstrapLogger();
            
        try
        {
            Log.Information("Iniciando aplicación");
            
            var host = CreateHostBuilder(args).Build();
            
            // Verificar migraciones de base de datos al inicio
            using (var scope = host.Services.CreateScope())
            {
                var services = scope.ServiceProvider;
                try
                {
                    var context = services.GetRequiredService<CompassDbContext>();
                    context.Database.Migrate();
                    Log.Information("Migraciones de base de datos aplicadas correctamente");
                }
                catch (Exception ex)
                {
                    Log.Error(ex, "Error al aplicar migraciones de base de datos");
                }
            }
            
            host.Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "La aplicación terminó inesperadamente");
            throw;
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog((context, services, configuration) => configuration
                .ReadFrom.Configuration(context.Configuration)
                .ReadFrom.Services(services)
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.ApplicationInsights(
                    services.GetRequiredService<TelemetryConfiguration>(),
                    TelemetryConverter.Traces))
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
                
                // Configurar timeout para solicitudes largas
                webBuilder.UseKestrel(options =>
                {
                    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
                    options.Limits.RequestHeadersTimeout = TimeSpan.FromMinutes(1);
                });
            })
            .ConfigureServices(services =>
            {
                // Agregar servicio para monitorear uso de recursos
                services.AddHostedService<ResourceMonitorBackgroundService>();
            });
}

// Servicio en segundo plano para monitorear recursos del sistema
public class ResourceMonitorBackgroundService : BackgroundService
{
    private readonly IMetricsRecorder _metrics;
    private readonly ILogger<ResourceMonitorBackgroundService> _logger;
    private readonly TimeSpan _period = TimeSpan.FromSeconds(15);
    
    public ResourceMonitorBackgroundService(
        IMetricsRecorder metrics,
        ILogger<ResourceMonitorBackgroundService> logger)
    {
        _metrics = metrics;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using PeriodicTimer timer = new PeriodicTimer(_period);
        
        while (!stoppingToken.IsCancellationRequested && await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                // Monitoreo de memoria
                var currentMemory = GC.GetTotalMemory(false);
                _metrics.RecordValue("system_memory_used_bytes", currentMemory);
                
                // Monitoreo de CPU (ejemplo simple)
                var cpuUsage = GetCurrentCpuUsage();
                _metrics.RecordValue("system_cpu_usage_percent", cpuUsage);
                
                // Monitoreo de threads activos
                var activeThreads = ThreadPool.ThreadCount;
                _metrics.RecordValue("system_active_threads", activeThreads);
                
                // Log solo si hay valores anómalos
                if (cpuUsage > 80 || currentMemory > 1024 * 1024 * 1024) // CPU > 80% o memoria > 1GB
                {
                    _logger.LogWarning(
                        "Recursos del sistema en niveles elevados: CPU: {CpuUsage}%, Memoria: {MemoryUsage}MB",
                        cpuUsage,
                        currentMemory / (1024 * 1024));
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error al monitorear recursos del sistema");
            }
        }
    }
    
    private float GetCurrentCpuUsage()
    {
        // Implementación real usaría librería como System.Diagnostics.Process
        // Este es un placeholder para el ejemplo
        return new Random().Next(5, 95);
    }
}
```

## Beneficios de la Solución

Esta implementación resuelve el problema al:

- Proporcionar visibilidad completa del estado del sistema mediante health checks graduales
- Facilitar el diagnóstico temprano de problemas a través de métricas y dashboards
- Establecer mecanismos de alertas automatizadas ante condiciones anómalas
- Permitir detección proactiva de degradación antes de que afecte a los usuarios
- Implementar patrones de resiliencia (retry, circuit breaker) para manejar fallas transitorias
- Monitorear el rendimiento para facilitar decisiones de escalamiento y optimización
- Generar información valiosa para análisis post-mortem de incidentes
- Proporcionar endpoints estándar para integración con sistemas de monitoreo externos

## Conclusión

Un sistema de monitoreo y health checks es una parte crítica de cualquier aplicación empresarial moderna. No solo permite la detección temprana de problemas, sino que también proporciona los datos necesarios para mantener y mejorar continuamente el rendimiento y la fiabilidad del sistema. La implementación de health checks graduales, métricas significativas, y mecanismos de resiliencia permite a los equipos adoptar un enfoque proactivo para la gestión de la infraestructura, reduciendo significativamente el tiempo medio de detección y resolución de incidentes. Además, estos sistemas proporcionan datos valiosos para decisiones de escalamiento, optimizaciones de rendimiento y mejoras arquitectónicas basadas en patrones reales de uso.