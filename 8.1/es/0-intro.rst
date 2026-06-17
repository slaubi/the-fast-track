¿De qué trata este libro?
===========================

Symfony es uno de los proyectos PHP más exitosos. Se trata tanto de un framework full-stack (cliente y servidor), sólido y completo, como de un conjunto de componentes reutilizables.

Desde el lanzamiento de Symfony 2.0 en 2011, el proyecto ha alcanzado ahora su madurez. Siento que todo lo que hemos hecho en los últimos años forma un gran conjunto. Nuevos componentes de bajo nivel, integraciones de alto nivel con otro software, herramientas que ayudan a los desarrolladores a mejorar su productividad. La experiencia del desarrollador ha mejorado sustancialmente sin sacrificar la flexibilidad. Nunca ha sido tan divertido usar Symfony para un proyecto.

Si eres nuevo en Symfony, este libro muestra el poder del framework y cómo puede mejorar tu productividad mientras desarrollas una aplicación, paso a paso.

En el caso de que ya seas un desarrollador de Symfony, deberías redescubrirlo. El framework ha evolucionado drásticamente durante los últimos años y la experiencia del desarrollador ha mejorado significativamente. Tengo la sensación de que muchos desarrolladores de Symfony todavía están "atados" a los viejos hábitos y les cuesta mucho aceptar las nuevas formas de desarrollar aplicaciones con Symfony. Puedo entender algunas de las razones. El ritmo al que ha evolucionado es asombroso. Cuando se trabaja a tiempo completo en un proyecto, los desarrolladores no tienen tiempo para seguir todo lo que sucede en la comunidad. Lo sé de primera mano, ya que ni yo mismo sería capaz de hacerlo. Todo lo contrario.

Y no se trata sólo de nuevas formas de hacer las cosas. También se trata de nuevos componentes: Cliente HTTP, Mailer, Workflow, Messenger. Cambian las reglas del juego. Deberían cambiar la forma en que piensas sobre una aplicación Symfony.

También siento la necesidad de un nuevo libro ya que la Web ha evolucionado mucho. Temas como `APIs`_ , `SPAs`_ , `containerización`_ , `Despliegue Continuo`_ , y muchos otros deben ser tratados hoy en día.

¿Sigue mereciendo tu tiempo un libro ahora que un LLM puede escribir el código por ti? Me hice esta pregunta mientras trabajaba en esta edición. Mi respuesta es un rotundo sí. Uso herramientas de IA todos los días, y ahora escriben casi todo mi código; pero no han cambiado lo que importa. Sigues necesitando entender la arquitectura de tu aplicación, decidir qué construir, revisar el código generado y detectar lo que está sutilmente mal. No puedes revisar código que no entiendes; y no puedes dirigir una herramienta hacia una arquitectura que no eres capaz de imaginar. El argumento para leer libros es el criterio, no la nostalgia.

El criterio necesita comprensión, y la comprensión es exactamente lo que este libro construye: un modelo mental completo de una aplicación Symfony, desde el primer commit hasta la producción. Con ese modelo en tu cabeza, la próxima vez que un LLM genere un controlador o un handler de Messenger para ti, sabrás si es correcto, y por qué. Incluso pondremos uno a trabajar dentro de la propia aplicación para moderar comentarios.

Tu tiempo es oro. No esperes párrafos largos, ni largas explicaciones sobre los conceptos básicos. El libro trata más sobre el viaje. Por dónde empezar. Qué código escribir. Cuándo. Cómo. Trataré de generar algún interés en temas importantes y te dejaré decidir si quieres aprender y profundizar más.

Tampoco quiero replicar la documentación existente. Su calidad es excelente. Haré referencia a la documentación de forma abundante en la sección "Ir más allá" al final de cada paso/capítulo. Considera este libro como una lista de indicadores hacia más recursos.

El libro describe la creación de una aplicación, desde cero hasta ponerla en producción. Sin embargo, no desarrollaremos todo para que esté listo para producción. El resultado no será perfecto. Tomaremos atajos. Incluso podríamos saltarnos la gestión de algunos casos límite, la validación o las pruebas. No siempre se respetarán las buenas prácticas. Pero vamos a tocar casi todos los aspectos de un proyecto moderno de Symfony.

Mientras empezaba a trabajar en este libro, lo primero que hice fue programar la aplicación final. Quedé impresionado con el resultado y la velocidad que pude mantener mientras añadía funciones, con muy poco esfuerzo. Eso es gracias a la documentación y al hecho de que Symfony sabe hacerse a un lado en tu camino. Estoy seguro de que Symfony se puede mejorar de muchas maneras (y he tomado algunas notas sobre posibles mejoras), pero la experiencia del desarrollador es mucho mejor que hace unos años. Quiero contárselo al mundo.

El libro está dividido en pasos. Cada paso se subdivide en sub-pasos. Deben ser de lectura rápida. Pero lo más importante es que te invito a programar a medida que lees. Escribe el código, pruébalo, impleméntalo, ajústalo.

Por último, pero no menos importante, no dudes en pedir ayuda si te quedas atascado. Puede que encuentres un caso límite o un error tipográfico en el código que escribiste que puede ser difícil de encontrar y corregir. Pregunta. Tenemos una comunidad maravillosa en `Slack`_ and `GitHub`_ .

¿Listo para programar? ¡Disfruta!

.. _`APIs`: https://en.wikipedia.org/wiki/Application_programming_interface
.. _`SPAs`: https://en.wikipedia.org/wiki/Single-page_application
.. _`containerización`: https://en.wikipedia.org/wiki/OS-level_virtualization
.. _`Despliegue Continuo`: https://en.wikipedia.org/wiki/Continuous_deployment
.. _`Slack`: https://symfony.com/slack
.. _`GitHub`: https://github.com/symfony/symfony/discussions
