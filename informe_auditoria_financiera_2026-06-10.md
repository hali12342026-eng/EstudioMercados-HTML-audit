# Informe de auditoría financiera y técnica

**Proyecto:** `mpsolana/EstudioMercados-HTML`  
**Copia auditada:** https://github.com/hali12342026-eng/EstudioMercados-HTML-audit  
**Fecha:** 10/06/2026  
**Objetivo:** revisar bugs, glitches serios y cálculos que podían no corresponderse con matemáticas financieras rigurosas o podían inducir a una interpretación económica incorrecta.

## Resumen ejecutivo

La aplicación tiene una base útil para análisis de mercados, activos y carteras, pero había varios puntos donde el resultado mostrado podía ser financieramente engañoso aunque la pantalla pareciera funcionar. Se han corregido los errores más relevantes sin rediseñar la aplicación.

Los cambios principales afectan a:

1. **Análisis de cartera:** se corrige la gestión de pesos cuando falla un ticker y se alinea la matriz de covarianzas con los pesos reales de cada activo.
2. **Correlaciones de cartera:** se dejan de sumar rentabilidades simples para periodos mensuales/semestrales/anuales y se pasa a usar rentabilidad compuesta.
3. **Divisas:** se corrige la interpretación de pares `EURXXX=X` para que el gráfico realmente muestre la divisa frente al euro, no el euro frente a esa divisa.
4. **Rentabilidades anualizadas móviles:** se corrige la media anualizada para calcularla sobre observaciones anualizadas, no anualizando la media de rentabilidades totales.
5. **Tabla anual:** se sustituye el agrupamiento mecánico por bloques de 252 sesiones por agrupación real de años calendario.
6. **Robustez estadística:** se añaden protecciones ante arrays vacíos o muestras insuficientes en media, desviación típica, percentiles y formato porcentual.

## Cambios aplicados

### 1. Pesos de cartera tras fallo de descarga

**Problema:** si uno o varios tickers fallaban al descargar datos, el código los eliminaba, pero no renormalizaba los pesos restantes.  

**Por qué importa:** una cartera 50/50 donde falla un activo pasaba a comportarse como si el activo válido pesara 50% y el resto quedara implícitamente como efectivo sin rentabilidad, aunque el usuario no hubiera definido caja. Eso reduce artificialmente volatilidad, rentabilidad y drawdown.

**Corrección:** tras eliminar tickers fallidos, los pesos restantes se renormalizan para sumar 100%.

### 2. Volatilidad/covarianza de cartera alineada con tickers reales

**Problema:** el vector de pesos se construía desde `entries`, mientras la matriz de covarianzas se construía desde `tickers`. Aunque normalmente coincidían, dependía del orden y podía desalinearse en casos con fallos o cambios de estructura.

**Por qué importa:** si los pesos no coinciden exactamente con el orden de los activos en la matriz, la volatilidad de cartera deja de representar la cartera real.

**Corrección:** se crea un mapa `ticker → peso` y el vector de pesos se construye siguiendo exactamente el orden de `tickers` usado por la matriz.

### 3. Correlaciones agregadas por periodo

**Problema:** para correlaciones mensuales, semestrales o anuales, el código agregaba retornos diarios sumándolos.

**Por qué importa:** la suma de retornos simples es una aproximación pobre cuando hay volatilidad. El retorno correcto de un periodo es compuesto:  

`(1+r1) × (1+r2) × ... × (1+rn) - 1`

En activos volátiles, la diferencia puede ser material y afectar a la correlación estimada.

**Corrección:** se sustituyó la suma por capitalización compuesta de retornos diarios.

### 4. Escala de color en matriz de correlación

**Problema:** la escala visual iba de azul a blanco, por lo que el cero no quedaba claramente neutral y las correlaciones positivas no tenían un color intuitivo diferenciado.

**Por qué importa:** una matriz de correlación debe permitir ver rápido diversificación real: negativo, neutro y positivo. Una escala mal centrada puede inducir a interpretar correlaciones moderadas como inocuas.

**Corrección:** se aplica escala divergente: azul para correlación negativa, blanco para cero, rojo para positiva.

### 5. Divisas frente al euro

**Problema:** Yahoo devuelve pares como `EURUSD=X`, que significan USD por 1 EUR. Si se etiqueta como USD/EUR o “divisa vs EUR”, hay que invertir la serie. Antes, una subida de `EURUSD` se mostraba como subida del USD frente al EUR, cuando en realidad es apreciación del euro frente al dólar.

**Por qué importa:** es un error económico directo de signo/dirección. Puede llevar a concluir que una divisa se aprecia frente al euro cuando ocurre lo contrario.

**Corrección:** se transforman las series `EURXXX=X` a `XXX/EUR` mediante `1 / precio`. Las etiquetas pasan a mostrarse como `USD/EUR`, `GBP/EUR`, `JPY/EUR`, etc.

### 6. Conversión a EUR en simulaciones

**Problema:** se convertía todo el histórico usando el último EURUSD disponible.

**Por qué importa:** convertir un histórico completo con el tipo de cambio actual no es un backtest en euros; es una repricing artificial a spot que puede distorsionar la lectura histórica.

**Corrección:** el histórico se deja en unidades nativas. La conversión aproximada a EUR se limita al punto de partida/visualización forward usando spot, con etiqueta explícita `EUR (spot aprox.)`.

### 7. Rentabilidades anualizadas móviles

**Problema:** la media anualizada se calculaba anualizando la media de rentabilidades totales de las ventanas.

**Por qué importa:** por convexidad y volatilidad, anualizar una media de retornos totales no equivale a promediar las rentabilidades anualizadas de cada ventana.

**Corrección:** cada ventana se anualiza individualmente y luego se calcula la media de esas observaciones anualizadas.

### 8. Estadísticas anuales

**Problema:** la tabla anual se calculaba cortando la serie en bloques de 252 observaciones desde el inicio, no por años calendario.

**Por qué importa:** una fila “anual” podía mezclar partes de dos años naturales, lo que no corresponde al uso financiero habitual de análisis anual.

**Corrección:** se agrupa por año calendario real y se calcula rentabilidad y volatilidad dentro de cada año.

### 9. Percentiles y funciones estadísticas básicas

**Problema:** el percentil usaba el valor inferior sin interpolación, y media/desviación podían devolver `NaN` en muestras vacías o insuficientes.

**Por qué importa:** en tablas de simulación, VaR o distribuciones, un percentil mal calculado o un `NaN` silencioso deteriora la calidad del análisis.

**Corrección:** percentiles interpolados y funciones defensivas para arrays vacíos o muestras inferiores a dos datos.

## Bugs/glitches técnicos corregidos

- Se mantiene la validación sintáctica del JavaScript inline con `node --check`.
- Se evita que la eliminación de tickers descargados fallidos deje carteras con pesos implícitos incorrectos.
- Se reducen lecturas visuales erróneas en correlación y divisas.
- Se conserva el diseño general y el flujo de uso existente para no introducir cambios funcionales innecesarios.

## Limitaciones que siguen pendientes

Estas cuestiones no se han tocado para evitar cambiar demasiado el alcance sin una revisión funcional más larga:

1. **Dependencia de Yahoo Finance:** los datos pueden fallar, cambiar de ticker o venir con huecos. Para uso profesional conviene mostrar fuente, fecha de última descarga y advertencia de calidad de datos.
2. **Simulación Monte Carlo:** sigue siendo un modelo simplificado basado en distribución normal/lognormal de retornos históricos. No captura colas gruesas, cambios de régimen, autocorrelación, volatilidad estocástica ni crisis de liquidez.
3. **Rentabilidades con dividendos:** según el campo que devuelva Yahoo, puede no estar garantizado que todos los activos estén en total return comparable. Esto es crítico para ETFs, índices price return vs total return y fondos.
4. **Horizontes por sesiones fijas:** algunos apartados siguen usando 21/63/252 sesiones como aproximación mensual/trimestral/anual. Es aceptable para análisis rápido, pero no idéntico a calendario real.
5. **Costes, impuestos y divisa de inversión:** la herramienta no parece incorporar comisiones, fiscalidad, spreads ni cobertura de divisa.

## Validaciones realizadas

- Revisión estática del código principal `index.html`.
- Extracción de scripts inline y validación sintáctica con Node.js: correcta.
- Revisión del diff con finales de línea preservados para evitar cambios masivos artificiales.
- Commit y push realizados en repo propio.

## Enlace al repositorio corregido

https://github.com/hali12342026-eng/EstudioMercados-HTML-audit

Commit principal: `8f87141` — `fix: tighten financial calculations and market displays`

## Conclusión

La versión corregida es más consistente con matemáticas financieras básicas y evita varios sesgos de interpretación importantes, especialmente en cartera, correlaciones y divisas. Para convertirla en una herramienta robusta de uso profesional, el siguiente paso sería auditar fuente de datos, total return/dividendos, supuestos de simulación y validación visual con casos de prueba conocidos.
