# Item 5: Prefira injeção de dependência para recursos de hardwiring

Muitas classes dependem de um ou mais recursos subjacentes. Por exemplo, um corretor ortográfico depende de um dicionário. Não é incomum ver essas classes implementadas como classes de utilitários estáticos [Item 4]():

```java
// Uso impróprio de utilitário estático - inflexível e não testável!
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    private SpellChecker() {} // Não instanciável
    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```

Da mesma forma, não é incomum vê-los implementados como *singletons* [Item 3]():

Nenhuma dessas abordagens é satisfatória, porque presumem que existe apenas um dicionário que vale a pena usar. Na prática, cada idioma tem seu próprio dicionário e dicionários especiais são usados para vocabulários especiais. Além disso, pode ser desejável usar um dicionário especial para teste. É uma ilusão supor que um único dicionário será suficiente para sempre.

Você poderia tentar fazer com que o `SpellChecker` suporte vários dicionários tornando o campo de dicionário não final e adicionando um método para alterar o dicionário em um corretor ortográfico existente, mas isso seria estranho, sujeito a erros e impraticável em uma configuração simultânea. Classes de utilitários estáticos e *singletons* são inadequados para classes cujo comportamento é parametrizado por um recurso subjacente.

O que é necessário é a capacidade de suportar várias instâncias da classe (em nosso exemplo, `SpellChecker`), cada uma delas usando o recurso desejado pelo cliente (em nosso exemplo, o dicionário). Um padrão simples que satisfaz esse requisito é **passar o recurso para o construtor ao criar uma nova instância**. Esta é uma forma de injeção de dependência: o dicionário é uma dependência do corretor ortográfico e é injetado no corretor ortográfico quando é criado.

``` java
// A injeção de dependência fornece flexibilidade e capacidade de teste
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }

    public List<String> suggestions(String typo) { ... }
}

```

O padrão de injeção de dependência é tão simples que muitos programadores o usam por anos sem saber que tem um nome. Embora nosso exemplo de verificador ortográfico tenha apenas um único recurso (o dicionário), a injeção de dependência funciona com um número arbitrário de recursos e gráficos de dependência arbitrários. Ele preserva a imutabilidade [Item 17](), para que vários clientes possam compartilhar objetos dependentes (assumindo que os clientes desejam os mesmos recursos subjacentes). A injeção de dependência é igualmente aplicável a construtores, *static factories* [Item 1]() e *builders*  [Item 2]().

Uma variante útil do padrão é passar uma fábrica de recursos para o construtor. Uma fábrica é um objeto que pode ser chamado repetidamente para criar instâncias de um tipo. Essas fábricas incorporam o padrão *Factory Method* [Gamma95](). A interface `Supplier<T>`, introduzida no Java 8, é perfeita para representar fábricas. Métodos que levam um `Supplier<T>` na entrada devem normalmente restringir o parâmetro de tipo da fábrica usando um tipo curinga limitado [Item 31]() para permitir que o cliente passe em uma fábrica que cria qualquer subtipo de um tipo especificado. Por exemplo, aqui está um método que faz um mosaico usando uma fábrica fornecida pelo cliente para produzir cada bloco:

``` java
Mosaic create(Supplier<? extends Tile> tileFactory) {
    ... 
}
``` 

Embora a injeção de dependência melhore muito a flexibilidade e a capacidade de teste, ela pode atrapalhar grandes projetos, que normalmente contêm milhares de dependências. Essa confusão pode ser praticamente eliminada usando uma estrutura de injeção de dependência, como [Dagger](), [Guice]() ou [Spring](). O uso dessas estruturas está além do escopo deste livro, mas observe que as APIs projetadas para injeção de dependência manual são adaptadas trivialmente para uso por essas estruturas.

Em resumo, não use um *singleton* ou classe de utilitário estático para implementar uma classe que dependa de um ou mais recursos subjacentes cujo comportamento afete o da classe e não faça com que a classe crie esses recursos diretamente. Em vez disso, passe os recursos, ou fábricas para criá-los, para o construtor (ou fábrica ou construtor estático). Essa prática, conhecida como injeção de dependência, aumentará muito a flexibilidade, a capacidade de reutilização e a testabilidade de uma classe.