# Plantillas
Son generalmente documentos HTML.
Se usan para separar la parte lógica de la visual

##  ¿Cómo se usan?
1. Crear un objeto de tipo template y leerlo
2. Crear un contexto: Son los datos adicionales que pueda usar ese html, por ejemplo contenido dinámico. SIEMPRE se debe de crear el contexto aunque esté vacío.
3. Renderizar el objeto template. Se le debe de pasar el contexto como parámetro.

## Uso de variables 
Para el uso de las variables, vamos a usar el contexto. Se mandará como parte de un diccionario en sus argumentos
De modo que tengamos:
contexto = Context({ "<clave>":<variable> })

Y en el documento de la plantilla/vista HTML se despliega de la siguiente forma:
<p>El nombre del usuario es: {{ <clave> }} </p>

Cabe destacar que solo se despliega UN valor a la vez, entonces si tuviéramos varias variables se haría de la siguiente forma:
{{ <clave> }} {{ <clave }}

También sería bueno mencionar, que podemos pasar variables, clases, etc, en el contexto. 

## Acceso a Objetos y propiedades desde plantillas
Retomando la programación orientada a objetos, también podemos crear clases y acceder a las instancias de dicha clase.
Para esto podemos crear una clase en el mismo archivo de "Views.py".

Donde definimos a una clase Persona que tenga nombre y apellido.
====================================================
class Persona(object):
    # Constructor
    def __init__(self, nombre, apellido):
        self.nombre = nombre
        self.apellido = apellido


Creamos un objeto de tipo persona en nuestra función
====================================================
def saludo(request):
    p1 = Persona("Pablo", "Del Valle")


Y la mandamos en el contexto de la función, en este caso, saludo:
====================================================
contexto = Context({"nombre_persona": p1.nombre, "apellido_persona":p1.apellido})



