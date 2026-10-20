# Forge — Gerador de projetos web

> Ferramenta web que gera boilerplates Next.js ou React+Vite configurados com sua stack (Tailwind, shadcn, banco + ORM, auth, deploy) e devolve o projeto pronto em um `.zip` para download.

[**Repositório**](https://github.com/nulldevelop/project-init)

<!-- Adicione uma captura de tela real do sistema, se disponível. -->

## Sobre o projeto

Montar um projeto do zero com as convenções da equipe (package manager, linter, auth, ORM, deploy) repete sempre as mesmas decisões e gera atrito. O **Forge** resolve isso: no navegador você seleciona as opções de stack e, em segundos, baixa um `.zip` com o projeto já estruturado, instalável com um único comando.

O público-alvo são desenvolvedores da equipe (Null Development) que precisam iniciar novos projetos web seguindo o padrão documentado no `AGENT.md` — e qualquer pessoa que queira um boilerplate coeso sem configurar tudo manualmente.

O diferencial técnico é o gerador stateless no servidor: um único endpoint `POST /api/generate` valida a configuração, seleciona apenas os módulos de template relevantes (7 módulos: base, styling, database, auth, deploy, extras, editor) e monta o ZIP com JSZip — sem banco de dados, autenticação ou persistência.

## Funcionalidades

- [x] Gerador de projetos **Next.js** (App Router) ou **React + Vite**.
- [x] Suporte a TypeScript ou JavaScript, com pnpm, npm ou yarn no projeto gerado.
- [x] Tailwind CSS 4 opcional, com shadcn/ui, Magic UI e Kibo UI (os dois últimos adicionam shadcn automaticamente).
- [x] Banco MySQL ou PostgreSQL com Prisma ou Drizzle (ou nenhum), incluindo `.env.local.example` e cliente de conexão.
- [x] Autenticação Better Auth ou NextAuth v5 (NextAuth desabilitado para projetos Vite), com middleware de rotas protegidas.
- [x] Templates de deploy: Vercel, Hostinger (PM2 + Nginx) e IIS/iisnode (IIS desabilitado para Vite).
- [x] Extras opcionais: Docker (multi-stage + docker-compose com banco), GitHub Actions (self-hosted), Stripe e Resend.
- [x] Configuração de editor: Zed ou VS Code (settings + extensões Biome/Tailwind/Prisma/Symbols).
- [x] Validação do nome do projeto no cliente e no servidor (`/^[a-z0-9-]+$/`), com feedback em português.
- [x] Toast de sucesso com comando de instalação copiável (`cd <nome> && pnpm install`).
- [ ] Testes automatizados, rate limiting no endpoint de geração e CI do próprio app.

> Não há TODOs/FIXMEs no código; a lista acima reflete lacunas reais verificadas (sem framework de teste, sem CI, sem proteção contra abuso no `/api/generate`).

## Tecnologias

- **Frontend:** Next.js 16.2.9 (App Router, React Compiler habilitado), React 19.2.4, Tailwind CSS 4.3.0, lucide-react, componentes UI próprios (tiles de opção, cards de seção, ícone de chama), fontes Geist/Geist Mono.
- **Backend:** Next.js Route Handler único (`POST /api/generate`) — sem servidor separado, sem banco, sem auth nesta aplicação.
- **Banco de dados:** nenhum nesta aplicação (apenas nos templates gerados: MySQL 8 ou PostgreSQL 16 via Docker, com Prisma 7 ou Drizzle).
- **Infraestrutura:** nenhuma configurada para o próprio Forge (sem Docker/CI no repositório); os templates gerados incluem `vercel.json`, PM2 + Nginx, `web.config`/iisnode, Dockerfile e GitHub Actions.
- **Testes:** nenhum framework configurado.
- **Qualidade de código:** Biome 2.2.0 (lint + format), TypeScript 5 estrito.

## Arquitetura

Aplicação stateless de página única: formulário client (`src/app/page.tsx`) → `POST /api/generate` → gerador → ZIP no browser.

```
src/app/page.tsx                 # estado ProjectConfig + regras entre campos + toast
src/app/api/generate/route.ts    # valida nome → generateProject() → application/zip
src/lib/generator/
├── index.ts      # monta o FileMap e zipia sob pasta raiz com o nome do projeto
├── types.ts      # ProjectConfig e uniões de opções
└── templates/    # base, styling, database, auth, deploy, extras, editor
```

Regras entre campos no formulário (evitam stacks inválidas):

- `next-auth` e deploy IIS desabilitados quando o framework é React + Vite.
- Escolher banco força ORM (padrão Prisma); banco "nenhum" limpa o ORM.
- Desligar Tailwind remove as libs de UI; Magic UI/Kibo UI adicionam shadcn automaticamente.

Cada módulo de template emite apenas os arquivos relevantes para a configuração (ex.: `deploy.ts` só escreve `vercel.json` se Vercel estiver selecionado), o que mantém o ZIP enxuto e o gerador fácil de estender.

## Pré-requisitos

- Node.js 20+ para desenvolver o Forge (artefatos gerados assumem Node 22 — base do Dockerfile e do workflow de CI template)
- npm ou pnpm (o padrão da equipe é pnpm; o repositório hoje possui `package-lock.json`)
- Nenhum serviço externo, chave de API ou banco de dados é necessário para rodar o Forge

## Como executar localmente

### 1. Clone o repositório

```bash
git clone https://github.com/nulldevelop/project-init.git
cd project-init
```

### 2. Instale as dependências

```bash
pnpm install
```

> O repositório também funciona com `npm install` (há `package-lock.json` commitado), mas o padrão da equipe é pnpm.

### 3. Configure as variáveis de ambiente

Este projeto não utiliza variáveis de ambiente — nenhuma configuração é necessária.

> As variáveis listadas nos templates gerados (`DATABASE_URL`, `BETTER_AUTH_SECRET`, `AUTH_SECRET`, `STRIPE_SECRET_KEY`, `RESEND_API_KEY`, etc.) pertencem aos projetos baixados, não ao Forge.

### 4. Prepare o banco de dados

Não aplicável — o Forge não possui banco de dados.

### 5. Inicie a aplicação

```bash
pnpm dev
```

A aplicação sobe em `http://localhost:3000`. Configure a stack no formulário e clique em gerar para baixar o `.zip`.

> Comandos úteis: `pnpm build`, `pnpm start`, `pnpm lint` (Biome check), `pnpm format`.

## Testes

Ainda não há testes automatizados no projeto (nenhum framework instalado, nenhum script de teste). As áreas que mais se beneficiariam primeiro são: `src/lib/generator/` (cada template emitiria os arquivos esperados para configurações representative — matriz de combinações) e a validação da rota `/api/generate`.

## Decisões técnicas

- **Stateless sem banco/auth**: o gerador não precisa persistir nada — a configuração inteira viaja no corpo do POST, o que simplifica deploy e elimina superfície de ataque.
- **Templates como funções TypeScript puras** (`ProjectConfig` → `Record<path, string>`): fáceis de testar unitariamente e de compor (só os módulos pedidos entram no ZIP).
- **Regras entre campos no client + validação no server**: UX evita combinações inválidas; o server revalida o nome do projeto por segurança.
- **Convenções da equipe embutidas nos templates**: Biome no lugar de ESLint/Prettier, colocation `_components`/`_actions`/`_data-access`/`_schemas`, aliases de path, Better Auth preferido ao NextAuth — tudo alinhado ao `AGENT.md`.
- **React Compiler habilitado no app do Forge** (`reactCompiler: true`), antecipando o padrão que se quer nos projetos gerados.

### Inconsistências conhecidas entre o app e os templates

- O `package.json` gerado ancora `next: ^15.2.0` enquanto o próprio Forge roda Next 16.2.9; Biome gerado é `^1.9.4` vs 2.2.0 aqui.
- O schema Prisma gerado emite o `datasource` sem `url = env("DATABASE_URL")` — precisa ser corrigido manualmente ou no template.
- O `AGENT.md` exige pnpm, mas o lockfile commitado atual é o do npm.

## Limitações conhecidas

- Sem rate limiting ou captcha no `/api/generate` — geração é barata, mas o endpoint é público.
- Sem testes e sem CI para o próprio Forge.
- Templates gerados podem divergir das versões mais novas usadas neste repositório (ver inconsistências acima).
- UI majoritariamente em português, mas o `lang` do HTML está como `en`.
- Sem licença definida.

## Contribuição

Projeto interno da Null Development; contribuições externas não são aceitas no momento. Para estender o gerador, adicione ou edite módulos em `src/lib/generator/templates/` e opções em `src/lib/generator/types.ts`, seguindo o `AGENT.md` (padrões de stack e convenções). Commits no formato Conventional Commits; lint/format com Biome.

## Licença

Sem licença definida. Todos os direitos reservados.

## Autor

**Pedro Andrade (nulldevelop / devpedroA)**

- GitHub: https://github.com/nulldevelop
