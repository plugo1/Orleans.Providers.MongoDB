# Análisis de Orleans.Providers.MongoDB y migración a .NET 10

## 1. Objetivo y alcance

Este documento describe la implementación actual de `Orleans.Providers.MongoDB`, su estructura, el registro de servicios, la inicialización de clustering, reminders y grain storage, y los puntos que deben verificarse al migrar desde .NET 8/Orleans 9 hacia .NET 10/Orleans 10.

El análisis está basado en:

- El código del proyecto `Orleans.Providers.MongoDB`.
- El submódulo `libs/orleans` actualmente checkoutado en `v10.3.1`.
- La comparación del código de Orleans entre `v9.0.1` y `v10.3.1`.
- Los cambios locales existentes en el workspace al momento de realizar el análisis.
- La documentación oficial de .NET 10, Orleans y MongoDB C# Driver.

### Estado del workspace

Al comenzar el análisis ya existían cambios locales relacionados con la migración:

- `net8.0` a `net10.0`.
- Orleans Reminders `9.0.1` a `10.3.1`.
- MongoDB.Driver `3.0.0` a `3.11.2`.
- SDK de CI .NET 8 a .NET 10.
- Migración del proyecto de tests a xUnit v3 y Microsoft.Testing.Platform.
- Cambio de referencias a proyectos de tests del submódulo de Orleans.

Esos cambios se consideran parte del trabajo de migración existente. Este documento no los revierte ni los atribuye a este análisis.

## 2. Resumen ejecutivo

El provider implementa tres capacidades principales de Orleans:

| Capacidad | Contrato Orleans | Implementación |
|---|---|---|
| Membership/clustering | `IMembershipTable` | `MongoMembershipTable` |
| Descubrimiento de gateways | `IGatewayListProvider` | `MongoGatewayListProvider` |
| Reminders | `IReminderTable` | `MongoReminderTable` |
| Persistencia de estado | `IGrainStorage` | `MongoGrainStorage` |

La arquitectura usa un único `IMongoClient` registrado en DI y factories que seleccionan ese cliente para cada provider. Cada provider crea sus colecciones de forma lazy. La primera operación que accede a una colección ejecuta su configuración, creación de índices y, opcionalmente, la configuración de shard key para Cosmos DB.

La migración a .NET 10 no implica, por sí sola, un cambio en el modelo de datos del provider. Los riesgos principales están en:

1. Cambios del contrato y comportamiento de Orleans 10.
2. Nuevas validaciones del servicio de reminders.
3. Cambios de compatibilidad del MongoDB C# Driver 3.x.
4. Cambio del target framework del paquete, que deja de ser consumible directamente desde aplicaciones `net8.0`.
5. Tests que todavía deben ejecutarse de extremo a extremo.

## 3. Estructura del repositorio

```text
Orleans.Providers.MongoDB/
├── Orleans.Providers.MongoDB/
│   ├── Configuration/
│   │   ├── JsonGrainStateSerializerOptions.cs
│   │   ├── MongoDBGatewayListProviderOptions.cs
│   │   ├── MongoDBGrainStorageConfigurator.cs
│   │   ├── MongoDBGrainStorageOptions.cs
│   │   ├── MongoDBGrainStorageOptionsValidator.cs
│   │   ├── MongoDBMembershipStrategy.cs
│   │   ├── MongoDBMembershipTableOptions.cs
│   │   ├── MongoDBOptions.cs
│   │   ├── MongoDBOptionsValidator.cs
│   │   └── MongoDBRemindersOptions.cs
│   ├── Membership/
│   │   ├── MongoGatewayListProvider.cs
│   │   ├── MongoMembershipTable.cs
│   │   └── Store/
│   │       ├── EtagHelper.cs
│   │       ├── Factory.cs
│   │       ├── IMongoMembershipCollection.cs
│   │       ├── MembershipBase.cs
│   │       ├── MongoMembershipDocument.cs
│   │       ├── MongoSuspectTime.cs
│   │       ├── Multiple/
│   │       ├── MultipleDeprecated/
│   │       └── Single/
│   ├── Reminders/
│   │   ├── MongoReminderTable.cs
│   │   └── Store/
│   │       ├── MongoReminderCollection.cs
│   │       └── MongoReminderDocument.cs
│   ├── StorageProviders/
│   │   ├── JsonBsonConverter.cs
│   │   ├── MongoGrainStorage.cs
│   │   ├── MongoGrainStorageCollection.cs
│   │   ├── MongoGrainStorageFactory.cs
│   │   └── Serializers/
│   │       ├── BinaryGrainStateSerializer.cs
│   │       ├── BsonGrainStateSerializer.cs
│   │       ├── IGrainStateSerializer.cs
│   │       └── JsonGrainStateSerializer.cs
│   ├── Utils/
│   │   ├── CollectionBase.cs
│   │   ├── DefaultMongoClientFactory.cs
│   │   ├── Guard.cs
│   │   ├── IMongoClientFactory.cs
│   │   ├── MongoExtensions.cs
│   │   └── MongoProviderErrorCodes.cs
│   ├── MongoDBClientExtensions.cs
│   ├── MongoDBSiloExtensions.cs
│   ├── ServiceCollectionExtensions.cs
│   └── Orleans.Providers.MongoDB.csproj
├── UnitTest/
├── Test/
├── Benchmarks/
└── libs/orleans/
```

### Dependencias principales

El proyecto provider referencia actualmente:

```xml
<TargetFrameworks>net10.0</TargetFrameworks>
<PackageReference Include="Microsoft.Orleans.Reminders" Version="10.3.1" />
<PackageReference Include="MongoDB.Driver" Version="3.11.2" />
```

La implementación también depende de APIs de Orleans Core, hosting, clustering, storage y serialization, transitivamente a través de `Microsoft.Orleans.Reminders` y las APIs usadas por el código.

## 4. Arquitectura de registro

### 4.1 Registro del cliente MongoDB

El punto de entrada es `UseMongoDBClient`:

```csharp
builder.UseMongoDBClient("mongodb://localhost");
```

El flujo es:

```text
UseMongoDBClient
  -> ConfigureServices
    -> AddMongoDBClient
      -> TryAddSingleton<IMongoClient>
      -> TryAddSingleton<IMongoClientFactory>
```

`ServiceCollectionExtensions.AddMongoDBClient` crea el cliente a partir de una connection string o de un factory de `MongoClientSettings`.

```csharp
services.AddMongoDBClient(provider =>
{
    var settings = MongoClientSettings.FromConnectionString(connectionString);
    return settings;
});
```

El cliente es singleton, que es el modelo recomendado para el driver MongoDB. Se reutilizan conexiones y pools entre las distintas áreas del provider.

### 4.2 Nombres de cliente

Los providers llaman a `IMongoClientFactory.Create` con nombres como:

- `Membership`.
- `Reminder`.
- `Storage`.

La factory predeterminada devuelve el mismo `IMongoClient` para todos esos nombres. El nombre sirve como punto de extensión para una factory personalizada, no como separación automática de clientes.

### 4.3 Validación

Todos los providers usan `IConfigurationValidator`. `MongoDBOptions.Validate` exige `DatabaseName`. Storage además valida que `KeyGenerator` no sea null.

La configuración inválida debe detectarse durante el arranque del silo, no esperar a la primera llamada de negocio.

## 5. Configuración de la aplicación

### 5.1 Configuración completa de un silo

```csharp
var host = Host.CreateDefaultBuilder(args)
    .UseOrleans((context, silo) =>
    {
        silo
            .UseMongoDBClient("mongodb://localhost:27017")
            .UseMongoDBClustering(options =>
            {
                options.DatabaseName = "Orleans";
                options.CollectionPrefix = "Prod_";
                options.Strategy = MongoDBMembershipStrategy.Multiple;
            })
            .UseMongoDBReminders(options =>
            {
                options.DatabaseName = "Orleans";
                options.CollectionPrefix = "Prod_";
            })
            .AddMongoDBGrainStorageAsDefault(options =>
            {
                options.DatabaseName = "Orleans";
                options.CollectionPrefix = "Grains_";
            });
    })
    .Build();
```

Para `Multiple`, MongoDB debe soportar transacciones, normalmente mediante un replica set o un cluster sharded correctamente configurado.

### 5.2 Configuración desde `IConfiguration`

Las extensiones aceptan `IConfiguration` directamente:

```csharp
builder.UseMongoDBReminders(configuration.GetSection("Orleans:Reminders"));
builder.UseMongoDBMembershipTable(configuration.GetSection("Orleans:Membership"));
```

La connection string se registra separadamente mediante `UseMongoDBClient`.

### 5.3 Settings avanzados del cliente

Cuando la connection string no alcanza para expresar la configuración, se puede crear `MongoClientSettings`:

```csharp
builder.UseMongoDBClient(serviceProvider =>
{
    var configuration = serviceProvider
        .GetRequiredService<IOptions<MyMongoConfiguration>>()
        .Value;

    var settings = MongoClientSettings.FromConnectionString(configuration.ConnectionString);
    settings.Credential = MongoCredential.CreateCredential(
        configuration.AuthenticationDatabase,
        configuration.UserName,
        configuration.Password);

    return settings;
});
```

### 5.4 Opciones compartidas

`MongoDBOptions` contiene:

| Opción | Función |
|---|---|
| `DatabaseName` | Base MongoDB utilizada por el provider. Obligatoria. |
| `ClientName` | Nombre disponible para factories personalizadas. |
| `CollectionPrefix` | Prefijo aplicado a colecciones. |
| `CreateShardKeyForCosmos` | Intenta crear shard key `_id` en la base `admin`. |
| `CollectionConfigurator` | Personaliza `MongoCollectionSettings`. |

El provider no crea bases de datos como una operación separada. MongoDB las materializa al crear colecciones o escribir documentos.

## 6. Inicialización del silo

Cada integración participa en una fase distinta del lifecycle de Orleans.

### 6.1 Membership

`MongoMembershipTable.InitializeMembershipTable` crea la implementación concreta de collection según `Strategy`.

```text
MongoMembershipTable
  -> Factory.CreateCollection
    -> SingleMembershipCollection
    -> MultipleMembershipCollection
    -> MultipleDeprecatedMembershipCollection
```

El provider almacena `ClusterId` como `deploymentId` lógico de las operaciones de membership.

### 6.2 Gateway list provider

`MongoGatewayListProvider.InitializeGatewayListProvider` crea una collection usando la misma estrategia configurada para membership. Luego `GetGateways` busca silos activos con `ProxyPort > 0` y convierte sus direcciones en `Uri` de gateway.

### 6.3 Storage

El storage se registra mediante `AddGrainStorage`. `MongoGrainStorage` se instancia con las opciones ya postconfiguradas. Las collections concretas se crean bajo demanda por combinación de nombre de estado y tipo de grain.

### 6.4 Inicialización lazy de collections

La clase base `CollectionBase<TEntity>` mantiene una referencia lazy a `IMongoCollection<TEntity>`.

La primera lectura o escritura que accede a `Collection` ejecuta:

1. `GetDatabase(DatabaseName)`.
2. Cálculo del nombre de colección.
3. Aplicación de `CollectionConfigurator`.
4. Obtención de `IMongoCollection<TEntity>`.
5. Intento opcional de configurar shard key.
6. Creación y limpieza de índices mediante `SetupCollection`.

La inicialización está protegida por un lock, pero las operaciones de MongoDB dentro de `SetupCollection` son síncronas.

## 7. Reminders en detalle

### 7.1 Componentes

Orleans provee el servicio de runtime `LocalReminderService`. Este servicio usa la implementación registrada de `IReminderTable`, que en este proyecto es `MongoReminderTable`.

```text
Grain
  -> IReminderRegistry
    -> LocalReminderService
      -> IReminderTable
        -> MongoReminderTable
          -> MongoReminderCollection
            -> MongoDB
```

`UseMongoDBReminders` realiza dos registros relevantes:

```csharp
services.AddReminders();
services.AddSingleton<IReminderTable, MongoReminderTable>();
```

`AddReminders` de Orleans registra `LocalReminderService`, `IReminderRegistry`, validación de opciones, instrumentos y participación en el lifecycle del silo.

### 7.2 Orden de arranque

En Orleans `10.3.1`, `LocalReminderService` participa en `ServiceLifecycleStage.RuntimeGrainServices`. En esa etapa:

1. Se ejecuta `MongoReminderTable.StartAsync`.
2. Se crea el objeto `MongoReminderCollection`.
3. Se confirma el acceso estructural a la tabla según el contrato del servicio.
4. El servicio local comienza a cargar y reconciliar reminders en background.
5. Las operaciones de tabla se ejecutan durante esa carga o ante llamadas del grain.

La implementación actual de `MongoReminderTable.StartAsync` solo asigna `collection` y retorna `Task.CompletedTask`.

### 7.3 Importante: la inicialización real es diferida

La conexión efectiva, la creación de la colección, la creación de índices y el intento de shard key ocurren en el primer acceso a `CollectionBase.Collection`, no dentro de `StartAsync`.

Consecuencias:

- El silo puede superar la etapa de `StartAsync` aunque MongoDB no esté disponible.
- Un fallo de permisos o conectividad puede aparecer durante la carga posterior.
- La creación de índices puede competir con la primera operación.
- El `CancellationToken` recibido por `MongoReminderTable.StartAsync` no se utiliza.

Esto debe considerarse al definir readiness probes y políticas de retry.

### 7.4 Opciones del provider

```csharp
builder.UseMongoDBReminders(options =>
{
    options.DatabaseName = "Orleans";
    options.CollectionPrefix = "Prod_";
    options.CreateShardKeyForCosmos = false;
    options.PurgeLegacyIndexes = false;
});
```

`PurgeLegacyIndexes` está marcado como obsolete y debe tratarse como una opción transitoria de migración.

### 7.5 Colección y documento

La colección de reminders es:

```text
{CollectionPrefix}OrleansReminderV2
```

El documento contiene:

| Campo | Uso |
|---|---|
| `_id` / `Id` | `{ServiceId}_{GrainId}_{ReminderName}` |
| `ServiceId` | Aislamiento entre deployments Orleans. |
| `GrainId` | Identidad serializada del grain. |
| `ReminderName` | Nombre lógico del reminder. |
| `Etag` | Concurrencia optimista. |
| `Period` | Periodicidad. |
| `GrainHash` | Hash uniforme para distribución/rango. |
| `StartAt` | Primera ejecución, normalizada a UTC cuando es unspecified. |
| `IsDeleted` | Campo legacy nullable. |

El índice principal es:

```text
ServiceId ascending, GrainHash ascending
```

### 7.6 Operaciones

#### `ReadRows(GrainId)`

Busca por `ServiceId`, `GrainHash` y `GrainId`. Devuelve todos los reminders del grain.

#### `ReadRow(GrainId, reminderName)`

Busca por el `_id` compuesto.

#### `ReadRows(begin, end)`

Implementa el rango Orleans `(begin, end]`. Si el rango cruza el final del anillo, usa una condición OR:

```text
GrainHash <= end OR GrainHash > begin
```

#### `UpsertRow`

Genera un nuevo ETag y actualiza `Etag`, `Period` y `StartAt`. Usa un upsert condicionado por la identidad completa del documento.

#### `RemoveRow`

Elimina por `_id` y ETag. Si el ETag no coincide, devuelve `false`.

### 7.7 ETags y concurrencia

Orleans usa ETags para que una eliminación no borre accidentalmente una versión reemplazada por otra operación. El provider respeta ese contrato en `RemoveRow`.

Punto a corregir/verificar: en `UpsertRow`, si Mongo devuelve duplicate key, se intenta un `ReplaceOneAsync` de fallback, pero después el código vuelve a lanzar la excepción original. Eso puede producir una operación persistida seguida de una respuesta de error.

La condición correcta debería verificarse con un test de concurrencia que confirme el resultado observable, no solo el estado de MongoDB.

### 7.8 Carga y reconciliación

El servicio de Orleans no ejecuta todos los reminders de todos los grains en cada silo. Cada silo carga los reminders correspondientes a su rango del anillo consistente y los reconcilia periódicamente.

En Orleans `10.3.1` las opciones relevantes son:

| Opción | Default aproximado | Regla |
|---|---:|---|
| `MinimumReminderPeriod` | 1 minuto | No puede ser negativa. |
| `RefreshReminderListPeriod` | 5 minutos | Debe ser positiva. |
| `ReminderLoadingWindow` | 10 minutos | Debe ser positiva y mayor o igual al refresh period. |
| `InitializationTimeout` | 5 minutos | Limita la inicialización del servicio. |

Una configuración anterior que tenga `ReminderLoadingWindow` menor que `RefreshReminderListPeriod` puede compilar y fallar al validar durante el arranque de Orleans 10.

## 8. Membership y clustering

### 8.1 Estrategias

```csharp
public enum MongoDBMembershipStrategy
{
    SingleDocument,
    Multiple,
    MultipleDeprecated
}
```

#### `SingleDocument`

- Un documento por deployment.
- Simple y adecuado para clusters pequeños.
- Menos eficiente para grandes cantidades de silos.

#### `Multiple`

- Un documento por silo.
- Un documento adicional para la versión de la tabla.
- Usa transacciones MongoDB.
- Recomendado frente a `MultipleDeprecated`.
- Requiere replica set o infraestructura equivalente.

#### `MultipleDeprecated`

- Un documento por silo.
- No utiliza el protocolo extendido de membership.
- Mantener solo por compatibilidad con instalaciones existentes.

### 8.2 Persistencia de identidad

Las estrategias múltiples forman identificadores a partir de deployment, IP, puerto y generación del silo. El código normaliza IPv4-mapped IPv6 mediante `MapToIPv4`.

La migración debe preservar:

- `ClusterId`.
- Base MongoDB.
- Prefijo de colecciones.
- Formato de `SiloAddress`.
- Reglas de generación del silo.

Orleans 10 agrega APIs de parseo modernas a `SiloAddress`, pero conserva `FromParsableString`, que usa el provider. No se observó un cambio de formato persistido en la comparación realizada.

## 9. Grain storage

### 9.1 Registro

```csharp
builder.AddMongoDBGrainStorageAsDefault(options =>
{
    options.DatabaseName = "Orleans";
    options.CollectionPrefix = "Grains_";
});
```

Para providers nombrados:

```csharp
builder.AddMongoDBGrainStorage("PubSubStore", options =>
{
    options.DatabaseName = "Orleans";
});
```

### 9.2 Resolución del serializer

El provider registra `JsonGrainStateSerializer` como default. La postconfiguración intenta resolver primero un serializer keyed con el nombre del provider y luego el serializer global.

Serializers disponibles:

| Serializer | Persistencia | Consideración |
|---|---|---|
| JSON | BSON convertido desde Newtonsoft.Json | Default y compatible con documentos legibles. |
| BSON | BSON nativo | Requiere revisar mapas y tipos. |
| Binary | Serializer binario de Orleans dentro de BSON | Más compacto, menos legible. |

Cambiar el serializer de una colección existente no es una migración transparente: los documentos antiguos deben seguir siendo legibles o migrarse explícitamente.

### 9.3 Concurrencia

Los documentos de grain storage usan:

```text
_id
_etag
_doc
```

`WriteAsync` condiciona el update por `_id` y `_etag`. En caso de conflicto, intenta consultar el ETag persistido y lanzar `InconsistentStateException`.

Punto a revisar: la rama fallback de reemplazo asigna un ETag incorrecto (`grainKey`) al documento mientras el objeto `grainState` recibe otro valor (`newETag`). Debe cubrirse con una prueba de conflicto/upsert concurrente.

## 10. Breaking changes y riesgos de migración

### 10.1 Cambios confirmados

| Área | Cambio | Impacto | Acción |
|---|---|---|---|
| Target framework | `net8.0` a `net10.0` | El nuevo paquete no es consumible directamente desde aplicaciones net8.0. | Decidir si se mantiene multi-targeting. |
| Orleans | `9.0.1` a `10.3.1` | Cambios en validación, nullable annotations y contratos internos. | Compilar y ejecutar tests contra Orleans 10. |
| Reminders | Validación de `ReminderLoadingWindow` y refresh period | Configuraciones inválidas fallan en startup. | Revisar configuración productiva. |
| Reminders | `Timeout.InfiniteTimeSpan` ya no es válido para due time/period | Código consumidor puede lanzar `ArgumentOutOfRangeException`. | Buscar usos en aplicaciones. |
| Driver | MongoDB.Driver 3.x descarta soporte de Server 4.2 o anterior desde 3.10 | Instalaciones antiguas no soportadas. | Confirmar MongoDB Server productivo. |
| Driver | LINQ2 eliminado en Driver 3.0 | Queries LINQ antiguas pueden fallar. | Este provider usa builders para las rutas principales, pero probar extensiones/custom code. |
| Driver | Cambios de GUID representation | Relevante para BSON directo y modelos con Guid. | Verificar documentos existentes con GUID. |
| Tests | xUnit v3 + Microsoft.Testing.Platform | Cambia ejecución, filtros y descubrimiento. | Validar CI y comandos locales. |

### 10.2 Cambios de API de Orleans revisados

Comparando Orleans `v9.0.1` con `v10.3.1`:

- `IReminderTable.StartAsync` continúa disponible.
- `IReminderTable.StopAsync` continúa disponible.
- `Init` continúa como API obsolete de compatibilidad.
- `ReadRow` y `UpsertRow` pasan a expresar nulabilidad (`ReminderEntry?` y `string?`) en Orleans 10.
- `SiloAddress` incorpora `IParsable<SiloAddress>` e `IUtf8SpanParsable<SiloAddress>`.
- `SiloAddress.FromParsableString` continúa disponible.
- Orleans agrega validaciones más estrictas en `ReminderOptions`.

El provider ya es compatible con las firmas anulables porque las implementaciones retornan valores que pueden ser null (`reminder?.ToEntry()` y el resultado de upsert).

### 10.3 Cambios de .NET 10 aplicables

Los breaking changes generales de .NET 10 que potencialmente aplican son:

- Cambios de APIs obsolete.
- Cambios de `Microsoft.Extensions.Configuration` sobre valores null.
- Cambios en keyed services con `AnyKey`.
- Cambios de SDK/MSBuild, restore y auditoría transitive de NuGet.
- Cambio por defecto de `dotnet new sln` a SLNX, aunque este repositorio conserva `.sln`.

No se identificó en el código del provider un uso directo que dependa de `BackgroundService`, `System.Linq.AsyncEnumerable`, WinForms, networking o APIs específicas afectadas por esos cambios.

### 10.4 Riesgos de comportamiento

#### Arranque aparentemente exitoso

`MongoReminderTable.StartAsync` no hace I/O y no usa el token. Las readiness probes deben comprobar una operación real contra MongoDB o esperar la validación de infraestructura correspondiente.

#### Despliegue rolling

La colección de reminders es `OrleansReminderV2`. La opción `PurgeLegacyIndexes` y la limpieza de índices deben coordinarse con silos viejos. El propio código advierte que remover índices incorrectamente durante una transición con versiones `<=9.3.0` puede producir collection scans.

#### Persistencia de estado

No cambiar simultáneamente base, prefijo, key generator y serializer sin un plan de migración. Cualquiera de esos cambios puede hacer que Orleans parezca no encontrar el estado existente.

#### MongoDB Cosmos

`CreateShardKeyForCosmos` ejecuta un comando administrativo y captura cualquier `MongoException` como “probablemente ya creado”. Esto puede ocultar también errores reales de permisos o configuración.

## 11. Compatibilidad de datos

### Reminders

Antes de actualizar:

1. Respaldar la colección `{prefix}OrleansReminderV2`.
2. Confirmar que el `ServiceId` del silo no cambia.
3. Confirmar que `GrainId.ToString()` produce el mismo formato esperado.
4. Confirmar que los campos `StartAt` se interpretan como UTC.
5. Revisar índices existentes y nombres legacy.

### Membership

No cambiar de estrategia durante la actualización sin revisar las colecciones y el protocolo de convivencia. `Multiple` usa una colección de versión y transacciones que no son equivalentes a `SingleDocument`.

### Grain storage

Verificar documentos generados por cada serializer que se use en producción. En especial:

- `DateTime` y `DateTimeOffset`.
- `Guid`.
- `decimal`.
- Tipos polimórficos.
- Propiedades con nombres que comienzan con `$`.
- Estados antiguos sin campo `_doc`.

## 12. Pruebas existentes

El proyecto tiene tests para:

```text
UnitTest/
├── Membership/
│   ├── MongoMembershipTableTests_Single.cs
│   ├── MongoMembershipTableTests_Multiple.cs
│   └── MongoMembershipTableTests_MultipleDeprecated.cs
├── Reminders/
│   └── MongoReminderTableTests.cs
├── Serializers/
│   └── SerializationTests.cs
└── Storage/
    └── StorageTests.cs
```

Los tests usan `MongoSandbox.Core` y crean:

- Un MongoDB standalone para reminders y algunas pruebas generales.
- Un MongoDB single-node replica set para membership `Multiple`.

La suite heredada desde Orleans incluye conformance tests para:

- Inicialización idempotente.
- Upserts paralelos.
- Identidades con caracteres especiales.
- Eliminación repetida.
- Lecturas por rango.
- Lecturas por grain.

Faltan o deben confirmarse explícitamente las siguientes pruebas de migración:

- Arranque con MongoDB no disponible.
- Cancelación de `StartAsync`.
- Fallback duplicate key de reminders.
- Fallback duplicate key de grain storage.
- Compatibilidad de documentos generados con Driver 3.0.
- Compatibilidad de GUID y decimal.
- Rolling deployment entre provider anterior y provider actualizado.
- Validación de `ReminderOptions` con valores inválidos.

## 13. Checklist de migración

### Antes de compilar

- [ ] Confirmar SDK .NET 10 instalado.
- [ ] Confirmar versión productiva de MongoDB Server.
- [ ] Confirmar versión productiva de Orleans.
- [ ] Confirmar si el provider debe continuar soportando `net8.0`.
- [ ] Inicializar submódulos recursivamente.
- [ ] Revisar `global.json` y `Directory.Build.*` si existen.

### Antes de desplegar

- [ ] Revisar `ReminderOptions`.
- [ ] Mantener `ServiceId` y `ClusterId`.
- [ ] Mantener database y collection prefixes.
- [ ] Respaldar reminders, membership y grain storage.
- [ ] Verificar replica set si se usa membership `Multiple`.
- [ ] Confirmar permisos para crear índices y shard keys.
- [ ] Ejecutar tests contra las versiones soportadas de MongoDB.
- [ ] Probar convivencia durante rolling deployment.

### Después de desplegar

- [ ] Confirmar que cada silo pasa readiness.
- [ ] Confirmar que reminders existentes se cargan.
- [ ] Registrar latencia y errores de `ReadRows(begin, end)`.
- [ ] Confirmar ausencia de collection scans inesperados.
- [ ] Verificar ETags y escrituras concurrentes.
- [ ] Verificar gateways descubiertos por el cliente.

## 14. Recomendaciones técnicas

### Prioridad alta

1. Ejecutar build y tests con el estado actual de la migración.
2. Agregar un test que compruebe el fallback duplicate key de `MongoReminderCollection.UpsertRow`.
3. Agregar un test que compruebe el ETag de la rama fallback de `MongoGrainStorageCollection.WriteAsync`.
4. Revisar si `MongoReminderTable.StartAsync` debe materializar la colección para fallar temprano.
5. Evaluar el uso real de `CancellationToken` en la inicialización.

### Prioridad media

1. Actualizar el README, que todavía contiene referencias a `TesterInternal` y ejemplos con APIs antiguas.
2. Sustituir logging interpolado por `LoggerMessage` o templates estructurados.
3. Hacer explícita la política de índices legacy y eliminar la opción obsolete cuando termine la transición.
4. Evaluar multi-targeting `net8.0;net10.0` si se necesita una transición gradual de consumidores.

### Prioridad baja

1. Revisar nombres de métodos públicos y documentación XML.
2. Agregar métricas específicas de latencia de MongoDB por provider.
3. Documentar los nombres exactos de colecciones y campos como parte del contrato operativo.

## 15. Conclusión

La implementación está razonablemente aislada detrás de contratos estándar de Orleans y conserva el formato principal de los datos. La migración a .NET 10 no requiere rediseñar MongoDB, pero sí exige una validación completa del arranque y de las rutas de concurrencia.

El mayor riesgo no es un cambio sintáctico del framework, sino asumir que una compilación exitosa significa que el provider está listo. Deben probarse especialmente:

- Inicialización real de collections.
- Conectividad y permisos durante el lifecycle.
- Validaciones nuevas de reminders.
- Concurrencia y ETags.
- Compatibilidad de documentos existentes.
- Rolling deployment.

El estado actual debe considerarse **migración parcialmente implementada hasta completar build, tests y pruebas de compatibilidad de datos**.

## 16. Verificación ejecutada

### Build del provider

Comando:

```bash
dotnet build Orleans.Providers.MongoDB/Orleans.Providers.MongoDB.csproj --configuration Release
```

Resultado:

- Exitoso.
- 0 warnings.
- 0 errors.

### Descubrimiento y ejecución de tests

El proyecto de tests usa Microsoft.Testing.Platform. Para que `dotnet test` use el runner correcto con el SDK .NET 10 fue necesario agregar un `global.json` en la raíz del repositorio:

```json
{
  "sdk": {
    "rollForward": "major",
    "version": "10.0.400"
  },
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Sin ese archivo, `dotnet test` intenta usar el target VSTest y el SDK .NET 10 lo rechaza para proyectos basados en Microsoft.Testing.Platform.

El descubrimiento directo del assembly confirmó 60 tests.

### Tests de reminders

Comando equivalente ejecutado directamente con el runner MTP:

```bash
dotnet UnitTest/bin/Release/net10.0/Orleans.Providers.MongoDB.UnitTest.dll \
  --filter-class Orleans.Providers.MongoDB.UnitTest.Reminders.MongoReminderTableTests
```

Resultado:

- 30 tests ejecutados.
- 30 exitosos.
- 0 fallos.
- Duración aproximada: 8 segundos.

Esto valida en Orleans 10 las operaciones de tabla, ETags, upserts concurrentes, rangos del anillo, identidad, eliminación, reinicio y conformance tests de reminders.

### Test de storage

Resultado del test focalizado:

```text
ThrowErrorWhenUpdatingCollectionWithAUniqueIndex: FAILED
Could not find an implementation for interface Orleans.Providers.MongoDB.UnitTest.Storage.TestGrains.IConstrainedGrain
```

El fallo sucede al resolver el grain, antes de la aserción MongoDB, en `UnitTest/Storage/StorageTests.cs:51`. Por lo tanto, no demuestra un fallo de la lógica de storage. Indica que la migración del proyecto de tests o su code generation no está registrando la implementación de `IConstrainedGrain`.

Durante la compilación de tests también aparecieron warnings de conflicto entre `Microsoft.Testing.Extensions.MSBuild` `2.0.2` y `2.3.3`. La referencia `2.0.2` llega transitivamente desde los proyectos de tests de Orleans y la `2.3.3` está referenciada directamente por este proyecto. Conviene alinear esas versiones antes de confiar plenamente en la suite completa.

### Estado de la verificación

| Verificación | Estado |
|---|---|
| Compilación del provider | Pass |
| Descubrimiento de tests | Pass, 60 tests |
| Tests de reminders | Pass, 30/30 |
| Test de storage | Bloqueado por codegen/registro de grain |
| Suite completa | Pendiente |
| Matriz MongoDB 6/7/8 | Pendiente |
| Compatibilidad con datos existentes | Pendiente |

## Referencias

- `Orleans.Providers.MongoDB/MongoDBSiloExtensions.cs`
- `Orleans.Providers.MongoDB/MongoDBClientExtensions.cs`
- `Orleans.Providers.MongoDB/ServiceCollectionExtensions.cs`
- `Orleans.Providers.MongoDB/Reminders/MongoReminderTable.cs`
- `Orleans.Providers.MongoDB/Reminders/Store/MongoReminderCollection.cs`
- `Orleans.Providers.MongoDB/Reminders/Store/MongoReminderDocument.cs`
- `Orleans.Providers.MongoDB/Utils/CollectionBase.cs`
- `Orleans.Providers.MongoDB/StorageProviders/MongoGrainStorage.cs`
- `Orleans.Providers.MongoDB/StorageProviders/MongoGrainStorageCollection.cs`
- `libs/orleans/src/Orleans.Reminders/ReminderService/LocalReminderService.cs`
- `libs/orleans/src/Orleans.Reminders/Options/ReminderOptions.cs`
- `libs/orleans/src/Orleans.Reminders/SystemTargetInterfaces/IReminderTable.cs`
- `libs/orleans/src/Orleans.Reminders/Hosting/SiloBuilderReminderExtensions.cs`
- [Breaking changes in .NET 10](https://learn.microsoft.com/en-us/dotnet/core/compatibility/10)
- [MongoDB C# Driver 3.x upgrade guide](https://www.mongodb.com/docs/drivers/csharp/current/reference/upgrade/v3/)
- [Orleans 10.0 release notes](https://github.com/dotnet/orleans/releases/tag/v10.0.0)
