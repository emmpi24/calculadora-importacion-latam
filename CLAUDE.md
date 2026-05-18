# Calculadora de Importación LATAM — Piloto Uruguay — Claude Project Guide

## Contexto del desarrollador
El usuario (Ema) es **principiante absoluto** en desarrollo web. Este es su **primer proyecto completo**. Está aprendiendo con Claude Code y Antigravity. Priorizar siempre:
- Explicar brevemente qué hace cada paso antes de ejecutarlo
- Comandos de terminal de a uno, nunca en bloque sin contexto
- Avisar antes de hacer algo que pueda romper el proyecto

---

## ¿Qué es este proyecto?
Herramienta web con alcance **Latinoamérica** que combina dos funciones en una sola pantalla:

1. **Buscador de productos:** el usuario escribe lo que quiere comprar y el sitio genera links directos a los resultados de búsqueda en Amazon, MercadoLibre, eBay, AliExpress, Temu y Tiendamia. Cada link lleva el código de afiliado embebido cuando corresponde. Si el usuario compra, se genera una comisión automática.

2. **Calculadora de costos por país:** el usuario selecciona su país y la herramienta aplica los impuestos, aranceles, umbrales de exención e intermediarios que correspondan. Muestra el total en USD y en moneda local.

**Alcance y estado:**
- **Piloto activo: Uruguay** — único país completamente funcional al lanzamiento.
- **Próximamente:** Argentina, Brasil, Chile, Colombia, Perú, México — visibles en el selector pero deshabilitados hasta validar sus reglas tributarias.
- **Arquitectura preparada desde el día 1** para agregar países sin reescribir la lógica de cálculo (ver "Arquitectura de países" abajo).

**Problema que resuelve:** nadie sabe cuánto termina pagando realmente al importar desde el exterior, y las reglas cambian por país. Buscar en cada tienda por separado además es engorroso.

---

## Stack Técnico
| Capa | Tecnología |
|---|---|
| Lenguaje | HTML + CSS + JavaScript puro |
| Framework | Ninguno |
| Hosting | Vercel o GitHub Pages (gratuito) |
| Repositorio | GitHub |
| Base de datos | Ninguna — todo ocurre en el navegador |

**¿Por qué tan simple?** Es el primer proyecto. Sin frameworks ni dependencias. Un solo archivo HTML. El objetivo es aprender el flujo completo: construir → deployar → monetizar.

---

## Flujo del usuario (referencia crítica)

```
1. Usuario entra al sitio
2. Escribe el producto que busca (ej: "guantes de boxeo")
3. El sitio muestra tarjetas por tienda:
   - 🛍️ Amazon   → link directo a resultados con código de afiliado
   - 🛒 Tiendamia → link directo a resultados con código de afiliado
   - 🛍️ eBay      → link directo a resultados con código de afiliado
4. Usuario hace clic → llega directo a los resultados en la tienda
5. Si compra → comisión acreditada automáticamente
6. Debajo del buscador: calculadora para estimar costo final con impuestos
```

**Importante:** los links directos a resultados de búsqueda SON el botón de compra.
No hay botón separado a la página principal de cada tienda — eso sería redundante y rompe el flujo.

---

## Estructura de links de afiliado

```
Amazon:    https://www.amazon.com/s?k=BUSQUEDA&tag=TU_CODIGO
Tiendamia: https://www.tiendamia.com/search?q=BUSQUEDA
eBay:      https://www.ebay.com/sch/i.html?_nkw=BUSQUEDA
```

La búsqueda del usuario reemplaza dinámicamente `BUSQUEDA` en cada URL via JavaScript.
Los códigos de afiliado se configuran una sola vez en el archivo y se insertan automáticamente en todos los links.

---

## Calculadora — Lógica de cálculo

La lógica es **genérica y data-driven**: lee el país activo desde el objeto `PAISES` y aplica sus reglas. Los números concretos viven en la configuración, no en el código de cálculo.

```
cfg = PAISES[paisActivo]

subtotal = precio_producto + envio_internacional (usuario ingresa el envío real)

SI subtotal > cfg.umbralIVA:
  IVA = subtotal × cfg.iva
SINO:
  IVA = 0  (exento)

SI subtotal > cfg.umbralArancel:
  base    = (cfg.arancelMode === 'excedente') ? subtotal - cfg.umbralArancel : subtotal
  arancel = max(0, base × cfg.arancel - cfg.arancelDescuento)
SINO:
  arancel = 0  (exento)

intermediario = cfg.intermediarios.find(id).costo(subtotal)

total_USD   = subtotal + IVA + arancel + intermediario
total_local = total_USD × tipo_de_cambio
```

**Estimación automática de envío (clave para que la calculadora sea útil):**

La calculadora ahora **estima el envío automáticamente** según tienda + país + peso. Solo pide al usuario lo mínimo indispensable.

| Tienda | Campos pedidos | Envío |
|---|---|---|
| **Amazon / eBay / Tiendamia** | Precio + Peso | Auto-estimado por peso (editable) |
| **Temu / AliExpress** | Solo Precio | Incluido en el precio → no se suma |
| **Mercado Libre internacional** | Precio + Envío manual | Usuario ingresa el envío |
| **Mercado Libre local** | Precio + Envío (opcional) | Sin impuestos importación |

**Fórmula del envío estimado:** `envío = fijo + (peso × porKg) + (precio × tarifaPct)`

**Tabla de costos estimados (`COSTOS_ENVIO` en index.html):**

| Tienda | País | Fijo (USD) | Por kg (USD) | % sobre precio |
|---|---|---|---|---|
| Tiendamia | UY | $5 | $22 | 9.5% |
| Tiendamia | CL | $6 | $26 | 9.5% |
| Tiendamia | CO | $6 | $25 | 9.5% |
| Tiendamia | PE | $6 | $25 | 9.5% |
| Tiendamia | MX | No opera | — | — |
| Amazon | UY/CL/CO/PE | $12 | $15 | 0% |
| Amazon | MX | $10 | $12 | 0% |
| eBay | UY/CL/CO/PE | $8 | $18 | 0% |
| eBay | MX | $7 | $15 | 0% |

**Reglas de visibilidad:**
- Si el usuario no ingresa peso, se asume 0.5 kg como default razonable.
- El campo "Envío" siempre es editable: si el usuario sabe el envío real, lo ingresa y se respeta.
- Si la tienda no opera en el país (ej. Tiendamia MX), el botón Calcular se deshabilita y se muestra mensaje.

**Doble umbral**: a partir de 2026 algunos países tienen umbrales distintos para IVA y arancel (Colombia: IVA desde $50, arancel desde $200). El campo `umbralExento` único quedó deprecado — usar `umbralIVA` y `umbralArancel` por separado.

**`arancelMode: 'excedente'`** y **`arancelDescuento`**: preparados para activar Argentina (arancel sobre lo que supera USD 400) y Brasil (60% II - USD 20 fijo). No usados con los 5 países activos al 2026.

### Tasas oficiales por país (mayo 2026)

| País | IVA / equivalente | Arancel | Umbral IVA | Umbral arancel | Fuente |
|---|---|---|---|---|---|
| 🇺🇾 Uruguay | 22% (IVA) | 10% | $200 USD | $200 USD | Aduana Uruguay |
| 🇨🇱 Chile | 19% (IVA) | 6% | **$0** | $500 USD | [SII Chile](https://www.sii.cl/destacados/iva_bienes/) |
| 🇨🇴 Colombia | 19% (IVA) | 10% | $50 USD | $200 USD | [DIAN](https://www.dian.gov.co/Viajeros-y-Servicios-aduaneros/Paginas/Modalidad-de-trafico-postal-y-envios-urgentes.aspx) |
| 🇵🇪 Perú | 18% (IGV) | 4% | $200 USD | $200 USD | [SUNAT](https://www.sunat.gob.pe/orientacionaduanera/importaFacil/index.html) |
| 🇲🇽 México | 16% (tasa global) | — | $50 USD | $50 USD | [SAT](http://omawww.sat.gob.mx/aduanas/importando_exportando/guia_importacion/Paginas/contribuciones_que_puedan_causarse_con_importacion.aspx) |

**Notas críticas**:
- **Chile**: IVA 19% se cobra desde el primer dólar (cambio oct 2025). Arancel solo aplica sobre USD 500.
- **Colombia**: Decreto 1474/2025 redujo el umbral de exención de IVA de $200 a $50.
- **México**: "Tasa global SAT" 16% (boleta aduanal) cubre IVA + arancel en un solo cobro entre USD 50–1000. Por eso `arancel: 0`.
- **Productos chinos**: en México 2026 tienen aranceles diferenciales (TIGIE 5-50%). La calculadora muestra disclaimer al seleccionar Temu o AliExpress.

### Países "Próximamente" (no activos)

- 🇦🇷 **Argentina**: cupo USD 3000/año, IVA 21%, arancel 50% sobre excedente de USD 400. Régimen cambia frecuentemente.
- 🇧🇷 **Brasil**: Remessa Conforme con II 60% (descuento USD 20) + ICMS estadual 17-20%. Cálculo complejo.

### Casos especiales por tienda

- **Mercado Libre**: selector "Tipo de compra" (Internacional / Local). Local omite IVA y arancel — solo precio + envío local.
- **Temu / AliExpress**: disclaimer informativo sobre aranceles diferenciales en México. El cálculo usa la tasa general.

### Intermediarios por país

- **UY**: Tiendamia (+20%), OCA (+10% + $8), Correo UY (+5% + $5)
- **CL**: Chilexpress (+10% + $6), Correos de Chile (+5% + $4), DHL (+15% + $10)
- **CO**: Servientrega (+8% + $5), Coordinadora (+6% + $4), DHL (+15% + $10)
- **PE**: Serpost (+5% + $3), Olva Courier (+8% + $5), DHL (+15% + $10)
- **MX**: Correos de México (+5% + $4), Estafeta (+8% + $5), DHL (+15% + $10)

---

## Arquitectura de países (LATAM)

El objeto `PAISES` en `index.html` centraliza todas las reglas tributarias por país. La función `calcular()` nunca tiene números hardcoded — todo viene de `PAISES[paisActivo]`.

### Cómo agregar un país nuevo

1. Investigar y validar las reglas tributarias oficiales del país (autoridad aduanera).
2. Completar el objeto del país dentro de `PAISES`:
   ```js
   AR: {
     nombre: "Argentina",
     bandera: "🇦🇷",
     moneda: "ARS",
     tasaUSDDefault: 1050,
     iva: 0.21,
     arancel: 0.50,
     umbralIVA: 0,             // siempre paga IVA
     umbralArancel: 400,       // arancel sobre USD 400
     arancelMode: 'excedente', // arancel solo sobre lo que supera el umbral
     arancelDescuento: 0,
     ivaLabel: 'IVA',
     arancelLabel: 'Arancel',
     organismo: 'AFIP/ARCA',
     fuenteOficial: 'https://www.afip.gob.ar/...',
     notaPais: 'Régimen courier: cupo USD 3000/año, máx. 5 envíos.',
     activo: true,             // ← cambiar a true cuando se valide
     intermediarios: [
       { id: "directo", nombre: "Sin intermediario", costo: () => 0 },
       // ...intermediarios locales
     ]
   }
   ```
3. **No tocar** `calcular()` ni la UI. El selector y el formulario se actualizan solos.

### Regla importante
**Nunca activar un país sin verificar sus reglas tributarias reales**. Si se muestra un cálculo incorrecto, se pierde credibilidad. Mejor mantenerlo como "Próximamente" hasta tener fuentes oficiales.

---

## Requisitos mínimos de compra y restricciones por tienda

**IMPORTANTE:** Esta información cambia frecuentemente. Se actualiza cada cierto tiempo pero siempre verifica directamente en la tienda antes de completar tu compra.

### Por país y tienda

| País | Tienda | Requisito mínimo / Restricción | Verificar en |
|---|---|---|---|
| 🇺🇾 **Uruguay** | **Temu** | Monto mínimo ~$1200 pesos UYU (USD 30-40 aprox.). Varía con promociones. | [temu.com](https://www.temu.com) |
| | **Mercado Libre** | Cupo anual USD 800 en máx 5 envíos. Máx 3 productos iguales. Requiere clave fiscal nivel 2. | [mercadolibre.com.uy](https://www.mercadolibre.com.uy) |
| | **Tiendamia** | Cupo anual USD 800 desde mayo 2026 (antes USD 200/compra). Solo productos nuevos desde USA. | [tiendamia.com.uy](https://tiendamia.com.uy) |
| | **Amazon / eBay** | Sin mínimo de plataforma. Depende del vendedor individual. | [amazon.com](https://www.amazon.com) / [ebay.com](https://www.ebay.com) |
| | **AliExpress** | Sin mínimo de plataforma. Algunos vendedores cobran envío según montos. | [aliexpress.com](https://www.aliexpress.com) |
| 🇨🇱 **Chile** | **Temu** | Requisitos varían por promociones. Verificar directamente. | [temu.com](https://www.temu.com) |
| | **Tiendamia** | No disponible en Chile actualmente. | N/A |
| | **Mercado Libre** | No disponible para compras internacionales. | [mercadolibre.cl](https://www.mercadolibre.cl) |
| | **Amazon / eBay / AliExpress** | Sin mínimo de plataforma. | Sitios oficiales |
| 🇨🇴 **Colombia** | **Temu** | Requisitos varían por promociones. Verificar directamente. | [temu.com](https://www.temu.com) |
| | **Tiendamia** | Disponible. Verifica límites según aduanas locales. | [coltiendamia.online](https://coltiendamia.online/) |
| | **Mercado Libre** | No disponible para compras internacionales. | [mercadolibre.com.co](https://www.mercadolibre.com.co) |
| | **Amazon / eBay / AliExpress** | Sin mínimo de plataforma. | Sitios oficiales |
| 🇵🇪 **Perú** | **Temu** | Requisitos varían por promociones. Verificar directamente. | [temu.com](https://www.temu.com) |
| | **Tiendamia** | Máximo USD 200 por compra. Límite anual USD 2000 (máx 3 compras con DNI). | [tiendamia.com.pe](https://tiendamia.com.pe) |
| | **Mercado Libre** | No disponible para compras internacionales. | [mercadolibre.com.pe](https://www.mercadolibre.com.pe) |
| | **Amazon / eBay / AliExpress** | Sin mínimo de plataforma. | Sitios oficiales |
| 🇲🇽 **México** | **Temu** | Requisitos varían por promociones. Verificar directamente. | [temu.com](https://www.temu.com) |
| | **Tiendamia** | No disponible en México actualmente. | N/A |
| | **AliExpress (chino)** | Sin mínimo. Aranceles TIGIE 5-50% según producto (este cálculo usa tasa general). | [aliexpress.com](https://www.aliexpress.com) |
| | **Mercado Libre** | No disponible para compras internacionales. | [mercadolibre.com.mx](https://www.mercadolibre.com.mx) |
| | **Amazon / eBay** | Sin mínimo de plataforma. | Sitios oficiales |

**Notas:**
- Los requisitos pueden cambiar sin previo aviso. Cada tienda actualiza sus políticas.
- Los disclaimer se muestran automáticamente en la calculadora cuando seleccionas tienda + país.
- Mercado Libre internacionales: en 2026 solo habilitan desde Argentina. Otros países pueden agregar esta funcionalidad posteriormente.
- El cupo anual de Uruguay (USD 800) es un límite de importaciones preferencial — superar esto puede afectar futuros envíos.

---

## Monetización
- **Afiliados Amazon:** código embebido en cada link de búsqueda (comisión 4–10% por compra)
- **Afiliados Tiendamia:** código embebido en cada link de búsqueda
- **AdSense (futuro):** una vez que haya tráfico aprobado por Google

> ⚠️ Amazon PA API (para mostrar imágenes y precios reales) requiere aprobación previa con tráfico demostrable. El MVP usa links de búsqueda sin API — funciona igual para generar comisiones.

---

## SEO — palabras clave objetivo
Incluir estas frases en el título, descripción meta y texto visible desde el día uno:
- "calculadora importación Uruguay"
- "cuánto pago de impuestos comprando en Amazon Uruguay"
- "Tiendamia costo real Uruguay"
- "comprar en eBay desde Uruguay impuestos"

---

## Diseño y UX
- Responsive — debe verse bien en celular
- Una sola página (sin rutas múltiples)
- Buscador arriba, calculadora abajo
- Tarjetas por tienda con ícono y color distintivo de cada una
- Sin imágenes de productos (requieren API aprobada — fase futura)
- Aviso claro si la compra supera o no el límite de $200 USD

---

## Flujo de trabajo de desarrollo
```
Archivo local (index.html) → GitHub (repositorio) → Vercel (sitio en vivo)
```
Cada `git push` actualiza el sitio automáticamente. Deploy instantáneo por ser archivo estático.

---

## Reglas para Claude Code (Cleo)
- Todo el proyecto vive en un solo archivo `index.html` al inicio
- No usar librerías externas si se puede evitar
- Priorizar que funcione correctamente antes de que se vea perfecta
- Si hay dos formas de hacer algo, elegir la más simple
- Explicar el "por qué" de cada decisión — el usuario aprende mientras construye
- No agregar funciones nuevas hasta que la actual esté deployada y probada
- Los disclaimers y requisitos mínimos de tiendas se actualizan en CLAUDE.md regularmente (información puede cambiar)
- La moneda del precio cambia automáticamente al seleccionar país (mejor UX, el usuario siempre puede cambiarla manualmente)

---

## Fases de evolución (no construir todo junto)

```
FASE 1 — MVP LATAM con 5 países activos (estado actual)
  → Buscador con links de afiliado a 6 tiendas (Amazon, MercadoLibre, eBay,
    AliExpress, Temu, Tiendamia)
  → Calculadora data-driven con UY, CL, CO, PE, MX activos
  → Lógica de doble umbral (IVA/arancel pueden tener umbrales distintos)
  → Casos especiales: Mercado Libre local y disclaimer Temu/AliExpress
  → Diseño con paleta naranja/violeta neon sobre fondo gris degradé
  → Globo terráqueo decorativo
  → Deploy en Vercel

FASE 2 — Activar Argentina y Brasil
  → Investigar reglas tributarias oficiales actualizadas (AFIP/ARCA, Receita Federal)
  → Completar configs en PAISES usando arancelMode='excedente' (AR)
    y arancelDescuento (BR)
  → Validar cálculos con casos reales antes de marcar activo: true
  → Validar cada cálculo con casos reales antes de habilitarlo

FASE 3 — Integración real de productos (cuando haya tráfico)
  → Solicitar Amazon PA API
  → Mostrar imágenes, precios y nombres reales
  → Catálogo visual completo

FASE 4 — Monetización adicional
  → Solicitar Google AdSense
  → Optimizar SEO por país (landings localizadas)
```

---

## Estado actual del proyecto — LISTO PARA PUBLICACIÓN

### ✅ Completado
- [x] Entorno local configurado (Node.js, Git)
- [x] index.html con buscador funcionando
- [x] Calculadora con **5 países LATAM activos**: 🇺🇾 UY, 🇨🇱 CL, 🇨🇴 CO, 🇵🇪 PE, 🇲🇽 MX
- [x] 🇦🇷 AR y 🇧🇷 BR como "Próximamente" (regulación inestable)
- [x] Tasas oficiales verificadas: SII, DIAN, SUNAT, SAT (mayo 2026)
- [x] Lógica de doble umbral (IVA y arancel con umbrales independientes)
- [x] Selector de tienda (Amazon, eBay, Tiendamia, Mercado Libre, AliExpress, Temu)
- [x] ML local vs internacional
- [x] Disclaimers dinámicos por tienda/país
- [x] Cambio automático de moneda al seleccionar país
- [x] Campo de peso para auto-estimación de envío (Amazon, eBay, Tiendamia)
- [x] Envío incluido automático para Temu/AliExpress (no pide peso)
- [x] Botón "Calcular" se deshabilita si tienda no opera en el país (ej. Tiendamia MX)
- [x] Tabla de restricciones de tiendas en CLAUDE.md
- [x] Arquitectura data-driven `PAISES` lista para expansión
- [x] Diseño visual moderno (naranja+violeta neon, fondo gris degradé, globo)
- [x] SEO básico (título, descripción meta, keywords)
- [x] Disclaimers de confiabilidad (envío real de tienda, impuestos precisos)

### ⏳ Próximas etapas
- [ ] Repositorio GitHub creado
- [ ] Sitio deployado en Vercel
- [ ] Links de afiliado Tiendamia configurados
- [ ] Links de afiliado Amazon configurados (AMAZON_TAG es placeholder)
- [ ] AdSense solicitado (con tráfico inicial)
- [ ] Amazon PA API (cuando haya volumen de usuarios)
