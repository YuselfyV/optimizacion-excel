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
