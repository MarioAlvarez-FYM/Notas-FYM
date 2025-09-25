# Análisis de Cálculos de Liquidación y Negociación en la Aplicación E-Bill

Este documento detalla los lugares en el código fuente de la aplicación E-Bill donde se realizan cálculos relacionados con la liquidación y negociación de facturas.

## Resumen General

La lógica de negocio para la liquidación y negociación se encuentra principalmente en el proyecto `BusinessFacade`. La aplicación utiliza un patrón de diseño de estrategia para manejar diferentes tipos de liquidaciones y negociaciones dependiendo del "fondeador" (por ejemplo, E-Bill, Alianza). La clase `ContextLiquidacionNegociacion` se utiliza para seleccionar y ejecutar la estrategia apropiada.

Los controladores en `WebApplication1` exponen estos servicios como puntos finales de API, mientras que el proyecto `SondaNegociacion` parece manejar el procesamiento asíncrono de estas operaciones.

## Archivos y Clases Relevantes

A continuación se detallan los archivos y clases más importantes involucrados en los cálculos:

### 1. `BusinessFacade/FactoringPaqueteLiquidacionFacade.cs`

Esta clase gestiona el procesamiento de paquetes de liquidación y negociación.

- **`CalcularLiquidacionParaFacturaAsync(FAC_PQT_PaqueteNegociacionFactoring paquete, int idFactura)`**:
    - **Propósito**: Punto de entrada para calcular la liquidación de una sola factura dentro de un paquete.
    - **Lógica de Cálculo**:
        1.  Crea un objeto `Negociacion`.
        2.  Utiliza `ContextLiquidacionNegociacion` para establecer una estrategia de liquidación (`ILiquidacion`).
        3.  La estrategia se determina dinámicamente en tiempo de ejecución basándose en la configuración del fondeador (`ObtenerParametrizacionLiquidacionXFondeador`).
        4.  Invoca el método `LiquidarFacturaAsync` de la estrategia seleccionada para realizar el cálculo real.

- **`ProcesarPaqueteDeLiquidacionAsync(FAC_PQT_PaqueteNegociacionFactoring paqueteAProcesar, int maxDetalles)`**:
    - **Propósito**: Orquesta el proceso de liquidación para un paquete completo de facturas.
    - **Lógica de Cálculo**:
        1.  Itera sobre los detalles (facturas) del paquete.
        2.  Para cada factura, llama a `CalcularLiquidacionParaFacturaAsync`.
        3.  Acumula los resultados y actualiza el estado general del paquete (`LIQUIDACION_EXITOSO`, `LIQUIDACION_FALLIDO`).
        4.  Calcula los totales del paquete, como `ValorFacialTotal`, `ValorAceptadoTotal` y `ValorGiroTotal`.

- **`RealizarLlamadasExternas(...)`**:
    - **Propósito**: Realiza las llamadas a servicios externos para el proceso de negociación.
    - **Lógica de Cálculo**:
        1.  Utiliza `ContextLiquidacionNegociacion` para establecer una estrategia de negociación (`INegociacion`).
        2.  Invoca el método `NegociarFacturaAsync` de la estrategia para ejecutar la negociación.

### 2. `BusinessFacade/LiquidacionNegociacionGateWayFacade.cs`

Actúa como una puerta de enlace entre los controladores de la API y la lógica de negocio principal.

- **`CalcularLiquidacion(Negociacion negociacion)`**:
    - **Propósito**: Método expuesto a la API para iniciar el cálculo de una liquidación.
    - **Lógica de Cálculo**:
        1.  Llama a `RealizarIntegracionLiquidacionNegociacion`, que utiliza el patrón de estrategia.
        2.  Selecciona la estrategia de liquidación (`ILiquidacion`) a través de `CrearInstanciaLiquidacion` y la ejecuta.

- **`ConfirmarDeclinarLiquidacion(JObject jObject)`**:
    - **Propósito**: Maneja la confirmación o declinación de una liquidación.
    - **Lógica de Cálculo**:
        1.  Similar a `CalcularLiquidacion`, utiliza el `ContextLiquidacionNegociacion` para ejecutar la estrategia de negociación (`INegociacion`) correspondiente.

### 3. `BusinessFacade/LiquidacionNegociacionFacade.cs`

Esta clase contiene la lógica central para manejar los datos de liquidación y negociación y persistirlos en la base de datos.

- **`InsertarLiquidacion(JToken bodyLiquidacion)`**:
    - **Propósito**: Inserta los datos de una liquidación en la base de datos después de un cálculo exitoso.
    - **Lógica de Cálculo**:
        1.  Llama a `InsertarCabeceraLiquidacion` y `InsertarDetalleLiquidacion`.

- **`InsertarCabeceraLiquidacion(DominioeBillDataContext dc, JToken liquidacionEbill)`**:
    - **Propósito**: Inserta el registro de cabecera `FAC_LQF_LiquidacionFondeador`.
    - **Lógica de Cálculo**:
        - Asigna valores directamente desde el objeto `liquidacionEbill` a las propiedades del objeto `FAC_LQF_LiquidacionFondeador`.
        - Realiza conversiones de tipos de datos y cálculos simples, como `Utilities.ConvertirAValorDecimal`.

- **`InsertarDetalleLiquidacion(DominioeBillDataContext dc, int idLiquidacion, JArray detalleLiquidacionEbillArray)`**:
    - **Propósito**: Inserta los registros de detalle `FAC_DLQ_DetalleLiquidacionFondeador`.
    - **Lógica de Cálculo**:
        - Itera sobre cada factura en `detalleLiquidacionEbillArray`.
        - Extrae valores como `valorAceptado`, `dias`, `valorGiro`, `descuentoFinancieroCxC`, etc.
        - Realiza conversiones de datos y asigna los valores a un nuevo objeto `FAC_DLQ_DetalleLiquidacionFondeador` antes de insertarlo.

- **`ConfirmarDeclinarNegociacion(DominioeBillDataContext dc)`**:
    - **Propósito**: Maneja la lógica de base de datos para confirmar o declinar una negociación.
    - **Lógica de Cálculo**:
        - Si la negociación es confirmada (`ConfirmarDeclinarLiquidacionFinal == 1`), llama a `factoryConfirmingTxDatos.InsertarFactoringConfirming(IdLiquidacionDelFondeador)`.

### 4. `BusinessFacade/RegistroNegociacionManualFacade.cs`

Esta clase se encarga del registro de negociaciones manuales a partir de un archivo CSV.

- **`CrearRegistroNegociacion(RegistroCsv registro)`**:
    - **Propósito**: Procesa una fila de un archivo CSV y crea un objeto `FAC_FactoryConfirming` para ser guardado.
    - **Lógica de Cálculo**:
        1.  Realiza un parsing y validación de los campos del CSV (`TASA_NEGOCIACION`, `TASA_DDP`, `FECHA_DE_COMPRA`, `VALOR_COMPRA`, etc.).
        2.  Calcula varios campos del objeto `FAC_FactoryConfirming`:
            - `FAC_Tasa_Negociacion`
            - `FAC_ValorCompra`
            - `FAC_ValorCompraReferenciador`
            - `FAC_ValorComisionReferenciador` (calculado como `valorCompra - valorCompraReferenciador`)
            - `FAC_DiasNegociados`
            - `FAC_NuevoValorPago`
            - `FAC_ValorNomBaseLiquidacion` (basado en el saldo de la factura)

### 5. Controladores de API (`WebApplication1/Controllers/`)

- **`LiquidacionController.cs`**:
    - **`CalcularLiquidacion(Negociacion negociacion)`**: Punto de entrada de la API que recibe la solicitud de liquidación y la pasa a `LiquidacionNegociacionGateWayFacade`.

- **`NegociacionController.cs`**:
    - **`GenerarArchivoNegociaciones(JObject data)`** y **`CargarNegociacionesManuales(JArray data)`**: Puntos de entrada para el proceso de negociación manual, que es manejado por `RegistroNegociacionManualFacade`.

## Conclusión

Los cálculos de liquidación y negociación están distribuidos en varias capas de la aplicación. El flujo principal para los cálculos automáticos se inicia en los controladores, pasa por el `GateWayFacade`, y utiliza un patrón de estrategia (`ContextLiquidacionNegociacion`) para ejecutar la lógica específica del fondeador. Los resultados se persisten en la base de datos a través de `LiquidacionNegociacionFacade`.

Por otro lado, los cálculos para negociaciones manuales se realizan en `RegistroNegociacionManualFacade` a partir de los datos de un archivo CSV.
