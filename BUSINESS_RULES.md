# BUSINESS_RULES.md

## 1. Usuários e contas

1. Cada usuário deve possuir uma conta identificável por e-mail único.
2. O cadastro e o login do MVP usam e-mail e senha; login pelo Google não faz parte do MVP.
3. O cadastro exige os campos obrigatórios definidos na especificação da funcionalidade.
4. Senhas nunca podem ser armazenadas em texto puro.
5. Um usuário não autenticado não pode acessar sua biblioteca privada, arquivos ou operações de compartilhamento.
6. Cada usuário pode consultar e editar apenas os dados de seu próprio perfil, respeitando as permissões administrativas que venham a ser definidas.

## 2. Livros e propriedade

1. Todo livro cadastrado deve possuir um proprietário.
2. Um livro pode ser cadastrado manualmente ou associado a um arquivo importado.
3. O cadastro deve possuir, no mínimo, título e autor; os demais metadados são opcionais quando não conhecidos.
4. Um livro pode ter várias versões históricas de arquivo, mas pode ter no máximo uma versão atual por vez. Essa invariável deve ser protegida pelas regras da aplicação e pelo banco de dados, inclusive contra operações concorrentes.
5. O proprietário pode editar os metadados, remover o livro de sua biblioteca e controlar seus compartilhamentos.
6. Cada importação inicial bem-sucedida e cada substituição bem-sucedida deve gerar um novo registro de arquivo. Os metadados das versões anteriores devem ser preservados.
7. O proprietário pode substituir o arquivo atual. O novo arquivo deve passar pelas mesmas validações de formato, conteúdo, tamanho e codificação aplicáveis a um upload. A validação completa deve ocorrer antes de efetivar a substituição. Se falhar, a versão atual e o progresso de leitura devem permanecer inalterados.
8. Após a validação completa, a nova versão passa a ser a única atual e a versão anterior passa a histórica e fica imediatamente inacessível. A substituição não altera os metadados do livro nem as permissões de compartilhamento e reinicia o progresso de todos os usuários do livro, incluindo proprietário e destinatários.
9. Versões históricas nunca podem ser lidas ou baixadas pela aplicação. A falha ao remover fisicamente uma versão antiga não desfaz a substituição: a nova versão permanece atual e funciona normalmente, enquanto a anterior permanece histórica, inacessível e com a remoção pendente registrada para recuperação.
10. O registro e os metadados da versão anterior devem ser preservados. Somente após a confirmação da remoção física, a aplicação limpa `storageKey` e registra a data em `deletedAt`.
11. A remoção física pendente deve permitir novas tentativas de recuperação sem restaurar o acesso à versão histórica. Não se pode limpar `storageKey` nem registrar `deletedAt` como remoção concluída antes da confirmação.
12. A exclusão de um livro tem duas etapas. Quando a exclusão lógica é iniciada, o livro e todas as suas versões ficam imediatamente inacessíveis para todos, e seus compartilhamentos ativos são revogados. A exclusão só é concluída depois que todos os arquivos físicos das versões forem removidos com confirmação.
13. A exclusão lógica preserva os metadados do livro, os registros de progresso e os registros históricos de todas as versões de arquivo no banco de dados.
14. Se a remoção física de qualquer versão falhar durante a exclusão, a exclusão permanece pendente e não pode ser apresentada como concluída. A pendência deve permitir novas tentativas sem restaurar o acesso ao livro ou a qualquer versão.
15. Após a confirmação da remoção física de cada versão, seu registro histórico é preservado, `storageKey` é limpo e `deletedAt` é preenchido.
16. Livros sem arquivo digital podem existir como registros de catálogo, mas não podem ser abertos no leitor nem baixados.

## 3. Formatos e arquivos

1. Na primeira versão, são aceitos PDF, EPUB e TXT.
2. A extensão do arquivo não é suficiente para validar o upload: o servidor deve verificar também o tipo de conteúdo.
3. O limite padrão é 50 MiB, exatamente 52.428.800 bytes. O valor deve ser configurável por variável de ambiente ou configuração centralizada, sem duplicação do valor em diferentes pontos do código. A validação acontece no servidor; arquivos acima do limite são recusados com mensagem clara.
4. Arquivos TXT devem usar codificação UTF-8.
5. Arquivos não suportados, inválidos ou corrompidos não devem ser adicionados à biblioteca.
6. O nome original do arquivo deve ser preservado como metadado, mas o nome físico de armazenamento deve ser gerado pela aplicação.
7. Um usuário só pode baixar um arquivo se for seu proprietário ou possuir permissão de compartilhamento que inclua download.
8. A aplicação não deve disponibilizar publicamente arquivos privados por URLs permanentes sem autorização.
9. O armazenamento de desenvolvimento é local e deve ser acessado por uma abstração que permita futura migração para armazenamento de objetos em produção; o provedor de produção ainda não está definido.
10. O sistema não deve remover arquivos ainda usados por uma versão atual de um livro válido. Arquivos de versões históricas e de livros excluídos devem ser removidos conforme as regras de substituição e exclusão, preservando seus registros de metadados.

## 4. Leitura e progresso

1. Somente a versão atual de um livro, se válida e suportada pelo leitor, pode ser aberta.
2. O leitor deve respeitar as permissões de acesso ao livro.
3. O progresso de leitura pertence ao usuário, não ao livro globalmente.
4. O progresso deve ser salvo por usuário e livro, permitindo que usuários diferentes mantenham posições distintas.
5. Se não houver progresso salvo, a leitura começa no início.
6. A posição salva deve respeitar as características do formato; a representação concreta por formato depende da solução de leitura escolhida e ainda precisa ser definida.
7. A aplicação deve informar quando um formato ainda não possui suporte de leitura, sem simular que o livro foi aberto.

## 5. Compartilhamento

1. O compartilhamento é feito pelo proprietário para um usuário cadastrado, identificado por e-mail.
2. O destinatário precisa possuir uma conta ativa para acessar o livro compartilhado.
3. O compartilhamento é privado: livros não aparecem em buscas públicas apenas por terem sido compartilhados.
4. Na primeira versão, o proprietário pode conceder permissão de leitura e, separadamente, permissão de download; download pode ser concedido sem leitura.
5. Se o destinatário tiver `canDownload = true` e `canRead = false`, o livro deve aparecer na biblioteca de compartilhamentos recebidos. Nessa condição, só podem ser disponibilizados o título e a capa; descrição, autor, editora, categoria, conteúdo do arquivo e demais detalhes não podem ser expostos.
6. Nessa condição, o download funciona normalmente e o acesso ao leitor é negado. A autorização e a limitação dos dados retornados devem ser validadas no servidor, não apenas na interface.
7. O destinatário não pode editar os metadados, excluir o livro original ou compartilhar novamente o arquivo.
8. O proprietário pode revogar o compartilhamento a qualquer momento.
9. Após a revogação, o destinatário perde o acesso pela aplicação.
10. A exclusão lógica do livro revoga todos os seus compartilhamentos ativos e impede o acesso dos destinatários.
11. Revogar o acesso não apaga cópias que o destinatário já tenha baixado; a interface deve deixar isso claro antes de conceder download.

## 6. Privacidade e segurança

1. Toda consulta, leitura, alteração ou download de arquivo deve verificar autenticação e autorização no servidor.
2. Não confiar apenas em controles de interface para proteger operações.
3. Mensagens de erro não devem revelar senhas, tokens, caminhos internos ou dados privados de outros usuários.
4. O sistema deve registrar informações mínimas necessárias para diagnosticar falhas de upload e operações relevantes, sem registrar o conteúdo dos livros.
5. A aplicação deve respeitar direitos autorais: o usuário é responsável por possuir autorização para armazenar e compartilhar os arquivos adicionados.

## 7. Resultado esperado

Uma operação só é considerada concluída quando as validações forem aprovadas, os dados persistidos e as permissões respeitadas. Estados de recuperação pendente devem ser registrados explicitamente, sem serem tratados como conclusão física.

Na substituição, a efetivação da nova versão não depende da remoção física da versão anterior. Uma falha de remoção não desfaz a substituição: a nova versão permanece atual e funcional, enquanto a anterior permanece histórica e inacessível, com remoção pendente registrada para novas tentativas. O registro de remoção só é concluído após confirmação física.

Na exclusão de um livro, o início da exclusão lógica torna imediatamente o livro e todas as versões inacessíveis e revoga os compartilhamentos ativos. A exclusão só é concluída quando todos os arquivos físicos forem removidos com confirmação. Uma falha em qualquer remoção mantém a exclusão pendente, permite novas tentativas e não restaura o acesso. Após a confirmação de cada remoção, o registro histórico da versão é preservado, `storageKey` é limpo e `deletedAt` é preenchido.

## 8. Decisões ainda pendentes

- Campos obrigatórios adicionais no cadastro, além de e-mail e senha.
- Representação da posição de leitura específica para cada formato e solução de leitura.
- Mecanismo operacional de recuperação e nova tentativa quando a remoção física de uma versão falha.
