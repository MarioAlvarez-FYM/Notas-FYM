Con gusto te ayudo a revisar el cálculo de "Liquidación Confirming Estándar". Basado en mi análisis, la lógica principal se encuentra en el proyecto `BusinessFacade`.

### Día de la Semana de Pago

En cuanto a tu pregunta sobre el día de la semana, estás en lo correcto. **El día de la semana para el pago se toma del Convenio y no de una parametrización general.**

*   **Proyecto:** `BusinessFacade`
*   **Archivo:** `EBill/EBillLiquidacionFacade.cs`
*   **Función:** `LiquidarFacturas`

Dentro de este método, se obtiene la configuración de la costumbre de pago a través de la función `ObtenerParametrizacionCostumbrePagoXFondeador(IdFondeador)`. Esta configuración se encapsula en el objeto `ParametrizacionCostumbrePago`, que a su vez contiene la configuración para "ProntoPago", "Programada" y "CalculoDescuento".

La clase `BusinessObjects.SupportObjects.ConfiguracionCostumbrePago` contiene las propiedades `DiaSemanaNumero` y `DiaSemanaTexto`, que definen el día de la semana para el pago.

### Lógica para Sofasa

Respecto a si la lógica del día de la semana de pago se usa para **Sofasa**, es muy probable que exista una lógica específica para este cliente. Mi análisis muestra que hay reglas de negocio y validaciones particulares para Sofasa en el `eBillCore.xml`.

Sin embargo, la implementación exacta de esta lógica parece estar dentro de la clase `ServiciosNegociacion`, la cual es invocada desde `EBillLiquidacionFacade.cs`. Desafortunadamente, el código fuente de `ServiciosNegociacion` no se encuentra en el proyecto, por lo que no puedo darte los detalles precisos de cómo se maneja el día de pago para Sofasa.

### Fecha Calculada de Pago en Confirming

Sí, **la fecha calculada de pago es una parte fundamental del proceso de confirming**.

*   **Proyecto:** `BusinessFacade`
*   **Archivo:** `LiquidacionNegociacionFacade.cs`
*   **Función:** `InsertarDetalleLiquidacion`

En esta función, se utiliza la variable `fechaCalculadaPago`, que se obtiene de la respuesta del cálculo de liquidación. Esto confirma que la fecha calculada se utiliza y se persiste en la base de datos.

La clase `BusinessObjects.SupportObjects.ConfiguracionCostumbrePago` también incluye la propiedad `FechaCalculada`, lo que refuerza la importancia de esta fecha en el proceso.

Al igual que con la lógica de Sofasa, el cálculo exacto de la `fechaCalculadaPago` probablemente reside en la clase `ServiciosNegociacion`, a la que no tengo acceso.

### Resumen y Próximos Pasos

En resumen, la lógica que buscas se encuentra principalmente en el proyecto `BusinessFacade`. Te recomiendo revisar los siguientes archivos para un análisis más profundo:

*   `BusinessFacade/EBill/EBillLiquidacionFacade.cs`: Para entender cómo se inicia el proceso de liquidación y cómo se obtiene la configuración del convenio.
*   `BusinessObjects/SupportObjects/ConfiguracionCostumbrePago.cs`: Para ver la estructura de datos que contiene los parámetros de la costumbre de pago.
*   `BusinessFacade/LiquidacionNegociacionFacade.cs`: Para observar cómo se guardan los resultados de la liquidación.

Para entender los detalles del cálculo para **Confirming Estándar** y la lógica específica de **Sofasa**, sería necesario revisar el código de la clase `ServiciosNegociacion`, que parece estar en una dependencia externa.
