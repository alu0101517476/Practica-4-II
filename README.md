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

1. `GameSignals.cs`

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

2. `CubeInteractor.cs`

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

3. `Shield.cs`

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

4. `HumanoidController.cs`

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

### **Ejercicio 4**

En este ejercicio se implementó una mecánica basada en la detección de proximidad entre el cubo y un objeto de referencia. Cuando el cubo entra en esa zona, se activa un evento global que provoca comportamientos diferentes en los humanoides según su tipo.

Se crearon dos tipos de humanoides:

Tipo 1: al activarse el evento, se teletransportan directamente a un escudo objetivo fijado de antemano.

Tipo 2: al activarse el evento, se orientan automáticamente hacia un objeto específico de la escena.

Para detectar la proximidad, se colocó un objeto de referencia con un SphereCollider en modo Trigger y el script ProximityZone. Este objeto actúa como sensor y emite una señal cuando el cubo entra en su área.

Se utilizó un sistema de eventos mediante el script `GameSignals`, que permite comunicar la detección al resto de los objetos de forma ordenada.

Cada humanoide tiene el script `HumanoidReaction`, que interpreta el evento y ejecuta la acción correspondiente según su tipo.

Con esta estructura, se logró que el comportamiento de los personajes se active automáticamente al acercarse el cubo al objeto de referencia, cumpliendo todos los requisitos del enunciado.

El código de los Scripts es el siguiente:

1. `GameSignals.cs`

```C#
using System;
using UnityEngine;

public enum HumanoidType { Type1, Type2 }
public enum ShieldType { Type1, Type2 }

public static class GameSignals
{
    // EJERCICIO 3 (cubo toca humanoide -> reaccionan los Type1)
    public static event Action<HumanoidType, Transform> OnCubeHitsHumanoid;

    public static void EmitCubeHitsHumanoid(HumanoidType hitType, Transform cube)
        => OnCubeHitsHumanoid?.Invoke(hitType, cube);

    // (Si además usas el ejercicio de “proximidad”)
    public static event Action<Transform> OnCubeNearReference;
    public static void EmitCubeNearReference(Transform reference)
        => OnCubeNearReference?.Invoke(reference);
}

```

2. `ProximityZone.cs`

```C#
using UnityEngine;

[RequireComponent(typeof(SphereCollider))]
public class ProximityZone : MonoBehaviour
{
    [Tooltip("Etiqueta que debe tener el Cubo para activar la proximidad.")]
    public string cubeTag = "Cube";

    void Reset()
    {
        var col = GetComponent<SphereCollider>();
        col.isTrigger = true;
        col.radius = 3f; // Ajusta tu radio de proximidad
    }

    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag(cubeTag))
        {
            GameSignals.EmitCubeNearReference(transform);
        }
    }
}

```

3. `HumanoidReaction.cs`

```C#
using UnityEngine;

public class HumanoidReaction : MonoBehaviour
{
    [Header("Tipo de humanoide")]
    public HumanoidType type = HumanoidType.Type1; // usa el enum definido en GameSignals.cs

    [Header("Solo Type1: escudo/posición de teletransporte")]
    public Transform teleportTarget;

    [Header("Solo Type2: objeto a mirar al activarse")]
    public Transform lookAtTarget;
    public float rotateSpeedDegPerSec = 360f; // velocidad de giro

    private bool _shouldOrient;

    void OnEnable()  => GameSignals.OnCubeNearReference += OnCubeNearReference;
    void OnDisable() => GameSignals.OnCubeNearReference -= OnCubeNearReference;

    private void OnCubeNearReference(Transform referenceObject)
    {
        if (type == HumanoidType.Type1)
        {
            if (teleportTarget != null)
            {
                // Teletransporte directo
                transform.position = teleportTarget.position;
            }
        }
        else // Type2
        {
            if (lookAtTarget != null)
            {
                // Activa giro suave en Update
                _shouldOrient = true;

                // Si lo quieres instantáneo, usa esta línea y comenta la anterior:
                // transform.rotation = Quaternion.LookRotation((lookAtTarget.position - transform.position).normalized, Vector3.up);
            }
        }
    }

    void Update()
    {
        if (type != HumanoidType.Type2 || !_shouldOrient || lookAtTarget == null) return;

        Vector3 dir = lookAtTarget.position - transform.position;
        dir.y = 0f; // ignorar inclinación vertical (opcional)
        if (dir.sqrMagnitude < 1e-6f) return;

        Quaternion targetRot = Quaternion.LookRotation(dir.normalized, Vector3.up);
        transform.rotation = Quaternion.RotateTowards(
            transform.rotation,
            targetRot,
            rotateSpeedDegPerSec * Time.deltaTime
        );

        // Detener giro cuando ya apunta al objetivo
        if (Quaternion.Angle(transform.rotation, targetRot) < 1f)
            _shouldOrient = false;
    }
}

```

En el siguiente GIF se puede ver el comportamiento del ejercicio resuelto: 

![Ejercicio 4](Img/EVENTOS%204.gif)

### **Ejercicio 5**

Para implementar la mecánica solicitada, se configuró una escena donde el cubo actúa como jugador y puede desplazarse manualmente en el editor para simular la interacción. Se crearon dos tipos de escudos en la escena (Tipo 1 y Tipo 2), los cuales otorgan diferentes puntuaciones al ser recogidos: 5 puntos para los escudos de tipo 1 y 10 puntos para los de tipo 2.

Para lograr esta funcionalidad se desarrollaron los siguientes elementos:

Script `Shield.cs`: identifica el tipo de escudo y define cuántos puntos otorga. Al ser recogido, el escudo se desactiva para evitar múltiples colisiones.

Script `ScoreManager.cs`: gestiona la puntuación total del jugador y muestra el resultado actualizado en la consola cada vez que se recoge un escudo.

Script `PlayerCollector.cs`: asignado al cubo. Detecta colisiones mediante OnTriggerEnter o OnCollisionEnter, identifica si el objeto golpeado es un escudo y, en ese caso, añade los puntos correspondientes e informa del evento en la consola.

Cada escudo fue configurado con un collider físico, y el cubo se preparó con collider y rigidbody, permitiendo detectar correctamente las colisiones desde el editor. La prueba de funcionamiento se realizó moviendo manualmente el cubo en la escena para verificar que, al colisionar con cada escudo, este desaparece y la puntuación se actualiza correctamente.

Con esta estructura se consigue un sistema funcional de recolección de objetos con puntuación dinámica, cumpliendo los requisitos especificados en el enunciado.

El código de los Scripts comentados es el siguiente: 

1. `Shield.cs`
```C#
using UnityEngine;

[RequireComponent(typeof(Collider))]
public class Shield : MonoBehaviour
{
    [Header("Tipo y puntos")]
    public ShieldType type = ShieldType.Type1;  // Ahora usa el enum definido globalmente
    [Tooltip("Si es 0, se calculará por tipo: Type1=5, Type2=10")]
    public int customPoints = 0;

    private bool _collected = false;

    public int Points =>
        customPoints > 0
        ? customPoints
        : (type == ShieldType.Type1 ? 5 : 10);

    public void Collect()
    {
        if (_collected) return;
        _collected = true;
        gameObject.SetActive(false);
    }
}

```

2. `ScoreManager.cs`

```C#
using UnityEngine;

public class ScoreManager : MonoBehaviour
{
    public int Score { get; private set; } = 0;

    private static ScoreManager _instance;
    public static ScoreManager I
    {
        get
        {
            if (_instance == null)
            {
                _instance = FindObjectOfType<ScoreManager>();
                if (_instance == null)
                {
                    var go = new GameObject("ScoreManager");
                    _instance = go.AddComponent<ScoreManager>();
                }
            }
            return _instance;
        }
    }

    public void AddPoints(int points, ShieldType type, string fromName)
    {
        Score += points;
        Debug.Log($"[Score] +{points} (Shield {type}) por colisión con '{fromName}'. Total = {Score}");
    }
}

```

3. `PlayerCollector.cs`

```C#
using UnityEngine;

[RequireComponent(typeof(Collider))]
public class PlayerCollector : MonoBehaviour
{
    [Header("¿Usar Trigger en el collider del cubo? (si no, será colisión física)")]
    public bool useTrigger = true;

    void Reset()
    {
        var col = GetComponent<Collider>();
        col.isTrigger = useTrigger;

        // Recomendado un Rigidbody para el cubo (cinemático si es trigger)
        var rb = GetComponent<Rigidbody>();
        if (rb == null) rb = gameObject.AddComponent<Rigidbody>();
        rb.isKinematic = useTrigger;     // trigger: cinemático
        rb.useGravity = false;
    }

    // --- Vía Trigger ---
    void OnTriggerEnter(Collider other)
    {
        if (!useTrigger) return;
        TryCollect(other.gameObject, "OnTriggerEnter");
    }

    // --- Vía Colisión física ---
    void OnCollisionEnter(Collision collision)
    {
        if (useTrigger) return;
        TryCollect(collision.collider.gameObject, "OnCollisionEnter");
    }

    private void TryCollect(GameObject hit, string source)
    {
        // El escudo puede estar en el mismo objeto o en un padre
        if (hit.TryGetComponent<Shield>(out var shield) == false)
            shield = hit.GetComponentInParent<Shield>();

        if (shield == null) return;

        // Sumar y registrar
        ScoreManager.I.AddPoints(shield.Points, shield.type, hit.name);

        // Desactivar/recoger
        shield.Collect();

        // (Ayuda visual en consola)
        Debug.Log($"[{source}] El cubo chocó con: {hit.name}");
    }
}

```

En el siguiente GIF se puede ver el ejercicio en funcionamiento: 

![Ejercicio 5](Img/EVENTOS%205.gif)

Como se puede apreciar, en la consola aparece la puntuación que obtiene el cubo al tocar cada escudo

### **Ejercicio 6**

Para resolver el ejercicio, se creó una interfaz utilizando TextMeshPro para mostrar la puntuación en pantalla.
El sistema funciona a través del ScoreManager, que almacena los puntos y emite un evento cuando la puntuación cambia.
El script `ScoreUI` recibe este evento y actualiza el texto del Canvas automáticamente.
El cubo detecta los escudos mediante colisiones, suma la puntuación correspondiente y activa la actualización de la interfaz.
Con esta estructura, la puntuación del jugador se muestra en tiempo real de forma clara y automática.

Los Scripts que se han utilizado para este ejercicio son los siguientes: 

1. `ScoreUI.cs`

```C#
using UnityEngine;
using TMPro;

public class ScoreUI : MonoBehaviour
{
    public TextMeshProUGUI scoreText;

    void Start()
    {
        UpdateScore(0); // Inicializa en 0
        ScoreManager.I.OnScoreChanged += UpdateScore; // Suscribirse al evento
    }

    void OnDestroy()
    {
        ScoreManager.I.OnScoreChanged -= UpdateScore; // Buenas prácticas
    }

    void UpdateScore(int newScore)
    {
        scoreText.text = "Puntuación: " + newScore;
    }
}

```

2. `ScoreManager.cs`

```C#
using UnityEngine;
using System;

public class ScoreManager : MonoBehaviour
{
    public int Score { get; private set; }

    // Singleton con autocreación segura
    private static ScoreManager _instance;
    public static ScoreManager I
    {
        get
        {
            if (_instance == null)
            {
                _instance = FindObjectOfType<ScoreManager>();
                if (_instance == null)
                {
                    var go = new GameObject("ScoreManager");
                    _instance = go.AddComponent<ScoreManager>();
                }
            }
            return _instance;
        }
    }

    public event Action<int> OnScoreChanged;

    void Awake()
    {
        if (_instance != null && _instance != this) { Destroy(gameObject); return; }
        _instance = this;
        DontDestroyOnLoad(gameObject);
    }

    public void AddPoints(int points, ShieldType type, string fromName)
    {
        Score += points;
        Debug.Log($"[Score] +{points} (Shield {type}) por '{fromName}'. Total = {Score}");
        OnScoreChanged?.Invoke(Score);
    }
}

```

El GIF que muestra en funcionamiento la escena es el siguiente:

![Ejercicio 6](Img/EVENTOS%206.gif)

### **Ejercicio 7**

Para mostrar un mensaje de felicitación al alcanzar 100 puntos, se añadió un texto en la interfaz gráfica mediante un elemento UI (TextMeshPro), ubicado en la parte central de la pantalla y desactivado al inicio. A través del script `ScoreUI`, se detecta cuando la puntuación del jugador alcanza un múltiplo de 100 utilizando un evento del `ScoreManager`. Cuando esto ocurre, el script activa el texto de “Enhorabuena” durante dos segundos mediante una corrutina, y luego lo oculta automáticamente. Con esta lógica se consigue que el mensaje aparezca de forma temporal cada vez que el jugador supera un nuevo umbral de 100 puntos.

El único script que se ha modificado es `ScoreUI.cs`

```C#
using UnityEngine;
using TMPro;
using System.Collections;

public class ScoreUI : MonoBehaviour
{
    [Header("Referencias UI")]
    public TextMeshProUGUI scoreText;
    public TextMeshProUGUI congratsText;

    private int lastMilestone = 0;

    void Start()
    {
        UpdateScore(0);
        ScoreManager.I.OnScoreChanged += UpdateScore;
        congratsText.gameObject.SetActive(false); // ocultar mensaje al inicio
    }

    void OnDestroy()
    {
        ScoreManager.I.OnScoreChanged -= UpdateScore;
    }

    void UpdateScore(int newScore)
    {
        scoreText.text = "Puntuación: " + newScore;

        // Cada 100 puntos muestra mensaje
        if (newScore >= lastMilestone + 100)
        {
            lastMilestone = newScore - (newScore % 100); // marca el siguiente múltiplo
            StartCoroutine(ShowCongrats());
        }
    }

    IEnumerator ShowCongrats()
    {
        congratsText.text = "¡Enhorabuena!";
        congratsText.gameObject.SetActive(true);
        yield return new WaitForSeconds(2f);
        congratsText.gameObject.SetActive(false);
    }
}

```

A continuación se muestra un GIF con el funcionamiento del ejercicio:

![Ejercicio 7](Img/EVENTOS%207.gif)

### **Ejercicio 8**

La escena del proyecto que hemos escogido es una escena sencilla en la que tenemos un personaje capsula y unos personajes de tipo zombie. Cuando el personaje capsula mira a los zombies, estos se vuelven más pequeños, contando como que se han recolectado con la mirada. En la siguiente imagen se puede ver lo que es un prototipo muy básico de la escena del proyecto

![Ejercicio 8](Img/Escena%20proyecto.png)

A continuación, se muestra un vídeo con la mecánica de recolección de los zombies

![Video Ejercicio 8](Img/Funcionamiento%20recolección.gif)

### **Ejercicio 9**

En mi caso, en el ejercicio 3 ya apliqué que el ejercicio se resolviera con el cubo siendo un objeto físico. Esto lo podemos comprobar cuando un cubo cae desde el cielo por la gravedad y colisiona con uno de los enemigos de tipo 2, los cuales son verdes y los enemigos de tipo 1, que son rojos, van hacia el otro cubo.
En el siguiente GIF se puede ver este comportamiento:

![Ejercicio 9](Img/EVENTOS%20Ejercicio%203.gif)