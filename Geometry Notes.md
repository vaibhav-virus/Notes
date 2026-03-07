# Geometry Notes

## Common Assumptions

```math
     A = (X_{A} , Y_{A}, Z_{A}) 
```

```math
    B = (X_{B} , Y_{B}, Z_{B})
```

```math
    C = (X_{C} , Y_{C}, Z_{C})
```

## Geometry basics

* ### Calculate distance between points A and B

```math
|\overline{AB}|= Distance = \sqrt{(X_{B} - X_{A})^{2} + (Y_{B} - Y_{A})^{2} + (Z_{B} - Z_{A})^{2}}
```

* ### Calculate vector from point A to point B

```math
\vec{AB}=(X_{B} - X_{A})i + (Y_{B} - Y_{A})j + (Z_{B} - Z_{A})k
```

* ### Dot product of two vectors A and B

    Let $` \vec{A} `$ and $` \vec{B} `$ be our two vectors.

    >Dot product of two vectors returns a scalar i.e a number.

    1. **If** $` \vec{A} . \vec{B} = 0 `$ **Then** Both vectors are **perpendicular** to each other.
    2. **If** $` \vec{A} . \vec{B} = positive `$ **Then** Both vectors are in **same** directional sense.
    3. **If** $` \vec{A} . \vec{B} = negative `$ **Then** Both vectors are in **opposite** directional sense.

* ### Cross product of two vectors A and B

    Let $` \vec{A} `$ and $` \vec{B} `$ be two vectors.
    >Cross product of two vectors returns a new vector which is perpendicular to both the input vectors.

    1. **If** $` \vec{A} \times \vec{B} = \vec{0} `$( Zero vector)  **Then** Both vectors A and B are **Parallel** to eachother.
    2. **If** $` \vec{A} \times \vec{B} = \vec{N} `$ **Then** $` \vec{N} `$ is **Normal/Perpendicular** to $` \vec{A} `$ and $` \vec{B} `$.

## Geometry problems

* ### How to check if a point P is on a line segment AB?

* ### Approach 1 — Distance Sum

    Distance AB = Distance AP + Distance PB

    Click on image to play.
    [![IMAGE ALT TEXT HERE](https://i.ytimg.com/vi/rOoPLrGnizY/hq720.jpg?sqp=-oaymwEcCNAFEJQDSFXyq4qpAw4IARUAAIhCGAFwAcABBg==&rs=AOn4CLDb8G9QaPU-PGW3uGSBAyMNeHxnQw)](https://youtu.be/rOoPLrGnizY?si=pi2m3fnxqjqahNw5&t=74)

* ### Approach 2 — Cross Product (Preferred in CAD/Graphics)

    > If P is on line AB, then `AB × AP = 0` (vectors are parallel/collinear) AND P is within the bounding box of A and B.

    **Step 1:** Compute cross product of $`\vec{AB}`$ and $`\vec{AP}`$

    ```math
    \vec{AB} \times \vec{AP} = \vec{0} \implies P \text{ is collinear with A and B}
    ```

    **Step 2:** Check P is between A and B (not just on the infinite line):

    ```math
    \min(X_A, X_B) \le X_P \le \max(X_A, X_B)
    ```
    ```math
    \min(Y_A, Y_B) \le Y_P \le \max(Y_A, Y_B)
    ```

    ```cpp
    bool isOnSegment(Point A, Point B, Point P) {
        // Cross product (2D): AB × AP
        double cross = (B.x - A.x) * (P.y - A.y) - (B.y - A.y) * (P.x - A.x);
        if (std::abs(cross) > EPSILON) return false; // not collinear

        // Check bounding box
        return std::min(A.x, B.x) <= P.x && P.x <= std::max(A.x, B.x) &&
               std::min(A.y, B.y) <= P.y && P.y <= std::max(A.y, B.y);
    }
    ```

---

* ### How to check if point P is inside a Circle?

    > **Key idea:** P is inside the circle if its distance from the centre C is less than the radius r.

    Given: Circle with centre $`C = (C_x, C_y)`$ and radius $`r`$.

    ```math
    (P_x - C_x)^2 + (P_y - C_y)^2 < r^2
    ```

    > ⚡ **Pro tip:** Compare squared distances — avoids the expensive `sqrt()`. Critical in real-time CAD hit-testing!

    | Result | Meaning |
    |---|---|
    | $`d^2 < r^2`$ | P is **inside** the circle |
    | $`d^2 = r^2`$ | P is **on** the circle boundary |
    | $`d^2 > r^2`$ | P is **outside** the circle |

    ```cpp
    bool isInsideCircle(Point C, double r, Point P) {
        double dx = P.x - C.x;
        double dy = P.y - C.y;
        return (dx*dx + dy*dy) < (r * r); // no sqrt needed!
    }
    ```

    **3D Extension — Point inside a Sphere:**

    ```math
    (P_x - C_x)^2 + (P_y - C_y)^2 + (P_z - C_z)^2 < r^2
    ```

    ```cpp
    bool isInsideSphere(Point3D C, double r, Point3D P) {
        double dx = P.x - C.x, dy = P.y - C.y, dz = P.z - C.z;
        return (dx*dx + dy*dy + dz*dz) < (r * r);
    }
    ```

---

* ### How to check if point P is inside an Axis-Aligned Rectangle (AABB)?

    > **AABB = Axis-Aligned Bounding Box.** Sides are parallel to X and Y axes. Used everywhere in CAD viewport picking, collision detection, and UI hit-testing.

    Given: Rectangle defined by bottom-left corner $`A = (x_{min}, y_{min})`$ and top-right corner $`B = (x_{max}, y_{max})`$.

    ```math
    x_{min} \le P_x \le x_{max} \quad \text{AND} \quad y_{min} \le P_y \le y_{max}
    ```

    ```cpp
    bool isInsideAABB(Point minPt, Point maxPt, Point P) {
        return P.x >= minPt.x && P.x <= maxPt.x &&
               P.y >= minPt.y && P.y <= maxPt.y;
    }
    ```

    **3D Extension — Point inside an Axis-Aligned Box:**

    ```cpp
    bool isInsideAABB3D(Point3D minPt, Point3D maxPt, Point3D P) {
        return P.x >= minPt.x && P.x <= maxPt.x &&
               P.y >= minPt.y && P.y <= maxPt.y &&
               P.z >= minPt.z && P.z <= maxPt.z;
    }
    ```

    > **Interview Gotcha:** Always ask — is the rectangle axis-aligned or rotated? If rotated, you need an **OBB (Oriented Bounding Box)** test instead (see below).

---

* ### How to check if point P is inside an Oriented Rectangle (OBB)?

    > **OBB = Oriented Bounding Box.** The rectangle can be rotated at any angle. Used in CAD for rotated parts/components.

    **Key Idea — Transform to Local Space:**

    An OBB has a centre $`C`$, two local axes $`\hat{u}`$ (width direction) and $`\hat{v}`$ (height direction), and half-extents $`h_w`$ and $`h_h`$.

    **Step 1:** Compute the vector from centre C to point P:

    ```math
    \vec{d} = P - C
    ```

    **Step 2:** Project $`\vec{d}`$ onto the **local axes** of the rectangle:

    ```math
    p_u = \vec{d} \cdot \hat{u}, \quad p_v = \vec{d} \cdot \hat{v}
    ```

    **Step 3:** Check if projections are within half-extents:

    ```math
    |p_u| \le h_w \quad \text{AND} \quad |p_v| \le h_h
    ```

    ```cpp
    bool isInsideOBB(Point C, Vec2 uAxis, Vec2 vAxis,
                     double halfW, double halfH, Point P) {
        Vec2 d = { P.x - C.x, P.y - C.y };
        double pu = dot(d, uAxis); // project onto local X
        double pv = dot(d, vAxis); // project onto local Y
        return std::abs(pu) <= halfW && std::abs(pv) <= halfH;
    }
    ```

    > 💡 **Why this works:** By projecting into local space, the rotated rectangle becomes an AABB problem!

---

* ### How to check if point P is inside Triangle ABC?

    > This is one of the most important tests in 3D CAD and graphics — used in ray-triangle intersection, mesh picking, UV mapping, and rendering.

    ---

    * #### Method 1 — Same-Side / Sign of Cross Products

        > **Intuition:** If P is inside triangle ABC, then P must be on the **same side** of every edge (AB, BC, CA) as the opposite vertex.

        For each edge, compute the cross product. If all signs are the same, P is inside.

        **For edge AB:** Check P and C are on the same side.

        ```math
        \text{sign}(\vec{AB} \times \vec{AP}) == \text{sign}(\vec{AB} \times \vec{AC})
        ```

        **For edge BC:** Check P and A are on the same side.

        ```math
        \text{sign}(\vec{BC} \times \vec{BP}) == \text{sign}(\vec{BC} \times \vec{BA})
        ```

        **For edge CA:** Check P and B are on the same side.

        ```math
        \text{sign}(\vec{CA} \times \vec{CP}) == \text{sign}(\vec{CA} \times \vec{CB})
        ```

        ```cpp
        double cross2D(Vec2 O, Vec2 A, Vec2 B) {
            // Cross product of OA × OB
            return (A.x - O.x) * (B.y - O.y) - (A.y - O.y) * (B.x - O.x);
        }

        bool isInsideTriangle_SameSide(Point A, Point B, Point C, Point P) {
            double d1 = cross2D(A, B, P);
            double d2 = cross2D(B, C, P);
            double d3 = cross2D(C, A, P);

            bool has_neg = (d1 < 0) || (d2 < 0) || (d3 < 0);
            bool has_pos = (d1 > 0) || (d2 > 0) || (d3 > 0);

            return !(has_neg && has_pos); // all same sign = inside
        }
        ```

    ---

    * #### Method 2 — Barycentric Coordinates ⭐ (Most Elegant — CAD/GPU Favourite)

        > **Intuition:** Any point P inside triangle ABC can be written as a weighted combination of A, B, C. If all weights are non-negative and sum to 1, P is inside.

        ```math
        P = \alpha A + \beta B + \gamma C \quad \text{where} \quad \alpha + \beta + \gamma = 1
        ```

        ```math
        \alpha \ge 0, \quad \beta \ge 0, \quad \gamma \ge 0 \implies P \text{ is inside triangle ABC}
        ```

        **Computing Barycentric Coords using Areas:**

        ```math
        \alpha = \frac{\text{Area}(PBC)}{\text{Area}(ABC)}, \quad
        \beta  = \frac{\text{Area}(APC)}{\text{Area}(ABC)}, \quad
        \gamma = \frac{\text{Area}(ABP)}{\text{Area}(ABC)}
        ```

        > Area of a triangle from cross product:
        ```math
        \text{Area}(XYZ) = \frac{1}{2} |(\vec{XY} \times \vec{XZ})|
        ```

        ```cpp
        bool isInsideTriangle_Barycentric(Point A, Point B, Point C, Point P) {
            // Using cross products (proportional to area, skip the /2)
            auto cross = [](Point O, Point X, Point Y) {
                return (X.x-O.x)*(Y.y-O.y) - (X.y-O.y)*(Y.x-O.x);
            };

            double areaABC = cross(A, B, C);
            double alpha   = cross(P, B, C) / areaABC; // weight of A
            double beta    = cross(A, P, C) / areaABC; // weight of B
            double gamma   = cross(A, B, P) / areaABC; // weight of C

            return alpha >= 0 && beta >= 0 && gamma >= 0;
            // gamma = 1 - alpha - beta, so no need to compute separately
        }
        ```

        > **Interview gold:** Barycentric coords are used in GPU rasterization, texture interpolation, and normal interpolation across triangle faces.

    ---

    * #### Method 3 — Area Sum Method

        > **Intuition:** If P is inside triangle ABC, the three smaller triangles PAB, PBC, PCA together have the **same area** as ABC.

        ```math
        \text{Area}(ABC) = \text{Area}(PAB) + \text{Area}(PBC) + \text{Area}(PCA)
        ```

        > If P is outside, the sum of the three sub-triangle areas will be **greater** than Area(ABC).

        ```cpp
        double triangleArea(Point X, Point Y, Point Z) {
            return 0.5 * std::abs((Y.x-X.x)*(Z.y-X.y) - (Z.x-X.x)*(Y.y-X.y));
        }

        bool isInsideTriangle_AreaSum(Point A, Point B, Point C, Point P) {
            double areaABC = triangleArea(A, B, C);
            double areaPAB = triangleArea(P, A, B);
            double areaPBC = triangleArea(P, B, C);
            double areaPCA = triangleArea(P, C, A);
            return std::abs(areaPAB + areaPBC + areaPCA - areaABC) < EPSILON;
        }
        ```

    ---

    * #### Summary Table — Triangle Methods

        | Method | Speed | Handles Edge Cases | Notes |
        |---|---|---|---|
        | Same-Side (Cross Product) | Fast | Yes | Intuitive, easy to remember |
        | Barycentric | Fast | Yes | Used in GPU shaders, gives weights |
        | Area Sum | Slower (abs + floats) | Needs epsilon | Easy to explain in interviews |

---

* ### How to check if point P is inside an Arbitrary Polygon? (Ray Casting)

    > Used in CAD for complex region selections, 2D sketch containment checks, GIS, and UI.

    **Key Idea:** Cast a ray from P in any direction (e.g. +X). Count how many times it crosses the polygon boundary. **Odd = inside, Even = outside.**

    ```
    P ----ray----> | edge | edge |
                    cross  cross
                    (1)    (2) → even = outside
    ```

    ```cpp
    bool isInsidePolygon_RayCast(std::vector<Point>& poly, Point P) {
        int n = poly.size();
        bool inside = false;
        for (int i = 0, j = n-1; i < n; j = i++) {
            Point vi = poly[i], vj = poly[j];
            // Check if ray from P horizontally crosses edge vj->vi
            if (((vi.y > P.y) != (vj.y > P.y)) &&
                (P.x < (vj.x - vi.x) * (P.y - vi.y) / (vj.y - vi.y) + vi.x)) {
                inside = !inside;
            }
        }
        return inside;
    }
    ```

    > **Edge case to mention in interviews:** What if the ray passes exactly through a vertex? Handle by consistently treating vertices on the boundary as one side.

---

* ### Bonus — Normal of a Triangle (Critical for 3D CAD)

    > The normal tells you which way a face is pointing — used for lighting, backface culling, and boolean operations in CAD.

    Given triangle with vertices A, B, C:

    ```math
    \hat{N} = \frac{\vec{AB} \times \vec{AC}}{|\vec{AB} \times \vec{AC}|}
    ```

    ```cpp
    Vec3 triangleNormal(Point3D A, Point3D B, Point3D C) {
        Vec3 AB = { B.x-A.x, B.y-A.y, B.z-A.z };
        Vec3 AC = { C.x-A.x, C.y-A.y, C.z-A.z };
        Vec3 N  = cross(AB, AC); // cross product
        return normalize(N);
    }
    ```

    > **Interview link:** Use this normal + dot product to check if point P is on the same side as the outward-facing normal of a face (used in solid geometry / B-Rep models in Autodesk tools).

---

* ### Bonus — Is Point P on the Correct Side of a Plane?

    > Planes are fundamental in 3D CAD — cutting planes, workplanes, clipping. Every solid in B-Rep is bounded by planar (or curved) faces.

    Given: A plane defined by a point $`Q`$ on the plane and its normal $`\hat{N}`$.

    ```math
    (\vec{QP}) \cdot \hat{N} > 0 \implies P \text{ is in front of the plane (same side as normal)}
    ```
    ```math
    (\vec{QP}) \cdot \hat{N} = 0 \implies P \text{ is on the plane}
    ```
    ```math
    (\vec{QP}) \cdot \hat{N} < 0 \implies P \text{ is behind the plane}
    ```

    ```cpp
    double signedDistToPlane(Point3D Q, Vec3 N, Point3D P) {
        // Positive = in front, Negative = behind
        Vec3 QP = { P.x-Q.x, P.y-Q.y, P.z-Q.z };
        return dot(QP, N); // assumes N is unit vector for true distance
    }
    ```

    > **Real-world use:** In CSG (Constructive Solid Geometry) operations like Union/Intersection/Subtract in Autodesk Fusion 360, each face's plane is used to classify points and determine which geometry to keep.

