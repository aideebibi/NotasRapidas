# Command Injection
Veremos cómo se utiliza un ataque de inyección de comandos para comprometer la seguridad de una aplicación web y cómo escribir código de manera más segura para protegerse contra este tipo de ataques.

## Escenario
El panel de aplicaciones vulnerables carga la aplicación TradeADMIN , un portal de informes y administración de cuentas basado en la web. Los usuarios registrados de la aplicación pueden iniciar sesión en el sistema para realizar funciones de administración de cuentas, como agregar usuarios al sistema y ejecutar informes analíticos.

Alice es una comerciante y una usuaria registrada de la aplicación. Mientras que Bob es el atacante.
![Sistema Command Injection](/img/CommandInjection_1.PNG)

El equipo de desarrollo agregó recientemente una nueva función para permitir a los usuarios de TradeADMIN realizar análisis estadísticos de sus operaciones históricas con fines de gestión de riesgos.
![Análisis del sistema Command Injection](/img/CommandInjection_2.PNG)

## Código Vulnerable
Para crear el informe analítico, el código del lado del servidor de la aplicación web invoca una aplicación C++ personalizada llamada `statlab` para realizar el análisis estadístico computacionalmente intensivo.
Analicemos primero cómo se implementa esta funcionalidad para la aplicación TradeADMIN .

```java
1 // The following is a code snippet illustrating the use of insecure command execution function in Java
2 public class CommandExecuter {
3     public string executeCommand(String userName)
4     {
5         try {
6             String myUid = userName;
7             Runtime rt = Runtime.getRuntime();
8             rt.exec("/usr/bin/statlab " + ”-“ + myUid); // Call statlab with Alice's username
9             // process results for userID and return output in HTML.
10            // ...
11        }catch(Exception e)
12        {
13            e.printStackTrace();
14        }    
15    }
```
Analizando el código:
* Línea 6: Los desarrolladores de TradeADMIN han implementado el método `executeCommand` que acepta un argumento `String userName`. El método acepta un nombre de usuario válido registrado en el portal TradeADMIN.
* Línea 8: El método `executeCommand` llama a la función `java.lang.Runtime.exec()` que invoca la aplicación `/usr/bin/statlab`. El método `exec()` acepta además `myUID` como parámetro, cuyo valor se pasa al programa `statlab` como argumento. Para el perfil de Alice, la cadena `exec()` resultante ejecutada por sería: `/usr/bin/statlab -alice`
* Línea 9: El controlador de informes `statlab` de TradeADMIN analiza la salida del programa y la salida se pasa a una plantilla de vista para representar el resultado en HTML.


## Inyección de comandos
Para comprender cómo funcionan los ataques de inyección de comandos, analicemos la URL que se pasa a nuestro código del lado del servidor.
`https://tradeadmin.codebashing.com/console/execute?username=alicia`
Cuando Alice genera un informe de análisis de riesgos, el servidor TradeADMIN analiza la URL anterior, específicamente la cadena de consulta `username=alice`, que luego se pasa a la función para generar informes analíticos para la cuenta comercial de Alice.
Podemos modificar la url del sistema agregando al final la cadena `;id` algo así: `https://tradeadmin.codebashing.com/console/execute?username=alice;id`

![Análisis del sistema Command Injection](/img/CommandInjection_3.PNG)
¡Interesante! No solo se muestra el informe analítico para Alice , sino que también se muestra un error de salida inesperado , y contiene la salida del comando `;id` que se ejecutó en el servidor.

Tenga en cuenta que en los sistemas UNIX `;id` es un comando que devuelve la identificación de usuario del usuario que ejecutó el comando. Pero, ¿cómo logró Alice ejecutar este comando?

Veamos el código vulnerable para comprender cómo funciona el ataque de ejecución de comandos a nivel de código.
```java
1 // The following is a code snippet illustrating the use of insecure command execution function in Java
2 public class CommandExecuter {
3     public string executeCommand(String userName)
4     {
5         try {
6             String myUid = userName;
7             Runtime rt = Runtime.getRuntime();
8             rt.exec("/usr/bin/statlab " + ”-“ + myUid); // Call statlab with Alice's username
9             // process results for userID and return output in HTML.
10            // ...
11        }catch(Exception e)
12        {
13            e.printStackTrace();
14        }    
15    }
```
* Línea 3: Tras la modificación de nuestra URL, el método `executeCommand` se pasa con el siguiente argumento: `alice;id`
* Línea 6: A la variable de cadena `myUid` se le asigna el valor de cadena de consulta de Alice `alice;id`
* Línea 7: El método `executeCommand` inicializa un entorno de tiempo de ejecución invocando un método `Runtime.getRuntime()` que permite que la aplicación TradeADMIN interactúe con su entorno de tiempo de ejecución.
* Línea 8: Finalmente `executeCommand` invoca el programa `statlab` a través de `exec()`, pasando la cadena de consulta modificada de Alice `alice;id` como argumento a través de `myUid` la variable de cadena. El comando final ejecutado en el servidor TradeADMIN sería: `statlab -alice;id`. El valor `myUid` no se valida de ninguna manera antes de pasar al método `exec()`. Dado que el carácter `;` se interpreta como un separador de comandos en sistemas operativos similares a UNIX, la cadena de consulta de Alice `exec()` se interpreta por método como dos comandos separados, es decir, `statlab -alice` y `id` Sin embargo la salida del comando `id` no puede ser analizado por el template y el error generado por la aplicación contiene el resultado del comando `id`.

## Solución
```java
1 // The following is a code snippet illustrating the use of insecure command execution function in Java
2 public class CommandExecuter {
3     public string executeCommand(String userName)
4     {
5         try {
6             String myUid = userName;
7             if (!Pattern.matches("[0-9A-Za-z]+", myUiD)) {  
8                   return false;
9             }
10            Runtime rt = Runtime.getRuntime();
11             rt.exec("/usr/bin/statlab " + ”-“ + myUid); // Call statlab with Alice's username
12             // process results for userID and return output in HTML.
13            // ...
14        }catch(Exception e)
15        {
16            e.printStackTrace();
17        }    
18    }
```


En nuestro ejemplo de código modificado, se introduce una verificación adicional que realiza la validación de entrada contra la variable `myUiD`. Para lograr esto, utilizamos el método de Java `Pattern.matches()` para ejecutar una búsqueda de expresiones regulares en variables, identificando caracteres no alfanuméricos, por ejemplo ;, <, >, , ", ', &.

Si se encuentran caracteres no alfanuméricos, la verificación fallará y regresará, evitando así que se pasen al programa caracteres de shell de control maliciosos.
Nota: Aunque la corrección propuesta es suficiente para remediar nuestro ejemplo vulnerable, la lógica general y el diseño de seguridad para el método `executeCommand()` se pueden mejorar significativamente al no aceptar el valor `myUiD` proporcionado por el usuario a través del parámetro `username` (línea 6).

Un mejor enfoque sería extraer el nombre de usuario de Alice de un registro de la base de datos o una variable de índice estático que se establece durante el proceso de creación de la cuenta de Alice , que luego se puede pasar como argumento al programa para su ejecución.




