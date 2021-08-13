# Item 7: Elimine referências de objetos obsoletos

Se você mudou de uma linguagem com gerenciamento manual de memória, como C ou C ++, para uma linguagem com coleta de lixo, como Java, seu trabalho como programador ficou muito mais fácil pelo fato de que seus objetos são automaticamente recuperados quando você termina com eles. Parece quase mágica quando você a experimenta pela primeira vez. Isso pode facilmente levar à impressão de que você não precisa pensar sobre o gerenciamento de memória, mas isso não é bem verdade. Considere a seguinte implementação de pilha simples:

``` java
// Você consegue identificar o "vazamento de memória"?
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;   
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
    * Garanta espaço para pelo menos mais um elemento, 
    * praticamente dobrando a capacidade cada vez que o * array precisa crescer.
    **/
    private void ensureCapacity() {
        if (elements.length == size)
        elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}

```

Não há nada obviamente errado com este programa (mas veja o Item 29 para uma versão genérica). Você poderia testá-lo exaustivamente e ele passaria em todos os testes com louvor, mas há um problema à espreita. Em termos gerais, o programa tem um “vazamento de memória”, que pode se manifestar silenciosamente como desempenho reduzido devido ao aumento da atividade do coletor de lixo ou aumento da pegada de memória. Em casos extremos, esses vazamentos de memória podem causar paginação de disco e até mesmo falha do programa com um `OutOfMemoryError`, mas essas falhas são relativamente raras.

Então, onde está o vazamento de memória? Se uma pilha aumentar e depois diminuir, os objetos que foram retirados da pilha não serão coletados como lixo, mesmo se o programa que está usando a pilha não tiver mais referências a eles. Isso ocorre porque a pilha mantém referências obsoletas a esses objetos. Uma referência obsoleta é simplesmente uma referência que nunca será desreferenciada novamente. Nesse caso, quaisquer referências fora da “parte ativa” da matriz do elemento são obsoletas. A parte ativa consiste nos elementos cujo índice é menor que o tamanho.

Vazamentos de memória em linguagens com coleta de lixo (mais apropriadamente conhecidas como retenções não intencionais de objetos) são insidiosos. Se uma referência de objeto for retida não intencionalmente, não apenas esse objeto será excluído da coleta de lixo, mas também quaisquer objetos referenciados por esse objeto e assim por diante. Mesmo que apenas algumas referências de objeto sejam retidas involuntariamente, muitos, muitos objetos podem ser impedidos de serem coletados como lixo, com efeitos potencialmente grandes no desempenho.

A solução para esse tipo de problema é simples: anule as referências assim que se tornarem obsoletas. No caso de nossa classe Stack, a referência a um item se torna obsoleta assim que é retirado da pilha. A versão corrigida do método pop é parecida com esta:

``` java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Elimine referência obsoleta
    return result;
}

```

Um benefício adicional de anular referências obsoletas é que se elas forem posteriormente desreferenciadas por engano, o programa irá falhar imediatamente com uma `NullPointerException`, ao invés de silenciosamente fazer a coisa errada. É sempre benéfico detectar erros de programação o mais rápido possível.

Quando os programadores são atingidos pela primeira vez por esse problema, eles podem compensar anulando todas as referências de objeto assim que o programa terminar de usá-lo. Isso não é necessário nem desejável; ele desorganiza o programa desnecessariamente. **Anular as referências de objetos deve ser a exceção, e não a norma**. A melhor maneira de eliminar uma referência obsoleta é deixar a variável que continha a referência sair do escopo. Isso ocorre naturalmente se você definir cada variável no escopo mais estreito possível [item 57]().

Então, quando você deve anular uma referência? Qual aspecto da classe Stack a torna suscetível a vazamentos de memória? Simplificando, ele gerencia sua própria memória. O pool de armazenamento consiste nos elementos da matriz de elementos (as células de referência do objeto, não os próprios objetos). Os elementos na parte ativa da matriz (conforme definido anteriormente) são alocados e aqueles no restante da matriz são livres. O coletor de lixo não tem como saber disso; para o coletor de lixo, todas as referências de objeto na matriz de elementos são igualmente válidas. Apenas o programador sabe que a parte inativa da matriz não é importante. O programador comunica efetivamente esse fato ao coletor de lixo anulando manualmente os elementos do array assim que eles se tornam parte da parte inativa.

De modo geral, **sempre que uma classe gerencia sua própria memória, o programador deve estar alerta para vazamentos de memória**. Sempre que um elemento é liberado, todas as referências de objeto contidas no elemento devem ser anuladas.

**Outra fonte comum de vazamentos de memória são os caches**. Depois de colocar uma referência de objeto em um cache, é fácil esquecer que ela está lá e deixá-la no cache muito depois de se tornar irrelevante. Existem várias soluções para este problema. Se você tiver sorte o suficiente para implementar um cache para o qual uma entrada é relevante exatamente desde que haja referências à sua chave fora do cache, represente o cache como um `WeakHashMap`; as entradas serão removidas automaticamente após se tornarem obsoletas. Lembre-se de que `WeakHashMap` é útil apenas se o tempo de vida desejado das entradas de cache for determinado por referências externas à chave, não ao valor.

Mais comumente, a vida útil de uma entrada de cache é menos bem definida, com entradas se tornando menos valiosas com o tempo. Nessas circunstâncias, o cache deve ocasionalmente ser limpo de entradas que caíram em desuso. Isso pode ser feito por um thread em segundo plano (talvez um `ScheduledThreadPoolExecutor`) ou como um efeito colateral da adição de novas entradas ao cache. A classe LinkedHashMap facilita a última abordagem com seu método `removeEldestEntry`. Para caches mais sofisticados, você pode precisar usar `java.lang.ref` diretamente.

**Uma terceira fonte comum de vazamentos de memória são os ouvintes e outros retornos de chamada**. Se você implementar uma API em que os clientes registram retornos de chamada, mas não os cancela explicitamente, eles se acumularão, a menos que você execute alguma ação. Uma maneira de garantir que os retornos de chamada sejam coletados como lixo imediatamente é armazenar apenas referências fracas a eles, por exemplo, armazenando-os apenas como chaves em um `WeakHashMap`.

Como os vazamentos de memória normalmente não se manifestam como falhas óbvias, eles podem permanecer presentes em um sistema por anos. Normalmente, eles são descobertos apenas como resultado de uma inspeção cuidadosa do código ou com o auxílio de uma ferramenta de depuração conhecida como criador de perfil de heap. Portanto, é muito desejável aprender a antecipar problemas como esse antes que ocorram e evitar que aconteçam.