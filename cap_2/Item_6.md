# Item 6: Evite criar objetos desnecessários

Frequentemente, é apropriado reutilizar um único objeto em vez de criar um novo objeto funcionalmente equivalente sempre que necessário. A reutilização pode ser mais rápida e elegante. Um objeto sempre pode ser reutilizado se for imutável [item 17](). Como um exemplo extremo do que não fazer, considere esta declaração:

```java
String s = new String("bikini"); // NÃO FAÇA ISSO!
```

A instrução cria uma nova instância de String cada vez que é executada e nenhuma dessas criações de objeto é necessária. O argumento para o construtor String ("biquíni") é em si uma instância de String, funcionalmente idêntica a todos os objetos criados pelo construtor. Se esse uso ocorrer em um loop ou em um método frequentemente invocado, milhões de instâncias de String podem ser criadas desnecessariamente.

A versão melhorada é simplesmente a seguinte:

```java
String s = "biquíni";
```

Esta versão usa uma única instância de String, em vez de criar uma nova cada vez que é executada. Além disso, é garantido que o objeto será reutilizado por qualquer outro código executado na mesma máquina virtual que contenha o mesmo literal de string [JLS, 3.10.5](https://docs.oracle.com/javase/specs/jls/se16/html/jls-3.html#jls-3.10.5).

Muitas vezes, você pode evitar a criação de objetos desnecessários usando métodos de fábrica estáticos (Item 1) em vez de construtores em classes imutáveis que fornecem ambos. Por exemplo, o método de fábrica Boolean.valueOf (String) é preferível ao construtor Boolean (String), que foi descontinuado em Java 9. O construtor deve criar um novo objeto cada vez que é chamado, enquanto o método de fábrica nunca é obrigado a fazer assim e não na prática. Além de reutilizar objetos imutáveis, você também pode reutilizar objetos mutáveis se souber que eles não serão modificados.

Algumas criações de objetos são muito mais caras do que outras. Se você vai precisar de um "objeto caro" repetidamente, pode ser aconselhável armazená-lo em cache para reutilização. Infelizmente, nem sempre é óbvio quando você está criando tal objeto. Suponha que você queira escrever um método para determinar se uma string é um numeral romano válido. Esta é a maneira mais fácil de fazer isso usando uma expressão regular:

```java
// O desempenho pode ser melhorado!
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}

```

O problema com essa implementação é que ela depende do método `String.matches`. Embora `String.matches` seja a maneira mais fácil de verificar se uma string corresponde a uma expressão regular, não é adequado para uso repetido em situações críticas de desempenho. O problema é que ele cria internamente uma instância de `Pattern` para a expressão regular e a usa apenas uma vez, após o que se torna elegível para a coleta de lixo. Criar uma instância de `Pattern` é caro porque requer a compilação da expressão regular em uma máquina de estado finito.

Para melhorar o desempenho, compile explicitamente a expressão regular em uma instância de `Pattern` (que é imutável) como parte da inicialização da classe, armazene-a em cache e reutilize a mesma instância para cada invocação do método `isRomanNumeral`:

```java
// Reutilizar objetos caros para melhorar o desempenho
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile( "^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}

```

A versão aprimorada de `isRomanNumeral` fornece ganhos de desempenho significativos se invocada com frequência. Na minha máquina, a versão original leva 1,1 µs em uma string de entrada de 8 caracteres, enquanto a versão aprimorada leva 0,17 µs, o que é 6,5 vezes mais rápido. Não apenas o desempenho melhorou, mas, indiscutivelmente, também a clareza. Criar um campo final estático para a instância de Pattern, de outra forma invisível, nos permite dar a ela um nome, que é muito mais legível do que a própria expressão regular.

Se a classe que contém a versão aprimorada do método `isRomanNumeral` for inicializada, mas o método nunca for invocado, o campo `ROMAN` será inicializado desnecessariamente. Seria possível eliminar a inicialização inicializando lentamente o campo [Item 83]() na primeira vez que o método `isRomanNumeral` for chamado, mas isso não é recomendado. Como costuma acontecer com a inicialização lenta, isso complicaria a implementação sem nenhuma melhoria mensurável de desempenho [Item 67]().

Quando um objeto é imutável, é óbvio que ele pode ser reutilizado com segurança, mas há outras situações em que é muito menos óbvio, até mesmo contra-intuitivo. Considere o caso dos adaptadores [Gamma95](), também conhecidos como visualizações. Um adaptador é um objeto que delega a um objeto de apoio, fornecendo uma interface alternativa. Como um adaptador não tem estado além daquele de seu objeto de apoio, não há necessidade de criar mais de uma instância de um determinado adaptador para um determinado objeto.

Por exemplo, o método `keySet` da interface `Map` retorna uma visualização `Set` do objeto `Map`, consistindo em todas as chaves no mapa. Ingenuamente, pareceria que cada chamada para `keySet` teria que criar uma nova instância de `Set`, mas cada chamada para `keySet` em um determinado objeto `Map` pode retornar a mesma instância de `Set`. Embora a instância `Set` retornada seja tipicamente mutável, todos os objetos retornados são funcionalmente idênticos: quando um dos objetos retornados muda, o mesmo ocorre com todos os outros, porque todos são apoiados pela mesma instância `Map`. Embora seja amplamente inofensivo criar várias instâncias do objeto de exibição `keySet`, é desnecessário e não traz benefícios.

Outra maneira de criar objetos desnecessários é o *autoboxing*, que permite ao programador misturar tipos primitivos primitivos e em caixa, encaixotando e desencaixotando automaticamente conforme necessário. O autoboxing embaça, mas não apaga a distinção entre tipos primitivos primitivos e em caixa. Existem distinções semânticas sutis e diferenças de desempenho não tão sutis [Item 61](). Considere o método a seguir, que calcula a soma de todos os valores `int` positivos. Para fazer isso, o programa precisa usar aritmética longa porque um `int` não é grande o suficiente para conter a soma de todos os valores `int` positivos:

``` java
// terrivelmente lento! Você consegue identificar a criação do objeto?
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;
    return sum;
}

```

Este programa obtém a resposta certa, mas é muito mais lento do que deveria, devido a um erro tipográfico de um caractere. A variável `sum` é declarada como `Long` em vez de `long`, o que significa que o programa constrói cerca de 2³¹ instâncias `Long` desnecessárias (aproximadamente uma para cada vez que i `long` é adicionado à `som` `Long`). Alterar a declaração de `sum` de `Long` para `long` reduz o tempo de execução de 6,3 s para 0,59 s na minha máquina. A lição é clara: **prefira primitivas a primitivas encaixotadas e fique atento ao autoboxing não intencional**.

Este item não deve ser interpretado incorretamente para sugerir que a criação de objetos é cara e deve ser evitada. Ao contrário, a criação e a recuperação de pequenos objetos cujos construtores fazem pouco trabalho explícito é barata, especialmente em implementações JVM modernas. A criação de objetos adicionais para aumentar a clareza, simplicidade ou poder de um programa é geralmente uma coisa boa.

Por outro lado, evitar a criação de objetos mantendo seu próprio pool de objetos é uma má ideia, a menos que os objetos no pool sejam extremamente pesados. O exemplo clássico de um objeto que justifica um pool de objetos é uma conexão de banco de dados. O custo de estabelecer a conexão é suficientemente alto para que faça sentido reutilizar esses objetos. De modo geral, no entanto, manter seus próprios pools de objetos desorganiza seu código, aumenta o consumo de memória e prejudica o desempenho. Implementações modernas de JVM têm coletores de lixo altamente otimizados que superam facilmente tais pools de objetos em objetos leves.

O contraponto a este item é o [item 50]() sobre cópia defensiva. O item presente diz: "Não crie um novo objeto quando você deve reutilizar um existente", enquanto o [Item 50]() diz: "Não reutilize um objeto existente quando você deve criar um novo." Observe que a penalidade por reutilizar um objeto quando uma cópia defensiva é necessária é muito maior do que a penalidade por criar desnecessariamente um objeto duplicado. Deixar de fazer cópias defensivas quando necessário pode levar a bugs insidiosos e falhas de segurança; criar objetos desnecessariamente apenas afeta o estilo e o desempenho.