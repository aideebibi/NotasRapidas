# Ataque de tipo "xxe processing"
Se da en sistemas que realizan transacciones mediante archivos XML.

A continuación se muestra un ejemplo de un sistema en donde se realiza este tipo de ataque.
1. Los actores de nuestro escenario: Alice (Usuario) y Bob (Hacker)

Alice tiene el siguiente archivo XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<trades>
    <metadata>
        <name>Apple Inc</name>
        <stock>APPL</stock>
        <trader>
            <name>C.K Frode</name>
        </trader>
        <units>1500</units>
        <price>106</price>
        <name>Microsoft Corp</name>
        <stock>MSFT</stock>
        <trader>
            <name>C.K Frode</name>
        </trader>
        <units>5000</units>
        <price>45</price>
        <name>Amazon Inc</name>
        <stock>AMZN</stock>
        <trader>
            <name>C.K Frode</name>
        </trader>
        <units>4500</units>
        <price>195</price>
    </metadata>
</trades>
```
Al cargar este archivo se obtiene lo siguiente en la aplicación:

## Entrada XXE Maliciosa
Un ataque XXE funciona aprovechando una característica de XML, a saber, XML eXternal Entities (XXE), que permite cargar recursos XML externos en un documento XML.
Al enviar un archivo XML que define una entidad externa con un file://URI, un atacante puede engañar efectivamente al analizador SAX de la aplicación para que lea el contenido de archivos arbitrarios que residen en el sistema de archivos del lado del servidor.

Entonces Bob va a subir el siguiente archivo al sistema:
```xml
<!DOCTYPE foo [<!ELEMENT foo ANY >
<!ENTITY bar SYSTEM "file:///etc/passwd" >]>

<trades>
    <metadata>
        <name>Apple Inc</name>
        <stock>APPL</stock>
        <trader>
            <foo>&bar;</foo>
            <name>C.K Frode</name>
        </trader>
        <units>1500</units>
        <price>106</price>
    </metadata>
</trades>
```
Vamos a analizar el archivo que acaba de cargar Bob:
* La declaración `DOCTYPE foo` hace referencia a un archivo externo de definición de tipo de documento (DTD), que hemos denominado `foo`
* La declaración XML `ELEMENT foo ANY` declara que `foo` DTD puede contener cualquier combinación de datos analizables.
* Finalmente usamos la declaración XML `ENTITY` para cargar datos adicionales desde un recurso externo. La sintaxis de la declaración es `ENTITY name SYSTEM URI` donde `URI` está la ruta completa a una URL remota o un archivo local. En nuestro ejemplo definimos la etiqueta ENTITY para cargar el contenido de "file:///etc/passwd"
* La línea `<foo>&bar;</foo>` asigna nuestra etiqueta `foo` a la entidad externa `&bar;` que apunta a "file:///etc/passwd"
* Entonces cuando este documento XML sea procesado por el analizador SAX de BatchTRADER, cualquier instancia de `&bar;` será reemplazada por el contenido del archivo /etc/passwd.

## Accediendo a los archivos del sistema
Una vez que Bob sube el archivo al sistema obtiene lo siguiente:
![Bob XXE](/img/XXE_2.PNG)

Bob ahora tiene acceso al archivo del servidor web de BatchTRADER /etc/passwd. Analicemos cómo se explotó la falla de procesamiento XXE.
Mediante el uso de entidades XML, la palabra clave ***SYSTEM*** hace que un analizador XML lea datos de un URIy permite sustituirlos en el documento.

En pocas palabras, se engañó al analizador XML para que accediera a un recurso que el desarrollador de la aplicación no pretendía que fuera accesible, en este caso, un archivo en el sistema de archivos local del servidor remoto.
En este ejemplo, el siguiente código XML obtuvo el archivo /etc/passwd presente en el servidor web de BatchTRADER y se lo mostró al usuario de la aplicación.
` <!ENTITY bar SYSTEM "/etc/passwd" >]> `

Debido a esta vulnerabilidad, se podría obtener cualquier archivo en el servidor remoto (o más precisamente, cualquier archivo al que el servidor web tenga acceso de lectura).
Un atacante malicioso podría usar esto fácilmente para obtener acceso a archivos arbitrarios, como archivos del sistema o archivos que contienen datos confidenciales, como contraseñas o datos de usuario.

## Código vulnerable
Las aplicaciones web que utilizan bibliotecas XML son particularmente vulnerables a los ataques de procesamiento de entidades externas (XXE) porque la configuración predeterminada para la mayoría de los analizadores XML SAX es tener XXE habilitado de manera predeterminada.

Para utilizar estos analizadores de forma segura, debe deshabilitar explícitamente la referencia a entidades externas en la implementación del analizador SAX que utilice. Volveremos a revisar esto en la sección de remediación más adelante.

Ahora echemos un vistazo a cómo el analizador SAX del lado del servidor de BatchTRADE está configurado para procesar archivos XML entrantes.

```java
1 public class TradeDocumentBuilderFactory {
2    public static DocumentBuilderFactory newDocumentBuilderFactory() {
3        DocumentBuilderFactory documentBuilderFactory = DocumentBuilderFactory.newInstance();
4        try {
5              documentBuilderFactory.setFeature("http://xml.org/sax/features/external-general-entities", true);
6              documentBuilderFactory.setFeature("http://xml.org/sax/features/external-parameter-entities", true);
7        } catch(ParserConfigurationException e) {
8            throw new RuntimeException(e);
9        }
10        return documentBuilderFactory;
11    }
```
Analicemos el código:
* Línea 3: El analizador SAX del lado del servidor de BatchTRADER se desarrolla utilizando la clase `DocumentBuilderFactory`, parte de la API de Java para el procesamiento de XML, o JAXP para abreviar, que permite que las aplicaciones analicen y transformen documentos XML.
* Línea 4: Para analizar el archivo trade.xml, `DocumentBuilderFactory` se crea una nueva instancia de JAXP usando el método estático `newInstance()`
* Línea 6: La clase `DocumentBuilderFactory` contiene además un método `setFeature(String,boolean)` que se puede utilizar para establecer funciones en el analizador SAX subyacente
* Línea 6: En este ejemplo, los desarrolladores configuraron el analizador SAX utilizando el setFeaturemétodo para habilitar la carga `external-general-entities` configurando su valor en `true`
* Línea 7: De manera similar, el analizador SAX también se ha configurado para procesar `external-parameter-entities` entidades. Ambas opciones permiten que el analizador SAX cargue entidades externas que, cuando se especifican dentro de nuestro archivo `trade.xml`, pueden ser abusadas por un atacante para leer archivos arbitrarios del sistema. Si se configuraran en `false` el analizador SAX, rechazaría automáticamente la referencia de entidades externas.

## Solución
Debido a que la entrada XML proporcionada por el usuario proviene de una "fuente no confiable", es muy difícil validar correctamente el documento XML para evitar este tipo de ataque.

En su lugar, el procesador XML debe configurarse para usar solo una definición de tipo de documento (DTD) definida localmente y rechazar cualquier DTD en línea que se especifique dentro de los documentos XML proporcionados por el usuario.

Debido al hecho de que existen numerosos motores de análisis XML disponibles, cada uno tiene su propio mecanismo para deshabilitar DTD en línea para evitar XXE. Es posible que deba buscar en la documentación de su analizador XML cómo "deshabilitar DTD en línea" específicamente.

Veamos cómo se puede aplicar la solución anterior a nuestro ejemplo vulnerable para remediar la vulnerabilidad XXE.

```java
1 public class TradeDocumentBuilderFactory {
2    public static DocumentBuilderFactory newDocumentBuilderFactory() {
3        DocumentBuilderFactory documentBuilderFactory = DocumentBuilderFactory.newInstance();
4        try {
5 //              documentBuilderFactory.setFeature("http://xml.org/sax/features/external-general-entities", true);
6 //              documentBuilderFactory.setFeature("http://xml.org/sax/features/external-parameter-entities", true);
7                 documentBuilderFactory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
8                 documentBuilderFactory.setFeature("http://xml.org/sax/features/external-general-entities", false);
9                 documentBuilderFactory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);    
10        } catch(ParserConfigurationException e) {
11            throw new RuntimeException(e);
12        }
13        return documentBuilderFactory;
14    }
15 }
```
Analicemos el código:
* Línea 8: El método más sólido para protegerse contra los ataques XXE es configurar el analizador XML de aplicaciones para que no permita declaraciones `DOCTYPE`. Esto se hace configurando el parámetro `disallow-doctype-decl` de los analizadores en verdadero. Con este conjunto, se produce una excepción si nuestro `trade.xml` contiene una declaración `DOCTYPE` y se detiene el análisis, lo que evita que la vulnerabilidad exponga información confidencial.
* Línea 9: Sin embargo, si la aplicación requiere declaraciones `DOCTYPE`, una buena alternativa es configurar el analizador SAX del lado del servidor para que no permita la declaración de entidades externas estableciendo el valor `external-general-entities` para `false`
* Línea 10: Del mismo modo, también podemos deshabilitar `external-parameter-entities` a través del método `setFeature`.



