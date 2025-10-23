# PR4_II_KyliamChinea

##exercise1:

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





