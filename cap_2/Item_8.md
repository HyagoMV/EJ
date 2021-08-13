# Item 8: Evite finalizadores e produtos de limpeza

**Finalizadores são imprevisíveis, muitas vezes perigosos e geralmente desnecessários**. Seu uso pode causar comportamento errático, baixo desempenho e problemas de portabilidade. Os finalizadores têm alguns usos válidos, que abordaremos posteriormente neste item, mas como regra, você deve evitá-los. A partir do Java 9, os finalizadores foram descontinuados, mas ainda estão sendo usados ​​pelas bibliotecas Java. A substituição do Java 9 para finalizadores são os *cleaners*. **Os *cleaners* são menos perigosos do que os finalizadores, mas ainda assim imprevisíveis, lentos e geralmente desnecessários**.

Os programadores C++ são advertidos a não pensar nos finalizadores ou *cleaners* como análogos do Java aos destruidores C++. Em C++, destruidores são a maneira normal de recuperar os recursos associados a um objeto, uma contraparte necessária para os construtores. Em Java, o coletor de lixo recupera o armazenamento associado a um objeto quando ele se torna inacessível, não exigindo nenhum esforço especial por parte do programador. Os destruidores C++ também são usados ​​para recuperar outros recursos que não são de memória. Em Java, um bloco `try-with-resources` ou `try-finally` é usado para esse propósito [item 9]().

Uma deficiência dos finalizadores e *cleaners* é que não há garantia de que eles serão executados prontamente [JLS, 12.6](https://docs.oracle.com/javase/specs/jls/se16/html/jls-12.html#jls-12.6). Pode levar um tempo arbitrariamente longo entre o momento em que um objeto se torna inacessível e o momento em que seu finalizador ou *cleaners* é executado. Isso significa que você **nunca deve fazer nada que seja crítico em um finalizador ou *cleaners***. Por exemplo, é um erro grave depender de um finalizador ou *cleaners* para fechar arquivos porque os descritores de arquivos abertos são um recurso limitado. Se muitos arquivos forem deixados abertos como resultado do atraso do sistema em executar os finalizadores ou *cleaners*, um programa pode falhar porque não consegue mais abrir os arquivos.

A rapidez com que os finalizadores e *cleaners* são executados é principalmente uma função do algoritmo de coleta de lixo, que varia amplamente entre as implementações. O comportamento de um programa que depende da rapidez do finalizador ou da execução do *cleaners* pode variar da mesma forma. É inteiramente possível que tal programa funcione perfeitamente no JVM no qual você o testa e falhe miseravelmente naquele preferido pelo seu cliente mais importante.

A finalização tardia não é apenas um problema teórico. Fornecer um finalizador para uma classe pode atrasar arbitrariamente a recuperação de suas instâncias. Um colega depurou um aplicativo GUI de longa execução que estava morrendo misteriosamente com um `OutOfMemoryError`. A análise revelou que, no momento de sua morte, o aplicativo tinha milhares de objetos gráficos em sua fila de finalização apenas esperando para serem finalizados e recuperados. Infelizmente, o encadeamento do finalizador estava executando com uma prioridade mais baixa do que outro encadeamento do aplicativo, então os objetos não estavam sendo finalizados na taxa em que se tornaram elegíveis para finalização. A especificação da linguagem não oferece garantia de qual thread executará os finalizadores, portanto, não há maneira portável de evitar esse tipo de problema, exceto se abster de usar os finalizadores. Os *cleaners* são um pouco melhores do que os finalizadores nesse aspecto porque os autores da classe têm controle sobre seus próprios threads de limpeza, mas os *cleaners* ainda são executados em segundo plano, sob o controle do coletor de lixo, portanto, não pode haver garantia de limpeza imediata.

A especificação não fornece apenas nenhuma garantia de que os finalizadores ou *cleaners* serão executados prontamente; ele não oferece nenhuma garantia de que eles serão executados. É totalmente possível, até provável, que um programa seja encerrado sem executá-lo em alguns objetos que não estão mais acessíveis. Como consequência, você **nunca deve depender de um finalizador ou *cleaners* para atualizar o estado persistente**. Por exemplo, depender de um finalizador ou *cleaners* para liberar um bloqueio persistente em um recurso compartilhado, como um banco de dados, é uma boa maneira de interromper todo o sistema distribuído.

Não se deixe seduzir pelos métodos `System.gc` e `System.runFinalization`. Eles podem aumentar as chances de finalizadores ou *cleaners* serem executados, mas não garantem isso. Uma vez, dois métodos reivindicaram fazer essa garantia: `System.runFinalizersOnExit` e seu gêmeo maligno, `Runtime.runFinalizersOnExit`. Esses métodos são fatalmente falhos e foram descontinuados por décadas [ThreadStop]().

Outro problema com os finalizadores é que uma exceção não capturada lançada durante a finalização é ignorada e a finalização desse objeto é encerrada [JLS, 12.6](https://docs.oracle.com/javase/specs/jls/se16/html/jls-12.html#jls-12.6). Exceções não detectadas podem deixar outros objetos em um estado corrompido. Se outro thread tentar usar um objeto corrompido, poderá ocorrer um comportamento arbitrário não determinístico. Normalmente, uma exceção não capturada encerrará o encadeamento e imprimirá um rastreamento de pilha, mas não se ocorrer em um finalizador - ela nem mesmo imprimirá um aviso. Os *cleaners* não têm esse problema porque uma biblioteca que usa um *cleaners* tem controle sobre seu encadeamento.

**Há uma grande penalidade de desempenho para o uso de finalizadores e *cleaners***. Na minha máquina, o tempo para criar um objeto `AutoCloseable` simples, para fechá-lo usando `try-with-resources` e para que o coletor de lixo o recupere é cerca de 12 ns. Em vez disso, usar um finalizador aumenta o tempo para 550 ns. Em outras palavras, é cerca de 46 vezes mais lento para criar e destruir objetos com finalizadores. Isso ocorre principalmente porque os finalizadores inibem a coleta de lixo eficiente. Os *cleaners* são comparáveis em velocidade aos finalizadores se você os usar para *cleaners* todas as instâncias da classe (cerca de 500 ns por instância na minha máquina), mas os *cleaners* são muito mais rápidos se você usá-los apenas como uma rede de segurança, conforme discutido abaixo. Nessas circunstâncias, criar, limpar e destruir um objeto leva cerca de 66 ns na minha máquina, o que significa que você paga um fator de cinco (não cinquenta) pelo seguro de uma rede de segurança se não a usar.

**Os finalizadores têm um sério problema de segurança: eles abrem sua classe para ataques de finalizador**. A ideia por trás de um ataque de finalizador é simples: se uma exceção é lançada de um construtor ou seus equivalentes de serialização - os métodos `readObject` e `readResolve` [Capítulo 12]() - o finalizador de uma subclasse maliciosa pode ser executado no objeto parcialmente construído que deveria ter “Morreu na videira”. Este finalizador pode registrar uma referência ao objeto em um campo estático, evitando que seja coletado como lixo. Uma vez que o objeto malformado foi registrado, é uma questão simples invocar métodos arbitrários neste objeto que nunca deveria ter sido permitido em primeiro lugar. **Lançar uma exceção de um construtor deve ser suficiente para evitar que um objeto passe a existir; na presença de finalizadores, não é.** Esses ataques podem ter consequências terríveis. As classes finais são imunes a ataques de finalizador porque ninguém pode escrever uma subclasse maliciosa de uma classe final. **Para proteger as classes não finais de ataques de finalizador, escreva um método de finalização que não faça nada.**

Portanto, o que você deve fazer em vez de escrever um finalizador ou *cleaners* para uma classe cujos objetos encapsulam recursos que requerem encerramento, como arquivos ou threads? **Faça com que sua classe implemente AutoCloseable** e exija que seus clientes invoquem o método `close` em cada instância quando ele não for mais necessário, normalmente usando `try-with-resources` para garantir o encerramento mesmo diante de exceções [Item 9](). Um detalhe que vale a pena mencionar é que a instância deve acompanhar se foi fechada: o método `close`  deve registrar em um campo que o objeto não é mais válido, e outros métodos devem verificar este campo e lançar uma `IllegalStateException` se forem chamados depois o objeto foi fechado.

Então, para que servem os *cleaners* e finalizadores? Eles têm talvez dois usos legítimos. Uma é atuar como uma rede de segurança no caso de o proprietário de um recurso negligenciar a chamada de seu método de fechamento. Embora não haja garantia de que o *cleaners* ou finalizador será executado prontamente (ou de todo), é melhor liberar o recurso tarde do que nunca se o cliente não o fizer. Se você está pensando em escrever um finalizador de rede de segurança, pense muito sobre se a proteção vale o custo. Algumas classes da biblioteca Java, como `FileInputStream`, `FileOutputStream`, `ThreadPoolExecutor` e `java.sql.Connection`, têm finalizadores que servem como redes de segurança.

Um segundo uso legítimo de *cleaners* diz respeito a objetos com pares nativos. Um par nativo é um objeto nativo (não Java) ao qual um objeto normal delega por meio de métodos nativos. Como um par nativo não é um objeto normal, o coletor de lixo não sabe sobre ele e não pode recuperá-lo quando seu par Java é recuperado. Um *cleaners* ou finalizador pode ser um veículo apropriado para essa tarefa, presumindo que o desempenho seja aceitável e o par nativo não tenha recursos críticos. Se o desempenho for inaceitável ou o par nativo contiver recursos que devem ser recuperados imediatamente, a classe deve ter um método de fechamento, conforme descrito anteriormente. 

Os *cleaners* são um pouco complicados de usar. Abaixo está uma classe de `Room` simples demonstrando a facilidade. Suponhamos que os quartos devam ser limpos antes de serem recuperados. A classe `Room` implementa `AutoCloseable`; o fato de que sua rede de segurança de limpeza automática utiliza um limpador é apenas um detalhe de implementação. Ao contrário dos finalizadores, os *cleaners* não poluem a API pública de uma classe:

``` java
// Uma classe que pode ser fechada automaticamente usando um limpador como rede de segurança 
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create (); 
    
    // Recurso que requer limpeza. Não deve se referir a um Quarto! 
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room
        
        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // Invocado por método de fechamento ou limpador
        @Override public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }
    // O estado deste quarto, compartilhado com nosso lavável
    private final State state;

    // Nosso limpável. Limpa a sala quando é elegível para gc
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override public void close() {
        cleanable.clean();
    }
}

        
```

A classe aninhada estática *State* contém os recursos exigidos pelo *cleaners* para limpar a sala. Nesse caso, é simplesmente o campo `numJunkPiles`, que representa a quantidade de bagunça na sala. Mais realisticamente, pode ser um `final long` que contém um ponteiro para um par nativo. *State* implementa `Runnable`, e seu método `run` é chamado no máximo uma vez, pelo `Cleanable` que obtemos quando registramos nossa instância de *State* com nosso limpador no construtor `Room`. A chamada para o método executado será acionada por uma de duas coisas: Normalmente, é acionada por uma chamada para o método próximo a `Room` chamando o método `cleanable`. Se o cliente não conseguir chamar o método `close` no momento em que uma instância de `Room` estiver qualificada para a coleta de lixo, o limpador irá (esperançosamente) chamar o método run de State.

É fundamental que uma instância do Estado não se refira à sua instância. Se o fizesse, criar uma circularidade que impediria a instância `Room` de se tornar elegível para a coleta de lixo (e de ser limpa automaticamente). Portanto, State deve ser uma classe aninhada estática porque as classes aninhadas não estáticas apelidam a suas delimitadoras [Item 24](). Da mesma forma, é desaconselhável usar um lambda porque eles podem capturar facilmente a objetos que os envolvem.

Como dissemos antes, o *cleaner* de quarto é usado apenas como uma rede de segurança. Se os clientes cercarem todas as informações de `Room` in blocos de tentativa com recurso, a limpeza automática nunca será necessária. Este cliente bem comportado demonstra esse comportamento:

```java
public class Adulto {
    public static void main (String [] args) {
        try (Room myRoom = new Room (7)) {
            System.out.println ("Goodbye"); 
            }
        }
}
```

Como seria de esperar, filho para o programa adulto imprime "Goodbye", seguido por "Cleaning room". Mas e esse programa malcomportado, que nunca limpa seu ambiente? 

```java
public class Teenager {
    public static void main (String [] args) {
        new Room (99); 
        System.out.println ("Paz"); 
    }
}
```

Você pode esperar que ele imprima Peace out, seguido por Cleaning room, mas na minha máquina, ele nunca imprime Cleaning room; simplesmente sai. Essa é uma imprevisibilidade da qual falamos antes. A especificação do Cleaner diz: “O comportamento dos limpadores durante System.exit é específico da implementação. Nenhuma garantia é feita sobre se as ações de limpeza são invocadas ou não. ”Embora a especificação não diga isso, o mesmo vale para a saída normal do programa. Na minha máquina, adicionar a linha `System.gc()` ao método principal do Teenager é suficiente para fazer com que ele imprima Cleaning room antes de sair, mas não há garantia de que você verá o mesmo comportamento em sua máquina.

Em resumo, não use limpadores, ou em versões anteriores ao Java 9, finalizadores, exceto como uma rede de segurança ou para encerrar recursos nativos não relacionados. Mesmo assim, tome cuidado com a indeterminação e como consequências de desempenho.