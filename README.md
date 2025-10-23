# PR4_II_KyliamChinea

## exercise1:

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
        // Log the collision details for debugging
        Debug.Log($"{name}: OnCollisionEnter with '{collision.collider.name}' (tag='{collision.collider.tag}')");

        // Support cube tagged either 'Cube' or 'PlayerCube' (some projects use different tags)
        if (collision.collider.CompareTag("Cube"))
        {
            Debug.Log($"{name}: Collision with cube detected - invoking OnMiEvento (if any subscribers).\nCollider: {collision.collider.name}, Tag: {collision.collider.tag}");
            // Dispara el evento, pasando el Rigidbody del propio cilindro
            Rigidbody rb = GetComponent<Rigidbody>();
            OnMiEvento?.Invoke(rb);
        }
    }

    // (Opcional) método público para disparar el evento desde otros scripts
    public void DispararEvento() 
    {
        Debug.Log($"{name}: DispararEvento() called - invoking OnMiEvento (if any subscribers)");
        OnMiEvento?.Invoke(GetComponent<Rigidbody>());
    }
}

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




## exercise3:

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

using System;
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class CylinderNotificador : MonoBehaviour
{
    // Evento público — pasa el Rigidbody del cilindro a los observadores
    public event Action<Rigidbody> OnMiEvento;

    // Ensure we have a non-trigger collider so OnCollisionEnter fires
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

    // Control sencillo para hacer la prueba: si el cubo colisiona con este humanoide, disparar evento.
    private void OnCollisionEnter(Collision collision)
    {
        // Log the collision details for debugging
        Debug.Log($"{name}: OnCollisionEnter with '{collision.collider.name}' (tag='{collision.collider.tag}')");

        // Si colisiona con el cubo, notificar a los observadores
        if (collision.collider.CompareTag("Cube") || collision.collider.CompareTag("PlayerCube"))
        {
            Debug.Log($"{name}: Collision with cube detected - invoking OnMiEvento (if any subscribers).\nCollider: {collision.collider.name}, Tag: {collision.collider.tag}");
            Rigidbody rb = GetComponent<Rigidbody>();
            OnMiEvento?.Invoke(rb);
        }
    }

    // (Opcional) método público para disparar el evento desde otros scripts
    public void DispararEvento() 
    {
        Debug.Log($"{name}: DispararEvento() called - invoking OnMiEvento (if any subscribers)");
        OnMiEvento?.Invoke(GetComponent<Rigidbody>());
    }
}

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
    private Rigidbody moveTargetRb; // objetivo actual al que moverse (shield o humanoide)

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

    // Callback que sigue la firma del evento: recibe el Rigidbody del cilindro que disparó el evento
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
        // If touched by the cube: notify the global notifier
        if (collision.collider.CompareTag("Cube") || collision.collider.CompareTag("PlayerCube"))
        {
            HumanoidNotifier.NotifyHumanoidTouched(gameObject.tag, rb);
            Debug.Log($"{name}: Fui tocado por el cubo. Notifiqué al HumanoidNotifier. My tag={gameObject.tag}");
            return;
        }

        // Otherwise, if collided with assigned shields, change color on the appropriate child renderer
        if (collision.collider.attachedRigidbody != null)
        {
            Rigidbody otherRb = collision.collider.attachedRigidbody;
            if (otherRb == shieldType1 || otherRb == shieldType2)
            {
                // Find renderer: prefer child whose name contains 'mesh' (case-insensitive), else first renderer found
                Renderer rend = FindPreferredChildRenderer();
                if (rend != null)
                {
                    // Instantiate material so we don't modify sharedMaterial across instances
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

    // Busca un Renderer en este GameObject o sus hijos.
    // Prioriza los GameObjects cuyo nombre contiene la subcadena 'mesh' (case-insensitive).
    private Renderer FindPreferredChildRenderer()
    {
        // Busca todos los renderers en hijos (incluye el propio GameObject)
        Renderer[] rends = GetComponentsInChildren<Renderer>(true);
        if (rends == null || rends.Length == 0) return null;

        // Primero intenta encontrar uno cuyo nombre contenga 'mesh' (insensible a mayúsculas)
        for (int i = 0; i < rends.Length; i++)
        {
            if (rends[i] == null || rends[i].gameObject == null) continue;
            string n = rends[i].gameObject.name;
            if (!string.IsNullOrEmpty(n) && n.ToLower().Contains("mesh"))
                return rends[i];
        }

        // Si no se encuentra ninguno con 'mesh', devolver el primero útil
        return rends[0];
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

## Exercise 4


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

        // Use English tags 'Type1'/'Type2' (project uses these). If your tags are different, adapt here.
        if (CompareTag("Type1"))
        {
            // Teletransportar al escudo de inmediato
            rb.position = escudoObjetivo1.position;
            rb.velocity = Vector3.zero; // asegurar que no siga moviéndose
            triggered = false; // ya no necesitamos FixedUpdate
            Debug.Log($"{name}: Soy Type1. Me teletransporté a escudoObjetivo1 at {escudoObjetivo1.position}");
        }
        else if (CompareTag("Type2"))
        {
        // Rotar hacia objetivo
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
            // Mover Rigidbody hacia target
            Vector3 dir = moveTargetPos - rb.position;
            dir.y = 0; // mantener altura
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
            // Rotar hacia objetivo
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
        // Accept either 'PlayerCube' or 'Cube' tags (some projects use one or the other)
        if (other.CompareTag("PlayerCube") || other.CompareTag("Cube"))
        {
            Debug.Log($"{name}: Cubo ha entrado! (collider: {other.name}, tag: {other.tag})");
            OnCuboEntrado?.Invoke();
            return;
        }

        // Helpful debug when nothing happens
        Debug.Log($"{name}: OnTriggerEnter ignored object '{other.name}' with tag '{other.tag}'");
    }
}





## Exercise 5

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



## Exercise 6
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



## Exercise 7


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

