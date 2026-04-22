Este documento constituye la especificación técnica y guía de interpretación analítica para **STROMA**, un instrumento de inspección espectral y análisis de audio en tiempo real desarrollado en 2026.

---

# STROMA — Audio Analysis Instrument
### Especificaciones Técnicas y Marco Matemático

STROMA es un motor de análisis de señales digitales (DSP) basado en navegador que descompone la señal de audio de entrada en múltiples vectores de datos para la evaluación de la pureza tonal, la complejidad espectral y la energía de transitorios.

## 1. Fundamentos Matemáticos por Módulo

El aplicativo opera bajo una arquitectura de flujo continuo con un tamaño de buffer (FFT) de **4096 muestras**, permitiendo un balance óptimo entre resolución temporal y frecuencia.

### A. Spectrogram · CQT (Transformada de Q Constante)
A diferencia de una FFT lineal, la visualización aquí emula una resolución logarítmica para alinearse con la percepción auditiva humana.
* **Mapeo de Intensidad:** Utiliza una escala cromática donde la amplitud $A$ en el bin $k$ se normaliza: $V = \frac{|X(k)|}{max(|X|)}$.
* **Colorimetría:** Transita de azules (baja energía) a blancos (picos de amplitud), permitiendo identificar la "huella" de armónicos en el espectro.

### B. Pitch Estimate · Algoritmo pYIN
La detección de altura no se basa en un cruce por cero simple, sino en una variante de la **Función de Diferencia Cuadrática Media Acumulada (ACF)**.
* **Fórmula de Autocorrelación:** $r(\tau) = \sum_{n=0}^{N-\tau-1} x(n)x(n+\tau)$.
* **Interpolación Parabólica:** Para obtener una precisión sub-píxel en los Hertz, se calcula el vértice de la parábola formada por el lag máximo y sus vecinos.
* **Confidence (Confianza):** Es el ratio entre el pico de autocorrelación y el valor en lag zero. Un valor $>0.5$ indica una señal periódica (tonal); $<0.3$ indica ruido o inarmonicidad.

### C. Spectral Metrics (Descriptores de Audio)
* **Spectral Centroid (Centroide):** Representa el "centro de masa" del espectro. Indica qué tan "brillante" es el sonido.
    $$\text{Centroid} = \frac{\sum_{k=0}^{N-1} f(k)x(k)}{\sum_{k=0}^{N-1} x(k)}$$
* **Spectral Flatness (Planitud):** Ratio entre la media geométrica y la media aritmética del espectro. Un valor cercano a $1.0$ indica ruido blanco; cercano a $0.0$ indica un tono puro.
* **Shannon Entropy (Entropía):** Mide el desorden o la dispersión de la energía espectral.
    $$H = -\sum p_i \log_2(p_i)$$
* **Zero Crossing Rate (ZCR):** El ritmo a la que la señal cambia de signo. Es un indicador clave para distinguir entre sonidos sibilantes (fricativos) y sonidos vocálicos.

### D. Copular Metric · $\Sigma(f, c, H)$
Esta es la métrica propietaria de STROMA. Es un **operador de agregación ponderada** que sintetiza la "estabilidad" de la señal:
$$\Sigma = (w_1 \cdot \text{conf}) + (w_2 \cdot [1-\text{flatness}]) + (w_3 \cdot [1-\text{entropy}_{norm}]) + (w_4 \cdot \text{flux}_{norm})$$
Donde los pesos ($w$) priorizan la certeza del tono y el enfoque espectral.

---

## 2. Guía Metacognitiva: ¿Cómo "leer" STROMA?

Para un operador de análisis, no se trata de mirar gráficos, sino de interpretar **correlaciones vectoriales**. Aquí se explica cómo decodificar la información para una toma de decisiones técnica:

### La Correlación Pitch/Confianza
* **Escenario:** El *Piano Roll* muestra puntos dispersos y la barra de *Confidence* oscila violentamente por debajo de 0.40.
* **Interpretación:** Estás ante una señal no determinista (ruido, percusión inarmónica o interferencia). No intentes afinar esta señal; el algoritmo pYIN te está advirtiendo que la periodicidad es insuficiente para un análisis de nota fiable.

### El Triángulo de la Complejidad (Flatness vs. Entropy vs. Centroid)
* **Sonido "Afilado" y Puro:** Verás un *Centroid* bajo pero una *Flatness* casi en 0. La *Entropy* será mínima. Esto indica una onda senoidal o un instrumento con pocos armónicos pero muy definidos.
* **Sonido "Saturado" o Ruidoso:** El *Centroid* subirá hacia los 5k-10k, la *Flatness* se disparará hacia el 0.8 y la *Entropy* saturará el panel. Metacognitivamente, esto te dice que la energía está distribuida uniformemente: hay saturación de datos, "caos" en la señal.

### Interpretación de la Copular Metric $\Sigma$
Esta tabla es tu **índice de "Presencia Real"**.
* Si la línea púrpura de la *Copular Metric* se mantiene por encima del umbral de **0.5**, el evento sonoro es intencional, armónicamente rico y técnicamente "sólido".
* Si hay picos de *Flux* (línea verde en el panel de métricas) que no elevan la *Copular Metric*, son "falsos positivos" o ruidos de transitorios (golpes en el micro, aire) que no contienen información musical útil.

### El Event Log como Historial de Probabilidad
Cada entrada en el log no es una verdad absoluta, es una **instancia de alta probabilidad**. Si el log registra una nota con `conf:0.89`, puedes tratar ese dato como un valor de calibración. Si registra `conf:0.45`, es un residuo espectral que debe ser ignorado en el análisis de composición.

---

## 3. Instrucciones de Reproducción

Para desplegar la instancia de análisis:

1.  **Entorno:** Requiere un navegador compatible con `Web Audio API` (Chrome 80+, Firefox 75+, Edge).
2.  **Hardware:** Micrófono con respuesta de frecuencia plana preferiblemente.
3.  **Ejecución:**
    * Guardar el código proporcionado como `index.html`.
    * Abrir en el navegador mediante un servidor local o directamente el archivo.
    * Hacer clic en el botón **[▶ MIC]** para inicializar el `AudioContext`.
    * Observar el pie de página para verificar la latencia de proceso (idealmente $<20ms$ para análisis en vivo).

**Advertencia técnica:** El análisis se realiza en el hilo principal del navegador. Para sesiones de alta carga, cerrar pestañas en segundo plano para evitar "jitter" en la captura de los bins de la FFT.
