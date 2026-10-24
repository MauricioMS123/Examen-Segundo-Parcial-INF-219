# Examen-Segundo-Parcial-INF-219
# Clínica Vida Salud · Prototipo de interfaz de usuario

Prototipo web interactivo del **Sistema Web para la Gestión de Citas Médicas para la Clínica Vida Salud**. Documenta la sección de Prototipos de Interfaz de Usuario (metodología PUDS) y permite navegar las vistas de los **17 casos de uso (CU01–CU17)**, organizados en 3 ciclos de desarrollo.

Todo está en un único archivo, `index.html`, hecho con HTML, [Tailwind CSS](https://tailwindcss.com) por CDN y JavaScript Vanilla. No necesita instalación ni servidor: se abre en el navegador (requiere internet para cargar Tailwind).

**Demo en línea:** `https://TU_USUARIO.github.io/clinica-vida-salud-prototipo/`
**Acceso de prueba:** `admin@vidasalud.bo` / `1234`

## Contenido

- [Cómo funciona el prototipo](#cómo-funciona-el-prototipo)
- [Ciclo I: Funcionalidades fundamentales](#ciclo-i-funcionalidades-fundamentales) (CU01–CU09)
- [Ciclo II: Gestión y seguimiento de citas médicas](#ciclo-ii-gestión-y-seguimiento-de-citas-médicas) (CU10–CU14 y CU17)
- [Ciclo III: Historial y reportes administrativos](#ciclo-iii-historial-y-reportes-administrativos) (CU15–CU16)
- [Cómo ejecutarlo](#cómo-ejecutarlo)

## Cómo funciona el prototipo

- **Selector de rol** en la barra superior: Administrador, Recepcionista o Médico (el paciente no tiene cuenta interna). El menú lateral bloquea con un candado las vistas que el rol activo no puede usar (regla RN12).
- **Menú lateral** organizado por ciclos, con el código y el nombre de cada caso de uso.
- **Datos de prueba en memoria:** 4 especialidades, 5 médicos, 5 pacientes con CI boliviano, horarios de 30 minutos, indisponibilidades y citas en todos los estados.
- **Badges de estado:** Programada (azul), Confirmada (índigo), Atendida (verde), Cancelada (rojo) e Inasistencia (ámbar).

```js
// Vistas autorizadas por rol (RN12)
const ACC = {
  Administrador: ['CU01','CU02','CU03','CU04','CU05','CU07','CU15','CU16'],
  Recepcionista: ['CU01','CU02','CU06','CU08','CU09','CU10','CU11','CU12','CU13','CU15','CU17'],
  'Médico':      ['CU01','CU02','CU12','CU14']
};

// Al cambiar de rol, si la vista actual no está autorizada se abre la primera permitida
const setR = r => {
  S.r = r;
  if (!ACC[r].includes(S.cu)) S.cu = ACC[r][2];
  go(S.cu);
};
```

```js
// Cada cita guarda código, paciente, médico, especialidad, fecha, hora de inicio, estado y motivo
const ct = (c, p, m, sp, f, i, e, mo, x = {}) =>
  ({ c: 'CIT-' + c, p: 'PAC-00' + p, m: 'MED-00' + m, sp, f, i, e, mo, cr: '2026-10-20', ...x });

ct('0001', 1, 1, 'Medicina General', TODAY, '08:00', 'Programada', 'Control general', { asi: '08:02' });
```

---

## Ciclo I: Funcionalidades fundamentales

### CU01 · Iniciar sesión
**Actor:** todos los usuarios con cuenta. Pantalla de acceso con correo o identificador y contraseña, y tres botones de prueba rápida para ver las alertas FA01 (campos vacíos), FA02 (credenciales incorrectas) y FA03 (cuenta inactiva).

```js
const FAM = {
  FA01: ['amber', 'Debe ingresar el identificador de acceso y la contraseña.'],
  FA02: ['red',   'Credenciales incorrectas. Verifique sus datos e intente nuevamente.'],
  FA03: ['red',   'La cuenta está inactiva. Contacte al administrador del sistema.']
};

const login = () => {
  let u = $('#lu').value.trim(), p = $('#lp').value;
  if (!u || !p) return fa('FA01');                  // campos vacíos
  let x = U.find(x => x.u == u && x.p == p);
  if (!x) return fa('FA02');                        // credenciales incorrectas
  if (x.s != 'Activo') return fa('FA03');           // cuenta inactiva
  S.r = x.r; toast('Bienvenido, ' + x.u); go(ACC[x.r][2]);
};
```

### CU02 · Cerrar sesión
**Actor:** todos los usuarios con cuenta. Menú desplegable en la esquina superior derecha con los datos del perfil y el rol activo. El botón «Cerrar sesión» pide confirmación y redirige a CU01.

```js
const um = () => { S.um = !S.um; $('#um').classList.toggle('hidden', !S.um); };

// Modal de confirmación; al aceptar cierra la sesión y vuelve a CU01
const out = () => mod('Cerrar sesión', `<p>¿Desea cerrar la sesión de <b>${PF[S.r][0]}</b>?</p> ...`);
// botón «Sí, cerrar sesión»:
cm(); toast('Sesión cerrada correctamente'); go('CU01');
```

### CU03 · Gestionar usuarios y roles
**Actor:** Administrador. Tabla de usuarios con rol y estado, buscador, modal «Registrar usuario» (identificador, contraseña, rol y estado) y acciones para Modificar y Desactivar la cuenta.

CU03, CU04, CU05 y CU06 comparten el mismo motor de formularios. Cada caso de uso solo define su configuración en el objeto `CF`:

```js
const CF = {
  u: { a: U, t: 'Usuarios registrados', nw: 'Registrar usuario', ph: 'Buscar por usuario o rol',
    cols: ['Identificador de acceso', 'Rol', 'Estado'],
    row: x => [x.u, x.r, bd(x.s)],
    txt: x => x.u + x.r,                                   // texto que usa el buscador
    f: [
      { k: 'u', l: 'Identificador de acceso', req: 1, uq: 1 },   // obligatorio y único
      { k: 'p', l: 'Contraseña', t: 'password', req: 1 },
      { k: 'r', l: 'Rol', o: ['Administrador', 'Recepcionista', 'Médico'] },
      { k: 's', l: 'Estado', o: AE }
    ] },
  // m: médicos · e: especialidades · p: pacientes
};
```

Guardar valida campos obligatorios y duplicados; Desactivar alterna el estado de la cuenta:

```js
const sv = (k, i) => {
  let c = CF[k], o = {};
  for (let f of c.f) {
    let v = f.chk ? [...document.querySelectorAll('.ck:checked')].map(z => z.value)
                  : $('#f_' + f.k).value.trim();
    if (f.req && !v.length) return toast('Campo obligatorio: ' + f.l, 1);
    if (f.uq && c.a.some((y, j) => j != i && nz(y[f.k]) == nz(v)))
      return toast(f.l + ': ya existe un registro con ese valor', 1);
    o[f.k] = v;
  }
  i == null ? c.a.push(o) : Object.assign(c.a[i], o);
  cm(); $('#tb').innerHTML = tb(k); toast('Registro guardado correctamente');
};

const dis = (k, i) => {
  let x = CF[k].a[i];
  x.s = x.s == 'Activo' ? 'Inactivo' : 'Activo';
  $('#tb').innerHTML = tb(k); toast('Estado actualizado: ' + x.s);
};
```

### CU04 · Gestionar médicos
**Actor:** Administrador. Buscador por nombre o especialidad, tabla de médicos y formulario con código autogenerado, número de registro profesional único, nombres, apellidos, teléfono, correo, una o varias especialidades activas (casillas) y estado.

```js
m: { a: M, K: 'MED', t: 'Médicos registrados', nw: 'Registrar médico',
  ph: 'Buscar por nombre o especialidad',
  txt: x => x.nom + x.ape + x.esp,                          // busca por nombre o especialidad
  f: [
    { k: 'c',   l: 'Código de médico', auto: 1 },            // autogenerado
    { k: 'reg', l: 'Número de registro profesional', req: 1, uq: 1 },
    { k: 'nom', l: 'Nombres', req: 1 },
    { k: 'ape', l: 'Apellidos', req: 1 },
    { k: 'tel', l: 'Teléfono', req: 1 },
    { k: 'cor', l: 'Correo electrónico', t: 'email', req: 1 },
    { k: 'esp', l: 'Especialidades activas', chk: 1, req: 1 },  // casillas
    { k: 's',   l: 'Estado', o: AE }
  ] }
```

```js
// Código autogenerado y casillas con solo las especialidades activas
let v = f.auto ? x.c || c.K + '-' + String(c.a.length + 1).padStart(3, '0') : ...;
E.filter(e => e.s == 'Activo').map(e => `<label><input type="checkbox" class="ck" value="${e.n}"> ${e.n}</label>`)
```

### CU05 · Gestionar especialidades
**Actor:** Administrador. Buscador por nombre y listado de especialidades. El nombre es obligatorio y no admite duplicados aunque cambien mayúsculas, espacios o tildes.

```js
e: { a: E, K: 'ESP', t: 'Especialidades', nw: 'Registrar especialidad', ph: 'Buscar por nombre',
  f: [
    { k: 'c', l: 'Código de especialidad', auto: 1 },
    { k: 'n', l: 'Nombre', req: 1, uq: 1 },                  // único
    { k: 'd', l: 'Descripción (opcional)' },
    { k: 's', l: 'Estado', o: AE }
  ] }

// Normalización que detecta duplicados: "Pediatría", "pediatria " y "PEDIATRÍA" son iguales
const nz = s => String(s).normalize('NFD').replace(/[\u0300-\u036f]/g, '').trim().toLowerCase();
```

### CU06 · Gestionar pacientes
**Actor:** Recepcionista. Buscador por CI, nombre o apellido, tabla de pacientes y formulario con código autogenerado, CI único, nombres, apellidos, fecha de nacimiento, sexo, teléfono, correo y dirección (opcionales) y estado.

```js
p: { a: P, K: 'PAC', t: 'Pacientes registrados', nw: 'Registrar paciente',
  ph: 'Buscar por CI, nombre o apellido',
  txt: x => x.ci + x.nom + x.ape,
  f: [
    { k: 'c',   l: 'Código de paciente', auto: 1 },
    { k: 'ci',  l: 'Documento de identidad (CI)', req: 1, uq: 1 },   // único
    { k: 'nom', l: 'Nombres', req: 1 },
    { k: 'ape', l: 'Apellidos', req: 1 },
    { k: 'fn',  l: 'Fecha de nacimiento', t: 'date', req: 1 },
    { k: 'sx',  l: 'Sexo', o: ['Femenino', 'Masculino'] },
    { k: 'tel', l: 'Teléfono', req: 1 },
    { k: 'cor', l: 'Correo electrónico (opcional)', t: 'email' },
    { k: 'dir', l: 'Dirección (opcional)' },
    { k: 's',   l: 'Estado', o: AE }
  ] }
```

### CU07 · Gestionar horarios médicos
**Actor:** Administrador. Selector de médico activo con dos pestañas: *Configuración del horario médico* (día de lunes a sábado, hora de inicio, hora de finalización y duración de cita, 30 minutos por defecto) y *Registrar indisponibilidad* (rango de fecha y hora, y motivo: Ausencia, Permiso o Vacaciones), con un aviso automático si hay citas afectadas.

```js
const gh = () => {                                           // guardar horario
  let h = { m: S.m7 || 'MED-001', d: $('#hd7').value, i: $('#hi').value, f: $('#hf').value,
            dur: +$('#hu').value || 30 };                    // 30 min por defecto
  if (!h.i || !h.f || h.f <= h.i)
    return toast('La hora de finalización debe ser posterior a la de inicio', 1);
  H = H.filter(x => !(x.m == h.m && x.d == h.d));            // reemplaza el horario de ese día
  H.push(h); go('CU07'); toast('Horario guardado');
};

const av7 = () => {                                          // aviso de citas afectadas
  let m = S.m7 || 'MED-001',
      i = $('#xi').value.replace('T', ' '), f = $('#xf').value.replace('T', ' '),
      a = C.filter(c => vg(c) && ov(c, { m, i, f }));
  // si a.length > 0 muestra: «Hay N cita(s) afectada(s) en este rango: CIT-0008…»
  // si no, confirma que no hay citas vigentes afectadas
};
```

### CU08 · Consultar disponibilidad médica
**Actor:** Recepcionista. Filtros por especialidad, médico activo y fecha. La grilla muestra cada bloque como Disponible, Ocupado por cita o Bloqueado por indisponibilidad médica.

```js
// Bloques de la fecha según el horario del médico
const slots = (m, f) =>
  H.filter(h => h.m == m && h.d == DIAS[new Date(f + 'T12:00').getDay()])
   .flatMap(h => {
     let r = [], t = h.i;
     while (add(t, h.dur) <= h.f) { r.push(t); t = add(t, h.dur); }
     return r;
   });

// Estado de cada bloque
const sl = (m, f, t, ex) => {
  let x = X.find(x => x.m == m && f + ' ' + t >= x.i && f + ' ' + t < x.f);
  if (x) return ['Bloqueado', x.mot];                        // indisponibilidad médica
  let c = C.find(c => c.m == m && c.f == f && c.i == t && vg(c) && c.c != ex);
  return c ? ['Ocupado', c.c] : ['Disponible', ''];          // cita vigente o libre
};

// Grilla visual
const g = slots(f.m, f.d).map(t => {
  let [s, d] = sl(f.m, f.d, t);
  return `<div class="border rounded-lg p-2 text-center ${cl[s]}">
            <b>${t}–${add(t, 30)}</b><div>${s}${d ? ' · ' + d : ''}</div></div>`;
}).join('');
```

### CU09 · Programar cita médica
**Actor:** Recepcionista. Formulario paso a paso: código de cita y fecha de creación automáticos, paciente activo, especialidad, médico habilitado, fecha, bloque horario libre (la hora de fin se calcula sola), motivo opcional y estado inicial Programada.

```js
// Solo se ofrecen los bloques libres del médico en la fecha elegida
let fr = n.m && n.d
  ? slots(n.m, n.d).filter(t => sl(n.m, n.d, t)[0] == 'Disponible')
  : [];

const gc = () => {                                           // guardar cita
  let n = S.n9;
  if (!n.p || !n.e || !n.m || !n.d || !n.t)
    return toast('Complete paciente, especialidad, médico, fecha y horario', 1);
  C.push({ c: 'CIT-' + String(C.length + 1).padStart(4, '0'),   // código autogenerado
           p: n.p, m: n.m, sp: n.e, f: n.d, i: n.t,
           e: 'Programada', mo: n.mo, cr: TODAY });             // estado inicial y fecha de creación
  S.n9 = null; go('CU09'); toast('Cita programada correctamente');
};
```

---

## Ciclo II: Gestión y seguimiento de citas médicas

### CU10 · Reprogramar cita médica
**Actor:** Recepcionista. Buscador de citas vigentes y un panel comparativo con los *Datos anteriores* frente a la *Nueva programación* (nueva fecha, nueva hora u otro médico de la misma especialidad).

```js
// Solo citas Programadas o Confirmadas
const vg = c => c.e == 'Programada' || c.e == 'Confirmada';

// Médicos de la misma especialidad de la cita
M.filter(m => m.s == 'Activo' && m.esp.includes(s.sp))

// Bloques libres; la propia cita no cuenta como ocupada (4.º parámetro)
slots(n.m, n.f).filter(t => sl(n.m, n.f, t, s.c)[0] == 'Disponible')

const gr = () => {                                           // guardar cambios
  let s = C.find(c => c.c == S.s10), n = S.n10;
  if (!n.t) return toast('Seleccione la nueva hora', 1);
  s.rep = { f: s.f, i: s.i, m: s.m };                        // conserva los datos anteriores (se ven en CU15)
  Object.assign(s, { m: n.m, f: n.f, i: n.t });
  S.s10 = null; go('CU10'); toast('Cita reprogramada correctamente');
};
```

### CU11 · Cancelar cita médica
**Actor:** Recepcionista. Buscador de citas Programadas o Confirmadas, resumen de la cita y motivo de cancelación obligatorio. Al confirmar, el estado cambia a Cancelada y el horario queda libre.

```js
const cn = () => {
  let s = C.find(c => c.c == S.s11), mt = $('#mc').value.trim();
  if (!mt) return toast('El motivo de cancelación es obligatorio', 1);
  s.e = 'Cancelada'; s.can = mt;                             // al dejar de ser vigente, sl() libera el bloque
  S.s11 = null; go('CU11'); toast('Cita cancelada. El horario quedó libre');
};
```

### CU12 · Consultar agenda médica
**Actores:** Recepcionista y Médico. La Recepcionista filtra por médico y fecha. Si el rol activo es Médico, el filtro queda bloqueado a su propia identificación (RN18). La tabla se ordena por horario.

```js
const md = S.r == 'Médico',
      f  = S.f12 || (S.f12 = { m: '', d: TODAY });
if (md) f.m = 'MED-001';                                     // RN18: solo su propia agenda
// el selector de médico se deshabilita cuando md es verdadero

const srt = (a, b) => (a.f + a.i).localeCompare(b.f + b.i);  // orden por fecha y hora
C.filter(c => (!f.m || c.m == f.m) && (!f.d || c.f == f.d)).sort(srt).map(ag)
```

### CU13 · Registrar asistencia e inasistencia
**Actor:** Recepcionista. Lista de las citas del día con dos acciones distintas (RN15 y RN16). *Registrar asistencia* marca la presencia con la hora y no cambia el estado. *Registrar inasistencia* cambia el estado a Inasistencia solo si el horario ya terminó.

```js
const NOW = '10:15';                                         // hora simulada para la demostración

const as = c => {                                            // asistencia: guarda la hora, el estado no cambia
  C.find(x => x.c == c).asi = NOW;
  go('CU13'); toast('Asistencia registrada a las ' + NOW + '. El estado de la cita no cambia');
};

const ia = c => {                                            // inasistencia
  let x = C.find(y => y.c == c);
  if (add(x.i, 30) > NOW) return toast('RN16: el horario previsto aún no ha finalizado', 1);
  if (x.asi) return toast('El paciente ya registró asistencia', 1);
  x.e = 'Inasistencia'; go('CU13'); toast('Inasistencia registrada');
};
```

### CU14 · Registrar cita como atendida
**Actor:** Médico. Agenda del médico autenticado; las citas vigentes del día tienen el botón «Registrar como atendida», que pide confirmación antes de cambiar el estado a Atendida (RN16 y RN17).

```js
// Agenda de hoy del médico autenticado
C.filter(c => c.m == 'MED-001' && c.f == TODAY).sort(srt)

const at = c => mod('Confirmar atención',
  `<p>¿Confirma que la cita <b>${c}</b> de <b>${pn(C.find(x => x.c == c).p)}</b> fue atendida?</p>
   <button onclick="C.find(x => x.c == '${c}').e = 'Atendida'; go('CU14');
                    toast('Cita registrada como atendida')">Confirmar</button>`);
```

### CU17 · Consultar citas afectadas por indisponibilidad médica
**Actor:** Recepcionista. Tabla de alerta con las citas vigentes que se superponen con una ausencia, permiso o vacación del médico, con la indisponibilidad relacionada y botones directos a CU10 y CU11.

```js
// Una cita queda afectada si se solapa con el rango de la indisponibilidad del mismo médico
const ov = (c, x) =>
  c.m == x.m && c.f + ' ' + c.i < x.f && c.f + ' ' + add(c.i, 30) > x.i;

let a = C.filter(vg)
         .map(c => [c, X.find(x => ov(c, x))])               // cita + indisponibilidad relacionada
         .filter(([c, x]) => x);

// Botones de acción directa: abren CU10 o CU11 con la cita ya seleccionada
const pick = (k, c) => {
  S['s' + k] = c;
  let x = C.find(y => y.c == c);
  if (k == 10) S.n10 = { m: x.m, f: x.f, t: '' };
  go('CU' + k);
};
// «Ir a Reprogramar (CU10)» → pick(10, código)    «Ir a Cancelar (CU11)» → pick(11, código)
```

---

## Ciclo III: Historial y reportes administrativos

### CU15 · Consultar historial de citas
**Actores:** Administrador y Recepcionista. Filtros por código o paciente, médico, estado y fecha, con una tabla histórica y un detalle de cada cita: datos generales, fecha de creación, antecedentes de reprogramación (anterior frente a nuevo) e información de cancelación.

```js
const l15 = () => {
  let f = S.f15;
  return T([...], C.filter(c =>
      (!f.m || c.m == f.m) && (!f.e || c.e == f.e) && (!f.d || c.f == f.d) &&
      nz(c.c + pn(c.p)).includes(nz(f.q))                    // código o nombre del paciente
    ).sort(srt).map(...));
};

const dt15 = c => {                                          // modal de detalle
  let x = C.find(y => y.c == c);
  mod('Detalle de la cita ' + c, dl([...cd(x),
    ['Fecha de creación', x.cr],
    ['Motivo de consulta', x.mo || '—'],
    ['Antecedentes de reprogramación',
      x.rep ? `Anterior: ${x.rep.f} ${x.rep.i} · ${mn(x.rep.m)}<br>Nuevo: ${x.f} ${x.i} · ${mn(x.m)}`
            : 'Sin reprogramaciones'],
    ['Información de cancelación', x.can ? 'Motivo: ' + x.can : 'No aplica']]));
};
```

### CU16 · Generar reportes administrativos
**Actor:** Administrador. Generador con los 7 tipos de reporte. Los filtros cambian según el tipo elegido y la vista previa muestra contadores y una tabla.

| Reporte | Descripción | Filtros |
|---|---|---|
| REP01 | Citas por fecha | Fecha |
| REP02 | Citas por médico | Médico y rango de fechas |
| REP03 | Citas por especialidad | Especialidad y rango de fechas |
| REP04 | Citas por estado | Estado y rango de fechas |
| REP05 | Pacientes registrados | Ninguno |
| REP06 | Resumen de atenciones por período | Rango de fechas |
| REP07 | Agenda diaria por médico | Médico y fecha |

```js
// Cada reporte declara qué filtros necesita: f = fecha, r = rango, m = médico, e = especialidad, s = estado
const RP = {
  REP01: ['Citas por fecha',                    ['f']],
  REP02: ['Citas por médico',                   ['m', 'r']],
  REP03: ['Citas por especialidad',             ['e', 'r']],
  REP04: ['Citas por estado',                   ['s', 'r']],
  REP05: ['Pacientes registrados',              []],
  REP06: ['Resumen de atenciones por período',  ['r']],
  REP07: ['Agenda diaria por médico',           ['m', 'f']]
};

// Selección de citas según el tipo de reporte
const r = c => c.f >= q.d1 && c.f <= q.d2;
let cs = C.filter(c => ({
  REP01: c.f == q.f,
  REP02: r(c) && (!q.m || c.m == q.m),
  REP03: r(c) && (!q.e || c.sp == q.e),
  REP04: r(c) && (q.s == 'Todos' || c.e == q.s),
  REP06: r(c),
  REP07: c.f == q.f && (!q.m || c.m == q.m)
})[t]);

// REP06: totales y porcentajes de atendidas, canceladas e inasistencias
const ce = e => cs.filter(c => c.e == e).length,
      pc = e => n ? Math.round(ce(e) * 100 / n) : 0;


