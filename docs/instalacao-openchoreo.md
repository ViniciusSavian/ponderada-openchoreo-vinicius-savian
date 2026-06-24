# Instalação do OpenChoreo v1.1.1

Aluno: Vinicius Savian
Caminho escolhido: A (instalação completa)
Data de execução: 24/06/2026

Este documento registra como instalei o OpenChoreo localmente, validei a interface Backstage e
publiquei a aplicação de exemplo (React Starter), além das limitações que encontrei no caminho.

## Ambiente

| Item | Valor |
|---|---|
| Sistema operacional | macOS 26.2 (build 25C56) |
| Chip | Apple Silicon (arm64) |
| RAM total da máquina | 18 GB |
| CPUs | 11 |
| Docker | Docker Desktop 29.2.1 |
| Recursos do Docker | 11 CPUs e 7,65 GiB de RAM |

Os requisitos mínimos da documentação são 4 GB de RAM e 2 CPUs (8 GB e 4 CPUs se for instalar
também o Workflow Plane). Minha máquina atende o mínimo com folga em CPU, mas a RAM disponível para
o Docker (7,65 GiB) fica logo abaixo dos 8 GB recomendados. Por causa disso decidi fazer a
instalação base, sem os planos opcionais de build (CI/CD) e de observabilidade, que pesam bastante
na memória. A base já inclui Control Plane, Data Plane e Backstage, que é o necessário para validar
a plataforma e subir a aplicação de exemplo.

Saída do comando que verifica os recursos dentro de um container:

```bash
docker run --rm alpine:latest sh -c "echo 'Memory:'; free -h; echo; echo 'CPU Cores:'; nproc"
```

```
Memory:
              total        used        free      shared  buff/cache   available
Mem:           7.7G        3.9G      802.9M       60.0M        3.0G        3.5G
Swap:       1024.0M      838.5M      185.5M
CPU Cores:
11
```

Print: [evidencias/01-recursos-docker.png](evidencias/01-recursos-docker.png)

## Sobre o Colima

A documentação recomenda, no Apple Silicon, usar o Colima com VZ e Rosetta:

```bash
colima start --vm-type=vz --vz-rosetta --cpu 4 --memory 8
```

Eu já uso o Docker Desktop no dia a dia (com outros containers rodando), então preferi testar
primeiro com ele antes de instalar o Colima e mexer no ambiente. As versões mais recentes do Docker
Desktop passaram a suportar `--network=host` no macOS, que é o que o quick-start precisa para expor
as portas. No fim deu certo: o Docker Desktop expôs as portas 8080 (Backstage) e 19080 (gateway)
direto no host, então não precisei do Colima. Deixo o detalhe na seção de limitações.

## Preparação

A porta 8080, que o Backstage usa, estava ocupada por um container de outro projeto meu
(`ai-messages-kafka-ui`). Tive que parar esse container antes de começar:

```bash
docker stop ai-messages-kafka-ui
lsof -i :8080   # confirmar que ficou livre
```

## Passo 1 — Dev Container

Rodei o comando oficial do quick-start:

```bash
docker run --rm -it --name openchoreo-quick-start \
  --pull always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --network=host \
  ghcr.io/openchoreo/quick-start:v1.1.1
```

Ele baixa a imagem, monta o socket do Docker (para criar o cluster k3d de dentro do container) e usa
a rede do host. Ao entrar, o container abre um shell já com `docker`, `k3d`, `kubectl` e `helm`
disponíveis.

Print: [evidencias/02-container-iniciado.png](evidencias/02-container-iniciado.png)

## Passo 2 — Instalação

Dentro do container:

```bash
./install.sh --version v1.1.1
```

A instalação levou por volta de 6 minutos e meio e terminou com código de saída 0. O script cria o
cluster k3d (k3s rodando em Docker) e instala via Helm o cert-manager, external-secrets, kgateway,
o Thunder (identity provider), o OpenBao (secrets), o Control Plane e o Data Plane. No fim ele
provisiona os recursos base (namespace, ClusterDataPlane, environments, deployment pipeline,
project e os component types) e configura o Backstage.

Apareceu um aviso de que o preload de imagens falhou (`Image preloading failed - continuing with
installation`). Na prática isso só deixa os primeiros deploys um pouco mais lentos, porque as
imagens são baixadas sob demanda; não atrapalhou a instalação.

Print: [evidencias/03-instalacao-completa.png](evidencias/03-instalacao-completa.png)

## Passo 3 — Status

```bash
./check-status.sh
```

Os componentes do núcleo (Infrastructure, Control Plane e Data Plane) ficaram todos como READY. O
Backstage apareceu como PENDING nos primeiros instantes, porque ainda estava subindo, e ficou
pronto pouco depois — confirmei quando a URL `http://openchoreo.localhost:8080/` passou a responder
200. Os planos Workflow e Observability aparecem como NOT INSTALLED, o que é esperado já que fiz a
instalação base. No total ficaram 23 pods em Running.

Print: [evidencias/04-check-status.png](evidencias/04-check-status.png)

## Passo 4 — Backstage

URL: http://openchoreo.localhost:8080/
Login de admin: `admin@openchoreo.dev` / `Admin@123`

O login é por OIDC. A tela inicial mostra "Sign in using OpenChoreo"; ao clicar em Sign In ele
redireciona para o Thunder (em `thunder.openchoreo.localhost:8080`), onde coloquei usuário e senha,
e em seguida volta autenticado para o Backstage. No dashboard dá para ver o Cluster Data Plane
"default" conectado.

Prints:
- Tela de login: [evidencias/05-backstage-login.png](evidencias/05-backstage-login.png)
- Dashboard logado: [evidencias/06-backstage-dashboard.png](evidencias/06-backstage-dashboard.png)

## Passo 5 — Deploy da aplicação React

Dentro do container:

```bash
./deploy-react-starter.sh
```

Terminou com código 0. O script cria o Component e o Workload, espera o ReleaseBinding sincronizar,
o Deployment ficar disponível e a rota HTTP ficar pronta. A URL gerada foi:

```
http://http-react-starter-development-default-cde5190f.openchoreoapis.localhost:19080
```

Testei com `curl` a partir do Mac e respondeu 200. Vale notar que essa URL é diferente do exemplo da
documentação (`react-starter-development-default...`): a versão atual coloca um prefixo `http-` e um
hash no final do hostname, então o certo é usar a URL que o próprio script imprime.

Prints:
- Deploy (SUCCESS + URL): [evidencias/07-react-deploy.png](evidencias/07-react-deploy.png)
- Aplicação no navegador: [evidencias/08-react-app-rodando.png](evidencias/08-react-app-rodando.png)
- Componente no Backstage: [evidencias/11-react-componente-backstage.png](evidencias/11-react-componente-backstage.png)

## Passo 6 — Recursos criados

Comandos executados dentro do container:

```bash
kubectl get namespaces -l openchoreo.dev/control-plane=true
kubectl get clusterdataplanes
kubectl get environments -A
kubectl get projects -A
kubectl get clustercomponenttypes
kubectl get components -A
```

Saídas (resumidas):

```
# namespace com o label de control-plane
default

# clusterdataplanes
default

# environments
development, staging, production

# projects
default

# clustercomponenttypes
scheduled-task (cronjob), service (deployment), web-application (deployment), worker (deployment)

# components
react-starter  ->  project default, tipo deployment/web-application
```

Prints:
- Namespaces: [evidencias/09-kubectl-namespaces.png](evidencias/09-kubectl-namespaces.png)
- Demais recursos: [evidencias/10-kubectl-resources.png](evidencias/10-kubectl-resources.png)

### O que é cada recurso

- **Namespace com label `control-plane=true`**: é onde ficam os controladores e os CRDs do
  OpenChoreo, ou seja, o "cérebro" da plataforma. Na instalação base esse label foi para o
  namespace `default`.
- **ClusterDataPlane**: representa o cluster onde as aplicações de fato rodam. Aqui é o próprio
  cluster k3d local (aparece como conectado no Backstage).
- **Environments**: os ambientes de deploy (development, staging e production), que organizam o
  ciclo de vida e a promoção de uma aplicação entre estágios.
- **Project**: um agrupamento lógico de componentes, mais ou menos como um time ou produto. No
  quick-start existe o projeto `default`.
- **ClusterComponentTypes**: os tipos de componente que a plataforma suporta — service,
  web-application e worker (que viram deployment) e scheduled-task (que vira cronjob).
- **Component**: uma aplicação concreta implantada dentro de um project. No meu caso é o
  `react-starter`, do tipo web-application.
- **Backstage**: a interface web que serve de portal, onde dá para ver os projects, components,
  data planes e as relações entre eles em um lugar só.

## Limitações e observações

- A RAM disponível para o Docker (7,65 GiB) ficou no limite do recomendado, então não instalei os
  planos opcionais de build e observabilidade. Para habilitá-los seria
  `./install.sh --version v1.1.1 --with-build --with-observability`, de preferência depois de subir
  a memória do Docker para 8 GB ou mais.
- Apesar de a documentação recomendar o Colima no Apple Silicon, consegui rodar tudo só com o Docker
  Desktop 29.2.1: o `--network=host` expôs as portas 8080 e 19080 no host e as duas responderam 200.
- A porta 8080 estava ocupada por outro container meu e precisou ser liberada antes de instalar.
- A URL da aplicação gerada pela versão atual é diferente da que aparece no exemplo da documentação.
- O aviso de falha no preload de imagens deixa os primeiros deploys mais lentos, mas não impede a
  instalação.
- Os domínios `.localhost` resolvem para 127.0.0.1 no macOS sem precisar configurar nada.
