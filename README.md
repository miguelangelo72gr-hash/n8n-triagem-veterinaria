# Triagem veterinária noturna com n8n

Esse projeto nasceu de um problema na rotina do gerente de uma clínica veterinária: ele recebia mensagens e ligações de madrugada no número de emergência, mas muitas eram sobre assuntos que podiam esperar até o horário de atendimento. Isso interrompia o sono dele e afetava sua disposição no dia seguinte.

Construí a automação para fazer uma primeira triagem dessas mensagens, responder às dúvidas de rotina e acionar o plantonista quando identificasse uma possível emergência.

## Arquitetura e funcionamento das IAs

Dividi o projeto em dois workflows. O principal cuida da operação: entrada pelo WhatsApp, filtros, horário de atendimento, transcrição de áudio, agrupamento de mensagens, armazenamento e envio. O subworkflow concentra a interpretação das mensagens e a lógica de triagem.

Essa separação permite mudar prompts, testar modelos e ajustar a classificação sem precisar alterar toda a integração com o WhatsApp.

**Por que não usei um AI Agent**

Não usei o nó AI Agent em nenhum ponto. Cada chamada de IA tem uma tarefa definida, recebe o contexto necessário para aquela tarefa e devolve um resultado que o workflow processa.

Essa decisão foi inspirada nas discussões do Andrej Karpathy sobre autonomia parcial: escolher quanto controle entregar ao modelo e criar formas de conferir o que ele faz. A aplicação dessa ideia neste projeto foi uma decisão minha de engenharia. [Referência: Software Is Changing (Again)](https://www.youtube.com/watch?v=LCEmiRjPEtQ)

Horários, consultas ao banco, contadores, novas tentativas em caso de falha e envio de alertas já têm regras claras. Deixei essas tarefas com os nós do n8n e com JavaScript. As IAs entram onde preciso interpretar linguagem ou escrever uma resposta considerando o relato do cliente.

O modelo não escolhe ferramentas, não executa SQL e não decide sozinho quais ações realizar. Os caminhos possíveis já estão definidos nas conexões do workflow.

**1. Decidir se a conversa precisa de triagem**

A primeira IA é o `Gate Contextual`. Ela recebe a mensagem atual e o histórico recente para decidir entre três saídas: `triagem`, `nao_intervir` e `incerto`.

Essa etapa resolve um problema de contexto. Um “obrigado” depois de uma mensagem da equipe pode ser apenas o encerramento da conversa. Já “obrigado, mas ele começou a vomitar” precisa continuar na triagem.

A resposta vem em JSON, com decisão, confiança e motivo. Depois, um nó de código confere o resultado. O silêncio só é permitido quando a decisão é `nao_intervir`, a confiança informada é alta e as regras do código não encontram sinais clínicos ou pedidos de ajuda.

Se a saída vier inválida ou houver dúvida, o fluxo continua para a triagem. A confiança informada pelo modelo funciona como um dos critérios de controle; ela não é tratada como uma medida comprovada da chance de ele estar certo na avaliação clínica.

**2. Classificar o relato sem escrever a resposta**

O `Classificador` recebe a mensagem e o histórico, mas sua função termina na classificação. Ele devolve um JSON com campos definidos por schema:

- `classificacao`: emergência, não emergência ou ambíguo.
- `criterio_disparado`: o sinal que levou à escalada.
- `precisa_mais_info`: se falta contexto para continuar.
- `intent`: se a mensagem é um relato clínico, saudação, pergunta de horário ou assunto fora do escopo.
- `confianca` e `tem_pergunta_info`: informações usadas na etapa seguinte.

Separar classificação e escrita facilita entender o comportamento. Consigo verificar qual decisão o modelo tomou antes de analisar como a resposta foi escrita.

O schema limita o formato da saída, mas não garante que a interpretação esteja correta. Por isso, o resultado ainda passa pelo nó `Pos-Regra`.

**3. Aplicar as regras de encaminhamento em código**

O `Pos-Regra` confere a classificação, consulta o contador de perguntas e define o próximo caminho.

Se faltarem informações, o fluxo pode fazer uma pergunta complementar. O contador fica no Redis e limita essa coleta a três rodadas. Se o caso continuar sem uma conclusão depois disso, ele é escalado, evitando uma conversa que fica perguntando indefinidamente.

Uma classificação inválida também segue para escalada. Já uma pergunta apenas sobre horário pode receber um texto montado pelo código, sem outra chamada de IA para escrever a resposta.

**4. Escrever dentro da rota escolhida**

Usei dois compositores com instruções diferentes.

O `Compositor Nao-Emergencia` recebe um modo definido pelo código: perguntar, orientar, cumprimentar ou redirecionar um assunto fora do escopo. No modo de coleta, o prompt pede uma pergunta por vez e usa o histórico para evitar repetição.

O `Compositor Emergencia` recebe a mensagem, o critério identificado e a indicação de que também existe uma pergunta sobre informações da clínica. Seu prompt orienta uma resposta curta de acolhimento e encaminhamento, com limites sobre diagnóstico, medicação e promessas de melhora.

Também controlo o contexto enviado. Os dados de horário, telefone e endereço só entram no compositor de não emergência quando as condições do fluxo pedem. Quando necessário, o horário oficial é acrescentado pelo código, evitando deixar o modelo escrever esse dado por conta própria.

**Custo e controle**

Os nós de decisão usam GPT-4.1-mini com temperatura zero e saída estruturada. Os compositores usam temperatura baixa e limites menores de saída. Há também nós separados com Gemini para testes de composição.

A economia vem de evitar processamento desnecessário: mensagens agrupadas antes da triagem, filtros antes das chamadas e respostas fixas quando o conteúdo já é conhecido. Quando o gate decide pelo silêncio, as etapas seguintes não são chamadas.

Essa divisão também tem um custo: uma mensagem pode passar pelo gate, pelo classificador e por um compositor. A escolha foi aceitar essas chamadas curtas e com funções bem definidas para conseguir acompanhar cada decisão, ajustar uma etapa por vez e manter o controle das ações no workflow.

## Como testei e ajustei as respostas

Também usei IA para ajudar a criar os evals do projeto, seguindo um framework que desenvolvi para esse tipo de implementação. A ideia era ter um processo para conferir as respostas e acompanhar os ajustes, sem depender só de testar algumas mensagens e achar que estavam boas.

**Primeiro, defini o que seria uma boa resposta**

Como parte do framework, a IA montou tabelas com perguntas e mensagens de exemplo. Para cada caso, eu indicava quais respostas eram desejáveis e quais não eram.

Essa etapa serviu para deixar os critérios claros e conferir se a IA que avaliaria as respostas estava julgando de acordo com o que eu esperava.

**Depois, conferi o julgamento da própria IA**

Usei uma LLM como avaliadora das respostas, no formato conhecido como *LLM as a judge*. Mas, antes de usar os relatórios para orientar os ajustes, precisava conferir se esse juiz estava avaliando direito.

As tabelas serviam de referência para comparar o julgamento da IA com os critérios que eu tinha definido. Se ela aprovasse uma resposta que eu considerava inadequada, por exemplo, o processo de avaliação também precisava de ajuste.

Isso era importante porque um relatório positivo teria pouco valor se o próprio avaliador estivesse usando critérios errados.

**Com os critérios definidos, vieram os testes do workflow**

A IA criou vários cenários e conversas de teste, executou o workflow e avaliou as respostas seguindo esse processo. Depois, reuniu os resultados em um relatório para eu revisar.

A partir dos resultados, fui ajustando o comportamento do fluxo e as instruções dos modelos. O processo seguiu esse ciclo: testar, avaliar, revisar os problemas e ajustar.

Os evals ajudaram a encontrar comportamentos que precisavam de correção e deram uma referência mais clara para trabalhar. O julgamento da IA entrou como apoio nessa revisão, junto dos critérios que eu tinha definido.

## Uso em produção

O projeto continua em produção na clínica, e o retorno do gerente foi bastante positivo.

Com o uso real, apareceram erros e situações que não tinham surgido nos testes. Fui corrigindo esses problemas e ajustando o fluxo conforme eles apareciam, melhorando o sistema a partir do que acontecia no atendimento.

Os evals ajudaram a preparar o workflow, e o acompanhamento em produção trouxe novos casos para continuar os ajustes.
