# optimizacion-excel
Pruebas para la optimizacion del excel 
// 4. Lanzar el procesamiento en segundo plano con virtual threads optimizados
Thread.startVirtualThread(() -> {
    try {
        log.info("Iniciando generación de Excel para la planilla {}", requestDto.getNoPlanilla());

        // ✅ 1. Precargar catálogos o tablas de homologación (para evitar consultas repetitivas)
        Map<String, String> estadosMap = homologacionPendigilService.getAllEstadosAsMap();
        Map<String, TipoDocumentoDTO> tiposDocsMap = tiposDocumentosService.getAllTiposDocumentosAsMap();

        // ✅ 2. Configuración de paginación
        int pageSize = 500;
        int offset = 0;

        while (true) {
            // ✅ 3. Consultar lote paginado desde Oracle
            List<DetallesPlanillaEntity> planillasBatch =
                    depositoPlanillaRepo.buscarDetallesPlanillaPorRango(requestDto, offset, pageSize);

            if (planillasBatch.isEmpty()) {
                log.info("No hay más registros para procesar. Finalizando...");
                break;
            }

            log.info("Procesando lote desde {} hasta {} ({} registros)", offset, offset + pageSize, planillasBatch.size());

            // ✅ 4. Procesar homologación y mapeo en paralelo dentro del lote
            List<DetallesPlanillaDto> detallesPlanillaDtos = planillasBatch
                    .parallelStream()
                    .map(DetallesPlanillaMapper.INSTANCE::merge)
                    .map(dto -> homologarCamposOptimizado(dto, estadosMap, tiposDocsMap))
                    .toList();

            // ✅ 5. Enviar en sublotes al servicio externo del Excel (para evitar saturar la red o memoria)
            List<List<DetallesPlanillaDto>> segmentos = partition(detallesPlanillaDtos, 100);

            segmentos.forEach(segmento ->
                    Thread.startVirtualThread(() -> {
                        try {
                            excelService.enviarTransaccion(segmento, requestDto);
                            log.info("Lote de {} registros enviado correctamente al servicio Excel.", segmento.size());
                        } catch (Exception e) {
                            log.error("Error enviando lote al servicio Excel: {}", e.getMessage(), e);
                        }
                    })
            );

            // Incrementar el offset para el siguiente bloque
            offset += pageSize;
        }

        log.info("✅ Generación de Excel completada para planilla {}", requestDto.getNoPlanilla());
    } catch (Exception e) {
        log.error("Error en el procesamiento en segundo plano de la planilla {}: {}", requestDto.getNoPlanilla(), e.getMessage(), e);
    }
});



private DetallesPlanillaDto homologarCamposOptimizado(
        DetallesPlanillaDto dto,
        Map<String, String> estadosMap,
        Map<String, TipoDocumentoDTO> tiposDocsMap
) {
    // Homologar estado si existe en el mapa
    if (dto.getEstado() != null && !dto.getEstado().isBlank()) {
        String estadoDesc = estadosMap.get(dto.getEstado().trim());
        if (estadoDesc != null) dto.setEstado(estadoDesc);
    }

    // Homologar tipo de documento
    if (dto.getTipoIdentificacion() != null && !dto.getTipoIdentificacion().isBlank()) {
        TipoDocumentoDTO tipoDoc = tiposDocsMap.get(dto.getTipoIdentificacion().trim());
        if (tipoDoc != null) dto.setTipoIdentificacion(tipoDoc.getAbreviatura());
    }

    return dto;
}


@Query(
  value = """
          SELECT * FROM DETALLES_PLANILLA 
          WHERE NUM_PLANILLA = :#{#dto.noPlanilla}
          OFFSET :offset ROWS FETCH NEXT :pageSize ROWS ONLY
          """,
  nativeQuery = true
)
List<DetallesPlanillaEntity> buscarDetallesPlanillaPorRango(
        @Param("dto") RequestDto dto,
        @Param("offset") int offset,
        @Param("pageSize") int pageSize
);

public List<DetallesPlanillaEntity> buscarDetallesPlanillaPorRango(RequestDto dto, int offset, int pageSize) {
    String sql = """
        SELECT * FROM DETALLES_PLANILLA
        WHERE NUM_PLANILLA = ?
        OFFSET ? ROWS FETCH NEXT ? ROWS ONLY
    """;




    

    return jdbcTemplate.query(sql,
            new Object[]{dto.getNoPlanilla(), offset, pageSize},
            new BeanPropertyRowMapper<>(DetallesPlanillaEntity.class));
}
@Repository
public class DetallesPlanillaImplRepository implements IDetallesPlanillaRepository {

    @PersistenceContext
    private EntityManager entityManager;

    private static final String QUERY_CONSULTA_DEPOSITO_PLANILLAS = """
        SELECT * 
        FROM RE.RE_DETALLES_PLANILLA RDP
        WHERE RDP.NUM_PLANILLA = :NUM_PLANILLA
          AND (:TIPO_IDENTIFICACION IS NULL OR RDP.TIPO_IDENTIFICACION = :TIPO_IDENTIFICACION)
          AND (:NUMERO_IDENTIFICACION IS NULL OR RDP.NUMERO_IDENTIFICACION = :NUMERO_IDENTIFICACION)
          AND (:ESTADO IS NULL OR RDP.ESTADO = :ESTADO)
          AND (:NUP IS NULL OR RDP.NUP = :NUP)
        ORDER BY %s
        OFFSET :offset ROWS FETCH NEXT :limit ROWS ONLY
    """;

    // 🔹 Cache simple para homologaciones (en memoria)
    private final Map<String, String> homologacionCache = new ConcurrentHashMap<>();

    @Override
    public List<DetallesPlanillaEntity> buscarDetallesPlanilla(RequestDto request) {

        // 🔹 Parámetros de paginación (valores quemados si no se reciben)
        int pageSize = 1000;
        int currentPage = 0;

        List<DetallesPlanillaEntity> allResults = new ArrayList<>();

        while (true) {
            String queryStr = String.format(QUERY_CONSULTA_DEPOSITO_PLANILLAS, getOrderBy(request));
            Query query = entityManager.createNativeQuery(queryStr, DetallesPlanillaEntity.class);

            query.setParameter("NUM_PLANILLA", request.getNumPlanilla());
            query.setParameter("TIPO_IDENTIFICACION", request.getTipoIdentificacion());
            query.setParameter("NUMERO_IDENTIFICACION", request.getNumeroIdentificacion());
            query.setParameter("ESTADO", request.getEstadoPlanilla());
            query.setParameter("NUP", request.getNoCuenta());
            query.setParameter("offset", currentPage * pageSize);
            query.setParameter("limit", pageSize);

            List<DetallesPlanillaEntity> results = query.getResultList();
            if (results.isEmpty()) break;

            // 🔹 Procesar en paralelo con Virtual Threads
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                List<CompletableFuture<DetallesPlanillaEntity>> futures = results.stream()
                        .map(entity -> CompletableFuture.supplyAsync(() -> mapWithHomologacion(entity), executor))
                        .toList();

                List<DetallesPlanillaEntity> processed = futures.stream()
                        .map(CompletableFuture::join)
                        .toList();

                allResults.addAll(processed);
            }

            // 🔹 Avanza de página
            if (results.size() < pageSize) break;
            currentPage++;
        }

        return allResults;
    }

    private String getOrderBy(RequestDto request) {
        return switch (request.getOrderDesc()) {
            case "SUMA_APORTES" -> "RDP.SUMA_APORTES";
            case "ESTADO" -> "RDP.ESTADO";
            case "PRIMER_APELLIDO" -> "RDP.PRIMER_APELLIDO";
            default -> "RDP.PRIMER_APELLIDO";
        };
    }

    /**
     * 🔹 Aplica la homologación con cache para no repetir consultas.
     */
    private DetallesPlanillaEntity mapWithHomologacion(DetallesPlanillaEntity entity) {
        String codigo = entity.getCodigoCampo();
        String descripcion = homologacionCache.computeIfAbsent(codigo, this::consultarDescripcion);
        entity.setDescripcionCampo(descripcion);
        return entity;
    }

    private String consultarDescripcion(String codigo) {
        // 🔹 Aquí haces la consulta real (o llamada a otro servicio)
        Query q = entityManager.createNativeQuery("SELECT DESCRIPCION FROM TABLA_HOMOLOGACION WHERE CODIGO = :codigo");
        q.setParameter("codigo", codigo);
        return (String) q.getSingleResult();
    }
}
