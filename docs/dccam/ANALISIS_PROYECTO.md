# DCCAM_Performer_RPA — análisis del proyecto (baseline para la migración de selectores)

Fuente: `DCCAM_Performer_RPA-dev_3.zip`. Se analizó estáticamente (sin ejecutar).

## 1. Qué es
- Proceso **Performer** basado en REFramework (state machine) para **DCC – Data Compliance & Classification** de DHL (región AM).
- Toma items de una cola de Orchestrator (`OrchestratorQueueName`), abre la web DCC, hace login, entra a *Data Cleansing → Existing Shipment Records*, toma el siguiente embarque "To do", extrae los datos, aplica reglas de limpieza/validación (por país) y hace *Submit* en DCC.
- Studio 19.10.1, `UiPath.UIAutomation.Activities 20.10.6` (**actividades clásicas**), `targetFramework: Legacy`, VB. ~388 XAML.
- Navegador: `UiApp = msedge.exe`, `Browser = Edge`. Hoy DCC corre en **Edge con IE Mode** (por eso hay tantos `<wnd app='iexplore.exe'>`).

## 2. Flujo (Main.xaml)
Estados: **Init → Get Transaction Data → Process Transaction → End Process**.
- **Init**: `InitAllSettings` (Config.xlsx + assets) → `KillAllProcesses` → `InitAllApplications`.
- **Get Next Shipment**: `ShouldStopCloseDCC`, `NextItemValidation` (navegación en DCC, 85 invocaciones), `GetTransactionData`, `GetCountryConfiguration`.
- **Process**: 1er item → `Utils\OpenWebsite` o `OpenWebsiteModern` (usa `transactionItem.SpecificContent("Url")`) → `DCC\LoginDCC` (`ServiceArea`, `Facility` del item) → `DCC\NavToCleansing`. Luego según `countryConfig("IsReEngineeringVersion")`: `Process.xaml` (legacy, 225 invocaciones) o `NewVersionReEngineering\NVProcess.xaml`, que invoca **dinámicamente** `activeProcess("XAML")` (nombre del XAML sale de BD, no de estático).
- **Recuperación**: `RecoverLogOut`, `RecoverKill`, `RefreshQueue`, `DCC_UnlockUser`/`UnlockUsers`, `ClosePopUps`, `ValidateErrorRecord`, `TakeScreenshot`, `SendEmail`.
- **Cierre**: `CloseDCC`, `DCC_Logout`, `clearCache`.
- Item de cola (SpecificContent) usado: `Url`, `ServiceArea`, `Facility`, `ApplicationExceptionMail`. **La URL que decide el modo llega por `Url`.**

## 3. Carpetas
| Carpeta | Contenido |
|---|---|
| `Framework`, `Framework Utils` | REFramework + acciones de flujo (NextItemValidation, WaitNextRecord, Recover*, Action*). |
| `DCC` | Login/Logout/Close, NavToCleansing, lectura/escritura de consignee, line items, reglas por país (AR/CA/CL/CR/PA/PE/PY), validaciones. |
| `Utils` | Submit, UpdateField, GetInformation, ClosePopUps, OpenWebsite(Modern), PDF/OCR, monedas, logs. |
| `GetBusinessData` | GetShipmentData (57 selectores, tablas con `css-selector`/`parentname`), GetConsignee. |
| `NewVersionReEngineering` | Versión "reingeniería" (NV*): reglas configurables por BD, NVSubmitData, NVPostSummitValidation, NVDCCGetLineItems, etc. |
| `NADB_Rules`, `CMFRules`, `SSLRules`, `GCCP`, `SpecificRules`, `Regex`, `Mexico`, `Panama`, `PreCall` | Lógica de negocio/APIs; mayormente sin UI de DCC. |
| `TaxID/*` | Búsquedas en sitios externos (Serasa, MC2, Equifax, SUNAT, NOSIS…): **fuera de alcance** (no dependen de la URL de DCC). |
| `Data/Config.xlsx` | Settings/Constants/Assets (credenciales, BD, timeouts, `RetryOpenBrowser`, `WebBrowser`…). |
| `Test_Framework`, `_Test*.xaml`, `*_old`, `* - Copy` | Pruebas/código muerto. |

## 4. Selectores (806 en 107 XAML; inventario en `selector_inventory.csv`)
Alcance DCC (sin TaxID/sitios externos): ~460 selectores.

Contenedores `<html app=... title=...>` de DCC: `title='D*'` (mayoría), `'Data Compliance * Classification'`, `'Data Compliance & Classification'`, `'DCC - Session Time Out'`, `url='*dcc*'`.

Riesgos al pasar de IE Mode a modo estándar (Chromium), por frecuencia:
1. **Ventanas Win32 de IE** (`<wnd app='iexplore.exe' cls='#32770' title='Message from webpage'|'Internet Explorer'>`, `ctrlid=65535/1/20002`): ~60. Son los alert/confirm de JavaScript. En modo estándar no son ventanas de `iexplore.exe`. Presentes en `NavToCleansing`, `ClosePopUps`, `SubmitData`/`NVSubmitData`, `SubmitAction`, `NVPostSummitValidation`, etc. **Riesgo alto.**
2. **`aaname` en webctrl** (~108): el nombre accesible (MSAA en IE, cómputo distinto en Chromium). Varios con espacios finales (`'Your session has expired. Please login again. '`, `'*Data X*To do '`). Riesgo medio-alto.
3. **`tableRow/tableCol/idx/parentid/isleaf/css-selector`** (~29+26+29): dependen de la estructura DOM/MSAA. Riesgo medio (GetShipmentData, NavToCleansing, OpenWebsiteModern `tableRow='4'`).
4. **`title`/`url`** de la página: si el título/URL cambia entre hosts, se rompe el `<html>`. Riesgo medio.
5. `name=`/`id=` (`shipmentVO...`, `lineItemTable`, `userId`, `password`…): estables, **preferidos**.
6. `wnd app='jp2launcher.exe'` (Java blocked) y ventanas de Save As/Print (PDF/OCR): no dependen del modo.

## 5. Observaciones relevantes para los tiempos (~85 s)
- `Utils\OpenWebsiteModern`: `Delay 1 minute` dentro del `Do` y `RetryScope`; solo aplica al primer item.
- `Framework\OpenWebsite` tiene `title='allabolag.se*'` (código muerto, no alcanzable).
- Timeouts en Config: Short 5000, Medium 30000, Long 120000. Un selector que falla y entra en timeout suma 5–30 s por actividad: **cualquier selector "fallido con captura de excepción" degrada el tiempo** aunque el item termine en éxito.
- Ojo: hay alcanzabilidad dinámica (`activeProcess("XAML")`, `IsReEngineeringVersion` por país), por lo que solo la ejecución real confirma qué XAML se usan.

## 6. Estrategia de selectores (válidos para ambos modos)
- Usar `<html app='msedge.exe' title='D*' />` (o `url='*dcc*'`) — sin depender de `iexplore.exe`.
- Preferir `id`, `name`, `tag`, `type` sobre `aaname`, `idx`, `tableRow/Col`.
- Cuando haya que usar texto: `innertext` con comodines, sin espacios finales.
- Diálogos JS: definir selector por árbol de UI Explorer en cada modo y buscar un selector común (p. ej. sin `app`) o un patrón con comodín.

## 7. Registro de cambios
Se registra en `CAMBIOS_SELECTORES.md` (XAML | actividad | selector producción | selector nuevo | motivo).
