# Item 1: Consider *static factory methods* ao invés de construtores

A maneira tradicional de uma classe permitir que um cliente obtenha uma instância é fornecer um construtor público. Existe outra técnica que deve fazer parte do kit de ferramentas de todo programador. Uma classe pode fornecer um *public static method*, que é simplesmente um método estático que retorna uma instância da classe. Um exemplo simples está na classe `Boolean`. O método [Boolean.valueOf(boolean)](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/lang/Boolean.html#valueOf(boolean)) converte um valor primitivo do tipo `boolean` em uma referência ao objeto `Boolean`.

``` Java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

Observe que um *static factory methods* **não é igual ao** *Design Pattern Factory Method* [Gamma95](). O *static factory methods* descrito neste item não tem equivalente direta ao Design Pattern.

Uma classe pode fornecer a seus clientes *static factory methods* em vez de, ou além de, construtores públicos. Fornecer um *static factory methods* em vez de um construtor público tem vantagens e desvantagens

1) **Uma vantagem dos *static factory methods* é que, ao contrário dos construtores, eles têm nomes.** 

Se os parâmetros para um construtor não descrevem, por si só, o objeto que está sendo retornado, uma fábrica estática com um nome bem escolhido é mais fácil de usar e o código do cliente resultante mais fácil de ler. Por exemplo, o construtor [BigInteger (int, int, Random)](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/math/BigInteger.html#%3Cinit%3E(int,java.util.Random)), que retorna um `BigInteger` que provavelmente é primo, teria sido melhor expresso como um *static factory methods* denominado [BigInteger.probablePrime(int, Random)](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/math/BigInteger.html#probablePrime(int,java.util.Random)).

Uma classe pode ter apenas um único construtor com uma determinada assinatura.

Uma classe pode ter apenas um único construtor com uma assinatura fornecida.

Sabe-se que os programadores contornam essa restrição fornecendo dois construtores cujas listas de parâmetros diferem apenas na ordem de seus tipos de parâmetro. Esta é uma ideia muito ruim. O usuário de tal API nunca será capaz de lembrar qual construtor é qual e acabará chamando o errado por engano. Pessoas lendo código que usa esses construtores não saberão o que o código faz sem consultar a documentação da classe.

Por terem nomes, os *static factory methods* não compartilham a restrição discutida no parágrafo anterior. Nos casos em que uma classe parece exigir vários construtores com a mesma assinatura, substitua os construtores por *static factory methods* e nomes cuidadosamente escolhidos para destacar suas diferenças.

2) **Uma segunda vantagem dos *static factory methods* é que, ao contrário dos construtores, eles não são obrigados a criar um novo objeto cada vez que são invocados.** 

Isso permite que classes imutáveis ​​[Item 17](cap_4.md) usem instâncias pré-construídas ou armazenem em cache as instâncias conforme são construídas e as dispensem repetidamente para evitar a criação de objetos duplicados desnecessários. O método [Boolean.valueOf(boolean)](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/lang/Boolean.html#valueOf(boolean)) ilustra essa técnica: ele nunca cria um objeto. Esta técnica é semelhante ao padrão *Flyweight* [Gamma95](Ref.md). Pode melhorar muito o desempenho se objetos equivalentes forem solicitados com frequência, especialmente se sua criação for cara.

A capacidade dos *static factory methods* de retornar o mesmo objeto de invocações repetidas permite que as classes mantenham um controle estrito sobre quais instâncias existem a qualquer momento. As classes que fazem isso são consideradas controladoras de instância. Existem vários motivos para escrever classes controladoras de instância. O controle de instância permite que uma classe garanta que é um *singleton* [Item 3](cap_2.md) ou não instável [Item 4](cap_2.md). Além disso, permite que uma classe de valor imutável [Item 17](cap_4.md) garanta que não existam duas instâncias iguais: `a.equals(b)` se e somente se `a == b`. Esta é a base do padrão *Flyweight* [Gamma95](Ref.md). Os tipos *enums*[item 34](cap_6.md) fornecem essa garantia.

3) **Uma terceira vantagem dos *static factory methods* é que, ao contrário dos construtores, eles podem retornar um objeto de qualquer subtipo de seu tipo de retorno.** 

Isso oferece grande flexibilidade na escolha da classe do objeto retornado.

Uma aplicação dessa flexibilidade é que uma API pode retornar objetos sem tornar suas classes públicas. Ocultar classes de implementação dessa maneira leva a uma API muito compacta. Esta técnica se presta a estruturas baseadas em interface [Item 20](), onde as interfaces fornecem tipos de retorno naturais para *static factory methods*

Antes do Java 8, as interfaces não podiam ter métodos estáticos. Por convenção, os *static factory methods* para uma interface denominada `Type` foram colocados em uma classe complementar não instável [Item 4]() denominada `Types`. Por exemplo, o Java *Collections Framework* tem quarenta e cinco implementações de utilitários de suas interfaces, fornecendo coleções não modificáveis, coleções sincronizadas e assim por diante. Quase todas essas implementações são exportadas por meio de *static factory methods* em uma classe não instável (java.util.Collections). As classes dos objetos retornados são todas não públicas.

A API do *Collections Framework* é muito menor do que seria se tivesse exportado quarenta e cinco classes públicas separadas, uma para cada implementação de conveniência. Não é apenas o volume da API que é reduzido, mas o peso conceitual: o número e a dificuldade dos conceitos que os programadores devem dominar para usar a API. O programador sabe que o objeto retornado possui precisamente a API especificada por sua interface, portanto, não há necessidade de ler a documentação da classe adicional para a classe de implementação. Além disso, o uso de um *static factory methods* exige que o cliente consulte o objeto retornado pela interface em vez da classe de implementação, o que geralmente é uma boa prática [Item 64]().

A partir do Java 8, a restrição de que as interfaces não podem conter métodos estáticos foi eliminada, portanto, normalmente há poucos motivos para fornecer uma classe complementar não instável ao invés uma interface. Muitos membros públicos e estáticos que estariam em casa em tal classe, em vez disso, devem ser colocados na própria interface. Observe, entretanto, que ainda pode ser necessário colocar a maior parte do código de implementação por trás desses métodos estáticos em uma classe privada de pacote separada. Isso ocorre porque o Java 8 requer que todos os membros estáticos de uma interface sejam públicos. Java 9 permite métodos estáticos privados, mas os campos estáticos e as classes de membros estáticos ainda precisam ser públicos

4) **Uma quarta vantagem das *factory methods* é que a classe do objeto retornado pode variar de chamada para chamada em função dos parâmetros de entrada.**

 Qualquer subtipo do tipo de retorno declarado é permitido. A classe do objeto retornado também pode variar de uma versão para outra.

A classe `EnumSet` [Item 36]() não tem construtores públicos, apenas *factory methods*. Na implementação do OpenJDK, eles retornam uma instância de uma das duas subclasses, dependendo do tamanho do tipo de enum subjacente: se ele tiver sessenta e quatro ou menos elementos, como a maioria dos tipos de enum tem, as *factory methods* retornam uma instância `RegularEnumSet`, que é apoiado por um único `long`; se o tipo `enum` tiver sessenta e cinco ou mais elementos, as fábricas retornam uma instância `JumboEnumSet`, apoiada por uma matriz de `long`.

A existência dessas duas classes de implementação é invisível para os clientes. Se `RegularEnumSet` deixasse de oferecer vantagens de desempenho para pequenos tipos de `enum`, ele poderia ser eliminado de uma versão futura sem efeitos nocivos. Da mesma forma, uma versão futura pode adicionar uma terceira ou quarta implementação do `EnumSet` se for benéfico para o desempenho. Os clientes não sabem nem se importam com a classe do objeto que recebem da fábrica; eles se preocupam apenas com o fato de que é alguma subclasse de `EnumSet`.

5) **Uma quinta vantagem das *factory methods* é que a classe do objeto retornado não precisa existir quando a classe que contém o método é escrita.**

Esses *static factory methods* flexíveis formam a base das estruturas de provedor de serviços, como a API de conectividade de banco de dados Java (JDBC). Uma estrutura de provedor de serviço é um sistema no qual os provedores implementam um serviço, e o sistema disponibiliza as implementações aos clientes, desacoplando os clientes das implementações.

Existem três componentes essenciais em uma estrutura de provedor de serviço: uma interface de serviço, que representa uma implementação; uma API de registro de provedor, que os provedores usam para registrar implementações; e uma API de acesso ao serviço, que os clientes usam para obter instâncias do serviço. A API de acesso ao serviço pode permitir que os clientes especifiquem critérios para escolher uma implementação. Na ausência de tais critérios, a API retorna uma instância de uma implementação padrão ou permite que o cliente percorra todas as implementações disponíveis.

A API de acesso ao serviço é a fábrica estática flexível que forma a base da estrutura do provedor de serviço. Um quarto componente opcional de uma estrutura de provedor de serviço é uma interface de provedor de serviço, que descreve um objeto fábrica que produz instâncias da interface de serviço. Na ausência de uma interface de provedor de serviço, as implementações devem ser instanciadas reflexivamente [Item 65](). No caso do JDBC, Connection desempenha o papel da interface de serviço, `DriverManager.registerDriver` é a API de registro do provedor, `DriverManager`.`getConnection` é a API de acesso ao serviço e Driver é a interface do provedor de serviço.

Existem muitas variantes do padrão de estrutura do provedor de serviços. Por exemplo, a API de acesso ao serviço pode retornar uma interface de serviço mais rica para os clientes do que aquela fornecida pelos provedores. Este é o padrão *Bridge* [Gamma95]. As estruturas de injeção de dependência [Item 5]() podem ser vistas como provedores de serviços poderosos. Desde Java 6, a plataforma inclui uma estrutura de provedor de serviços de uso geral, `java.util.ServiceLoader`, então você não precisa, e geralmente não deve, escrever seu próprio [Item 59](). JDBC não usa ServiceLoader, pois o primeiro é anterior ao último.

A principal limitação de fornecer apenas *static factory methods* é que as classes sem construtores públicos ou protegidos não podem ser subclasses. Por exemplo, é impossível criar uma subclasse de qualquer uma das classes de implementação de conveniência no *Collections Framework*. Indiscutivelmente, isso pode ser uma bênção disfarçada porque incentiva os programadores a usar composição em vez de herança [Item 18]() e é necessário para tipos imutáveis ​​[Item 17]().

Uma segunda deficiência dos *static factory methods* é que eles são difíceis de serem encontrados pelos programadores. Eles não se destacam na documentação da API da maneira que os construtores, portanto, pode ser difícil descobrir como instanciar uma classe que forneça *static factory methods* em vez de construtores. A ferramenta Javadoc algum dia pode chamar a atenção para *static factory methods*. Enquanto isso, você pode reduzir esse problema chamando a atenção para *factory methods* na documentação de classe ou interface e aderindo às convenções de nomenclatura comuns. Aqui estão alguns nomes comuns para *static factory methods*. Esta lista está longe de ser exaustiva:

* **from** - Um método de conversão de tipo que usa um único parâmetro e retorna uma instância correspondente desse tipo, por exemplo: 

``` Java
Data d = Data.from(instant);
```

* **of — Um método de agregação que recebe vários parâmetros e retorna uma instância desse tipo que os incorpora, por exemplo: 

```java
Set <Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
```

* **valueOf** - Uma alternativa mais detalhada de e de, por exemplo: 

```java
BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
```

* **instance** ou **getInstance** - Retorna uma instância que é descrita por seus parâmetros (se houver), mas não pode ser considerada como tendo o mesmo valor, por exemplo: 

``` Java
StackWalker luke = StackWalker.getInstance(options);
```

* **create** ou **newInstance** - Como instance ou getInstance, exceto que o método garante que cada chamada retorne uma nova instância, por exemplo: 

``` Java
Object newArray = Array.newInstance(classObject, arrayLen);
```

* **getType** - Como getInstance, mas usado se o método de fábrica estiver em uma classe diferente. Tipo é o tipo de objeto retornado pelo método de fábrica, por exemplo: 

```Java
FileStore fs = Files.getFileStore(path);
```

* **newType** - Como newInstance, mas usado se o método de fábrica estiver em uma classe diferente. Tipo é o tipo de objeto retornado pelo método de fábrica, por exemplo:

``` Java
BufferedReader br = Files.newBufferedReader(path);
```

* **type** - Uma alternativa concisa para getType e newType, por exemplo: 

``` Java
List <Complaint> litany = Collections.list(legacyLitany);
```

Em resumo, os *static factory methods* e os construtores públicos têm seus usos e vale a pena entender seus méritos relativos. Freqüentemente, *factory methods* são preferíveis, portanto, evite o reflexo de fornecer construtores públicos sem primeiro considerar as *factory methods*.