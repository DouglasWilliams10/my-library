
# ARCHITECTURE.md

## 1. Visão geral

O MyLibrary será uma aplicação web organizada como um monólito modular, com separação de responsabilidades entre apresentação, aplicação, domínio e infraestrutura.

Essa arquitetura busca manter o desenvolvimento simples e sustentável, permitindo que cada módulo tenha responsabilidades bem definidas e possa evoluir sem comprometer os demais.

A aplicação utilizará uma única base de código e uma única aplicação implantável no MVP. Não haverá necessidade de microsserviços, filas externas ou infraestrutura distribuída nesta etapa.

As regras de negócio deverão permanecer independentes dos detalhes de interface, banco de dados e armazenamento de arquivos sempre que possível.

## 2. Estrutura inicial do projeto

A estrutura inicial prevista é:

```text
src/
├── app/
│   ├── (auth)/             # Cadastro, login e autenticação
│   ├── (library)/          # Biblioteca, livros e leitura
│   └── api/                # Endpoints HTTP
├── components/             # Componentes reutilizáveis de interface
├── modules/
│   ├── users/              # Usuários, autenticação e perfil
│   ├── books/              # Catálogo, metadados e ciclo de vida dos livros
│   ├── files/              # Upload, validação, versões e download
│   ├── reader/             # Leitura e progresso
│   └── sharing/            # Compartilhamentos e permissões
├── lib/                    # Configurações e utilitários compartilhados
└── generated/              # Código gerado por ferramentas, quando aplicável

prisma/
├── schema.prisma
└── migrations/

storage/                    # Arquivos armazenados localmente durante o desenvolvimento

docs/                       # Apresentação e documentação do produto

tests/                      # Testes de integração e ponta a ponta
```

A estrutura poderá ser ajustada durante a implementação, desde que as responsabilidades dos módulos e as regras de dependência sejam preservadas.

## 3. Camadas e responsabilidades

### 3.1. Apresentação

Responsável pela interação com o usuário.

Inclui:

- Páginas e layouts da aplicação.
- Formulários e componentes visuais.
- Navegação e apresentação dos dados.
- Comunicação com os endpoints e casos de uso disponibilizados pela aplicação.

A apresentação não deve implementar regras de negócio nem acessar diretamente o banco de dados ou o armazenamento físico.

### 3.2. Aplicação

Responsável por coordenar os casos de uso e organizar a execução das operações.

Inclui:

- Cadastro, autenticação e gerenciamento de usuários.
- Cadastro e gerenciamento de livros.
- Importação, substituição e remoção de arquivos.
- Compartilhamento e revogação de permissões.
- Leitura e atualização do progresso.
- Coordenação das operações persistentes de remoção e recuperação.

A camada de aplicação deve verificar as condições necessárias para executar cada operação e solicitar ao domínio a aplicação das regras correspondentes.

### 3.3. Domínio

Responsável pelas entidades, regras de negócio e validações independentes de framework.

Inclui:

- Identidade e perfil do usuário.
- Propriedade e ciclo de vida dos livros.
- Histórico e situação das versões de arquivos.
- Permissões de compartilhamento.
- Progresso individual de leitura.
- Estados e regras dos processos de remoção.

O domínio não deve depender diretamente de Next.js, React, Prisma ou bibliotecas específicas de interface.

### 3.4. Infraestrutura

Responsável por implementar os recursos técnicos utilizados pelas demais camadas.

Inclui:

- Persistência com PostgreSQL e Prisma.
- Autenticação e gerenciamento de sessão.
- Adaptadores de armazenamento de arquivos.
- Integrações com bibliotecas externas.
- Processamento de tarefas persistentes de remoção.

Os detalhes dos provedores e das bibliotecas devem permanecer isolados sempre que possível.

## 4. Regras de dependência

As seguintes regras deverão orientar a implementação:

1. A apresentação utiliza os recursos disponibilizados pela camada de aplicação.
2. A aplicação coordena os casos de uso e aplica as regras do domínio.
3. A infraestrutura implementa os recursos técnicos necessários aos casos de uso.
4. O domínio não deve depender de frameworks, banco de dados ou serviços externos.
5. Componentes visuais não devem acessar diretamente o banco de dados.
6. Operações privadas devem verificar a identidade e as permissões do usuário no servidor.
7. O acesso aos arquivos deve ocorrer por meio de uma abstração de armazenamento.
8. A lógica de negócio não deve depender de caminhos físicos ou detalhes específicos do provedor de armazenamento.

## 5. Módulos e entidades do domínio

### 5.1. User

Representa o usuário cadastrado na aplicação.

Responsabilidades:

- Armazenar os dados de identificação e perfil.
- Manter as credenciais de autenticação de forma segura.
- Relacionar o usuário aos seus livros, compartilhamentos e progressos.

### 5.2. Book

Representa um livro pertencente a um usuário.

Responsabilidades:

- Armazenar os metadados do livro.
- Identificar seu proprietário.
- Manter sua situação no catálogo.
- Controlar sua exclusão lógica.
- Relacionar-se às versões de arquivos e aos compartilhamentos.

Um livro poderá existir sem possuir um arquivo importado.

### 5.3. BookFile

Representa uma versão de arquivo associada a um livro.

Responsabilidades:

- Armazenar os metadados de cada versão.
- Manter a referência ao arquivo físico enquanto ele existir.
- Identificar a versão atual e as versões históricas.
- Registrar a situação da remoção física.

Várias versões históricas poderão existir para um mesmo livro, mas somente uma poderá estar atual por vez.

Os metadados históricos deverão ser preservados mesmo depois da remoção física dos arquivos.

### 5.4. Library

Representa a organização dos livros na biblioteca de cada usuário.

Responsabilidades:

- Relacionar usuários aos livros que possuem ou receberam por compartilhamento.
- Permitir a apresentação dos livros conforme as permissões disponíveis.
- Respeitar a propriedade e a situação de cada livro.

### 5.5. Share

Representa o compartilhamento de um livro entre usuários.

Responsabilidades:

- Identificar proprietário, destinatário e livro compartilhado.
- Controlar permissões independentes de leitura e download.
- Registrar a situação do compartilhamento.
- Permitir a revogação das permissões.

### 5.6. ReadingProgress

Representa o progresso individual de leitura de um usuário em determinado livro.

Responsabilidades:

- Registrar a posição de leitura.
- Relacionar o progresso ao usuário e ao livro.
- Permitir a continuidade da leitura.
- Respeitar as características do formato do arquivo.

A representação concreta da posição de leitura deverá ser definida conforme as soluções escolhidas para PDF, EPUB e TXT.

### 5.7. BookDeletion

Representa o processo persistente de exclusão de um livro.

Responsabilidades:

- Registrar o início da exclusão lógica.
- Manter o estado do processo.
- Permitir acompanhar a conclusão das remoções físicas.
- Preservar o estado pendente quando houver falhas.
- Registrar a conclusão somente após todas as remoções aplicáveis serem confirmadas.

Estados previstos:

- `pending`: exclusão lógica realizada, mas existem remoções físicas pendentes.
- `completed`: todas as remoções físicas aplicáveis foram confirmadas.

O registro deverá permanecer disponível independentemente da existência dos arquivos físicos.

### 5.8. FileRemovalTask

Representa uma tarefa persistente de remoção física.

Responsabilidades:

- Identificar o arquivo ou recurso local que deverá ser removido.
- Preservar a referência de armazenamento necessária à operação.
- Identificar o motivo da remoção.
- Controlar tentativas, falhas e conclusão.
- Permitir recuperação após reinicializações da aplicação.

Estados previstos:

- `pending`: tarefa aguardando processamento.
- `in_progress`: tarefa em execução.
- `failed`: última tentativa apresentou falha.
- `completed`: remoção física confirmada ou arquivo já inexistente.

As tarefas poderão ser utilizadas tanto na substituição de versões quanto na exclusão de livros.

## 6. Armazenamento de arquivos

O acesso aos arquivos deverá ocorrer por meio de uma interface de armazenamento, mantendo as regras de negócio independentes do provedor utilizado.

### 6.1. Estratégia de armazenamento

- **Desenvolvimento:** armazenamento local, fora do controle de versão do Git.
- **Produção:** a arquitetura deverá permitir migração para armazenamento de objetos.
- **Banco de dados:** armazenará metadados e referências dos arquivos, não o conteúdo binário dos livros.

O provedor de armazenamento de produção e sua configuração ainda não estão definidos.

### 6.2. Importação e versões

Cada importação inicial ou substituição validada deverá criar um novo registro de versão.

A aplicação deverá garantir que:

- Uma versão atual seja identificada por livro.
- Versões anteriores permaneçam registradas como históricas.
- Somente a versão atual possa ser lida ou baixada.
- Versões históricas permaneçam inacessíveis, inclusive enquanto a remoção física estiver pendente.

A invariável de apenas uma versão atual por livro deverá ser protegida pela aplicação e pelo banco de dados, inclusive diante de operações concorrentes.

A estratégia técnica para garantir essa proteção com Prisma e PostgreSQL deverá ser avaliada antes de ser definida.

### 6.3. Substituição de arquivos

A substituição deverá seguir esta sequência:

1. Receber o novo arquivo.
2. Validar completamente o arquivo antes de alterar a versão atual.
3. Efetivar a nova versão como atual.
4. Tornar a versão anterior histórica e imediatamente inacessível.
5. Registrar uma tarefa persistente para remover fisicamente a versão anterior.
6. Processar a remoção por meio do mecanismo de recuperação.

A criação da nova versão, a alteração da versão atual e o registro da tarefa de remoção deverão ser coordenados em uma operação persistente consistente.

Se a remoção física da versão anterior falhar:

- A nova versão continuará atual e funcional.
- A substituição não será desfeita.
- A versão anterior permanecerá inacessível.
- A tarefa continuará pendente para novas tentativas.

O progresso de leitura dos usuários deverá ser reiniciado após uma substituição concluída com sucesso.

### 6.4. Exclusão lógica de livros

A exclusão deverá bloquear imediatamente o acesso ao livro e a todas as suas versões.

O processo deverá:

1. Registrar a exclusão lógica do livro.
2. Revogar os compartilhamentos ativos.
3. Preservar os metadados do livro e das versões.
4. Preservar os registros de progresso de leitura.
5. Registrar as tarefas de remoção física aplicáveis.
6. Processar as remoções até sua conclusão.

A exclusão deverá permanecer pendente enquanto existir qualquer remoção física aplicável não concluída.

A exclusão somente poderá ser considerada concluída após a confirmação de todas as remoções físicas aplicáveis.

### 6.5. Referências de armazenamento e histórico

Os registros históricos das versões deverão ser preservados.

Enquanto uma remoção física estiver pendente:

- A referência de armazenamento necessária à tentativa de remoção deverá ser preservada.
- A versão deverá permanecer inacessível.
- O processo deverá continuar recuperável.

Somente após a confirmação da remoção física:

- A referência de armazenamento (`storageKey`) poderá ser limpa.
- A data da remoção (`deletedAt`) deverá ser registrada.
- A tarefa correspondente poderá ser marcada como concluída.

A existência de metadados históricos não deverá permitir recuperar ou disponibilizar arquivos removidos.

## 7. Recuperação persistente de remoções

O mecanismo de recuperação deverá permanecer dentro do monólito modular.

O MVP não exige microsserviços, filas externas ou provedores de processamento distribuído.

### 7.1. Responsabilidades

- **Módulo de livros:** coordena a substituição e a exclusão lógica.
- **Módulo de arquivos:** valida arquivos, controla versões e solicita remoções físicas.
- **Persistência:** registra `BookDeletion` e `FileRemovalTask` no banco de dados.
- **Adaptador de armazenamento:** executa a remoção física por meio da interface definida.
- **Processador de recuperação:** identifica e processa tarefas persistentes.

### 7.2. Persistência das tarefas

A criação de uma tarefa de remoção deverá ocorrer na mesma transação persistente que efetiva a substituição ou a exclusão lógica correspondente.

Assim, uma operação não poderá ser considerada efetivada sem que sua remoção pendente esteja registrada.

As tarefas deverão permanecer armazenadas no banco de dados mesmo após reinicializações da aplicação.

### 7.3. Processamento e concorrência

O processador deverá:

- Identificar tarefas elegíveis para execução.
- Evitar que duas execuções processem simultaneamente a mesma tarefa.
- Registrar o início de cada tentativa.
- Identificar execuções abandonadas.
- Permitir a recuperação de tarefas após reinicializações.
- Marcar tarefas como concluídas somente após confirmação da remoção física.

Uma tarefa em execução deverá possuir um mecanismo de identificação da execução e um prazo ou critério de expiração, permitindo recuperar operações abandonadas.

Os detalhes técnicos de reivindicação, concorrência e expiração deverão ser definidos durante a implementação.

### 7.4. Falhas e novas tentativas

Quando uma remoção apresentar falha:

- A tarefa deverá permanecer persistida.
- O diagnóstico técnico mínimo deverá ser registrado.
- A quantidade de tentativas e a data da última tentativa deverão ser preservadas.
- A tarefa deverá permanecer elegível para recuperação conforme a política de novas tentativas.

As novas tentativas deverão utilizar espera progressiva entre falhas, com intervalo máximo configurável ou definido pela implementação.

No MVP, uma tarefa não deverá ser abandonada definitivamente apenas por atingir uma quantidade arbitrária de tentativas.

### 7.5. Idempotência

A operação de remoção física deverá ser idempotente.

Se o arquivo já não existir, a remoção poderá ser considerada concluída, desde que essa condição seja confirmada pelo adaptador de armazenamento.

Tarefas concluídas não deverão ser executadas novamente pelo mecanismo automático.

### 7.6. Capas

Capas armazenadas localmente deverão participar do mecanismo de remoção quando forem substituídas ou quando o livro for excluído.

Capas referenciadas por URLs externas não deverão ser enviadas ao adaptador de armazenamento nem removidas do servidor externo.

### 7.7. Conclusão da exclusão

O registro `BookDeletion` somente poderá passar para `completed` quando todas as tarefas de remoção física aplicáveis ao livro estiverem concluídas.

Se qualquer remoção falhar:

- O livro continuará inacessível.
- Os compartilhamentos permanecerão revogados.
- O processo continuará com estado pendente.
- As tarefas deverão permanecer disponíveis para novas tentativas.

A aplicação não poderá apresentar uma exclusão pendente como concluída.

## 8. Segurança e privacidade

A arquitetura deverá garantir que os recursos privados sejam protegidos no servidor.

Diretrizes:

- Senhas nunca deverão ser armazenadas em texto puro.
- O cadastro e o login do MVP utilizarão e-mail e senha.
- Login pelo Google não fará parte do MVP.
- Rotas e operações privadas exigirão autenticação.
- O servidor verificará propriedade ou permissão de compartilhamento antes de permitir leitura ou download.
- As permissões de leitura e download serão independentes.
- A autorização deverá restringir o acesso à versão atual do arquivo.
- Identificadores ou referências de versões históricas não poderão permitir acesso ao conteúdo.
- Arquivos enviados deverão ser validados por extensão, tipo de conteúdo e tamanho.
- Arquivos TXT deverão utilizar codificação UTF-8.
- Segredos e credenciais deverão permanecer em variáveis de ambiente e fora do Git.

### 8.1. Compartilhamento sem permissão de leitura

Quando um usuário receber somente permissão de download:

- O livro poderá aparecer em sua biblioteca.
- O usuário poderá visualizar somente o título e a capa.
- Descrição, autor, editora, categoria e conteúdo não deverão ser disponibilizados.
- O leitor interno deverá permanecer bloqueado.
- O download será permitido somente se a respectiva permissão estiver ativa.

A autorização deverá ser aplicada no servidor, independentemente do comportamento da interface.

### 8.2. Bloqueio durante remoções pendentes

O início da exclusão lógica deverá bloquear imediatamente o acesso ao livro e a todas as suas versões.

Se uma remoção física falhar, o arquivo poderá continuar existindo fisicamente por algum período, mas não poderá ser disponibilizado por nenhum caminho da aplicação.

Na substituição, somente a versão nova permanecerá acessível.

Na exclusão, nenhuma versão permanecerá acessível, inclusive para o proprietário.

Falhas de armazenamento não poderão restaurar permissões ou reativar versões históricas.

## 9. Estratégia de testes

A aplicação deverá possuir testes adequados às responsabilidades de cada camada.

### 9.1. Testes unitários

Deverão validar regras de domínio, incluindo:

- Propriedade dos livros.
- Permissões de compartilhamento.
- Independência entre leitura e download.
- Controle de versões atuais e históricas.
- Regras de substituição.
- Exclusão lógica.
- Estados de remoção.
- Reinicialização do progresso de leitura.

### 9.2. Testes de integração

Deverão validar a interação entre aplicação, persistência, autenticação e armazenamento.

Cenários importantes:

- Cadastro e autenticação.
- Importação e validação de arquivos.
- Substituição de versões.
- Persistência de tarefas de remoção.
- Recuperação após falhas.
- Concorrência na definição da versão atual.
- Exclusão lógica e revogação de compartilhamentos.
- Autorização de leitura e download.
- Preservação dos metadados históricos.

### 9.3. Testes ponta a ponta

Deverão validar os principais fluxos da aplicação sob a perspectiva do usuário.

Incluem:

- Cadastro, login e logout.
- Gerenciamento da biblioteca.
- Importação e leitura de livros.
- Download conforme permissões.
- Compartilhamento e revogação.
- Substituição de arquivos.
- Exclusão de livros.
- Comportamento durante remoções pendentes.

## 10. Diretrizes de evolução

A arquitetura deverá permitir evolução gradual sem comprometer o MVP.

Possíveis evoluções futuras incluem:

- Migração do armazenamento local para armazenamento de objetos.
- Ampliação dos formatos de arquivo suportados.
- Melhorias no leitor e na experiência de leitura.
- Recursos adicionais de organização e personalização.
- Processamento de arquivos mais avançado.
- Melhorias na automação e recuperação de tarefas.

Essas evoluções não deverão introduzir complexidade desnecessária antes que exista uma necessidade concreta.

As decisões técnicas específicas deverão ser documentadas nos arquivos de especificação correspondentes e refletidas na implementação.
