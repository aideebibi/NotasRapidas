## Plantillas
Son generalmente documentos HTML.
Se usan para separar la parte lógica de la visual

#  ¿Cómo se usan?
1. Crear un objeto de tipo template y leerlo
2. Crear un contexto: Son los datos adicionales que pueda usar ese html, por ejemplo contenido dinámico. SIEMPRE se debe de crear el contexto aunque esté vacío.
3. Renderizar el objeto template. Se le debe de pasar el contexto como parámetro.
