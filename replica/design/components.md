# Componentes (especificación)

Adaptación nativa: SwiftUI en iOS, Compose for Wear OS (Material 3) en el reloj. Iconos: SF Symbols (iOS) y Material Symbols (reloj, Apache 2.0). Ningún recurso de Merge.

```
EstadoConexion (I04, W02, W09)
  variantes  conectado, reconectando, desconectado, sin emparejar
  tokens     success / warning / danger / text-muted; icono + texto (nunca solo color)
  a11y       VoiceOver/TalkBack leen "Conectado al reloj, hace 2 minutos"

FilaApp (I05)
  partes     icono genérico + nombre de la app + interruptor
  estados    activada, desactivada, vista por primera vez (etiqueta "nueva")
  a11y       el interruptor nombra la app

CodigoEmparejamiento (I03, W01)
  partes     6 dígitos grandes (lo muestra el sistema); la app muestra la instrucción
  estados    esperando, coincide, rechazado, tiempo agotado

BotonesLlamada (W04)
  variantes  aceptar (success), rechazar (danger); 48 dp mínimo, separados
  estados    activo, enviando, sin conexión (deshabilitado)

TarjetaMetrica (I06, W05)
  partes     etiqueta, valor, unidad, hora del último dato
  estados    con dato, sin dato ("—"), pendiente de enviar

FilaLog (I07)
  partes     hora, nivel, mensaje en mono; exportar como .txt por la hoja de compartir

Toast / Snackbar
  variantes  info, error; error siempre dice qué hacer ("Activa el Bluetooth")
```
