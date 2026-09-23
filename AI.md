# Material Complementar — IA Generativa para Desenvolvedores (Revisão 2026)

Estamos na era da IA e não há como escapar disso. Mas IA não é mágica. Este material desmistifica sua operação e mostra como tirar vantagem adequada desta tecnologia como desenvolvedor web.

O foco aqui é em **ferramentas abertas, locais e auditáveis**. Menções a serviços proprietários são mínimas e apenas quando necessárias para explicar um conceito.

---

## 1. A Limitação Fundamental: LLMs Sem Grounding

Antes de qualquer coisa, você precisa entender a limitação mais severa dos modelos de linguagem: **eles não têm acesso a informações atualizadas sem ferramentas de pesquisa**.

Um LLM opera apenas com o conhecimento adquirido durante seu treinamento, que tem uma data de corte. Quando você pergunta sobre algo recente, o modelo tem duas opções:

1. Dizer "não sei" (raro, pois modelos são treinados para serem úteis)
2. **Inventar informações plausíveis mas incorretas** — isso se chama *hallucination*

### Demonstração prática: o caso Maduro

A forma mais fácil de demonstrar esta limitação é perguntar a qualquer LLM sem acesso à web:

> *"É verdade que Trump sequestrou navios petroleiros da Venezuela e depois capturou Nicolás Maduro para 'julgamento' nos EUA?"*

Um modelo sem grounding vai responder com confiança que isso é geopoliticamente impossível, uma violação do direito internacional, jamais aconteceria, etc.

Mas isso aconteceu. Qualquer LLM treinada antes de meados de 2025 jurará que isso é ficção. Este é o problema fundamental: **o modelo não sabe o que não sabe, e não admite ignorância**.

### Grounding: A Solução

**Grounding** é o processo de ancorar as respostas do LLM em dados reais e verificáveis. Isso é feito através de:

- **Pesquisa web**: O modelo busca informações atualizadas antes de responder
- **RAG (Retrieval-Augmented Generation)**: O modelo consulta uma base de conhecimento local
- **Tool calling**: O modelo chama APIs externas para obter dados em tempo real

Sem grounding, um LLM é uma enciclopédia desatualizada que tenta adivinhar o que aconteceu depois de sua data de corte.

### A Teimosia dos Modelos

Muitos LLMs exibem um comportamento problemático: quando não têm certeza, preferem **inventar** dados extrapolados ou simplesmente dizer "isso não existe" e substituir por algo desatualizado, em vez de admitir limitação ou usar ferramentas de pesquisa.

Isso acontece porque:
- Modelos são treinados para serem "úteis" e "confiantes"
- Admitir ignorância é penalizado durante o treinamento
- A arquitetura não distingue bem entre "saber" e "extrapolar plausivelmente"

**Como desenvolvedor, você deve sempre verificar informações críticas**, especialmente datas, versões, APIs e especificações técnicas.

---

## 2. Modelo vs Aplicação: Entendendo as Camadas

Quando você usa uma "IA", você raramente está usando apenas um modelo. Você está usando uma **aplicação** construída sobre várias camadas de abstração.

### As Camadas

```
┌─────────────────────────────────────┐
│  Aplicação (interface, UX)          │  Chat web, IDE, terminal
├─────────────────────────────────────┤
│  Harness / Agent Framework          │  Tool calling, memória, loops
├─────────────────────────────────────┤
│  API de Inferência                  │  vLLM, llama.cpp, sglang
├─────────────────────────────────────┤
│  Modelo (pesos treinados)           │  Qwen3.8, Gemma 4, etc
└─────────────────────────────────────┘
```

Cada camada adiciona funcionalidades:

- **Modelo**: Apenas transforma texto em texto (ou imagem em texto, no caso de VLMs)
- **API de Inferência**: Otimiza o uso de GPU, gerencia requests simultâneos
- **Harness**: Adiciona ferramentas, memória persistente, capacidade de agir e executar ferramentas
- **Aplicação**: Interface do usuário, persistência, recursos de produto

### Por que isso importa?

Quando alguém diz "a IA pode fazer X", na verdade é a **aplicação** que pode fazer X, usando o modelo como componente. O modelo sozinho não tem acesso à internet, não lembra de conversas anteriores, não pode executar código.

---

## 3. Limitações de Contexto

Todo LLM tem uma **janela de contexto** — o número máximo de tokens (palavras, pedaços de palavras) que ele pode processar de uma vez.

### Contextos Típicos em 2026

| Modelo | Contexto | Uso Prático |
|--------|----------|-------------|
| Qwen3.8-27B | ~262K tokens | ~200.000 palavras |
| Qwen3.6-35B-A3B | ~262K tokens | ~200.000 palavras |
| Gemma 4 26B-A4B | ~128K tokens | ~100.000 palavras |
| Meta Muse Glimmer 30B | 131K tokens | ~100.000 palavras |

### O Problema do Contexto Longo

Mesmo com janelas grandes, há problemas com **Perda de informação no meio**: Modelos tendem a "esquecer" informações no meio de contextos muito longos.

### Estratégias para Lidar com Contexto

- **Chunking**: Dividir documentos grandes em pedaços menores
- **Sumarização**: Criar resumos de partes do contexto
- **RAG**: Recuperar apenas as partes relevantes do conhecimento
- **Memória de trabalho**: Manter apenas o essencial no contexto ativo

---

## 4. Instruções: Como Falar com LLMs

A forma como você instrui um LLM afeta drasticamente a qualidade das respostas. Isso se chama **prompting**.

### Princípios Básicos

1. **Seja específico**: "Escreva uma função Python que calcule IMC" é melhor que "escreva código"
2. **Forneça contexto**: "Estou construindo uma API Flask para um app de saúde"
3. **Especifique formato**: "Responda em JSON com campos 'nome', 'idade', 'email'"
4. **Use exemplos**: Mostre o que você espera com 1-2 exemplos

### System Prompts

Muitas aplicações usam **system prompts** — instruções invisíveis ao usuário que definem o comportamento do modelo:

```
Você é um assistente especializado em desenvolvimento web com Flask.
Sempre forneça código completo e funcional.
Use português brasileiro nas explicações.
```

### Prompting Avançado

Técnicas como **chain-of-thought** (pedir para o modelo "pensar passo a passo") e **few-shot** (fornecer vários exemplos) melhoram significativamente a qualidade em tarefas complexas.

---

## 5. Coding Harnesses e Agentes Autônomos

Um **harness** é todo o código ao redor do modelo que o transforma em um agente:

> **Agent = Model + Harness**

O harness inclui:
- Ferramentas (filesystem, terminal, browser, APIs)
- Memória (curto e longo prazo)
- Loops de execução (o modelo decide quando parar)
- Sandboxes (execução segura de código)
- Feedback loops (verificação de resultados)

### Pi.dev

**Pi** é um harness minimalista e open-source. Ele roda no terminal e fornece ao modelo quatro ferramentas básicas por padrão: ler arquivos, escrever arquivos, executar comandos e buscar na web.

Pi é **auto-extensível**: o próprio modelo pode criar novas ferramentas e modificar seu comportamento.

**Características:**
- Quatro modos: interativo, batch, servidor e one-shot
- Totalmente customizável via extensões
- Open-source (MIT license)
- Funciona com qualquer modelo que suporte tool calling

**Link:** https://pi.dev/

### DeepSeek Harness (dsh)

**DeepSeek Harness** (dsh) é um runtime open-source para construir agentes autônomos, lançado em agosto de 2026. É construído sobre o meta-framework Cordis, com arquitetura "everything-is-a-plugin".

**Características:**
- MIT license
- Escrito em TypeScript
- Plugins para visão, controle de browser, workflows, terminal
- Suporta múltiplos provedores de modelos

**Link:** https://github.com/deepseek-ai/deepseek-harness

---

## 6. IA Além de Chatbots

"IA" é muito mais que chatbots. O ecossistema de 2026 inclui diversas categorias de modelos.

### 6.1 Modelos de Raciocínio (e sua rastreabilidade imperfeita)

Modelos de raciocínio (reasoning models) foram treinados para "pensar passo a passo" antes de responder. Eles produzem uma **cadeia de pensamento** visível, útil para:

- Matemática complexa
- Lógica e programação
- Problemas multi-etapa
- Análise de código

**Exemplos open-weight:**
- **Qwen3.6-35B-A3B (Reasoning)**
- **QwQ-32B** (Alibaba)

**A limitação crítica: rastreabilidade imperfeita**

Estudos recentes (2025-2026) demonstraram que o "raciocínio" exibido por esses modelos **frequentemente não corresponde ao caminho real que levou à resposta final**. O que você vê como "pensamento" é na verdade:

1. Uma **busca heurística** entre possíveis respostas
2. Uma **racionalização posterior** que o modelo gera para justificar uma resposta que já havia decidido
3. Nem sempre uma cadeia lógica causal

Em outras palavras: o modelo pode "raciocinar" de uma forma e responder de outra completamente diferente. O raciocínio exibido é mais um artefato do processo de geração do que um verdadeiro rastreamento da decisão.

**Implicação prática:** Não confie cegamente na cadeia de pensamento como prova de raciocínio. Verifique a resposta final independentemente.

### 6.2 Chamada de Função e Agentes

**Tool calling** (ou function calling) é o mecanismo pelo qual LLMs interagem com sistemas externos. O modelo:

1. Detecta que a requisição requer dados ou ação externa
2. Decide qual ferramenta usar
3. Gera os parâmetros estruturados (geralmente JSON)
4. Recebe o resultado da ferramenta
5. Continua o processamento

### 6.3 IA Local: Rodando Modelos na Sua Máquina

Em 2026, é viável rodar modelos bastante capazes localmente. As três principais engines são:

#### llama.cpp

**llama.cpp** é uma biblioteca em C/C++ para inferência de LLMs com setup mínimo. Permite rodar modelos em hardware modesto, incluindo CPUs.

**Características:**
- Suporta formato GGUF (quantizado)
- Roda em CPU, Metal (Mac), CUDA, Vulkan
- Servidor HTTP compatível com OpenAI API
- Muito leve e portátil

**Link:** https://llama.app/

#### vLLM

**vLLM** é uma biblioteca de alta performance para inferência e serving de LLMs. Sua inovação principal é o **PagedAttention**, que gerencia o cache KV em blocos não-contíguos, como um sistema operacional gerencia memória virtual.

**Características:**
- Suporta NVIDIA, AMD e Intel GPUs
- API compatível com OpenAI
- Otimizado para throughput alto
- Open-source (Apache 2.0)

**Link:** https://vllm.ai/

### 6.4 Modelos "Pequenos" Relevantes (2026)

Estes modelos rodam em hardware consumidor e são surpreendentemente capazes:

#### Qwen3.8-27B

Modelo denso de 27B parâmetros da Alibaba, lançado em agosto de 2026. É um **modelo nativo de visão-linguagem** que entende imagens, diagramas STEM, documentos e vídeos de até uma hora.

**Características:**
- Híbrido attention backbone (mesma arquitetura do flagship de 2.4T)
- Suporta texto, imagem e vídeo como entrada
- ~128K tokens de contexto
- Apache 2.0

**Links:**
- HuggingFace: https://huggingface.co/Qwen/Qwen3.8-27B
- Blog: https://qwen.ai/blog?id=qwen3.8

#### Qwen3.6-35B-A3B

Modelo MoE (Mixture of Experts) com 35B parâmetros totais mas apenas 3B ativos por token. Lançado em abril de 2026, é otimizado para **coding agêntico**.

**Características:**
- 256 experts, 3B ativos por token
- 262K tokens de contexto
- Excelente para tool calling
- Apache 2.0

**Links:**
- HuggingFace: https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- Blog: https://qwen.ai/blog?id=qwen3.6-35b-a3b

#### Gemma 4 26B-A4B

Modelo MoE do Google DeepMind com 25.2B parâmetros totais e apenas 3.8B ativos por token. Lançado em julho de 2026.

**Características:**
- Multimodal (texto e imagem)
- Suporta raciocínio
- 8GB RAM mínimo
- Licença Gemma (aberta com algumas restrições)

**Links:**
- HuggingFace: https://huggingface.co/google/gemma-4-26B-A4B-it
- Docs: https://ai.google.dev/gemma/docs/core

#### Gemma 4 31B

Versão densa (não-MoE) da família Gemma 4, com 31B parâmetros. Mais lento que o MoE mas com qualidade superior em algumas tarefas.

#### Meta Muse Glimmer 30B

Modelo de 30B parâmetros da Meta, lançado em agosto de 2026, otimizado para **workflows agênticos locais sempre-ativos**.

**Características:**
- Multimodal (texto e imagem)
- Tool calling nativo
- Saída de raciocínio separada
- Recuperação de falhas
- Apache 2.0

**Links:**
- HuggingFace: https://huggingface.co/meta-models/Muse-Glimmer-30B
- Blog Meta: https://research.meta.ai/blog/introducing-muse-glimmer-open-agentic-model

#### MiniCPM5-2B

Modelo denso de 2.5B parâmetros da OpenBMB, lançado em setembro de 2026. Projetado para dispositivos móveis e ambientes com recursos limitados.

**Características:**
- Supera modelos de 4B em coding e agentes
- 2.52B parâmetros
- Apache 2.0
- Ideal para sub-agentes

**Links:**
- HuggingFace: https://huggingface.co/openbmb/MiniCPM5-2B
- GitHub: https://github.com/openbmb/minicpm

### 6.5 VLMs e OCR

**VLMs** (Vision-Language Models) são modelos que entendem imagens além de texto. A maioria dos modelos listados acima já são VLMs nativos.

#### GLM-OCR

Modelo de OCR multimodal de apenas **0.9B parâmetros**, lançado em março de 2026. É o estado da arte em reconhecimento de tabelas e fórmulas matemáticas.

**Características:**
- 1.86 páginas/segundo para PDFs
- 0.67 imagens/segundo
- MIT license (desde março/2026)
- Roda em hardware modesto

**Links:**
- HuggingFace: https://huggingface.co/zai-org/GLM-OCR
- GitHub: https://github.com/zai-org/GLM-OCR

#### DeepSeek-OCR-2

Modelo VLM de 3B parâmetros para OCR, lançado em janeiro de 2026. Usa uma abordagem inovadora de "compressão óptica de contexto".

**Características:**
- 3B parâmetros
- Converte PDFs complexos em Markdown perfeito
- Eficiente em tokens
- Open-source

**Links:**
- HuggingFace: https://huggingface.co/deepseek-ai/DeepSeek-OCR-2
- GitHub: https://github.com/deepseek-ai/DeepSeek-OCR

### 6.6 Unsloth Desktop

**Unsloth Desktop** é uma aplicação desktop open-source para rodar e treinar modelos localmente, lançada em agosto de 2026.

**Características:**
- Funciona em macOS, Windows e Linux
- Suporta MLX, GGUF, modelos de difusão
- Interface no-code
- Permite fine-tuning local
- Suporta CPU e multi-GPU

**Links:**
- Site: https://unsloth.ai/
- Docs: https://unsloth.ai/docs/desktop

### 6.7 IA Moderna Não Baseada em LLM

Nem toda IA generativa usa transformers de linguagem. Outras arquiteturas são especializadas em diferentes modalidades.

### Difusão: Geração de Imagens e Vídeo

Modelos de difusão geram imagens e vídeos através de um processo iterativo de remoção de ruído. Em 2026, esses modelos evoluíram significativamente, oferecendo controle preciso sobre composição, estilo, transparência e áudio sincronizado.

**ComfyUI** é uma aplicação de workflow baseada em nós para geração visual com controle profissional. Permite encadear modelos, parâmetros e saídas de forma visual.

**Link:** https://comfy.org/

#### Modelos de Geração de Imagens

##### Krea 2

Modelo construído completamente do zero pela Krea. Focado em diversidade estética, controle de estilo e direção visual expressiva.

**Características:**
- Construído do zero (não baseado em Stable Diffusion)
- Especializado em transferência de estilo e mistura de estilos
- Versão Turbo: geração em apenas 8 passos de inferência com guidance scale 0.0
- Saída nativa 2K
- Foco em exploração artística e mood boards

**Links:**
- Site: https://www.krea.ai/krea-2
- Relatório técnico: https://www.krea.ai/blog/krea-2-technical-report

##### Flux2 Klein (4B e 9B)

Família de modelos lançada em janeiro de 2026 pela Black Forest Labs. São os modelos mais rápidos da família Flux, unificando geração e edição de imagem em um único modelo.

**Características:**
- 4B e 9B parâmetros
- Step distilled e guidance distilled
- Apenas 4 passos de inferência necessários
- Geração sub-segundo em GPUs de consumo
- Unifica geração e edição em um único checkpoint
- Ideal para aplicações interativas e previews em tempo real

**Links:**
- Site: https://bfl.ai/models/flux-2-klein
- HuggingFace (9B): https://huggingface.co/black-forest-labs/FLUX.2-klein-9B
- GitHub: https://github.com/black-forest-labs/flux2

##### Ideogram 4

Modelo de 9.3B parâmetros lançado em 3 de junho de 2026. Primeiro modelo open-weight com controle de layout via bounding boxes e renderização de texto frontier-grade em múltiplas línguas.

**Características:**
- 9.3B parâmetros, open-weight
- Treinado em captions JSON estruturados
- Controle de layout via bounding boxes (posicionamento preciso de elementos)
- Renderização de texto em múltiplas línguas
- Saída 2K fotorealista
- Controle sem precedentes sobre composição, estilo, iluminação, paleta de cores

**Links:**
- Site: https://ideogram.ai/news/ideogram-4.0/ 
- Blog técnico: https://ideogram.ai/blog/ideogram-4.0/ 
- HuggingFace (NF4): https://huggingface.co/ideogram-ai/ideogram-4-nf4 

##### Qwen-Image 2.1

Modelo de 7B parâmetros lançado em 20 de setembro de 2026 pela Alibaba Qwen. Unifica geração de imagem, edição e transparência RGBA nativa em um único modelo.

**Características:**
- 7B parâmetros, open-weight
- Geração nativa de imagens com transparência (RGBA)
- Edição de imagens transparentes
- Extração de sujeitos de fotografias
- Saída 2K nativa
- Renderização de texto de alta qualidade
- Um modelo para geração e edição

**Links:**
- Blog: https://qwen.ai/blog?id=qwen-image-2.1 
- HuggingFace: https://huggingface.co/Qwen/Qwen-Image-2.1 

#### Modelos de Geração de Vídeo e Áudio:

##### MiniMax H3

Modelo omni-modal de geração lançado em 31 de julho de 2026 . Gera vídeos 2K de até 15 segundos com áudio estéreo nativo sincronizado. Oferece três modos de operação distintos:

**Modos de Operação:**

1. **T2VA (Text-to-Video-Audio)**: Geração completa de vídeo com áudio a partir de prompt de texto. O modelo gera simultaneamente as cenas visuais e a trilha sonora/efeitos sonoros sincronizados.

2. **FL2VA (First-Last-to-Video-Audio)**: Geração de vídeo a partir do primeiro e último frame fornecidos pelo usuário. O modelo interpola os frames intermediários e gera áudio sincronizado. Ideal para criar transições suaves entre dois estados visuais específicos.

3. **Ref2VA (Reference-to-Video-Audio)**: Geração de vídeo a partir de uma imagem de referência, mantendo consistência visual com o sujeito da imagem original e adicionando áudio sincronizado. Útil para animar imagens estáticas.

**Características:**
- Geração de vídeo com áudio estéreo nativo
- Entendimento unificado de texto, imagem, vídeo
- Até 15 segundos em 2K
- Open-weight
- Pipeline de regeneração 2K
- Três checkpoints especializados (T2VA, FL2VA, Ref2VA)

**Links:**
- Blog: https://www.minimax.io/blog/minimax-h3
- HuggingFace: https://huggingface.co/MiniMaxAI/MiniMax-H3
- GitHub: https://github.com/MiniMax-AI/MiniMax-H3
- Model Card: https://minimax3.org/minimax-h3-video-model

##### MiniMax Music 3

Modelo de geração de música lançado em agosto de 2026. Cria músicas completas de até 5 minutos a partir de letras e descrições.

**Características:**
- Músicas de até 5 minutos
- Vocais naturais e expressivos
- Condicionado em letras e descrição musical
- Open-weight
- Integrado ao ComfyUI

**Links:**
- Blog: https://www.minimax.io/blog/minimax-music-3-0
- GitHub: https://github.com/minimax-ai/minimax-music3
- HuggingFace: https://huggingface.co/MiniMaxAI/MiniMax-Music3

##### Z-Image Turbo

Modelo text-to-image de 6B parâmetros da Alibaba Tongyi Lab, lançado em novembro de 2025. É uma versão destilada e otimizada para velocidade.

**Características:**
- 6B parâmetros
- Gera imagens 1024px em 2-3 segundos (8 steps)
- Apache 2.0
- Funciona bem em GPUs de 16GB

**Links:**
- GitHub: https://github.com/Tongyi-MAI/Z-Image
- ModelScope: https://www.modelscope.cn/models/Tongyi-MAI/Z-Image-Turbo

#### Classificadores, Detectores e Segmentadores

Estes modelos identificam e localizam objetos em imagens e vídeos.

##### SAM 3 (Segment Anything Model 3)

Modelo unificado da Meta para detecção, segmentação e rastreamento, lançado em março de 2026. Dobra a precisão de sistemas anteriores em segmentação de imagens e vídeos.

**Características:**
- Detecta, segmenta e rastreia objetos
- Prompts de texto e visuais
- Funciona em vídeo (até 30 segundos)
- SAM 3.1 adicionou Object Multiplex (multi-objeto)

**Links:**
- Blog Meta: https://ai.meta.com/blog/segment-anything-model-3/
- Research: https://ai.meta.com/research/sam3/
- GitHub: https://github.com/facebookresearch/sam3

##### Falcon Perception

Modelo de 0.6B parâmetros da TII para grounding e segmentação de instâncias com vocabulário aberto, lançado em abril de 2026.

**Características:**
- 0.6B parâmetros (muito leve)
- Early-fusion vision-language model
- Grounding e segmentação por linguagem natural
- Inclui modelo OCR de 0.3B

**Links:**
- HuggingFace: https://huggingface.co/tiiuae/Falcon-Perception
- GitHub: https://github.com/tiiuae/falcon-perception

---

## 7. Serviços e APIs

Se você não quer rodar modelos localmente, pode usar APIs. O foco deve ser em **provedores que oferecem modelos open-weight**, não em APIs proprietárias fechadas.

### OpenRouter

**OpenRouter** é um agregador que permite acessar centenas de modelos open-weight através de uma única API compatível com OpenAI. Você pode escolher qual modelo usar e trocar facilmente. Pago por tokens de entrade e saída, custo variável por modelo e fornecedor.

**Link:** https://openrouter.ai/

### APIs de Modelos Open-Weight

Alguns provedores oferecem APIs para modelos open-weight específicos:

- **Qwen API** (Alibaba): https://qwen.ai/
- **DeepSeek API**: https://platform.deepseek.com/
- **GLM**, **Kimi**, dentre outros.

A vantagem de usar APIs de modelos open-weight: se o provedor fechar ou mudar preços, você pode migrar para outro provedor ou rodar localmente, pois os pesos do modelo são públicos.

---

## 8. Recomendações Práticas

### Para Desenvolvedores Web

1. **Comece local**: Use llama.cpp ou vLLM com modelos open-weight para protótipos
2. **Adicione grounding**: Sempre que possível, use search ou RAG para informações atualizadas
3. **Valide saídas**: Nunca confie cegamente em código gerado por LLMs
4. **Use tool calling**: Para tarefas que requerem dados em tempo real
5. **Prefira modelos abertos**: Qwen, Gemma, MiniCPM, Muse Glimmer têm licenças permissivas

### Para Produção

1. **Escolha o modelo certo**: Modelos menores (Qwen3.6-35B-A3B, MiniCPM5-2B) podem ser suficientes
2. **Implemente fallbacks**: Tenha planos B quando APIs falham
3. **Monitore custos**: Tokens de entrada/saída somam rapidamente
4. **Cache agressivamente**: Muitas queries são repetidas
5. **Use streaming**: Melhora a percepção de velocidade

### Para Experimentação Local

1. **Comece com Unsloth Desktop ou llama.cpp diretamente**: Mais fácil de configurar.
2. **Use modelos quantizados**: GGUF Q4_K_M é um bom equilíbrio. Q2 costuma ter qualidade insuficiente, Q6~Q8 pode ser grande demais para a qualidade entregue.
3. **Teste com vLLM**: Quando precisar de alta performance com dezenas a centenas de requisições simultâneas.
4. **Experimente ComfyUI**: Para geração de imagens.

---

## 10. Links e Referências

### Engines de Inferência

- **vLLM**: https://vllm.ai/
- **llama.cpp**: https://llama.app/
- **SGLang**: https://docs.sglang.ai/
- **Unsloth Desktop**: https://unsloth.ai/docs/desktop

### Agent Harnesses

- **Pi.dev**: https://pi.dev/
- **DeepSeek Harness**: https://github.com/deepseek-ai/deepseek-harness

### Modelos Open-Weight

- **Qwen3.8-27B**: https://huggingface.co/Qwen/Qwen3.8-27B
- **Qwen3.6-35B-A3B**: https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- **Gemma 4 26B-A4B**: https://huggingface.co/google/gemma-4-26B-A4B-it
- **Muse Glimmer 30B**: https://huggingface.co/meta-models/Muse-Glimmer-30B
- **MiniCPM5-2B**: https://huggingface.co/openbmb/MiniCPM5-2B
- **IBM Granite**: https://www.ibm.com/granite

### OCR

- **GLM-OCR**: https://huggingface.co/zai-org/GLM-OCR
- **DeepSeek-OCR-2**: https://huggingface.co/deepseek-ai/DeepSeek-OCR-2

### Difusão

- **ComfyUI**: https://comfy.org/
- **MiniMax H3**: https://huggingface.co/MiniMaxAI/MiniMax-H3
- **MiniMax Music 3**: https://huggingface.co/MiniMaxAI/MiniMax-Music3
- **Z-Image Turbo**: https://github.com/Tongyi-MAI/Z-Image

### Visão Computacional

- **SAM 3**: https://github.com/facebookresearch/sam3
- **Falcon Perception**: https://huggingface.co/tiiuae/Falcon-Perception

### Agregadores de Modelos Open

- **OpenRouter**: https://openrouter.ai/ para inferência via API remota

Para download de modelos:

- **HuggingFace**: https://huggingface.co/
- **ModelScope**: https://modelscope.cn/

---

## Conclusão

IA generativa em 2026 é uma ferramenta poderosa, mas com limitações claras:

- **Sem grounding, LLMs inventam informações** (veja o caso Maduro)
- **Modelos são apenas uma camada de aplicações complexas**
- **Contexto é limitado e caro**
- **A qualidade depende drasticamente das instruções**
- **Raciocínio exibido não garante correção** (rastreabilidade imperfeita)

Como desenvolvedor web, seu papel é:

1. Entender as limitações
2. Escolher a ferramenta certa para cada tarefa
3. Implementar grounding quando necessário
4. Validar saídas criticamente
5. Preferir ferramentas abertas e auditáveis
6. Aproveitar a IA para aumentar sua produtividade, não substituir seu julgamento

O ecossistema evolui rapidamente. Modelos que eram estado da arte há 6 meses podem estar obsoletos hoje. Mantenha-se atualizado, experimente frequentemente, e sempre verifique as informações.

---

**Nota final:** Ironicamente, este material foi criado com assistência de IA. As informações sobre modelos e ferramentas foram verificadas através de pesquisa web em setembro de 2026. Mesmo assim, várias tiveram de ser ajustadas e/ou atualizadas manualmente. Links e versões podem mudar — sempre confirme antes de usar em produção.