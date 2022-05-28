# Compensacão de Lag

O cenário que temos até agora parece funcionar muito bem para percebermos movimentações, pois temos:

* Dado um tempo n, nosso servidor recebe informações de todos os clientes.
* Servidor processa todas as informações e transmite as atualizações.
* Estas atualizações são periódicas e de baixa frequência.
* Clientes enviam informações e verificam seus efeitos localmente.
* Clientes recebem as atualizações de estado do jogo:
    1. Reconciliam com os efeitos que previram.
    2. Interpolam os efeitos dos outros personagens.
* Cliente se ve no presente, mas ve os outros cliente no passado.

Esta situação é geralmente ótima, a menos quando precisamos garantir situações como um tiro na cabeça, que qualquer pequena variação pode causa um erro, pois as informações de tempo e espaço são muito sensíveis. É ai que entra a compensação de lag.

Imagine o cenário na qual você é uma sniper mirando perfeitamente na cabeça de um personagem "imóvel", um tiro dificil de errar. Você atira e, magicamente, nada acontece. Você se irrita, sai da partida e desliga o jogo pensando como pode ter errado aquele tiro perfeito e, pior, a pessoa que você devia ter matado te matou. Este é o efeito de lag temporal, pois seu tiro ocorreu em um personagem que estava 100 ms no passado, para quem gosta de física, é como se a velocidade da luz fosse muito muito muito inferior a que realmente é. Felizmente, existem algumas estratégias para resolver este efeito. Vamos detalhar como isso pode ser reolvido:

1. Você deu um tiro, seu cliente enviou as informações para o servidor, mas desta vez enviou mais informacões além do botão que você clicou, pois eviou o botão que você aperto, o exato momento temporal que você apertou o botão (e se o botão de mira estava sendo apertado) e o que estava exatamente em sua mira neste instante.
2. Como o servidor está recebendo todos momentos temporais, ele pode reconstruir os eventos temporalmente ordenados, ou seja, o servidor pode reconstruir o mundo no exato momento de seu tiro, assim como para todos outros clientes.
3. Sabendo o que sua arma estava mirando no momento de seu tiro, a cabeça de seu inimigo, seu presente passa a ser considerado como válido no servidor, já que ele compensa está diferenca.
4. O servidor processa o tiro e transmite para todos os clientes, deixado seu oponente furioso por ter levado um headshot.

E é no passo dois que a compensacão de lag ocorre.

## Conclusão

Primeira coisa que fizemos foi entender qual o grande problema do desenvolvimento de servidores para jogos, pessoas querendo trapacear, e a partir disso entendemos qual a soluacão básica, um cliente que só envia comandos pro servidor e um servidor autoritário. Vimos que com um servidor autoritátio alguns problemas de defasamento temporal pode ocorrer entre a informação que temos e a informação que o servidor nos obriga a ter. Para reduzir estes problemas aprendemos as técncias de predição e de reconciliação, mas descobrimos problemas de sincronização com outros clientes. Para resolver os problemas de sincronização aprendemos as técncias de dead reckoning e interpolação de entidades, que são ótimas técnicas, mas ainda podem falhar na hora que ações muito sensíveis espacialmente são executadas. Para resolver este problema de ações sensíveis, aprendemos compensação de lag, mas ainda nos falta por a mão na massa. Nos próximos capítulos vamos explorar um jogo simples de tiro e um exemplo de servidor para ele.