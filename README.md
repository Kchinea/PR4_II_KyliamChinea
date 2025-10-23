# Práctica 4 — Resumen de ejercicios

## Índice

- [Ejercicio 1](#ejercicio-1)
- [Ejercicio 2 (adaptación a humanoides)](#ejercicio-2-adaptaci%C3%B3n-a-humanoides)
- [Ejercicio 3 (notificador global y escudos)](#ejercicio-3-notificador-global-y-escudos)
- [Ejercicio 4 (teletransporte y orientación)](#ejercicio-4-teletransporte-y-orientaci%C3%B3n)
- [Ejercicio 5 (recolectar escudos - consola)](#ejercicio-5-recolectar-escudos---consola)
- [Ejercicio 6 (UI de puntuación)](#ejercicio-6-ui-de-puntuaci%C3%B3n)
- [Ejercicio 7 (recompensas cada 100 puntos)](#ejercicio-7-recompensas-cada-100-puntos)

---

> Archivo con todos los scripts: `pract4.md` (incluido en el repo). Cada sección abajo contiene el enunciado breve, cómo se resolvió y un GIF de ejemplo (placeholder) enlazado.

## Ejercicio 1

Enunciado

Crea una escena con 5 esferas: las rojas etiquetadas `Type1` y las verdes `Type2`. Añade un cubo y un cilindro. Cuando el cubo colisiona con el cilindro, las esferas `Type1` deben dirigirse hacia una de las `Type2` que selecciones manualmente, y las `Type2` se mueven hacia el cilindro. El cilindro debe enviar un mensaje y las esferas deben estar suscritas.

Solución (breve)

- Se implementa un `CylinderNotificador` que lanza un evento (Action<Rigidbody>) cuando detecta colisión con el cubo.
- Las esferas usan `SphereRespuesta` que se suscribe al evento en `Start()` y en la callback decide el comportamiento según su tag: `Type1` establece como objetivo la `Rigidbody` asignada (`targetType2Rb`); `Type2` se mueve hacia el `Rigidbody` del cilindro recibido en el evento.
- Movimiento hecho en `FixedUpdate()` modificando `rb.velocity` (no tocar transform) para respetar física.

Código

Ver scripts en `pract4.md` (buscar `CylinderNotificador`, `SphereRespuesta`, `CubeController`).

Código (clases usadas)

```csharp
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class SphereRespuesta : MonoBehaviour
{
	[Header("Movimiento (m/s)")]
	public float speed = 5f;

	[Header("Si eres Type1: asigna aquí la Rigidbody de la Type2 objetivo (en el inspector)")]
	public Rigidbody targetType2Rb; // asignar solo para Type1 (si no se asigna avisará)

	// Referencia al notificador (puedes arrastrarla en el inspector, o dejarla null y buscar en Start)
	public CylinderNotificador notificador;

	private Rigidbody rb;
	private Rigidbody moveTargetRb; // objetivo actual al que moverse (cylinderRb o targetType2Rb)

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();

		// Buscar un collider existente
		Collider col = GetComponent<Collider>();
		if (col == null)
			col = GetComponentInChildren<Collider>(true);

		// Si no hay collider, añadir uno automáticamente
		if (col == null)
		{
			SphereCollider sc = gameObject.AddComponent<SphereCollider>();
			sc.radius = 0.5f;
			sc.isTrigger = false;
			Debug.Log($"{name}: No tenía collider. Se añadió automáticamente un SphereCollider.");
		}
		rb.constraints = RigidbodyConstraints.FreezeRotation;
	}

	private void Start()
	{
		// Si no asignaste el notificador en el inspector, buscar el primero en la escena
		if (notificador == null)
		{
			notificador = FindObjectOfType<CylinderNotificador>();
			if (notificador == null)
				Debug.LogWarning($"{name}: No se encontró CylinderNotificador en la escena. Asigna uno en el inspector.");
		}

		// Suscribirse al evento (si existe)
		if (notificador != null)
		{
			notificador.OnMiEvento += MiRespuesta;
			Debug.Log($"{name}: Suscrito a OnMiEvento del notificador '{notificador.name}'");
		}
	}

	private void OnDestroy()
	{
		// Desuscribirse para evitar memory leaks / null refs
		if (notificador != null)
			notificador.OnMiEvento -= MiRespuesta;
	}

	// Callback que sigue la firma del evento: recibe el Rigidbody del cilindro que disparó el evento
	private void MiRespuesta(Rigidbody cylinderRb)
	{
		Debug.Log($"{name}: MiRespuesta invoked by cylinder '{cylinderRb.name}'");
		// Determinar comportamiento según tag
		if (CompareTag("Type1"))
		{
			if (targetType2Rb != null)
			{
				moveTargetRb = targetType2Rb;
				Debug.Log($"{name}: Soy Type1, me moveré hacia targetType2Rb '{targetType2Rb.name}'");
			}
			else
			{
				Debug.LogWarning($"{name}: Soy Type1 pero no tengo targetType2Rb asignado.");
			}
		}
		else if (CompareTag("Type2"))
		{
			// Type2 se mueve hacia el cilindro (el Rigidbody pasado en el evento)
			moveTargetRb = cylinderRb;
			Debug.Log($"{name}: Soy Type2, me moveré hacia el cilindro '{cylinderRb.name}'");
		}
	}

	// Movimiento con física: usar FixedUpdate y ajustar velocity (no tocar transform)
	private void FixedUpdate()
	{
		if (moveTargetRb == null) return;

		Vector3 dir = (moveTargetRb.position - rb.position);
		float dist = dir.magnitude;
		if (dist < 0.1f)
		{
			// Llegamos: parar
			rb.velocity = Vector3.zero;
			moveTargetRb = null;
			return;
		}

		Vector3 desiredVel = dir.normalized * speed;

		// Suavizar un poco la transición:
		rb.velocity = Vector3.Lerp(rb.velocity, desiredVel, 0.2f);
		// Si prefieres imponer directamente:
		// rb.velocity = desiredVel;
	}
}
```

```csharp
using System;
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class CylinderNotificador : MonoBehaviour
{
	// Evento público — pasa el Rigidbody del cilindro a los observadores
	public event Action<Rigidbody> OnMiEvento;

	// Control sencillo para hacer la prueba: si el cubo colisiona con el cilindro, disparar evento.
	private void OnCollisionEnter(Collision collision)
	{
		Debug.Log($"{name}: OnCollisionEnter with '{collision.collider.name}' (tag='{collision.collider.tag}')");
		if (collision.collider.CompareTag("Cube"))
		{
			Debug.Log($"{name}: Collision with cube detected - invoking OnMiEvento (if any subscribers).\nCollider: {collision.collider.name}, Tag: {collision.collider.tag}");
			// Dispara el evento, pasando el Rigidbody del propio cilindro
			Rigidbody rb = GetComponent<Rigidbody>();
			OnMiEvento?.Invoke(rb);
		}
	}

	// (totalmente opcional) método público para disparar el evento desde otros scripts
	public void DispararEvento() 
	{
		Debug.Log($"{name}: DispararEvento() called - invoking OnMiEvento (if any subscribers)");
		OnMiEvento?.Invoke(GetComponent<Rigidbody>());
	}
}
```

```csharp
using UnityEngine;

// Mueve el cubo con Rigidbody usando las teclas WASD o flechas
public class CubeController : MonoBehaviour
{
	public float speed = 5f;
	Rigidbody rb;
	void Awake()
	{
		rb = GetComponent<Rigidbody>();
	}
	void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");
		Vector3 move = new Vector3(h, 0f, v) * speed;
		// Preserve current Y velocity (gravity)
		Vector3 vel = new Vector3(move.x, rb.velocity.y, move.z);
		rb.velocity = vel;
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio1](./PR4_GIF1.gif)


---

## Ejercicio 2 (adaptación a humanoides)

Enunciado

Sustituye los objetos geométricos por humanoides. Existen humanoides de tipo 1 y tipo 2, así como escudos tipo 1 y tipo 2. Cuando el cubo colisiona con cualquier humanoide tipo 2, los del grupo 1 se acercan a un escudo seleccionado. Cuando el cubo toca un humanoide del grupo 1, los tipo 2 se dirigen hacia los escudos de grupo 2. Si un humanoide colisiona con un escudo físico, debe cambiar de color.

Solución (breve)

- Se reutiliza la idea del notificador: cada humanoide con componente `CylinderNotificador` (o su versión) detecta al cubo y dispara un evento.
- Para gestionar grupos y escudos se añaden referencias públicas (`shieldType1`, `shieldType2`) en `SphereRespuesta` (o versión adaptada a humanoides) y la callback global determina a qué shield moverse según la etiqueta del humanoide tocado.
- En `OnCollisionEnter` de las humanoides se detecta si se ha chocado con un `Rigidbody` que corresponde a un escudo y, en ese caso, se cambia el color del `Renderer` preferido (se crea una nueva instancia de material para no afectar a `sharedMaterial`).

Código

```csharp
using System;
using UnityEngine;

// Notificador global: cualquier humanoide puede avisar cuando es tocado por el cubo.
public static class HumanoidNotifier
{
	// Parámetros: tag del humanoide tocado (ej. "Type1"/"Type2"), Rigidbody del humanoide
	public static event Action<string, Rigidbody> OnHumanoidTouched;

	public static void NotifyHumanoidTouched(string humanoidTag, Rigidbody humanoidRb)
	{
		OnHumanoidTouched?.Invoke(humanoidTag, humanoidRb);
		Debug.Log($"HumanoidNotifier: NotifyHumanoidTouched invoked for tag={humanoidTag}, rb={humanoidRb?.name}");
	}
}
```

```csharp
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class SphereRespuesta : MonoBehaviour
{
	[Header("Movimiento (m/s)")]
	public float speed = 5f;

	[Header("Si eres Type1: asigna aquí la Rigidbody de la Type2 objetivo (en el inspector)")]
	public Rigidbody targetType2Rb; // asignar solo para Type1 (si no se asigna avisará)

	[Header("Escudos (asigna en inspector)")]
	public Rigidbody shieldType1; // objeto físico (rigidbody) del escudo del grupo 1
	public Rigidbody shieldType2; // objeto físico (rigidbody) del escudo del grupo 2

	// Nota: ya no usamos CylinderNotificador. Usamos HumanoidNotifier global.
	private Rigidbody rb;
	private Rigidbody moveTargetRb; 

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();

		// Ensure collider exists and is non-trigger
		Collider col = GetComponent<Collider>() ?? GetComponentInChildren<Collider>(true);
		if (col == null)
		{
			SphereCollider sc = gameObject.AddComponent<SphereCollider>();
			sc.radius = 0.5f;
			sc.isTrigger = false;
			Debug.Log($"{name}: No tenía collider. Se añadió automáticamente un SphereCollider.");
		}
		else if (col.isTrigger)
		{
			col.isTrigger = false;
			Debug.LogWarning($"{name}: Collider.isTrigger estaba activado. Lo desactivé para asegurar colisiones físicas.");
		}

		rb.constraints = RigidbodyConstraints.FreezeRotation;
	}

	private void Start()
	{
		// Suscribirse al notificador global de humanoides
		HumanoidNotifier.OnHumanoidTouched += OnGlobalHumanoidTouched;
		Debug.Log($"{name}: Suscrito a HumanoidNotifier.OnHumanoidTouched");
	}

	private void OnDestroy()
	{
		// Desuscribirse del notificador global
		HumanoidNotifier.OnHumanoidTouched -= OnGlobalHumanoidTouched;
	}

	// Callback que responde al evento global cuando cualquier humanoide es tocado por el cubo
	private void OnGlobalHumanoidTouched(string touchedTag, Rigidbody touchedRb)
	{
		Debug.Log($"{name}: OnGlobalHumanoidTouched received - touchedTag={touchedTag}, touchedRb={touchedRb?.name}");

		// Si el cubo tocó un Type2 => los Type1 se mueven hacia shieldType1
		if (touchedTag == "Type2" && CompareTag("Type1"))
		{
			if (shieldType1 != null)
			{
				moveTargetRb = shieldType1;
				Debug.Log($"{name}: Evento: Type2 tocado -> Type1 me muevo hacia shieldType1 '{shieldType1.name}'");
			}
			else
			{
				Debug.LogWarning($"{name}: Evento: Type2 tocado pero no tengo shieldType1 asignado.");
			}
		}

		// Si el cubo tocó un Type1 => los Type2 se mueven hacia shieldType2
		if (touchedTag == "Type1" && CompareTag("Type2"))
		{
			if (shieldType2 != null)
			{
				moveTargetRb = shieldType2;
				Debug.Log($"{name}: Evento: Type1 tocado -> Type2 me muevo hacia shieldType2 '{shieldType2.name}'");
			}
			else
			{
				Debug.LogWarning($"{name}: Evento: Type1 tocado pero no tengo shieldType2 asignado.");
			}
		}
	}

	// Cuando colisione con un escudo físico, cambiar color
	private void OnCollisionEnter(Collision collision)
	{
		if (collision.collider.CompareTag("Cube") || collision.collider.CompareTag("PlayerCube"))
		{
			HumanoidNotifier.NotifyHumanoidTouched(gameObject.tag, rb);
			Debug.Log($"{name}: Fui tocado por el cubo. Notifiqué al HumanoidNotifier. My tag={gameObject.tag}");
			return;
		}
		if (collision.collider.attachedRigidbody != null)
		{
			Rigidbody otherRb = collision.collider.attachedRigidbody;
			if (otherRb == shieldType1 || otherRb == shieldType2)
			{
				//problema con los prefabs tuve que cambiar el render de la malla
				Renderer rend = FindPreferredChildRenderer();
				if (rend != null)
				{
					rend.material = new Material(rend.material);
					rend.material.color = Color.red; // cambia a rojo cuando colisiona con un escudo
					Debug.Log($"{name}: Colisioné con un escudo ({otherRb.name}) y cambié de color en renderer '{rend.gameObject.name}'.");
				}
				else
				{
					Debug.LogWarning($"{name}: Colisioné con un escudo ({otherRb.name}) pero no encontré Renderer en hijos ni en el objeto para cambiar color.");
				}
			}
		}
	}

	private Renderer FindPreferredChildRenderer()
	{
		Renderer[] rends = GetComponentsInChildren<Renderer>(true);
		if (rends == null || rends.Length == 0) return null;

		for (int i = 0; i < rends.Length; i++)
		{
			if (rends[i] == null || rends[i].gameObject == null) continue;
			string n = rends[i].gameObject.name;
			if (!string.IsNullOrEmpty(n) && n.ToLower().Contains("mesh"))
				return rends[i];
		}

		return rends[0];
	}

	private void FixedUpdate()
	{
		if (moveTargetRb == null) return;

		Vector3 dir = (moveTargetRb.position - rb.position);
		float dist = dir.magnitude;
		if (dist < 0.1f)
		{
			rb.velocity = Vector3.zero;
			moveTargetRb = null;
			return;
		}

		Vector3 desiredVel = dir.normalized * speed;
		rb.velocity = Vector3.Lerp(rb.velocity, desiredVel, 0.2f);
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio2](./PR4_GIF2.gif)
---

## Ejercicio 3 (notificador global y escudos)

Enunciado

Adapta la escena anterior para usar un notificador global (`HumanoidNotifier`) que informe cuándo cualquier humanoide es tocado por el cubo. Los Type1 se mueven a `shieldType1` cuando se toca un Type2, y los Type2 se mueven a `shieldType2` cuando se toca un Type1.

Solución (breve)

- Se crea un `HumanoidNotifier` estático con evento `OnHumanoidTouched` (Action<string, Rigidbody>), y cualquier humanoide notifica con su tag y su Rigidbody.
- Las esferas/humanoides se suscriben a ese evento y, según su propio tag, se fijan un `moveTargetRb` hacia el shield asignado.
- El cambio de color al tocar un escudo sigue realizándose en `OnCollisionEnter`.

Código

```csharp
using UnityEngine;

// Mueve el cubo con Rigidbody usando las teclas WASD o flechas
public class CubeController : MonoBehaviour
{
	public float speed = 5f;
	Rigidbody rb;
	void Awake()
	{
		rb = GetComponent<Rigidbody>();
	}
	void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");
		Vector3 move = new Vector3(h, 0f, v) * speed;
		// Preserve current Y velocity (gravity)
		Vector3 vel = new Vector3(move.x, rb.linearVelocity.y, move.z);
		rb.linearVelocity = vel;
	}
}
```

```csharp
using System;
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class CylinderNotificador : MonoBehaviour
{
	// Evento público — pasa el Rigidbody del cilindro a los observadores
	public event Action<Rigidbody> OnMiEvento;

	private void Awake()
	{
		Collider col = GetComponent<Collider>();
		if (col == null)
		{
			Debug.LogWarning($"{name}: No Collider encontrado. Añade un Collider para detectar colisiones físicas.");
		}
		else if (col.isTrigger)
		{
			col.isTrigger = false;
			Debug.LogWarning($"{name}: Collider.isTrigger estaba activado. Lo desactivé para asegurar colisiones físicas.");
		}
	}

	private void OnCollisionEnter(Collision collision)
	{
		Debug.Log($"{name}: OnCollisionEnter with '{collision.collider.name}' (tag='{collision.collider.tag}')");

		if (collision.collider.CompareTag("Cube") || collision.collider.CompareTag("PlayerCube"))
		{
			Debug.Log($"{name}: Collision with cube detected - invoking OnMiEvento (if any subscribers).\nCollider: {collision.collider.name}, Tag: {collision.collider.tag}");
			Rigidbody rb = GetComponent<Rigidbody>();
			OnMiEvento?.Invoke(rb);
		}
	}

	public void DispararEvento() 
	{
		Debug.Log($"{name}: DispararEvento() called - invoking OnMiEvento (if any subscribers)");
		OnMiEvento?.Invoke(GetComponent<Rigidbody>());
	}
}
```

```csharp
// (El código de HumanoidNotifier y SphereRespuesta ya está mostrado en Ejercicio 2)
```

Ejemplo visual (placeholder):

![Ejercicio3](./PR4_GIF3.gif)

---

## Ejercicio 4 (teletransporte y orientación)

Enunciado

Cuando el cubo se aproxima a un objeto de referencia, los humanoides del grupo 1 se teletransportan a un escudo objetivo fijado de antemano. Los humanoides del grupo 2 se orientan hacia un objeto ubicado en la escena.

Solución (breve)

- Se implementa `AreaNotificador` (con un collider en modo trigger) que dispara `OnCuboEntrado` cuando el cubo entra.
- `HumanoideRespuestaRB` se suscribe al notificador: los `Type1` usan `rb.position = escudoObjetivo1.position` para teletransportarse y detener su velocidad; los `Type2` giran usando `rb.MoveRotation` para orientarse hacia `lookTargetPos`.

Código

```csharp
using UnityEngine;

// Mueve el cubo con Rigidbody usando las teclas WASD o flechas
public class CubeController : MonoBehaviour
{
	public float speed = 5f;
	Rigidbody rb;
	void Awake()
	{
		rb = GetComponent<Rigidbody>();
		rb.freezeRotation = true;
	}
	void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");
		Vector3 move = new Vector3(h, 0f, v) * speed;
		// Preserve current Y velocity (gravity)
		Vector3 vel = new Vector3(move.x, rb.linearVelocity.y, move.z);
		rb.linearVelocity = vel;
	}
}
```

```csharp
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class HumanoideRespuestaRB : MonoBehaviour
{
	public float speed = 5f; // velocidad de giro / movimiento
	public Transform escudoObjetivo1; // destino Type1
	public Transform escudoObjetivo2; // destino Type2 (mirar)

	public AreaNotificador notificador;

	private Rigidbody rb;
	private Vector3 moveTargetPos; // para Type1
	private Vector3 lookTargetPos; // para Type2
	private bool triggered = false;

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();
		rb.freezeRotation = true;
	}

	private void Start()
	{
		if (notificador == null)
			notificador = FindObjectOfType<AreaNotificador>();

		if (notificador != null)
		{
			notificador.OnCuboEntrado += OnCuboEntrado;
			Debug.Log($"{name}: Suscrito a AreaNotificador '{notificador.name}'");
		}
	}

	private void OnDestroy()
	{
		if (notificador != null)
			notificador.OnCuboEntrado -= OnCuboEntrado;
	}

	private void OnCuboEntrado()
	{
		Debug.Log($"{name}: OnCuboEntrado recibido. Mi tag={gameObject.tag}");

		if (CompareTag("Type1"))
		{
			rb.position = escudoObjetivo1.position;
			rb.velocity = Vector3.zero; // asegurar que no siga moviéndose
			triggered = false;
			Debug.Log($"{name}: Soy Type1. Me teletransporté a escudoObjetivo1 at {escudoObjetivo1.position}");
		}
		else if (CompareTag("Type2"))
		{
			Vector3 dir = lookTargetPos - rb.position;
			dir.y = 0;
			if (dir != Vector3.zero)
			{
				Quaternion targetRot = Quaternion.LookRotation(dir);
				rb.MoveRotation(Quaternion.Slerp(rb.rotation, targetRot, 0.2f));
			}
		}
	}

	private void FixedUpdate()
	{
		if (!triggered) return;

		if (CompareTag("Type1"))
		{
			Vector3 dir = moveTargetPos - rb.position;
			dir.y = 0; 
			if (dir.magnitude < 0.1f)
			{
				rb.velocity = Vector3.zero;
				triggered = false;
				return;
			}
			rb.velocity = dir.normalized * speed;
		}
		else if (CompareTag("Type2"))
		{
			Vector3 dir = lookTargetPos - rb.position;
			dir.y = 0;
			if (dir != Vector3.zero)
			{
				Quaternion targetRot = Quaternion.LookRotation(dir);
				rb.MoveRotation(Quaternion.Slerp(rb.rotation, targetRot, 0.2f));
			}
		}
	}
}
```

```csharp
using UnityEngine;
using System;

[RequireComponent(typeof(Collider))]
public class AreaNotificador : MonoBehaviour
{
	// Evento que avisa que el cubo ha entrado
	public event Action OnCuboEntrado;

	private void Awake()
	{
		Collider col = GetComponent<Collider>();
		if (col == null)
			Debug.LogWarning($"{name}: No hay collider en este objeto.");
		else if (!col.isTrigger)
			col.isTrigger = true;
	}

	private void OnTriggerEnter(Collider other)
	{
		if (other.CompareTag("PlayerCube") || other.CompareTag("Cube"))
		{
			Debug.Log($"{name}: Cubo ha entrado! (collider: {other.name}, tag: {other.tag})");
			OnCuboEntrado?.Invoke();
			return;
		}

		Debug.Log($"{name}: OnTriggerEnter ignored object '{other.name}' with tag '{other.tag}'");
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio4](./PR4_GIF4.gif)

---

## Ejercicio 5 (recolectar escudos - consola)

Enunciado

Implementar la mecánica de recolectar escudos en la escena y actualizar la puntuación del jugador: los escudos tipo 1 suman 5 puntos y los tipo 2 suman 10. Mostrar la puntuación en la consola.

Solución (breve)

- `CubeCollector` (componente del cubo) implementa `OnTriggerEnter(Collider other)` y comprueba la etiqueta del objeto (`shieldType` o `shieldType1`) para sumar la puntuación correspondiente.
- Se hace `Destroy(other.gameObject)` para simular la recolección y se imprime la puntuación con `Debug.Log`.

Código

```csharp
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class CubeCollector : MonoBehaviour
{
	public float speed = 5f;
	private Rigidbody rb;

	private int score = 0; // puntuación total

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();
		rb.freezeRotation = true;
	}

	private void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");

		Vector3 move = new Vector3(h, 0f, v) * speed;
		Vector3 vel = new Vector3(move.x, rb.velocity.y, move.z);
		rb.velocity = vel;
	}

	// OnTriggerEnter para recolectar escudos
	private void OnTriggerEnter(Collider other)
	{
		if (other.CompareTag("shieldType"))
		{
			score += 5;
			Debug.Log($"Recolectaste {other.name} (Tipo1) → Puntuación: {score}");
			Destroy(other.gameObject); // opcional, si quieres que desaparezca
		}
		else if (other.CompareTag("shieldType1"))
		{
			score += 10;
			Debug.Log($"Recolectaste {other.name} (Tipo2) → Puntuación: {score}");
			Destroy(other.gameObject); // opcional
		}
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio5](./PR4_GIF5.gif)

---

## Ejercicio 6 (UI de puntuación)

Enunciado

Partiendo del script anterior, crea una interfaz que muestre la puntuación que va obteniendo el cubo.

Solución (breve)

- Se añade un `TMP_Text` (TextMeshPro) público en `CubeCollector` llamado `scoreText`.
- Cada vez que se actualiza la puntuación se llama a `UpdateScoreUI()` que escribe en `scoreText.text`.

Código

```csharp
using UnityEngine;
using UnityEngine.UI; // necesario para UI Text
using TMPro; // si usas TextMeshPro, descomenta

[RequireComponent(typeof(Rigidbody))]
public class CubeCollector : MonoBehaviour
{
	public float speed = 5f;
	private Rigidbody rb;
		[Header("UI")]
	public TMP_Text scoreText;

	private int score = 0; // puntuación total

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();
		rb.freezeRotation = true;
		UpdateScoreUI();
	}

	private void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");

		Vector3 move = new Vector3(h, 0f, v) * speed;
		Vector3 vel = new Vector3(move.x, rb.velocity.y, move.z);
		rb.velocity = vel;
	}

	// OnTriggerEnter para recolectar escudos
	private void OnTriggerEnter(Collider other)
	{
		if (other.CompareTag("shieldType"))
		{
			score += 5;
			Debug.Log($"Recolectaste {other.name} (Tipo1) → Puntuación: {score}");
			Destroy(other.gameObject); // opcional, si quieres que desaparezca
			UpdateScoreUI();
		}
		else if (other.CompareTag("shieldType1"))
		{
			score += 10;
			Debug.Log($"Recolectaste {other.name} (Tipo2) → Puntuación: {score}");
			Destroy(other.gameObject); // opcional
			UpdateScoreUI();
		}
	}
	private void UpdateScoreUI()
	{
		if (scoreText != null)
			scoreText.text = "Puntuación: " + score;
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio6](./PR4_GIF6.gif)

---

## Ejercicio 7 (recompensas cada 100 puntos)

Enunciado

Cada 100 puntos el jugador obtiene una recompensa que se muestra en la UI.

Solución (breve)

- `CubeCollector` guarda `lastRewardScore`, y cuando `score / 100 > lastRewardScore / 100` se dispara `ShowReward()` que escribe un mensaje en `rewardText` y lo borra tras 2s con una coroutine.

Código

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections;
using TMPro;

[RequireComponent(typeof(Rigidbody))]
public class CubeCollector : MonoBehaviour
{
	public float speed = 5f;
	private Rigidbody rb;

	private int score = 0; // puntuación total

	[Header("UI")]
	public TextMeshProUGUI scoreText;   // Text para puntuación
	public TextMeshProUGUI rewardText;  // Text para mostrar la recompensa

	private int lastRewardScore = 0; // para que no se repita la misma recompensa

	private void Awake()
	{
		rb = GetComponent<Rigidbody>();
		UpdateScoreUI();
		rb.freezeRotation = true;
	}

	private void FixedUpdate()
	{
		float h = Input.GetAxis("Horizontal");
		float v = Input.GetAxis("Vertical");

		Vector3 move = new Vector3(h, 0f, v) * speed;
		Vector3 vel = new Vector3(move.x, rb.velocity.y, move.z);
		rb.velocity = vel;
	}

	private void OnTriggerEnter(Collider other)
	{
		if (other.CompareTag("shieldType"))
		{
			score += 5;
			Destroy(other.gameObject);
			Debug.Log($"Recolectaste {other.name} (Type1) → Puntuación: {score}");
			UpdateScoreUI();
			CheckReward();
		}
		else if (other.CompareTag("shieldType1"))
		{
			score += 10;
			Destroy(other.gameObject);
			Debug.Log($"Recolectaste {other.name} (Type2) → Puntuación: {score}");
			UpdateScoreUI();
			CheckReward();
		}
	}

	private void UpdateScoreUI()
	{
		if (scoreText != null)
			scoreText.text = "Puntuación: " + score;
	}

	private void CheckReward()
	{
		// Cada 100 puntos y solo una vez por múltiplo
		if (score / 100 > lastRewardScore / 100)
		{
			lastRewardScore = score;
			ShowReward("¡Has obtenido una recompensa!");
		}
	}

	private void ShowReward(string message)
	{
		if (rewardText != null)
		{
			rewardText.text = message;
			// Ocultar después de 2 segundos
			StartCoroutine(HideRewardAfterDelay(2f));
		}
	}

	private IEnumerator HideRewardAfterDelay(float delay)
	{
		yield return new WaitForSeconds(delay);
		if (rewardText != null)
			rewardText.text = "";
	}
}
```

Ejemplo visual (placeholder):

![Ejercicio7](./PR4_GIF7.gif)
---
