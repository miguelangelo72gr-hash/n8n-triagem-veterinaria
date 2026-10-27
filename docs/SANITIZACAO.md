# Cópia pública sanitizada

Estes arquivos são versões de portfólio de dois workflows n8n. Os fluxos originais não foram alterados. As cópias estão desativadas e precisam de configuração própria para funcionar.

## Dados removidos ou substituídos

- Todos os vínculos de credenciais, incluindo IDs e nomes.
- Telefones, identidade da clínica, localidades e instâncias de produção.
- Domínio da API, caminho original do webhook e referências a workflows de produção.
- Metadados de proprietário, compartilhamento, versões, projeto e instalação.
- IDs originais dos nós e webhooks. Os arquivos públicos usam identificadores de demonstração.
- Dados fixados e estado persistido: os arquivos públicos não incluem histórico de execução ou conversas reais.

Os prompts, código e conexões foram preservados com os dados identificadores substituídos. Em duas condições `notEmpty`, o atributo `singleValue` foi removido para compatibilidade com a validação da ferramenta de importação. As consultas SQL e nomes de tabelas representam dependências do fluxo; este repositório não inclui o banco nem suas credenciais.

## Configuração para importação

1. Importe `workflows/sub-triagem-cerebro.json` e depois `workflows/triagem-noturna.json`.
2. No nó `Chamar SUB-triagem-cerebro`, substitua `CONFIGURE_SUBWORKFLOW_ID` pelo ID do subworkflow importado.
3. Configure credenciais próprias para OpenAI, Gemini (nós de teste), Postgres, Redis, Supabase e HTTP Header Auth da Evolution API.
4. No nó `Config`, substitua `https://evolution.example.invalid` e `CONFIGURE_PLANTONISTA`; ajuste a identificação das unidades. No nó `Avisa Suporte Fallback`, configure `CONFIGURE_SUPORTE` e `INSTANCIA_FALLBACK_EXEMPLO` com a integração de suporte.
5. Ajuste telefones e informações da clínica no nó `banco_info`, prepare as tabelas utilizadas e configure seu próprio tratamento global de erros, se aplicável.
6. Valide em ambiente isolado antes de ativar. As integrações enviam mensagens e gravam dados quando configuradas.

## Verificação desta publicação

Foram verificados os destinos das 106 conexões, a sintaxe dos 22 nós de código e a ausência dos identificadores e contatos originais nos arquivos públicos. Os dois clones foram salvos no n8n e relidos para confirmar que estão desativados e sem credenciais vinculadas.

Não foram executados testes de integração ou avaliações clínicas nesta preparação. Esta cópia documenta um projeto de automação; sua publicação não certifica decisões clínicas.
