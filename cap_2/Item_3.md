# Item 3: Aplicar a propriedade singleton com um construtor privado ou um tipo de enum

Um singleton é simplesmente uma classe instanciada exatamente uma vez [Gamma95](). Os singletons geralmente representam um objeto sem estado, como uma função [Item 24](), ou um componente do sistema que é intrinsecamente exclusivo. **Tornar uma classe um singleton pode dificultar o teste de seus clientes** porque é impossível substituir uma implementação simulada por um singleton, a menos que implemente uma interface que sirva como seu tipo.

Existem duas maneiras comuns de implementar singletons. Ambos são baseados em manter o construtor privado e exportar um membro público estático para fornecer acesso à única instância. Em uma abordagem, o membro é um campo final:

``` java
// Singleton com campo público final
public class Elvis {
    public static final Elvis INSTANCE = new Elvis ();

    private Elvis() {...}

    public void leaveTheBuilding () {...}
}
```

O construtor privado é chamado apenas uma vez, para inicializar o campo final público estático `Elvis.INSTANCE`. A falta de um construtor público ou protegido garante um universo “monoelvístico”: existirá exatamente uma instância de `Elvis` assim que a classe Elvis for inicializada - nem mais, nem menos. Nada que um cliente faça pode mudar isso, com uma ressalva: um cliente privilegiado pode invocar o construtor privado reflexivamente [Item 65]() com a ajuda do método `AccessibleObject.setAccessible`. Se você precisar se defender contra esse ataque, modifique o construtor para que ele lance uma exceção se for solicitado a criar uma segunda instância.

Na segunda abordagem para implementar singletons, o membro público é um método de fábrica estático:

``` java
// Singleton com fábrica estática
public class Elvis {
    private static final Elvis INSTANCE = new Elvis ();

    private Elvis() {...}

    public static Elvis getInstance () {
        return INSTANCE;
    }

    public void leaveTheBuilding () {...}
}

```

Todas as chamadas para `Elvis.getInstance` retornam a mesma referência de objeto e nenhuma outra instância de Elvis será criada (com a mesma ressalva mencionada anteriormente).

A principal vantagem da abordagem de campo público é que a API deixa claro que a classe é um singleton: o campo estático público é final, portanto, sempre conterá a mesma referência de objeto. A segunda vantagem é que é mais simples.

Uma vantagem da abordagem de fábrica estática é que ela dá a você a flexibilidade de mudar de ideia sobre se a classe é um singleton sem alterar sua API. O método de fábrica retorna a única instância, mas pode ser modificado para retornar, digamos, uma instância separada para cada thread que o invoca. Uma segunda vantagem é que você pode escrever uma fábrica de singleton genérica se sua aplicação exigir [Item 30](). Uma vantagem final de usar uma fábrica estática é que uma referência de método pode ser usada como um fornecedor, por exemplo `Elvis::instance` é um `Supplier<Elvis>`. A menos que uma dessas vantagens seja relevante, a abordagem de campo público é preferível.

Para tornar serializável uma classe singleton que usa qualquer uma dessas abordagens [Capítulo 12](), não é suficiente simplesmente adicionar implementos Serializable à sua declaração. Para manter a garantia do singleton, declare todos os campos de instância como transitórios e forneça um método readResolve [Item 89](). Caso contrário, cada vez que uma instância serializada é desserializada, uma nova instância será criada, levando, no caso do nosso exemplo, a visões espúrias de Elvis. Para evitar que isso aconteça, adicione este método readResolve à classe Elvis:
``` java
// método readResolve para preservar a propriedade singleton
private Object readResolve () {
    // Devolva o verdadeiro Elvis e deixe o coletor de lixo
    // cuide do imitador de Elvis.
    return INSTANCE;
}
```

Uma terceira maneira de implementar um singleton é declarar um enum de elemento único: 
``` java
// Enum singleton - a abordagem preferida 
public enum Elvis { 
    INSTANCE; 

    public void leaveTheBuilding () {...}
}
``` 

Essa abordagem é semelhante à abordagem de campo público, mas é mais concisa, fornece o maquinário de serialização gratuitamente e fornece uma garantia férrea contra instanciação múltipla, mesmo em face de serialização sofisticada ou ataques de reflexão. Essa abordagem pode parecer um pouco artificial, **mas um tipo de enum de elemento único geralmente é a melhor maneira de implementar um singleton.** Observe que você não pode usar essa abordagem se seu singleton deve estender uma superclasse diferente de Enum (embora você possa declarar um enum para implementar interfaces).