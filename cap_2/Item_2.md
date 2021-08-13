# Item 2: Considere um construtor quando confrontado com muitos parâmetros de construtor

As *static factory methods* e construtores compartilham uma limitação: eles não se adaptam bem a um grande número de parâmetros opcionais. Considere o caso de uma classe que representa o rótulo de informações nutricionais que aparece nos alimentos embalados. Esses rótulos têm alguns campos obrigatórios - tamanho da porção, porções por embalagem e calorias por porção - e mais de vinte campos opcionais - gordura total, gordura saturada, gordura trans, colesterol, sódio e assim por diante. A maioria dos produtos tem valores diferentes de zero para apenas alguns desses campos opcionais.

Que tipo de construtores ou *static factory methods*você deve escrever para essa classe? Tradicionalmente, os programadores têm usado o padrão de construtor telescópico, no qual você fornece a um construtor apenas os parâmetros necessários, outro com um único parâmetro opcional, um terceiro com dois parâmetros opcionais e assim por diante, culminando em um construtor com todos os parâmetros opcionais. É assim que fica na prática. Para fins de brevidade, apenas quatro campos opcionais são mostrados:

```java
// Padrão de construtor telescópico - não dimensiona bem!
public class NutritionFacts {
    private final int servingSize;  // (mL) required
    private final int servings;     // (per container) required
    private final int calories;     // (per serving) optional
    private final int fat;          // (g/serving) optional
    private final int sodium;       // (mg/serving) optional
    private final int carbohydrate; // (g/serving) optional

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat,sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

Quando você deseja criar uma instância, você usa o construtor com a lista de parâmetros mais curta contendo todos os parâmetros que você deseja definir:

```java
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27);
```

Normalmente, esta invocação de construtor exigirá muitos parâmetros que você não deseja definir, mas você é forçado a passar um valor para eles de qualquer maneira. Nesse caso, passamos um valor de 0 para gordura. Com “apenas” seis parâmetros, isso pode não parecer tão ruim, mas rapidamente sai do controle conforme o número de parâmetros aumenta

Resumindo, o padrão do construtor telescópico funciona, mas é difícil escrever o código do cliente quando há muitos parâmetros e ainda mais difícil lê-lo. O leitor fica se perguntando o que todos esses valores significam e deve contar cuidadosamente os parâmetros para descobrir. Longas sequências de parâmetros digitados de forma idêntica podem causar erros sutis. Se o cliente acidentalmente reverter dois desses parâmetros, o compilador não reclamará, mas o programa se comportará mal em tempo de execução [Item 51]().

Uma segunda alternativa quando você se depara com muitos parâmetros opcionais em um construtor é o padrão JavaBeans, no qual você chama um construtor sem parâmetros para criar o objeto e, em seguida, chama métodos setter para definir cada parâmetro obrigatório e cada parâmetro opcional de interesse:

```java
// Padrão JavaBeans - permite inconsistência, exige mutabilidade
public class NutritionFacts {
    // Parâmetros inicializados com valores padrão (se houver)
    private int servingSize = -1;   // Required; no default value
    private int servings = -1;      // Required; no default value
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    public NutritionFacts() { }
    
    // Setters
    public void setServingSize(int val) { 
        servingSize = val; 
    }
    public void setServings(int val) { 
        servings = val; 
    }
    public void setCalories(int val) { 
        calories = val;
    }
    public void setFat(int val) { 
        fat = val; 
    }
    public void setSodium(int val) { 
        sodium = val;  
    }
    public void setCarbohydrate(int val) {
         carbohydrate    = val; 
    }
}

```

Este padrão não tem nenhuma das desvantagens do padrão do construtor telescópico. É fácil, embora um pouco prolixo, criar instâncias e ler o código resultante:

```java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
cocaCola.setSodium(35);
cocaCola.setCarbohydrate(27);
```

Infelizmente, o padrão JavaBeans tem suas próprias desvantagens sérias. Como a construção é dividida em várias chamadas, um JavaBean pode estar em um estado inconsistente no meio de sua construção. A classe não tem a opção de impor consistência apenas verificando a validade dos parâmetros do construtor. Tentar usar um objeto quando ele está em um estado inconsistente pode causar falhas que são muito removidas do código que contém o bug e, portanto, difíceis de depurar. Uma desvantagem relacionada é que o padrão JavaBeans impede a possibilidade de tornar uma classe imutável [Item 17]() e requer esforço adicional por parte do programador para garantir a segurança do thread

É possível reduzir essas desvantagens “congelando” manualmente o objeto quando sua construção estiver completa e não permitindo que seja usado até que seja congelado, mas esta variante é pesada e raramente usada na prática. Além disso, pode causar erros em tempo de execução porque o compilador não pode garantir que o programador chame o método freeze em um objeto antes de usá-lo.

Felizmente, há uma terceira alternativa que combina a segurança do padrão do construtor telescópico com a legibilidade do padrão JavaBeans. É uma forma do padrão Builder [Gamma95](). Em vez de fazer o objeto desejado diretamente, o cliente chama um construtor (ou fábrica estática) com todos os parâmetros necessários e obtém um objeto construtor. Em seguida, o cliente chama métodos semelhantes a setter no objeto construtor para definir cada parâmetro opcional de interesse. Finalmente, o cliente chama um método de construção sem parâmetros para gerar o objeto, que normalmente é imutável. O construtor é normalmente uma classe de membro estático [Item 24]() da classe que ele constrói. É assim que fica na prática:

```java
    // Builder Pattern
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        // Optional parameters - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
       
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) { 
            calories = val; return this; 
        }

        public Builder fat(int val) { 
            fat = val; return this; 
        }

        public Builder sodium(int val) { 
            sodium = val; return this; 
        }

        public Builder carbohydrate(int val) { 
            carbohydrate = val; return this; 
        }
        
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }

    }
    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}

```

A classe NutritionFacts é imutável e todos os valores padrão dos parâmetros estão em um só lugar. Os métodos setter do construtor retornam o próprio construtor para que as invocações possam ser encadeadas, resultando em uma API fluente. Esta é a aparência do código do cliente:

```java
NutritionFacts cocaCola = 
    new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .carbohydrate(27)
        .build();
```

Este código cliente é fácil de escrever e, mais importante, fácil de ler. O padrão Builder simula parâmetros opcionais nomeados, conforme encontrados em Python e Scala

As verificações de validade foram omitidas por questões de brevidade. Para detectar parâmetros inválidos o mais rápido possível, verifique a validade do parâmetro no construtor e nos métodos do construtor. Verifique as invariantes que envolvem vários parâmetros no construtor invocado pelo método de construção. Para garantir essas invariáveis contra ataques, faça as verificações nos campos do objeto após copiar os parâmetros do construtor [Item 50](). Se uma verificação falhar, lance uma IllegalArgumentException [Item 72]() cuja mensagem de detalhe indica quais parâmetros são inválidos [Item 75]().

O padrão Builder é adequado para hierarquias de classes. Use uma hierarquia paralela de construtores, cada um aninhado na classe correspondente. As classes abstratas têm construtores abstratos; classes concretas têm construtores concretos. Por exemplo, considere uma classe abstrata na raiz de uma hierarquia que representa vários tipos de pizza:

```java
// Builder pattern for class hierarchies
public abstract class Pizza {
    public enum Topping { 
        HAM, MUSHROOM, ONION, PEPPER, SAUSAGE 
    }

    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
        
        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }
        
        // Subclasses must override this method to return "this"
        protected abstract T self();
    }

    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone(); // See Item 50
    }
}
```

Observe que Pizza.Builder é um tipo genérico com um parâmetro de tipo recursivo (Item 30). Isso, junto com o método self abstrato, permite que o encadeamento de métodos funcione corretamente em subclasses, sem a necessidade de casts. Essa solução alternativa para o fato de que Java não possui um tipo próprio é conhecida como idioma de tipo próprio simulado.

Aqui estão duas subclasses concretas de Pizza, uma das quais representa uma pizza padrão no estilo de Nova York, a outra um calzone. O primeiro tem um parâmetro de tamanho obrigatório, enquanto o último permite que você especifique se o molho deve estar dentro ou fora:

```java
public class NyPizza extends Pizza {
  public enum Size { SMALL, MEDIUM, LARGE }
  
  private final Size size;

  public static class Builder extends Pizza.Builder<Builder> {
    private final Size size;
    
    public Builder(Size size) {
        this.size = Objects.requireNonNull(size);
    }
    
    @Override public NyPizza build() {
        return new NyPizza(this);
    }
    
    @Override protected Builder self() { 
        return this; 
    }
  }

  private NyPizza(Builder builder) {
    super(builder);
    size = builder.size;
  }
}

public class Calzone extends Pizza {
  private final boolean sauceInside;

  public static class Builder extends Pizza.Builder<Builder> {
    private boolean sauceInside = false; // Default

    public Builder sauceInside() {
        sauceInside = true;
        return this;
    }

    @Override public Calzone build() {
        return new Calzone(this);
    }

    @Override protected Builder self() { 
        return this; 
    }
  }

  private Calzone(Builder builder) {
    super(builder);
    sauceInside = builder.sauceInside;
  }
}

```

Observe que o método de construção no construtor de cada subclasse é declarado para retornar a subclasse correta: o método de construção de NyPizza.Builder retorna NyPizza, enquanto o de Calzone.Builder retorna Calzone. Essa técnica, em que um método de subclasse é declarado para retornar um subtipo do tipo de retorno declarado na superclasse, é conhecida como tipagem de retorno covariante. Ele permite que os clientes usem estes construtores sem a necessidade de casting.

O código do cliente para esses “construtores hierárquicos” é essencialmente idêntico ao código do construtor NutritionFacts simples. O código do cliente de exemplo mostrado a seguir pressupõe importações estáticas em constantes enum para brevidade:

```java
NyPizza pizza = 
    new NyPizza.Builder(SMALL)
        .addTopping(SAUSAGE)
        .addTopping(ONION)
        .build();

Calzone calzone = 
    new Calzone.Builder()
        .addTopping(HAM)
        .sauceInside()
        .build();
```

Uma vantagem secundária dos construtores sobre os construtores é que os construtores podem ter vários parâmetros varargs porque cada parâmetro é especificado em seu próprio método. Como alternativa, os construtores podem agregar os parâmetros passados em várias chamadas para um método em um único campo, conforme demonstrado no método addTopping anteriormente.

O padrão Builder é bastante flexível. Um único construtor pode ser usado repetidamente para construir vários objetos. Os parâmetros do construtor podem ser ajustados entre as chamadas do método de construção para variar os objetos que são criados. Um construtor pode preencher alguns campos automaticamente na criação do objeto, como um número de série que aumenta cada vez que um objeto é criado

O padrão Builder também tem desvantagens. Para criar um objeto, você deve primeiro criar seu construtor. Embora seja improvável que o custo de criação desse construtor seja perceptível na prática, ele pode ser um problema em situações críticas de desempenho. Além disso, o padrão Builder é mais detalhado do que o padrão construtor telescópico, portanto, ele deve ser usado apenas se houver parâmetros suficientes para fazer com que valha a pena, digamos quatro ou mais. Mas lembre-se de que você pode querer adicionar mais parâmetros no futuro. Mas se você começar com construtores ou *static factory methods*e mudar para um construtor quando a classe evoluir até o ponto em que o número de parâmetros fica fora de controle, os construtores obsoletos ou *static factory methods*se destacarão como um polegar machucado. Portanto, muitas vezes é melhor começar com um construtor em primeiro lugar.

Em resumo, o padrão Builder é uma boa escolha ao projetar classes cujos construtores ou *static factory methods*teriam mais do que um punhado de parâmetros, especialmente se muitos dos parâmetros forem opcionais ou de tipo idêntico. O código do cliente é muito mais fácil de ler e escrever com construtores do que com construtores telescópicos, e os construtores são muito mais seguros do que JavaBeans.