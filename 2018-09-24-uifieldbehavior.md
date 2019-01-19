---
title: UIFieldBehavior
author: Jordan Morgan
translator: Leo Picado
category: Cocoa
excerpt: >
  Como parte del rediseño de iOS en su sétima entrega,
  se dejó de lado el diseño esqueumorfista.
  Tomando su lugar emergió un nuevo pardigma,
  en el que a los controles de UI se les permitió
  tener la _sensación_ de objetos físicos y no tan solo
  verse como tales.
status:
  swift: 4.2
---

La marca de la década para iOS ha llegado y se ha ido.
El desarrollo de iOS fue inicialmente un arte en ascenso,
pero ahora se siente usado y desgastado.

Sin embargo, cuando salgo de mi zona de confort de
tablas, etiquetas, botones y otros parecidos,
comúnmente encuentro partes de Cocoa Touch que
pasé por alto u olvidé por completo.
Cuando sucede, es como tomar un viejo libro del estante;
la anticipación de lo que esconden sus páginas
inevitablemente te inunda.

`UIFieldBehavior` ha sido el último tomo empolvado
por descubrir dentro de UIKit.
Ciertamente usar o hablar de una API concebida para
modelar física compleja para elementos de UI no es
muy común. Pero cuando se ocupa, se _ocupa_, y no hay
alternativa. Y como los abastecedores de lo comúnmente
olvidado o poco usado es un tema excelente para el artículo
de esta semana de NSHipster.

---

Como parte del rediseño de iOS en su sétima entrega,
se dejó de lado el diseño esqueumorfista.
Tomando su lugar emergió un nuevo pardigma,
en el que a los controles de UI se les permitió
tener la _sensación_ de objetos físicos y no tan solo
verse como tales.
Se necesitarían nuevas API para iniciar esta
nueva era de diseño de UI, y así conocimos 
[UIKit Dynamics](https://developer.apple.com/documentation/uikit/animation_and_haptics/uikit_dynamics).

Hay muchos ejemplos del alcance de UIKit Dynamics
através de todo el SO:
la  pantalla de bloqueo que brinca,
las fotos deslizables,
las burbujeantes burbujas de mensajes
entre otras tantas interacciones, que usan alguna
variación de la API, de las que existen bastante.

- `UIAttachmentBehavior`:
  Crea una relación entre dos elementos,
  o un artículo y un punto ancla.
- `UICollisionBehavior`:
  Hace que uno o más objetos reboten entre sí
  en vez de traslaparse sin interacción.
- `UIFieldBehavior`:
  Permite que un elemento ó área participe en la física de campo.
- `UIGravityBehavior`:
  Aplica una fuerza gravitacional, o jalón.
- `UIPushBehavior`:
  Crea una fuerza instantánea o continua.
- `UISnapBehavior`:
  Produce un movimiento que reduce la intensidad con el tiempo.

Para este artículo daremos un vistazo a `UIFieldBehavior`,
que nuestros buenos amigos de Cupertino usaron para construir
la funcionalidad PiP de las llamadas de FaceTime.

{% asset facetime-picture-in-picture.png alt="FaceTime" title="Imagen: Apple Inc. Todos los derechos reservados." %}

## Entendiendo como funcionan los comportamientos de campo

Apple menciona que UIFieldBehavior aplica "física de campo",
¿Pero qué significa eso exactamente?
Afortunadamente, es más fácil de entender de lo que parece.

Hay muchos ejemplos de física de campo en el mundo real,
ya sea el tirón de un imán,
el rebote de un resorte,
La fuerza de la gravedad jalándote hacia la tierra.
Usando `UIFieldBehavior`, podemos designar áreas de
nuestras vistas para aplicar efectos de física
cuando interactuen entre en ellos.

El accesible diseño de la API nos permite realizar
efectos complejos de física, sin mucho más que una
llamada un método fábrica:

```swift
let drag = UIFieldBehavior.dragField()
```

```objc
UIFieldBehavior *drag = [UIFieldBehavior dragField];
```

Una vez que tenemos un campo de fuerza a nuestra
disposición es cuestión de ubicarlo en la pantalla y
definir el área sobre el cual tendrá influencia.

```swift
drag.position = view.center
drag.region = UIRegion(size: bounds.size)
```

```objc
drag.position = self.view.center;
drag.region = [[UIRegion alloc] initWithSize:self.view.bounds.size];
```

En caso de ocupar mayor control sobre el
comportamiento de un campo, se pueden
configurar sus propiedades `strength` y
`falloff` y cualquier otra propiedad adicional
relacionado a ese tipo de campo.

---

Todos los comportamientos de UIKit Dynamics
ocupan ser configurados antes de verlos en
acción y`UIFieldBehavior` no es la excepción.
El flujo se ve así:

- Crear una instancia de un `UIDynamicAnimator` que
  provea el contexto para cualquier animación que
  afecte sus ítems dinámicos.
- Inicializar los comportamientos a usar.
- Agregar las vistas que se estarán relacionadas a cada comportamiento.
- Agregar esos comportamientos al *animator* dinámico en el primer paso.

```swift
lazy var animator:UIDynamicAnimator = {
    return UIDynamicAnimator(referenceView: view)
}()

let drag = UIFieldBehavior.dragField()

// viewDidLoad:
drag.addItem(anotherView)
animator.addBehavior(drag)
```

```objc
@property (strong, nonatomic, nonnull) UIDynamicAnimator *animator;
@property (strong, nonatomic, nonnull) UIFieldBehavior *drag;

// viewDidLoad:
self.animator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];
self.drag = [UIFieldBehavior dragField];

[self.drag addItem:self.anotherView];
[self.animator addBehavior:self.drag];
```

{% warning do %}

Recuerda de mantener una referencia fuerte al objeto `UIKitDynamicAnimator`,
normalmente no es necesario hacer esto para conductas
porque el *animator* toma posesión del comportamiento una vez que se agrega.

{% endwarning %}

El ejemplo clásico de `UIFieldBehavior` lo vemos en FaceTime
donde es usado en la pequeña vista rectangular que presenta
la cámara frontal para adherirlo a todas las esquinas de los límites
de la vista.

## Cara a cara con campos elásticos

Durante una llamada de FaceTime se puede sacudir
la foto-en-foto hacia una de las esquinas de la pantalla,
la pregunta es ¿cómo logramos que se mueva tan
fluidamente pero que también se adhiera? 

Una opción er verificar el estado final
de un reconocedor de gestos, calcular hacia cual esquina
se dirige y crear la animación.
El problema con ese enfoque es que perdemos la
"salsa secreta" que Apple laboriosamente aplica a estas
pequeñas interacciones, como la interpolación y la reducción
de la fuerza aplicada a la imagen cuando se posiciona
en una esquina.

Éste es el ejemplo clásico para usar el campo elástico
provisto por `UIFieldBehavior`.  Si pensamos en cómo
funciona un resorte en la vida real, ejerce una fuerza
lineal igual a la cantidad de tensión que se le aplica. Por lo tanto,
si empujamos hacia abajo en un resorte en espiral
esperamos que vuelva a su lugar una vez que lo soltamos.

Esta es también la razón por la cual los campos elásticos
pueden ayudarnos a contener elementos de nuestra interfaz.
Si pensamos en un ring de boxeo y de como sus cuerdas elásticas
mantienen a los boxeadores dentro del ring, sin embargo, con
resortes, la cuerda se originaría desde el centro del ring y
tiraría hacia cada borde. 

Un campo elástico funciona de una manera parecida, imaginemos
por un momento, que los límites de nuestra vista estuvieran
dividos en cuatro rectángulos y tuvieramos estos resortes tirando
hacia los límites de cada uno. Los resortes serían "empujados" hacia
abajo, desde el centro del rectángulo, hacia el borde de su esquina.
Cuando la imagen llega a alguna de las esquinas, el resorte
se "suelta" y nos da el pequeño empujón que buscamos.

{% info do %}

El campo elástico se crea usando la
[Ley de elasticidad de Hooke](https://phys.org/news/2015-02-law.html)
para calcular la cantidad de fuerza que debe aplicarse
a los objetos dentro del campo.

{% endinfo %}

Para encargarnos del efecto de adhesión de la imagen
a cada esquina, podemos hacer algo como esto:

```swift
let scale = CGAffineTransform(scaleX: 0.5, y: 0.5)

for vertical in [\UIEdgeInsets.left,
                 \UIEdgeInsets.right]
{
    for horizontal in [\UIEdgeInsets.top,
                       \UIEdgeInsets.bottom]
    {
        let springField = UIFieldBehavior.springField()
        springField.position =
            CGPoint(x: layoutMargins[keyPath: horizontal],
                    y: layoutMargins[keyPath: vertical])
        springField.region =
            UIRegion(size: view.bounds.size.applying(scale))

        animator.addBehavior(springField)
        springField.addItem(facetimeAvatar)
    }
}
```

```objc
UIFieldBehavior *topLeftCornerField = [UIFieldBehavior springField];

// Esquina superior izquierda
topLeftCornerField.position = CGPointMake(self.layoutMargins.left, self.layoutMargins.top);
topLeftCornerField.region = [[UIRegion alloc] initWithSize:CGSizeMake(self.bounds.size.width/2, self.bounds.size.height/2)];

[self.animator addBehavior:topLeftCornerField];
[self.topLeftCornerField addItem:self.facetimeAvatar];

// Y se sigue con el resto de las esquinas
```

## Resolviendo problemas de física

No siempre es fácil conceptualizar las interacciones de en
campos de fuerza invisibles y es por esto que Apple anticipó
esta necesidad y provee, de una manera u otra, una forma
de depurar problemas de física.

Escondido dentro de `UIDynamicAnimator` existe una
propiedad Booleana llamada `debugEnabled` que al
cambiar su valor a `true` dibuja líneas rojas en la interfaz
para visualizar los efectos de los campos y su influencia.
Definitivamente una gran ayuda para poder entender como
interactúan entre ellos.

Esta API no está expuesta públicamente, pero se puede
desbloquar usando key-value coding:

```objc
@import UIKit;

#if DEBUG

@interface UIDynamicAnimator (Debugging)
@property (nonatomic, getter=isDebugEnabled) BOOL debugEnabled;
@end

#endif
```

ó

```swift
animator.setValue(true, forKey: "debugEnabled")
```

```objc
[self.animator setValue:@1 forKey:@"debugEnabled"];
```

Aunque crear la categoría requiere un poco más de trabajo,
es la mejor opción, dada la resbalosa naturaleza de key-value coding
nuestro código se puede romper en cualquier momento y el
precio de nuestra propia conveniencia es típicamente caro.

Con la depuración activada, pareciera como si cada
esquina tuviera un efecto de resorte asociado.
Sin embargo al correr nuestra aplicación se revela que
no es suficiente para completar el efecto que buscamos.

{% asset uidynamicanimator-debug.jpg %}

## Añadiendo comportamientos

Repasemos nuestra situación actual, para mejorar nuestro
entiendimiento de campos físicos. En este momento
tenemos algunos problemas:

1. La imagen puede salir volando fuera de la pantalla,
   ya que solamente nuestros campos elásticos la detiene
2. Tiene una tendencia a rotar en círculos
3. Es un poco lenta

UIKit Dynamics simula la física, talvez de una forma muy realista.

Afortundamente podemos mitigar todos estos indeseables
efectos secundarios. Realmente son arreglos triviales, el valor
verdadero es el saber el *porqué* los ocupamos es lo importante.

El primer arreglo es bastante mundano, se soluciona con el
comportamiento más fácilmente entendido dentro de UIKit Dynamics:
las colisiones. Para mejorar la reacción de nuestra imagen
cuando interactúa con un campo elástico ocupamos definir
sus propiedades físicas de una manera más explícita.
Idóneamente se comportaría como lo haría en la vida real,
siendo sujeto a las fuerzas de gravedad y fricción para
disminuir su movimiento.

En estos casos ocupamos de la ayuda de `UIDynamicItemBehavior`, nos
permite añadir propiedades físicas, a lo que de otra forma serían, vistas
abstractas que interactúan con un motor físico. Aunque UIKit nos provee
de valores predeterminados para cada una de las propiedades que
interactúan con el motor físico, no representan nuestras necesidades
actuales y UIKit Dynamics casi siempre se clasifica como un "caso particular".

Resulta fácil ver como la falta de una API como este sería
muy problemático. Resultaría muy dificil modelar un empujón, un tirón o
velocidad sin tener como especificar la densidad o masa del objeto.

```swift
let avatarPhysicalProperties= UIDynamicItemBehavior(items: [facetimeAvatar])
avatarPhysicalProperties.allowsRotation = false
avatarPhysicalProperties.resistance = 8
avatarPhysicalProperties.density = 0.02
```

```objc
UIDynamicItemBehavior *avatarPhysicalProperties = [[UIDynamicItemBehavior alloc] initWithItems:@[self.facetimeAvatar]];
avatarPhysicalProperties.allowsRotation = NO;
avatarPhysicalProperties.resistance = 8;
avatarPhysicalProperties.density = 0.02;
```

El comportamiento de la imagen ahora se asemeja más
a la realidad, se desacelera un poco después de ser empujado
por un campo elástico. La opciones de personalización de `UIDynamicItemBehavior`
son impresionantes, incluyen soporte para elasticidad, embestidas 
y anclajes, todo lo que se ocupa para seguir afinando la interacción
hasta que se sienta correcto.

Como si fuera poco, tiene soporte para agregar velocidad linear o angular
a un objeto, con esto vamos cerrando el capítulo con `UIDynamicItemBehavior`,
vamos a usar estas propiedades para darle un pequeño empujón
al final del reconocedor de gestos, para enviarlo hacia la esquina más cercana
y de esta manera, dejando que cada campo elástico finalice la labor:

```swift
// Inside a switch for a gesture recognizer...
case .canceled, .ended:
let velocity = panGesture.velocity(in: view)
facetimeAvatarBehavior.addLinearVelocity(velocity, for: facetimeAvatar)
```

```objc
// Inside a switch for a gesture recognizer...
case UIGestureRecognizerStateCancelled:
case UIGestureRecognizerStateEnded:
{
CGPoint velocity = [panGesture velocityInView:self.view];
[facetimeAvatarBehavior addLinearVelocity:velocity forItem:self.facetimeAvatar];
break;
}
```

Ya casi acabamos con nuestra falsa interfaz de FaceTime.

Para completar la experiencia, ocupamos considerar lo que
nuestra imagen de FaceTime debe hacer cuando llegue a
las esquinas de la vista del `animator`. Queremos que se quede
contenido dentro de los límites, pero actualmente nada lo
detiene para que se salga de la pantalla. Afortunadamente
UIKit Dynamics nos ofrece un comportamiento para estas
ocasiones, se llama `UICollisionBehavior`.

Gracias a su consitente API, seguiremos el mismo patrón que
hemos usado para cualquier comportamiento de UIKit Dynamics
para la crear la colisión:

```swift
let parentViewBoundsCollision = UICollisionBehavior(items: [facetimeAvatar])
parentViewBoundsCollision.translatesReferenceBoundsIntoBoundary = true
```

```objc
UICollisionBehavior *parentViewBoundsCollision = [[UICollisionBehavior alloc] initWithItems:@[self.facetimeAvatar]];
parentViewBoundsCollision.translatesReferenceBoundsIntoBoundary = YES;
```

Es importante ver el efecto de `translatesReferenceBoundsIntoBoundary`,
cuando el valor está en `true` toma los límites de nuestro `animator` como
los límites para la colisión. Recordemos que este fue el primer paso
cuando armamos nuestra pila de comportamientos dinámicos:

```swift
lazy var animator:UIDynamicAnimator = {
    return UIDynamicAnimator(referenceView: view)
}()
```

```objc
self.animator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];
```

Cuando anidamos varios comportamientos para que trabajen
como uno, podemos disfrutar de nuestro trabajo:

<video preload="none" src="{% asset uifieldbehavior-demo.mp4 @path %}" poster="{% asset uifieldbehavior-demo.png @path %}" width="320" controls></video>

<br/>

Si se quiere dejar de lado el comportamiento de esquinas "pegajosas"
de FaceTime, ésta es una buena posición, ya que `UIFieldBehavior` tiene
muchos más campos que solo el elástico. Un posible experimento es
remplazar el campo elástico con uno magnético o hacer que la imagen
gire constantemente alrededor de un punto dado.

---

iOS ya dejó de lado el diseño esqueumorfista y la experiencia
de usuario ha avanzado grandemente como resultado de este
cambio. Ya no ocupamos de [fieltro verde]({% asset game-center-felt.jpg @path %}) para indicarle a nuestros
usuarios que el Game Center representa juegos y como manejarlos.

UIKit Dynamics es una muestra de este cambio, ya los componentes de
UI no son meramente ilustraciones que asemejan elementos
de la vida real, sino que se comportan como lo hacen en la vida real.

Dejar de lado esta capa através de todo el SO abrió la puerta
para que UIKit Dynamics conectara nuestra expectativas sobre
cómo los elementos deben reaccionar a nuestras acciones. Estas
conexiones pueden parecer inconsecuentes al inicio, pero si no
estuvieran ahí, sería fácil darse cuenta que algo falta.

UIKit Dynamics nos ofrece muchos sabores en cuanto a comportamientos
físicos y sus campos son talvez unos de los más versátiles e
interesantes. La próxima vez que vea una oportunidad de crear una
conexión en su aplicación, `UIFieldBehavior` puede darle el empujón
inicial.