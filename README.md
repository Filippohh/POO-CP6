# CP 6 — A Madrugada do FiapBank: Go/No-Go + Pós-Mortem

**Grupo:** Debug Dogs

| Integrante | RM | Turma |
|---|---|---|
| Gabriel do Vale Vergueiro | 564439 | 2CCPX |
| Isabella Piñeiro Santana | 562779 | 2CCPX |
| Pedro Henrique Souza Barreto | 564437 | 2CCPX |
| Filippo Picino Mendez | 565994 | 2CCPX |
| Pam Mitchell Barbosa Durham | 562635 | 2CCPX |
| Gustavo Henrique Rodovalho Silva | 563695 | 2CCPX |

> O grupo não tinha registrado as aulas 1 e 2 neste repositório, então seguiu a seção
> **🔁 Recuperação** da ficha: viveu R1, ⚡A, R2, ⚡B, R3 (Dia 1) e R4, R5, ⚡C, ⚡D, R6 (Dia 2)
> na ordem, respeitando o "quem participa" de cada relâmpago, com um commit por rodada
> (`decisao: Rx - opcao Y (...)`). Depois: ⚡E, Rodada 7, epílogo e pós-mortem.
>
> **Protocolo:** condutor lê a evidência, debate com timer (3 min por rodada, 1 min nos
> relâmpagos), letra por consenso e **só então** a leitura das consequências.

## 📊 Placar 🔥 💰 🧹 (painel de guerra)

- 🔥 **Confiança** dos clientes no banco (mais é melhor)
- 💰 **Prejuízo** acumulado em R$ mil (menos é melhor)
- 🧹 **Dívida técnica** que fica para segunda-feira (menos é melhor)

**Placar inicial (começo do plantão, qui 19h):** 🔥 5 · 💰 0 · 🧹 0

| Momento | Escolha | Efeito | 🔥 | 💰 | 🧹 |
|---|---|---|---|---|---|
| Início | — | — | 5 | 0 | 0 |
| R1 · 19h12 | C | 🔥+1 💰+35 🧹−1 | 6 | 35 | −1 |
| ⚡A · 20h10 | não participou | — | 6 | 35 | −1 |
| R2 · 21h02 | D | 🔥0 💰0 🧹−1 | 6 | 35 | −2 |
| ⚡B · 22h40 | não participou | — | 6 | 35 | −2 |
| R3 · 22h50 | A | 🔥+1 💰+5 🧹0 | 7 | 40 | −2 |
| R4 · 00h47 | A | 🔥+1 💰+15 🧹+1 | 8 | 55 | −1 |
| R5 · 01h30 | A | 🔥−1 💰0 🧹0 | 7 | 55 | −1 |
| ⚡C · 02h20 | B | 🧹0 | 7 | 55 | −1 |
| ⚡D · 03h20 | não participou | — | 7 | 55 | −1 |
| R6 · 03h45 | A | 🔥0 💰0 🧹+1 | 7 | 55 | 0 |
| ⚡E · 05h58 | C | 🔥+1 | 8 | 55 | 0 |
| R7 · 06h15 | B | 🔥+1 💰+40 🧹−1 | 9 | 95 | −1 |
| **Final · 07h00** | **A Noite dos Heróis** | — | **9** | **95** | **−1** |

---

# Dia 1 · O Turno da Noite (qui 19h → 23h)

## RODADA 1 · O saldo que virou negativo (19h12)

- **Tipo:** rodada
- **Condutor:** Filippo
- **Voto:** C (5 × 1; Pedro votou A: "18 min e o bug morre")
- **Decisão:** **C) Corrigir + escrever o teste (`assertThrows`) antes de subir**

**Justificativa (mecanismo técnico):** é uma **exceção lançada e engolida pelo `catch`
no próprio método** (Aula 11). O `if` detecta saldo insuficiente e lança
`SaldoInsuficienteException`, mas o `catch` vazio logo abaixo a captura, e
`origem.debitar(valor + taxa)` roda **sempre**. A exceção existe para interromper o fluxo
e avisar quem chamou; engolida, ela vira um "sucesso" falso. A correção é tirar o
`try/catch` e deixar a exceção subir até o controller (que responde 422/400). Escolhemos C
e não A porque, sem o teste, nada impede que alguém recoloque o `catch` ("cliente
especial") e a regressão volte calada. O teste fixa a invariante *saldo nunca negativo*:

```java
@Test
void deveRecusarTransferenciaQuandoSaldoInsuficiente() {
    assertThrows(SaldoInsuficienteException.class,
        () -> service.realizar("PIX", 1001, 1002, 10_000.0));
    verify(repository, never()).save(any());
}
```

**Consequência:** demorou 40 min a mais (8 clientes afetados no total), mas subiu com o
`assertThrows` protegendo a invariante. **🔥+1 💰+35 🧹−1**

**Placar:** 🔥 6 · 💰 35 · 🧹 −1

---

## ⚡ EVENTO RELÂMPAGO A · A busca que não acha ninguém (20h10)

- **Participação:** **não participamos** (só para quem escolheu D/rollback na R1; nós
  escolhemos C). Lemos a evidência: nossa busca por titular segue funcionando.
- **Anotação do grupo:** o mecanismo é `==` entre `String`s (Aula 2), que compara
  referência e não conteúdo; o certo é usar a derived query `findByTitular` (Aula 13),
  que já existe no repository. A mesma armadilha do bug06 da CP 5.

**Placar:** sem alteração · 🔥 6 · 💰 35 · 🧹 −1

---

## RODADA 2 · A injeção que derreteu o banco (21h02)

- **Tipo:** rodada
- **Condutor:** Isabella
- **Voto:** D (4 × 2; Gustavo e Pam votaram A: "PreparedStatement sai 10 min antes")
- **Decisão:** **D) Usar a derived query `findByTitular` (Spring Data JPA)**
- **Desvio da carta:** não se aplica (não viemos do ⚡A escolhendo B).

**Justificativa (mecanismo técnico):** **concatenação de SQL** (Aulas 12 e 13). Em
`"... WHERE titular = '" + nome + "'"` o valor digitado pelo usuário vira **código SQL**:
com `' OR '1'='1` o `WHERE` fica sempre verdadeiro e devolve todas as contas. A cura são os
**parâmetros vinculados**: o valor vai separado do comando e nunca é interpretado como SQL.
A (PreparedStatement) e D resolvem a injeção do mesmo jeito, mas D usa a abstração que o
projeto já tinha: o `ContaRepository` com `findByTitular(String titular)`, que o Spring
Data traduz para um JPQL parametrizado. Assim apagamos o JDBC na mão do controller
(`Statement` aberto sem `close()`), em vez de reescrevê-lo às pressas. A blocklist (C) foi
descartada na hora: quem filtra caractere perde para o próximo `' OR '2'='2`.

```java
// antes: SQL montado com o texto do usuário
String sql = "SELECT * FROM contas WHERE titular = '" + nome + "'";
// depois: o banco recebe o valor como parâmetro
List<Conta> contas = contaRepository.findByTitular(nome);
```

**Consequência:** o caminho da arquitetura; ~50 min (mais que A), injeção morta, sem
código JDBC novo para manter. **🔥0 💰0 🧹−1**

**Placar:** 🔥 6 · 💰 35 · 🧹 −2

---

## ⚡ EVENTO RELÂMPAGO B · As conexões que não fecham (22h40)

- **Participação:** **não participamos** (só para quem escolheu A/PreparedStatement na
  R2). Com a derived query, quem abre e fecha conexão é o pool gerenciado pelo Spring/JPA.
- **Anotação do grupo:** o mecanismo é a **conexão que vaza sem `close()`** (Aula 12); a
  correção seria **try-with-resources** fechando `Connection`, `Statement` e `ResultSet`.
  Aumentar o pool (B) só adia o estouro.

**Placar:** sem alteração · 🔥 6 · 💰 35 · 🧹 −2

---

## RODADA 3 · A noite das contas gêmeas (22h50)

- **Tipo:** rodada
- **Condutor:** Gabriel
- **Voto:** A (4 × 2; Filippo e Isabella votaram D: "o banco é quem devia numerar")
- **Decisão:** **A) Corrigir o `getInstancia` do singleton (guardar a instância)**

**Justificativa (mecanismo técnico):** **Singleton sem instância guardada** (Aula 14). O
`getInstancia()` fazia `return new GeradorNumeroConta()`: o atributo `static instancia`
nunca era preenchido, cada chamada criava um gerador novo e o contador voltava a 1001.
O padrão só funciona se a instância criada for **guardada** no atributo estático e
devolvida nas próximas chamadas. É exatamente o bug01 que corrigimos na CP 5
(`GeradorProtocolo`), então já sabíamos o conserto e o tempo que ele leva:

```java
public static synchronized GeradorNumeroConta getInstancia() {
    if (instancia == null) {
        instancia = new GeradorNumeroConta();
    }
    return instancia;
}
```

O `synchronized` evita que duas requisições simultâneas criem dois geradores. A sequence
no banco (D) é a arquitetura ideal e ficou anotada para segunda; às 22h50, uma migração de
1h de madrugada era risco demais. UUID (B) quebraria o contrato de número sequencial.

**Consequência:** o padrão volta a funcionar em ~30 min; números únicos a partir de agora.
**🔥+1 💰+5 🧹0**

**Placar:** 🔥 7 · 💰 40 · 🧹 −2

---

# Dia 2 · A Madrugada Aperta (sex 00h → 04h)

## RODADA 4 · O comprovante que aponta pra pessoa errada (00h47)

- **Tipo:** rodada
- **Condutor:** Pedro
- **Voto:** A (4 × 2; Isabella e Gabriel votaram B: "corrigir sem reprocessar deixa a
  auditoria com dados errados")
- **Decisão:** **A) Corrigir a gravação e seguir (histórico antigo fica errado)**

**Justificativa (mecanismo técnico):** **registro invertido na entidade gravada** (Aula 13).
O construtor `Transferencia(tipo, numeroOrigem, numeroDestino, valor, taxa)` recebia
`destino.getNumero(), origem.getNumero()` (argumentos trocados). Como os dois são `int`,
o compilador aceita qualquer ordem: os saldos ficam certos (o débito e o crédito usam os
objetos), mas a entidade persistida pelo JPA guarda o extrato ao contrário. O conserto é
uma linha:

```java
Transferencia extrato = new Transferencia(tipo,
        origem.getNumero(), destino.getNumero(), valor, taxa);
```

A maioria escolheu A pela pressão do relógio: corrige os novos extratos em 30 min e
guarda as 2h de reprocessamento para depois da BF. A minoria avisou que estávamos
empurrando uma dívida de dados para a auditoria (voltamos a isso no pós-mortem).

**Consequência:** novos extratos certos em 30 min; o histórico gravado até agora segue
invertido, dívida de dados para a auditoria pós-BF. **🔥+1 💰+15 🧹+1**

**Placar:** 🔥 8 · 💰 55 · 🧹 −1

---

## RODADA 5 · A fábrica que devolve null (01h30)

- **Tipo:** rodada
- **Condutor:** Pam
- **Voto:** A (6 × 0)
- **Decisão:** **A) Fail-fast: exceção clara na factory (tipo inválido)**

**Justificativa (mecanismo técnico):** **factory com `return null` explodindo longe da
causa** (Aulas 8, 11 e 14). O `default -> null` do `switch` na `ContaFactory` não falha
onde o erro nasce: o `null` viaja pelo service e só explode no `save()` do repository,
com `"Entity must not be null"`, uma mensagem que não cita o tipo de conta nem a factory.
Fail-fast significa lançar a exceção **no ponto da causa**, com mensagem que diz o que
aconteceu, e deixar o controller traduzir para 400:

```java
default -> throw new IllegalArgumentException("Tipo de conta inválido: " + tipo);
```

Validar no controller (B) espalha a regra: o próximo endpoint esquece e o `null` volta.
Devolver CORRENTE (C) esconde o erro e entrega o produto errado. A factory é o único lugar
que conhece todos os tipos, então a validação fica nela (como fizemos no `construir()` do
builder na CP 5).

> Decisão de produto anotada para o pós-mortem: o fail-fast acaba com o erro esquisito, mas
> a UNIVERSITARIA continua sem implementação. Pedimos ao time um `case "UNIVERSITARIA"`
> com a classe nova para segunda.

**Consequência:** a factory rejeita na hora com "Tipo de conta inválido: UNIVERSITARIA";
o erro nasce e morre no mesmo lugar (~30 min). A campanha recebe um "não" claro.
**🔥−1 💰0 🧹0**

**Placar:** 🔥 7 · 💰 55 · 🧹 −1

---

## ⚡ EVENTO RELÂMPAGO C · O cadastro com metade dos campos (02h20)

- **Participação:** **participamos** (R5 = A: a conta segue no ar e o marketing quer a
  landing com só nome e saldo).
- **Condutor:** Gustavo
- **Voto:** B (6 × 0)
- **Decisão:** **B) Builder com valores padrão**

**Justificativa (mecanismo técnico):** **telescópio de construtores × Builder** (Aula 14).
Cada combinação nova de campos pede mais um construtor sobrecarregado (A), e com vários
parâmetros do mesmo tipo fica fácil passar argumentos na ordem errada (o mesmo
mecanismo da R4). O Builder nomeia cada campo, concentra os **valores padrão** e valida
tudo no `build()`, então o objeto só nasce válido. Aceitar `null` (C) espalha
`NullPointerException` pelo sistema, como vimos na R5.

```java
Conta conta = new ContaBuilder()
        .comTitular("Ana")
        .comSaldo(0.0)
        .build();   // tipo = CORRENTE por padrão; titular obrigatório validado aqui
```

**Consequência:** o Builder concentra defaults e validação no `build()`. **🧹0**

**Placar:** sem alteração · 🔥 7 · 💰 55 · 🧹 −1

---

## ⚡ EVENTO RELÂMPAGO D · O teste verde que não testa nada (03h20)

- **Participação:** **não participamos** (só para quem escolheu A ou C na R2; nós
  escolhemos D).
- **Anotação do grupo:** **teste sem assert nenhum** (Aula 15). Um teste sem `assert` só
  falha se lançar exceção: fica verde sem verificar nada. O certo seria
  `assertEquals(150.0, conta.getSaldo(), 0.0001)` depois do `depositar(50.0)`.

**Placar:** sem alteração · 🔥 7 · 💰 55 · 🧹 −1

---

## RODADA 6 · O teste vermelho às 03h45 (a janela fecha às 04h)

- **Tipo:** rodada
- **Condutor:** Filippo
- **Voto:** A (4 × 2; Isabella e Pedro votaram B: "o teste está dizendo que a taxa do
  contrato é 5")
- **Decisão:** **A) `@Disabled` no teste e subir na janela**

**Justificativa do grupo na hora (e mecanismo técnico):** é o conflito **contrato ×
código** (Aula 15). O teste `tedDeveCobrarTaxaDe5Reais` é o contrato executável ("TED =
R$ 5,00") e o serviço tem `double taxa = 50.00`, um número mágico sem constante e sem
comentário. A maioria raciocinou que as correções da noite (R1, R3, R4, R5) **tinham** que
subir antes do congelamento das 04h e que a taxa seria conferida com o gerente de manhã.
Então desligamos o teste com motivo explícito em vez de apagá-lo ou mudar o esperado para
50 (D, que faria a suíte confirmar o bug):

```java
@Disabled("conferir taxa da TED com o gerente: contrato diz 5.0, código diz 50.0")
@Test
void tedDeveCobrarTaxaDe5Reais() { ... }
```

O `@Disabled` não conserta nada: ele **desliga o alarme**. A suíte passa a ficar verde
sem rodar justamente o teste que apontava o bug. Assumimos esse risco conscientemente,
e ele ficou registrado aqui.

**Consequência:** sobe na janela. O teste fica `@Disabled("conferir taxa...")`: alarme
desligado. A taxa de R$ 50 segue no ar. **🔥0 💰0 🧹+1**

**Estado da suíte após a R6:** 🟠 **DOENTE** (R6 = A)

**Placar:** 🔥 7 · 💰 55 · 🧹 0

---

# Dia 3 · Go/No-Go + Pós-Mortem (sex 05h → 07h)

## 🩺 Antes do Go/No-Go: o estado do grupo

- **Placar em dia:** ✅ rodadas R1–R6 recuperadas e commitadas → 🔥 7 · 💰 55 · 🧹 0
- **Estado da suíte:** 🟠 **doente**, definido pela R6 = A (`@Disabled` no teste da taxa)

## ⚡ EVENTO RELÂMPAGO E · "A taxa é 5 ou 50?" (05h58)

- **Participação:** **participamos** (suíte doente: R6 = A). A taxa de R$ 50 segue no ar e
  uma jornalista perguntou ao gerente.
- **Condutor:** Isabella
- **Voto:** C (6 × 0, em 1 min)
- **Decisão:** **C) Reproduzir com um teste antes de responder**

**Justificativa (mecanismo técnico):** o teste que responde a pergunta **já existe**: é o
`tedDeveCobrarTaxaDe5Reais` que desligamos com `@Disabled` (Aula 15). Responder pelo
código (A) seria confirmar a cobrança indevida em público. Responder pela documentação sem
conferir (B) é "chutar com autoridade". Tiramos o `@Disabled`, rodamos `./mvnw test` e
pedimos ao gerente 1 minuto. O resultado foi `expected: <5.0> but was: <50.0>`: o contrato
diz 5 e **o código está cobrando 50**. O gerente respondeu à jornalista: *"A TED custa
R$ 5,00. Identificamos uma cobrança indevida e vamos corrigi-la antes da abertura."*

**Consequência:** o teste dá evidência; a equipe entra no go/no-go sabendo da taxa.
**🔥+1**

**Placar:** 🔥 8 · 💰 55 · 🧹 0

---

## RODADA 7 · Go/No-Go da Black Friday (06h15)

- **Tipo:** rodada final (go/no-go)
- **Condutor:** Gabriel
- **Estado da suíte na mesa:** 🟠 doente (R6 = A), mas com a taxa já confirmada pelo ⚡E
- **Voto:** B (5 × 1; Pam votou C: "PIX no ar salva parte do faturamento já às 08h")
- **Decisão:** **B) Corrigir o que a noite revelou + suíte verde, abrir 2h atrasado**

**Justificativa (mecanismo técnico):** o Dev 3 apontou o padrão da noite: todo bug morou
no `TransferenciaService`, 80 linhas que ninguém entende inteiras (exceção engolida na R1,
extrato invertido na R4, taxa mágica na R6). Abrir 100% (A) com a suíte doente seria
cobrar R$ 50 de cada TED no maior pico do ano, com o teste que prova o erro desligado.
Desligar a TED (C) só esconde a taxa até segunda. Adiar a campanha (D) joga fora a BF
inteira por um problema que se corrige em 2h. Usamos as 2h para:

1. **Corrigir a taxa** e matar o número mágico (Clean Code, como o clean04 da CP 5):
   ```java
   private static final double TAXA_TED = 5.00;   // contrato com o cliente: TED = R$ 5,00
   ```
2. **Suíte verde de verdade:** `@Disabled` removido, `tedDeveCobrarTaxaDe5Reais` passa
   sem mexer no valor esperado. O teste é o contrato e o código é que se ajusta a ele.
3. **Revisar o `TransferenciaService` inteiro com a suíte rodando:** o `assertThrows` da
   R1 seguiu verde e confirmamos o extrato com origem/destino na ordem certa (R4).

`./mvnw test` → `Tests run: 14, Failures: 0, Skipped: 0` (13 originais + o
`assertThrows` da R1).

**Consequência (coluna suíte DOENTE):** ao revisar o serviço, o time enxerga a taxa 50 e
conserta antes de abrir; as 2h doeram menos do que a taxa doeria. **🔥+1 💰+40 🧹−1**

**Placar:** 🔥 9 · 💰 95 · 🧹 −1

---

## 🏁 Epílogo · Amanhece a Black Friday (07h00)

### Placar final (congelado)

| 🔥 Confiança | 💰 Prejuízo | 🧹 Dívida técnica | Suíte |
|---|---|---|---|
| **9** | **95** | **−1** | 🟢 verde de verdade (após a R7) |

### Perfil da noite: **A Noite dos Heróis** 🦸

| Critério | Nosso valor | Atende? |
|---|---|---|
| A Noite dos Heróis: 🔥 7 ou mais **e** 🧹 até 2 | 🔥 9 · 🧹 −1 | ✅ |
| O Preço da Pressa: 💰 400 ou mais | 💰 95 | ❌ |
| Firefighters Crônicos: 🧹 6 ou mais | 🧹 −1 | ❌ |

> **Como amanhece o banco:** abriu às 10h, os clientes confiam e o time dorme (quase) em
> paz. O "quase" vem da dívida de dados da R4 (extratos antigos invertidos) e da conta
> Universitária que ainda não existe (R5).

---

# 📝 Pós-Mortem · Relatório de Incidente

**Incidente:** madrugada de plantão pré-Black Friday no FiapBank (qui 19h → sex 08h)
**Sistema:** API Spring Boot do FiapBank (`TransferenciaService`, `ContaFactory`,
`GeradorNumeroConta`, `ContaRepository`)
**Gravidade:** alta (dinheiro de cliente, segurança, conformidade com o Bacen)
**Resultado:** campanha aberta às **10h** (2h de atraso), sem cobrança indevida, sem
dados vazados. Placar final 🔥 9 · 💰 95 · 🧹 −1, perfil **A Noite dos Heróis**.

## 1. Linha do tempo da madrugada

| Relógio | Evento | Mecanismo (aula) | O que o grupo fez |
|---|---|---|---|
| qui 19h12 | **R1** · saldo negativo após transferências | exceção engolida pelo `catch` (Aula 11) | C · corrigiu + `assertThrows` |
| 20h10 | ⚡A · busca por titular vazia | `==` entre Strings × derived query (Aulas 2 e 13) | não participou (R1 ≠ D) |
| 21h02 | **R2** · SQL injection no busca-por-titular | concatenação de SQL × parâmetros vinculados (Aulas 12 e 13) | D · derived query `findByTitular` |
| 22h40 | ⚡B · pool de conexões esgotado | conexão sem `close()` / try-with-resources (Aula 12) | não participou (R2 ≠ A) |
| 22h50 | **R3** · duas contas com o número 1001 | Singleton sem instância guardada (Aula 14) | A · guardou a instância |
| sex 00h47 | **R4** · comprovante com origem/destino invertidos | registro invertido na entidade gravada (Aula 13) | A · corrigiu a gravação, sem reprocessar |
| 01h30 | **R5** · "Entity must not be null" na Universitária | factory com `return null` (Aulas 8, 11 e 14) | A · fail-fast na factory |
| 02h20 | ⚡C · cadastro com só nome e saldo | telescópio de construtores × Builder (Aula 14) | B · Builder com defaults |
| 03h20 | ⚡D · teste verde sem assert | teste sem assert (Aula 15) | não participou (R2 ≠ A/C) |
| 03h45 | **R6** · `tedDeveCobrarTaxaDe5Reais` vermelho | contrato × código, `@Disabled` (Aula 15) | A · `@Disabled` e subiu na janela |
| 05h58 | ⚡E · "a taxa é 5 ou 50?" | o teste desligado tinha a resposta (Aula 15) | C · reativou e rodou o teste |
| 06h15 | **R7** · Go/No-Go da Black Friday | — | B · corrigiu + suíte verde, abriu 10h |
| 07h00 | Epílogo | — | placar congelado: A Noite dos Heróis |

## 2. O caminho do grupo

| Rodada | Escolha | O que ela causou | 🔥 | 💰 | 🧹 | Placar após |
|---|---|---|---|---|---|---|
| R1 | C | +40 min e 8 clientes afetados, mas a invariante "saldo nunca negativo" ganhou teste | +1 | +35 | −1 | 6 · 35 · −1 |
| ⚡A | — | não participou | | | | 6 · 35 · −1 |
| R2 | D | injeção morta com a abstração que já existia; nenhum JDBC novo, então o ⚡B não nos atingiu | 0 | 0 | −1 | 6 · 35 · −2 |
| ⚡B | — | não participou | | | | 6 · 35 · −2 |
| R3 | A | números únicos a partir das 23h20; a sequence no banco ficou para segunda | +1 | +5 | 0 | 7 · 40 · −2 |
| R4 | A | extratos novos certos, **histórico antigo segue invertido** (dívida de dados) | +1 | +15 | +1 | 8 · 55 · −1 |
| R5 | A | erro claro na origem; a campanha Universitária recebeu um "não" honesto | −1 | 0 | 0 | 7 · 55 · −1 |
| ⚡C | B | Builder concentrou defaults e validação, telescópio evitado | | | 0 | 7 · 55 · −1 |
| ⚡D | — | não participou | | | | 7 · 55 · −1 |
| R6 | A | **alarme desligado**: subiu na janela com a taxa de R$ 50 no ar e a suíte doente | 0 | 0 | +1 | 7 · 55 · 0 |
| ⚡E | C | o teste reativado provou a cobrança indevida antes da imprensa | +1 | | | 8 · 55 · 0 |
| R7 | B | taxa corrigida, suíte verde de verdade, abertura às 10h | +1 | +40 | −1 | **9 · 95 · −1** |

**Pior momento:** 03h45 (R6). Com o relógio correndo, escolhemos desligar o único teste
que dizia a verdade sobre a taxa. Por duas horas, um bug que cobra 10× do cliente ficou no
ar com a suíte "verde".

**Melhor momento:** 05h58 (⚡E). Em vez de responder de memória, rodamos o teste. O mesmo
teste que tínhamos silenciado salvou o banco de confirmar em público uma cobrança indevida.

## 3. Causa raiz de 2 incidentes

### Incidente R1 · O saldo que virou negativo: **exceção engolida** (Aula 11)

- **Sintoma:** `POST /api/transferencias` devolvia sucesso para transferências sem saldo;
  um cliente ficou com −R$ 8.940.
- **Mecanismo:** dentro de `TransferenciaService.realizar()`, a regra
  `if (origem.getSaldo() < valor + taxa) throw new SaldoInsuficienteException(...)` estava
  dentro de um `try` cujo `catch (SaldoInsuficienteException e)` era **vazio** (só um
  comentário de "cliente especial"). Uma exceção serve para interromper o fluxo e
  propagar o erro até quem pode tratá-lo; capturada e ignorada no mesmo método, ela vira
  um não-evento, e a linha seguinte (`origem.debitar(valor + taxa)`) roda **sempre**.
- **Por que passou:** não havia teste para o caminho de erro. O caminho feliz funcionava,
  como nos bugs escondidos da CP 5 (bug07: catch genérico devolvendo `null`).
- **Correção:** remover o `try/catch`; a exceção sobe até o controller, que responde com
  erro. Teste `assertThrows(SaldoInsuficienteException.class, ...)` +
  `verify(repository, never()).save(any())`.
- **Prevenção:** regra de revisão: **`catch` vazio não passa no code review**. Se uma
  exceção é capturada, ela é tratada, relançada ou logada com motivo.

### Incidente R6 · O teste vermelho às 03h45: **contrato × código + `@Disabled`** (Aula 15)

- **Sintoma:** `TransferenciaServiceTest.tedDeveCobrarTaxaDe5Reais` →
  `expected: <5.0> but was: <50.0>`, 15 min antes do congelamento.
- **Mecanismo:** o teste é o **contrato executável** com o cliente (TED = R$ 5,00). O
  serviço tinha `double taxa = 50.00`, um **número mágico** sem constante e sem comentário,
  então ninguém sabia se 50 era intenção ou erro de digitação. Diante do conflito, a
  pergunta certa é "quem está errado, o contrato ou o código?". Nós respondemos "depois" e
  usamos `@Disabled`, que faz o JUnit **pular** o teste: a suíte fica verde porque **não
  roda** a verificação, não porque o código está certo. É um alarme desligado, parente
  do teste sem assert do ⚡D.
- **Por que passou:** a pressão da janela das 04h e a falta de uma regra de "suíte
  vermelha bloqueia o deploy".
- **Correção (R7):** `@Disabled` removido; `private static final double TAXA_TED = 5.00`
  com comentário apontando o contrato; a suíte voltou a 0 falhas **sem alterar o valor
  esperado do teste**.
- **Prevenção:** pipeline que **falha se houver `@Disabled` sem issue vinculada**;
  constantes nomeadas para todo valor de regra de negócio (taxas, limites).

## 4. O que faríamos diferente (2 rodadas)

### R6 · faríamos **B** (investigar e perder a janela) em vez de A

Desligar o teste custou 🧹+1 na hora e quase custou a reputação do banco às 05h58. Se
tivéssemos escolhido B às 03h45, a investigação teria revelado ali que o contrato diz R$ 5.
Teríamos perdido a janela (🔥−1 💰+30), mas a suíte ficaria saudável, o ⚡E nem nos
atingiria e a R7 seria decidida na coluna saudável. A lição: **teste vermelho é
informação, não obstáculo**. Quando o código e o contrato divergem, a dúvida se resolve
antes do deploy, não com o alarme desligado.

### R4 · faríamos **B** (corrigir + reprocessar os extratos) em vez de A

A minoria (Isabella e Gabriel) tinha razão. Ganhamos 🔥+1 com A, mas deixamos o histórico
de comprovantes invertido. Em banco, extrato é documento: reprocessar levaria ~2h às 00h47,
num horário em que ainda havia folga, e entregaria integridade total (🧹−1 em vez de +1).
Agora a auditoria pós-BF vai encontrar comprovantes errados, e corrigir dado histórico
depois que o cliente já baixou o PDF é bem mais caro. A lição: **corrigir o código não
corrige os dados que ele já gravou**.

## 5. Ações de acompanhamento (segunda-feira)

| # | Ação | Origem |
|---|---|---|
| 1 | Reprocessar os extratos gravados invertidos antes da R4 e reemitir comprovantes | R4 |
| 2 | Migrar a numeração de contas para uma sequence do banco | R3 |
| 3 | Implementar a `Universitaria` na `ContaFactory` (com teste) | R5 |
| 4 | Quebrar o `TransferenciaService` (validação, cálculo de taxa, gravação de extrato) | R7 |
| 5 | CI bloqueando deploy com suíte vermelha ou `@Disabled` sem justificativa | R6 |

## 6. Apresentação relâmpago (roteiro de 2 a 3 min): "a madrugada do Debug Dogs"

1. **(20 s) Abertura:** "Começamos a noite com 🔥5 e terminamos com 🔥9: A Noite dos
   Heróis. Mas teve um momento em que quase viramos manchete."
2. **(40 s) Caminho:** R1, corrigimos com teste; R2, derived query; R3, o mesmo singleton
   da CP 5; R4, corrigimos a gravação mas não o histórico; R5, fail-fast; ⚡C, Builder.
3. **(40 s) Pior momento:** 03h45, `@Disabled` no teste da taxa para não perder a janela.
   Às 05h58 a jornalista ligou, e foi esse teste, religado, que deu a resposta em 1 minuto.
4. **(40 s) Maior lição:** teste vermelho é informação. O `@Disabled` não conserta o bug,
   só desliga o alarme. E corrigir o código não corrige os dados já gravados (R4).
5. **(10 s) Fechamento:** a palavra da madrugada.

## 7. Uma palavra que resume a madrugada

# **Evidência.**

> Toda decisão boa da noite veio de evidência (o `assertThrows` da R1, o teste rodado no
> ⚡E). A pior veio de desligar a evidência (o `@Disabled` da R6).

*O banco abriu às 10h.* ⏰

