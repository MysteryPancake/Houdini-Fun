# Houdini `xyzdist()` + `primuv()` Remake

Ever wondered how `primuv()` and `xyzdist()` work? Me neither, but for performance reasons I had to remake them both.

## `xyzdist()`

`xyzdist()` uses an acceleration structure (likely BVH), which I can't recreate easily in VEX or OpenCL.

For this reason, the functions below only work when you know the primnum already. It returns the nearest surface position and primuvw.

What if you don't know the primnum? Try using `pcfind()` to get a few nearby primnums based on their centroids.

Most of the code below is from [*Real-Time Collision Detection* by Christer Ericson](https://www.r-5.org/files/books/computers/algo-list/realtime-3d/Christer_Ericson-Real-Time_Collision_Detection-EN.pdf).

| [Download the HIP file!](./hips/xyzdist_diy.hiplc?raw=true) |
| --- |

### `xyzdist()` in VEX

```js
// Find the closest point to P on a triangle, returns primnum and primuvw
void closestPointTriangle(vector P; vector p0; vector p1; vector p2; vector outP; vector outUVW) {
    // Check if P in vertex region outside A
    vector ab = p1 - p0;
    vector ac = p2 - p0;
    vector ap = P - p0;
    float d1 = dot(ab, ap);
    float d2 = dot(ac, ap);
    if (d1 <= 0 && d2 <= 0) {
        outP = p0;
        outUVW = {0, 0, 0};
        return;
    }
    
    // Check if P in vertex region outside B
    vector bp = P - p1;
    float d3 = dot(ab, bp);
    float d4 = dot(ac, bp);
    if (d3 >= 0 && d4 <= d3) {
        outP = p1;
        outUVW = {1, 0, 0};
        return;
    }

    // Check if P in edge region of AB, return projection of P onto AB
    float vc = d1 * d4 - d3 * d2;
    if (vc <= 0 && d1 >= 0 && d3 <= 0) {
        float v = d1 / (d1 - d3);
        outP = p0 + v * ab;
        outUVW = set(v, 0, 0);
        return;
    }

    // Check if P in vertex region outside C
    vector cp = P - p2;
    float d5 = dot(ab, cp);
    float d6 = dot(ac, cp);
    if (d6 >= 0 && d5 <= d6) {
        outP = p2;
        outUVW = {0, 1, 0};
        return;
    }

    // Check if P in edge region of AC, return projection of P onto AC
    float vb = d5 * d2 - d1 * d6;
    if (vb <= 0 && d2 >= 0 && d6 <= 0) {
        float w = d2 / (d2 - d6);
        outP = p0 + w * ac;
        outUVW = set(0, w, 0);
        return;
    }

    // Check if P in edge region of BC, return projection of P onto BC
    float va = d3 * d6 - d5 * d4;
    if (va <= 0 && (d4 - d3) >= 0 && (d5 - d6) >= 0) {
        float w = (d4 - d3) / (d4 - d3 + d5 - d6);
        outP = p1 + w * (p2 - p1);
        outUVW = set(1 - w, w, 0);
        return;
    }

    // P inside face region
    float denom = va + vb + vc;
    vb /= denom;
    vc /= denom;
    outP = p0 + ab * vb + ac * vc;
    outUVW = set(vb, vc, 0);
}

// Checks if P is outside a tetrahedron
int pointOutsideOfPlane(vector P; vector p0; vector p1; vector p2; vector p3) {
    vector abc = cross(p1 - p0, p2 - p0);
    return dot(P  - p0, abc) * dot(p3 - p0, abc) < 0;
}

// Returns interpolated position based on barycentric UVW coordinates
vector barycentric(vector p0; vector p1; vector p2; vector p3; vector uvw) {
    float u1 = 1 - uvw.x;
    float v1 = 1 - uvw.y;
    return p0 * u1 * v1 
         + p1 * u1 * uvw.y
         + p2 * uvw.x * uvw.y
         + p3 * uvw.x * v1;
}

// Given P and a primnum, returns the closest P and UVW
void xyzdist_diy(int geo; int prim; vector P; vector closestP; vector closestUVW) {
    
    int typeid = primintrinsic(geo, "typeid", prim);
    int pts[] = primpoints(geo, prim);
    int numpt = len(pts);
    
    if (numpt == 2) {
        // Line
        vector p0 = point(geo, "P", pts[0]);
        vector p1 = point(geo, "P", pts[1]);
        
        // Project onto clamped ab
        vector ab = p1 - p0;
        float t = clamp(dot(P - p0, ab) / length2(ab), 0, 1);
        
        closestP = p0 + ab * t;
        closestUVW = set(t, 0, 0);
        
    } else if (numpt == 3) {
        // Triangle
        vector p0 = point(geo, "P", pts[0]);
        vector p1 = point(geo, "P", pts[1]);
        vector p2 = point(geo, "P", pts[2]);
        
        closestPointTriangle(P, p0, p1, p2, closestP, closestUVW);
        
    } else if (numpt == 4 && typeid == 21) {
        // Tetrahedron
        vector p0 = point(geo, "P", pts[0]);
        vector p1 = point(geo, "P", pts[1]);
        vector p2 = point(geo, "P", pts[2]);
        vector p3 = point(geo, "P", pts[3]);
        
        vector tmpP, tmpUVW;
        float bestDist = 1e50;
        
        // If point outside face abc then compute closest point on abc
        if (pointOutsideOfPlane(P, p0, p1, p2, p3)) {
            closestPointTriangle(P, p0, p1, p2, tmpP, tmpUVW);
            float dist = length2(tmpP - P);
            if (dist < bestDist) {
                bestDist = dist;
                closestP = tmpP;
                closestUVW = tmpUVW;
            }
        }
        
        // Repeat test for face acd
        if (pointOutsideOfPlane(P, p0, p2, p3, p1)) {
            closestPointTriangle(P, p0, p2, p3, tmpP, tmpUVW);
            float dist = length2(tmpP - P);
            if (dist < bestDist) {
                bestDist = dist;
                closestP = tmpP;
                closestUVW = tmpUVW.zxy;
            }
        }
        
        // Repeat test for face adb
        if (pointOutsideOfPlane(P, p0, p3, p1, p2)) {
            closestPointTriangle(P, p0, p3, p1, tmpP, tmpUVW);
            float dist = length2(tmpP - P);
            if (dist < bestDist) {
                bestDist = dist;
                closestP = tmpP;
                closestUVW = tmpUVW.yzx;
            }
        }
        
        // Repeat test for face bdc
        if (pointOutsideOfPlane(P, p1, p3, p2, p0)) {
            closestPointTriangle(P, p1, p3, p2, tmpP, tmpUVW);
            float dist = length2(tmpP - P);
            if (dist < bestDist) {
                bestDist = dist;
                closestP = tmpP;
                closestUVW = set(1 - tmpUVW.x - tmpUVW.y, tmpUVW.y, tmpUVW.x);
            }
        }
        
        // Interpolate inside interior
        if (bestDist >= 1e50) {
            vector e1 = p1 - p0;
            vector e2 = p2 - p0;
            vector e3 = p3 - p0;
            vector d = P - p0;
            float u = dot(d,  cross(e2, e3));
            float v = dot(e1, cross(d,  e3));
            float w = dot(e1, cross(e2, d ));
            closestP = P;
            closestUVW = set(u, v, w) / dot(e1, cross(e2, e3));
        }
    } else if (numpt == 4) {
        // Quadrilateral
        vector p0 = point(geo, "P", pts[0]);
        vector p1 = point(geo, "P", pts[1]);
        vector p2 = point(geo, "P", pts[2]);
        vector p3 = point(geo, "P", pts[3]);
        
        // Houdini uses bilinear interpolation for quads, this is hard to solve
        // Below is a simple least squares approximation for good performance
        vector edge_v1 = p2 - p1;
        vector edge_v0 = p3 - p0;
        vector edge_u0 = p1 - p0;
        vector edge_u1 = p2 - p3;
        
        float tolerance = 1e-8;
        closestUVW = {0.5, 0.5, 0};
        
        for (int i = 0; i < 8; ++i) {
            vector r = barycentric(p0, p1, p2, p3, closestUVW) - P;
            if (length2(r) < tolerance) break;
            
            vector dP_du = lerp(edge_v0, edge_v1, closestUVW.y);
            vector dP_dv = lerp(edge_u0, edge_u1, closestUVW.x);
            float A11 = dot(dP_du, dP_du);
            float A12 = dot(dP_du, dP_dv);
            float A22 = dot(dP_dv, dP_dv);
            
            vector2 grad = set(dot(dP_du, r), dot(dP_dv, r));
            if (length2(grad) < tolerance) break;
            
            float det = A11 * A22 - A12 * A12;
            if (abs(det) < tolerance) break;
            
            closestUVW -= set(grad.x * A22 - grad.y * A12, grad.y * A11 - grad.x * A12) / det;
        }
        
        closestUVW = clamp(closestUVW, 0, 1);
        closestP = barycentric(p0, p1, p2, p3, closestUVW);
    } else {
        // General case
        vector tmpP, tmpUVW;
        int closestI;
        float bestDist = 1e50;
        
        // Add an imaginary triangle point in the center
        vector pos[];
        resize(pos, numpt);
        vector avgPos = 0;
        for (int i = 0; i < numpt; ++i) {
            pos[i] = point(geo, "P", pts[i]);
            avgPos += pos[i] / numpt;
        }
        
        // Triangle fan around the center
        vector prevPos = pos[0];
        for (int i = 0; i < numpt; ++i) {
            vector nextPos = pos[(i + 1) % numpt];
            closestPointTriangle(P, avgPos, prevPos, nextPos, tmpP, tmpUVW);
            float dist = length2(P - tmpP);
            if (dist < bestDist) {
                bestDist = dist;
                closestI = i;
                closestP = tmpP;
                closestUVW = tmpUVW;
            }
            prevPos = nextPos;
        }
        
        // Gradient around the center
        float u = (closestUVW.y / (closestUVW.x + closestUVW.y) + closestI) / numpt;
        // Gradient out from the center
        float v = 1 - closestUVW.x - closestUVW.y;
        closestUVW = set(u, v, 0);
    }
}

// We can't use BVH directly in VEX, so pcfind() is close enough
int prims[] = pcfind(2, "P", v@P, chf("radius"), chi("max_prims"));
vector tmpP, tmpUVW, closestP, closestUVW;
float bestDist = 1e50;

// Check through the closest few prims for the nearest match
foreach (int prim; prims) {
    xyzdist_diy(1, prim, v@P, tmpP, tmpUVW);
    float dist = length2(tmpP - v@P);
    if (dist < bestDist) {
        bestDist = dist;
        closestP = tmpP;
        closestUVW = tmpUVW;
    }
}

v@P = closestP;
v@Cd = closestUVW;
```

### `xyzdist()` in OpenCL

```c
#define entriesAt(_arr_, _idx_) ((_idx_ >= 0 && _idx_ < _arr_##_length) ? (_arr_##_index[_idx_+1] - _arr_##_index[_idx_]) : 0)
#define compAt(_arr_, _idx_, _compidx_) ((_idx_ >= 0 && _idx_ < _arr_##_length && _compidx_ >= 0 && _compidx_ < entriesAt(_arr_, _idx_)) ? _arr_[_arr_##_index[_idx_] + _compidx_] : 0)

// Everything below is from "Real-Time Collision Detection" by Christer Ericson

// Find the closest point to P on a triangle, returns primnum and primuvw
static void closestPointTriangle(
    const fpreal3 P,
    const fpreal3 p0,
    const fpreal3 p1,
    const fpreal3 p2,
    fpreal3 *outP,
    fpreal3 *outUVW)
{
    // Check if P in vertex region outside A
    const fpreal3 ab = p1 - p0;
    const fpreal3 ac = p2 - p0;
    const fpreal3 ap = P - p0;
    const fpreal d1 = dot(ab, ap);
    const fpreal d2 = dot(ac, ap);
    if (d1 <= 0.0f && d2 <= 0.0f)
    {
        (*outP) = p0;
        (*outUVW) = (fpreal3)(0.0f);
        return;
    }
    
    // Check if P in vertex region outside B
    const fpreal3 bp = P - p1;
    const fpreal d3 = dot(ab, bp);
    const fpreal d4 = dot(ac, bp);
    if (d3 >= 0.0f && d4 <= d3)
    {
        (*outP) = p1;
        (*outUVW) = (fpreal3)(1.0f, 0.0f, 0.0f);
        return;
    }

    // Check if P in edge region of AB, return projection of P onto AB
    fpreal vc = d1 * d4 - d3 * d2;
    if (vc <= 0.0f && d1 >= 0.0f && d3 <= 0.0f)
    {
        fpreal v = d1 / (d1 - d3);
        (*outP) = p0 + v * ab;
        (*outUVW) = (fpreal3)(v, 0.0f, 0.0f);
        return;
    }

    // Check if P in vertex region outside C
    const fpreal3 cp = P - p2;
    const fpreal d5 = dot(ab, cp);
    const fpreal d6 = dot(ac, cp);
    if (d6 >= 0.0f && d5 <= d6)
    {
        (*outP) = p2;
        (*outUVW) = (fpreal3)(0.0f, 1.0f, 0.0f);
        return;
    }

    // Check if P in edge region of AC, return projection of P onto AC
    fpreal vb = d5 * d2 - d1 * d6;
    if (vb <= 0.0f && d2 >= 0.0f && d6 <= 0.0f)
    {
        const fpreal w = d2 / (d2 - d6);
        (*outP) = p0 + w * ac;
        (*outUVW) = (fpreal3)(0.0f, w, 0.0f);
        return;
    }

    // Check if P in edge region of BC, return projection of P onto BC
    const fpreal va = d3 * d6 - d5 * d4;
    if (va <= 0.0f && (d4 - d3) >= 0.0f && (d5 - d6) >= 0.0f)
    {
        const fpreal w = (d4 - d3) / (d4 - d3 + d5 - d6);
        (*outP) = p1 + w * (p2 - p1);
        (*outUVW) = (fpreal3)(1.0f - w, w, 0.0f);
        return;
    }

    // P inside face region
    const fpreal denom = va + vb + vc;
    vb /= denom;
    vc /= denom;
    (*outP) = p0 + ab * vb + ac * vc;
    (*outUVW) = (fpreal3)(vb, vc, 0.0f);
}

// Checks if P is outside a tetrahedron
static int pointOutsideOfPlane(
    const fpreal3 P,
    const fpreal3 p0,
    const fpreal3 p1,
    const fpreal3 p2,
    const fpreal3 p3)
{
    const fpreal3 abc = cross(p1 - p0, p2 - p0);
    return dot(P - p0, abc) * dot(p3 - p0, abc) < 0.0f;
}

// Returns interpolated position based on barycentric UVW coordinates
static fpreal3 barycentric(
    const fpreal3 p0,
    const fpreal3 p1,
    const fpreal3 p2,
    const fpreal3 p3,
    const fpreal3 uvw)
{
    const fpreal u1 = 1.0f - uvw.x;
    const fpreal v1 = 1.0f - uvw.y;
    return p0 * u1 * v1 
         + p1 * u1 * uvw.y
         + p2 * uvw.x * uvw.y
         + p3 * uvw.x * v1;
}

// Squared length, prefixed in case this gets added later
static fpreal _length2(const fpreal3 v)
{
    return dot(v, v);
}

// Given P and a primnum, returns the closest P and UVW
static void _xyzdist(
    const int prim_id,
    const int typeid,
    const fpreal3 P,
    global fpreal *P_primpoints,
    global int *primpoints,
    global int *primpoints_index,
    const int primpoints_length,
    fpreal3 *closestP,
    fpreal3 *closestUVW)
{
    const int numpt = entriesAt(primpoints, prim_id);
    switch (numpt)
    {
        case 2: // Line
        {
            const int pt0 = compAt(primpoints, prim_id, 0);
            const int pt1 = compAt(primpoints, prim_id, 1);
            
            const fpreal3 p0 = vload3(pt0, P_primpoints);
            const fpreal3 p1 = vload3(pt1, P_primpoints);
            
            // Project onto clamped ab
            const fpreal3 ab = p1 - p0;
            const fpreal t = clamp(dot(P - p0, ab) / _length2(ab), 0.0f, 1.0f);

            (*closestP) = p0 + ab * t;
            (*closestUVW) = (fpreal3)(t, 0.0f, 0.0f);
            break;
        }
        case 3: // Triangle
        {
            const int pt0 = compAt(primpoints, prim_id, 0);
            const int pt1 = compAt(primpoints, prim_id, 1);
            const int pt2 = compAt(primpoints, prim_id, 2);
            
            const fpreal3 p0 = vload3(pt0, P_primpoints);
            const fpreal3 p1 = vload3(pt1, P_primpoints);
            const fpreal3 p2 = vload3(pt2, P_primpoints);

            closestPointTriangle(P, p0, p1, p2, closestP, closestUVW);
            break;
        }
        case 4:
        {
            const int pt0 = compAt(primpoints, prim_id, 0);
            const int pt1 = compAt(primpoints, prim_id, 1);
            const int pt2 = compAt(primpoints, prim_id, 2);
            const int pt3 = compAt(primpoints, prim_id, 3);
            
            const fpreal3 p0 = vload3(pt0, P_primpoints);
            const fpreal3 p1 = vload3(pt1, P_primpoints);
            const fpreal3 p2 = vload3(pt2, P_primpoints);
            const fpreal3 p3 = vload3(pt3, P_primpoints);

            if (typeid == 21) // Tetrahedron
            {
                fpreal3 tmpP, tmpUVW;
                fpreal bestDist = FLT_MAX;
                
                // If point outside face abc then compute closest point on abc
                if (pointOutsideOfPlane(P, p0, p1, p2, p3))
                {
                    closestPointTriangle(P, p0, p1, p2, &tmpP, &tmpUVW);
                    const fpreal dist = _length2(tmpP - P);
                    if (dist < bestDist)
                    {
                        bestDist = dist;
                        (*closestP) = tmpP;
                        (*closestUVW) = tmpUVW;
                    }
                }
                
                // Repeat test for face acd
                if (pointOutsideOfPlane(P, p0, p2, p3, p1))
                {
                    closestPointTriangle(P, p0, p2, p3, &tmpP, &tmpUVW);
                    const fpreal dist = _length2(tmpP - P);
                    if (dist < bestDist)
                    {
                        bestDist = dist;
                        (*closestP) = tmpP;
                        (*closestUVW) = tmpUVW.zxy;
                    }
                }
                
                // Repeat test for face adb
                if (pointOutsideOfPlane(P, p0, p3, p1, p2))
                {
                    closestPointTriangle(P, p0, p3, p1, &tmpP, &tmpUVW);
                    const fpreal dist = _length2(tmpP - P);
                    if (dist < bestDist)
                    {
                        bestDist = dist;
                        (*closestP) = tmpP;
                        (*closestUVW) = tmpUVW.yzx;
                    }
                }
                
                // Repeat test for face bdc
                if (pointOutsideOfPlane(P, p1, p3, p2, p0))
                {
                    closestPointTriangle(P, p1, p3, p2, &tmpP, &tmpUVW);
                    const fpreal dist = _length2(tmpP - P);
                    if (dist < bestDist)
                    {
                        bestDist = dist;
                        (*closestP) = tmpP;
                        (*closestUVW) = (fpreal3)(1.0f - tmpUVW.x - tmpUVW.y, tmpUVW.y, tmpUVW.x);
                    }
                }
                
                // Interpolate inside interior
                if (bestDist >= FLT_MAX)
                {
                    const fpreal3 e1 = p1 - p0;
                    const fpreal3 e2 = p2 - p0;
                    const fpreal3 e3 = p3 - p0;
                    const fpreal3 d = P - p0;
                    const fpreal u = dot(d,  cross(e2, e3));
                    const fpreal v = dot(e1, cross(d,  e3));
                    const fpreal w = dot(e1, cross(e2, d ));
                    (*closestP) = P;
                    (*closestUVW) = (fpreal3)(u, v, w) / dot(e1, cross(e2, e3));
                }
            }
            else // Quadrilateral
            {
                // Houdini uses bilinear interpolation for quads, this is hard to solve
                // Below is a simple least squares approximation for good performance
                const fpreal3 edge_v1 = p2 - p1;
                const fpreal3 edge_v0 = p3 - p0;
                const fpreal3 edge_u0 = p1 - p0;
                const fpreal3 edge_u1 = p2 - p3;
                
                const fpreal tolerance = 1e-8f;
                (*closestUVW) = (fpreal3)(0.5f, 0.5f, 0.0f);
                
                for (int i = 0; i < 8; ++i) {
                    const fpreal3 r = barycentric(p0, p1, p2, p3, (*closestUVW)) - P;
                    if (_length2(r) < tolerance) break;
                    
                    const fpreal3 dP_du = mix(edge_v0, edge_v1, closestUVW->y);
                    const fpreal3 dP_dv = mix(edge_u0, edge_u1, closestUVW->x);
                    const fpreal A11 = dot(dP_du, dP_du);
                    const fpreal A12 = dot(dP_du, dP_dv);
                    const fpreal A22 = dot(dP_dv, dP_dv);
                    
                    const fpreal2 grad = (fpreal2)(dot(dP_du, r), dot(dP_dv, r));
                    if (dot(grad, grad) < tolerance) break;
                    
                    const fpreal det = A11 * A22 - A12 * A12;
                    if (fabs(det) < tolerance) break;
                    
                    closestUVW->x -= (grad.x * A22 - grad.y * A12) / det;
                    closestUVW->y -= (grad.y * A11 - grad.x * A12) / det;
                }
                
                closestUVW->x = clamp(closestUVW->x, 0.0f, 1.0f);
                closestUVW->y = clamp(closestUVW->y, 0.0f, 1.0f);
                (*closestP) = barycentric(p0, p1, p2, p3, (*closestUVW));
            }
            break;
        }
        default: // General case
        {
            fpreal3 tmpP, tmpUVW;
            int closestI;
            fpreal bestDist = FLT_MAX;
            
            // Add an imaginary triangle point in the center
            // Nicer to store positions here, but OpenCL doesn't support dynamic arrays
            fpreal3 avgPos = (fpreal3)(0.0f);
            for (int i = 0; i < numpt; ++i)
            {
                const int pt = compAt(primpoints, prim_id, i);
                const fpreal3 pos = vload3(pt, P_primpoints);
                avgPos += pos / numpt;
            }
            
            // Triangle fan around the center
            const int prevPt = compAt(primpoints, prim_id, 0);
            fpreal3 prevPos = vload3(prevPt, P_primpoints);
            for (int i = 0; i < numpt; ++i)
            {
                const int nextPt = compAt(primpoints, prim_id, (i + 1) % numpt);
                const fpreal3 nextPos = vload3(nextPt, P_primpoints);
                closestPointTriangle(P, avgPos, prevPos, nextPos, &tmpP, &tmpUVW);
                const fpreal dist = _length2(P - tmpP);
                if (dist < bestDist) 
                {
                    bestDist = dist;
                    closestI = i;
                    (*closestP) = tmpP;
                    (*closestUVW) = tmpUVW;
                }
                prevPos = nextPos;
            }
            
            // Gradient around the center
            const fpreal u = (closestUVW->y / (closestUVW->x + closestUVW->y) + closestI) / numpt;
            // Gradient out from the center
            const fpreal v = 1.0f - closestUVW->x - closestUVW->y;
            (*closestUVW) = (fpreal3)(u, v, 0.0f);
            break;
        }
    }
}

kernel void testXyzdist( 
    const int _bound_P_length,
    global fpreal * restrict _bound_P,
    const int _bound_P_primpoints_length,
    global fpreal * restrict _bound_P_primpoints,
    const int _bound_P_centroids_length,
    global fpreal * restrict _bound_P_centroids,
    const int _bound_nearprims_length,
    global int * restrict _bound_nearprims_index,
    global int * restrict _bound_nearprims,
    const int _bound_primpoints_length,
    global int * restrict _bound_primpoints_index,
    global int * restrict _bound_primpoints,
    const int _bound_typeid_length,
    global int * restrict _bound_typeid,
    const int _bound_Cd_length,
    global fpreal * restrict _bound_Cd
)
{
    const int idx = get_global_id(0);
    if (idx >= _bound_P_length) return;

    const fpreal3 P = vload3(idx, _bound_P);
    fpreal3 tmpP, tmpUVW, closestP, closestUVW;
    fpreal bestDist = FLT_MAX;
    
    // Check through the closest few prims for the nearest match
    const int numprims = entriesAt(_bound_nearprims, idx);
    for (int i = 0; i < numprims; ++i)
    {
        const int prim_id = compAt(_bound_nearprims, idx, i);
        const int typeid = _bound_typeid[prim_id];
        _xyzdist(prim_id, typeid, P, _bound_P_primpoints,
                _bound_primpoints, _bound_primpoints_index, _bound_primpoints_length, &tmpP, &tmpUVW);
        
        const fpreal dist = _length2(tmpP - P);
        if (dist < bestDist) {
            bestDist = dist;
            closestP = tmpP;
            closestUVW = tmpUVW;
        }
    }
    
    vstore3(closestP, idx, _bound_P);
    vstore3(closestUVW, idx, _bound_Cd);
}
```

## `primuv()`

The functions below are designed for `@P`, but can be used on any other attribute by swapping `vector` to that attribute's type.

| [Download the HIP file!](./hips/primuv_diy.hiplc?raw=true) |
| --- |

### `primuv()` in VEX

```js
vector primuv_diy(int geo; string attr; int prim; vector uvw) {

    int typeid = primintrinsic(geo, "typeid", prim);
    int pts[] = primpoints(geo, prim);
    int numpt = len(pts);
    float u = uvw.x;
    float v = uvw.y;
    float w = uvw.z;
    
    if (numpt == 2) {
        // Line
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        return p0 * (1 - u) +
               p1 * u;
    } else if (numpt == 3) {
        // Triangle
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        return p0 * (1 - u - v) +
               p1 * u +
               p2 * v;
    } else if (numpt == 4 && typeid == 21) {
        // Tetrahedron
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        vector p3 = point(geo, attr, pts[3]);
        return p0 * (1 - u - v - w) +
               p1 * u +
               p2 * v +
               p3 * w;
    } else if (numpt == 4) {
        // Quadrilateral
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        vector p3 = point(geo, attr, pts[3]);
        float u1 = 1 - u;
        float v1 = 1 - v;
        return p0 * u1 * v1 +
               p1 * u1 * v +
               p2 * u * v +
               p3 * u * v1;
    } else {
        // General case
        vector pos = 0;
        float offset = u * numpt;
        int first = floor(offset);
        int last = (first + 1) % numpt;
        float blend = frac(offset);
        
        float weight_a = v / numpt;
        float weight_b = (1 - blend) * (1 - v);
        float weight_c = blend * (1 - v);
        
        for (int i = 0; i < numpt; ++i) {
            vector p = point(geo, attr, pts[i]);
            pos += p * (weight_a +
                        weight_b * (i == first) +
                        weight_c * (i == last));
        }
        return pos;
    }
}

// Example usage
v@P = primuv_diy(1, "P", i@hitprim, v@hitprimuv);
```

### `primuv()` in OpenCL

```c
#define entriesAt(_arr_, _idx_) ((_idx_ >= 0 && _idx_ < _arr_##_length) ? (_arr_##_index[_idx_+1] - _arr_##_index[_idx_]) : 0)
#define compAt(_arr_, _idx_, _compidx_) ((_idx_ >= 0 && _idx_ < _arr_##_length && _compidx_ >= 0 && _compidx_ < entriesAt(_arr_, _idx_)) ? _arr_[_arr_##_index[_idx_] + _compidx_] : 0)

static fpreal3 _primuv(
    global fpreal *P,
    global int *primpoints,
    global int *primpoints_index,
    const int primpoints_length,
    const int prim,
    const fpreal3 uvw,
    const int typeid)
{
    const int numpt = entriesAt(primpoints, prim);
    const fpreal u = uvw.x;
    const fpreal v = uvw.y;
    const fpreal w = uvw.z;
    
    switch (numpt)
    {
        case 2: // Line
        {
            const int pt0 = compAt(primpoints, prim, 0);
            const int pt1 = compAt(primpoints, prim, 1);
            
            const fpreal3 p0 = vload3(pt0, P);
            const fpreal3 p1 = vload3(pt1, P);
            
            return p0 * (1.0f - u) +
                   p1 * u;
        }
        case 3: // Triangle
        {
            const int pt0 = compAt(primpoints, prim, 0);
            const int pt1 = compAt(primpoints, prim, 1);
            const int pt2 = compAt(primpoints, prim, 2);
            
            const fpreal3 p0 = vload3(pt0, P);
            const fpreal3 p1 = vload3(pt1, P);
            const fpreal3 p2 = vload3(pt2, P);
            
            return p0 * (1.0f - u - v)
                 + p1 * u
                 + p2 * v;
        }
        case 4:
        {
            const int pt0 = compAt(primpoints, prim, 0);
            const int pt1 = compAt(primpoints, prim, 1);
            const int pt2 = compAt(primpoints, prim, 2);
            const int pt3 = compAt(primpoints, prim, 3);
            
            const fpreal3 p0 = vload3(pt0, P);
            const fpreal3 p1 = vload3(pt1, P);
            const fpreal3 p2 = vload3(pt2, P);
            const fpreal3 p3 = vload3(pt3, P);
            
            if (typeid == 21) // Tetrahedron
            {
                return p0 * (1.0f - u - v - w) +
                       p1 * u +
                       p2 * v +
                       p3 * w;
            }
            else // Quadrilateral
            {
                const fpreal u1 = 1.0f - u;
                const fpreal v1 = 1.0f - v;
                return p0 * u1 * v1 +
                       p1 * u1 * v +
                       p2 * u * v +
                       p3 * u * v1;
            }
        }
        default: // General case
        {
            fpreal3 pos = (fpreal3)(0.0f);
            const fpreal offset = u * numpt;
            const int first = floor(offset);
            const int last = (first + 1) % numpt;
            const fpreal blend = offset - floor(offset);
            
            const fpreal weight_a = v / numpt;
            const fpreal weight_b = (1.0f - blend) * (1.0f - v);
            const fpreal weight_c = blend * (1.0f - v);
            
            for (int i = 0; i < numpt; ++i) {
                const int pt = compAt(primpoints, prim, i);
                const fpreal3 p = vload3(pt, P);
                pos += p * (weight_a +
                            weight_b * (i == first) +
                            weight_c * (i == last));
            }
            return pos;
        }
    }
}

kernel void testPrimuv( 
    const int _bound_P_length,
    global fpreal * restrict _bound_P,
    const int _bound_P2_length,
    global fpreal * restrict _bound_P2,
    const int _bound_primpoints_length,
    global int * restrict _bound_primpoints_index,
    global int * restrict _bound_primpoints,
    const int _bound_typeid_length,
    global int * restrict _bound_typeid,
    const int _bound_hitprim_length,
    global int * restrict _bound_hitprim,
    const int _bound_hitprimuv_length,
    global fpreal * restrict _bound_hitprimuv
)
{
    const int idx = get_global_id(0);
    if (idx >= _bound_P_length) return;
    
    const int prim = _bound_hitprim[idx];
    const fpreal3 primuv = vload3(idx, _bound_hitprimuv);
    const int typeid = _bound_typeid[prim];
    
    const fpreal3 P = _primuv(_bound_P2, _bound_primpoints, _bound_primpoints_index, _bound_primpoints_length, prim, primuv, typeid);
    vstore3(P, idx, _bound_P);
}
```

### `primuv()` with normals in VEX

Since I was using this code for collision handling, I also needed the surface normals.

Note the tetrahedral normals don't match Houdini. I wanted them to be planar, so they face outwards instead of interpolating.

| [Download the HIP file!](./hips/primuvn_diy.hiplc?raw=true) |
| --- |

```js
void primuv_diy(int geo; string attr; int prim; vector uvw; vector outP; vector outN) {

    int typeid = primintrinsic(geo, "typeid", prim);
    int pts[] = primpoints(geo, prim);
    int numpt = len(pts);
    float u = uvw.x;
    float v = uvw.y;
    float w = uvw.z;
    
    if (numpt == 2) {
        // Line
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        outP = p0 * (1 - u) +
               p1 * u;
        // Lines don't really have normals
        outN = 0;
    } else if (numpt == 3) {
        // Triangle
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        outN = normalize(cross(p0 - p2, p0 - p1));
        outP = p0 * (1 - u - v) +
               p1 * u +
               p2 * v;
    } else if (numpt == 4 && typeid == 21) {
        // Tetrahedron
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        vector p3 = point(geo, attr, pts[3]);
        float k = 1 - u - v - w;
        outP = p0 * k +
               p1 * u +
               p2 * v +
               p3 * w;
        // For collisions, use planar normals instead of interpolated normals
        // Smallest weight determines the normal
        int min_face = 0;
        float min_weight = u;
        if (v < min_weight) { min_weight = v; min_face = 1; }
        if (w < min_weight) { min_weight = w; min_face = 2; }
        if (k < min_weight) { min_weight = k; min_face = 3; }
        if (min_face == 0) {
            outN = normalize(cross(p3 - p0, p3 - p2));
        } else if (min_face == 1) {
            outN = normalize(cross(p1 - p0, p1 - p3));
        } else if (min_face == 2) {
            outN = normalize(cross(p1 - p2, p1 - p0));
        } else {
            outN = normalize(cross(p1 - p3, p1 - p2));
        }
    } else if (numpt == 4) {
        // Quadrilateral
        vector p0 = point(geo, attr, pts[0]);
        vector p1 = point(geo, attr, pts[1]);
        vector p2 = point(geo, attr, pts[2]);
        vector p3 = point(geo, attr, pts[3]);
        float u1 = 1 - u;
        float v1 = 1 - v;
        outP = p0 * u1 * v1 +
               p1 * u1 * v +
               p2 * u * v +
               p3 * u * v1;
        // Average the normals of both triangles
        outN = normalize(cross(p0 - p2, p0 - p1) + cross(p0 - p3, p0 - p2));
    } else {
        // General case
        vector pos = 0, N = 0;
        float offset = u * numpt;
        int first = floor(offset);
        int last = (first + 1) % numpt;
        float blend = frac(offset);
        
        float weight_a = v / numpt;
        float weight_b = (1 - blend) * (1 - v);
        float weight_c = blend * (1 - v);
        
        vector prev_pos = point(geo, attr, pts[0]);
        pos += prev_pos * (weight_a +
                           weight_b * (0 == first) +
                           weight_c * (0 == last));
        
        for (int i = 1; i <= numpt; ++i) {
            vector p = point(geo, attr, pts[i % numpt]);
            if (i < numpt) {
                pos += p * (weight_a +
                            weight_b * (i == first) +
                            weight_c * (i == last));
            }
            // Newell's method
            N.x -= (prev_pos.y - p.y) * (prev_pos.z + p.z);
            N.y -= (prev_pos.z - p.z) * (prev_pos.x + p.x);
            N.z -= (prev_pos.x - p.x) * (prev_pos.y + p.y);
            prev_pos = p;
        }
        
        outN = normalize(N);
        outP = pos;
    }
}

// Example usage
primuv_diy(1, "P", i@hitprim, v@hitprimuv, v@P, v@N);
```
