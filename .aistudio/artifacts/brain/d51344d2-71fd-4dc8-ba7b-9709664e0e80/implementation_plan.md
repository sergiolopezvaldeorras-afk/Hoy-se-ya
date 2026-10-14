# Físicas de Colisión, Ragdoll Procedural y Sanción FIA para Mecánicos Rivales

Implementación de un subsistema de físicas dinámicas para los mecánicos de los garajes rivales en el pit lane, con respuesta de impacto tipo propulsión parabólica, agitación realista de brazos y piernas, reaparición automática tras 4 segundos, sanción acumulativa de +5 segundos en el tiempo de carrera y banner de comisarios FIA durante 3 segundos en el HUD.

## Decisiones de Usuario y Confirmaciones

> [!IMPORTANT]
> Se han integrado las preferencias seleccionadas por el usuario en la fase de clarificación:

- **Sanción de 5 segundos**: Se suma directamente al tiempo total de carrera del jugador (`lapTime` / tiempo de carrera final), acumulable si se atropella a más de un mecánico.
- **Estilo de animación de impacto**: Vuelo parabólico con rotación tridimensional e impulsos caóticos en las articulaciones de hombros, codos, caderas y rodillas simulando el impacto a alta velocidad.
- **Aviso en pantalla**: Banner superior rojo estilo comisarios FIA con advertencia sonora durante exactamente 3 segundos ("+5s PENALTY · PIT CREW INCIDENT").

---

## 1. Visión General del Sistema

Actualmente los mecánicos del garaje del jugador (`pitBoxX = -22.0`) realizan el pit stop interactivo, mientras que los mecánicos de los garajes rivales (`x = +22.0`, etc.) se compilan como una malla estática fusionada sin colisionador. 

Con este cambio:
1. Los mecánicos rivales conservan su jerarquía anatómica completa (`proceduralBody`, `pelvisGroup`, `leftArmIK`, `rightArmIK`, `leftLeg`, `rightLeg`, `headGroup`, `torsoGroup`).
2. Se integran en el bucle de detección de colisiones de `CollisionSubsystem` y `TrackBuilder` / `PitStopManager`.
3. Al colisionar el coche con un mecánico rival:
   - Sale despedido con un vector de fuerza proporcional a la velocidad del vehículo y el ángulo de impacto.
   - Entra en estado de ragdoll procedural con rotación física y oscilaciones dinámicas de brazos y piernas.
   - Sonido de impacto mecánico/físico contundente.
   - Se aplican +5.0 segundos de penalización al tiempo de carrera.
   - Se dispara la alerta de penalización FIA en el HUD durante 3.0 segundos con tono de advertencia.
   - Transcurridos exactamente 4.0 segundos, el mecánico reaparece en su coordenada original restableciendo su postura de trabajo.
4. Los mecánicos del garaje propio del jugador se mantienen protegidos en su zona de servicio para no interferir con las paradas en boxes reglamentarias.

---

## 2. Experiencia de Usuario y Diseño Visual

### Banner de Sanción FIA (HUD)
- **Posición**: Anclado en la parte superior central de la pantalla (debajo de los semáforos y fuera del cono visual principal del circuito).
- **Estética**: Banner rectangular estilo gráficos oficiales de retransmisión F1 / FIA Stewards:
  - Fondo negro profundo con ribete y acento carmesí (`#EF4444`).
  - Tipografía deportiva angular (`Chakra Petch` / `Plus Jakarta Sans`) en mayúsculas: `STEWARDS DECISION · +5s TIME PENALTY`.
  - Subtexto descriptivo: `COLLISION WITH RIVAL PIT CREW`.
  - Entrada y salida suave con transición de opacidad y escala sin alterar el rendimiento de renderizado.
- **Duración**: Desaparece automáticamente tras 3.0 segundos.
- **Sonido**: Tono sutil de advertencia FIA sintetizado o sample de aviso de radio.

### Animación Dinámica del Mecánico
- **Vuelo**: Trayectoria balística (gravedad $-18\,\text{m/s}^2$, velocidad inicial con componente $Y$ vertical y dispersión $XZ$).
- **Cinemática articular**:
  - Hombros y codos oscilando con frecuencias sinusoidales asimétricas y amortiguamiento.
  - Caderas y rodillas flexionándose y estirándose durante la trayectoria.
  - Rotación angular en los tres ejes ($X$, $Y$, $Z$).
- **Reaparición**: A los 4.0 segundos, tras tocar tierra o disiparse el impacto, el mecánico se resetea suavemente a su posición original con opacidad restaurada.

---

## 3. Arquitectura Técnica y Estrategia de Rendimiento

```
┌────────────────────────────────────────────────────────┐
│                   VehiclePhysics                       │
│    (Velocidad, posición, ángulo yaw del vehículo)      │
└──────────────────────────┬─────────────────────────────┘
                           │ Coordenadas (X, Y, Z)
                           ▼
┌────────────────────────────────────────────────────────┐
│               CollisionSubsystem & Pit                 │
│  - Chequeo de distancia frente a mecánicos rivales     │
│  - Exclusión estricta de mecánicos del propio garaje   │
│  - Detección de impacto (radio ~0.65m)                 │
└──────────────────────────┬─────────────────────────────┘
                           │ Evento de impacto
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌──────────────────────┐        ┌───────────────────────┐
│ RivalMechanicActor   │        │   RacingGameEngine    │
│ - Física balística   │        │ - +5s Penalty Time    │
│ - Ragdoll procedural │        │ - Disparo aviso FIA   │
│ - Timer 4.0s respawn │        │ - Audio de alerta     │
└──────────────────────┘        └───────────┬───────────┘
                                            │ Telemetría
                                            ▼
                                ┌───────────────────────┐
                                │       HUD.tsx         │
                                │ - Banner Stewards FIA │
                                │ - Temporizador 3 seg  │
                                └───────────────────────┘
```

### Optimización y Presupuesto de Rendimiento
- **Zero Allocations en Tick**: Los vectores y matrices de interpolación para las extremidades se preasignan como variables estáticas reutilizables (`_v1`, `_q1`, `_euler`).
- **Instancing y Culling**: Solo los mecánicos activos en estado de impacto ejecutan el cálculo articular dinámico por fotograma; en reposo permanecen estáticos sin coste computacional adicional.
- **Independencia de Tasa de Refresco**: Actualización basada en `delta` de tiempo para garantizar idéntico comportamiento a 60 FPS y 120 FPS.

---

## 4. Plan de Archivos a Modificar

1. **`src/game/pit/PitStopManager.ts`**:
   - Extraer los mecánicos rivales del proceso de consolidación estática para gestionarlos como `RivalMechanicActor`.
   - Almacenar la lista de mecánicos rivales dinámicos con coordenadas de reposo, orientación y referencias a sus huesos/articulaciones.
   - Implementar método `updateRivalMechanics(dt: number)` que procese la gravedad, rotación, agitación de extremidades y la cuenta atrás de 4.0 segundos para respawn.
   - Implementar `impartImpulseToMechanic(mech, carVel)`.

2. **`src/game/collisions/CollisionSubsystem.ts`**:
   - Agregar método de comprobación de colisión entre el coche y los mecánicos rivales de `PitStopManager`.
   - Al detectar solapamiento: aplicar impulso físico al mecánico, emitir partículas/sonido de impacto, descontar velocidad del coche y notificar a la máquina de estados de carrera.

3. **`src/game/RacingGameEngine.ts`**:
   - Añadir estado de penalización `totalPenaltySeconds` y `penaltyAlert: { text: string; timer: number } | null`.
   - Modificar la telemetría enviada a la UI para incluir el estado de la sanción y la alerta activa.
   - Sumar los 5 segundos al cronómetro de carrera.

4. **`src/components/HUD.tsx`**:
   - Integrar el componente visual de alerta FIA en la parte superior de la pantalla con animación CSS/Tailwind de 3 segundos.
   - Reflejar la penalización acumulada en el cuadro de tiempos.
