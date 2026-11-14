---
title: Sandbox
weight: 5
---

Sandbox é a camada de isolamento do Vectora para o acesso do agente a terminal e filesystem. Segue o
design do [ai-jail](https://github.com/akitaonrails/ai-jail), do Fabio Akita — as ações do agente
rodam _dentro_ da jaula, não atrás de uma única chamada de função sandboxed — adaptado à arquitetura
do Vectora: um único processo de backend de longa duração servindo vários workspaces simultâneos, em
vez de um processo por invocação.

## Modelo de ameaça

Sem sandbox, `terminal`, `file_write` e `file_edit` operam diretamente no processo do backend — o
agente (ou uma injeção de prompt) pode ler `~/.ssh`, exfiltrar segredos ou rodar comandos
destrutivos com os mesmos privilégios do próprio backend. O Sandbox existe pra limitar esse raio de
dano aos `rw_paths`/`ro_paths` declarados de um workspace, sem abrir mão completamente do acesso a
arquivo e terminal (isso anularia o propósito de um agente de código autônomo).

## Ativação por workspace

Se o workspace **não tem** `vectora.toml` e você não definiu uma política, o Vectora liga o sandbox
sozinho quando o ambiente suporta isolamento nativo (`bwrap` no Linux, WSL2 no Windows); sem esse
suporte, e sempre no macOS, ele fica desligado. Se o workspace tem `vectora.toml` sem a seção
`[sandbox]`, o sandbox fica desligado e o workspace se comporta como antes dessa feature existir.

### Quem define a política

A política que **amplia** o acesso do jail é sua, não do repositório. Um repositório clonado pode
trazer qualquer `vectora.toml`, então o Vectora guarda a sua política **fora do workspace**, em
`~/.vectora/sandbox-policies/` (um arquivo por workspace), e a edita pela API
`GET`/`PUT /workspaces/{id}/sandbox/policy` (a gravação exige workspace confiado).

O `[sandbox]` do `vectora.toml` do repositório só pode **restringir**:

- liga o sandbox e o `lockdown`, exige isolamento completo, acrescenta máscaras, desliga GPU;
- estreita `rw_paths`, `ro_paths`, `env_pass`, `allow_hosts`, `allow_tcp_ports` e `git_writable`, e
  muda a rede para um modo mais fechado.

O que ele pede e **amplia** (um caminho gravável novo, `env_pass`, `git_writable`, rede mais aberta,
outro `backend`, `docker_image`, `remote_host`) é ignorado, registrado no log e listado em
`ignored_repo_keys` na resposta da política. Desligar o sandbox que você ligou também é ignorado.

```toml
# Política do usuário (a mesma tabela que o repositório pode usar para restringir)
[sandbox]
enabled = true
rw_paths = ["/dados"]            # graváveis dentro da jaula, além do workspace
ro_paths = []                    # só leitura
mask = [".env", "**/*.pem", "**/.ssh/**", "**/.aws/**"]
env_pass = ["NPM_TOKEN"]         # variáveis do host que o jail pode herdar
git_writable = false             # true libera a escrita em .git/hooks e .git/config
network = "filtered"             # off | filtered | open
allow_hosts = ["api.exemplo.com", "*.cdn.exemplo.com"]
registry_hosts = true            # pip, npm, cargo, go, maven, composer e GitHub
lockdown = false                 # true: sem rede, perfil restrito de recursos
require_full_isolation = false   # true: não inicia sem seccomp e Landlock
```

TOML malformado, ou um campo com tipo inválido (por exemplo, uma string no lugar de uma lista),
falha **fechado** (lockdown) em vez de rodar desprotegido.

## O que o jail enxerga

- O sistema (`/usr`, `/bin`, `/lib`, `/lib64`, `/etc`) somente leitura, com os links simbólicos do
  host, e um `/tmp` próprio.
- O ambiente virtual do Python e o pacote `backend`, somente leitura. O restante do repositório do
  Vectora não é exposto.
- Um `$HOME` vazio em tmpfs. Nada do seu home aparece.
- **Ambiente limpo:** só `PATH`, `HOME` do jail, `LANG`, `TERM`, `TZ` e o que você liberar em
  `env_pass`. Chaves de provider e tokens do backend nunca chegam ao jail.
- Sessão nova, sem terminal de controle, e `no_new_privs`.
- `.git/hooks` e `.git/config` somente leitura (um hook gravado pelo agente rodaria fora do jail no
  seu próximo `git`); commits e branches continuam funcionando.

## Rede

O jail não tem rota de rede própria. No modo padrão, `filtered`, ele só alcança um proxy de saída
que roda fora do jail e decide cada conexão HTTPS (`CONNECT`) pela allowlist: os registries de
pacotes (se `registry_hosts` estiver ligado) mais os seus `allow_hosts`. O proxy recusa IP privado,
loopback e endereços de metadados, só aceita as portas 80 e 443, conecta ao IP já validado (sem
resolver o nome de novo, contra DNS rebinding) e registra cada decisão. HTTP sem TLS é recusado.

| Modo       | Efeito                                                                       |
| ---------- | ---------------------------------------------------------------------------- |
| `filtered` | Só os destinos permitidos, pelo proxy (padrão)                               |
| `off`      | Sem rede                                                                     |
| `open`     | Rede do host, sem filtro. Use só quando precisar; qualquer coisa sai por ela |

`lockdown = true` sempre corta a rede. No WSL2 o proxy não é alcançável pela distro, então a rede
fica aberta e o Vectora registra o aviso. Detalhes da política de egress das ferramentas em
[Políticas de egress](../../guides/egress-policies).

## Como funciona: um worker por workspace

Em vez de criar um processo sandboxed novo a cada chamada de tool, o Vectora mantém um único worker
jailado persistente vivo por workspace com `[sandbox]` habilitado — nasce na primeira ação
sandboxável, é encerrado quando o workspace fica ocioso. `file_write`, `file_edit` e comandos
`terminal` pontuais roteiam por esse worker via um protocolo JSON-lines pequeno
(`exec`/`read_file`/`write_file`).

No backend `local` (Linux, ou Windows via WSL2), o worker roda sob `bwrap` quando disponível
(Bubblewrap) com:

- Namespacing de filesystem restrito a `rw_paths`/`ro_paths`.
- Paths mascarados (`mask`) nunca expostos dentro da jaula, mesmo se caírem sob um path permitido.
- Landlock e limites de recursos aplicados pelo próprio worker.
- Um filtro seccomp-BPF (via `pyseccomp`/`libseccomp`) negando syscalls perigosas: `ptrace`,
  `mount`, `unshare`, `setns`, `bpf`, módulos de kernel, `io_uring`, `userfaultfd`, `keyctl`,
  `perf_event_open`, `ioctl(TIOCSTI)` e a criação de namespaces por `clone`. Sem `libseccomp` o
  worker roda sem o filtro e o isolamento fica **parcial**; com `require_full_isolation = true` ele
  nem inicia.
- Mudou a pasta ou a política do workspace, o worker é recriado na próxima ação.

No Windows, a ativação automática verifica uma distribuição WSL2 elegível, não a presença de `bwrap`
dentro dela. Se `bwrap` estiver ausente, o worker usa um processo soft nativo sem isolamento de
filesystem ou rede, mesmo com `lockdown = true`. No macOS não há ativação automática: o worker local
persistente tenta usar `bwrap`, não Seatbelt (`sandbox-exec`).

O modo soft não concede dispensa de HITL por sandbox. Até confirmar um worker vivo com isolamento
efetivo, valem as regras normais de aprovação; depois que o HITL depende desse isolamento, uma
substituição por worker soft é recusada.

Os demais backends (`docker`, `ssh`, `modal`) rodam o mesmo modelo de worker contra seu respectivo
primitivo de isolamento; `docker` também nega acesso a rede sob `lockdown`.

## Terminal interativo

A aba Terminal (PTY) roda no mesmo jail `bwrap` do backend `local`, com o mesmo ambiente limpo e a
mesma rede filtrada. Nos backends `docker`, `ssh` e `modal` ela ainda não é suportada: iniciar o
terminal interativo retorna um erro claro em vez de cair num shell sem proteção.

## Veja também

- [Usando o Workbench](../../guides/using-the-workbench) — as abas Terminal e Arquivos na prática
- [Sessões e Workspaces](../sessions-and-workspaces) — como um workspace é confiado e delimitado
