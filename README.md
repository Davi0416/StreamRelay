# StreamRelay

*Split tunneling* por fluxo para o Discord no Windows. Só a criação da sessão e o início de uma transmissão de tela
passam por um túnel WireGuard; voz, texto, login, a mídia depois de estabilizada e o resto do computador seguem pela
conexão direta.

> **Estado: experimental (0.5.0).** Leia [Estado de verificação](#estado-de-verificação) antes de usar.
>
> Gratuito e não comercial. Este repositório contém apenas esta descrição; o código-fonte não é público. O instalador está em
> [Releases](../../releases).

## Sumário

- [Como usar](#como-usar)
- [Motivação](#motivação)
- [Visão geral](#visão-geral)
- [Como funciona](#como-funciona)
- [Proteções e recuperação de falhas](#proteções-e-recuperação-de-falhas)
- [O painel](#o-painel)
- [Requisitos](#requisitos)
- [Instalação e desinstalação](#instalação-e-desinstalação)
- [Estado de verificação](#estado-de-verificação)
- [Limitações conhecidas](#limitações-conhecidas)
- [Componentes de terceiros](#componentes-de-terceiros)
- [Aviso](#aviso)

## Como usar

> [!IMPORTANT]
> - Use o Discord baixado do site oficial, com a sua conta já conectada. A versão da Microsoft Store, o PTB e o
>   Canary não são reconhecidos.
> - Desconecte qualquer outra VPN antes de usar. O StreamRelay não funciona com outro cliente de VPN ativo ao mesmo
>   tempo.
> - O arquivo `.conf` contém a chave privada da sua VPN. Não use o `.conf` de outra pessoa e não envie o seu para
>   ninguém.

### 1. Instalar

1. Baixe o instalador `StreamRelay_<versão>_x64-setup.exe` em [Releases](../../releases) e abra.
2. O instalador não tem assinatura de código, então o Windows pode avisar. Nesse caso, clique em
   **Mais informações** > **Executar assim mesmo**.
3. Aceite a permissão de administrador. É o único pedido de permissão; depois disso o uso não pede mais nada.
4. Quando o instalador perguntar se deve iniciar com o Windows, clique em **Sim**.

Na primeira instalação, a rede pode cair por alguns segundos e o Windows pode pedir para reiniciar.

### 2. Impedir o Discord de abrir sozinho com o Windows

O StreamRelay precisa estar pronto **antes** de o Discord abrir, e por isso ele mesmo abre o Discord.

1. Aperte **Ctrl + Shift + Esc** para abrir o Gerenciador de Tarefas.
2. Vá na aba **Aplicativos de inicialização**.
3. Clique em **Discord** > **Desabilitar**.

### 3. Obter um perfil WireGuard (`.conf`)

Funciona com qualquer VPN que forneça um perfil WireGuard com um `[Interface]` e um `[Peer]`. Exemplo com a Proton VPN:

1. Entre em <https://account.protonvpn.com/downloads>.
2. Desça até **WireGuard configuration**.
3. Preencha assim:

   | Opção | Valor |
   | --- | --- |
   | Nome | qualquer um |
   | Plataforma | **Windows** |
   | NAT-PMP | **desligado** |
   | VPN Accelerator | **ligado** |
   | Servidor | escolha um e clique em **Criar** |

4. Baixe o arquivo `.conf`.

### 4. Configurar e iniciar

1. Abra o painel do StreamRelay.
2. Arraste o `.conf` para **Perfil da VPN**, ou clique para selecionar o arquivo.
3. Fora de uma chamada, clique em **Iniciar**. O Discord fecha e abre sozinho; uma chamada em andamento cai.

### 5. Pronto

Quando o painel mostrar **Pronto**, use o Discord e compartilhe a tela normalmente. Fechar a janela do painel não para o
StreamRelay; para desligar, clique em **Parar**.

Se o painel avisar **O servidor da VPN não deixa o Discord passar**, gere na sua VPN um perfil de outro servidor,
importe e clique em **Iniciar** de novo.

### 6. Atualizar

Quando sair uma versão nova, o painel mostra **Nova versão disponível** com o botão **Atualizar**. Clique, aceite a
permissão do Windows e aguarde: o painel baixa o instalador, fecha, a atualização roda e o painel novo abre sozinho no
fim. O perfil da VPN, a escolha de iniciar com o Windows e o driver são mantidos. Se o StreamRelay estiver ativo, o
instalador pede para pará-lo antes.

Na versão 0.4.7 ou anterior, o botão se chama **Baixar** e abre a página da versão em [Releases](../../releases):
baixe o instalador novo e abra. Ele pergunta se deve atualizar e instala por cima, mantendo as mesmas configurações.

## Motivação

### O que o StreamRelay resolve

O StreamRelay estabelece o compartilhamento de tela do Discord através de uma VPN WireGuard sem colocar o resto do
tráfego dentro dela.

Para isso, ele roteia pelo túnel apenas duas fases curtas da conexão:

1. **Estabelecimento da sessão no gateway:** o handshake do WebSocket que termina em `READY`.
2. **Bootstrap da conexão de mídia da transmissão:** o *IP Discovery* UDP com o servidor de mídia.

Concluídas essas fases, o túnel deixa de ser usado. O gateway retoma a mesma sessão pela rota direta (`RESUME`), e a
mídia continua no mesmo socket UDP, agora também pela rota direta. O resultado é um túnel usado por alguns segundos e
para poucos fluxos, em vez de durante a sessão inteira. Os testes feitos até agora estão em
[Estado de verificação](#estado-de-verificação).

### Por que não usar uma VPN comum

As formas usuais de rotear tráfego por VPN têm granularidade grossa demais para esse caso:

| Abordagem | O que passa pelo túnel | Por quanto tempo |
| --- | --- | --- |
| *Full tunnel* (rota padrão pelo túnel) | Todo o tráfego do sistema: voz, jogos, navegador, downloads | Enquanto a VPN estiver ligada |
| *Split tunneling* por aplicativo | Todos os sockets do processo do Discord: voz, texto, mídia | A sessão inteira |
| **StreamRelay** (*split tunneling* por fluxo) | Só os fluxos de bootstrap: os IPs do gateway e o endpoint UDP da transmissão | Segundos, com prazo máximo |

Com o StreamRelay, voz, texto e a mídia já estabelecida seguem pela rota direta: sem o salto extra até o servidor VPN,
sem o encapsulamento do WireGuard e sem o MTU reduzido do túnel (1420).

### Princípios de projeto

- **Granularidade por fluxo:** a decisão é tomada por endpoint UDP e por destino /32, nunca pela rota padrão nem pelo
  processo inteiro.
- **Túnel ocioso por padrão:** o peer WireGuard começa com AllowedIPs vazio e sem rotas. Um destino só fica alcançável
  pelo túnel enquanto está explicitamente exposto.
- **Exposição com prazo:** a janela do gateway dura no máximo 20 s. O relay de mídia dura até a mídia estabilizar, ou
  no máximo 30 s.
- **Na dúvida, rota direta:** classificação ambígua, candidato antigo, prazo vencido ou formato desconhecido sempre
  resultam em rota direta, nunca em túnel.
- **Isolamento fail-closed:** filtros WFP na interface do túnel bloqueiam qualquer outro programa. Se outro processo
  tentar um destino exposto, a conexão falha em vez de vazar.
- **Fora de processo:** sem DLL, sem hook e sem leitura de memória do Discord. O motor usa só APIs públicas do Windows
  (IP Helper, WFP, NDIS), o log que o próprio Discord grava em disco e os cabeçalhos dos pacotes.
- **Rollback determinístico:** a tabela de rotas é salva antes de qualquer mudança e comparada no encerramento.
  Supervisor e watchdog fazem o rollback mesmo se o motor morrer.

## Visão geral

```
 Painel (sem elevação)
   │  schtasks /Run
   ▼
 Tarefa agendada elevada (registrada uma vez, na instalação)
   │
   ▼
 Launcher PowerShell ── pré-checagens ── driver NDIS ── baseline de rede
   │
   ▼
 Motor nativo (C++)  ◄── supervisor + watchdog nativos (rollback se algo morrer)
   ├─ túnel WireGuardNT temporário (AllowedIPs vazio, sem rotas)
   ├─ guarda WFP na interface do túnel
   ├─ leitor do log do renderer do Discord (somente leitura)
   ├─ captura UDP IPv4 na placa física (Windows Packet Filter)
   └─ relay por socket próprio + exposição /32 sob demanda
```

| Componente | Linguagem | Papel |
| --- | --- | --- |
| Painel `StreamRelay.exe` | Rust (Win32 nativo, empacotado pelo CLI do Tauri, sem webview) | Interface; não roda elevado; dispara as tarefas de iniciar/parar |
| Launcher | PowerShell | Pré-checagens, instalação/ativação do driver, baseline de rede, passagem para o supervisor, rollback |
| Motor `external-relay-engine.exe` | C++ | Túnel, guarda WFP, classificação, retenção, relay e migração |
| Supervisor / watchdog | C++ (modos do mesmo executável) | Espera por eventos, guarda de exposição de rotas, disparo do rollback |

## Como funciona

### 1. Identidade do alvo

O motor só atua sobre o `Discord.exe` instalado em `%LOCALAPPDATA%\Discord\app-<versão>\`, conferido por PID, horário
de criação do processo e caminho da imagem. Um PID reaproveitado por outro processo é recusado.

### 2. Túnel

- O perfil WireGuard do usuário (`.conf` com exatamente um `[Interface]` e um `[Peer]`) é lido em memória.
- O motor cria um adaptador **WireGuardNT** temporário com a lista de AllowedIPs do peer **vazia** e sem nenhuma rota.
  Métrica 5000 (IPv4 e IPv6), MTU 1420. Ele espera o handshake e depois apaga as chaves da memória.
- Enquanto nada é exposto, o túnel existe mas nenhum destino é alcançável por ele.
- Antes de ficar pronto, o motor confere se o gateway do Discord responde pelo túnel: o mesmo pedido de WebSocket que
  o Discord faz, aceito só com a resposta de upgrade e com a conexão saindo de fato pelo túnel. Cada servidor tem até
  3 tentativas.
- Se o servidor do perfil não responder, o motor troca só o servidor do túnel, mantendo a chave do perfil, por uma
  lista de servidores reserva que vem com o programa, em ordem, por até 30 s, e usa o primeiro que responder. A lista
  traz servidores gratuitos da Proton VPN. Se nenhum responder, o túnel volta ao servidor do perfil, o Discord segue
  pela conexão direta e o painel avisa.

### 3. Janela de bootstrap do gateway

Quando aparece um processo principal **novo** do Discord, ou quando o log indica que a sessão atual foi descartada
(`[GatewaySocket] [RESET]`), o motor expõe pelo túnel os IPv4 resolvidos de `gateway.discord.gg` (rota /32 + AllowedIP).

| Regra | Valor |
| --- | --- |
| Fecha depois do `READY` da sessão | 2 s |
| Prazo total da janela | 20 s |
| Tentativa sem conexão estabelecida | 8 s, depois **uma** nova tentativa |
| Depois de falhar | o Discord segue pela conexão direta e o painel avisa |

Ao fechar a janela, as conexões TCP 443 do Discord que ficaram no túnel são encerradas (`SetTcpEntry`, `DELETE_TCB`),
e o Discord reconecta pela conexão direta retomando a mesma sessão.

### 4. Classificação da transmissão

- **Log:** o motor acompanha o log do renderer do Discord a partir do fim, só em leitura, e guarda apenas o contexto
  da conexão (`default` ou `stream`), o destino IPv4:porta e o SSRC de áudio.
- **Pacotes:** o driver NDIS do Windows Packet Filter captura, na placa física, os datagramas UDP IPv4 de saída.
  Um datagrama só entra na máquina de estados se pertencer ao processo auditado do Discord, conforme a tabela UDP do
  Windows (PID, porta e horário do bind). Todo o resto é reencaminhado sem alteração.
- **Retenção:** o primeiro *IP Discovery* de cada endpoint (74 bytes, tipo 1) é retido por até 1 s, até casar
  **destino + SSRC** com um candidato do log. Ambiguidade, candidato velho ou prazo vencido resultam em liberação direta.

O motor lê apenas cabeçalhos: IP/UDP, os 8 bytes do cabeçalho de IP Discovery e o cabeçalho RTP/RTCP. O conteúdo da
mídia não é lido nem gravado.

### 5. Relay e migração

| Classificação | Ação |
| --- | --- |
| Transmissão de tela | Expõe só aquele destino (/32 + AllowedIP) e envia por um socket do próprio motor preso ao túnel (`IP_UNICAST_IF`); as respostas são entregues ao socket original do Discord |
| Voz padrão, desconhecido, ambíguo ou candidato antigo | Liberado pela conexão direta; o endpoint deixa de ser filtrado |

O endpoint volta para a conexão direta quando a mídia está de fato fluindo:

| Papel | Critério (pacotes RTP ≥ 100 bytes) |
| --- | --- |
| Assistindo (downlink) | ≥ 150 pacotes, ≥ 100 kB, ao longo de ≥ 2 s |
| Transmitindo (uplink) | ≥ 300 pacotes ao longo de ≥ 3 s **e** ≥ 3 RTCP do servidor |
| Sem prontidão | volta para a conexão direta após 30 s |

Na migração, o AllowedIP e as rotas são removidos primeiro, depois o filtro do endpoint é desativado e o socket de
relay é fechado.

### 6. Encerramento

Ao parar, o motor devolve os relays à conexão direta, remove regras, rotas e AllowedIPs, fecha o adaptador e desativa a
captura. O rollback compara a tabela de rotas com o baseline salvo antes de qualquer mudança.

## Proteções e recuperação de falhas

- **Guarda WFP do túnel:** filtros WFP em modo usuário, numa sessão dinâmica, bloqueiam todo connect/accept (IPv4 e
  IPv6) na interface do túnel, exceto do próprio motor e da imagem auditada do Discord. Ela é aberta antes de qualquer
  exposição e falha fechada: outro programa que tente um destino exposto **falha**, em vez de passar pela VPN.
- **Guarda de exposição:** o supervisor lê a tabela de rotas do sistema a cada 1 s. Qualquer /32 do túnel presente por
  ≥ 60 s encerra o motor e dispara o rollback.
- **Supervisor + watchdog:** se o motor, o supervisor ou o watchdog morrer, o outro processo faz o rollback.
- **Pré-checagens do launcher:** hash SHA-256 de cada arquivo contra o manifesto e assinatura válida do driver; recusa
  se já houver motor ou captura ativos, se houver uma chamada em andamento ou se não houver perfil importado.
- **Driver de outro programa:** uma instalação do Windows Packet Filter que já existia não é tomada como do produto. Ela
  é reutilizada só se o hash for idêntico, e é preservada na desinstalação.
- **Rede muda durante o uso:** o motor acompanha o adaptador físico por notificação do Windows (nada é consultado em
  intervalo fixo). Cabo ou Wi-Fi que caem e voltam na mesma rede não encerram a execução: a captura de pacotes é
  refeita sozinha. Se a rede mudar de verdade (outro gateway), a execução é desfeita; fora de uma chamada, o
  StreamRelay liga de novo sozinho, no máximo uma vez a cada 10 minutos; em uma chamada, ele espera ela terminar.
- **Motor que para sozinho:** nas mesmas condições (fora de uma chamada, limpeza da execução confirmada, sem outro
  religamento nos últimos 10 minutos), o StreamRelay também liga de novo sozinho se o motor parar de forma
  inesperada.
- **Servidor do perfil que não conecta:** os servidores reserva agora também entram quando o handshake com o servidor
  do perfil nunca fecha, não só quando ele recusa o Discord depois de conectado.

## O painel

O painel tem três controles:

- **Perfil da VPN:** arraste ou selecione um arquivo WireGuard `.conf`. Cada pessoa usa o perfil da própria VPN. O
  perfil fica só neste computador, numa pasta acessível apenas ao usuário, ao SYSTEM e aos administradores. Perfis
  recusados mostram a causa (seções, chave, endereço, endpoint, sintaxe).
- **Iniciar / Parar:** *Iniciar* fecha o Discord, arma o motor e reabre o Discord. Essa ordem é necessária para a janela
  do gateway, e por isso **uma chamada em andamento cai**. *Parar* desfaz tudo.
- **Iniciar com o Windows:** no logon, o painel roda sem janela, executa a mesma sequência e sai.

Estados mostrados: Parado, Iniciando…, Pronto, Em uso, Parando…, Erro (com o que fazer). Se a internet cair enquanto o
StreamRelay está ativo, o painel mostra **Sem internet** em vez de erro, e volta sozinho quando a rede normalizar. Todo
aviso de erro tem um botão **Copiar diagnóstico**, que copia os registros da execução (nunca o perfil da VPN nem
endereços da rede) para colar numa conversa de suporte. A janela não fecha enquanto o StreamRelay está iniciando,
parando, ou preparando a sessão do Discord; ela avisa o motivo e fecha sozinha ao terminar. Fechar a janela do painel
nos outros momentos não para o StreamRelay.

Ao abrir, o painel consulta uma vez a versão mais recente publicada em [Releases](../../releases). Se ela for mais nova
que a instalada, aparece um aviso com o botão **Atualizar**. Nada é baixado nem instalado sem esse clique. Ao clicar, o
painel baixa o instalador anexado a essa versão, confere se o arquivo bate com o SHA-256 que o GitHub publica para ele e
só então abre o instalador, já no modo de atualização. Se o download falhar, o arquivo não bater ou a permissão do
Windows for recusada, o painel continua aberto e avisa que não foi possível atualizar. Sem internet, o aviso não
aparece.

## Requisitos

- Windows x64.
- Discord para desktop instalado no local padrão do usuário.
- Adaptador físico Ethernet ou Wi-Fi com **um** endereço IPv4.
- Um perfil WireGuard próprio, de qualquer provedor ou servidor compatível.
- Permissão de administrador na instalação (um único UAC).

## Instalação e desinstalação

**Instalar:**
- copia os arquivos para `C:\Program Files\StreamRelay`;
- cria a pasta de dados de execução, gravável só por administradores;
- registra as duas tarefas agendadas (iniciar e parar, sem gatilho próprio);
- cria **uma** regra de firewall de entrada, só para o executável do motor e só no perfil Privado;
- instala o driver NDIS a partir do pacote original assinado (INF/CAT), ou reutiliza uma instalação idêntica;
- pergunta se deve iniciar com o Windows.

Na primeira instalação do driver, pode haver uma breve desconexão da rede e um pedido de reinício.

**Atualizar ou reinstalar** (instalador aberto com o StreamRelay já instalado):
- pergunta uma vez se deve atualizar, reinstalar a mesma versão ou instalar uma versão mais antiga por cima;
- instala por cima, na mesma pasta, sem passar pela desinstalação;
- mantém o perfil importado, a escolha de iniciar com o Windows, o driver e os dados de execução, e não pergunta de
  novo sobre iniciar com o Windows;
- se o motor estiver ativo, pede para pará-lo;
- para instalar do zero, desinstale antes pelas Configurações do Windows.

**Desinstalar:**
- para o motor, se estiver ativo;
- remove as tarefas, a regra de firewall e a entrada de início com o Windows;
- remove o driver só se foi o próprio instalador que o instalou e se identidade e hash conferirem;
- apaga os dados de execução, o perfil importado (que contém a chave privada) e os dados do painel.

## Estado de verificação

**Verificado:**

- Testes automatizados: 8 testes C++ de lógica pura, testes PowerShell do instalador e do launcher (parte deles em
  subconjunto, sem os trechos que exigem sistema vivo) e `cargo test` do
  painel. Todos PASS na base atual.
- Guarda WFP: 8/8 casos PASS num teste elevado com a mesma classe de filtro aplicada ao loopback. Inclui a remoção dos
  filtros quando o processo dono é encerrado à força.
- Com o backend anterior, um driver WFP de terceiros (testes de 11 e 12/10/2026), o relay seletivo funcionou
  com o Discord real:
  - assistindo e transmitindo, uma execução cada;
  - duas chamadas seguidas no modo contínuo;
  - a janela do gateway com uma sessão criada pela conexão direta, uma execução.

  Os resultados visuais são relatos do operador, apoiados pelos contadores do motor (relays iniciados, migrados e
  0 falhas).
- Com o backend NDIS atual:
  - testes próprios de UDP passaram: recepção conectada e não conectada, reuso de porta, parada normal e
    encerramento forçado;
  - a transmissão de tela funcionou com o Discord real;
  - a instalação funcionou num computador que ainda não tinha o driver.
- Servidores reserva: com um servidor da Proton VPN que não deixava o Discord passar, o motor trocou para um servidor
  reserva com a mesma chave e o Discord criou a sessão pelo túnel (uma execução, 15/10/2026). A mesma chave também foi
  aceita em outros servidores gratuitos da Proton VPN.
- Recuperação de rede: com uma transmissão de tela ativa, uma queda de 66 s do cabo Ethernet (voltando na mesma rede)
  foi recuperada sem nenhuma ação manual: a captura de pacotes foi refeita sozinha, a janela do gateway abriu na hora
  certa para a reconexão que o próprio Discord agendou, e a transmissão voltou a passar pelo relay (uma execução,
  15/10/2026).
- Desempenho:
  - com a versão atual, o uso de RAM fica em torno de 20 MB, e o uso de CPU depende do processador;
  - numa medição anterior (12/10/2026), o tempo do launcher até "armado" foi de ≈ 3,5–4 s, com o sistema já aquecido.

**NÃO verificado:**

- Repetição e variância; sessões de mais de 5 minutos; voltar a uma chamada; várias transmissões ou espectadores ao
  mesmo tempo; vazão do relay em picos de mídia.
- Latência adicionada pela captura NDIS durante jogos.
- Parada e limpeza pelo caminho supervisor → watchdog → rollback, e a guarda de exposição disparando com rotas reais.
  A guarda foi testada só em simulação.
- Partida no logon com o sistema frio.
- As 3 tentativas por servidor e a lista de reserva com perfis de outros provedores de VPN.
- Servidor reserva quando o handshake com o servidor do perfil nunca fecha (só o caso de recusa depois de conectado
  foi testado ao vivo).
- Atualização a partir de uma versão realmente mais antiga instalada. O botão **Atualizar** foi testado com um painel
  marcado como versão anterior: baixou e conferiu o instalador publicado e reinstalou por cima da mesma versão.
- Religar sozinho depois do motor parar de forma inesperada, ou depois de uma mudança real de rede (gateway
  diferente); o caminho testado ao vivo foi a mesma rede voltando depois de cair.
- O aviso **Sem internet** enquanto o StreamRelay está ativo, e o botão **Copiar diagnóstico**.

## Limitações conhecidas

- **Só IPv4.**
- **Não funciona junto com outra VPN:** outro cliente de VPN ativo ao mesmo tempo impede o uso. Desconecte as outras
  VPNs antes de clicar em *Iniciar*.
- **Iniciar derruba uma chamada em andamento**, porque o Discord é fechado e reaberto.
- **Uma conta por instalação:** as tarefas ficam ligadas à conta que instalou.
- **Instalador sem assinatura de código:** o SmartScreen vai avisar.
- **Janela do gateway vale para o sistema todo:** durante a janela, as rotas /32 do gateway valem para a máquina
  inteira. Outros programas que tentem esses IPs são bloqueados pela guarda WFP; eles falham, não vazam.
- **IPs do gateway:** o conjunto exposto vem do resolvedor DNS do sistema, e o Discord pode usar outros endereços.
- **Servidores reserva dependem do provedor:** eles usam a chave do seu perfil, então só ajudam se o provedor aceitar a
  mesma chave em vários servidores, como a Proton VPN gratuita aceitou. Uma chave que a VPN não aceita mais não é
  resolvida por eles: gere um perfil novo.
- **Driver permanece instalado:** depois de *Parar*, o driver NDIS continua instalado, com captura e regras desativadas.
- **Formato do Discord:** a classificação depende do formato do log do renderer e do IP Discovery de 74 bytes. Se o
  Discord mudar esse formato, o resultado seguro é liberar pela conexão direta, ou seja, sem relay.
- **Falha dupla:** se o motor travar com uma janela aberta e o supervisor e o watchdog morrerem juntos, nada fecha a
  janela.

## Componentes de terceiros

O instalador inclui componentes de terceiros, cada um sob a sua própria licença:

| Componente | Uso | Licença |
| --- | --- | --- |
| WireGuardNT (`wireguard.dll`) | Adaptador do túnel, usado só pela API pública de `wireguard.h` | WireGuard LLC "Prebuilt Binaries License"; o texto acompanha a DLL |
| SDK `ndisapi` (Windows Packet Filter) | Compilado estaticamente no motor | MIT |
| Driver `ndisrd` do Windows Packet Filter 3.6.2.1 | Captura NDIS, pacote original assinado | EULA do Windows Packet Filter (NT Kernel): freeware para uso pessoal (não comercial) ou educacional |
| Runtime do Microsoft Visual C++ (`vcruntime140`, `vcruntime140_1`, `msvcp140`) | DLLs do runtime ao lado do motor | Distributable Code do Microsoft Visual Studio 2022 |
| Fonte Geist | Embutida no painel | SIL Open Font License 1.1 |

## Aviso

O StreamRelay é um projeto independente, sem afiliação com nenhuma empresa ou projeto citado aqui. Ele não é afiliado,
patrocinado nem endossado pela Discord Inc., pela Microsoft, pela WireGuard LLC, pelos autores do
Windows Packet Filter nem por qualquer provedor de VPN. Esses nomes aparecem só para descrever compatibilidade e
componentes, e pertencem aos seus respectivos donos.

O StreamRelay é gratuito e não comercial: não é vendido, não tem cobrança de nenhum tipo e pode ser usado por qualquer
pessoa sem custo. O código-fonte não é público.

Software experimental, fornecido sem garantia de qualquer tipo.

O uso é por conta e risco de quem usa. O StreamRelay altera o roteamento de rede do computador e instala um driver de
captura de pacotes, e plataformas, serviços, jogos e programas de terceiros podem ter termos de uso ou mecanismos de
proteção que não admitem esse tipo de software. Quem instala é o único responsável por conferir esses termos e por
qualquer consequência do uso, como suspensão ou banimento de conta, bloqueio, perda de acesso ou de dados. O autor não
se responsabiliza por punições, danos ou prejuízos de nenhuma natureza decorrentes do uso do programa, nem por
incompatibilidades com outros softwares.
