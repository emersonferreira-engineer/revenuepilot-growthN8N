# Integração RevenuePilot ↔ n8n

Este workflow é a "IA copiloto" citada no README principal do RevenuePilot. Ele roda numa
instância n8n separada e é chamado pelo frontend apenas através de `createServerFn`
(`src/lib/ai/n8n.functions.ts`) — nunca diretamente do navegador — para que a URL do webhook e
qualquer credencial de modelo fiquem no servidor.

## Como os dois projetos se conectam

```
RevenuePilot (frontend)                    n8n (este workflow)
────────────────────────                   ─────────────────────────────
Tela de oportunidade                        1. Webhook RevenuePilot
  → botão "Analisar com IA"                    recebe o POST
  → createServerFn (server-side)  ─POST──►   2. Normalizar entrada
                                                 valida campos obrigatórios
                                                 e tipa as métricas
                                             3. Analisar com IA (LLM chain)
                                                 monta o prompt com os dados
                                                 e pede resposta em JSON
                                             4. Ollama Cloud Chat Model
                                                 modelo usado pela chain
                                             5. Validar saída
                                                 garante as 8 chaves do
                                                 contrato e o enum de
                                                 confiança (baixo/medio/alto)
Resposta validada (Zod)  ◄──JSON──────────  6. Responder ao RevenuePilot
  → renderizada na tela de oportunidade
```

## Contrato de dados

**Entrada** (enviada pelo RevenuePilot):

```json
{
  "store_id": "string",
  "periodo": { "inicio": "YYYY-MM-DD", "fim": "YYYY-MM-DD", "comparacao": "string" },
  "tipo_de_oportunidade": "conversao | aquisicao | produto | retencao",
  "metricas": { "receita_atual": 0, "receita_anterior": 0, "...": "..." },
  "evidencias": [{ "label": "string", "value": "string" }],
  "pergunta_de_analise": "string"
}
```

**Saída** (devolvida pelo n8n e validada com Zod no RevenuePilot):

```json
{
  "diagnostico": "string",
  "hipotese": "string",
  "acao_recomendada": "string",
  "impacto_estimado": "string",
  "nivel_de_confianca": "baixo | medio | alto",
  "dados_utilizados": ["string"],
  "riscos": ["string"],
  "metricas_de_sucesso": ["string"]
}
```

O node **Normalizar entrada** rejeita o payload se faltar algum campo obrigatório. O node
**Validar saída** rejeita a resposta do modelo se faltar alguma das 8 chaves ou se
`nivel_de_confianca` não for um dos três valores aceitos — nesses casos o fluxo lança erro em vez
de repassar um JSON incompleto para o frontend.

## Como importar

1. Em uma instância n8n, vá em **Workflows → Import from File**.
2. Selecione [`revenuepilot-workflow.json`](revenuepilot-workflow.json).
3. Configure a credencial `Ollama Cloud - SAM` (ou troque o node **Ollama Cloud Chat Model** pelo
   provedor de LLM de sua preferência, mantendo o formato de saída em JSON).
4. Ative o workflow e copie a URL do node **Webhook RevenuePilot**.
5. No RevenuePilot, cole essa URL na tela `/configuracoes`. Sem essa configuração, o RevenuePilot
   roda normalmente em modo demonstração (análise simulada, rotulada como *demo*).

## Observações

- Nenhum dado pessoal trafega nesse contrato — apenas métricas agregadas e evidências textuais.
- O `impacto_estimado` é sempre uma estimativa com premissa declarada pelo modelo, nunca um fato.
- Esse workflow é específico do RevenuePilot; os outros projetos (SDR, Assistente Comercial, Sam)
  têm suas próprias integrações, documentadas em seus respectivos repositórios.
