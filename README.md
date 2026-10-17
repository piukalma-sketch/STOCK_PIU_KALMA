# Stock de Pijamas — versión independiente (Vercel + Firebase)

Esta carpeta es la misma app que veníamos usando dentro de Claude, pero
adaptada para vivir como un sitio normal, fuera de Claude. Es un solo
archivo (`index.html`) con todo adentro (HTML, CSS y JS) — no hace falta
build ni `npm install`, Vercel lo sirve tal cual.

Lo único que cambió por dentro: antes el guardado de datos (stock,
catálogo, fotos) usaba una base de datos que te daba Claude. Ahora usa
**Firebase** (gratis para este tamaño de uso, plan "Spark" sin tarjeta
ni facturación), que vos controlás. Las fotos se guardan comprimidas
directo en Firestore (no se usa Firebase Storage), así que no hace
falta activar ningún plan de pago — ver la sección "Sobre las fotos"
más abajo.

## 1. Crear el proyecto de Firebase

1. Entrá a https://console.firebase.google.com y creá un proyecto nuevo
   (podés dejar Google Analytics desactivado, no hace falta).
2. Adentro del proyecto, andá a **Build → Firestore Database** → "Crear
   base de datos" → modo producción → elegí una región (cualquiera
   cercana a Argentina, ej. `southamerica-east1`).
3. Andá a **Build → Authentication** → "Comenzar" → pestaña **Sign-in
   method** → activá el proveedor **Anónimo**. (Esto es lo que hace que
   cualquiera que abra la página pueda leer y escribir el stock, sin
   tener que crearse una cuenta — como ya funcionaba antes.)

No hace falta activar Storage ni ningún plan de facturación (Blaze):
esta versión guarda todo, incluidas las fotos, dentro de Firestore, que
es gratis en el plan por defecto ("Spark") para este tamaño de uso.

## 2. Pegar la configuración en `index.html`

1. En la consola de Firebase: ícono de engranaje (arriba a la izquierda)
   → **Configuración del proyecto**.
2. Abajo, en "Tus apps", hacé clic en **</>** (Web) para agregar una app
   web. Ponele un nombre (ej. "stock-pijamas") y seguí los pasos — NO
   hace falta que marques "Firebase Hosting".
3. Te va a mostrar un objeto `firebaseConfig` con `apiKey`,
   `authDomain`, `projectId`, etc. Copialo.
4. Abrí `index.html`, buscá (cerca de la línea 20) este bloque:

   ```js
   const firebaseConfig = {
     apiKey: "REEMPLAZAR",
     authDomain: "REEMPLAZAR.firebaseapp.com",
     projectId: "REEMPLAZAR",
     storageBucket: "REEMPLAZAR.appspot.com",
     messagingSenderId: "REEMPLAZAR",
     appId: "REEMPLAZAR"
   };
   ```

   y reemplazalo por el que copiaste de Firebase.

Estos valores no son secretos (es normal que estén visibles en el
código del sitio); lo que de verdad protege los datos son las reglas
del paso siguiente.

## 3. Subir las reglas de seguridad

Con la [Firebase CLI](https://firebase.google.com/docs/cli) instalada
(`npm install -g firebase-tools`), desde esta misma carpeta:

```bash
firebase login
firebase init firestore   # elegí "usar un proyecto existente" y el que creaste
# cuando pregunte por el archivo de reglas, apuntá a firestore.rules (ya está acá)
firebase deploy --only firestore:rules
```

Si no querés instalar nada, también podés copiar el contenido de
`firestore.rules` a mano en la consola de Firebase (Firestore Database
→ Reglas).

(El archivo `storage.rules` que puede haber quedado en esta carpeta ya
no se usa — esta versión no depende de Firebase Storage. Lo podés
borrar sin problema.)

## 4. Deployar en Vercel

Con la [Vercel CLI](https://vercel.com/docs/cli) (`npm install -g vercel`):

```bash
vercel          # primer deploy (te va a preguntar un par de cosas, respondé lo que sugiere por defecto)
vercel --prod   # deploy a producción
```

O, más simple todavía si no querés usar la terminal: subí esta carpeta
a un repositorio de GitHub y en https://vercel.com → "Add New... →
Project" → importá ese repo. Vercel detecta que es un sitio estático y
no hace falta tocar nada en la configuración de build.

Cuando termine, te da una URL del tipo `https://tu-proyecto.vercel.app`
— esa es tu app, ya funcionando para quien la abra, en cualquier
dispositivo.

## Qué cambia respecto a la versión de Claude

- **El stock, catálogo y últimas cargas** se guardan en un único
  documento de Firestore (`appdata/shared`) y se sincronizan en tiempo
  real entre todos los que tengan la página abierta — igual que antes.
- **Las fotos** se guardan comprimidas (ya achicadas y convertidas a
  JPEG liviano) directo como texto dentro de un documento de Firestore,
  en una subcolección (`appdata/shared/photos/{modelo}_{color}`) — el
  comportamiento de uso es el mismo que antes (varias fotos por modelo,
  reemplazar una, poner como portada), pero ver la sección "Sobre las
  fotos" más abajo para las limitaciones de este enfoque.
- **Las descargas de Excel** (planilla modelo, stock actual) ahora se
  descargan directo con el mecanismo normal del navegador — ya no
  dependen de ninguna capacidad especial, así que deberían andar igual
  o mejor que antes.
- **El escáner de código de barras por cámara** debería funcionar igual
  o mejor, porque ya no corre dentro de un iframe con restricciones: en
  un sitio propio con HTTPS (que Vercel da gratis) el navegador pide el
  permiso de cámara de forma normal.
- **No hay "una cuenta por persona"**: todos los que entran a la URL
  comparten el mismo login anónimo y el mismo stock, tal como estaba
  configurado hasta ahora. Si en algún momento querés separar accesos
  (por ejemplo, un login con contraseña para el local y otro para vos),
  avisame y lo armamos — Firebase Authentication lo soporta sin
  rehacer nada de lo demás.

## Sobre las fotos (sin Firebase Storage)

Para no depender de Firebase Storage (que pide activar el plan de pago
Blaze), esta versión comprime cada foto en el navegador antes de
guardarla — la achica a un máximo de 700px de lado y la recomprime como
JPEG — y la guarda como texto directo dentro del documento de Firestore
de ese modelo+color.

Esto tiene algunas consecuencias a tener en cuenta:

- **Menor calidad/tamaño que una foto original**: suficiente para verse
  bien en el catálogo y para mandar por Instagram, pero no para hacer
  zoom extremo ni imprimir en gran tamaño.
- **Límite de fotos por modelo+color**: Firestore permite hasta ~1 MiB
  por documento. Cada foto comprimida ocupa en general entre 50 y 180
  KB, así que entran cómodamente varias fotos (gráficamente, unas 5 a
  8) por combinación de modelo+color antes de acercarse al límite. Si
  en algún momento se llega al límite, al subir una foto más va a
  aparecer un error de guardado — en ese caso, hay que borrar alguna
  foto vieja de esa combinación antes de subir una nueva.
- Si en el futuro hace falta más calidad o más fotos por modelo (por
  ejemplo para una tienda con catálogo muy visual), se puede volver a
  sumar Firebase Storage en cualquier momento sin tener que rehacer el
  resto de la app — avisame cuando llegue ese momento y lo armamos.

## Próximo paso pendiente: catálogo público para compartir

Quedó pendiente una página aparte, pública, con el catálogo (foto +
precio + galería de fotos por modelo) para mandar el link por
Instagram sin exponer el stock interno. Una vez que esta base esté
deployada y andando, armamos esa página como una segunda ruta del mismo
sitio (por ejemplo `/catalogo`), leyendo los mismos datos de Firebase.

## Cambios de guardado y velocidad (octubre 2026)

- **El guardado ya no falla en silencio.** Si algo impide guardar (permisos, sin
  conexión, demasiados datos), aparece un cartel rojo fijo arriba con el motivo.
  La carga masiva ya no dice "N filas cargadas" si no se pudo guardar: en ese caso
  no carga nada y avisa.
- **Más stock en el mismo documento.** El stock ahora se guarda en un formato
  compacto (campo `stock` de `appdata/shared`): cada texto se guarda una sola vez
  y cada fila solo referencia posiciones. Antes el documento de Firestore (límite
  de 1 MiB) se llenaba con unas 4.800 combinaciones; ahora entran más de 20.000.
  Se sigue leyendo el formato viejo (campo `items`), así que la migración ocurre
  sola en el primer guardado. **No hace falta cambiar las reglas de Firestore.**
- **No se pisa el stock si falla la lectura.** Si al abrir no se pudo leer lo
  guardado, la app no escribe nada hasta lograr leerlo.
- **Abre más rápido.** Las fotos del catálogo se leen recién al abrir la pestaña
  Catálogo, de a 24 tarjetas ("Ver más" carga las siguientes); solo se dibuja la
  pestaña que se está viendo; la tabla de stock muestra de a 200 líneas y arma el
  detalle de cada grupo recién al abrirlo; `xlsx` y el lector de códigos se
  descargan solo cuando se usan.
- **Respaldo.** En Inventario → Descargar Excel hay un botón "Descargar respaldo
  (.json)" con todo el stock y el catálogo (sin fotos).

> Importante al actualizar: el formato nuevo no lo entiende la versión vieja de la
> página. Después de publicar, cerrá y volvé a abrir la app en todos los
> dispositivos (una pestaña vieja que siga abierta vería el stock vacío).
