# OPTICS

```
PROCEDURE OPTICS(SetOfObjects, ε, MinPts)

    # Archivo/lista donde guardaremos el ordering final.
    OrderedFile ← vacío
    
    # Recorremos todos los objetos del dataset.
    FOR each Object in SetOfObjects DO
    
        # Si todavía no fue procesado,
        # comenzamos una expansión desde él.
        IF Object.Processed = FALSE THEN
            ExpandClusterOrder(
                SetOfObjects,
                Object,
                ε,
                MinPts,
                OrderedFile
            )
        END IF
    END FOR
    
    RETURN OrderedFile

END PROCEDURE
```

```
PROCEDURE ExpandClusterOrder(
    SetOfObjects,
    Object,
    ε,
    MinPts,
    OrderedFile
)

    # Obtener vecinos ε
    neighbors ← neighbors(Object, ε)

    # Marcar objeto como procesado
    Object.Processed ← TRUE

    # Inicializá reachability_distance como undefined
    Object.reachability_distance ← UNDEFINED

    # Calcular core-distance
    Object.core_distance ← CoreDistance(
        Object,
        neighbors,
        ε,
        MinPts
    )

    # Guardar objeto en el ordering
    OrderedFile.append(Object)

    # Si NO es core point, termina la expansión
    IF Object.core_distance = UNDEFINED THEN
        RETURN
    END IF

    # Crear priority queue
    OrderSeeds ← empty priority queue

    # Insertar vecinos alcanzables
    Update(OrderSeeds, neighbors, Object)

    # Expandir iterativamente
    WHILE OrderSeeds is not empty DO

        # Elegir el punto más prometedor
        # (menor reachability-distance)
        currentObject ← OrderSeeds.pop_smallest()

        # Obtener vecinos
        neighbors ← neighbors(currentObject, ε)

        # Marcar como procesado
        currentObject.Processed ← TRUE

        # Calcular core-distance
        currentObject.core_distance ← CoreDistance(
            currentObject,
            neighbors,
            ε,
            MinPts
        )

        # Guardar en ordering final
        OrderedFile.append(currentObject)

        # Si es core point, continuar expansión
        IF currentObject.core_distance ≠ UNDEFINED THEN

            Update(
                OrderSeeds,
                neighbors,
                currentObject
            )

        END IF

    END WHILE

END PROCEDURE
```

```
PROCEDURE neighbors(SetOfObjects, Object, ε)
# Función escrita como ejemplo conceptual.
# En la realidad se utilizan índices espaciales como R*-trees.

    # Lista resultado
    neighbors ← empty list

    # Revisar todos los objetos del dataset
    FOR each candidate in SetOfObjects DO

        # Calcular distancia
        d ← distance(Object, candidate)

        # Si está dentro del radio ε, agregarlo al vecindario
        IF d ≤ ε THEN
            neighbors.append(candidate)
        END IF

    END FOR

    RETURN neighbors

END PROCEDURE
```

```
PROCEDURE CoreDistance(
    Object,
    neighbors,
    ε,
    MinPts
)

    # Verificar si es core point
    IF size(neighbors) < MinPts THEN
        RETURN UNDEFINED
    END IF

    # Calcular distancias al objeto
    distances ← empty list

    FOR each neighbor in neighbors DO
        d ← distance(Object, neighbor)
        append d to distances
    END FOR

    # Ordenar distancias
    sort(distances)

    # Tomar distancia al vecino MinPts
    RETURN distances[MinPts]

END PROCEDURE
```

```
PROCEDURE Update(
    OrderSeeds,
    neighbors,
    CenterObject
)

    # Obtener core-distance del centro
    c_dist ← CenterObject.core_distance

    # Revisar todos los vecinos
    FOR each Object in neighbors DO

        # Ignorar procesados
        IF Object.Processed = FALSE THEN

            # Calcular nueva reachability-distance
            new_r_dist ← MAX(
                c_dist,
                distance(CenterObject, Object)
            )

            # Caso 1: objeto nunca visto
            IF Object.reachability_distance = UNDEFINED THEN

                Object.reachability_distance ← new_r_dist

                OrderSeeds.insert(
                    Object,
                    priority = new_r_dist
                )

            # Caso 2: ya estaba en la queue
            ELSE

                # Encontramos mejor camino
                IF new_r_dist < Object.reachability_distance THEN

                    Object.reachability_distance ← new_r_dist

                    OrderSeeds.decrease_key(
                        Object,
                        new_r_dist
                    )

                END IF

            END IF

        END IF

    END FOR

END PROCEDURE
```

## Extraer los clusters

```
PROCEDURE ExtractDBSCANClustering(
    OrderedFile,
    ε',
    MinPts
)

    cluster_id ← 0

    FOR each Object in OrderedFile DO

        # Caso 1: NO alcanzable desde cluster previo
        IF Object.reachability_distance > ε' THEN

            # Si además es core point, comienza un cluster nuevo
            IF Object.core_distance ≤ ε' THEN

                cluster_id ← cluster_id + 1

                Object.cluster ← cluster_id

            # Sino, es ruido
            ELSE

                Object.cluster ← NOISE

            END IF

        # Caso 2: alcanzable desde cluster actual
        ELSE

            Object.cluster ← cluster_id

        END IF

    END FOR

END PROCEDURE
```
