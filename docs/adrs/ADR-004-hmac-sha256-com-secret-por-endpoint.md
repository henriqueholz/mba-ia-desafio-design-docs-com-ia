# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

| Campo | Valor |
| --- | --- |
| **Status** | Aceito (com revisão de segurança obrigatória antes do deploy) |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:19–09:23 |
| **Decisores** | Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos) |
| **Relacionados** | [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [RFC](../RFC.md), [FDD](../FDD.md) |

## Contexto

Vamos enviar dados de pedidos para endpoints **fora da nossa infraestrutura**. O webhook é só outbound: os clientes recebem e não enviam ([09:02] Marcos). O cliente precisa conseguir verificar duas coisas:

1. **Autenticidade:** a requisição veio mesmo da nossa plataforma.
2. **Integridade:** o payload não foi adulterado no caminho.

A empresa já teve um cliente que vazou uma secret em log da própria aplicação ([09:22] Diego). O modelo precisa, portanto, limitar o estrago de um vazamento e permitir a troca da secret.

## Decisão

1. **HMAC-SHA256 sobre o corpo bruto da requisição**, enviado no header `X-Signature`. HMAC-SHA256 é o padrão de mercado e há biblioteca em qualquer stack.
2. **Uma secret única por endpoint de webhook**, gerada pela plataforma e devolvida ao cliente na criação. Não existe secret global. A configuração do webhook guarda `url + secret + customer_id + ativo`.
3. **Rotação pela API:** o cliente pede uma nova secret. A antiga continua **válida por mais 24 horas**, para ele migrar os sistemas. Depois disso, é descartada.
4. **TLS obrigatório:** a URL precisa ser `https://`. URLs `http://` são recusadas com erro de validação no schema Zod. Isso é validação, não decisão arquitetural ([09:23] Sofia).
5. **Revisão de segurança:** Sofia revisa a geração de secret e o HMAC. São reservados **no mínimo 2 dias úteis** antes do deploy ([09:46] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Secret global da plataforma** | Se vaza uma, vaza tudo: todos os clientes ficam comprometidos ao mesmo tempo ([09:21] Sofia). |
| **Rotação sem período de convivência** | O cliente teria que trocar a secret no mesmo instante em que a nossa muda, com risco de rejeitar entregas legítimas. O grace period de 24 h foi pedido explicitamente ([09:21] Sofia). |
| **Sem assinatura (confiar só em TLS)** | TLS protege o transporte, mas não prova ao cliente que foi a nossa plataforma que chamou o endpoint dele. *(Alternativa plausível, não debatida na reunião.)* |

## Consequências

**Positivas**

- O vazamento de uma secret afeta um único endpoint, e o cliente pode rotacioná-la sozinho.
- Os clientes verificam a origem e a integridade com bibliotecas padrão.
- O grace period de 24 h permite rotação sem downtime do lado do cliente.

**Negativas**

- A plataforma precisa guardar a secret de forma **recuperável** (o HMAC exige a secret original). Diferente de senha, ela não pode ser armazenada só como hash. Como protegê-la em repouso é uma questão em aberto no [RFC](../RFC.md#6-questões-em-aberto).
- Durante as 24 h de convivência, o envio precisa deixar o cliente validar com qualquer uma das duas secrets. O formato está detalhado no FDD e é ponto de revisão da Sofia.
- Mais estado no cadastro (secret anterior e expiração) e mais um endpoint.

**Trade-off explícito:** aceitamos a complexidade de gerenciar secrets individuais, rotacionáveis e recuperáveis, em troca de conter o impacto de vazamentos e dar ao cliente uma verificação de origem padrão de mercado.
