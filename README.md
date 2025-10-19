# Practica-4-II

Autor: Eric Bermúdez Hernández

Email: alu0101517476@ull.edu.es

----

**Descripción del trabajo realizado**

### **Ejercicio 1**

Para resolver el ejercicio, lo que hice fue:

- Crear 3 esferas rojas (Serán de tipo 1)

- Crear 2 esferas verdes (Serán de tipo 2)

- Crear un cubo, que será el que colisiona y ocasione que se dispare el evento

- Crear un cilindro, que será el que envie el mensaje

Después de crear todos los elementos, he configurado las etiquetas y las capas de los objetos. He creado los siguientes tags:

- Cube, Tipo1, Tipo2

He asignado las etiquetas de la siguiente manera:

- Cubo → tag `Cube`

- A cada esfera roja → tag `Tipo 1`

- A cada esfera verde → tag `Tipo 2`

A continuación, he configurado los colliders y los Rigidbody de la siguiente manera:

- Cilindro: Marqué su collider como isTrigger

- Cubo: Le añadí su Rigidbody

Después, cree el siguiente Script:

Código Script
```C#
using System;
using UnityEngine;

public class MechanicController : MonoBehaviour
{
    public static event Action<Transform> OnCylinderSignal;

    public enum Role { Cylinder, SphereType1, SphereType2 }
    public Role role = Role.Cylinder;

    public float speed = 3.5f;
    public Transform fixedType2Target; // usado solo por SphereType1

    private Transform _currentTarget;

    void OnEnable()
    {
        if (role == Role.SphereType1 || role == Role.SphereType2)
            OnCylinderSignal += HandleCylinderSignal;
    }

    void OnDisable()
    {
        if (role == Role.SphereType1 || role == Role.SphereType2)
            OnCylinderSignal -= HandleCylinderSignal;
    }

    void OnTriggerEnter(Collider other)
    {
        if (role == Role.Cylinder && other.CompareTag("Cube"))
            OnCylinderSignal?.Invoke(transform);
    }

    void Update()
    {
        if (_currentTarget == null) return;

        Vector3 toTarget = _currentTarget.position - transform.position;
        float dist = toTarget.magnitude;
        if (dist < 0.2f) { _currentTarget = null; return; }

        transform.position += toTarget.normalized * (speed * Time.deltaTime);
    }

    void HandleCylinderSignal(Transform cylinder)
    {
        if (role == Role.SphereType1)
        {
            if (fixedType2Target != null) _currentTarget = fixedType2Target;
        }
        else if (role == Role.SphereType2)
        {
            _currentTarget = cylinder;
        }
    }
}

```

Este Script lo añadí al cilindro, a cada esfera roja y a cada esfera verde. Gracias a este Script, a cada objeto se le añade en el inspector una opción llamada `Role` la cual configuré de la siguiente forma:

- Cilindro → Role = Cylinder

- A cada esfera roja → Role = SphereType1

- A cada esfera verde → Role = SphereType2

En las esferas de Tipo 1, arrastré en el campo `Fixed Type 2 Target` una de las esferas verdes, la cual será la esfera verde que perseguirá la roja. 

En el siguiente GIF, se muestra como funciona el Script:

![Ejercicio 1](Img/EVENTOS%201.gif)

### **Ejercicio 2**

Para este ejercicio, hice lo mismo que el ejercicio 1, pero sustituyendo tanto las bolas rojas como verdes por modelos humanoides distintos. El resultado es el siguiente que se muestra en el GIF:

![Ejercicio 3](Img/EVENTOS%202.gif)

### **Ejercicio 3**

En este ejercicio se creó una escena con humanoides de tipo 1 y tipo 2, y escudos de tipo 1 y tipo 2. El cubo actúa como activador: cuando toca un humanoide, envía un mensaje que hace que los humanoides tipo 1 se muevan hacia un escudo específico.

Si el cubo toca un humanoide tipo 2, los humanoides tipo 1 van hacia el escudo tipo 1.

Si el cubo toca un humanoide tipo 1, los humanoides tipo 1 van hacia el escudo tipo 2 más cercano.

Cuando un humanoide llega a un escudo y colisiona con él, cambia de color.

Para conseguir esto se usaron scripts que controlan los eventos, el movimiento de los humanoides y la detección de colisiones.

Los Scripts que se crearon para este ejercicio son los siguientes:

1. GameSignals.cs

```C#
using System;

public enum HumanoidType { Type1, Type2 }
public enum ShieldType { Type1, Type2 }

public static class GameSignals
{
    // Notifica: (tipo de humanoide golpeado por el cubo, transform del cubo)
    public static event Action<HumanoidType, UnityEngine.Transform> OnCubeHitsHumanoid;

    public static void EmitCubeHitsHumanoid(HumanoidType hitType, UnityEngine.Transform cube)
        => OnCubeHitsHumanoid?.Invoke(hitType, cube);
}

```

2. CubeInteractor.cs

```C#
using UnityEngine;

[RequireComponent(typeof(Collider))]
[RequireComponent(typeof(Rigidbody))]
public class CubeInteractor : MonoBehaviour
{
    void Reset()
    {
        var col = GetComponent<Collider>();
        col.isTrigger = true;

        var rb = GetComponent<Rigidbody>();
        rb.isKinematic = true;
        rb.useGravity = false;
    }

    void OnTriggerEnter(Collider other)
    {
        if (other.TryGetComponent<HumanoidController>(out var hum))
        {
            GameSignals.EmitCubeHitsHumanoid(hum.type, transform);
        }
    }
}

```

3. Shield.cs

```C#
using UnityEngine;

[RequireComponent(typeof(Collider))]
[RequireComponent(typeof(Rigidbody))]
public class Shield : MonoBehaviour
{
    public ShieldType type = ShieldType.Type1;

    void Reset()
    {
        var col = GetComponent<Collider>();
        col.isTrigger = false; // físico

        var rb = GetComponent<Rigidbody>();
        rb.isKinematic = false; // físico
        rb.useGravity = false;
        rb.constraints = RigidbodyConstraints.FreezeAll; // no se mueva
    }
}

```

4. HumanoidController.cs

```C#
using System.Linq;
using UnityEngine;

[RequireComponent(typeof(Collider))]
[RequireComponent(typeof(Rigidbody))]
public class HumanoidController : MonoBehaviour
{
    public HumanoidType type = HumanoidType.Type1;
    public float speed = 3.5f;
    public float stopDistance = 0.3f;

    [Header("Regla A: escudo destino cuando el cubo golpea a un Humanoid Type2")]
    public Transform selectedShieldType1; // asigna un escudo tipo1

    private Transform _currentTarget;
    private Renderer _rend;
    private Color _originalColor;
    private Rigidbody _rb;
    private Vector3 _prevPos;

    void Awake()
    {
        TryGetComponent(out _rend);
        if (_rend != null) _originalColor = _rend.material.color;

        _rb = GetComponent<Rigidbody>();
        _rb.isKinematic = true;   // nos movemos por MovePosition
        _rb.useGravity  = false;

        _prevPos = transform.position;
    }

    void OnEnable()  => GameSignals.OnCubeHitsHumanoid += OnCubeHitsHumanoid;
    void OnDisable() => GameSignals.OnCubeHitsHumanoid -= OnCubeHitsHumanoid;

    void Update()
    {
        // movimiento hacia el target
        if (_currentTarget != null)
        {
            Vector3 toTarget = _currentTarget.position - transform.position;
            float dist = toTarget.magnitude;

            if (dist <= stopDistance)
            {
                _currentTarget = null;
            }
            else
            {
                Vector3 step = toTarget.normalized * (speed * Time.deltaTime);
                Vector3 next = transform.position + step;

                // Si hay Rigidbody, usa MovePosition (mejor para colisiones kinemáticas)
                if (_rb != null && _rb.isKinematic)
                    _rb.MovePosition(next);
                else
                    transform.position = next;
            }
        }
    }

    // Reglas A y B
    void OnCubeHitsHumanoid(HumanoidType hitType, Transform cube)
    {
        if (type != HumanoidType.Type1) return; // solo afecta al grupo 1

        if (hitType == HumanoidType.Type2)
        {
            // REGLA A: ir al escudo seleccionado (Type1)
            if (selectedShieldType1 != null)
                _currentTarget = selectedShieldType1;
        }
        else if (hitType == HumanoidType.Type1)
        {
            // REGLA B: ir al escudo Type2 más cercano
            var shields = FindObjectsOfType<Shield>()
                          .Where(s => s.type == ShieldType.Type2)
                          .Select(s => s.transform)
                          .ToList();
            if (shields.Count > 0)
                _currentTarget = GetNearest(shields);
        }
    }

    Transform GetNearest(System.Collections.Generic.List<Transform> targets)
    {
        Transform best = null; float bestSqr = float.PositiveInfinity;
        Vector3 p = transform.position;
        foreach (var t in targets)
        {
            float d = (t.position - p).sqrMagnitude;
            if (d < bestSqr) { bestSqr = d; best = t; }
        }
        return best;
    }

    // REGLA C: cambio de color al colisionar con escudo físico
    void OnCollisionEnter(Collision collision)
    {
        if (collision.collider.TryGetComponent<Shield>(out var shield))
        {
            if (_rend != null) _rend.material.color = Color.yellow;
        }
    }

    // Si quieres que vuelva al color original al salir:
    void OnCollisionExit(Collision collision)
    {
        if (collision.collider.TryGetComponent<Shield>(out var shield))
        {
            if (_rend != null) _rend.material.color = _originalColor;
        }
    }
}

```

En el siguiente GIF se puede ver el funcionamiento de los Scripts, primero un cubo toca a los humanoides de tipo 1 y después otro cae y toca a los de tipo 2. Viendose en cada caso que es lo que pasa.

![Ejercicio 3](Img/EVENTOS%20Ejercicio%203.gif)