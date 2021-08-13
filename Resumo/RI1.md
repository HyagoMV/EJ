# Item 1 - Static Factory Methods

## Descrição
* É simplesmente um método estático que retorna uma instância da classe
* Static Factory Methods é diferente do Design Pattern Factory Method

## Vantagens
1)  Ao contrário do método construtor o desenvolvedor  **escolhe o nome** do *Static Factory Method*
    
    ``` java 
    BigInteger(...)                 // Método Construtor
    BigInteger.probablePrime(...)   // Static Factory Method 
    ```
    
    * Nomes bem escolhidos para os *Static Factory Methods* melhoram:
        * O uso da API, fazendo que usuários lebrem dos métodos corretos
        * A leitura do código e diminue as consultas a documentação
    * Não possuem a mesma restrição dos métodos construtores em relação a assinatura do método:
        * *Static Factory Methods* podem ter as mesma listas de parâmetros se o nome do método for diferente

2) Um *Static Factory Method* não é obrigados a criar um novo objeto cada vez que são invocados.

    ```java
    // A class Boolean é imutável
    public static Boolean valueOf(boolean b) {
        return b ? Boolean.TRUE : Boolean.FALSE; // Retorna sempre as mesma instâncias pré-construídas
    }
    ```
    * **Classes imutáveis** (Item 17) podem:
        * Usar instâncias pré-construídas 
        * Armazenar em cache as instâncias já construídas
        * Evitar a criação de objetos duplicados imprudentemente

## Desvantagens

## Exemplo

