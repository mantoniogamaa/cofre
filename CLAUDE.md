# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projeto

**Cofre** é um app de controle financeiro pessoal — despesas, receitas, lembretes via Google Calendar. Aplicação web single-page sem build system: todo o código está em um único arquivo `index.html` (~1300 linhas) com HTML, CSS e JavaScript inline.

## Desenvolvimento local

Não há etapa de build. Para rodar localmente com login funcional (Firebase Auth exige HTTPS ou localhost):

```bash
npx serve .
# acesse http://localhost:3000
```

Abrir `index.html` diretamente via `file://` não permite o login com Google.

## Arquitetura

### Tecnologias
- **Firebase Auth** — login via Google (popup + fallback redirect)
- **Firestore** — banco de dados em tempo real com listeners `onSnapshot`
- **Firestore offline cache** — `persistentLocalCache()` via IndexedDB; na 2ª visita os dados carregam do cache antes de sincronizar com o servidor
- **Google Calendar API** — criação/exclusão de eventos de lembrete via REST (token OAuth armazenado no `localStorage`)
- **Google Fonts (Inter)** — carregado de forma assíncrona para não bloquear o render

### Modelo de dados (Firestore)

Todas as coleções vivem sob `users/{uid}/`:

| Coleção | Conteúdo |
|---|---|
| `bills` | Despesas (recorrentes e avulsas) |
| `receitas` | Receitas (recorrentes e avulsas) |
| `paid` | Status de pago/recebido por mês (`{ano}-{mes}-{id}`) |
| `skipped` | Despesas recorrentes puladas em um mês específico |
| `bills_overrides` / `receitas_overrides` | Valor customizado de um lançamento para um mês específico |
| `bills_cal` / `receitas_cal` | IDs dos eventos criados no Google Calendar |

### Fluxo de dados

1. `onAuthStateChanged` → se logado, chama `loadData()`
2. `loadData()` registra 6 listeners `onSnapshot` (bills, receitas, paid, skipped, bills_overrides, receitas_overrides)
3. Cada listener chama `scheduleRender()` — debounce de 50ms que agrupa múltiplos disparos em um único `render()`
4. `render()` recalcula saldos e redesenha a lista do mês atual

### Padrões importantes

**`getVal(b)`** — sempre usar para obter o valor de um lançamento; respeita overrides mensais em vez de usar `b.value` diretamente.

**`sanitize(s)`** — obrigatório antes de inserir qualquer dado do usuário via `innerHTML`. Nunca concatenar `b.name`, `u.displayName` ou `u.email` diretamente em templates HTML.

**Gravação no Firestore** — o formulário fecha assim que o Firestore confirma. Operações do Google Calendar rodam em background (IIFE async) para não bloquear a UI.

**Event handlers** — todos expostos via `window.funcao = ...` por serem chamados em atributos `onclick` inline no HTML.

**`scheduleRender()`** — usar no lugar de `render()` dentro de listeners do Firestore para evitar renders redundantes em cascata.

### Categorias

Duas constantes hardcoded: `CAT_D` (despesas) e `CAT_R` (receitas). Adicionar novas categorias requer entrada em ambas as constantes e a classe CSS correspondente (`ci-nomecategoria`).
