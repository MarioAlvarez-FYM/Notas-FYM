# Documentación de Cálculos de Liquidación y Negociación en eBill

Este documento detalla los lugares en la aplicación eBill donde se realizan cálculos de liquidación y negociación.

## Resumen General

La lógica de liquidación y negociación está distribuida en varias clases, principalmente dentro del proyecto `BusinessFacade`. Las clases clave involucradas son:

*   `LiquidacionNegociacionFacade`: Orquesta el proceso de liquidación y negociación.
*   `LiquidacionNegociacionGateWayFacade`: Actúa como una puerta de enlace para las operaciones de liquidación y negociación, validando y actualizando cupos.
*   `EBillLiquidacionFacade`: Maneja la lógica de liquidación específica para EBill, delegando los cálculos a un componente externo.
*   `AlianzaLiquidacionFacade`: Maneja la lógica de liquidación para el fondeador "Alianza", integrándose con su sistema.
*   `RegistroNegociacionManualFacade`: Procesa negociaciones manuales a partir de archivos CSV.

A continuación se detallan los cálculos realizados en cada una de estas clases.

## `BusinessFacade.LiquidacionNegociacionGateWayFacade`

Esta clase actúa como una fachada para la lógica de liquidación y negociación, proporcionando métodos para calcular, confirmar y validar liquidaciones.

### `ValidarCupoValidoLiquidaciones`

Este método valida si hay suficiente cupo de crédito para un conjunto de liquidaciones.

*   **Lógica:**
    1.  Agrupa las liquidaciones por pagador.
    2.  Para cada pagador, obtiene el convenio y el cupo disponible.
    3.  Calcula el `valorGiroTotal` sumando la `SumaGiro` de cada liquidación en el grupo.
    4.  Compara el `valorGiroTotal` con el cupo disponible.
    5.  Si el `valorGiroTotal` es mayor que el cupo, se genera un mensaje de error.

*   **Fragmento de Código:**
    ```csharp
    // sumar valor giro
    decimal valorGiroTotal = 0;
    foreach (var liquidacion in grupo)
    {
        valorGiroTotal += liquidacion.SumaGiro;
    }

    // validar control de cupo
    var cupo = convenio.CVE_responsabilidad_cupo == "S"
        ? convenio.CVE_financiero_cupo_disponible
        : detalleConvenio.DVE_financiero_cupo_disponible;

    var esCupoPagador = convenio.CVE_responsabilidad_cupo == "S" ? "el pagador" : "el proveedor";
    var nitEmpresa = convenio.CVE_responsabilidad_cupo == "S" ? grupo.Key : empresaEmisora.Emp_RFC;

    if (cupo < valorGiroTotal)
    {
        error += "El cupo disponible para " + esCupoPagador + " con NIT " + nitEmpresa + " es menor al valor de la liquidación. \n" +
            "Cupo disponible: $ " + cupo.ToString("N2", new System.Globalization.CultureInfo("en-US")) + " y" +
            " Valor aceptado total: $ " + valorGiroTotal.ToString("N2", new System.Globalization.CultureInfo("en-US")) + ". \n";
    }
    ```

### `ActualizarCupoPorNegociaciones`

Este método actualiza el cupo de crédito después de que se han procesado las negociaciones.

*   **Lógica:**
    1.  Agrupa las liquidaciones por pagador.
    2.  Para cada pagador, obtiene el convenio.
    3.  Calcula el `valorGiroTotal` sumando la `SumaGiro` de cada liquidación en el grupo.
    4.  Resta el `valorGiroTotal` del cupo disponible en el convenio.

*   **Fragmento de Código:**
    ```csharp
    // sumar valor giro
    decimal valorGiroTotal = 0;
    foreach (var liquidacion in grupo)
    {
        valorGiroTotal += liquidacion.SumaGiro;
    }

    // actualizar cupo 
    if (convenio.CVE_responsabilidad_cupo == "S")
    {
        convenio.CVE_financiero_cupo_disponible -= valorGiroTotal;
        conveniosDAO.GestionarCupoConvenio(convenio);
    } 
    else
    {
        detalleConvenio.DVE_financiero_cupo_disponible -= valorGiroTotal;
        detConvDatos.GestionarDetalleConvenioUnico(detalleConvenio);
    }
    ```

## `BusinessFacade.RegistroNegociacionManualFacade`

Esta clase maneja el registro de negociaciones manuales a partir de un archivo CSV.

### `CrearRegistroNegociacion`

Este método crea un registro de negociación (`FAC_FactoryConfirming`) a partir de una fila del archivo CSV. Realiza los siguientes cálculos y asignaciones:

*   `FAC_ValorNominaFacial`: Se redondea el valor total de la factura a 2 decimales.
    ```csharp
    negociacion.FAC_ValorNominaFacial = Math.Round(factura.CFD_total, 2);
    ```
*   `FAC_ValorNominalNeto`: Se redondea el saldo de la factura (o el total a pagar si el saldo es nulo) a 2 decimales.
    ```csharp
    negociacion.FAC_ValorNominalNeto = Math.Round((decimal)(factura.CFD_saldo.HasValue ? factura.CFD_saldo : factura.CFD_total_a_pagar), 2);
    ```
*   `FAC_ValorNomBaseLiquidacion`: Igual que `FAC_ValorNominalNeto`.
    ```csharp
    negociacion.FAC_ValorNomBaseLiquidacion = Math.Round((decimal)(factura.CFD_saldo.HasValue ? factura.CFD_saldo : factura.CFD_total_a_pagar), 2);
    ```
*   `FAC_ValorCompra`: Se redondea el `valorCompra` del CSV a 2 decimales.
    ```csharp
    negociacion.FAC_ValorCompra = Math.Round(valorCompra, 2);
    ```
*   `FAC_ValorCompraReferenciador`: Se redondea el `valorCompraReferenciador` del CSV a 2 decimales.
    ```csharp
    negociacion.FAC_ValorCompraReferenciador = Math.Round(valorCompraReferenciador, 2);
    ```
*   `FAC_ValorComisionReferenciador`: Se calcula como la diferencia entre `valorCompra` y `valorCompraReferenciador`, redondeado a 2 decimales.
    ```csharp
    negociacion.FAC_ValorComisionReferenciador = Math.Round(valorCompra - valorCompraReferenciador, 2);
    ```
*   `FAC_NuevoValorPago`: Se trunca el `nuevoValorPago` del CSV.
    ```csharp
    negociacion.FAC_NuevoValorPago = Math.Truncate(nuevoValorPago);
    ```

## `BusinessFacade.EBill.EBillLiquidacionFacade`

Esta fachada maneja la lógica de liquidación para EBill.

### `LiquidarFacturas`

Este método delega el cálculo de la liquidación a la clase `ServiciosNegociacion`.

*   **Lógica:**
    1.  Crea una instancia de `ServiciosNegociacion`.
    2.  Llama al método `CalcularLiquidacion()` de `serviciosNegociacion`.
    3.  El resultado de `CalcularLiquidacion()` se utiliza para clasificar la respuesta de la liquidación.

*   **Fragmento de Código:**
    ```csharp
    ServiciosNegociacion serviciosNegociacion = new ServiciosNegociacion(negociacionPagador.IdFacturaList, negociacion.IdFondeador, idPagador, IdEmpresaProveedor, parametrizacionCostumbre);
    RespuestaLiquidacionEBill respuestaLiquidacionEBill = await Task.Run(() => serviciosNegociacion.CalcularLiquidacion());
    respuestaLiquidacion = JObject.FromObject(respuestaLiquidacionEBill);
    return await Task.Run(() => negociacionFacade.ClasificarRespuestaLiquidacion(respuestaLiquidacion, JsonConvert.SerializeObject(negociacionPagador), sNitEmpresa));
    ```
*   **Nota:** El código fuente de la clase `ServiciosNegociacion` no está disponible en el proyecto, por lo que los detalles exactos del cálculo de la liquidación no se pueden documentar.

## `BusinessFacade.Alianza.AlianzaLiquidacionFacade`

Esta fachada maneja la lógica de liquidación para el fondeador "Alianza".

### `LiquidarFacturas`

Este método se integra con el sistema de Alianza para realizar la liquidación.

*   **Lógica:**
    1.  Crea un archivo (`JArray`) con la información de las facturas.
    2.  Crea un objeto de liquidación (`JObject`) con datos como el NIT del pagador, fechas y cupo disponible.
    3.  Envía una solicitud `POST` al endpoint de liquidación de Alianza con el archivo y el objeto de liquidación.
    4.  El cálculo real de la liquidación se realiza en el sistema de Alianza.

*   **Fragmento de Código:**
    ```csharp
    JArray archivo = CrearArchivo(negociacionPagador.IdFacturaList);
    // ...
    JObject liquidacion = CrearLiquidacion(negociacionPagador.NitPagador, sNitEmpresa);
    request.Add("liquidacion", liquidacion);
    var tareaLiquidacionFacturaAlianza = Task.Run(() => { return LiquidarFacturasAlianza(request); });
    ```

## `BusinessFacade.LiquidacionNegociacionFacade`

Esta clase contiene la lógica central para manejar las liquidaciones y negociaciones.

### `InsertarDetalleLiquidacion`

Este método inserta los detalles de una liquidación en la base de datos. Aunque no realiza cálculos complejos, sí procesa y guarda los resultados de los cálculos realizados por otras clases.

*   **Lógica:**
    1.  Itera sobre el `detalleLiquidacionEbillArray`.
    2.  Extrae valores como `tasaCalculada_EA`, `valorGiro`, `descuentoFinanciero`, etc.
    3.  Crea un objeto `FAC_DLQ_DetalleLiquidacionFondeador` con estos valores.
    4.  Guarda el objeto en la base de datos.

*   **Fragmento de Código:**
    ```csharp
    decimal tasaCalculada_EA = Utilities.ConvertirAValorDecimal(detalleNegociacionFactura.Value<string>("tasaCalculadaEA"));
    // ...
    decimal valorGiro = Utilities.ConvertirAValorDecimal(detalleNegociacionFactura.Value<string>("valorGiro"));
    decimal descuentoFinanciero = Utilities.ConvertirAValorDecimal(detalleNegociacionFactura.Value<string>("descuentoFinancieroCxC"));
    // ...
    FAC_DLQ_DetalleLiquidacionFondeador fAC_DLQ_DetalleLiquidacionFondeador = new FAC_DLQ_DetalleLiquidacionFondeador
    {
        LQF_Id_LiquidacionFondeador = idLiquidacion,
        DLQ_Numero_Factura = numeroFactura,
        DLQ_ValorAceptado = decimal.Parse(valorAceptado, NumberStyles.AllowExponent | NumberStyles.AllowDecimalPoint),
        DLQ_Dias = decimal.Parse(dias, NumberStyles.AllowExponent | NumberStyles.AllowDecimalPoint),
        DLQ_Valor_Giro = valorGiro,
        CFD_Id_Cfd = IdFactura,
        DLQ_FechaConsignacion = fechaConsignacion,
        DLQ_FechaCalculadaPago = fechaCalculadaPago,
        DLQ_TasaCalculada_EA = tasaCalculada_EA,
        DLQ_Descuento_FinancieroCXC = descuentoFinanciero,
        DLQ_CONF_PlazoTotal = confFondPlazoTotal,
        DLQ_CONF_FechaPago = confFondFechaPago,
        DLQ_CONF_ValorPago = confFondValorPago
    };
    ```
