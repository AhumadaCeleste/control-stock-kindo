# KINDO OPS — Dark Store Hub

App web de control de stock, despacho, logística inversa y medios de pago para KINDO (indumentaria oversize, dark store General Paz, Córdoba).
Trabajo práctico de la Cátedra Logística y Medios de Pago (UTN FRC).

- **App:** https://ahumadaceleste.github.io/control-stock-kindo/
- **Informe del trabajo práctico:** [INFORME.md](INFORME.md)
- **Planilla (base de datos):** [Google Sheets](https://docs.google.com/spreadsheets/d/1WwYI8DMqHf3tyEuCnV-ealJwefoGn37YgRKfO7bpNV8/edit)

## Cómo usarla

1. Abrir la app (funciona en el celular o en la computadora).
2. Ingresar con el PIN de operador (cada usuario tiene el suyo en la hoja `Usuarios`).
3. Pestañas: **Despacho** (escanear EAN-13 o tipear SKU, por ejemplo `KND-REM-OV-M-NG`), **Retornos**, **Stock**, **Recepción** y **Pagos**.

## Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | App completa (HTML + CSS + JS en un solo archivo) |
| `index.html.txt` | Copia de la app en texto |
| `Código.gs.txt` | Backend en Google Apps Script vinculado a la planilla |
| `INFORME.md` | Informe con las justificaciones (cómo, por qué y para qué) |

## Versiones

- **v3.6.0-ui** — Pestaña **Pagos** (costo real de cobrar por pasarela y plazo) y **control de cobro y de salida** en el despacho: pago acreditado, domicilio y peso del bulto antes de despachar. El costo de cobro se descuenta del margen neto.
- **v3.5.0** — Costeo logístico directo, lead time y SLA, rótulo con QR, logística inversa con triage.
- Historial anterior: ver los commits.

Autoría original: Eugenio Regis y equipo (repositorio [kuque84/Control-Stock-Kindo](https://github.com/kuque84/Control-Stock-Kindo)).
