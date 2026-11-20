# jwt-app – Sistema de Gestão de Biblioteca (Front-end)

Aplicação web desenvolvida em **Next.js + React + TypeScript + Axios** integrada à `jwt-api`, com renovação automática de sessão via **Interceptor de Refresh Token** e fila de requisições concorrentes.

---

## 1. Como instalar e rodar

Certifique-se de que a `jwt-api` esteja rodando na porta `3000`, e então execute:

```bash
cd frontend-jwt-app
npm install
copy .envex .env
npm run dev
```

A aplicação front-end subirá em `http://localhost:3001`.

---

## 2. Variáveis de Ambiente (`.env`)

Baseadas no arquivo `.envex`:

```env
NEXT_PUBLIC_API_URL=http://localhost:3000/v1
```

---

## 3. Usuários de Teste (criados pelo Seed da API)

- **Administrador**: `admin@biblioteca.com` / Senha: `123456`
- **Atendente**: `atendente@biblioteca.com` / Senha: `123456`

---

## 4. Decisões Técnicas e Segurança no Armazenamento de Tokens

### 4.1 Armazenamento em `localStorage` e Riscos (Seção 4.3)
Os tokens (`accessToken` e `refreshToken`) e os dados básicos do usuário logado são armazenados no `localStorage` do navegador (com sincronização do cookie de rota para o middleware do Next.js):
- **Vantagens**: Facilita a leitura direta pelo interceptor do Axios (`lib/api.ts`) em chamadas client-side (`Authorization: Bearer <token>`) e simplifica a substituição imediata do par de tokens a cada rotação.
- **Riscos da escolha**: O `localStorage` é acessível via JavaScript no navegador. Caso a aplicação sofra uma vulnerabilidade de **XSS (Cross-Site Scripting)**, um script malicioso injetado na página poderia ler os tokens do `localStorage`.
- **Mitigações adotadas**:
  1. **Access Token curto (15 minutos)**: reduz a janela de exposição caso um token de acesso vaze.
  2. **Rotação de Refresh Token + Detecção de Reuso na API**: cada `refreshToken` só pode ser usado uma única vez. Se um invasor roubar e tentar reutilizar um `refreshToken` já rotacionado, o back-end detecta a reutilização e **revoga todas as sessões do usuário imediatamente**.
  3. **Sanitização nativa do React**: evita injeção direta de HTML/scripts nos componentes.

### 4.2 Interceptor de Renovação Automática e Fila de Concorrência (`lib/api.ts`)
- Ao receber **HTTP 401**, o interceptor verifica a flag `isRefreshing`:
  - Se `isRefreshing === false`, inicia **uma única chamada** para `POST /v1/auth/refresh`.
  - Se outras requisições falharem com `401` enquanto o refresh está em andamento, elas entram na fila `failedQueue` (Promises) e aguardam o novo token sem disparar novas requisições de refresh.
  - Assim que o novo par `{ accessToken, refreshToken }` retorna, todas as requisições da fila são refeitas automaticamente de forma transparente para o usuário.
