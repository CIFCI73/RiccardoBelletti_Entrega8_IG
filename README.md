# RiccardoBelletti_Entrega8_IG

Este proyecto empezó a partir de los ejemplos que habíamos trabajado en clase con Three.js, especialmente los scripts script_24_mapasitycleta.js y script_26_mapaosm.js.
El primero me sirvió como base técnica, porque ya mostraba cómo cargar una textura de mapa (.png), leer un archivo .csv y colocar objetos simples, como esferas, según unas coordenadas geográficas.
El segundo me inspiró más a nivel visual, ya que utilizaba geometrías 3D para representar edificios y quería hacer algo parecido, pero de forma más sencilla.
La idea principal era combinar esas dos cosas: tener un mapa 2D de la ciudad de Rávena (la ciudad donde nací) como fondo y encima colocar modelos 3D de los monumentos mas importantes, no solo esferas, sino figuras que representaran su tipo (iglesia, plaza, etc.).
De esta forma, el objetivo final era crear una visualización interactiva en 3D donde se pudieran ver los monumentos sobre el mapa, mover la cámara con OrbitControls y tener una pequeña interfaz con información.

El punto de partida técnico fue adaptar el script 24. En el código, cambié la imagen original mapaLPGC.png por MapaRavenna.png, que es la textura que uso ahora. También creé un archivo nuevo llamado monumentos.csv, (id;nome;lat;lon;categoria;x_map;y_map), donde guardé la información de los monumentos de Rávena.
Al principio, intenté usar el mismo sistema del script original: las coordenadas geográficas latitud y longitud, que después se convertían en posiciones del mapa con una función llamada Map2Range.
La función tomaba los valores de lon/lat y los transformaba en coordenadas x/y dentro del plano de la textura. En teoría, eso debería colocar los puntos donde correspondía, pero en la práctica no funcionó bien.
El primer problema fue precisamente ese: la calibración del mapa fallaba. Los puntos aparecían bastante lejos de su lugar correcto. Algunos, como los que estaban cerca del centro del mapa, coincidían más o menos bien, pero otros, como la estación o los monumentos en los bordes, aparecían totalmente desplazados.

La manera en la que resolvimos el problema de la proyección fue práctica y directa: en vez de seguir intentando ajustar fórmulas sobre lat/lon (que no funcionaban por la distorsión de la proyección del PNG), hicimos una calibración manual usando la propia escena 3D. Implementamos temporalmente una función onMouseClick que, con un Raycaster, detecta el punto del plano del mapa (mapa) donde hacemos click y devuelve las coordenadas del mundo 3D. Así conseguíamos las coordenadas reales en el sistema de la escena (x, y) de cada monumento, y luego las guardábamos en el CSV como x_map y y_map. El fragmento esencial que usamos es este (tal cual en el proyecto, lo teníamos comentado):

```javascript
function onMouseClick(event) {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);

  // buscamos la intersección con el plano 'mapa'
  const intersects = raycaster.intersectObject(mapa);

  if (intersects.length > 0) {
    const punto = intersects[0].point;
    console.log(`x: ${punto.x.toFixed(3)}, y: ${punto.y.toFixed(3)}`);
    alert(`Coordenadas copiadas en consola: x: ${punto.x.toFixed(3)}, y: ${punto.y.toFixed(3)}`);
  }
}
```
Estas coordenadas (x, y) se copiaron manualmente al archivo monumentos.csv como nuevas columnas x_map y y_map. Así, en lugar de intentar ajustar fórmulas de proyección sobre una textura distorsionada, obtuvimos valores totalmente precisos dentro del sistema 3D real.
Una vez completado el CSV con estas coordenadas calibradas, eliminamos todas las variables que el código original usaba para transformar lat/lon —como minLon, maxLon, Map2Range y los offsets—, y adaptamos la lectura del CSV en la función principal. Por ejemplo, esta parte original:
```javascript
const x = Map2Range(lon, minLon, maxLon, -mapa_w/2, mapa_w/2);
const y = Map2Range(lat, minLat, maxLat, mapa_h/2, -mapa_h/2);
```
se sustituyó por una versión:

```javascript
const x = parseFloat(columna[indices.x_map]);
const y = parseFloat(columna[indices.y_map]);
```
Con eso, cada monumento aparecía exactamente donde debía sobre la textura MapaRavenna.png, sin ningún error de escala o desplazamiento.
Con la parte de posicionamiento resuelta, el trabajo se centró en la representación y la interacción. Reemplacé la vieja función que creaba esferas por una “fábrica” más rica, CreaMonumentos3D, que decide forma, color y tamaño según la categoría leída del CSV.
He sustituido las esferas por un bloque que genera distintas geometrías según la categoría (iglesias,plazas, hubs y monumentos historico) del monumento, que también se añadió al CSV:

```javascript
function CreaMonumentos3D(px, py, id, categoria, nome) {
…
  switch (categoria) {
    case "chiesa":
      material = new THREE.MeshBasicMaterial({ color: coloreChiesa });
      altezza = scalaBase * 2;
      geometry = new THREE.CylinderBufferGeometry(scalaBase, scalaBase, altezza, 8);
      geometry.rotateX(Math.PI / 2);
      break;

    case "piazza":
      material = new THREE.MeshBasicMaterial({ color: colorePiazza });
      altezza = scalaBase * 0.2;
      geometry = new THREE.BoxBufferGeometry(scalaBase * 3, scalaBase * 2, altezza);
      break;

    case "monumento":
      material = new THREE.MeshBasicMaterial({ color: coloreMonumento });
      altezza = scalaBase * 2;
      geometry = new THREE.BoxBufferGeometry(scalaBase * 1.5, scalaBase * 1.5, altezza);
      break;

    case "hub":
      material = new THREE.MeshBasicMaterial({ color: coloreHub });
      altezza = scalaBase * 2.5;
      geometry = new THREE.BoxBufferGeometry(scalaBase * 2, scalaBase * 1.5, altezza);
      break;

    default:
      material = new THREE.MeshBasicMaterial({ color: 0xff00ff });
      altezza = scalaBase;
      geometry = new THREE.BoxBufferGeometry(scalaBase, scalaBase, altezza);
  }

  // Personalizaciones por ID (ej: estación o rocca)
  …
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set(px, py, altezza / 2);
  mesh.name = nome;
  objetos.push(mesh);
  scene.add(mesh);
}

```
Para la interactividad elegimos el Raycaster también en onMouseMove, pero esta vez intersectando con los objetos (los monumentos), no con el mapa. El comportamiento es simple y eficaz: cuando el cursor pasa por un objeto, lo ampliamos (hover effect), mostramos su nombre en un <div id="label"> y cuando el cursor sale, restauramos la escala. El bloque clave es este:

```javascript
function onMouseMove(event) {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);

  const intersects = raycaster.intersectObjects(objetos);

  if (intersects.length > 0) {
    const hoveredObject = intersects[0].object;
    if (currentIntersect !== hoveredObject) {
      if (currentIntersect) currentIntersect.scale.set(1,1,1);
      currentIntersect = hoveredObject;
      currentIntersect.scale.set(1.5,1.5,1.5);
      label.innerHTML = currentIntersect.name;
      label.style.display = "block";
    }
    let vector = new THREE.Vector3();
    currentIntersect.getWorldPosition(vector);
    vector.project(camera);
    let x = (vector.x * 0.5 + 0.5) * window.innerWidth;
    let y = (vector.y * -0.5 + 0.5) * window.innerHeight;
    label.style.left = x + "px";
    label.style.top = y - 30 + "px";
  } else {
    if (currentIntersect) currentIntersect.scale.set(1,1,1);
    label.style.display = "none";
    currentIntersect = null;
  }
}

```
Además del script principal en JavaScript, una parte del trabajo consistió en ajustar la estructura del archivo HTML para que sirviera de base visual y funcional al mapa 3D de Ravenna.
El documento se organiza de manera sencilla, pero contiene varios elementos cruciales. En la cabecera se definen los estilos generales y, sobre todo, la apariencia de la leyenda y de las etiquetas interactivas que aparecen al pasar el ratón sobre los puntos. Por ejemplo, el bloque:

```html
#legenda {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 10px 15px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}
```

sitúa la leyenda en la esquina inferior izquierda con fondo semitransparente y bordes redondeados. Este detalle fue añadido manualmente para mejorar la legibilidad sobre el mapa.
En el <body>, el contenedor principal div id="app" actúa como punto de montaje del canvas generado por Three.js desde el archivo src/index.js, que contiene toda la lógica de carga y renderizado. Justo después se incluye otro elemento fundamental, el div id="label", que funciona como cuadro emergente para mostrar el nombre del lugar cuando el usuario pasa el ratón sobre un marcador. Para hacerlo funcional, se añadió el siguiente bloque CSS:

```html
#label {
  position: absolute;
  display: none;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px 10px;
  border-radius: 5px;
  pointer-events: none;
}
```

Estas propiedades permiten que la etiqueta se posicione libremente sobre el canvas y no interfiera con el movimiento del ratón (pointer-events: none), manteniendo al mismo tiempo un estilo coherente con la interfaz.
Por último, en la parte inferior del cuerpo del HTML se encuentra el bloque <div id="legenda">, que incluye los distintos ítems de la leyenda, cada uno con su color y descripción. Las categorías, como “Iglesia / Sitio Religioso”, “Plaza” o “Monumento Histórico”, están representadas con pequeños círculos de color definidos mediante elementos como:

```html
<div class="legenda-colore" style="background-color: #e63946"></div>
<span>Iglesia / Sitio Religioso</span>
```


He subido el archivo monumento.csv y el vídeo de presentación a este repositorio.
