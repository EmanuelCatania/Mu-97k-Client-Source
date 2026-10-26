# Inventario x64 — Fase A

Fecha: 2026-10-26. Base: `origin/fase/4`, commit
`da5f18dfb2aa809df8e22bd5920c75949d00b2bd` (PR #241).
Rama de revisión: `feat/fase4-x64`. Este PR modifica únicamente este documento;
no modifica tipos, layouts, protocolo, assets ni build. La Fase B espera revisión.

## Alcance y cómo interpretar las cantidades

Se inspeccionaron los archivos versionados de C/C++ debajo de `src/`.
El anexo incluye el conteo reproducible por patrón, archivo y línea en la base.
Cada sitio es una coincidencia textual, no necesariamente un defecto ni una
instrucción ejecutada. Se quitan comentarios y literales; se incluyen cabeceras,
macros, ramas de preprocesador desactivadas y código generado de GL. Los conteos
se superponen y **no se suman**. No incluyen backups, librerías externas, assets
ni documentación. Las referencias de línea apuntan al commit base.

Las búsquedas no resuelven tipos ni alias interprocedurales: `(int)valor` puede ser
una conversión numérica válida; un parámetro `int` puede ocultar un puntero sin
cast local. Por ello las cifras son candidatos para auditar, no un total cerrado
de punteros truncados. Los ejemplos revisados manualmente figuran abajo. No hay
una certificación de exhaustividad semántica ni de equivalencia x86/x64.

IDA no estuvo accesible: el puente `ida76` devolvió conexión rechazada
(`WinError 10061`) al consultar estado y `0x004FDC00`. Las direcciones IDA que
siguen son anclajes ya documentados en el código; no se presenta una nueva
verificación del binario. No se hicieron arreglos inferidos.

## Resumen de clasificación

| Categoría | Clasificación | Trabajo previo a habilitar x64 |
|---|---|---|
| 1. Punteros en enteros y aritmética DAT_ | Mecánico para punteros confirmados; requiere decisión cuando hay bits de float o índices | Tipar origen, parámetro, retorno, global y todos los consumidores; `uintptr_t` al final de una cadena truncada no recupera bits |
| 2. Layouts con punteros y offsets | Requiere rediseñar el layout | Separar datos persistentes de objetos nativos; eliminar strides crudos antes de ensanchar punteros |
| 3. Disco y red | Mecánico para contratos ya comprobados; requiere decisión en formatos no tipados | Campos de ancho fijo, tamaños/offsets, lectores con muestras; conservar bytes y cifrado |
| 4. Globales y slots de uso mixto | Mecánico para bases inequívocas; requiere decisión por Type/SubType en slots mixtos | No ensanchar todos los DWORD: muchos son bits de float, colores, IDs o handles gráficos |
| 5. long, wchar_t, ABI, x87 | Requiere decisión; sustituciones mecánicas una vez fijado el contrato | Diferenciar Win64 LLP64 de Linux LP64 y texto UTF-16 de wchar_t nativo |
| 6. Punteros transportados por float | Mecánico sólo una vez confirmado el ABI real; otros candidatos requieren decisión | Reemplazar firma y llamadores juntos, conservando los float numéricos |

## 1. Punteros en enteros

Casos concretos: `src/Platform/ClientEntry.cpp:601-611` guarda malloc en
`GateAttribute` (DWORD), `DAT_07d29d20`, `DAT_07cf1ff8`, `DAT_07cf1ff0` (int),
y deriva `DAT_07d78068` de `DAT_055c9e44` (DWORD). En
`src/Model/Model_Items.cpp:27`, `(DWORD)(uintptr_t)DAT_06f42a58` sigue truncando
el resultado al asignar `DAT_05828d58`. `src/globals.h:1271-1272` también
reduce bases a unsigned int. Revisar los backups de esas bases, no sólo el global.

`src/Render/MoveJoint.cpp:192` usa `(int)(uintptr_t)o` al llamar a
`Joint_SegmentTick`. `src/Item/Item_InventoryGrid.cpp:365` castea a uintptr_t
después de sumar a un parámetro heredado: hay que corregir el contrato de entrada.
Los desplazamientos de tablas requieren `std::byte*`/puntero tipado y diferencias
`ptrdiff_t`; un índice de recurso o de red debe seguir con su ancho original.

`src/Render/Effect_Create.cpp:406,699` contiene conversiones entre float,
int y puntero que no autorizan un reemplazo global: pueden ser retornos
artefacto o slots por subtipo. El anexo 4b las separa del mero uso de uintptr_t.

## 2. Layouts y pools

Ya existen tipos con campos y aserciones. El problema no es sólo escribir nuevos
structs: varios consumidores reinterpretan sus punteros como int y mantienen
strides x86. `#pragma pack(1)` elimina padding, pero **no** vuelve de cuatro
bytes un puntero x64. No desactivar los static_assert para conseguir un build.

| Familia (9 entradas) | Contrato actual / punteros | Archivos principales y clasificación |
|---|---|---|
| ITEM de inventario | 0x44; sin punteros; sizeof y offsetof ya comprobados | `src/structs.h:220`, `src/Item/Item_InventoryGrid.cpp`, `Item_ClickHandler.cpp`, `Item_Inventory.cpp`; mecánico: extender uso de AsItem, conservar stride |
| Números de daño | 100 slots × 0x70; accesos crudos a valor, posición, color, lifetime | `src/Render/Render_LegacyDamageNumbers.cpp`, `Effect_Tick.cpp`, `Render_LegacyTrails.cpp`, `src/Combat/Skills_WarriorLegacy.cpp`, `src/Game/RenderSnapshots.cpp`; requiere tipar/verificar relleno, buen primer piloto |
| OBJECT / CHARACTER | 0x1BC / 0x394; Owner +0xFC y BoneMatrices +0x114 (276 decimal); CHARACTER incorpora OBJECT | `src/Entity/EntityTypes.h`, `src/Entity/`, `src/Scene/`, `src/Render/Entity_*`, `src/Render/RenderLinkObject.cpp`; rediseño nativo + conversión de accesos |
| ITEM_t de suelo | 0x204; ITEM +4 y OBJECT +0x48, hereda sus dos punteros | `src/structs.h:286`, `src/Net/Recv/Recv_Viewport.cpp:804`, `src/Render/Render_WorldHelpers.cpp:190,688`; rediseño ligado a OBJECT; no confundirlo con ITEM 0x44 |
| Efectos y objetos del mapa | paso OBJECT 0x1BC; Owner +0xFC, huesos +0x114; listas/buckets también guardan enlaces | `src/Render/Effect_Create.cpp`, `MoveEffect.cpp`, `Effect_Tick.cpp`, `src/Scene/ObjectRegistry.h`, `Scene_ObjectUpdate.cpp`; rediseño, auditar enlace y validadores de rango |
| JOINT | 0x9D8; Target +0x40; posición objetivo +0x44; MultiUse +0x9D4 | `src/Render/JointTypes.h`, `Joint_Create.cpp`, `MoveJoint.cpp`; rediseño; casts `*(int*)&Target` siguen leyendo/escribiendo sólo cuatro bytes |
| Partículas | paso 0x70; P_ENT +0x3C contiene dirección en int | `src/Render/Particle_Move.cpp:49`, `Particle_Spawn.cpp`, `Weather_Particles.cpp`, `ItemDrop_Render2.cpp`; rediseño del pool y macros P_* |
| BMD y subregistros | BMD 188 / 0xBC: seis punteros (+0x28,2C,30,34,38,40); Mesh/Bone/Action tienen subarrays y punteros | `src/Model/ModelTypes.h`, `src/Render/BMD_Load.cpp`, `BMD_Anim.cpp`, `BMD_DrawMesh.cpp`, `BMD_FilterData.cpp`; rediseño de runtime y lectura separada del archivo |
| Texturas: TexSlot y BITMAP_t | 0x38 heredado; pixels/Buffer +0x34, ambas vistas contienen puntero | `src/Render/Texture/Texture.cpp:100`, `src/structs.h:369`, `src/Render/ResourceTables.cpp`, consumidores g_BitmapsRaw/Bitmaps; unificar contrato y aserciones |

Las filas son familias, no conteos de accesos; el anexo 2 muestra los sitios
textuales por archivo. Las dos vistas TexSlot/BITMAP_t se cuentan en una misma
fila (9 familias). Los grupos 2c–2j separan referencias por familia; entidades y
efectos comparten OBJECT. Tampoco son un conteo de accesos efectivos: incluyen
declaraciones y llamadas de helpers, y no reconocen alias como Models.

Con el pack actual y sin cambiar campos, el crecimiento calculado es OBJECT
0x1C4, CHARACTER 0x39C, JOINT 0x9DC y BMD 0xD4 en x64: dos, dos heredados,
uno y seis punteros respectivamente. Es una consecuencia del layout, no un test
compilado. TexSlot usa alineación nativa y también puede agregar padding.
No fijar todavía un ABI x64 definitivo ni serializar estos tamaños.

`ResourceTables.cpp` reserva modelos con 0xBC y texturas con 0x38; migrar también
reservas, crecimiento, reset y conteo. Lo mismo aplica a rangos/strides en
`SceneObjects`, snapshots/interpolación y validadores como el de
`MoveEffect.cpp:77-82`; cambiar sólo el struct deja recorridos inconsistentes.

## 3. Formatos de disco y red

| Familia de disco (8 contratos) | Representación actual y verificación necesaria |
|---|---|
| Modelos .bmd | `src/Render/BMD_Load.cpp`: parsea buffer por cursor, construye punteros en memoria; separar cabeceras y registros de archivo de BMD, Mesh/Bone/Action nativos. Copiar bytes del archivo nunca debe copiar punteros |
| Terrain .att | `src/Terrain/Terrain_LegacyLoad.cpp:122`: 65539 bytes; cabecera 3 y 65536 celdas; conservar verificación y XOR; contrato de bytes + aserciones |
| EncTerrain .map | mismo archivo: cabecera de 2 bytes, tres capas de 65536 bytes, alpha se expande a float; conservar MapFileDecrypt, orden y normalización |
| EncTerrain .obj | mismo archivo: cabecera de 4 bytes, count short en +2 y registros de 30 bytes; ID, posición, ángulos, escala; no confundirlo con OBJECT runtime |
| item.bmd | `src/Item/Item_Data.cpp`: 0x8000 bytes de registros 0x40 + checksum DWORD; ITEM_ATTRIBUTE ya tiene sizeof=64 y offsetof en `src/structs.h:327-334`, y usa enteros exactos; TwoHand bool tiene aserción de 1 byte |
| skill.bmd | `src/Item/Skill_Data.cpp`: payload 0xA00, registros 0x28 + checksum DWORD. El código recorre 64 registros (0xA00/0x28), aunque un comentario dice 40; no copiar ese comentario como contrato |
| gate.bmd | `src/Item/Gate_Data.cpp`, GATE_ATTRIBUTE en `src/structs.h:337`; sizeof=9 y campos críticos ya comprobados |
| Otros bloques locales | `src/Item/Dialog_Data.cpp` (0x400), `Quest_Data.cpp` (0x248), `src/Local/Text_Data.cpp`; formalizar cabeceras/longitudes sin importar sizeof nativo |

La referencia a Terrain/EncTerrain describe los lectores actuales, incluidos sus
fallbacks; no demuestra que todas las variantes de assets estén soportadas.
Hay aserciones de disco **ya existentes** para items y gates: ampliarlas,
no duplicarlas ni afirmar que faltan todas. `ITEM` 68 no es `ITEM_ATTRIBUTE` 64.

Red: `src/Net/Protocol/ProtocolBase.h`, `ConnectServerProtocol.h`,
`GameServerProtocol.h`, `Protocol_Check.cpp` ya definen/verifican contratos.
El anexo 3b cuenta las aserciones textuales de esa carpeta. Auditar también
casts de paquetes en `src/Net/Recv/`, catálogo y wrappers CRT: los fread/fwrite
con FILE* disfrazado de int* no son campos persistentes, pero sus firmas también
deben preservar el puntero. No cambiar opcodes, packing ni texto en el cable.

Tests previstos por familia: leer muestras actuales, verificar hash y tamaño,
campos y checksum, comparar salida x86/x64 y round-trip cuando haya writer.
Cubrir truncamiento/longitudes inválidas sin imponer un formato nuevo.
No se agregaron tests en este PR exclusivamente documental.

## 4. Globales y slots mixtos

Bases confirmadas por asignación/consumo en fuente (9 símbolos):

| Símbolo | Tipo actual / evidencia |
|---|---|
| DAT_05828d58 | DWORD; `src/Model/Model_Items.cpp:27`, `ClientEntry.cpp:677`; Models desde ResourceTables |
| DAT_055c9e44 | DWORD; asignación de reserva y base desplazada en ClientEntry |
| DAT_07d78068 | int; tabla item y base calculada desde reserva anterior |
| DAT_07cf1ff0 | int; malloc(0x8000), copia sombra de items |
| DAT_07cf1ff8 | int; malloc(0xA00), copia sombra de skills |
| DAT_07d29d20 | int; malloc(0xA00), tabla principal de skills |
| GateAttribute | DWORD; malloc(900), registros de 9 bytes |
| DAT_055c9e48 | DWORD; malloc(0x764D4) en `src/Platform/ClientEntry.cpp:615`; reserva de entidades |
| CharactersClient / DAT_07abf5d0 | DWORD; base desplazada dentro de la reserva anterior (:616); `src/globals.cpp:710` |

Declaraciones/definiciones: `src/globals.h`, `src/globals.cpp`.
`g_ItemAttribute_Backup` se usa en `src/Render/BMD_LegacyDraw.cpp:414,644`
para restaurar la base mediante un cast a int; el barrido no encontró su
declaración en la base actual y no le asigna un tipo supuesto. Otros DWORD
pendientes de seguir por xrefs:
DAT_07e12945, DAT_07abf634, DAT_005592d8, DAT_005592d4. Sus comentarios sugieren
punteros, pero no se convierten sólo por el nombre. `Hero`, DAT_07cf1ffc y
DAT_06f42a58 ya son punteros. DAT_083a7ca0 es actualmente un alias DWORD al
primer slot de g_BitmapsRaw (`globals.h:1133`), no otra base independiente.
FogColor también requiere decisión: el comentario dice puntero/inicio de float.

Cuatro funciones prioritarias: `MoveEffect`, `Effect_Create`, `Joint_Create`,
`MoveJoint`. Dos campos nativos se siguen accediendo como palabra de cuatro
bytes: OBJECT::Owner y JOINT::Target. En `Joint_Create.cpp:143` se escribe
Target vía int*; en `MoveJoint.cpp:111,129,149` se lee como int. En
`Effect_Create.cpp:1211` se recuperan bits de Owner vía unsigned int*.
`MoveEffect.cpp:255` documenta el alias histórico float del owner.

Los anejos 4a/4b cuentan los sitios actuales de alias y mezcla sintáctica.
No todo alias float es un campo verdaderamente dual-use: puede ser la
representación elegida por el decompilador. Antes de migrar hay que construir
la matriz Type/SubType → campo → writers/readers en las cuatro funciones,
incluyendo MultiUse y temporales/reutilizaciones de parámetros. Esa matriz
no está confirmada contra IDA en esta sesión; requiere decisión. Elegir campos
separados o una unión discriminada sólo con evidencia; no hacer float→uintptr_t
numérico ni guardar ocho bytes sobre un slot de cuatro.

## 5. ABI, long, wchar_t y precisión

Windows x64 conserva int/long de 32 bits; Linux x64 tiene long y puntero de 64.
`long long` no debe confundirse con long. El patrón 5a excluye long long y long
double; las líneas de código generado GL también aparecen y no son cambios
automáticos. Revisar formatos printf/scanf y límites cuando se cambie un tipo.
`SoundManager::LevelToDirectSound` usa long para dB y `ClientSettings.h` para
strtol: no son por sí mismos almacenamiento de direcciones.

`wchar_t` nativo no sirve como contrato UTF-16 compartido; Posix/WinTypes.h
ya usa WCHAR uint16_t. Mantener bytes de disco/red y fronteras de codificación
existentes; usar tipos explícitos cuando corresponda. El anexo enumera usos.

`src/Platform/Posix/WinTypes.h:88-93` exige sizeof(void*)=4 y handles/parámetros
de ventana/socket de cuatro bytes. Los alias INT_PTR/WPARAM/SOCKET ya derivan
de intptr_t/uintptr_t: las aserciones y sus consumidores precisan revisión.
Buscar además conversiones implícitas, `%x/%u` para punteros y tamaños de memcpy;
el escaneo textual no prueba ausencia de supuestos sizeof(int)==sizeof(void*).

No se encontró asm inline con el patrón 5d en el alcance analizado (si el anexo
registra cero). Sí hay x87 traducido: float10 = long double en `src/types.h:17`
y `functions.h`, fsin/fcos y temporales en Math_3D, Entity_Reset y otros. La
precisión de long double difiere entre compiladores; conservar resultados con
tests de referencia antes de sustituir. Comentarios st7 no son asm ejecutable.
No copiar denormales sin f, locales supuestamente contiguos ni conversiones
numéricas donde el original transporta bits.

## 6. Parámetros float que son punteros

Un caso confirmado en fuente: FUN_004fdc00 (0x004FDC00).
Declaración `src/functions.h:1297`; definición activa
`src/Scene/Scene_ObjectUpdate.cpp:1016`, copia gateada en
`Scene_ObjectLegacy.cpp:46`; llamada activa `Scene_ObjectUpdate.cpp:629`.
En :626-628 el llamador hace DWORD→memcpy de cuatro bytes a float; el receptor
usa LODWORD(o). La firma debe pasar a OBJECT* (y llamadores/copia/declaración
juntos), una vez verificado contra IDA o test equivalente. No convertir el
valor float numéricamente a una dirección.

Otra firma del mismo tick original: `MoveObject_PerWorld(float param_1)`,
declarada en `src/functions.h:1198`, implementada como resumen en
`src/Scene/Scene_ObjectUpdate.cpp:21` y llamada sólo desde la copia gateada
anterior. En :31 hace `(int)param_1`, una conversión numérica; es un artefacto
ya documentado en esa fuente y no una implementación de referencia adicional.
Son dos nombres de función y dos implementaciones de FUN_004fdc00 para una
única dirección original; revisar las tres definiciones sin reactivar código.

Para similares, el anexo 6b enumera LODWORD/SLODWORD sin comentarios ni strings;
muchos son bits numéricos válidos. No hay un conteo confirmado de otras funciones
con este defecto: requiere rastrear parámetros y llamadores, no reemplazar todas
las firmas float. El conteo 6 mezcla llamadas, declaraciones y copias gateadas,
y no significa varias funciones distintas.

## Orden de PR y criterios de aceptación de la Fase B

1. Tipos: casts, globales y firmas por causa raíz, con conteo antes/después y
   cadena completa de consumidores. Los casos dual-use quedan en PR específico
   con evidencia, no en una sustitución masiva de DWORD.
2. Piloto de layouts: ITEM (ya tipado, sin punteros) o números de daño, después
   OBJECT/CHARACTER y suelo, efectos, joints, partículas, modelos/subregistros y
   texturas. PR por familia; conservar aserciones x86 y agregar las nativas x64
   y tests de operaciones antes de cambiar reservas/strides.
3. Disco/red: contratos exactos separados de runtime, tests con muestras y
   aserciones. No escribir sizeof(BMD/OBJECT/JOINT) en archivos/paquetes.
4. Build: agregar `windows-sdl3-x64-release` y `linux-x64-release`, mantener
   presets x86. `cmake/toolchains/linux-x86.cmake` conserva -m32: crear ruta x64
   propia; revisar `cmake/MuCompilerOptions.cmake`, dependencias, arquitectura,
   aserciones Posix y CI `.github/workflows/build-sdl3.yml`, `build-windows.yml`.
5. Cierre: distinguir nombres DAT_ históricos de direcciones absolutas reales;
   sustituir sólo las direcciones y recorridos verificados. El anexo de casts a
   direcciones es un mínimo sintáctico; no incluye tablas de funciones ni enteros
   usados como dirección sin cast explícito en la misma expresión.

Cada PR contra fase/4, commits por causa raíz con evidencia IDA/test y comentarios
en español. Para cada capa: compilar x86 y ambas x64 cuando estén habilitadas,
correr checks y sesión login → char-select → Lorencia → mover → inventario.
Conservar logs de la misma base/servidor/assets y comparar debug.log separando
sólo campos volátiles identificados (direcciones/tiempo). La igualdad de logs
por sí sola no certifica visuales, animación ni todos los subtipos de efecto.

## Verificación de este PR

- `check_cmake_sources.py`: OK (334 ClCompile, 163 ClInclude).
- `check_render_layer.py`: OK.
- `check_bmd_boundary.py`: OK (16 productores/contratos).
- Configuración `cmake --preset windows-sdl3-x86-release`: OK; generador local
  Visual Studio 18 2026, toolset v143, Win32 (MSVC 19.44).
- Build `cmake --build --preset windows-sdl3-x86-release -- -m -v:m`: OK,
  salida `bin/Client/main.exe`. Se pasaron los argumentos como lista para evitar
  que PowerShell separara `-v:m` en `-v:` y `m` (primer intento MSB1016).
  Hay advertencias preexistentes C4700 y C4789 (entre otras); no se corrigieron
  en este PR. Build satisfactorio no equivale a equivalencia de ejecución.
- No se ejecutó sesión de juego ni comparación de debug.log; este PR no cambia
  código ejecutable. No se compiló ni ejecutó x64 ni Linux.
- IDA: conexión rechazada; no se certificaron nuevos offsets contra el binario.

## Anexo: cantidades y sitios reproducibles

La tabla resumen y las tablas por archivo son candidatos textuales según las
expresiones indicadas. Una línea puede contener varios sitios; se repite en la
lista cuando corresponde. Cero significa ausencia del patrón, no prueba de
ausencia del problema. Los grupos 2 no son una deducción de la identidad del
pool: el mismo literal puede ser tamaño de archivo, flag u otra constante.

Archivos examinados: **500**.

| Patrón | Sitios | Archivos |
|---|---:|---:|
| 1a casts a entero estrecho (candidatos) | 6443 | 246 |
| 1b uintptr_t/intptr_t (revisar cadena completa) | 1201 | 155 |
| 1c aritmética directa sobre DAT_ | 1706 | 156 |
| 2a strides de layouts (candidatos) | 818 | 160 |
| 2b accesos por offset (candidatos) | 4637 | 177 |
| 2c items inventario (referencias tipadas) | 465 | 42 |
| 2d daño (referencias al pool) | 9 | 7 |
| 2e entidades y efectos (referencias tipadas) | 4558 | 85 |
| 2f items suelo (referencias tipadas) | 16 | 2 |
| 2g joints (referencias tipadas) | 971 | 8 |
| 2h partículas (pool y macros) | 676 | 8 |
| 2i modelos (referencias tipadas y base) | 346 | 66 |
| 2j texturas (vistas y base) | 64 | 18 |
| 3a entradas/salidas binarias | 69 | 22 |
| 3b aserciones de red | 223 | 3 |
| 3c aserciones de disco existentes | 11 | 1 |
| 4a alias de cuatro bytes en Owner/Target/BoneMatrices | 161 | 9 |
| 4b casts uintptr_t junto a float | 106 | 18 |
| 4c declaraciones escalares estrechas en globals.h (candidatos) | 580 | 1 |
| 5a long simple (no long long ni long double) | 23 | 12 |
| 5b wchar_t | 8 | 2 |
| 5c supuestos explícitos de ancho de puntero | 7 | 1 |
| 5d asm inline | 0 | 0 |
| 5e precisión extendida/x87 traducido | 553 | 34 |
| 6 transporte conocido FUN_004fdc00 | 4 | 3 |
| 6b LODWORD/SLODWORD (candidatos adicionales) | 241 | 51 |
| cierre direcciones absolutas casteadas (candidatos) | 41 | 10 |

### 1a casts a entero estrecho (candidatos)

Expresión (Python re):

```text
\(\s*(?:int|DWORD|uint|unsigned(?:\s+int)?|long|unsigned\s+long)\s*\)
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 146 | 479, 484, 515, 551, 590, 590, 591, 591, 636, 809, 810, 822, 853, 862, 862, 863, 906, 907, 908, 909, 920, 921, 931, 933, 933, 943, 943, 946, 948, 948, 959, 959, 968, 981, 985, 988, 992, 995, 1015, 1018, 1042, 1043, 1089, 1090, 1099, 1102, 1120, 1122, 1130, 1146, 1148, 1152, 1166, 1191, 1191, 1201, 1207, 1219, 1219, 1228, 1228, 1233, 1234, 1241, 1244, 1247, 1248, 1249, 1253, 1255, 1260, 1263, 1293, 1294, 1321, 1326, 1353, 1354, 1361, 1362, 1373, 1374, 1381, 1400, 1404, 1408, 1415, 1419, 1467, 1497, 1506, 1523, 1563, 1570, 1570, 1587, 1590, 1591, 1597, 1598, 1675, 1682, 1683, 1846, 1852, 1875, 1875, 1891, 1891, 1962, 1978, 2003, 2006, 2105, 2106, 2110, 2111, 2131, 2168, 2259, 2260, 2267, 2268, 2313, 2315, 2346, 2346, 2352, 2354, 2364, 2366, 2374, 2400, 2402, 2411, 2412, 2416, 2417, 2440, 2440, 2465, 2465, 2474, 2474, 2491, 2491 |
| `src/Combat/Combat_AttackEffect.cpp` | 34 | 50, 51, 55, 60, 74, 83, 84, 85, 85, 86, 87, 91, 92, 446, 493, 564, 646, 693, 801, 848, 936, 977, 1089, 1136, 1228, 1275, 1363, 1410, 1569, 1616, 1699, 1746, 1877, 2118 |
| `src/Combat/Combat_AttackStage.cpp` | 10 | 15, 30, 62, 125, 163, 230, 269, 275, 329, 340 |
| `src/Combat/Combat_LegacyAnimation.cpp` | 3 | 85, 91, 92 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 22 | 65, 278, 287, 288, 295, 296, 302, 313, 327, 336, 353, 369, 370, 379, 386, 387, 394, 395, 405, 416, 423, 424 |
| `src/Combat/Combat_Projectiles.cpp` | 38 | 18, 19, 21, 26, 29, 30, 47, 51, 54, 78, 78, 84, 84, 90, 90, 96, 96, 102, 102, 108, 108, 114, 114, 120, 120, 126, 126, 143, 143, 150, 150, 157, 157, 164, 164, 196, 213, 230 |
| `src/Combat/Combat_Targeting.cpp` | 10 | 24, 24, 25, 25, 48, 49, 64, 64, 65, 65 |
| `src/Combat/Skills.cpp` | 24 | 75, 81, 94, 96, 212, 223, 230, 231, 237, 276, 283, 290, 298, 307, 323, 331, 344, 351, 357, 370, 381, 392, 397, 405 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 16 | 118, 119, 165, 201, 240, 243, 274, 281, 282, 284, 355, 370, 409, 417, 419, 425 |
| `src/Combat/Skills_WizardElf.cpp` | 46 | 75, 81, 82, 98, 99, 102, 113, 117, 118, 135, 135, 136, 170, 173, 174, 192, 193, 200, 216, 299, 303, 307, 349, 350, 383, 401, 404, 423, 451, 458, 458, 490, 491, 497, 518, 522, 535, 535, 536, 536, 537, 542, 542, 561, 561, 562 |
| `src/Config/Config_Load.cpp` | 9 | 82, 137, 138, 255, 256, 256, 264, 281, 282 |
| `src/Config/UserSettings.cpp` | 1 | 40 |
| `src/Core/Legacy_Runtime.cpp` | 2 | 64, 69 |
| `src/Core/Runtime_Linker.cpp` | 1 | 76 |
| `src/Core/Runtime_Medium.cpp` | 41 | 46, 47, 48, 48, 181, 182, 184, 211, 211, 213, 215, 215, 217, 218, 232, 254, 256, 276, 277, 281, 299, 305, 307, 307, 313, 314, 431, 481, 491, 500, 504, 508, 548, 552, 554, 555, 556, 560, 560, 561, 637 |
| `src/Core/Runtime_Small.cpp` | 34 | 47, 54, 62, 67, 70, 90, 98, 121, 124, 147, 199, 241, 269, 279, 280, 281, 329, 381, 382, 412, 413, 421, 422, 425, 426, 462, 589, 590, 702, 711, 713, 741, 828, 829 |
| `src/Core/String_ResourceLegacy.cpp` | 1 | 55 |
| `src/Core/Window.cpp` | 12 | 41, 57, 58, 66, 67, 72, 72, 88, 93, 94, 99, 100 |
| `src/Entity/CharacterCloth.cpp` | 23 | 24, 41, 76, 79, 84, 86, 86, 101, 106, 107, 107, 131, 136, 137, 137, 163, 167, 169, 169, 222, 227, 229, 229 |
| `src/Entity/CharacterEffects.cpp` | 60 | 43, 45, 69, 85, 98, 138, 140, 150, 159, 166, 166, 166, 168, 180, 181, 185, 186, 187, 188, 189, 190, 194, 195, 196, 197, 198, 199, 200, 201, 202, 206, 207, 211, 215, 226, 228, 243, 251, 260, 261, 275, 281, 284, 286, 288, 302, 304, 306, 308, 310, 323, 324, 339, 341, 343, 345, 347, 372, 383, 386 |
| `src/Entity/EntityName.cpp` | 1 | 31 |
| `src/Entity/Entity_ActionLegacy.cpp` | 3 | 63, 70, 70 |
| `src/Entity/Entity_HeadMotion.cpp` | 3 | 67, 91, 92 |
| `src/Entity/Entity_Init.cpp` | 2 | 58, 59 |
| `src/Entity/Entity_LegacyEffects.cpp` | 6 | 124, 125, 126, 128, 129, 130 |
| `src/Entity/Entity_LegacyPoolCleanup.cpp` | 4 | 45, 65, 65, 89 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 4 | 39, 44, 81, 98 |
| `src/Entity/Entity_MoveClient.cpp` | 2 | 44, 44 |
| `src/Entity/Entity_Reset.cpp` | 5 | 79, 115, 115, 116, 117 |
| `src/Entity/Entity_Spawn.cpp` | 16 | 31, 32, 33, 42, 42, 80, 80, 108, 109, 144, 161, 162, 170, 171, 202, 254 |
| `src/Entity/ObjectPose.cpp` | 39 | 18, 19, 20, 21, 22, 26, 27, 28, 30, 31, 32, 42, 85, 88, 121, 124, 133, 135, 145, 148, 149, 164, 164, 165, 165, 185, 185, 187, 187, 187, 194, 194, 200, 201, 312, 396, 493, 500, 501 |
| `src/Game/Game_CharSelectTick.cpp` | 7 | 66, 67, 67, 88, 179, 221, 222 |
| `src/Game/Game_EnterWorldTick.cpp` | 18 | 93, 122, 302, 357, 357, 365, 366, 456, 457, 488, 564, 587, 588, 594, 594, 595, 596, 618 |
| `src/Game/Game_MainLoop.cpp` | 20 | 138, 145, 146, 148, 149, 150, 151, 159, 164, 277, 277, 277, 278, 278, 292, 303, 304, 305, 411, 469 |
| `src/Game/Game_SceneUpdate.cpp` | 16 | 62, 84, 180, 294, 306, 374, 425, 466, 469, 488, 523, 531, 553, 561, 640, 793 |
| `src/Game/MapManager.cpp` | 3 | 141, 148, 156 |
| `src/Game/PathFinder.cpp` | 177 | 54, 55, 56, 59, 60, 61, 63, 64, 79, 82, 82, 90, 93, 105, 105, 150, 158, 158, 165, 166, 184, 185, 202, 202, 231, 239, 239, 245, 246, 264, 265, 282, 282, 417, 419, 420, 427, 429, 430, 439, 439, 439, 440, 441, 443, 443, 444, 444, 445, 446, 446, 446, 449, 450, 451, 451, 451, 454, 459, 461, 465, 465, 469, 473, 473, 473, 474, 478, 482, 482, 482, 483, 488, 491, 491, 493, 494, 498, 498, 498, 499, 502, 503, 507, 507, 507, 508, 512, 513, 517, 522, 526, 527, 528, 529, 530, 531, 534, 535, 536, 538, 540, 543, 548, 553, 556, 557, 565, 567, 569, 573, 577, 578, 584, 585, 587, 589, 590, 599, 601, 602, 604, 604, 605, 605, 605, 607, 608, 609, 610, 611, 613, 623, 625, 627, 628, 632, 633, 650, 651, 652, 653, 655, 656, 657, 660, 661, 662, 665, 666, 667, 676, 677, 689, 698, 701, 703, 712, 714, 715, 717, 717, 718, 718, 718, 720, 721, 722, 723, 724, 726, 730, 731, 732, 739, 797, 804 |
| `src/Game/Player_InputTick.cpp` | 109 | 79, 80, 113, 114, 120, 127, 128, 135, 136, 206, 255, 256, 425, 426, 426, 436, 437, 437, 438, 438, 440, 440, 605, 623, 644, 658, 663, 667, 682, 683, 688, 688, 701, 722, 732, 737, 752, 755, 761, 761, 784, 785, 787, 790, 794, 819, 823, 824, 825, 826, 839, 855, 907, 981, 1224, 1249, 1257, 1258, 1260, 1317, 1318, 1330, 1330, 1336, 1337, 1387, 1396, 1396, 1397, 1397, 1399, 1399, 1404, 1417, 1417, 1420, 1420, 1437, 1438, 1447, 1448, 1455, 1455, 1457, 1457, 1467, 1471, 1476, 1477, 1478, 1479, 1488, 1488, 1491, 1491, 1531, 1532, 1558, 1558, 1559, 1559, 1605, 1606, 1613, 1613, 1624, 1624, 1668, 1669 |
| `src/Game/RenderFxSnapshots.cpp` | 10 | 44, 79, 92, 105, 117, 118, 124, 138, 148, 152 |
| `src/Game/RenderInterpolation.cpp` | 5 | 16, 62, 63, 125, 181 |
| `src/Game/RenderSnapshots.cpp` | 3 | 109, 256, 256 |
| `src/Game/SimulationPreparation.cpp` | 21 | 75, 78, 112, 129, 135, 136, 138, 140, 141, 142, 146, 148, 150, 151, 153, 167, 167, 186, 186, 187, 187 |
| `src/Game/Timer.cpp` | 3 | 58, 61, 74 |
| `src/Input/Cursor_Render.cpp` | 10 | 42, 46, 47, 75, 76, 76, 117, 118, 121, 122 |
| `src/Input/Input_Timing.cpp` | 8 | 40, 48, 52, 56, 58, 59, 62, 65 |
| `src/Input/Input_WndProc.cpp` | 14 | 53, 79, 80, 181, 193, 195, 259, 260, 260, 356, 365, 369, 372, 372 |
| `src/Input/Mouse_Hover.cpp` | 15 | 78, 79, 324, 391, 393, 434, 436, 437, 442, 452, 454, 502, 503, 541, 595 |
| `src/Item/ChaosMixRates.cpp` | 3 | 68, 69, 76 |
| `src/Item/ContentCatalog.cpp` | 20 | 92, 202, 207, 304, 335, 338, 339, 403, 424, 446, 461, 473, 477, 491, 519, 520, 599, 658, 850, 856 |
| `src/Item/Dialog_Data.cpp` | 1 | 31 |
| `src/Item/Gate_Data.cpp` | 2 | 53, 80 |
| `src/Item/Item_ChaosMix.cpp` | 27 | 77, 82, 82, 83, 83, 85, 85, 85, 86, 86, 108, 120, 124, 127, 127, 129, 129, 133, 194, 196, 197, 212, 224, 226, 227, 243, 354 |
| `src/Item/Item_ClickHandler.cpp` | 104 | 189, 189, 192, 192, 248, 288, 307, 309, 380, 381, 385, 386, 400, 569, 602, 633, 637, 637, 638, 638, 657, 665, 666, 666, 667, 667, 672, 686, 691, 709, 719, 719, 720, 721, 721, 743, 748, 799, 809, 854, 912, 946, 984, 986, 987, 995, 996, 997, 1001, 1002, 1003, 1008, 1009, 1010, 1018, 1021, 1023, 1024, 1047, 1067, 1068, 1069, 1075, 1103, 1174, 1176, 1241, 1242, 1243, 1249, 1250, 1251, 1258, 1259, 1260, 1267, 1268, 1269, 1292, 1292, 1293, 1293, 1294, 1294, 1295, 1295, 1312, 1312, 1313, 1313, 1330, 1330, 1331, 1331, 1332, 1332, 1333, 1333, 1407, 1407, 1421, 1421, 1422, 1422 |
| `src/Item/Item_Data.cpp` | 4 | 78, 91, 129, 143 |
| `src/Item/Item_Display.cpp` | 34 | 29, 30, 83, 83, 93, 94, 97, 244, 244, 245, 248, 248, 249, 350, 351, 372, 372, 374, 374, 480, 481, 482, 509, 519, 520, 525, 555, 560, 570, 610, 611, 621, 634, 635 |
| `src/Item/Item_Durability.cpp` | 29 | 57, 62, 112, 121, 132, 222, 222, 338, 357, 378, 378, 381, 421, 450, 464, 467, 467, 469, 470, 485, 485, 486, 499, 499, 501, 502, 513, 513, 514 |
| `src/Item/Item_Equip.cpp` | 19 | 70, 75, 77, 78, 79, 80, 98, 98, 102, 102, 119, 130, 131, 141, 143, 146, 224, 227, 231 |
| `src/Item/Item_EquipmentAutoSwap.cpp` | 2 | 29, 82 |
| `src/Item/Item_EquipmentQueries.cpp` | 2 | 223, 223 |
| `src/Item/Item_Inventory.cpp` | 17 | 114, 120, 209, 219, 226, 230, 231, 231, 231, 240, 282, 294, 295, 295, 295, 383, 395 |
| `src/Item/Item_InventoryGrid.cpp` | 45 | 18, 30, 43, 43, 57, 57, 57, 60, 60, 63, 63, 79, 80, 91, 91, 91, 94, 94, 94, 165, 165, 214, 219, 231, 253, 254, 255, 266, 267, 271, 276, 291, 295, 299, 310, 311, 322, 330, 351, 352, 358, 363, 388, 392, 431 |
| `src/Item/Item_InventoryRender.cpp` | 23 | 41, 163, 164, 233, 236, 239, 267, 268, 351, 352, 353, 355, 356, 361, 362, 368, 377, 378, 411, 412, 412, 415, 418 |
| `src/Item/Item_LegacyGridInsert.cpp` | 1 | 159 |
| `src/Item/Item_LegacyHelpers.cpp` | 4 | 59, 59, 83, 83 |
| `src/Item/Item_LegacyLinker.cpp` | 4 | 227, 332, 339, 526 |
| `src/Item/Item_Move.cpp` | 2 | 60, 61 |
| `src/Item/Item_ServerValue.cpp` | 9 | 58, 65, 181, 195, 197, 256, 261, 287, 299 |
| `src/Item/Item_SpecialOption.cpp` | 2 | 17, 18 |
| `src/Item/NPC_Data.cpp` | 1 | 26 |
| `src/Item/Skill_Data.cpp` | 6 | 57, 67, 105, 123, 124, 134 |
| `src/Local/ClientText.cpp` | 1 | 77 |
| `src/Local/Text_Data.cpp` | 4 | 64, 64, 69, 69 |
| `src/Math/Math_3D.cpp` | 4 | 163, 172, 173, 174 |
| `src/Math/Math_LegacyAngle.cpp` | 1 | 52 |
| `src/Math/Math_LegacyTransforms.cpp` | 2 | 88, 89 |
| `src/Model/Model_Items.cpp` | 51 | 27, 32, 161, 162, 163, 164, 165, 170, 171, 172, 173, 174, 179, 180, 181, 182, 183, 187, 188, 189, 190, 191, 192, 193, 194, 195, 198, 227, 228, 229, 230, 231, 277, 278, 279, 280, 281, 282, 283, 284, 285, 286, 287, 288, 289, 290, 291, 292, 293, 294, 295 |
| `src/Model/Model_Misc.cpp` | 62 | 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 67, 100, 136, 137, 138, 139, 140, 141, 142, 143 |
| `src/Model/Model_Players.cpp` | 96 | 18, 19, 21, 22, 26, 29, 30, 31, 32, 33, 34, 35, 39, 41, 42, 46, 47, 51, 52, 53, 54, 55, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 77, 78, 79, 83, 87, 90, 91, 92, 92, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 119, 120, 121, 122, 127, 129, 142, 143, 149, 180, 184, 201, 205, 223, 266, 267, 268, 269, 285, 286, 287, 303, 304, 305, 306 |
| `src/Monster/Monster.cpp` | 10 | 525, 525, 542, 583, 584, 620, 638, 691, 710, 1055 |
| `src/Monster/Monster_Data.cpp` | 12 | 56, 57, 58, 59, 60, 61, 62, 123, 123, 149, 149, 149 |
| `src/Net/Crypto.cpp` | 116 | 121, 130, 130, 188, 197, 197, 232, 233, 233, 235, 237, 238, 241, 273, 280, 286, 298, 298, 301, 302, 319, 319, 322, 323, 326, 328, 354, 358, 364, 376, 376, 395, 395, 401, 403, 405, 506, 528, 528, 529, 529, 541, 541, 625, 625, 627, 627, 673, 681, 681, 697, 703, 704, 706, 707, 708, 709, 710, 724, 753, 781, 781, 787, 787, 804, 815, 815, 820, 855, 863, 864, 917, 946, 961, 963, 995, 1005, 1016, 1024, 1035, 1039, 1060, 1077, 1079, 1085, 1088, 1104, 1114, 1122, 1134, 1135, 1142, 1161, 1162, 1163, 1164, 1205, 1225, 1273, 1274, 1275, 1276, 1280, 1338, 1344, 1344, 1352, 1388, 1389, 1399, 1418, 1419, 1420, 1438, 1441, 1521 |
| `src/Net/HWID.cpp` | 1 | 39 |
| `src/Net/MuEmu.cpp` | 2 | 57, 59 |
| `src/Net/Net_Connect.cpp` | 13 | 45, 47, 133, 137, 137, 154, 154, 173, 197, 197, 214, 214, 236 |
| `src/Net/Net_Events.cpp` | 15 | 46, 46, 60, 60, 64, 89, 105, 144, 164, 165, 165, 170, 172, 173, 173 |
| `src/Net/Net_LegacyAuth.cpp` | 3 | 79, 87, 119 |
| `src/Net/Net_LegacyRuntime.cpp` | 3 | 51, 51, 195 |
| `src/Net/Net_Process.cpp` | 12 | 664, 683, 684, 684, 685, 721, 722, 724, 727, 727, 744, 769 |
| `src/Net/Network.cpp` | 18 | 87, 238, 248, 254, 257, 259, 260, 263, 265, 267, 270, 275, 298, 304, 325, 328, 408, 486 |
| `src/Net/PacketFrame.cpp` | 1 | 77 |
| `src/Net/Ping.cpp` | 1 | 35 |
| `src/Net/Reconnect.cpp` | 3 | 216, 216, 217 |
| `src/Net/Recv/Recv_Account.cpp` | 7 | 103, 143, 206, 309, 395, 395, 411 |
| `src/Net/Recv/Recv_Character.cpp` | 54 | 62, 181, 183, 195, 197, 219, 225, 255, 256, 257, 426, 442, 492, 539, 542, 548, 548, 591, 643, 643, 704, 705, 720, 720, 741, 852, 856, 856, 856, 857, 857, 858, 865, 892, 978, 1004, 1024, 1084, 1093, 1114, 1253, 1378, 1383, 1470, 1498, 1498, 1498, 1499, 1499, 1532, 1635, 1641, 1668, 1712 |
| `src/Net/Recv/Recv_Chat.cpp` | 7 | 29, 29, 37, 61, 68, 68, 126 |
| `src/Net/Recv/Recv_Combat.cpp` | 62 | 16, 22, 32, 36, 39, 47, 51, 59, 60, 61, 62, 63, 66, 91, 92, 143, 203, 203, 208, 274, 276, 277, 280, 280, 310, 386, 390, 414, 422, 423, 492, 492, 492, 546, 547, 548, 550, 552, 553, 554, 555, 556, 557, 560, 636, 648, 663, 716, 716, 717, 718, 749, 790, 791, 804, 819, 863, 882, 884, 885, 886, 893 |
| `src/Net/Recv/Recv_Event.cpp` | 3 | 43, 44, 198 |
| `src/Net/Recv/Recv_Guild.cpp` | 23 | 125, 143, 169, 186, 186, 209, 209, 235, 290, 300, 303, 317, 346, 350, 384, 393, 397, 403, 412, 421, 430, 433, 449 |
| `src/Net/Recv/Recv_Item.cpp` | 46 | 41, 52, 52, 52, 309, 315, 319, 320, 321, 326, 347, 351, 355, 357, 357, 481, 539, 582, 594, 626, 700, 755, 756, 759, 760, 761, 761, 762, 762, 818, 863, 893, 893, 894, 894, 900, 922, 956, 970, 993, 996, 997, 1014, 1018, 1023, 1025 |
| `src/Net/Recv/Recv_Party.cpp` | 1 | 25 |
| `src/Net/Recv/Recv_Quest.cpp` | 8 | 43, 52, 64, 107, 136, 139, 145, 146 |
| `src/Net/Recv/Recv_Trade.cpp` | 4 | 79, 115, 132, 147 |
| `src/Net/Recv/Recv_Viewport.cpp` | 34 | 72, 120, 171, 178, 190, 243, 248, 324, 390, 410, 411, 415, 430, 443, 446, 450, 454, 513, 559, 620, 621, 656, 699, 761, 787, 788, 817, 817, 817, 823, 831, 833, 868, 908 |
| `src/Net/SecondPassword.cpp` | 403 | 205, 213, 229, 319, 320, 321, 322, 337, 337, 338, 338, 351, 351, 352, 352, 367, 368, 421, 429, 460, 461, 504, 504, 505, 505, 506, 506, 507, 507, 524, 524, 525, 525, 526, 526, 527, 527, 536, 537, 537, 538, 538, 539, 539, 551, 551, 552, 552, 553, 553, 554, 554, 584, 584, 585, 585, 586, 586, 587, 587, 610, 611, 612, 631, 631, 632, 632, 633, 633, 634, 634, 661, 661, 662, 662, 663, 663, 664, 664, 670, 670, 671, 671, 672, 672, 673, 673, 677, 677, 678, 678, 679, 679, 680, 680, 684, 684, 685, 685, 686, 686, 687, 687, 693, 693, 694, 694, 695, 695, 696, 696, 700, 700, 701, 701, 702, 702, 703, 703, 713, 717, 718, 720, 750, 750, 751, 758, 768, 771, 771, 772, 772, 773, 773, 774, 774, 781, 781, 782, 782, 783, 783, 784, 784, 803, 806, 820, 825, 831, 863, 864, 866, 866, 893, 893, 945, 946, 950, 951, 960, 969, 970, 984, 985, 1020, 1020, 1021, 1021, 1022, 1022, 1023, 1023, 1027, 1033, 1047, 1051, 1059, 1060, 1068, 1068, 1069, 1069, 1070, 1070, 1071, 1071, 1100, 1102, 1104, 1104, 1105, 1105, 1173, 1173, 1174, 1174, 1175, 1175, 1176, 1176, 1198, 1198, 1199, 1199, 1200, 1200, 1201, 1201, 1230, 1230, 1231, 1231, 1232, 1232, 1233, 1233, 1256, 1264, 1265, 1266, 1266, 1267, 1267, 1280, 1281, 1281, 1282, 1282, 1292, 1293, 1293, 1294, 1294, 1308, 1308, 1309, 1309, 1310, 1310, 1311, 1311, 1435, 1448, 1449, 1449, 1457, 1470, 1542, 1544, 1588, 1592, 1592, 1593, 1593, 1594, 1595, 1605, 1622, 1623, 1624, 1624, 1640, 1652, 1656, 1661, 1662, 1666, 1668, 1670, 1694, 1707, 1729, 1730, 1731, 1731, 1837, 1838, 1839, 1839, 1903, 1904, 1905, 1905, 1930, 2152, 2218, 2225, 2249, 2300, 2371, 2377, 2569, 2570, 2573, 2577, 2618, 2618, 2619, 2653, 2654, 2673, 2712, 2941, 2946, 2966, 2975, 3010, 3045, 3060, 3061, 3085, 3103, 3112, 3119, 3120, 3163, 3196, 3200, 3200, 3233, 3241, 3241, 3266, 3274, 3288, 3294, 3300, 3306, 3316, 3321, 3327, 3329, 3331, 3342, 3353, 3371, 3377, 3401, 3402, 3402, 3408, 3436, 3436, 3443, 3443, 3454, 3456, 3464, 3464, 3474, 3474, 3484, 3484, 3486, 3486, 3493, 3493, 3505, 3505, 3514, 3514, 3526, 3540, 3542, 3542, 3581, 3581, 3627, 3701, 3743, 3752, 3782, 3787, 3789, 3801, 3843, 3845, 3846, 3852, 3854, 3855, 3878, 3962, 3966, 3994, 4007, 4010, 4018, 4019, 4020, 4021, 4022, 4035, 4036, 4074, 4250, 4522, 4531, 4549, 4579, 4623, 4624, 4810 |
| `src/Party/Party.cpp` | 8 | 143, 151, 186, 379, 379, 404, 405, 445 |
| `src/Path/Path_Legacy.cpp` | 24 | 96, 130, 236, 236, 371, 371, 372, 372, 378, 378, 400, 460, 530, 531, 533, 534, 542, 543, 566, 566, 570, 571, 623, 623 |
| `src/Path/Path_LegacyReset.cpp` | 5 | 56, 58, 59, 60, 63 |
| `src/Physics/Cloth_Simulation.cpp` | 25 | 62, 82, 83, 88, 92, 95, 120, 130, 153, 249, 252, 433, 434, 484, 494, 495, 514, 531, 576, 597, 599, 759, 875, 944, 1052 |
| `src/Platform/ClientEntry.cpp` | 15 | 298, 383, 409, 593, 601, 602, 603, 608, 611, 615, 656, 662, 668, 677, 754 |
| `src/Platform/FontFrontend.cpp` | 4 | 206, 278, 288, 347 |
| `src/Platform/Posix/WinTypes.h` | 1 | 90 |
| `src/Platform/SDL3/SDLWindow.cpp` | 1 | 152 |
| `src/Platform/Win32/ClientWndProc.cpp` | 1 | 78 |
| `src/Platform/Win32/MiniDump.cpp` | 1 | 99 |
| `src/Platform/Win32/Win32Audio.cpp` | 3 | 88, 109, 373 |
| `src/Platform/Win32/Win32Diagnostics.h` | 9 | 47, 67, 67, 67, 68, 68, 68, 77, 98 |
| `src/Platform/Win32/Win32GLContext.cpp` | 1 | 89 |
| `src/Platform/Win32/Win32Timer.cpp` | 2 | 78, 88 |
| `src/Render/Ambient_Particles.cpp` | 66 | 127, 128, 131, 133, 135, 141, 143, 149, 176, 182, 188, 190, 192, 193, 198, 201, 202, 202, 208, 209, 213, 214, 219, 221, 222, 222, 228, 231, 233, 234, 240, 243, 259, 266, 266, 270, 273, 278, 284, 285, 287, 287, 288, 288, 293, 295, 313, 318, 320, 321, 325, 330, 332, 340, 342, 348, 349, 354, 376, 379, 384, 385, 386, 390, 395, 400 |
| `src/Render/BMD_Anim.cpp` | 7 | 51, 52, 56, 58, 69, 72, 83 |
| `src/Render/BMD_Collision.cpp` | 12 | 37, 42, 53, 75, 88, 89, 90, 90, 93, 96, 96, 96 |
| `src/Render/BMD_DrawMesh.cpp` | 45 | 109, 111, 147, 152, 159, 182, 185, 185, 191, 194, 194, 205, 205, 218, 228, 236, 237, 238, 251, 260, 261, 262, 264, 282, 319, 319, 320, 320, 321, 321, 353, 358, 360, 361, 362, 439, 456, 469, 470, 471, 473, 489, 498, 499, 499 |
| `src/Render/BMD_FilterData.cpp` | 5 | 29, 92, 113, 135, 194 |
| `src/Render/BMD_LegacyDraw.cpp` | 32 | 59, 60, 63, 65, 68, 73, 85, 144, 155, 157, 411, 414, 641, 644, 706, 1244, 1336, 1478, 1479, 1480, 1605, 1606, 1608, 1663, 1673, 1726, 1844, 1885, 1888, 1896, 1901, 1916 |
| `src/Render/BMD_LegacyEffects.cpp` | 3 | 40, 110, 119 |
| `src/Render/BMD_LegacyLoading.cpp` | 12 | 99, 310, 348, 350, 350, 350, 352, 352, 352, 360, 439, 472 |
| `src/Render/BMD_Load.cpp` | 71 | 56, 57, 58, 76, 157, 165, 166, 167, 177, 178, 179, 180, 181, 182, 183, 184, 189, 189, 189, 207, 208, 209, 215, 215, 216, 216, 217, 217, 218, 218, 219, 219, 237, 249, 250, 251, 252, 265, 299, 315, 323, 337, 346, 351, 355, 356, 357, 377, 378, 379, 393, 394, 395, 409, 411, 411, 444, 447, 448, 471, 575, 578, 582, 584, 585, 586, 625, 627, 641, 655, 682 |
| `src/Render/BMD_SetupRender.cpp` | 60 | 38, 39, 40, 52, 53, 54, 62, 63, 64, 77, 78, 79, 92, 93, 94, 107, 108, 109, 112, 112, 113, 129, 130, 131, 138, 139, 140, 150, 151, 152, 160, 161, 162, 170, 171, 172, 175, 175, 176, 180, 180, 181, 185, 185, 186, 209, 210, 211, 217, 218, 219, 237, 238, 239, 250, 251, 252, 267, 268, 269 |
| `src/Render/BMD_SkinUpdate.cpp` | 8 | 41, 70, 103, 106, 113, 135, 135, 286 |
| `src/Render/Backend/GL/BufferedDraw.cpp` | 1 | 82 |
| `src/Render/Backend/GL/MeshDraw.cpp` | 1 | 147 |
| `src/Render/Backend/GL/TerrainRenderer.cpp` | 13 | 218, 218, 219, 220, 228, 229, 235, 310, 310, 310, 310, 365, 400 |
| `src/Render/Camera.cpp` | 17 | 110, 114, 130, 134, 148, 152, 293, 304, 343, 343, 344, 344, 349, 376, 377, 414, 439 |
| `src/Render/Camera_Movement.cpp` | 4 | 33, 36, 74, 79 |
| `src/Render/Camera_Unproject.cpp` | 2 | 26, 28 |
| `src/Render/Effect_Combat.cpp` | 2 | 99, 105 |
| `src/Render/Effect_Create.cpp` | 549 | 53, 167, 179, 218, 219, 222, 227, 266, 267, 268, 269, 291, 294, 297, 300, 305, 310, 314, 317, 363, 366, 369, 406, 417, 421, 427, 430, 482, 483, 485, 486, 502, 503, 505, 506, 627, 676, 680, 683, 686, 699, 707, 714, 732, 735, 740, 759, 769, 772, 812, 901, 914, 917, 920, 931, 942, 947, 950, 953, 957, 974, 977, 980, 983, 986, 989, 1054, 1057, 1131, 1134, 1137, 1140, 1145, 1152, 1236, 1276, 1298, 1299, 1302, 1302, 1316, 1317, 1320, 1320, 1331, 1332, 1333, 1339, 1355, 1363, 1384, 1396, 1397, 1398, 1412, 1413, 1416, 1416, 1430, 1431, 1434, 1434, 1467, 1471, 1471, 1490, 1494, 1494, 1497, 1498, 1499, 1524, 1525, 1528, 1528, 1544, 1545, 1548, 1548, 1561, 1562, 1563, 1567, 1568, 1569, 1583, 1587, 1587, 1600, 1604, 1604, 1631, 1632, 1632, 1635, 1638, 1639, 1676, 1680, 1680, 1694, 1698, 1698, 1705, 1706, 1707, 1716, 1728, 1730, 1736, 1740, 1740, 1766, 1770, 1770, 1783, 1787, 1787, 1794, 1795, 1796, 1817, 1818, 1818, 1821, 1824, 1825, 1870, 1871, 1874, 1874, 1890, 1891, 1894, 1894, 1907, 1908, 1909, 1924, 1928, 1928, 1942, 1946, 1946, 1955, 1956, 1957, 1960, 1961, 1962, 1975, 1976, 1979, 1979, 1993, 1994, 1997, 1997, 2031, 2035, 2035, 2054, 2058, 2058, 2061, 2062, 2063, 2088, 2089, 2092, 2092, 2108, 2109, 2112, 2112, 2125, 2126, 2127, 2131, 2132, 2133, 2147, 2151, 2151, 2164, 2168, 2168, 2195, 2196, 2196, 2199, 2202, 2203, 2240, 2244, 2244, 2258, 2262, 2262, 2269, 2270, 2271, 2280, 2292, 2294, 2300, 2304, 2304, 2318, 2319, 2320, 2334, 2338, 2338, 2351, 2355, 2355, 2382, 2383, 2383, 2386, 2389, 2390, 2415, 2416, 2417, 2431, 2435, 2435, 2448, 2452, 2452, 2477, 2477, 2479, 2481, 2486, 2490, 2503, 2504, 2505, 2519, 2520, 2523, 2523, 2537, 2538, 2541, 2541, 2575, 2579, 2579, 2598, 2602, 2602, 2605, 2606, 2607, 2631, 2632, 2635, 2635, 2651, 2652, 2655, 2655, 2668, 2669, 2670, 2674, 2675, 2676, 2690, 2694, 2694, 2707, 2711, 2711, 2738, 2739, 2739, 2742, 2745, 2746, 2783, 2787, 2787, 2801, 2805, 2805, 2812, 2813, 2814, 2823, 2835, 2837, 2843, 2847, 2847, 2861, 2862, 2863, 2877, 2878, 2881, 2881, 2895, 2896, 2899, 2899, 2915, 2919, 2920, 2921, 2935, 2939, 2939, 2952, 2956, 2956, 2983, 2984, 2984, 2987, 2990, 2991, 3027, 3028, 3031, 3031, 3047, 3048, 3051, 3051, 3064, 3065, 3066, 3082, 3086, 3086, 3100, 3104, 3104, 3111, 3112, 3113, 3122, 3134, 3136, 3142, 3146, 3146, 3160, 3161, 3162, 3176, 3180, 3180, 3193, 3197, 3197, 3232, 3236, 3236, 3254, 3258, 3258, 3261, 3262, 3263, 3288, 3289, 3292, 3292, 3308, 3309, 3312, 3312, 3325, 3326, 3327, 3331, 3332, 3333, 3347, 3351, 3351, 3364, 3368, 3368, 3395, 3396, 3396, 3399, 3402, 3403, 3430, 3434, 3434, 3448, 3452, 3452, 3459, 3460, 3461, 3470, 3481, 3481, 3483, 3485, 3491, 3508, 3509, 3510, 3524, 3528, 3528, 3541, 3545, 3545, 3572, 3573, 3573, 3576, 3579, 3580, 3617, 3621, 3621, 3635, 3639, 3639, 3646, 3647, 3648, 3655, 3668, 3670, 3676, 3680, 3680, 3753, 3765, 3777, 3779, 3785, 3789, 3789, 3801, 3802, 3803, 3817, 3818, 3821, 3821, 3835, 3836, 3839, 3839, 3873, 3877, 3877, 3896, 3900, 3900, 3903, 3904, 3905, 3931, 3935, 3935, 3948, 3952, 3952, 3963, 3964, 3965, 3973, 3977, 3978, 3979, 3993, 3994, 3997, 3997, 4011, 4012, 4014, 4014, 4048, 4052, 4052, 4071, 4075, 4075, 4078, 4079, 4080, 4105, 4106, 4109, 4109, 4125, 4126, 4129, 4129, 4142, 4143, 4144 |
| `src/Render/Effect_Tick.cpp` | 12 | 40, 48, 66, 109, 110, 135, 144, 153, 154, 162, 163, 245 |
| `src/Render/Entity_AnimationLegacy.cpp` | 25 | 56, 57, 58, 72, 76, 77, 80, 81, 87, 90, 97, 101, 168, 173, 181, 243, 249, 250, 251, 263, 273, 284, 287, 295, 345 |
| `src/Render/Entity_DrawAt.cpp` | 20 | 110, 118, 119, 120, 121, 122, 123, 124, 137, 138, 140, 141, 143, 147, 148, 150, 151, 153, 157, 165 |
| `src/Render/Entity_DrawByType.cpp` | 105 | 181, 252, 253, 254, 260, 261, 262, 268, 269, 270, 289, 290, 291, 292, 298, 299, 300, 301, 341, 342, 343, 344, 345, 346, 347, 353, 354, 355, 356, 360, 361, 362, 363, 364, 365, 366, 372, 373, 374, 375, 414, 415, 416, 430, 431, 432, 435, 436, 437, 441, 442, 443, 452, 453, 454, 458, 459, 460, 461, 527, 528, 529, 534, 535, 536, 548, 549, 550, 551, 557, 558, 559, 560, 562, 563, 564, 571, 572, 573, 579, 580, 581, 603, 604, 605, 606, 612, 613, 614, 615, 660, 670, 671, 672, 678, 679, 680, 697, 698, 699, 705, 706, 707, 745, 751 |
| `src/Render/Entity_DrawSetup.cpp` | 65 | 50, 54, 155, 156, 157, 158, 162, 166, 167, 168, 171, 172, 173, 176, 177, 178, 181, 182, 183, 194, 195, 196, 209, 210, 211, 220, 221, 222, 230, 231, 232, 328, 329, 330, 334, 335, 336, 346, 358, 359, 360, 370, 371, 372, 378, 379, 380, 387, 388, 389, 396, 397, 398, 407, 408, 409, 418, 419, 420, 432, 433, 434, 454, 455, 456 |
| `src/Render/Entity_PrepareRender.cpp` | 1 | 46 |
| `src/Render/Entity_Render.cpp` | 30 | 189, 189, 190, 191, 366, 368, 371, 379, 381, 405, 414, 415, 416, 417, 418, 419, 420, 430, 431, 434, 435, 437, 440, 441, 444, 445, 447, 450, 457, 461 |
| `src/Render/Entity_Render_3D.cpp` | 66 | 179, 197, 200, 201, 205, 207, 209, 210, 215, 217, 219, 220, 230, 236, 236, 236, 245, 246, 247, 247, 251, 307, 307, 307, 322, 331, 331, 353, 354, 354, 364, 364, 364, 382, 382, 382, 411, 412, 459, 459, 460, 468, 470, 470, 470, 471, 480, 481, 483, 485, 491, 508, 515, 529, 530, 539, 552, 568, 602, 605, 607, 616, 616, 619, 621, 622 |
| `src/Render/Entity_UpdateRender.cpp` | 75 | 82, 84, 99, 105, 105, 105, 108, 114, 152, 153, 178, 181, 194, 198, 210, 212, 218, 227, 240, 252, 256, 280, 354, 422, 423, 425, 432, 437, 445, 451, 470, 472, 478, 499, 504, 557, 587, 587, 591, 591, 602, 602, 603, 616, 617, 637, 651, 652, 693, 700, 700, 714, 716, 716, 717, 718, 719, 726, 726, 728, 733, 733, 737, 742, 742, 747, 747, 756, 756, 762, 764, 764, 765, 766, 767 |
| `src/Render/EquipmentVisuals.cpp` | 6 | 46, 93, 104, 209, 215, 221 |
| `src/Render/Font_LegacyFactory.cpp` | 1 | 71 |
| `src/Render/GL_LegacyState.cpp` | 11 | 93, 94, 95, 96, 97, 107, 107, 110, 110, 117, 120 |
| `src/Render/HUD_Pass1.cpp` | 20 | 165, 166, 178, 180, 181, 201, 240, 257, 258, 276, 277, 281, 284, 296, 346, 445, 445, 446, 446, 488 |
| `src/Render/HUD_Pass2.cpp` | 27 | 182, 253, 254, 257, 260, 262, 275, 337, 343, 369, 388, 389, 450, 474, 476, 481, 499, 503, 556, 570, 576, 576, 620, 620, 661, 665, 675 |
| `src/Render/HUD_Pass3.cpp` | 24 | 36, 181, 286, 303, 346, 347, 352, 363, 363, 364, 364, 531, 539, 626, 627, 658, 658, 659, 659, 660, 660, 661, 661, 664 |
| `src/Render/HUD_Pass4.cpp` | 54 | 186, 216, 240, 240, 250, 262, 276, 283, 328, 328, 328, 328, 329, 329, 329, 329, 339, 339, 339, 339, 340, 340, 340, 340, 352, 352, 352, 352, 353, 353, 353, 353, 357, 357, 368, 368, 368, 368, 369, 369, 369, 369, 373, 373, 479, 480, 485, 528, 533, 576, 577, 577, 733, 734 |
| `src/Render/HUD_Pass5.cpp` | 17 | 177, 209, 221, 222, 256, 257, 260, 261, 353, 372, 375, 376, 376, 386, 386, 411, 412 |
| `src/Render/HUD_Pass6.cpp` | 136 | 82, 126, 147, 147, 148, 148, 153, 153, 154, 154, 272, 272, 345, 348, 361, 367, 373, 400, 407, 428, 429, 434, 447, 447, 466, 466, 525, 525, 555, 633, 633, 653, 653, 682, 682, 797, 811, 811, 815, 818, 830, 832, 853, 854, 857, 868, 871, 879, 896, 898, 914, 924, 950, 953, 1017, 1027, 1030, 1054, 1060, 1065, 1068, 1072, 1073, 1076, 1081, 1084, 1088, 1089, 1092, 1102, 1116, 1120, 1128, 1132, 1169, 1203, 1204, 1242, 1252, 1253, 1331, 1337, 1340, 1368, 1368, 1370, 1370, 1372, 1372, 1375, 1391, 1392, 1392, 1393, 1393, 1393, 1394, 1394, 1394, 1394, 1435, 1440, 1453, 1453, 1489, 1499, 1500, 1506, 1551, 1562, 1565, 1569, 1577, 1592, 1640, 1640, 1666, 1680, 1705, 1706, 1727, 1727, 1727, 1728, 1776, 1776, 1831, 1831, 1834, 1834, 1850, 1850, 1851, 1859, 1873, 1878 |
| `src/Render/ItemDrop_Render2.cpp` | 27 | 59, 86, 88, 103, 105, 113, 114, 115, 125, 125, 127, 128, 139, 141, 157, 158, 170, 174, 174, 175, 194, 197, 236, 254, 254, 256, 257 |
| `src/Render/Joint_Create.cpp` | 146 | 235, 347, 388, 486, 531, 535, 535, 549, 553, 553, 565, 567, 581, 594, 594, 613, 617, 617, 620, 622, 743, 774, 774, 788, 792, 792, 802, 804, 818, 831, 831, 850, 854, 854, 857, 859, 895, 898, 931, 932, 989, 1039, 1092, 1093, 1095, 1098, 1099, 1106, 1107, 1109, 1112, 1113, 1120, 1227, 1253, 1256, 1278, 1304, 1307, 1495, 1506, 1512, 1515, 1518, 1643, 1646, 1668, 1671, 1674, 1690, 1737, 1742, 1746, 1951, 1961, 1964, 2009, 2070, 2072, 2091, 2091, 2104, 2108, 2108, 2135, 2136, 2136, 2138, 2140, 2141, 2144, 2183, 2183, 2196, 2200, 2200, 2204, 2205, 2210, 2212, 2217, 2219, 2237, 2237, 2250, 2254, 2254, 2271, 2279, 2279, 2285, 2326, 2328, 2342, 2346, 2346, 2359, 2363, 2363, 2390, 2391, 2391, 2393, 2395, 2396, 2399, 2448, 2448, 2461, 2465, 2465, 2469, 2470, 2475, 2477, 2482, 2484, 2502, 2502, 2515, 2519, 2519, 2536, 2544, 2544, 2550 |
| `src/Render/LooseWingVisuals.cpp` | 5 | 102, 108, 108, 122, 127 |
| `src/Render/MaterialVisuals.cpp` | 2 | 27, 88 |
| `src/Render/MoveEffect.cpp` | 146 | 147, 161, 198, 237, 423, 426, 433, 471, 610, 643, 648, 665, 683, 684, 685, 707, 717, 725, 725, 734, 734, 738, 767, 767, 770, 782, 803, 811, 812, 814, 817, 830, 833, 834, 903, 906, 924, 926, 928, 1015, 1015, 1049, 1049, 1062, 1066, 1069, 1069, 1077, 1129, 1157, 1162, 1203, 1322, 1354, 1362, 1383, 1461, 1482, 1491, 1491, 1515, 1515, 1518, 1518, 1522, 1524, 1533, 1546, 1546, 1549, 1549, 1553, 1555, 1559, 1561, 1572, 1572, 1575, 1575, 1579, 1581, 1585, 1599, 1607, 1629, 1631, 1638, 1684, 1724, 1732, 1735, 1738, 1751, 1755, 1759, 1795, 1856, 1921, 1929, 1932, 1935, 1959, 1962, 1974, 1977, 1977, 1985, 1986, 2010, 2011, 2015, 2016, 2032, 2034, 2035, 2044, 2044, 2055, 2057, 2058, 2059, 2061, 2065, 2095, 2099, 2100, 2108, 2112, 2113, 2114, 2145, 2153, 2154, 2157, 2158, 2188, 2251, 2323, 2323, 2426, 2427, 2450, 2476, 2477, 2477, 2478 |
| `src/Render/MoveEffect_Helpers.cpp` | 2 | 238, 261 |
| `src/Render/MoveJoint.cpp` | 28 | 192, 332, 359, 363, 366, 426, 474, 564, 684, 700, 702, 773, 818, 908, 1134, 1192, 1203, 1232, 1244, 1253, 1399, 1426, 1427, 1609, 1628, 1630, 1632, 1633 |
| `src/Render/Particle.cpp` | 2 | 123, 126 |
| `src/Render/Particle_Legacy.cpp` | 1 | 23 |
| `src/Render/Particle_Move.cpp` | 43 | 228, 232, 241, 274, 275, 279, 280, 284, 285, 301, 302, 315, 333, 340, 343, 354, 357, 379, 380, 383, 384, 387, 388, 397, 398, 453, 454, 457, 458, 461, 462, 471, 472, 583, 584, 596, 687, 688, 728, 867, 868, 871, 953 |
| `src/Render/Particle_Spawn.cpp` | 30 | 71, 367, 371, 379, 383, 408, 422, 440, 442, 444, 452, 465, 574, 579, 591, 625, 627, 634, 641, 664, 680, 701, 832, 835, 840, 842, 846, 847, 850, 859 |
| `src/Render/Player_Render.cpp` | 2 | 93, 97 |
| `src/Render/RenderLinkObject.cpp` | 18 | 125, 140, 144, 302, 577, 578, 588, 598, 599, 600, 601, 608, 609, 610, 611, 625, 701, 703 |
| `src/Render/Render_Frame.cpp` | 2 | 571, 575 |
| `src/Render/Render_LegacyBillboards.cpp` | 31 | 60, 61, 62, 64, 66, 135, 136, 136, 139, 183, 184, 184, 187, 228, 230, 238, 240, 243, 245, 249, 251, 270, 272, 287, 290, 305, 316, 339, 340, 347, 372 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 1 | 102 |
| `src/Render/Render_LegacyLinker.cpp` | 3 | 353, 353, 376 |
| `src/Render/Render_LegacyTrails.cpp` | 2 | 56, 79 |
| `src/Render/Render_LegacyTransforms.cpp` | 3 | 162, 162, 168 |
| `src/Render/Render_PlayerEquipment.cpp` | 9 | 72, 221, 490, 534, 588, 684, 708, 713, 807 |
| `src/Render/Render_WorldHelpers.cpp` | 35 | 124, 126, 223, 231, 247, 268, 291, 343, 443, 468, 689, 698, 699, 706, 707, 714, 750, 764, 765, 808, 814, 901, 904, 915, 916, 979, 983, 1003, 1016, 1028, 1042, 1058, 1173, 1197, 1220 |
| `src/Render/SMD_Parser.cpp` | 21 | 132, 216, 249, 308, 308, 348, 363, 369, 381, 409, 423, 443, 512, 513, 516, 525, 636, 637, 786, 803, 921 |
| `src/Render/Scene_CharPreview.cpp` | 3 | 31, 32, 50 |
| `src/Render/Sprite.cpp` | 4 | 145, 146, 149, 149 |
| `src/Render/Terrain_Render.cpp` | 2 | 94, 98 |
| `src/Render/Terrain_Water.cpp` | 13 | 50, 54, 70, 73, 81, 82, 86, 87, 93, 94, 96, 97, 104 |
| `src/Render/Texture/Texture.cpp` | 11 | 184, 184, 184, 230, 486, 614, 616, 620, 627, 629, 632 |
| `src/Render/Weather.cpp` | 74 | 172, 174, 174, 182, 184, 184, 203, 221, 223, 223, 231, 233, 233, 254, 261, 264, 281, 284, 294, 295, 335, 348, 463, 464, 465, 467, 469, 477, 479, 481, 483, 513, 518, 527, 528, 529, 538, 539, 540, 575, 587, 589, 591, 610, 648, 656, 656, 664, 667, 670, 673, 676, 679, 682, 696, 697, 713, 721, 725, 728, 730, 731, 742, 743, 746, 747, 751, 752, 756, 757, 762, 763, 789, 794 |
| `src/Render/Weather_Particles.cpp` | 77 | 63, 64, 64, 71, 72, 114, 115, 118, 140, 141, 142, 147, 147, 148, 149, 149, 150, 151, 151, 152, 153, 153, 154, 155, 155, 156, 161, 185, 186, 187, 202, 202, 203, 206, 206, 206, 207, 207, 207, 208, 208, 208, 209, 209, 209, 210, 210, 210, 226, 227, 255, 255, 256, 257, 257, 258, 259, 259, 260, 266, 266, 267, 268, 268, 269, 270, 270, 271, 290, 290, 291, 292, 292, 293, 294, 294, 295 |
| `src/Render/WorldEffects.cpp` | 1 | 64 |
| `src/Scene/Scene_AssetLoad.cpp` | 1 | 156 |
| `src/Scene/Scene_CharSelect.cpp` | 51 | 200, 235, 292, 293, 293, 293, 376, 383, 392, 410, 414, 425, 428, 432, 438, 441, 445, 449, 480, 483, 488, 493, 495, 501, 508, 512, 515, 518, 523, 527, 531, 533, 536, 540, 615, 617, 632, 633, 634, 634, 634, 634, 637, 640, 642, 644, 644, 645, 647, 664, 666 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 195 | 70, 70, 80, 80, 82, 92, 92, 94, 94, 97, 99, 102, 114, 114, 114, 115, 118, 118, 119, 122, 123, 123, 131, 137, 137, 137, 141, 143, 148, 149, 153, 158, 170, 172, 172, 172, 172, 173, 179, 180, 183, 184, 185, 189, 190, 191, 195, 199, 202, 203, 204, 219, 321, 448, 478, 568, 569, 569, 570, 583, 583, 588, 588, 597, 598, 603, 610, 618, 622, 622, 624, 624, 625, 627, 628, 633, 633, 634, 637, 662, 805, 805, 834, 841, 871, 871, 1148, 1200, 1214, 1223, 1223, 1226, 1226, 1231, 1263, 1272, 1273, 1276, 1276, 1277, 1280, 1281, 1281, 1283, 1290, 1313, 1314, 1314, 1322, 1330, 1416, 1428, 1430, 1482, 1483, 1483, 1483, 1486, 1487, 1487, 1488, 1499, 1499, 1500, 1557, 1570, 1581, 1583, 1589, 1591, 1613, 1615, 1620, 1623, 1624, 1625, 1626, 1634, 1640, 1642, 1648, 1653, 1658, 1662, 1666, 1676, 1681, 1687, 1689, 1692, 1694, 1723, 1735, 1751, 1762, 1770, 1774, 1779, 1782, 1782, 1843, 1856, 1857, 1857, 2039, 2127, 2128, 2138, 2139, 2163, 2164, 2173, 2174, 2177, 2220, 2247, 2248, 2249, 2292, 2293, 2294, 2295, 2296, 2297, 2298, 2313, 2339, 2366, 2383, 2396, 2397, 2410, 2445, 2462, 2474 |
| `src/Scene/Scene_Dispatch.cpp` | 6 | 139, 146, 150, 166, 173, 177 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 15 | 47, 48, 246, 246, 259, 298, 298, 299, 299, 300, 300, 303, 303, 304, 304 |
| `src/Scene/Scene_Intro.cpp` | 11 | 192, 242, 242, 253, 253, 270, 288, 288, 297, 297, 319 |
| `src/Scene/Scene_LegacyHelpers.cpp` | 1 | 32 |
| `src/Scene/Scene_Login.cpp` | 9 | 105, 137, 151, 185, 186, 216, 230, 257, 282 |
| `src/Scene/Scene_Login_ServerSelect.cpp` | 52 | 212, 212, 229, 235, 235, 263, 263, 328, 333, 333, 349, 349, 352, 352, 362, 362, 369, 370, 380, 380, 383, 383, 393, 393, 396, 397, 422, 426, 429, 429, 431, 431, 461, 467, 468, 478, 480, 487, 487, 497, 499, 500, 507, 508, 508, 517, 522, 522, 525, 525, 532, 533 |
| `src/Scene/Scene_MapTick.cpp` | 12 | 42, 50, 50, 50, 55, 58, 65, 68, 74, 74, 77, 77 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 3 | 41, 75, 94 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 33 | 31, 85, 112, 431, 480, 571, 572, 573, 575, 608, 627, 661, 665, 669, 675, 739, 767, 791, 794, 825, 826, 854, 855, 868, 869, 873, 874, 878, 879, 883, 957, 957, 996 |
| `src/Scene/Scene_Objects.cpp` | 1 | 396 |
| `src/Scene/Scene_OpenWorld.cpp` | 3 | 114, 115, 115 |
| `src/Scene/Scene_ServerSelect_Input.cpp` | 20 | 117, 117, 118, 118, 147, 147, 150, 153, 164, 168, 171, 184, 184, 185, 185, 190, 202, 203, 215, 237 |
| `src/Sound/Sound_DS3D.cpp` | 3 | 156, 157, 169 |
| `src/Sound/Sound_Queue.cpp` | 2 | 59, 111 |
| `src/Terrain/Terrain.cpp` | 13 | 414, 415, 432, 433, 434, 438, 446, 465, 477, 477, 504, 582, 613 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 152 | 63, 72, 123, 173, 181, 183, 187, 196, 206, 260, 261, 265, 266, 267, 270, 273, 274, 275, 276, 277, 278, 279, 282, 286, 287, 288, 289, 290, 291, 292, 293, 298, 302, 308, 309, 310, 311, 312, 313, 438, 439, 440, 441, 442, 443, 444, 445, 446, 447, 448, 449, 450, 451, 452, 453, 454, 455, 456, 459, 460, 461, 462, 463, 464, 465, 466, 467, 468, 469, 470, 472, 473, 474, 475, 476, 477, 478, 479, 480, 481, 482, 483, 484, 485, 486, 487, 488, 489, 490, 491, 492, 493, 494, 495, 496, 497, 498, 499, 500, 501, 502, 503, 504, 505, 506, 507, 508, 509, 510, 511, 512, 513, 514, 515, 516, 517, 518, 519, 520, 521, 522, 523, 524, 525, 526, 527, 528, 529, 530, 531, 532, 533, 534, 535, 536, 537, 538, 539, 540, 541, 542, 543, 544, 545, 546, 547, 548, 549, 550, 551, 552, 553 |
| `src/Terrain/Terrain_LegacyRender.cpp` | 3 | 45, 46, 155 |
| `src/Terrain/Terrain_Light.cpp` | 17 | 43, 45, 183, 187, 197, 200, 204, 204, 205, 206, 210, 210, 211, 212, 215, 220, 221 |
| `src/Terrain/Terrain_RayCollision.cpp` | 4 | 75, 75, 75, 75 |
| `src/Terrain/Terrain_RenderBlocks.cpp` | 12 | 28, 29, 52, 52, 93, 110, 111, 114, 115, 117, 126, 131 |
| `src/Terrain/Terrain_RenderFace.cpp` | 5 | 52, 52, 52, 194, 260 |
| `src/Terrain/Terrain_Utils.cpp` | 3 | 38, 42, 43 |
| `src/Text/TextAtlas.cpp` | 2 | 35, 117 |
| `src/Trade/Trade.cpp` | 3 | 144, 153, 339 |
| `src/UI/CharMenu_Build.cpp` | 18 | 80, 274, 278, 283, 328, 367, 369, 398, 402, 405, 427, 429, 443, 448, 448, 449, 570, 627 |
| `src/UI/Chat.cpp` | 36 | 72, 75, 75, 75, 119, 184, 184, 218, 251, 265, 319, 338, 376, 383, 395, 434, 494, 504, 504, 513, 513, 557, 578, 578, 580, 580, 585, 619, 621, 622, 634, 658, 660, 661, 673, 740 |
| `src/UI/ChatListBox.cpp` | 308 | 110, 110, 111, 111, 113, 113, 114, 114, 118, 118, 118, 118, 118, 241, 242, 262, 277, 281, 348, 360, 363, 450, 463, 483, 485, 493, 644, 644, 645, 645, 659, 675, 681, 681, 682, 683, 684, 693, 693, 694, 697, 700, 706, 706, 707, 710, 713, 726, 726, 727, 732, 733, 733, 734, 734, 754, 772, 801, 802, 818, 819, 832, 848, 850, 856, 860, 872, 881, 882, 889, 890, 909, 913, 916, 925, 935, 944, 953, 958, 964, 968, 972, 973, 974, 976, 1010, 1017, 1023, 1023, 1023, 1024, 1031, 1031, 1032, 1069, 1070, 1072, 1074, 1075, 1083, 1108, 1108, 1109, 1109, 1111, 1111, 1112, 1114, 1114, 1115, 1116, 1117, 1129, 1129, 1135, 1155, 1185, 1185, 1215, 1215, 1220, 1247, 1247, 1256, 1256, 1263, 1264, 1286, 1311, 1428, 1429, 1445, 1455, 1455, 1465, 1465, 1466, 1468, 1484, 1484, 1484, 1485, 1507, 1507, 1509, 1526, 1562, 1606, 1607, 1608, 1609, 1620, 1638, 1639, 1640, 1641, 1651, 1654, 1655, 1655, 1657, 1754, 1755, 1765, 1777, 1778, 1780, 1781, 1783, 1784, 1786, 1787, 1788, 1789, 1800, 1800, 1800, 1800, 1811, 1811, 1814, 1815, 1815, 1816, 1816, 1819, 1820, 1820, 1826, 1826, 1829, 1829, 1843, 1847, 1848, 1848, 1850, 1889, 1889, 1897, 1897, 1901, 1901, 1902, 1902, 1914, 1914, 1937, 1941, 1950, 1951, 1951, 1953, 1955, 1955, 1957, 1959, 1963, 1981, 1997, 2014, 2015, 2016, 2017, 2025, 2027, 2028, 2028, 2029, 2029, 2088, 2110, 2114, 2123, 2129, 2131, 2132, 2236, 2236, 2242, 2242, 2243, 2243, 2246, 2263, 2266, 2267, 2271, 2272, 2273, 2290, 2291, 2292, 2293, 2294, 2296, 2298, 2299, 2299, 2300, 2300, 2344, 2345, 2346, 2347, 2348, 2357, 2358, 2368, 2370, 2370, 2377, 2405, 2450, 2452, 2462, 2465, 2478, 2480, 2625, 2636, 2647, 2647, 2650, 2657, 2684, 2688, 2695, 2698, 2703, 2707, 2714, 2717, 2735, 2736, 2743, 2748, 2757, 2758, 2761, 2762, 2763, 2770, 2775, 2788, 2789, 2810, 2818 |
| `src/UI/Chat_Bubbles.cpp` | 12 | 69, 73, 73, 94, 95, 96, 102, 149, 156, 157, 203, 238 |
| `src/UI/Chat_Gestures.cpp` | 2 | 80, 80 |
| `src/UI/Chat_InputTick.cpp` | 30 | 151, 158, 175, 244, 247, 329, 330, 356, 356, 362, 362, 374, 377, 447, 456, 499, 514, 516, 520, 547, 552, 555, 649, 650, 705, 731, 732, 733, 876, 881 |
| `src/UI/Chat_Send.cpp` | 9 | 38, 41, 53, 55, 72, 81, 90, 113, 116 |
| `src/UI/EmojiPicker.cpp` | 3 | 70, 71, 123 |
| `src/UI/EventTimer.cpp` | 6 | 47, 149, 151, 152, 152, 169 |
| `src/UI/GoldenArcher.cpp` | 32 | 51, 52, 68, 69, 104, 215, 215, 226, 226, 229, 229, 234, 234, 241, 241, 245, 245, 308, 481, 481, 481, 481, 490, 490, 490, 603, 607, 607, 607, 929, 947, 1076 |
| `src/UI/HealthBar.cpp` | 2 | 26, 79 |
| `src/UI/MiniMap.cpp` | 16 | 54, 54, 54, 54, 119, 119, 119, 119, 122, 127, 127, 136, 320, 333, 334, 335 |
| `src/UI/MoveList.cpp` | 6 | 33, 33, 39, 113, 114, 204 |
| `src/UI/OptionsMenu.cpp` | 12 | 42, 91, 91, 91, 91, 103, 103, 104, 178, 178, 518, 518 |
| `src/UI/Quest_Legacy.cpp` | 25 | 32, 35, 77, 101, 101, 102, 117, 119, 148, 167, 167, 174, 174, 187, 196, 196, 197, 197, 204, 204, 209, 217, 229, 266, 275 |
| `src/UI/RenderItemInfo.cpp` | 63 | 114, 114, 261, 434, 535, 555, 566, 818, 821, 824, 827, 831, 834, 837, 838, 838, 842, 905, 1073, 1076, 1159, 1160, 1161, 1163, 1658, 1659, 1685, 1726, 1767, 1767, 1807, 1834, 1851, 1915, 1922, 1965, 1966, 1974, 1986, 2000, 2014, 2028, 2029, 2039, 2040, 2085, 2085, 2087, 2088, 2090, 2121, 2148, 2157, 2158, 2170, 2170, 2175, 2223, 2223, 2245, 2245, 2248, 2250 |
| `src/UI/UI_DialogInterface.cpp` | 1 | 51 |
| `src/UI/UI_GuildLegacy.cpp` | 17 | 105, 143, 143, 148, 148, 153, 161, 161, 170, 170, 178, 178, 179, 210, 223, 235, 235 |
| `src/UI/UI_GuildMark.cpp` | 5 | 15, 16, 81, 82, 141 |
| `src/UI/UI_IME.cpp` | 3 | 13, 13, 77 |
| `src/UI/UI_InGameMenu.cpp` | 29 | 73, 92, 93, 114, 115, 138, 159, 163, 384, 385, 386, 388, 440, 466, 472, 475, 479, 592, 600, 637, 640, 641, 644, 709, 711, 742, 921, 975, 976 |
| `src/UI/UI_LegacyExterns.cpp` | 56 | 107, 119, 119, 156, 165, 203, 204, 205, 206, 210, 289, 313, 314, 324, 327, 342, 445, 491, 499, 506, 512, 522, 542, 543, 545, 547, 549, 551, 615, 618, 676, 679, 680, 681, 682, 683, 685, 687, 704, 739, 739, 781, 782, 821, 929, 936, 940, 947, 1003, 1032, 1032, 1038, 1065, 1065, 1089, 1090 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 7 | 63, 65, 69, 70, 71, 98, 133 |
| `src/UI/UI_LegacyText.cpp` | 3 | 93, 93, 93 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 9 | 148, 149, 158, 235, 351, 360, 368, 377, 388 |
| `src/UI/UI_LegacyWidgets.cpp` | 2 | 264, 266 |
| `src/UI/UI_SkillHotkeys.cpp` | 9 | 40, 86, 87, 88, 89, 94, 95, 96, 114 |
| `src/UI/UI_StatsPanel.cpp` | 82 | 49, 62, 64, 67, 68, 75, 83, 89, 94, 99, 110, 115, 126, 133, 164, 166, 170, 240, 242, 245, 266, 280, 280, 283, 308, 310, 313, 427, 427, 507, 507, 570, 570, 582, 582, 635, 635, 649, 658, 658, 675, 679, 679, 681, 706, 706, 726, 726, 740, 740, 798, 803, 810, 812, 812, 813, 814, 828, 836, 845, 874, 878, 880, 888, 894, 894, 896, 897, 905, 921, 925, 925, 927, 938, 938, 946, 946, 948, 951, 953, 953, 955 |
| `src/UI/UI_TextBitmap.cpp` | 16 | 56, 58, 118, 135, 228, 233, 233, 235, 235, 255, 278, 278, 279, 279, 301, 305 |
| `src/Util/Misc.cpp` | 8 | 48, 77, 100, 101, 131, 136, 187, 189 |
| `src/functions.h` | 17 | 70, 85, 92, 202, 203, 207, 318, 332, 333, 334, 386, 768, 770, 773, 813, 814, 817 |
| `src/globals.cpp` | 4 | 520, 521, 522, 523 |
| `src/globals.h` | 3 | 1271, 1272, 2444 |
| `src/types.h` | 10 | 33, 34, 34, 35, 36, 37, 37, 38, 39, 39 |

### 1b uintptr_t/intptr_t (revisar cadena completa)

Expresión (Python re):

```text
\b(?:uintptr_t|intptr_t)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 57 | 466, 515, 920, 931, 945, 948, 948, 959, 959, 968, 975, 988, 995, 1018, 1042, 1043, 1065, 1102, 1122, 1130, 1152, 1165, 1200, 1207, 1233, 1234, 1249, 1255, 1263, 1321, 1326, 1381, 1394, 1408, 1419, 1563, 1579, 1590, 1591, 1597, 1598, 1843, 1844, 1845, 1858, 1917, 1962, 2009, 2033, 2074, 2167, 2177, 2342, 2353, 2365, 2374, 2397 |
| `src/Combat/Combat_AttackEffect.cpp` | 17 | 50, 60, 66, 66, 67, 67, 70, 70, 71, 78, 83, 85, 86, 86, 87, 89, 91 |
| `src/Combat/Combat_AttackStage.cpp` | 9 | 181, 268, 269, 353, 357, 359, 363, 363, 367 |
| `src/Combat/Combat_LegacyAnimation.cpp` | 2 | 113, 115 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 2 | 186, 247 |
| `src/Combat/Combat_Projectiles.cpp` | 39 | 77, 78, 78, 83, 84, 84, 89, 90, 90, 95, 96, 96, 101, 102, 102, 107, 108, 108, 113, 114, 114, 119, 120, 120, 125, 126, 126, 142, 143, 143, 149, 150, 150, 156, 157, 157, 163, 164, 164 |
| `src/Combat/Skills.cpp` | 24 | 75, 81, 94, 96, 212, 223, 230, 231, 237, 276, 283, 290, 298, 307, 323, 331, 344, 351, 357, 370, 381, 392, 397, 405 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 12 | 40, 42, 86, 89, 155, 156, 157, 205, 269, 270, 271, 370 |
| `src/Combat/Skills_WizardElf.cpp` | 7 | 74, 74, 75, 77, 113, 170, 459 |
| `src/Core/Runtime_Externs.cpp` | 1 | 64 |
| `src/Core/Runtime_Linker.cpp` | 1 | 76 |
| `src/Core/Runtime_Medium.cpp` | 1 | 281 |
| `src/Core/Runtime_Small.cpp` | 8 | 267, 271, 272, 276, 279, 280, 281, 329 |
| `src/Core/Window.cpp` | 1 | 35 |
| `src/Entity/CharacterCloth.cpp` | 1 | 24 |
| `src/Entity/EntityName.cpp` | 5 | 26, 26, 27, 27, 29 |
| `src/Entity/EntityName.h` | 1 | 21 |
| `src/Entity/EntityTypes.h` | 2 | 246, 249 |
| `src/Entity/Entity_Init.cpp` | 1 | 73 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 3 | 44, 86, 104 |
| `src/Entity/Entity_MoveClient.cpp` | 1 | 27 |
| `src/Entity/ObjectPose.cpp` | 18 | 350, 355, 371, 375, 390, 402, 413, 414, 416, 416, 454, 454, 463, 464, 475, 496, 502, 503 |
| `src/Game/FramePointerSet.h` | 1 | 14 |
| `src/Game/Game_CharSelectTick.cpp` | 5 | 130, 175, 179, 221, 222 |
| `src/Game/Game_EnterWorldTick.cpp` | 3 | 302, 648, 670 |
| `src/Game/Game_MainLoop.cpp` | 3 | 411, 442, 469 |
| `src/Game/Game_SceneUpdate.cpp` | 3 | 374, 466, 640 |
| `src/Game/HeroVitals.cpp` | 1 | 13 |
| `src/Game/MapManager.cpp` | 3 | 141, 148, 156 |
| `src/Game/Party_NameMatch.cpp` | 1 | 25 |
| `src/Game/PathFinder.cpp` | 3 | 442, 797, 804 |
| `src/Game/Player_InputTick.cpp` | 10 | 372, 821, 1235, 1236, 1317, 1318, 1322, 1336, 1337, 1473 |
| `src/Game/RenderFxSnapshots.cpp` | 7 | 104, 104, 107, 107, 150, 150, 150 |
| `src/Game/RenderInterpolation.cpp` | 9 | 15, 15, 15, 123, 123, 123, 179, 179, 179 |
| `src/Game/RenderSnapshots.cpp` | 9 | 41, 106, 106, 106, 108, 132, 163, 190, 332 |
| `src/Game/SimulationPreparation.cpp` | 2 | 142, 142 |
| `src/Input/Input.cpp` | 1 | 407 |
| `src/Input/Input_Timing.cpp` | 2 | 40, 40 |
| `src/Input/Mouse_Hover.cpp` | 1 | 603 |
| `src/Item/ContentCatalog.cpp` | 13 | 87, 87, 88, 88, 90, 118, 124, 125, 126, 156, 225, 586, 658 |
| `src/Item/Item_ClickHandler.cpp` | 22 | 160, 177, 399, 503, 569, 633, 653, 709, 719, 799, 866, 922, 958, 973, 984, 997, 1003, 1010, 1023, 1024, 1069, 1100 |
| `src/Item/Item_Display.cpp` | 6 | 29, 30, 83, 97, 260, 355 |
| `src/Item/Item_Durability.cpp` | 4 | 50, 52, 54, 108 |
| `src/Item/Item_Equip.cpp` | 9 | 24, 69, 70, 114, 139, 140, 176, 177, 222 |
| `src/Item/Item_EquipmentAutoSwap.cpp` | 2 | 34, 35 |
| `src/Item/Item_EquipmentQueries.cpp` | 2 | 27, 28 |
| `src/Item/Item_Inventory.cpp` | 8 | 50, 197, 209, 231, 240, 295, 345, 395 |
| `src/Item/Item_InventoryGrid.cpp` | 4 | 214, 219, 231, 365 |
| `src/Item/Item_LegacyGridInsert.cpp` | 1 | 76 |
| `src/Item/Item_LegacyLinker.cpp` | 1 | 217 |
| `src/Item/Item_Move.cpp` | 2 | 60, 61 |
| `src/Item/Item_ServerValue.cpp` | 1 | 161 |
| `src/Item/Item_SpecialOption.cpp` | 1 | 17 |
| `src/Item/RightClickMove.cpp` | 2 | 23, 45 |
| `src/Model/ModelTypes.h` | 1 | 82 |
| `src/Model/Model_Items.cpp` | 3 | 27, 321, 329 |
| `src/Model/Model_Players.cpp` | 10 | 349, 349, 350, 350, 351, 351, 352, 352, 353, 353 |
| `src/Monster/Monster.cpp` | 8 | 638, 653, 1069, 1074, 1102, 1114, 1117, 1122 |
| `src/Net/Crypto.cpp` | 6 | 401, 1161, 1162, 1163, 1164, 1352 |
| `src/Net/MuEmu.cpp` | 1 | 171 |
| `src/Net/Net_Connect.cpp` | 2 | 45, 47 |
| `src/Net/Net_Process.cpp` | 1 | 664 |
| `src/Net/Network.cpp` | 8 | 76, 87, 254, 267, 270, 304, 325, 328 |
| `src/Net/Reconnect.cpp` | 3 | 73, 76, 76 |
| `src/Net/Recv/Recv_Account.cpp` | 2 | 206, 309 |
| `src/Net/Recv/Recv_Character.cpp` | 30 | 24, 65, 84, 163, 225, 413, 425, 464, 494, 539, 542, 665, 672, 685, 704, 705, 852, 865, 978, 1019, 1201, 1220, 1250, 1443, 1470, 1505, 1635, 1641, 1658, 1694 |
| `src/Net/Recv/Recv_Chat.cpp` | 3 | 33, 33, 37 |
| `src/Net/Recv/Recv_Combat.cpp` | 34 | 16, 22, 32, 36, 39, 47, 51, 59, 60, 61, 62, 63, 84, 86, 91, 92, 171, 210, 221, 246, 367, 468, 519, 590, 645, 648, 661, 683, 800, 804, 817, 844, 886, 897 |
| `src/Net/Recv/Recv_Event.cpp` | 1 | 173 |
| `src/Net/Recv/Recv_Guild.cpp` | 12 | 100, 101, 120, 125, 186, 186, 209, 209, 425, 430, 444, 512 |
| `src/Net/Recv/Recv_Item.cpp` | 18 | 25, 52, 59, 83, 182, 227, 700, 716, 759, 760, 895, 914, 956, 989, 993, 996, 997, 1051 |
| `src/Net/Recv/Recv_Quest.cpp` | 11 | 31, 53, 65, 88, 89, 90, 107, 136, 139, 145, 146 |
| `src/Net/Recv/Recv_Viewport.cpp` | 31 | 72, 83, 94, 107, 171, 178, 190, 207, 243, 248, 268, 331, 357, 360, 390, 411, 429, 430, 430, 443, 446, 450, 454, 559, 598, 713, 761, 831, 833, 868, 908 |
| `src/Net/SecondPassword.cpp` | 110 | 757, 802, 805, 819, 824, 830, 1047, 1448, 1449, 1484, 1601, 1622, 1652, 1656, 1662, 1666, 1670, 1698, 1729, 1730, 1813, 1924, 1927, 2048, 2218, 2221, 2225, 2226, 2274, 2300, 2309, 2452, 2946, 2981, 2984, 2994, 2996, 3003, 3006, 3032, 3112, 3119, 3120, 3435, 3436, 3436, 3442, 3443, 3443, 3463, 3464, 3464, 3465, 3466, 3473, 3474, 3474, 3483, 3484, 3484, 3485, 3486, 3486, 3492, 3493, 3493, 3497, 3498, 3504, 3505, 3505, 3513, 3514, 3514, 3528, 3529, 3541, 3542, 3542, 3617, 3618, 3646, 3647, 3655, 3656, 3660, 3661, 3665, 3666, 3668, 3669, 3677, 3678, 3680, 3681, 3684, 3685, 3715, 3716, 3718, 3719, 3732, 3733, 3737, 3738, 3758, 3759, 4074, 4103, 4549 |
| `src/Net/ServerCharacterStats.cpp` | 3 | 15, 21, 37 |
| `src/Path/Path_Legacy.cpp` | 2 | 304, 326 |
| `src/Platform/ClientEntry.cpp` | 3 | 369, 409, 677 |
| `src/Platform/GL/glad.h` | 2 | 1683, 1684 |
| `src/Platform/ISocket.h` | 1 | 4 |
| `src/Platform/Posix/WinTypes.h` | 5 | 21, 22, 23, 24, 25 |
| `src/Platform/Win32/ClientWndProc.cpp` | 1 | 78 |
| `src/Render/Ambient_Particles.cpp` | 7 | 112, 112, 113, 114, 118, 288, 407 |
| `src/Render/BMD_Anim.cpp` | 3 | 51, 52, 98 |
| `src/Render/BMD_DrawMesh.cpp` | 4 | 152, 152, 251, 251 |
| `src/Render/BMD_FilterData.cpp` | 10 | 33, 33, 65, 70, 70, 139, 139, 169, 173, 173 |
| `src/Render/BMD_LegacyDraw.cpp` | 1 | 1849 |
| `src/Render/BMD_LegacyLoading.cpp` | 3 | 84, 543, 561 |
| `src/Render/Effect_Combat.cpp` | 2 | 76, 82 |
| `src/Render/Effect_Create.cpp` | 60 | 167, 179, 181, 186, 218, 219, 222, 227, 265, 384, 406, 441, 627, 699, 822, 823, 852, 907, 908, 1211, 1333, 1363, 1398, 1499, 1563, 1569, 1707, 1796, 1909, 1957, 1962, 2063, 2127, 2133, 2271, 2320, 2417, 2505, 2607, 2670, 2676, 2814, 2863, 2921, 3066, 3113, 3162, 3263, 3327, 3333, 3461, 3510, 3648, 3732, 3803, 3905, 3965, 3979, 4080, 4144 |
| `src/Render/Effect_Tick.cpp` | 9 | 31, 31, 56, 270, 270, 308, 330, 330, 351 |
| `src/Render/Entity_AnimationLegacy.cpp` | 15 | 192, 199, 206, 208, 209, 211, 211, 230, 263, 376, 378, 379, 380, 380, 385 |
| `src/Render/Entity_Render.cpp` | 2 | 309, 366 |
| `src/Render/Entity_UpdateRender.cpp` | 7 | 47, 67, 346, 375, 451, 451, 499 |
| `src/Render/EquipmentVisuals.cpp` | 4 | 43, 43, 43, 45 |
| `src/Render/Font_Layout.cpp` | 1 | 30 |
| `src/Render/HUD_Pass1.cpp` | 4 | 83, 257, 296, 468 |
| `src/Render/HUD_Pass2.cpp` | 7 | 90, 325, 329, 563, 563, 564, 564 |
| `src/Render/HUD_Pass3.cpp` | 1 | 36 |
| `src/Render/HUD_Pass4.cpp` | 3 | 404, 524, 559 |
| `src/Render/HUD_Pass5.cpp` | 6 | 62, 215, 279, 348, 399, 403 |
| `src/Render/HUD_Pass6.cpp` | 12 | 82, 400, 599, 620, 1102, 1111, 1201, 1337, 1340, 1440, 1551, 1666 |
| `src/Render/JointTypes.h` | 1 | 71 |
| `src/Render/Joint_Create.cpp` | 21 | 82, 82, 85, 567, 581, 622, 804, 818, 859, 2072, 2204, 2205, 2212, 2219, 2271, 2328, 2469, 2470, 2477, 2484, 2536 |
| `src/Render/MoveEffect.cpp` | 7 | 77, 78, 237, 648, 675, 1822, 2114 |
| `src/Render/MoveJoint.cpp` | 8 | 153, 192, 1622, 1625, 1628, 1630, 1632, 1633 |
| `src/Render/Particle_Render.cpp` | 3 | 102, 102, 102 |
| `src/Render/RenderLinkObject.cpp` | 2 | 108, 635 |
| `src/Render/Render_Frame.cpp` | 1 | 409 |
| `src/Render/Render_LegacyLinker.cpp` | 1 | 371 |
| `src/Render/Render_PlayerEquipment.cpp` | 17 | 72, 130, 130, 133, 188, 203, 206, 220, 220, 222, 490, 493, 712, 712, 714, 807, 807 |
| `src/Render/Render_WorldHelpers.cpp` | 20 | 122, 123, 124, 126, 144, 215, 215, 217, 443, 698, 705, 764, 893, 896, 979, 983, 1003, 1013, 1042, 1058 |
| `src/Render/SMD_Parser.cpp` | 18 | 346, 363, 418, 423, 428, 601, 606, 650, 768, 768, 769, 769, 772, 776, 839, 853, 862, 870 |
| `src/Render/Texture/Texture.cpp` | 2 | 691, 743 |
| `src/Render/Weather.cpp` | 8 | 254, 348, 465, 477, 481, 518, 656, 697 |
| `src/Scene/ObjectRegistry.h` | 2 | 12, 13 |
| `src/Scene/Scene_AssetLoad.cpp` | 1 | 156 |
| `src/Scene/Scene_CharSelect.cpp` | 22 | 376, 376, 376, 382, 392, 392, 392, 413, 425, 425, 425, 427, 431, 438, 438, 438, 440, 444, 448, 536, 536, 536 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 40 | 448, 1148, 1566, 1575, 1611, 1639, 1691, 1723, 1726, 1735, 1738, 1875, 1903, 1935, 1951, 1960, 1962, 1965, 2127, 2128, 2133, 2138, 2139, 2151, 2222, 2247, 2248, 2249, 2292, 2293, 2294, 2295, 2296, 2297, 2298, 2315, 2329, 2339, 2368, 2384 |
| `src/Scene/Scene_Dispatch.cpp` | 2 | 150, 177 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 3 | 246, 246, 259 |
| `src/Scene/Scene_Intro.cpp` | 3 | 192, 192, 192 |
| `src/Scene/Scene_Login.cpp` | 3 | 105, 105, 105 |
| `src/Scene/Scene_Login_ServerSelect.cpp` | 5 | 235, 263, 333, 362, 393 |
| `src/Scene/Scene_MapTick.cpp` | 5 | 58, 58, 58, 68, 68 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 7 | 230, 319, 627, 767, 912, 957, 957 |
| `src/Scene/Scene_OpenWorld.cpp` | 2 | 114, 115 |
| `src/Sound/Music.cpp` | 2 | 67, 72 |
| `src/Sound/Sound_DS3D.cpp` | 3 | 156, 157, 169 |
| `src/Terrain/Terrain.cpp` | 1 | 613 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 2 | 206, 641 |
| `src/UI/CharMenu_Build.cpp` | 14 | 105, 327, 367, 367, 369, 369, 427, 427, 429, 429, 478, 479, 628, 629 |
| `src/UI/Chat.cpp` | 9 | 51, 184, 218, 218, 376, 376, 434, 434, 647 |
| `src/UI/ChatListBox.cpp` | 19 | 484, 1526, 1620, 1871, 2014, 2015, 2016, 2017, 2110, 2114, 2123, 2129, 2193, 2201, 2263, 2290, 2291, 2293, 2294 |
| `src/UI/ChatTypes.h` | 1 | 45 |
| `src/UI/Chat_Bubbles.cpp` | 5 | 23, 94, 95, 96, 102 |
| `src/UI/Chat_Gestures.cpp` | 1 | 24 |
| `src/UI/Chat_InputTick.cpp` | 4 | 68, 78, 125, 473 |
| `src/UI/HealthBar.cpp` | 1 | 101 |
| `src/UI/MiniMap.cpp` | 5 | 44, 255, 266, 266, 287 |
| `src/UI/MoveList.cpp` | 2 | 107, 108 |
| `src/UI/Quest_Legacy.cpp` | 8 | 77, 117, 148, 174, 197, 204, 229, 275 |
| `src/UI/RenderItemInfo.cpp` | 25 | 41, 1416, 1426, 1428, 1658, 1685, 1741, 1741, 1743, 1743, 1758, 1767, 1767, 1803, 1901, 1903, 1957, 2085, 2087, 2088, 2090, 2105, 2187, 2248, 2250 |
| `src/UI/UIState.cpp` | 1 | 20 |
| `src/UI/UI_GuildLegacy.cpp` | 3 | 133, 153, 153 |
| `src/UI/UI_InGameMenu.cpp` | 2 | 646, 934 |
| `src/UI/UI_LegacyExterns.cpp` | 6 | 135, 305, 306, 498, 821, 821 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 2 | 66, 72 |
| `src/UI/UI_LegacyText.cpp` | 1 | 122 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 1 | 388 |
| `src/UI/UI_SkillHotkeys.cpp` | 1 | 80 |
| `src/UI/UI_StatsPanel.cpp` | 41 | 49, 49, 62, 266, 266, 283, 283, 385, 386, 427, 507, 570, 582, 635, 658, 675, 675, 679, 681, 681, 706, 726, 740, 810, 822, 845, 845, 874, 878, 878, 894, 896, 897, 897, 905, 905, 924, 927, 938, 946, 953 |
| `src/UI/UI_TextBitmap.cpp` | 1 | 143 |
| `src/Util/Misc.cpp` | 3 | 48, 77, 87 |
| `src/globals.h` | 3 | 1282, 1283, 1284 |
| `src/structs.h` | 2 | 279, 295 |

### 1c aritmética directa sobre DAT_

Expresión (Python re):

```text
\bDAT_[0-9a-fA-F]+\s*(?:\+|\-|\[)
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 10 | 1591, 1598, 1857, 1951, 2009, 2033, 2342, 2353, 2365, 2399 |
| `src/Combat/Combat_AttackEffect.cpp` | 68 | 399, 418, 429, 459, 461, 466, 474, 519, 521, 535, 546, 599, 618, 629, 659, 661, 666, 674, 754, 773, 784, 814, 816, 821, 829, 892, 894, 907, 918, 949, 951, 956, 1042, 1061, 1072, 1102, 1104, 1109, 1117, 1181, 1200, 1211, 1241, 1243, 1248, 1256, 1316, 1335, 1346, 1376, 1378, 1383, 1391, 1522, 1541, 1552, 1582, 1584, 1589, 1597, 1655, 1657, 1670, 1681, 1712, 1714, 1719, 1727 |
| `src/Combat/Combat_AttackStage.cpp` | 2 | 332, 363 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 2 | 75, 258 |
| `src/Combat/Skills.cpp` | 1 | 106 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 6 | 40, 42, 205, 374, 423, 438 |
| `src/Combat/Skills_WizardElf.cpp` | 2 | 22, 459 |
| `src/Entity/CharacterCloth.cpp` | 3 | 32, 116, 191 |
| `src/Entity/CharacterEffects.cpp` | 4 | 31, 115, 120, 123 |
| `src/Entity/Entity_ActionLegacy.cpp` | 1 | 63 |
| `src/Entity/Entity_BoneEffects.cpp` | 2 | 52, 74 |
| `src/Entity/Entity_Init.cpp` | 2 | 29, 73 |
| `src/Entity/Entity_LegacyPoolCleanup.cpp` | 3 | 57, 82, 85 |
| `src/Entity/Entity_Spawn.cpp` | 2 | 43, 200 |
| `src/Entity/ObjectPose.cpp` | 6 | 71, 347, 350, 368, 371, 478 |
| `src/Game/Game_EnterWorldTick.cpp` | 26 | 208, 302, 365, 366, 371, 383, 456, 457, 502, 525, 604, 608, 608, 619, 624, 624, 637, 637, 648, 651, 651, 670, 674, 688, 692, 692 |
| `src/Game/Game_MainLoop.cpp` | 5 | 150, 151, 209, 212, 295 |
| `src/Game/Game_SceneUpdate.cpp` | 14 | 67, 188, 341, 342, 343, 346, 374, 393, 397, 401, 426, 663, 675, 827 |
| `src/Game/Player_InputTick.cpp` | 18 | 127, 128, 135, 136, 430, 623, 644, 648, 660, 664, 821, 1188, 1189, 1235, 1249, 1437, 1472, 1643 |
| `src/Game/RenderDerivedState.cpp` | 4 | 61, 66, 69, 71 |
| `src/Game/RenderInterpolation.cpp` | 4 | 123, 143, 157, 179 |
| `src/Game/RenderSnapshots.cpp` | 3 | 162, 168, 276 |
| `src/Game/SimulationPreparation.cpp` | 3 | 124, 130, 163 |
| `src/Game/Timer.cpp` | 2 | 39, 40 |
| `src/Input/Cursor_Render.cpp` | 7 | 46, 47, 98, 117, 118, 121, 122 |
| `src/Input/Input.cpp` | 4 | 304, 309, 310, 337 |
| `src/Input/Input_WndProc.cpp` | 7 | 182, 186, 201, 278, 289, 290, 357 |
| `src/Input/Mouse_Hover.cpp` | 25 | 67, 77, 77, 93, 94, 122, 162, 191, 242, 243, 244, 247, 248, 249, 250, 256, 304, 306, 348, 453, 486, 545, 556, 592, 600 |
| `src/Item/ContentCatalog.cpp` | 2 | 225, 586 |
| `src/Item/Dialog_Data.cpp` | 1 | 28 |
| `src/Item/Filter_Data.cpp` | 2 | 20, 34 |
| `src/Item/Item_ChaosMix.cpp` | 1 | 363 |
| `src/Item/Item_ClickHandler.cpp` | 30 | 104, 225, 291, 292, 293, 294, 295, 298, 299, 300, 301, 302, 477, 486, 1001, 1002, 1008, 1009, 1250, 1251, 1259, 1260, 1268, 1269, 1293, 1295, 1330, 1331, 1332, 1333 |
| `src/Item/Item_Data.cpp` | 6 | 76, 145, 147, 149, 156, 157 |
| `src/Item/Item_Display.cpp` | 4 | 372, 374, 525, 560 |
| `src/Item/Item_Durability.cpp` | 4 | 121, 464, 486, 514 |
| `src/Item/Item_Equip.cpp` | 1 | 27 |
| `src/Item/Item_InventoryGrid.cpp` | 1 | 201 |
| `src/Item/Item_InventoryRender.cpp` | 3 | 416, 419, 431 |
| `src/Item/Item_LegacyGridInsert.cpp` | 1 | 76 |
| `src/Item/Item_LegacyLinker.cpp` | 2 | 100, 101 |
| `src/Item/NPC_ModelLoad.cpp` | 2 | 26, 156 |
| `src/Item/Skill_Data.cpp` | 3 | 126, 127, 136 |
| `src/Model/Model_Players.cpp` | 5 | 349, 350, 351, 352, 353 |
| `src/Monster/Monster.cpp` | 3 | 521, 638, 653 |
| `src/Monster/Monster_Data.cpp` | 41 | 145, 148, 189, 285, 294, 301, 313, 317, 320, 329, 338, 352, 359, 372, 381, 391, 392, 399, 407, 429, 438, 453, 462, 471, 480, 489, 496, 505, 506, 515, 558, 567, 570, 584, 595, 603, 610, 617, 623, 630, 637 |
| `src/Net/Crypto.cpp` | 6 | 1005, 1361, 1532, 1536, 1540, 1544 |
| `src/Net/Net_Connect.cpp` | 2 | 124, 185 |
| `src/Net/Net_PacketSession.cpp` | 1 | 80 |
| `src/Net/Recv/Recv_Account.cpp` | 5 | 239, 263, 282, 282, 284 |
| `src/Net/Recv/Recv_Character.cpp` | 11 | 163, 413, 425, 494, 665, 672, 679, 680, 685, 1019, 1365 |
| `src/Net/Recv/Recv_Chat.cpp` | 1 | 33 |
| `src/Net/Recv/Recv_Combat.cpp` | 6 | 84, 210, 221, 645, 800, 897 |
| `src/Net/Recv/Recv_Event.cpp` | 1 | 173 |
| `src/Net/Recv/Recv_Guild.cpp` | 2 | 21, 531 |
| `src/Net/Recv/Recv_Item.cpp` | 5 | 83, 407, 430, 662, 699 |
| `src/Net/Recv/Recv_Quest.cpp` | 2 | 31, 88 |
| `src/Net/Recv/Recv_Trade.cpp` | 1 | 56 |
| `src/Net/Recv/Recv_Viewport.cpp` | 4 | 83, 94, 107, 786 |
| `src/Net/SecondPassword.cpp` | 134 | 207, 213, 223, 223, 224, 224, 225, 225, 234, 239, 330, 404, 415, 472, 478, 504, 505, 506, 507, 525, 527, 536, 537, 538, 551, 552, 553, 554, 585, 587, 608, 610, 611, 612, 631, 632, 633, 634, 662, 664, 671, 673, 678, 680, 685, 687, 694, 696, 701, 703, 717, 718, 720, 750, 757, 758, 768, 771, 772, 773, 774, 781, 782, 783, 784, 802, 803, 805, 806, 819, 820, 824, 825, 830, 831, 1020, 1021, 1022, 1023, 1068, 1069, 1070, 1071, 1173, 1174, 1175, 1176, 1198, 1199, 1200, 1201, 1230, 1231, 1232, 1233, 1264, 1265, 1280, 1292, 1308, 1309, 1310, 1311, 1495, 1497, 1592, 1593, 1840, 1906, 1924, 1927, 2048, 2161, 2170, 2218, 2700, 2701, 2702, 2703, 2752, 2752, 2752, 2753, 2753, 2753, 2754, 2754, 2754, 2755, 2755, 2755, 3032, 4007, 4759 |
| `src/Path/Path_Legacy.cpp` | 6 | 377, 377, 380, 392, 399, 399 |
| `src/Physics/Cloth_Simulation.cpp` | 1 | 62 |
| `src/Platform/ClientEntry.cpp` | 4 | 609, 616, 686, 692 |
| `src/Render/Ambient_Particles.cpp` | 8 | 128, 135, 153, 263, 326, 337, 365, 366 |
| `src/Render/BMD_Anim.cpp` | 1 | 98 |
| `src/Render/BMD_Collision.cpp` | 4 | 60, 61, 62, 63 |
| `src/Render/BMD_DrawMesh.cpp` | 10 | 196, 232, 233, 295, 408, 458, 459, 460, 462, 489 |
| `src/Render/BMD_FilterData.cpp` | 6 | 26, 90, 112, 115, 132, 192 |
| `src/Render/BMD_LegacyDraw.cpp` | 8 | 91, 92, 144, 150, 157, 1336, 1849, 1885 |
| `src/Render/BMD_LegacyEffects.cpp` | 2 | 114, 123 |
| `src/Render/BMD_LegacyLoading.cpp` | 9 | 84, 341, 348, 350, 352, 360, 400, 543, 561 |
| `src/Render/BMD_Load.cpp` | 30 | 486, 488, 489, 490, 491, 492, 537, 538, 539, 540, 541, 542, 543, 544, 545, 546, 547, 548, 549, 550, 551, 552, 553, 554, 555, 556, 557, 558, 559, 560 |
| `src/Render/BMD_SetupRender.cpp` | 1 | 334 |
| `src/Render/BMD_SkinUpdate.cpp` | 21 | 43, 45, 47, 75, 81, 145, 161, 171, 180, 182, 200, 201, 202, 205, 207, 209, 210, 218, 240, 252, 279 |
| `src/Render/Camera_Login.cpp` | 1 | 19 |
| `src/Render/Camera_Movement.cpp` | 1 | 53 |
| `src/Render/Effect_Combat.cpp` | 4 | 76, 82, 171, 212 |
| `src/Render/Effect_Create.cpp` | 88 | 147, 694, 826, 1289, 1368, 1403, 1461, 1475, 1476, 1484, 1505, 1515, 1574, 1608, 1668, 1702, 1757, 1791, 1851, 1862, 1916, 1950, 1966, 2025, 2039, 2040, 2048, 2069, 2079, 2138, 2172, 2232, 2266, 2325, 2359, 2420, 2456, 2510, 2569, 2583, 2584, 2592, 2612, 2622, 2681, 2715, 2775, 2809, 2868, 2926, 2960, 3008, 3019, 3074, 3108, 3167, 3223, 3240, 3241, 3270, 3270, 3280, 3338, 3372, 3422, 3456, 3515, 3549, 3609, 3643, 3806, 3867, 3881, 3882, 3890, 3913, 3922, 3984, 4042, 4056, 4057, 4065, 4086, 4096, 4157, 4180, 4181, 4191 |
| `src/Render/Effect_Tick.cpp` | 3 | 31, 329, 330 |
| `src/Render/Entity_AnimationLegacy.cpp` | 10 | 55, 213, 214, 214, 273, 274, 344, 363, 381, 382 |
| `src/Render/Entity_DrawAt.cpp` | 1 | 115 |
| `src/Render/Entity_DrawByType.cpp` | 1 | 237 |
| `src/Render/Entity_DrawSetup.cpp` | 3 | 150, 215, 219 |
| `src/Render/Entity_Render.cpp` | 5 | 212, 213, 232, 287, 413 |
| `src/Render/Entity_RenderAll_3D.cpp` | 2 | 104, 127 |
| `src/Render/Entity_Render_3D.cpp` | 2 | 155, 584 |
| `src/Render/Entity_UpdateRender.cpp` | 5 | 85, 446, 458, 539, 733 |
| `src/Render/EquipmentVisuals.cpp` | 5 | 120, 122, 126, 168, 190 |
| `src/Render/GL_State.cpp` | 1 | 305 |
| `src/Render/HUD_Pass2.cpp` | 1 | 76 |
| `src/Render/HUD_Pass4.cpp` | 4 | 345, 346, 362, 363 |
| `src/Render/HUD_Pass6.cpp` | 6 | 95, 127, 128, 810, 1577, 1592 |
| `src/Render/ItemDrop_Render2.cpp` | 3 | 46, 62, 63 |
| `src/Render/Joint_Create.cpp` | 19 | 82, 522, 581, 598, 599, 761, 818, 835, 836, 2077, 2112, 2168, 2222, 2258, 2333, 2367, 2433, 2487, 2523 |
| `src/Render/LooseWingVisuals.cpp` | 2 | 93, 117 |
| `src/Render/MaterialVisuals.cpp` | 4 | 83, 146, 154, 156 |
| `src/Render/MoveEffect.cpp` | 2 | 682, 2424 |
| `src/Render/MoveJoint.cpp` | 7 | 153, 336, 370, 858, 983, 1103, 1160 |
| `src/Render/Particle_Move.cpp` | 6 | 699, 707, 861, 896, 960, 1105 |
| `src/Render/Particle_Render.cpp` | 2 | 108, 144 |
| `src/Render/RenderLinkObject.cpp` | 15 | 109, 143, 343, 356, 391, 402, 403, 406, 416, 417, 442, 457, 510, 520, 532 |
| `src/Render/Render_Frame.cpp` | 1 | 669 |
| `src/Render/Render_LegacyBillboards.cpp` | 2 | 57, 66 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 1 | 138 |
| `src/Render/Render_LegacyLinker.cpp` | 4 | 90, 363, 439, 440 |
| `src/Render/Render_LegacyTrails.cpp` | 1 | 55 |
| `src/Render/Render_PlayerEquipment.cpp` | 4 | 188, 220, 677, 712 |
| `src/Render/Render_WorldHelpers.cpp` | 15 | 122, 123, 144, 215, 217, 688, 790, 814, 893, 896, 939, 1013, 1181, 1197, 1212 |
| `src/Render/SMD_Parser.cpp` | 6 | 296, 546, 839, 862, 863, 864 |
| `src/Render/SkillEffect_Render.cpp` | 1 | 120 |
| `src/Render/Terrain_Render.cpp` | 1 | 122 |
| `src/Render/Terrain_Water.cpp` | 5 | 66, 66, 67, 68, 112 |
| `src/Render/Weather.cpp` | 72 | 166, 189, 215, 238, 261, 264, 265, 274, 281, 284, 285, 294, 295, 310, 346, 350, 372, 392, 393, 394, 397, 433, 436, 449, 450, 451, 456, 460, 464, 478, 482, 508, 512, 515, 516, 517, 530, 531, 543, 545, 561, 572, 580, 615, 618, 619, 622, 623, 626, 627, 658, 658, 665, 666, 671, 672, 677, 678, 687, 690, 699, 700, 722, 734, 735, 753, 765, 766, 767, 768, 768, 771 |
| `src/Render/Weather_Particles.cpp` | 22 | 63, 64, 74, 104, 105, 107, 108, 111, 133, 134, 135, 144, 163, 164, 166, 180, 181, 182, 196, 197, 198, 205 |
| `src/Render/WorldEffects.cpp` | 1 | 59 |
| `src/Scene/Scene_AssetLoad.cpp` | 2 | 163, 164 |
| `src/Scene/Scene_CharSelect.cpp` | 9 | 259, 275, 295, 297, 352, 378, 395, 542, 632 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 13 | 181, 892, 894, 1189, 1276, 1281, 1290, 1723, 1726, 1735, 1738, 1899, 2106 |
| `src/Scene/Scene_Dispatch.cpp` | 1 | 115 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 3 | 61, 238, 266 |
| `src/Scene/Scene_Intro.cpp` | 2 | 232, 280 |
| `src/Scene/Scene_Login_ServerSelect.cpp` | 17 | 235, 263, 333, 344, 362, 369, 377, 393, 396, 422, 426, 429, 431, 467, 468, 508, 519 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 1 | 106 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 5 | 86, 631, 672, 724, 798 |
| `src/Scene/Scene_Objects.cpp` | 9 | 61, 157, 431, 434, 435, 436, 437, 438, 439 |
| `src/Scene/Scene_Resources.cpp` | 2 | 27, 207 |
| `src/Scene/Scene_ServerSelect_Input.cpp` | 13 | 65, 68, 72, 75, 147, 159, 160, 191, 194, 195, 203, 215, 264 |
| `src/Sound/Sound_DS3D.cpp` | 3 | 91, 92, 93 |
| `src/Sound/Sound_Queue.cpp` | 7 | 20, 27, 29, 112, 112, 127, 145 |
| `src/Terrain/Terrain.cpp` | 9 | 415, 440, 444, 448, 465, 465, 466, 477, 477 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 10 | 225, 283, 641, 687, 689, 690, 691, 692, 693, 694 |
| `src/Terrain/Terrain_LegacyRender.cpp` | 5 | 58, 61, 64, 67, 75 |
| `src/Terrain/Terrain_Light.cpp` | 9 | 51, 53, 54, 55, 193, 193, 194, 195, 223 |
| `src/Terrain/Terrain_RenderFace.cpp` | 6 | 45, 87, 87, 109, 199, 260 |
| `src/Terrain/Terrain_Tile.cpp` | 4 | 19, 19, 28, 28 |
| `src/Terrain/Terrain_Utils.cpp` | 4 | 52, 53, 54, 55 |
| `src/Trade/Trade.cpp` | 1 | 108 |
| `src/UI/CharMenu_Build.cpp` | 26 | 428, 442, 447, 496, 496, 497, 497, 506, 507, 512, 513, 523, 527, 529, 531, 540, 543, 560, 561, 561, 562, 568, 571, 600, 628, 669 |
| `src/UI/Chat.cpp` | 21 | 99, 100, 131, 158, 185, 267, 279, 338, 379, 438, 443, 444, 481, 482, 502, 506, 621, 622, 660, 661, 748 |
| `src/UI/Chat_InputTick.cpp` | 30 | 78, 311, 312, 351, 352, 353, 354, 364, 450, 454, 473, 493, 499, 514, 518, 531, 549, 555, 618, 619, 620, 633, 657, 714, 715, 716, 722, 725, 726, 837 |
| `src/UI/Chat_Log.cpp` | 1 | 96 |
| `src/UI/Chat_Send.cpp` | 1 | 46 |
| `src/UI/GoldenArcher.cpp` | 4 | 367, 801, 827, 828 |
| `src/UI/RenderItemInfo.cpp` | 110 | 481, 482, 483, 491, 492, 493, 501, 502, 503, 511, 512, 513, 756, 757, 758, 766, 767, 768, 776, 777, 778, 860, 861, 862, 870, 871, 872, 1050, 1051, 1052, 1060, 1061, 1062, 1080, 1081, 1082, 1136, 1137, 1138, 1147, 1148, 1149, 1188, 1189, 1190, 1272, 1273, 1275, 1324, 1438, 1482, 1491, 1536, 1725, 1818, 1826, 1827, 1887, 1888, 1889, 1892, 1893, 1930, 1932, 1933, 1938, 1939, 1950, 1951, 1952, 1980, 1981, 1982, 1994, 1995, 1996, 2006, 2007, 2008, 2019, 2020, 2021, 2030, 2031, 2032, 2041, 2042, 2043, 2076, 2077, 2083, 2097, 2153, 2154, 2175, 2176, 2177, 2181, 2182, 2212, 2213, 2214, 2218, 2219, 2224, 2225, 2226, 2238, 2239, 2244 |
| `src/UI/UI_DialogInterface.cpp` | 6 | 59, 60, 61, 62, 63, 64 |
| `src/UI/UI_GuildLegacy.cpp` | 12 | 88, 190, 193, 194, 195, 196, 197, 214, 225, 237, 239, 257 |
| `src/UI/UI_GuildMark.cpp` | 17 | 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 87 |
| `src/UI/UI_InGameMenu.cpp` | 12 | 23, 85, 86, 87, 396, 434, 616, 637, 640, 646, 648, 972 |
| `src/UI/UI_LegacyExterns.cpp` | 20 | 90, 91, 378, 379, 383, 384, 388, 389, 395, 396, 402, 403, 409, 410, 416, 417, 425, 426, 431, 432 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 2 | 66, 92 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 1 | 320 |
| `src/UI/UI_StatsPanel.cpp` | 7 | 155, 472, 651, 805, 806, 891, 921 |
| `src/Util/Misc.cpp` | 1 | 160 |
| `src/globals.cpp` | 124 | 267, 293, 302, 303, 389, 391, 537, 587, 655, 666, 667, 668, 708, 709, 720, 730, 798, 800, 804, 815, 847, 851, 911, 926, 934, 947, 948, 956, 959, 979, 981, 1090, 1098, 1178, 1183, 1193, 1197, 1203, 1243, 1244, 1245, 1246, 1254, 1259, 1262, 1279, 1280, 1291, 1292, 1293, 1294, 1295, 1296, 1297, 1298, 1299, 1300, 1301, 1302, 1308, 1309, 1310, 1311, 1312, 1313, 1314, 1315, 1316, 1368, 1371, 1375, 1397, 1419, 1426, 1464, 1476, 1477, 1485, 1506, 1536, 1546, 1568, 1571, 1605, 1606, 1610, 1629, 1631, 1642, 1752, 1764, 1955, 2001, 2002, 2010, 2012, 2025, 2035, 2053, 2054, 2055, 2056, 2057, 2058, 2059, 2119, 2181, 2193, 2257, 2311, 2325, 2330, 2367, 2368, 2456, 2465, 2538, 2549, 2550, 2554, 2694, 2728, 2729, 2803 |
| `src/globals.h` | 196 | 245, 277, 286, 293, 294, 347, 349, 350, 448, 483, 507, 542, 580, 581, 582, 583, 584, 585, 586, 587, 588, 589, 590, 591, 592, 593, 594, 595, 596, 597, 598, 599, 600, 601, 602, 603, 604, 612, 620, 629, 665, 683, 708, 717, 721, 722, 739, 741, 745, 755, 778, 780, 781, 782, 783, 790, 850, 862, 880, 887, 888, 900, 901, 905, 906, 907, 908, 925, 927, 938, 939, 940, 949, 1042, 1044, 1045, 1046, 1047, 1048, 1051, 1052, 1054, 1109, 1165, 1166, 1167, 1168, 1169, 1170, 1171, 1177, 1200, 1201, 1202, 1203, 1213, 1216, 1247, 1248, 1288, 1289, 1290, 1291, 1292, 1293, 1294, 1295, 1297, 1298, 1299, 1300, 1301, 1302, 1303, 1304, 1305, 1306, 1307, 1308, 1310, 1311, 1312, 1313, 1314, 1315, 1316, 1317, 1318, 1335, 1339, 1343, 1347, 1413, 1417, 1418, 1422, 1423, 1442, 1486, 1487, 1488, 1489, 1510, 1511, 1512, 1513, 1542, 1557, 1591, 1592, 1597, 1598, 1619, 1621, 1637, 1728, 1740, 1954, 1993, 1994, 2013, 2018, 2020, 2027, 2034, 2035, 2036, 2042, 2053, 2054, 2055, 2056, 2057, 2058, 2059, 2121, 2137, 2161, 2193, 2243, 2265, 2270, 2304, 2305, 2306, 2383, 2384, 2392, 2481, 2488, 2692, 2716, 2717, 2807, 2809, 2844 |
| `src/structs.h` | 1 | 508 |

### 2a strides de layouts (candidatos)

Expresión (Python re):

```text
\b(?:0x394|0x1bc|0x9d8|0x204|0x70|0x44|0xbc|0x38|188|916|444|2520)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 13 | 838, 921, 931, 946, 1166, 1201, 1326, 1690, 1858, 1951, 2009, 2100, 2406 |
| `src/Combat/Combat_AttackEffect.cpp` | 5 | 379, 1286, 1794, 1989, 2108 |
| `src/Combat/Combat_AttackStage.cpp` | 7 | 33, 125, 274, 282, 332, 353, 363 |
| `src/Combat/Combat_LegacyAnimation.cpp` | 3 | 91, 92, 134 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 6 | 75, 186, 247, 258, 347, 398 |
| `src/Combat/Combat_Targeting.cpp` | 1 | 20 |
| `src/Combat/Skills.cpp` | 3 | 106, 107, 403 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 6 | 88, 205, 416, 423, 451, 456 |
| `src/Combat/Skills_WizardElf.cpp` | 7 | 87, 88, 89, 159, 199, 437, 459 |
| `src/Core/Runtime_Medium.cpp` | 1 | 279 |
| `src/Core/Runtime_Small.cpp` | 2 | 504, 675 |
| `src/Entity/CharacterAttributeView.h` | 4 | 12, 13, 15, 16 |
| `src/Entity/CharacterCloth.cpp` | 3 | 32, 116, 117 |
| `src/Entity/CharacterEffects.cpp` | 3 | 31, 115, 260 |
| `src/Entity/EntityTypes.h` | 3 | 169, 171, 208 |
| `src/Entity/EntityView.h` | 1 | 11 |
| `src/Entity/Entity_ActionLegacy.cpp` | 1 | 63 |
| `src/Entity/Entity_BoneEffects.cpp` | 3 | 52, 74, 92 |
| `src/Entity/Entity_HeadMotion.cpp` | 3 | 73, 92, 98 |
| `src/Entity/Entity_Init.cpp` | 2 | 29, 72 |
| `src/Entity/Entity_LegacyPoolCleanup.cpp` | 1 | 49 |
| `src/Entity/Entity_Lookup.cpp` | 1 | 19 |
| `src/Entity/Entity_MoveClient.cpp` | 2 | 47, 54 |
| `src/Entity/Entity_Spawn.cpp` | 1 | 200 |
| `src/Entity/ObjectPose.cpp` | 21 | 21, 27, 31, 71, 240, 282, 283, 318, 325, 327, 347, 350, 368, 371, 436, 455, 455, 456, 491, 491, 511 |
| `src/Game/Game_CharSelectTick.cpp` | 1 | 225 |
| `src/Game/Game_EnterWorldTick.cpp` | 19 | 151, 302, 319, 345, 371, 372, 473, 512, 648, 651, 652, 652, 653, 670, 674, 675, 675, 676, 699 |
| `src/Game/Game_SceneUpdate.cpp` | 4 | 67, 374, 401, 780 |
| `src/Game/Party_NameMatch.cpp` | 1 | 29 |
| `src/Game/Player_InputTick.cpp` | 7 | 430, 648, 821, 1236, 1249, 1437, 1473 |
| `src/Game/RenderDerivedState.cpp` | 5 | 66, 69, 70, 71, 75 |
| `src/Game/RenderFxSnapshots.cpp` | 9 | 33, 34, 35, 36, 37, 38, 39, 41, 42 |
| `src/Game/RenderInterpolation.cpp` | 18 | 57, 57, 57, 58, 58, 58, 59, 60, 60, 77, 81, 124, 124, 125, 157, 180, 180, 181 |
| `src/Game/RenderSnapshots.cpp` | 3 | 162, 168, 276 |
| `src/Game/SimulationPreparation.cpp` | 5 | 98, 124, 130, 132, 163 |
| `src/Input/Input.cpp` | 1 | 329 |
| `src/Input/Mouse_Hover.cpp` | 9 | 93, 94, 162, 349, 378, 490, 506, 516, 592 |
| `src/Item/ContentCatalog.cpp` | 3 | 79, 225, 586 |
| `src/Item/Item_ChaosMix.cpp` | 1 | 63 |
| `src/Item/Item_ClickHandler.cpp` | 9 | 78, 411, 481, 487, 800, 974, 1086, 1342, 1381 |
| `src/Item/Item_Display.cpp` | 7 | 417, 452, 453, 481, 482, 525, 560 |
| `src/Item/Item_Durability.cpp` | 5 | 284, 327, 463, 466, 497 |
| `src/Item/Item_Equip.cpp` | 1 | 70 |
| `src/Item/Item_EquipmentQueries.cpp` | 1 | 57 |
| `src/Item/Item_InventoryGrid.cpp` | 16 | 35, 51, 53, 74, 75, 86, 90, 130, 280, 310, 321, 329, 365, 387, 391, 398 |
| `src/Item/Item_LegacyGridInsert.cpp` | 5 | 77, 87, 98, 100, 142 |
| `src/Item/Item_LegacyLinker.cpp` | 5 | 86, 89, 90, 93, 96 |
| `src/Item/NPC_ModelLoad.cpp` | 2 | 26, 156 |
| `src/Local/Text_Data.cpp` | 2 | 192, 192 |
| `src/Math/Math_3D.cpp` | 1 | 173 |
| `src/Model/ModelTypes.h` | 2 | 58, 67 |
| `src/Model/Model_Gates.cpp` | 2 | 59, 100 |
| `src/Monster/Monster.cpp` | 2 | 513, 523 |
| `src/Monster/Monster_Data.cpp` | 18 | 189, 198, 264, 285, 309, 320, 389, 389, 390, 414, 504, 523, 550, 570, 577, 687, 699, 706 |
| `src/Net/Crypto.cpp` | 5 | 1005, 1083, 1105, 1146, 1371 |
| `src/Net/Net_Process.cpp` | 2 | 829, 835 |
| `src/Net/Network.cpp` | 1 | 56 |
| `src/Net/Reconnect.cpp` | 1 | 75 |
| `src/Net/Recv/Recv_Character.cpp` | 7 | 157, 464, 494, 672, 685, 1019, 1365 |
| `src/Net/Recv/Recv_Chat.cpp` | 1 | 33 |
| `src/Net/Recv/Recv_Combat.cpp` | 13 | 84, 177, 237, 371, 472, 521, 592, 645, 689, 749, 800, 847, 871 |
| `src/Net/Recv/Recv_Event.cpp` | 2 | 173, 230 |
| `src/Net/Recv/Recv_Guild.cpp` | 4 | 124, 427, 446, 519 |
| `src/Net/Recv/Recv_Item.cpp` | 11 | 39, 155, 465, 467, 529, 531, 550, 552, 568, 786, 788 |
| `src/Net/Recv/Recv_Quest.cpp` | 3 | 31, 88, 117 |
| `src/Net/Recv/Recv_Trade.cpp` | 2 | 30, 31 |
| `src/Net/Recv/Recv_Viewport.cpp` | 10 | 83, 94, 107, 209, 270, 430, 533, 533, 656, 932 |
| `src/Net/SecondPassword.cpp` | 19 | 708, 737, 741, 856, 1495, 1497, 1653, 1924, 2161, 2468, 3023, 3032, 3085, 3377, 3402, 3581, 3710, 4007, 4019 |
| `src/Physics/Cloth_Simulation.cpp` | 2 | 62, 798 |
| `src/Platform/ClientEntry.cpp` | 1 | 616 |
| `src/Platform/SDL3/SDLInput.cpp` | 1 | 15 |
| `src/Render/Ambient_Particles.cpp` | 4 | 113, 114, 263, 276 |
| `src/Render/BMD_Anim.cpp` | 1 | 98 |
| `src/Render/BMD_DrawMesh.cpp` | 6 | 152, 185, 218, 251, 372, 408 |
| `src/Render/BMD_LegacyDraw.cpp` | 7 | 63, 85, 117, 1336, 1712, 1885, 1908 |
| `src/Render/BMD_LegacyEffects.cpp` | 1 | 99 |
| `src/Render/BMD_LegacyLoading.cpp` | 9 | 341, 348, 350, 352, 360, 400, 415, 427, 480 |
| `src/Render/BMD_Load.cpp` | 6 | 184, 219, 540, 543, 554, 606 |
| `src/Render/BMD_SetupRender.cpp` | 24 | 42, 57, 67, 73, 82, 88, 97, 103, 116, 119, 123, 143, 155, 179, 184, 189, 192, 204, 216, 222, 227, 254, 259, 272 |
| `src/Render/Backend/GL/Materials.cpp` | 1 | 212 |
| `src/Render/Effect_Combat.cpp` | 4 | 76, 82, 171, 212 |
| `src/Render/Effect_Create.cpp` | 3 | 369, 4157, 4191 |
| `src/Render/Effect_Tick.cpp` | 1 | 240 |
| `src/Render/Entity_AnimationLegacy.cpp` | 8 | 55, 72, 147, 153, 231, 234, 330, 350 |
| `src/Render/Entity_DrawAt.cpp` | 3 | 115, 121, 123 |
| `src/Render/Entity_DrawByType.cpp` | 7 | 237, 393, 416, 673, 700, 745, 753 |
| `src/Render/Entity_DrawSetup.cpp` | 21 | 150, 161, 187, 188, 190, 199, 200, 202, 214, 215, 218, 219, 225, 235, 245, 382, 391, 401, 412, 425, 463 |
| `src/Render/Entity_Render.cpp` | 6 | 241, 289, 368, 413, 417, 419 |
| `src/Render/Entity_RenderAll_3D.cpp` | 1 | 134 |
| `src/Render/Entity_Render_3D.cpp` | 3 | 155, 180, 197 |
| `src/Render/Entity_UpdateRender.cpp` | 9 | 85, 197, 211, 228, 445, 446, 451, 458, 733 |
| `src/Render/EquipmentVisuals.cpp` | 6 | 22, 120, 122, 126, 168, 190 |
| `src/Render/EquipmentVisuals.h` | 1 | 24 |
| `src/Render/HUD_Pass1.cpp` | 1 | 484 |
| `src/Render/HUD_Pass3.cpp` | 7 | 71, 75, 76, 77, 78, 79, 82 |
| `src/Render/HUD_Pass5.cpp` | 4 | 337, 350, 352, 355 |
| `src/Render/HUD_Pass6.cpp` | 4 | 985, 1095, 1518, 1820 |
| `src/Render/ItemDrop_Render2.cpp` | 2 | 46, 59 |
| `src/Render/JointTypes.h` | 1 | 48 |
| `src/Render/Joint_Create.cpp` | 47 | 84, 124, 245, 268, 279, 293, 314, 353, 399, 414, 464, 649, 684, 710, 909, 934, 961, 1003, 1067, 1130, 1146, 1161, 1194, 1287, 1349, 1356, 1385, 1400, 1409, 1420, 1431, 1462, 1478, 1564, 1575, 1693, 1750, 1774, 1801, 1816, 1845, 1861, 1929, 1982, 2012, 2030, 2061 |
| `src/Render/Joint_Render.cpp` | 1 | 58 |
| `src/Render/LooseWingVisuals.cpp` | 2 | 93, 117 |
| `src/Render/MaterialVisuals.cpp` | 1 | 83 |
| `src/Render/MoveEffect.cpp` | 15 | 82, 84, 85, 86, 87, 88, 308, 311, 322, 341, 405, 682, 684, 1079, 2424 |
| `src/Render/MoveJoint.cpp` | 2 | 153, 1002 |
| `src/Render/Particle.cpp` | 1 | 185 |
| `src/Render/Particle_Legacy.cpp` | 2 | 17, 33 |
| `src/Render/Particle_Move.cpp` | 5 | 49, 52, 64, 111, 861 |
| `src/Render/Particle_Render.cpp` | 5 | 103, 103, 103, 108, 146 |
| `src/Render/Particle_Spawn.cpp` | 7 | 60, 314, 343, 455, 459, 491, 514 |
| `src/Render/Player_Render.cpp` | 1 | 95 |
| `src/Render/RenderLinkObject.cpp` | 3 | 109, 144, 684 |
| `src/Render/Render_Frame.cpp` | 1 | 411 |
| `src/Render/Render_LegacyBillboards.cpp` | 7 | 57, 61, 230, 240, 245, 251, 272 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 1 | 140 |
| `src/Render/Render_LegacyLinker.cpp` | 2 | 90, 363 |
| `src/Render/Render_LegacyTrails.cpp` | 2 | 65, 69 |
| `src/Render/Render_PlayerEquipment.cpp` | 5 | 127, 221, 677, 713, 807 |
| `src/Render/Render_WorldHelpers.cpp` | 22 | 122, 123, 127, 144, 191, 215, 217, 319, 616, 696, 711, 721, 762, 792, 811, 814, 893, 896, 1173, 1181, 1197, 1212 |
| `src/Render/ResourceTables.cpp` | 2 | 66, 67 |
| `src/Render/SMD_Parser.cpp` | 11 | 224, 224, 236, 297, 305, 419, 571, 770, 778, 812, 839 |
| `src/Render/SkillEffect_Render.cpp` | 1 | 120 |
| `src/Render/Sprite.cpp` | 1 | 180 |
| `src/Render/Terrain_Render.cpp` | 1 | 130 |
| `src/Render/Texture/Texture.cpp` | 3 | 98, 680, 727 |
| `src/Render/Trail_Render.cpp` | 1 | 113 |
| `src/Render/Weather.cpp` | 6 | 121, 309, 312, 325, 395, 508 |
| `src/Render/Weather_Particles.cpp` | 1 | 103 |
| `src/Render/WorldEffects.cpp` | 1 | 59 |
| `src/Scene/Scene_CharSelect.cpp` | 10 | 275, 290, 295, 297, 347, 362, 394, 418, 469, 597 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 13 | 364, 390, 417, 1723, 1735, 1844, 1876, 1899, 2097, 2228, 2229, 2234, 2235 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 5 | 53, 79, 191, 265, 266 |
| `src/Scene/Scene_Intro.cpp` | 2 | 153, 154 |
| `src/Scene/Scene_Login_ServerSelect.cpp` | 1 | 275 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 1 | 60 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 7 | 86, 724, 798, 1102, 1183, 1306, 1310 |
| `src/Scene/Scene_Objects.cpp` | 6 | 70, 73, 75, 157, 265, 309 |
| `src/Scene/Scene_Resources.cpp` | 5 | 27, 31, 36, 149, 208 |
| `src/Scene/Scene_ServerSelect_Input.cpp` | 1 | 82 |
| `src/Sound/Sound_Queue.cpp` | 7 | 20, 25, 27, 29, 30, 53, 110 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 11 | 225, 232, 240, 292, 294, 294, 376, 481, 486, 584, 591 |
| `src/Terrain/Terrain_Light.cpp` | 1 | 112 |
| `src/Trade/Trade.cpp` | 5 | 108, 395, 481, 548, 548 |
| `src/UI/CharMenu_Build.cpp` | 1 | 483 |
| `src/UI/ChatListBox.cpp` | 5 | 340, 342, 1911, 2552, 2560 |
| `src/UI/Chat_Bubbles.cpp` | 2 | 261, 274 |
| `src/UI/Chat_Gestures.cpp` | 1 | 41 |
| `src/UI/Chat_InputTick.cpp` | 7 | 69, 78, 147, 154, 178, 501, 890 |
| `src/UI/GoldenArcher.cpp` | 2 | 218, 839 |
| `src/UI/MiniMap.cpp` | 1 | 258 |
| `src/UI/RenderItemInfo.cpp` | 1 | 43 |
| `src/UI/UI_InGameMenu.cpp` | 3 | 93, 184, 396 |
| `src/UI/UI_LegacyExterns.cpp` | 2 | 194, 423 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 4 | 78, 79, 80, 94 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 6 | 73, 75, 259, 270, 320, 385 |
| `src/UI/UI_StatsPanel.cpp` | 4 | 423, 566, 630, 651 |
| `src/Util/Misc.cpp` | 1 | 160 |
| `src/globals.cpp` | 17 | 720, 730, 747, 798, 800, 838, 839, 1254, 1259, 1371, 1752, 1916, 1955, 2025, 2033, 2035, 2465 |
| `src/globals.h` | 22 | 600, 612, 620, 648, 739, 741, 940, 1213, 1216, 1728, 1890, 1954, 2007, 2009, 2010, 2011, 2012, 2013, 2034, 2041, 2042, 2392 |
| `src/structs.h` | 1 | 292 |

### 2b accesos por offset (candidatos)

Expresión (Python re):

```text
\*\s*\([^;\n()]*\*\s*\)\s*\([^;\n]*\+\s*(?:0x[0-9a-fA-F]+|[0-9]+)
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 85 | 466, 542, 575, 576, 584, 587, 588, 589, 811, 812, 852, 953, 954, 981, 985, 1002, 1003, 1068, 1080, 1081, 1089, 1090, 1200, 1343, 1344, 1348, 1349, 1365, 1366, 1367, 1397, 1400, 1404, 1581, 1588, 1639, 1640, 1665, 1681, 1691, 1850, 1851, 1852, 1859, 1860, 1861, 1862, 1863, 1864, 1866, 1952, 1953, 1961, 1970, 1971, 1972, 2013, 2033, 2041, 2075, 2076, 2087, 2088, 2101, 2102, 2116, 2117, 2135, 2136, 2137, 2140, 2141, 2163, 2167, 2311, 2314, 2342, 2353, 2365, 2407, 2411, 2412, 2421, 2422, 2455 |
| `src/Combat/Combat_AttackEffect.cpp` | 44 | 418, 449, 466, 535, 568, 618, 649, 666, 773, 804, 821, 907, 939, 956, 1061, 1092, 1109, 1200, 1231, 1248, 1335, 1366, 1383, 1459, 1463, 1464, 1490, 1491, 1492, 1541, 1572, 1589, 1670, 1702, 1719, 1808, 1829, 1842, 1860, 1882, 2021, 2042, 2078, 2123 |
| `src/Combat/Combat_AttackStage.cpp` | 20 | 70, 83, 135, 163, 274, 275, 345, 346, 346, 350, 357, 357, 359, 363, 363, 403, 406, 409, 411, 415 |
| `src/Combat/Combat_LegacyAnimation.cpp` | 22 | 91, 92, 118, 133, 134, 135, 136, 138, 139, 140, 141, 142, 143, 144, 145, 148, 167, 179, 180, 181, 182, 183 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 4 | 208, 208, 208, 264 |
| `src/Combat/Combat_Projectiles.cpp` | 3 | 29, 30, 61 |
| `src/Combat/Combat_Targeting.cpp` | 5 | 24, 25, 28, 29, 30 |
| `src/Combat/Skills.cpp` | 6 | 93, 97, 343, 349, 383, 398 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 43 | 164, 165, 169, 206, 207, 208, 210, 211, 213, 219, 220, 221, 227, 228, 229, 240, 250, 257, 258, 259, 260, 261, 262, 264, 270, 280, 284, 306, 307, 374, 377, 379, 413, 414, 423, 443, 444, 445, 446, 447, 448, 449, 450 |
| `src/Combat/Skills_WizardElf.cpp` | 20 | 74, 75, 77, 87, 88, 89, 98, 99, 102, 159, 199, 299, 303, 307, 370, 379, 380, 381, 384, 459 |
| `src/Core/Runtime_Medium.cpp` | 72 | 109, 110, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 191, 194, 198, 199, 200, 201, 202, 203, 204, 211, 211, 215, 215, 218, 219, 229, 230, 232, 236, 237, 299, 305, 307, 314, 332, 375, 413, 415, 416, 419, 420, 422, 460, 463, 463, 463, 467, 491, 508, 508, 516, 521, 523, 526, 531, 532, 536, 538, 540, 542, 548, 555, 560, 568, 573, 575, 578, 579, 580, 685 |
| `src/Core/Runtime_Small.cpp` | 43 | 166, 167, 168, 199, 201, 239, 241, 299, 305, 306, 307, 308, 334, 338, 340, 418, 463, 464, 475, 476, 477, 478, 479, 598, 599, 617, 618, 619, 647, 648, 658, 659, 672, 699, 700, 701, 714, 725, 726, 728, 848, 849, 850 |
| `src/Entity/CharacterCloth.cpp` | 7 | 35, 80, 102, 132, 178, 191, 223 |
| `src/Entity/CharacterEffects.cpp` | 25 | 51, 120, 123, 138, 150, 156, 159, 168, 260, 261, 275, 276, 281, 292, 323, 324, 361, 362, 363, 364, 365, 366, 390, 391, 392 |
| `src/Entity/Entity_ActionLegacy.cpp` | 1 | 63 |
| `src/Entity/Entity_BoneEffects.cpp` | 8 | 52, 53, 74, 82, 88, 91, 92, 93 |
| `src/Entity/Entity_HeadMotion.cpp` | 13 | 21, 22, 23, 24, 26, 27, 28, 59, 73, 86, 92, 97, 98 |
| `src/Entity/Entity_Init.cpp` | 2 | 75, 76 |
| `src/Entity/Entity_LegacyEffects.cpp` | 42 | 49, 50, 51, 52, 54, 56, 60, 61, 65, 68, 69, 73, 74, 78, 80, 81, 82, 83, 85, 86, 87, 91, 92, 96, 97, 97, 98, 98, 100, 100, 101, 101, 104, 104, 106, 109, 117, 118, 136, 137, 138, 153 |
| `src/Entity/Entity_LegacyPoolCleanup.cpp` | 1 | 47 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 2 | 82, 83 |
| `src/Entity/Entity_Lookup.cpp` | 1 | 20 |
| `src/Entity/Entity_Reset.cpp` | 1 | 72 |
| `src/Entity/Entity_Spawn.cpp` | 2 | 98, 200 |
| `src/Entity/ObjectPose.cpp` | 81 | 18, 19, 20, 21, 22, 26, 27, 28, 30, 31, 32, 72, 110, 113, 114, 116, 117, 118, 119, 123, 135, 136, 140, 141, 142, 143, 145, 146, 147, 148, 149, 151, 157, 159, 160, 162, 163, 170, 171, 172, 173, 175, 176, 177, 197, 200, 201, 208, 209, 216, 229, 239, 240, 241, 325, 351, 353, 355, 358, 372, 373, 374, 375, 379, 390, 391, 395, 401, 402, 411, 413, 414, 463, 464, 475, 493, 496, 500, 501, 502, 503 |
| `src/Game/Game_EnterWorldTick.cpp` | 4 | 371, 513, 651, 674 |
| `src/Game/Game_SceneUpdate.cpp` | 21 | 251, 252, 253, 254, 255, 256, 257, 266, 267, 268, 269, 270, 271, 272, 280, 281, 282, 283, 284, 285, 286 |
| `src/Game/MapManager.cpp` | 2 | 44, 45 |
| `src/Game/Party_NameMatch.cpp` | 7 | 37, 44, 46, 47, 58, 66, 67 |
| `src/Game/PathFinder.cpp` | 145 | 54, 55, 56, 59, 60, 61, 63, 79, 82, 90, 93, 98, 99, 102, 105, 117, 118, 122, 150, 152, 153, 156, 158, 165, 166, 181, 184, 185, 191, 192, 195, 202, 231, 233, 234, 237, 239, 245, 246, 264, 265, 271, 272, 275, 282, 320, 321, 324, 361, 362, 365, 417, 419, 420, 429, 430, 439, 441, 443, 444, 445, 446, 449, 450, 451, 454, 469, 473, 474, 478, 482, 483, 494, 498, 499, 503, 507, 508, 527, 528, 529, 530, 531, 534, 535, 536, 538, 543, 548, 551, 556, 567, 569, 573, 577, 578, 584, 585, 587, 589, 592, 593, 595, 596, 599, 601, 602, 604, 605, 608, 609, 610, 611, 625, 627, 632, 652, 653, 655, 656, 657, 660, 661, 662, 665, 666, 667, 698, 703, 705, 706, 708, 709, 712, 714, 715, 717, 718, 721, 722, 723, 724, 730, 731, 774 |
| `src/Game/Player_InputTick.cpp` | 24 | 372, 427, 428, 432, 433, 435, 437, 438, 442, 623, 644, 721, 1188, 1189, 1323, 1324, 1396, 1397, 1404, 1405, 1476, 1477, 1668, 1669 |
| `src/Game/RenderInterpolation.cpp` | 1 | 157 |
| `src/Game/RenderSnapshots.cpp` | 4 | 162, 173, 175, 176 |
| `src/Game/SimulationPreparation.cpp` | 6 | 100, 107, 117, 118, 125, 165 |
| `src/Input/Cursor_Render.cpp` | 2 | 83, 98 |
| `src/Input/Input_WndProc.cpp` | 2 | 289, 290 |
| `src/Input/Mouse_Hover.cpp` | 34 | 93, 94, 122, 162, 191, 351, 354, 363, 364, 365, 394, 401, 403, 404, 492, 493, 494, 508, 509, 517, 518, 519, 546, 548, 549, 550, 551, 557, 559, 565, 566, 567, 592, 600 |
| `src/Item/ContentCatalog.cpp` | 13 | 117, 118, 122, 123, 124, 125, 126, 226, 227, 230, 234, 235, 587 |
| `src/Item/Gate_Data.cpp` | 3 | 51, 84, 85 |
| `src/Item/Item_ChaosMix.cpp` | 68 | 76, 82, 83, 85, 86, 89, 120, 124, 127, 129, 194, 197, 212, 224, 227, 361, 365, 366, 378, 383, 395, 400, 403, 405, 412, 416, 420, 424, 428, 431, 434, 438, 442, 445, 458, 460, 462, 469, 473, 483, 499, 500, 503, 508, 516, 517, 518, 519, 528, 529, 530, 536, 537, 538, 543, 548, 556, 557, 558, 559, 560, 561, 562, 563, 564, 565, 566, 567 |
| `src/Item/Item_ClickHandler.cpp` | 6 | 553, 554, 936, 937, 938, 967 |
| `src/Item/Item_Display.cpp` | 5 | 84, 244, 245, 453, 570 |
| `src/Item/Item_Durability.cpp` | 14 | 137, 171, 241, 242, 277, 291, 301, 312, 466, 468, 469, 497, 500, 501 |
| `src/Item/Item_Equip.cpp` | 15 | 77, 78, 81, 82, 118, 131, 139, 140, 148, 179, 195, 204, 214, 227, 231 |
| `src/Item/Item_EquipmentAutoSwap.cpp` | 2 | 55, 56 |
| `src/Item/Item_EquipmentQueries.cpp` | 2 | 36, 37 |
| `src/Item/Item_InventoryGrid.cpp` | 15 | 51, 53, 122, 123, 130, 131, 205, 230, 254, 255, 321, 329, 351, 352, 446 |
| `src/Item/Item_InventoryRender.cpp` | 7 | 73, 103, 185, 234, 235, 284, 439 |
| `src/Item/Item_LegacyGridInsert.cpp` | 1 | 79 |
| `src/Item/Item_LegacyLinker.cpp` | 3 | 86, 482, 485 |
| `src/Item/NPC_ModelLoad.cpp` | 2 | 27, 156 |
| `src/Item/Quest_UIState.cpp` | 1 | 18 |
| `src/Item/RightClickMove.cpp` | 1 | 48 |
| `src/Math/Math_3D.cpp` | 4 | 163, 172, 173, 174 |
| `src/Math/Math_LegacyTransforms.cpp` | 1 | 88 |
| `src/Model/Model_Items.cpp` | 31 | 323, 324, 325, 327, 333, 336, 337, 338, 339, 340, 344, 346, 347, 351, 355, 359, 363, 364, 365, 366, 367, 368, 369, 370, 371, 372, 373, 374, 377, 378, 379 |
| `src/Model/Model_Players.cpp` | 10 | 349, 349, 350, 350, 351, 351, 352, 352, 353, 353 |
| `src/Monster/Monster.cpp` | 6 | 509, 527, 590, 653, 1071, 1107 |
| `src/Monster/Monster_Data.cpp` | 94 | 190, 192, 194, 195, 196, 197, 198, 199, 201, 202, 205, 206, 208, 212, 213, 216, 217, 225, 228, 231, 234, 237, 241, 244, 248, 251, 254, 257, 260, 263, 264, 267, 274, 285, 294, 301, 313, 317, 320, 329, 338, 352, 359, 372, 381, 391, 392, 399, 400, 407, 429, 438, 453, 462, 471, 480, 489, 496, 505, 506, 515, 522, 523, 524, 525, 533, 534, 535, 536, 537, 549, 550, 551, 558, 567, 570, 576, 577, 578, 584, 595, 603, 610, 617, 623, 630, 637, 640, 647, 653, 662, 669, 676, 688 |
| `src/Net/Crypto.cpp` | 89 | 233, 241, 273, 274, 280, 286, 301, 302, 322, 323, 326, 328, 354, 355, 358, 364, 401, 403, 405, 415, 705, 706, 707, 708, 709, 710, 711, 791, 804, 958, 959, 961, 963, 964, 991, 995, 995, 1005, 1006, 1074, 1075, 1076, 1077, 1079, 1080, 1084, 1088, 1089, 1101, 1104, 1105, 1109, 1115, 1127, 1146, 1202, 1203, 1205, 1209, 1210, 1248, 1249, 1250, 1251, 1273, 1274, 1275, 1280, 1336, 1338, 1339, 1340, 1342, 1344, 1366, 1382, 1383, 1552, 1552, 1553, 1553, 1554, 1554, 1555, 1555, 1556, 1556, 1557, 1557 |
| `src/Net/Net_Connect.cpp` | 5 | 116, 173, 177, 236, 281 |
| `src/Net/Net_Events.cpp` | 2 | 50, 51 |
| `src/Net/Net_LegacyAuth.cpp` | 1 | 56 |
| `src/Net/Net_LegacyRuntime.cpp` | 32 | 248, 249, 250, 252, 253, 254, 256, 257, 258, 260, 261, 262, 264, 265, 272, 273, 289, 292, 293, 294, 298, 299, 300, 305, 306, 307, 308, 315, 316, 317, 318, 321 |
| `src/Net/Net_PacketSession.cpp` | 2 | 28, 80 |
| `src/Net/Reconnect.cpp` | 1 | 76 |
| `src/Net/Recv/Recv_Account.cpp` | 5 | 263, 285, 286, 287, 288 |
| `src/Net/Recv/Recv_Character.cpp` | 146 | 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 88, 89, 90, 91, 92, 93, 264, 378, 379, 382, 383, 384, 385, 386, 389, 390, 391, 392, 395, 396, 399, 400, 401, 404, 405, 412, 413, 425, 431, 432, 433, 434, 435, 436, 556, 643, 653, 654, 655, 656, 658, 659, 660, 665, 679, 742, 802, 803, 804, 805, 806, 808, 809, 811, 812, 813, 814, 815, 818, 819, 838, 839, 840, 841, 842, 843, 844, 845, 846, 847, 848, 849, 850, 858, 940, 941, 942, 943, 944, 945, 946, 947, 948, 949, 950, 951, 952, 953, 954, 957, 963, 964, 966, 967, 968, 968, 969, 969, 971, 996, 997, 1000, 1001, 1004, 1092, 1105, 1107, 1365, 1456, 1462, 1463, 1464, 1465, 1466, 1662, 1663, 1665, 1666, 1696, 1697, 1699, 1700, 1703, 1704, 1706, 1707, 1710, 1711, 1736, 1758 |
| `src/Net/Recv/Recv_Combat.cpp` | 57 | 12, 23, 24, 45, 76, 79, 101, 159, 178, 204, 205, 206, 212, 234, 239, 240, 258, 271, 295, 296, 297, 372, 474, 480, 522, 593, 651, 652, 653, 661, 690, 723, 727, 728, 729, 731, 732, 733, 735, 736, 737, 739, 740, 741, 742, 790, 791, 807, 808, 809, 817, 848, 872, 879, 889, 890, 891 |
| `src/Net/Recv/Recv_Event.cpp` | 5 | 148, 149, 150, 151, 174 |
| `src/Net/Recv/Recv_Guild.cpp` | 7 | 105, 224, 230, 353, 428, 446, 531 |
| `src/Net/Recv/Recv_Item.cpp` | 14 | 467, 531, 552, 662, 697, 699, 714, 715, 717, 718, 788, 896, 897, 990 |
| `src/Net/Recv/Recv_Quest.cpp` | 10 | 31, 33, 101, 102, 117, 119, 128, 129, 132, 133 |
| `src/Net/Recv/Recv_Trade.cpp` | 4 | 31, 118, 119, 168 |
| `src/Net/Recv/Recv_Viewport.cpp` | 25 | 69, 71, 210, 271, 358, 409, 532, 542, 543, 544, 561, 846, 847, 848, 849, 850, 852, 855, 857, 857, 858, 858, 859, 859, 861 |
| `src/Net/SecondPassword.cpp` | 212 | 330, 472, 608, 713, 717, 718, 720, 768, 855, 901, 1256, 1430, 1489, 1490, 1495, 1497, 1532, 1533, 1534, 1536, 1645, 1647, 1659, 1679, 1739, 1924, 2161, 2170, 2189, 2190, 2191, 2192, 2193, 2199, 2202, 2203, 2205, 2207, 2217, 2222, 2224, 2226, 2266, 2305, 2313, 2314, 2315, 2316, 2318, 2319, 2335, 2340, 2345, 2350, 2355, 2360, 2361, 2362, 2363, 2364, 2365, 2366, 2367, 2368, 2369, 2407, 2408, 2409, 2412, 2413, 2416, 2419, 2420, 2421, 2422, 2423, 2424, 2427, 2429, 2454, 2460, 2461, 2478, 2489, 2490, 2569, 2570, 2651, 2652, 2900, 2902, 2903, 2903, 2903, 2904, 2911, 2912, 2912, 2912, 2913, 2913, 2913, 2914, 2921, 2924, 2925, 2926, 2927, 2927, 2927, 2928, 2928, 2928, 2929, 2929, 2929, 2930, 2969, 2973, 3018, 3019, 3021, 3023, 3032, 3146, 3147, 3148, 3203, 3214, 3215, 3216, 3236, 3249, 3250, 3351, 3360, 3376, 3379, 3380, 3393, 3408, 3431, 3432, 3479, 3480, 3501, 3502, 3510, 3511, 3547, 3548, 3551, 3552, 3601, 3602, 3604, 3605, 3606, 3610, 3613, 3614, 3615, 3638, 3651, 3670, 3671, 3690, 3692, 3694, 3696, 3739, 3747, 3749, 3760, 3776, 3777, 3778, 3785, 3809, 3810, 3814, 3825, 3888, 3889, 3949, 3952, 3959, 3961, 3975, 3978, 4010, 4018, 4019, 4020, 4021, 4022, 4088, 4089, 4190, 4250, 4522, 4531, 4554, 4555, 4558, 4559, 4585, 4586, 4587, 4602, 4609, 4610 |
| `src/Party/Party.cpp` | 16 | 161, 162, 210, 211, 404, 425, 426, 427, 429, 430, 443, 459, 460, 461, 463, 464 |
| `src/Path/Path_Legacy.cpp` | 16 | 392, 399, 401, 402, 454, 455, 463, 464, 500, 501, 533, 534, 546, 547, 572, 573 |
| `src/Physics/Cloth_Simulation.cpp` | 111 | 29, 32, 33, 34, 35, 42, 44, 45, 60, 62, 67, 69, 115, 151, 211, 212, 213, 221, 223, 224, 225, 226, 240, 242, 243, 244, 259, 260, 264, 268, 270, 295, 316, 317, 318, 339, 340, 341, 342, 343, 344, 345, 346, 360, 363, 364, 365, 369, 379, 382, 384, 385, 386, 387, 388, 389, 390, 391, 401, 402, 403, 409, 410, 411, 418, 420, 421, 422, 423, 425, 441, 442, 443, 445, 460, 462, 463, 464, 465, 466, 473, 474, 475, 476, 477, 478, 479, 541, 542, 543, 544, 545, 591, 594, 627, 628, 629, 630, 631, 759, 762, 766, 795, 797, 798, 799, 806, 807, 817, 919, 1048 |
| `src/Platform/ClientEntry.cpp` | 1 | 567 |
| `src/Platform/Win32/Win32Audio.cpp` | 2 | 374, 375 |
| `src/Platform/Win32/Win32Diagnostics.h` | 1 | 81 |
| `src/Render/Ambient_Particles.cpp` | 7 | 128, 135, 136, 176, 266, 365, 366 |
| `src/Render/BMD_Anim.cpp` | 4 | 53, 106, 108, 111 |
| `src/Render/BMD_Collision.cpp` | 9 | 37, 42, 43, 53, 55, 56, 57, 58, 96 |
| `src/Render/BMD_DrawMesh.cpp` | 41 | 147, 159, 185, 191, 194, 205, 209, 218, 230, 236, 237, 238, 241, 252, 260, 261, 262, 270, 284, 314, 319, 320, 321, 353, 358, 360, 361, 362, 417, 418, 421, 424, 436, 439, 448, 459, 460, 469, 470, 471, 499 |
| `src/Render/BMD_LegacyDraw.cpp` | 40 | 60, 61, 63, 73, 76, 77, 85, 90, 108, 132, 134, 140, 1342, 1362, 1364, 1366, 1368, 1370, 1669, 1670, 1671, 1699, 1703, 1704, 1705, 1707, 1708, 1709, 1711, 1712, 1886, 1887, 1900, 1902, 1903, 1904, 1905, 1906, 1907, 1908 |
| `src/Render/BMD_LegacyEffects.cpp` | 13 | 29, 31, 32, 39, 90, 98, 99, 100, 102, 107, 108, 110, 118 |
| `src/Render/BMD_LegacyLoading.cpp` | 16 | 85, 139, 189, 341, 361, 362, 363, 401, 414, 415, 426, 427, 490, 544, 548, 562 |
| `src/Render/BMD_Load.cpp` | 132 | 56, 57, 58, 122, 157, 165, 166, 167, 169, 170, 171, 177, 178, 179, 180, 181, 182, 183, 184, 207, 208, 209, 215, 216, 217, 218, 219, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 249, 250, 251, 252, 255, 256, 257, 260, 265, 299, 301, 315, 316, 317, 318, 319, 320, 323, 324, 337, 339, 344, 347, 351, 355, 356, 357, 359, 360, 363, 364, 377, 378, 379, 393, 394, 395, 409, 411, 444, 447, 448, 451, 456, 471, 480, 494, 497, 501, 502, 505, 521, 525, 528, 533, 534, 537, 538, 539, 540, 541, 542, 543, 544, 545, 546, 547, 548, 549, 550, 551, 552, 553, 554, 555, 556, 557, 558, 559, 560, 564, 575, 578, 582, 584, 585, 625, 626, 627, 632, 652, 671, 672, 673 |
| `src/Render/BMD_SetupRender.cpp` | 125 | 38, 39, 40, 41, 42, 43, 44, 45, 46, 52, 53, 54, 55, 56, 57, 62, 63, 64, 66, 67, 69, 72, 73, 77, 78, 79, 81, 82, 84, 87, 88, 92, 93, 94, 96, 97, 99, 102, 103, 107, 108, 109, 112, 113, 115, 116, 118, 119, 122, 123, 125, 129, 130, 131, 134, 138, 139, 140, 141, 142, 143, 145, 150, 151, 152, 153, 154, 155, 160, 161, 162, 164, 170, 171, 172, 175, 176, 178, 179, 180, 181, 183, 184, 185, 186, 188, 189, 191, 192, 197, 202, 203, 204, 209, 210, 211, 215, 216, 217, 218, 219, 223, 226, 227, 232, 237, 238, 239, 242, 250, 251, 252, 253, 254, 255, 256, 257, 265, 267, 268, 269, 270, 271, 272, 302 |
| `src/Render/BMD_SkinUpdate.cpp` | 6 | 180, 200, 202, 205, 207, 209 |
| `src/Render/Backend/GL/MeshDraw.cpp` | 8 | 157, 158, 158, 159, 159, 160, 162, 163 |
| `src/Render/Effect_Combat.cpp` | 15 | 67, 70, 79, 80, 81, 82, 83, 105, 169, 171, 176, 182, 183, 184, 211 |
| `src/Render/Effect_Create.cpp` | 138 | 50, 52, 54, 55, 56, 57, 58, 59, 255, 256, 257, 521, 826, 827, 828, 830, 905, 906, 1061, 1112, 1120, 1155, 1219, 1245, 1281, 1325, 1339, 1368, 1369, 1370, 1439, 1449, 1451, 1505, 1507, 1555, 1617, 1618, 1632, 1639, 1713, 1714, 1736, 1803, 1804, 1818, 1825, 1851, 1853, 1901, 2002, 2013, 2015, 2069, 2071, 2119, 2181, 2182, 2196, 2203, 2277, 2278, 2300, 2368, 2369, 2383, 2390, 2465, 2466, 2486, 2546, 2557, 2559, 2612, 2614, 2662, 2724, 2725, 2739, 2746, 2820, 2821, 2843, 2904, 2915, 2969, 2970, 2984, 2991, 3008, 3010, 3058, 3119, 3120, 3142, 3202, 3213, 3215, 3270, 3271, 3319, 3381, 3382, 3396, 3403, 3467, 3468, 3491, 3558, 3559, 3573, 3580, 3654, 3656, 3676, 3748, 3762, 3763, 3785, 3844, 3855, 3857, 3913, 3914, 3957, 3970, 4019, 4030, 4032, 4086, 4088, 4136, 4158, 4160, 4180, 4181, 4192, 4193 |
| `src/Render/Effect_Tick.cpp` | 8 | 40, 41, 42, 242, 249, 250, 251, 253 |
| `src/Render/Entity_AnimationLegacy.cpp` | 56 | 56, 57, 72, 76, 77, 80, 81, 87, 88, 100, 145, 146, 147, 148, 149, 153, 168, 173, 181, 192, 193, 206, 207, 208, 209, 217, 230, 231, 234, 242, 243, 249, 250, 251, 263, 284, 286, 295, 306, 311, 312, 313, 315, 329, 330, 331, 345, 347, 350, 376, 377, 378, 379, 385, 386, 397 |
| `src/Render/Entity_DrawAt.cpp` | 20 | 118, 119, 120, 121, 122, 123, 124, 136, 137, 138, 140, 141, 143, 146, 147, 148, 150, 151, 153, 161 |
| `src/Render/Entity_DrawByType.cpp` | 126 | 87, 90, 91, 92, 93, 94, 107, 108, 109, 115, 116, 117, 127, 128, 129, 131, 132, 133, 136, 137, 141, 145, 146, 252, 253, 254, 260, 261, 262, 268, 269, 270, 289, 290, 291, 292, 298, 299, 300, 301, 341, 342, 343, 344, 345, 346, 347, 353, 354, 355, 356, 360, 361, 362, 363, 364, 365, 366, 372, 373, 374, 375, 414, 415, 416, 426, 430, 431, 432, 435, 436, 437, 441, 442, 443, 452, 453, 454, 458, 460, 461, 527, 528, 529, 534, 535, 536, 548, 549, 550, 551, 557, 558, 559, 560, 562, 563, 564, 571, 572, 573, 579, 580, 581, 603, 604, 605, 606, 612, 613, 614, 615, 670, 671, 672, 678, 679, 680, 697, 698, 699, 705, 706, 707, 733, 745 |
| `src/Render/Entity_DrawSetup.cpp` | 120 | 151, 155, 156, 157, 158, 159, 160, 161, 162, 166, 167, 168, 171, 172, 173, 176, 177, 178, 181, 182, 183, 185, 186, 187, 188, 189, 190, 194, 195, 196, 197, 198, 199, 200, 201, 202, 209, 210, 211, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 230, 231, 232, 235, 236, 241, 243, 244, 245, 246, 247, 248, 262, 263, 264, 267, 268, 269, 284, 285, 286, 289, 290, 291, 304, 307, 312, 321, 325, 326, 328, 329, 330, 334, 335, 336, 358, 359, 360, 370, 371, 372, 378, 379, 380, 387, 388, 389, 396, 397, 398, 407, 408, 409, 418, 419, 420, 432, 433, 434, 450, 454, 455, 456, 460, 461, 462, 463 |
| `src/Render/Entity_Render.cpp` | 36 | 188, 292, 293, 297, 298, 302, 311, 313, 330, 334, 339, 343, 347, 350, 368, 373, 414, 415, 416, 417, 418, 419, 420, 429, 430, 431, 434, 435, 437, 439, 440, 441, 444, 445, 447, 452 |
| `src/Render/Entity_Render_3D.cpp` | 47 | 155, 160, 179, 195, 197, 230, 245, 246, 247, 260, 261, 353, 355, 381, 384, 385, 391, 411, 422, 425, 428, 431, 440, 449, 452, 457, 461, 466, 467, 476, 484, 485, 486, 502, 529, 532, 539, 541, 545, 552, 554, 568, 574, 604, 608, 618, 625 |
| `src/Render/Entity_UpdateRender.cpp` | 116 | 56, 57, 58, 67, 70, 82, 90, 99, 121, 122, 124, 133, 134, 135, 136, 138, 139, 146, 151, 159, 165, 166, 170, 173, 181, 195, 199, 211, 213, 219, 228, 233, 234, 238, 239, 244, 253, 265, 266, 268, 273, 275, 276, 278, 279, 283, 294, 295, 317, 318, 319, 324, 325, 326, 349, 350, 351, 368, 369, 370, 378, 379, 380, 418, 420, 422, 423, 432, 445, 446, 459, 474, 486, 494, 496, 497, 500, 526, 539, 585, 590, 597, 601, 616, 617, 637, 646, 647, 648, 652, 658, 662, 666, 667, 691, 692, 693, 695, 709, 710, 712, 713, 714, 717, 718, 719, 723, 724, 728, 733, 734, 761, 762, 765, 766, 767 |
| `src/Render/Font_LegacyFactory.cpp` | 2 | 67, 68 |
| `src/Render/HUD_Pass1.cpp` | 8 | 168, 169, 170, 171, 364, 365, 485, 486 |
| `src/Render/HUD_Pass2.cpp` | 20 | 90, 325, 329, 430, 449, 455, 555, 575, 578, 581, 588, 591, 594, 658, 661, 665, 670, 671, 697, 698 |
| `src/Render/HUD_Pass3.cpp` | 7 | 209, 212, 213, 274, 276, 291, 293 |
| `src/Render/HUD_Pass5.cpp` | 8 | 62, 211, 215, 233, 279, 348, 399, 403 |
| `src/Render/HUD_Pass6.cpp` | 49 | 127, 128, 414, 452, 496, 585, 586, 686, 720, 796, 853, 876, 886, 905, 909, 924, 950, 977, 978, 980, 981, 985, 990, 991, 1010, 1011, 1027, 1035, 1037, 1038, 1039, 1042, 1043, 1045, 1047, 1048, 1050, 1057, 1058, 1065, 1081, 1095, 1101, 1127, 1162, 1519, 1690, 1726, 1817 |
| `src/Render/ItemDrop_Render2.cpp` | 3 | 241, 242, 270 |
| `src/Render/Joint_Create.cpp` | 54 | 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 73, 74, 75, 262, 263, 264, 324, 325, 326, 514, 559, 573, 754, 797, 810, 1343, 1344, 1345, 1348, 1349, 1350, 1659, 1660, 1661, 1820, 1821, 1822, 2121, 2122, 2136, 2144, 2163, 2205, 2267, 2268, 2376, 2377, 2391, 2399, 2428, 2470, 2532, 2533 |
| `src/Render/Joint_Render.cpp` | 3 | 61, 78, 90 |
| `src/Render/MaterialVisuals.cpp` | 6 | 84, 85, 146, 149, 154, 156 |
| `src/Render/MoveEffect.cpp` | 83 | 123, 124, 125, 126, 127, 133, 137, 144, 151, 152, 153, 159, 165, 166, 167, 169, 172, 173, 174, 177, 178, 181, 195, 200, 201, 202, 206, 213, 218, 219, 220, 223, 231, 243, 472, 509, 674, 678, 683, 684, 685, 815, 818, 823, 882, 890, 945, 987, 988, 990, 1064, 1221, 1222, 1223, 1243, 1286, 1287, 1288, 1294, 1383, 1599, 1672, 1673, 1675, 1692, 1693, 1707, 1708, 1709, 1759, 1795, 1820, 1821, 1823, 1824, 1831, 1835, 1838, 2066, 2384, 2426, 2457, 2458 |
| `src/Render/MoveEffect_Helpers.cpp` | 11 | 172, 173, 174, 177, 178, 212, 215, 217, 219, 220, 225 |
| `src/Render/MoveJoint.cpp` | 61 | 113, 114, 115, 131, 132, 133, 151, 400, 401, 402, 404, 405, 406, 407, 446, 447, 448, 449, 450, 451, 452, 453, 454, 537, 538, 539, 580, 604, 644, 645, 646, 825, 826, 827, 950, 951, 952, 997, 998, 999, 1001, 1002, 1003, 1023, 1024, 1025, 1049, 1050, 1051, 1083, 1084, 1085, 1118, 1119, 1120, 1297, 1298, 1515, 1516, 1517, 1637 |
| `src/Render/Particle.cpp` | 37 | 50, 51, 52, 53, 55, 56, 57, 59, 63, 64, 66, 69, 70, 99, 100, 102, 105, 109, 120, 121, 122, 125, 127, 187, 188, 189, 190, 191, 197, 198, 199, 200, 201, 202, 203, 204, 205 |
| `src/Render/Particle_Legacy.cpp` | 1 | 19 |
| `src/Render/Particle_Move.cpp` | 56 | 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 64, 197, 198, 199, 219, 220, 221, 351, 401, 405, 832, 834, 835, 861, 862, 867, 873, 874, 875, 968, 969, 970, 997, 998, 999, 1055, 1056, 1057, 1059, 1060, 1061 |
| `src/Render/Particle_Render.cpp` | 10 | 112, 113, 115, 150, 151, 152, 153, 154, 155, 156 |
| `src/Render/Particle_Spawn.cpp` | 166 | 98, 115, 137, 184, 197, 255, 323, 325, 329, 330, 332, 333, 334, 336, 337, 338, 340, 342, 343, 347, 350, 351, 355, 356, 367, 371, 379, 381, 383, 388, 390, 394, 397, 398, 399, 400, 401, 407, 408, 410, 412, 413, 415, 421, 422, 423, 425, 440, 442, 444, 445, 451, 452, 455, 459, 461, 465, 468, 469, 479, 481, 482, 491, 498, 499, 502, 503, 504, 505, 507, 508, 514, 515, 525, 526, 535, 543, 544, 557, 558, 559, 560, 561, 562, 563, 564, 565, 566, 574, 582, 584, 585, 591, 593, 603, 604, 605, 623, 625, 627, 633, 634, 640, 641, 644, 646, 650, 651, 654, 656, 657, 663, 665, 666, 669, 671, 672, 677, 678, 679, 680, 683, 685, 686, 688, 690, 691, 702, 703, 724, 725, 727, 732, 736, 760, 763, 766, 767, 768, 769, 770, 771, 778, 780, 781, 783, 784, 785, 788, 789, 790, 806, 816, 818, 830, 831, 832, 839, 840, 846, 847, 848, 855, 857, 859, 862 |
| `src/Render/Player_Render.cpp` | 1 | 91 |
| `src/Render/RenderLinkObject.cpp` | 11 | 95, 113, 140, 302, 577, 578, 588, 601, 609, 610, 611 |
| `src/Render/Render_Frame.cpp` | 10 | 418, 419, 420, 535, 536, 537, 559, 560, 561, 669 |
| `src/Render/Render_LegacyBillboards.cpp` | 31 | 55, 57, 58, 60, 61, 62, 63, 66, 67, 229, 230, 236, 239, 240, 244, 245, 248, 250, 251, 253, 271, 272, 286, 287, 288, 293, 297, 298, 314, 315, 318 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 3 | 145, 146, 153 |
| `src/Render/Render_LegacyLinker.cpp` | 18 | 91, 92, 105, 364, 365, 405, 413, 414, 431, 432, 439, 441, 445, 446, 447, 456, 457, 466 |
| `src/Render/Render_LegacyTrails.cpp` | 9 | 59, 60, 61, 62, 63, 64, 82, 94, 102 |
| `src/Render/Render_PlayerEquipment.cpp` | 39 | 76, 127, 133, 188, 203, 206, 221, 227, 507, 535, 589, 604, 639, 640, 677, 679, 713, 766, 767, 771, 773, 776, 777, 780, 781, 782, 783, 786, 787, 795, 796, 799, 800, 814, 815, 816, 828, 829, 833 |
| `src/Render/Render_WorldHelpers.cpp` | 44 | 156, 157, 158, 217, 219, 220, 221, 232, 268, 273, 274, 275, 278, 279, 280, 284, 291, 292, 296, 297, 298, 312, 313, 446, 617, 724, 794, 797, 798, 807, 827, 872, 876, 894, 895, 896, 897, 898, 916, 1104, 1105, 1117, 1118, 1119 |
| `src/Render/SMD_Parser.cpp` | 72 | 107, 108, 122, 186, 188, 191, 192, 193, 196, 202, 203, 208, 209, 210, 211, 212, 213, 214, 216, 220, 221, 222, 223, 224, 246, 248, 252, 253, 299, 303, 351, 363, 368, 369, 374, 376, 382, 383, 384, 390, 391, 392, 394, 396, 417, 426, 430, 432, 433, 442, 443, 449, 457, 474, 488, 492, 493, 494, 496, 497, 498, 546, 774, 775, 778, 780, 785, 786, 788, 862, 898, 905 |
| `src/Render/Sprite.cpp` | 1 | 149 |
| `src/Render/Terrain_Render.cpp` | 13 | 64, 66, 70, 72, 76, 77, 80, 87, 90, 102, 124, 125, 126 |
| `src/Render/Trail_Render.cpp` | 3 | 53, 75, 92 |
| `src/Render/Weather.cpp` | 33 | 190, 191, 203, 239, 240, 261, 264, 265, 274, 281, 284, 285, 294, 295, 346, 350, 351, 433, 464, 469, 470, 475, 513, 530, 531, 572, 643, 644, 701, 702, 734, 735, 763 |
| `src/Render/Weather_Particles.cpp` | 17 | 104, 105, 107, 108, 111, 133, 134, 135, 163, 164, 166, 180, 181, 182, 196, 197, 198 |
| `src/Render/WorldEffects.cpp` | 3 | 62, 63, 68 |
| `src/Scene/Scene_CharSelect.cpp` | 6 | 262, 339, 340, 341, 395, 467 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 167 | 70, 80, 82, 92, 94, 97, 99, 102, 114, 122, 137, 141, 143, 148, 149, 153, 158, 172, 173, 181, 183, 184, 189, 190, 193, 195, 198, 199, 202, 203, 308, 321, 323, 325, 368, 373, 395, 398, 399, 400, 496, 508, 534, 535, 547, 549, 552, 568, 569, 570, 583, 588, 597, 598, 618, 622, 625, 627, 628, 633, 658, 689, 690, 692, 693, 697, 698, 699, 700, 701, 705, 706, 707, 708, 709, 710, 711, 712, 713, 714, 715, 716, 717, 718, 719, 720, 721, 722, 976, 977, 996, 997, 1020, 1040, 1041, 1063, 1064, 1091, 1263, 1272, 1273, 1314, 1322, 1397, 1422, 1424, 1588, 1593, 1615, 1620, 1622, 1653, 1724, 1726, 1736, 1738, 1742, 1877, 2121, 2131, 2134, 2136, 2170, 2187, 2188, 2223, 2230, 2236, 2242, 2254, 2258, 2262, 2269, 2270, 2272, 2273, 2277, 2278, 2283, 2284, 2287, 2316, 2320, 2322, 2325, 2331, 2333, 2369, 2370, 2377, 2382, 2384, 2387, 2389, 2391, 2393, 2394, 2400, 2404, 2409, 2410, 2420, 2427, 2455, 2456, 2458, 2460 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 24 | 67, 75, 76, 77, 78, 79, 80, 87, 88, 90, 117, 121, 131, 260, 281, 286, 287, 288, 298, 299, 300, 303, 304, 305 |
| `src/Scene/Scene_Intro.cpp` | 9 | 151, 225, 257, 258, 270, 276, 301, 302, 319 |
| `src/Scene/Scene_LegacyHelpers.cpp` | 12 | 28, 31, 46, 47, 50, 51, 55, 56, 61, 62, 66, 67 |
| `src/Scene/Scene_Login_ServerSelect.cpp` | 1 | 212 |
| `src/Scene/Scene_MapTick.cpp` | 3 | 42, 55, 65 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 2 | 55, 100 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 167 | 45, 46, 247, 248, 463, 464, 494, 495, 496, 505, 506, 507, 515, 516, 517, 525, 526, 527, 571, 572, 573, 595, 611, 653, 654, 677, 725, 743, 759, 761, 762, 764, 765, 768, 786, 795, 796, 825, 826, 831, 839, 841, 843, 849, 850, 851, 852, 905, 946, 948, 999, 1001, 1059, 1060, 1067, 1074, 1078, 1083, 1088, 1092, 1098, 1102, 1103, 1104, 1105, 1118, 1124, 1126, 1136, 1137, 1138, 1139, 1145, 1151, 1155, 1168, 1178, 1183, 1184, 1211, 1218, 1227, 1228, 1233, 1234, 1235, 1236, 1240, 1242, 1244, 1246, 1248, 1250, 1252, 1254, 1260, 1262, 1264, 1266, 1268, 1270, 1272, 1274, 1286, 1293, 1294, 1297, 1302, 1303, 1306, 1307, 1310, 1311, 1318, 1322, 1327, 1331, 1332, 1335, 1358, 1361, 1363, 1365, 1366, 1370, 1373, 1374, 1378, 1382, 1385, 1386, 1387, 1390, 1392, 1398, 1399, 1403, 1404, 1409, 1411, 1412, 1419, 1422, 1423, 1427, 1429, 1430, 1435, 1436, 1438, 1445, 1448, 1451, 1452, 1455, 1463, 1466, 1471, 1474, 1475, 1478, 1479, 1480, 1481, 1493, 1494, 1497 |
| `src/Scene/Scene_Objects.cpp` | 9 | 61, 76, 431, 434, 435, 436, 437, 438, 439 |
| `src/Scene/Scene_Resources.cpp` | 4 | 30, 31, 150, 151 |
| `src/Scene/Scene_ServerSelect_Input.cpp` | 1 | 191 |
| `src/Sound/Sound_DS3D.cpp` | 4 | 91, 92, 93, 161 |
| `src/Sound/Sound_Queue.cpp` | 10 | 34, 36, 37, 55, 101, 102, 109, 110, 111, 113 |
| `src/Terrain/Terrain.cpp` | 3 | 465, 466, 656 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 14 | 128, 183, 192, 193, 196, 283, 295, 687, 689, 690, 691, 692, 693, 694 |
| `src/Terrain/Terrain_Light.cpp` | 21 | 97, 98, 99, 100, 103, 104, 107, 108, 109, 112, 113, 114, 124, 137, 138, 139, 142, 144, 154, 155, 156 |
| `src/Trade/Trade.cpp` | 16 | 162, 163, 243, 267, 287, 311, 312, 322, 399, 420, 421, 455, 456, 485, 553, 555 |
| `src/UI/CharMenu_Build.cpp` | 5 | 660, 661, 679, 680, 681 |
| `src/UI/Chat.cpp` | 1 | 506 |
| `src/UI/ChatListBox.cpp` | 64 | 888, 1249, 1251, 1254, 1269, 1271, 1300, 1447, 1449, 1452, 1460, 1864, 1867, 1871, 2413, 2414, 2415, 2508, 2509, 2510, 2511, 2512, 2513, 2515, 2517, 2528, 2551, 2552, 2559, 2560, 2561, 2566, 2567, 2568, 2645, 2647, 2664, 2684, 2703, 2729, 2730, 2732, 2734, 2735, 2736, 2738, 2741, 2745, 2750, 2756, 2757, 2758, 2760, 2761, 2762, 2763, 2770, 2775, 2782, 2788, 2789, 2875, 2894, 2904 |
| `src/UI/Chat_Bubbles.cpp` | 3 | 24, 262, 275 |
| `src/UI/Chat_Gestures.cpp` | 3 | 28, 38, 41 |
| `src/UI/Chat_InputTick.cpp` | 24 | 70, 79, 99, 138, 141, 142, 151, 158, 170, 364, 473, 493, 499, 514, 518, 531, 618, 619, 620, 657, 714, 715, 716, 837 |
| `src/UI/Chat_Log.cpp` | 1 | 83 |
| `src/UI/Chat_Send.cpp` | 1 | 70 |
| `src/UI/GoldenArcher.cpp` | 12 | 105, 112, 113, 114, 115, 123, 131, 513, 1001, 1027, 1028, 1029 |
| `src/UI/MiniMap.cpp` | 5 | 45, 46, 260, 261, 279 |
| `src/UI/MoveList.cpp` | 4 | 110, 115, 117, 118 |
| `src/UI/Quest_Legacy.cpp` | 12 | 29, 79, 80, 90, 93, 98, 156, 186, 207, 212, 229, 266 |
| `src/UI/RenderItemInfo.cpp` | 16 | 43, 1153, 1154, 1155, 1156, 1674, 1676, 1678, 1682, 1830, 1843, 1906, 2129, 2140, 2158, 2188 |
| `src/UI/UI_GuildLegacy.cpp` | 2 | 111, 115 |
| `src/UI/UI_InGameMenu.cpp` | 2 | 93, 435 |
| `src/UI/UI_LegacyExterns.cpp` | 11 | 150, 194, 200, 201, 202, 222, 302, 342, 344, 345, 423 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 1 | 80 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 63 | 65, 65, 66, 66, 67, 67, 68, 69, 70, 71, 71, 72, 72, 73, 73, 74, 75, 83, 84, 97, 99, 100, 102, 105, 106, 107, 108, 109, 111, 215, 216, 222, 223, 229, 231, 232, 233, 234, 235, 236, 238, 250, 255, 256, 259, 262, 265, 266, 267, 268, 269, 270, 277, 279, 288, 292, 295, 297, 307, 320, 337, 338, 385 |
| `src/UI/UI_SkillHotkeys.cpp` | 1 | 81 |
| `src/Util/Misc.cpp` | 14 | 60, 61, 68, 80, 81, 90, 100, 101, 131, 136, 169, 170, 178, 181 |
| `src/globals.cpp` | 1 | 574 |
| `src/globals.h` | 69 | 470, 471, 474, 552, 553, 554, 555, 556, 557, 558, 559, 560, 561, 562, 563, 564, 565, 566, 567, 568, 569, 570, 571, 581, 582, 583, 584, 585, 586, 587, 588, 589, 590, 591, 592, 593, 594, 595, 596, 597, 598, 599, 600, 602, 603, 604, 629, 905, 906, 907, 908, 961, 962, 1045, 1046, 1047, 1048, 1155, 1158, 1168, 1169, 1170, 1171, 1585, 2035, 2036, 2330, 2339, 2340 |

### 2c items inventario (referencias tipadas)

Expresión (Python re):

```text
\b(?:ITEM|AsItem)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Skills_WarriorLegacy.cpp` | 11 | 104, 104, 104, 105, 124, 124, 124, 125, 125, 126, 126 |
| `src/Core/Runtime_Small.cpp` | 23 | 194, 196, 197, 197, 199, 200, 200, 201, 293, 294, 295, 356, 362, 363, 364, 365, 388, 394, 395, 396, 397, 433, 434 |
| `src/Entity/EntityTypes.h` | 2 | 150, 234 |
| `src/Entity/Entity_Spawn.cpp` | 1 | 246 |
| `src/Game/Player_InputTick.cpp` | 7 | 430, 431, 431, 432, 433, 437, 438 |
| `src/Game/RenderDerivedState.cpp` | 3 | 66, 67, 67 |
| `src/Item/ChaosMixRates.cpp` | 1 | 14 |
| `src/Item/ChaosMixRates.h` | 1 | 9 |
| `src/Item/ContentCatalog.cpp` | 10 | 356, 356, 391, 391, 451, 451, 462, 462, 475, 479 |
| `src/Item/Item_ChaosMix.cpp` | 11 | 118, 214, 360, 372, 397, 398, 415, 456, 472, 474, 481 |
| `src/Item/Item_ClickHandler.cpp` | 38 | 435, 435, 436, 437, 440, 440, 443, 443, 446, 446, 447, 447, 449, 453, 457, 457, 458, 458, 468, 469, 472, 472, 479, 481, 487, 498, 523, 523, 531, 531, 552, 665, 748, 1015, 1025, 1033, 1033, 1176 |
| `src/Item/Item_Display.cpp` | 2 | 611, 635 |
| `src/Item/Item_Durability.cpp` | 12 | 45, 52, 52, 53, 53, 53, 94, 130, 133, 225, 357, 379 |
| `src/Item/Item_Equip.cpp` | 7 | 50, 112, 115, 119, 123, 177, 238 |
| `src/Item/Item_EquipmentAutoSwap.cpp` | 5 | 77, 77, 77, 78, 83 |
| `src/Item/Item_Inventory.cpp` | 34 | 25, 37, 60, 60, 72, 72, 127, 205, 206, 207, 214, 215, 219, 220, 225, 226, 227, 228, 229, 230, 234, 234, 235, 278, 278, 278, 280, 336, 336, 338, 350, 386, 387, 389 |
| `src/Item/Item_InventoryGrid.cpp` | 11 | 79, 80, 204, 205, 205, 209, 211, 228, 229, 229, 230 |
| `src/Item/Item_InventoryRender.cpp` | 26 | 12, 41, 46, 51, 54, 57, 60, 65, 68, 71, 81, 92, 109, 111, 116, 132, 229, 346, 346, 410, 410, 440, 440, 441, 441, 441 |
| `src/Item/Item_LegacyGridInsert.cpp` | 20 | 56, 63, 65, 91, 92, 99, 99, 110, 116, 117, 118, 119, 134, 146, 157, 159, 166, 167, 169, 170 |
| `src/Item/Item_LegacyLinker.cpp` | 6 | 189, 189, 192, 205, 205, 525 |
| `src/Item/Item_Move.cpp` | 3 | 66, 91, 91 |
| `src/Item/Item_ServerValue.cpp` | 7 | 41, 80, 103, 155, 157, 157, 157 |
| `src/Item/Item_SpecialOption.cpp` | 1 | 12 |
| `src/Item/Item_TypeLegacy.cpp` | 2 | 41, 43 |
| `src/Item/RightClickMove.cpp` | 7 | 13, 36, 36, 38, 38, 39, 49 |
| `src/Item/RightClickMove.h` | 5 | 14, 18, 18, 21, 21 |
| `src/Net/Protocol/GameServerProtocol.h` | 3 | 692, 716, 730 |
| `src/Net/Recv/NetRecv.h` | 2 | 68, 127 |
| `src/Net/Recv/Recv_Character.cpp` | 4 | 1670, 1670, 1671, 1676 |
| `src/Net/Recv/Recv_Item.cpp` | 48 | 21, 23, 40, 41, 41, 42, 42, 43, 43, 44, 50, 51, 52, 52, 98, 126, 165, 165, 166, 177, 197, 216, 216, 217, 319, 319, 321, 323, 323, 324, 324, 325, 325, 326, 762, 786, 787, 788, 888, 914, 925, 929, 931, 932, 933, 934, 947, 953 |
| `src/Net/Recv/Recv_Trade.cpp` | 3 | 30, 31, 31 |
| `src/Net/Recv/Recv_Viewport.cpp` | 11 | 65, 66, 66, 809, 818, 819, 820, 823, 824, 825, 843 |
| `src/Net/SecondPassword.cpp` | 17 | 1479, 1479, 1481, 1487, 1492, 1532, 1533, 1534, 1535, 1536, 1602, 1602, 2905, 2905, 2905, 2906, 2908 |
| `src/Render/HUD_Pass4.cpp` | 11 | 499, 505, 506, 510, 518, 519, 560, 560, 561, 561, 561 |
| `src/Render/HUD_Pass5.cpp` | 8 | 174, 174, 176, 176, 177, 177, 177, 183 |
| `src/Render/HUD_Pass6.cpp` | 13 | 251, 251, 272, 959, 959, 960, 960, 1518, 1519, 1519, 1521, 1527, 1527 |
| `src/Render/MaterialVisuals.cpp` | 7 | 26, 26, 26, 27, 27, 27, 27 |
| `src/Render/Render_WorldHelpers.cpp` | 3 | 616, 617, 617 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 4 | 26, 2148, 2153, 2154 |
| `src/UI/RenderItemInfo.cpp` | 26 | 131, 370, 460, 742, 847, 1036, 1117, 1331, 1411, 1423, 1744, 1745, 1754, 1754, 1754, 1756, 1756, 1757, 1758, 1768, 1887, 1900, 1900, 1929, 1956, 1956 |
| `src/functions.h` | 8 | 439, 440, 456, 1168, 1173, 1174, 1184, 1302 |
| `src/structs.h` | 41 | 220, 250, 251, 252, 253, 254, 255, 256, 257, 258, 259, 260, 261, 262, 263, 264, 265, 266, 267, 268, 269, 270, 271, 272, 273, 274, 275, 276, 277, 279, 279, 279, 280, 280, 280, 281, 281, 281, 289, 289, 293 |

### 2d daño (referencias al pool)

Expresión (Python re):

```text
\bDAT_07c80110\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Skills_WarriorLegacy.cpp` | 1 | 438 |
| `src/Game/RenderSnapshots.cpp` | 1 | 168 |
| `src/Render/Effect_Tick.cpp` | 3 | 329, 330, 330 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 1 | 138 |
| `src/Render/Render_LegacyTrails.cpp` | 1 | 55 |
| `src/globals.cpp` | 1 | 1254 |
| `src/globals.h` | 1 | 1213 |

### 2e entidades y efectos (referencias tipadas)

Expresión (Python re):

```text
\b(?:OBJECT|CHARACTER|AsObject|AsCharacter)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 169 | 545, 583, 586, 643, 906, 907, 932, 932, 935, 936, 942, 942, 952, 952, 1001, 1001, 1011, 1011, 1012, 1012, 1012, 1012, 1013, 1013, 1015, 1015, 1019, 1019, 1038, 1061, 1071, 1071, 1094, 1094, 1095, 1095, 1095, 1095, 1096, 1096, 1103, 1107, 1107, 1108, 1108, 1108, 1108, 1109, 1109, 1123, 1131, 1136, 1136, 1137, 1137, 1137, 1137, 1138, 1138, 1146, 1146, 1153, 1185, 1190, 1190, 1193, 1194, 1218, 1218, 1221, 1222, 1227, 1227, 1238, 1238, 1239, 1239, 1239, 1239, 1240, 1240, 1241, 1241, 1247, 1248, 1258, 1258, 1264, 1277, 1283, 1284, 1289, 1290, 1299, 1300, 1300, 1301, 1301, 1353, 1354, 1390, 1413, 1413, 1420, 1446, 1446, 1447, 1447, 1449, 1449, 1495, 1496, 1511, 1524, 1544, 1544, 1545, 1545, 1562, 1564, 1565, 1572, 1574, 1595, 1611, 1741, 1742, 1843, 1844, 1859, 1860, 1861, 1862, 1863, 1863, 1864, 1864, 1935, 2004, 2084, 2113, 2133, 2135, 2136, 2137, 2142, 2142, 2145, 2146, 2182, 2183, 2204, 2206, 2207, 2210, 2248, 2249, 2312, 2320, 2347, 2351, 2363, 2426, 2429, 2430, 2445, 2462, 2482, 2499 |
| `src/Combat/Combat_AttackEffect.cpp` | 242 | 379, 379, 382, 495, 575, 581, 581, 582, 582, 583, 583, 697, 699, 699, 702, 702, 703, 703, 704, 704, 707, 707, 708, 708, 709, 709, 713, 713, 714, 714, 715, 715, 730, 730, 740, 742, 742, 850, 852, 852, 853, 853, 854, 854, 855, 855, 856, 856, 859, 859, 879, 883, 883, 985, 985, 986, 986, 987, 987, 1000, 1002, 1002, 1003, 1009, 1011, 1011, 1012, 1012, 1013, 1013, 1025, 1140, 1142, 1142, 1144, 1144, 1145, 1145, 1148, 1154, 1154, 1155, 1155, 1156, 1156, 1277, 1283, 1286, 1299, 1299, 1300, 1412, 1421, 1421, 1422, 1422, 1424, 1424, 1433, 1434, 1434, 1443, 1443, 1444, 1444, 1445, 1445, 1446, 1446, 1447, 1447, 1449, 1449, 1475, 1475, 1476, 1476, 1477, 1477, 1478, 1478, 1498, 1498, 1499, 1499, 1500, 1500, 1501, 1501, 1620, 1630, 1630, 1631, 1631, 1633, 1633, 1759, 1759, 1760, 1760, 1761, 1761, 1766, 1776, 1776, 1777, 1777, 1778, 1778, 1788, 1797, 1800, 1808, 1808, 1820, 1829, 1829, 1842, 1842, 1852, 1860, 1860, 1872, 1877, 1882, 1882, 1897, 1910, 1910, 1911, 1911, 1912, 1912, 1913, 1913, 1914, 1914, 1915, 1915, 1929, 1929, 1929, 1931, 1931, 1932, 1932, 1933, 1933, 1934, 1934, 1949, 1953, 1953, 1954, 1954, 1955, 1955, 1969, 1971, 1971, 1979, 1979, 1980, 1980, 1981, 1981, 1994, 1996, 1996, 1999, 1999, 2000, 2000, 2001, 2001, 2014, 2021, 2021, 2034, 2042, 2042, 2054, 2054, 2061, 2061, 2062, 2065, 2065, 2078, 2078, 2090, 2113, 2118, 2123, 2123, 2138, 2138, 2145, 2145, 2145, 2148, 2148 |
| `src/Combat/Combat_AttackStage.cpp` | 110 | 18, 29, 35, 45, 46, 47, 55, 71, 84, 95, 96, 97, 117, 123, 124, 136, 145, 148, 149, 150, 166, 169, 170, 171, 190, 191, 192, 193, 201, 203, 204, 206, 210, 216, 217, 221, 222, 223, 227, 239, 243, 248, 256, 265, 276, 283, 283, 285, 286, 287, 309, 330, 330, 331, 337, 339, 339, 339, 345, 346, 350, 351, 353, 357, 358, 358, 358, 358, 359, 359, 359, 359, 360, 360, 361, 361, 361, 361, 362, 362, 362, 362, 362, 363, 363, 363, 363, 364, 364, 364, 364, 364, 368, 371, 371, 371, 371, 371, 371, 371, 371, 380, 380, 380, 385, 386, 387, 388, 389, 390 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 65 | 57, 70, 71, 71, 72, 72, 73, 73, 185, 185, 186, 244, 248, 248, 249, 249, 259, 259, 274, 283, 298, 307, 309, 320, 320, 320, 322, 322, 323, 323, 323, 323, 331, 333, 333, 333, 333, 333, 333, 340, 342, 342, 342, 342, 342, 342, 348, 350, 350, 350, 350, 350, 350, 375, 384, 384, 385, 385, 399, 401, 411, 421, 421, 422, 422 |
| `src/Combat/Combat_Projectiles.cpp` | 2 | 18, 19 |
| `src/Combat/Combat_Targeting.cpp` | 6 | 39, 40, 41, 61, 62, 63 |
| `src/Combat/Skills.cpp` | 30 | 73, 76, 76, 79, 82, 82, 154, 156, 162, 172, 173, 173, 174, 174, 174, 174, 175, 175, 209, 262, 324, 324, 325, 325, 368, 372, 372, 372, 390, 417 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 51 | 89, 89, 89, 89, 89, 92, 93, 93, 97, 97, 98, 98, 155, 156, 164, 164, 165, 167, 206, 207, 208, 212, 212, 219, 220, 221, 234, 235, 243, 258, 259, 261, 279, 281, 282, 320, 321, 322, 324, 338, 338, 339, 341, 341, 345, 345, 346, 350, 467, 468, 469 |
| `src/Combat/Skills_WizardElf.cpp` | 19 | 92, 93, 94, 192, 193, 200, 200, 464, 465, 480, 480, 481, 481, 503, 504, 509, 509, 545, 545 |
| `src/Entity/CharacterCloth.cpp` | 19 | 31, 31, 52, 112, 112, 112, 143, 143, 143, 143, 144, 144, 144, 145, 175, 178, 180, 187, 208 |
| `src/Entity/CharacterEffects.cpp` | 6 | 28, 30, 112, 114, 117, 117 |
| `src/Entity/EntityName.cpp` | 18 | 16, 23, 25, 27, 36, 38, 40, 40, 43, 45, 49, 50, 54, 56, 59, 61, 62, 63 |
| `src/Entity/EntityName.h` | 6 | 20, 21, 21, 25, 26, 29 |
| `src/Entity/EntityTypes.h` | 92 | 20, 62, 102, 103, 103, 169, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 183, 184, 185, 186, 187, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 246, 246, 246, 247, 247, 247, 248, 248, 248, 249, 249, 249, 250, 250, 250, 251, 251, 251 |
| `src/Entity/Entity_ActionLegacy.cpp` | 7 | 62, 69, 71, 72, 72, 73, 74 |
| `src/Entity/Entity_Init.cpp` | 22 | 33, 33, 34, 34, 35, 35, 38, 38, 39, 39, 42, 43, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 3 | 99, 100, 101 |
| `src/Entity/Entity_MoveClient.cpp` | 7 | 31, 31, 31, 31, 31, 35, 40 |
| `src/Entity/Entity_Reset.cpp` | 52 | 50, 51, 52, 53, 54, 55, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 76, 82, 96, 99, 104, 107, 109, 116, 119, 124, 128, 130, 145, 148, 151, 158, 160, 162, 171, 175, 182, 184, 186, 189, 191, 195, 197, 208, 211, 214, 222 |
| `src/Entity/Entity_Spawn.cpp` | 4 | 23, 23, 24, 24 |
| `src/Entity/ObjectPose.cpp` | 34 | 18, 19, 20, 21, 22, 24, 26, 27, 28, 30, 31, 32, 41, 42, 43, 51, 51, 65, 65, 82, 82, 283, 310, 310, 347, 368, 394, 394, 396, 465, 465, 465, 465, 465 |
| `src/Game/Game_SceneUpdate.cpp` | 21 | 63, 66, 69, 74, 79, 85, 86, 87, 88, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130 |
| `src/Game/Party_NameMatch.cpp` | 5 | 30, 30, 30, 31, 31 |
| `src/Game/Player_InputTick.cpp` | 115 | 688, 688, 692, 692, 697, 697, 700, 700, 701, 701, 705, 705, 706, 706, 730, 741, 742, 743, 744, 747, 751, 755, 755, 776, 778, 778, 781, 789, 789, 797, 797, 818, 818, 822, 822, 822, 823, 824, 825, 826, 831, 833, 833, 834, 834, 835, 835, 836, 836, 838, 838, 859, 932, 933, 933, 934, 934, 935, 935, 936, 937, 938, 939, 1123, 1123, 1124, 1124, 1126, 1126, 1177, 1184, 1185, 1186, 1205, 1206, 1250, 1251, 1255, 1260, 1260, 1261, 1261, 1265, 1266, 1329, 1381, 1383, 1401, 1403, 1407, 1408, 1428, 1428, 1429, 1435, 1438, 1438, 1439, 1439, 1442, 1443, 1444, 1445, 1466, 1480, 1481, 1489, 1589, 1590, 1605, 1605, 1606, 1606, 1607, 1623 |
| `src/Game/RenderDerivedState.cpp` | 14 | 43, 61, 61, 62, 62, 62, 62, 64, 67, 67, 69, 70, 71, 90 |
| `src/Game/RenderInterpolation.cpp` | 33 | 16, 16, 16, 18, 18, 20, 21, 22, 23, 23, 24, 24, 27, 27, 27, 27, 29, 30, 31, 32, 32, 33, 33, 33, 33, 72, 73, 75, 143, 143, 146, 157, 165 |
| `src/Game/RenderSnapshots.cpp` | 38 | 100, 100, 101, 102, 107, 109, 109, 111, 111, 113, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 129, 129, 129, 130, 131, 131, 132, 134, 137, 146, 146, 146, 147, 148, 162, 189, 331 |
| `src/Game/SimulationPreparation.cpp` | 16 | 75, 100, 100, 122, 123, 123, 142, 216, 216, 217, 217, 223, 223, 224, 225, 226 |
| `src/Input/Input_WndProc.cpp` | 5 | 189, 199, 206, 319, 330 |
| `src/Input/Mouse_Hover.cpp` | 11 | 386, 387, 391, 391, 413, 413, 424, 425, 426, 436, 437 |
| `src/Item/Item_ClickHandler.cpp` | 6 | 561, 562, 565, 566, 570, 571 |
| `src/Item/Item_Equip.cpp` | 33 | 62, 63, 66, 66, 69, 81, 82, 91, 91, 91, 91, 98, 98, 102, 102, 112, 112, 139, 140, 141, 143, 158, 158, 167, 167, 169, 169, 176, 177, 234, 234, 238, 238 |
| `src/Item/Item_Equip.h` | 2 | 5, 5 |
| `src/Monster/Monster.cpp` | 104 | 596, 596, 597, 603, 603, 609, 610, 611, 612, 613, 614, 615, 616, 617, 618, 619, 622, 622, 629, 629, 630, 630, 631, 631, 638, 638, 660, 662, 669, 669, 675, 676, 677, 678, 684, 684, 690, 690, 697, 697, 698, 698, 699, 699, 705, 706, 707, 708, 709, 716, 717, 718, 719, 725, 726, 727, 728, 734, 735, 736, 742, 743, 744, 745, 1032, 1032, 1033, 1034, 1035, 1036, 1037, 1038, 1038, 1039, 1039, 1040, 1040, 1041, 1041, 1042, 1043, 1066, 1066, 1070, 1072, 1073, 1073, 1086, 1087, 1087, 1087, 1087, 1092, 1092, 1106, 1108, 1111, 1111, 1113, 1113, 1116, 1116, 1119, 1119 |
| `src/Net/Net_LegacyRuntime.cpp` | 1 | 15 |
| `src/Net/Recv/Recv_Character.cpp` | 72 | 204, 212, 503, 504, 505, 506, 507, 508, 511, 511, 512, 512, 513, 513, 516, 518, 524, 524, 525, 525, 526, 526, 527, 528, 530, 530, 536, 548, 689, 692, 694, 695, 696, 696, 697, 698, 743, 744, 744, 744, 744, 746, 746, 746, 746, 748, 748, 748, 748, 750, 750, 1021, 1022, 1508, 1508, 1509, 1509, 1516, 1516, 1518, 1518, 1521, 1522, 1523, 1523, 1553, 1553, 1555, 1555, 1619, 1619, 1640 |
| `src/Net/Recv/Recv_Combat.cpp` | 59 | 386, 386, 390, 390, 391, 394, 394, 394, 395, 395, 395, 399, 399, 400, 401, 403, 416, 417, 417, 418, 534, 538, 539, 544, 544, 562, 562, 563, 564, 656, 657, 704, 705, 717, 717, 718, 718, 724, 724, 735, 735, 739, 739, 740, 740, 741, 741, 812, 813, 854, 854, 854, 854, 855, 855, 856, 856, 857, 858 |
| `src/Net/Recv/Recv_Guild.cpp` | 15 | 102, 102, 104, 106, 107, 114, 115, 525, 526, 526, 527, 532, 533, 542, 543 |
| `src/Net/Recv/Recv_Viewport.cpp` | 146 | 116, 131, 131, 132, 132, 132, 135, 135, 136, 137, 137, 140, 140, 140, 141, 142, 142, 145, 145, 145, 146, 147, 147, 150, 150, 150, 151, 152, 152, 155, 155, 155, 156, 157, 157, 160, 160, 160, 161, 162, 162, 165, 166, 166, 170, 173, 174, 215, 223, 224, 225, 232, 233, 235, 236, 238, 242, 242, 244, 247, 280, 281, 282, 283, 284, 362, 363, 364, 365, 379, 379, 382, 384, 385, 386, 387, 387, 388, 389, 432, 438, 439, 440, 447, 451, 455, 462, 462, 463, 463, 464, 464, 465, 466, 467, 468, 469, 470, 471, 471, 479, 479, 625, 625, 626, 631, 631, 632, 632, 633, 634, 635, 636, 637, 638, 639, 639, 646, 646, 652, 653, 731, 731, 732, 733, 733, 734, 734, 735, 736, 737, 738, 739, 740, 741, 741, 742, 742, 743, 743, 744, 745, 758, 758, 807, 807 |
| `src/Net/SecondPassword.cpp` | 580 | 1787, 1790, 1790, 1791, 1791, 1791, 1791, 1792, 1792, 1792, 1792, 1793, 1793, 1794, 1794, 1816, 1816, 1818, 1819, 1820, 1821, 1828, 1828, 1837, 1837, 1838, 1838, 1843, 1850, 1852, 1853, 1855, 1885, 1895, 1903, 1903, 1904, 1904, 1906, 1906, 1921, 1921, 1922, 1922, 1922, 1922, 1954, 1958, 1960, 1965, 1966, 1968, 1969, 1970, 1971, 1972, 1973, 1979, 1979, 1990, 1991, 1992, 1993, 1994, 1995, 2037, 2084, 2158, 2158, 2158, 2158, 2159, 2159, 2159, 2159, 2246, 2246, 2257, 2267, 2298, 2313, 2314, 2315, 2316, 2317, 2317, 2318, 2319, 2327, 2331, 2336, 2341, 2346, 2351, 2356, 2360, 2361, 2362, 2363, 2364, 2365, 2366, 2367, 2368, 2369, 3085, 3096, 3097, 3098, 3099, 3100, 3128, 3130, 3131, 3131, 3134, 3135, 3136, 3138, 3139, 3140, 3149, 3153, 3155, 3156, 3156, 3156, 3157, 3157, 3157, 3158, 3159, 3159, 3159, 3160, 3161, 3163, 3164, 3169, 3170, 3170, 3171, 3172, 3172, 3173, 3174, 3175, 3175, 3177, 3177, 3178, 3178, 3179, 3179, 3180, 3181, 3182, 3186, 3187, 3188, 3190, 3193, 3194, 3194, 3195, 3198, 3205, 3208, 3208, 3209, 3209, 3210, 3211, 3212, 3213, 3213, 3214, 3215, 3216, 3216, 3224, 3225, 3226, 3248, 3253, 3258, 3258, 3259, 3259, 3263, 3277, 3320, 3323, 3323, 3325, 3336, 3337, 3337, 3340, 3349, 3351, 3357, 3358, 3369, 3372, 3377, 3382, 3383, 3397, 3399, 3400, 3405, 3406, 3409, 3411, 3413, 3414, 3418, 3418, 3424, 3436, 3443, 3448, 3449, 3450, 3455, 3464, 3474, 3484, 3486, 3493, 3501, 3502, 3505, 3510, 3511, 3514, 3523, 3524, 3525, 3542, 3547, 3548, 3551, 3552, 3558, 3564, 3565, 3568, 3570, 3573, 3573, 3573, 3573, 3581, 3588, 3589, 3592, 3594, 3597, 3597, 3597, 3626, 3626, 3634, 3635, 3636, 3637, 3637, 3641, 3645, 3651, 3670, 3675, 3689, 3707, 3714, 3736, 3760, 3766, 3780, 3780, 3793, 3800, 3800, 3808, 3815, 3816, 3817, 3817, 3818, 3818, 3819, 3820, 3821, 3821, 3822, 3826, 3833, 3833, 3833, 3836, 3839, 3842, 3843, 3844, 3845, 3846, 3848, 3851, 3852, 3853, 3854, 3855, 3860, 3862, 3864, 3865, 3866, 3869, 3869, 3873, 3875, 3876, 3877, 3878, 3879, 3884, 3884, 3886, 3886, 3886, 3891, 3892, 3916, 3916, 3947, 3949, 3952, 3958, 3960, 3968, 3987, 3987, 3988, 3988, 3989, 3989, 3993, 4007, 4007, 4011, 4011, 4011, 4011, 4012, 4012, 4013, 4013, 4018, 4018, 4019, 4019, 4020, 4020, 4021, 4021, 4022, 4022, 4035, 4035, 4036, 4036, 4039, 4071, 4071, 4075, 4094, 4094, 4095, 4095, 4096, 4096, 4111, 4116, 4116, 4116, 4116, 4119, 4119, 4119, 4119, 4126, 4126, 4134, 4134, 4135, 4135, 4143, 4143, 4151, 4151, 4151, 4151, 4153, 4153, 4162, 4162, 4166, 4167, 4167, 4168, 4168, 4169, 4169, 4187, 4187, 4188, 4197, 4197, 4203, 4203, 4203, 4203, 4213, 4213, 4219, 4219, 4220, 4220, 4221, 4221, 4225, 4225, 4226, 4226, 4229, 4229, 4239, 4239, 4242, 4242, 4247, 4247, 4251, 4251, 4260, 4260, 4261, 4261, 4262, 4262, 4263, 4263, 4270, 4270, 4270, 4270, 4271, 4271, 4271, 4271, 4271, 4271, 4273, 4273, 4273, 4273, 4273, 4273, 4280, 4280, 4281, 4281, 4288, 4288, 4289, 4289, 4297, 4297, 4298, 4298, 4299, 4299, 4301, 4301, 4303, 4303, 4306, 4306, 4306, 4306, 4316, 4316, 4317, 4317, 4344, 4344, 4346, 4346, 4349, 4354, 4354, 4389, 4389, 4389, 4389, 4396, 4396, 4397, 4397, 4401, 4401, 4402, 4402, 4413, 4413, 4414, 4414, 4415, 4415, 4423, 4423, 4443, 4443, 4444, 4444, 4450, 4450, 4451, 4451, 4461, 4461, 4462, 4462, 4466, 4466, 4466, 4466, 4476, 4476, 4501, 4501, 4505, 4505, 4505, 4505, 4506, 4506, 4522, 4522, 4527, 4527, 4528, 4528, 4530, 4530, 4533, 4533, 4538, 4538, 4538, 4548, 4548, 4550, 4550, 4552, 4552, 4584, 4592, 4598, 4601, 4601, 4613, 4614, 4617, 4618, 4623, 4624, 4627 |
| `src/Path/Path_Legacy.cpp` | 54 | 447, 448, 451, 454, 455, 468, 478, 478, 479, 479, 483, 485, 486, 487, 492, 492, 492, 492, 493, 493, 494, 494, 494, 494, 499, 500, 501, 502, 502, 503, 503, 521, 521, 524, 525, 526, 533, 534, 550, 550, 551, 551, 561, 561, 562, 562, 564, 565, 595, 595, 596, 596, 603, 603 |
| `src/Physics/Cloth_Simulation.cpp` | 5 | 1049, 1049, 1053, 1064, 1079 |
| `src/Platform/IInput.h` | 1 | 6 |
| `src/Platform/SDL3/SDLInput.cpp` | 3 | 74, 88, 93 |
| `src/Platform/Text.h` | 3 | 211, 214, 214 |
| `src/Platform/Win32/Win32Input.cpp` | 3 | 84, 88, 91 |
| `src/Render/BMD_Anim.cpp` | 11 | 97, 107, 115, 116, 120, 120, 132, 133, 136, 137, 138 |
| `src/Render/BMD_LegacyDraw.cpp` | 13 | 1899, 1900, 1901, 1902, 1903, 1904, 1905, 1906, 1907, 1908, 1910, 1910, 1916 |
| `src/Render/Backend/GL/BufferedDraw.cpp` | 3 | 102, 104, 105 |
| `src/Render/Effect_Combat.cpp` | 22 | 65, 75, 79, 80, 81, 83, 85, 86, 88, 90, 93, 93, 94, 95, 96, 98, 98, 98, 100, 100, 101, 108 |
| `src/Render/Effect_Create.cpp` | 531 | 162, 167, 168, 169, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 186, 188, 189, 192, 193, 195, 196, 197, 199, 200, 201, 202, 203, 218, 222, 231, 232, 234, 234, 239, 240, 241, 242, 243, 244, 245, 246, 251, 252, 254, 255, 255, 256, 256, 257, 257, 260, 261, 266, 267, 268, 269, 271, 280, 292, 292, 294, 297, 300, 301, 302, 305, 306, 306, 307, 310, 310, 314, 317, 319, 327, 328, 329, 330, 331, 336, 337, 341, 346, 358, 359, 360, 363, 366, 367, 372, 373, 374, 375, 383, 384, 388, 392, 396, 398, 398, 399, 399, 400, 400, 403, 408, 411, 412, 417, 418, 421, 422, 423, 424, 427, 427, 430, 430, 431, 431, 432, 433, 434, 435, 435, 436, 436, 441, 450, 450, 454, 456, 457, 458, 464, 464, 468, 468, 469, 469, 473, 474, 493, 494, 510, 512, 513, 514, 531, 532, 533, 534, 534, 538, 539, 544, 550, 555, 555, 556, 556, 557, 557, 568, 569, 570, 571, 576, 578, 580, 581, 585, 593, 594, 595, 604, 605, 605, 606, 606, 607, 608, 609, 617, 618, 619, 620, 621, 622, 634, 635, 635, 636, 636, 637, 643, 644, 647, 650, 651, 652, 666, 667, 669, 670, 671, 672, 676, 680, 683, 686, 687, 687, 690, 694, 696, 696, 697, 697, 698, 698, 699, 702, 707, 714, 715, 716, 717, 722, 723, 724, 725, 732, 735, 735, 736, 736, 737, 740, 742, 743, 744, 745, 750, 751, 752, 753, 759, 760, 760, 761, 762, 763, 769, 772, 772, 773, 773, 774, 775, 776, 777, 778, 779, 784, 788, 791, 792, 793, 797, 798, 799, 809, 812, 822, 823, 829, 839, 848, 849, 850, 851, 852, 858, 859, 860, 862, 862, 863, 866, 866, 871, 871, 872, 873, 874, 875, 876, 877, 880, 880, 881, 882, 886, 886, 887, 891, 894, 894, 896, 896, 898, 901, 902, 902, 903, 903, 904, 904, 905, 905, 906, 906, 907, 907, 908, 911, 911, 921, 921, 922, 922, 923, 923, 927, 932, 932, 936, 947, 950, 953, 954, 955, 957, 960, 965, 965, 966, 966, 967, 967, 968, 969, 970, 974, 977, 980, 990, 992, 992, 993, 993, 994, 994, 1009, 1016, 1016, 1017, 1017, 1018, 1018, 1019, 1020, 1021, 1036, 1038, 1038, 1041, 1042, 1045, 1048, 1050, 1051, 1054, 1069, 1080, 1082, 1083, 1083, 1084, 1084, 1085, 1089, 1091, 1092, 1096, 1097, 1101, 1102, 1103, 1105, 1106, 1131, 1137, 1137, 1140, 1140, 1141, 1142, 1145, 1146, 1147, 1148, 1150, 1151, 1152, 1162, 1163, 1165, 1167, 1168, 1169, 1173, 1174, 1175, 1179, 1180, 1181, 1183, 1184, 1187, 1188, 1189, 1191, 1192, 1193, 1194, 1195, 1196, 1199, 1200, 1201, 1203, 1204, 1205, 1211, 1215, 1227, 1229, 1229, 1230, 1231, 1232, 1234, 1235, 1236, 1237, 1238, 1238, 1240, 1241, 1253, 1255, 1255, 1256, 1257, 1258, 1259, 1265, 1266, 1266, 1267, 1267, 1272, 1272, 1273, 1273, 1274, 1343, 1343, 1344, 1344, 1345, 1345, 1346, 1346, 1347, 1347, 1348, 1348, 1355, 1356, 1356, 1357, 1357, 1358, 1358, 1359, 1363, 1375, 1388, 1388, 1389, 1389, 1390, 1391, 3704, 3705, 3706, 3713, 3714, 3715, 3717, 3721, 3722, 3723, 3728, 3730, 3731, 3732, 3735, 3736, 3737, 3743, 3747, 3749, 3969, 3973, 3973 |
| `src/Render/Entity_AnimationLegacy.cpp` | 16 | 52, 55, 57, 59, 60, 62, 73, 85, 88, 93, 93, 94, 94, 95, 95, 100 |
| `src/Render/Entity_DrawByType.cpp` | 220 | 175, 175, 237, 245, 251, 259, 267, 274, 276, 276, 277, 277, 278, 278, 283, 294, 294, 295, 295, 296, 296, 316, 316, 317, 319, 319, 320, 322, 322, 323, 325, 326, 326, 327, 334, 349, 349, 350, 350, 351, 351, 368, 368, 369, 369, 370, 370, 379, 379, 380, 380, 381, 381, 390, 390, 391, 391, 392, 394, 394, 395, 395, 396, 405, 405, 406, 406, 407, 407, 412, 413, 417, 418, 419, 467, 471, 472, 473, 476, 482, 483, 483, 485, 486, 486, 488, 489, 489, 491, 492, 492, 500, 500, 501, 501, 502, 502, 505, 506, 506, 516, 516, 517, 517, 518, 531, 531, 532, 532, 533, 538, 538, 539, 539, 540, 553, 553, 554, 554, 555, 555, 575, 575, 576, 576, 577, 577, 587, 587, 588, 588, 589, 591, 592, 592, 593, 593, 595, 595, 596, 596, 597, 597, 608, 608, 609, 609, 610, 610, 622, 622, 623, 623, 624, 629, 656, 661, 666, 666, 667, 667, 668, 668, 674, 674, 675, 675, 676, 676, 682, 682, 683, 683, 684, 684, 693, 693, 694, 694, 695, 695, 701, 701, 702, 702, 703, 703, 709, 709, 710, 710, 711, 711, 718, 718, 719, 719, 720, 720, 726, 726, 727, 727, 728, 728, 735, 736, 736, 737, 741, 747, 748, 748, 754, 755, 760, 760, 762, 762, 763 |
| `src/Render/Entity_Render.cpp` | 7 | 372, 381, 381, 382, 382, 382, 383 |
| `src/Render/Entity_UpdateRender.cpp` | 15 | 47, 56, 57, 58, 67, 70, 70, 70, 526, 528, 535, 570, 571, 608, 608 |
| `src/Render/EquipmentVisuals.cpp` | 27 | 22, 44, 46, 46, 76, 77, 78, 101, 107, 114, 114, 128, 128, 128, 136, 137, 137, 138, 172, 177, 177, 217, 217, 218, 218, 218, 221 |
| `src/Render/EquipmentVisuals.h` | 2 | 17, 19 |
| `src/Render/HUD_Pass1.cpp` | 2 | 248, 248 |
| `src/Render/HUD_Pass3.cpp` | 1 | 662 |
| `src/Render/HUD_Pass4.cpp` | 1 | 390 |
| `src/Render/JointTypes.h` | 1 | 30 |
| `src/Render/LooseWingVisuals.cpp` | 21 | 11, 11, 17, 17, 52, 52, 54, 56, 56, 77, 89, 89, 90, 91, 92, 97, 98, 108, 108, 115, 115 |
| `src/Render/LooseWingVisuals.h` | 2 | 11, 11 |
| `src/Render/MaterialVisuals.cpp` | 4 | 55, 55, 72, 72 |
| `src/Render/MoveEffect.cpp` | 769 | 421, 422, 431, 460, 462, 463, 469, 470, 470, 471, 472, 477, 478, 479, 492, 498, 499, 499, 499, 500, 500, 500, 501, 501, 501, 502, 502, 503, 503, 503, 503, 504, 506, 507, 510, 513, 519, 520, 521, 522, 538, 539, 540, 541, 547, 547, 548, 550, 551, 552, 572, 576, 577, 578, 579, 579, 580, 580, 590, 591, 604, 605, 609, 618, 618, 618, 624, 624, 628, 628, 645, 645, 650, 650, 650, 654, 654, 658, 658, 658, 664, 664, 669, 674, 683, 684, 685, 686, 702, 703, 704, 705, 748, 751, 751, 752, 752, 752, 753, 753, 754, 754, 755, 756, 760, 761, 771, 772, 772, 772, 774, 775, 781, 788, 789, 792, 795, 802, 804, 809, 830, 832, 840, 840, 848, 848, 849, 850, 851, 856, 856, 864, 864, 865, 865, 866, 866, 872, 872, 873, 873, 874, 874, 875, 881, 884, 887, 891, 892, 892, 893, 893, 894, 895, 895, 895, 902, 903, 903, 903, 905, 906, 906, 906, 916, 918, 919, 924, 926, 928, 929, 929, 931, 932, 943, 946, 946, 949, 950, 952, 953, 957, 961, 962, 964, 966, 967, 967, 968, 969, 969, 970, 972, 974, 979, 979, 982, 983, 986, 990, 993, 997, 1002, 1002, 1002, 1003, 1003, 1004, 1004, 1006, 1007, 1008, 1009, 1010, 1010, 1011, 1011, 1013, 1014, 1016, 1017, 1019, 1020, 1021, 1022, 1023, 1024, 1047, 1048, 1051, 1052, 1057, 1057, 1059, 1064, 1066, 1066, 1067, 1071, 1087, 1087, 1088, 1092, 1093, 1095, 1097, 1098, 1115, 1125, 1126, 1136, 1140, 1141, 1143, 1143, 1146, 1146, 1151, 1153, 1157, 1162, 1164, 1174, 1174, 1175, 1176, 1177, 1179, 1191, 1192, 1193, 1196, 1200, 1202, 1210, 1211, 1220, 1223, 1224, 1225, 1231, 1233, 1235, 1235, 1235, 1236, 1236, 1236, 1237, 1237, 1237, 1248, 1249, 1265, 1271, 1275, 1285, 1288, 1289, 1290, 1291, 1291, 1294, 1300, 1301, 1303, 1309, 1310, 1312, 1313, 1315, 1316, 1317, 1342, 1343, 1345, 1352, 1353, 1360, 1361, 1364, 1364, 1365, 1381, 1382, 1382, 1383, 1384, 1388, 1388, 1394, 1396, 1407, 1408, 1408, 1408, 1409, 1409, 1409, 1410, 1413, 1414, 1415, 1415, 1416, 1416, 1419, 1419, 1420, 1420, 1421, 1421, 1422, 1422, 1423, 1423, 1423, 1424, 1425, 1425, 1426, 1426, 1427, 1428, 1431, 1432, 1433, 1433, 1434, 1435, 1439, 1440, 1442, 1442, 1448, 1448, 1453, 1454, 1454, 1456, 1456, 1460, 1460, 1464, 1464, 1464, 1465, 1465, 1466, 1466, 1467, 1468, 1469, 1470, 1470, 1471, 1472, 1472, 1479, 1479, 1481, 1483, 1489, 1490, 1493, 1493, 1493, 1495, 1496, 1497, 1498, 1532, 1533, 1539, 1539, 1558, 1559, 1560, 1563, 1584, 1585, 1586, 1586, 1589, 1589, 1589, 1590, 1590, 1591, 1591, 1592, 1593, 1594, 1594, 1595, 1595, 1597, 1598, 1599, 1600, 1606, 1606, 1611, 1612, 1613, 1614, 1615, 1615, 1616, 1616, 1618, 1619, 1620, 1628, 1630, 1633, 1633, 1633, 1634, 1654, 1655, 1655, 1655, 1656, 1656, 1656, 1657, 1658, 1661, 1663, 1663, 1666, 1666, 1667, 1667, 1671, 1674, 1674, 1676, 1677, 1678, 1679, 1679, 1683, 1683, 1683, 1684, 1684, 1686, 1686, 1689, 1689, 1690, 1690, 1692, 1693, 1705, 1708, 1709, 1710, 1710, 1711, 1715, 1726, 1727, 1731, 1732, 1746, 1747, 1752, 1753, 1758, 1759, 1760, 1766, 1766, 1767, 1769, 1770, 1773, 1774, 1775, 1792, 1795, 1796, 1801, 1802, 1810, 1812, 1812, 1819, 1820, 1823, 1824, 1828, 1830, 1832, 1834, 1835, 1836, 1837, 1838, 1842, 1845, 1846, 1848, 1849, 1851, 1860, 1860, 1863, 1863, 1864, 1865, 1866, 1866, 1866, 1869, 1870, 1871, 1901, 1903, 1911, 1911, 1912, 1912, 1913, 1913, 1914, 1923, 1924, 1927, 1928, 1929, 1947, 1947, 1962, 1966, 1967, 1980, 1980, 1997, 1998, 1999, 2047, 2047, 2050, 2051, 2062, 2063, 2065, 2065, 2066, 2076, 2076, 2087, 2089, 2091, 2092, 2098, 2111, 2116, 2117, 2117, 2118, 2118, 2119, 2119, 2120, 2123, 2125, 2128, 2130, 2131, 2132, 2133, 2133, 2134, 2134, 2135, 2135, 2135, 2139, 2141, 2141, 2149, 2150, 2152, 2155, 2155, 2157, 2171, 2177, 2181, 2188, 2194, 2196, 2196, 2202, 2202, 2204, 2218, 2224, 2228, 2235, 2241, 2241, 2244, 2244, 2251, 2253, 2253, 2259, 2259, 2262, 2262, 2265, 2266, 2266, 2267, 2267, 2268, 2268, 2271, 2271, 2272, 2272, 2273, 2273, 2274, 2282, 2282, 2285, 2285, 2285, 2286, 2286, 2287, 2291, 2293, 2297, 2297, 2298, 2298, 2299, 2299, 2300, 2300, 2301, 2301, 2302, 2302, 2303, 2304, 2304, 2305, 2306, 2306, 2307, 2308, 2309, 2309, 2310, 2311, 2312, 2316, 2325, 2325, 2332, 2332, 2335, 2336, 2340, 2348, 2348, 2348, 2349, 2349, 2349, 2350, 2350, 2350, 2351, 2352, 2353, 2354, 2355, 2356, 2357, 2357, 2357, 2358, 2359, 2360, 2361, 2363, 2363, 2364, 2364, 2366, 2366, 2367, 2368, 2368, 2369, 2370, 2370, 2372, 2384, 2385, 2420, 2422, 2425, 2426, 2429, 2437, 2438, 2439, 2441, 2442, 2442, 2443, 2443, 2444, 2444, 2463, 2464, 2469, 2472 |
| `src/Render/RenderLinkObject.cpp` | 45 | 111, 111, 112, 112, 141, 141, 144, 144, 152, 152, 153, 153, 154, 154, 161, 303, 303, 309, 309, 310, 310, 311, 311, 569, 569, 570, 570, 571, 571, 572, 572, 573, 573, 585, 585, 586, 617, 620, 620, 683, 683, 699, 699, 721, 721 |
| `src/Render/Render_PlayerEquipment.cpp` | 25 | 143, 154, 156, 592, 630, 631, 632, 663, 664, 666, 668, 679, 688, 700, 703, 704, 705, 706, 711, 738, 739, 745, 749, 857, 858 |
| `src/Render/Render_WorldHelpers.cpp` | 42 | 864, 865, 866, 867, 868, 869, 870, 884, 885, 885, 886, 887, 888, 890, 895, 897, 898, 900, 903, 903, 903, 905, 905, 906, 910, 910, 911, 912, 913, 920, 921, 922, 923, 924, 925, 926, 1174, 1174, 1198, 1198, 1212, 1212 |
| `src/Render/SMD_Parser.cpp` | 37 | 139, 140, 142, 143, 144, 146, 148, 149, 150, 150, 151, 151, 156, 157, 157, 158, 158, 160, 160, 572, 572, 573, 573, 575, 578, 578, 579, 579, 583, 583, 584, 590, 840, 840, 848, 860, 870 |
| `src/Render/WorldEffects.cpp` | 16 | 53, 54, 54, 55, 56, 57, 58, 63, 79, 83, 84, 85, 86, 86, 90, 91 |
| `src/Scene/ObjectRegistry.h` | 6 | 23, 25, 27, 27, 28, 30 |
| `src/Scene/Scene_CharSelect.cpp` | 5 | 202, 202, 207, 207, 207 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 169 | 440, 440, 442, 442, 451, 452, 453, 454, 455, 456, 458, 459, 460, 461, 463, 464, 465, 466, 468, 469, 470, 471, 473, 474, 475, 476, 676, 690, 692, 693, 697, 698, 699, 700, 701, 702, 705, 706, 707, 708, 709, 710, 711, 712, 713, 714, 715, 716, 717, 717, 718, 718, 719, 719, 720, 720, 721, 721, 722, 722, 1559, 1559, 1602, 1605, 1607, 1608, 1618, 1632, 1658, 1662, 1672, 1685, 1700, 1700, 1701, 1701, 1702, 1702, 1704, 1708, 1718, 1718, 1722, 1722, 1730, 1730, 1734, 1734, 1753, 1753, 1756, 1763, 1763, 1768, 1769, 1773, 1804, 1805, 1807, 1807, 1810, 1810, 1814, 1821, 1821, 1823, 1823, 1825, 1836, 1836, 1861, 1861, 1862, 1862, 1863, 1863, 1900, 1900, 1907, 1908, 1910, 1915, 1916, 1918, 1923, 1925, 1926, 1934, 1937, 1938, 1941, 1949, 1953, 1954, 1957, 1971, 1972, 1973, 1974, 1975, 1982, 1984, 1987, 1993, 1994, 1997, 2003, 2004, 2006, 2012, 2013, 2015, 2021, 2022, 2024, 2027, 2028, 2029, 2030, 2031, 2032, 2033, 2034, 2035, 2036, 2037, 2038, 2327, 2330 |
| `src/Scene/Scene_EntityLifecycle.cpp` | 22 | 269, 272, 276, 277, 278, 279, 280, 285, 285, 285, 286, 287, 288, 298, 299, 300, 303, 304, 305, 306, 306, 306 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 12 | 49, 49, 50, 53, 63, 64, 65, 66, 67, 72, 74, 75 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 168 | 15, 35, 56, 58, 65, 68, 70, 79, 85, 87, 89, 90, 106, 108, 113, 114, 115, 116, 160, 176, 176, 181, 185, 195, 204, 204, 217, 222, 231, 247, 248, 251, 251, 251, 252, 252, 252, 253, 253, 253, 255, 256, 256, 257, 257, 258, 258, 259, 259, 262, 263, 264, 265, 266, 279, 282, 285, 286, 290, 294, 303, 307, 311, 312, 315, 322, 331, 333, 334, 340, 341, 343, 350, 351, 352, 352, 353, 358, 359, 363, 364, 367, 370, 371, 372, 379, 380, 385, 386, 387, 389, 391, 392, 393, 393, 396, 400, 401, 404, 410, 411, 412, 412, 414, 416, 417, 419, 421, 422, 423, 424, 431, 433, 436, 797, 811, 812, 813, 828, 833, 834, 835, 837, 839, 840, 841, 844, 846, 847, 849, 850, 851, 859, 860, 861, 865, 870, 875, 880, 884, 905, 907, 909, 915, 921, 921, 932, 932, 933, 933, 934, 934, 937, 938, 940, 942, 943, 946, 947, 947, 948, 949, 949, 992, 996, 999, 1001, 1014 |
| `src/Sound/Sound_DS3D.cpp` | 2 | 38, 44 |
| `src/Terrain/Terrain.cpp` | 10 | 631, 632, 632, 632, 632, 636, 637, 637, 639, 647 |
| `src/UI/Chat.cpp` | 1 | 581 |
| `src/UI/ChatTypes.h` | 1 | 23 |
| `src/UI/Chat_Bubbles.cpp` | 9 | 145, 145, 149, 150, 150, 185, 189, 221, 225 |
| `src/UI/GoldenArcher.cpp` | 2 | 1067, 1067 |
| `src/UI/PanelAnchor.h` | 1 | 19 |
| `src/functions.h` | 5 | 411, 411, 910, 914, 971 |
| `src/structs.h` | 3 | 290, 290, 294 |

### 2f items suelo (referencias tipadas)

Expresión (Python re):

```text
\b(?:ITEM_t|AsGroundItem)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Net/Recv/Recv_Viewport.cpp` | 3 | 804, 806, 806 |
| `src/structs.h` | 13 | 286, 292, 293, 294, 295, 295, 295, 296, 296, 296, 297, 297, 297 |

### 2g joints (referencias tipadas)

Expresión (Python re):

```text
\b(?:JOINT|AsJoint)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Entity/Entity_LegacyTeleport.cpp` | 4 | 38, 38, 39, 41 |
| `src/Net/SecondPassword.cpp` | 4 | 2940, 2940, 2941, 2943 |
| `src/Render/Effect_Tick.cpp` | 3 | 65, 65, 66 |
| `src/Render/JointTypes.h` | 31 | 20, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 71, 71, 71, 72, 72, 72, 73, 73, 73 |
| `src/Render/Joint_Create.cpp` | 360 | 91, 92, 93, 94, 95, 108, 109, 114, 115, 117, 118, 120, 121, 122, 133, 134, 135, 138, 139, 140, 141, 143, 150, 154, 157, 157, 158, 158, 159, 161, 164, 164, 165, 165, 166, 168, 171, 171, 172, 172, 173, 175, 176, 176, 177, 177, 182, 183, 184, 185, 186, 187, 191, 208, 209, 209, 210, 210, 232, 235, 241, 243, 253, 253, 254, 254, 261, 262, 263, 264, 265, 289, 290, 291, 309, 325, 326, 347, 349, 369, 383, 388, 393, 408, 423, 462, 476, 477, 477, 478, 478, 481, 483, 484, 484, 484, 485, 485, 485, 488, 502, 504, 505, 506, 510, 635, 647, 665, 666, 666, 667, 667, 678, 708, 722, 723, 724, 724, 725, 726, 727, 730, 730, 732, 733, 734, 737, 738, 741, 741, 742, 742, 745, 746, 747, 747, 748, 872, 878, 879, 879, 886, 891, 895, 901, 931, 932, 938, 977, 978, 979, 980, 981, 981, 982, 982, 986, 989, 993, 994, 1001, 1019, 1019, 1020, 1020, 1021, 1039, 1043, 1044, 1046, 1047, 1047, 1048, 1048, 1065, 1079, 1080, 1080, 1081, 1082, 1083, 1084, 1087, 1087, 1089, 1095, 1101, 1109, 1115, 1118, 1118, 1119, 1119, 1122, 1123, 1124, 1124, 1125, 1192, 1206, 1207, 1208, 1208, 1209, 1210, 1211, 1214, 1214, 1216, 1217, 1218, 1221, 1222, 1225, 1225, 1226, 1226, 1229, 1230, 1231, 1231, 1232, 1235, 1235, 1236, 1236, 1237, 1237, 1243, 1253, 1263, 1285, 1304, 1314, 1318, 1319, 1343, 1344, 1344, 1345, 1348, 1349, 1349, 1350, 1352, 1381, 1446, 1451, 1506, 1512, 1512, 1515, 1525, 1536, 1561, 1562, 1592, 1593, 1594, 1594, 1595, 1596, 1597, 1598, 1598, 1614, 1615, 1615, 1619, 1619, 1620, 1620, 1626, 1640, 1643, 1646, 1647, 1655, 1658, 1660, 1661, 1665, 1668, 1671, 1674, 1674, 1675, 1689, 1690, 1701, 1709, 1733, 1733, 1734, 1734, 1759, 1759, 1760, 1760, 1763, 1783, 1785, 1786, 1787, 1811, 1821, 1822, 1825, 1827, 1828, 1829, 1832, 1853, 1870, 1871, 1877, 1902, 1903, 1903, 1904, 1904, 1907, 1909, 1910, 1910, 1911, 1911, 1927, 1941, 1942, 1942, 1943, 1943, 1946, 1948, 1949, 1949, 1949, 1950, 1950, 1950, 1953, 1958, 1961, 1964, 1965, 2003, 2008, 2009, 2020, 2024, 2025, 2026, 2039, 2062, 2064, 2162, 2319, 2427, 2584 |
| `src/Render/MoveJoint.cpp` | 531 | 54, 57, 66, 70, 71, 72, 94, 95, 96, 101, 107, 111, 129, 149, 164, 168, 171, 174, 177, 178, 181, 182, 185, 188, 188, 189, 189, 190, 190, 194, 196, 197, 199, 207, 213, 220, 238, 248, 296, 296, 297, 298, 299, 299, 303, 304, 305, 310, 310, 311, 311, 314, 329, 331, 333, 336, 337, 341, 342, 345, 346, 349, 360, 370, 371, 376, 376, 377, 377, 381, 392, 394, 395, 396, 396, 397, 397, 404, 405, 406, 409, 413, 416, 417, 418, 421, 422, 423, 423, 424, 424, 425, 425, 425, 428, 429, 430, 431, 435, 438, 439, 441, 441, 442, 450, 451, 455, 460, 463, 464, 465, 467, 468, 469, 469, 470, 470, 471, 471, 472, 478, 479, 480, 482, 483, 491, 492, 494, 494, 495, 496, 496, 498, 499, 500, 502, 505, 512, 516, 517, 519, 522, 523, 526, 533, 535, 536, 550, 556, 558, 568, 572, 573, 574, 576, 591, 597, 599, 604, 606, 619, 621, 632, 633, 643, 645, 646, 648, 655, 657, 668, 669, 704, 728, 730, 731, 732, 733, 737, 738, 739, 741, 742, 743, 744, 747, 763, 767, 775, 776, 778, 780, 784, 792, 793, 794, 800, 801, 802, 803, 810, 811, 815, 816, 816, 818, 818, 819, 820, 820, 824, 825, 826, 827, 829, 829, 831, 831, 832, 835, 836, 837, 837, 838, 839, 840, 843, 844, 845, 846, 848, 849, 850, 852, 853, 854, 855, 858, 879, 881, 882, 883, 892, 893, 893, 894, 894, 895, 898, 899, 899, 900, 900, 902, 902, 903, 903, 904, 904, 911, 912, 915, 924, 925, 926, 927, 928, 929, 936, 938, 939, 940, 941, 942, 945, 946, 947, 949, 953, 954, 955, 961, 962, 964, 965, 966, 969, 969, 974, 975, 977, 978, 983, 985, 985, 987, 987, 991, 998, 999, 1002, 1003, 1005, 1010, 1018, 1019, 1021, 1022, 1023, 1024, 1024, 1025, 1025, 1025, 1026, 1028, 1029, 1030, 1035, 1036, 1037, 1041, 1044, 1045, 1046, 1048, 1052, 1056, 1057, 1058, 1060, 1061, 1062, 1064, 1065, 1066, 1072, 1075, 1075, 1082, 1083, 1084, 1085, 1089, 1089, 1091, 1092, 1093, 1094, 1096, 1097, 1098, 1099, 1103, 1112, 1116, 1117, 1118, 1119, 1121, 1124, 1124, 1125, 1125, 1125, 1126, 1127, 1127, 1127, 1129, 1129, 1130, 1130, 1131, 1131, 1136, 1139, 1139, 1148, 1160, 1162, 1165, 1166, 1178, 1178, 1180, 1180, 1180, 1185, 1187, 1187, 1188, 1188, 1189, 1189, 1191, 1193, 1195, 1196, 1196, 1198, 1199, 1200, 1201, 1203, 1203, 1204, 1204, 1205, 1205, 1208, 1210, 1211, 1212, 1213, 1236, 1239, 1248, 1251, 1275, 1276, 1277, 1278, 1281, 1282, 1283, 1286, 1287, 1288, 1289, 1290, 1295, 1297, 1298, 1300, 1301, 1302, 1306, 1307, 1307, 1309, 1309, 1310, 1311, 1312, 1317, 1318, 1320, 1321, 1322, 1322, 1326, 1328, 1329, 1330, 1330, 1331, 1341, 1342, 1363, 1363, 1364, 1364, 1365, 1366, 1367, 1367, 1368, 1368, 1382, 1384, 1385, 1385, 1386, 1386, 1387, 1388, 1393, 1394, 1396, 1397, 1397, 1398, 1398, 1433, 1464, 1480, 1480, 1481, 1481, 1482, 1482, 1484, 1509, 1514, 1515, 1516, 1517, 1522, 1529, 1552, 1553, 1562, 1562, 1563, 1563, 1564, 1565, 1565, 1566, 1566, 1567, 1567, 1591, 1592, 1592, 1593, 1593, 1594, 1594, 1604, 1605, 1607, 1613, 1614, 1616, 1617, 1627, 1628, 1634, 1635 |
| `src/Render/Render_WorldHelpers.cpp` | 4 | 342, 342, 343, 346 |
| `src/Render/SMD_Parser.cpp` | 34 | 666, 667, 668, 669, 670, 678, 692, 696, 696, 697, 697, 698, 698, 699, 702, 702, 704, 704, 705, 705, 706, 709, 709, 711, 711, 712, 712, 713, 715, 715, 716, 716, 717, 717 |

### 2h partículas (pool y macros)

Expresión (Python re):

```text
\b(?:DAT_07abf5f0|P_(?:BASE|ACTIVE|TYPE|SUB|SCALE|POSX|POSY|POSZ|DIRX|ANGLE|LR|LG|LB|LIFE|ENT|ROT|FRAME|GRAV|VELX|VELY|VELZ|EX0|EX1|EX2|TGX|TGY|TGZ))\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Game/RenderFxSnapshots.cpp` | 1 | 35 |
| `src/Game/RenderInterpolation.cpp` | 1 | 57 |
| `src/Render/ItemDrop_Render2.cpp` | 1 | 46 |
| `src/Render/Particle_Move.cpp` | 668 | 36, 36, 37, 37, 38, 38, 39, 39, 40, 40, 41, 41, 42, 42, 43, 43, 44, 44, 45, 45, 46, 46, 47, 47, 48, 48, 49, 49, 50, 50, 51, 51, 52, 52, 53, 53, 54, 54, 55, 55, 56, 56, 57, 57, 58, 58, 59, 59, 60, 60, 61, 61, 62, 62, 113, 116, 117, 119, 123, 124, 125, 129, 141, 141, 142, 142, 142, 142, 143, 143, 144, 145, 146, 155, 156, 157, 158, 159, 160, 160, 161, 161, 162, 162, 163, 164, 165, 166, 166, 167, 167, 167, 172, 173, 175, 176, 177, 184, 190, 191, 192, 196, 197, 198, 199, 201, 202, 207, 208, 209, 210, 211, 217, 218, 219, 220, 221, 222, 224, 226, 227, 228, 230, 231, 232, 239, 240, 241, 246, 247, 248, 250, 251, 252, 253, 258, 260, 260, 262, 262, 264, 264, 271, 275, 280, 285, 287, 287, 288, 288, 289, 289, 291, 291, 292, 293, 294, 295, 296, 297, 302, 307, 312, 313, 315, 315, 317, 317, 319, 319, 320, 321, 322, 324, 324, 325, 325, 326, 327, 328, 329, 333, 335, 335, 337, 337, 338, 338, 339, 345, 346, 347, 351, 353, 357, 358, 359, 360, 361, 361, 362, 362, 363, 363, 370, 371, 371, 372, 372, 373, 373, 374, 375, 376, 380, 384, 388, 390, 391, 392, 393, 394, 398, 401, 402, 404, 405, 406, 408, 409, 409, 412, 415, 415, 415, 419, 419, 419, 426, 426, 427, 427, 428, 428, 429, 430, 431, 432, 433, 434, 435, 436, 437, 439, 445, 445, 446, 446, 447, 447, 448, 449, 450, 454, 458, 462, 464, 465, 466, 467, 468, 472, 477, 478, 479, 481, 481, 482, 487, 487, 492, 493, 493, 495, 496, 497, 502, 503, 504, 505, 506, 507, 508, 509, 521, 523, 524, 525, 529, 530, 531, 532, 533, 534, 535, 536, 540, 541, 542, 543, 544, 545, 546, 547, 548, 552, 554, 555, 556, 557, 558, 559, 560, 564, 565, 566, 569, 570, 571, 572, 573, 577, 578, 579, 580, 581, 584, 584, 585, 585, 586, 591, 592, 593, 593, 597, 598, 599, 601, 606, 607, 608, 609, 610, 611, 612, 614, 616, 617, 618, 622, 623, 624, 625, 626, 627, 628, 629, 630, 631, 632, 634, 635, 637, 638, 639, 640, 641, 642, 647, 648, 649, 650, 651, 652, 653, 654, 658, 659, 661, 662, 663, 667, 668, 668, 669, 669, 670, 670, 671, 672, 673, 674, 675, 688, 689, 690, 691, 692, 699, 699, 707, 707, 715, 717, 718, 719, 722, 723, 724, 727, 727, 728, 728, 728, 728, 732, 735, 736, 737, 738, 740, 741, 741, 741, 743, 743, 744, 745, 745, 746, 747, 748, 748, 749, 753, 759, 767, 768, 769, 770, 771, 771, 772, 773, 773, 774, 775, 776, 777, 778, 780, 780, 781, 781, 782, 782, 789, 790, 791, 792, 793, 794, 795, 801, 801, 802, 803, 804, 813, 819, 820, 821, 822, 825, 826, 827, 828, 829, 830, 831, 832, 833, 834, 835, 836, 837, 838, 858, 863, 872, 873, 874, 875, 879, 881, 882, 882, 883, 884, 885, 891, 896, 897, 898, 900, 902, 904, 906, 907, 908, 909, 911, 912, 913, 914, 917, 918, 919, 920, 922, 923, 924, 925, 933, 933, 934, 935, 935, 937, 939, 953, 954, 960, 960, 962, 963, 964, 965, 966, 967, 968, 969, 970, 977, 978, 979, 982, 983, 986, 986, 989, 992, 993, 996, 997, 997, 998, 998, 999, 999, 1002, 1003, 1004, 1008, 1011, 1011, 1012, 1013, 1014, 1017, 1017, 1018, 1018, 1023, 1024, 1025, 1026, 1027, 1028, 1029, 1030, 1031, 1032, 1033, 1038, 1038, 1039, 1041, 1042, 1043, 1051, 1054, 1055, 1055, 1056, 1056, 1057, 1057, 1057, 1058, 1059, 1060, 1061, 1062, 1067, 1068, 1069, 1076, 1078, 1079, 1080, 1087, 1088, 1089, 1090, 1091, 1092, 1093, 1093, 1094, 1095, 1096, 1101, 1101, 1102, 1103, 1105, 1105, 1111, 1111, 1123, 1124, 1125, 1126, 1127, 1128, 1129, 1130, 1131, 1132, 1133, 1134, 1135, 1136, 1137, 1138, 1139, 1140, 1141, 1142, 1143, 1144, 1145, 1146, 1147, 1148, 1149 |
| `src/Render/Particle_Spawn.cpp` | 1 | 312 |
| `src/Scene/Scene_OpenWorld.cpp` | 2 | 33, 33 |
| `src/globals.cpp` | 1 | 720 |
| `src/globals.h` | 1 | 612 |

### 2i modelos (referencias tipadas y base)

Expresión (Python re):

```text
\b(?:BMD|AsModel|DAT_05828d58)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat_AttackEffect.cpp` | 2 | 80, 101 |
| `src/Combat/Combat_AttackStage.cpp` | 4 | 23, 332, 363, 363 |
| `src/Combat/Combat_LegacyAnimation.cpp` | 1 | 87 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 4 | 74, 75, 249, 258 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 1 | 423 |
| `src/Entity/CharacterCloth.cpp` | 2 | 32, 116 |
| `src/Entity/CharacterEffects.cpp` | 2 | 31, 115 |
| `src/Entity/Entity_ActionLegacy.cpp` | 2 | 61, 63 |
| `src/Entity/Entity_BoneEffects.cpp` | 2 | 52, 74 |
| `src/Entity/Entity_Spawn.cpp` | 1 | 200 |
| `src/Entity/ObjectPose.cpp` | 3 | 71, 350, 371 |
| `src/Game/Game_SceneUpdate.cpp` | 1 | 67 |
| `src/Game/RenderDerivedState.cpp` | 2 | 57, 104 |
| `src/Game/RenderFxSnapshots.cpp` | 2 | 57, 88 |
| `src/Game/SimulationPreparation.cpp` | 4 | 124, 130, 163, 199 |
| `src/Input/Input_Timing.cpp` | 3 | 61, 61, 62 |
| `src/Item/ContentCatalog.cpp` | 4 | 224, 225, 569, 586 |
| `src/Item/NPC_ModelLoad.cpp` | 3 | 18, 26, 156 |
| `src/Model/ModelTypes.h` | 32 | 15, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 82, 82, 82, 83, 83, 83, 84, 84, 84 |
| `src/Model/Model_Items.cpp` | 2 | 27, 321 |
| `src/Model/Model_Players.cpp` | 5 | 349, 350, 351, 352, 353 |
| `src/Monster/Monster.cpp` | 2 | 638, 653 |
| `src/Monster/Monster_Data.cpp` | 39 | 189, 285, 294, 301, 313, 317, 320, 329, 338, 352, 359, 372, 381, 391, 392, 399, 407, 429, 438, 453, 462, 471, 480, 489, 496, 505, 506, 515, 558, 567, 570, 584, 595, 603, 610, 617, 623, 630, 637 |
| `src/Net/Crypto.cpp` | 2 | 1005, 1083 |
| `src/Net/Recv/Recv_Combat.cpp` | 2 | 75, 84 |
| `src/Net/SecondPassword.cpp` | 19 | 1924, 1927, 2161, 2170, 3032, 3085, 3096, 3097, 3098, 3099, 3100, 3350, 3351, 3377, 3959, 3959, 3961, 3961, 4007 |
| `src/Physics/Cloth_Simulation.cpp` | 1 | 62 |
| `src/Platform/ClientEntry.cpp` | 2 | 675, 677 |
| `src/Render/Ambient_Particles.cpp` | 1 | 263 |
| `src/Render/BMD_Anim.cpp` | 6 | 45, 46, 47, 82, 83, 98 |
| `src/Render/BMD_DrawMesh.cpp` | 3 | 152, 251, 408 |
| `src/Render/BMD_LegacyDraw.cpp` | 2 | 1336, 1885 |
| `src/Render/BMD_LegacyLoading.cpp` | 11 | 84, 105, 138, 341, 348, 350, 352, 360, 400, 543, 561 |
| `src/Render/BMD_Load.cpp` | 14 | 615, 618, 637, 641, 645, 648, 655, 659, 662, 668, 672, 673, 677, 682 |
| `src/Render/Effect_Combat.cpp` | 4 | 68, 76, 171, 212 |
| `src/Render/Entity_AnimationLegacy.cpp` | 1 | 55 |
| `src/Render/Entity_DrawAt.cpp` | 1 | 115 |
| `src/Render/Entity_DrawByType.cpp` | 1 | 237 |
| `src/Render/Entity_DrawSetup.cpp` | 3 | 150, 215, 219 |
| `src/Render/Entity_Render.cpp` | 1 | 413 |
| `src/Render/Entity_Render_3D.cpp` | 1 | 155 |
| `src/Render/Entity_UpdateRender.cpp` | 4 | 85, 446, 458, 733 |
| `src/Render/EquipmentVisuals.cpp` | 11 | 120, 122, 122, 122, 126, 168, 186, 186, 190, 192, 198 |
| `src/Render/LooseWingVisuals.cpp` | 10 | 50, 66, 79, 79, 93, 96, 96, 117, 120, 120 |
| `src/Render/MaterialVisuals.cpp` | 5 | 82, 83, 83, 84, 85 |
| `src/Render/MoveEffect.cpp` | 3 | 679, 682, 2424 |
| `src/Render/MoveJoint.cpp` | 1 | 153 |
| `src/Render/Particle_Move.cpp` | 1 | 861 |
| `src/Render/RenderLinkObject.cpp` | 17 | 109, 111, 112, 113, 114, 127, 143, 152, 153, 154, 309, 310, 311, 318, 319, 320, 321 |
| `src/Render/Render_LegacyBillboards.cpp` | 1 | 57 |
| `src/Render/Render_LegacyLinker.cpp` | 2 | 90, 363 |
| `src/Render/Render_PlayerEquipment.cpp` | 4 | 188, 220, 677, 712 |
| `src/Render/Render_WorldHelpers.cpp` | 13 | 79, 122, 123, 144, 215, 814, 861, 893, 938, 1013, 1170, 1181, 1210 |
| `src/Render/SMD_Parser.cpp` | 23 | 179, 180, 181, 185, 201, 207, 216, 220, 221, 222, 223, 224, 225, 226, 227, 237, 768, 812, 813, 814, 815, 816, 817 |
| `src/Render/Terrain_Render.cpp` | 1 | 63 |
| `src/Render/Weather.cpp` | 1 | 508 |
| `src/Render/WorldEffects.cpp` | 1 | 59 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 5 | 1723, 1726, 1735, 1738, 1843 |
| `src/Scene/Scene_ObjectInteraction.cpp` | 7 | 54, 58, 63, 64, 65, 66, 67 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 7 | 86, 87, 216, 289, 293, 798, 1012 |
| `src/Scene/Scene_Objects.cpp` | 9 | 61, 76, 431, 434, 435, 436, 437, 438, 439 |
| `src/Scene/Scene_Resources.cpp` | 4 | 27, 150, 151, 207 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 15 | 233, 241, 283, 295, 558, 641, 642, 643, 687, 689, 690, 691, 692, 693, 694 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 1 | 320 |
| `src/globals.cpp` | 1 | 656 |
| `src/globals.h` | 1 | 1148 |

### 2j texturas (vistas y base)

Expresión (Python re):

```text
\b(?:TexSlot|BITMAP_t|g_BitmapsRaw|Bitmaps)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Core/Runtime_Medium.cpp` | 2 | 279, 350 |
| `src/Render/BMD_LegacyLoading.cpp` | 3 | 480, 519, 519 |
| `src/Render/BMD_Load.cpp` | 3 | 265, 266, 278 |
| `src/Render/Entity_DrawByType.cpp` | 1 | 138 |
| `src/Render/HUD_Pass3.cpp` | 3 | 710, 711, 711 |
| `src/Render/HUD_Pass4.cpp` | 6 | 733, 734, 735, 753, 754, 754 |
| `src/Render/Particle_Move.cpp` | 1 | 64 |
| `src/Render/Texture/Texture.cpp` | 7 | 100, 114, 168, 252, 293, 523, 523 |
| `src/Scene/Scene_Intro.cpp` | 1 | 149 |
| `src/Scene/Scene_Resources.cpp` | 1 | 36 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 1 | 376 |
| `src/Terrain/Terrain_LegacyRender.cpp` | 1 | 116 |
| `src/Terrain/Terrain_RenderFace.cpp` | 5 | 51, 95, 95, 195, 195 |
| `src/UI/UI_GuildMark.cpp` | 8 | 14, 14, 81, 82, 84, 140, 141, 141 |
| `src/UI/UI_TextBitmap.cpp` | 7 | 211, 212, 212, 212, 212, 264, 265 |
| `src/globals.cpp` | 1 | 1151 |
| `src/globals.h` | 8 | 1129, 1133, 1134, 1135, 1136, 1137, 1138, 1139 |
| `src/structs.h` | 5 | 369, 488, 489, 489, 489 |

### 3a entradas/salidas binarias

Expresión (Python re):

```text
\b(?:crt_)?(?:fread|fwrite)\s*\(
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Core/Legacy_Runtime.cpp` | 2 | 68, 69 |
| `src/Item/Dialog_Data.cpp` | 1 | 30 |
| `src/Item/Gate_Data.cpp` | 2 | 54, 79 |
| `src/Item/Item_Data.cpp` | 4 | 82, 93, 116, 118 |
| `src/Item/Quest_Data.cpp` | 1 | 39 |
| `src/Item/Skill_Data.cpp` | 4 | 59, 69, 93, 95 |
| `src/Local/Text_Data.cpp` | 1 | 60 |
| `src/Net/Crypto.cpp` | 5 | 1516, 1531, 1535, 1539, 1543 |
| `src/Platform/ClientEntry.cpp` | 1 | 329 |
| `src/Platform/Posix/PosixIdentity.h` | 1 | 13 |
| `src/Platform/SDL3/SDLAudio.cpp` | 1 | 65 |
| `src/Platform/SDL3/SDLFont.cpp` | 1 | 307 |
| `src/Render/BMD_FilterData.cpp` | 8 | 42, 43, 66, 67, 148, 149, 170, 171 |
| `src/Render/BMD_LegacyLoading.cpp` | 2 | 309, 310 |
| `src/Render/BMD_Load.cpp` | 24 | 88, 607, 608, 609, 610, 611, 619, 620, 621, 623, 624, 625, 626, 627, 632, 637, 649, 650, 652, 663, 665, 666, 672, 673 |
| `src/Render/Texture/ImageCodec.cpp` | 2 | 36, 77 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 1 | 842 |
| `src/Sound/SoundManager.cpp` | 1 | 34 |
| `src/Terrain/Terrain.cpp` | 1 | 534 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 3 | 67, 122, 176 |
| `src/UI/RenderItemInfo.cpp` | 1 | 110 |
| `src/functions.h` | 2 | 725, 829 |

### 3b aserciones de red

Expresión (Python re):

```text
\bstatic_assert\s*\(
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Net/Protocol/ConnectServerProtocol.h` | 11 | 21, 22, 29, 30, 40, 41, 42, 53, 54, 61, 62 |
| `src/Net/Protocol/GameServerProtocol.h` | 206 | 29, 30, 31, 32, 33, 44, 45, 46, 58, 59, 60, 61, 62, 70, 79, 80, 88, 89, 90, 100, 101, 111, 112, 126, 127, 128, 129, 130, 131, 132, 140, 141, 149, 150, 158, 159, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 205, 206, 207, 208, 209, 210, 211, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 264, 265, 266, 267, 268, 269, 270, 271, 272, 273, 274, 275, 276, 277, 296, 297, 298, 299, 300, 301, 302, 303, 304, 305, 306, 307, 333, 334, 335, 336, 337, 338, 339, 340, 341, 342, 343, 344, 345, 346, 347, 348, 349, 374, 375, 376, 377, 378, 379, 380, 381, 382, 383, 384, 385, 386, 387, 396, 397, 405, 406, 407, 408, 417, 418, 426, 427, 428, 429, 438, 439, 448, 449, 450, 451, 452, 462, 463, 464, 478, 479, 480, 481, 482, 483, 484, 485, 494, 495, 503, 504, 505, 515, 516, 517, 518, 530, 531, 532, 533, 542, 543, 544, 554, 555, 556, 557, 565, 566, 588, 589, 590, 644, 645, 646, 647, 648, 649, 650, 651, 652, 653, 654, 655, 669, 670, 671, 672, 673, 682, 710, 721, 742, 758, 767, 768 |
| `src/Net/Protocol/ProtocolBase.h` | 6 | 16, 17, 35, 45, 54, 64 |

### 3c aserciones de disco existentes

Expresión (Python re):

```text
\bstatic_assert\s*\([^;]*(?:ITEM_ATTRIBUTE|GATE_ATTRIBUTE|sizeof\(bool\))
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/structs.h` | 11 | 327, 328, 329, 330, 331, 332, 333, 334, 350, 351, 352 |

### 4a alias de cuatro bytes en Owner/Target/BoneMatrices

Expresión (Python re):

```text
\*\s*\(\s*(?:int|unsigned int|uint|DWORD|undefined4|float)\s*\*\s*\)\s*&\s*As(?:Object|Character|Joint)\([^\n;]*?\)->(?:Object\.)?(?:Owner|Target|BoneMatrices)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat_AttackEffect.cpp` | 20 | 699, 730, 742, 1002, 1478, 1501, 1808, 1829, 1842, 1860, 1882, 1934, 1971, 1996, 2021, 2042, 2061, 2078, 2123, 2145 |
| `src/Combat/Combat_AttackStage.cpp` | 6 | 71, 84, 136, 345, 350, 357 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 3 | 72, 249, 259 |
| `src/Net/SecondPassword.cpp` | 43 | 3382, 3383, 3839, 3848, 3862, 3864, 3869, 3873, 3949, 3952, 4143, 4153, 4162, 4197, 4213, 4229, 4242, 4251, 4280, 4281, 4288, 4289, 4297, 4298, 4301, 4303, 4316, 4317, 4344, 4346, 4354, 4396, 4397, 4401, 4402, 4423, 4476, 4506, 4522, 4527, 4528, 4533, 4538 |
| `src/Render/Effect_Create.cpp` | 12 | 179, 255, 256, 257, 822, 823, 905, 906, 907, 908, 1211, 3732 |
| `src/Render/Joint_Create.cpp` | 13 | 143, 261, 732, 1089, 1216, 1343, 1344, 1345, 1348, 1349, 1350, 1658, 1811 |
| `src/Render/MoveEffect.cpp` | 36 | 470, 507, 520, 522, 579, 618, 674, 788, 809, 832, 840, 881, 943, 986, 1220, 1233, 1285, 1382, 1396, 1598, 1611, 1671, 1684, 1692, 1693, 1705, 1758, 1792, 1819, 1830, 1901, 2062, 2087, 2291, 2384, 2385 |
| `src/Render/MoveJoint.cpp` | 26 | 111, 129, 149, 213, 519, 533, 572, 604, 643, 767, 778, 780, 810, 818, 824, 949, 991, 1021, 1048, 1072, 1075, 1082, 1117, 1203, 1295, 1484 |
| `src/Render/RenderLinkObject.cpp` | 2 | 141, 303 |

### 4b casts uintptr_t junto a float

Expresión (Python re):

```text
\(float\s*\*?\)[^;\n]*\buintptr_t\b|\buintptr_t\b[^;\n]*\(float\s*\*?\)
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat.cpp` | 4 | 2167, 2342, 2353, 2365 |
| `src/Combat/Combat_AttackEffect.cpp` | 2 | 78, 89 |
| `src/Combat/Skills_WizardElf.cpp` | 1 | 75 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 2 | 86, 104 |
| `src/Net/Recv/Recv_Character.cpp` | 1 | 865 |
| `src/Net/Recv/Recv_Combat.cpp` | 1 | 86 |
| `src/Net/Recv/Recv_Quest.cpp` | 3 | 107, 136, 139 |
| `src/Net/Recv/Recv_Viewport.cpp` | 1 | 559 |
| `src/Net/SecondPassword.cpp` | 61 | 2981, 2984, 2994, 2996, 3003, 3006, 3435, 3436, 3442, 3443, 3463, 3464, 3465, 3466, 3473, 3474, 3483, 3484, 3485, 3486, 3492, 3493, 3497, 3498, 3504, 3505, 3513, 3514, 3528, 3529, 3541, 3542, 3617, 3618, 3646, 3647, 3655, 3656, 3660, 3661, 3665, 3666, 3668, 3669, 3677, 3678, 3680, 3681, 3684, 3685, 3715, 3716, 3718, 3719, 3732, 3733, 3737, 3738, 3758, 3759, 4103 |
| `src/Render/Effect_Create.cpp` | 17 | 218, 219, 222, 227, 384, 406, 441, 627, 699, 822, 823, 852, 907, 908, 1211, 1363, 3732 |
| `src/Render/Entity_AnimationLegacy.cpp` | 1 | 230 |
| `src/Render/Entity_UpdateRender.cpp` | 1 | 67 |
| `src/Render/Render_PlayerEquipment.cpp` | 2 | 203, 206 |
| `src/Render/SMD_Parser.cpp` | 3 | 601, 606, 650 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 2 | 1566, 1575 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 2 | 230, 319 |
| `src/Terrain/Terrain.cpp` | 1 | 613 |
| `src/UI/RenderItemInfo.cpp` | 1 | 2085 |

### 4c declaraciones escalares estrechas en globals.h (candidatos)

Expresión (Python re):

```text
\bextern\s+(?:"C"\s+)?(?:DWORD|int|uint|unsigned(?:\s+int)?|long)\s+\w+\s*(?=;|=)
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/globals.h` | 580 | 22, 23, 24, 25, 26, 27, 36, 38, 40, 42, 44, 46, 48, 50, 52, 55, 57, 59, 60, 62, 64, 66, 68, 70, 72, 74, 76, 78, 80, 82, 83, 84, 87, 88, 90, 97, 98, 99, 102, 104, 106, 107, 109, 111, 113, 116, 118, 120, 122, 124, 126, 128, 130, 132, 134, 136, 138, 140, 142, 144, 145, 150, 152, 154, 156, 157, 160, 162, 165, 167, 169, 171, 173, 174, 176, 178, 180, 182, 184, 186, 188, 190, 191, 199, 200, 208, 210, 211, 213, 215, 216, 217, 218, 219, 220, 221, 224, 225, 226, 227, 228, 230, 231, 232, 233, 234, 239, 241, 244, 246, 247, 248, 249, 262, 263, 264, 269, 270, 271, 272, 273, 274, 278, 281, 281, 282, 287, 288, 289, 290, 292, 297, 301, 302, 303, 306, 320, 322, 323, 338, 339, 340, 341, 342, 343, 344, 345, 346, 348, 395, 396, 397, 398, 399, 400, 401, 418, 419, 420, 421, 422, 433, 434, 436, 437, 438, 439, 440, 441, 442, 450, 451, 452, 454, 455, 457, 458, 459, 461, 462, 463, 464, 465, 475, 478, 479, 480, 481, 486, 487, 488, 489, 490, 491, 493, 494, 497, 499, 500, 501, 511, 513, 516, 517, 522, 529, 529, 531, 535, 536, 537, 538, 540, 541, 605, 606, 608, 609, 611, 617, 618, 619, 639, 640, 641, 642, 656, 657, 662, 664, 668, 669, 673, 674, 675, 676, 677, 678, 679, 680, 685, 686, 687, 688, 689, 690, 691, 692, 694, 695, 696, 697, 698, 699, 700, 701, 702, 703, 706, 707, 726, 738, 740, 742, 747, 748, 749, 750, 751, 759, 760, 761, 762, 763, 771, 772, 773, 774, 775, 776, 777, 788, 789, 792, 793, 794, 795, 796, 797, 798, 799, 801, 802, 803, 805, 806, 816, 817, 818, 819, 820, 821, 822, 823, 824, 825, 827, 846, 847, 848, 849, 851, 854, 855, 856, 857, 858, 863, 864, 883, 884, 889, 926, 928, 929, 930, 931, 932, 933, 934, 935, 951, 963, 964, 965, 971, 980, 990, 991, 992, 1001, 1002, 1003, 1004, 1005, 1006, 1011, 1012, 1013, 1014, 1024, 1026, 1033, 1035, 1057, 1064, 1077, 1078, 1079, 1083, 1085, 1086, 1087, 1088, 1089, 1090, 1091, 1094, 1095, 1096, 1097, 1098, 1105, 1106, 1107, 1108, 1110, 1111, 1112, 1113, 1114, 1115, 1116, 1117, 1118, 1119, 1120, 1121, 1122, 1140, 1141, 1142, 1143, 1148, 1221, 1231, 1241, 1244, 1245, 1254, 1350, 1351, 1352, 1353, 1354, 1355, 1359, 1360, 1371, 1372, 1388, 1401, 1405, 1410, 1411, 1427, 1431, 1435, 1441, 1455, 1470, 1492, 1493, 1498, 1501, 1554, 1590, 1600, 1610, 1712, 1715, 1716, 1747, 1934, 1935, 1936, 1943, 1948, 1965, 1966, 1970, 1972, 1979, 1982, 1992, 2014, 2015, 2019, 2045, 2060, 2123, 2124, 2125, 2126, 2127, 2128, 2129, 2130, 2131, 2147, 2148, 2149, 2150, 2154, 2160, 2190, 2191, 2192, 2202, 2203, 2206, 2207, 2211, 2212, 2221, 2222, 2223, 2224, 2257, 2263, 2264, 2273, 2274, 2275, 2278, 2279, 2280, 2284, 2285, 2286, 2287, 2296, 2297, 2300, 2313, 2314, 2389, 2398, 2407, 2433, 2434, 2438, 2439, 2489, 2490, 2492, 2493, 2494, 2495, 2496, 2497, 2498, 2499, 2500, 2501, 2502, 2506, 2507, 2508, 2509, 2510, 2511, 2512, 2513, 2514, 2515, 2516, 2517, 2518, 2519, 2520, 2521, 2522, 2523, 2524, 2525, 2526, 2527, 2528, 2530, 2531, 2542, 2546, 2547, 2548, 2549, 2557, 2563, 2575, 2627, 2632, 2654, 2656, 2657, 2658, 2659, 2661, 2667, 2675, 2678, 2681, 2697, 2698, 2704, 2705, 2706, 2707, 2714, 2715, 2776, 2778, 2780, 2789, 2792, 2822, 2823, 2825, 2826, 2827, 2828 |

### 5a long simple (no long long ni long double)

Expresión (Python re):

```text
(?<!\w)(?<!long )long\b(?!\s+(?:long|double))
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Config/UserSettings.cpp` | 1 | 36 |
| `src/Net/Recv/Recv_Account.cpp` | 1 | 103 |
| `src/Platform/ClientSettings.h` | 1 | 139 |
| `src/Platform/GL/glad.h` | 6 | 1634, 1635, 1689, 1690, 1697, 1698 |
| `src/Platform/SDL3/SDLAudio.cpp` | 1 | 61 |
| `src/Platform/SDL3/SDLTimer.cpp` | 1 | 53 |
| `src/Render/Texture/ImageCodec.cpp` | 1 | 31 |
| `src/Sound/SoundManager.cpp` | 3 | 42, 46, 51 |
| `src/Sound/SoundManager.h` | 1 | 52 |
| `src/UI/EventTimer.cpp` | 4 | 149, 151, 152, 152 |
| `src/UI/MoveList.cpp` | 1 | 204 |
| `src/UI/OptionsMenu.cpp` | 2 | 518, 518 |

### 5b wchar_t

Expresión (Python re):

```text
\bwchar_t\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Platform/ClientEntry.cpp` | 3 | 369, 369, 369 |
| `src/Platform/Win32/Win32Input.cpp` | 5 | 37, 37, 79, 80, 80 |

### 5c supuestos explícitos de ancho de puntero

Expresión (Python re):

```text
\bsizeof\s*\(\s*(?:void\s*\*|HWND|HDC|HFONT|WPARAM|LPARAM|SOCKET)\s*\)\s*==\s*4
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Platform/Posix/WinTypes.h` | 7 | 88, 92, 92, 92, 93, 93, 93 |

### 5d asm inline

Expresión (Python re):

```text
\b(?:__asm|__asm__)\b|\basm\s*\(
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| — | 0 | — |

### 5e precisión extendida/x87 traducido

Expresión (Python re):

```text
\b(?:float10|long\s+double|fsin|fcos|fpatan|fptan|_controlfp|__ftol|_ftol)\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Entity/CharacterEffects.cpp` | 2 | 220, 267 |
| `src/Entity/Entity_BoneEffects.cpp` | 2 | 57, 73 |
| `src/Entity/Entity_Reset.cpp` | 25 | 47, 108, 108, 108, 108, 109, 109, 129, 129, 129, 129, 130, 130, 190, 190, 190, 190, 192, 192, 210, 210, 210, 210, 211, 211 |
| `src/Input/Cursor_Render.cpp` | 5 | 115, 115, 115, 115, 116 |
| `src/Math/Math_3D.cpp` | 43 | 42, 43, 44, 46, 46, 46, 46, 48, 48, 48, 48, 49, 49, 49, 49, 51, 51, 51, 51, 54, 55, 55, 55, 60, 60, 60, 62, 65, 65, 65, 65, 66, 66, 66, 66, 68, 68, 68, 68, 69, 69, 69, 69 |
| `src/Physics/Cloth_Simulation.cpp` | 1 | 190 |
| `src/Render/Ambient_Particles.cpp` | 7 | 92, 277, 277, 277, 278, 279, 308 |
| `src/Render/BMD_SetupRender.cpp` | 25 | 25, 26, 27, 120, 120, 120, 120, 127, 127, 127, 127, 128, 128, 156, 157, 158, 158, 159, 159, 234, 234, 234, 234, 235, 235 |
| `src/Render/Backend/GL/FragmentPipeline.cpp` | 2 | 66, 69 |
| `src/Render/Camera.cpp` | 17 | 41, 41, 41, 42, 42, 43, 43, 44, 44, 52, 52, 53, 54, 55, 55, 56, 56 |
| `src/Render/Effect_Create.cpp` | 28 | 110, 111, 112, 328, 328, 328, 328, 329, 330, 330, 330, 330, 331, 573, 573, 574, 574, 574, 575, 575, 575, 577, 577, 577, 579, 579, 579, 862 |
| `src/Render/Entity_AnimationLegacy.cpp` | 1 | 420 |
| `src/Render/Entity_DrawByType.cpp` | 10 | 479, 479, 479, 480, 480, 743, 743, 743, 744, 744 |
| `src/Render/Entity_DrawSetup.cpp` | 9 | 46, 452, 452, 452, 452, 453, 453, 455, 456 |
| `src/Render/Entity_Render_3D.cpp` | 40 | 132, 414, 414, 414, 414, 415, 415, 416, 431, 431, 431, 431, 432, 432, 433, 433, 434, 440, 440, 440, 440, 441, 441, 442, 442, 574, 574, 574, 574, 575, 575, 576, 576, 578, 590, 590, 590, 590, 592, 592 |
| `src/Render/GL_2D.cpp` | 1 | 144 |
| `src/Render/GL_LegacyState.cpp` | 4 | 116, 117, 119, 120 |
| `src/Render/Joint_Create.cpp` | 20 | 38, 39, 1784, 1785, 1785, 1785, 1786, 1787, 1787, 1787, 1788, 1826, 1827, 1827, 1827, 1828, 1829, 1829, 1829, 1830 |
| `src/Render/MoveEffect.cpp` | 67 | 280, 281, 548, 747, 747, 747, 747, 748, 748, 750, 750, 750, 750, 751, 751, 772, 803, 803, 804, 804, 810, 810, 810, 810, 811, 811, 812, 812, 813, 813, 814, 815, 816, 816, 817, 818, 981, 981, 981, 981, 982, 982, 1052, 1081, 1088, 1094, 1427, 1467, 1493, 1494, 1592, 1633, 1767, 1772, 1914, 1981, 2048, 2121, 2121, 2121, 2121, 2121, 2122, 2122, 2123, 2366, 2366 |
| `src/Render/MoveEffect_Helpers.cpp` | 7 | 302, 302, 303, 303, 304, 305, 306 |
| `src/Render/MoveJoint.cpp` | 30 | 573, 715, 716, 718, 720, 721, 723, 1285, 1286, 1305, 1308, 1351, 1352, 1356, 1357, 1360, 1361, 1366, 1367, 1434, 1436, 1437, 1438, 1440, 1440, 1501, 1558, 1559, 1565, 1566 |
| `src/Render/Particle_Move.cpp` | 75 | 69, 207, 207, 207, 207, 210, 224, 224, 224, 224, 225, 230, 239, 239, 239, 239, 240, 247, 247, 247, 247, 248, 248, 315, 315, 315, 316, 316, 317, 317, 318, 318, 319, 319, 333, 333, 333, 334, 334, 335, 335, 336, 336, 337, 337, 408, 408, 408, 408, 408, 409, 409, 504, 504, 504, 504, 505, 505, 590, 592, 592, 592, 592, 593, 593, 595, 595, 600, 600, 600, 600, 601, 601, 952, 952 |
| `src/Render/Particle_Spawn.cpp` | 23 | 294, 295, 652, 652, 652, 652, 653, 653, 655, 667, 667, 667, 667, 668, 668, 670, 681, 682, 684, 684, 687, 687, 689 |
| `src/Render/Render_LegacyTransforms.cpp` | 2 | 173, 173 |
| `src/Render/Terrain_Water.cpp` | 12 | 43, 44, 82, 82, 87, 87, 104, 107, 107, 107, 108, 109 |
| `src/Render/Weather.cpp` | 38 | 140, 563, 599, 599, 599, 602, 602, 603, 621, 621, 625, 625, 634, 634, 634, 635, 635, 636, 636, 636, 638, 638, 639, 639, 639, 639, 640, 640, 640, 640, 642, 642, 776, 777, 777, 777, 779, 783 |
| `src/Scene/Scene_CharSelect.cpp` | 13 | 197, 197, 198, 198, 198, 224, 550, 550, 550, 553, 553, 555, 556 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 6 | 1027, 1028, 1032, 1038, 1039, 1042 |
| `src/Terrain/Terrain.cpp` | 1 | 625 |
| `src/Terrain/Terrain_Light.cpp` | 20 | 88, 105, 105, 105, 105, 106, 106, 176, 177, 178, 206, 206, 212, 212, 220, 230, 230, 230, 231, 232 |
| `src/Util/Misc.cpp` | 5 | 169, 169, 171, 173, 175 |
| `src/functions.h` | 7 | 9, 9, 600, 601, 627, 653, 846 |
| `src/stdafx.h` | 3 | 18, 22, 23 |
| `src/types.h` | 2 | 17, 17 |

### 6 transporte conocido FUN_004fdc00

Expresión (Python re):

```text
\bFUN_004fdc00\s*\(
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Scene/Scene_ObjectLegacy.cpp` | 1 | 46 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 2 | 629, 1016 |
| `src/functions.h` | 1 | 1297 |

### 6b LODWORD/SLODWORD (candidatos adicionales)

Expresión (Python re):

```text
\b(?:LODWORD|SLODWORD)\s*\(
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Combat_LegacyAnimation.cpp` | 2 | 29, 33 |
| `src/Combat/Combat_LegacyAttackEffects.cpp` | 2 | 19, 23 |
| `src/Combat/Skills_WarriorLegacy.cpp` | 2 | 22, 26 |
| `src/Config/Config_TokenParser.cpp` | 2 | 21, 25 |
| `src/Core/Runtime_Externs.cpp` | 2 | 32, 36 |
| `src/Core/Runtime_Linker.cpp` | 2 | 21, 25 |
| `src/Core/String_ResourceLegacy.cpp` | 2 | 33, 37 |
| `src/Entity/Entity_ActionLegacy.cpp` | 2 | 30, 34 |
| `src/Entity/Entity_BoneEffects.cpp` | 2 | 23, 27 |
| `src/Entity/Entity_LegacyEffects.cpp` | 2 | 30, 34 |
| `src/Entity/Entity_LegacyPoolCleanup.cpp` | 2 | 23, 27 |
| `src/Entity/Entity_LegacyTeleport.cpp` | 2 | 21, 25 |
| `src/GameGuard/GameGuard_LegacyHealth.cpp` | 2 | 21, 25 |
| `src/Item/Item_Durability.cpp` | 2 | 24, 28 |
| `src/Item/Item_LegacyGridInsert.cpp` | 2 | 19, 23 |
| `src/Item/Item_LegacyHelpers.cpp` | 2 | 22, 26 |
| `src/Item/Item_LegacyLinker.cpp` | 2 | 21, 25 |
| `src/Item/Item_TypeLegacy.cpp` | 2 | 20, 24 |
| `src/Math/Math_LegacyAngle.cpp` | 2 | 22, 26 |
| `src/Math/Math_LegacyTransforms.cpp` | 2 | 20, 24 |
| `src/Net/Net_LegacyAuth.cpp` | 2 | 32, 36 |
| `src/Net/Net_LegacyRuntime.cpp` | 2 | 34, 39 |
| `src/Path/Path_Legacy.cpp` | 2 | 32, 36 |
| `src/Path/Path_LegacyReset.cpp` | 2 | 30, 34 |
| `src/Render/BMD_LegacyDraw.cpp` | 2 | 31, 35 |
| `src/Render/BMD_LegacyLoading.cpp` | 2 | 26, 30 |
| `src/Render/Entity_AnimationLegacy.cpp` | 2 | 31, 35 |
| `src/Render/Font_LegacyFactory.cpp` | 2 | 33, 37 |
| `src/Render/GL_LegacyState.cpp` | 2 | 27, 31 |
| `src/Render/Render_LegacyBillboards.cpp` | 2 | 36, 40 |
| `src/Render/Render_LegacyDamageNumbers.cpp` | 2 | 25, 29 |
| `src/Render/Render_LegacyLinker.cpp` | 2 | 22, 26 |
| `src/Render/Render_LegacyRotatedRect.cpp` | 2 | 33, 37 |
| `src/Render/Render_LegacyTrails.cpp` | 2 | 33, 37 |
| `src/Render/Render_LegacyTransforms.cpp` | 2 | 32, 36 |
| `src/Render/Render_WorldHelpers.cpp` | 2 | 36, 40 |
| `src/Render/SMD_Parser.cpp` | 2 | 41, 45 |
| `src/Scene/Scene_CharSelect_Nav.cpp` | 2 | 50, 55 |
| `src/Scene/Scene_LegacyCleanup.cpp` | 2 | 20, 24 |
| `src/Scene/Scene_ObjectLegacy.cpp` | 3 | 19, 23, 47 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 134 | 1011, 1067, 1074, 1078, 1083, 1088, 1092, 1097, 1098, 1102, 1103, 1104, 1105, 1109, 1110, 1114, 1114, 1115, 1118, 1124, 1126, 1136, 1137, 1138, 1139, 1145, 1151, 1155, 1155, 1168, 1178, 1183, 1184, 1191, 1194, 1203, 1204, 1205, 1211, 1218, 1227, 1228, 1233, 1233, 1233, 1234, 1235, 1235, 1235, 1236, 1240, 1242, 1242, 1244, 1246, 1246, 1248, 1250, 1252, 1254, 1260, 1262, 1264, 1266, 1268, 1270, 1272, 1274, 1282, 1286, 1293, 1294, 1297, 1302, 1303, 1307, 1311, 1318, 1322, 1327, 1331, 1332, 1335, 1344, 1351, 1358, 1361, 1363, 1365, 1366, 1370, 1373, 1374, 1378, 1382, 1385, 1386, 1387, 1390, 1392, 1394, 1398, 1399, 1403, 1404, 1409, 1411, 1412, 1419, 1422, 1423, 1427, 1429, 1430, 1435, 1436, 1438, 1445, 1448, 1451, 1452, 1455, 1463, 1466, 1471, 1474, 1475, 1478, 1479, 1480, 1481, 1493, 1494, 1497 |
| `src/Terrain/Terrain_LegacyLoad.cpp` | 2 | 32, 36 |
| `src/Terrain/Terrain_LegacyRender.cpp` | 2 | 21, 25 |
| `src/UI/UI_DialogInterface.cpp` | 2 | 21, 25 |
| `src/UI/UI_GuildLegacy.cpp` | 2 | 36, 40 |
| `src/UI/UI_LegacyExterns.cpp` | 2 | 43, 47 |
| `src/UI/UI_LegacyGameHelpers.cpp` | 2 | 33, 37 |
| `src/UI/UI_LegacyText.cpp` | 2 | 24, 28 |
| `src/UI/UI_LegacyWidgetSystem.cpp` | 2 | 33, 37 |
| `src/UI/UI_TextBitmap.cpp` | 8 | 226, 233, 235, 247, 256, 258, 262, 263 |
| `src/ghidra_compat.h` | 2 | 17, 21 |

### cierre direcciones absolutas casteadas (candidatos)

Expresión (Python re):

```text
\([^;\n()]*\*\s*\)\s*0x[0-9a-fA-F]{6,8}\b
```

| Archivo | Sitios | Líneas en la base |
|---|---:|---|
| `src/Combat/Skills_WarriorLegacy.cpp` | 8 | 310, 386, 389, 394, 396, 401, 404, 472 |
| `src/Platform/Win32/Win32Audio.cpp` | 1 | 55 |
| `src/Render/Effect_Create.cpp` | 4 | 515, 1039, 1238, 1385 |
| `src/Render/Entity_Render_3D.cpp` | 1 | 488 |
| `src/Render/MoveEffect.cpp` | 16 | 445, 557, 659, 1029, 1105, 1272, 1371, 1642, 1782, 1867, 1946, 2098, 2111, 2178, 2225, 2400 |
| `src/Render/MoveEffect_Helpers.cpp` | 2 | 230, 253 |
| `src/Render/Weather.cpp` | 3 | 269, 289, 693 |
| `src/Scene/Scene_ObjectUpdate.cpp` | 1 | 478 |
| `src/Sound/Music.cpp` | 2 | 133, 139 |
| `src/Trade/Trade.cpp` | 3 | 132, 148, 337 |

### Reproducción del barrido

Ejecutar este bloque Python en la raíz del commit base (o de este PR, que sólo
agrega documentación). Usa las expresiones del anexo sin depender de archivos
externos ni incorporar herramientas al build. No aplica el preprocesador ni un
AST; produce los mismos candidatos, con las limitaciones indicadas arriba.

```python
from pathlib import Path
import re, subprocess
documento = Path("doc-port/x64-inventario.md").read_text(encoding="utf-8")
patrones = re.findall(r"### ([^\n]+)\n\nExpresión \(Python re\):\n\n```text\n([^\n]+)\n```", documento)
archivos = subprocess.check_output(["git", "ls-files", "src"], text=True).splitlines()
archivos = [p for p in archivos if Path(p).suffix in (".cpp", ".h", ".c", ".hpp", ".inl")]
ignorar = re.compile('//[^\\n]*|/\\*.*?\\*/|"(?:\\\\.|[^"\\\\])*"|\\\'(?:\\\\.|[^\\\'\\\\])*\\\'', re.S)
for nombre, patron in patrones:
    sitios = 0
    afectados = 0
    for ruta in archivos:
        if nombre == "3b aserciones de red" and not ruta.startswith("src/Net/Protocol/"):
            continue
        if nombre.startswith("4c ") and ruta != "src/globals.h":
            continue
        texto = Path(ruta).read_text(encoding="utf-8-sig", errors="replace")
        texto = ignorar.sub(lambda m: "".join("\n" if c == "\n" else " " for c in m.group()), texto)
        coincidencias = list(re.finditer(patron, texto, re.I if nombre.startswith("2") else 0))
        if coincidencias:
            lineas = [texto.count("\n", 0, m.start()) + 1 for m in coincidencias]
            print(nombre, ruta, len(lineas), lineas)
            sitios += len(lineas)
            afectados += 1
    print(nombre, "TOTAL", sitios, "ARCHIVOS", afectados)
```
