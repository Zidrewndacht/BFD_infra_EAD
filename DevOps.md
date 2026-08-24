# Aula: DevOps — O que realmente é (e o que não é)

**Pré-requisitos:** Flask, Git/GitHub, noções de terminal, HTTP.
**Objetivo:** Entender DevOps como prática de engenharia, não como buzzword. Saber onde aplicar, onde não aplicar e quais são os trade-offs reais.

---

## 0: O problema que DevOps tenta resolver

### 0.1 O "Muro da Confusão" (Wall of Confusion)

Até meados dos anos 2000, o fluxo típico de entrega de software em empresas grandes era:

```
[Time de Dev] ──escreve código──▶ [Joga por cima do muro] ──▶ [Time de Ops]
   (foco em features)                                      (foco em estabilidade)
```

- **Dev** queria mudar coisas rápido (features, correções).
- **Ops** queria que nada mudasse (estabilidade, uptime).
- Os objetivos eram **intrinsecamente conflitantes**.

O resultado era previsível: deploys raros (trimestrais, às vezes anuais), gigantescos e catastróficos. Quando algo quebrava em produção, Dev dizia "na minha máquina funciona" e Ops dizia "seu código é lixo".

### 0.2 A origem real do termo

O termo "DevOps" foi cunhado por Patrick Debois em 2009, ao organizar a primeira conferência *DevOpsDays*. Não surgiu de um whitepaper corporativo — surgiu da frustração de engenheiros que queriam parar de brigar entre si.

> **Definição honesta:** DevOps é um conjunto de práticas, ferramentas e (sim) mudanças culturais que visam **encurtar o ciclo entre escrever código e ele estar em produção**, mantendo qualidade.
>
> Não é um cargo. Não é um time. Não é um software que você compra.

### 0.3 O que DevOps NÃO é

| Afirmação de marketing | Realidade |
| :--- | :--- |
| "DevOps é uma cultura transformacional" | É engenharia de software aplicada a infraestrutura. |
| "Contrate um DevOps Engineer" | Devs devem saber operar; Ops devem saber codar. O "cargo DevOps" muitas vezes é só SysAdmin renomeado com salário maior. |
| "DevOps elimina bugs" | Não. Ele faz com que bugs cheguem em produção mais rápido **e** sejam corrigidos mais rápido. |
| "Toda empresa precisa de DevOps" | Uma padaria com um site WordPress não precisa. |

---

## 1: Os pilares práticos (esqueça os "7 Cs")

A literatura de marketing gosta de empacotar DevOps em acrônimos. Na prática, tudo se resume a **quatro coisas concretas**:

### 1.1 Automação de tudo que é repetitivo

Se você faz algo mais de duas vezes, automatize. Isso inclui:
- Build do código
- Rodar testes
- Deploy em staging/produção
- Provisionar servidores
- Rotacionar logs e backups

### 1.2 Infraestrutura como Código (IaC)

Servidores, redes e bancos de dados são descritos em arquivos versionados no Git, não configurados à mão via SSH.

### 1.3 Entrega contínua (CI/CD)

Cada commit no `main` pode, em tese, ir para produção. Deploys deixam de ser "eventos" e viram rotina.

### 1.4 Observabilidade (não "monitoramento")

Você não monitora "se o servidor está online". Você observa **o que o sistema está fazendo** via métricas, logs estruturados e traces distribuídos.

---

## 2: CI/CD — A espinha dorsal

### 2.1 O que significam as siglas

| Termo | O que é | O que faz na prática |
| :--- | :--- | :--- |
| **CI** (Integração Contínua) | Vários devs mergendo código no mesmo branch várias vezes ao dia. | Roda testes automaticamente em cada push. Se quebrar, ninguém merge mais nada até arrumar. |
| **CD — Entrega Contínua** | O código está **sempre pronto** para ir a produção. | Deploy em staging é automático. Deploy em produção é um clique. |
| **CD — Deploy Contínuo** | O código vai para produção **automaticamente**. | Se passar nos testes, já está no ar. Sem humano no caminho. |

> **Nota importante:** Deploy Contínuo é menos comum fora de empresas de grande escala (Netflix, Amazon, GitHub). A maioria das empresas sérias faz **Entrega Contínua** — o humano aprova o deploy, mas não faz o trabalho manual.

### 2.2 Um pipeline realista de CI/CD

```
git push
   │
   ▼
┌─────────────────────────────┐
│ 1. Lint e análise estática  │  (ruff, eslint)
├─────────────────────────────┤
│ 2. Auditoria de dependências│  (pip-audit, npm audit)
├─────────────────────────────┤
│ 3. Testes unitários         │  (pytest, jest)
├─────────────────────────────┤
│ 4. Build da imagem Docker   │  (docker build)
├─────────────────────────────┤
│ 5. Testes de integração     │  (contra banco real, API mockada)
├─────────────────────────────┤
│ 6. Push para registry       │  (Docker Hub, ECR, GHCR)
├─────────────────────────────┤
│ 7. Deploy em staging        │  (automático)
├─────────────────────────────┤
│ 8. Testes E2E / Smoke       │  (Playwright, Cypress)
├─────────────────────────────┤
│ 9. Aprovação manual         │  (ou automático, se tiver confiança)
├─────────────────────────────┤
│ 10. Deploy em produção      │  (canary, blue-green ou rolling)
└─────────────────────────────┘
```

> **Nota de ecossistema Node.js:** Se você está aplicando CI em projetos JavaScript, **nunca use `npm install`** em pipelines. Use **`npm ci`** (Clean Install). Ele apaga o `node_modules` existente e instala *exatamente* o que está travado no `package-lock.json`, garantindo builds determinísticos idênticos aos da sua máquina.

### 2.3 Ferramentas populares (e quando usar cada uma)

| Ferramenta | Onde roda | Custo | Melhor para |
| :--- | :--- | :--- | :--- |
| **GitHub Actions** | Nuvem do GitHub | Grátis até 2000 min/mês | Projetos já no GitHub, times pequenos/médios |
| **GitLab CI** | Nuvem ou self-hosted | Grátis com limites | Equipes que querem tudo em um lugar só |
| **Jenkins** | Self-hosted | Grátis (mas paga servidor) | Empresas legadas com necessidades bizarras de customização |
| **CircleCI** | Nuvem | Pago | Times que não querem configurar nada |
| **ArgoCD / Flux** | Dentro do seu cluster K8s | Grátis | GitOps (ver Bloco 5.4) |

### 2.4 Exemplo mínimo: GitHub Actions para um projeto Flask

Crie o arquivo `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configurar Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Instalar dependências
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest ruff pip-audit
      
      - name: Rodar Linter
        run: ruff check .
      
      - name: Auditoria de Segurança
        run: pip-audit
      
      - name: Rodar testes
        run: pytest tests/
```

**O que acontece na prática:**
1. Você abre um Pull Request.
2. O GitHub sobe uma VM Ubuntu limpa.
3. Clona seu código, instala Python, roda lint, verifica vulnerabilidades e executa `pytest`.
4. Se tudo passar, aparece um ✅ verde no PR.
5. Se falharem, aparece um ❌ vermelho e o merge pode ser bloqueado.

**Isso é CI.** Nada mais, nada menos.

---

## 3: Containers — O porquê real

### 3.1 O problema que Docker resolve

> "Funciona na minha máquina."

Essa frase resume 80% dos bugs de deploy da história. Uma aplicação Python depende de:
- Versão do Python
- Versão das bibliotecas (e das dependências C delas)
- Variáveis de ambiente
- Arquivos de configuração
- Sistema de arquivos
- Bibliotecas do SO (`libpq`, `libssl`, etc.)

**Docker empacota tudo isso numa única imagem.** Se roda na sua máquina, roda em qualquer lugar que tenha Docker.

### 3.2 O que Docker NÃO é

- **Não é uma VM.** Não há um SO convidado inteiro rodando. Containers compartilham o kernel do host.
- **Não é "mais leve que VM".** É diferente. VMs isolam hardware; containers isolam processos.
- **Não é obrigatório.** Você pode fazer deploy de Flask direto num Linux via systemd. É só mais chato.

### 3.3 Dockerfile mínimo para uma app Flask

```dockerfile
# Imagem base oficial do Python (slim = menor, sem compiladores)
FROM python:3.12-slim

# Evita que o Python bufferize stdout (importante para logs)
ENV PYTHONUNBUFFERED=1

# Diretório de trabalho dentro do container
WORKDIR /app

# Copia só o requirements primeiro (cache do Docker)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copia o código da aplicação
COPY . .

# Expõe a porta (documentação, não mágica)
EXPOSE 8000

# Comando de execução
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "livepoll:create_app()"]
```

**Comandos essenciais:**
```bash
docker build -t minha-app .          # Constrói a imagem
docker run -p 8000:8000 minha-app    # Roda o container
docker ps                            # Lista containers rodando
docker logs <container_id>           # Vê os logs
```

### 3.4 Quando NÃO usar Docker

- **Projetos pequenos com deploy simples** (ex: VPS com `git pull && systemctl restart`).
- **Quando a equipe não sabe usar.** Docker mal configurado é pior que não usar Docker.

---

## 4: Orquestração — Kubernetes e seus irmãos

### 4.1 O problema

Você tem 1 container. Rode com `docker run`.
Você tem 5 containers. Use `docker compose`.
Você tem 500 containers em 50 máquinas, com auto-scaling, failover automático e deploy sem downtime? **Aí você precisa de orquestração.**

### 4.2 O que um orquestrador faz

- Decide **em qual máquina** cada container vai rodar.
- **Reinicia** containers que morreram.
- **Escala** horizontalmente baseado em CPU/memória/requests.
- Faz **rolling updates** (troca containers velhos por novos sem downtime).
- Gerencia **rede interna** (service discovery).
- Gerencia **segredos** e configurações.

### 4.3 Kubernetes (K8s) — a realidade

Kubernetes é o padrão da indústria. Mas:

| Vantagem | Desvantagem |
| :--- | :--- |
| Resolve problemas de escala massiva | Curva de aprendizado brutal |
| Ecossistema gigante | Complexidade operacional alta |
| Multi-cloud | Um cluster K8s mal mantido vira um pesadelo |
| Padrão de mercado (empregabilidade) | Overkill para 90% das empresas |

> **Regra prática:** se você tem menos de ~20 serviços e menos de ~1000 req/s, K8s provavelmente é exagero. Use **PaaS** (Render, Railway, Fly.io) ou **containers gerenciados** (AWS ECS, Azure Container Apps).

### 4.4 Alternativas mais simples

- **Docker Swarm:** mais simples que K8s, quase abandonado.
- **Nomad (HashiCorp):** bom meio-termo, usado por Cloudflare.
- **ECS Fargate (AWS) / Container Apps (Azure):** "Kubernetes sem a dor".
- **Fly.io / Render:** PaaS que esconde a infraestrutura.
- **Coolify:** PaaS open-source que você roda na sua própria VPS. Excelente meio-termo entre VPS nua e Kubernetes.

---

## 5: Infrastructure as Code (IaC)

### 5.1 O problema do "clique-clique"

Configurar um servidor à mão via console web da AWS ou via SSH funciona. Até o dia em que você precisa:
- Recriar o mesmo ambiente em outra região.
- Recuperar de um desastre.
- Revisar o que mudou no último mês.

Sem IaC, isso é impossível.

### 5.2 Ferramentas

| Ferramenta | O que provisiona | Linguagem |
| :--- | :--- | :--- |
| **Terraform** | Qualquer cloud (AWS, Azure, GCP, etc.) | HCL |
| **OpenTofu** | Qualquer cloud | HCL | Fork open-source real do Terraform (após mudança de licença BSL em 2023). Drop-in replacement. |
| **Pulumi** | Qualquer cloud | Python/TypeScript/Go |
| **Ansible** | Configuração de servidores existentes | YAML |
| **CloudFormation** | Só AWS | YAML/JSON |
| **Bicep** | Só Azure | DSL própria |

### 5.3 Exemplo mental (Terraform/OpenTofu)

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name = "flask-app"
  }
}
```

Roda `terraform apply` (ou `tofu apply`), e uma EC2 é criada. Muda o código, roda de novo, e a infraestrutura é atualizada. Apaga? `terraform destroy`.

### 5.4 GitOps — a evolução do IaC

No GitOps, o repositório Git é a **única fonte da verdade** sobre o estado da infraestrutura. Um agente (ArgoCD, Flux) roda dentro do cluster observando o Git. Quando algo muda no repositório, o agente aplica as mudanças automaticamente.

Vantagens reais:
- Todo deploy tem um commit associado (auditoria grátis).
- Rollback = `git revert`.
- Não há "alguém mudou algo direto no console".

---

## 6: Observabilidade — Os três pilares

### 6.1 Por que "monitoramento" não basta

Monitoramento tradicional responde: *"o sistema está saudável?"* (resposta: sim/não, baseado em thresholds).

Observabilidade responde: *"por que o sistema está se comportando assim?"* — e permite que você faça perguntas que **não previu** quando criou o sistema.

### 6.2 Os três pilares

| Pilar | O que é | Exemplo |
| :--- | :--- | :--- |
| **Logs** | Eventos discretos no tempo | `[2026-08-12 14:32:01] ERROR: user_id=42 failed login` |
| **Métricas** | Números agregados ao longo do tempo | `requests_per_second`, `p95_latency_ms` |
| **Traces** | Rastreio de uma requisição através de vários serviços | Request `abc-123` passou por API → Auth → DB em 47ms |

### 6.3 Ferramentas populares

- **Logs:** ELK (Elasticsearch, Logstash, Kibana), Loki (Grafana), Datadog
- **Métricas:** Prometheus + Grafana, Datadog, CloudWatch
- **Traces:** Jaeger, Tempo (Grafana), Honeycomb, Datadog APM

### 6.4 O que um desenvolvedor júnior PRECISA saber

1. **Logs estruturados** (JSON, não texto livre) são infinitamente mais úteis.
2. **Nunca logar dados sensíveis** (senhas, tokens, PII).
3. **Correlation IDs** permitem rastrear uma requisição entre serviços.
4. **Latência p95/p99** importa mais que a média.
5. **Alertas ruins destroem times.** Se 50% dos alertas não exigem ação, apague ou ajuste eles.

---

## 7: Onde DevOps falha — Limitações reais

### 7.1 Complexidade oculta

Cada ferramenta nova adiciona complexidade. Um stack "DevOps moderno" pode ter:
- Kubernetes
- Istio (service mesh)
- Prometheus + Grafana
- Vault
- ArgoCD
- Terraform
- GitHub Actions
- Docker Registry
- Cert-Manager
- External-DNS

Para um app que faz 100 requisições por dia, isso é **absurdo**.

### 7.2 Custo

- **AWS ECS Fargate:** ~$0.04/hora por 1 vCPU + 2GB RAM.
- **Cluster EKS (K8s gerenciado):** $0.10/hora **só pelo cluster**, fora os nodes.
- **Ferramentas SaaS de observabilidade:** facilmente $500+/mês para times pequenos.

Muitas startups morrem por gastar mais em infraestrutura que em desenvolvimento.

### 7.3 "DevOps Theater"

Empresas que adotam as ferramentas mas não mudam os processos:
- Têm Jenkins mas fazem deploy mensal.
- Têm Kubernetes mas só um serviço rodando.
- Têm "time DevOps" que age como o antigo time de Ops, só com nome diferente.

Resultado: gastaram milhões e continuam com os mesmos problemas.

### 7.4 Burnout e on-call

A cultura "tudo em produção o tempo todo" pode levar a plantões 24/7 mal estruturados. Empresas sérias pagam **adicional de on-call** e têm política de *blameless postmortems*. Startups "ágil" às vezes só exploram o time.

---

## 8: Resumo para a vida real

### 8.1 Para uma aplicação pequena (1 dev, 1 app)

```
Código no GitHub
   ↓
GitHub Actions roda testes
   ↓
Deploy via SSH no VPS (systemd + gunicorn)
   ↓
Logs em arquivo, nginx na frente
```

**É DevOps? Sim.** CI existe, o deploy é automatizável, não há "muro".

### 8.2 Para uma aplicação média (3-10 devs, alguns serviços)

```
GitHub + GitHub Actions
   ↓
Docker images em GHCR
   ↓
Deploy em PaaS (Fly.io, Railway) ou Coolify self-hosted
   ↓
Grafana Cloud free tier + logs do PaaS
```

### 8.3 Para aplicações grandes (múltiplos times, milhões de usuários)

```
GitOps com ArgoCD
   ↓
Kubernetes (EKS/GKE/AKS)
   ↓
Terraform/OpenTofu para infraestrutura
   ↓
Stack completo de observabilidade
   ↓
Times de plataforma (Platform Engineering)
```

### 8.4 A regra de ouro

> **Adote complexidade apenas quando a dor de não adotá-la for maior que a dor de mantê-la.**

Não use Kubernetes porque é "moderno". Use porque você tem problemas que só ele resolve. A maioria das pessoas não tem.

---

## 9: 🧪 Exercício prático — CI com GitHub Actions + Docker

**Objetivo:** pegar um app Flask que você já tem e adicionar CI automático.

### Passo 1: Adicionar testes ao seu projeto Flask

Crie `tests/test_app.py`:

```python
import pytest
from livepoll import create_app

@pytest.fixture
def client():
    app = create_app()
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_index_retorna_200(client):
    response = client.get('/')
    assert response.status_code == 200

def test_api_status_retorna_json(client):
    response = client.get('/api/status')
    assert response.status_code == 200
    data = response.get_json()
    assert 'votos' in data
```

### Passo 2: Adicionar pytest às dependências

No `requirements.txt`:
```
flask==3.0.0
gunicorn==21.2.0
pytest==8.0.0
ruff==0.3.0
pip-audit==2.7.0
```

### Passo 3: Criar o workflow

Crie `.github/workflows/ci.yml` (como no Bloco 2.4).

### Passo 4: Push e observe

Abra um PR com um teste quebrado de propósito. Veja o CI falhar. Corrija. Veja o CI passar. Mergeie.

---

## 10: Referências
- [Red Hat — What is DevOps](https://www.redhat.com/pt-br/topics/devops)
- [IBM — DevOps](https://www.ibm.com/br-pt/think/topics/devops)
- [AWS — What is DevOps](https://aws.amazon.com/pt/devops/what-is-devops/)
- [Azure — What is DevOps](https://azure.microsoft.com/pt-br/resources/cloud-computing-dictionary/what-is-devops)

---

## 11: Checklist — o que um dev pleno espera que você saiba

- [ ] Explicar CI vs CD (entrega vs deploy contínuo)
- [ ] Escrever um Dockerfile funcional para uma app Flask
- [ ] Configurar um workflow básico no GitHub Actions
- [ ] Explicar a diferença entre VM e container
- [ ] Saber quando K8s é overkill
- [ ] Diferenciar logs, métricas e traces
- [ ] Entender por que "na minha máquina funciona" é um problema de engenharia, não de azar
- [ ] Questionar adoção de ferramentas ("por que precisamos disso?")
- [ ] Conhecer a bifurcação Terraform/OpenTofu e suas implicações de licença
- [ ] Usar `npm ci` em vez de `npm install` em pipelines de CI