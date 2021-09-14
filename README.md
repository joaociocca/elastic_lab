# elastic_lab

Um repositório pra testes com o Elastic Stack.

Aparentemente, alguma coisa mudou nesse tempo que eu deixei o projeto abandonado, então acho que, melhor do que tentar refazer meus passos e descobrir onde tudo deu certo e tudo deu errado, vai ser pegar o aprendizado do que rolou e recomeçar.

O último commit do plano original foi [incluir DumpsterFire nos próximos passos](https://github.com/joaociocca/elastic_lab/commit/5eaa23c2f094aa8489631c7d466e2440fc2a0e71).

## Objetivo

Montar uma base de materiais pra explicar como atuar em defesa e ataque, usando ferramentas como Elastic Stack, Sysmon e outras, tudo com containers.

A ideia inicial era simplesmente explorar coisas pra fazer com o Elastic Stack e documentar os passos. Mas mesmo pra mim, retomando o projeto depois de um tempão de abandono, ficou confuso o que rolou e o que não rolou, ou mesmo como.

## Começando do princípio

Não há motivos para subir múltiplas instâncias de Elasticsearch, ou configurar persistência de keystore. Precisamos de um Elasticsearch e um Kibana rodando - isso já tem todo na [documentação oficial, "Running the Elastic Stack on Docker"](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html).

Mas como só precisamos de um, teremos um `docker-compose.yml` ainda mais simples, mas nomes de hosts "elasticsearch" e "kibana". Vamos usar a rede padrão do Docker pra facilitar o resto do caminho.

## Precisamos ver

Se a ideia é como atacar e defender, precisamos ver o que acontece. O método mais fácil que eu achei até agora foi com [essa resposta do Stack Overflow](https://stackoverflow.com/a/61855965/1985023) pra identificar como capturar o tráfego que tá rolando.

Juntando isso com um container pro Wireshark/Tshark a gente tem a captura dos pacotes. Já conseguimos ver o Kibana conversar com o Elasticsearch, por exemplo. O container do [Wireshark da LinuxServer.io](https://hub.docker.com/r/linuxserver/wireshark) parece bastante suficiente, então a gente joga a configuração dele pra dentro do nosso `docker-compose.yml` e bota pra subir.

### Descendo no buraco do coelho

Mas só subir o container com Wireshark não adianta muita coisa, porque ele só vai ver o que for pra ele! A gente precisa, então, redirecionar o tráfego que vai para os containers que a gente quer monitorar e mandá-los pro container do Wireshark, certo? Como fazer isso?

Mais uma vez, [o Stack Overflow vem ao resgate](https://stackoverflow.com/questions/38740667/docker-traffic-mirroring), podemos usar o `tc` pra fazer isso. Todos containers do Docker ganham interfaces de rede virtuais na máquina hospedeira, então a gente usa ele espelhar o tráfego!

Apenas pra ver se funciona, como só temos os containers do Elasticsearch e do Kibana, vamos espelhar eles.

Pegando da resposta no "Precisamos ver", a gente identifica quais são as interfaces de cada um. No meu exemplo atual, o Elasticsearch é a interface `veth5cbaf23` e o Kibana é a interface `veth30bb21a`, enquanto o Wireshark é a interface `veth6e558d8`. Aplicando [a resposta do Matt](https://stackoverflow.com/a/38747127/1985023), temos os dois comandos, abaixo:

```bash
sudo tc qdisc add dev veth5cbaf23 ingress
sudo tc filter add dev veth5cbaf23 parent ffff: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 1:1 \
  action mirred egress mirror dev vethbe0c796
sudo tc qdisc replace dev veth5cbaf23 parent root handle 10: prio
sudo tc filter add dev veth5cbaf23 parent 10: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 10:1 \
  action mirred egress mirror dev vethbe0c796
```

```bash
sudo tc qdisc add dev veth30bb21a ingress
sudo tc filter add dev veth30bb21a parent ffff: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 1:1 \
  action mirred egress mirror dev vethbe0c796
sudo tc qdisc replace dev veth30bb21a parent root handle 10: prio
sudo tc filter add dev veth30bb21a parent 10: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 10:1 \
  action mirred egress mirror dev vethbe0c796
```

Como a ideia dessa configuração é só pra testar e ver funcionando, depois de feita dá um pulo lá no Kibana (pela configuração padrão estará em localhost:5601) e veja o tráfego aparecendo no Wireshark. Se quiser ler mais sobre Traffic Control, [dá uma olhada nesse artigo](https://medium.com/swlh/traffic-mirroring-with-linux-tc-df4d36116119).

Testamos, vimos que funciona. Bora remover esses redirecionamentos.

```bash
sudo tc filter del dev veth5cbaf23 parent ffff:
sudo tc filter del dev veth5cbaf23 parent 10:
sudo tc filter del dev veth30bb21a parent ffff:
sudo tc filter del dev veth30bb21a parent 10:
```

## Um lugar para atacar

Próximo passo, então, é termos algo para atacar, certo? O "Damn Vulnerable Web Application" também tem uma imagem pro Docker, mas a oficial está bem defasada. Fuçando um pouco mais [eu consegui encontrar uma melhor](https://hub.docker.com/r/cytopia/dvwa), mas que vem cheia de firula - o que nos interessa é o `docker-compose.yml` e o `.env` do [github deles](https://github.com/cytopia/docker-dvwa).

Copiando o arquivo `.env` pro nosso repositório e o que interessa do `docker-compose.yml` pro nosso próprio, fazemos um primeiro teste subindo só esses dois containers. Acessamos `localhost:8000`, logamos com admin/password e precisamos criar o banco na primeira execução.

Rodou bonito? Agora a brincadeira começa...

## Identificando quem é quem, lá dentro

Uma coisa muito importante é sabermos quem é quem, qual IP pertence a qual container, afinal estão todos "lá dentro". Isso é importante porque pra nós vai interessar principalmente um IP: o do DVWA.

Dá pra conseguir listar todo mundo com um comando só:

```bash
docker network ls | grep elastic_lab_default | cut -d' ' -f1 | xargs -I{} sh -c "docker network inspect -f '{{json .Containers}}' {} | jq '.[] | .Name + \":\" + .IPv4Address'"
 ```

Esse comando vai listar as redes do docker, pegar apenas a `elastic_lab_default`, extrair seu ID e repassar ele pro `network inspect`, que gera um JSON. Esse JSON é analizado pelo `jq` e nos exibe lindamente apenas os nomes dos containers e seus IPs:

```plain
"elastic_lab_dvwa_db_1:172.21.0.3/16"
"elastic_lab_dvwa_web_1:172.21.0.5/16"
"kibana:172.21.0.6/16"
"wireshark:172.21.0.4/16"
"elasticsearch:172.21.0.2/16"
```

Agora a gente sabe que a gente quer visualizar as coisas envolvendo o `172.21.0.5`.

## Vendo nossos ataques, básico

Então estamos com Elasticsearch, Kibana, Wireshark e DVWA rodando, certo? O mais simples pra gente começar a ver nossos ataques é simplesmente redirecionar o tráfego do container da DVWA pro Wireshark.

Como eu sou preguiçoso, vamos melhorar isso: a gente precisa pegar o ID do container do DVWA, passar pro `docker exec cat /sys/class/net/eth0/iflink` pra identificar o número da interface de rede, depois pegar esse número e filtrar ele no `ip address show` pra identificar a interface, certo?

```bash
docker ps | grep dvwa_web | cut -d' ' -f 1 | xargs -I{} docker exec {} cat /sys/class/net/eth0/iflink | xargs -I{} sh -c 'ip a|grep "^{}"' | cut -d' ' -f2 | cut -d'@' -f1
```

Dá pra trocar o "dvwa_web" pelo nome de qualquer outro container que você queira rapidamente identificar a interface de rede virtual.

Os dois últimos `cut` servem pra exibir apenas o nome da interface de rede, e mais nada. Sem o que vem depois da arroba, pq pelo menos nos testes daqui quando eu fiz usando @ o `tc` deu erro e ignorou, mas quando fiz sem ele funcionou certinho. Pro meu exemplo, a interface é a `vethc3a204a`.

```bash
sudo tc qdisc add dev vethc3a204a ingress
sudo tc filter add dev vethc3a204a parent ffff: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 1:1 \
  action mirred egress mirror dev vethbe0c796
sudo tc qdisc replace dev vethc3a204a parent root handle 10: prio
sudo tc filter add dev vethc3a204a parent 10: \
  protocol all prio 2 u32 \
  match u32 0 0 flowid 10:1 \
  action mirred egress mirror dev vethbe0c796
```

Pro primeiro exemplo, vamos brincar com [XSS Stored](http://localhost:8000/vulnerabilities/xss_s/) e ver [como fica no Wireshark](http://localhost:3000/)? Abrindo o Wireshark, vamos partir pra capturar apenas o que nos interessa! Além de selecionar a interface `eth0`, inclua o filtro de captura "`host 172.21.0.5`", assim filtramos todo tráfego com o IP do DVWA que vamos atacar.

  ATENÇÃO, filtro de captura NÃO é filtro de exibição! Ele deve ser definido na mesma tela da escolha da interface, logo acima da listagem das interfaces

Um XSS Stored bem simples é `<script>alert(document.cookie)</script>`. Ao incluir essa mensagem e clicar em "Sign Guestbook" a página será recarregada e receberemos um popup nos mostrando o conteúdo do nosso cookie para essa sessão.

Lá no Wireshark a gente consegue visualizar a requisição POST com o conteúdo do ataque XSS e a resposta, já contendo o código malicioso. Podemos repetir isso e visualizar o tráfego de todos os demais ataques ao DVWA, como por exemplo do SQL Injection.

Mas nós queremos mais, certo?

## Vendo nossos ataques no Kibana

Aqui a gente tem vários caminhos a seguira parte complexa é decidir qual...

## ToDo's

* Incluir capturas pra ilustrar as coisas.
