# A CLI with Spring Shell

# 1. Visão Geral
Simplificando, o projeto Spring Shell fornece um shell interativo para processamento de comandos e construção de uma CLI completa usando o modelo de programação Spring.

Neste artigo, exploraremos seus recursos, classes principais e anotações, e implementaremos vários comandos e personalizações personalizados.

# 2. Dependência Maven
Primeiro, precisamos adicionar a dependência spring-shell ao nosso pom.xml:

```
<dependency>
    <groupId>org.springframework.shell</groupId>
    <artifactId>spring-shell</artifactId>
    <version>1.2.0.RELEASE</version>
</dependency>
```

A versão mais recente deste artefato pode ser encontrada aqui.

# 3. Acessando o Shell
Existem duas maneiras principais de acessar o shell em nossos aplicativos.

A primeira é inicializar o shell no ponto de entrada de nosso aplicativo e permitir que o usuário insira os comandos:

```
public static void main(String[] args) throws IOException {
    Bootstrap.main(args);
}
```

A segunda é obter um JLineShellComponent e executar os comandos programaticamente:

```
Bootstrap bootstrap = new Bootstrap();
JLineShellComponent shell = bootstrap.getJLineShellComponent();
shell.executeCommand("help");
```

Usaremos a primeira abordagem, uma vez que é mais adequada para os exemplos neste artigo, no entanto, no código-fonte você pode encontrar casos de teste que usam a segunda forma.

# 4. Comandos
Já existem vários comandos integrados no shell, como clear, help, exit, etc., que fornecem a funcionalidade padrão de cada CLI.

Comandos personalizados podem ser expostos adicionando métodos marcados com a anotação @CliCommand dentro de um componente Spring implementando a interface CommandMarker.

Cada argumento desse método deve ser marcado com uma anotação @CliOption, se não fizermos isso, encontraremos vários erros ao tentar executar o comando.

4.1. Adicionando Comandos ao Shell
Primeiro, precisamos deixar o shell saber onde estão nossos comandos. Para isso, é necessário que o arquivo META-INF / spring / spring-shell-plugin.xml esteja presente em nosso projeto, aí podemos usar a funcionalidade de escaneamento de componentes do Spring:

```
<beans ... >
    <context:component-scan base-package="org.isac.shell.simple" />
</beans>
```

Depois que os componentes são registrados e instanciados pelo Spring, eles são registrados com o analisador de shell e suas anotações são processadas.

Vamos criar dois comandos simples, um para pegar o conteúdo de um URL e exibi-lo e outro para salvar esse conteúdo em um arquivo:

```
@Component
public class SimpleCLI implements CommandMarker {

    @CliCommand(value = { "web-get", "wg" })
    public String webGet(
      @CliOption(key = "url") String url) {
        return getContentsOfUrlAsString(url);
    }
    
    @CliCommand(value = { "web-save", "ws" })
    public String webSave(
      @CliOption(key = "url") String url,
      @CliOption(key = { "out", "file" }) String file) {
        String contents = getContentsOfUrlAsString(url);
        try (PrintWriter out = new PrintWriter(file)) {
            out.write(contents);
        }
        return "Done.";
    }
}
```

Observe que podemos passar mais de uma string para os atributos de valor e chave de @CliCommand e @CliOption respectivamente, isso nos permite expor vários comandos e argumentos que se comportam da mesma forma.

Agora, vamos verificar se tudo está funcionando conforme o esperado:

```
spring-shell>web-get --url https://www.google.com
<!doctype html ... 
spring-shell>web-save --url https://www.google.com --out contents.txt
Done.
```

### 4.2. Disponibilidade de comandos
Podemos usar a anotação @CliAvailabilityIndicator em um método que retorna um booleano para alterar, em tempo de execução, se um comando deve ser exposto no shell.

Primeiro, vamos criar um método para modificar a disponibilidade do comando web-save:

```
private boolean adminEnableExecuted = false;

@CliAvailabilityIndicator(value = "web-save")
public boolean isAdminEnabled() {
    return adminEnableExecuted;
}
```

Agora, vamos criar um comando para alterar a variável adminEnableExecuted:

```
@CliCommand(value = "admin-enable")
public String adminEnable() {
    adminEnableExecuted = true;
    return "Admin commands enabled.";
}
```

Finalmente, vamos verificar:

```
spring-shell>web-save --url https://www.google.com --out contents.txt
Command 'web-save --url https://www.google.com --out contents.txt'
  was found but is not currently available
  (type 'help' then ENTER to learn about this command)
spring-shell>admin-enable
Admin commands enabled.
spring-shell>web-save --url https://www.google.com --out contents.txt
Done.
```

### 4.3. Argumentos Requeridos
Por padrão, todos os argumentos do comando são opcionais. No entanto, podemos torná-los obrigatórios com o atributo obrigatório da anotação @CliOption:

```
@CliOption(key = { "out", "file" }, mandatory = true)
```

Agora, podemos testar que, se não o introduzirmos, resultará em um erro:

```
spring-shell>web-save --url https://www.google.com
You should specify option (--out) for this command
```

### 4.4. Argumentos Padrão
Um valor de chave vazio para um @CliOption torna esse argumento o padrão. Lá, receberemos os valores introduzidos no shell que não fazem parte de nenhum argumento nomeado:

```
@CliOption(key = { "", "url" })
```

Agora, vamos verificar se funciona conforme o esperado:

```
spring-shell>web-get https://www.google.com
<!doctype html ...
```

### 4.5. Ajudando Usuários
As anotações @CliCommand e @CliOption fornecem um atributo de ajuda que nos permite orientar nossos usuários ao usar o comando de ajuda integrado ou ao tabular para obter o preenchimento automático.

Vamos modificar nosso web-get para adicionar mensagens de ajuda personalizadas:

```
@CliCommand(
  // ...
  help = "Displays the contents of an URL")
public String webGet(
  @CliOption(
    // ...
    help = "URL whose contents will be displayed."
  ) String url) {
    // ...
}
```

Agora, o usuário pode saber exatamente o que nosso comando faz:

```
spring-shell>help web-get
Keyword:                    web-get
Keyword:                    wg
Description:                Displays the contents of a URL.
  Keyword:                  ** default **
  Keyword:                  url
    Help:                   URL whose contents will be displayed.
    Mandatory:              false
    Default if specified:   '__NULL__'
    Default if unspecified: '__NULL__'

* web-get - Displays the contents of a URL.
* wg - Displays the contents of a URL.
```

# 5. Personalização

Existem três maneiras de personalizar o shell implementando as interfaces BannerProvider, PromptProvider e HistoryFileNameProvider, todas com implementações padrão já fornecidas.

Além disso, precisamos usar a anotação @Order para permitir que nossos provedores tenham precedência sobre essas implementações.

Vamos criar um novo banner para começar nossa customização:

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SimpleBannerProvider extends DefaultBannerProvider {

    public String getBanner() {
        StringBuffer buf = new StringBuffer();
        buf.append("=======================================")
            .append(OsUtils.LINE_SEPARATOR);
        buf.append("*          Isac Shell             *")
            .append(OsUtils.LINE_SEPARATOR);
        buf.append("=======================================")
            .append(OsUtils.LINE_SEPARATOR);
        buf.append("Version:")
            .append(this.getVersion());
        return buf.toString();
    }

    public String getVersion() {
        return "1.0.1";
    }

    public String getWelcomeMessage() {
        return "Welcome to Isac CLI";
    }

    public String getProviderName() {
        return "Isac Banner";
    }
}
```

Observe que também podemos alterar o número da versão e a mensagem de boas-vindas.

Agora, vamos mudar o prompt:

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SimplePromptProvider extends DefaultPromptProvider {

    public String getPrompt() {
        return "isac-shell";
    }

    public String getProviderName() {
        return "Isac Prompt";
    }
}
```

Finalmente, vamos modificar o nome do arquivo de histórico:

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SimpleHistoryFileNameProvider
  extends DefaultHistoryFileNameProvider {

    public String getHistoryFileName() {
        return "isac-shell.log";
    }

    public String getProviderName() {
        return "Isac History";
    }

}
```

O arquivo de histórico gravará todos os comandos executados no shell e será colocado junto com nosso aplicativo.

Com tudo no lugar, podemos chamar nosso shell e vê-lo em ação:

```
=======================================
*          Isac Shell             *
=======================================
Version:1.0.1
Welcome to Isac CLI
isac-shell>
```

# 6. Conversores
Até agora, usamos apenas tipos simples como argumentos para nossos comandos. Tipos comuns como Inteiro, Data, Enum, Arquivo, etc., têm um conversor padrão já registrado.

Ao implementar a interface do Conversor, também podemos adicionar nossos conversores para receber objetos personalizados.

Vamos criar um conversor que pode transformar uma String em um URL:

```
@Component
public class SimpleURLConverter implements Converter<URL> {

    public URL convertFromText(
      String value, Class<?> requiredType, String optionContext) {
        return new URL(value);
    }

    public boolean getAllPossibleValues(
      List<Completion> completions,
      Class<?> requiredType,
      String existingData,
      String optionContext,
      MethodTarget target) {
        return false;
    }

    public boolean supports(Class<?> requiredType, String optionContext) {
        return URL.class.isAssignableFrom(requiredType);
    }
}
```

Finalmente, vamos modificar nossos comandos web-get e web-save:

```
public String webSave(... URL url) {
    // ...
}

public String webSave(... URL url) {
    // ...
}
```

Como você deve ter adivinhado, os comandos se comportam da mesma forma.

# 7. Conclusão
Neste artigo, demos uma breve olhada nos principais recursos do projeto Spring Shell. Conseguimos contribuir com nossos comandos e personalizar o shell com nossos provedores, alteramos a disponibilidade de comandos de acordo com as diferentes condições de tempo de execução e criamos um conversor de tipo simples.