# Diagrama de Flujo ETL - Modelo y Real Transpuesto

## Proceso ETL Completo: Bosques Solares ISAGEN

Este documento describe el flujo detallado del proceso ETL implementado en el notebook `modelo_y_real_traspuesto_dia_por_ghi_poa_energia.ipynb`.

---

## 📋 Configuración Inicial

**PASO 1: CONFIGURACIÓN INICIAL**
```
📅 Parámetros de configuración:
├── Período: 2024-11-22 00:00:00 a 2025-10-16 23:55:00
├── Intervalo: 5 minutos (288 registros/día)
├── Plantas: BSB 500, BSB 501, BSB 502, BSB 503, BSB 504
└── Rutas de archivos configuradas
```

**PASO 2: CREAR GRILLA BASE**
```
🏗️ Cross-Join de fechas y plantas:
├── Total intervalos: ~210,000 (período completo)
├── Total plantas: 5
└── Registros grilla base: ~1,050,000
```

---

## 📊 Carga y Procesamiento de Datos

### Rama A: DATOS DEL MODELO

**PASO 3A: CARGA DATOS MODELO**
```
📊 Archivos CSV (desde fila 11):
├── Variables: date, GlobInc, GlobEff, E_Grid, PR
├── Transformación fecha: /90 → /2025
├── Filtrado de registros inválidos
└── Extracción nombre planta del archivo
```

**PASO 4A: AGREGACIÓN HORARIA MODELO**
```
📈 Agrupación por [Año, Mes, Día, Hora, Planta]:
├── E_Grid (Energía): SUMA (kW → kWh)
├── GlobInc (POA): PROMEDIO (W/m² → Wh/m²)
└── GlobEff (GHI): PROMEDIO (W/m² → Wh/m²)
```

**PASO 5A: AGREGACIÓN DIARIA MODELO**
```
📅 Agrupación por [Año, Mes, Día, Planta]:
├── Ener_kWh: SUMA (kWh → kWh/día)
├── POA_Wh/m2: SUMA (Wh/m² → Wh/m²/día)
└── GHI_Wh/m2: SUMA (Wh/m² → Wh/m²/día)
```

**PASO 6A: DUPLICACIÓN TEMPORAL**
```
📋 Extensión del modelo:
├── Datos originales: 2025
├── Duplicación para: 2024
└── Resultado: Modelo disponible para 2024 y 2025
```

### Rama B: DATOS REALES

**PASO 3B: CARGA DATOS EM (Estaciones Meteorológicas)**
```
🌡️ Archivos TXT (desde fila 5):
├── Plantas con EM: BSB 500, BSB 501, BSB 502
├── Sensores duales:
│   ├── POA: Irrad_POA_1 + Irrad_POA_2
│   ├── GHI: Irrad_GHI_1 + Irrad_GHI_2
│   ├── Temp. Módulo: Temp_Modulo_1 + Temp_Modulo_2
│   └── Temp. Ambiente: Temp_Amb_1 + Temp_Amb_2
└── Limpieza: "NAN" → null, valores negativos → 0
```

**PASO 3C: CARGA DATOS ENERGÍA**
```
⚡ Archivos Excel (5 minutal):
├── Variable: ENERGÍA ACTIVA (kWh)
├── Redondeo temporal a intervalos 5 min
├── Clave join: Planta|Fecha-Hora
└── Plantas: BSB 500, BSB 501, BSB 502, BSB 503, BSB 504
```

**PASO 4B: COMBINACIÓN DATOS REALES**
```
🔗 Joins secuenciales:
├── Grilla Base ← LEFT JOIN → Datos EM
├── Resultado anterior ← LEFT JOIN → Datos Energía
└── Clave temporal redondeada a intervalos 5 min
```

**PASO 5B: LIMPIEZA DATOS REALES**
```
🧹 Proceso de limpieza:
├── Valores negativos → 0
├── Valores NaN → 0 (para cálculos)
├── Filtrado registros inválidos
└── Validación rangos temporales
```

**PASO 6B: AGREGACIÓN HORARIA REALES**
```
📈 Agrupación por [año, mes, día, hora, Planta]:
├── Sensores duales: PROMEDIO((Sensor_1 + Sensor_2) / 2)
│   ├── POA_W/m2 = (POA_1 + POA_2) / 2
│   ├── GHI_W/m2 = (GHI_1 + GHI_2) / 2
│   ├── Tmod_C = (Tmod_1 + Tmod_2) / 2
│   └── Tamb_C = (Tamb_1 + Tamb_2) / 2
└── Energía_kWh: SUMA
```

**PASO 7B: AGREGACIÓN DIARIA REALES**
```
📅 Agrupación por [año, mes, día, Planta]:
├── Ener_kWh_Real: SUMA (kWh → kWh/día)
├── POA_Wh/m2_Real: SUMA (Wh/m² → Wh/m²/día)
├── GHI_Wh/m2_Real: SUMA (Wh/m² → Wh/m²/día)
├── Tmod_C: PROMEDIO (°C)
└── Tamb_C: PROMEDIO (°C)
```

---

## 🔗 Combinación y Transposición Final

**PASO 8: JOIN REALES + MODELO**
```
🔗 Combinación por [año, mes, día, Planta]:
├── Datos Reales (left) ← LEFT JOIN → Datos Modelo (right)
├── Resultado: Tabla con columnas Real y Modelo
└── Manejo de valores faltantes
```

**PASO 9: TRANSPOSICIÓN FINAL**
```
🔄 Estructura transpuesta:
├── Crear registros "Real":
│   ├── Tipo = "Real"
│   ├── Ener_kWh = Ener_kWh_Real
│   ├── GHI_Wh/m2 = GHI_Wh/m2_Real
│   └── POA_Wh/m2 = POA_Wh/m2_Real
├── Crear registros "Modelo":
│   ├── Tipo = "Modelo"
│   ├── Ener_kWh = Ener_kWh_Modelo
│   ├── GHI_Wh/m2 = GHI_Wh/m2_Modelo
│   └── POA_Wh/m2 = POA_Wh/m2_Modelo
└── Estructura final: [Fecha, año, mes, día, Planta, Tipo, Métricas]
```

**PASO 10: EXPORTACIÓN**
```
📤 Generación de salidas:
├── CSV principal para análisis
├── Excel con múltiples hojas:
│   ├── Datos_Transpuestos (completo)
│   ├── Hoja por planta (BSB_500, BSB_501, etc.)
│   ├── Datos_Real / Datos_Modelo
│   ├── Resumen_Estadistico
│   └── Datos_Validos (filtrados)
└── Estadísticas de exportación
```

---

## 📊 Reglas de Agregación y Conversiones

### Conversiones de Unidades

| **Fuente** | **Variable Original** | **Unidad Original** | **Agregación Horaria** | **Agregación Diaria** | **Unidad Final** |
|------------|----------------------|-------------------|------------------------|----------------------|------------------|
| **Modelo** | GlobInc (POA) | W/m² | PROMEDIO → Wh/m² | SUMA → Wh/m²/día | Wh/m²/día |
| **Modelo** | GlobEff (GHI) | W/m² | PROMEDIO → Wh/m² | SUMA → Wh/m²/día | Wh/m²/día |
| **Modelo** | E_Grid | kW | SUMA → kWh | SUMA → kWh/día | kWh/día |
| **EM** | POA_1/2 | W/m² | PROMEDIO DUAL → Wh/m² | SUMA → Wh/m²/día | Wh/m²/día |
| **EM** | GHI_1/2 | W/m² | PROMEDIO DUAL → Wh/m² | SUMA → Wh/m²/día | Wh/m²/día |
| **EM** | Tmod_1/2 | °C | PROMEDIO DUAL | PROMEDIO | °C |
| **EM** | Tamb_1/2 | °C | PROMEDIO DUAL | PROMEDIO | °C |
| **Energía** | ENERGÍA ACTIVA | kWh | SUMA | SUMA → kWh/día | kWh/día |

### Flujo de Agregaciones

```
📊 DATOS 5-MINUTALES (288 registros/día/planta)
         ↓ AGREGACIÓN HORARIA
📈 DATOS HORARIOS (24 registros/día/planta)
         ↓ AGREGACIÓN DIARIA  
📅 DATOS DIARIOS (1 registro/día/planta)
         ↓ TRANSPOSICIÓN
🔄 DATOS TRANSPUESTOS (2 registros/día/planta: Real + Modelo)
```

---

## 🎯 Resultado Final

### Estructura de la Tabla Final

| **Campo** | **Tipo** | **Descripción** |
|-----------|----------|-----------------|
| `Fecha` | DateTime | Fecha del día (YYYY-MM-DD) |
| `ano` | Integer | Año (2024, 2025) |
| `mes` | Integer | Mes (1-12) |
| `dia` | Integer | Día del mes (1-31) |
| `Planta` | String | Identificador planta (BSB 500-504) |
| `Tipo` | String | "Real" o "Modelo" |
| `Ener_kWh` | Float | Energía diaria (kWh/día) |
| `GHI_Wh/m2` | Float | Radiación global horizontal diaria (Wh/m²/día) |
| `POA_Wh/m2` | Float | Radiación plano del arreglo diaria (Wh/m²/día) |

### Métricas de Calidad

- **Plantas procesadas**: 5 (BSB 500-504)
- **Período de análisis**: ~330 días
- **Registros esperados por planta**: ~660 (330 días × 2 tipos)
- **Total registros finales**: ~3,300
- **Cobertura EM**: 3 plantas (BSB 500, 501, 502)
- **Cobertura Energía**: 5 plantas (completa)

---

## 🔍 Notas Técnicas

### Limitaciones Identificadas
1. **Datos EM**: Solo disponibles para 3 de 5 plantas
2. **Años de referencia**: Modelo 2025 duplicado para 2024
3. **Intervalos faltantes**: Manejados con valores 0 después de limpieza
4. **Sensores duales**: Promediados cuando ambos están disponibles

### Validaciones Implementadas
1. **Filtrado temporal**: Solo registros dentro del rango configurado
2. **Limpieza de valores**: Negativos convertidos a 0
3. **Validación de joins**: Claves temporales sincronizadas
4. **Control de calidad**: Estadísticas por paso del proceso

---

*Documento generado para el proyecto Bosques Solares - ISAGEN*  
*Última actualización: Noviembre 2025*