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

