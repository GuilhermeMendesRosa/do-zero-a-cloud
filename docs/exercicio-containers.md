# Exercício prático: Kanban em containers no WSL

Neste exercício você vai usar Podman pelo terminal do Ubuntu no WSL para iniciar três containers separados: PostgreSQL, API Spring Boot e frontend. O arquivo Compose prepara a comunicação entre eles e o armazenamento do banco. Você vai iniciar os serviços um de cada vez e observar logs, paradas, reinícios e persistência.

> Antes desta atividade, conclua os checkpoints do backend usando [`resolucao-todos.md`](resolucao-todos.md). A API ainda tem endpoints incompletos no projeto inicial.

## O que você vai precisar

- Windows 10 versão 2004 (build 19041) ou mais recente, ou Windows 11;
- Ubuntu 24.04 instalado no WSL 2;
- o fork do repositório clonado no Windows;
- conexão com a internet para baixar pacotes, imagens e dependências do projeto.

## 1. Instalar o WSL com Ubuntu 24.04

Abra **PowerShell como Administrador** e instale o WSL com Ubuntu 24.04:

```powershell
wsl --install -d Ubuntu-24.04
```

Reinicie o Windows se for solicitado. Na primeira abertura do Ubuntu, crie um nome de usuário e uma senha Linux. Ao digitar a senha, os caracteres não aparecem na tela; isso é esperado.

No PowerShell, confirme que a distribuição está na versão 2:

```powershell
wsl --list --verbose
```

A linha `Ubuntu-24.04` deve mostrar `2` na coluna `VERSION`. Se mostrar `1`, execute:

```powershell
wsl --set-version Ubuntu-24.04 2
```

Consulte [a documentação oficial de instalação do WSL](https://learn.microsoft.com/windows/wsl/install) se o Windows solicitar atualizações ou uma reinicialização adicional.

## 2. Abrir o WSL na pasta do fork

No Explorador de Arquivos, abra a pasta raiz do fork clonado. Abra um terminal nessa pasta (clique com o botão direito e escolha **Abrir no Terminal**). O terminal pode iniciar em PowerShell; confira que o prompt está na raiz do projeto e execute:

```powershell
wsl -d Ubuntu-24.04
```

Depois da instalação inicial do WSL descrita acima, os comandos do exercício serão executados no terminal Ubuntu, não no PowerShell.

No Ubuntu, confirme a pasta atual e veja os arquivos do projeto:

```bash
pwd
ls
```

`pwd` deve mostrar o caminho do fork, normalmente começando por `/mnt/c/Users/.../do-zero-a-cloud`. A lista deve incluir `backend` e `frontend`. Se não estiver na raiz do fork, abra outro PowerShell, vá para a pasta do projeto no Windows e inicie o WSL dali.

## 3. Instalar Podman e Compose pelo terminal

No Ubuntu, atualize os índices de pacotes e instale Podman, o provedor Compose e `curl`:

```bash
sudo apt update
sudo apt install -y podman podman-compose curl
```

Digite a senha Linux criada na primeira abertura, se for solicitada. Confira as instalações:

```bash
podman --version
podman-compose --version
podman info
```

`podman info` deve exibir os dados da instalação Linux local. Podman executa containers diretamente; não é necessário iniciar um serviço Docker nem instalar Podman Desktop. O pacote `podman-compose` lê o arquivo Compose e chama Podman.

As instruções do Podman para Ubuntu estão na [documentação oficial de instalação](https://podman.io/docs/installation); o pacote Compose está no [repositório de pacotes do Ubuntu](https://packages.ubuntu.com/podman-compose).

## 4. Baixar `compose.yaml` para o fork antigo

Como este fork foi criado antes de o arquivo Compose ser adicionado ao projeto, baixe uma cópia do arquivo original. Faça isso **na raiz do fork**, onde `pwd` mostrou o caminho do projeto:

```bash
curl -fL -o compose.yaml https://raw.githubusercontent.com/GuilhermeMendesRosa/do-zero-a-cloud/main/compose.yaml
```

Confirme que o arquivo foi baixado e não está vazio:

```bash
test -s compose.yaml && echo "compose.yaml pronto"
```

O comando deve imprimir `compose.yaml pronto`. Se `compose.yaml` já existir, confira se é a versão deste exercício antes de substituí-lo. Se `curl` mostrar erro 404, confirme que você está na raiz do fork e que o arquivo já foi publicado na branch `main` do repositório original. A atividade depende dessa publicação.

## 5. Iniciar o PostgreSQL

Inicie somente o banco:

```bash
podman-compose up -d banco
```

`-d` deixa o container executando em segundo plano. Na primeira vez, Podman baixa a imagem oficial do PostgreSQL. Aguarde e confira:

```bash
podman ps
podman logs --tail 30 kanban-banco
```

Espere uma mensagem semelhante a `database system is ready to accept connections`. O arquivo Compose prepara o banco `kanban`, usuário `postgres` e senha local `postgres`. No computador, o banco fica acessível pela porta `5433`; dentro da rede do Compose, continua usando `5432`.

## 6. Iniciar a API

Quando o banco estiver pronto, inicie o backend:

```bash
podman-compose up -d backend
```

Na primeira execução, Podman constrói a imagem usando `backend/Dockerfile`. A etapa baixa dependências Maven e pode levar alguns minutos. Confira os logs:

```bash
podman logs --tail 50 kanban-backend
```

Espere o Spring Boot iniciar e verifique o health check:

```bash
curl http://localhost:8090/actuator/health
```

A resposta esperada contém `"status":"UP"`. Dentro do Compose, a API encontra o banco pelo nome `banco`; o arquivo já deixa essa configuração pronta.

## 7. Iniciar o frontend

Inicie o terceiro serviço:

```bash
podman-compose up -d frontend
```

O primeiro build também pode levar alguns minutos. Confira os três containers:

```bash
podman ps
```

Abra [http://localhost:5173](http://localhost:5173) no navegador do Windows. Acesse **Configurações → Auditoria da API** e execute os checkpoints para confirmar o fluxo frontend → API → PostgreSQL.

## 8. Experimente comandos e observe o comportamento

### Listar containers e imagens

```bash
podman ps
podman ps -a
podman images
```

`podman ps` mostra os containers em execução; `podman ps -a` inclui os parados. `podman images` lista as imagens baixadas e construídas.

### Acompanhar os logs da API

```bash
podman logs -f kanban-backend
```

O `-f` acompanha novas mensagens. Pressione `Ctrl+C` para encerrar o acompanhamento; o container continua ligado.

### Consultar as tabelas do banco

```bash
podman exec kanban-banco psql -U postgres -d kanban -c "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' ORDER BY table_name;"
```

Esse comando executa `psql` dentro do container do PostgreSQL e mostra as tabelas criadas pela API.

### Criar um quadro e confirmar que ficou salvo

```bash
curl -X POST http://localhost:8090/api/v1/board \
  -H 'Content-Type: application/json' \
  -d '{"name":"Quadro de teste"}'

curl http://localhost:8090/api/v1/board
```

O primeiro comando cria o quadro. O segundo lista os quadros persistidos no PostgreSQL.

### Parar e reiniciar somente a API

```bash
podman stop kanban-backend
podman ps -a
podman start kanban-backend
podman logs --tail 30 kanban-backend
curl http://localhost:8090/actuator/health
```

Enquanto o backend estiver parado, a página não conseguirá buscar dados. Depois de iniciar o container, o health check deve voltar para `UP` e o quadro de teste deve continuar na lista.

### Parar e iniciar os serviços do Compose

```bash
podman-compose stop
podman ps -a
podman-compose start
podman ps
```

Parar e iniciar mantém o volume nomeado do PostgreSQL e, com ele, os dados.

## 9. Encerrar e limpar

Para parar e remover os containers e a rede criados pelo Compose:

```bash
podman-compose down
```

O volume `dados-postgres` permanece. Assim, subir os serviços novamente mantém os dados. Para resetar o exercício e começar com banco vazio, removendo também os containers, a rede e o volume com os dados, execute:

```bash
podman-compose down --volumes
```

Esse comando mantém as imagens baixadas para que a próxima execução não precise baixá-las novamente.

## Problemas comuns

| Sintoma | O que conferir |
| --- | --- |
| `wsl` não é reconhecido no PowerShell | Atualize o Windows, instale o WSL em um PowerShell como Administrador e reinicie o computador. |
| `podman` não é reconhecido no Ubuntu | Confira que o terminal aberto é Ubuntu 24.04 no WSL e repita `sudo apt update` e `sudo apt install -y podman podman-compose`. |
| `podman-compose` não é reconhecido | Confira se o pacote `podman-compose` terminou de instalar com `sudo apt install -y podman-compose`. |
| O download de `compose.yaml` retorna 404 | Verifique o caminho atual com `pwd`, refaça o download na raiz do fork e confirme que o arquivo está publicado na branch `main` original. |
| O backend não conecta ao banco | Confira `podman ps -a` e `podman logs kanban-banco`. Espere o banco ficar pronto e reinicie a API com `podman restart kanban-backend`. |
| Uma porta já está em uso | Confira containers com `podman ps` e libere a porta ocupada entre 5433, 8090 e 5173. Se 5433 estiver ocupada, altere somente a porta publicada do banco em `compose.yaml` (por exemplo, `5434:5432`); a API continua usando `banco:5432`. |
| A página abre, mas a API falha | Confira o health check e os logs do backend. Confirme também que os checkpoints dos TODOs foram implementados. |
| Os builds demoram | A primeira execução baixa imagens e bibliotecas. Aguarde a conclusão e leia os logs do container que falhou. |

## O que observar

- O Dockerfile é a receita usada para construir a imagem do backend e do frontend.
- Cada container é um processo em execução a partir de uma imagem.
- A API e o PostgreSQL são containers separados; endereço e credenciais chegam à API por variáveis de ambiente.
- Os dados do banco ficam em um volume nomeado e sobrevivem à parada dos containers.
- `podman-compose` inicia serviços descritos em `compose.yaml`, individualmente ou em conjunto.
