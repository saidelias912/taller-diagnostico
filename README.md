# Taller Diag (MVP)

App para talleres mecánicos: el mecánico carga el **código OBD-II** y/o los **síntomas**, el sistema devuelve un **ranking de fallas probables** con **pruebas para confirmar**, una IA lo explica en lenguaje simple y todo queda en el **historial del vehículo**. Cuando el mecánico confirma la falla real, el sistema aprende (de forma anónima y global).

- **App**: Expo + React Native (celular y web con el mismo código)
- **Backend**: Vercel Functions (`api/`), motor de reglas + llamada al LLM
- **Base de datos, login y aislamiento por taller**: Supabase (Postgres + Auth + RLS)

---

## 1. Base de datos (Supabase)

1. Creá un proyecto en https://supabase.com.
2. Andá a **SQL Editor → New query**, pegá **todo** el contenido de `supabase/schema.sql` y ejecutá (**Run**).
3. Nueva query, pegá `supabase/seed.sql` y ejecutá (carga la base de conocimiento inicial: 25 fallas, 23 códigos, 13 síntomas y pruebas).
4. **Project Settings → API**: copiá la **Project URL** y la **anon public key**.
5. (Recomendado para probar rápido) **Authentication → Providers → Email**: desactivá *Confirm email*. Si lo dejás activo, cada usuario nuevo tiene que confirmar su email antes de entrar.

## 2. Configurar variables

```bash
cp .env.example .env
# completá EXPO_PUBLIC_SUPABASE_URL y EXPO_PUBLIC_SUPABASE_ANON_KEY
npm install
```

## 3. Backend en Vercel (necesario para diagnosticar)

1. Subí esta carpeta a un repositorio de GitHub e importalo en https://vercel.com (framework: **Other**; el comando de build ya está en `vercel.json`).
2. En **Settings → Environment Variables** cargá (ver `.env.example`):
   - `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY` (se usan al compilar la web)
   - `SUPABASE_URL`, `SUPABASE_ANON_KEY` (las usa la función `api/diagnose`)
   - `ANTHROPIC_API_KEY` (clave de la API de IA; si falta, el sistema igual funciona con una explicación automática simple)
   - opcionales: `LLM_MODEL` (por defecto `claude-haiku-5-5`), `DAILY_LLM_LIMIT` (por defecto 100 llamadas por taller por día)
3. Deploy. La web queda en `https://tu-proyecto.vercel.app`, la API en `/api/diagnose`.

> La **service_role key de Supabase no se usa en ningún lado**. La API actúa siempre con el token del usuario, así que las reglas de seguridad (RLS) también valen para el servidor.

## 4. Probar en local

```bash
# En .env poné EXPO_PUBLIC_API_URL=https://tu-proyecto.vercel.app  (tu deploy de Vercel)
npx expo start --web      # navegador
npx expo start            # celular: escaneá el QR con Expo Go
```

## 5. Convertirte en administrador global

1. Registrate en la app con tu email.
2. En el SQL Editor ejecutá (con tu email):
   ```sql
   insert into public.platform_admins (user_id)
   select id from auth.users where email = 'TU_EMAIL@ejemplo.com';
   ```
3. Cerrá sesión y volvé a entrar: aparece la pestaña **Admin** (métricas, baja de talleres y usuarios). El admin **no puede leer** clientes, vehículos ni diagnósticos de ningún taller: solo ve conteos y estados.

## 6. Cómo probar el aislamiento entre talleres

1. Creá dos cuentas distintas y un taller para cada una (en dos navegadores o ventana de incógnito).
2. Cargá vehículos y diagnósticos en el taller A: el taller B no tiene que ver nada.
3. Para probar el aprendizaje con un solo taller: en `get_learning` (schema.sql) el mínimo de talleres es `2`. Mientras probás podés pasar `min_workshops = 1` cambiando el valor por defecto de esa función.

---

## Cómo decide el sistema

`api/_lib/engine.ts` (motor puro, con tests: `npm run test:engine`):

- Cada código OBD-II y cada síntoma reparte probabilidad entre las fallas candidatas según sus **pesos** (`kb_dtc_fault`, `kb_symptom_fault`).
- Las evidencias se combinan multiplicando (tipo Naive Bayes, con suavizado) y se normaliza a 100 %.
- **Aprendizaje**: cada falla confirmada suma una confirmación a (código, marca, falla). Esas confirmaciones se suman a los pesos iniciales. Solo cuentan cuando vienen de **al menos 2 talleres distintos**.
- **Privacidad**: lo que viaja a la base global es solo código + marca + falla + un hash del taller (para contar talleres distintos). Nada de patente, cliente, modelo exacto ni notas.
- La **IA solo redacta la explicación**; el ranking lo decide el motor. Si la IA falla o se pasó el tope diario, se muestra una explicación automática.

## Estructura

```
app/                  Pantallas (Expo Router)
  login, onboarding, blocked
  (tabs)/             Inicio, Vehículos, Taller (equipo), Admin
  vehicle/            Alta y ficha con historial
  diagnose/           Asistente de 3 pasos y resultado + confirmación
src/                  Tema, componentes, sesión, cliente de Supabase y de la API
api/diagnose.ts       Endpoint de diagnóstico (valida usuario, corre motor, llama a la IA, guarda)
api/_lib/             engine.ts (motor), llm.ts (IA), http.ts (CORS/cliente), engine.test.ts
supabase/schema.sql   Tablas, funciones, RLS y panel admin
supabase/seed.sql     Base de conocimiento inicial
```

## Límites conocidos del MVP

- Un usuario pertenece a **un solo taller**.
- El código OBD-II se **escribe a mano** (la lectura por Bluetooth ELM327 es una fase posterior).
- El aprendizaje usa (código + marca); los síntomas todavía no aprenden.
- La base inicial es chica y está pensada para validar el flujo: revisala con un mecánico antes de usarla en serio.
- El resultado es una **sugerencia**: el mecánico debe confirmarla con pruebas.
- Los identificadores de la app (`com.tuempresa.tallerdiag`) en `app.json` hay que cambiarlos antes de publicar en tiendas.
