# Lógica de servicios WAS antiguo (JAX-WS + EJB)

1 servicio(s) WAS antiguo (JAX-WS + EJB) detectado(s), con 1 operación(es) real(es) — 0 completamente resuelta(s) (DAO + stored procedure + constantes reales identificados).

Patrón real reconocido: un endpoint `@WebService(endpointInterface = ...)` cuyo método público delega a un método privado, que invoca un DAO `@Local`/`@Stateless` real (stored procedure vía `CallableStatement`, IN/OUT por posición), resolviendo código/mensaje de error contra una clase `Constantes` real — el equivalente, para este tipo de proyecto, de `Main`/`legacy-esql-procedure-logic` para IIB.

## Operaciones

### `GarantiasSARWS.consultarGarantia` ⚠️
- Archivo: `src\ec\com\pichincha\consultagarantiassar\service\impl\GarantiasSARWS.java`
- Request: `GarantiaCliente` → Response: `GarantiasSARResponse`
- ⚠️ El método público no delega a un método privado reconocible — tiene su propio cuerpo; revisar manualmente.
- DAO real: `GarantiasSARDao.consultarInformacionGarantiasSAR` (impl: `GarantiasSARImpl`)
- JNDI real: `Constantes.JNDI` = `jdbc/siglo`
- Stored procedure real: `?` (valor no resuelto)
- Parámetros reales: ninguno detectado
- Excepción(es) real(es): `GarantiaException`

## Constantes reales

- `Constantes.CODIGO_ERROR` = `0`
- `Constantes.MENSAJE_ERROR` = `OK`
- `Constantes.CODIGO_ERROR_EXCEPCION` = `99`
- `Constantes.MENSAJE_ERROR_EXCEPCION` = `Error al realizar la consulta`
- `Constantes.UNIDAD_PERSISTENCIA` = `GarantiasDB`
- `Constantes.PRC_CONSULTA_GARANTIAS` = `{call RIESGO.PKG_RGS_RIESGO_GLOBAL.PRC_CONSULTA_GARANTIAS(?,?,?,?,?,?,?,?,?)}`
- `Constantes.MENSAJE_NO_DATOS` = `No existe datos`
- `Constantes.JNDI` = `jdbc/siglo`
- `Constantes.CODIGO_TIPO_GARANTIA` = `CODIGO_TIPO_GARANTIA`
- `Constantes.TIPO_GARANTIA` = `TIPO_GARANTIA`
- `Constantes.CODIGO_SUBTIP_GARANTIA` = `CODIGO_SUBTIP_GARANTIA`
- `Constantes.SUBTIP_GARANTIA` = `SUBTIP_GARANTIA`
- `Constantes.NRO_GARANTIA` = `NRO_GARANTIA`
- `Constantes.DETALLE_GAR` = `DETALLE_GAR`
- `Constantes.MONTO_REALIZACION` = `MONTO_REALIZACION`
- `Constantes.CODIGO_COBERTURA` = `CODIGO_COBERTURA`
- `Constantes.COBERTURA` = `COBERTURA`
- `Constantes.NRO_GARANTIA_INTERVINIENTE` = `NRO_GARANTIA`
- `Constantes.IDENTIFICACION` = `IDENTIFICACION`
- `Constantes.NOMBRE` = `NOMBRE`
- `Constantes.CODIGO_TIPO_RELACION` = `CODIGO_TIPO_RELACION`
- `Constantes.TIPO_RELACION` = `TIPO_RELACION`
- `Constantes.FEC_AVALUO` = `FEC_AVALUO`
- `Constantes.FEC_VCTO_GARANTIA` = `FEC_VCTO_GARANTIA`
- `Constantes.VALOR_AVALUO` = `VALOR_AVALUO`
- `Constantes.FEC_VCTO_POLIZA` = `FEC_VCTO_POLIZA`

## Excepciones reales

- `GarantiaException` extiende `Exception` — ✅ 4 constructores estándar reales (`src\ec\com\pichincha\consultagarantiassar\exception\GarantiaException.java`)

## Recomendaciones
- No se pudo resolver con un modelo el delegate real de 1 operación(es) de `GarantiasSARWS` (consultarGarantia) — Bedrock no disponible (The security token included in the request is invalid.); revisar manualmente.
- No se pudo resolver con un modelo la lógica real de acceso a datos de "GarantiasSARDao.consultarInformacionGarantiasSAR" — Bedrock no disponible (The security token included in the request is invalid.); revisar manualmente.
- `GarantiasSARWS.consultarGarantia`: enlace con el DAO real `GarantiasSARDao.consultarInformacionGarantiasSAR` INFERIDO (por ser el único DAO real y la única operación real sin resolver en todo el proyecto) — confirmar manualmente que corresponde antes de migrar.
