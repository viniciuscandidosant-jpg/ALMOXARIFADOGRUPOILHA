# Almoxarifado de Uniformes e EPI — Ilha do Caranguejo VIX

Sistema completo de controle de uniformes, calçados e EPI: inventário, estoque por
tamanho, entradas (compra e devolução), saídas (entrega definitiva e empréstimo com
recibo), contagem física e painéis. Front-end em arquivo único, banco no Supabase —
mesma arquitetura do NEXUS.

## Arquivos

| Arquivo | O que é |
|---|---|
| `sql/01_schema.sql` | Tabelas, funções, triggers, RPCs e views |
| `sql/02_rls.sql` | Row Level Security e perfis de acesso |
| `sql/03_seed.sql` | Carga inicial com os itens da sua planilha |
| `sql/04_tipos.sql` | Novos tipos de movimento — **rodar sozinho** |
| `sql/05_lavanderia_dotacao.sql` | Ciclo de lavanderia e dotação mínima |
| `index.html` | O sistema inteiro (publicar no GitHub Pages) |

---

## 1. Banco

**Já está no ar.** Projeto Supabase dedicado, separado do NEXUS Requisições:

- Projeto: **NEXUS - Almoxarifado Grupo Ilha** (`vribagnmtksszdfczjcy`), região São Paulo
- Todas as migrations aplicadas, cadastro base carregado, nenhum dado de operação

Os arquivos em `sql/` ficam como histórico versionado e para recriar o banco do zero
se algum dia precisar. Nesse caso, rode **na ordem**:

1. `01_schema.sql`
2. `02_rls.sql`
3. `03_seed.sql`
4. `04_tipos.sql` — **execute este arquivo sozinho, e só depois o próximo.** O Postgres
   não deixa usar um valor novo de enum na mesma transação em que ele foi criado.
5. `05_lavanderia_dotacao.sql`

O seed já cria as 4 unidades do grupo, as 5 categorias, as 4 grades de tamanho
(PP–EXG, 38–50, 36–48 e Tamanho Único), os 23 itens da planilha e as 111 posições
de estoque (item × tamanho), todas zeradas.

## 2. Criar seu acesso

**Authentication → Users → Add user** — crie seu e-mail e senha.

O primeiro usuário do banco vira **Gestor ativo automaticamente**. Todos os seguintes
entram como Consulta e ficam bloqueados até você liberar em **Itens e grades →
Usuários e acessos**. Crie o seu primeiro, antes de qualquer outro.

**Perfis:**

- **Gestor** — tudo: cadastros, ajustes, descarte, cancelamento de lançamento, acessos.
- **Operador** — lança entrada, entrega, empréstimo, devolução e contagem. Não mexe em cadastro nem cancela nada.
- **Consulta** — só lê.

## 3. Publicar

O `index.html` já vem com a URL e a chave pública do projeto preenchidas. É só
commitar na raiz do repositório e ligar **Settings → Pages → Deploy from branch**.

A chave publishable é pública por natureza — vai no código do navegador de qualquer
jeito. Quem protege os dados é a RLS: sem um registro ativo em `almox_usuarios`,
a chave não lê nem escreve nada.

---

## 4. Como o sistema trabalha

**Estoque** — a tela principal reproduz sua planilha: uma matriz item × tamanho, uma
para cada grade. O número grande é o disponível; quando há peça na lavanderia, aparece
um `+N lav` embaixo. Célula amarela é abaixo do mínimo, cinza é zerada. Clicar na célula
abre o histórico daquele tamanho específico e os atalhos de lançamento.

**Entrada por compra** — fornecedor, nota, custo unitário por linha. O custo alimenta
o valor imobilizado do painel.

**Entrega definitiva** — funcionário e justificativa são obrigatórios (o banco recusa
o lançamento sem isso, não só a tela). Gera comprovante de entrega com termo de
responsabilidade e campo de assinatura, referenciando o art. 462 §1º da CLT e a NR-6.

**Empréstimo** — gera um código no seu padrão (`EMP-VIX-260827-1`, com o contador
reiniciando por dia e por unidade), imprime o recibo para assinatura e mantém a peça
como pendência até voltar. Devolução pode ser parcial. Se a peça não voltar, a baixa
converte o pendente em entrega definitiva com motivo registrado.

**Entrada por devolução** — para peça que volta fora de empréstimo: desligamento,
troca de tamanho, peça não usada. Exige funcionário e motivo, e você escolhe o destino:
peça usada vai para a lavanderia, peça limpa volta direto ao estoque.

**Lavanderia** — cada posição de estoque tem dois saldos: *disponível* (pronto para
entregar) e *lavanderia* (peça suja, fora do estoque entregável). Uniforme devolvido
por desligamento e peça que volta de empréstimo caem na lavanderia, não no disponível
— exatamente como acontece na prática. Dali você monta uma remessa (`LAV-VIX-260827-1`),
envia para a lavanderia e, no retorno, informa quantas voltaram limpas e quantas foram
condenadas. As limpas entram no disponível, as condenadas saem de vez com a baixa
registrada. Remessa parada há mais de 10 dias aparece marcada.

O banco não deixa burlar: saída direta do bolsão da lavanderia é recusada, e você não
consegue enviar mais peças do que realmente estão sujas.

**Dotação** — cada funcionário precisa ter um mínimo de peças em posse. A regra padrão
já vem carregada: 2 camisas e 2 peças de baixo (calça ou bermuda) por pessoa. A tela
Dotação mostra quem está abaixo, por grupo, com botão de entrega direto na linha.
Peça emprestada não conta para a dotação — ela volta.

Regras ficam em **Itens e grades → Regras de dotação**. Sem cargo nem setor, a regra
vale para todos; preenchendo um deles, ela substitui a regra geral para quem se encaixa
(útil para exigir 3 camisas da cozinha, por exemplo). Um item só entra na conta se tiver
um **grupo de dotação** preenchido no cadastro — o seed já marcou Camisa, Calça/Bermuda,
Avental e Calçado.

**Inventário** — abre a contagem por categoria (você conta camisas numa semana,
calçados na outra), você digita o que achou na prateleira, e ao finalizar o sistema
gera sozinho os ajustes de entrada e saída, cada um justificado com o código do
inventário. Só uma contagem aberta por vez.

**Cancelamento** — nada é apagado. Cancelar um lançamento estorna o saldo e deixa a
linha no histórico marcada, com motivo e data. Lançamento vinculado a empréstimo não
pode ser cancelado: use a devolução ou a baixa.

**Rede de segurança** — `Recalcular saldos` reconstrói os dois saldos (disponível e
lavanderia) a partir do histórico de movimentações. Se algum dia o número parecer
errado, roda e compara.

## 5. Crescer o cadastro

**Novo uniforme:** Itens e grades → Novo item. Escolha a grade e o sistema cria a
posição de estoque de cada tamanho automaticamente.

**Novo tamanho** (um 52, um XGG): abra a grade → `+ tamanho`. Ele é criado para todos
os itens daquela grade de uma vez.

**Nova grade inteira** (calçado infantil, avental por comprimento): Nova grade, e
digite os rótulos separados por vírgula.

Remover um tamanho não apaga histórico — ele é desativado e some da matriz,
preservando as movimentações antigas.

## 6. Observação sobre a carga inicial

Na sua planilha, `CALÇA - CÂMARA FRIA` está na tabela de numeração 36–48 (a mesma dos
calçados), então o seed manteve fielmente essa grade. Se a calça na verdade usa a
numeração 38–50, é só editar o item e trocar a grade — as posições novas são criadas
na hora.

Os itens marcados como EPI (aventais transparentes, calçados e todo o kit de câmara
fria) já vêm com a flag ligada, e o kit de câmara fria vem marcado como *exige
devolução ao desligamento*.
