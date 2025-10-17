# Practica-4-II

Autor: Eric Bermúdez Hernández

Email: alu0101517476@ull.edu.es

----

**Descripción del trabajo realizado**

### **Ejercicio 1**

# **Hecho, explicar lo que hice y poner gif**


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
