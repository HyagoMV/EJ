# Item 9: Prefira tentar com recursos para tentar finalmente

As bibliotecas Java incluem muitos recursos que devem ser fechados manualmente chamando um método de fechamento. Os exemplos incluem InputStream, OutputStream e java.sql.Connection. O fechamento de recursos é freqüentemente esquecido pelos clientes, com consequências de desempenho previsivelmente terríveis. Embora muitos desses recursos usem finalizadores como rede de segurança, os finalizadores não funcionam muito bem [Item 8]().

Historicamente, uma instrução try-finally era a melhor maneira de garantir que um recurso seria fechado corretamente, mesmo em face de uma exceção ou retorno:

```java
// try-finally - No longer the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}

```

Isso pode não parecer ruim, mas fica pior quando você adiciona um segundo recurso:

```java
// try-finally is ugly when used with more than one resource!
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}

```

Pode ser difícil de acreditar, mas mesmo bons programadores erraram na maioria das vezes. Para começar, entendi errado na página 88 do Java Puzzlers [Bloch05], e ninguém percebeu durante anos. Na verdade, dois terços dos usos do método close nas bibliotecas Java estavam errados em 2007.

Mesmo o código correto para fechar recursos com instruções try-finally, conforme ilustrado nos dois exemplos de código anteriores, tem uma deficiência sutil. O código no bloco try e no bloco finally é capaz de lançar exceções. Por exemplo, no método firstLineOfFile, a chamada para readLine poderia lançar uma exceção devido a uma falha no dispositivo físico subjacente, e a chamada para fechar poderia falhar pelo mesmo motivo. Nessas circunstâncias, a segunda exceção oblitera completamente a primeira. Não há registro da primeira exceção no rastreamento de pilha de exceções, o que pode complicar muito a depuração em sistemas reais - geralmente é a primeira exceção que você deseja ver para diagnosticar o problema. Embora seja possível escrever código para suprimir a segunda exceção em favor da primeira, virtualmente ninguém o fez porque é muito prolixo.

Todos esses problemas foram resolvidos de uma só vez quando o Java 7 introduziu a instrução try-with-resources [JLS, 14.20.3]. Para ser utilizável com essa construção, um recurso deve implementar a interface AutoCloseable, que consiste em um único método de fechamento de retorno de vazio. Muitas classes e interfaces nas bibliotecas Java e em bibliotecas de terceiros agora implementam ou estendem AutoCloseable. Se você escrever uma classe que representa um recurso que deve ser fechado, sua classe também deve implementar AutoCloseable.

Veja como fica nosso primeiro exemplo usando try-with-resources:

```java
// try-with-resources - the the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}

```

E aqui está a aparência do nosso segundo exemplo usando try-with-resources:

```java
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}

```

Não apenas as versões de teste com recursos são mais curtas e mais legíveis do que os originais, mas também fornecem diagnósticos muito melhores. Considere o método firstLineOfFile. Se exceções forem lançadas pela chamada readLine e pelo fechamento (invisível), a última exceção será suprimida em favor da primeira. Na verdade, várias exceções podem ser suprimidas para preservar a exceção que você realmente deseja ver. Essas exceções suprimidas não são meramente descartadas; eles são impressos no rastreamento da pilha com uma notação informando que foram suprimidos. Você também pode acessá-los programaticamente com o método getSuppressed, que foi adicionado ao Throwable no Java 7.

Você pode colocar cláusulas catch em instruções try-with-resources, da mesma forma que faz com instruções try-finally regulares. Isso permite que você lide com exceções sem poluir seu código com outra camada de aninhamento. Como um exemplo ligeiramente artificial, aqui está uma versão do nosso método firstLineOfFile que não lança exceções, mas leva um valor padrão para retornar se não puder abrir o arquivo ou ler a partir dele:

```java
// try-with-resources with a catch clause
static String firstLineOfFile(String path, String defaultVal) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }
}

```

A lição é clara: sempre use try-with-resources em vez de tentarfinalmente ao trabalhar com recursos que devem ser fechados. O código resultante é mais curto e claro, e as exceções que ele gera são mais úteis. A instrução try-with-resources torna mais fácil escrever o código correto usando recursos que devem ser fechados, o que era praticamente impossível usando try-finally