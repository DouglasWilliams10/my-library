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

- PostgreSQL gerenciado pelo Neon, utilizado desde o ambiente de desenvolvimento.
- Prisma ORM: versão instalada no projeto, utilizado para acesso ao banco, gerenciamento do schema e migrações.
- A integridade dos dados, incluindo o limite de no máximo uma versão atual por livro diante de operações concorrentes, deve ser protegida também no banco de dados. A estratégia concreta deve ser definida após avaliar as capacidades e limitações do Prisma com PostgreSQL; não presumir uma solução antes dessa avaliação.
- As URLs e credenciais de conexão devem ser fornecidas por variáveis de ambiente, nunca embutidas no código ou nos arquivos de especificação.
- O arquivo `.env` deve permanecer fora do Git. O `.env.example` deve conter somente nomes de variáveis e exemplos fictícios, sem credenciais reais.

## Autenticação

- Auth.js: versão e adaptador compatíveis com a versão do Next.js instalada.
- Cadastro e login do MVP com e-mail e senha; login pelo Google não faz parte do MVP.
- Senhas de contas locais devem ser armazenadas usando hash seguro.
- Sessões e autorização devem ser validadas no servidor.

## Arquivos e leitura

- PDF: visualizador compatível com navegador.
- EPUB: biblioteca de leitura EPUB compatível com React.
- TXT: leitura com codificação UTF-8.
- Cada livro poderá ter várias versões históricas de arquivo, mas no máximo uma versão atual; a aplicação só permitirá ler ou baixar a versão atual.
- O limite de upload será configurável centralizadamente, com valor padrão de 50 MiB (52.428.800 bytes); a validação será feita no servidor e o valor não deve ser duplicado em diferentes pontos do código.
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
- Nomes das variáveis de ambiente de conexão do Neon e configuração específica de conexão do Prisma.

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
