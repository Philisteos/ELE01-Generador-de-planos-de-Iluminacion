# Generador de planos de ILUMINACIÓN — Revit 2027

Pipeline de graphs de Dynamo (4.0, CPython3, **sin paquetes externos**) para documentar
sectores de una planta de iluminación: planta y cuatro elevaciones por sector, distribución
en láminas, cotas y tags de riel RUC, y la tabla de tipos de riel por lámina.

> **El sector ES la caja.** Un sector es una **vista 3D con el Section Box activo**, y nada
> más: ni assemblies, ni grupos, ni ejes marcados. Lo que delimita el sector ya es
> geométrico —crop en X/Y, View Range o far clip en profundidad—, así que ningún paso
> vuelve a describir el recorte elemento por elemento.

> **Origen**: adaptación de `T003-Generador de planos - Aceros`, que a su vez viene de
> `T001-Generador de planos` (fundaciones). Lo de acero no se extendió: se reemplazó. Si
> buscás el comportamiento de assemblies, plantas N.I.P.B./P.T./T.A., grating o rótulos de
> perfil, está en T003.

## Requisitos

- Revit 2027 (los `.dyn` están guardados en formato Dynamo 4.0).
- **Vistas 3D con Section Box** que delimiten cada sector, nombradas de forma que el filtro
  las distinga (por defecto contienen `SECTOR`).
- Viñetas (title blocks) cargadas en el proyecto.
- Las leyendas que estampa 04 (`ILU-SIMBOLOGIA ILUMINACION`, `ILU-NOMENCLATURA`) creadas en
  el modelo. Si falta alguna, 04 lo avisa en el log y coloca las demás.
- Ningún paquete de Dynamo: todo es out-of-the-box + Python embebido.

**Primera vez en una máquina nueva**: abrir cada `.dyn` en Dynamo (no en Player), correr y
guardar. Eso registra los inputs para Dynamo Player. Después se opera todo desde Player.

> ⚠️ **Cuando un paso suma o renombra inputs hay que volver a abrirlo en Dynamo**, o Player
> sigue mostrando la lista vieja.

### Estado al 2026-08-18

Modelo de referencia: `1820-400-E-MOD-001_detached`, escala de trabajo **1:75**.

| | |
|---|---|
| Rieles RUC en el modelo | **217** (+ 77 mordazas que quedan fuera por el prefijo de familia) |
| Familias de riel | `RIEL RUC GALVANIZADO SUELO`, `... MURO`, `... MURO VERT.` |
| Tipos distintos por largo | **11** (R-01 … R-11) |
| Sectores documentados | SECTOR A (láminas A1 y A2) |

## Orden de ejecución

| # | Graph | Estado | Qué hace |
|---|-------|--------|----------|
| 0 | `00_Vistas de sector.dyn` | ✅ | Por cada vista 3D con Section Box: 1 planta + 4 elevaciones (A/B/C/D) |
| 1 | `01_Calcular y crear laminas.dyn` | ✅ | Calcula cuántas láminas hacen falta y las crea |
| 2 | `02_Colocar vistas en laminas.dyn` | ✅ | Coloca las vistas en flujo dentro de las láminas |
| 3 | `03_Taggear plantas y cortes.dyn` | ✅ | Cadenas de cotas entre rieles de suelo + tags de tipo (`R-01`, `R-02`…) sobre todos los rieles |
| 4 | `04_TABLAS y Listados.dyn` | ✅ | La TABLA N°1 de tipos de riel, las leyendas pegadas a la esquina y el bloque de notas |

**Flujo**: 00 → 01 → 02 → 03 → 04. El 04 va al final porque necesita las láminas ya armadas
y las vistas ya colocadas para saber qué se ve en cada hoja.

---

## Contratos entre pasos

Tres acuerdos sostienen el pipeline. Romper cualquiera de ellos rompe los pasos siguientes
sin que ningún error lo diga.

### 1. Nombres de vista

00 nombra sus vistas `"<sector> - PLANTA"` y `"<sector> - ELEV A|B|C|D"`, y **01, 02, 03 y
04 reconocen sus vistas por ese nombre y nada más**. El modelo tiene cientos de vistas de
otras disciplinas y ninguna se llama así.

No hay parámetro-sello además del nombre: sería describir dos veces la misma pertenencia,
con el riesgo de que las dos descripciones se desincronicen.

### 2. Una lámina, un sector

01 fija la regla: **cada lámina lleva vistas de un solo sector**. Las vistas de un sector
fluyen por tantas láminas como haga falta, pero el sector siguiente siempre empieza en una
lámina nueva. Gracias a eso, "el sector de esta lámina" nunca es ambiguo en 04.

### 3. El código `R-xx` sale del largo

El código de tipo de riel es una consecuencia del largo, no un dato nuevo: se juntan los
largos de **todos** los rieles RUC del modelo, se ordenan de menor a mayor y se reparten
`R-01`, `R-02`, `R-03`… La numeración es **global al modelo**, así que `R-02` significa lo
mismo en todos los planos del paquete.

El largo sale del parámetro de **instancia** `Longitud`, redondeado al milímetro. Un riel
sin ese parámetro inicializado no se etiqueta y no entra en ninguna tabla.

> ⚠️ **El nombre del tipo miente y no se usa nunca.** Hay un tipo llamado `150 mm` cuya
> `Longitud` es **600 mm exactos**, y un `100mm` de MURO VERT. que mide **108,05**. Además
> los nombres no son consistentes entre familias: para el mismo largo conviven `300`,
> `300mm` y `300 mm`, y hay un `200  mm` con dos espacios. Adivinar el largo desde ahí sería
> fabricar un dato.

Tabla vigente (medida el 2026-08-18):

| Código | R-01 | R-02 | R-03 | R-04 | R-05 | R-06 | R-07 | R-08 | R-09 | R-10 | R-11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Largo (mm) | 100 | **108** | 200 | 250 | 300 | 400 | 500 | **600** | 700 | 900 | 1000 |
| Unidades | 60 | 3 | 28 | 8 | — | 50 | — | 1 | 4 | 5 | 7 |

Los dos resaltados son datos sucios del modelo, no del pipeline: `R-02` es el `100mm` que
mide 108,05 y `R-08` es el `150 mm` que mide 600. Se redondea al **milímetro entero** —para
que 300,000 y 300,054 sean el mismo riel— pero **no** a la medida comercial: si un riel mide
108, la tabla dice 108 y el problema se ve.

---

## 00 — Vistas de sector

Una planta y cuatro elevaciones (A Norte, B Este, C Sur, D Oeste) por cada vista 3D con
Section Box que pase el filtro. Todas comparten escala (1:75 en este proyecto) y se les
aplica view template (`1P_ILU` para plantas, `1C_ILU` para elevaciones).

> ⚠️ **`Procesar TODOS` arranca en False a propósito.** Tener el Section Box encendido no
> convierte a una vista 3D en un sector: el modelo tiene varias vistas 3D de trabajo con la
> caja activa (`DETALLE 1`, `DET 2 LAM-004`, `detalle nuevo fe`). El 2026-08-11, con True,
> el graph dejó 40 vistas y 01 pidió 9 láminas. **El filtro por nombre es lo que decide qué
> es un sector.**

Qué desapareció respecto del 00 de acero, y por qué:

- **El Far Clip Offset ya no es input.** En una elevación de eje había que elegir a mano
  hasta dónde ver porque un eje es una línea infinita. La caja sí tiene profundidad propia
  —es la definición del sector—, así que el far clip sale de ella. Un input solo podría
  contradecirla.
- **Los offsets de corte por tipo de planta** se fueron con los niveles por assembly. Acá
  hay un solo nivel (`nivel 1 luminarias`) y toda la geometría vertical se deriva de la caja.
- **El aislamiento por miembros** (`IsolateElementsTemporary` + conversión a permanente) se
  fue entero, por el motivo del encabezado: la caja no es una aproximación al sector, es el
  sector.

## 01 / 02 — Láminas

01 calcula cuántas láminas hacen falta simulando **exactamente** el mismo empaquetado que
hace 02, y las crea. 02 coloca planta y elevaciones fluyendo de izquierda a derecha y de
arriba hacia abajo.

> ⚠️ **`empaquetar()` tiene que ser idéntica en los dos graphs.** Si las dos versiones se
> separan, 01 calcula una cantidad de láminas que no coincide con lo que 02 termina
> colocando, y el error aparece recién al final, con láminas de más o vistas sin colocar. El
> input de margen también debe coincidir (hoy 5 mm en los dos).

Qué desapareció respecto del pipeline de acero:

- **La reserva del lado derecho.** En acero, la tabla de cantidades se estampaba en la
  primera lámina de cada assembly y esa lámina perdía 150 mm de ancho. Acá las vistas usan
  el **lienzo completo** en todas las láminas. *(Ver la nota de 04: cuando volvió a aparecer
  una tabla, se resolvió buscando hueco en vez de reabrir esta decisión.)*
- **La regla de matcheo de leyendas.** No es que las leyendas no sirvan —el modelo tiene
  `ILU-SIMBOLOGIA ILUMINACION`, `ILU-NOMENCLATURA`, `ILU-TABLAS DE RIELES`—, es que la regla
  de acero (nombre de leyenda = nombre de tipo de un miembro del assembly) no sobrevive sin
  assemblies. Se eliminó en vez de dejarse a medias. *(Las leyendas volvieron en 04, pero por
  lista explícita y pegadas a una esquina: no hay nada que adivinar.)*

## 03 — Taggear plantas y cortes

Dos anotaciones sobre las plantas de sector. Las elevaciones no se tocan.

### A. Cadenas de cotas exteriores

Cuatro cadenas por planta —arriba, abajo, izquierda y derecha—, por fuera del marco, entre
los rieles RUC **de suelo** (los de muro se excluyen: en planta son un cuadradito de 42×42
sin dirección útil, y mezclarlos daría una cota que no significa nada).

Cada riel se acota perpendicular a su propio eje, contra los planos de referencia
`Center (Left/Right)` y `Center (Front/Back)` de la familia. **No** se acota contra las
caras del sólido: Revit descarta esas referencias al comitear y la cota se crea sin error y
desaparece.

> ⚠️ **Dos trampas que hacen que una cota se cree y no se vea**: el *annotation crop* (si
> está activo recorta todo lo que quede fuera; el graph agranda sus offsets) y el
> **paralelismo entre referencias** (Revit exige que todas las referencias de una cadena
> sean paralelas entre sí con un criterio mucho más estricto que `AngularTolerance`; un solo
> riel torcido hace fallar la cadena completa, así que se descartan los desviados y se
> cuentan en el log).

> ⚠️ **Una cota es de este paso si CUALQUIERA de sus referencias toca un riel**, no solo la
> primera. Mirar solo la primera dejaba pasar cualquier cota cuya primera referencia fuera
> otra cosa —una línea del CAD importado, un eje, un muro—: dos cotas viejas de SECTOR A
> sobrevivieron a dos *rehacer* seguidos y el graph dibujaba las suyas al lado. El precio es
> que un *rehacer* también se lleva una cota puesta a mano que toque un riel, y es el precio
> correcto: estas vistas las mantiene el pipeline y *rehacer* es explícito.

### B. Tags de tipo de riel

Un texto `R-01`, `R-02`… con una flecha a cada riel de ese tipo que tenga cerca, igual que
la nota del plano tipo. Entran **todos** los rieles RUC de la planta, de suelo y de muro: el
filtro "y además contiene" es exclusivo de las cotas.

**Es un `TextNote` con leaders, no un `IndependentTag`.** Se probó primero con un tag de
categoría, que es lo correcto en abstracto porque es asociativo, y no sirvió por dos motivos
que no se arreglan desde el graph: la familia `Multi Categoría TAG` dibuja un **hexágono que
no muestra el parámetro** donde se escribe el código (los 195 tags de la prueba salieron
vacíos, y ninguna API dice qué parámetro dibuja el label de una anotación), y a 1:75 ese
hexágono es más grande que el riel que anota. El plano de referencia tampoco usa un tag:
usa una nota de texto con flechas.

Lo que se pierde es la asociatividad: si alguien cambia un riel de 400 a 300, el texto
miente hasta que se vuelva a correr el paso con *rehacer*.

**Agrupación y colocación** — el equilibrio costó cuatro iteraciones y los dos extremos
están medidos:

| Intento | Resultado |
|---|---|
| Radio 25 mm de papel, encadenamiento simple | **195 textos**, uno por riel: no se agrupó ni un par |
| Radio 60 mm, encadenamiento simple | **18 textos** con abanicos de 40 flechas cruzando la planta de esquina a esquina |
| Sin encadenar (grupo contra semilla) + tope de 4 | ~50 textos, ninguna flecha más larga que el radio |

El encadenamiento simple (A con B, B con C) no acota nada cuando los rieles de un mismo
largo vienen en corridas continuas cada metro y medio: el grupo se propaga por todo el
edificio. La versión vigente agrupa por **línea física contigua y código**.

Para la posición del texto el criterio es, en este orden: (1) que no caiga en la franja de
las cotas —las cadenas van por fuera del marco, pero sus líneas de referencia bajan hasta
cada riel y llenan de líneas la banda de adentro del borde—, (2) que no se pise con otro
texto, y (3) que la **flecha más larga sea lo más corta posible**.

> El criterio de flecha corta trae un regalo: para una corrida de rieles, la posición
> **perpendicular** a la corrida gana sola. Puesto al costado, el texto tendría que alcanzar
> el riel del otro extremo; puesto en el medio de un lado, le alcanza con la mitad. Es
> exactamente donde el plano tipo pone la nota, y no hubo que escribir la regla.

**Plano de referencia**: la vista `DISPOSICION GENERAL ALUMBRADO` del propio modelo. Ahí
cada `R-0x` está pegado a su riel con una flecha de 10–20 mm de papel que no cruza el
dibujo. Es la unidad de medida del asunto. Ojo que esa vista **no etiqueta todos los
rieles** y la nuestra sí tiene que hacerlo: siempre vamos a tener más etiquetas.

## 04 — TABLAS y Listados

Tres cosas por lámina de sector: la TABLA N°1 —tipos de riel RUC—, las leyendas de la hoja
y el bloque de notas generales.

Se llama "TABLAS y Listados" en plural porque va a juntar más —circuitos, luminarias,
notas—, pero cada una se agrega cuando esté definida. Reemplaza por completo a
`04_Tabla de materiales`, la tabla de acero heredada de T003: no se adaptó, se tiró. Está en
el historial de git (`94c3e7a:"04_Tabla de materiales.dyn"`).

### La TABLA N°1

Una por lámina de sector, con las filas de **los tipos que se ven en esa lámina**. Un sector
repartido en dos láminas da dos tablas distintas: una tabla que liste tipos que no están
dibujados en la hoja que uno tiene delante obliga a buscarlos en otra hoja, y eso es
información equivocada, no información de más.

Es un **`ViewSchedule` nativo por lámina** (`TBL_RIELES_A1`), así que queda vivo.

> ⚠️ **`Longitud` no se puede poner en un schedule.** Es un parámetro **de familia** de las
> familias de riel, y Revit no ofrece los parámetros de familia como campo de schedule: no
> aparecen en la lista y no hay API que los agregue. Se comprueba en un minuto: una
> **MORDAZA**, que es otra familia de la misma categoría, **no tiene** `Longitud`, mientras
> que `En plano` sí aparece en la mordaza y hasta en las vistas — que es la firma de un
> parámetro de **proyecto**.
>
> Por eso el largo se lee de `Longitud` y se **copia** a un parámetro numérico de proyecto
> (`LARGO_TBL`) que el graph crea si no existe, con GUID derivado del nombre para que sea el
> mismo en todos los modelos de la oficina. La copia es dato derivado y se desactualiza: si
> cambia un largo, la columna miente hasta que se vuelva a correr el paso.

> ⚠️ **El filtro por lámina va por `En plano`**, con la lista de hojas delimitada
> (`;A1;A2;`) y filtro por `;A1;`. Sin los delimitadores, la lámina A1 se llevaría también
> los rieles de A11 y A12.

> ⚠️ **`ITEM` es el número del código, no un contador de fila.** Un schedule no tiene campo
> "número de fila", así que ITEM tiene que ser un parámetro escrito en el riel — y un riel
> que aparece en dos láminas no puede tener dos ITEM distintos a la vez. La lámina que usa
> R-01, R-03 y R-04 imprime **1, 3 y 4**.

**Dónde se para la tabla**: en el hueco libre más cercano a la esquina elegida. Hay que
buscarlo porque 01 y 02 reparten las vistas por el lienzo completo (ver arriba). Ocupado es
los viewports con su título, la franja inferior de la viñeta y el cajetín; esos dos últimos
no se pueden deducir de la geometría de la viñeta —es una sola familia que ocupa toda la
hoja— así que son inputs, y son constantes del proyecto: se cargan una vez y no se tocan
más (hoy 55 mm de franja y 290×105 mm de cajetín).

El tamaño de la tabla no se estima: se coloca, se le mide el bounding box real y recién ahí
se la mueve al hueco. Estimar el alto de un schedule por la cantidad de filas y el tipo de
letra es adivinar, y errarle por dos milímetros la deja pisando una vista.

> **Qué mirar en el primer run de una máquina nueva**: que la tabla salga con **una fila por
> tipo y no una por riel**. Depende de que Revit colapse filas con *Itemize* apagado
> ignorando el campo oculto del filtro. El graph se autochequea y avisa en el log si las
> filas no cuadran con los tipos de la lámina. Si aparece ese aviso, la salida es dibujar la
> tabla con líneas y textos en vez de filtrar por parámetro.

### Las leyendas de la lámina

Además de la tabla, cada lámina de sector lleva las leyendas que se le pidan —hoy
`ILU-SIMBOLOGIA ILUMINACION` y `ILU-NOMENCLATURA`— **apiladas y pegadas contra la esquina
superior derecha**, la primera tocando el borde y la siguiente tocando a la primera.

Van en **todas** las láminas del sector por el mismo motivo por el que la tabla no es la
misma en todas: la hoja que uno tiene delante tiene que poder leerse sola, y una simbología
que está en la otra hoja no sirve de nada.

Son leyendas de verdad —vistas de tipo *Legend*, las únicas de Revit que se pueden colocar
en varias láminas—, buscadas **por nombre exacto**. Si una no está en el modelo, se avisa en
el log y las demás se colocan igual.

**El tipo de viewport es un input** (`Sin titulo` en esta plantilla), porque una leyenda con
el rótulo `ILU-SIMBOLOGIA ILUMINACION` escrito debajo no es lo que se dibuja. Deducirlo no
alcanza: el modelo tiene varios tipos con *Show Title* apagado —sobras de otras disciplinas—
así que "cualquiera sin rótulo" da un tipo distinto en cada modelo. Si el campo se deja
vacío se cae justamente a eso y el log dice cuál eligió; si el nombre no existe, el log
**lista todos los tipos de viewport del modelo** para copiar el correcto.

> ⚠️ **Los tipos de viewport no tienen categoría.** Su parámetro `Category` vale `-1`, así
> que `OfCategory(OST_Viewports).WhereElementIsElementType()` devuelve **cero tipos y ningún
> error**: el síntoma es que las leyendas salen con el tipo por defecto —el que tenga título—
> y el log dice que no encontró ninguno sin rótulo. Se buscan por `ElementType` con
> `FamilyName == "Viewport"`, más los tipos que ya usan los viewports del documento por si
> ese `FamilyName` viene traducido.

> **Contra qué esquina se pega**: contra el mismo rectángulo útil que usan 01 y 02 para
> repartir las vistas —bounding box de la viñeta menos el margen—, **corrido 3,2 mm a la
> izquierda y 1,9 mm hacia abajo**. El bounding box de la viñeta y el marco *dibujado* no
> coinciden, y esa es la diferencia medida sobre la lámina: es una constante de la viñeta
> (`AJUSTE_LEYENDAS_X` / `AJUSTE_LEYENDAS_Y` en el graph), no del sector. Si algún día cambia
> la viñeta hay que volver a medirla.

La pila **no busca hueco**: se pega a la esquina y punto. El alto de cada leyenda se **mide**
después de colocarla, igual que el de la tabla. Si queda montada sobre una vista se coloca
igual y el log lo dice — 01 y 02 reparten las vistas por el lienzo completo, así que el
choque es posible.

Las leyendas se estampan **antes** que la tabla a propósito: una vez colocadas son viewports
de la lámina, así que el barrido que busca hueco para la tabla las cuenta como ocupado sin
que haya que decírselo.

> Si las leyendas ya están en la lámina y no se corre con *rehacer*, no se tocan: puede
> haber alguien que las movió a mano y este paso no tiene por qué saber más que esa persona.

### El bloque de notas

Las **notas generales 1 a 13** —dos `TextNote` del tipo `2mm wf 0.7 RomanD`: la primera hasta
la 7, la segunda desde `CONTINUACION DE NOTAS:`— se **escriben desde cero** en cada lámina de
sector, con `TextNote.Create`. El texto, el tipo, el ancho y la posición son constantes del
graph (`NOTAS`): no se copian de ninguna otra lámina y el paso no necesita que exista una
lámina modelo para funcionar.

La posición va medida desde el **vértice inferior izquierdo de la viñeta**, no en coordenadas
de lámina: es lo único común entre láminas distintas, porque las coordenadas de lámina
dependen de dónde haya quedado insertada la viñeta.

> ⚠️ **De dónde salen esos cuatro números: del diagnóstico, no de transcribir el plano.**
> Revit no deja leer la posición de una nota desde fuera del modelo —no es un parámetro, es
> la propiedad `Coord`— y el texto copiado a ojo desde el plano pierde justamente las
> tabulaciones con que están alineados los números de nota. Las dos cosas hay que
> preguntárselas al documento.
>
> Para eso está el input `19.`: se le pasa el número de una lámina que ya tenga las notas
> puestas a mano y el log escupe, por cada una, su posición, ancho, tipo, alineación y texto
> con los tabs escapados, con el formato exacto que espera `NOTAS`. Se pegan esas líneas en
> la constante y el input vuelve a quedar vacío — es una herramienta de una sola vez, no
> parte del funcionamiento normal.
>
> **Con `NOTAS` ya cargado, el diagnóstico además verifica**: empareja por texto lo que hay
> en la lámina contra lo que dice la constante y avisa si algo difiere. Un tab de más, o un
> retorno de carro convertido en salto de línea, no se ven en el plano pero son texto
> distinto — esta es la única forma de saberlo.

Mientras `NOTAS` esté vacío el paso no escribe ninguna nota y lo dice en el log. Es a
propósito: inventar una posición aproximada sería peor que no poner nada, porque una nota
corrida se ve igual de bien que una nota bien puesta hasta que alguien mide.

**Valores vigentes** (medidos sobre `A1` el 2026-08-18, desde el vértice inferior izquierdo
de la viñeta):

| Nota | x (mm) | y (mm) | Ancho (mm) | Tipo | Alineación |
|---|---|---|---|---|---|
| 1 a 7 | 372,239 | 50,812 | 174,441 | `2mm wf 0.7 RomanD` | Left / Top |
| 8 a 13 | 372,239 | 83,312 | 174,441 | `2mm wf 0.7 RomanD` | Left / Top |

> El bloque de continuación (8 a 13) va **arriba** del primero —su `y` es mayor y las dos se
> alinean por el borde superior—. Así están en el plano: no es un error de transcripción y no
> hay que "arreglarlo".

Las líneas se separan con **`\r`**, que es lo que Revit guarda internamente en un
`TextNote`. Escribirlas con `\n` daría un texto distinto.

> ⚠️ Con *rehacer*, las notas que ya estaban en la lámina **se borran** y se vuelven a
> escribir. Eso se lleva también una nota puesta a mano — el mismo trato que hacen las cotas
> de 03, por el mismo motivo: estas láminas las mantiene el pipeline y *rehacer* es explícito.

Lo que dicen hoy, para referencia (la fuente es la constante `NOTAS`, no esta tabla):

| # | Nota |
|---|---|
| 1 | DIMENSIONES Y ELEVACIONES EN mm. (S.I.C.) |
| 2 | COTAS PREVALECEN SOBRE EL DIBUJO. |
| 3 | FUNDACIONES, SOPORTES, POR ESPECIALIDAD OO.CC. |
| 4 | PARA DETALLES DE CIRCUITOS ELECTRICOS VER DOCUMENTO N° 1820-E-LIS-002. |
| 5 | LAS LUMINARIAS … DEBEN INSTALARSE SEGUN ALTURA INDICADA EN LA TABLA N°2 Y N°3, DESDE EL N.P.T. EN PLATAFORMA DE TRANSITO PEATONAL (GRATING). |
| 6 | TODAS LAS LUMINARIAS EXTERIORES DEBERAN DAR CUMPLIMIENTO AL DS1/2022 DEL MMA (contaminación lumínica). |
| 7 | LAS CANALIZACIONES A LA VISTA DEBERAN SER EN PVC Sch.80 RESISTENTE A LOS RAYOS UV, SEGUN TABLA N°4.23 DEL PLIEGO RIC N°04. |
| 8 | PARA VER CANALIZACIONES DE ELECTRICIDAD E INSTRUMENTACION IR A PLANOS DEL RECUADRO DE REFERENCIAS. |
| **9** | **TODOS LOS RIELES SON DE DIMENSION 42x42x2,5mm, DONDE LA TABLA N°1 INDICA EL LARGO DE CADA UNO DE ELLOS.** |
| 10 | EQUIPOS DE EMERGENCIA … PLIEGO RIC N°8 PUNTO N°10 Y ANEXO 8.1, AUTONOMIA DE AL MENOS DOS HORAS. |
| 11 | VER MEMORIA DE CALCULO DE ALUMBRADO EN DOCUMENTO N° 1820-E-MC-002. |
| 12 | VER DETALLES DE LUMINARIA EN HOJA DE DATOS, DOCUMENTO N° 1820-E-HDD-001. |
| 13 | VER POTENCIA DE LUMINARIAS EN TABLA DE SIMBOLOGIA. |

> La **nota 9** es la que le da sentido a la TABLA N°1 y por eso el título de la tabla es
> `TABLA N°1 (NOTA 9)`: si alguna vez se renumeran las notas, hay que cambiar los dos.

---

## Deudas conocidas

- **La regla del código `R-xx` vive escrita en dos graphs.** 03 la usa para el texto del
  plano y 04 para las filas de la tabla, cada uno con su implementación. Si se cambia en uno
  solo, el síntoma es un plano que dice `R-05` con una tabla que dice `R-06`. La forma de
  saldarla: que 03 vuelva a escribir el parámetro y que 04 solo lea.
- **03 ya no escribe `TEXTO PARA TAG`.** Se perdió en una reescritura del código; hoy no
  molesta porque 04 lo escribe por su cuenta, pero es la mitad de la deuda anterior.
- **03 se quedó sin su header de documentación** (1228 → 570 líneas). El resto del pipeline
  lo lleva; este quedó como la excepción. Buena parte de lo que decía está recogido acá.
- **Dos rieles con datos sucios**: el `100mm` de MURO VERT. que mide 108,05 y el `150 mm` de
  MURO que mide 600 (id 7884652). Salen en la tabla como `R-02` y `R-08`. Hay que decidir si
  se limpia el modelo o se convive con ellos.
- **Sin ingerir en CEREBRO BIOS.** La wiki tiene T001 ingerido y T002/T003 pendientes; ELE01
  no figura en ningún archivo. Este README es la fuente para cuando se haga el ingest.
