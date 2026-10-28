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

