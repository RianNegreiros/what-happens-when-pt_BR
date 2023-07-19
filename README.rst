Este repositório é a versão traduzida para o português do Brasil do recurso de `Alex Waynor`_, `"What happens when..."`_. 

Alex Waynor não está associado a este trabalho nem revisou a tradução.

O que acontece quando...
========================

Este repositório é uma tentativa de responder a velha pergunta da entrevista "O que acontece quando você digita google.com na caixa de endereço do seu navegador e pressiona enter?"

Exceto em vez da história usual, nós vamos tentar responder essa pergunta o mais detalhado possível. Sem pular nada.

Este é um processo colaborativo, então mergulhe e tente ajudar! Há muitos detalhes faltando, apenas esperando você para adiciona-lo! Então nós envie um pull request, por favor!

Tudo isso é licenciado sob os termos da licença `Creative Commons Zero`_.

Leia isto em `简体中文`_ (Chinês simplificado), `日本語`_ (Japonês), `한국어`_
(Koreano), `Espanhol`_ e também a versão original em `Inglês`_.

Tabela de conteúdo
==================

.. contents::
  :backlinks: none
  :local:

A tecla "g" é pressionada
-------------------------

As seções a seguir explicam as ações físicas do teclado
e as interrupções do sistema operacional. Ao pressionar a tecla "g" o navegador recebe o
evento e as funções de preenchimento automático entram em ação.
Dependendo do algoritmo do seu navegador e se você estiver em
modo privado/anônimo ou não, várias sugestões serão apresentadas
para você no menu suspenso abaixo da barra de URL. A maioria desses algoritmos classifica
e priorize os resultados com base no histórico de pesquisa, favoritos, cookies e
pesquisas populares da Internet na totalidade. Enquanto você digita
"google.com" muitos blocos de código serão executados e as sugestões serão refinadas
com cada pressionamento de tecla. Pode até sugerir "google.com" antes de terminar de digitar
isto.

A tecla "enter" toca o fundo
----------------------------

Para escolher um ponto zero, vamos escolher a tecla Enter no teclado pressionando até o ponto mais baixo.
Neste ponto, um circuito elétrico específico para a tecla Enter é fechada (direta ou capacitivamente).
Isso permite uma pequena quantidade de corrente fluir para o circuito lógico do teclado, que verifica o estado de cada interruptor de chave, debounce(processo de eliminação de uma oscilação em alguma coisa) o ruído elétrico do fechamento intermitente rápido eo converte em um número inteiro de código númerico, neste caso 13.
O controlador de teclado então codifica o código númerico para transporte para o computador.
Isso agora é quase universal por meio de um Universal Serial Bus (USB) ou conexão Bluetooth, mas tem historicamente sido por conexões PS/2 ou ADB.

*No caso de um teclado USB:*

- O circuito USB do teclado é alimentado pela fonte de 5V fornecida no pino 1 do controlador de host USB do computador.

- O código númerico gerado é armazenado pela memória interna do circuito do teclado de um registrador chamado "endpoint".

- O controlador USB pesquisa esse "endpoint" a cada ~10ms (valor mínimo declarado pelo teclado), então ele obtém o valor do código númerico armazenado nele.

- Este Valor vai para o USB SIE (Serial Interface Engine) para ser convertido em um ou mais pacotes USB que seguem o protocolo USB de baixo nível.

- Esses pacotes são enviados por um sinal elétrico diferencial sobre pinos D+ e D- (os dois do meio) a uma velocidade máxima de 1,5 Mb/s, como um HID (Dispositivo de Interface Humana) é sempre declarado como um "dispositivo de baixa velocidade" (conformidade do USB 2.0).

- Este sinal é então decodificado no controlador USB host do computador e interpretado pelo driver do teclado do dispositivo de interface humana (HID) do computador. O valor da chave é então passado para a camada de operação de abstração de hardware do sistema.

*No caso do Teclado Virtual (como em dispositivos touch screen):*

- Quando o usuário coloca o dedo em uma tela de toque capacitiva moderna, um pequena quantidade de corrente é transferida para o dedo. Isso completa o circuito através do campo eletrostático da camada condutora e cria uma queda de tensão naquele ponto da tela. O ``controlador de tela`` então gera uma interrupção relatando a coordenada de pressionamento de tecla.

- Em seguida, o sistema operacional móvel notifica o aplicativo atualmente focado em um evento de pressionamento em um de seus elementos GUI (que agora é o aplicativo de teclado virtual botões).

- O teclado virtual agora pode gerar uma interrupção de software para enviar um mensagem 'tecla pressionada' de volta para o sistema operacional.

- Esta interrupção notifica o aplicativo atualmente focado de um evento 'tecla pressionada'.

Disparos de interrupção [NÃO para teclados USB]
-----------------------------------------------

O teclado envia sinais em sua linha de solicitação de interrupção (IRQ), que é mapeada para um ``vetor de interrupção`` (inteiro) pelo controlador de interrupção.
A CPU usa a ``Interrupt Descriptor Table`` (IDT) para mapear os vetores de interrupção para funções (``interrupt handlers``) fornecidas pelo kernel. Quando um interrupção chega, a CPU indexa o IDT com o vetor de interrupção e executa o manipulador apropriado. Assim, o kernel é inserido.

(No Windows) Uma mensagem ``WM_KEYDOWN`` é enviada para o aplicativo
--------------------------------------------------------------------

O transporte HID passa o evento key down para o driver ``KBDHID.sys`` que converte o uso HID em um scancode. Neste caso, o código de varredura é ``VK_RETURN`` (``0x0D``).
O driver ``KBDHID.sys`` interage com o ``KBDCLASS.sys`` (driver de classe de teclado).
Este motorista é responsável por lidar com todas as entradas de teclado e teclado de maneira segura.
Em seguida, chama em ``Win32K.sys`` (após potencialmente passar a mensagem por terceiros filtros de teclado instalados). Isso tudo acontece no modo kernel.

``Win32K.sys`` descobre qual janela é a janela ativa através do API ``GetForegroundWindow()``.
Esta API fornece o identificador de janela do caixa de endereço do navegador. A "bomba de mensagem" principal do Windows então chama ``SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)``.
``lParam`` é uma máscara de bitsque indica mais informações sobre o pressionamento de tecla: contagem de repetições (0 neste caso), o código de varredura real (pode ser dependente do OEM, mas geralmente não seria para ``VK_RETURN``), se teclas estendidas (por exemplo, alt, shift, ctrl) também foram pressionados (eles não estavam) e algum outro estado.

A API ``SendMessage`` do Windows é uma função direta que adiciona a mensagem a uma fila para o manipulador de janela específico (``hWnd``).
Mais tarde, a principal função de processamento de mensagens (chamada ``WindowProc``) atribuída para o ``hWnd`` é chamado para processar cada mensagem na fila.

A janela (``hWnd``) que está ativa é, na verdade, um controle de edição e o ``WindowProc`` neste caso tem um manipulador de mensagens para mensagens ``WM_KEYDOWN``.
Este código procura dentro do 3º parâmetro passado para ``SendMessage`` (``wParam``) e, por ser ``VK_RETURN`` sabe que o usuário apertou a tecla ENTER.

(No OS X) Um NSEvent ``KeyDown`` é enviado para o aplicativo
------------------------------------------------------------

O sinal de interrupção aciona um evento de interrupção no teclado I/O Kit kext motorista.
O motorista traduz o sinal em um código-chave passado para o Processo ``WindowServer`` do OS X. 
Como resultado, o ``WindowServer`` despacha um evento para quaisquer aplicativos apropriados (por exemplo, ativos ou de escuta) por meio de seus Mach port onde é colocado em uma fila de eventos. 
Os eventos podem então ser lidos de esta fila por threads com privilégios suficientes chamando o função ``mach_ipc_dispatch``.
Isso ocorre mais comumente por, e é tratado por um loop de evento principal ``NSApplication``, via um ``NSEvent`` de ``NSEventType`` ``KeyDown``.

(No GNU/Linux) o servidor Xorg escuta os códigos-chave
------------------------------------------------------

Quando um ``servidor X`` gráfico é usado, o ``X`` usará o evento genérico driver ``evdev`` para obter o pressionamento de tecla.
Um remapeamento de códigos-chave para códigos de varredura é feito com mapas de teclas e regras específicas do ``servidor X``.
Quando o mapeamento do scancode da tecla pressionada estiver completo, o ``servidor X`` envia o caractere para o ``gerenciador de janelas`` (DWM, metacity, i3, etc), para que o ``gerenciador de janelas``, por sua vez, envia o caractere para a janela em foco.
A API gráfica da janela que recebe o caractere imprime o símbolo de fonte apropriado no campo de foco apropriado.

Analisar URL
------------

* O navegador agora tem as seguintes informações contidas na URL (Uniform Resource Locator):

    - ``Protocol``  "http"
        Usa 'Hyper Text Transfer Protocol'

    - ``Resource``  "/"
        Busca a página principal (index)

É um URL ou uma pesquisa?
--------------------------------

Quando nenhum protocolo ou nome de domínio válido é fornecido, o navegador continua a alimentar o texto fornecido na caixa de endereço para o mecanismo de pesquisa padrão do navegador.
Em muitos casos, o URL tem um texto especial anexado a ele para informar ao mecanismo de pesquisa que veio da barra de URL de um navegador específico.

Converter caracteres Unicode não ASCII no nome do host
------------------------------------------------------

* O navegador verifica o nome do host em busca de caracteres que não estão em ``a-z``, ``A-Z``, ``0-9``, ``-``, or ``.``.

* Como o nome do host é ``google.com``, não haverá nenhum, mas se houver o navegador aplicaria a codificação `Punycode`_ à parte do nome do host do URL.

Verifique a lista de HSTS
-------------------------

* O navegador verifica seu "HSTS pré-carregado (HTTP Strict Transport Security)" lista.
Esta é uma lista de sites que solicitaram contato via apenas HTTPS.

* Se o site estiver na lista, o navegador envia sua solicitação via HTTPS em vez de HTTP.
  Caso contrário, a solicitação inicial é enviada via HTTP.
  (Observe que um site ainda pode usar a política HSTS *sem* estar no Lista HTS.
  A primeira solicitação HTTP para o site por um usuário receberá um resposta solicitando que o usuário envie apenas solicitações HTTPS.
  No entanto, isso uma única solicitação HTTP pode potencialmente deixar o usuário vulnerável a um `ataque de downgrade`_, e é por isso que a lista HSTS está incluída na web moderna navegadores.)

Pesquisa de DNS
---------------

* O navegador verifica se o domínio está em seu cache. (para ver o DNS Cache em Chrome, vá para `chrome://net-internals/#dns <chrome://net-internals/#dns>`_).
* Se não for encontrado, o navegador chama a função da biblioteca ``gethostbyname`` (varia de acordo com OS) para fazer a pesquisa.
* ``gethostbyname`` verifica se o hostname pode ser resolvido por referência no arquivo ``hosts`` local (cuja localização `varia conforme o SO`_) antes de tentar resolva o nome do host por meio do DNS.
* Se ``gethostbyname`` não o tiver armazenado em cache nem puder encontrá-lo nos ``hosts`` arquivo então ele faz uma requisição ao servidor DNS configurado na rede pilha. Normalmente, é o roteador local ou o servidor DNS de cache do ISP.
* Se o servidor DNS estiver na mesma sub-rede, a biblioteca de rede segue o ``ARP process`` abaixo para o servidor DNS.
* Se o servidor DNS estiver em uma sub-rede diferente, a biblioteca de rede segue o ``processo ARP`` abaixo para o IP do gateway padrão.

Processo ARP
------------

Para enviar um ARP (Protocolo de Resolução de Endereço) difundir a rede a biblioteca de pilha precisa do endereço IP de destino para pesquisar.
Ele também precisa saber o endereço MAC da interface que ele usará para enviar o broadcast ARP.

O cache ARP é verificado primeiro quanto a uma entrada ARP para nosso IP de destino.
Se estiver em o cache, a função da biblioteca retorna o resultado: Target IP = MAC.

Se a entrada não estiver no cache ARP:

* A tabela de rotas é consultada para ver se o endereço IP de destino está em algum dos as sub-redes na tabela de rotas locais. 
Se for, a biblioteca usa o interface associada a essa sub-rede. Se não for, a biblioteca usa o interface que possui a sub-rede do nosso gateway padrão.

* O endereço MAC da interface de rede selecionada é pesquisado.

* A biblioteca de rede envia uma camada 2 (camada de enlace de dados do `modelo OSI`_) Solicitação ARP:

``Requisição ARP``::

    Sender MAC: interface:mac:address:here
    Sender IP: interface.ip.goes.here
    Target MAC: FF:FF:FF:FF:FF:FF (Broadcast)
    Target IP: target.ip.goes.here

Dependendo do tipo de hardware entre o computador e o roteador:

Diretamente conectado:

* Se o computador estiver conectado diretamente ao roteador, a resposta do roteador com uma ``Resposta ARP`` (veja abaixo)

Hub:

* Se o computador estiver conectado a um hub, o hub transmitirá o requisição ARP de todas as outras portas. Se o roteador estiver conectado no mesmo "fio", ele responderá com uma ``Resposta ARP`` (veja abaixo).

Switch:

* Se o computador estiver conectado a um switch, o switch verificará seu local tabela CAM/MAC para ver qual porta tem o endereço MAC que estamos procurando.
  Se o switch não tem entrada para o endereço MAC, ele irá retransmitir o ARP requisição para todas as outras portas.

* Se o switch tiver uma entrada na tabela MAC/CAM, ele enviará a solicitação ARP para a porta, com o endereço MAC que estamos procurando.

* Se o roteador estiver no mesmo "fio", ele responderá com um ``Resposta ARP`` (Veja abaixo)

``Resposta ARP``::

    Sender MAC: target:mac:address:here
    Sender IP: target.ip.goes.here
    Target MAC: interface:mac:address:here
    Target IP: interface.ip.goes.here

Agora que a biblioteca de rede tem o endereço IP do nosso servidor DNS ou
o gateway padrão pode retomar seu processo de DNS:

* O cliente DNS estabelece um soquete para a porta UDP 53 no servidor DNS, usando uma porta de origem acima de 1023.
* Se o tamanho da resposta for muito grande, o TCP será usado.
* Se o servidor DNS local/ISP não o tiver, uma pesquisa recursiva será solicitado, que flui pela lista de servidores DNS até que o SOA seja alcançado, e se encontrada, uma resposta é retornada.

Abertura de um socket
---------------------

Depois que o navegador recebe o endereço IP do servidor de destino, ele leva isso e o número de porta fornecido no URL (o padrão do protocolo HTTP é a porta 80 e HTTPS para a porta 443) e faz uma chamada para a função da biblioteca do sistema chamado ``socket`` e solicita um fluxo de soquete TCP - ``AF_INET/AF_INET6`` e ``SOCK_STREAM``.

* Esta solicitação é passada primeiro para a Camada de Transporte onde um segmento TCP é trabalhado.
  A porta de destino é adicionada ao cabeçalho e uma porta de origem é escolhido dentro do intervalo de portas dinâmicas do kernel (ip_local_port_range em Linux).

* Este segmento é enviado para a Camada de Rede, que envolve um IP adicional cabeçalho.
  O endereço IP do servidor de destino, bem como o do máquina atual é inserida para formar um pacote.

* O próximo pacote chega na Camada de Enlace. Um cabeçalho de quadro é adicionado que inclui o endereço MAC do NIC da máquina, bem como o endereço MAC do o gateway (roteador local).
  Como antes, se o kernel não conhece o MAC endereço do gateway, ele deve transmitir uma consulta ARP para localizá-lo.

Neste ponto, o pacote está pronto para ser transmitido através de:

* `Ethernet`_
* `WiFi`_
* `Cellular data network`_

Para a maioria das conexões de Internet domésticas ou de pequenas empresas, o pacote passará de seu computador, possivelmente por meio de uma rede local e, em seguida, por um modem (MOdulator/DEModulator) que converte 1's e 0's digitais em um analógico sinal adequado para transmissão por telefone, cabo ou telefonia sem fio conexões.
Na outra ponta da conexão está outro modem que converte o sinal analógico de volta em dados digitais para serem processados pela próxima rede node`_ onde os endereços de e para seriam analisados posteriormente.

A maioria das empresas maiores e algumas conexões residenciais mais recentes terão fibra
ou conexões Ethernet diretas, caso em que os dados permanecem digitais e
é passado diretamente para o próximo `network node`_  para processamento.

Eventualmente, o pacote chegará ao roteador que gerencia a sub-rede local. De
lá, continuará a viajar para a fronteira do sistema autônomo (AS)
roteadores, outros ASes e, finalmente, para o servidor de destino. Cada roteador ao longo
the way extrai o endereço de destino do cabeçalho IP e o encaminha para
o próximo salto apropriado. O campo time to live (TTL) no cabeçalho IP é
decrementado em um para cada roteador que passa. O pacote será descartado se
o campo TTL chegar a zero ou se o roteador atual não tiver espaço em sua fila
(talvez devido ao congestionamento da rede).

Este envio e recebimento acontece várias vezes seguindo o fluxo de conexão TCP:

* O cliente escolhe um número de sequência inicial (ISN) e envia o pacote para o servidor com o bit SYN definido para indicar que está configurando o ISN
* Servidor recebe SYN e se estiver de bom humor:
    * O servidor escolhe seu próprio número de sequência inicial
    * O servidor define SYN para indicar que está escolhendo seu ISN
    * O servidor copia o (cliente ISN +1) para seu campo ACK e adiciona o sinalizador ACK para indicar que está acusando o recebimento do primeiro pacote
* O cliente reconhece a conexão enviando um pacote:
    * Aumenta seu próprio número de sequência
    * Aumenta o número de confirmação do receptor
    * Define o campo ACK
* Os dados são transferidos da seguinte forma:
    * Como um lado envia N bytes de dados, ele aumenta seu SEQ por esse número
    * Quando o outro lado confirma o recebimento daquele pacote (ou uma string de pacotes), ele envia um pacote ACK com o valor ACK igual ao último sequência recebida do outro
* Para fechar a conexão:
    * O mais próximo envia um pacote FIN
    * O outro lado confirma o pacote FIN e envia seu próprio FIN
    * O mais próximo reconhece o FIN do outro lado com um ACK

Aperto de mão TLS
-----------------

* O computador cliente envia uma mensagem ``ClientHello`` para o servidor com seu versão do Transport Layer Security (TLS), lista de algoritmos de cifra e métodos de compressão disponíveis.

* O servidor responde com uma mensagem ``ServerHello`` ao cliente com o versão TLS, cifra selecionada, métodos de compactação selecionados e o servidor certificado público assinado por uma CA (Autoridade Certificadora).
  O certificado contém uma chave pública que será usada pelo cliente para criptografar o restante o aperto de mão até que uma chave simétrica possa ser acordada.

* O cliente verifica o certificado digital do servidor em relação à sua lista de CAs confiáveis. Se a confiança puder ser estabelecida com base na CA, o cliente gera uma string de bytes pseudo-aleatórios e criptografa isso com o servidor chave pública.
  Esses bytes aleatórios podem ser usados para determinar a chave simétrica.

* O servidor descriptografa os bytes aleatórios usando sua chave privada e os usa bytes para gerar sua própria cópia da chave mestra simétrica.

* O cliente envia uma mensagem ``Finished`` para o servidor, criptografando um hash da transmissão até este ponto com a chave simétrica.

* O servidor gera seu próprio hash e, em seguida, descriptografa o hash enviado pelo cliente para verificar se corresponde.
  Em caso afirmativo, ele envia sua própria mensagem ``Finished`` para o cliente, também criptografado com a chave simétrica.

* A partir de agora a sessão TLS transmite os dados do aplicativo (HTTP) criptografados com a chave simétrica acordada.

Se um pacote for descartado
---------------------------

Às vezes, devido ao congestionamento da rede ou conexões de hardware instáveis, os pacotes TLS serão descartados antes de chegarem ao seu destino.
O remetente então tem para decidir como reagir.
O algoritmo para isso é chamado `TCP congestion control`_.
Isso varia dependendo do remetente; os algoritmos mais comuns são `cubic`_ em sistemas operacionais mais recentes e `New Reno`_ em quase todos os outros.

* O cliente escolhe uma `congestion window`_ com base no `maximum segment size`_ (MSS) da conexão.
* Para cada pacote confirmado, a janela dobra de tamanho até atingir o 'limiar de início lento'. Em algumas implementações, esse limite é adaptativo.
* Após atingir o limite de início lento, a janela aumenta aditivamente para cada pacote reconhecido. Se um pacote for descartado, a janela reduz exponencialmente até que outro pacote seja reconhecido.

Protocolo HTTP
--------------

Se o navegador da Web usado foi escrito pelo Google, em vez de enviar um HTTP solicitação para recuperar a página, ele enviará uma solicitação para tentar negociar com o servidor um "upgrade" de HTTP para o protocolo SPDY.

Se o cliente estiver usando o protocolo HTTP e não suportar SPDY, ele enviará um requisição ao servidor do formulário:

    GET / HTTP/1.1
    Host: google.com
    Connection: close
    [other headers]

Onde ``[other headers]`` refere-se a uma série de pares chave-valor separados por dois pontos
formatado de acordo com a especificação HTTP e separado por novas linhas únicas.
(Isso pressupõe que o navegador da Web usado não possui nenhum bug que viole o
Especificação HTTP. Isso também assume que o navegador da web está usando ``HTTP/1.1``,
caso contrário, pode não incluir o cabeçalho ``Host`` na solicitação e a versão
especificado na requisição ``GET`` será ``HTTP/1.0`` ou ``HTTP/0.9``.)

HTTP/1.1 define a opção de conexão "fechar" para o remetente sinalizar que a conexão será fechada após a conclusão da resposta. Por exemplo,

    Connection: close

Aplicativos HTTP/1.1 que não suportam conexões persistentes DEVEM incluir a opção de conexão "fechar" em todas as mensagens.

Após enviar a solicitação e os cabeçalhos, o navegador da Web envia um único nova linha para o servidor indicando que o conteúdo da solicitação está concluído.

O servidor responde com um código de resposta indicando o status da solicitação e responde com uma resposta do formulário:

    200 OK
    [response headers]

Seguido por uma única nova linha e, em seguida, envia uma carga útil do conteúdo HTML de ``www.google.com``.
O servidor pode fechar a conexão ou, se cabeçalhos enviados pelo cliente solicitado, mantenha a conexão aberta para ser reutilizada para mais solicitações.

Se os cabeçalhos HTTP enviados pelo navegador da Web incluírem informações suficientes para o servidor web para determinar se a versão do arquivo armazenado em cache pela web navegador não foi modificado desde a última recuperação (ou seja, se o navegador da web incluiu um cabeçalho ``ETag``), ele pode responder com uma solicitação de a forma:

    304 Not Modified
    [response headers]

E nenhuma carga útil, e o navegador da Web, em vez disso, recupera o HTML de seu cache.

Após analisar o HTML, o navegador da Web (e servidor) repete esse processo para cada recurso (imagem, CSS, favicon.ico, etc) referenciado pela página HTML, exceto que ao invés de ``GET / HTTP/1.1`` a requisição será ``GET /$(URL relativo a www.google.com) HTTP/1.1``.

Se o HTML fizer referência a um recurso em um domínio diferente do ``www.google.com``, o navegador web volta para as etapas envolvidas na resolvendo o outro domínio, e segue todos os passos até este ponto para aquele domínio.
O cabeçalho ``Host`` no pedido será definido para o apropriado nome do servidor em vez de ``google.com``.

Manusear solicitação do servidor HTTP
-------------------------------------

O servidor HTTPD (HTTP Daemon) é aquele que manuseá com as solicitações/respostas no o lado do servidor.
Os servidores HTTPD mais comuns são Apache ou nginx para Linux e IIS para Windows.

* O HTTPD (HTTP Daemon) recebe a solicitação.
* O servidor divide a solicitação nos seguintes parâmetros:
    * Método de solicitação HTTP (``GET``, ``HEAD``, ``POST``, ``PUT``, ``PATCH``, ``DELETE``, ``CONNECT``, ``OPTIONS`` ou ``TRACE``). No caso de uma URL inserida diretamente na barra de endereço, será ``GET``.
    * Domínio, neste caso - google.com.
    * Caminho/página solicitado, neste caso - / (já que nenhum caminho/página específico foi solicitado, / é o caminho padrão).
* O servidor verifica se existe um Virtual Host configurado no servidor que corresponde a google.com.
* O servidor verifica se google.com pode aceitar solicitações GET.
* O servidor verifica se o cliente tem permissão para usar este método (por IP, autenticação, etc.).
* Se o servidor tiver um módulo de reescrita instalado (como mod_rewrite para Apache ou reescrita de URL para IIS), ele tenta corresponder a solicitação a um dos regras configuradas.
  Se uma regra correspondente for encontrada, o servidor usa essa regra para reescrever o pedido.
* O servidor vai puxar o conteúdo que corresponde ao pedido, no nosso caso, ele retornará ao arquivo de índice, pois "/" é o arquivo principal (alguns casos podem substituir isso, mas esse é o método mais comum).
* O servidor analisa o arquivo de acordo com o manipulador. Se o Google está rodando em PHP, o servidor usa PHP para interpretar o arquivo de índice e transmite a saída para o cliente.

Por baixo dos panos do Navegador
---------------------------------

Uma vez que o servidor forneça os recursos (HTML, CSS, JS, imagens, etc.)
para o navegador ele passa pelo processo abaixo:

* Análise - HTML, CSS, JS
* Renderização - Construir Árvore DOM → Árvore de Renderização → Layout da Árvore de Renderização → Pintando a árvore de renderização

Navegador
----------

A funcionalidade do navegador é apresentar o recurso da web que você escolher, por solicitando-o do servidor e exibindo-o na janela do navegador.
O recurso geralmente é um documento HTML, mas também pode ser um PDF, imagem ou algum outro tipo de conteúdo. A localização do recurso é especificado pelo usuário usando um URI (Uniform Resource Identifier).

A maneira como o navegador interpreta e exibe arquivos HTML é especificada nas especificações HTML e CSS. Estas especificações são mantidas pela organização W3C (World Wide Web Consortium), que é a organização de padrões para a web.

As interfaces de usuário do navegador têm muito em comum umas com as outras. Entre o
elementos comuns da interface do usuário são:

* Uma barra de endereço para inserir um URI
* Botões de voltar e avançar
* Opções de marcação
* Atualizar e parar botões para atualizar ou interromper o carregamento de documentos atuais
* Botão Home que leva você à sua página inicial

**Estrutura de alto nível do navegador**

Os componentes dos navegadores são:

* **Interface do usuário:** A interface do usuário inclui a barra de endereços, botão voltar/avançar, menu de favoritos, etc.
  Todas as partes do navegador exibir, exceto a janela onde você vê a página solicitada.
* **Mecanismo do navegador:** o mecanismo do navegador organiza ações entre a IU e o mecanismo de renderização.
* **Mecanismo de renderização:** O mecanismo de renderização é responsável por exibir conteúdo solicitado. Por exemplo, se o conteúdo solicitado for HTML, o mecanismo de renderização analisa HTML e CSS e exibe o conteúdo analisado em a tela.
* **Rede:** a rede lida com chamadas de rede, como solicitações HTTP, usando diferentes implementações para diferentes plataformas por trás de um interface independente de plataforma.
* **Interface da interface do usuário:** a infraestrutura da interface do usuário é usada para desenhar widgets básicos como combinação caixas e janelas.
  Este back-end expõe uma interface genérica que não é específico da plataforma.
  Embaixo, ele usa métodos de interface do usuário do sistema operacional.
* **Mecanismo JavaScript:** O mecanismo JavaScript é usado para analisar e executar o código JavaScript.
* **Armazenamento de dados:** O armazenamento de dados é uma camada de persistência.
  O navegador pode precisa salvar todos os tipos de dados localmente, como cookies. Navegadores também suporte a mecanismos de armazenamento como localStorage, IndexedDB, WebSQL e sistema de arquivo.

Análise de HTML
---------------

O mecanismo de renderização começa a obter o conteúdo do solicitado
documento da camada de rede. Isso será geralmente feito em blocos de 8kB.

A principal tarefa do analisador HTML é analisar a marcação HTML em uma árvore de análise.

A árvore de saída (a "árvore de análise") é uma árvore de elementos e atributos DOM nós. DOM é a abreviação de Document Object Model.
É a apresentação do objeto do documento HTML e a interface dos elementos HTML para o mundo exterior como JavaScript.
A raiz da árvore é o objeto "Documento".
Antes de qualquer manipulação via script, o DOM tem uma relação quase um-para-um com a marcação.

**O algoritmo de análise**

O HTML não pode ser analisado usando os analisadores regulares de cima para baixo ou de baixo para cima.

As razões são:

* A natureza misericordiosa da linguagem.
* O fato de os navegadores terem tolerância a erros tradicional para suportar bem casos conhecidos de HTML inválido.
* O processo de análise é reentrante. Para outros idiomas, a fonte não mudar durante a análise, mas em HTML, código dinâmico (como elementos de script contendo chamadas `document.write()`) pode adicionar tokens extras, então a análise processo realmente modifica a entrada.

Incapaz de usar as técnicas de análise regulares, o navegador utiliza um parser para analisar HTML.
O algoritmo de análise é descrito em detalhes pela especificação HTML5.

O algoritmo consiste em duas etapas: tokenização e construção da árvore.

**Ações quando a análise é concluída**

O navegador começa a buscar recursos externos vinculados à página (CSS, imagens, arquivos JavaScript, etc.).

Nesta fase, o navegador marca o documento como interativo e inicia scripts de análise no modo "adiado": aqueles que devem ser executado após o documento ser analisado.
O estado do documento é definido como "complete" e um evento "load" é disparado.

Observe que nunca há um erro de "sintaxe inválida" em uma página HTML.
Correção de navegadores qualquer conteúdo inválido e continue.

Interpretação do CSS
------------------

* Analisar arquivos CSS, conteúdo da tag ``<style>`` e atributo ``style`` valores usando `"CSS lexical and syntax grammar"`_
* Cada arquivo CSS é analisado em um ``objeto StyleSheet``, onde cada objeto contém regras CSS com seletores e objetos correspondentes à gramática CSS.
* Um analisador CSS pode ser de cima para baixo ou de baixo para cima quando um gerador de analisador específico é usado.

Renderização de página
----------------------

* Crie uma 'Frame Tree' ou 'Render Tree' percorrendo os nós DOM e calculando os valores de estilo CSS para cada nó.
* Calcule a largura preferencial de cada nó na 'Frame Tree' de baixo para cima somando a largura preferencial dos nós filhos e a largura do nó margens horizontais, bordas e preenchimento.
* Calcule a largura real de cada nó de cima para baixo, alocando cada nó largura disponível para seus filhos.
* Calcule a altura de cada nó de baixo para cima aplicando quebra automática de texto e somando as alturas do nó filho e as margens, bordas e preenchimento do nó.
* Calcule as coordenadas de cada nó usando as informações calculadas acima.
* Etapas mais complicadas são executadas quando os elementos são ``floated``, posicionado ``absolutely`` ou ``relatively``, ou outros recursos complexos são usados. Ver http://dev.w3.org/csswg/css2/ e http://www.w3.org/Style/CSS/current-work para mais detalhes.
* Crie camadas para descrever quais partes da página podem ser animadas como um grupo sem ser re-rasterizado. Cada quadro/objeto de renderização é atribuído a uma camada.
* As texturas são alocadas para cada camada da página.
* Os objetos de moldura/renderização para cada camada são percorridos e os comandos de desenho são executados para sua respectiva camada. Isso pode ser rasterizado pela CPU ou desenhado diretamente na GPU usando D2D/SkiaGL.
* Todas as etapas acima podem reutilizar valores calculados da última vez que o página da Web foi renderizada, de modo que as alterações incrementais exijam menos trabalho.
* As camadas da página são enviadas para o processo de composição onde são combinadas com camadas para outro conteúdo visível como o navegador chrome, iframes e painéis adicionais.
* As posições finais da camada são calculadas e os comandos compostos são emitidos via Direct3D/OpenGL. O(s) buffer(s) de comando da GPU são liberados para a GPU para renderização assíncrona e o quadro é enviado para o servidor de janela.

Renderização de GPU
--------------------

* Durante o processo de renderização, as camadas de computação gráfica podem usar propósito ``CPU`` ou o processador gráfico ``GPU`` também.

* Ao usar ``GPU`` para cálculos de renderização gráfica, o as camadas de software dividem a tarefa em várias partes, para que possa aproveitar de paralelismo maciço ``GPU`` para cálculos de ponto flutuante necessários para o processo de renderização.

Windows Server
-------------

Pós-renderização e execução induzida pelo usuário
-------------------------------------------------

Após a conclusão da renderização, o navegador executa o código JavaScript como resultado de algum mecanismo de temporização (como uma animação Google Doodle) ou usuário interação (digitando uma consulta na caixa de pesquisa e recebendo sugestões).
Plugins como Flash ou Java também podem ser executados, embora não neste momento em a página inicial do Google. Os scripts podem fazer com que solicitações de rede adicionais sejam executado, bem como modificar a página ou seu layout, causando outra rodada de renderização e pintura de páginas.

.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"CSS lexical and syntax grammar"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://pt.wikipedia.org/wiki/Punycode
.. _`Ethernet`: https://pt.wikipedia.org/wiki/Ethernet
.. _`WiFi`: https://pt.wikipedia.org/wiki/IEEE_802.11
.. _`Cellular data network`: https://www.industrialethernetu.com/courses/ie504.html
.. _`analog-to-digital converter`: https://pt.wikipedia.org/wiki/Conversor_anal%C3%B3gico-digital
.. _`network node`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
.. _`TCP congestion control`: https://en.wikipedia.org/wiki/TCP_congestion_control
.. _`cubic`: https://en.wikipedia.org/wiki/CUBIC_TCP
.. _`New Reno`: https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_New_Reno
.. _`congestion window`: https://en.wikipedia.org/wiki/TCP_congestion_control#Congestion_window
.. _`maximum segment size`: https://pt.wikipedia.org/wiki/MSS
.. _`varia conforme o SO` : https://pt.wikipedia.org/wiki/Hosts_(arquivo)#Localiza%C3%A7%C3%A3o_do_arquivo_hosts
.. _`简体中文`: https://github.com/skyline75489/what-happens-when-zh_CN
.. _`한국어`: https://github.com/SantonyChoi/what-happens-when-KR
.. _`日本語`: https://github.com/tettttsuo/what-happens-when-JA
.. _`ataque de downgrade`: https://en.wikipedia.org/wiki/SSL_stripping#Research
.. _`Modelo OSI`: https://pt.wikipedia.org/wiki/Modelo_OSI
.. _`Espanhol`: https://github.com/gonzaleztroyano/what-happens-when-ES
.. _`Inglês`: https://github.com/alex/what-happens-when
.. _`"What happens when..."`: https://github.com/alex/what-happens-when
.. _`Alex Waynor`: https://github.com/alex
