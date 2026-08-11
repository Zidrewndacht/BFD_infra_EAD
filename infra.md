# Aula: Infraestrutura para Web Services — Do Processo ao Cloud

**Pré-requisitos:** Flask, Gunicorn, noções de hardware (CPU, RAM, discos), HTTP.
**Objetivo:** Entender onde seu código Python realmente roda, quais são as opções de hospedagem disponíveis no mercado, e como escolher entre elas com base em números reais de tráfego e custo.

> **Nota sobre preços:** Os valores nesta aula foram verificados em **agosto de 2026**. Preços de nuvem e hospedagem mudam com frequência. Os valores marcados com ✅ foram confirmados na data de verificação; os marcados com ⚠️ são estimativas baseadas em dados recentes e devem ser conferidos nas páginas de pricing indicadas em cada tabela. Cotações usadas: EUR ≈ R$ 6,20 · USD ≈ R$ 5,70.

---

## 0: O que roda abaixo do seu Flask

Antes de falar de onde hospedar, precisamos entender **o que** estamos hospedando. Até aqui, você rodou Flask com `app.run(debug=True)`. Isso é inaceitável em produção por dois motivos:

1. O servidor de desenvolvimento do Werkzeug é **single-threaded** e não foi feito para concorrência.
2. `debug=True` expõe um shell Python remoto para qualquer um que force um erro 500.

### 0.1 O modelo real: WSGI + Application Server

Em produção, o Flask é apenas um **aplicativo WSGI** — uma função Python que recebe um request e devolve um response. Quem efetivamente **escuta a rede, gerencia conexões e distribui trabalho** é um servidor de aplicação dedicado:

```
Internet
   │
   ▼
┌─────────────────────┐
│  Nginx / Caddy      │  ← Reverse proxy (TLS, static files, rate limiting)
│  (porta 80/443)     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Gunicorn / uWSGI   │  ← Application server WSGI
│  (porta 8000)       │
│  ┌────┐┌────┐┌────┐ │
│  │ W1 ││ W2 ││ W3 │ │  ← Worker processes (cada um é uma instância do Flask)
│  └────┘└────┘└────┘ │
└─────────────────────┘
```

**Gunicorn** (Green Unicorn) é o padrão da indústria Python. Ele é um **master process** que gerencia N **worker processes**, cada um carregando uma cópia completa da sua aplicação Flask na memória.

```bash
# Comando típico de produção:
gunicorn --workers 4 --bind 0.0.0.0:8000 --timeout 120 livepoll:create_app()
```

### 0.2 A regra prática de dimensionamento de workers

| Métrica | Fórmula | Exemplo |
| :--- | :--- | :--- |
| **CPU-bound** (cálculos pesados, ML) | `2 × CPU_cores + 1` | 4 cores → 9 workers |
| **I/O-bound** (banco, APIs externas, Flask típico) | `2 × CPU_cores + 1` **até** `4 × CPU_cores` | 2 cores → 5 a 9 workers |

**Por que essa fórmula?** Cada worker Python tem o GIL (Global Interpreter Lock), então um worker usa **uma CPU por vez**. Mas enquanto um worker espera I/O (consulta SQL, resposta de API), ele libera a CPU para outro worker. Ter mais workers que CPUs faz sentido para I/O-bound.

### 0.3 Memória: o verdadeiro limitador

Cada worker Flask carrega:
- O interpretador Python (~30-50 MB)
- Seu código + bibliotecas importadas (Flask, SQLAlchemy, etc.) (~50-150 MB)
- Dados em cache, conexões de BD, estado da aplicação

**Na prática:** cada worker Flask consome entre **100 e 300 MB de RAM**. Em uma VPS de 1 GB de RAM, você consegue rodar no máximo **3 a 4 workers** confortavelmente.

---

## 1: A evolução da infraestrutura — Do ferro ao container

### 1.1 Linha do tempo da abstração

```
1995: Bare Metal       → Você compra o servidor físico, coloca no rack
2005: VPS              → Você aluga uma fatia de um servidor (virtualização)
2012: Cloud IaaS       → Você aluga VMs sob demanda, por hora (AWS EC2)
2013: PaaS             → Você sobe o código, a plataforma cuida do resto (Heroku)
2014: Containers       → Você empacota o app + dependências (Docker)
2015: Orquestração     → Você gerencia 500 containers (Kubernetes)
2018: Serverless       → Você sobe uma função, paga por milissegundo (Lambda)
```

Nenhuma dessas opções **substitui** as anteriores. Cada uma resolve um problema diferente. Vamos entender cada uma.

---

## 2: Bare Metal e Virtualização

### 2.1 Bare Metal (Servidor Físico)

Você compra um servidor enterprise (Dell PowerEdge, HPE ProLiant), coloca num data center (colocation), paga energia, refrigeração, link de internet redundante e um técnico para trocar disco queimado às 3h da manhã.

**Custo estimado:** R$ 40.000 (configuração básica) a R$ 200.000+ (com GPUs, muita RAM, storage NVMe). Para um servidor web de médio porte, espere algo na faixa de **R$ 60.000-120.000**. ⚠️ Estimativa — varia muito com cotação do dólar e configuração.

**Quando faz sentido em 2026:**
- Empresas com requisitos regulatórios específicos (bancos, governo)
- Cargas de trabalho de GPU intensivas (treinamento de LLMs)
- Operações com >R$ 50k/mês em cloud que querem cortar custos

**Para 99,9% dos casos: NÃO.**

### 2.2 VPS (Virtual Private Server)

Uma empresa (Hetzner, DigitalOcean, Linode, Contabo) compra servidores físicos potentes, instala um hypervisor (Proxmox, KVM, VMware) e "fateia" o hardware em **máquinas virtuais** isoladas. Você aluga uma fatia por um valor fixo mensal.

**Características reais:**
- Você recebe root SSH num Linux (geralmente Ubuntu/Debian)
- Os recursos alocados (CPU, RAM) são **dedicados** ou **compartilhados** (leia o contrato!)
- Você é 100% responsável por: patches de segurança, firewall, backups, atualizações
- O IP é fixo e você controla DNS

**Preços em agosto de 2026:**

| Provedor | Configuração | Preço | Página de pricing | Status |
| :--- | :--- | :--- | :--- | :--- |
| [Hetzner Cloud](https://www.hetzner.com/cloud/) (CX23) | 2 vCPU, 4 GB RAM, 40 GB NVMe | **€ 5,49/mês** (~R$ 34) | [hetzner.com/cloud](https://www.hetzner.com/cloud/) | ✅ Verificado |
| [Hetzner Cloud](https://www.hetzner.com/cloud/cost-optimized/) (Cost-Optimized) | 2 vCPU, 4 GB RAM, 40 GB NVMe | **€ 5,99/mês** (~R$ 37) | [hetzner.com/cloud/cost-optimized](https://www.hetzner.com/cloud/cost-optimized/) | ✅ Verificado |
| [Hetzner Cloud](https://www.hetzner.com/cloud/regular-performance/) (próximo tier) | 2 vCPU, 8 GB RAM, 80 GB NVMe | **€ 8,99/mês** (~R$ 56) | [hetzner.com/cloud/regular-performance](https://www.hetzner.com/cloud/regular-performance/) | ✅ Verificado |
| [Contabo](https://contabo.com/en/cloud-vps/) (Cloud VPS 10) | 4 vCPU, 8 GB RAM, 200 GB SSD | **~€ 7,50/mês** (~R$ 47) | [contabo.com/en/cloud-vps](https://contabo.com/en/cloud-vps/) | ⚠️ Estimativa |
| [DigitalOcean](https://www.digitalocean.com/pricing/droplets) (Basic Droplet) | 2 vCPU, 4 GB RAM, 80 GB SSD | **~US$ 24/mês** (~R$ 137) | [digitalocean.com/pricing/droplets](https://www.digitalocean.com/pricing/droplets) | ⚠️ Estimativa |
| [Linode (Akamai)](https://www.linode.com/pricing/) (Shared 4GB) | 2 vCPU, 4 GB RAM, 80 GB SSD | **~US$ 24/mês** (~R$ 137) | [linode.com/pricing](https://www.linode.com/pricing/) | ⚠️ Estimativa |
| [Oracle Cloud Free](https://www.oracle.com/cloud/free/) (Ampere A1) | 4 OCPU ARM, 24 GB RAM | **Grátis** | [oracle.com/cloud/free](https://www.oracle.com/cloud/free/) | ⚠️ Estimativa (ver nota) |

> **Atenção sobre a Hetzner:** os preços acima refletem os **reajustes de 2026**. A Hetzner fez múltiplos aumentos ao longo do ano, com o maior em **15 de junho de 2026**. O antigo plano CX22 (€ 3,79/mês) foi substituído pelo CX23 (€ 5,49/mês). Algumas instâncias dedicadas tiveram aumento de até 3,1x. Se você encontrar material antigo citando preços menores da Hetzner, está desatualizado.

> **Atenção sobre a Contabo:** historicamente se posiciona como "mais recursos por menos dinheiro", mas tem reputação mista em performance (noisy neighbors mais agressivos, I/O de disco inconsistente). Vale testar antes de comprometer.

> **Atenção sobre a Oracle Cloud Free:** o free tier (4 OCPU ARM + 24 GB RAM) é real e generoso, mas há relatos consistentes de dificuldade em criar conta (cartão recusado), instâncias sendo reclaimadas por inatividade e capacidade indisponível em certas regiões. Não é confiável para produção.

**Atenção ao "vCPU":** na maioria das VPS baratas, 1 vCPU é uma **fração de tempo** de um core físico (ex: 25% de um core Xeon). Você pode sofrer com **noisy neighbors** — outros clientes no mesmo servidor consumindo CPU e deixando seu app lento.

### 2.3 O setup clássico de uma VPS

Para colocar seu Flask no ar numa VPS, você faz (manualmente):

```bash
# 1. Atualiza o sistema
apt update && apt upgrade -y

# 2. Instala Python, Nginx, supervisor
apt install python3.12 python3-venv nginx supervisor -y

# 3. Cria usuário sem privilégios de root
adduser --disabled-password flaskapp

# 4. Clona o código (via git ou rsync)
su - flaskapp
git clone https://github.com/seu-usuario/seu-app.git
cd seu-app && python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 5. Configura systemd para rodar gunicorn
# Cria /etc/systemd/system/flaskapp.service

# 6. Configura Nginx como reverse proxy
# Cria /etc/nginx/sites-available/flaskapp

# 7. Configura TLS com Let's Encrypt
apt install certbot python3-certbot-nginx -y
certbot --nginx -d seudominio.com
```

**Prós:**
- Controle total
- Custo baixo e previsível
- Aprendizado real de Linux/administração

**Contras:**
- Você **é** o sysadmin. Servidor caiu às 3h da manhã de sábado? Acorda.
- Atualizar Python, renovar certificado SSL, configurar firewall — tudo na sua mão.
- Escalar horizontalmente (adicionar mais servidores) é complexo.

---

## 3: Cloud Computing (IaaS) — AWS, Azure, GCP

### 3.1 O que é, realmente

Cloud é uma empresa (Amazon, Google, Microsoft) que tem **milhões de servidores** em data centers ao redor do mundo e os aluga **por segundo**, via API. A grande inovação não é a tecnologia — é o **modelo de negócio**:

- Paga apenas pelo que usar
- Provisiona recursos em segundos (via API ou console web)
- Escala automaticamente (com configuração)
- Oferece centenas de serviços gerenciados (banco, fila, CDN, ML)

### 3.2 Os três grandes

| Provedor | Market share (2025) | Ponto forte | Pricing |
| :--- | :--- | :--- | :--- |
| **AWS** (Amazon) | ~31% | Ecossistema maior, mais maduro | [aws.amazon.com/pricing](https://aws.amazon.com/pricing/) |
| **Azure** (Microsoft) | ~25% | Integração com Windows/Active Directory | [azure.microsoft.com/pricing](https://azure.microsoft.com/en-us/pricing) |
| **GCP** (Google) | ~11% | Kubernetes, BigQuery, preços competitivos | [cloud.google.com/pricing](https://cloud.google.com/pricing) |

### 3.3 Serviços essenciais (mapeamento)

| Conceito | AWS | Azure | GCP |
| :--- | :--- | :--- | :--- |
| VM (VPS na cloud) | EC2 | Virtual Machines | Compute Engine |
| Armazenamento de objetos | S3 | Blob Storage | Cloud Storage |
| Banco relacional gerenciado | RDS (Postgres, MySQL) | Azure SQL / Database for PostgreSQL | Cloud SQL |
| Banco NoSQL | DynamoDB | Cosmos DB | Firestore / Bigtable |
| CDN | CloudFront | Azure CDN | Cloud CDN |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Load Balancer | ELB/ALB | Application Gateway | Cloud Load Balancing |
| Kubernetes gerenciado | EKS | AKS | GKE |
| Funções serverless | Lambda | Azure Functions | Cloud Functions |

### 3.4 O lado sombrio do Cloud: a conta

Cloud pode gerar **contas surpresa catastróficas** se não houver alertas de billing configurados:

- **Firebase US$ 30.356 em 72h:** campanha de crowdfunding colombiana viralizou e uma única linha de código (`this.loadPayments()`) fazia cada visita ler 16.000 documentos do Firestore, gerando 40 bilhões de requests em 48h. [Relato original](https://medium.com/hackernoon/how-we-spent-30k-usd-in-firebase-in-less-than-72-hours-307490bd24d)

- **AWS Lambda US$ 47.283 em um fim de semana:** função Lambda processava imagens e salvava de volta no **mesmo bucket S3** que a disparava, criando loop recursivo que rodou por 38 horas. [Relato original](https://devrimozcay.medium.com/our-aws-bill-hit-47-000-in-one-weekend-because-of-a-single-line-of-code-32f6b9cb0cf1)

- **AWS US$ 2.847 em 2 dias:** dev júnior montou projeto de aprendizado num fim de semana, esqueceu instâncias caras rodando, storage crescendo e logs explodindo. [Relato original](https://medium.com/aws-in-plain-english/i-accidentally-spent-3-000-on-aws-in-one-weekend-here-are-the-7-mistakes-that-caused-it-b4adc717ccbb)

- **AWS US$ 2.657 por um arquivo em S3 + CDN:** imagem de VM de 13,7 GB hospedada publicamente; CDN não fazia cache de arquivos >512MB e cada download ia direto à origem. [Relato original](https://chrisshort.net/the-aws-bill-heard-around-the-world/)

**O que todos têm em comum:** nenhum foi ataque sofisticado. Todos foram erros de configuração ou falta de entendimento do modelo de billing. E todos teriam sido detectados em minutos se houvesse **alertas de billing** configurados: 

- **AWS:** AWS Budgets com alertas em US$ 10, US$ 50, US$ 100 (email + SMS)
- **GCP:** Budget Alerts no Billing
- **Azure:** Cost Management + Budgets
- **Firebase:** Alertas de uso no console

**A regra da cloud:**
> Cloud é mais barato que ter servidor próprio quando você precisa de **elasticidade**. Cloud é mais caro quando você tem carga **constante e previsível**.

### 3.5 Quando usar cloud de verdade

- Sua empresa tem orçamento e equipe dedicada (DevOps/SRE)
- Você precisa de serviços gerenciados (RDS, SQS, S3) que não existem em VPS
- Sua carga varia muito (10x entre dia e noite, picos sazonais)
- Você precisa de presença geográfica global (latência baixa em vários continentes)
- Você está construindo um SaaS e quer vender para outras empresas (elas confiam em AWS/Azure)

---

## 4: PaaS (Platform as a Service) — O meio-termo sensato

### 4.1 O conceito

Você faz `git push` e a plataforma:
- Detecta que é uma app Python
- Instala dependências
- Constrói a imagem
- Provisiona um container
- Configura TLS
- Expõe via HTTPS
- Faz log de tudo

Você não vê servidor, não configura nginx, não renova SSL.

### 4.2 Principais players (e a realidade de cada um)

| Plataforma | Preço mínimo | Página de pricing | Prós | Contras | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| [Render](https://render.com/) | **~US$ 7/mês** (Starter) | [render.com/pricing](https://render.com/pricing) | Simples, bom free tier de testes, suporta Docker | Dorme após 15 min de inatividade no plano free | ⚠️ Estimativa |
| [Railway](https://railway.app/) | **~US$ 5/mês** (Hobby) + usage | [railway.app/pricing](https://railway.app/pricing) | UI excelente, deploys rápidos | Preço sobe rápido, limites rígidos | ⚠️ Estimativa |
| [Fly.io](https://fly.io/) | **~US$ 5/mês** (shared-cpu-1x) | [fly.io/docs/about/pricing](https://fly.io/docs/about/pricing/) | Roda em edge (baixa latência global), Docker nativo | Curva de aprendizado maior, billing confuso | ⚠️ Estimativa |
| [Heroku](https://www.heroku.com/) | **~US$ 5/mês** (Basic) | [heroku.com/pricing](https://www.heroku.com/pricing) | Pioneiro, documentação excelente | **Caro** para o que entrega. O plano "Eco" foi descontinuado | ⚠️ Estimativa |
| [PythonAnywhere](https://www.pythonanywhere.com/) | **~US$ 5/mês** (Hacker) | [pythonanywhere.com/pricing](https://www.pythonanywhere.com/pricing/) | Ótimo para iniciantes, Jupyter notebooks | Só Python, limitado a sites "pequenos" | ⚠️ Estimativa |
| [Vercel](https://vercel.com/) | **Free tier** (Pro: ~US$ 20/mês) | [vercel.com/pricing](https://vercel.com/pricing) | Excelente para Next.js/frontend | Não é pensado para Flask/Python backend | ⚠️ Estimativa |

> **Nota sobre o Heroku:** o plano **Eco Dyno** (US$ 5/mês, pool de dynos que dormem) foi **descontinuado**. Se você encontrar material antigo mencionando "Heroku Eco", está desatualizado. O plano de entrada atual é o **Basic**.

> **Nota sobre a Fly.io:** o free tier original foi significativamente reduzido ou removido entre 2024 e 2025. O modelo agora é predominantemente pay-as-you-go. Verifique a página de pricing antes de assumir custos.

### 4.3 O trade-off do PaaS

**Você terceiriza o sofrimento da infraestrutura e paga por isso.**

- Uma VPS de 2 vCPU/4 GB na Hetzner custa **~R$ 37/mês**
- O mesmo recurso no Render custa **~R$ 43/mês** 
- No Heroku, **~R$ 145/mês** 

**A pergunta certa:** O seu tempo (como desenvolvedor) vale mais que a diferença? Se você gasta 4 horas por mês configurando e mantendo servidor, e sua hora vale R$ 100, PaaS compensa. Se você gosta de mexer com Linux e quer aprender, VPS compensa.

### 4.4 Deploy real no Render (exemplo)

1. Crie um `render.yaml` no seu projeto Flask:

```yaml
services:
  - type: web
    name: meu-flask-app
    runtime: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "gunicorn livepoll:create_app()"
    envVars:
      - key: SECRET_KEY
        generateValue: true
      - key: DATABASE_URL
        fromDatabase:
          name: meu-db
          property: connectionString
```

2. Conecte o repositório GitHub
3. Deploy automático a cada push no `main`

**Tempo do commit ao HTTPS em produção: ~3 minutos.**

---

## 5: Containers e Docker

### 5.1 O problema real que Docker resolve

Não é "funciona na minha máquina" — isso é consequência. O problema é:

> **Reprodutibilidade de ambiente em escala.**

Você tem 15 desenvolvedores no time. Cada um com uma versão diferente de Python, Postgres, Redis. O staging tem outra configuração. Produção tem outra. Bugs surgem "misteriosamente" em produção. Docker resolve isso empacotando **tudo** (SO, libs, Python, código) numa imagem que roda idêntica em qualquer lugar.

### 5.2 O que Docker realmente é

- **Imagem:** Um arquivo imutável com sistema de arquivos + instruções. É o "CD de instalação" do seu app.
- **Container:** Uma instância rodando de uma imagem. É um processo Linux isolado via **namespaces** e **cgroups** (não é VM!).
- **Registry:** Repositório de imagens (Docker Hub, GitHub Container Registry, AWS ECR).

### 5.3 Dockerfile realista para Flask

```dockerfile
# Etapa 1: Builder (instala dependências, compila o que for necessário)
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .

# Compila wheels se possível (mais rápido de instalar)
RUN pip install --user --no-cache-dir -r requirements.txt

# Etapa 2: Runtime (imagem final, menor)
FROM python:3.12-slim

# Usuário não-root (segurança básica)
RUN useradd -m appuser && chown appuser:appuser /app
USER appuser
WORKDIR /app

# Copia apenas o que foi instalado no builder
COPY --from=builder /root/.local /home/appuser/.local
ENV PATH=/home/appuser/.local/bin:$PATH

# Copia código da aplicação
COPY --chown=appuser:appuser . .

# Não roda como root
EXPOSE 8000

# Gunicorn com workers adequados
CMD ["gunicorn", \
     "--workers", "3", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "livepoll:create_app()"]
```

**Tamanho da imagem final:** ~150-250 MB (vs 1-2 GB se você usar `python:3.12` full)

### 5.4 Docker Compose para desenvolvimento local

Quando seu app tem múltiplos serviços (Flask + Postgres + Redis), use Compose:

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/app
      - FLASK_DEBUG=1

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=app
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up       # Sobe tudo
docker compose down     # Derruba tudo
docker compose logs -f  # Vê logs
```

---

## 6: Serverless — O hype e a realidade

### 6.1 O conceito

Você escreve uma função Python. A cloud:
- Provisiona um container quando alguém chama
- Executa a função
- Destrói o container
- Cobra apenas pelos milissegundos executados

**Sem servidor para gerenciar. Sem custo quando não há tráfego.**

### 6.2 Principais ofertas

| Provedor | Serviço | Tempo máximo | Free tier | Pricing |
| :--- | :--- | :--- | :--- | :--- |
| AWS | Lambda | 15 minutos | 1M invocações/mês | [aws.amazon.com/lambda/pricing](https://aws.amazon.com/lambda/pricing/) |
| Google | Cloud Functions | 60 min (2nd gen) | 2M invocações/mês | [cloud.google.com/functions/pricing](https://cloud.google.com/functions/pricing) |
| Azure | Functions | 10 min (Consumption) | 1M invocações/mês | [azure.microsoft.com/pricing/details/functions](https://azure.microsoft.com/en-us/pricing/details/functions/) |
| Vercel | Serverless Functions | 10 segundos | Generoso | [vercel.com/pricing](https://vercel.com/pricing) |
| Cloudflare | Workers | 30 segundos (CPU time) | 100k req/dia | [workers.cloudflare.com](https://workers.cloudflare.com/) |

### 6.3 Quando serverless faz sentido

✅ **Faz sentido:**
- Tarefas assíncronas (processar imagem, enviar email)
- Webhooks (GitHub, Stripe, Mercado Pago)
- APIs de baixo tráfego (< 100 req/dia)
- Cron jobs (agendamentos)
- Protótipos e MVPs

❌ **NÃO faz sentido:**
- APIs com alta frequência (paga mais que uma VPS)
- Aplicações com conexões persistentes (WebSocket)
- Cargas que exigem **cold start** baixo (<500ms)
- Processamento de CPU intensivo
- Aplicações que precisam manter estado em memória (cache)

### 6.4 O problema do Cold Start

Quando uma função serverless não é invocada por alguns minutos, a plataforma **destrói o container**. Na próxima invocação:

1. Provisionar container (~200ms)
2. Baixar código (~100ms)
3. Carregar Python + dependências (~500ms-2s para Flask grande)
4. Executar sua função (~50ms)

**Total: 800ms-2,5s de latência na primeira requisição.** Para uma API pública, isso é inaceitável. Para um webhook processado em background, não importa.

### 6.5 O problema do custo em escala

**Cenário:** API com 10 milhões de requests/mês, 200ms de execução, 256 MB RAM. ⚠️ Estimativa.

| Opção | Custo mensal | Página de pricing |
| :--- | :--- | :--- |
| AWS Lambda | **~R$ 60-80/mês** | [aws.amazon.com/lambda/pricing](https://aws.amazon.com/lambda/pricing/) |
| VPS Hetzner (2 vCPU, 4 GB) | **~R$ 37/mês** (✅ verificado) | [hetzner.com/cloud](https://www.hetzner.com/cloud/) |
| Render Starter | **~R$ 43/mês** (⚠️ US$ 7) | [render.com/pricing](https://render.com/pricing) |

**Cálculo Lambda:**
- 10 milhões de invocações × US$ 0,20/milhão = US$ 2,00
- Duração: 10M × 0,2s × 256MB = 512.000 GB-segundos × US$ 0,0000166667 ≈ US$ 8,53
- Free tier: 1M invocações + 400.000 GB-s grátis → desconto pequeno
- **Total: ~US$ 10-12/mês ≈ R$ 60-70/mês**

**Conclusão atualizada:** para 10M requests/mês, Lambda e VPS têm custo **similar** (~R$ 60-80). Lambda só se torna significativamente mais caro acima de ~50-100M requests/mês, ou quando a execução é longa (>1s) e com muita memória.

---

## 7: Dimensionamento na prática — Números reais

### 7.1 Como estimar a carga da sua aplicação

**Métricas-chave:**
- **Requests por segundo (RPS):** Quantas requisições sua API recebe
- **Latência média e p95:** Quanto tempo cada request leva
- **Memória por worker:** Quanto cada processo Flask consome
- **CPU por request:** Quão pesado é o processamento

**Como medir:**
```bash
# Ferramenta: wrk (benchmark HTTP)
wrk -t4 -c100 -d30s http://seudominio.com/api/status
```

### 7.2 Cenários de dimensionamento

#### Cenário 1: Site pessoal / portfólio / blog
- **Tráfego:** 100 visitas/dia, 5.000/mês
- **Pico:** 10 requests simultâneos
- **Stack ideal:** GitHub Pages (grátis), Vercel, Cloudflare Pages, ou VPS mínima
- **Custo:** R$ 0 a R$ 40/mês

#### Cenário 2: Startup SaaS em MVP
- **Tráfego:** 1.000 usuários ativos, ~100.000 req/mês
- **Pico:** 50 requests simultâneos
- **Stack ideal:** Render/Railway ou VPS 4GB com Postgres gerenciado
- **Custo:** R$ 120-350/mês

#### Cenário 3: E-commerce de porte médio
- **Tráfego:** 10.000 usuários/dia, 5M req/mês
- **Pico:** 500 requests simultâneos (Black Friday)
- **Stack ideal:** 2-3 VPSs com load balancer, Postgres replicado, Redis cache, CDN
- **Custo:** R$ 900-2.500/mês

#### Cenário 4: SaaS B2B em escala
- **Tráfego:** 500.000 usuários, 100M req/mês
- **Stack ideal:** Kubernetes (EKS/GKE), banco gerenciado, CDN global, observabilidade completa
- **Custo:** R$ 15.000-100.000/mês + equipe de SRE

### 7.3 A regra prática para começar

> **Use a coisa mais simples que resolve seu problema hoje. Reavalie quando doer.**

Para 90% dos projetos de curso, estágio e MVP:
- **VPS barata (Hetzner/Contabo) + Nginx + Gunicorn + systemd** → simples, barato, você aprende
- **Render/Railway** → se você não quer administrar Linux
- **AWS/Azure/GCP** → apenas se estiver aprendendo para fins de carreira ou tiver requisitos específicos

---

## 8: Resumo — Qual escolher?

```
┌─────────────────────────────────────────────────────┐
│          Onde hospedar seu Flask?                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
          ┌────────────────────────┐
          │ Seu app tem tráfego    │
          │ constante 24/7?        │
          └───────┬────────┬───────┘
                  │        │
                 SIM      NÃO
                  │        │
                  ▼        ▼
     ┌────────────────┐  ┌──────────────────────┐
     │ Quer gerenciar │  │ Serverless           │
     │ Linux/SO?      │  │ (Lambda, CF Workers) │
     └────┬─────┬─────┘  └──────────────────────┘
          │     │
         SIM   NÃO
          │     │
          ▼     ▼
    ┌──────┐  ┌──────┐
    │ VPS  │  │ PaaS │
    └──────┘  └──────┘
```

### Tabela de decisão rápida

| Situação | Melhor escolha | Por quê |
| :--- | :--- | :--- |
| Projeto de curso / portfólio | Vercel / GitHub Pages / Render free | Grátis, deploy simples |
| MVP de startup | Railway / Render / Fly.io | Deploy automático, sem administrar SO |
| Aprendendo Linux/sysadmin | Hetzner VPS + Nginx | Barato, controle total, aprendizado real |
| Empresa pequena consolidada | VPS boa + managed Postgres | Custo previsível, performance |
| Startup em crescimento | AWS/GCP com EKS ou ECS | Escala, serviços gerenciados, contratação |
| SaaS B2B enterprise | AWS/Azure com arquitetura complexa | Compliance, SLA, suporte |
| Job de processamento esporádico | Lambda / Cloud Functions | Paga só quando roda |

---

## 9: 🧪 Exercício prático — Deploy na Render

**Objetivo:** Colocar seu projeto Flask (da aula 4 - Flaskr ou livepoll) em produção real com HTTPS.

### Passo 1: Preparar o projeto

1. Garanta que tem `requirements.txt` com todas as dependências:
```txt
flask==3.0.0
gunicorn==21.2.0
```

2. Garanta que seu app usa a **Application Factory** (`create_app()`)

3. Adicione a variável de ambiente para porta (Render exige):
```python
# No final do arquivo que inicia o app
import os
if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8000))
    app.run(host="0.0.0.0", port=port)
```

### Passo 2: Adicionar Procfile

Crie um arquivo `Procfile` na raiz do projeto:
```
web: gunicorn "livepoll:create_app()" --bind 0.0.0.0:$PORT
```

### Passo 3: Deploy

1. Crie conta em [render.com](https://render.com)
2. Clique em **New → Web Service**
3. Conecte seu repositório GitHub
4. Configure:
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `gunicorn "livepoll:create_app()" --bind 0.0.0.0:$PORT`
   - **Instance Type:** Free
5. Deploy!

### Passo 4: Verificar

Acesse `https://seu-app.onrender.com` e veja seu Flask rodando com HTTPS válido.

**Parabéns.** Seu código está em produção.

---

## 10: Checklist — O que um dev pleno espera que você saiba

- [ ] Explicar a diferença entre VPS, Cloud IaaS, PaaS e Serverless
- [ ] Saber quando cada opção faz sentido (e quando NÃO faz)
- [ ] Configurar Gunicorn com número adequado de workers
- [ ] Escrever um Dockerfile multi-stage para Flask
- [ ] Entender o conceito de cold start em serverless
- [ ] Calcular custos reais de diferentes opções de hospedagem
- [ ] Saber que "cloud" não é sempre mais barato que VPS
- [ ] Explicar o que é um reverse proxy (Nginx) e por que usar
- [ ] Entender que preços de cloud mudam e devem ser verificados
