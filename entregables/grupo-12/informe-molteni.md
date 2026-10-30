# Informe individual — Grupo 12

**Integrante:** Baltazar Molteni · **Fecha:** jueves 29/10/2026
**Destinatario:** gerente de operaciones / comité de crisis
**Organización:** Cooperativa Canal 2 América (caso ficticio inspirado en la
cooperativa de cable e internet de América, partido de Rivadavia; los datos son
inventados para el TP). Da televisión por cable, internet por fibra y wifi, y tiene el
canal local de noticias. Atiende a unos 8.000 abonados. Usa tres sistemas:

- **Oficina virtual (PHP):** los abonados ven y pagan facturas, piden altas
  (subiendo DNI y comprobante de domicilio) y hacen reclamos.
- **API de la app (Node.js):** la usan la app de abonados y la de los técnicos, que
  cargan instalaciones y reparaciones desde la calle.
- **Gestión y facturación (Python):** sistema interno que arma la facturación mensual
  y los débitos automáticos.

---

## 1. Situación y riesgo

El 21/10 se publicaron dos fallas graves en programas sobre los que corren nuestros
sistemas. En el peor caso, las dos permiten que alguien de afuera tome control del
servidor sin usuario ni contraseña. No son errores de nuestro código: alcanza con usar
la versión vieja del programa.

| Sistema | Falla | ¿Se usa en ataques reales? | ¿Está en Internet? | Riesgo |
|---|---|---|---|---|
| Oficina virtual (PHP) | PHP2Shell | **Sí.** CISA la sumó el 26/10 a su lista de fallas explotadas | Sí | **Alto** |
| API de la app (Node.js) | Node2Shell | No confirmado al 27/10, solo demostraciones | Sí | **Medio-alto** |
| Gestión y facturación (Python) | Misma familia que Node2Shell (librería Brotli) | No confirmado | No, es interno | **Medio** |

**Qué está en juego:**

- **Oficina virtual:** nombre, DNI, domicilio y teléfono de 8.000 abonados, más los
  datos de los que pagan por débito automático. Una filtración nos expone ante la
  Agencia de Acceso a la Información Pública (Ley 25.326) y en un pueblo chico la
  noticia corre rápido: la confianza en la cooperativa es lo más difícil de recuperar.
- **API de la app:** si se cae, los técnicos no reciben las órdenes de trabajo y se
  atrasan instalaciones y reparaciones de internet y cable.
- **Gestión y facturación:** no se alcanza desde Internet. Para llegar, alguien tendría
  que haber entrado antes por la oficina virtual o la API. Pero es el que más duele si
  cae: sin él no sale la facturación del mes.

**Lo que más me preocupa no es la falla en sí.** El registro de actividad del 27/10
muestra a la oficina virtual y a la API ejecutando comandos que en su trabajo normal
nunca ejecutan, y después al sistema interno haciendo lo mismo (detalle en el punto
3). Si fue un ataque, actualizar cierra la puerta pero no saca a quien ya entró. Por
eso el plan tiene dos frentes en paralelo: cerrar la puerta y revisar si alguien pasó.

**Decisiones que necesito del comité hoy:**

1. Autorizar el cambio de emergencia en la oficina virtual (menos de 2 h).
2. Autorizar que, si se confirma el compromiso, se desconecte la oficina virtual aunque
   los abonados no puedan pagar online por unas horas (siguen pudiendo pagar en la
   sede y por los medios habituales).
3. Designar quién habla con abonados y medios, para que haya una sola voz.

---

## 2. Plan de acción priorizado

| # | Acción | Sistema | Plazo | Dueño | Reversión |
|---|---|---|---|---|---|
| 1 | Apagar la opción de PHP que usa el ataque (`session.upload_progress`) y **comprobar en el servidor funcionando** que quedó apagada | Oficina virtual | Hoy, menos de 2 h | Encargado de Sistemas | Volver a prenderla y reiniciar (minutos) |
| 2 | Copiar los registros a otro servidor antes de tocar nada más y empezar la revisión del punto 3 | Los tres | Hoy, en paralelo con el 1 | Encargado de Sistemas + proveedor externo de seguridad | No aplica (solo se copia) |
| 3 | Si la revisión confirma acceso: aislar de la red la oficina virtual y la API y poner una página de mantenimiento | Oficina virtual y API | Apenas se confirme | Gerente decide, Sistemas ejecuta | Volver a conectarlos |
| 4 | Actualizar PHP a 8.3.36, guardando antes una copia de la versión actual | Oficina virtual | Menos de 24 h | Encargado de Sistemas | Volver a la copia guardada de 8.3.35 |
| 5 | Mientras no se actualice Node: que el proxy rechace datos comprimidos con Brotli en las rutas que no los usan | API de la app | Menos de 24 h | Jefe de Red | Sacar la regla del proxy |
| 6 | Actualizar Node.js a 22.23.4 en ventana de mantenimiento, de noche, cuando no hay técnicos en la calle | API de la app | Menos de 72 h | Desarrollador (proveedor de la app) | Volver a 22.23.3 desde la copia guardada |
| 7 | Cambiar contraseñas y claves que usan la oficina virtual y la API (base de abonados, correo, pasarela de pagos) | Oficina virtual y API | Después del 2, antes de 72 h | Encargado de Sistemas | No aplica |
| 8 | Actualizar Brotli a 1.2.1, probando antes con una facturación de prueba y lejos del cierre del mes | Gestión y facturación | Próxima ventana, menos de 7 días | Desarrollador | Volver a 1.0.9 |
| 9 | Comunicar a personal, abonados y, si corresponde, a la autoridad de datos personales | Todos | Ver abajo | Gerente + asesor legal + Canal 2 (prensa) | No aplica |

**Por qué este orden.** Las tres fallas tienen el mismo puntaje de gravedad (9.2 sobre
10), así que ese número no sirve para decidir. Usé tres preguntas: ¿ya se está usando
para atacar?, ¿está en Internet?, ¿qué perdemos si cae? La oficina virtual responde
"sí" a las tres y por eso va primero, con una medida que se aplica en minutos y se
puede deshacer. La API está en Internet pero sin ataques confirmados: le ponemos una
barrera en el proxy y la actualizamos de noche, sin frenar a los técnicos. Facturación
es interna y va última, salvo que la revisión diga otra cosa.

**Por qué el paso 1 insiste en "comprobar".** En el laboratorio del grupo vimos que el
archivo de configuración puede tener el cambio y aun así el servidor no leerlo; el
panel igual mostraba todo "sano" porque solo revisa que el servicio esté prendido
([serra-06](capturas/serra-06-selinux-montajes.txt)). Una medida no se informa como
aplicada hasta verla funcionando.

**Por qué hay un proveedor externo.** Una cooperativa de este tamaño tiene una o dos
personas en sistemas. Si hay que revisar registros, aplicar cambios y atender
reclamos al mismo tiempo, no alcanzan. Conviene tener contratado de antemano a quién
llamar.

**Qué cambiaría el orden:**

- Si Node2Shell entra en la lista de CISA o aparece un ataque público, la API pasa a
  menos de 24 h, igual que la oficina virtual.
- Si la revisión confirma que alguien entró, el paso 3 deja de ser condicional y
  facturación sube al mismo nivel que la oficina virtual, porque el registro muestra
  actividad rara también ahí.
- Si falta poco para el cierre de facturación, el paso 8 se pasa para después del
  cierre, con el sistema vigilado mientras tanto.

**Comunicación (paso 9).** Tenemos una ventaja que pocas empresas tienen: nuestro propio
canal y nuestras redes. Hay que usarla con cuidado.

- **Personal y técnicos, hoy:** que puede haber cortes cortos de la oficina virtual y
  la app, y a quién avisar si ven algo raro. Que no respondan consultas de abonados
  con versiones propias.
- **Abonados, cuando sepamos si se vieron datos y antes de 72 h:** un aviso corto en
  redes, en la oficina virtual y en el canal, con qué pasó en términos simples, qué
  hicimos, qué datos podrían estar involucrados y qué tienen que hacer. Lo más
  importante: avisar que **la cooperativa nunca pide datos de tarjeta ni claves por
  teléfono o WhatsApp**, porque después de una filtración lo primero que llega son las
  estafas a nombre de la empresa.
- **Mientras dure la oficina virtual caída:** recordar en redes y en el canal que se
  puede pagar en la sede y por los medios de siempre, sin recargo.
- **Autoridad de datos personales:** si se confirma acceso a datos, lo decide el
  asesor legal.
- **Regla:** no decir "no pasó nada" hasta tener la revisión terminada. Avisar tarde
  genera desconfianza, pero desmentir después es peor. Y nunca dar detalles técnicos
  (versiones, sistemas) que le sirvan a un atacante.

---

## 3. Indicadores de compromiso

**Lo que confirmaría que entraron:**

- Que la oficina virtual o la API lancen una consola de comandos o programas de
  descarga. El registro del 27/10 lo muestra: la oficina virtual corriendo `id`
  (09:42) y `wget` (09:44), y la API corriendo `curl` (09:43), las dos descargas hacia
  direcciones externas desconocidas.
- Que el firewall muestre que esas descargas **salieron y tuvieron respuesta**.
- Archivos nuevos en las carpetas de la oficina virtual, usuarios nuevos o tareas
  programadas que nadie pidió.
- Consultas o descargas de la base de abonados fuera de lo normal, sobre todo de los
  datos de débito automático.
- Abonados que reportan llamados o mensajes falsos a nombre de la cooperativa con
  datos que solo teníamos nosotros.

**Lo que lo descartaría:**

- Que esos comandos los haya lanzado un proceso nuestro documentado (por ejemplo, un
  chequeo de monitoreo).
- Que el firewall muestre que no salió ninguna conexión hacia esas direcciones.
- Que las tres sesiones en el servidor de base de datos de ese día tengan dueño: una
  persona de sistemas, una tarea anotada y un origen que no sea la oficina virtual ni
  la API.

**Señal de alerta adicional.** El orden de los hechos es oficina virtual, API y recién
después el sistema interno (`uname -a` a las 09:45). Ese recorrido, de lo público a lo
interno, es el que haría alguien que entra y empieza a moverse. Una sola línea no
alcanza para afirmarlo, pero alcanza para revisar facturación ahora y no la semana que
viene.

**Lo que no podemos saber con lo que registramos hoy:**

- **Quién fue y desde dónde:** el registro no guarda usuario ni dirección de origen.
- **Si se llevaron datos:** registra qué programas corren, no qué información sale.
- **Si los comandos funcionaron:** no queda anotado el resultado.
- **Antes y después:** son 9 minutos de un solo día; no hay con qué comparar.

Para cerrar estas preguntas hay que cruzar con los registros del proxy, del firewall y
de la base de datos. Si esos registros no existen o se borran rápido, hay que decirlo
en el informe al comité y no completar el hueco con suposiciones.

---

## 4. Transferencia: OpenSSH "regreSSHion" (CVE-2024-6387)

Elegí una falla real y distinta a las del TP, que para un proveedor de internet es
muy relevante: casi todos los servidores y muchos equipos de red se administran por
SSH, y en el registro aparecen sesiones SSH en el servidor de base de datos. Es una
falla en el servidor SSH de Linux (OpenSSH 8.5p1 a 9.7p1) que permite ejecutar código
como administrador sin contraseña, aprovechando una "carrera" en el manejo del tiempo
de espera del login. Se corrige en OpenSSH 9.8p1. Sobre CVE-2026-115560 (la que sugiere
la consigna), lo único que dice el material es que se corrige en el próximo bugfix
de PHP y que hasta entonces hay que mantener la medida del paso 1; como no encontré
más datos verificables, preferí aplicar el método a un caso donde sí puedo comprobarlos.

Aplicando las mismas tres preguntas:

1. **¿Lo tenemos?** Revisar la versión real de OpenSSH en cada servidor y en los
   equipos de red que lo usen. Ojo con las distribuciones que corrigen sin cambiar el
   número de versión: igual que con Brotli dentro de Node, el número que muestra el
   programa puede engañar y hay que mirar el aviso de la distribución.
2. **¿Se está usando?** Es una falla grave en teoría pero difícil en la práctica: el
   ataque necesita muchos intentos durante horas. Habría que consultar la lista de
   CISA y la probabilidad de explotación (EPSS) antes de fijar plazos.
3. **¿Está en Internet?** Es la pregunta que más pesa. En un proveedor chico es común
   dejar SSH abierto para entrar desde afuera a resolver una urgencia. Si solo acepta
   conexiones desde la red de administración, entra en la ventana normal; si está
   abierto a Internet, va en menos de 24 h.

**Qué cambia en el plan respecto a PHP2Shell:**

- **La mitigación es distinta y tiene costo:** existe un ajuste de configuración
  (`LoginGraceTime 0`) que corta el ataque, pero puede dejar a sistemas sin poder
  entrar si alguien satura las conexiones. Antes de aplicarlo hay que decidir qué
  preferimos arriesgar.
- **La medida más barata es de red:** limitar quién puede conectarse por SSH (solo
  desde la red interna o una VPN). Eso protege contra esta falla y contra las que
  vengan.
- **La detección cambia:** acá la señal no es "un servicio que lanza una consola",
  sino muchísimos intentos de conexión fallidos desde la misma dirección durante horas.
- **El impacto es mayor:** el atacante entra directamente como administrador, así que
  los controles que nos protegieron en el laboratorio (usuario sin permisos, disco de
  solo lectura) no aplican. Y si llega a los equipos de red, puede cortar internet y
  cable a todo el pueblo. La prioridad sube aunque la explotación sea más difícil.

---

## 5. Lecciones y procesos

- **Inventario.** Necesitamos saber qué versión real corre en cada servidor, incluido
  lo que viene "adentro" de otros programas. La lista de componentes de la API no
  mostraba Brotli porque viene dentro de Node.js.
- **Cambios.** Un cambio está hecho cuando se comprueba funcionando, no cuando se
  guarda el archivo. Y antes de actualizar, la versión anterior tiene que estar
  guardada: en el laboratorio volver atrás tardó 17 segundos porque estaba a mano.
- **Detección.** Alertar cuando un servicio web lanza comandos, y guardar los
  registros con usuario, origen y al menos 30 días de historia en otro servidor.
- **Comunicación.** Tener escrito de antemano un mensaje base para abonados, quién lo
  aprueba y quién es el único vocero. En una crisis no hay tiempo para redactar desde
  cero ni para discutir quién habla.
- **Roles y decisiones.** Definir antes de la próxima crisis quién puede decidir
  apagar la oficina virtual o la app, para que no dependa de juntar al consejo de
  administración. Tener contratado un proveedor de seguridad para cuando el equipo
  propio no alcance. Y que el comité reciba cada 2 horas un resumen corto: qué sabemos,
  qué hicimos y qué falta decidir.
