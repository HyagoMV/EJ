# Item 4: Impeder a criação de instancia com um construtor privado (Editado)Restaurar original

Ocasionalmente, você desejará escrever uma classe que seja apenas um agrupamento de métodos e campos estáticos. Essas classes adquiriram má reputação porque algumas pessoas abusam delas para evitar pensar em termos de objetos, mas elas têm usos válidos. Eles podem ser usados para agrupar métodos relacionados em valores primitivos ou matrizes, na forma de `java.lang.Math` ou `java.util.Arrays`. Eles também podem ser usados para agrupar métodos estáticos, incluindo fábricas [Item 1](), para objetos que implementam alguma interface, na forma de `java.util.Collections`. (A partir do Java 8, você também pode colocar esses métodos na interface, supondo que sejam seus para modificar.) Por último, essas classes podem ser usadas para agrupar métodos em uma classe final, uma vez que você não pode colocá-los em uma subclasse.

Essas classes de utilitários não foram projetadas para serem instanciadas: uma instância não faria sentido. Na ausência de construtores explícitos, no entanto, o compilador fornece um construtor padrão público e sem parâmetros. Para um usuário, esse construtor é indistinguível de qualquer outro. Não é incomum ver classes instanciáveis involuntariamente em APIs publicadas.

**Tentar impeder a criação de instanciai tornando uma classe abstrata não funciona**. A classe pode ter uma subclasse e instanciar a subclasse. Além disso, induz o usuário a pensar que a classe foi projetada para herança [Item 19)](). Existe, no entanto, um idioma simples para garantir a não instabilidade. Um construtor padrão é gerado apenas se uma classe não contém construtores explícitos, portanto, uma classe pode se tornar não instável incluindo um construtor privado:

```java
// Classe de utilidade não instanciável
public class UtilityClass {
    // Suprimir construtor padrão para não ser instanciável
    private UtilityClass() {
        throw new AssertionError();
    }
... // Restante omitido
}
```

Como o construtor explícito é privado, ele fica inacessível fora da classe. O  `AssertionError` não é estritamente necessário, mas fornece-lo é seguro caso o construtor seja invocado acidentalmente de dentro da classe. Ele garante que a classe nunca será instanciada em nenhuma circunstância. Esse forma é ligeiramente contra-intuitivo porque o construtor é fornecido expressamente para que não possa ser chamado. Portanto, é aconselhável incluir um comentário, conforme mostrado anteriormente.

Como efeito colateral, esse forma também evita que a classe seja subclassificada. Todos os construtores devem invocar um construtor de superclasse, explícita ou implicitamente, e uma subclasse não teria nenhum construtor de superclasse acessível para invocar