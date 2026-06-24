# Ponderada — OpenChoreo Local

Atividade individual de instalação e validação de uma instância local do **OpenChoreo v1.1.1**,
com deploy da aplicação de exemplo (React Starter), feita no meu MacBook.

- **Aluno:** Vinicius Savian
- **Caminho:** A — instalação completa
- **Data:** 24/06/2026

## Documentação

A documentação completa do processo está em
[docs/instalacao-openchoreo.md](docs/instalacao-openchoreo.md): ambiente usado, comandos
executados, status da instalação, recursos criados e as limitações que encontrei.

As evidências (prints e saídas de terminal) estão em [docs/evidencias/](docs/evidencias/).

## URLs

- Backstage (portal): http://openchoreo.localhost:8080/ — login `admin@openchoreo.dev` / `Admin@123`
- Aplicação React: http://http-react-starter-development-default-cde5190f.openchoreoapis.localhost:19080

## Como subir o ambiente (resumo)

```bash
# Dev Container oficial do OpenChoreo
docker run --rm -it --name openchoreo-quick-start \
  --pull always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --network=host \
  ghcr.io/openchoreo/quick-start:v1.1.1

# Dentro do container:
./install.sh --version v1.1.1
./deploy-react-starter.sh
```
