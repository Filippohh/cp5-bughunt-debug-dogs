# Checkpoint 5 — Bug Hunt PetFiap

## Identificação

**Grupo:** Debug Dogs

| Integrante | RM | Turma |
|---|---|---|
| Gabriel do Vale Vergueiro | 564439 | 2CCPX |
| Isabella Piñeiro Santana | 562779 | 2CCPX |
| Pedro Henrique Souza Barreto | 564437 | 2CCPX |
| Filippo Picino Mendez | 565994 | 2CCPX |
| Pam Mitchell Barbosa Durham | 562635 | 2CCPX |
| Gustavo Henrique Rodovalho Silva | 563695 | 2CCPX |

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |
| **Total de testes novos escritos** | 6 / 6 |
| **Suíte final (Run As → JUnit Test)** | 26 testes (20 entregues + 6 novos), 0 falhas |

---

## Parte 1 — Bugs encontrados

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | `GeradorProtocoloTest`: `assertSame` falhou (dois objetos diferentes) e `deveGerarProtocolosSequenciais` deu `expected: <2> but was: <1>` | `GeradorProtocolo.getInstancia()` (~linha 15): fazia `return new GeradorProtocolo()` sem guardar em `instancia`, então cada chamada criava um gerador novo com contador zerado | `instancia = new GeradorProtocolo();` antes do `return`; `getInstancia()` e `proximo()` ficaram `synchronized`, cumprindo o "thread-safe" que o comentário prometia | Padrão Singleton (Aula 14), atributo `static` |
| bug02 | `AtendimentoBuilderTest.deveMontarAtendimentoCompleto`: `expected: <Rex> but was: <null>` | `AtendimentoBuilder.comPet()` (~linha 21): `petNome = petNome;` atribuía o parâmetro a ele mesmo | `this.petNome = petNome;` | Palavra-chave `this`, sombreamento de atributo por parâmetro (POO) |
| bug03 | `deveRecusarMontagemSemNomeDoPet` / `deveRecusarMontagemSemPorte`: "Expected IllegalArgumentException to be thrown, but nothing was thrown" | `AtendimentoBuilder.construir()` (~linha 38): não validava nada; o comentário dizia que a validação era "do controller", que também não validava | `construir()` lança `IllegalArgumentException` se nome do pet ou porte forem nulos/vazios — o objeto só nasce válido | Padrão Builder (Aula 14), exceções unchecked (Aula 11) |
| bug04 | `deveCriarTosaQuandoTipoForTosa`: `expected: <Tosa> but was: <Banho>` | `AtendimentoFactory.criar()` (~linha 15): `case "TOSA" -> new Banho(...)` (copiar/colar) | `case "TOSA" -> new Tosa(...)` | Padrão Factory (Aula 14), polimorfismo |
| bug05 | `devePreencherOsDadosDoPetNaConsulta`: `expected: <Mimi> but was: <null>` | `ConsultaVeterinaria` (construtor, ~linha 17): chamava `super()` vazio e descartava todos os parâmetros | `super(protocolo, petNome, petPorte, tutorNome, dataHora);` | Herança e construtores (`super`) |
| bug06 | `deveRecusarAgendamentoComHorarioJaOcupado`: esperava `HorarioOcupadoException`, veio `NullPointerException` (o conflito não foi detectado, o código seguiu para `save` não stubado) | `AgendaService.agendar()` (~linha 31): comparava `petNome` e `dataHora` com `==` (referência) em vez de conteúdo | Trocado por `.equals()` nas duas comparações | `==` vs `.equals()` (Aula 7) |
| bug07 | `deveLancarExcecaoQuandoAtendimentoNaoExiste`: "Expected AtendimentoNaoEncontradoException ... nothing was thrown" | `AgendaService.buscarPorId()` (~linha 46): `try/catch (Exception e) { return null; }` engolia a exceção lançada pelo `orElseThrow` | Removido o `try/catch`; a exceção chega ao controller, que responde 404 | Tratamento de exceções, catch genérico (Aula 11) |
| bug08 | Teste novo `BanhoTest.deveCobrarPrecoDaTabelaQuandoBanhoForPorPorte`: `expected: <60.0> but was: <100.0>` | `Banho.calcularPreco()` (~linha 25): valores de PEQUENO (100) e GRANDE (60) invertidos | PEQUENO → 60, MEDIO → 80, GRANDE → 100, como na tabela do contrato | Regras de negócio no model / polimorfismo |
| bug09 | Teste novo `TosaTest.deveDurar60MinutosQuandoForTosa`: `expected: <60> but was: <30>` | `Tosa` (~linha 37): `getDuracaoMinutos(String porte)` era uma **sobrecarga** — o polimorfismo chamava o `getDuracaoMinutos()` de `Atendimento` (30) | Assinatura corrigida para `getDuracaoMinutos()` com `@Override` | Sobrescrita vs sobrecarga, `@Override` (Aula 7) |
| bug10 | Teste novo `AgendaServiceTest.deveRecusarCancelamentoQuandoAtendimentoJaConcluido`: nada foi lançado e o status virou CANCELADO | `Atendimento.cancelar()` (~linha 68): mudava o status sem verificar o estado atual | Lança `StatusInvalidoException` se o status não for AGENDADO (igual ao `concluir()`) | Encapsulamento de regras de transição de estado, exceções customizadas |
| bug11 | Teste novo `AgendaServiceTest.deveRecusarAgendamentoQuandoDataHoraNoPassado`: esperava `IllegalArgumentException`, veio `NullPointerException` (passou direto, consultou o banco) | `AgendaService.agendar()` (~linha 28): não existia validação de data | Antes de qualquer acesso ao repository: data nula ou anterior a `LocalDateTime.now()` → `IllegalArgumentException` (controller responde 400) | Validação de regra de negócio no service, fail-fast |
| bug12 | Achado em code review e confirmado subindo a API com H2: `POST /api/atendimentos` respondia **500** — `IdentifierGenerationException: Identifier of entity 'Banho' must be manually assigned before calling 'persist()'` | `Atendimento` (~linha 14): `@Id` sem `@GeneratedValue`; ninguém atribuía o `id`, então o Hibernate recusava o `persist` | `@GeneratedValue(strategy = GenerationType.IDENTITY)` no `id` | JPA / mapeamento de entidades (Aula 13) |

## Parte 2 — Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `AtendimentoFactory.criar(int p, String t, String n, String po, String tu, LocalDateTime d)` | Nomes significativos — parâmetros de uma/duas letras escondem o significado (e facilitaram o bug04) | Renomeados para `protocolo, tipo, petNome, petPorte, tutorNome, dataHora` |
| clean02 | `AtendimentoController.calcularDescontoFidelidade()` + bloco de comentário "futuro" | Código morto / YAGNI — método privado nunca chamado | Método e comentário removidos (o histórico do git guarda a ideia) |
| clean03 | `System.out.println` em `AgendaService.agendar()` e no construtor do `GeradorProtocolo` | Uso de `System.out` em aplicação real: sem nível, sem controle, sem destino configurável | Logger SLF4J (`LoggerFactory.getLogger`), que já vem com o Spring Boot, com mensagens parametrizadas `{}` |
| clean04 | Strings `"AGENDADO"`, `"CONCLUIDO"`, `"CANCELADO"` repetidas em `Atendimento` e `AgendaService` | Números/strings mágicas e DRY — um erro de digitação compilaria normalmente | Constantes `STATUS_AGENDADO`, `STATUS_CONCLUIDO`, `STATUS_CANCELADO` em `Atendimento`, usadas no model e no service |
| clean05 | `@Autowired` em atributo no `AgendaService` e no `AtendimentoController` | Injeção por campo: dependência escondida, atributo não pode ser `final`, dificulta testes | Injeção via construtor com atributos `private final` (o Spring injeta pelo construtor; o `@InjectMocks` do Mockito também) |
| clean06 | Comentário do `POST` no `AtendimentoController` dizia `POST /api/atendimentos?tutorNome=Ana` e o exemplo usava data já passada | Comentários que mentem / desatualizados | Comentário descreve os parâmetros reais e o exemplo usa data futura |

## Parte 3 — Testes novos (regras que estavam sem cobertura)

| # | Teste escrito (classe.método) | Regra coberta | Resultado ao escrever (vermelho/verde) |
|---|---|---|---|
| teste01 | `BanhoTest.deveCobrarPrecoDaTabelaQuandoBanhoForPorPorte` | Banho: R$ 60 / 80 / 100 por porte | 🔴 Vermelho — revelou o **bug08** (preços invertidos) |
| teste02 | `TosaTest.deveDurar60MinutosQuandoForTosa` | Tosa dura 60 min | 🔴 Vermelho — revelou o **bug09** (sobrecarga no lugar de sobrescrita) |
| teste03 | `AgendaServiceTest.deveRecusarCancelamentoQuandoAtendimentoJaConcluido` | `cancelar()` em CONCLUIDO é recusado (`StatusInvalidoException`) e nada é salvo | 🔴 Vermelho — revelou o **bug10** (cancelar sem checar status) |
| teste04 | `AgendaServiceTest.deveRecusarAgendamentoQuandoDataHoraNoPassado` | Data/hora no passado → `IllegalArgumentException`, banco nem é consultado (`verify(..., never())`) | 🔴 Vermelho — revelou o **bug11** (sem validação de data) |
| teste05 | `ConsultaVeterinariaTest.deveCustar150ReaisQuandoQualquerPorte` | Consulta custa R$ 150 fixo, independentemente do porte | 🟢 Verde de cara — regra já estava correta |
| teste06 | `AgendaServiceTest.deveCancelarQuandoAtendimentoEstaAgendado` | `cancelar()` em AGENDADO vira CANCELADO e é salvo | 🟢 Verde de cara — regra já estava correta |

---

## Parte 4 — Perguntas de reflexão

### 1. A suíte como contrato (Aula 15)
Rodamos a suíte antes de tocar em qualquer coisa e anotamos os 9 vermelhos. Cada mensagem já apontava o lugar: `expected: <Rex> but was: <null>` no `AtendimentoBuilderTest` mostrou que o nome entrava no builder e se perdia, o que levou direto ao `petNome = petNome` sem `this` no `comPet()`. `expected: <Tosa> but was: <Banho>` apontou o `case` da factory, e `expected: <2> but was: <1>` mostrou que o contador do Singleton "zerava" a cada chamada. O caso mais interessante foi o `NullPointerException` no lugar da `HorarioOcupadoException`: ele mostrou que o conflito não foi detectado (por causa do `==`) e que o código seguiu até o `save`. A vantagem sobre o curl é que a suíte roda em segundos, sem Oracle, sem subir o Spring, e repete os 26 cenários a cada correção — foi assim que vimos, depois de cada commit, que nenhuma correção quebrou outra regra. Com curl teríamos que montar cada cenário (pet duplicado, atendimento concluído…) à mão, toda vez.

### 2. Mock e injeção de dependência (Aulas 13 a 15)
Em produção quem injeta é o container do Spring: ao criar o bean `AgendaService` ele encontra o construtor `AgendaService(AtendimentoRepository repository)` (depois do clean05; antes era o `@Autowired` no campo) e passa a implementação real do `AtendimentoRepository` que o Spring Data gera, ligada ao Oracle. No teste, quem faz esse papel é o Mockito: o `@Mock` cria um `AtendimentoRepository` falso, e o `@InjectMocks` cria o `AgendaService` passando esse falso no mesmo construtor. Como o service depende só da interface, ele não sabe se o outro lado é o Oracle ou um mock. O mock só responde o que ensinamos (`when(repository.findById(1L)).thenReturn(Optional.of(agendado))`) e registra as chamadas, então dá para verificar até que o banco **não** foi tocado (`verify(repository, never()).save(any())`). Nada de JPA, conexão ou contexto Spring é criado, por isso o teste roda sem banco e sem rede.

### 3. `==` vs `.equals()` (Aula 7)
No `AgendaService.agendar()` o código fazia `a.getPetNome() == novo.getPetNome() && a.getDataHora() == novo.getDataHora()`. O `==` compara referências (se é o mesmo objeto na memória), não o conteúdo. No teste, o horário da nova tentativa vem de `LocalDateTime.parse(existente.getDataHora().toString())` — mesmo valor, objeto diferente — então o `==` dava `false` e o agendamento duplicado passava. Com o nome do pet o `==` "funcionava por sorte": `"Rex"` escrito no código é um literal, e a JVM coloca literais iguais no *String pool*, então os dois `"Rex"` do teste eram a mesma referência. Numa requisição real o nome vem do HTTP/banco, e cada String é um objeto novo, então nem o nome bateria. A correção trocou as duas comparações por `.equals()`, que `String` e `LocalDateTime` sobrescrevem para comparar valor.

### 4. Sobrescrita vs sobrecarga (Aula 7)
`Atendimento` declara `getDuracaoMinutos()` (sem parâmetros, retorna 30). A `Tosa` declarou `getDuracaoMinutos(String porte)` — mesmo nome, lista de parâmetros diferente — o que é uma **sobrecarga** (overload): um método novo, que só é chamado se alguém passar uma String. Já a **sobrescrita** (override) exige a mesma assinatura e é ela que o polimorfismo usa em tempo de execução. Como o controller chama `atendimento.getDuracaoMinutos()` sobre o tipo `Atendimento`, a JVM procurava a versão sem parâmetros, não achava na `Tosa` e caía na da superclasse: 30 minutos em vez de 60. Compila sem erro porque sobrecarga é legal. Com `@Override` em cima do método errado, o compilador teria recusado ("method does not override or implement a method from a supertype"), e o bug nem chegaria a existir — por isso a correção já inclui o `@Override`, como na `Banho`.

### 5. Singleton manual vs bean do Spring (Aula 14)
O `GeradorProtocolo` deveria garantir uma única instância (construtor `private`, atributo `static instancia`) para que todo o sistema compartilhe o mesmo contador e os protocolos saiam 1, 2, 3… O bug era que `getInstancia()` fazia `return new GeradorProtocolo()` sem guardar o objeto em `instancia`: o `if (instancia == null)` era sempre verdadeiro, cada chamada criava um gerador novo com `contador = 0`, e todos os protocolos saíam 1. Corrigimos guardando a instância e deixando os métodos `synchronized`, porque o comentário prometia thread-safety que o código não tinha (duas requisições simultâneas poderiam criar duas instâncias ou repetir um número). O `AgendaService` não corre esse risco porque quem controla o ciclo de vida dele é o container do Spring: um `@Service` é por padrão um bean de escopo *singleton*, criado uma vez na inicialização e injetado em quem precisar — ninguém faz `new AgendaService()` e não há código escrito à mão para errar.

### 6. Cobertura de testes: onde parar? (Aula 15)
Vale manter os dois que ficaram verdes (`deveCustar150ReaisQuandoQualquerPorte` e `deveCancelarQuandoAtendimentoEstaAgendado`): eles documentam a regra e protegem contra regressão. Um exemplo real: ao corrigir o bug10 mudamos o `cancelar()`, e o teste06 garantiu que o caminho feliz (AGENDADO → CANCELADO) continuou funcionando. Em um projeto com prazo, priorizaríamos as regras de negócio e os caminhos de erro, não 100% de cobertura. Neste checkpoint, os 4 bugs escondidos estavam justamente nas regras sem teste, e dois deles em caminhos de erro (cancelar concluído, data no passado) — o caminho feliz "funcionava". Getters/setters e o `PetFiapApplication` não precisam de teste; preço, transições de status e conflito de agenda, sim, porque é onde um erro custa dinheiro ou um atendimento duplicado. Cobertura de 100% pode virar meta vazia: o bug12 (`@GeneratedValue`) só apareceria em um teste de integração com banco, não em mais testes unitários.

---

## Parte 5 — Espaço livre (opcional)

```
- O bug12 não aparecia na suíte unitária (o repository é mockado). Encontramos lendo
  a entidade e confirmamos subindo a API com H2 em memória, passando a URL do H2 só
  por argumento de linha de comando — o application.properties continua com
  SEU_RM/SUA_SENHA.
- Observação: o controller gera o protocolo ANTES de validar/agendar, então uma
  requisição recusada (400/409) consome um número e a sequência pula (ex.: 1, 5).
  Não alteramos para manter as mudanças mínimas, mas fica registrado como melhoria.
```
