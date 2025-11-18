# Documentação Frontend - Sistema de Solicitação de Acesso

## 🎯 Estratégia de Decisão

**IMPORTANTE:** Frontend é **opcional** e **diferencial**. Só comece após ter:
- ✅ Backend 100% funcional
- ✅ Todos os testes passando (cobertura ≥ 80%)
- ✅ Docker Compose funcionando
- ✅ Load balancing testado
- ✅ Documentação completa

**Quando começar o frontend:**
- ✅ Após concluir Fase 9 do ROADMAP
- ✅ Dia 7 ou superior
- ✅ Backend validado completamente

---

## 📋 Árvore de Decisão

```
┌─────────────────────────────────────────┐
│ Backend 100% completo e testado?        │
└─────────────────┬───────────────────────┘
                  │
         ┌────────┴────────┐
         │ SIM             │ NÃO → FOQUE NO BACKEND!
         ▼
┌─────────────────────────────────────────┐
│ Quanto tempo disponível para frontend?  │
└─────────────────┬───────────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
   0-4h         4-8h        8-12h+
     │            │            │
     ▼            ▼            ▼
  Não faça    HTML/JS     React+Vite
   (polish    Vanilla      (APRENDER)
   backend)  (FALLBACK)
```

---

## 🚀 Cenário Preferencial: React + Vite + TypeScript

**Use se tiver: 8-12 horas disponíveis**

### Objetivos de Aprendizado
- ⚛️ Fundamentos de React (Components, Props, State, Hooks)
- 🔄 React Router (Navegação)
- 📡 Integração com API REST (Axios)
- 🔐 Autenticação JWT
- 📝 Formulários e Validação
- 🎨 Estilização com Tailwind CSS

### Stack Tecnológica
```json
{
  "runtime": "Node.js 20+",
  "framework": "React 18",
  "build": "Vite 5",
  "language": "TypeScript",
  "styling": "Tailwind CSS",
  "http": "Axios",
  "routing": "React Router DOM 6",
  "forms": "React Hook Form",
  "ui": "Shadcn/ui (opcional)"
}
```

---

## 📦 Setup Inicial React

### Passo 1: Criar Projeto

```bash
# No diretório raiz do projeto
npm create vite@latest frontend -- --template react-ts

cd frontend

# Instalar dependências
npm install

# Dependências adicionais
npm install axios react-router-dom
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Passo 2: Configurar Tailwind

**tailwind.config.js:**
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

**src/index.css:**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

### Passo 3: Estrutura de Pastas

```
frontend/
├── public/
├── src/
│   ├── components/
│   │   ├── Layout.tsx
│   │   ├── LoginForm.tsx
│   │   ├── RequestForm.tsx
│   │   ├── RequestList.tsx
│   │   └── RequestCard.tsx
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   └── DashboardPage.tsx
│   ├── services/
│   │   └── api.ts
│   ├── contexts/
│   │   └── AuthContext.tsx
│   ├── types/
│   │   └── index.ts
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── Dockerfile
├── nginx.conf
├── package.json
└── tsconfig.json
```

---

## 🔧 Código Base React

### 1. Tipos TypeScript (src/types/index.ts)

```typescript
export interface User {
  email: string;
  name: string;
  department: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface AuthResponse {
  token: string;
  type: string;
  expiresIn: number;
  username: string;
}

export interface Module {
  id: number;
  name: string;
  description: string;
  active: boolean;
  allowedDepartments: string[];
  incompatibleModules: Module[];
}

export interface AccessRequest {
  id: number;
  protocol: string;
  modules: Module[];
  justification: string;
  urgent: boolean;
  status: 'ATIVO' | 'NEGADO' | 'CANCELADO';
  requestedAt: string;
  expiresAt?: string;
  denialReason?: string;
}

export interface CreateAccessRequestDto {
  moduleIds: number[];
  justification: string;
  urgent: boolean;
}

export interface PageResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalPages: number;
  totalElements: number;
}
```

### 2. API Service (src/services/api.ts)

```typescript
import axios from 'axios';
import type {
  LoginRequest,
  AuthResponse,
  AccessRequest,
  CreateAccessRequestDto,
  Module,
  PageResponse
} from '../types';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor para adicionar token em todas as requisições
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Interceptor para tratar erros globalmente
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// Auth
export const login = async (credentials: LoginRequest): Promise<AuthResponse> => {
  const { data } = await api.post<AuthResponse>('/auth/login', credentials);
  return data;
};

// Access Requests
export const createAccessRequest = async (
  request: CreateAccessRequestDto
): Promise<AccessRequest> => {
  const { data } = await api.post<AccessRequest>('/access-requests', request);
  return data;
};

export const getMyAccessRequests = async (
  params?: {
    search?: string;
    status?: string;
    page?: number;
    size?: number;
  }
): Promise<PageResponse<AccessRequest>> => {
  const { data } = await api.get<PageResponse<AccessRequest>>('/access-requests', { params });
  return data;
};

export const cancelAccessRequest = async (
  id: number,
  reason: string
): Promise<void> => {
  await api.patch(`/access-requests/${id}/cancel`, { cancellationReason: reason });
};

// Modules
export const getModules = async (): Promise<Module[]> => {
  const { data } = await api.get<Module[]>('/modules');
  return data;
};

export default api;
```

### 3. Context de Autenticação (src/contexts/AuthContext.tsx)

```typescript
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import { login as apiLogin } from '../services/api';
import type { LoginRequest, User } from '../types';

interface AuthContextType {
  user: User | null;
  token: string | null;
  login: (credentials: LoginRequest) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(null);

  useEffect(() => {
    // Carregar token do localStorage na inicialização
    const storedToken = localStorage.getItem('token');
    const storedUser = localStorage.getItem('user');

    if (storedToken && storedUser) {
      setToken(storedToken);
      setUser(JSON.parse(storedUser));
    }
  }, []);

  const login = async (credentials: LoginRequest) => {
    try {
      const response = await apiLogin(credentials);

      const userData: User = {
        email: credentials.email,
        name: response.username,
        department: '', // Pode vir da resposta se necessário
      };

      setToken(response.token);
      setUser(userData);

      localStorage.setItem('token', response.token);
      localStorage.setItem('user', JSON.stringify(userData));
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    }
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        token,
        login,
        logout,
        isAuthenticated: !!token,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

### 4. Página de Login (src/pages/LoginPage.tsx)

```typescript
import { useState, FormEvent } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      await login({ email, password });
      navigate('/dashboard');
    } catch (err: any) {
      setError(err.response?.data?.message || 'Erro ao fazer login');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-lg shadow-md w-full max-w-md">
        <h1 className="text-2xl font-bold text-center mb-6">
          Sistema de Acesso
        </h1>

        {error && (
          <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
            {error}
          </div>
        )}

        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Email
            </label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
              className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="seu.email@company.com"
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Senha
            </label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
              className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="••••••••"
            />
          </div>

          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:bg-blue-300 transition-colors"
          >
            {loading ? 'Entrando...' : 'Entrar'}
          </button>
        </form>

        <div className="mt-6 text-sm text-gray-600">
          <p className="font-semibold mb-2">Credenciais de teste:</p>
          <ul className="space-y-1">
            <li>• TI: ti.user@company.com</li>
            <li>• Financeiro: financeiro.user@company.com</li>
            <li>• RH: rh.user@company.com</li>
            <li>• Operações: operacoes.user@company.com</li>
            <li className="mt-1 text-xs">Senha: senha123</li>
          </ul>
        </div>
      </div>
    </div>
  );
}
```

### 5. Dashboard (src/pages/DashboardPage.tsx)

```typescript
import { useState, useEffect } from 'react';
import { useAuth } from '../contexts/AuthContext';
import { useNavigate } from 'react-router-dom';
import { getMyAccessRequests, getModules, createAccessRequest } from '../services/api';
import type { AccessRequest, Module, CreateAccessRequestDto } from '../types';

export default function DashboardPage() {
  const { user, logout } = useAuth();
  const navigate = useNavigate();

  const [requests, setRequests] = useState<AccessRequest[]>([]);
  const [modules, setModules] = useState<Module[]>([]);
  const [loading, setLoading] = useState(true);
  const [showForm, setShowForm] = useState(false);

  // Form state
  const [selectedModules, setSelectedModules] = useState<number[]>([]);
  const [justification, setJustification] = useState('');
  const [urgent, setUrgent] = useState(false);
  const [formError, setFormError] = useState('');
  const [formSuccess, setFormSuccess] = useState('');

  useEffect(() => {
    loadData();
  }, []);

  const loadData = async () => {
    try {
      setLoading(true);
      const [requestsData, modulesData] = await Promise.all([
        getMyAccessRequests({ page: 0, size: 10 }),
        getModules(),
      ]);
      setRequests(requestsData.content);
      setModules(modulesData);
    } catch (error) {
      console.error('Error loading data:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setFormError('');
    setFormSuccess('');

    if (selectedModules.length === 0 || selectedModules.length > 3) {
      setFormError('Selecione entre 1 e 3 módulos');
      return;
    }

    if (justification.length < 20 || justification.length > 500) {
      setFormError('Justificativa deve ter entre 20 e 500 caracteres');
      return;
    }

    try {
      const request: CreateAccessRequestDto = {
        moduleIds: selectedModules,
        justification,
        urgent,
      };

      await createAccessRequest(request);
      setFormSuccess('Solicitação criada com sucesso!');

      // Limpar formulário
      setSelectedModules([]);
      setJustification('');
      setUrgent(false);
      setShowForm(false);

      // Recarregar lista
      loadData();
    } catch (error: any) {
      setFormError(error.response?.data?.message || 'Erro ao criar solicitação');
    }
  };

  const handleModuleToggle = (moduleId: number) => {
    setSelectedModules((prev) => {
      if (prev.includes(moduleId)) {
        return prev.filter((id) => id !== moduleId);
      } else {
        if (prev.length >= 3) {
          setFormError('Máximo de 3 módulos permitidos');
          return prev;
        }
        setFormError('');
        return [...prev, moduleId];
      }
    });
  };

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'ATIVO':
        return 'bg-green-100 text-green-800';
      case 'NEGADO':
        return 'bg-red-100 text-red-800';
      case 'CANCELADO':
        return 'bg-gray-100 text-gray-800';
      default:
        return 'bg-blue-100 text-blue-800';
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-xl">Carregando...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100">
      {/* Header */}
      <header className="bg-white shadow">
        <div className="max-w-7xl mx-auto px-4 py-4 sm:px-6 lg:px-8 flex justify-between items-center">
          <div>
            <h1 className="text-2xl font-bold text-gray-900">Dashboard</h1>
            <p className="text-sm text-gray-600">Bem-vindo, {user?.name || user?.email}</p>
          </div>
          <button
            onClick={logout}
            className="bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700"
          >
            Sair
          </button>
        </div>
      </header>

      <main className="max-w-7xl mx-auto px-4 py-8 sm:px-6 lg:px-8">
        {/* Actions */}
        <div className="mb-6">
          <button
            onClick={() => setShowForm(!showForm)}
            className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700"
          >
            {showForm ? 'Cancelar' : '+ Nova Solicitação'}
          </button>
        </div>

        {/* Success Message */}
        {formSuccess && (
          <div className="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded mb-4">
            {formSuccess}
          </div>
        )}

        {/* Form */}
        {showForm && (
          <div className="bg-white p-6 rounded-lg shadow mb-6">
            <h2 className="text-xl font-bold mb-4">Nova Solicitação de Acesso</h2>

            {formError && (
              <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
                {formError}
              </div>
            )}

            <form onSubmit={handleSubmit} className="space-y-4">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">
                  Módulos (selecione 1-3)
                </label>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-2">
                  {modules.map((module) => (
                    <label
                      key={module.id}
                      className={`flex items-center p-3 border rounded cursor-pointer ${
                        selectedModules.includes(module.id)
                          ? 'border-blue-500 bg-blue-50'
                          : 'border-gray-300'
                      }`}
                    >
                      <input
                        type="checkbox"
                        checked={selectedModules.includes(module.id)}
                        onChange={() => handleModuleToggle(module.id)}
                        className="mr-2"
                      />
                      <div>
                        <div className="font-medium">{module.name}</div>
                        <div className="text-xs text-gray-500">{module.description}</div>
                      </div>
                    </label>
                  ))}
                </div>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Justificativa (20-500 caracteres)
                </label>
                <textarea
                  value={justification}
                  onChange={(e) => setJustification(e.target.value)}
                  required
                  minLength={20}
                  maxLength={500}
                  rows={4}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                  placeholder="Descreva detalhadamente o motivo da solicitação..."
                />
                <div className="text-xs text-gray-500 mt-1">
                  {justification.length}/500 caracteres
                </div>
              </div>

              <div>
                <label className="flex items-center">
                  <input
                    type="checkbox"
                    checked={urgent}
                    onChange={(e) => setUrgent(e.target.checked)}
                    className="mr-2"
                  />
                  <span className="text-sm font-medium text-gray-700">
                    Marcar como urgente
                  </span>
                </label>
              </div>

              <button
                type="submit"
                className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700"
              >
                Enviar Solicitação
              </button>
            </form>
          </div>
        )}

        {/* Requests List */}
        <div className="bg-white rounded-lg shadow">
          <div className="px-6 py-4 border-b">
            <h2 className="text-xl font-bold">Minhas Solicitações</h2>
          </div>

          <div className="divide-y">
            {requests.length === 0 ? (
              <div className="px-6 py-8 text-center text-gray-500">
                Nenhuma solicitação encontrada
              </div>
            ) : (
              requests.map((request) => (
                <div key={request.id} className="px-6 py-4 hover:bg-gray-50">
                  <div className="flex justify-between items-start mb-2">
                    <div>
                      <div className="font-mono text-sm text-gray-600">
                        {request.protocol}
                      </div>
                      <div className="text-lg font-medium">
                        {request.modules.map((m) => m.name).join(', ')}
                      </div>
                    </div>
                    <span
                      className={`px-3 py-1 rounded-full text-sm font-medium ${getStatusColor(
                        request.status
                      )}`}
                    >
                      {request.status}
                    </span>
                  </div>

                  <p className="text-sm text-gray-600 mb-2">
                    {request.justification}
                  </p>

                  <div className="flex items-center gap-4 text-xs text-gray-500">
                    <span>
                      Solicitado em:{' '}
                      {new Date(request.requestedAt).toLocaleDateString('pt-BR')}
                    </span>
                    {request.urgent && (
                      <span className="text-red-600 font-medium">🔴 URGENTE</span>
                    )}
                    {request.expiresAt && (
                      <span>
                        Expira em:{' '}
                        {new Date(request.expiresAt).toLocaleDateString('pt-BR')}
                      </span>
                    )}
                  </div>

                  {request.denialReason && (
                    <div className="mt-2 text-sm text-red-600 bg-red-50 p-2 rounded">
                      Motivo da negação: {request.denialReason}
                    </div>
                  )}
                </div>
              ))
            )}
          </div>
        </div>
      </main>
    </div>
  );
}
```

### 6. App Principal (src/App.tsx)

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider, useAuth } from './contexts/AuthContext';
import LoginPage from './pages/LoginPage';
import DashboardPage from './pages/DashboardPage';

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <>{children}</> : <Navigate to="/login" />;
}

function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route
            path="/dashboard"
            element={
              <ProtectedRoute>
                <DashboardPage />
              </ProtectedRoute>
            }
          />
          <Route path="/" element={<Navigate to="/dashboard" />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}

export default App;
```

### 7. Variáveis de Ambiente (.env)

```bash
VITE_API_URL=http://localhost/api
```

### 8. Package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

---

## 🐳 Docker para React

### Dockerfile

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci

# Copy source
COPY . .

# Build
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html

# Nginx config for SPA
RUN echo 'server { \
    listen 80; \
    location / { \
        root /usr/share/nginx/html; \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Adicionar ao docker-compose.yml

```yaml
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: access-control-frontend
    ports:
      - "3000:80"
    depends_on:
      - nginx
    networks:
      - app-network
```

---

## 🎨 Cenário Fallback: HTML/JS Vanilla + Bootstrap

**Use se tiver: 4-8 horas disponíveis**

### Estrutura Simples

```
frontend/
├── index.html          (Login)
├── dashboard.html      (Dashboard)
├── css/
│   └── style.css       (Estilos customizados)
├── js/
│   ├── app.js          (Lógica compartilhada)
│   ├── login.js        (Login)
│   └── dashboard.js    (Dashboard)
└── nginx.conf          (Servir arquivos estáticos)
```

### index.html (Login)

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login - Sistema de Acesso</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="css/style.css">
</head>
<body class="bg-light">
    <div class="container">
        <div class="row justify-content-center align-items-center min-vh-100">
            <div class="col-md-5">
                <div class="card shadow">
                    <div class="card-body p-5">
                        <h2 class="text-center mb-4">Sistema de Acesso</h2>

                        <div id="errorAlert" class="alert alert-danger d-none"></div>

                        <form id="loginForm">
                            <div class="mb-3">
                                <label for="email" class="form-label">Email</label>
                                <input type="email" class="form-control" id="email" required>
                            </div>

                            <div class="mb-3">
                                <label for="password" class="form-label">Senha</label>
                                <input type="password" class="form-control" id="password" required>
                            </div>

                            <button type="submit" class="btn btn-primary w-100">
                                <span id="loginText">Entrar</span>
                                <span id="loginSpinner" class="spinner-border spinner-border-sm d-none"></span>
                            </button>
                        </form>

                        <div class="mt-4 small text-muted">
                            <p class="fw-bold">Credenciais de teste:</p>
                            <ul class="list-unstyled">
                                <li>• TI: ti.user@company.com</li>
                                <li>• Financeiro: financeiro.user@company.com</li>
                                <li>• RH: rh.user@company.com</li>
                                <li>• Operações: operacoes.user@company.com</li>
                                <li class="mt-2">Senha: <strong>senha123</strong></li>
                            </ul>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script src="js/app.js"></script>
    <script src="js/login.js"></script>
</body>
</html>
```

### js/app.js (Lógica compartilhada)

```javascript
const API_URL = 'http://localhost/api';

// Função para fazer requisições
async function apiRequest(endpoint, options = {}) {
    const token = localStorage.getItem('token');

    const headers = {
        'Content-Type': 'application/json',
        ...options.headers
    };

    if (token) {
        headers['Authorization'] = `Bearer ${token}`;
    }

    try {
        const response = await fetch(`${API_URL}${endpoint}`, {
            ...options,
            headers
        });

        if (response.status === 401) {
            localStorage.removeItem('token');
            window.location.href = 'index.html';
            throw new Error('Não autorizado');
        }

        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || 'Erro na requisição');
        }

        return await response.json();
    } catch (error) {
        console.error('API Error:', error);
        throw error;
    }
}

// Verificar se está autenticado
function checkAuth() {
    const token = localStorage.getItem('token');
    if (!token && window.location.pathname !== '/index.html') {
        window.location.href = 'index.html';
    }
}

// Logout
function logout() {
    localStorage.removeItem('token');
    window.location.href = 'index.html';
}
```

### js/login.js

```javascript
document.getElementById('loginForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const email = document.getElementById('email').value;
    const password = document.getElementById('password').value;
    const errorAlert = document.getElementById('errorAlert');
    const loginText = document.getElementById('loginText');
    const loginSpinner = document.getElementById('loginSpinner');

    // Mostrar loading
    loginText.classList.add('d-none');
    loginSpinner.classList.remove('d-none');
    errorAlert.classList.add('d-none');

    try {
        const response = await apiRequest('/auth/login', {
            method: 'POST',
            body: JSON.stringify({ email, password })
        });

        localStorage.setItem('token', response.token);
        window.location.href = 'dashboard.html';
    } catch (error) {
        errorAlert.textContent = error.message || 'Erro ao fazer login';
        errorAlert.classList.remove('d-none');
    } finally {
        loginText.classList.remove('d-none');
        loginSpinner.classList.add('d-none');
    }
});
```

### dashboard.html

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard - Sistema de Acesso</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <!-- Navbar -->
    <nav class="navbar navbar-dark bg-primary">
        <div class="container-fluid">
            <span class="navbar-brand mb-0 h1">Dashboard</span>
            <button class="btn btn-outline-light" onclick="logout()">Sair</button>
        </div>
    </nav>

    <div class="container mt-4">
        <!-- Botão Nova Solicitação -->
        <button class="btn btn-primary mb-4" data-bs-toggle="modal" data-bs-target="#newRequestModal">
            + Nova Solicitação
        </button>

        <div id="successAlert" class="alert alert-success d-none"></div>
        <div id="errorAlert" class="alert alert-danger d-none"></div>

        <!-- Lista de Solicitações -->
        <div class="card">
            <div class="card-header">
                <h5>Minhas Solicitações</h5>
            </div>
            <div class="card-body">
                <div id="requestsList">
                    <div class="text-center py-5">
                        <div class="spinner-border text-primary"></div>
                        <p class="mt-2">Carregando...</p>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- Modal Nova Solicitação -->
    <div class="modal fade" id="newRequestModal" tabindex="-1">
        <div class="modal-dialog modal-lg">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">Nova Solicitação</h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <form id="newRequestForm">
                    <div class="modal-body">
                        <div class="mb-3">
                            <label class="form-label">Módulos (1-3)</label>
                            <div id="modulesList"></div>
                        </div>

                        <div class="mb-3">
                            <label for="justification" class="form-label">Justificativa (20-500 caracteres)</label>
                            <textarea class="form-control" id="justification" rows="4" required minlength="20" maxlength="500"></textarea>
                            <div class="form-text"><span id="charCount">0</span>/500</div>
                        </div>

                        <div class="form-check">
                            <input class="form-check-input" type="checkbox" id="urgent">
                            <label class="form-check-label" for="urgent">Urgente</label>
                        </div>
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancelar</button>
                        <button type="submit" class="btn btn-primary">Enviar</button>
                    </div>
                </form>
            </div>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script src="js/app.js"></script>
    <script src="js/dashboard.js"></script>
</body>
</html>
```

### js/dashboard.js

```javascript
checkAuth();

let modules = [];
let requests = [];

// Carregar dados
async function loadData() {
    try {
        [modules, requestsData] = await Promise.all([
            apiRequest('/modules'),
            apiRequest('/access-requests?page=0&size=10')
        ]);

        requests = requestsData.content;

        renderModules();
        renderRequests();
    } catch (error) {
        showError('Erro ao carregar dados: ' + error.message);
    }
}

// Renderizar módulos
function renderModules() {
    const container = document.getElementById('modulesList');
    container.innerHTML = modules.map(module => `
        <div class="form-check">
            <input class="form-check-input module-checkbox" type="checkbox" value="${module.id}" id="module${module.id}">
            <label class="form-check-label" for="module${module.id}">
                <strong>${module.name}</strong>
                <br>
                <small class="text-muted">${module.description}</small>
            </label>
        </div>
    `).join('');
}

// Renderizar solicitações
function renderRequests() {
    const container = document.getElementById('requestsList');

    if (requests.length === 0) {
        container.innerHTML = '<p class="text-center text-muted py-4">Nenhuma solicitação encontrada</p>';
        return;
    }

    container.innerHTML = requests.map(request => {
        const statusClass = {
            'ATIVO': 'success',
            'NEGADO': 'danger',
            'CANCELADO': 'secondary'
        }[request.status];

        return `
            <div class="border-bottom py-3">
                <div class="d-flex justify-content-between align-items-start">
                    <div>
                        <small class="text-muted">${request.protocol}</small>
                        <h6>${request.modules.map(m => m.name).join(', ')}</h6>
                    </div>
                    <span class="badge bg-${statusClass}">${request.status}</span>
                </div>
                <p class="small mb-1">${request.justification}</p>
                <small class="text-muted">
                    ${new Date(request.requestedAt).toLocaleDateString('pt-BR')}
                    ${request.urgent ? '🔴 URGENTE' : ''}
                </small>
                ${request.denialReason ? `<div class="alert alert-danger mt-2 py-1 small">Motivo: ${request.denialReason}</div>` : ''}
            </div>
        `;
    }).join('');
}

// Contador de caracteres
document.getElementById('justification').addEventListener('input', (e) => {
    document.getElementById('charCount').textContent = e.target.value.length;
});

// Limitar seleção de módulos
document.addEventListener('change', (e) => {
    if (e.target.classList.contains('module-checkbox')) {
        const checked = document.querySelectorAll('.module-checkbox:checked');
        if (checked.length > 3) {
            e.target.checked = false;
            showError('Máximo de 3 módulos permitidos');
        }
    }
});

// Enviar nova solicitação
document.getElementById('newRequestForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const selectedModules = Array.from(document.querySelectorAll('.module-checkbox:checked'))
        .map(cb => parseInt(cb.value));
    const justification = document.getElementById('justification').value;
    const urgent = document.getElementById('urgent').checked;

    if (selectedModules.length === 0 || selectedModules.length > 3) {
        showError('Selecione entre 1 e 3 módulos');
        return;
    }

    try {
        await apiRequest('/access-requests', {
            method: 'POST',
            body: JSON.stringify({
                moduleIds: selectedModules,
                justification,
                urgent
            })
        });

        showSuccess('Solicitação criada com sucesso!');

        // Fechar modal e recarregar
        bootstrap.Modal.getInstance(document.getElementById('newRequestModal')).hide();
        e.target.reset();
        loadData();
    } catch (error) {
        showError(error.message);
    }
});

function showSuccess(message) {
    const alert = document.getElementById('successAlert');
    alert.textContent = message;
    alert.classList.remove('d-none');
    setTimeout(() => alert.classList.add('d-none'), 5000);
}

function showError(message) {
    const alert = document.getElementById('errorAlert');
    alert.textContent = message;
    alert.classList.remove('d-none');
    setTimeout(() => alert.classList.add('d-none'), 5000);
}

// Carregar ao iniciar
loadData();
```

### css/style.css

```css
body {
    min-height: 100vh;
}

.min-vh-100 {
    min-height: 100vh;
}

.form-check-label {
    cursor: pointer;
}

.module-checkbox {
    cursor: pointer;
}
```

---

## 📝 Checklist de Implementação

### React + Vite (8-12h)
- [ ] Setup do projeto com Vite
- [ ] Instalar dependências (axios, react-router-dom, tailwind)
- [ ] Configurar Tailwind CSS
- [ ] Criar estrutura de pastas
- [ ] Implementar tipos TypeScript
- [ ] Criar API service com interceptors
- [ ] Criar AuthContext
- [ ] Implementar página de login
- [ ] Implementar dashboard
- [ ] Testar integração com backend
- [ ] Criar Dockerfile
- [ ] Adicionar ao docker-compose
- [ ] Testar build de produção
- [ ] Documentar no README

### HTML/JS Vanilla (4-8h)
- [ ] Criar estrutura de arquivos
- [ ] Implementar login.html
- [ ] Implementar dashboard.html
- [ ] Criar app.js (lógica compartilhada)
- [ ] Criar login.js
- [ ] Criar dashboard.js
- [ ] Adicionar estilos customizados
- [ ] Testar integração com backend
- [ ] Configurar nginx para servir arquivos
- [ ] Adicionar ao docker-compose
- [ ] Testar em produção
- [ ] Documentar no README

---

## 📖 Atualizar README.md

Adicionar seção de frontend:

```markdown
## Frontend (Diferencial)

### Tecnologias
- React 18 + TypeScript
- Vite 5
- Tailwind CSS
- React Router DOM
- Axios

### Executar Frontend

**Desenvolvimento:**
```bash
cd frontend
npm install
npm run dev
```

**Com Docker:**
```bash
docker-compose up frontend
```

**Acessar:** http://localhost:3000

### Funcionalidades
- ✅ Login com JWT
- ✅ Dashboard de solicitações
- ✅ Criar nova solicitação (1-3 módulos)
- ✅ Visualizar histórico
- ✅ Validações em tempo real
- ✅ Tratamento de erros
- ✅ Loading states
- ✅ Design responsivo
```

---

## 🎯 Resumo Final

**Decisão:**
1. ✅ **Backend 100%** → Pré-requisito obrigatório
2. ⏰ **Tempo disponível ≥ 8h** → React + Vite (aprender)
3. ⏰ **Tempo disponível 4-8h** → HTML/JS Vanilla (rápido)
4. ⏰ **Tempo < 4h** → Não faça frontend, foque em polish do backend

**Prioridade:** Backend > Testes > Docker > Documentação > Frontend

Este documento está pronto para ser usado como guia quando você chegar na fase de frontend! 🚀
