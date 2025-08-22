## **Análisis de los resultados:**

### **El particionador funciona perfectamente:**

- **topK=1**: Retiene solo ~7.4% de probabilidad (ultra disperso)
- **topK=5**: Retiene ~27.4% de probabilidad
- **topK=10**: Retiene ~45.5% de probabilidad
- **topK=20**: Retiene ~70.9% de probabilidad
- **topK=30**: Retiene ~88.2% de probabilidad
- **topK=40**: Retiene ~97.2% de probabilidad
- **topK=50**: Retiene ~99.9% de probabilidad (casi sin poda)

### **Pero los tiempos son prácticamente constantes:**

- **Rango total**: 29.46s - 32.62s
- **Diferencia máxima**: Solo 3.16 segundos (10%)
- **Todas las medias están alrededor de ~32 segundos**

## **Conclusión clave:**

Con un PDFA de **250 estados nominales**, el algoritmo `PDFAQuantizationNAryTreeLearner` **NO es computacionalmente sensible** a la sparsity del particionado de probabilidades.

### **¿Por qué sucede esto?**

1. **El costo computacional dominante** está en operaciones como:
    - Construcción del árbol de observación
    - Comparaciones de hipótesis de modelos
    - Operaciones matriciales sobre el PDFA completo
    - I/O y gestión de estructuras de datos
2. **El particionado representa <0.05% del tiempo total** (como vimos antes)
3. **Para modelos de este tamaño**, las diferencias en sparsity no impactan significativamente la complejidad algorítmica