# Gestión de Arriendos y Servicios · Grupo Terracop

Resumen del proyecto para retomarlo en otra conversación. Al abrir una conversación nueva, pedir: *"lee CONTEXTO.md del repo Aleterracop/control-arriendo"*.

## Qué es
- **index.html**: panel gerencial (solo lectura) para don Nelson, gerente de Finanzas. Lee los datos EN VIVO desde la app mediante la función `resumen_gerencial()` (sin costos ni utilidades). Si la función no existe, usa el último Excel publicado (snapshots).
- **app.html**: app de gestión para Alejandro Osorio Correa (Encargado de Gestión Comercial y Control). Solo admin (`es_admin()`).
- Hosting: GitHub Pages (`aleterracop.github.io/control-arriendo/`). Base de datos y login: Supabase (proyecto `noauwcqdxkvnbnyhqhed`), RLS en todas las tablas.

## Scripts SQL (ejecutar en orden en el SQL Editor de Supabase)
1. `sql/01_estructura.sql`: tablas mandantes, equipo_tipos, personal, materiales, parametros, flota, arriendos, periodos, horometros, edps.
2. `sql/02_datos.sql`: datos iniciales (80 equipos de flota, catálogo, 5 arriendos Fenix, horómetros, EDP).
3. `sql/03_cotizador.sql`: tablas cotizaciones y ajustes (firma, datos de empresa) + cotizaciones 1265 y 1322.
4. `sql/04_cotizaciones_historicas.sql`: crea cotizaciones adjudicadas desde los arriendos (1248, 1296, 1307, 1310).
5. `sql/05_conteo_dias.sql`: columna `arriendos.conteo_dias` (corridos | reportados).
7. `sql/07_estado_dia.sql`: columna `horometros.estado` (operativo, sin_operacion, mantencion, falla_terracop, falla_mandante, taller).
8. `sql/08_ajuste_horometros.sql`: ajuste HYPS-60 según reportes (10-09 = 7321, report 09-11, corte EDP 2 al 17-10).
9. `sql/09_flota_planilla.sql`: flota con año, patente, modelo, propiedad (propio/leasing/arriendo) y tarifas de lista sin/con operador.
10. `sql/10_costos_provisorios_y_1221.sql`: costo provisorio del catálogo = lista sin operador ÷ 1,2; HYPS-60 = cotización 1221.
6. `sql/06_resumen_gerencial.sql`: función security definer `resumen_gerencial()` que alimenta el panel gerencial en vivo.

## Funciones de app.html
- **Inicio**: KPI y avisos "Qué hacer hoy":
  - 85% y 100% de la base de horas;
  - término a 5 días o menos;
  - prórroga sin OC;
  - EDP por enviar;
  - HES pendiente;
  - entregar a Finanzas;
  - desmovilización sin cobrar.
- **Cotizar**:
  - editor de ítems agrupados (A, A.1…);
  - asistente "+ Equipo del catálogo" (equipo + operador + alimentación + alojamiento + combustible + utilidad → precio sugerido);
  - APU por ítem (🧮);
  - neto, IVA, costo y utilidad, con opción de ocultarlos;
  - "Oferta para el cliente": solo la tabla de oferta económica con firma, para pegar en el informe propio (copiar como imagen o tabla, PNG o PDF);
  - descripción con sugerencias del catálogo: al elegir, se llenan solos la unidad, el costo y un precio sugerido (editables);
  - botón 👁 por ítem: "costo interno" (no se cobra ni aparece en la oferta, pero descuenta de la utilidad). Ej.: movilización no cobrada;
  - aviso "Antes de enviar, revise": UF, dólar, diésel, IVA (editables) y costos del catálogo antiguos (>90 días) o ítems sin costo;
  - "Pasar a arriendo o prórroga";
  - "Crear extensión": nueva cotización "– Extensión N", sin movilización, enlazada al mismo arriendo.
- **Arriendos**:
  - estados cotizado / palabra (adjudicado sin OC) / con_oc / terminado / cerrado;
  - pestañas Resumen, Horómetros o "Días y reportes", Períodos, EDP y HES;
  - botones Prorrogar, Registrar OC, Terminar servicio (EDP final con desmovilización), Nuevo EDP.
- **Flota**: agregar, editar y dar de baja equipos.
- **Catálogo**: equipos (tarifa vacía = pendiente de precio), personal, materiales, parámetros, Empresa y firma.
- **Mandantes**: `requiere_hes` (mineras sí; particulares no, se registra la aprobación).

## Reglas de negocio acordadas
- Tarifas de lista (planilla 16-11): "tarifa máquina" = precio sin operador; "con operador" = precio con operación continua turno día (2 operadores en rotación, mes completo c/u). Siempre conviene más con operador.
- Patente confirmada: HSZC-48 (EXC-206). HYPS-60 = EXC-209, cotización 1221.
- Costos del catálogo son PROVISORIOS (lista sin operador ÷ 1,2; martillo 2.875.000) hasta tener costos reales del taller (mantención, desgaste, seguro, reposición; leasing: cuota mensual). Todo precio y costo debe poder editarse a mano.
- Base de horas por EDP: 180 h (mínimo mensual). Prórrogas normalmente de 180 h; Fenix exige cotización por cada extensión.
- La movilización se cobra en el 1er EDP; las extensiones no la cobran; la desmovilización va en el EDP final.
- La HES la emite el mandante al aprobar el EDP. Con N° de HES (o aprobación, si es particular), el EDP pasa a Finanzas. La facturación no es del área.
- Horas trabajadas sobre lo cobrado en un EDP se arrastran al período siguiente; si se trabajó menos (mínimo), no se descuenta. El historial de horómetros muestra los cortes de cada EDP y marca saltos de horómetro o de report (se guarda lo que envía faena). Exporta a Excel el respaldo "REGISTRO DIARIO DE HORAS MÁQUINA".
- Días con falla del equipo (Terracop) o en taller no se cobran en arriendos por día.
- Cobro por hora (hm/hh) usa horómetro obligatorio. Cobro por día o mes cuenta días corridos o días con reporte; el horómetro es opcional (el martillo no tiene).
- Cotizaciones en neto; el EDP agrega IVA 19%.
- La movilización la ejecuta Transporte Terracop y la factura internamente a Terracop: su tarifa interna es el costo (catálogo "Flete Transporte Terracop – …"). Lo mismo con Maestranza Terracop. A veces se cobra solo la movilización, se rebaja o se absorbe en el precio del equipo.
- El precio sugerido se calcula como costo × (1 + GG) × (1 + utilidad). Utilidad por defecto: 20%.

## Pendiente
- EDP automático con formato Fenix: portada + "Estado Pago Contratista" (contrato, avance anterior, avance del período, acumulado, IVA) + respaldo de horómetros. Ejemplo recibido: EDP N°00001 OC 4300002595.
- Avisos diarios por correo.
- Respaldo del EDP N°1 con posibles errores de tipeo:
  - 05-11: horómetro inicial 6803 → ¿6903?
  - report 27654 → ¿27564?
  - salto entre 6792 y 6800 (22-23 oct).
