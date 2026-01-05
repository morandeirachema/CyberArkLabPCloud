Buenos días,

sobre lo que vamos a comentar en la reunión que os he lanzado para dentro de un rato, los puntos que quiero que trabajéis sobre el laboratorio son los siguientes:

•	Levantar un AD nuestro (o bien desde 0 o reutilizar el de Jon, hablad con el para que os diga en que servidor tiene montado el DC) e integrarlo como IdP.

•	Generar una estructura mínima en el DIT de Usuarios (parecido al modelo TIER) y Grupos, tenerlo ordenado y estructurado. Esto lo comentamos luego en detalle.

•	Hacer limpieza de usuarios locales: actualizar direcciones de correo con el nuevo @, homogeneizar cuentas (habrá que revisar que licenciamiento y para cuentos usuarios nos dan) y que todos tengamos tanto el usuario local como un análogo del Dominio que demos de alta. Borrar usuarios \ cuentas de personas que ya no estén en el equipo (Yago, Olmedo, Ana etc).

•	Crear las PolicySet que sean necesarias para tener 2FA normalizado tanto por mail como por teléfono corporativo. Esto es algo mas complejo y seguramente lo tengáis que ver conmigo o con alguien del equipo que os cuente como va.

•	Configurar MobileApp.

•	Tener una infra de PSM en HA + SIA en HA (yo desplegaría 2 y 2 en las mismas 2 máquinas de cada par). El PSMP no creo que nos haga excesiva falta visto lo visto aunque si tenemos un par de máquinas Linux que nos sobren, podemos desplegar SIA sobre Linux también para tener los 2 casos de uso de este conector montados.

•	Establecer una nomenclatura de Safes + Platforms reglada y que todo el mundo asuma que hay que mantenerla cuando se hagan pruebas, pilotos etc. Esto lo tendré que decidir yo, pero os lo paso si tengo algún rato durante los días que voy a trabajar en Navidades.

•	Este ya sería algo más avanzado, pero queda muy pintón en demos y va de la mano de otros productos del portfolio: integrar la autenticación para al menos un pequeño colectivo de cuentas con OKTA. Este si que lo dejamos para la vuelta de navidades, que es bastante mas lío y tengo que recuperar el acceso Admin que me dieron en su momento.
