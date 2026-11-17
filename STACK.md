# STACK.md

## Objetivo

Registrar as tecnologias efetivamente utilizadas no MyLibrary. As versões exatas devem ser atualizadas a partir das dependências instaladas e do arquivo de lock do projeto. Não inventar versões nem atualizar dependências sem necessidade.

## Runtime e aplicação

- Node.js: versão LTS instalada no ambiente.
- TypeScript: versão instalada no projeto.
- Next.js: versão instalada no projeto.
- React: versão compatível instalada pelo Next.js.

## Interface

- Tailwind CSS: versão instalada no projeto.
- Componentes React reutilizáveis.
- Identidade visual clássica e aconchegante, com tokens semânticos para cores, tipografia, superfícies e estados.

## Dados e persistência

- PostgreSQL: banco de dados relacional.
- Prisma ORM: versão instalada no projeto.
- Migrações Prisma para versionar alterações no esquema.

## Autenticação

- Auth.js: versão e adaptador compatíveis com a versão do Next.js instalada.
- Cadastro e login do MVP com e-mail e senha; login pelo Google não faz parte do MVP.
- Senhas de contas locais devem ser armazenadas usando hash seguro.
- Sessões e autorização devem ser validadas no servidor.

## Arquivos e leitura

- PDF: visualizador compatível com navegador.
- EPUB: biblioteca de leitura EPUB compatível com React.
- TXT: leitura com codificação UTF-8.
- Cada livro terá apenas um arquivo principal no MVP.
- O limite de upload será configurável, com valor padrão inicial de 50 MB.
- Armazenamento: adaptador local durante o desenvolvimento, por trás de uma abstração que permita migração futura para armazenamento de objetos em produção. O provedor de produção ainda não está definido.

A biblioteca escolhida para EPUB e as bibliotecas de leitura devem ser definidas e registradas com suas versões reais durante a implementação. Não assumir provedor de armazenamento em produção antes de sua definição.

## Qualidade

- ESLint: versão instalada.
- TypeScript: verificação de tipos.
- Testes unitários: framework a definir na configuração inicial.
- Testes ponta a ponta: ferramenta a definir conforme necessidade.

## Decisões técnicas ainda pendentes

- Versões efetivas das dependências, conforme `package.json` e arquivo de lock.
- Biblioteca de leitura EPUB e bibliotecas de leitura para os formatos suportados.
- Provedor e configuração de armazenamento de objetos para produção.
- Frameworks de testes unitários e ponta a ponta.
- Estratégia de hash de senha compatível com a solução de autenticação escolhida.

## Ferramentas de desenvolvimento

- Visual Studio Code.
- Git e GitHub.
- GitHub Copilot integrado ao Visual Studio Code.
- npm como gerenciador de pacotes, salvo decisão documentada em contrário.

## Comandos

Registrar aqui os comandos reais definidos no `package.json`. Como referência, a aplicação deverá oferecer comandos equivalentes a:

```bash
npm install
npm run dev
npm run lint
npm run typecheck
npm run test
npm run build
```

Não manter comandos nesta lista se eles não existirem no `package.json`. Atualizar este arquivo quando a stack, versões, ferramentas ou comandos mudarem.
