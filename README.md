# Tutorial: Sistema de Autenticación con React y Supabase

Este tutorial te guiará paso a paso en la creación de un sistema completo de autenticación utilizando React y Supabase. Al final, tendrás una aplicación funcional con registro de usuarios, inicio de sesión y un panel de control.

## 📋 Índice

1. [Requisitos Previos](#requisitos-previos)
2. [Configuración Inicial](#configuración-inicial)
3. [Instalación de Dependencias](#instalación-de-dependencias)
4. [Estructura del Proyecto](#estructura-del-proyecto)
5. [Implementación de Componentes](#implementación-de-componentes)
6. [Configuración de Supabase](#configuración-de-supabase)

## ✨ Lo que Vamos a Construir

- Sistema de registro con validación
- Inicio de sesión seguro
- Panel de usuario protegido
- Base de datos con perfiles de usuario
- Interfaz moderna con PrimeReact

## 📋 Requisitos Previos

- Node.js (versión 14 o superior)
- Una cuenta en [Supabase](https://supabase.com)
- npm o yarn

## 🛠️ Configuración del Proyecto

### 1. Crear un Proyecto en Supabase

1. Ve a [Supabase](https://supabase.com) y crea una nueva cuenta o inicia sesión
2. Crea un nuevo proyecto
3. Guarda la URL del proyecto y la anon key (las necesitaremos después)

## Configuración de Supabase (Último Paso)

Este es el último paso después de tener toda la aplicación React funcionando. Necesitarás configurar la base de datos en Supabase para almacenar los perfiles de usuario.

### 1. Crear la Tabla de Perfiles

En el Editor SQL de Supabase, ejecuta este código:

```sql
-- Crear la tabla de perfiles
create table profiles (
  id uuid references auth.users on delete cascade primary key,
  display_name text,
  email text,
  phone text,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null,
  updated_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Habilitar Row Level Security
alter table profiles enable row level security;

-- Políticas de seguridad

-- 1. Permitir a los usuarios ver su propio perfil
create policy "Users can view own profile"
  on profiles for select
  using (auth.uid() = id);

-- 2. Permitir a los usuarios crear su perfil (solo si no existe)
create policy "Users can create profile once"
  on profiles
  as permissive
  for insert
  to anon
  with check (
    NOT EXISTS (
      SELECT 1 FROM profiles 
      WHERE profiles.id = auth.uid()
    )
  );

-- 3. Permitir a los usuarios actualizar su perfil
create policy "Users can update own profile"
  on profiles for update
  using (auth.uid() = id)
  with check (auth.uid() = id);

-- 4. Trigger para actualizar el timestamp
create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger profiles_updated_at
  before update on profiles
  for each row
  execute function update_updated_at();
```

### 2. Explicación de las Políticas de Seguridad

1. **Vista del Perfil**: Los usuarios solo pueden ver su propio perfil
2. **Creación del Perfil**: 
   - Solo se permite crear un perfil una vez
   - Verifica que no exista un perfil previo para ese usuario
3. **Actualización del Perfil**: Solo pueden actualizar su propio perfil
4. **Actualización Automática**: El trigger actualiza la fecha de modificación

## Paso 1: Crear el Proyecto

```bash
# Crear nuevo proyecto con Vite
npm create vite@latest login-supabase -- --template react

# Entrar al directorio
cd login-supabase

# Instalar dependencias base
npm install
```

## Paso 2: Instalar Dependencias

```bash
# Supabase - Para autenticación y base de datos
npm install @supabase/supabase-js

# React Router - Para navegación
npm install react-router-dom

# PrimeReact - Para UI
npm install primereact primeicons
npm install primeflex
```

## Paso 3: Configurar Variables de Entorno

1. Crea un archivo `.env` en la raíz del proyecto:

```env
VITE_SUPABASE_URL=tu_url_de_supabase
VITE_SUPABASE_ANON_KEY=tu_anon_key
```

2. Crea el archivo de configuración de Supabase (`src/config/supabase.js`):

```javascript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
```

## Paso 4: Crear la Estructura de Carpetas

```bash
mkdir src/components
mkdir src/config
```

Estructura resultante:
```
src/
├── components/     # Componentes de React
├── config/        # Configuraciones (Supabase)
├── App.jsx        # Componente principal
└── main.jsx       # Punto de entrada
```

## Paso 5: Configurar el Enrutador

Actualiza `src/main.jsx`:

```javascript
import React from 'react'
import ReactDOM from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App'

// Estilos de PrimeReact
import 'primereact/resources/themes/lara-light-indigo/theme.css'
import 'primereact/resources/primereact.min.css'
import 'primeicons/primeicons.css'
import 'primeflex/primeflex.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
)
```

## Paso 6: Crear los Componentes

### 6.1 Componente de Registro (`src/components/Register.jsx`)

```javascript
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { supabase } from '../config/supabase';
import { InputText } from 'primereact/inputtext';
import { Password } from 'primereact/password';
import { Button } from 'primereact/button';
import { Message } from 'primereact/message';

export default function Register() {
  const navigate = useNavigate();
  const [formData, setFormData] = useState({
    displayName: '',
    email: '',
    password: '',
    phone: ''
  });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      // 1. Registrar el usuario en Auth
      const { data: { user }, error: signUpError } = await supabase.auth.signUp({
        email: formData.email,
        password: formData.password,
        options: {
          data: {
            display_name: formData.displayName,
            phone: formData.phone
          }
        }
      });

      if (signUpError) throw signUpError;

      // 2. Crear el perfil del usuario
      const { error: profileError } = await supabase
        .from('profiles')
        .insert([{
          id: user.id,
          display_name: formData.displayName,
          email: formData.email,
          phone: formData.phone
        }]);

      if (profileError) throw profileError;

      // 3. Redirigir al login
      navigate('/login', { 
        state: { 
          message: 'Registro exitoso. Por favor, verifica tu correo electrónico.' 
        }
      });
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 flex items-center justify-center p-4">
      <div className="max-w-md w-full bg-white rounded-lg shadow-md p-8">
        <h2 className="text-2xl font-bold text-center text-gray-800 mb-8">
          Registro de Usuario
        </h2>

        {error && (
          <Message severity="error" text={error} className="w-full mb-4" />
        )}

        <form onSubmit={handleSubmit} className="space-y-6">
          <div className="flex flex-col gap-2">
            <label htmlFor="displayName" className="text-sm font-medium text-gray-700">
              Nombre para mostrar
            </label>
            <InputText
              id="displayName"
              name="displayName"
              value={formData.displayName}
              onChange={handleChange}
              required
              className="w-full"
            />
          </div>

          <div className="flex flex-col gap-2">
            <label htmlFor="email" className="text-sm font-medium text-gray-700">
              Correo electrónico
            </label>
            <InputText
              id="email"
              name="email"
              type="email"
              value={formData.email}
              onChange={handleChange}
              required
              className="w-full"
            />
          </div>

          <div className="flex flex-col gap-2">
            <label htmlFor="password" className="text-sm font-medium text-gray-700">
              Contraseña
            </label>
            <Password
              id="password"
              name="password"
              value={formData.password}
              onChange={handleChange}
              toggleMask
              required
              className="w-full"
              feedback={true}
            />
          </div>

          <div className="flex flex-col gap-2">
            <label htmlFor="phone" className="text-sm font-medium text-gray-700">
              Teléfono
            </label>
            <InputText
              id="phone"
              name="phone"
              type="tel"
              value={formData.phone}
              onChange={handleChange}
              required
              className="w-full"
            />
          </div>

          <Button
            type="submit"
            label={loading ? 'Registrando...' : 'Registrarse'}
            disabled={loading}
            className="w-full"
          />

          <div className="text-center mt-4">
            <button
              type="button"
              onClick={() => navigate('/login')}
              className="text-blue-600 hover:text-blue-800"
            >
              ¿Ya tienes cuenta? Inicia sesión
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}
```

### 6.2 Componente de Login (`src/components/Login.jsx`)

```javascript
import { useState, useRef } from 'react';
import { useNavigate } from 'react-router-dom';
import { InputText } from 'primereact/inputtext';
import { Password } from 'primereact/password';
import { Button } from 'primereact/button';
import { Card } from 'primereact/card';
import { Toast } from 'primereact/toast';
import { supabase } from '../config/supabase';

export default function Login() {
  const navigate = useNavigate();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const toast = useRef(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);

    try {
      const { error } = await supabase.auth.signInWithPassword({
        email,
        password,
      });

      if (error) throw error;

      // Éxito - la redirección la maneja App.jsx
    } catch (error) {
      toast.current.show({
        severity: 'error',
        summary: 'Error',
        detail: error.message,
        life: 3000,
      });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex align-items-center justify-content-center min-h-screen bg-gray-100">
      <Toast ref={toast} />
      <Card className="w-full max-w-30rem mx-3">
        <h2 className="text-center text-2xl font-bold mb-4">Iniciar Sesión</h2>
        <form onSubmit={handleSubmit} className="flex flex-column gap-2">
          <div className="flex flex-column gap-2">
            <label htmlFor="email">Correo electrónico</label>
            <InputText
              id="email"
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
              className="w-full"
            />
          </div>

          <div className="flex flex-column gap-2">
            <label htmlFor="password">Contraseña</label>
            <Password
              id="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              toggleMask
              required
              className="w-full"
              feedback={false}
            />
          </div>

          <Button
            type="submit"
            label="Ingresar"
            className="w-full"
            loading={loading}
          />
          
          <div className="text-center mt-4">
            <button
              type="button"
              onClick={() => navigate('/register')}
              className="text-blue-600 hover:text-blue-800"
            >
              ¿No tienes cuenta? Regístrate aquí
            </button>
          </div>
        </form>
      </Card>
    </div>
  );
}
```

### 6.3 Componente Dashboard (`src/components/Dashboard.jsx`)

```javascript
import { useState, useEffect } from 'react';
import { supabase } from '../config/supabase';
import { Button } from 'primereact/button';
import { Message } from 'primereact/message';

function Dashboard({ user, onLogout }) {
  const [profile, setProfile] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    getProfile();
  }, [user]);

  async function getProfile() {
    try {
      setLoading(true);
      
      const { data, error } = await supabase
        .from('profiles')
        .select('*')
        .eq('id', user.id)
        .single();

      if (error) throw error;
      setProfile(data);
    } catch (error) {
      setError(error.message);
    } finally {
      setLoading(false);
    }
  }

  const handleLogout = async () => {
    try {
      const { error } = await supabase.auth.signOut();
      if (error) throw error;
      if (onLogout) onLogout();
    } catch (error) {
      setError(error.message);
    }
  };

  if (loading) {
    return (
      <div className="flex justify-content-center align-items-center min-h-screen">
        <i className="pi pi-spin pi-spinner text-4xl"></i>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100 p-4">
      <div className="max-w-4xl mx-auto bg-white rounded-lg shadow-md p-6">
        <div className="flex justify-content-between align-items-center mb-6">
          <h1 className="text-3xl font-bold">Panel de Control</h1>
          <Button
            icon="pi pi-sign-out"
            label="Cerrar Sesión"
            severity="danger"
            onClick={handleLogout}
          />
        </div>

        {error && (
          <Message severity="error" text={error} className="w-full mb-4" />
        )}

        {profile && (
          <div className="grid">
            <div className="col-12 md:col-6">
              <div className="p-4 border-round bg-gray-50">
                <h3 className="text-xl mb-4">Información del Perfil</h3>
                <div className="flex flex-column gap-4">
                  <div>
                    <label className="font-bold block mb-1">Nombre:</label>
                    <span>{profile.display_name}</span>
                  </div>
                  <div>
                    <label className="font-bold block mb-1">Email:</label>
                    <span>{profile.email}</span>
                  </div>
                  <div>
                    <label className="font-bold block mb-1">Teléfono:</label>
                    <span>{profile.phone}</span>
                  </div>
                  <div>
                    <label className="font-bold block mb-1">Miembro desde:</label>
                    <span>
                      {new Date(profile.created_at).toLocaleDateString()}
                    </span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

export default Dashboard;
```

## Paso 7: Configurar las Rutas

Actualiza `src/App.jsx` con las rutas protegidas:

```javascript
import { useState, useEffect } from 'react'
import { Routes, Route, Navigate, useNavigate } from 'react-router-dom'
import { supabase } from './config/supabase'
import Login from './components/Login'
import Register from './components/Register'
import Dashboard from './components/Dashboard'

function App() {
  const [user, setUser] = useState(null)
  const navigate = useNavigate()

  useEffect(() => {
    // Verificar sesión actual
    supabase.auth.getSession().then(({ data: { session } }) => {
      setUser(session?.user ?? null)
    })

    // Escuchar cambios en la autenticación
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      setUser(session?.user ?? null)
    })

    return () => subscription.unsubscribe()
  }, [])

  // Componente para proteger rutas
  const ProtectedRoute = ({ children }) => {
    if (!user) return <Navigate to="/login" replace />
    return children
  }

  return (
    <Routes>
      <Route path="/login" element={
        user ? <Navigate to="/dashboard" replace /> : <Login />
      } />
      <Route path="/register" element={
        user ? <Navigate to="/dashboard" replace /> : <Register />
      } />
      <Route path="/dashboard" element={
        <ProtectedRoute>
          <Dashboard user={user} />
        </ProtectedRoute>
      } />
      <Route path="/" element={<Navigate to={user ? "/dashboard" : "/login"} replace />} />
    </Routes>
  )
}

export default App
```

## Paso 8: Ejecutar la Aplicación

```bash
npm run dev
```

La aplicación estará disponible en `http://localhost:5173`

## Paso 9: Configurar Supabase

Ahora puedes proceder con la configuración de Supabase que se detalla al final de este documento.

## 📚 Recursos Adicionales

- [Documentación de Supabase](https://supabase.com/docs)
- [Documentación de React Router](https://reactrouter.com)
- [Documentación de PrimeReact](https://primereact.org)

## 📝 Notas Importantes

1. Asegúrate de ejecutar primero el SQL en Supabase antes de intentar registrar usuarios.
2. La política de seguridad `Users can create profile once` evita que un usuario cree múltiples perfiles.
3. El sistema incluye validación de contraseñas y campos requeridos.
4. Los usuarios necesitarán verificar su correo electrónico para activar su cuenta.

## 🧳 Próximos Pasos Sugeridos

1. Agregar recuperación de contraseña
2. Implementar edición de perfil
3. Agregar avatar de usuario
4. Mejorar la validación de formularios
5. Agregar más información al perfil

## 📁 Estructura del Proyecto

```
src/
├── components/
│   ├── Login.jsx       # Componente de inicio de sesión
│   ├── Register.jsx    # Componente de registro
│   └── Dashboard.jsx   # Panel de control del usuario
├── config/
│   └── supabase.js     # Configuración de Supabase
└── App.jsx             # Componente principal y rutas
```

## 💻 Implementación

### 1. Configuración de Supabase (`src/config/supabase.js`)

```javascript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
```

### 2. Componente de Registro (`src/components/Register.jsx`)

El componente de registro maneja:
- Formulario de registro con validación
- Creación de usuario en Supabase Auth
- Creación de perfil en la tabla `profiles`
- Redirección al login después del registro exitoso

### 3. Componente de Login (`src/components/Login.jsx`)

El componente de login maneja:
- Formulario de inicio de sesión
- Autenticación con Supabase
- Redirección al dashboard después del login exitoso
- Link para registro de nuevos usuarios

### 4. Dashboard (`src/components/Dashboard.jsx`)

El dashboard muestra:
- Información del perfil del usuario
- Opción para cerrar sesión
- Datos como nombre, email y teléfono

### 5. Gestión de Rutas (`src/App.jsx`)

El componente App maneja:
- Rutas protegidas
- Estado de autenticación
- Redirecciones basadas en el estado del usuario

## 🔒 Seguridad

- Las contraseñas son manejadas de forma segura por Supabase
- Row Level Security (RLS) asegura que los usuarios solo puedan:
  - Ver su propio perfil
  - Crear su perfil al registrarse
  - Actualizar su propia información

## 🎨 Estilos y UI

El proyecto utiliza PrimeReact para:
- Componentes de formulario
- Mensajes de error y éxito
- Diseño responsivo
- Temas y estilos consistentes

## 📚 Recursos Adicionales

- [Documentación de Supabase](https://supabase.com/docs)
- [Documentación de React Router](https://reactrouter.com)
- [Documentación de PrimeReact](https://primereact.org)

## 🤝 Contribuir

1. Haz un Fork del proyecto
2. Crea una rama para tu característica (`git checkout -b feature/AmazingFeature`)
3. Haz commit de tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push a la rama (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

## 📝 Licencia

Este proyecto está bajo la Licencia MIT. Ver el archivo `LICENSE` para más detalles.

```sql
-- Crear trigger para actualizar automáticamente updated_at
create or replace function public.handle_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

-- Aplicar el trigger a la tabla profiles
create trigger on_profiles_updated
  before update on profiles
  for each row
  execute procedure handle_updated_at();
```

## Instrucciones de Uso

1. Ve al panel de control de Supabase
2. Haz clic en "SQL Editor" en el menú de la izquierda
3. Copia y pega cada bloque de código SQL en orden
4. Ejecuta cada bloque haciendo clic en "RUN" o presionando Ctrl+Enter

## Verificación

Para verificar que todo se ha creado correctamente, puedes ejecutar:

```sql
-- Verificar que la tabla existe
select * from profiles;

-- Verificar las políticas de seguridad
select * from pg_policies where tablename = 'profiles';
```
#
