# KINDO — Control de stock, logística y medios de pago

**Trabajo práctico integrador · Cátedra Logística y Medios de Pago (UTN FRC)**
Docente: Lic. José I. Martino
Integrantes: Eugenio Regis · Ahumada María Celeste · Burgos Brenda · Acosta Mariana

**App en funcionamiento:** https://ahumadaceleste.github.io/control-stock-kindo/
**Planilla de datos (Google Sheets):** https://docs.google.com/spreadsheets/d/1nBd4eo3RYftmdLqjy-d8laDXFnUQEt2fzUH_KYRw_dw/edit
**Código:** `index.html` (app web) · `Código.gs.txt` (backend en Google Apps Script)

> Cada decisión se presenta con el criterio de la cátedra: **cómo** se aplica la herramienta, **por qué** se eligió y **para qué** sirve (qué objetivo resuelve). Cuando faltó un dato, el supuesto se aclara y se justifica.

---

## 1. El caso

KINDO vende indumentaria urbana oversize ("Sin Etiquetas": remeras, buzos, camperas y conjuntos) por Tiendanube. Opera desde un **dark store en barrio General Paz (Córdoba)**: un depósito sin atención al público, pensado solo para preparar y despachar pedidos online.

| Dato | Valor |
|---|---|
| SKUs activos | 14 (4 familias × talle × color) |
| Unidades en stock | 387 |
| Valor del inventario a costo | $10.983.360 |
| Facturación mensual estimada (venta prom. × PVP) | $12.425.000 |
| Margen bruto por prenda | 52% (costo = 48% del PVP) |
| Proveedores | Cba Textil (básicos) y QIWE (estampería), ambos locales |
| Lead time de reposición | 15 días |
| Canales de entrega | Uber Flash (Córdoba Capital), Andreani (federal), retiro en el dark store |

## 2. Arquitectura del sistema de información

**Cómo:** una app web (un solo `index.html`, funciona en el celular del operador) se conecta por API a un Google Apps Script que lee y escribe la planilla de Google Sheets. Cada operador entra con un PIN propio (se guarda como hash SHA-256) y cada movimiento queda registrado en la hoja `Logs_Movimientos` con fecha, operador, IP y dispositivo.

**Por qué:** es una pyme. Google Sheets no tiene costo, todo el equipo ya lo sabe usar y permite auditar a mano. La app evita que el operador tenga que editar celdas, que es donde aparecen los errores de carga.

**Para qué:** que la información fluya entre áreas en tiempo real. Lo que despacha el depósito se refleja enseguida en el stock que ve compras, y la trazabilidad permite reconstruir cualquier pedido (quién lo preparó, cuándo, en qué caja y por qué canal salió).

| Hoja | Función |
|---|---|
| `Control_Stock` | Maestro de artículos: EAN-13, SKU, ubicación, ABC, costos, stock, SS, ROP, pedido sugerido |
| `Config_KPIs` | Parámetros logísticos (lead time, coeficiente de SS, lote) y tablero de KPIs |
| `Grafico_Diente_Sierra` | Simulación del ciclo de inventario con estacionalidad |
| `Usuarios` | Operadores, rol y PIN |
| `Logs_Movimientos` | Auditoría de cada movimiento |
| `Logistica_Inversa` | Prendas devueltas no aptas (segunda / merma) |

## 3. Procesos que cubre la app

| Módulo | Cómo funciona | Por qué / para qué |
|---|---|---|
| **Despacho (pick & pack)** | Se escanea el EAN-13 con la cámara, la app muestra la ubicación exacta (pasillo-estante-nivel), calcula el costo logístico y el lead time de preparación, y emite el rótulo del bulto con QR y tracking | Reduce el tiempo de búsqueda y los errores de picking, y mide el SLA de preparación (objetivo: 45 min) |
| **Control de cobro y de salida** *(nuevo v3.6)* | Antes de despachar exige: pago **acreditado** (no solo aprobado), domicilio validado contra el titular de la compra y peso del bulto en balanza dentro de ±10% del peso esperado | Prevención de fraude y de faltantes en la caja (ver §7) |
| **Logística inversa** | Triage de devoluciones: APTO vuelve al stock, NO APTO va a segunda/merma | Recuperar valor y medir causas de devolución (talle, calce, falla) |
| **Stock** | Lista con estado OK / REORDEN / QUIEBRE según el ROP | Saber qué reponer y cuándo |
| **Recepción** | Ingreso de mercadería por proveedor | Mantener el inventario físico igual al del sistema |
| **Pagos** *(nuevo v3.6)* | Simulador del costo real de cobrar cada producto con cada pasarela y plazo | Elegir el medio de cobro y guiar al cliente hacia el más conveniente (ver §6) |

**Layout en U del depósito:** la mercadería entra por un lado y sale por el otro sin cruzar flujos. Los productos Clase A van en las zonas bajas y cerca del muelle de despacho, los C en zonas altas y alejadas. *Para qué:* recorrer menos distancia en los productos que más se mueven.

## 4. Gestión de inventario: fórmulas y justificación

### 4.1 Fórmulas usadas en la planilla

| Indicador | Fórmula | Ejemplo: Remera M Negro |
|---|---|---|
| Venta diaria | Venta prom. mensual ÷ 30 | 30 ÷ 30 = **1 u/día** |
| Stock de seguridad (SS) | Coef. SS × venta mensual | 30% × 30 = **9 u** |
| Punto de pedido (ROP) | Venta diaria × lead time + SS | 1 × 15 + 9 = **24 u** |
| Pedido sugerido | Si stock ≤ ROP → 1,5 meses de venta | stock 36 > 24 → **0** (no se pide) |
| Cobertura | Stock ÷ venta diaria | 36 ÷ 1 = **36 días** |

### 4.2 ¿Por qué un stock de seguridad del 30%?

No tenemos el desvío estándar histórico de la demanda, así que no se puede usar la fórmula estadística (SS = Z × σ × √LT). **Supuesto:** usamos un coeficiente fijo, justificado así:

- **Proveedores locales con 15 días de lead time.** El 30% de la venta mensual equivale a unos **9 días de venta**: cubre un atraso del proveedor de hasta 60% del lead time sin quebrar stock.
- **La demanda de indumentaria tiene dos picos al año**, cuando cambia la temporada (otoño-invierno y primavera-verano), más fechas como el Día de la Madre y las fiestas. En esos meses la venta sube hasta 50% (factor 1,5 en la simulación de octubre). El 30% absorbe un pico moderado; en los picos fuertes se recalcula el SS con la venta estacional (ver §4.3).
- **El margen es alto (52%)** y el costo de quebrar stock (venta perdida + cliente que se va a la competencia) es mayor que el costo de tener ~9 días de mercadería extra.
- **No se sube más** porque el stock inmoviliza capital: hoy ya hay $10,98 millones en inventario.

### 4.3 Modelo diente de sierra con estacionalidad

Simulación para Remera M Off-White en **octubre (factor 1,5)**: venta base 15 u/mes → 22,5 u/mes estacional → 0,75 u/día. SS = 30% × 22,5 ≈ **7 u**. ROP = 0,75 × 15 + 6,75 ≈ **18 u**. Capacidad máxima del estante: 50 u.

*Para qué:* ver cuándo se dispara el pedido y comprobar que la reposición llega (15 días después) antes de entrar en el stock de seguridad. Si la temporada se adelanta o el proveedor se atrasa, el gráfico muestra cuántos días de margen quedan.

### 4.4 Clasificación ABC — análisis crítico

**Criterio de la planilla:** A = 20% de los SKUs que generan ~80% de la facturación; B = 30%; C = 50%.

**Cómo lo verificamos:** ordenamos los 14 SKUs por facturación mensual (venta prom. × PVP) y calculamos el porcentaje acumulado:

| # | SKU | Clase en planilla | Facturación/mes | % | % acumulado |
|---|---|---|---|---|---|
| 1 | KND-REM-OV-G-NG | A | $1.680.000 | 13,5% | 13,5% |
| 2 | KND-REM-OV-M-NG | A | $1.260.000 | 10,1% | 23,7% |
| 3 | KND-BUZ-OV-G-NG | A | $1.224.000 | 9,9% | 33,5% |
| 4 | KND-REM-OV-G-GR | A | $1.050.000 | 8,5% | 42,0% |
| 5 | KND-BUZ-OV-M-NG | A | $1.020.000 | 8,2% | 50,2% |
| 6 | KND-CAM-OV-G-NG | **B** | $850.000 | 6,8% | 57,0% |
| 7 | KND-REM-OV-M-GR | A | $840.000 | 6,8% | 63,8% |
| 8 | KND-REM-OV-G-BL | A | $840.000 | 6,8% | 70,5% |
| 9 | KND-BUZ-OV-G-GR | B | $816.000 | 6,6% | 77,1% |
| 10 | KND-BUZ-OV-M-GR | B | $680.000 | 5,5% | 82,6% |
| 11 | KND-CAM-OV-M-NG | B | $680.000 | 5,5% | 88,0% |
| 12 | KND-REM-OV-M-BL | **A** | $630.000 | 5,1% | 93,1% |
| 13 | KND-CON-UR-G-GR | C | $475.000 | 3,8% | 96,9% |
| 14 | KND-CON-UR-M-GR | C | $380.000 | 3,1% | 100,0% |

**Conclusiones:**
- La demanda está **muy repartida**: no se cumple el 80/20 (hacen falta 10 de 14 SKUs para llegar al 80%). Es lógico porque todas las prendas tienen el mismo margen y los talles y colores se reparten la venta.
- En la planilla hay **8 SKUs clase A (57%)**, cuando el criterio dice 20%. Se clasificó por familia de producto (todas las remeras son A) y no por facturación de cada SKU. Por eso la Remera M Off-White (puesto 12) es A y la Campera G Negra (puesto 6) es B.
- **Propuesta:** reclasificar por facturación del SKU. A = hasta ~50% acumulado (puestos 1 a 5), B = hasta ~90% (6 a 11), C = el resto (12 a 14). *Para qué:* concentrar el control semanal y las ubicaciones cerca del despacho donde realmente está la plata, y bajar el stock de seguridad de lo que menos rota.
- **Observación:** los buzos Clase A tienen 66-67 días de cobertura contra ~36 de las remeras. Hay sobrestock en buzos (capital inmovilizado) que conviene frenar en el próximo pedido.

## 5. Indicadores de desempeño (KPIs)

| KPI | Cómo se mide | Meta | Para qué |
|---|---|---|---|
| Lead time de preparación | Hora del despacho − hora de ingreso de la orden | ≤ 45 min | Cumplir la promesa de entrega express (3 h) |
| Cumplimiento SLA | Pedidos dentro de SLA ÷ total | ≥ 98,5% | Detectar cuellos de botella |
| Rotación / cobertura | Stock ÷ venta diaria | 30–45 días | Evitar quiebres y sobrestock |
| Alertas de reorden / quiebres | SKUs con stock ≤ ROP / con stock 0 | 0 quiebres | Reponer a tiempo |
| Costo logístico por pedido | Packaging + flete | Seguimiento | Ver qué canal conviene |
| Margen de contribución neto | PVP − costo − logística − **costo de cobro** | > 30% | Saber cuánto deja realmente cada venta |
| Pagos verificados antes de despachar *(nuevo)* | Despachos con pago acreditado ÷ total | 100% | Cero envíos sin cobrar |
| Bultos rechazados por peso *(nuevo)* | Bultos fuera de tolerancia ÷ total | < 1% | Medir errores de preparación |
| Tasa de contracargos *(nuevo)* | Contracargos ÷ ventas con tarjeta | < 0,5% | Medir fraude |
| Tasa de devoluciones por causa | Retornos por motivo ÷ despachos | Seguimiento | Atacar la causa (ej.: guía de talles) |

## 6. Medios de pago: costo real de cobrar

> "La venta no se cierra hasta que está cobrada." El medio de cobro es el último eslabón: si falla, se pierde todo lo anterior.

### 6.1 Análisis cuantitativo

**Cómo:** para cada pasarela y plazo de acreditación calculamos comisión + IVA, el costo en pesos, el neto que llega a la cuenta y qué parte del margen bruto se lleva. El módulo **Pagos** de la app hace este cálculo para cualquier producto. Comisiones relevadas en noviembre de 2026 (sin IVA en la fuente; acá se suma IVA 21%). Pago Nube con el plan Esencial de Tiendanube.

**Remera Oversize — PVP $42.000 — margen bruto $21.840**

| Pasarela | Medio | Acreditación | Comisión c/IVA | Costo | Neto recibido | % del margen |
|---|---|---|---|---|---|---|
| Mercado Pago | Dinero en cuenta | al instante | 0,97% | $407 | $41.593 | 1,9% |
| MODO (vía Payway) | Débito | 1 día | 0,97% | $407 | $41.593 | 1,9% |
| Pago Nube | Transferencia | 1 día | 1,81% | $762 | $41.238 | 3,5% |
| MODO (vía Payway) | Crédito | 10 días | 2,18% | $915 | $41.085 | 4,2% |
| Mercado Pago | Tarjeta de crédito | 18 días | 4,10% | $1.723 | $40.277 | 7,9% |
| Pago Nube | Tarjeta / MODO | 14 días | 4,22% | $1.774 | $40.226 | 8,1% |
| Ualá Bis | Cualquier tarjeta | al instante | 5,93% | $2.490 | $39.510 | 11,4% |
| Pago Nube | Tarjeta / MODO | 1 día | 7,37% | $3.095 | $38.905 | 14,2% |
| Mercado Pago | Tarjeta de crédito | al instante | 7,61% | $3.197 | $38.803 | 14,6% |
| GOcuotas* | Débito en 2-4 cuotas | según contrato | ≈9,07% | $3.812 | $38.188 | 17,5% |

\* GOcuotas: comisión de referencia (≈7,5% + IVA) publicada en prensa; el valor real depende del contrato. No se incluyen retenciones de IIBB/SIRTAC ni el costo por transacción de Tiendanube cuando no se usa Pago Nube.

**Lectura:**
- Con tarjeta, **el plazo pesa más que la marca**: cobrar al instante cuesta casi el doble que esperar 14-18 días ($3.095–$3.197 contra $1.723–$1.774 por remera).
- Entre la opción más barata y la más cara hay **15,6 puntos del margen** por cada venta.
- Si a eso se suma la logística (~$5.000 con Uber Flash) e impuestos, se ve por qué "vender barato" no siempre deja plata.

### 6.2 Análisis cualitativo: qué usa nuestro cliente

- **Las tarjetas siguen siendo el medio principal** (~67% de los pagos online, según el material de la cátedra), y en ropa con ticket de $42.000–$95.000 el cliente busca **cuotas**.
- **Las billeteras virtuales son el medio que más crece.** No ofrecerlas es perder ventas.
- **Débito:** según NubeCommerce 2026 es solo el 4,7% de los pagos en e-commerce, pero buena parte del público joven (nuestro target "oversize urbano") **no tiene tarjeta de crédito**. GOcuotas les da cuotas con débito.
- **Regionalización:** vendemos sobre todo en Córdoba Capital (Uber Flash) y al interior (Andreani). En Córdoba son fuertes **Naranja X** y las tarjetas de bancos provinciales, así que hay que aceptarlas y conviene buscar promociones con ellas (por ejemplo, un día fijo con descuento con Naranja X). Si se vende a la Patagonia, hay que sumar las tarjetas de los bancos provinciales de allá.
- **Confianza:** parte de los clientes prefiere pagar con Mercado Pago aunque sea más caro para el comercio, porque confía en su protección al comprador. Sacarlo puede bajar la conversión.

### 6.3 Decisión propuesta para KINDO

| Decisión | Por qué | Para qué |
|---|---|---|
| **Pasarela principal: Pago Nube (tarjeta a 14 días)** | Es de las más baratas con tarjeta (4,22%), no paga el costo por transacción de Tiendanube, incluye MODO y cuotas, y se concilia sola | Bajar el costo de cobro sin perder las cuotas |
| **Acreditación a 14 días y no al instante** | El margen (52%) y el stock actual (36-67 días de cobertura) nos dejan esperar; los proveedores locales cobran a 15 días | Ahorrar ~$1.300 por remera respecto de cobrar al instante |
| **Transferencia con 1% de descuento** | Cuesta 1,81% contra 4,22% con tarjeta: son 2,4 puntos de diferencia: trasladamos 1 punto al cliente y nos quedamos con 1,4 | Guiar al cliente hacia el medio más barato para el comercio |
| **Mercado Pago como segunda opción** | Por confianza y por el dinero en cuenta (0,97%) | No perder ventas de clientes que solo confían en MP |
| **GOcuotas a prueba por 60 días** | Es caro, pero abre el segmento joven sin tarjeta de crédito | Medir si suma ventas nuevas; si no, se saca |
| **Promociones regionales con Naranja X** | Es la tarjeta más fuerte de Córdoba | Aumentar la conversión local |

*Revisión:* las comisiones y los medios cambian cada pocos meses. La tabla del módulo Pagos se actualiza en un solo lugar (`PAYMENT_OPTIONS` en `index.html`).

## 7. Fraude y control de salida

**Tipos de fraude que nos pueden afectar:** tarjetas robadas, robo de cuenta, contracargos ("no me llegó" o "llegaron menos prendas") y transferencias falsas (comprobantes truchos).

| Medida (implementada en v3.6) | Cómo | Por qué | Para qué |
|---|---|---|---|
| **Pago acreditado antes de despachar** | El botón de despacho no se habilita hasta que el operador confirma que el dinero está acreditado en la cuenta | Un pago "aprobado" puede caerse, y un comprobante de transferencia se puede falsificar | No enviar mercadería sin cobrar |
| **Validación del domicilio** | En envíos (no en retiro) se confirma que el domicilio coincide con el titular / DNI de la compra | Las compras con tarjeta robada suelen enviarse a otra dirección | Bajar el riesgo de fraude con tarjeta |
| **Control de peso del bulto** | El sistema calcula el peso esperado (prendas + packaging) y exige que la balanza dé ±10%. Si da menos, avisa "posible faltante" | Reenviar una prenda faltante duplica el costo logístico, y un faltante real o inventado termina en contracargo | Detectar errores de armado antes de que salga la caja |
| **Trazabilidad del bulto** | Rótulo con QR, tracking, orden, operador y hora; todo queda en `Logs_Movimientos` junto con el medio de pago y el peso | Ante un contracargo, la plataforma pide pruebas de entrega | Tener evidencia (más el remito firmado o el tracking de Andreani) |

## 8. Herramientas de resolución de problemas aplicadas

### 8.1 Los 5 porqués — Pedido #TN-20415 fuera de SLA

El log del 1/10/2026 registra la orden #TN-20415 despachada por Andreani con **328 minutos** de lead time (SLA objetivo: 45 min).

1. **¿Por qué tardó 328 min?** Porque la orden recién se preparó horas después de entrar.
2. **¿Por qué?** Porque nadie la vio: entró fuera del turno del operador.
3. **¿Por qué?** Porque las órdenes de Tiendanube no llegan al depósito con aviso, el operador las revisa a mano.
4. **¿Por qué?** Porque la app no está integrada con las notificaciones de Tiendanube.
5. **¿Por qué?** Porque al diseñar el sistema se priorizó el control de stock y no la entrada de pedidos.

**Causa raíz:** falta de integración entre ventas y depósito. **Acción:** activar la notificación de nueva venta de Tiendanube a un grupo del depósito y definir una franja de corte (las órdenes que entran después de X hora salen al día siguiente, y eso se le comunica al cliente al comprar).

### 8.2 Las 8 disciplinas (8D) — Bulto con una prenda faltante

| D | Acción |
|---|---|
| D1 Equipo | Operador de depósito + responsable de atención al cliente + responsable de compras |
| D2 Problema | El cliente reclama que llegaron menos prendas que las pagadas |
| D3 Contención | Reenviar la prenda faltante y guardar la evidencia del reclamo |
| D4 Causa raíz | Se armó la caja de memoria, sin verificar contra la orden |
| D5 Acción correctiva | Control de peso obligatorio antes de despachar (implementado) |
| D6 Implementación | Versión 3.6 de la app: el botón no se habilita si el peso está fuera de ±10% |
| D7 Prevención | Medir el KPI "bultos rechazados por peso" y revisarlo cada semana |
| D8 Cierre | Comunicar el nuevo procedimiento al equipo y reconocer el aporte |

### 8.3 Plan 5W2H — Implementación de los controles de cobro y de salida

| Pregunta | Respuesta |
|---|---|
| **What** (qué) | Verificar pago acreditado, domicilio y peso antes de cada despacho |
| **Why** (por qué) | Evitar envíos sin cobrar, fraude con tarjeta y faltantes en la caja |
| **Where** (dónde) | Puesto de despacho del dark store General Paz |
| **When** (cuándo) | Desde la versión 3.6, en todos los pedidos |
| **Who** (quién) | El operador de turno; el administrador revisa el log una vez por semana |
| **How** (cómo) | Desde la app: medio de pago, checks obligatorios y balanza |
| **How much** (cuánto) | Una balanza digital (precio a cotizar; es una compra única) y ~1 minuto más por pedido; se recupera con evitar un solo reenvío ($5.000–$7.400) por semana |

## 9. Por qué no se puede culpar solo al depósito

Cuando un pedido llega tarde, incompleto o se cae el cobro, la falla casi nunca es de un solo sector. El pedido atraviesa **ventas** (la web y la promesa de entrega), **cobranzas** (que el pago esté acreditado), **compras** (que haya stock), **depósito** (picking y armado) y **transporte** (Uber/Andreani). Por eso la app comparte el mismo stock y el mismo log entre todas las áreas, y por eso los KPIs están definidos por proceso y no por persona.

## 10. Prioridades para los próximos 30 días

1. **Semana 1:** usar en todos los pedidos los controles de cobro y de peso, comprar la balanza y actualizar las comisiones en la app.
2. **Semana 2:** reclasificar el ABC por facturación del SKU (§4.4) y reubicar en el depósito los productos que pasan a Clase A.
3. **Semana 2:** configurar Pago Nube a 14 días, el descuento por transferencia y la notificación de órdenes nuevas al depósito.
4. **Semana 3:** frenar la reposición de buzos hasta que la cobertura baje a ~45 días.
5. **Semana 4:** revisar los KPIs del mes (SLA, contracargos, bultos rechazados, devoluciones por talle) y decidir si GOcuotas sigue.

## 11. Mejoras técnicas pendientes

- Quitar de la hoja `Usuarios` la columna con el PIN en texto plano y dejar solo el hash; quitar del backend el campo de depuración que devuelve el PIN ingresado.
- Que el backend guarde el medio de pago y el peso en columnas propias (hoy viajan dentro del detalle de la orden en `Logs_Movimientos`, para que la versión 3.6 funcione sin cambiar el Apps Script ya publicado).

## Fuentes

- Comisiones de Pago Nube: Centro de Atención de Tiendanube, "¿Cuáles son las comisiones de Pago Nube?" (actualizado 25/09/2026).
- Comisiones de Mercado Pago (Link de pago / Checkout Pro): calculadora de comisiones de Mercado Pago 2026 de SPOM Solutions; iProfesional (28/07/2026).
- Ualá Bis: Centro de Atención de Tiendanube, "Preguntas frecuentes sobre Ualá Bis" (11/02/2026).
- MODO / Payway: iProUP, "Mercado Pago, Ualá, MODO: cuál es más barato para cobrar" (16/01/2026).
- GOcuotas: iProfesional y La Nación (comisión promedio de referencia); blog de Tiendanube (NubeCommerce 2026, débito 4,7% de los pagos).
- Material de clase de la cátedra (participación de medios de pago, fraude, embudo de conversión).
