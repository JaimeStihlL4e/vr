# Plan de desarrollo — Prototipo de rehabilitación en VR con terapeuta virtual

> Documento vivo. Se actualiza al cerrar cada fase.

## Contexto

Se quiere comprobar si es **viable vender** un producto de terapia virtual: una PWA que se ejecuta en un móvil montado en un adaptador de gafas VR, donde un terapeuta virtual indica al usuario qué ejercicios hacer. El objetivo del código es **responder a esa pregunta**, no construir un producto.

**Principio rector:** antes de desarrollar nada, se recurre a lo que ya existe. El prototipo debe ser **casi todo integración y casi nada código propio**.

## Qué se reutiliza en lugar de desarrollar

| En vez de desarrollar | Se reutiliza |
|---|---|
| Vite + TypeScript + Three.js | **A-Frame 1.8** — framework declarativo en HTML que ya incluye Three.js r184, WebXR, controles de cabeza, carga de glTF y animaciones. Sin build ni TypeScript: un `index.html` |
| Estéreo y giroscopio a mano | **webxr-polyfill** con `cardboard: true`, que emula una sesión `immersive-vr` con distorsión de lente sobre el giroscopio |
| Menús e interacción por mirada | componente `cursor` de A-Frame con `fuse` (activación por permanencia) |
| Modelar un avatar | avatar de **Ready Player Me** + animaciones de **Mixamo** (o modelos CC0 de Quaternius) |
| Motor de voz | **Web Speech API** (`SpeechSynthesis`), incluida en el navegador, con voces en español |
| Empaquetado PWA | manifiesto escrito a mano + service worker mínimo de caché (~20 líneas). Workbox es innecesario a esta escala |
| Servidor HTTPS | Netlify o GitHub Pages; evita montar certificados locales |

Código propio previsto: el `index.html` de la escena, el guion de la rutina como un array, y un par de componentes pequeños de A-Frame. Poco más.

## Aclaración técnica importante

**WebXR no sustituye a Three.js ni a A-Frame.** Es la API del navegador que da acceso al visor (pose de la cabeza, render estéreo); no dibuja nada por sí sola y siempre necesita un motor de render encima. Lo que sustituye al conjunto Vite + TypeScript + Three.js es A-Frame, que trae ese motor incluido y además habla WebXR por nosotros.

Sobre el soporte de `immersive-vr` en teléfonos circula la idea de que se dio por cerrado con el fin de la era Cardboard. **La Fase 0 demuestra que no es así en el dispositivo de pruebas**: Chrome 153 sobre Android da sesión VR nativa, sin polyfill. Por eso el plan empieza comprobándolo en el móvil concreto en lugar de fiarse de la documentación: si hay sesión nativa, A-Frame la usa sin más; si no la hubiera, el webxr-polyfill la emula y A-Frame no nota la diferencia. En ambos casos el resto del plan es idéntico, que es justo lo que hace barata la comprobación.

## Lo que el prototipo debe demostrar

1. Que se instala como app desde el móvil y arranca sin navegador visible.
2. Que dentro del adaptador se ve en estéreo y mirar alrededor funciona con fluidez.
3. Que un terapeuta virtual con voz dirige una rutina corta de principio a fin.

Nada más. Si esas tres cosas convencen, el producto es viable; si no, ninguna funcionalidad añadida lo arregla.

## Fuera de alcance (deliberadamente)

Medición de los movimientos del usuario, historial de sesiones, base de datos, exportación, cuentas, motor configurable de rutinas, audio pregrabado, mandos o hand-tracking, tests automáticos. Eso es producto, no viabilidad.

---

## Fases

Cinco pasos, en orden. Cada uno deja algo probado en el móvil real.

### Fase 0 — Prueba de capacidades en el móvil ✅ COMPLETADA

- [x] `test.html` desplegado y ejecutado en el dispositivo de pruebas

**Resultados** (Chrome 153, Android, Adreno 650 / Snapdragon 865, pantalla 393×873 @2.75x):

| Comprobación | Resultado | Consecuencia |
|---|---|---|
| `immersive-vr` **nativo** | **soportado** | No hace falta polyfill. A-Frame entra en VR por la vía estándar |
| `immersive-ar` nativo | soportado | El dispositivo tiene ARCore; abre la puerta a una variante en AR más adelante |
| Sesión VR real | abierta y cerrada, **2 vistas** | El estéreo funciona de verdad, no solo la declaración de soporte |
| Polyfill | cargado, cede ante el nativo | Queda como red de seguridad para otros dispositivos, sin coste |
| Giroscopio | permiso concedido, eventos recibidos | Disponible, aunque con sesión nativa el tracking lo gestiona Chrome |
| Voces `es-ES` | 2 (es_ES, es_US), reproducción completada | La voz del terapeuta es viable sin grabar audio |
| Pantalla completa + bloqueo horizontal | ambos correctos | La PWA puede ocupar la pantalla sin interfaz de navegador |
| WebGL 2 | disponible | Margen de sobra para la escena prevista |

**Conclusión:** los tres riesgos que podían hundir el prototipo quedan descartados. El camino es WebXR nativo, sin emulación.

### Fase 1 — Esqueleto A-Frame, PWA instalable y desplegada

- [ ] `index.html` con A-Frame (vendorizado en local para que funcione offline), escena mínima con suelo, cielo y un par de cajas
- [ ] `manifest.webmanifest`: `display: "fullscreen"`, `orientation: "landscape"`, iconos 192/512 + maskable, meta tags de iOS
- [ ] Service worker de caché sencillo que precachee A-Frame, el modelo y los iconos
- [ ] Despliegue en Netlify/GitHub Pages; desde aquí cada avance se publica y se prueba en el teléfono
- [ ] Pantalla de inicio: título, aviso de prototipo (no es un dispositivo médico) y botón "Entrar en VR"
- [ ] `git init` y primer commit

**Aceptación:** la PWA se instala desde la URL pública y arranca en modo avión.

### Fase 2 — Entrar en VR en el móvil

*Simplificada tras la Fase 0: con sesión nativa disponible, A-Frame entra en VR por sí solo y el polyfill sale del camino crítico.*

- [ ] El botón "Entrar en VR" resuelve todo en un único gesto de usuario: `scene.enterVR()`, pantalla completa, bloqueo en horizontal y *warm-up* del sintetizador de voz — que sin gesto previo queda mudo en móvil
- [ ] `webxr-polyfill` cargado solo si `isSessionSupported('immersive-vr')` devuelve `false`, como red de seguridad para otros dispositivos. Sin esfuerzo de desarrollo si no se activa
- [ ] Comprobar el encaje con el adaptador: nitidez, separación interocular y recentrado de la vista

**Aceptación:** con el móvil en el adaptador se ve estéreo cómodo, mirar alrededor responde sin deriva y se sale sin reiniciar.

### Fase 3 — Sala y terapeuta virtual

La fase de más valor comercial: es lo que se enseña para vender.

- [ ] Sala sobria y luminosa con `a-plane`, `a-sky` y algún modelo CC0; sin sombras dinámicas
- [ ] Avatar de Ready Player Me con `gltf-model` + animaciones de Mixamo vía `animation-mixer`: reposo y demostraciones. Si las animaciones se complican, alternativa aceptada: figura estilizada con movimiento simple — lo importante es que haya alguien que habla y demuestra
- [ ] Voz con `SpeechSynthesis` en `es-ES` y subtítulos con `a-text` siempre visibles: accesibilidad y red de seguridad si el TTS falla o se corta, cosa habitual en iOS

**Aceptación:** el terapeuta saluda con voz en español, los subtítulos acompañan y demuestra un movimiento mientras habla.

### Fase 4 — Rutina guiada y cierre

- [ ] Guion de 3 o 4 ejercicios (rotación cervical, flexo-extensión, inclinación lateral, seguimiento con la mirada) como un array en un `.js`: texto, animación, repeticiones y descanso. Sin motor ni configuración
- [ ] Avance por temporizador con conteo hablado y mensaje de cierre; interacción limitada a "empezar" y "salir" con el `cursor` de A-Frame en modo `fuse`
- [ ] `README.md` breve: qué es, cómo se prueba, limitaciones conocidas
- [ ] Prueba en 2 móviles distintos

**Aceptación:** una rutina completa se ejecuta de principio a fin sin tocar la pantalla.

---

## Verificación de extremo a extremo

1. Abrir la URL pública en un móvil e instalar la PWA.
2. Modo avión, lanzar desde el icono: arranca sin red.
3. Pulsar "Entrar en VR" y conceder el permiso de giroscopio.
4. Colocar el móvil en el adaptador: estéreo correcto, sala visible, mirada fluida.
5. El terapeuta saluda, demuestra y dirige la rutina hasta el final.
6. Salir con la mirada.

Si un tercero completa esos seis pasos sin ayuda, el prototipo ha cumplido su función.

## Riesgos y cómo se cubren

- ~~**El polyfill lleva años sin mantenimiento activo.**~~ **Descartado en la Fase 0:** el dispositivo de pruebas da sesión `immersive-vr` nativa, así que el prototipo no depende del polyfill. Sigue cargado como red de seguridad para otros teléfonos, pero un fallo suyo ya no bloquea nada.
- **La voz sintética se comporta de forma irregular en iOS.** Cubierto por los subtítulos permanentes desde el primer momento. El dispositivo de pruebas es Android y ya se ha verificado que reproduce español correctamente.
- **Un solo dispositivo probado.** Todo lo anterior está confirmado en un Snapdragon 865 con Chrome. Un teléfono de gama media o un iPhone pueden comportarse de otro modo; la Fase 4 contempla probar en un segundo móvil.
- **Cinetosis.** La cámara nunca se desplaza por código: todo el movimiento lo genera el usuario. Sesión corta y modo sentado.

## Ritmo de trabajo

Una sesión de trabajo por fase, en orden. La Fase 0 puede desbloquear o encarecer el resto, así que no se salta. Si una fase se alarga, se cierra con el alcance reducido y se anota, en lugar de arrastrar trabajo a medias.

---

*Aviso: prototipo de evaluación. No es un dispositivo médico y no sustituye el criterio de un profesional sanitario.*

Fuentes consultadas: [A-Frame 1.8.0](https://vr.org/articles/aframe-1-8-0-webxr-open-source-june-2026) · [WebXR Browser Support in 2026](https://www.testmuai.com/learning-hub/webxr-compatible-browsers/) · [Chrome Hardware Support](https://m-blix.github.io/immersiveweb.dev/chrome-support.html) · [webxr-polyfill](https://github.com/immersive-web/webxr-polyfill)
